---
name: planetscale-postgres-serverless
description: >-
  Connecting to PlanetScale Postgres from JavaScript/TypeScript using the Neon
  serverless driver (`@neondatabase/serverless`) — the supported way to reach
  PlanetScale Postgres from serverless/edge runtimes with no persistent TCP
  (Cloudflare Workers, Deno Deploy, Netlify, Vercel Edge). Covers the
  HTTP mode (`neon()`, one-shot queries, `sql.query`, `sql.transaction`) vs the
  WebSocket mode (`Pool`/`Client`, interactive transactions, `pg` drop-in), the
  **required** `neonConfig` settings for PlanetScale (`fetchEndpoint`,
  `wsProxy`, `pipelineConnect = false` for SCRAM-SHA-256), connection-string
  format (`*.pg.psdb.cloud`, port 5432 vs 6432 PSBouncer), Node 19+ / `ws`
  polyfill, the "WebSocket can't outlive one request" rule, wiring it into
  Drizzle / Prisma / Kysely, running migrations, and when to use plain `pg` +
  PSBouncer instead. Use for `pscale_pw_`, `.pg.psdb.cloud`, `neonConfig`,
  `wsProxy`, `fetchEndpoint`, "PlanetScale Postgres edge/serverless".
---

# PlanetScale Postgres with the Neon serverless driver

PlanetScale Postgres is **wire-compatible with the Neon serverless driver**
(`@neondatabase/serverless`), so that driver is the supported way to query a
PlanetScale Postgres database from a runtime that can't hold a TCP connection
open between requests — Cloudflare Workers, Vercel Edge, Deno Deploy, Netlify
Functions. It speaks Postgres over **HTTP (fetch)** or **WebSocket** instead of
raw TCP.

- Driver: `@neondatabase/serverless` — <https://github.com/neondatabase/serverless>
  (docs `neon.com/docs/serverless/serverless-driver`, config
  `github.com/neondatabase/serverless/blob/main/CONFIG.md`).
- PlanetScale guide:
  <https://planetscale.com/docs/postgres/connecting/neon-serverless-driver>.
- **This skill reflects `@neondatabase/serverless` v1.1.0** (`vendor/neon-serverless`
  if you're in the `violets-skills` repo; that path is not present where this
  plugin is installed). **v1.0.0+ requires Node.js ≥ 19.** The v1 template
  function is template-tag-only (no `sql('...' )` call form).
- **This is not PlanetScale MySQL.** That product uses a different driver
  (`@planetscale/database`). This skill is only for the **Postgres** product.
- If you use **Drizzle ORM**, wire the driver in as below and see the
  `drizzle-orm` skill (`reference/drivers-and-perf.md`) for the ORM side.

## Decide first: do you even need this driver?

The Neon driver trades features and a little latency for working without TCP.
On a platform that **can** hold a pooled TCP connection, use plain `pg` against
PlanetScale's **PSBouncer** on port **6432** instead — it's faster and fully
featured.

| Platform | Use |
| --- | --- |
| Cloudflare Workers (no Hyperdrive), Deno Deploy, Netlify Functions | **Neon serverless driver** (this skill) |
| Cloudflare Workers **+ Hyperdrive** | `pg` via Hyperdrive |
| Vercel (Edge or Node Functions), AWS Lambda, Railway, Render, Fly, a VPS, Docker | `pg` + PSBouncer on port **6432** |

`reference/neonconfig-and-frameworks.md` has the `pg` + PSBouncer setup and its
transaction-pooling caveats.

## Connection string

```
postgresql://<user>:<pscale_pw_…>@<id>.pg.psdb.cloud:5432/<database>
```

- Host ends in **`.pg.psdb.cloud`**; password is **`pscale_pw_…`**.
- **Port 5432** = direct to Postgres (bounded by the cluster's
  `max_connections`). **Port 6432** = PSBouncer pooling (transaction mode). The
  Neon driver over HTTP/WS does its own connection handling — 5432 is fine and
  usual; use 6432 only for the plain-`pg` path.
- PlanetScale enforces TLS; `?sslmode=verify-full` is accepted and harmless.
- Get credentials from the PlanetScale dashboard → your database → **Connect**
  (pick "Neon serverless driver" or "Other" / Prisma to reveal the parts).
  Store the whole URL as `DATABASE_URL`.

## HTTP mode — one-shot queries

Lowest latency for a single query or a fixed batch. **No interactive
transactions, no session state, no `LISTEN`/`NOTIFY`.** 64 MB request/response cap.

```ts
import { neon, neonConfig } from "@neondatabase/serverless";

// ── REQUIRED for PlanetScale ──────────────────────────────────────────
neonConfig.fetchEndpoint = (host) => `https://${host}/sql`;

const sql = neon(process.env.DATABASE_URL!);

const posts = await sql`SELECT * FROM posts WHERE id = ${postId}`;   // parameterised
const one   = await sql.query("SELECT * FROM posts WHERE id = $1", [postId]);  // dynamic text
const tbl   = cond ? "a_posts" : "b_posts";
const rows  = await sql`SELECT * FROM ${sql.unsafe(tbl)} WHERE id = ${postId}`; // trusted identifier

// non-interactive transaction: array of queries, one round-trip, all-or-nothing
const [p, t] = await sql.transaction(
  [sql`SELECT * FROM posts LIMIT ${10}`, sql`SELECT * FROM tags`],
  { isolationLevel: "RepeatableRead", readOnly: true },
);
```

- **`neon()`'s template function is template-tag-only.** `sql("SELECT …")` as a
  plain call is a type + runtime error (SQL-injection guard). Use `sql.query()`
  for dynamic SQL with `$1` placeholders, `sql.unsafe()` for trusted
  table/column names.
- `neon(url, { arrayMode, fullResults, fetchOptions, types, authToken })`:
  `fullResults` returns `{ rows, fields, rowCount, command }` (node-postgres
  shape); `fetchOptions: { signal }` gives you a timeout via `AbortController`.

## WebSocket mode — sessions & interactive transactions

Full `pg`-compatible `Pool` / `Client`. Use when you need an interactive
transaction (read → decide → write in one tx), `SET`/session state, or you're
porting a `node-postgres` codebase.

```ts
import { Pool, neonConfig } from "@neondatabase/serverless";
// Node.js only (no global WebSocket): import ws from "ws";  // + `bufferutil`

// ── REQUIRED for PlanetScale ──────────────────────────────────────────
neonConfig.pipelineConnect = false;                       // SCRAM-SHA-256 (no cleartext pw)
neonConfig.wsProxy = (host, port) => `${host}/v2?address=${host}:${port}`;
// neonConfig.webSocketConstructor = ws;                   // Node.js only

export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    const pool = new Pool({ connectionString: env.DATABASE_URL });
    try {
      const { rows } = await pool.query("SELECT * FROM posts WHERE id = $1", [1]);
      // interactive tx:
      const client = await pool.connect();
      try {
        await client.query("BEGIN");
        const bal = (await client.query("SELECT balance FROM acct WHERE id=$1 FOR UPDATE", [1])).rows[0];
        await client.query("UPDATE acct SET balance = balance - $1 WHERE id=$2", [10, 1]);
        await client.query("COMMIT");
      } catch (e) { await client.query("ROLLBACK"); throw e; }
      finally { client.release(); }
      return Response.json(rows);
    } finally {
      ctx.waitUntil(pool.end());        // ⚠️ see below
    }
  },
};
```

- **A WebSocket connection cannot outlive one request** in a serverless/edge
  handler. **Create, use, and close the `Pool`/`Client` inside the handler.**
  Do **not** hoist a `Pool` to module scope (it works in `wrangler dev`, then
  hangs or errors in production). `ctx.waitUntil(pool.end())` (Workers) or
  `pool.end()` in a `finally` is the pattern.
- `neonConfig.pipelineConnect = false` is **mandatory** — PlanetScale requires
  SCRAM-SHA-256 and the default `"password"` pipelining assumes cleartext auth.
- `neonConfig.wsProxy` **must** include `?address=${host}:${port}` — that's how
  PlanetScale's proxy learns which backend to reach. Protocol is omitted
  (the driver adds `wss://`).
- Node.js needs `neonConfig.webSocketConstructor = ws` (`npm i ws bufferutil`).
  Browsers / Workers / Deno have a native `WebSocket` — don't import `ws` there.

## `neonConfig` — global vs per-client

`neonConfig` imported from the package sets **global defaults**; a `Client` also
has a `client.neonConfig` for per-instance overrides. **`fetchEndpoint`,
`poolQueryViaFetch` and `fetchFunction` are global-only.** Set the PlanetScale
values once at module load, before constructing any `neon()` / `Pool`. Full
option-by-option reference (incl. the experimental pure-JS TLS options you
should ignore) is in `reference/neonconfig-and-frameworks.md`.

## Using it with an ORM

| ORM | How |
| --- | --- |
| **Drizzle** | `drizzle-orm/neon-http` (HTTP) or `drizzle-orm/neon-serverless` (WS): set the `neonConfig` above, then `drizzle(process.env.DATABASE_URL!, { schema })` or `drizzle({ client: neon(url) / new Pool(...) , schema })`. See the `drizzle-orm` skill. |
| **Prisma** | `@prisma/adapter-neon` driver adapter + `previewFeatures = ["driverAdapters"]`; pass a configured `Pool`. |
| **Kysely** | `kysely-neon` dialect (or a custom dialect wrapping `Pool`). |
| **raw** | just `neon()` / `Pool` as above. |

The ORM sits on top; the `neonConfig` requirements and the mode choice are
unchanged.

## Migrations

The Neon driver is a *runtime* client. Run schema migrations with a tool that
connects however it likes:

- **Drizzle Kit** against PlanetScale Postgres: use a normal
  `dialect: "postgresql"` + `dbCredentials.url` config and let it connect with
  `pg` (Drizzle Kit auto-detects `pg`); or run generated SQL through
  `psql` / the PlanetScale CLI. The Neon driver is not a Drizzle Kit driver.
- Prisma Migrate / Atlas / raw `psql`: connect on **5432** (or 6432) directly —
  migrations run from your machine or CI, not from the edge function, so TCP is
  available.
- Apply migrations from CI, not from the serverless handler.

## Common pitfalls

- **Skipped `neonConfig.fetchEndpoint`** (HTTP) or **`wsProxy` / `pipelineConnect`**
  (WS) → connection failures or auth errors. All three PlanetScale-specific
  settings are required; they are *not* needed for actual Neon.
- **Hoisted a `Pool` to module scope** on Workers/edge → works locally, hangs in
  prod. WS pools are per-request.
- **Called `sql("SELECT …")`** as a function on v1 → error. Template tag,
  `sql.query()`, or `sql.unsafe()`.
- **Used HTTP mode for an interactive transaction** → there isn't one; either
  restructure as `sql.transaction([...])` (queries fixed up front) or switch to
  WebSocket mode.
- **Node < 19** → the driver won't run (needs global `fetch` / `WebSocket`).
- **Reached for this driver on Vercel Functions / Lambda** → PlanetScale
  recommends plain `pg` + PSBouncer `:6432` there; the serverless driver is for
  Workers / Deno / Netlify.
- **Left prepared statements on with PSBouncer `:6432`** (the `pg` path) →
  transaction-pooling mode breaks server-side prepared statements; disable them
  (`pg` ≥ 8 does not use them by default; ORMs may — see the reference file).
- **Confused with PlanetScale MySQL** — `@planetscale/database` /
  `drizzle-orm/planetscale` is a different product and driver.
