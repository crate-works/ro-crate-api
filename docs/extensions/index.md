---
title: Extensions
mdx.format: md
---

# API Extensions

This guide explains how the RO-Crate API specification is extended: the
extension model, the curated registry, feature detection through
[`/capabilities`](/docs/getting-started/capabilities), and the rules clients
must follow. Each registered extension has its own page describing it in
detail.

## The Extension Model

The core specification defines the endpoints and fields every implementation
must provide. Beyond that core, functionality is added through **extensions**:
optional, well-defined units of behaviour that implementations choose to
provide.

Every extension has:

- **A stable identifier** (e.g. `segments`) used to declare and detect it
- **A schema** defining the response properties it adds
- **Semantics** describing how those properties behave
- **A capability schema** defining the extra details an implementation
  communicates about the extension in `/capabilities`

Extensions are optional. An implementation remains conformant while
implementing only the extensions relevant to its corpus — or none at all.

## The Curated Registry

Extensions are registered in the specification itself. Each registered
extension is defined in the OpenAPI document: its properties appear inline in
core schemas as optional fields, each tagged with an `x-extension: <id>`
annotation so the boundary between core and extension is machine-readable.
Normative detail lives in the schema descriptions and is rendered on the
generated API reference pages; each extension also has a guide page here with
examples and implementation notes.

To propose a new extension (or a new variant within an existing one, such as a
new segment type), open a pull request against
[this specification repository](https://github.com/Language-Research-Technology/ro-crate-api).
Registration keeps identifiers stable and collision-free, and gives all
consumers a single authoritative definition.

### Registered Extensions

| Identifier | Adds | Capability details |
| --- | --- | --- |
| [`segments`](./segments) | `searchExtra.segments` — structured drill-down locations (PDF pages, time-aligned annotations) for full-text search matches inside files | none |
| [`deposit`](./deposit/) | Write access through deposit sessions against storage objects — the deposit and storage-object endpoints, plus `storageObjectIds` on Entity and `storageObjectId` on File | `idMinting`, `fileUpload`, `tombstonePolicy`, optional `depositTtlSeconds` and `maxFileSizeBytes` |

### Legacy Extensions

A handful of properties used by the [oni-ui](https://github.com/Language-Research-Technology/oni-ui)
implementation predate the registry. They are documented on the
[Legacy Extensions](./legacy) page until they are formally registered.

### Experimental Extensions

Unregistered, private experiments MUST use an `x-` prefixed identifier (e.g.
`x-my-experiment`). The `x-` prefix is a sanctioned lane for experimentation
that cannot collide with registered identifiers. Experimental extensions are
outside the scope of this specification; register them before relying on them
across implementations.

## Feature Detection

Implementations declare their implemented extensions in the `extensions`
member of [`/capabilities`](/docs/getting-started/capabilities), a map of
extension identifier to a details object. Detection is a simple key
lookup: an archive supports segments exactly when
`"segments" in capabilities.extensions`.

## Client Rules

Clients that work across implementations MUST follow these rules:

1. **Feature-detect before use**: check `/capabilities` rather than probing
   responses or hard-coding per-archive behaviour
2. **Tolerate absent extensions**: extension properties are optional; degrade
   gracefully when an extension (or an individual property) is not present
3. **Skip unknown variants**: where an extension defines a discriminated union
   (such as segment `type`), skip values you do not recognise rather than
   fail — new variants are added by spec revision and deployed clients must
   keep working
