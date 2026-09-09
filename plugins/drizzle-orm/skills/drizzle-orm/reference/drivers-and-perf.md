# Drivers, serverless performance, ecosystem

`drizzle-orm` 0.45.x. Docs: `/docs/connect-overview`, `/docs/perf-queries`,
`/docs/perf-serverless`, `/docs/read-replicas`, `/docs/cache`.

## Choosing the driver entrypoint

The `drizzle` **import path is the driver**. Install the driver package
yourself; Drizzle wraps it.

| Import | Client package | For |
| --- | --- | --- |
| `drizzle-orm/node-postgres` | `pg` | Postgres over TCP, long-lived processes, connection pool |
| `drizzle-orm/postgres-js` | `postgres` | Postgres over TCP, lighter; common on serverless-with-TCP |
| `drizzle-orm/neon-http` | `@neondatabase/serverless` | Neon **or PlanetScale Postgres**, one-shot HTTP queries (edge) — **`transaction()` throws; use `db.batch`** |
| `drizzle-orm/neon-serverless` | `@neondatabase/serverless` | Neon **or PlanetScale Postgres** over WebSocket — full transactions, `pg` drop-in |
| `drizzle-orm/vercel-postgres` | `@vercel/postgres` | Vercel Postgres |
| `drizzle-orm/planetscale` | `@planetscale/database` | PlanetScale **MySQL** over HTTP (not the Postgres product — see Neon driver row) |
| `drizzle-orm/mysql2` | `mysql2` | MySQL over TCP |
| `drizzle-orm/libsql` | `@libsql/client` | Turso / libSQL / local SQLite file |
| `drizzle-orm/better-sqlite3` | `better-sqlite3` | Node embedded SQLite (sync) |
| `drizzle-orm/bun-sqlite` / `drizzle-orm/bun-sql` | Bun built-ins | Bun SQLite / Bun Postgres |
| `drizzle-orm/d1` | Cloudflare / celld binding | Cloudflare D1 — see `reference/cloudflare-d1-do.md` |
| `drizzle-orm/durable-sqlite` | `ctx.storage` | Durable Object SQLite (Cloudflare / celld) — **synchronous** — see `reference/cloudflare-d1-do.md` |
| `drizzle-orm/pglite` | `@electric-sql/pglite` | Embedded WASM Postgres (tests, local) |
| `drizzle-orm/expo-sqlite` / `op-sqlite` | Expo / OP | On-device mobile SQLite — **embedded migrations only**, no `push`/`pull` |
| `drizzle-orm/aws-data-api/pg` | AWS SDK | Aurora Serverless Data API |
| `drizzle-orm/{pg,mysql,sqlite}-proxy` | your HTTP handler | Custom transport |

## Initialising `db`

```ts
// 1. connection string — Drizzle creates and owns the client/pool
export const db = drizzle(process.env.DATABASE_URL!, { schema });

// 2. options object
export const db = drizzle({ connection: { connectionString: url, ssl: true }, schema, logger: true, casing: "snake_case" });

// 3. bring your own client (control pooling, reuse elsewhere)
import { Pool } from "pg";
const pool = new Pool({ connectionString: url, max: 10 });
export const db = drizzle({ client: pool, schema });
```

`drizzle()` options: `schema` (enables `db.query.*`), `logger`
(`true` | a `Logger`), `casing` (`"snake_case"` | `"camelCase"`), `cache`
(see below), `mode` (`mysql2` only: `"default"` | `"planetscale"`).
`db.$client` → underlying driver.

### Logging

`{ logger: true }` prints every query. Custom:
```ts
import { DefaultLogger } from "drizzle-orm/logger";
const logger = new DefaultLogger({ writer: { write: (m) => myLog(m) } });
// or implement Logger { logQuery(query, params) }
```

## Neon serverless driver — HTTP and WebSocket (also PlanetScale Postgres)

The `@neondatabase/serverless` package is a `pg`-compatible client that reaches
Postgres over **HTTP (fetch)** or **WebSocket** instead of raw TCP — the way to
run Postgres from an edge runtime (Cloudflare Workers, Vercel Edge, Deno Deploy)
that has no TCP sockets. Drizzle wraps it with **two** entrypoints:

| | `drizzle-orm/neon-http` | `drizzle-orm/neon-serverless` |
| --- | --- | --- |
| Transport | one HTTPS request per query (`neon()` fn) | WebSocket, `Pool` / `Client` |
| Latency | lowest for a single query | + WS handshake on a cold pool |
| `transaction()` | **throws** — none | full interactive transactions |
| `db.batch([...])` | yes (atomic, one round-trip) | — (use `transaction()`) |
| `pg` drop-in | no | yes |
| Use for | most edge reads/writes; one-shot queries | multi-statement logic that must be interactive |

```ts
// HTTP
import { drizzle } from "drizzle-orm/neon-http";
export const db = drizzle(process.env.DATABASE_URL!, { schema });
//        BYO:  import { neon } from "@neondatabase/serverless";
//              drizzle({ client: neon(url), schema })

// WebSocket
import { drizzle } from "drizzle-orm/neon-serverless";
export const db = drizzle(process.env.DATABASE_URL!, { schema });
//        Node.js (no global WebSocket) needs a polyfill:
import ws from "ws";                       // + `bufferutil`
export const db = drizzle({ connection: process.env.DATABASE_URL!, ws, schema });
//        or set it globally once: neonConfig.webSocketConstructor = ws;
//        BYO:  import { Pool } from "@neondatabase/serverless";
//              drizzle({ client: new Pool({ connectionString: url }), schema })
```

- **WebSocket connections can't outlive one request** in a serverless/edge
  handler — with the BYO `Pool` form, create and `pool.end()` (or
  `ctx.waitUntil(pool.end())`) inside the handler. The connection-string
  shorthand handles a short-lived pool for you.
- `neon-http`'s underlying `neon()` also has its own non-interactive
  `sql.transaction([...])`, but from Drizzle just use `db.batch`.
- Driver v1.0+ needs Node 19+ (for `fetch`/`WebSocket`).

### PlanetScale Postgres

PlanetScale's **Postgres** product is wire-compatible with the Neon serverless
driver, so it uses these same Drizzle entrypoints (`neon-http` /
`neon-serverless`) — **not** `drizzle-orm/planetscale`, which is for PlanetScale
**MySQL**. Point `DATABASE_URL` at the PlanetScale Postgres connection string
(`postgresql://user:pscale_pw_…@XXXX.pg.psdb.cloud:5432/db`) and set the
required `neonConfig` **before** calling `drizzle()`:

```ts
import { neonConfig } from "@neondatabase/serverless";

// HTTP mode — required:
neonConfig.fetchEndpoint = (host) => `https://${host}/sql`;

// WebSocket mode — required (PlanetScale enforces SCRAM-SHA-256):
neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (host, port) => `${host}/v2?address=${host}:${port}`;
neonConfig.webSocketConstructor = ws;      // Node.js only

// then, unchanged:
import { drizzle } from "drizzle-orm/neon-http";        // or neon-serverless
export const db = drizzle(process.env.DATABASE_URL!, { schema });
```

Same HTTP-vs-WebSocket trade-off as Neon: HTTP for single/batch queries,
WebSocket for interactive transactions and session features. Ref:
<https://planetscale.com/docs/postgres/connecting/neon-serverless-driver>. For
the driver setup in depth (full `neonConfig`, per-framework snippets, the `pg` +
PSBouncer `:6432` alternative, migrations), see the
**`planetscale-postgres-serverless`** skill.

## Serverless & edge

- **Create `db` at module scope, never per request.** A warm Lambda / Worker
  reuses the module, so it reuses the connection and any prepared statements.
- **Prepared statements** are the main perf lever on warm invocations — hoist
  them to module scope too:
  ```ts
  const getUser = db.select().from(users).where(eq(users.id, sql.placeholder("id"))).prepare("get_user");
  export const handler = async (e) => getUser.execute({ id: e.id });
  ```
  Edge functions that tear down immediately gain little from this; TCP-pooled
  serverless (Lambda, ~15 min lifetime) gains a lot.
- **Transaction-mode poolers** (PgBouncer, Supabase port `6543`, Prisma
  Accelerate, RDS Proxy in some modes) **cannot use named prepared
  statements**. Disable them:
  - `postgres-js`: `drizzle(postgres(url, { prepare: false }))`
  - `node-postgres`: don't use `.prepare(...)`; keep pool `max` low.
- **HTTP drivers**: one request per query, no pooling to worry about.
  `neon-http`'s `transaction()` throws — use `db.batch([...])` for atomic
  multi-statement. `drizzle-orm/planetscale` and `neon-serverless` do support
  `transaction()`.
- **Cloudflare D1 / Durable Objects (and celld)**: create `db` in the handler
  (D1) or DO constructor; D1 has no transactions (`db.batch`), DO SQLite is
  synchronous with sync transaction callbacks. Full setup + migration flow in
  **`reference/cloudflare-d1-do.md`**.
- Keep `pool.max` small (edge/serverless concurrency × connections can exhaust
  the DB fast).

## Read replicas

```ts
import { withReplicas } from "drizzle-orm/pg-core";     // or mysql-core / sqlite-core

const db = withReplicas(primaryDb, [read1, read2]);
await db.select().from(users);          // → a random replica
await db.insert(users).values(...);     // → primary
await db.$primary.select().from(users); // force primary for a read
// custom picker: withReplicas(primary, [r1, r2], (replicas) => weightedPick(replicas))
```
All writes and transactions go to primary; `SELECT` goes to a replica unless you
use `$primary`.

## Caching (opt-in only)

Drizzle caches nothing by default. Provide a `cache` to `drizzle()`:

```ts
import { upstashCache } from "drizzle-orm/cache/upstash";
const db = drizzle(url, { cache: upstashCache({ global: false, config: { ex: 60 } }) });
```

- **`global: false`** (default): only queries with `.$withCache()` read cache.
  **`global: true`**: every `SELECT` reads cache; opt out per query with
  `.$withCache(false)`.
- `.$withCache({ config: { ex } , tag: "key", autoInvalidate: false })`.
- **Auto-invalidation is on by default** — any `insert`/`update`/`delete` on a
  table drops cached queries that touched it. `autoInvalidate: false` ⇒ eventual
  consistency until TTL.
- Manual: `db.$cache.invalidate({ tables: [users] })` / `{ tags: "key" }`.
- Custom backend: `class extends Cache { strategy(); get(); put(); onMutate() }`.
- **Not covered**: raw `db.execute`, `db.batch`, transactions, relational
  queries (`db.query.*`), `better-sqlite3` / Durable Objects / Expo, AWS Data
  API, views.

## Validation — `drizzle-zod` (also `drizzle-valibot`, `drizzle-typebox`, `drizzle-arktype`)

Generate a validator from a table so the runtime shape stays in lockstep with
the schema. (On 0.45 these are **separate packages**; in v1 they move to
`drizzle-orm/zod` etc.)

```ts
import { createInsertSchema, createSelectSchema, createUpdateSchema } from "drizzle-zod";

const insertUser = createInsertSchema(users, {
  email: (s) => s.email(),                 // refine a field
  age: (s) => s.min(0),
});
const body = insertUser.parse(req.body);   // validate at the API edge
await db.insert(users).values(body);
```

- `createInsertSchema`: required = `notNull` && no default; drops generated cols.
- `createSelectSchema`: full row shape (also works on views, enums).
- `createUpdateSchema`: everything optional; drops generated cols.
- `createSchemaFactory({ zodInstance, coerce: { date: true } })` for an extended
  Zod (`@hono/zod-openapi`) or coercion.
- Needs `drizzle-orm ≥ 0.36`, Zod ≥ 3.25 for `drizzle-zod ≥ 0.6`.
- Data-type → schema mapping (int ranges, `numeric`→string, `json` union,
  `uuid`→`z.string().uuid()`, points→tuples) is in `/docs/zod`.

## Seeding — `drizzle-seed`

Deterministic fake data from a seeded PRNG. Needs `drizzle-orm ≥ 0.36.4`.

```ts
import { seed, reset } from "drizzle-seed";
import * as schema from "./schema";

await reset(db, schema);                          // TRUNCATE ... CASCADE (pg) / DELETE (sqlite)
await seed(db, schema, { count: 100, seed: 42 }).refine((f) => ({
  users: {
    count: 20,
    columns: { name: f.fullName(), email: f.email() },
    with: { posts: 10 },                          // 10 posts per user (one-to-many only)
  },
  posts: {
    columns: {
      description: f.valuesFromArray({ values: ["...", "..."] }),
      rank: f.weightedRandom([
        { weight: 0.7, value: f.int({ minValue: 1, maxValue: 10 }) },
        { weight: 0.3, value: f.int({ minValue: 11, maxValue: 100 }) },
      ]),
    },
  },
}));
```

- `with` only works parent → child (`users` with `posts`, not the reverse), and
  can't infer references across circular FKs — you pick the direction.
- Generators: `f.firstName lastName fullName email phoneNumber companyName
  jobTitle streetAddress city state country postcode int number date
  loremIpsum valuesFromArray default weightedRandom uuid` … (`/docs/seed-functions`).
- The `pgTable` third argument isn't type-checked by `drizzle-seed` (works at
  runtime).

## Other bits

- **`drizzle-graphql`**: auto GraphQL schema from Drizzle schema.
- **`eslint-plugin-drizzle`**: catches `delete`/`update` with no `where`.
- **`db.$count(table, where?)`**, **`getTableColumns(table)`**,
  **`getTableConfig(table)`**, **`is(value, Column)`** (use instead of
  `instanceof`), **`drizzle.mock()`** (connection-less `db` for tests).
