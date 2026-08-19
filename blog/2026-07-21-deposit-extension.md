---
slug: deposit-extension
title: "Deposits: write access via RO-Crates"
authors: [johnf]
tags: [paradisec, ldaca]
---

Version 0.3.0 of the RO-Crate API specification adds deposit — the first
write pathway in an otherwise read-only API. It sits in the core
specification but is optional to provide, so read-only catalogs stay
conformant. Writes flow through deposit sessions against RO-Crates: a
metadata document plus all the files it references, deposited and stored as
a unit,
from which the implementation materialises catalog entities by its own
rules.

{/* truncate */}

## Why RO-Crates, not entity writes

A single RO-Crate routinely describes many catalog entities — a collection,
its items, every file they contain, and the people and organisations
connected to them; the entity model is extensible, so the list doesn't end
there. Entity-granular
write endpoints would force depositors to decompose an RO-Crate they already hold
into a sequence of per-entity calls, and force the API to referee partial
failures across that sequence.

The unit a depositor actually holds is the metadata document plus its files,
so that is
the unit of deposit. Depositors send RO-Crates; the implementation
materialises entities from them, and entities remain read-only projections.
How materialisation works is implementation-defined — PARADISEC turns one
item RO-Crate into an item entity plus a file entity per media file, while LDaCA
may explode a whole corpus RO-Crate into collection, item, file, person, and
organisation entities from a single deposit.

## The deposit session

A deposit is a staging area with an atomic publish at the end:

1. **Create** — `POST /deposits` opens a deposit for a new RO-Crate
   (client-proposed or server-minted ID, per the declared `idMinting` mode);
   `POST /ro-crate/{id}/deposits` opens an update deposit against an
   existing one.
2. **Stage** — `PUT /deposit/{id}/metadata` stages the RO-Crate (full replace)
   and `PUT /deposit/{id}/file/{fileId}` stages files, either inline or via a
   presigned upload target, in any order.
3. **Finalise** — `POST /deposit/{id}/finalise` validates and publishes
   atomically, synchronously (200) or asynchronously (202 with polling via
   `GET /deposit/{id}`). A failed finalise returns the deposit to `open` with
   the violations recorded — staged content is never lost to a metadata typo.

`DELETE /deposit/{id}` aborts an open deposit.

Updates use metadata-as-manifest carry-forward: stage a new metadata document plus only the
changed files. Unchanged files carry forward from the pinned baseline by
`@id`, files absent from the new metadata document drop out, and each finalise replaces
the RO-Crate wholesale — the last finalise wins as a unit.

## The read surface and linkage

Deposited RO-Crates are readable: `GET /ro-crates` lists them,
`GET /ro-crate/{id}` returns a lean body, and
`GET /ro-crate/{id}/metadata` returns the deposited crate verbatim. The
two surfaces are bidirectionally linked — `entityIds` on the RO-Crate
answers "what did my deposit create", while `roCrateIds` on entities
and `roCrateId` on files point back at the source of truth.

Deletion is `DELETE /ro-crate/{id}`, plus a narrow
`DELETE /entity/{id}` valid only for entities no RO-Crate contributes
to. Deleted URIs follow a single per-implementation tombstone policy —
`410` with a `Tombstone` body, or plain `404`.

## Discovering write support

`/capabilities` gains a required `deposit` block. Every implementation
declares its position — a read-only catalog says so outright rather than
leaving a key out, so clients never have to read absence as a "no":

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

A read-only catalog declares `"deposit": { "supported": false }` and nothing
more.

Clients read the declared modes rather than probing, and write operations
require the new coarse OAuth2 `write` scope; finer-grained authorisation is
implementation-defined.

## Where to go next

- The [Deposits guide](/docs/deposit) covers the model in depth, with
  walkthroughs of [depositing](/docs/deposit/depositing),
  [updating](/docs/deposit/updating), and
  [deletion & lifecycle](/docs/deposit/lifecycle).
- The [API reference](/docs/api) documents the deposit and RO-Crate
  endpoints and schemas.
- The [changelog](https://github.com/Language-Research-Technology/ro-crate-api/blob/main/CHANGELOG.md)
  records the full 0.3.0 change set.
