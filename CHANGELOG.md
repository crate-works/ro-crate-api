# Changelog

All notable changes to the RO-Crate API specification are in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
The specification uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

A **Breaking** change stops an existing server or client from working until it
changes. Each breaking change says what to do.

## [Unreleased]

### Fixed

- `GET /capabilities` is now public in `openapi.yaml`. It has no security
  requirement and no `401` or `403` responses. Before, it inherited the global
  security requirement, and the Authentication guide listed it under the
  `read` scope. The specification now says that the endpoint must not require
  authentication, and that its response must be the same for every caller.
- The read endpoints now allow anonymous access in `openapi.yaml`. Before,
  the global security requirement had no anonymous option, so every read
  formally needed authentication, although the guides showed public access.
  An archive can serve every read publicly and implement no security scheme.

## [0.4.0] - 2026-10-12

### Changed

- **Breaking.** The route for an Entity's RO-Crate Metadata Document is now
  `/entity/{id}/metadata`. It was `/entity/{id}/rocrate`. This applies to
  `GET` and `HEAD`. The new name matches `/ro-crate/{id}/metadata`. The old
  route is removed. It has no alias, and servers return 404 for it.
  The operation IDs change from `getEntityCrate` and `headEntityCrate` to
  `getEntityMetadata` and `headEntityMetadata`. This renames the methods in
  generated client libraries.
  _What to do:_ servers serve the new route. Clients call the new route.
- **Breaking.** The specification now defines the values that a `date` search
  filter accepts. A value is a date (`YYYY-MM-DD`) or a full RFC 3339
  date-time. The server rejects other values, such as `2020` or `2020-12`,
  with a 400 `ValidationError`.
  A date means the full day in UTC. For example, `"lte": "2020-12-31"`
  includes all of 31 December 2020. A date-time without a time zone is UTC.
  These rules apply to ranges and to exact values.
  Before, the specification said only "ISO 8601 strings". A server could read
  `"lte": "2020-12-31"` as the start of that day, and drop the results from
  later in the day.
  _What to do:_ servers that already treat a date as the full UTC day need no
  change. Other servers must change.
- A range bound must have the same JSON type as its filter: a string for a
  `date` filter, a number for a `number` filter. The server rejects a
  mismatch with a 400 `ValidationError`.
- The PARADISEC server URL is now
  <https://admin-catalog.paradisec.org.au/api/v1/oni>. It was on
  `catalog.paradisec.org.au`. The Overview guide uses the new URL.
- The specification moved to the CrateWorks organisation. The site is now at
  <https://ro-crate-api.crate-works.org>, and the repository is at
  <https://github.com/crate-works/ro-crate-api>. The links in `openapi.yaml`
  use the new addresses. The old address,
  `language-research-technology.github.io/ro-crate-api`, does not work and
  does not redirect.

### Fixed

- Three examples used `2021-01-01` as the end of 2020, so they also included
  1 January 2021. They now use `2020-12-31`. The examples are in the
  `FilterRange` schema, the search `filters` example, and the Capabilities
  guide.

## [0.3.0] - 2026-08-19

### Added

- Deposit: a way to write to the catalog. A client writes a complete
  RO-Crate: the RO-Crate Metadata Document and all the files it references.
  The server stores the RO-Crate as one unit, and makes Entities from it by
  its own rules. Deposit is the only way to write. Entities stay read-only.
- Deposit endpoints:
  - `POST /deposits` starts a Deposit for a new RO-Crate. The client gives
    the ID, or the server makes one. The `idMinting` capability says which.
  - `POST /ro-crate/{id}/deposits` starts a Deposit that updates an existing
    RO-Crate.
  - `PUT /deposit/{id}/metadata` adds the RO-Crate Metadata Document. Each
    call replaces the last one.
  - `PUT /deposit/{id}/file/{fileId}` adds a file. The client sends the
    bytes, or gets a presigned upload URL. The `Content-Type` header selects
    which.
  - The client can add the metadata and the files in any order.
  - `POST /deposit/{id}/finalise` validates the Deposit and publishes it as
    one unit. The server replies 200 when it is done, or 202 when it
    continues in the background. After a 202, the client polls
    `GET /deposit/{id}`.
  - If finalise fails, the Deposit stays open and records the errors. The
    server keeps the metadata and files, so the client can fix the problem
    and try again.
  - `DELETE /deposit/{id}` cancels a Deposit.
- Updates. To update an RO-Crate, the client sends a new RO-Crate Metadata
  Document and only the files that changed. The server keeps the other files
  that the new document lists, matched by `@id`. It removes files that the
  new document does not list. The server fixes the starting version when the
  Deposit starts. Each finalise replaces the whole RO-Crate, so the last
  finalise wins.
- Read endpoints for RO-Crates:
  - `GET /ro-crates` lists RO-Crates.
  - `GET /ro-crate/{id}` returns a short summary: `id`, `entityIds`,
    timestamps and `access`.
  - `GET /ro-crate/{id}/metadata` and its `HEAD` twin return the RO-Crate
    Metadata Document exactly as deposited.
- Links between the catalog and RO-Crates: `roCrateIds` on Entities,
  `roCrateId` on files, and `entityIds` on RO-Crates.
- Deletes:
  - `DELETE /ro-crate/{id}` deletes an RO-Crate. The specification sets no
    conditions, but a server can refuse with a 409 and a reason.
  - `DELETE /entity/{id}` deletes an Entity. It works only for an Entity that
    no RO-Crate contributes to.
  - A deleted RO-Crate, Entity or file URI returns `410` with a `Tombstone`
    body, or `404`. Each server uses one of these for all deleted URIs.
- **Breaking.** A required `tombstonePolicy` field in `/capabilities`. It says
  if deleted URIs return `410` or `404`. Servers without Deposit need it too,
  because any catalog can have URIs that no longer resolve.
  _What to do:_ servers add `tombstonePolicy` to `/capabilities`.
- **Breaking.** A required `deposit` field in `/capabilities`. Its `supported`
  field says if the server supports Deposit. When it does, `idMinting` and
  `fileUpload` are required, and `depositTtlSeconds` and `maxFileSizeBytes`
  are optional.
  _What to do:_ servers add `deposit` to `/capabilities`. A read-only server
  sends `"deposit": { "supported": false }`.
- A `write` OAuth2 scope. All write operations need it.

### Changed

- `GET /entity/{id}/rocrate` can return a stored document or a generated one.
  The server decides. The document must be a valid RO-Crate whose root data
  entity describes the Entity. To get the document exactly as deposited, use
  `GET /ro-crate/{id}/metadata`.

## [0.2.0] - 2026-07-20

### Added

- A `date` or `number` search filter can take an array of `gte`/`lte` ranges.
  An Entity matches if it matches any of the ranges. For example, a client
  can select several years on one date facet. The server rejects an array
  that mixes exact values and ranges, and a range on a `string` or `boolean`
  filter, with a 400 `ValidationError`.

## [0.1.0] - 2026-07-13

### Added

- Extensions. A registered extension has a stable ID, a schema and defined
  behaviour in the OpenAPI document. Its properties have an
  `x-extension: <id>` annotation. An unregistered, experimental extension
  uses an ID that starts with `x-`.
- **Breaking.** A required `GET /capabilities` endpoint. It says which version
  of the specification the server implements (`apiVersion`), which extensions
  it supports (`extensions`), and what search it supports (`search`).
  _What to do:_ servers add this endpoint.
- The `search` capability lists the server's search `filters` and `facets`.
  Each filter has a `type` (`string`, `date`, `number` or `boolean`) and an
  optional `label`. Each facet must also be a filter, so a client can always
  apply a facet value as a filter.
- A search filter can take a `gte`/`lte` range, as well as an array of exact
  values. Only `date` and `number` filters accept ranges. The server rejects
  a filter that it did not declare, and a range on a `string` or `boolean`
  filter, with a 400 `ValidationError`.
- The `segments` extension, the first registered extension. It adds an
  optional `searchExtra.segments` array to search results. Each segment
  points to a full-text match in a file: a PDF page (`PageSegment`) or a
  time-aligned annotation such as ELAN (`TimeAlignedAnnotationSegment`). The
  `type` field identifies the kind of segment.
- This changelog.

### Changed

- Extensions have their own section in the docs, with one page for each
  extension. The index page explains extensions and the registry. The
  `segments` page has optional notes about Elasticsearch. The legacy Oni-UI
  properties have their own page until they are registered.

## [0.0.1] - 2024-12-20

### Added

- The first version of the specification: Entities, files and search, with
  authentication, rate limiting and errors.
