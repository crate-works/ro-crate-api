---
title: About
---

# About the RO-Crate API

The RO-Crate API project emerged from the need to provide consistent,
programmatic access to research data collections that conform to the [RO-Crate
specification](https://www.researchobject.org/ro-crate/).

## Background

Research institutions worldwide collect and curate vast amounts of digital
cultural heritage, language data, and research outputs. While the RO-Crate
specification provides an excellent framework for metadata packaging,
institutions needed a standardised way to:

- **Expose collections programmatically** for integration with research
  workflows
- **Enable discovery** across distributed repositories
- **Support complex search** requirements for specialised domains
- **Maintain consistent APIs** regardless of underlying storage systems

## The Problem We Solve

Before the RO-Crate API specification, each repository implemented their own
custom API, leading to:

- **Integration complexity**: Researchers and developers had to learn multiple
  APIs
- **Limited interoperability**: Data discovery tools couldn't work across
  repositories
- **Inconsistent experiences**: Different error handling, authentication, and
  response formats
- **Reduced discoverability**: Valuable research data remained siloed in
  individual systems

## Our Solution

The RO-Crate API specification provides:

### 🎯 **Standardised Interface**

A single API specification that repositories can implement, enabling tools and
applications to work across different institutions and platforms.

### 🔍 **Rich Search Capabilities**

Advanced search functionality including full-text search, geographic filtering,
faceted search, and domain-specific filters for language and cultural data.

### 🔐 **Flexible Authentication**

Support for public access, API keys, and OAuth2/OpenID Connect to accommodate
different institutional requirements and use cases.

### 📊 **Complete Metadata Access**

Full RO-Crate entity information with support for hierarchical navigation, file
access, and conformance to specialised profiles.

## Current Implementations

The specification is already in production use at major research institutions:

### Language Data Commons of Australia (LDaCA)

**[data.ldaca.edu.au](https://data.ldaca.edu.au)**

LDaCA hosts language and cultural data from across Australia, making it
discoverable and accessible to researchers worldwide. Their RO-Crate API
implementation enables:

- Cross-institutional data discovery
- Integration with analysis workflows
- Support for complex linguistic metadata

### PARADISEC (Pacific and Regional Archive)

**[catalog.paradisec.org.au](https://catalog.paradisec.org.au)**

PARADISEC preserves endangered languages and cultural heritage from the
Asia-Pacific region. Their API implementation supports:

- Geographic search across the Pacific region
- Language-specific filtering and faceting
- Integration with digital preservation workflows

## Project Goals

### Short Term

- **Wider Adoption**: Encourage more repositories to implement the specification
- **Enhanced Features**: Add advanced search capabilities and metadata
  enhancements
- **Better Tooling**: Develop SDKs and client libraries for popular programming
  languages

### Long Term

- **Write Operations**: Support for creating and updating RO-Crate entities
  through the API
- **Distributed Discovery**: Enable federated search across multiple
  repositories
- **Advanced Analytics**: Support for research analytics and usage metrics

## Community and Governance

The specification is maintained by
[CrateWorks](https://crate-works.org), a workbench of open-source tools for
RO-Crate research data.

The RO-Crate API is developed as an open specification with input from:

- **Research institutions** implementing and using the API
- **The RO-Crate community** ensuring alignment with core principles
- **Domain experts** in digital humanities, linguistics, and cultural heritage
- **Software developers** building tools and applications

### Contributing

We welcome contributions from the community:

- **Specification improvements**: Propose enhancements via GitHub issues
- **Implementation feedback**: Share experiences from real-world deployments  
- **Documentation**: Help improve guides and examples
- **Testing**: Validate the specification against different use cases

## Technical Foundation

The API builds on established web standards:

- **OpenAPI 3.1.0**: For precise API specification and documentation
- **JSON-LD**: For rich, linked metadata following RO-Crate conventions
- **HTTP**: Standard protocols for authentication, caching, and error handling
- **REST**: Predictable resource-oriented architecture

## Impact and Future

By standardizing access to RO-Crate collections, we're enabling:

### For Researchers

- **Easier data discovery** across institutional boundaries  
- **Simplified integration** with analysis tools and workflows
- **Better reproducibility** through consistent data access patterns

### For Institutions  

- **Reduced integration costs** through shared API clients and tools
- **Increased data discoverability** and citation
- **Future-proofed systems** built on open standards

### For Developers

- **Consistent interfaces** across different data repositories
- **Rich ecosystem** of tools and applications
- **Lower barrier to entry** for research data applications

## Get Involved

Ready to join the RO-Crate API community?

- **Try the API**: Start with our [Getting Started guide](/docs/getting-started)
- **Implement the spec**: Use our [OpenAPI specification](/docs/api)
- **Join discussions**: Participate in the RO-Crate community
- **Contribute**: Submit issues and improvements on
  [GitHub](https://github.com/crate-works/ro-crate-api)

Together, we're building the foundation for the next generation of research data
infrastructure.
