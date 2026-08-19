# Changelog

All notable changes to the RO-Crate API specification are documented in this
file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the specification adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-07-21

### Added

- Deposit: write access through deposit sessions
  against RO-Crates — a metadata document plus all the files it references,
  deposited and stored as a unit, from which the implementation materialises
  catalog entities by its own rules. The RO-Crate is the sole write
  pathway; entities remain read-only projections.
- Deposit session endpoints: `POST /deposits` creates a deposit for a new
  RO-Crate (client-proposed or server-minted ID per the `idMinting`
  capability) and `POST /ro-crate/{id}/deposits` opens an update
  deposit; staging via `PUT /deposit/{id}/metadata` (full replace) and
  `PUT /deposit/{id}/file/{fileId}` (inline bytes or presigned upload,
  discriminated by Content-Type), in any order; `POST /deposit/{id}/finalise`
  validates and publishes atomically, synchronously (200) or asynchronously
  (202 with polling via `GET /deposit/{id}`); `DELETE /deposit/{id}` aborts.
  A failed finalise returns the deposit to `open` with the violations
  recorded — staged content is never lost to a metadata typo.
- Update deposits use metadata-as-manifest carry-forward: stage a new metadata document plus
  only the changed files; unchanged files carry forward from the baseline by
  `@id`, and files absent from the new metadata document drop out. The baseline is pinned
  at deposit creation and each finalise replaces the RO-Crate
  wholesale, so the last finalise wins as a unit.
- The RO-Crate read surface: `GET /ro-crates` (minimal list),
  `GET /ro-crate/{id}` (lean body — `id`, `entityIds`, timestamps,
  `access`) and `GET /ro-crate/{id}/metadata` (the deposited crate,
  verbatim, with a HEAD twin). Bidirectional linkage between the catalog and
  RO-Crates: a plural `roCrateIds` on entities, a singular
  `roCrateId` on files, and `entityIds` on the RO-Crate.
- Deletion: `DELETE /ro-crate/{id}` with no spec-mandated
  preconditions (implementations MAY refuse with a 409 and a reason), and a
  constrained `DELETE /entity/{id}` valid only for entities no RO-Crate
  contributes to. Deleted URIs follow a single per-implementation
  tombstone policy — `410` with a new `Tombstone` schema, or plain `404` —
  declared in `/capabilities` and covering entities alike.
- A required top-level `deposit` block in `/capabilities`. Its `supported`
  flag states whether the implementation provides the deposit surface;
  where it does, `idMinting`, `fileUpload` and `tombstonePolicy` accompany
  it, plus optional `depositTtlSeconds` and `maxFileSizeBytes`. Deposit is
  optional core rather than a registered extension: it is specified in the
  core document, is not keyed in `capabilities.extensions`, and read-only
  implementations remain conformant by declaring
  `"deposit": { "supported": false }`. Write operations require the new
  coarse OAuth2 `write` scope.

### Changed

- `GET /entity/{id}/rocrate` is now specified as implementation-defined in
  provenance: the document may be stored or derived from the entity's
  contributing RO-Crates, but must always be a valid RO-Crate whose
  root data entity describes the entity. An original deposited metadata document is
  retrieved verbatim from `GET /ro-crate/{id}/metadata`.

## [0.2.0] - 2026-07-20

### Added

- A `date` or `number` search filter value may now also be a non-empty array
  of `gte`/`lte` range objects, matched as an OR of the ranges — an entity
  matches when any of the ranges matches. This lets a client express several
  disjoint periods on one filter, such as a date facet where the user selects
  multiple years. Arrays mixing exact terms and range objects are rejected
  with a 400 `ValidationError`, as is any range syntax on a `string` or
  `boolean` filter.

## [0.1.0] - 2026-07-13

### Added

- An extension framework: extensions are first-class citizens of the
  specification, registered in the OpenAPI document with a stable identifier,
  a schema, and semantics. Extension properties are tagged with an
  `x-extension: <id>` annotation. Unregistered experiments use an `x-`
  prefixed identifier.
- A required `GET /capabilities` endpoint through which every conformant
  implementation declares the spec version it targets (`apiVersion`), the
  extensions it implements (`extensions`), and its search capabilities
  (`search`). **Conformance-affecting**: existing implementations must add
  this endpoint.
- The `search` capability object declares the implementation's search
  `filters` (each with a required `type` — `string`, `date`, `number`, or
  `boolean` — and an optional `label`) and its search `facets`. Every facet
  field must also be declared as a filter, so a facet value the user clicks
  can always be applied as a filter.
- Search request `filters` values may now be an inclusive `gte`/`lte` range
  object as well as an array of exact values. Ranges are valid only for
  filters declared with type `date` or `number`. Requests using an undeclared
  filter field, or a range on a `string`/`boolean` filter, are rejected with
  a 400 `ValidationError`.
- The `segments` extension — the first registered extension. Adds an optional
  `searchExtra.segments` array to search hits: structured drill-down locations
  (PDF pages via `PageSegment`, time-aligned annotations such as ELAN via
  `TimeAlignedAnnotationSegment`) for full-text matches inside files,
  expressed as a discriminated union on `type`.
- This changelog.

### Changed

- The Extensions guide moved to a dedicated top-level docs section with a
  page per extension: the extension model and registry on the index page, a
  detailed `segments` page (including non-normative Elasticsearch
  implementation notes), and the legacy Oni-UI properties on their own page
  pending registration.

## [0.0.1] - 2024-12-20

### Added

- Initial specification: entities, files, and search endpoints with
  authentication, rate limiting, and error handling conventions.
