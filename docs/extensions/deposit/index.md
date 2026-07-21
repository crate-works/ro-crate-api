---
title: "Extension: deposit"
mdx.format: md
---

# The `deposit` Extension

The core specification is read-only: entities, crates, files, and search. The
`deposit` extension adds the write pathway. It is deliberately not
entity-level CRUD — all writes flow through **deposit sessions** against
**storage objects**, and catalog entities remain read-only projections that
the implementation derives from what was deposited.

The extension covers three things as one unit:

- the **deposit session** endpoints — open a deposit, stage an RO-Crate and
  files, finalise
- the **storage-object read surface** — list, retrieve, and delete storage
  objects, and fetch a deposited crate verbatim
- the **linkage fields** added to core schemas (`storageObjectIds` on
  [Entity](/docs/api/schemas/entity), `storageObjectId` on
  [File](/docs/api/schemas/file))

The read surface is not separable from the write pathway: readable storage
objects without a deposit pathway is not a state this specification supports.

## Why Storage Objects, Not Entity Writes

A single RO-Crate routinely describes many catalog entities — a collection,
its items, every file they contain, and the people and organisations
connected to them; the entity model is extensible, so the list doesn't end
there. Entity-granular
write endpoints would force depositors to decompose a crate they already have
into a sequence of per-entity calls, and force the API to referee partial
failures across that sequence.

The unit a depositor actually holds is the crate plus its files. The
extension makes that the unit of deposit:

> A **storage object** is the real thing on disk — an RO-Crate plus all the
> files it references, deposited and stored as a unit.

Depositors send storage objects; the implementation **materialises** catalog
entities from them by its own rules. There are no entity write endpoints
(with one narrow exception for
[orphaned entities](./lifecycle#deleting-a-contributor-less-entity)).

## Materialisation

How a storage object becomes catalog entities is **implementation-defined**.
The contract is only that once a deposit reports `complete`, the entities
materialised from it are readable, and the linkage fields below let clients
traverse between the two surfaces. Two real archives illustrate how much the
rules can differ:

- **PARADISEC**: a storage object is one item's crate and its media files.
  Materialisation yields the item entity plus one file entity per media
  file — a small, fixed shape.
- **LDaCA**: a storage object may be a whole corpus crate. Materialisation
  explodes it into collection, item, file, person, and organisation
  entities — one deposit, many entities.

Materialisation can also be **many-to-one**: several storage objects may
contribute to a single merged entity. If two deposited crates both describe
the same speaker (same `@id`), an implementation may materialise one Person
entity carrying both storage objects in its `storageObjectIds`. This merge
machinery is also what makes
[curation storage objects](./lifecycle#enriching-entities-curation-storage-objects)
work.

Because entities are projections, re-materialisation can change them. A later
deposit — of the same storage object or a different one — may add, alter, or
remove entities. Whether entities that lose their last contributor are pruned
or retained is likewise implementation-defined (see
[Deletion & Lifecycle](./lifecycle)).

## The Linkage Fields

The two surfaces — deposited storage objects and materialised entities — are
bidirectionally linked:

- **`entityIds`** on a [storage object](/docs/api/schemas/storageobject): the
  entities materialised from its current version. This is the authoritative
  answer to "what did my deposit create", and it can change as later deposits
  alter the materialisation.
- **`storageObjectIds`** on an [Entity](/docs/api/schemas/entity): the
  storage objects whose current versions contribute to the entity. Usually
  one; more when materialisation merges contributions.
- **`storageObjectId`** on a [File](/docs/api/schemas/file): the storage
  object whose deposit supplied the file's bytes. Singular, because bytes
  arrive in exactly one deposit. Optional, to accommodate files predating any
  storage object.

With the extension present, `GET /entity/{id}/rocrate` is specified as
implementation-defined in provenance: the document may be a stored crate or a
view derived from the crate(s) of the entity's contributing storage objects,
but it MUST always be a valid RO-Crate whose root data entity describes the
entity. To retrieve an original deposited crate verbatim, use
[`GET /storage-object/{id}/crate`](/docs/api/get-storage-object-crate).

## Declaring the Capability

An implementation that provides deposit declares the extension in
[`/capabilities`](/docs/getting-started/capabilities). Presence of the
`deposit` key is how clients discover write support — there is no separate
write flag:

```json
{
  "apiVersion": "0.3.0",
  "extensions": {
    "deposit": {
      "idMinting": "both",
      "fileUpload": ["inline", "presigned"],
      "tombstonePolicy": "410",
      "depositTtlSeconds": 604800,
      "maxFileSizeBytes": 5368709120
    }
  }
}
```

- **`idMinting`** (required): who mints storage-object IDs — `client`
  (depositor proposes), `server` (implementation mints), or `both` (client
  may propose, server fills gaps).
- **`fileUpload`** (required): the staging modes supported, a set drawn from
  `inline` (bytes in the staging request) and `presigned` (metadata in the
  staging request, bytes uploaded directly to a returned target). Future
  modes may be added; clients ignore values they do not recognise.
- **`tombstonePolicy`** (required): `"410"` or `"404"` — what deleted
  resource URIs return. One policy covers storage objects and the entity
  knock-on alike; see [Deletion & Lifecycle](./lifecycle#tombstones).
- **`depositTtlSeconds`** (optional): the expiry horizon for abandoned
  deposits. Absent means expiry is implementation-defined — don't rely on a
  particular window.
- **`maxFileSizeBytes`** (optional): the largest file a deposit may stage.
  Absent means no declared limit.

Deliberately *not* declared: whether finalise runs synchronously or
asynchronously (server's discretion per request — one client code path
handles both), and the storage-object visibility pattern (observable through
behaviour; clients don't branch on it before acting).

### Authentication

Deposit and storage-object write operations require the coarse OAuth2
`write` scope (see [Authentication](/docs/getting-started/authentication)).
Finer-grained authorisation — who may deposit what — is
implementation-defined and expressed through ordinary `403` responses.

## Storage-Object Visibility

Whether storage objects are readable beyond their depositor is
**implementation-defined**. Implementations should choose one of two named
patterns and apply it consistently:

- **Public surface**: storage objects are listable and retrievable by
  anyone; each carries an entity-style `access` object saying whether the
  caller may fetch the crate. Recommended derivation: grant crate access
  only if the caller has metadata access to *every* entity materialised from
  the object, since the deposited crate is the union of their metadata.
  *Pros*: provenance is publicly traversable (any reader can follow
  `storageObjectIds` to the source of truth); citations to deposited crates
  resolve for everyone. *Cons*: access derivation must be computed and kept
  consistent with entity-level access; the surface must be hardened like any
  public catalog surface.
- **Depositor-only surface**: storage-object reads require the `write`
  scope — a management surface for depositors and curators, not a catalog
  surface. *Pros*: simple to reason about; no access derivation. *Cons*:
  provenance links are dead ends for ordinary readers; the deposited crate
  is not citable as a public artefact.

Either way the storage-object resource carries the `access` object, so
client code is identical under both patterns.

## The Endpoints

| Operation | Purpose |
| --- | --- |
| [`POST /deposits`](/docs/api/create-deposit) | Open a deposit for a new storage object |
| [`POST /storage-object/{id}/deposits`](/docs/api/create-update-deposit) | Open an update deposit for an existing storage object |
| [`GET /deposit/{id}`](/docs/api/get-deposit) | Deposit state, staged files, recorded errors; the polling resource |
| [`PUT /deposit/{id}/crate`](/docs/api/stage-deposit-crate) | Stage the RO-Crate (full replace) |
| [`PUT /deposit/{id}/file/{fileId}`](/docs/api/stage-deposit-file) | Stage a file (inline or presigned) |
| [`DELETE /deposit/{id}/file/{fileId}`](/docs/api/unstage-deposit-file) | Remove a staged file |
| [`POST /deposit/{id}/finalise`](/docs/api/finalise-deposit) | Validate, publish, materialise |
| [`DELETE /deposit/{id}`](/docs/api/abort-deposit) | Abort an open deposit |
| [`GET /storage-objects`](/docs/api/list-storage-objects) | List storage objects |
| [`GET /storage-object/{id}`](/docs/api/get-storage-object) | Retrieve a storage object |
| [`GET /storage-object/{id}/crate`](/docs/api/get-storage-object-crate) | The deposited crate, verbatim |
| [`DELETE /storage-object/{id}`](/docs/api/delete-storage-object) | Delete a storage object |
| [`DELETE /entity/{id}`](/docs/api/delete-entity) | Delete a contributor-less entity |

The guides walk through the flows:

- [Depositing](./depositing) — create → stage → finalise, both upload modes
- [Updating a Storage Object](./updating) — the carry-forward delta model
- [Deletion & Lifecycle](./lifecycle) — deleting, tombstones, enrichment

## Client Rules

- **Feature-detect before use**: check `"deposit" in
  capabilities.extensions` and read the declared modes rather than probing.
- **One code path for finalise**: branch on the deposit's returned `state`,
  not on an expectation of sync or async behaviour.
- **Treat `entityIds` as live**: the materialised-entity list reflects the
  *current* materialisation and can change as other deposits land.
- **Degrade gracefully**: the linkage fields are extension properties;
  absent means the implementation (or that resource) doesn't carry them,
  not that the resource is invalid.
