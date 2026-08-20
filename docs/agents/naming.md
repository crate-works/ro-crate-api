# Naming

What the terms mean is in `CONTEXT.md`. This is how they are spelled and how
operations are named in `openapi.yaml`.

## The four renderings of "RO-Crate"

| Context | Form | Example |
| --- | --- | --- |
| Prose, summaries, descriptions | `RO-Crate` | "the entity's RO-Crate metadata document" |
| Path segments | `ro-crate` | `/ro-crate/{id}/metadata` |
| Schema and identifier names | `RoCrate` | `RoCrateIdParameter`, `getRoCrateMetadata` |
| JSON property names | `roCrate` | `roCrateId`, `roCrateIds`, `roCrates` |

Never `rocrate`, never bare `crate`.

Do not "fix" the one surviving `rocrate`: the published schema slug
`docs/api/schemas/rocrate.schema.mdx`, which the generator derives from the
`RoCrate` schema name. It is a published URL and renaming it is out of scope.

## `operationId`

camelCase: a verb naming the action, then the resource nouns it acts on. The
verb is the domain action, not the HTTP method.

| Kind | Verb | Example |
| --- | --- | --- |
| Read one | `get` / `head` | `getEntityMetadata`, `headRoCrateMetadata` |
| Read a collection | `list` | `listEntities`, `listRoCrates` |
| Write | the domain verb | `createDeposit`, `stageDepositFile`, `finaliseDeposit` |

Two shipped identifiers do not follow this and must not be changed:
`search-entities` and `createUpdateDeposit`.

An `operationId` is public surface — it becomes a reference page slug and a
method name in generated clients — so renaming one is a breaking change.

## Path segments

Lowercase and hyphenated. Name what the endpoint returns rather than the
resource it hangs off: `{resource}/{id}/metadata` for a metadata document, on
both `/entity/{id}` and `/ro-crate/{id}`.
