# Capability fields are required

## Context

Clients build their UI from `GET /capabilities` instead of per-archive
configuration. When a field is missing, a client cannot tell whether the
implementation lacks the feature or whether its capability document is
incomplete. An optional field with a default also lets a server that forgets
the field claim a feature it can't serve.

## Decision

Every capability member that states whether a feature is supported is
required. An implementation states its position explicitly, even when the
answer is "not supported". Support is stated in one of two ways:

- **A `supported` flag** on a required object, such as
  `deposit.supported: false` or `search.geo.supported: false`.
- **A presence map**, such as `extensions` or `search.searchTypes`. The map is
  required, and a key's presence means the feature is supported. An absent key
  means "not supported", never "unknown".

Two kinds of field are optional, because their absence has a defined meaning:

- **Limits**, such as `deposit.maxFileSizeBytes`, `deposit.depositTtlSeconds`
  and `search.maxResultWindow`. Absent means no declared limit.
- **Display details**, such as `label`.

## Consequences

- Adding a required capability member is a breaking change, because existing
  servers must add it.
- Clients read a capability directly and need no defaults for missing fields.
- A feature that might grow details later is declared as an object (a
  `supported` flag or a presence-map value), not as a bare boolean, so new
  fields can be added beside it.
