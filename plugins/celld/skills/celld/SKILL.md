---
name: celld
description: >-
  Working with celld — Deno's self-hosted, distributed Durable Objects daemon
  that runs a Cloudflare Workers app (Workers, Durable Objects, KV, Queues, D1,
  R2, Workflows, Cron, static assets) on your own machines from your existing
  wrangler.json, with durable state in an S3/GCS/Azure bucket you own. Use when
  writing or deploying a Worker/Durable Object app to celld, running or
  operating a celld fleet, using the `celld` CLI (`dev`, `deploy`, `diagnose`,
  `cell list`, `cell gc`, `d1`, `kv`, `queue`, `r2`), deploying a Python
  Worker or a Dynamic Worker (Worker Loader) to celld, choosing bucket storage,
  controlling bucket growth with epoch GC, tuning `CELLD_*` environment
  variables, passing secrets or Worker vars without writing them to the bucket,
  planning a version upgrade, or reasoning about celld's one-writer / RPO=0
  durability guarantees.
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
- **This skill reflects celld v0.6.2** (tagged 2026-10-07). celld calls v0.6.x
  a **beta** release (it was alpha through v0.5.x); the internal operator API
  is still explicitly **alpha**, and `CELLD_*` defaults can still change
  between releases. Only the latest release gets security fixes. Keep operator
  tooling and the celld binary on the same release — if the project is on a
  different version, verify anything version-sensitive against that release's
  docs.
- **This skill is self-contained** — `SKILL.md` plus `reference/architecture.md`,
  `reference/operations.md`, `reference/cloudflare-compat.md`. For anything
  deeper, go to the source: `celld --help` on the installed binary, the docs at
  the link above (or fetch `docs/*.md` from the GitHub repo), and in the repo
  itself `crates/celld/protocol.rs` (bucket-contract types),
  `crates/celld/main/cli.rs` (full CLI help), `docs/services/*.md` (one page
  per service, each with its Cloudflare differences), and `examples/*` (one
  runnable Wrangler project per service). If you happen to be working inside the
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

**Bucket growth:** each activation can add a new epoch prefix, and by default
celld **deletes none of them** — a cell's bucket bytes grow with every
activation. Set `CELLD_LTX_RETENTION_SECS` (v0.6.1+, every node on ≥v0.6.1
first) to enable **epoch GC**: the owner deletes prefixes no restore reads,
keeping its own epoch, the one before, and anything younger than the grace.
Preview it with `celld cell gc --dry-run`. Epoch GC needs list-after-write
consistency (don't enable it on a multi-region Tigris Global/Dual-region
bucket) and skips facet streams.

**Durability (RPO=0):** celld never acknowledges a write until it survives a
failure. `CELLD_DURABILITY=fleet` (the default): the owner streams each write
to one or two follower nodes and acks once they've `fsync`'d it (a lab fleet
measured ~25 ms) or the bucket upload finishes, whichever comes first. A node
with only one follower recruits a second as soon as another node is available
(v0.6.2). **A single node has no
follower**, so every write waits for a bucket round trip (~90 ms region-local,
~600 ms otherwise) — run **2+ nodes if write latency matters**. `bucket` mode
always waits for the bucket. There is no setting to skip the durability wait —
`CELLD_OUTPUT_GATE` was removed in v0.5.0; celld always proves a write durable
before it acknowledges it.

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

- Opens a **local SQLite object store** — no Docker, no cloud bucket (unless
  the project has a `containers` entry — see *Containers*). A regular node and
  the operator subcommands **cannot** use this local backend.
- State lives in `.celld/dev` under the project and **survives restarts**. Add
  `.celld/` to the app's `.gitignore`.
- Reads a `.dev.vars` file beside the Wrangler config, as `wrangler dev` does;
  **without `.dev.vars` it reads `.env`, then `.env.local`** (which overrides
  `.env`) — v0.6.2. Each `NAME=value` entry becomes a Worker var and overrides
  a same-named entry in `vars`. Lines may start with `export`, and a quoted
  value may span lines (e.g. a PEM key); trailing comments and `${VAR}`
  references are not supported. An entry that isn't a valid binding name (or
  collides with another binding) is skipped with a warning in `.env` files but
  **stops the build** in `.dev.vars`. Only `celld dev` reads these files —
  `celld deploy` never does, so a local credential doesn't reach a fleet. Add
  `.dev.vars`, `.env`, `.env.local` to `.gitignore`; editing one rebuilds the
  app with no restart.
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
  response. `POST /reload` also rebuilds unchanged code.
- A resident Durable Object moves to new code at a safe point (no request /
  alarm running, no output awaiting durability, no regular WebSocket open).
  Objects that reach no safe point within `CELLD_DEPLOY_MAX_AGE_S` (default 60)
  are forced: running work cancelled, regular WebSockets closed with code 1012
  — exactly what a Cloudflare deployment does. During the window a request on
  one deployment can call a Durable Object on the other, so **two adjacent
  versions must accept each other's calls**.
- Worker code needs `esbuild` on `PATH`. An unknown key in the Wrangler config
  stops the deploy. Since v0.5.1, Wrangler's `define` and module `rules`
  (`Text` / `Data` / `CompiledWasm`, globs of the form `**/*.ext` or `*.ext`
  only, no `fallthrough` key) are accepted, so e.g. `.sql` / `.txt` imports
  as Text modules work. See `reference/cloudflare-compat.md` for the accepted
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
  `CELLD_VERSION=v0.6.2`; verify with `gh attestation verify <asset> --repo
  denoland/celld`).

## Operator CLI

All operator subcommands take the shared fleet flags (`--bucket`,
`--endpoint`, `--region`, or `CELLD_BUCKET` / `S3_ENDPOINT` / `AWS_REGION`).
`d1`, `kv`, `queue`, and `diagnose` need a **running fleet** — they find a
node via bucket leases and sign requests with the fleet secret read from the
bucket; `r2` and `cell` read the bucket directly. Data goes to stdout,
messages to stderr (a closed stdout pipe is a clean stop). Listings are
bounded to 1000 rows; continue with `--after CURSOR` or read everything with
`--all`; `--json` gives one object per line.

| Command | Purpose |
| --- | --- |
| `celld diagnose` | Enumerate node leases, then signed direct probe of each live peer. Reports expired records, unsafe/malformed advertise addresses, unreachable peers, auth failures, protocol mismatches, and each node's load sample. `--peer NODE_ID` (repeatable) restricts it; `--read-only` skips the bucket write probe. |
| `celld cell list [CLASS]` | List Durable Object instances (`Class:ID` per line). An instance appears only after its first event (its owner then writes an ownership record); a derived-only ID doesn't. Reserved runtime cells show with `"reserved": true`. |
| `celld d1 execute\|migrations apply\|migrations list DB` | Run SQL / migrations against a deployed D1 database. `DB` is a `database_name` from `d1_databases`. Migration files are `NNNN_description.sql` in `migrations/` (`.sql` case-insensitive). |
| `celld kv get\|put\|delete\|list\|info NS` + `celld kv bulk get\|put\|delete` | Read/write a deployed KV namespace. `NS` is the `id` from `kv_namespaces`, verbatim. Bulk uses Wrangler's file format, so `wrangler kv bulk get` exports straight into `celld kv bulk put`. |
| `celld queue info\|peek\|purge\|pause\|resume\|redrive QUEUE` | Inspect/control a deployed Queue. `pause` stops delivery while producers keep sending; `purge` needs `--force`. |
| `celld r2 get\|head\|put\|delete\|list BUCKET [KEY]` | (v0.5.1+) Read/write the objects behind an `r2_buckets` binding — replaces `wrangler r2 object`. `BUCKET` is the `bucket_name`, not the binding name. **Needs no running node**, so a release pipeline can publish an artifact before deploying. `put` takes `--path FILE` or `--pipe`, plus `--content-type` etc. (→ `httpMetadata`) and `--metadata '{"k":"v"}'` (→ `customMetadata`). |
| `celld cell gc --dry-run [CLASS]` | (v0.6.1+) Per cell, list the superseded epoch prefixes epoch GC could delete, their bytes, and the restore base. Writes nothing; `--grace-secs N` sets the preview grace. Real deletion happens only on nodes with `CELLD_LTX_RETENTION_SECS` set. |

Managed control plane subcommands also exist (`celld connect`, `credentials`,
`token`, `disconnect`) for connecting an installation to celld.dev's Managed
Control Plane.

See `reference/operations.md` for fleet operation in depth: the internal
operator API (`/state`, `/reload`, `/shutdown`, `/evict`, `/rebalance/pause`), autoscaler
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
`workflow`, `queues` (producer + consumer + dead-letter queue),
`static-assets` (`_headers`/`_redirects`, no Worker), `vectordb` (`sqlite_vec`
flag), `wasm` (Rust via workers-rs), `facets` (Worker Loader + a facet class),
`dynamic-worker-tails` (Worker Loader + Tail Worker), `python` (Python
Worker), `container` / `sandbox` (Cloudflare Sandbox SDK on a container),
`pi` / `opencode` (agent loops in a DO).

Two behaviors that now match workerd and can break code that worked on older
celld: every export of the main module must be a handler object or a class
(`export const X = "..."` fails the Worker's start, v0.6.0), and returning a
WebSocket response to a request without `Upgrade: websocket` fails with a
`TypeError` / HTTP 500 (v0.6.2).

**Compatibility highlights** (full list + every gap in
`reference/cloudflare-compat.md`):

- **Yes:** Workers, Durable Objects (SQLite storage, alarms, hibernating
  WebSockets, Facets — each facet with its own replicated SQLite stream since
  v0.6.0), static assets (`_headers`/`_redirects`, no edge
  cache/compression), Cron Triggers, Dynamic Workers (Worker Loader, declared
  with the `worker_loaders` config key — no env var; `limits` and `tails` in
  `WorkerCode` supported since v0.5.1), KV, Queues, D1, Workflows, R2. Most
  runtime APIs — fetch, streams, WebSockets, Web Crypto (incl. Ed25519 /
  X25519 since v0.6.0), WebAssembly (including `no_bundle` prebuilt
  deployments), HTMLRewriter, TCP sockets.
- **Experimental (opt-in):** Containers (the `containers` config key; needs a
  Docker/Podman daemon on every node that serves the class — see
  *Containers*), including `@cloudflare/containers` and the Cloudflare
  Sandbox SDK.
- **No:** Workers AI (no binding, no HTTP adapter — call the provider
  directly), Vectorize, Hyperdrive, Browser Rendering, Email Workers,
  BroadcastChannel, `tail`/`email` handlers.
- **Partial:** Python Workers (v0.6.1+: `fetch` handlers only, Pyodide 0.28 —
  see *Python Workers* below), Node.js compat (a fixed module set), Cache API
  (always-miss).
- **Key differences:** no TLS termination (do it at ingress); invalid UTF-8 in
  SQLite `TEXT` decodes as U+FFFD (since v0.6.0 — it used to error; store
  arbitrary bytes as `BLOB`); one writer per KV namespace, per queue, and per
  D1 database (add more to scale writes); `wrangler.toml` is not accepted —
  use `wrangler.jsonc` / `wrangler.json`; unknown top-level config keys
  (incl. `routes`) stop the deploy.
- **Hot-cell overload:** a cell admits 64 concurrent fetch events
  (`CELLD_MAX_CELL_REQUESTS`); excess gets HTTP 503 + `Retry-After: 1` +
  `X-Celld-Overload: cell`.

## Containers (experimental)

A `containers` entry (`class_name`, `image`, `name`, `instance_type`,
`max_instances`) binds a Docker/Podman image to a SQLite-backed Durable Object
class. `celld deploy`/`celld dev` build or pull the image with `docker` on
`PATH` (or `CELLD_DOCKER`); a deploy builds for `linux/amd64` by default
(`CELLD_CONTAINER_PLATFORM` overrides), uploads it to the bucket as
`deploy/images/<key>.tar`, and a node loads it into its own engine on first
use — no registry contact. `instance_type` maps to Cloudflare's CPU/memory
shapes (default `dev`); `max_instances` is a fleet-wide soft cap computed from
the shared node sample, so two nodes starting at once can briefly exceed it.

- **Every node that serves the class needs a Docker/Podman daemon.** A node
  without one refuses `start()` for that class but serves everything else.
  `CELLD_CONTAINER_RUNTIME` (or a per-class `runtime` key) selects the OCI
  runtime — use `runsc`/`kata` to give untrusted container code its own
  kernel; the default runtime is only a namespace boundary.
- celld fences every container's network with nftables via a one-shot
  privileged `celld-fence` image (shipped with the deployment): with
  `enableInternet: true` it reaches the Internet but not the node, other
  nodes, private ranges, or link-local addresses; `enableInternet: false`
  gets no route out. A node that can't install the fence starts no container.
- A container's disk is ephemeral: it survives an idle eviction
  (`setInactivityTimeout()`, default 10 min) but not a move to another node, a
  node restart, or a reset.
- `ctx.container` supports `start()` (`entrypoint`/`env`/`enableInternet`/
  `labels`; no `hardTimeout`), `monitor()`, `destroy()`, `signal()`,
  `getTcpPort()`, `exec()`, `setInactivityTimeout()`; no `inspect()`,
  snapshots, or outbound interception. `@cloudflare/containers` and
  `@cloudflare/sandbox` (on the `cloudflare/sandbox` image) run as published —
  see `examples/container` / `examples/sandbox`.
- `instance_type` takes `lite` (old name `dev`, the default: 1/16 CPU,
  256 MiB), `basic`, `standard-1` (alias `standard`) … `standard-4`. The node
  counts each running container's memory cap as committed memory, so a
  container-heavy node reports no headroom and sheds cells — size node memory
  for the largest instance type plus headroom.
- Config keys, `ctx.container`, and the security boundary can change without
  notice — this feature is explicitly experimental, not just opt-in.

## Python Workers (partial, v0.6.1+)

A Worker whose `main` is a `.py` file uses the Cloudflare Python Workers API
(`from workers import WorkerEntrypoint, Response`); the same project runs on
Cloudflare. celld runs **Pyodide 0.28.3 / CPython 3.13.2** (`pyodide_2025_0`
wheel ABI).

- Build: `uv run pywrangler sync` (vendors packages into `python_modules/`),
  then `celld dev .` / `celld deploy .`. The deploy bundles the Pyodide
  runtime, `python_modules/`, and `.py`/`.txt`/`.html`/`.sql`/`.bin`/`.wasm`
  files below `main`'s directory — nothing downloads at run time. The first
  build fetches Pyodide into `~/.cache/celld/pyodide-0.28.3`
  (`CELLD_PYTHON_RUNTIME_DIR` overrides; a populated dir allows offline
  builds). Still needs `esbuild`.
- Config: `compatibility_flags` must include `python_workers`;
  `compatibility_date` must be ≥ 2026-04-21 (or add the flags that date
  implies) and **before 2026-09-08** unless you add `no_python_workers_314` —
  from that date Cloudflare runs Python 3.14, which celld doesn't.
- Supported: `Default(WorkerEntrypoint).fetch`, `workers.Response` /
  `workers.fetch`, `self.env` bindings (tested with KV), `self.ctx.waitUntil`,
  pure-Python packages, `from js import ...`.
- **Not supported:** Durable Objects, Workflows, Cron, or Queue consumers in
  Python (deploy refuses them — keep those in JS), named entrypoints / RPC to
  Python, packages with compiled extensions (`.so`), `ssl` / `sqlite3` /
  `lzma` / OpenSSL `hashlib` algorithms, `pyodide.ffi.run_sync`, memory
  snapshots (so sync FastAPI handlers don't work).
- **Upgrade every node to ≥v0.6.1 before the first Python deployment.** The
  deployment needs the `python-workers-v1` feature but `celld deploy` doesn't
  check nodes: an older running node keeps serving its previous deployment
  (the fleet then serves two versions) and an older node that starts while a
  Python deployment is current exits.

## Secrets and Worker vars

celld has **no encrypted secret store** — nothing like `wrangler secret put`.
`celld dev` reads a local `.dev.vars` (or `.env` / `.env.local`) file for convenience (see *Develop
locally*), but a deployed fleet has no equivalent — `celld deploy` never reads
it. A deployed Worker's **vars** (`plain_text` bindings, read as `env.NAME`)
come from exactly one source, resolved on each node when it builds a
deployment: **`vars` in `wrangler.json`**, each entry baked into the
deployment manifest as a `plain_text` binding. celld requires every value to
be a **string** (no JSON/object vars).

**The manifest is written to the bucket in the clear.** `celld deploy` uploads
`deploy/<name>/<version>/manifest.json` to your S3/GCS/Azure bucket, so every
value in `wrangler.json` `vars` lands there unencrypted — and stays, because old
deployment versions are immutable. Anyone with bucket read access can read them.

**There is currently no way to keep a value out of the bucket for a deployed
fleet.** Through v0.4.1, a `CELLD_VARS_FILE` path and `CELLD_VAR_<NAME>` env
vars let you set per-node overrides, read locally at build time and never
uploaded — this skill used to point to them as the answer. **v0.5.0 removed
both.** celld now refuses to start if either is set (`crates/celld/env_vars.rs`
`REMOVED`/`REMOVED_PREFIX`), with the message "set `vars` in the Wrangler
config, or `.dev.vars` for `celld dev`" — but `.dev.vars` only feeds `celld
dev`; `celld deploy`'s `Options.vars` doc comment is explicit that `celld
deploy` supplies none, "so a local credential cannot reach a fleet."

- To keep a real secret out of the bucket, don't put it in a Worker var at
  all: fetch it at request time from an external secret manager/KMS your
  Worker code calls, or terminate it at your ingress/proxy instead of handing
  it to the Worker.
- Declaring the name in `wrangler.json` `vars` with an empty or placeholder
  value still goes to the bucket like any other value — it is not a safe
  fallback secret.

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

Each node runs the storage-contract test at startup — mandatory, no opt-out —
and stops if a required property is missing or the store silently ignores a
condition. `celld diagnose --read-only` runs the same probe on demand with a
credential that cannot write. celld reserves these bucket prefixes — the
application must not write under them: `probe/`, `cells/`, `nodes/`,
`node-cells/`, `fleet/`, `deploy/`, `deploy-blobs/`, `log/`, `wake/`,
`telemetry/`.

## Telemetry (off by default)

`CELLD_OTEL=1` records a span per request/event/outbound-fetch/cell-start and a
log record per `console.log`, writing Parquet to the fleet bucket under
`telemetry/` (query with DuckDB). Set `CELLD_OTEL` to a full collector base URL
(e.g. `http://collector:4318`) to send OTLP/HTTP instead — there's no separate
sink switch. Reads W3C `traceparent`. Details + DuckDB queries in
`reference/operations.md`.

## Common pitfalls

- **Single-node writes are slow** — no follower, so every write waits for the
  bucket. Run 2+ nodes when latency matters.
- **`celld dev` state is not migrated** across config changes; errors point at
  the stored value, not the config. Use `--clean`.
- **Some version upgrades are not rolling-safe** (v0.1→v0.2, v0.3→v0.4,
  v0.4.1→v0.5.0, and **v0.5.1→v0.6.0 with `fleet` durability** — a v0.6.0
  node refuses a v0.5.1 follower). Others are rolling but gate features until
  every node is upgraded (v0.6.0→v0.6.1: don't set
  `CELLD_LTX_RETENTION_SECS`, deploy a Python Worker, or raise
  `CELLD_MAX_ASSET_FILE_BYTES` above 25 MiB until all nodes run v0.6.1). Check
  `reference/operations.md` before upgrading a fleet.
- **Bucket usage grows with every activation** unless
  `CELLD_LTX_RETENTION_SECS` enables epoch GC (off by default).
- **The internal listener is unauthenticated** for most operator routes and has
  no TLS — a trusted private network is mandatory.
- **`wrangler.toml` and `routes` are rejected.** Convert config to JSON and
  configure routing in your ingress. (`define` and `rules` are accepted since
  v0.5.1, but a rule with `fallthrough` or a non-`**/*.ext` glob is refused.)
- **Bucket credentials = full fleet control.** Scope each credential to one
  fleet bucket.
- **`wrangler.json` `vars` go to the bucket in the clear** and stay in every
  past deployment version. celld has no secret store, and as of v0.5.0 no
  per-node override either (`CELLD_VARS_FILE`/`CELLD_VAR_*` were removed) — see
  *Secrets and Worker vars*.
- celld is **beta** (operator API alpha): pin `CELLD_VERSION`, keep operator
  tooling and binary on the same release, expect `CELLD_*` defaults to shift.
