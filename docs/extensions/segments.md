---
sidebar_position: 1
title: "Extension: segments"
mdx.format: md
---

# The `segments` Extension

Full-text search tells a user *which file* matched their query. For a
three-hundred-page PDF of field notes or an hour of transcribed audio, that is
not enough — the user still has to hunt for the match inside the file. The
`segments` extension closes that gap: each search hit can carry a list of
**segments**, structured locations inside the file where the match occurred,
so clients can deep-link the user straight to the matching page of a PDF or
the matching utterance in a media player.

Segments appear as an optional `searchExtra.segments` array on
[search](/docs/api/search-entities) hits. The array is ranked by relevance and
capped at an implementation-defined limit.

## Declaring the Capability

An implementation that provides segments declares the extension in
[`/capabilities`](/docs/getting-started/capabilities):

```json
{
  "apiVersion": "0.3.0",
  "extensions": {
    "segments": {}
  }
}
```

The extension currently has no extra details to communicate, so the value is
an empty object — presence of the `segments` key is the declaration.

Clients detect support with a key lookup
(`"segments" in capabilities.extensions`) and MUST degrade gracefully when
the key is absent — hide the deep-link affordance, don't fail.

## The Response Shape

Each segment is one of the registered segment types, discriminated by `type`.
Two types are registered:

### `page`

A match located on a page of a paginated document, such as a PDF.

```json
{
  "type": "page",
  "page": 3,
  "highlight": ["a wordlist of <em>West Alor</em> terms for kinship"]
}
```

- **`page`** (integer, ≥ 1): the 1-based page number the match occurred on
- **`highlight`** (array of strings): matched text fragments from this page,
  using the same marking convention as `searchExtra.highlight`

### `time-aligned-annotation`

A match located in a time-aligned annotation, such as an ELAN annotation
tier.

```json
{
  "type": "time-aligned-annotation",
  "tier": "A_phrase-segnum-en",
  "startMs": 83000,
  "endMs": 87500,
  "highlight": ["the speaker lists <em>West Alor</em> place names"]
}
```

- **`tier`** (string): the identifier of the annotation tier the match
  occurred in (for ELAN, the `TIER_ID`)
- **`startMs`** / **`endMs`** (integers, ≥ 0): the annotation's time range,
  in milliseconds from the beginning of the media
- **`highlight`** (array of strings): matched text fragments from this
  annotation

### A Full Search Hit

A search for `West Alor` might return this hit for a PDF of field notes with
an accompanying transcription:

```json
{
  "id": "https://catalog.paradisec.org.au/repository/NT1/001/NT1-001-001A.pdf",
  "name": "NT1-001-001A.pdf",
  "entityType": "http://schema.org/MediaObject",
  "searchExtra": {
    "score": 0.87,
    "highlight": { "content": ["notes on <em>West Alor</em> vocabulary"] },
    "segments": [
      {
        "type": "page",
        "page": 3,
        "highlight": ["a wordlist of <em>West Alor</em> terms for kinship"]
      },
      {
        "type": "time-aligned-annotation",
        "tier": "A_phrase-segnum-en",
        "startMs": 83000,
        "endMs": 87500,
        "highlight": ["the speaker lists <em>West Alor</em> place names"]
      }
    ]
  }
}
```

A client can render "matched on page 3" as a link opening the PDF viewer at
that page, and "matched at 1:23" as a link starting media playback at 83
seconds.

## Client Rules

- **Segments are optional per hit.** Absent or empty means the hit had no
  structured content to point into — the hit itself is still valid.
- **Skip unknown types.** New segment types are added by revision of this
  specification. A client that encounters a `type` it does not recognise MUST
  skip that segment rather than fail, so deployed clients keep working as the
  union grows.
- **Expect a cap.** Segments are ranked by relevance and capped at an
  implementation-defined limit per hit, so a full-looking list is not
  necessarily exhaustive.

New segment types are proposed by pull request against the
[specification repository](https://github.com/Language-Research-Technology/ro-crate-api) —
see [the registry](./#the-curated-registry).

## Implementation Notes (Non-Normative)

This section is guidance, not specification. It sketches one proven way to
implement segments with Elasticsearch; any implementation that produces
conformant responses is equally valid.

### Index-time extraction

Segments must exist in the index before they can be searched. At ingest
time, extract per-segment records from each file:

- **PDFs and other paginated documents**: extract text page by page (e.g.
  with `pdftotext` or Apache Tika), producing one record per page with its
  1-based page number.
- **ELAN files (`.eaf`)**: walk each annotation tier, producing one record
  per annotation with the tier's `TIER_ID` and the annotation's time slot
  values in milliseconds.

### Mapping

Store the extracted records as [`nested`](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html)
documents on the file's entity document, so each segment's fields stay
associated with each other:

```json
{
  "mappings": {
    "properties": {
      "segments": {
        "type": "nested",
        "properties": {
          "type": { "type": "keyword" },
          "text": { "type": "text" },
          "page": { "type": "integer" },
          "tier": { "type": "keyword" },
          "startMs": { "type": "long" },
          "endMs": { "type": "long" }
        }
      }
    }
  }
}
```

### Querying

Combine the entity-level query with a `nested` query over the segments, and
use [`inner_hits`](https://www.elastic.co/guide/en/elasticsearch/reference/current/inner-hits.html)
to retrieve the matching segments — `inner_hits` returns the top-scoring
nested documents per hit, which is exactly the ranked, capped list the
extension requires:

```json
{
  "query": {
    "bool": {
      "should": [
        { "match": { "content": "West Alor" } },
        {
          "nested": {
            "path": "segments",
            "query": { "match": { "segments.text": "West Alor" } },
            "inner_hits": {
              "size": 5,
              "highlight": { "fields": { "segments.text": {} } }
            }
          }
        }
      ]
    }
  }
}
```

Set `inner_hits.size` to your chosen per-hit segment cap. Each inner hit maps
directly onto a response segment: the `type` field selects `page` or
`time-aligned-annotation`, the per-type fields come from the nested source,
and the `highlight` fragments become the segment's `highlight` array.
