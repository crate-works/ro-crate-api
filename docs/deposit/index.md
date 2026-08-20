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

Every implementation declares its position in a
[required `deposit` capability block](#declaring-the-capability).

The deposit surface covers three things as one unit:

- the **deposit session** endpoints — open a deposit, stage a metadata
  document and files, finalise
- the **RO-Crate read surface** — list, retrieve, and delete RO-Crates, and
  fetch a deposited metadata document verbatim
- the **linkage fields** on the read schemas (`roCrateIds` on
  [Entity](/docs/api/schemas/entity), `roCrateId` on
  [File](/docs/api/schemas/file))

## Why RO-Crates, Not Entity Writes

The unit a depositor actually holds is the whole RO-Crate, so that is the
unit of deposit:

> An **RO-Crate** is a metadata document plus all the files it references,
> deposited and stored as a unit.

Depositors send RO-Crates; the implementation **materialises** catalog
entities from them by its own rules. There are no entity write endpoints
(with one narrow exception for
[orphaned entities](./lifecycle#deleting-a-contributor-less-entity)). The
[announcement post](/blog/deposits) sets out the reasoning.

## Materialisation

How an RO-Crate becomes catalog entities is **implementation-defined**.
The contract is only that once a deposit reports `complete`, the entities
materialised from it are readable, and the linkage fields below let clients
traverse between the two surfaces. Two real archives illustrate how much the
rules can differ:

- **PARADISEC**: an RO-Crate is one item's metadata document and its media
  files. Materialisation yields the item entity plus one file entity per
  media file — a small, fixed shape.
- **LDaCA**: an RO-Crate may describe a whole corpus. Materialisation
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

`GET /entity/{id}/metadata` has implementation-defined provenance: the
document may be a stored metadata document, or a view derived from the
metadata documents of the entity's contributing RO-Crates. Either way it
MUST be a valid RO-Crate whose root data entity describes the entity. To
retrieve an original deposited metadata document verbatim, use
[`GET /ro-crate/{id}/metadata`](/docs/api/get-ro-crate-metadata).

## Declaring the Capability

The `deposit` block in
[`/capabilities`](/docs/getting-started/capabilities#deposit) is **required
of every implementation** — read-only catalogs included. A client never has
to infer read-only-ness from a missing key; each implementation says where it
stands:

```json
{
  "apiVersion": "0.3.0",
  "deposit": {
    "supported": true,
    "idMinting": "both",
    "fileUpload": ["inline", "presigned"],
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

`supported` is the single flag clients check; when it is `false` the
remaining fields MUST be omitted. The
[Capabilities guide](/docs/getting-started/capabilities#deposit) documents
what each field governs.

Deletion behaviour is declared separately, in the top-level
[`tombstonePolicy`](/docs/getting-started/capabilities#tombstonepolicy):
it governs entity and file URIs as well as RO-Crate ones, so every
implementation declares it, deposit or not.

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
**implementation-defined**. Implementations should pick one of two patterns
and apply it consistently:

- **Public surface**: RO-Crates are listable and retrievable by anyone, with
  the `access` object governing metadata-document retrieval. Recommended
  derivation: grant metadata access only if the caller has metadata access
  to *every* entity materialised from the RO-Crate, since the deposited
  metadata document is the union of their metadata.
- **Depositor-only surface**: RO-Crate reads require the `write` scope — a
  management surface for depositors and curators, not a catalog surface.

The RO-Crate resource carries the `access` object either way, so client code
is identical under both.

## The Endpoints

| Operation | Purpose |
| --- | --- |
| [`POST /deposits`](/docs/api/create-deposit) | Open a deposit for a new RO-Crate |
| [`POST /ro-crate/{id}/deposits`](/docs/api/create-update-deposit) | Open an update deposit for an existing RO-Crate |
| [`GET /deposit/{id}`](/docs/api/get-deposit) | Deposit state, staged files, recorded errors; the polling resource |
| [`PUT /deposit/{id}/metadata`](/docs/api/stage-deposit-metadata) | Stage the metadata document (full replace) |
| [`PUT /deposit/{id}/file/{fileId}`](/docs/api/stage-deposit-file) | Stage a file (inline or presigned) |
| [`DELETE /deposit/{id}/file/{fileId}`](/docs/api/unstage-deposit-file) | Remove a staged file |
| [`POST /deposit/{id}/finalise`](/docs/api/finalise-deposit) | Validate, publish, materialise |
| [`DELETE /deposit/{id}`](/docs/api/abort-deposit) | Abort an open deposit |
| [`GET /ro-crates`](/docs/api/list-ro-crates) | List RO-Crates |
| [`GET /ro-crate/{id}`](/docs/api/get-ro-crate) | Retrieve an RO-Crate |
| [`GET /ro-crate/{id}/metadata`](/docs/api/get-ro-crate-metadata) | The deposited metadata document, verbatim |
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
- **Degrade gracefully**: `roCrateIds` and `roCrateId` are optional; absent
  means the implementation (or that resource) doesn't carry them, not that
  the resource is invalid.
