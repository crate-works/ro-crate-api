---
title: Updating a Storage Object
mdx.format: md
---

# Updating a Storage Object

An update is a new deposit against an existing storage object:

```http
POST /storage-object/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FNT1%2F001/deposits
```

(`404` if the object doesn't exist.) From here the session is [the same as a
create deposit](./depositing) — same staging endpoints, same finalise, same
state machine. What differs is how the staged content resolves against the
version being replaced.

## Crate-as-Manifest Carry-Forward

An update deposit starts logically empty. You stage a **new crate** plus
**only the files that changed**. At finalise, the new crate is the
authoritative file manifest, and each file entity in it resolves in order:

1. **Staged in this deposit** — the new bytes win.
2. **Carried forward from the baseline by `@id`** — the existing bytes are
   kept, without re-upload.
3. **Unresolved** — the reference stays dangling (see below).

Files present in the baseline version but **absent from the new crate drop
out** of the new version. There is no explicit file-delete call against a
storage object — the crate says what the new version holds.

The common cases all fall out of this one rule:

- **Fix a metadata typo**: stage the corrected crate, finalise. No files
  staged; everything carries forward. (There is no separate
  replace-crate-only fast path — this *is* it.)
- **Replace one recording**: stage the corrected crate (if the metadata
  changed) or none at all, stage the new bytes at the file's `@id`,
  finalise. Every other file carries forward.
- **Remove a file**: stage a crate that no longer references it, finalise.
- **Add a file**: stage a crate that references it, stage its bytes,
  finalise.

## Unresolved References

An unresolved reference is **not a protocol error** — the file simply
`404`s when followed. This is the same non-policing integrity stance as the
rest of the specification: a crate may legitimately reference material the
archive does not hold. Strict archives MAY reject unresolved references at
finalise via implementation-defined validation (the uniform `422`
violations shape).

The same resolution rule applies to create deposits — there is no baseline,
so every file entity without staged bytes is simply unresolved.

## Concurrency

- **The baseline is pinned when the deposit is opened.** Carry-forward
  resolves against the version that was current at `POST
  /storage-object/{id}/deposits` time, recorded as the deposit's
  `createdAt`.
- **Finalise replaces the storage object wholesale.** The last finalise
  wins *as a unit*: a published version is always exactly what one
  depositor described — never a mix of two sessions.
- Concurrent open deposits against one storage object are allowed.
  Implementations MAY reject a finalise whose baseline has been superseded
  with `409`; clients should be prepared to re-open a deposit against the
  new current version.

## No Version History

The API surface is current-version-only. The pinned baseline is internal
deposit state, not an exposed history surface: each successful finalise
produces a new current version, and prior versions are not addressable.
Implementations are free to keep full histories internally (for example in
OCFL); exposing them through the API would be a future extension.
