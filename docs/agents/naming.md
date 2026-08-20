# Naming

Terminology — what an RO-Crate is, what a metadata document is — lives in
`CONTEXT.md`. This file covers how those terms are spelled and how operations
are named in `openapi.yaml`.

## The four renderings of "RO-Crate"

One concept, four forms, chosen by where the word sits:

| Context | Form | Example |
| --- | --- | --- |
| Prose, summaries, descriptions | `RO-Crate` | "the entity's RO-Crate metadata document" |
| Path segments | `ro-crate` | `/ro-crate/{id}/metadata` |
| Schema and identifier names | `RoCrate` | `RoCrateIdParameter`, `getRoCrateMetadata` |
| JSON property names | `roCrate` | `roCrateId`, `roCrateIds`, `roCrates` |

Never `rocrate`, never bare `crate`. Both were in the 0.1.0 surface: the path
form went in 0.4.0, and the `RoCrate` schema's published slug
(`docs/api/schemas/rocrate.schema.mdx`, linked from `docs/deposit/index.md`)
is still lowercase because the generator derives it from the schema name.
Renaming that slug is deferred, not settled — see crate-works/ro-crate-api#33.

## `operationId`

camelCase, and shaped as **a verb naming the action, then the resource nouns
the operation acts on**. The verb is the domain action, not the HTTP method:

| Kind | Verb | Example |
| --- | --- | --- |
| Read one | `get` / `head` | `getEntityMetadata`, `headRoCrateMetadata` |
| Read a collection | `list` | `listEntities`, `listRoCrates` |
| Write | the domain verb | `createDeposit`, `stageDepositFile`, `finaliseDeposit`, `abortDeposit` |

Two shipped identifiers predate the rule and are left alone because changing
one is a breaking change: `search-entities` (kebab-case) and
`createUpdateDeposit` (`POST /ro-crate/{id}/deposits`, which names neither of
its path nouns).

That breakage is the reason to get these right first time. An `operationId`
becomes the slug of a published reference page and the method name in every
generated client library, so it is part of the public surface.

## Path segments

Lowercase and hyphenated. Name what the endpoint returns rather than the
resource it hangs off — hence `{resource}/{id}/metadata` for a metadata
document, on both `/entity/{id}` and `/ro-crate/{id}`.
