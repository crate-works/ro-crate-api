---
sidebar_position: 7
title: Error Handling
---

# Error Handling

The RO-Crate API uses standard HTTP status codes and provides detailed error
information to help developers handle errors gracefully.

## Error Response Format

All error responses follow a consistent JSON structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      // Additional error-specific information
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Error Response Fields

- **code**: Machine-readable error identifier
- **message**: Human-readable description
- **details**: Additional context (varies by error type)
- **requestId**: Unique identifier for debugging (UUID format)

## HTTP Status Codes

### 2xx Success

| Code | Description | When it occurs |
|------|-------------|----------------|
| **200 OK** | Request successful | Normal successful responses |
| **206 Partial Content** | Partial file download | Range requests for files |

### 3xx Redirection

| Code | Description | When it occurs |
|------|-------------|----------------|
| **302 Found** | Temporary redirect | File downloads (when not using `noRedirect`) |

### 4xx Client Errors

| Code | Description | When it occurs |
|------|-------------|----------------|
| **400 Bad Request** | Invalid request parameters | Malformed requests, invalid filters, undeclared search types, sorts or geo, offsets past the result window |
| **401 Unauthorized** | Authentication required | Missing or invalid credentials |
| **403 Forbidden** | Access denied | Insufficient permissions |
| **404 Not Found** | Resource not found | Invalid entity ID, file not found |
| **416 Range Not Satisfiable** | Invalid range request | Bad Range header for file downloads |
| **422 Unprocessable Entity** | Invalid query syntax | Advanced queries that do not parse or name undeclared fields |
| **429 Too Many Requests** | Rate limit exceeded | Too many requests in time window |

### 5xx Server Errors

| Code | Description | When it occurs |
|------|-------------|----------------|
| **500 Internal Server Error** | Server error | Unexpected server failures |

## Common Error Types

### 1. Validation Errors (400)

**Scenario**: Invalid parameters or request format

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
        },
        {
          "field": "conformsTo",
          "message": "invalid profile URL",
          "value": "invalid-url"
        }
      ]
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Common Causes:**

- Invalid `limit` or `offset` values
- Malformed entity IDs
- Invalid filter parameters
- Missing required fields
- A `searchType`, `sort` or filter the implementation does not declare in
  `/capabilities`
- `boundingBox` or `geohashPrecision` when geo is not supported, or
  `geohashPrecision` without `boundingBox`
- `offset + limit` past the declared `search.maxResultWindow`

### 2. Authentication Errors (401)

**Scenario**: Missing or invalid authentication

```json
{
  "error": {
    "code": "AUTHENTICATION_REQUIRED",
    "message": "Valid authentication credentials are required",
    "details": {
      "supportedMethods": ["ApiKey", "OAuth2", "OpenID"]
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440001"
  }
}
```

**Common Causes:**

- Missing API key or token
- Expired OAuth2 token
- Invalid API key format
- Malformed Authorization header

### 3. Authorization Errors (403)

**Scenario**: Insufficient permissions

```json
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "Access token does not have the required scope",
    "details": {
      "requiredScopes": ["read"],
      "providedScopes": [],
      "resource": "entities"
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440002"
  }
}
```

### 4. Not Found Errors (404)

**Scenario**: Resource doesn't exist

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "The requested entity was not found",
    "details": {
      "entityId": "https://catalog.paradisec.org.au/repository/MISSING/001",
      "resourceType": "entity"
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440003"
  }
}
```

### 5. Rate Limiting Errors (429)

**Scenario**: Too many requests

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please retry after the specified time.",
    "details": {
      "retryAfter": 60,
      "limit": 1000,
      "remaining": 0,
      "resetTime": 1609459260
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440004"
  }
}
```

**Response Headers:**

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1609459260
Retry-After: 60
```

### 6. Search Query Errors (422)

**Scenario**: An advanced query that does not parse, or that names a field
the implementation does not declare

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid query at position 28: expected a field value or opening bracket",
    "details": {
      "violations": [
        {
          "field": "query",
          "message": "expected a field value or opening bracket",
          "value": "name: John AND (description:",
          "position": 28
        }
      ]
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440005"
  }
}
```

The `message` says what was expected. `position` is optional: when present,
it is the 0-based offset, in Unicode code points, where the error was found,
so a client can highlight that part of the query.
