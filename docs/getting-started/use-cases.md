---
sidebar_position: 4
title: Common Use Cases
---

# Common Use Cases

This guide demonstrates common patterns and workflows when working with
RO-Crate APIs.

## 1. Browsing Collections Hierarchically

RO-Crate collections can be organised hierarchically. Here's how to navigate
them:

### Finding Root Collections

Start by finding top-level collections:

```bash
curl "https://data.ldaca.edu.au/api/entities?conformsTo=https://w3id.org/ldac/profile%23Collection&limit=20"
```

### Exploring Sub-Collections

Use the `memberOf` parameter to find sub-collections:

```bash
# Find collections that belong to a parent collection
curl "https://data.ldaca.edu.au/api/entities?memberOf=https://catalog.paradisec.org.au/repository/NT1"
```

## 2. Searching for Language Data

Language archives often need sophisticated search capabilities:

### Basic Text Search

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "indigenous languages",
    "limit": 20
  }'
```

### Advanced Boolean Search

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "searchType": "advanced",
    "query": "name: \"West Alor\" AND description: linguistic",
    "limit": 10
  }'
```

### Filtered Search with Facets

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "language documentation",
    "filters": {
      "inLanguage": ["English", "Italian"],
      "mediaType": ["audio/wav", "text/plain"]
    },
    "limit": 20
  }'
```

### Geographic Search

Find resources within a specific geographic area:

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "*",
    "boundingBox": {
      "topRight": {
        "lat": -33.7,
        "lng": 151.3
      },
      "bottomLeft": {
        "lat": -34.1,
        "lng": 150.9
      }
    },
    "geohashPrecision": 7,
    "limit": 50
  }'
```

## 3. Retrieving RO-Crate Metadata

### Get Complete RO-Crate JSON-LD

Access the raw RO-Crate metadata for any entity:

```bash
curl "https://data.ldaca.edu.au/api/entity/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001/metadata"
```

**Response:**

```json
{
  "@context": "https://w3id.org/ro/crate/1.1/context",
  "@graph": [
    {
      "@id": "ro-crate-metadata.json",
      "@type": "CreativeWork",
      "conformsTo": {
        "@id": "https://w3id.org/ro/crate/1.1"
      },
      "about": {
        "@id": "./"
      }
    },
    {
      "@id": "./",
      "@type": "Dataset",
      "name": "Recordings of West Alor languages",
      "description": "A compilation of recordings featuring various West Alor languages"
    }
  ]
}
```

This is useful for:

- Validating RO-Crate compliance
- Accessing extended metadata not exposed in the entity API
- Archival and preservation workflows
- Integration with RO-Crate tools

## 4. Working with Files

### Understanding Entities vs Files

The API provides two ways to work with files:

- **`/entities`** - Returns RO-Crate entities including MediaObjects (files that are part of the RO-Crate metadata)
- **`/files`** - Returns files from the repository's file system

**Important**: Not all files are represented as RO-Crate entities. MediaObject entities are typically a subset of all files in the repository.

### Listing Files from the File System

List all files in the repository:

```bash
# List all files
curl "https://data.ldaca.edu.au/api/files"

# List files attached to a specific entity
curl "https://data.ldaca.edu.au/api/files?memberOf=https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001"

# Paginate through files
curl "https://data.ldaca.edu.au/api/files?limit=50&offset=0"
```

### Listing MediaObject Entities

List files that are part of the RO-Crate:

```bash
curl "https://data.ldaca.edu.au/api/entities?entityType=http://schema.org/MediaObject"
```

For MediaObject entities, the entity `id` is also the file identifier — use it directly with the `/file/{id}` endpoint:

```json
{
  "id": "https://catalog.paradisec.org.au/repository/LRB/001/recording.wav",
  "name": "recording.wav",
  "entityType": "http://schema.org/MediaObject",
  ...
}
```

### Accessing File Content

```bash
# Direct file download
curl "https://data.ldaca.edu.au/api/file/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001%2Frecording.wav" -o recording.wav
```

### Download as Attachment

Force download with a custom filename:

```bash
curl "https://data.ldaca.edu.au/api/file/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001%2Frecording.wav?disposition=attachment&filename=my-recording.wav"
```

### Getting File Location

Get the file location without downloading:

```bash
curl "https://data.ldaca.edu.au/api/file/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001%2Frecording.wav?noRedirect=true"
```

**Response:**

```json
{
  "location": "https://s3.amazonaws.com/bucket/recording.wav?signed-url"
}
```

### Partial Content Download

Use HTTP range requests for streaming or resuming downloads:

```bash
# Download first 1KB
curl -H "Range: bytes=0-1023" \
  "https://data.ldaca.edu.au/api/file/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001%2Frecording.wav"
```

## 5. Paginating Through Large Result Sets

### Basic Pagination

```bash
# First page
curl "https://data.ldaca.edu.au/api/entities?limit=100&offset=0"

# Second page
curl "https://data.ldaca.edu.au/api/entities?limit=100&offset=100"

# Third page
curl "https://data.ldaca.edu.au/api/entities?limit=100&offset=200"
```

## 6. Working with Communication Modes

Language archives often categorise data by communication mode:

### Finding Spoken Language Data

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "*",
    "filters": {
      "communicationMode": ["SpokenLanguage"]
    }
  }'
```

### Finding Sign Language Resources

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "*",
    "filters": {
      "communicationMode": ["SignedLanguage"]
    }
  }'
```

### Multi-modal Search

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "traditional songs",
    "filters": {
      "communicationMode": ["Song", "SpokenLanguage"]
    }
  }'
```

## 7. Building Search Interfaces with Facets

### Getting Faceted Search Results

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "language",
    "limit": 20
  }'
```

**Response includes facets:**

```json
{
  "total": 156,
  "entities": [...],
  "facets": {
    "inLanguage": [
      {"name": "English", "count": 45},
      {"name": "Spanish", "count": 23},
      {"name": "French", "count": 12}
    ],
    "mediaType": [
      {"name": "audio/wav", "count": 67},
      {"name": "text/plain", "count": 45},
      {"name": "video/mp4", "count": 23}
    ]
  }
}
```

## Performance Tips

1. **Use appropriate page sizes**: 50-100 entities per request
2. **Implement caching**: Cache frequently accessed entities
3. **Sort consistently**: Use `sort` parameter for predictable pagination
4. **Respect rate limits**: Add delays between requests
5. **Use specific filters**: Reduce result sets with targeted filters
6. **Batch operations**: Group related requests together
7. **Handle errors gracefully**: Implement retry logic with exponential backoff
