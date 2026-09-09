---
name: drizzle-orm
description: >-
  Working with Drizzle ORM and Drizzle Kit in a TypeScript project — declaring
  a schema (`pgTable` / `mysqlTable` / `sqliteTable`, column types and their
  `mode` options, enums, indexes, foreign keys, relations), querying with the
  SQL-like builder (`db.select`/`insert`/`update`/`delete`, joins, `sql`
  operator, transactions, prepared statements, `$dynamic`) or the relational
  query builder (`db.query.*.findMany` with `with`), running migrations
  (`drizzle-kit generate` / `migrate` / `push` / `pull`, `drizzle.config.ts`),
  connecting a driver (node-postgres, postgres.js, the Neon serverless driver in
  HTTP and WebSocket modes — including for PlanetScale Postgres — planetscale
  MySQL, libsql, bun, pglite, Cloudflare D1, Durable Object SQLite /
  `durable-sqlite`, also when running on celld), serverless/edge performance,
  read replicas, caching,
  `drizzle-zod` / `drizzle-seed`, and debugging Drizzle's type-inference and
  date/number mapping surprises. Also covers migrating toward Drizzle v1.
---

# Drizzle ORM

Drizzle is a **thin, typed, dialect-specific SQL query builder** for TypeScript
(MIT, by Drizzle Team) plus **Drizzle Kit**, a companion CLI for migrations.
It has zero runtime dependencies, emits exactly the SQL you'd expect, and does
**no** hidden magic — no lazy loading, no identity map, no query caching unless
you opt in. "If you know SQL, you know Drizzle."

- Upstream: <https://github.com/drizzle-team/drizzle-orm> ·
  <https://github.com/drizzle-team/drizzle-orm-docs> · docs <https://orm.drizzle.team/docs>
- **This skill targets `drizzle-orm` 0.45.x and `drizzle-kit` 0.31.x** — what
  `npm install drizzle-orm drizzle-kit` gives you today. A `drizzle-orm@1.0`
  line exists but is still a release candidate; its APIs (`defineRelations`,
  relational-queries v2, casing builders, validators folded into `drizzle-orm`)
  are **different** and are covered separately in
  `reference/v1-migration.md`. **The public docs site is mid-transition and
  shows a mix of both** — when a doc page shows `defineRelations`,
  `snakeCase.table`, `db._query`, or tells you to `npm i drizzle-orm@rc`, it is
  describing v1, not the stable line this skill covers.
- **This skill is self-contained** — `SKILL.md` plus `reference/schema.md`,
  `reference/queries.md`, `reference/kit-migrations.md`,
  `reference/drivers-and-perf.md`, `reference/cloudflare-d1-do.md`,
  `reference/v1-migration.md`. For anything
  deeper: `npx drizzle-kit --help`, the docs site above, or the upstream repos
  (Claude can `WebFetch` GitHub). If you're working inside the `violets-skills`
  repo, checkouts are vendored at `vendor/drizzle-orm` (tag `0.45.2`) and
  `vendor/drizzle-orm-docs` (commit `11695b7`, 2025-11-19 — the last pre-v1
  docs snapshot); those paths do **not** exist where this plugin is installed.

## Mental model

| Piece | What it is |
| --- | --- |
| **`drizzle-orm`** | The library you import in app code. Dialect-specific: `drizzle-orm/pg-core`, `drizzle-orm/mysql-core`, `drizzle-orm/sqlite-core`, `drizzle-orm/singlestore-core`, `drizzle-orm/gel-core`. Plus a driver entry per client: `drizzle-orm/node-postgres`, `drizzle-orm/postgres-js`, `drizzle-orm/neon-http`, `drizzle-orm/libsql`, `drizzle-orm/d1`, … |
| **schema** | Plain `.ts` files exporting `pgTable(...)` etc. **The single source of truth.** Drizzle ORM reads it for types; Drizzle Kit reads it to diff migrations. Everything Kit should see must be `export`ed. |
| **`db`** | The client, from `drizzle(...)`. Exposes the SQL-like builder (`db.select()`, `db.insert()`, …) always, and the relational builder (`db.query.<table>`) only if you pass `{ schema }`. |
| **`drizzle-kit`** | Dev-dependency CLI. Reads `drizzle.config.ts`. Never bundles a driver — it reuses whichever driver is already in your project for the configured `dialect`. |
| **the two query APIs** | **SQL-like** (`db.select().from(users).leftJoin(...)`) mirrors SQL 1:1, returns flat/nested-by-join rows, you map relations yourself. **Relational** (`db.query.users.findMany({ with: { posts: true } })`) returns a nested object tree in **one** SQL statement via lateral-joined subqueries. Same `db`, pick per query. |

**Drizzle does exactly what you write.** A `.where()` you build conditionally
is the whole filter; there is no implicit `deleted_at IS NULL`, no automatic
`updatedAt`. `db.select()` with no `.where()` selects every row;
`db.delete(users)` with no `.where()` truncates the table.

## Is Drizzle a fit?

Good fit: you want SQL semantics with TypeScript types, you deploy to
serverless/edge (zero deps, one round-trip queries, HTTP drivers), you want the
schema in version control, you're comfortable reading SQL. Drizzle supports
PostgreSQL, MySQL, SQLite, SingleStore and Gel, each through the standard
community driver.

Weaker fit: you want an Active-Record / entity-graph ORM that tracks dirty
state and cascades saves (Drizzle has none of that — it's a query builder);
you need a database Drizzle has no dialect for; you want migrations that
auto-handle every destructive rename without confirmation.

## Declare the schema

```ts
// src/db/schema.ts
import { pgTable, integer, varchar, timestamp, index } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  email: varchar({ length: 256 }).notNull().unique(),
  name: varchar({ length: 256 }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index("users_name_idx").on(t.name),
]);

export const posts = pgTable("posts", {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  authorId: integer("author_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  title: varchar({ length: 256 }).notNull(),
});
```

- **A bare column key becomes the DB column name** (`createdAt` → `"createdAt"`).
  Pass a string (`varchar("first_name")`) or set `casing: "snake_case"` on
  `drizzle(...)` to map. **`casing` only affects columns whose name you did
  *not* pass explicitly**, and it changes generated SQL — set it before you
  generate any migration and never flip it afterward.
- The **third `pgTable` argument** is a callback returning an **array** of
  indexes / composite constraints / FKs / checks / RLS policies. (Older code
  returns an object; the array form is current.)
- `.references(() => other.id)` needs a **thunk** because of declaration order;
  self-references additionally need an explicit `: AnyPgColumn` return type or
  the standalone `foreignKey({...})` helper.
- Full detail — every column type, the critical **`mode` options**
  (`bigint`/`numeric`/`timestamp`/`date` in pg; `integer({ mode: "boolean" | "timestamp" })`
  in SQLite), `.$type<T>()`, `.$defaultFn()` / `.$onUpdateFn()`, enums,
  identity vs `serial`, indexes, generated columns, sequences, views, schemas,
  RLS, `customType`, multi-project `pgTableCreator` — is in **`reference/schema.md`**.

## Connect a driver

```ts
// src/db/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema";

// simplest — Drizzle creates & owns a pg Pool
export const db = drizzle(process.env.DATABASE_URL!, { schema });

// or pass your own client to control the pool
import { Pool } from "pg";
const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });
export const db = drizzle({ client: pool, schema });
```

- **`{ schema }` is what turns on `db.query.*`** (the relational builder) and
  its nested-result types. Without it you only get the SQL-like builder.
- The `drizzle` **import path is the driver**: `node-postgres` / `postgres-js`
  for a TCP Postgres; `neon-http` (one-shot HTTPS) / `neon-serverless`
  (WebSocket, interactive transactions) for **Neon *or* PlanetScale Postgres**
  from an edge runtime; `planetscale` for PlanetScale **MySQL**; `libsql` for
  Turso/libSQL; `d1` / `durable-sqlite` for Cloudflare (and celld);
  `bun-sql` / `bun-sqlite` for Bun; `pglite` for embedded Postgres.
- `db.$client` gets you the underlying driver instance.
- Per-driver connection snippets, the **Neon serverless driver** (HTTP vs
  WebSocket, and the `neonConfig` settings to point it at **PlanetScale
  Postgres**), **serverless/edge patterns** (hoist `db` above the handler,
  reuse prepared statements, pooler "transaction mode" ⇒ disable prepared
  statements), `withReplicas()`, the `cache` option, and the validator/seed
  companion packages are in **`reference/drivers-and-perf.md`**.

### Cloudflare D1 & Durable Object SQLite (also celld)

Both are SQLite dialects — schema in `drizzle-orm/sqlite-core`.

```ts
// D1 — drizzle-orm/d1, one db per request from the binding
const db = drizzle(env.DB, { schema });        // no transactions → db.batch([...])

// Durable Object storage — drizzle-orm/durable-sqlite, SYNCHRONOUS
this.db = drizzle(ctx.storage, { schema });    // in the DO constructor, once
ctx.blockConcurrencyWhile(async () => migrate(this.db, migrations));  // migrations bundle
// tx callback is sync here:  this.db.transaction((tx) => { tx.insert(...).run() })
```

- **D1 migrations**: `drizzle-kit generate` → apply the `.sql` with
  `wrangler d1 migrations apply` / `celld d1 migrations apply DB`, **or** set
  `driver: "d1-http"` (Cloudflare-hosted D1 only) to let `drizzle-kit migrate`/
  `push`/`studio` reach a remote D1.
- **DO migrations**: `drizzle-kit generate` with `driver: "durable-sqlite"`
  emits `drizzle/migrations.js` (a `{ journal, migrations }` bundle);
  `import migrations from "../drizzle/migrations"` + runtime
  `migrate(this.db, migrations)`. No `push`/`pull`/`studio` for a DO.
- **celld** runs the identical Workers/DO/D1 app on your own machines from
  `wrangler.jsonc` — Drizzle code is unchanged; operate deployed D1 with
  `celld d1 ...`. Full detail (batch semantics, `blockConcurrencyWhile`,
  round-trip advice, the wrangler `Text` rule for `.sql` imports) is in
  **`reference/cloudflare-d1-do.md`**.

## Query — SQL-like

```ts
import { and, eq, desc, sql } from "drizzle-orm";

const rows = await db.select().from(users).where(eq(users.id, 1));           // typed rows
await db.select({ id: users.id, name: users.name }).from(users);             // partial select
await db.insert(users).values({ email: "a@b.c" }).returning();               // pg/sqlite RETURNING
await db.update(users).set({ name: "Dan" }).where(eq(users.id, 1));
await db.delete(users).where(eq(users.id, 1));

// join — result is grouped by table key, and the joined side is nullable
const withPosts = await db.select()
  .from(users)
  .leftJoin(posts, eq(posts.authorId, users.id));
// withPosts: { users: {...}, posts: {...} | null }[]
```

- **Each builder method is callable once** (one `.where`, one `.limit`). To
  build a query up across functions, call `.$dynamic()` first. Multiple
  conditions go inside one `and(...)` / `or(...)`, not multiple `.where()`.
- **`.where(undefined)` is ignored** — that's the idiom for optional filters:
  `.where(term ? ilike(t.name, term) : undefined)` or `and(...filters)` where
  `filters: SQL[]` may be empty.
- **`leftJoin` makes the joined table `| null` in the type.** For `sql`
  expressions selected off a left-joined table you must annotate
  `sql<string | null>` yourself — Drizzle can't infer it.
- Operators (`eq ne gt gte lt lte inArray between like ilike isNull isNotNull
  and or not exists arrayContains …`), the **`sql` template** (parameterised;
  `sql.raw()` is **not** — never interpolate user input into `sql.raw`),
  `sql<T>` (type hint only, no runtime cast — use `.mapWith(Number)` for that),
  aggregates + the `db.$count()` helper, `count()` returning `bigint`/string,
  CTEs (`db.$with`), set operations, subqueries, `$dynamic` pagination,
  transactions (+ isolation levels, `tx.rollback()`, savepoints), prepared
  statements with `sql.placeholder(...)`, and `db.batch([...])` — all in
  **`reference/queries.md`**.

## Query — relational

```ts
// relations live in the schema file (or a sibling), using relations()
import { relations } from "drizzle-orm";
export const usersRelations = relations(users, ({ many }) => ({ posts: many(posts) }));
export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));

// then, with drizzle(..., { schema }):
const tree = await db.query.users.findMany({
  where: (u, { eq }) => eq(u.id, 1),
  columns: { email: false },              // include/exclude columns
  with: { posts: { limit: 5, orderBy: (p, { desc }) => [desc(p.id)] } },
  extras: { lower: (u, { sql }) => sql<string>`lower(${u.name})`.as("lower") },
});
```

- `relations()` is an **application-level** join hint only — it creates **no**
  foreign key and does not touch migrations. FKs and `relations()` are
  independent; define whichever you need.
- `findFirst()` adds `LIMIT 1`. `findMany({ with: {...} })` still produces
  **one** SQL statement (lateral-joined subqueries).
- In `0.45.x` the `where`/`orderBy`/`extras` callbacks take
  `(table, operators)` — reference columns via the callback param, and `extras`
  fields **must** be `.as("alias")`d.
- **PlanetScale + `mysql2`**: pass `mode: "planetscale"` to `drizzle(...)` or
  the lateral joins the relational builder emits will fail. (Native
  `drizzle-orm/planetscale` handles this itself.)
- `offset` on nested `with` relations is **not** supported on the 0.45 line
  (top-level only); the `cache` option does not apply to `db.query.*` yet.
- Full relational-builder reference (one-to-one / one-to-many / many-to-many
  junction tables, `relationName` disambiguation, prepared relational queries,
  indexing strategy for `with`) is in **`reference/queries.md`**.

## Migrations with Drizzle Kit

```ts
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  dialect: "postgresql",              // postgresql | mysql | sqlite | turso | singlestore | gel
  schema: "./src/db/schema.ts",       // file OR glob OR folder
  out: "./drizzle",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

Two workflows — **pick one and stick to it**:

| | Command | Use when |
| --- | --- | --- |
| **Generate + apply** | `drizzle-kit generate` → commit the `.sql` → `drizzle-kit migrate` (or `migrate()` at app startup) | Production. Reviewable, ordered, versioned migration files under `out/`. |
| **Push** | `drizzle-kit push` | Local dev / rapid prototyping / preview branches. Diffs schema → DB and applies directly, **no SQL files**. |

- **`push` has real limitations** — it silently skips some changes (index
  expression / `.where()` / operator-class edits; generated-column expression
  changes) and will drop columns/tables on destructive diffs. Don't run it
  against a database whose history you care about. `--strict` prompts,
  `--verbose` prints SQL, `--force` auto-accepts data loss.
- `generate` respects **renames** (it prompts "column renamed or
  dropped+added?") — answer carefully; a wrong answer means data loss.
- `drizzle-kit pull` introspects an existing DB into a `schema.ts` +
  `relations.ts` (database-first).
- `drizzle-kit` picks the driver from your project automatically; exceptions
  (`aws-data-api`, `pglite`, `d1-http`) need an explicit `driver:` in the
  config.
- The applied-migrations log lives in `__drizzle_migrations` (in the `drizzle`
  schema for Postgres). `extensionsFilters`, `schemaFilter`, `tablesFilter`,
  `entities.roles` keep Kit from fighting `postgis` / Supabase / Neon
  system objects.
- The six official workflows, `drizzle-kit check` / `up` / `export`, custom
  (`--custom`) migrations, runtime `migrate()` per driver, breakpoints, and
  team/branch conflict handling are in **`reference/kit-migrations.md`**.

## Common pitfalls

- **You're reading v1 docs by accident.** `defineRelations`, `db._query`,
  `snakeCase.table`, `column.array("[][]")` (string form), `getColumns`,
  `drizzle-orm@rc` are all v1. On 0.45.x use `relations()`, `db.query`, `casing`
  on `drizzle()`, `column.array().array()` (chained), `getTableColumns`. See
  `reference/v1-migration.md`.
- **`numeric` / `decimal` and 64-bit `bigint` come back as `string`** by
  default (precision safety). Opt into `{ mode: "number" }` / `{ mode: "bigint" }`
  only when the range is safe.
- **`timestamp` mapping.** pg `timestamp` (no tz) with `mode: "date"` hands JS
  `Date` back and forth but the DB ignores offset; `timestamp({ withTimezone: true })`
  stores UTC and returns it converted to the server's `TimeZone`. `mode: "string"`
  does zero mapping. **SQLite has no date type** — use
  `integer({ mode: "timestamp" })` (seconds → `Date`) or `"timestamp_ms"`.
- **SQLite has no boolean** — `integer({ mode: "boolean" })`. SQLite `text`
  JSON: prefer `text({ mode: "json" })` over `blob`.
- **`.$type<T>()`, `{ enum: [...] }`, `sql<T>` are compile-time only.** No
  runtime validation, no cast. Validate at the edge with `drizzle-zod`.
- **`count()` returns `bigint` (pg) / string (mysql)** → wrap with
  `.mapWith(Number)` or use `db.$count()`.
- **`update().set({ x: undefined })` is a no-op**, not `SET x = NULL`. Pass
  `null` explicitly to null a column.
- **Forgot to `export` a table / relation** ⇒ Drizzle Kit doesn't see it ⇒
  migration silently omits it.
- **`db.delete(t)` / `db.update(t)` / `db.select()` with no `.where()`** hits
  every row. `drizzle-kit push --force` skips the data-loss prompt.
- **`sql.raw()` and `sql.identifier()` do not parameterise.** Never feed user
  input to them. (`sql.identifier()` / `sql.as()` had a SQL-injection fix in
  `drizzle-orm@0.45.2` — be on ≥ 0.45.2.)
- **Serverless:** create `db` at module scope, not per request; behind a
  transaction-mode pooler (PgBouncer, Supabase `:6543`, Prisma Accelerate)
  disable prepared statements (`postgres(url, { prepare: false })` /
  `Pool` without named statements).
- **`wrangler.toml` isn't it** — `drizzle.config.ts` is TS/JS only, and the
  schema must be plain exported consts (no dynamic table creation Kit can't
  statically read).
