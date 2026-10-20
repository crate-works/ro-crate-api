# Changelog

All notable changes to the RO-Crate API specification are in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
The specification uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

A **Breaking** change stops an existing server or client from working until it
changes. Each breaking change says what to do.

## [Unreleased]

### Added

- `search.maxResultWindow` in `/capabilities`. It is optional. It gives the
  largest `offset + limit` that `POST /search` serves, and the server rejects
  a request past it with a 400 `ValidationError`. Absent means no limit.
- An optional `position` on each `ValidationError` violation. For an error in
  a search query, it is the 0-based offset, in Unicode code points, where the
  error was found.

### Changed

- **Breaking.** `/capabilities` has a required `search.searchTypes` map.
  `basic` is always present, and `advanced` is optional. `advanced.fields`
  lists the fields that an advanced query can name. The server rejects an
  undeclared `searchType` with a 400 `ValidationError`, and an advanced query
  that names an undeclared field with a 422.
  _What to do:_ servers declare `searchTypes`, and reject the rest. Clients
  hide advanced search when `advanced` is absent, and build the advanced
  search form from `advanced.fields`.
- **Breaking.** `/capabilities` has a required `search.geo` block with a
  `supported` flag. When it is `false`, the server rejects `boundingBox` and
  `geohashPrecision` with a 400 `ValidationError`.
  _What to do:_ servers declare `geo`. Clients hide the map when
  `supported` is `false`.
- **Breaking.** `/capabilities` has a required top-level `sorts` map. It
  lists the sorts for `POST /search` and `GET /entities`, and it always
  includes `id`. On both endpoints, `sort` is now any string, not a fixed
  list. The server rejects an undeclared sort with a 400 `ValidationError`.
  `relevance` is always valid on `/search`, and never on `/entities`.
  _What to do:_ servers declare the sorts they support, and reject the rest.
  Clients build sort menus from `sorts`.
- **Breaking.** Every sort must give the same order on every page of the same
  request. The server chooses how to break ties.
  _What to do:_ servers that sort on a field with duplicate values add a
  tie-breaker, such as `id`.
- **Breaking.** `sort: relevance` is now the server's best-match order. It
  need not follow `searchExtra.score`. Scores can be compared only within one
  response.
  _What to do:_ clients do not compare scores between responses.
- **Breaking.** The search `total` must be exact.
  _What to do:_ servers that report an estimate or a lower bound count
  exactly. On OpenSearch, set `track_total_hits: true`.
- **Breaking.** The specification now defines the advanced query grammar. It
  has fielded terms, quoted phrases, `AND`, `OR`, unary `NOT`, brackets and
  field groups such as `name: (a OR b)`. Operators are uppercase only.
  Wildcards and other engine syntax are literal text. The server rejects a
  query that does not parse with a 422 `ValidationError`. Before, the grammar
  was defined only by example, and servers could add their own syntax.
  _What to do:_ servers parse the grammar and reject queries that don't
  parse, then translate the parsed query for their backend, so engine syntax
  such as `*` in a value stays literal. Clients send only the grammar.
- **Breaking.** The specification now defines word matching for both search
  types. Every word must match, in any order, ignoring case. In basic search
  this applies to the whole query. In advanced search it applies to each
  value, and `AND`, `OR` and `NOT` combine the results. Quoted text is a
  phrase. Basic search has no operators. Servers may also
  stem words, fold diacritics and ignore stop words.
  _What to do:_ servers that match any word, or that read operators in basic
  search, change to these rules.
- **Breaking.** An empty query (`""` or only whitespace) matches every
  Entity that the caller can see, for both search types.
  _What to do:_ clients that send `"*"` to match everything send `""`.
  Servers return every visible Entity for an empty query.
- **Breaking.** When the server declares facets, every search response
  includes `facets`, with every declared key. A facet with no matches has an
  empty array. A server with no facets declares `{}` and may omit `facets`.
  Only `string`, `boolean` and `date` filters can be facets.
  _What to do:_ servers return every declared facet.
- **Breaking.** A `date` facet has one bucket per UTC year with matches. The
  bucket `name` is the four-digit year, such as `"2001"`, and the buckets are
  in year order, oldest first. Buckets for `string` and `boolean` facets are
  in count order, highest first, with ties in name order. To filter on a year,
  the client sends a range, such as `{ "gte": "2001-01-01", "lte":
  "2001-12-31" }`.
  _What to do:_ servers that return other bucket keys, such as epoch
  milliseconds, return years. Clients send year ranges, not the year as an
  exact value.
- **Breaking.** `geohashPrecision` needs a `boundingBox`. The server rejects
  `geohashPrecision` without one with a 400 `ValidationError`.
  _What to do:_ clients send both together.
- **Breaking.** A `boundingBox` with `bottomLeft.lng` greater than
  `topRight.lng` now crosses the antimeridian. Servers that support geo must
  accept it. A box with `bottomLeft.lat` greater than `topRight.lat` is a 400
  `ValidationError`.
  _What to do:_ servers that treat such a box as empty or invalid search the
  two longitude ranges on each side of 180°.

### Fixed

- The Error Handling guide's 422 example used the code
  `INVALID_QUERY_SYNTAX`, which `ValidationError` does not allow. It now uses
  `VALIDATION_ERROR`, with the error in `details.violations`.
- The Capabilities guide said that every declared facet appears in the search
  response, but the response schema made `facets` optional. The search
  response description now states when `facets` must be present.
- The Error Handling guide's 401, 403 and 429 examples now match the schemas
  in `openapi.yaml`. They used the codes `AUTHENTICATION_REQUIRED` and
  `INSUFFICIENT_PERMISSIONS` instead of `UNAUTHORIZED` and `FORBIDDEN`, and
  `details` fields that the schemas do not define. The 400 example no longer
  names `conformsTo`, which is not a parameter.
- The Overview and Use Cases guides used `conformsTo` for an Entity's type.
  The Use Cases guide now filters `GET /entities` with `entityType`. The
  Overview guide's example Entity now has the fields in the `Entity` schema,
  not `conformsTo`, `recordType`, `createdAt` and `updatedAt`.
- `GET /capabilities` is now public in `openapi.yaml`. It has no security
  requirement and no `401` or `403` responses. Before, it inherited the global
  security requirement, and the Authentication guide listed it under the
  `read` scope. The specification now says that the endpoint must not require
  authentication, and that its response must be the same for every caller.
- The read endpoints now allow anonymous access in `openapi.yaml`. Before,
  the global security requirement had no anonymous option, so every read
  formally needed authentication, although the guides showed public access.
  An archive can serve every read publicly and implement no security scheme.
- The RO-Crate `access` flags now have one meaning each. `content` governs
  only the deposited RO-Crate Metadata Document. Before, the schema said that
  it also governed file retrieval, which conflicted with each File's own
  `access`. The Deposits guide recommended that `metadata` access control
  the document, which conflicted with the schema. It now recommends
  `content`. The Authorization guide has a new section for RO-Crate access.

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
