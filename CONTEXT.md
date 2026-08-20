# RO-Crate API

The specification for a standardised HTTP API over RO-Crate research data
collections, published as a Docusaurus site with a reference generated from
`openapi.yaml`.

Spelling and `operationId` conventions live in `docs/agents/naming.md`.

## Language

**RO-Crate**:
A dataset packaged with machine-readable metadata. The whole package, not the
JSON file that describes it.
_Avoid_: rocrate, crate, storage object

**RO-Crate Package**:
The concrete form an RO-Crate takes — Attached, a directory carrying a payload
of files, or Detached.
_Avoid_: crate directory, bundle

**RO-Crate Metadata Document**:
The JSON-LD document describing an RO-Crate. Part of an RO-Crate, never the
whole of one.
_Avoid_: rocrate, ro-crate-metadata.json

**Entity**:
A Collection, Object, or MediaObject in the catalog, materialised from
RO-Crates by rules the implementation chooses.
_Avoid_: record, item, resource

**Deposit**:
A session through which an RO-Crate is written. The sole write pathway.
_Avoid_: upload, submission, ingest
