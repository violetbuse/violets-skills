# Drizzle on Cloudflare — D1 and Durable Object SQLite

`drizzle-orm` 0.45.x. Both are SQLite dialects (`drizzle-orm/sqlite-core` for
the schema). Docs: `/docs/connect-cloudflare-d1`, `/docs/connect-cloudflare-do`.

**Running on celld?** celld executes the *same* Workers / Durable Objects / D1
application on your own machines from your `wrangler.json` — the Drizzle code
below is byte-for-byte identical. Differences: celld needs
`wrangler.jsonc`/`wrangler.json` (not `.toml`) and rejects unknown top-level
config keys; you operate a deployed D1 database with `celld d1 execute` /
`celld d1 migrations apply` instead of `wrangler d1`. See the `celld` skill.

---

## D1

D1 is Cloudflare's managed serverless SQLite. One binding per database, exposed
on `env`.

```ts
// src/schema.ts  — plain sqlite-core
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";
export const users = sqliteTable("users", {
  id: integer().primaryKey({ autoIncrement: true }),
  email: text().notNull().unique(),
  name: text(),
});
```

```ts
// src/index.ts
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

export interface Env { DB: D1Database }

export default {
  async fetch(req: Request, env: Env) {
    const db = drizzle(env.DB, { schema });        // { schema } enables db.query.*
    const rows = await db.select().from(schema.users);
    return Response.json(rows);
  },
};
```

- **Create `db` inside the handler** (per request) — the `env` binding only
  exists there. It's cheap; there's no connection to pool.
- **No interactive transactions.** D1 is auto-commit. Use `db.batch([...])` for
  an atomic, single-round-trip sequence:
  ```ts
  await db.batch([
    db.insert(users).values({ email: "a@b.c" }),
    db.update(users).set({ name: "x" }).where(eq(users.id, 1)),
  ]);
  ```
  A failing statement rolls the whole batch back.
- Sync-style helpers still work: `.all()` `.get()` `.run()` `.values()`.
- `db.$client` → the raw `D1Database`.
- The `cache` option is **not** applied to `db.batch`.

### D1 migrations

`wrangler.json` (celld: `wrangler.jsonc`):
```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-09-26",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [{
    "binding": "DB",
    "database_name": "my-db",
    "database_id": "<id>",
    "migrations_dir": "drizzle"
  }]
}
```

Two ways to apply generated SQL:

1. **wrangler / celld runs the `.sql` files** (recommended). Set
   `migrations_dir` to your Drizzle `out` folder, `drizzle-kit generate`, then:
   - Cloudflare-managed: `wrangler d1 migrations apply my-db --local` / `--remote`
   - celld fleet: `celld d1 migrations apply DB`
   Wrangler/celld track applied migrations in their own table; you don't need
   drizzle's runtime migrator.
2. **Drizzle Kit over the D1 HTTP API** — lets `drizzle-kit migrate` / `push` /
   `pull` / `studio` talk to a *remote* D1 directly:
   ```ts
   // drizzle.config.ts
   export default defineConfig({
     schema: "./src/schema.ts",
     out: "./drizzle",
     dialect: "sqlite",
     driver: "d1-http",
     dbCredentials: {
       accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
       databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
       token: process.env.CLOUDFLARE_D1_TOKEN!,   // API token with D1:Edit
     },
   });
   ```
   Only works against Cloudflare-hosted D1 (it's the Cloudflare API), not a
   local `wrangler dev` DB and not celld.

There is also `migrate()` from `drizzle-orm/d1/migrator` (runs bundled
migrations from inside the Worker) but the wrangler/celld flow is simpler for
D1 — keep the runtime migrator for Durable Objects (below).

---

## Durable Object SQLite storage

A SQLite-backed Durable Object has its own private database at `ctx.storage`.
Drizzle wraps it with `drizzle-orm/durable-sqlite`. **This dialect is
synchronous** — the DB runs in the same isolate as the DO.

```ts
/// <reference types="@cloudflare/workers-types" />
import { DurableObject } from "cloudflare:workers";
import { drizzle, type DrizzleSqliteDODatabase } from "drizzle-orm/durable-sqlite";
import { migrate } from "drizzle-orm/durable-sqlite/migrator";
import migrations from "../drizzle/migrations";      // generated bundle, see below
import { users } from "./schema";

export class Counter extends DurableObject {
  db: DrizzleSqliteDODatabase;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    this.db = drizzle(ctx.storage, { logger: false });
    // run migrations before any request touches the DB
    ctx.blockConcurrencyWhile(async () => {
      await migrate(this.db, migrations);
    });
  }

  async addUser(u: typeof users.$inferInsert) {
    await this.db.insert(users).values(u);
    return this.db.select().from(users);
  }
}

export default {
  async fetch(req: Request, env: Env) {
    const stub = env.COUNTER.get(env.COUNTER.idFromName("main"));
    return Response.json(await stub.addUser({ email: "a@b.c", name: "Ann" }));
  },
};
```

- **`drizzle(ctx.storage, { schema? })`** — pass `ctx.storage`, not
  `ctx.storage.sql`. `{ schema }` still enables `db.query.*`.
- **Instantiate `this.db` once in the constructor**, not per method — the DO
  instance is long-lived.
- **Migrations run in the constructor inside `ctx.blockConcurrencyWhile(...)`**
  so no request sees an unmigrated DB. If you skip that, call `await
  migrate(this.db, migrations)` at the top of every method that reads the DB.
- Every call from the Worker to the stub is a round-trip. **Bundle the work**:
  one DO method that does several queries beats several stub calls. Inside the
  DO, DB access is local and fast.
- **Transactions are synchronous** here — the callback is **not** `async`:
  ```ts
  this.db.transaction((tx) => {
    tx.insert(users).values(a).run();
    tx.insert(users).values(b).run();
  });
  ```
  (The async-callback form used by every other driver does not apply.)
- The DO SQLite database counts against the DO's storage limits; keep schemas
  lean, index what you query.

### Durable Object migrations bundle

The DO migrator takes `{ journal, migrations }`, not a folder path. Drizzle Kit
generates that bundle as `drizzle/migrations.js` **when `driver` is
`durable-sqlite`**:

```ts
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  out: "./drizzle",
  schema: "./src/schema.ts",
  dialect: "sqlite",
  driver: "durable-sqlite",
});
```

```
drizzle-kit generate        # writes NNNN_*.sql + meta/_journal.json + migrations.js
```

`migrations.js` is:
```js
import journal from "./meta/_journal.json";
import m0000 from "./0000_init.sql";
export default { journal, migrations: { m0000 } };
```

```jsonc
// wrangler.jsonc
"migrations": [{ "tag": "v1", "new_sqlite_classes": ["Counter"] }]
```
(`migrations` here is the **DO class** migration list — unrelated to Drizzle.
It's how Cloudflare/celld learn a class is SQLite-backed. Both accept it.)

On **Cloudflare**, for the `.sql` imports in `migrations.js` to resolve at build
time, wrangler needs a text rule:
```jsonc
// wrangler.jsonc — Cloudflare only
"rules": [{ "type": "Text", "globs": ["**/*.sql"], "fallthrough": true }],
```

> **celld does not support this.** As of celld v0.4.1 the `.sql`-as-Text-module
> approach is blocked end to end:
> - `rules` is not an accepted `wrangler.json` key — `celld deploy` (and `celld
>   dev`) abort on any unknown top-level key.
> - celld's bundler runs esbuild with only `--loader:.wasm=copy`; esbuild has no
>   default `.sql` loader, so `import m0000 from "./0000_init.sql"` fails to
>   bundle.
> - `no_bundle: true` + a real `wrangler` build doesn't help: celld uploads only
>   the single entry module, so wrangler's separate `.sql` Text modules are
>   dropped and the import is unresolved at runtime.
>
> **Workaround on celld:** don't rely on the `.sql` imports. After `drizzle-kit
> generate`, run a small codegen step that reads the `NNNN_*.sql` files and
> rewrites `drizzle/migrations.js` (or a sibling `.ts`) with the SQL inlined as
> string literals instead of `import` statements — e.g.
> ```js
> import journal from "./meta/_journal.json";
> const m0000 = `CREATE TABLE ...`;   // inlined from 0000_init.sql
> export default { journal, migrations: { m0000 } };
> ```
> Then it's plain JS, esbuild bundles it normally, and no `rules` key is needed.
> Feed the result to `migrate(this.db, migrations)` as usual. (A `CELLD_ESBUILD`
> wrapper that appends `--loader:.sql=text` also works but leans on an
> undocumented seam.)

- **`push` / `pull` / `studio` don't work against a Durable Object** — there's
  no endpoint. Always `generate` + runtime `migrate()`.
- The DO migrator writes its applied-migration log to `__drizzle_migrations`
  **inside that DO's storage** — every DO instance migrates itself on first
  construction.

---

## Which for what

| Need | Use |
| --- | --- |
| Shared relational DB for the whole Worker | **D1** |
| Per-entity state (one DB per room / user / tenant / agent), colocated with compute, transactional | **Durable Object SQLite** |
| Both | D1 for shared reference data, DO SQLite for hot per-entity state |

On celld the same choice applies, and celld's one-writer-per-DO guarantee makes
the DO-per-tenant pattern especially clean.
