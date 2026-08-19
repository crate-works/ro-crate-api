---
title: Deposits
mdx.format: md
---

# Deposits and RO-Crates

Reading — entities, their metadata, files, and search — is mandatory for every
implementation. Deposit is the write pathway, and it is **optional core**:
part of the core specification, but a read-only catalog remains fully
conformant without it. It is deliberately not entity-level CRUD — all writes
flow through **deposit sessions** against **RO-Crates**, and catalog
entities remain read-only projections that the implementation derives from
what was deposited.

Deposit is not an extension. It is not keyed in `capabilities.extensions`,
and its fields carry no `x-extension` annotation; instead every
implementation declares its position in a
[required `deposit` capability block](#declaring-the-capability).

The deposit surface covers three things as one unit:

- the **deposit session** endpoints — open a deposit, stage a metadata
  document and files, finalise
- the **RO-Crate read surface** — list, retrieve, and delete RO-Crates, and
  fetch a deposited metadata document verbatim
- the **linkage fields** on the read schemas (`roCrateIds` on
  [Entity](/docs/api/schemas/entity), `roCrateId` on
  [File](/docs/api/schemas/file))

The read surface is not separable from the write pathway: readable RO-Crates
without a deposit pathway is not a state this specification supports.

## Why RO-Crates, Not Entity Writes

A single RO-Crate routinely describes many catalog entities — a collection,
its items, every file they contain, and the people and organisations
connected to them; the entity model is extensible, so the list doesn't end
there. Entity-granular write endpoints would force depositors to decompose an
RO-Crate they already hold into a sequence of per-entity calls, and force the
API to referee partial failures across that sequence.

The unit a depositor actually holds is the whole RO-Crate. The specification
makes that the unit of deposit:

> An **RO-Crate** is the real thing on disk — a metadata document plus all
> the files it references, deposited and stored as a unit.

Depositors send RO-Crates; the implementation **materialises** catalog
entities from them by its own rules. There are no entity write endpoints
(with one narrow exception for
[orphaned entities](./lifecycle#deleting-a-contributor-less-entity)).

## Materialisation

How an RO-Crate becomes catalog entities is **implementation-defined**.
The contract is only that once a deposit reports `complete`, the entities
materialised from it are readable, and the linkage fields below let clients
traverse between the two surfaces. Two real archives illustrate how much the
rules can differ:

- **PARADISEC**: an RO-Crate is one item's crate and its media files.
  Materialisation yields the item entity plus one file entity per media
  file — a small, fixed shape.
- **LDaCA**: an RO-Crate may be a whole corpus crate. Materialisation
  explodes it into collection, item, file, person, and organisation
  entities — one deposit, many entities.

Materialisation can also be **many-to-one**: several RO-Crates may
contribute to a single merged entity. If two deposited RO-Crates both describe
the same speaker (same `@id`), an implementation may materialise one Person
entity carrying both RO-Crates in its `roCrateIds`. This merge
machinery is also what makes
[curation RO-Crates](./lifecycle#enriching-entities-curation-ro-crates)
work.

Because entities are projections, re-materialisation can change them. A later
deposit — of the same RO-Crate or a different one — may add, alter, or
remove entities. Whether entities that lose their last contributor are pruned
or retained is likewise implementation-defined (see
[Deletion & Lifecycle](./lifecycle)).

## The Linkage Fields

The two surfaces — deposited RO-Crates and materialised entities — are
bidirectionally linked:

- **`entityIds`** on a [RO-Crate](/docs/api/schemas/rocrate): the
  entities materialised from its current version. This is the authoritative
  answer to "what did my deposit create", and it can change as later deposits
  alter the materialisation.
- **`roCrateIds`** on an [Entity](/docs/api/schemas/entity): the
  RO-Crates whose current versions contribute to the entity. Usually
  one; more when materialisation merges contributions.
- **`roCrateId`** on a [File](/docs/api/schemas/file): the RO-Crate whose
  deposit supplied the file's bytes. Singular, because bytes
  arrive in exactly one deposit. Optional, to accommodate files predating any
  RO-Crate.

Where deposit is supported, `GET /entity/{id}/rocrate` is specified as
implementation-defined in provenance: the document may be a stored metadata document or a
view derived from the crate(s) of the entity's contributing RO-Crates,
but it MUST always be a valid RO-Crate whose root data entity describes the
entity. To retrieve an original deposited metadata document verbatim, use
[`GET /ro-crate/{id}/metadata`](/docs/api/get-ro-crate-metadata).

## Declaring the Capability

The `deposit` block in [`/capabilities`](/docs/getting-started/capabilities)
is **required of every implementation** — read-only catalogs included. A
client never has to infer read-only-ness from a missing key; each
implementation says where it stands:

```json
{
  "apiVersion": "0.3.0",
  "deposit": {
    "supported": true,
    "idMinting": "both",
    "fileUpload": ["inline", "presigned"],
    "tombstonePolicy": "410",
    "depositTtlSeconds": 604800,
    "maxFileSizeBytes": 5368709120
  }
}
```

A read-only catalog declares the block just as plainly:

```json
{
  "apiVersion": "0.3.0",
  "deposit": { "supported": false }
}
```

- **`supported`** (required): whether the deposit and RO-Crate
  endpoints are provided. This is the single flag clients check. When it is
  `false`, the remaining fields are omitted.
- **`idMinting`** (required): who mints RO-Crate IDs — `client`
  (depositor proposes), `server` (implementation mints), or `both` (client
  may propose, server fills gaps).
- **`fileUpload`** (required): the staging modes supported, a set drawn from
  `inline` (bytes in the staging request) and `presigned` (metadata in the
  staging request, bytes uploaded directly to a returned target). Future
  modes may be added; clients ignore values they do not recognise.
- **`tombstonePolicy`** (required): `"410"` or `"404"` — what deleted
  resource URIs return. One policy covers RO-Crates and the entity
  knock-on alike; see [Deletion & Lifecycle](./lifecycle#tombstones).
- **`depositTtlSeconds`** (optional): the expiry horizon for abandoned
  deposits. Absent means expiry is implementation-defined — don't rely on a
  particular window.
- **`maxFileSizeBytes`** (optional): the largest file a deposit may stage.
  Absent means no declared limit.

The three required detail fields are required only when `supported` is
`true`; they carry no meaning for a read-only catalog and MUST be omitted
there.

Deliberately *not* declared: whether finalise runs synchronously or
asynchronously (server's discretion per request — one client code path
handles both), and the RO-Crate visibility pattern (observable through
behaviour; clients don't branch on it before acting).

### Authentication

Deposit and RO-Crate write operations require the coarse OAuth2
`write` scope (see [Authentication](/docs/getting-started/authentication)).
Finer-grained authorisation — who may deposit what — is
implementation-defined and expressed through ordinary `403` responses.

## RO-Crate Visibility

Whether RO-Crates are readable beyond their depositor is
**implementation-defined**. Implementations should choose one of two named
patterns and apply it consistently:

- **Public surface**: RO-Crates are listable and retrievable by
  anyone; each carries an entity-style `access` object saying whether the
  caller may fetch the metadata document. Recommended derivation: grant metadata access
  only if the caller has metadata access to *every* entity materialised from
  the object, since the deposited metadata document is the union of their metadata.
  *Pros*: provenance is publicly traversable (any reader can follow
  `roCrateIds` to the source of truth); citations to deposited crates
  resolve for everyone. *Cons*: access derivation must be computed and kept
  consistent with entity-level access; the surface must be hardened like any
  public catalog surface.
- **Depositor-only surface**: RO-Crate reads require the `write`
  scope — a management surface for depositors and curators, not a catalog
  surface. *Pros*: simple to reason about; no access derivation. *Cons*:
  provenance links are dead ends for ordinary readers; the deposited metadata document
  is not citable as a public artefact.

Either way the RO-Crate resource carries the `access` object, so
client code is identical under both patterns.

## The Endpoints

| Operation | Purpose |
| --- | --- |
| [`POST /deposits`](/docs/api/create-deposit) | Open a deposit for a new RO-Crate |
| [`POST /ro-crate/{id}/deposits`](/docs/api/create-update-deposit) | Open an update deposit for an existing RO-Crate |
| [`GET /deposit/{id}`](/docs/api/get-deposit) | Deposit state, staged files, recorded errors; the polling resource |
| [`PUT /deposit/{id}/metadata`](/docs/api/stage-deposit-metadata) | Stage the RO-Crate (full replace) |
| [`PUT /deposit/{id}/file/{fileId}`](/docs/api/stage-deposit-file) | Stage a file (inline or presigned) |
| [`DELETE /deposit/{id}/file/{fileId}`](/docs/api/unstage-deposit-file) | Remove a staged file |
| [`POST /deposit/{id}/finalise`](/docs/api/finalise-deposit) | Validate, publish, materialise |
| [`DELETE /deposit/{id}`](/docs/api/abort-deposit) | Abort an open deposit |
| [`GET /ro-crates`](/docs/api/list-ro-crates) | List RO-Crates |
| [`GET /ro-crate/{id}`](/docs/api/get-ro-crate) | Retrieve an RO-Crate |
| [`GET /ro-crate/{id}/metadata`](/docs/api/get-ro-crate-metadata) | The deposited crate, verbatim |
| [`DELETE /ro-crate/{id}`](/docs/api/delete-ro-crate) | Delete an RO-Crate |
| [`DELETE /entity/{id}`](/docs/api/delete-entity) | Delete a contributor-less entity |

The guides walk through the flows:

- [Depositing](./depositing) — create → stage → finalise, both upload modes
- [Updating an RO-Crate](./updating) — the carry-forward delta model
- [Deletion & Lifecycle](./lifecycle) — deleting, tombstones, enrichment

## Client Rules

- **Feature-detect before use**: check `capabilities.deposit.supported` and
  read the declared modes rather than probing.
- **One code path for finalise**: branch on the deposit's returned `state`,
  not on an expectation of sync or async behaviour.
- **Treat `entityIds` as live**: the materialised-entity list reflects the
  *current* materialisation and can change as other deposits land.
- **Degrade gracefully**: the linkage fields are optional; absent means the
  implementation (or that resource) doesn't carry them, not that the
  resource is invalid.
