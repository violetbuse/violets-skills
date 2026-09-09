# `neonConfig`, framework setup, ORMs, and the `pg` alternative

Reference for `@neondatabase/serverless` v1.1.0 against PlanetScale Postgres.
`CONFIG.md` in the driver repo is the prose reference but is **slightly stale**
(the real `fetchEndpoint` default and the transaction-option key differ) —
verify defaults against `src/shims/net/index.ts` (`Socket.defaults`) and option
names against `src/httpTypes.ts`. PlanetScale guide:
`planetscale.com/docs/postgres/connecting/neon-serverless-driver`.

## The three PlanetScale-required settings

Set these **once at module load**, before any `neon()` / `Pool` / `Client`:

```ts
import { neonConfig } from "@neondatabase/serverless";

// HTTP mode:
neonConfig.fetchEndpoint = (host) => `https://${host}/sql`;

// WebSocket mode:
neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (host, port) => `${host}/v2?address=${host}:${port}`;
```

Why each is needed:

| Setting | Neon default | PlanetScale needs | Reason |
| --- | --- | --- | --- |
| `fetchEndpoint` | rewrites the **first** hostname label to `api.` (`xxx.pg.psdb.cloud` → `api.pg.psdb.cloud/sql`) — a Neon-ism | `(host) => \`https://${host}/sql\`` | The default assumes Neon's `api.`-prefixed proxy host. PlanetScale serves the SQL-over-HTTP endpoint on the **database host itself**, so you must override to use `host` verbatim. |
| `pipelineConnect` | `"password"` | `false` | Default pipelines startup+auth+first-query assuming **cleartext password** auth. PlanetScale mandates **SCRAM-SHA-256**, which can't be pipelined. |
| `wsProxy` | `host => host + '/v2'` | `(host, port) => \`${host}/v2?address=${host}:${port}\`` | PlanetScale's WS proxy is generic (not per-database); the `?address=` query tells it which Postgres backend to dial. |

`webSocketConstructor` is additionally required in Node.js (see below).

## Full `neonConfig` reference

Set on the imported `neonConfig` (global) or on `client.neonConfig` (per
`Client`). `fetchEndpoint`, `fetchFunction`, `poolQueryViaFetch` are **global
only**.

| Option | Default | Use |
| --- | --- | --- |
| `webSocketConstructor` | `undefined` | WS impl for envs with no global `WebSocket` (Node.js). `import ws from "ws"`. |
| `fetchEndpoint` | default rewrites first host label to `api.` (Neon proxy) | HTTP query endpoint. **PlanetScale: `(host) => \`https://${host}/sql\``.** Also change for local dev / non-443. |
| `wsProxy` | `host => host + '/v2'` | WS proxy address (protocol omitted). **PlanetScale: `(h,p) => \`${h}/v2?address=${h}:${p}\``**. |
| `pipelineConnect` | `"password"` | `false` when the server doesn't do cleartext-password auth. **PlanetScale: `false`.** |
| `poolQueryViaFetch` | `false` (may change) | Experimental: route `Pool.query()` through HTTP fetch (lower latency) when no pool event listeners are attached. Global only. |
| `fetchFunction` | `undefined` | Alternative `fetch` implementation. Global only. |
| `coalesceWrites` | `true` | Merge multiple run-loop writes into one WS message. Leave on. |
| `forceDisablePgSSL` | `true` | Disable Postgres-protocol TLS (safe: the WS layer is TLS). Leave on. |
| `useSecureWebSocket` | `true` | `wss://` vs `ws://`. Leave on. |
| `disableWarningInBrowsers` | `false` | Silence the "you're connecting from a browser" console warning (v1.0.1+). |
| `pipelineTLS`, `subtls`, `rootCerts`, `fetchConnectionCache` | — | **Experimental pure-JS TLS only. Ignore for PlanetScale.** |

## HTTP query function options (`neon(url, opts)` / `sql.query(text, params, opts)`)

| Option | Effect |
| --- | --- |
| `arrayMode: true` | Rows as arrays, not objects. |
| `fullResults: true` | Return `{ rows, fields, rowCount, rowAsArray, command }` (node-postgres shape) instead of a bare row array. |
| `fetchOptions: {…}` | Merged into the `fetch()` call — `{ signal }` for `AbortController` timeouts, `{ priority: "high" }`, etc. Not settable per-query inside `sql.transaction()`. |
| `types: { getTypeParser }` | Override `pg-types` parsers (e.g. parse `NUMERIC` as float, `DATE` as `Date`). |
| `authToken: string \| () => string \| Promise<string>` | Sets the `Authorization` header — for Neon RLS / third-party auth. Rarely relevant to PlanetScale. |

`sql.transaction(queriesOrFn, opts)`: `queriesOrFn` is an array of queries **or**
a non-`async` function `(txn) => [txn\`…\`, txn\`…\`]`. `opts` adds
`isolationLevel` (`"ReadUncommitted" | "ReadCommitted" | "RepeatableRead" |
"Serializable"`), `readOnly`, `deferrable`. It is **not interactive** — every
query is fixed before the round-trip.

## Framework setup

### Cloudflare Workers (no Hyperdrive)

```ts
// src/index.ts
import { neon, neonConfig, Pool } from "@neondatabase/serverless";

neonConfig.fetchEndpoint = (h) => `https://${h}/sql`;
neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (h, p) => `${h}/v2?address=${h}:${p}`;

export interface Env { DATABASE_URL: string }

export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    // HTTP — no cleanup needed
    const sql = neon(env.DATABASE_URL);
    const rows = await sql`SELECT now()`;

    // WebSocket — per request, close it
    const pool = new Pool({ connectionString: env.DATABASE_URL });
    try {
      await pool.query("SELECT 1");
    } finally {
      ctx.waitUntil(pool.end());
    }
    return Response.json(rows);
  },
};
```
- Native `WebSocket` — do **not** add `ws`.
- `wrangler.jsonc`: no special binding, `DATABASE_URL` goes in `.dev.vars` /
  `wrangler secret put DATABASE_URL`.
- With **Hyperdrive** instead: don't use this driver — use `pg` with the
  Hyperdrive connection string.

### Vercel

PlanetScale recommends **plain `pg` + PSBouncer `:6432`** on Vercel (both Edge
and Node functions can pool). If you specifically want the serverless driver on
Vercel Edge, the setup is identical to Workers (native `WebSocket`, no `ws`).

### Deno Deploy

```ts
import { neon, neonConfig } from "jsr:@neon/serverless";   // JSR, not npm
neonConfig.fetchEndpoint = (h) => `https://${h}/sql`;
const sql = neon(Deno.env.get("DATABASE_URL")!);
```
Native `WebSocket`. `deno add jsr:@neon/serverless`.

### Netlify Functions

npm `@neondatabase/serverless`; Node 19+ runtime. Node has no global
`WebSocket`, so for WS mode add `ws` + `bufferutil` and
`neonConfig.webSocketConstructor = ws`. HTTP mode needs nothing extra.

### Node.js (scripts, long-lived servers)

If it's a *long-lived* server you usually want plain `pg` + PSBouncer, not this
driver. If you do use it (e.g. sharing one code path with an edge deploy):

```ts
import ws from "ws";                        // npm i ws bufferutil
import { Pool, neonConfig } from "@neondatabase/serverless";

neonConfig.webSocketConstructor = ws;
neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (h, p) => `${h}/v2?address=${h}:${p}`;

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
// here a module-scope pool is fine — it's a real server, not a request handler
```

## ORM integration

### Drizzle ORM

```ts
// HTTP
import { neon, neonConfig } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
neonConfig.fetchEndpoint = (h) => `https://${h}/sql`;
export const db = drizzle({ client: neon(process.env.DATABASE_URL!), schema });
//  or:  drizzle(process.env.DATABASE_URL!, { schema })

// WebSocket
import { Pool, neonConfig } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-serverless";
neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (h, p) => `${h}/v2?address=${h}:${p}`;
// neonConfig.webSocketConstructor = ws;   // Node only
export const db = drizzle({ client: new Pool({ connectionString: url }), schema });
```
`neon-http` → `db.batch([...])`, no `db.transaction()`. `neon-serverless` →
full `db.transaction(async (tx) => …)`. See the `drizzle-orm` skill for the ORM
side.

### Prisma

Driver adapter (`@prisma/adapter-neon`):
```prisma
generator client { provider = "prisma-client-js"; previewFeatures = ["driverAdapters"] }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }
```
```ts
import { Pool, neonConfig } from "@neondatabase/serverless";
import { PrismaNeon } from "@prisma/adapter-neon";
import { PrismaClient } from "@prisma/client";

neonConfig.pipelineConnect = false;
neonConfig.wsProxy = (h, p) => `${h}/v2?address=${h}:${p}`;
const adapter = new PrismaNeon(new Pool({ connectionString: process.env.DATABASE_URL }));
export const prisma = new PrismaClient({ adapter });
```
Run `prisma migrate` / `db push` over a direct `:5432` connection (a separate
`DIRECT_URL`), not through the adapter.

### Kysely

`kysely-neon` provides a `NeonDialect` (HTTP) and `NeonHTTPDialect` /
pool-based dialects; or write a `PostgresDialect` around a per-request `Pool`.
Same `neonConfig` requirements.

## The `pg` + PSBouncer alternative (`:6432`)

For Lambda / Vercel Functions / Railway / Render / a VPS — anywhere a pooled TCP
connection survives — this is faster and fully featured:

```ts
import { Pool } from "pg";
export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,   // …@<host>:6432/<db>?sslmode=verify-full
  max: 5,
});
```
- Connect on **6432** for PSBouncer (transaction pooling) or **5432** direct.
- PlanetScale's dashboard "Connect" tab gives the exact host + SSL params
  (`sslnegotiation=direct sslmode=verify-full sslrootcert=system` for `psql`).
- **Transaction-pooling mode (`:6432`) breaks**: protocol-level prepared
  statements that persist across transactions, temp tables, `LISTEN`/`NOTIFY`,
  session advisory locks, and `SET` beyond one transaction. Use **5432** for
  DDL / migrations / `pg_dump` / long transactions / session features.
- **Disable server-side prepared statements** for ORMs on `:6432`:
  - `postgres` (postgres.js): `postgres(url, { prepare: false })`
  - `node-postgres`: doesn't use them unless you call `.prepare()` — fine
  - Prisma: add `?pgbouncer=true` to the pooled URL
  - Drizzle: uses whichever the driver does — with `postgres.js` set
    `prepare: false`; don't call `.prepare()` on queries.

## Migrations

The serverless driver is a runtime client only. Migrate with a tool that opens
its own connection from your machine or CI (where TCP is available), on **5432**:

- **Drizzle Kit**: `dialect: "postgresql"`, `dbCredentials.url` pointing at
  `:5432`; it connects with `pg`. `drizzle-kit generate` / `migrate` / `push` /
  `studio` all work. There is no "neon" Drizzle Kit driver.
- **Prisma**: `directUrl`/`DIRECT_URL` at `:5432` for `migrate`.
- **Atlas / raw `psql`**: `:5432` with the SSL params from the dashboard.
- Never run migrations from inside the serverless handler.
