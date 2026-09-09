# Drizzle Kit & migrations

`drizzle-kit` 0.31.x. CLI installed as a dev dependency (`npm i -D drizzle-kit`),
configured by `drizzle.config.ts`. It **never bundles a driver** — it reuses the
driver already in your project for the configured `dialect`. Docs:
`/docs/kit-overview`, `/docs/drizzle-config-file`, `/docs/migrations`.

## `drizzle.config.ts`

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "postgresql",              // postgresql | mysql | sqlite | turso | singlestore | gel
  schema: "./src/db/schema.ts",       // string | string[] — file, folder, or glob
  out: "./drizzle",                   // migration files + snapshots + pulled schema
  dbCredentials: { url: process.env.DATABASE_URL! },

  // driver exceptions only — Kit auto-detects otherwise
  // driver: "pglite" | "aws-data-api" | "d1-http" | "durable-sqlite" | "expo",

  // keep Kit off things it doesn't own
  schemaFilter: ["public"],           // pg schemas to manage (default ["public"])
  tablesFilter: ["!_*"],              // glob(s); default "*"
  extensionsFilters: ["postgis"],     // ignore tables created by these extensions
  entities: { roles: { provider: "supabase" } },  // don't manage provider roles

  migrations: {
    table: "__drizzle_migrations",    // applied-migrations log
    schema: "drizzle",                // pg only
    prefix: "timestamp",              // migration file name prefix
  },
  breakpoints: true,                  // insert --> statement-breakpoint (needed for mysql/sqlite)
  strict: true,                       // push: confirm before executing
  verbose: true,                      // push: print SQL
});
```

- **TS/JS config only** — no `wrangler.toml`, no JSON. Multiple stages ⇒
  multiple config files + `--config=drizzle-prod.config.ts`.
- `schema` must resolve to **statically analysable** exported `const`s. Kit
  imports the file; anything it can't see statically (dynamically built tables,
  non-exported consts) is missing from the diff with no warning.
- `dbCredentials` accepts `url` **or** `{ host, port, user, password, database,
  ssl }`. `ssl` (pg): `boolean | "require" | "allow" | "prefer" | "verify-full"
  | tls.SecureContextOptions`.
- Driver-specific credential shapes: `aws-data-api`
  (`{ database, resourceArn, secretArn }`), `d1-http`
  (`{ accountId, databaseId, token }` — Cloudflare-hosted D1 only), `pglite`
  (`{ url: "./db/" }` or `":memory:"`), `turso`
  (`{ url: "libsql://...", authToken }`).
- `driver: "durable-sqlite"` and `driver: "expo"` take **no `dbCredentials`** —
  they only change `generate` to also emit a bundled `drizzle/migrations.js`
  (`{ journal, migrations }`) for the runtime migrator. `push`/`pull`/`studio`/
  `migrate` don't apply (there's no endpoint to reach a Durable Object or an
  on-device DB). See `reference/cloudflare-d1-do.md`.

## Commands

| Command | Does |
| --- | --- |
| `drizzle-kit generate` | Diff schema vs last snapshot → write `NNNN_name.sql` + `snapshot.json` under `out/`. **Does not touch the DB.** |
| `drizzle-kit migrate` | Apply pending `.sql` files, recording each in `__drizzle_migrations`. |
| `drizzle-kit push` | Diff schema vs **live DB** → apply directly. No SQL files. |
| `drizzle-kit pull` | Introspect live DB → generate `schema.ts` + `relations.ts` + snapshot under `out/`. |
| `drizzle-kit check` | Detect collisions/race conditions across generated migrations (team branches). |
| `drizzle-kit up` | Upgrade old snapshot JSON to the current internal format. |
| `drizzle-kit export` | Print the schema's full DDL to stdout (feed Atlas etc.). |
| `drizzle-kit studio` | Local DB browser at `local.drizzle.studio` (dev only, not for a VPS). |

Flags: `generate --name=init` / `--custom` · `push --strict --verbose --force` ·
`--config=path`.

## The two workflows — pick ONE

### Generate + apply (production)

```
edit schema.ts  →  drizzle-kit generate  →  review + commit drizzle/NNNN_*.sql  →  drizzle-kit migrate
```
- Migrations are ordered, reviewable, replayable. `migrate` is idempotent
  (skips already-logged files).
- Apply at deploy time (`drizzle-kit migrate` in CI) **or** at app startup:
  ```ts
  import { migrate } from "drizzle-orm/node-postgres/migrator";   // path matches your driver
  await migrate(db, { migrationsFolder: "./drizzle" });
  ```
  Startup migration suits monoliths / zero-downtime deploys; for serverless run
  it once in a deploy step, not per cold start.
- `generate` **prompts on ambiguous diffs** — "is `full_name` a rename of
  `name`, or a drop + add?" A wrong answer drops the column's data. Review the
  generated SQL before committing.
- `--custom --name=seed` makes an **empty** `.sql` file for data seeding or DDL
  Kit can't emit (JS/TS custom migrations are not supported yet).
- `breakpoints: true` inserts `--> statement-breakpoint` so MySQL/SQLite (which
  can't run multiple DDL statements per transaction) apply one at a time.

### Push (dev / prototyping)

```
edit schema.ts  →  drizzle-kit push
```
- Great for local iteration, throwaway preview-branch databases, and DBs that
  are themselves branchable (Neon, PlanetScale).
- **Do not point `push` at a database whose history matters.** It has no
  migration files to review and:
  - **silently skips** changes to an existing index's `.on()` expressions,
    `.using()`, `.where()`, or `.op()` operator classes (workaround: comment
    the index out → push → re-add changed → push);
  - **silently skips** generated-column expression changes (pg/mysql) — you
    must drop & recreate the column;
  - **drops columns/tables** on a destructive diff. `--strict` prompts;
    `--force` auto-accepts the data loss.
- `push` and `pull` only touch `public` (pg) unless `schemaFilter` says
  otherwise, and respect `tablesFilter` / `extensionsFilter`.

### Database-first (`pull`)

For a DB managed elsewhere: `drizzle-kit pull` writes `out/schema.ts` (+
`relations.ts`, `snapshot.json`). Configure key casing with
`introspect: { casing: "camel" | "preserve" }`. Then import that schema in app
code; re-pull when the DB changes.

## Applied-migrations log

`drizzle-kit migrate` and runtime `migrate()` record each file in
`__drizzle_migrations` (pg: in a `drizzle` schema by default). `push` does
**not** use this table. Rename it with `migrations.table` / `migrations.schema`.

## Teams / branches

- Commit `out/` (both `.sql` and `_meta`/`snapshot.json`).
- Two branches that both `generate` create files with **different timestamps
  but possibly conflicting DDL** → run `drizzle-kit check` in CI; regenerate the
  loser after merge.
- Never edit an already-applied migration file — add a new one.

## Provider system objects

`postgis`, Supabase (`auth`, `storage`, roles), Neon (`neon_identity`, roles),
RLS roles you didn't declare — all trip up `push`/`pull`/`generate`. Fence them
off:

```ts
extensionsFilters: ["postgis"],
schemaFilter: ["public"],
entities: { roles: { provider: "supabase", exclude: ["extra_role"] } },  // or provider: "neon", or roles: false
```
Mark individually-referenced external objects `.existing()` in the schema
(`pgRole("authenticated").existing()`, `pgView(...).existing()`,
`pgTable` via `drizzle-orm/supabase`'s `authUsers`).
