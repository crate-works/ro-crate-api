---
slug: extensions-and-capabilities
title: "Introducing API Extensions, /capabilities, and the segments extension"
authors: [johnf]
tags: [paradisec, ldaca]
---

Version 0.1.0 of the RO-Crate API specification makes extensions first-class
citizens: a curated extension registry, a new required `GET /capabilities`
endpoint for feature detection, and the first registered extension —
`segments` — which lets search hits point inside files.

{/* truncate */}

## Why

Archives implementing this API extend it with implementation-specific response
properties, but until now the specification had no rigorous way to define,
discover, or evolve those extensions. Portal implementers had no way to know
what a given archive supports — which extensions, or even which search facets —
without per-archive configuration or probing.

## The extension registry

Extensions are now specified in the OpenAPI document itself. Every extension
has a stable identifier, a schema, and semantics. Extension properties appear
inline in core schemas as optional fields, each tagged with an
`x-extension: <id>` annotation, so the boundary between core and extension is
machine-readable. Anyone can propose a new extension via a pull request to the
specification repository; unregistered private experiments use an `x-`
prefixed identifier.

Implementations choose which extensions to implement — you remain conformant
while implementing only the ones relevant to your corpus.

## GET /capabilities

Every conformant implementation now declares what it supports through a single
endpoint:

```json
{
  "apiVersion": "0.1.0",
  "extensions": { "segments": {} },
  "search": {
    "filters": {
      "inLanguage": { "type": "string", "label": "Language" },
      "createdAt": { "type": "date", "label": "Date created" }
    },
    "facets": { "inLanguage": { "label": "Language" }, "mediaType": {} }
  }
}
```

Detection is a key lookup: an archive supports an extension exactly when its
identifier appears in `extensions`. The `search` object lets portals build
their search UI dynamically instead of hard-coding per-archive field lists:
`filters` declares the fields a search request may filter on, each with a
type (`string`, `date`, `number`, or `boolean`) that tells the portal which
UI element to render — and `date` and `number` filters accept inclusive
`gte`/`lte` range objects in search requests. `facets` declares the fields
returned with facet counts, and every facet field is guaranteed to also be
filterable, so clicking a facet value always works as a filter.

This is the one conformance-affecting change in 0.1.0: existing
implementations need to add the endpoint.

## The segments extension

`segments` is the first registered extension. It adds an optional
`searchExtra.segments` array to search hits, carrying structured drill-down
locations for full-text matches inside files — a 1-based page number for PDF
matches, or an ELAN tier and time range for transcription matches — each with
highlight fragments. Portals can deep-link users straight to the matching page
of a PDF or the matching utterance in a media player.

Segments are a discriminated union on `type`, so generated client types give
you precise per-variant fields. New segment types arrive by spec revision, and
clients skip types they don't recognise, so deployed clients keep working as
the union grows.

## Where to go next

- The [Extensions guide](/docs/extensions) explains the extension model, the
  registry, and the client rules; the [segments page](/docs/extensions/segments)
  covers the first extension in depth, including implementation notes.
- The [API reference](/docs/api) documents `/capabilities` and the segment
  schemas.
- The specification now has a
  [changelog](https://github.com/crate-works/ro-crate-api/blob/main/CHANGELOG.md)
  recording changes from 0.1.0 onwards.
