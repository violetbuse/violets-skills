---
name: celld
description: >-
  Working with celld — Deno's self-hosted, distributed Durable Objects daemon
  that runs a Cloudflare Workers app (Workers, Durable Objects, KV, Queues, D1,
  R2, Workflows, Cron, static assets) on your own machines from your existing
  wrangler.json, with durable state in an S3/GCS/Azure bucket you own. Use when
  writing or deploying a Worker/Durable Object app to celld, running or
  operating a celld fleet, using the `celld` CLI (`dev`, `deploy`, `diagnose`,
  `cell`, `d1`, `kv`, `queue`), choosing bucket storage, tuning `CELLD_*`
  environment variables, passing secrets or Worker vars without writing them to
  the bucket, planning a version upgrade, or reasoning about celld's
  one-writer / RPO=0 durability guarantees.
---

# celld

celld is an open-source daemon (`celld`, written in Rust, Apache-2.0, by Deno
Land Inc.) that runs a **Cloudflare Workers application on your own machines**.
It deploys from the `wrangler.json` / `wrangler.jsonc` you already have and
stores long-term state in a bucket you own (S3-compatible, Google Cloud
Storage, or Azure Blob Storage). No serving control plane, no consensus
service.

- Upstream: <https://github.com/denoland/celld> · docs <https://celld.dev/docs>
  (the docs site is generated from the repo's `docs/` folder — same content).
- **This skill reflects celld v0.4.1** (tagged 2026-09-05). celld is **alpha**:
  APIs, the operator API, and `CELLD_*` defaults can change between releases.
  Keep operator tooling and the celld binary on the same release — if the
  project is on a different version, verify anything version-sensitive against
  that release's docs.
- **This skill is self-contained** — `SKILL.md` plus `reference/architecture.md`,
  `reference/operations.md`, `reference/cloudflare-compat.md`. For anything
  deeper, go to the source: `celld --help` on the installed binary, the docs at
  the link above (or fetch `docs/*.md` from the GitHub repo), and in the repo
  itself `crates/celld/protocol.rs` (bucket-contract types),
  `crates/celld/main/cli.rs` (full CLI help), and `examples/*` (one runnable
  Wrangler project per service). If you happen to be working inside the
  `violets-skills` repo, a checkout is vendored at `vendor/celld/`; that path
  does **not** exist where this plugin is installed.

## Mental model

| Term | Meaning |
| --- | --- |
| **cell** | A Durable Object: a named server with its own private SQLite database. One writer, single-threaded. Everything stateful in celld is a cell — a KV namespace, a queue, a D1 database, a Workflow, and the R2 index are each a cell in a reserved class (`__`-prefixed). |
| **node** | One `celld` process. You run one per machine. Every node embeds V8 and runs Wrangler bundles. |
| **fleet** | The set of nodes sharing one bucket. A fleet runs **one** application. Add capacity by starting another node against the same bucket — no join command, no membership list. |
| **bucket** | The root of authority. Holds deployments, cell SQLite state (as LTX data), ownership records, node leases, and the peer-auth secret. Whoever holds its credentials controls the fleet. |

**Cell lifecycle:** `inactive` (only an object in the bucket, costs ~nothing) →
`resident` in memory (`active` while working, `idle` while waiting) →
`hibernated` (evicted from memory but keeps hibernatable WebSocket clients and
stays on its node). A cell keeps **no memory** across transitions — the
constructor re-runs on the next event, like a Cloudflare cold start.

**Ownership & correctness:** exactly one node serves a cell at a time. A node
claims a cell with a conditional bucket write carrying a fencing **epoch**; the
object store decides who wins, so two nodes can't both own it. Claims expire
unless renewed, so a dead machine releases its cells with no failure detector.
Replicated SQLite data is written under `cells/<cell>/ltx/e<epoch>/`, so a node
that lost ownership writes only into a superseded prefix.

**Durability (RPO=0):** celld never acknowledges a write until it survives a
failure. `CELLD_DURABILITY=fleet` (the default): the owner streams each write
to one or two follower nodes and acks once they've `fsync`'d it (a lab fleet
measured ~25 ms), uploading to the bucket afterward. **A single node has no
follower**, so every write waits for a bucket round trip (~90 ms region-local,
~600 ms otherwise) — run **2+ nodes if write latency matters**. `bucket` mode
always waits for the bucket. `CELLD_OUTPUT_GATE=0` removes the wait and accepts
possible loss of an acknowledged write.

See `reference/architecture.md` for the full protocol (fencing, epoch-chain
restore, takeover recovery, self-fencing, bucket requirements).

## Is a workload a good fit?

Cells fit workloads that divide into named, stateful units: real-time apps
(one cell per game/chat room/doc holds its own WebSockets, no external bus),
AI agents (one cell per agent — memory, schedule, inbox in its SQLite DB; idle
agents cost ~nothing), and sharded web apps (one cell per user/tenant/device —
no shared-database contention because there's no shared database).

Out of scope: anything needing the Cloudflare edge network, a GPU, or a
browser farm. Not safe for hostile multi-tenant use — one fleet trusts its
application code, its nodes, and its operators.

## Develop locally: `celld dev`

```sh
celld dev                 # current dir; needs a wrangler.jsonc/json
celld dev ./examples/counter
celld dev --port 3000     # Worker listener; default http://127.0.0.1:9876
celld dev --host 0.0.0.0  # expose Worker listener to the network
celld dev --logs          # show node warning/info logs (errors always show)
celld dev --clean         # wipe .celld/dev first, start from empty state
celld dev --no-watch      # disable auto rebuild/restart on file changes
```

- Opens a **local SQLite object store** — no Docker, no cloud bucket. A regular
  node and the operator subcommands **cannot** use this local backend.
- State lives in `.celld/dev` under the project and **survives restarts**. Add
  `.celld/` to the app's `.gitignore`.
- **Gotcha:** `celld dev` does **not** migrate persisted state across a config
  change. An object can keep a value the new config rejects, and the resulting
  error names the stored value, not the config — looks unrelated to the change.
  Use `--clean` to start fresh.
- Watches the project and rebuilds on source/config change; a failed build
  leaves the running app up. Ignores `.celld`, `.git`, `.wrangler`,
  `node_modules`, `target` at every depth; add globs with `--watch-ignore`.
- Worker projects need **`esbuild` on `PATH`**; asset-only projects don't.
- `NO_COLOR` disables color, `FORCE_COLOR` forces it (`NO_COLOR` wins).

## Deploy: `celld deploy`

```sh
celld deploy .                     # PROJECT dir or wrangler config; default cwd
celld deploy . --bucket s3://my-cells-bucket \
  --endpoint https://ACCOUNT.r2.cloudflarestorage.com --region auto
celld deploy . --bucket gs://my-cells-bucket    # no --endpoint/--region
celld deploy . --bucket az://my-container        # AZURE_STORAGE_ACCOUNT_NAME set
celld deploy . --dry-run           # bundle + print version, write nothing
celld deploy . --json
```

- A fleet runs **one** application; `celld deploy` writes the deployment
  objects and moves the `deploy/current.json` pointer. The `name` in the
  config must be 1–63 lowercase ASCII letters/digits/internal hyphens.
- **No node restarts.** Each node polls `deploy/current.json` every 30 s
  (`CELLD_DEPLOY_POLL_S`) and adopts the new deployment in place; `POST
  /reload` on the internal listener adopts it now. A build failure leaves the
  current deployment serving and is reported in the log and the `/reload`
  response. `POST /reload` also rebuilds unchanged code, so a `CELLD_VARS_FILE`
  edit applies without a restart.
- A resident Durable Object moves to new code at a safe point (no request /
  alarm running, no output awaiting durability, no regular WebSocket open).
  Objects that reach no safe point within `CELLD_DEPLOY_MAX_AGE_S` (default 60)
  are forced: running work cancelled, regular WebSockets closed with code 1012
  — exactly what a Cloudflare deployment does. During the window a request on
  one deployment can call a Durable Object on the other, so **two adjacent
  versions must accept each other's calls**.
- Worker code needs `esbuild` on `PATH`. An unknown key in the Wrangler config
  stops the deploy. See `reference/cloudflare-compat.md` for the accepted
  config subset and every service gap.

## Run a node

```sh
# Local / single node: default listener is enough
celld --bucket "$CELLD_BUCKET" --endpoint "$S3_ENDPOINT" --region "$AWS_REGION"

# Fleet node: bind public and internal listeners separately
celld --bucket "$CELLD_BUCKET" --endpoint "$S3_ENDPOINT" --region "$AWS_REGION" \
  --listen 0.0.0.0:8080 \
  --internal-listen 10.0.0.12:8081 \
  --advertise node-a.internal:8081
```

- **Two listeners.** `--listen` (default `127.0.0.1:8080`) serves the deployed
  Worker — put it behind your ingress/TLS. `--internal-listen` (default
  `127.0.0.1:0`) serves peer traffic + the **unauthenticated** operator API —
  keep it on a trusted private network or an encrypted overlay (WireGuard /
  Tailscale), never public.
- An explicit `--advertise` requires an explicit `--internal-listen`; a
  non-loopback `--listen` also requires an explicit `--internal-listen`. celld
  rejects a literal public IP in `--advertise` unless `--unsafe-public-advertise`.
  celld cannot verify that the advertised address routes to the internal
  listener — you must.
- Nodes discover each other through bucket leases. Start a second node with the
  same bucket settings and a distinct reachable internal address; no other
  config. The load balancer must include new nodes in rotation (the listener
  that receives a request decides where a new cell activates).
- Run under a **supervisor that restarts with no attempt limit** and waits ≥1
  lease lifetime between attempts (systemd, Docker restart policy, Kubernetes).
  A node that loses its lease **self-fences** and exits with code 3
  (`SELF-FENCE:` log line).
- Container: `ghcr.io/denoland/celld` (Linux x86-64 / ARM64). Persist
  `CELLD_WATCH`, pass the AWS credential env through, expose 8080 via the LB,
  keep 8081 private.
- Installer: `curl -fsSL https://celld.dev/install.sh | sh` (pin with
  `CELLD_VERSION=v0.4.1`; verify with `gh attestation verify <asset> --repo
  denoland/celld`).

## Operator CLI

All operator subcommands take the shared fleet flags (`--bucket`,
`--endpoint`, `--region`, or `CELLD_BUCKET` / `S3_ENDPOINT` / `AWS_REGION`) and
need a **running fleet** — they find a node via bucket leases and sign requests
with the fleet secret read from the bucket. Data goes to stdout, messages to
stderr. Listings are bounded to 1000 rows; continue with `--after CURSOR` or
read everything with `--all`; `--json` gives one object per line.

| Command | Purpose |
| --- | --- |
| `celld diagnose` | Enumerate node leases, then signed direct probe of each live peer. Reports expired records, unsafe/malformed advertise addresses, unreachable peers, auth failures, protocol mismatches, and each node's load sample. `--peer NODE_ID` (repeatable) restricts it; `--read-only` skips the bucket write probe. |
| `celld cell list [CLASS]` | List Durable Object instances (`Class:ID` per line). An instance appears only after its first event (its owner then writes an ownership record); a derived-only ID doesn't. Reserved runtime cells show with `"reserved": true`. |
| `celld d1 execute\|migrations apply\|migrations list DB` | Run SQL / migrations against a deployed D1 database. `DB` is a `database_name` from `d1_databases`. Migration files are `NNNN_description.sql` in `migrations/` (`.sql` case-insensitive). |
| `celld kv get\|put\|delete\|list\|info NS` + `celld kv bulk get\|put\|delete` | Read/write a deployed KV namespace. `NS` is the `id` from `kv_namespaces`, verbatim. Bulk uses Wrangler's file format, so `wrangler kv bulk get` exports straight into `celld kv bulk put`. |
| `celld queue info\|peek\|purge\|pause\|resume\|redrive QUEUE` | Inspect/control a deployed Queue. `pause` stops delivery while producers keep sending; `purge` needs `--force`. |

Managed control plane subcommands also exist (`celld connect`, `credentials`,
`token`, `disconnect`) for connecting an installation to celld.dev's Managed
Control Plane.

See `reference/operations.md` for fleet operation in depth: the internal
operator API (`/state`, `/reload`, `/shutdown`, `/rebalance/pause`), autoscaler
signals, ownership balancing, memory-pressure shedding, graceful shutdown /
rollout, and the per-version upgrade exceptions (several upgrades are **not**
rolling-safe).

## Writing the application

The application is **ordinary Cloudflare Workers code** — it must also run on
`workerd` (that's how celld tests compatibility). The celld repo's `examples/`
directory has one runnable Wrangler project per service; deploy any of them
straight from its own directory with `celld deploy .`. A minimal SQLite-backed
Durable Object (the `counter` example):

```js
// A SQLite-backed Durable Object (examples/counter)
export class Counter {
  constructor(state, env) { this.state = state; }
  async fetch(request) {
    let n = (await this.state.storage.get("n")) ?? 0;
    n++;
    await this.state.storage.put("n", n);
    return Response.json({ n, url: request.url });
  }
}
export default {
  async fetch(request, env) {
    const name = new URL(request.url).searchParams.get("name") ?? "default";
    return env.COUNTER.get(env.COUNTER.idFromName(name)).fetch(request);
  },
};
```

```jsonc
{
  "name": "counter",
  "main": "index.js",
  "compatibility_date": "2026-01-01",
  "durable_objects": { "bindings": [{ "name": "COUNTER", "class_name": "Counter" }] },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["Counter"] }]
}
```

Example projects (`examples/<name>/` in the celld repo): `hello` (stateless fetch),
`webapi`, `counter` / `router` / `async` (Durable Object + SQLite storage),
`rpc` (JS RPC via `getByName`), `wsecho` (hibernating WebSocket),
`wsclient` (outbound WebSocket from a DO), `alarm`, `cron`, `d1`, `kv`, `r2`,
`workflow`, `vectordb` (`sqlite_vec` flag), `wasm` (Rust via workers-rs),
`pi` / `opencode` (agent loops in a DO).

**Compatibility highlights** (full list + every gap in
`reference/cloudflare-compat.md`):

- **Yes:** Workers, Durable Objects (SQLite storage, alarms, hibernating
  WebSockets), static assets (`_headers`/`_redirects`, no edge cache/compression),
  Cron Triggers, KV, Queues, D1, Workflows, R2. Most runtime APIs — fetch,
  streams, WebSockets, Web Crypto, WebAssembly, HTMLRewriter, TCP sockets.
- **Experimental (opt-in):** Dynamic Workers / Worker Loader
  (`CELLD_WORKER_LOADER`), Durable Object Facets, `env.AI` HTTP adapter
  (`CELLD_AI_URL`).
- **No:** Workers AI (native), Vectorize, Hyperdrive, Browser Rendering, Email
  Workers, Python Workers, BroadcastChannel, `tail`/`email` handlers.
- **Partial:** Node.js compat (a fixed module set), Cache API (always-miss).
- **Key differences:** no TLS termination (do it at ingress); SQLite `TEXT`
  rejects invalid UTF-8 (use `BLOB`); one writer per KV namespace and per
  queue (add more to scale writes); `wrangler.toml` is not accepted — use
  `wrangler.jsonc` / `wrangler.json`; unknown top-level config keys
  (incl. `routes`) stop the deploy.
- **Hot-cell overload:** a cell admits 64 concurrent fetch events
  (`CELLD_MAX_CELL_REQUESTS`); excess gets HTTP 503 + `Retry-After: 1` +
  `X-Celld-Overload: cell`.

## Secrets and Worker vars

celld has **no encrypted secret store** — nothing like `wrangler secret put`,
and it does **not** read `.dev.vars`. Configuration values reach the Worker as
**vars** (`plain_text` bindings, read as `env.NAME`), resolved on each node when
it builds a deployment. Three sources, later wins:

1. **`vars` in `wrangler.json`** — each entry becomes a `plain_text` binding
   baked into the deployment manifest. celld requires every value to be a
   **string** (no JSON/object vars).
2. **`CELLD_VARS_FILE`** — path, on the node, to a dotenv-style file: `NAME=value`
   per line; blank lines and `#` comments ignored; one surrounding pair of `'` or
   `"` stripped; no escapes, no multi-line values.
3. **`CELLD_VAR_<NAME>`** env vars on the node — `CELLD_VAR_API_KEY` sets binding
   `API_KEY`.

**The manifest is written to the bucket in the clear.** `celld deploy` uploads
`deploy/<name>/<version>/manifest.json` to your S3/GCS/Azure bucket, so every
value in `wrangler.json` `vars` lands there unencrypted — and stays, because old
deployment versions are immutable. Anyone with bucket read access can read them.

**To pass a secret without pushing it to the bucket:** keep it out of `vars` and
supply it on each node through `CELLD_VARS_FILE` (or `CELLD_VAR_*`). Those are
read locally at build time and never uploaded. The Worker still reads it as
`env.SECRET_NAME`, exactly like any other var.

- Resolution is **per node**. Every node in the fleet needs the same file/env,
  delivered out of band (systemd `EnvironmentFile=`, a mounted secret, your
  config manager). A var present on only some nodes makes requests behave
  differently depending on which node served them.
- Declaring the name in `wrangler.json` `vars` with an empty or placeholder
  value is fine — a node-level source overrides it — and keeps the binding
  visible and type-checked. That placeholder still goes to the bucket, so never
  make it a usable fallback secret.
- Editing `CELLD_VARS_FILE` then `POST /reload` (or waiting for the 30 s poll)
  applies new values with **no restart** — `/reload` rebuilds even when the code
  is unchanged.
- `celld dev` reads `vars` from `wrangler.json` and also honors
  `CELLD_VARS_FILE` / `CELLD_VAR_*` from its own environment.

## Bucket storage

celld needs conditional writes (create-if-absent and compare-and-swap), exact
ranged reads, and read-after-write consistency. **Qualified:** Amazon S3,
Cloudflare R2, Google Cloud Storage, Tigris, Azure Blob Storage. **Do not
use:** Backblaze B2, Hetzner, DigitalOcean Spaces (no working conditional
writes — two nodes can then own one cell). MinIO CE passes the storage test but
is not production-qualified.

```sh
# Cloudflare R2 (S3-compatible)
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=... AWS_REGION=auto
export S3_ENDPOINT=https://ACCOUNT_ID.r2.cloudflarestorage.com
export CELLD_BUCKET=s3://YOUR-BUCKET        # optional /PREFIX lets fleets share a bucket

# GCS: Application Default Credentials; CELLD_BUCKET=gs://YOUR-BUCKET (no endpoint/region)
# Azure: AZURE_STORAGE_ACCOUNT_NAME + one credential family; CELLD_BUCKET=az://YOUR-CONTAINER
```

Each node runs the storage-contract test at startup (`CELLD_STORAGE_PROBE=0`
disables) and stops if a required property is missing or the store silently
ignores a condition. `celld diagnose` runs the same probe on demand. celld
reserves these bucket prefixes — the application must not write under them:
`probe/`, `cells/`, `nodes/`, `node-cells/`, `fleet/`, `deploy/`,
`deploy-blobs/`, `wake/`, `telemetry/`.

## Telemetry (off by default)

`CELLD_OTEL=1` records a span per request/event/outbound-fetch/cell-start and a
log record per `console.log`. Default sink writes Parquet to the fleet bucket
under `telemetry/` (query with DuckDB); `CELLD_OTEL_SINK=otlp` sends OTLP/HTTP
to a collector instead. Reads W3C `traceparent`. Details + DuckDB queries in
`reference/operations.md`.

## Common pitfalls

- **Single-node writes are slow** — no follower, so every write waits for the
  bucket. Run 2+ nodes when latency matters.
- **`celld dev` state is not migrated** across config changes; errors point at
  the stored value, not the config. Use `--clean`.
- **Some version upgrades are not rolling-safe** (v0.1→v0.2, v0.3→v0.4). Check
  `reference/operations.md` before upgrading a fleet.
- **The internal listener is unauthenticated** for most operator routes and has
  no TLS — a trusted private network is mandatory.
- **`wrangler.toml` and `routes` are rejected.** Convert config to JSON and
  configure routing in your ingress.
- **Bucket credentials = full fleet control.** Scope each credential to one
  fleet bucket.
- **`wrangler.json` `vars` go to the bucket in the clear** and stay in every
  past deployment version. celld has no secret store — pass secrets per node via
  `CELLD_VARS_FILE` / `CELLD_VAR_*`, which never leave the machine. See
  *Secrets and Worker vars*.
- celld is **alpha**: pin `CELLD_VERSION`, keep operator tooling and binary on
  the same release, expect `CELLD_*` defaults to shift.
