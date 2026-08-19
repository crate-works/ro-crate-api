---
title: Deletion & Lifecycle
mdx.format: md
---

# Deletion & Lifecycle

RO-Crates are deleted directly; what happens to the entities
materialised from them is the implementation's call. This page covers the
delete operation, tombstones, the entity knock-on, and the two patterns that
replace entity-level writes: curation RO-Crates for enrichment, and
the one constrained entity delete.

## Deleting an RO-Crate

```http
DELETE /ro-crate/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FNT1%2F001
```

Deletion is not a deposit session — there is nothing to stage and the
operation is already atomic. It removes the RO-Crate's metadata document
and files. Implementations needing slow teardown MAY return `202 Accepted`.

The specification mandates **no preconditions**. An implementation MAY
refuse a delete with `409` and a reason body per its own policy — curatorial
holds, cross-references, retention rules — but none is required.

## The Entity Knock-On

What deletion does to the entities materialised from the RO-Crate is
**implementation-defined**, with the same latitude as re-deposit pruning:

- An entity whose `roCrateIds` still lists another contributor
  survives — deleting one contributor just removes it from the list. The
  merged-entity case needs no special protection; it is the same machinery
  as any re-materialisation.
- An entity left with no contributors MAY be pruned — or deliberately
  retained. Retention is legitimate: an archive may keep an Organisation
  entity alive after its last contributing RO-Crate is deleted.

Entity URIs have no independent deletion semantics: entities disappear (or
not) per the implementation's lifecycle rules, and when they do, their URIs
adopt the same tombstone policy as everything else.

### Previewing a Delete

There is no dry-run endpoint. Clients can approximate one:

1. `GET /ro-crate/{id}` and read `entityIds`.
2. For each listed entity, `GET /entity/{id}` and inspect
   `roCrateIds`.
3. Any entity whose `roCrateIds` contains only this RO-Crate is
   a pruning candidate.

Treat the result as an **indication, not a promise** — the entity lifecycle
is implementation-defined, so a candidate may still be retained.

## Tombstones

Each implementation adopts **one** tombstone policy, declared as the
top-level [`tombstonePolicy`](/docs/getting-started/capabilities#tombstonepolicy)
in `/capabilities`, covering deleted RO-Crate URIs (including
`/ro-crate/{id}/metadata`) and the entity knock-on alike — no mixing:

- **`"410"`**: deleted URIs return `410 Gone` with a
  [Tombstone](/docs/api/schemas/tombstone) body:

  ```json
  {
    "id": "https://catalog.paradisec.org.au/repository/NT1/001",
    "resourceType": "RO-Crate",
    "deletedAt": "2026-07-21T04:12:00Z",
    "reason": "Withdrawn at the depositor's request"
  }
  ```

  An entity or file tombstone MAY carry a `roCrateId` naming the
  RO-Crate whose deletion orphaned it.

- **`"404"`**: hard delete — deleted URIs are indistinguishable from URIs
  that never existed.

### ID Reuse

A deleted RO-Crate ID MAY be recreated (`POST /deposits` with the
previously used ID); implementations MAY refuse with `409`. Recreation is a
**new RO-Crate, not a continuation** — no version lineage and no
relationship to the deleted RO-Crate's entities is implied.

The policies pair naturally, though neither pairing is mandated: `410`
tombstones suit archives that reserve IDs forever (trustworthy citations);
`404` hard-delete suits clean reuse, including the delete-and-recreate flow
for changing an RO-Crate's ID.

## Open Deposits When the RO-Crate Is Deleted

**Delete wins.** An open deposit targeting a deleted RO-Crate can no
longer finalise: the finalise fails, the failure is recorded in the
deposit's `errors`, and the deposit returns to `open` — the same shape as a
validation failure. The client's move is to abort. Implementations MAY
auto-abort such deposits; cautious archives can instead refuse the delete
via the implementation-defined `409` above.

## Enriching Entities: Curation RO-Crates

There is no entity UPDATE. To enrich an entity beyond what its original
deposit said — without touching that deposit — deposit a small **curation
RO-Crate** whose metadata document mentions the entity's `@id` and carries
the extra metadata. The ordinary
[many-to-one merge](./#materialisation) combines the contributions.

For example, to flesh out an Organisation entity that a collection deposit
mentioned only by name, deposit a one-entity RO-Crate:

```json
{
  "@id": "https://ror.org/03zparb35",
  "@type": "Organization",
  "name": "Australian National University",
  "url": "https://www.anu.edu.au/"
}
```

The merged entity now lists both RO-Crates in `roCrateIds` —
the enrichment survives re-deposit of the original RO-Crate, carries its own
provenance, and flows through the normal session pathway with nothing new
to learn.

## Deleting a Contributor-less Entity

The one entity-level write:
[`DELETE /entity/{id}`](/docs/api/delete-entity) is valid **only** when the
entity's `roCrateIds` is empty, and returns `409` otherwise. It
exists because no session-based operation can reach an entity that no
RO-Crate contributes to — such as a retained Organisation whose last
contributor was deleted. Deposited data can never be touched this way:
while any RO-Crate contributes, the delete is refused and changes go
through [deposit sessions](./updating) instead.
