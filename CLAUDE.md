# ro-crate-api

Docusaurus site documenting the RO-Crate API, including OpenAPI-generated API docs.

## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues via the `gh` CLI; external PRs are not a triage surface. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles use their default names (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` at the repo root and ADRs in `docs/adr/` (both excluded from the Docusaurus build). See `docs/agents/domain.md`.

## Naming conventions

Terminology and the three spellings of "RO-Crate" live in `CONTEXT.md`. Two
authoring rules that are not glossary entries:

- **`operationId` = verb + path nouns.** `GET /entity/{id}/metadata` is
  `getEntityMetadata`; `HEAD /ro-crate/{id}/metadata` is
  `headRoCrateMetadata`. camelCase throughout. These identifiers become
  published reference-page slugs and generated client method names, so
  changing one is a breaking change. `search-entities` predates the rule and
  is left alone for that reason.
- **Path segments are lowercase and hyphenated**, and name what the endpoint
  returns rather than the resource it hangs off — hence
  `{resource}/{id}/metadata` for a metadata document.
