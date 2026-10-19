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
  "apiVersion": "0.4.0",
  "deposit": {
    "supported": true,
    "idMinting": "both",
    "fileUpload": ["inline", "presigned"]
  },
  "tombstonePolicy": "410",
  "extensions": {
    "segments": {}
  },
  "sorts": {
    "id": { "label": "Identifier" },
    "name": { "label": "Name" },
    "originatedOn": { "label": "Origination date" }
  },
  "search": {
    "searchTypes": {
      "basic": {},
      "advanced": {
        "fields": { "name": { "label": "Title" }, "description": {} }
      }
    },
    "geo": { "supported": true },
    "maxResultWindow": 10000,
    "filters": {
      "inLanguage": { "type": "string", "label": "Language" },
      "mediaType": { "type": "string" },
      "createdAt": { "type": "date", "label": "Date created" }
    },
    "facets": {
      "inLanguage": { "label": "Language" },
      "mediaType": {},
      "createdAt": { "label": "Date created" }
    }
  }
}
```

The full schema is documented in the
[API reference](/docs/api/get-capabilities).

### `apiVersion`

The version of this specification the implementation targets. Use it to reason
about core-level differences between archives as the specification evolves —
the [changelog](https://github.com/crate-works/ro-crate-api/blob/main/CHANGELOG.md)
records what changed in each version.

### `deposit`

Whether the implementation provides the optional
[deposit surface](/docs/deposit), and on what terms. This member is
**required**: every implementation declares its position explicitly, so a
read-only catalog is never mistaken for one whose capability document happens
to be incomplete.

`supported` is the flag clients check. A read-only catalog declares:

```json
{ "deposit": { "supported": false } }
```

- **`supported`** (required): whether the deposit and RO-Crate endpoints are
  provided. When it is `false`, the remaining fields MUST be omitted.
- **`idMinting`** (required when supported): who mints RO-Crate IDs —
  `client` (the depositor proposes), `server` (the implementation mints), or
  `both` (the client may propose, the server fills gaps).
- **`fileUpload`** (required when supported): the file staging modes
  supported, drawn from `inline` (bytes in the staging request) and
  `presigned` (metadata in the staging request, bytes uploaded directly to a
  returned target). Future modes may be added; ignore values you do not
  recognise.
- **`depositTtlSeconds`** (optional): the expiry horizon for abandoned
  deposits. Absent means expiry is implementation-defined — don't rely on a
  particular window.
- **`maxFileSizeBytes`** (optional): the largest file a deposit may stage.
  Absent means no declared limit.

The [Deposits guide](/docs/deposit) covers how these play out in practice.

### `tombstonePolicy`

What deleted resource URIs return — `"410"` (a `410 Gone` carrying a
[Tombstone](/docs/api/schemas/tombstone) body) or `"404"` (indistinguishable
from a URI that never existed). One policy covers RO-Crate, entity and file
URIs alike; implementations do not mix them.

This member is **required** of every implementation, deposit surface or not:
entities and files are mandatory core, so any catalog can have a URI that
used to resolve, and a client following a stale link needs to know which
answer to expect. See
[Deletion & Lifecycle](/docs/deposit/lifecycle#tombstones).

### `extensions`

The registered extensions the implementation provides, as a map of extension
identifier to a details object describing how that extension is provided.
Presence of a key means the extension is implemented; the value is an empty
object when the extension has no extra details to communicate.

Detection is a simple key lookup: an archive supports segments exactly when
`"segments" in capabilities.extensions`. See the
[Extensions guide](/docs/extensions) for the extension model and the rules
clients must follow.

### `sorts`

The sorts the implementation supports on `POST /search` and `GET /entities`,
as a map of sort name to its declaration, with an optional display `label`.
Build one sort menu from this map for search results and Entity lists. Every
sort supports both `asc` and `desc`.

`id` is always declared: it is the default for `GET /entities`. `relevance` is
never declared: it is always available on `POST /search` (and is its default),
and never on `GET /entities`. A request using any other sort that is not
declared here is rejected with a 400 `ValidationError`.

Every sort gives the same order on every page of the same request, so paging
never repeats or skips an Entity. The implementation chooses how to break ties.

### `search.searchTypes`

The search types the implementation supports, as a map. Presence of a key
means the search type is supported.

- **`basic`** is always present, as an empty object. A basic query is free
  text: every word must be in the Entity, in any order, ignoring case, and
  text in double quotes is a phrase. It has no operators.
- **`advanced`** is optional. When present, its `fields` map lists the fields
  an advanced query may name, each with an optional display `label`. Build an
  advanced search form from it. Every advanced field matches as text; use
  `search.filters` for exact values, dates and ranges.

A search request with an undeclared `searchType` is rejected with a 400
`ValidationError`, so hide the advanced search toggle when `advanced` is
absent. An advanced query that names an undeclared field is rejected with a
422. The query grammar is in the
[API reference](/docs/api/search-entities).

Ignore search types you do not recognise; new ones are added by spec revision.

### `search.geo`

Whether the implementation supports geo search: `boundingBox` and
`geohashPrecision` in the request, and `geohashGrid` in the response. This
member is **required**. An implementation without geo declares:

```json
{ "geo": { "supported": false } }
```

Hide the map when `supported` is `false`: a request with `boundingBox` or
`geohashPrecision` is rejected with a 400 `ValidationError`.

When geo is supported, `geohashPrecision` needs a `boundingBox`. A box with
`bottomLeft.lng` greater than `topRight.lng` crosses the antimeridian, which
happens when a user pans the map across the Pacific. Such boxes are valid.

### `search.maxResultWindow`

The largest `offset + limit` that `POST /search` serves. A request past it is
rejected with a 400 `ValidationError`. Absent means no limit.

`total` is always exact, so it can be larger than the window. Stop page links
at the window rather than at `total`.

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
    "createdAt": { "gte": "2020-01-01", "lte": "2020-12-31" }
  }
}
```

A `date` value — a range bound or an exact value — is either a calendar date
(`YYYY-MM-DD`) or a full RFC 3339 date-time. Anything else, including a partial
date such as `2020` or `2020-12`, is rejected with a 400 `ValidationError`.
A calendar date covers the whole of its day in UTC; a date-time is used exactly
as given, and one carrying no timezone offset is read as UTC:

| Value | Resolves to |
| --- | --- |
| `2020-01-01` as `gte` | `2020-01-01T00:00:00Z` |
| `2020-12-31` as `lte` | every instant before `2021-01-01T00:00:00Z` — at millisecond precision, `2020-12-31T23:59:59.999Z` |
| `2020-12-31` as an exact value | any instant within that UTC day |
| `2020-12-31T10:00:00Z` anywhere | that instant alone |

So the range above is the whole of 2020: a year picker sends the year's first
and last dates, not the next year's first. Prefer ranges to exact values on
`date` filters — an exact date value is a whole-day window in disguise, which
is rarely what "exact" suggests.

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
a range to a `string` or `boolean` filter, mixing exact values and range
objects in one array, or giving a bound whose JSON type does not match the
filter's declared type — are rejected with a 400 `ValidationError`, so build
filter UI from this map rather than hard-coding field lists. Hide filters
whose `type` you do not recognise; new types are added by spec revision.

### `search.facets`

The facet fields the implementation supports in search, as a map of field name
to its declaration, with an optional display `label`. Each field listed here is
guaranteed to also be declared in `search.filters`, with type `string`,
`boolean` or `date`, so a facet value the user clicks can always be applied as
a filter on the next request. A `number` filter cannot be a facet.

An implementation that supports no facets declares `{}`, and its search
responses may omit `facets`. Otherwise every search response includes
`facets` with every declared key; a facet with no matches has an empty array.

Buckets for a `string` or `boolean` facet are ordered by count, highest first,
with ties ordered by name. A `date` facet has one bucket per UTC year that has
matches, named with the year as four digits (zero-padded, so `"0850"`) and
ordered oldest first:

```json
{
  "facets": {
    "createdAt": [
      { "name": "1965", "count": 9 },
      { "name": "1972", "count": 6 }
    ]
  }
}
```

To filter on a date bucket, send its year as a range, not as an exact value.
The exact value `"1965"` is a partial date, which is rejected:

```json
{
  "filters": {
    "createdAt": { "gte": "1965-01-01", "lte": "1965-12-31" }
  }
}
```

Several selected years become an array of ranges, as in
[`search.filters`](#searchfilters).

## Using Capabilities

`/capabilities` never requires authentication, and its response is the same
for every caller. Fetch it once when your client first connects to an archive,
before the user signs in, and cache the result — it describes the deployment,
not individual requests.
Degrade gracefully when a capability is absent: hide the feature rather than
failing.
