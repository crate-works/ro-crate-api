---
sidebar_position: 2
title: Overview
mdx.format: md
---

# Getting Started

This guide will help you make your first API call to an RO-Crate API
implementation.

## Prerequisites

- A running RO-Crate API implementation (see [AROCAPI](https://github.com/Language-Research-Technology/arocapi))
- Basic understanding of REST APIs
- HTTP client (curl, Postman, or programming language HTTP library)

## Base URL

Each RO-Crate API implementation will have its own base URL. Examples:

- LDaCA: `https://data.ldaca.edu.au/api`
- PARADISEC: `https://admin-catalog.paradisec.org.au/api/v1/oni`

## Authentication

The RO-Crate API supports three authentication methods:

### 1. No Authentication (Public Access)

Many endpoints may be publicly accessible:

```bash
curl https://data.ldaca.edu.au/api/entities
```

### 2. API Key

For server-to-server integrations:

```bash
curl -H "X-API-Key: your-api-key" \
  https://data.ldaca.edu.au/api/entities
```

### 3. OAuth2/OpenID Connect For user-facing applications (see [Authentication

Guide](./authentication.md) for details).

## Making Your First Request

### List Entities

The simplest request is to list entities:

```bash
curl https://data.ldaca.edu.au/api/entities
```

**Response:**

```json
{
  "total": 42,
  "entities": [
    {
      "id": "https://catalog.paradisec.org.au/repository/LRB/001",
      "name": "Recordings of West Alor languages",
      "description": "A compilation of recordings featuring various West Alor languages",
      "conformsTo": "https://w3id.org/ldac/profile#Collection",
      "recordType": "RepositoryCollection",
      "createdAt": "2023-01-01T12:00:00Z",
      "updatedAt": "2023-01-02T12:00:00Z"
    }
  ]
}
```

### Get a Specific Entity

Use the entity's ID to get detailed information:

```bash
curl https://data.ldaca.edu.au/api/entity/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001
```

Note: The entity ID must be URL-encoded when used in the path.

### Search Entities

Perform a basic text search:

```bash
curl -X POST https://data.ldaca.edu.au/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "language recordings",
    "limit": 10
  }'
```

### Get RO-Crate Metadata

Retrieve the raw RO-Crate JSON-LD metadata for any entity:

```bash
curl https://data.ldaca.edu.au/api/entity/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001/metadata
```

This returns the complete RO-Crate metadata conforming to the RO-Crate specification.

### List Files

List all files in the repository:

```bash
curl https://data.ldaca.edu.au/api/files
```

You can filter files by memberOf to show files attached to a specific entity:

```bash
curl https://data.ldaca.edu.au/api/files?memberOf=https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001
```

**Note**: The `/files` endpoint returns files from the repository's file system. Not all files are represented as RO-Crate entities. To list MediaObject entities (files that are part of the RO-Crate), use `/entities?entityType=http://schema.org/MediaObject`.

### Access File Content

For MediaObject entities, you can directly access the file content:

```bash
curl https://data.ldaca.edu.au/api/file/https%3A%2F%2Fcatalog.paradisec.org.au%2Frepository%2FLRB%2F001%2Ffile.wav
```

This endpoint supports:

- Content disposition (inline or attachment)
- Custom filenames
- HTTP range requests for partial content
- Redirects to file storage locations

## Understanding Responses

### Success Responses

All successful responses include:

- **200 OK**: Request successful
- **Content-Type: application/json**
- Structured data following the API schema

### Error Responses

Error responses follow a consistent format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": {
      "violations": [
        {
          "field": "limit",
          "message": "must be between 1 and 1000",
          "value": 2000
        }
      ]
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Rate Limiting

If you encounter rate limiting, you'll receive:

- **429 Too Many Requests**
- `X-RateLimit-Limit`: Request limit per time window
- `X-RateLimit-Remaining`: Requests remaining
- `X-RateLimit-Reset`: When the limit resets
- `Retry-After`: Seconds to wait before retrying

## Next Steps

- [Authentication Guide](./authentication.md) - Set up OAuth2 or API key
  authentication
- [Use Cases](./use-cases.md) - Common patterns and workflows
- [Error Handling](./error-handling.md) - Handle errors gracefully
