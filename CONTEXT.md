# RO-Crate API

The specification for a standardised HTTP API over RO-Crate research data
collections, published as a Docusaurus site with a reference generated from
`openapi.yaml`.

## Language

**RO-Crate**:
A dataset packaged with machine-readable metadata — the whole package, not
the JSON file that describes it. Written with the hyphen and both capitals in
prose.
_Avoid_: rocrate, crate, storage object

**RO-Crate Package**:
The concrete form an RO-Crate takes, Attached (a directory carrying a payload
of files) or Detached. RO-Crate 1.2's term; 1.1 does not separate the two.
_Avoid_: crate directory, bundle

**RO-Crate Metadata Document**:
The JSON-LD document describing an RO-Crate. It is part of an RO-Crate, never
the whole of one, so an endpoint returning it is named for the metadata and
not for the crate.
_Avoid_: the crate, ro-crate-metadata.json — that is a filename, not a concept

**Entity**:
A Collection, Object, or MediaObject in the catalog. Entities are read-only
projections that an implementation materialises from RO-Crates by its own
rules; they have no write endpoints.
_Avoid_: record, item, resource

**Deposit**:
A session through which an RO-Crate is written. The sole write pathway, and
optional core — a read-only catalog is conformant without it.
_Avoid_: upload, submission, ingest

## Spelling by context

One concept, three renderings, chosen by where the word sits:

| Context | Form | Example |
| --- | --- | --- |
| Prose and summaries | `RO-Crate` | "the entity's RO-Crate metadata document" |
| Path segments | `ro-crate` | `/ro-crate/{id}/metadata` |
| Identifiers and schema names | `RoCrate` | `RoCrateIdParameter`, `getRoCrateMetadata` |

Never `rocrate`, never bare `crate`. Both forms were in the 0.1.0 surface
and have since been removed from it.
