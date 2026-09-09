# Drizzle v1 — what changes, and how to tell docs apart

**This skill targets stable `drizzle-orm` 0.45.x / `drizzle-kit` 0.31.x.**
`drizzle-orm@1.0` is in **release candidate** (`npm i drizzle-orm@rc
drizzle-kit@rc`). Use this file to (a) recognise when a doc page or code sample
is describing v1 rather than 0.45, and (b) understand the migration if a project
is on / moving to v1.

Sources: `/docs/v0-v1-changes`, `/docs/relations-v1-v2`, `/docs/upgrade-v1`
(these pages exist in the docs and are explicitly v1).

## "Am I looking at v1 docs?" — quick tells

| If you see… | It's… | On 0.45 use instead |
| --- | --- | --- |
| `import { defineRelations } from "drizzle-orm"` | v1 | `relations()` per table |
| `db._query.users.findMany(...)` | v1 (legacy RQB alias) | `db.query.users.findMany(...)` |
| `where: { id: 1, name: { like: "A%" } }` (object filter in RQB) | v1 (RQB v2) | `where: (u, { eq }) => eq(u.id, 1)` |
| `snakeCase.table(...)` / `camelCase.table(...)` | v1 | `drizzle({ casing: "snake_case" })` |
| `column.array("[][]")` (string arg for multi-dim) | v1 | `column.array().array()` (chain for multi-dim on 0.45) |
| `import { createInsertSchema } from "drizzle-orm/zod"` | v1 | `from "drizzle-zod"` (separate package) |
| `pgTable.withRLS("users", {...})` | v1 | `pgTable("users", {...}).enableRLS()` |
| `getColumns(table)` | v1 | `getTableColumns(table)` |
| `r.many.groups({ from: r.users.id.through(r.utg.userId), ... })` | v1 (`through` m2m) | junction table + relate through it |
| `npm i drizzle-orm@rc` / `@1.0.0-beta.*` / `@1.0.0-rc.*` | v1 | `npm i drizzle-orm` (→ 0.45.x) |

The live docs site is mid-migration and mixes both. The vendored docs snapshot
in this repo (`vendor/drizzle-orm-docs` @ `11695b7`) is the last **pre-v1**
state and matches 0.45.

## Breaking changes in v1 (summary)

### Relations — RQB v2

- `relations()` (one call per table, all passed to `drizzle`) is **removed**.
  Replaced by a single `defineRelations(schema, (r) => ({...}))` passed as
  `drizzle(url, { relations })` (note: `relations`, not `schema`).
  ```ts
  // v1
  import { defineRelations } from "drizzle-orm";
  export const relations = defineRelations(schema, (r) => ({
    users: { posts: r.many.posts(), invitee: r.one.users({ from: r.users.invitedBy, to: r.users.id }) },
    posts: { author: r.one.users({ from: r.posts.authorId, to: r.users.id }) },
  }));
  const db = drizzle(url, { relations });
  ```
- `fields` → `from`, `references` → `to` (both accept a value or array).
  `relationName` → `alias`.
- `many()` no longer needs a matching `one()` on the other side.
- New `optional: false` (makes a `one` relation non-nullable at the type level),
  `where` predefined filters on the target table (polymorphic-ish relations).
- **`through` for many-to-many** — declare the junction once in the relation and
  query the related table directly, no junction hop.
- **RQB `where` / `orderBy` are now objects**, not callbacks:
  `where: { age: { gt: 18 }, OR: [...], RAW: (t) => sql`...` }`,
  `orderBy: { id: "asc" }`. Filtering by a relation and `offset` on nested
  `with` are now supported.
- No `mode: "planetscale"` needed anymore (one strategy for all MySQL).
- `db.query` is v2; `db._query` is the deprecated v1 compat shim.
- Removed exports from `drizzle-orm` / `drizzle-orm/relations`: `relations`,
  `createOne`, `createMany`, `Relations`, `RelationConfig`,
  `createTableRelationsHelpers`, `getOperators`, `extractTablesRelationalConfig`,
  and more — the old types now live under a `V1.` namespace.
- Migration path: `drizzle-kit pull` (v1) can emit a v2 `relations.ts`.

### Casing

`drizzle({ casing: "camelCase" })` is replaced by per-entity builders:
```ts
import { snakeCase, camelCase } from "drizzle-orm/pg-core";
export const users = snakeCase.table("users", { fullName: text() });   // → full_name
// snakeCase.{table,view,materializedView,schema}
```

### Validators folded in

`drizzle-zod` / `drizzle-valibot` / `drizzle-typebox` / `drizzle-arktype`
become `drizzle-orm/zod`, `/valibot`, `/typebox` (typebox lib) or
`/typebox-legacy` (`@sinclair/typebox`), `/arktype`; new
`drizzle-orm/effect-schema`. The standalone packages still work but stop getting
new features.

### Columns

- **`.array()` is no longer chainable** — multi-dim arrays use a string arg:
  `column.array("[][]")`, `column.array("[][][]")` (0.45 chained
  `column.array().array()`).
- **`.generatedAlwaysAs()` only accepts `sql\`...\`` or `() => sql\`...\``**,
  not a raw string.
- `.enableRLS()` on a table is deprecated → `pgTable.withRLS("users", {...})`.

### Utilities

`getTableColumns` → `getColumns` (since `1.0.0-beta.2`).

### Internal type params

`drizzle` DB / session / transaction / migrator classes and `DrizzleConfig`
swap their `TSchema` generic for a `TRelations` generic. Matters only if you
wrote helper types over Drizzle internals.

### New dialects

v1 adds `cockroach-core` / `drizzle-orm/cockroach` and `mssql-core` /
`drizzle-orm/mssql`. These **do not exist on 0.45** — a doc that imports them is
v1.

## Should a project upgrade?

- v1 is still RC as of the pinned refs. For production, 0.45.x is the stable
  target.
- If starting fresh and comfortable with an RC, v1's `defineRelations` + RQB v2
  are genuinely nicer and the docs increasingly assume them.
- Mid-project: the relations rewrite is the bulk of the work; `drizzle-kit
  pull` generates the v2 `relations.ts` for you. The casing, `.array`, and
  validator-import changes are mechanical.
- Re-verify anything version-sensitive against the target release's changelog
  when bumping.
