---
sidebar_position: 5
title: Capabilities
mdx.format: md
---

# Capabilities

Every conformant implementation provides `GET /capabilities`: a single endpoint
that declares what the implementation supports. Clients feature-detect against
it instead of relying on per-archive configuration or probing responses.

## The Response

```json
{
  "apiVersion": "0.3.0",
  "extensions": {
    "segments": {},
    "deposit": {
      "idMinting": "both",
      "fileUpload": ["inline", "presigned"],
      "tombstonePolicy": "410"
    }
  },
  "search": {
    "filters": {
      "inLanguage": { "type": "string", "label": "Language" },
      "mediaType": { "type": "string" },
      "createdAt": { "type": "date", "label": "Date created" }
    },
    "facets": {
      "inLanguage": { "label": "Language" },
      "mediaType": {}
    }
  }
}
```

The full schema is documented in the
[API reference](/docs/api/get-capabilities).

### `apiVersion`

The version of this specification the implementation targets. Use it to reason
about core-level differences between archives as the specification evolves —
the [changelog](https://github.com/Language-Research-Technology/ro-crate-api/blob/main/CHANGELOG.md)
records what changed in each version.

### `extensions`

The registered extensions the implementation provides, as a map of extension
identifier to a details object describing how that extension is provided.
Presence of a key means the extension is implemented; the value is an empty
object when the extension has no extra details to communicate.

Detection is a simple key lookup: an archive supports segments exactly when
`"segments" in capabilities.extensions`. See the
[Extensions guide](/docs/extensions) for the extension model and the rules
clients must follow.

### `search.filters`

The fields that may be used in the search request's `filters` object, as a map
of field name to its declaration. Each filter declares a required `type` —
`string`, `date`, `number`, or `boolean` — and an optional display `label`.

The type tells you which UI element suits the field (a date picker for `date`,
a toggle for `boolean`) and which request syntax it accepts: every filter
accepts an array of exact values, and `date` and `number` filters additionally
accept an inclusive range object:

```json
{
  "filters": {
    "inLanguage": ["English"],
    "createdAt": { "gte": "2020-01-01", "lte": "2021-01-01" }
  }
}
```

A `date` or `number` filter also accepts a non-empty array of range objects,
matched as an OR of the ranges — an entity matches when any of the ranges
matches. This is how a UI lets the user select several disjoint periods, such
as two years in a date facet:

```json
{
  "filters": {
    "createdAt": [
      { "gte": "1965-01-01", "lte": "1965-12-31" },
      { "gte": "1972-01-01", "lte": "1972-12-31" }
    ]
  }
}
```

Requests using a filter field the implementation did not declare — or sending
a range to a `string` or `boolean` filter, or mixing exact values and range
objects in one array — are rejected with a 400 `ValidationError`, so build
filter UI from this map rather than hard-coding field lists. Hide filters
whose `type` you do not recognise; new types are added by spec revision.

### `search.facets`

The facet fields the implementation supports in search, as a map of field name
to its declaration, with an optional display `label`. Each field listed here
appears in the search response's facet counts, and is guaranteed to also be
declared in `search.filters` — so a facet value the user clicks can always be
applied as a filter on the next request.

## Using Capabilities

Fetch `/capabilities` once when your client starts a session with an archive
and cache the result — it describes the deployment, not individual requests.
Degrade gracefully when a capability is absent: hide the feature rather than
failing.
