---
slug: deposit-extension
title: "The deposit extension: write access via storage objects"
authors: [johnf]
tags: [paradisec, ldaca]
---

Version 0.3.0 of the RO-Crate API specification registers the `deposit`
extension — the first write pathway in an otherwise read-only API. Writes flow
through deposit sessions against storage objects: an RO-Crate plus all the
files it references, deposited and stored as a unit, from which the
implementation materialises catalog entities by its own rules.

{/* truncate */}

## Why storage objects, not entity writes

A single RO-Crate routinely describes many catalog entities — a collection,
its items, every file they contain, and the people and organisations
connected to them; the entity model is extensible, so the list doesn't end
there. Entity-granular
write endpoints would force depositors to decompose a crate they already hold
into a sequence of per-entity calls, and force the API to referee partial
failures across that sequence.

The unit a depositor actually holds is the crate plus its files, so that is
the unit of deposit. Depositors send storage objects; the implementation
materialises entities from them, and entities remain read-only projections.
How materialisation works is implementation-defined — PARADISEC turns one
item crate into an item entity plus a file entity per media file, while LDaCA
may explode a whole corpus crate into collection, item, file, person, and
organisation entities from a single deposit.

## The deposit session

A deposit is a staging area with an atomic publish at the end:

1. **Create** — `POST /deposits` opens a deposit for a new storage object
   (client-proposed or server-minted ID, per the declared `idMinting` mode);
   `POST /storage-object/{id}/deposits` opens an update deposit against an
   existing one.
2. **Stage** — `PUT /deposit/{id}/crate` stages the RO-Crate (full replace)
   and `PUT /deposit/{id}/file/{fileId}` stages files, either inline or via a
   presigned upload target, in any order.
3. **Finalise** — `POST /deposit/{id}/finalise` validates and publishes
   atomically, synchronously (200) or asynchronously (202 with polling via
   `GET /deposit/{id}`). A failed finalise returns the deposit to `open` with
   the violations recorded — staged content is never lost to a metadata typo.

`DELETE /deposit/{id}` aborts an open deposit.

Updates use crate-as-manifest carry-forward: stage a new crate plus only the
changed files. Unchanged files carry forward from the pinned baseline by
`@id`, files absent from the new crate drop out, and each finalise replaces
the storage object wholesale — the last finalise wins as a unit.

## The read surface and linkage

Deposited storage objects are readable: `GET /storage-objects` lists them,
`GET /storage-object/{id}` returns a lean body, and
`GET /storage-object/{id}/crate` returns the deposited crate verbatim. The
two surfaces are bidirectionally linked — `entityIds` on the storage object
answers "what did my deposit create", while `storageObjectIds` on entities
and `storageObjectId` on files point back at the source of truth.

Deletion is `DELETE /storage-object/{id}`, plus a narrow
`DELETE /entity/{id}` valid only for entities no storage object contributes
to. Deleted URIs follow a single per-implementation tombstone policy —
`410` with a `Tombstone` body, or plain `404`.

## Discovering write support

Presence of the `deposit` key in `/capabilities` is how clients discover
write support — there is no separate write flag:

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

Clients read the declared modes rather than probing, and write operations
require the new coarse OAuth2 `write` scope; finer-grained authorisation is
implementation-defined.

## Where to go next

- The [deposit extension guide](/docs/extensions/deposit) covers the model in
  depth, with walkthroughs of [depositing](/docs/extensions/deposit/depositing),
  [updating](/docs/extensions/deposit/updating), and
  [deletion & lifecycle](/docs/extensions/deposit/lifecycle).
- The [API reference](/docs/api) documents the deposit and storage-object
  endpoints and schemas.
- The [changelog](https://github.com/Language-Research-Technology/ro-crate-api/blob/main/CHANGELOG.md)
  records the full 0.3.0 change set.
