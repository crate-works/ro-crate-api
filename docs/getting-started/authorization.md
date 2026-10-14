---
sidebar_position: 4
title: Authorization
---

# Authorization

The RO-Crate API uses an `access` property on each entity which signals to the
consumer that level of access the current user has to metadata and content.
This is designed to allow consumers of the API like Oni UI to communicate what
is accessible and redirect to enrollment URLs when access is restricted.

## Access Property Structure

The API uses an `access` property to communicate access permissions. The structure differs between **Entities**, **Files** and **RO-Crates**:

### Entity Access Structure

Entities (Collections, Objects, and MediaObjects) include both metadata and content access controls:

```json
{
  "access": {
    "metadata": true,
    "content": false,
    "metadataAuthorizationUrl": "https://example.com/apply?item=123",
    "contentAuthorizationUrl": "https://example.com/apply?item=123"
  }
}
```

**Required Fields:**
- **`metadata`** (boolean): Whether the current user has access to view the entity's metadata
- **`content`** (boolean): Whether the current user has access to download or view the entity's content files

**Optional Fields:**
- **`metadataAuthorizationUrl`** (string): URL where users can request access to metadata when `metadata` is `false`
- **`contentAuthorizationUrl`** (string): URL where users can request access to content when `content` is `false`

### File Access Structure

Files (accessed via `/files` endpoints) only include content access controls, as file metadata is always accessible:

```json
{
  "access": {
    "content": true
  }
}
```

**Required Fields:**
- **`content`** (boolean): Whether the current user has access to download the file

**Optional Fields:**
- **`contentAuthorizationUrl`** (string): URL where users can request access to content when `content` is `false`

### RO-Crate Access Structure

RO-Crates (accessed via `/ro-crate` endpoints, where the implementation supports
deposit) use the same fields as Entities, but `content` means something
different:

- **`metadata`** (boolean): Whether the current user has access to the RO-Crate
  itself (`GET /ro-crate/{id}`)
- **`content`** (boolean): Whether the current user has access to the deposited
  RO-Crate Metadata Document (`GET /ro-crate/{id}/metadata`)

An RO-Crate's `content` does not govern files. Each File's own `access`
governs its retrieval.

One RO-Crate can materialise many entities, and its RO-Crate Metadata Document
holds the metadata of all of them. Implementations SHOULD set `content` to
`true` only if the current user has `metadata` access to every entity
materialised from the RO-Crate.

**Note:** All examples in the sections below demonstrate Entity access patterns. For Files, only the content-related fields apply

## Authorization Rules

### Validation Requirements

When implementing the API, the following validation rules MUST be enforced:

- If `metadata` is `false`, then `metadataAuthorizationUrl` MUST be provided
- If `content` is `false`, then `contentAuthorizationUrl` MUST be provided
- Entities that violate these rules should not be surfaced in API responses

### Implementation Flexibility

The authorization system provides maximum flexibility to implementors:

- **Enrollment URL format**: Completely open to implementor design
- **Authorization flow**: The process after visiting an enrollment URL is up to
the implementor
- **Access determination**: How the API determines user access is
implementation-specific

## Access Scenarios and Examples

### 1. Full Access

Both metadata and content are accessible to the current user:

```json
{
  "id": "https://catalog.example.com/repository/PUBLIC/001",
  "name": "Public Research Data",
  "access": {
    "metadata": true,
    "content": true
  }
}
```

### 2. Metadata Only Access

User can view metadata but must request access for content:

```json
{
  "id": "https://catalog.example.com/repository/RESTRICTED/001",
  "name": "Sensitive Audio Recordings",
  "access": {
    "metadata": true,
    "content": false,
    "contentAuthorizationUrl": "https://ethics.example.com/apply?collection=RESTRICTED-001"
  }
}
```

### 3. No Access

User must request access for both metadata and content:

```json
{
  "id": "https://catalog.example.com/repository/PRIVATE/001",
  "name": "Confidential Collection",
  "access": {
    "metadata": false,
    "content": false,
    "metadataAuthorizationUrl": "https://portal.example.com/metadata-access?id=PRIVATE-001",
    "contentAuthorizationUrl": "https://portal.example.com/content-access?id=PRIVATE-001"
  }
}

```

### 4. Content Only Access

User can access content but not detailed metadata (rare scenario):

```json
{
  "id": "https://catalog.example.com/repository/CLASSIFIED/001",
  "name": "Research Materials",
  "access": {
    "metadata": false,
    "content": true,
    "metadataAuthorizationUrl": "https://clearance.example.com/request?item=CLASSIFIED-001"
  }
}
```

### 5. Invalid Configurations

These examples show invalid configurations that should result in errors:

```json
// ERROR: content is false but no contentAuthorizationUrl provided
{
  "access": {
    "metadata": true,
    "content": false
    // Missing contentAuthorizationUrl
  }
}

// ERROR: metadata is false but no metadataAuthorizationUrl provided
{
  "access": {
    "metadata": false,  // Missing metadataAuthorizationUrl
    "content": true
  }
}
```

## Client Implementation Guidelines

### Checking Access

Before attempting to access entity metadata or content, clients should check
the access property:

```javascript
function canAccessMetadata(entity) {
  return entity.access.metadata === true;
}

function canAccessContent(entity) {
  return entity.access.content === true;
}

function getEnrollmentUrl(entity, type) {
  if (type === 'metadata' && !entity.access.metadata) {
    return entity.access.metadataAuthorizationUrl;
  }

  if (type === 'content' && !entity.access.content) {
    return entity.access.contentAuthorizationUrl;
  }
  return null;
}
```

### Handling Restricted Access

When access is denied, present the enrollment URL to users:

```javascript
if (!canAccessContent(entity)) {
  const enrollmentUrl = getEnrollmentUrl(entity, 'content');
  // Display message: "Access restricted. Apply for access at: {enrollmentUrl}"
}
```

## Use Cases

### Public Archives

Most content publicly accessible with some restricted materials:

- `metadata: true, content: true` - Open research data
- `metadata: true, content: false` - Culturally sensitive materials requiring
approval

### Institutional Repositories

Metadata discoverable but content restricted to authenticated users:

- `metadata: true, content: false` - Internal research requiring institutional login

### Sensitive Collections

Both metadata and content require special authorization:

- `metadata: false, content: false` - Classified or highly sensitive materials

## Implementation Notes

- Authorization URLs can point to any system: web forms, OAuth providers,
institutional portals, etc.
- The API does not prescribe the enrollment process - this is entirely up to
the implementor
- Access permissions may be user-specific, time-based, or dependent on other
factors
- Consider caching authorization decisions to improve performance
- Ensure authorization URLs are accessible and provide clear instructions to
users
