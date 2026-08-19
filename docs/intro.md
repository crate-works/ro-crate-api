---
sidebar_position: 1
title: Introduction
---

# Research Object Crate API (RO-Crate-API)

The RO-Crate API provides a standardised REST interface for accessing and
managing [RO-Crate](https://www.researchobject.org/ro-crate/) collections.
RO-Crate is a community effort to establish a lightweight approach to
packaging research data with machine-readable metadata, making it easier
to share, preserve, and reuse research outputs.

## What is RO-Crate?

Research Object Crate (RO-Crate) is a lightweight approach to packaging
research data with their metadata. It uses JSON-LD and schema.org to provide
rich, structured descriptions of research datasets, software, workflows, and
other digital objects. RO-Crate makes research data more findable, accessible,
interoperable, and reusable (FAIR).

## Why an API?

While RO-Crate defines excellent standards for metadata packaging, institutions
need programmatic ways to:

- **Discover** research data across repositories
- **Access** metadata and files programmatically  
- **Search** collections with complex queries
- **Integrate** with existing research workflows
- **Build** applications on top of research data

The RO-Crate API specification addresses these needs by providing a consistent
REST interface that abstracts the underlying storage systems and search
engines.

## Key Features

### 🔍 **Powerful Search**

- Full-text search across metadata
- Advanced boolean queries
- Geographic search with bounding boxes
- Faceted search with filters
- Language and communication mode filtering

### 🔐 **Flexible Authentication**

- Public access for open data
- API key authentication for integrations
- OAuth2/OpenID Connect for user applications
- Fine-grained access control

### 📊 **Rich Metadata**

- Complete RO-Crate entity information
- Hierarchical collection browsing (Collections, Objects, and MediaObjects)
- Access to raw RO-Crate metadata
- File access and download
- Conformance to LDAC profiles

### 📥 **Deposit**

- Write access through deposit sessions against RO-Crates
- Inline or presigned file upload
- Atomic publish, with catalog entities materialised from what was deposited
- Optional — read-only catalogs remain fully conformant

### 🚀 **Developer Friendly**

- RESTful design following best practices
- Comprehensive OpenAPI specification
- Rate limiting with clear headers
- Detailed error responses with request IDs

## Current Implementations

The RO-Crate API is already in use by several major research data repositories:

- **[LDaCA](https://data.ldaca.edu.au)**: Language Data Commons of Australia
- **[PARADISEC](https://catalog.paradisec.org.au)**: Pacific and Regional
  Archive for Digital Sources in Endangered Cultures

## API Capabilities

### Entity Management

- List entities with filtering and pagination
- List files with filtering and pagination
- Retrieve detailed entity information for Collections, Objects, and MediaObjects
- Navigate collection hierarchies
- Access file content directly
- Retrieve raw RO-Crate JSON-LD metadata

### Advanced Search

- Free-text search across all metadata
- Boolean query support for complex searches
- Geographic search within bounding boxes
- Faceted search with dynamic filters
- Communication mode and language filtering

### File Access

- Direct file downloads
- Partial content support (byte-range requests)
- Configurable content disposition
- Location-based redirects for distributed storage

## Understanding Entities vs Files

The API provides two complementary ways to access content:

### `/entities` Endpoints

The `/entities` endpoints return RO-Crate entities, which can be:

- **Collections** - Groups of related items
- **Objects** - Individual items that may contain files
- **MediaObjects** - Individual files that are part of the RO-Crate

For MediaObject entities, the entity `id` is also the file identifier — use it directly with the `/file/{id}` and `/files` endpoints to access or list the corresponding file.

### `/files` Endpoints

The `/files` endpoints return files from the repository's file system.

**Important**: In a repository, not all files are necessarily represented as file entities in an RO-Crate. Therefore:

- MediaObject entities will typically be a **subset** of all files
- Some files may be accessible via `/files` but not appear in `/entities`
- Files that are part of the RO-Crate metadata will have corresponding MediaObject entities

Use `/entities` when you need RO-Crate metadata about files, and `/files` when you need to browse or access the repository's file system directly.

## Getting Started

Choose your path based on your needs:

### 🚀 **Quick Start**

New to the API? Start here for your first API call:
→ [Getting Started Guide](./getting-started/overview.md)

### 🔑 **Authentication Setup**  

Need to access private data or set up authentication?
→ [Authentication Guide](./getting-started/authentication.md)

### 💡 **Common Patterns**

Looking for specific workflows and use cases?
→ [Use Cases Guide](./getting-started/use-cases.md)

### 🔧 **API Reference**

Need detailed endpoint documentation?
→ [API Reference](./api/)

## Community and Contribution

The RO-Crate API specification is developed openly:

- **Specification Repository**:
  [github.com/Language-Research-Technology/ro-crate-api](https://github.com/Language-Research-Technology/ro-crate-api)
- **Issue Tracking**: Report bugs and request features
- **Discussion**: Join the RO-Crate community discussions

## Standards and Compliance

The API builds on established standards:

- **OpenAPI 3.1.0** for API specification
- **JSON-LD** for metadata serialization
- **Schema.org** for vocabulary
- **OAuth2/OpenID Connect** for authentication
- **HTTP Range Requests** for file operations

Ready to get started? Head to the [Getting Started Guide](./getting-started/overview.md)
for your first API call!
