# Operating a celld fleet

Distilled from celld v0.5.0: `docs/README.md`, `docs/security.md`,
`docs/telemetry.md`, `docs/limitations.md`, and `crates/celld/main/cli.rs` in
<https://github.com/denoland/celld> (fetch those, or run `celld --help`, for
the primary text). celld is alpha — the operator API and `CELLD_*` defaults can
change between releases; keep operator tooling and the binary on the same
release, and re-check anything version-sensitive against the project's release.

## The two listeners

| Listener | Flag / env | Default | Serves | Protection |
| --- | --- | --- | --- | --- |
| Public | `--listen` / `CELLD_ADDR` | `127.0.0.1:8080` | the deployed Worker; reserves only `/.well-known/celld/health` | ingress/proxy terminates TLS + authenticates users |
| Internal | `--internal-listen` / `CELLD_INTERNAL_ADDR` | `127.0.0.1:0` | peer protocol + operator API | **trusted private network / encrypted overlay only**; never public |

- `--advertise` / `CELLD_ADVERTISE` gives peers an address that reaches the
  internal listener. Explicit advertise ⇒ explicit internal listen. A
  non-loopback public listener ⇒ explicit internal listen. Literal public IP in
  advertise is rejected unless `--unsafe-public-advertise` /
  `CELLD_UNSAFE_PUBLIC_ADVERTISE=1` (which does **not** resolve hostnames or
  restrict the internal listener).
- celld can't verify that the advertised host/port routes to the internal
  listener — you must route it there and **not** to the public listener.
- Peer fetch/RPC/WebSocket traffic crosses the internal network as **plaintext
  HTTP** with only a protocol version. The fleet HMAC authenticates tunnel
  *establishment* and control/reserved-cell requests (with a clock limit +
  replay protection) but does not sign inner call bytes or encrypt anything.
  The private network is the confidentiality boundary.
- Forwarded headers: celld ignores `X-Forwarded-Host` / `X-Forwarded-Proto`
  unless `--trust-forwarded-headers` / `CELLD_TRUST_FORWARDED_HEADERS=1` (only
  with a trusted proxy that replaces both). Without a trusted proxy the `Host`
  header sets `request.url`'s host; celld falls back to `celld.local`. An
  application must not make an authorization decision on an unchecked hostname.
- Request body limit: 1 GiB default on the public listener and `/do/<ID>`
  (`CELLD_MAX_REQUEST_BODY_BYTES`); status 413 over the limit.

## Internal operator API (alpha, unauthenticated routes)

Reachable on the internal listener. A release can change paths/formats.

| Route | Effect |
| --- | --- |
| `GET /state` | Live counters: `owned_cells`, `occupied`, `capacity_waiting`, `activation_waiting`, `restoring`, `shedding`, `handed_off`, `rebalanced`, `rebalance_failed`, memory values, the deployment(s) served/draining, moving objects, and each resident object's deployment. `node_load` = the same sample the node lease publishes. |
| `POST /reload` | Adopt `deploy/current.json` now; also rebuilds unchanged code (applies a `CELLD_VARS_FILE` edit). Response reports a build failure. |
| `POST /shutdown` | Start the graceful ownership handoff. `?handoff=preserve` prepares a same-node reload and keeps ownership records. |
| `POST /rebalance/pause` / `POST /rebalance/resume` | Pause/resume ownership balancing. One paused lease pauses the **whole fleet**. |
| `/cell/<SCOPE>`, `/evict/<SCOPE>`, `/do/<ID>` | Resolve/activate, evict a resident cell, direct request to an ordinary Durable Object. `/do/<ID>` refuses reserved runtime classes (D1, Workflows, KV, Queues) — those use the HMAC-authenticated `/runtime/<SCOPE>`. |
| `GET /.well-known/celld/health` (public) | Boolean. 503 during a drain and before a joining node settles — gate rolling updates on it. Carries no utilization number. |

Operators must not call `/peer/*` and other reserved peer paths directly.

## Deployment rollout (in place, no restart)

Each node reads `deploy/current.json` every `CELLD_DEPLOY_POLL_S` (30 s) and
adopts the new deployment beside the running one, switching new requests over
in one step; in-flight requests finish on the old one. A build failure leaves
the current deployment serving. A resident Durable Object moves at a safe point
(no request/alarm running, no output awaiting durability, no regular WebSocket
open); one that reaches no safe point within `CELLD_DEPLOY_MAX_AGE_S` (60,
`0` = force at adoption) is forced — work cancelled, regular WebSockets closed
1012. During the window a request on one deployment can call a DO on the other,
so **adjacent versions must accept each other's calls**. Storage, epoch, and
hibernatable WebSockets are kept; the move touches no bucket.

## Worker vars and secrets

celld has no encrypted secret store (no `wrangler secret`). `celld dev` reads
a local `.dev.vars` file (`NAME=value` per line, quotes stripped) for
developer convenience and turns each entry into a Worker var, overriding a
same-named `vars` entry; only `celld dev` reads it, so it never reaches a
deployed fleet. A deployed fleet's Worker vars (`plain_text` bindings, read as
`env.NAME`) come from exactly one source: `vars` in `wrangler.json` — string
values only, baked into `deploy/<name>/<version>/manifest.json` at
deployment-build time.

`celld deploy` uploads the manifest to the bucket **unencrypted**, and past
versions are immutable — so anything in `wrangler.json` `vars` is readable by
any bucket reader forever. **v0.5.0 removed the only mechanism that used to
avoid this**: a per-node `CELLD_VARS_FILE` (dotenv-style path, read at build
time) and the `CELLD_VAR_<NAME>` family, neither ever uploaded. celld now
refuses to start if either is set (`crates/celld/env_vars.rs`, the
`REMOVED`/`REMOVED_PREFIX` table), and `celld deploy`'s `Options.vars` doc
comment confirms `celld deploy` itself reads no node-local override at all —
"so a local credential cannot reach a fleet." There is currently no supported
way to give a deployed fleet a value that isn't in the bucket manifest; fetch
secrets from an external store at request time, or keep them at the ingress
layer instead of as a Worker var.

## Ownership balancing

- Every node reads a shared fleet sample every 5 s
  (`CELLD_REBALANCE_INTERVAL_MS`; `0` disables balancing but the bucket-format
  gate still reads the sample). One node refreshes it from the node leases
  (`fleet/capacity-v1.json`); others read the result.
- Each node has an ownership target = fleet owned cells ÷ node weights
  (`CELLD_PLACEMENT_WEIGHT`, default CPU count). The node with the most owned
  cells per unit weight hands ≤ 32 **hibernated** cells per sample (a fixed
  ceiling — no longer an env var; the release ceiling `CELLD_RELEASES` still
  applies) to the peer furthest below its target (receiver fills to 2% below
  target so a stale sample can't cause a trade-back).
- Only a hibernated cell moves — one ownership-record write + one signed
  acquire; it stays hibernated on the new owner; parked hibernatable WebSockets
  close 1012 so clients reconnect. A **resident** cell moves only after idle
  eviction (`CELLD_IDLE_EVICT_S`) hibernates it — so a fleet without idle
  eviction balances only cells that hibernate on their own.
- **Idle eviction is revocable (since v0.5.0).** The node proves the cell
  durable, then stops its runtime, then writes a handoff snapshot — a request
  that arrives before the runtime stops cancels the eviction outright (cell
  keeps its runtime and epoch); one that arrives after the stop but before the
  handoff snapshot is written still gets the cell back at the *same* epoch
  (nothing durable has been published yet); only after that snapshot write
  does a new request wait for the cell to restart at a new epoch. A bucket
  proof makes the first (pre-stop) window longer than a fleet proof does. If
  the runtime doesn't stop within `CELLD_OPERATION_DEADLINE_MS`, the node
  keeps the cell resident and waits one more idle period before retrying —
  same wait after a revoked eviction gives the cell back.
- A draining node, and a node with a cold-activation backlog (`restoring` in
  its lease), receives no cells. The fleet moves nothing while any lease lacks
  a weight, so a rolling upgrade to a weight-aware version completes first.

## Memory-pressure shedding

- Threshold: 80% of available memory by default (`CELLD_MAX_RSS_MB`; `0`
  disables the threshold **and** the absolute cap). Applied to the greater of
  allocator-adjusted RSS and the allocator-adjusted active cgroup working set
  (`memory.current` − `inactive_file`, minus measured allocator slack).
- Absolute cap: 95% of available memory on the whole cgroup charge (process RSS
  if no cgroup charge). Logs a warning when it applies. A `CELLD_MAX_RSS_MB` ≥
  95% makes the cap the effective limit. If available memory is unreadable, the
  cap is 125% of an explicit threshold.
- Under pressure celld durably replicates and fences the least-recently-used
  idle cells, republishes them **unowned without resetting epochs**, and
  refuses new unowned cells. It never sheds a cell with active work or a live
  host WebSocket. Each limit releases at 80% of its own value, independently.
- `CELLD_PRESSURE_OWNERSHIP=release` (default) lets a shed cell move to another
  node; `sticky` keeps it cache-local so it wakes on the same node.
- V8 heap: 128 MB per isolate by default (`CELLD_V8_HEAP_LIMIT_MB`), matching a
  Cloudflare Durable Object. Each hibernatable WebSocket client holds heap
  state — ~50,000 clients at the default, ~512 MB for 100,000. Above 90% an
  isolate refuses a new hibernatable WebSocket (and stops materializing SQL
  result sets); it serves again below 75%. An idle isolate above that share
  gets a forced GC rather than a process restart.

## Autoscaler signals

celld does not scale itself. Two surfaces feed an external autoscaler:

- **Node lease** `nodes/<node>.json` (renewed at ⅓ of `CELLD_TTL_MS`, default
  10000) carries a load block: `owned_cells`, `placement_weight`,
  `resident_cells`, `host_websockets`, `rss_bytes`, `in_use_bytes`,
  `cpu_percent_x100`, `open_fds`, `pressured`, `memory_headroom`, `shed_cells`,
  `restoring`, `sampled_ms`. Readable with the bucket credentials alone.
- **`GET /state`** on the internal listener — the live counters above, plus
  (since v0.5.0) `remote_route_refreshes`: cached owner/capacity routes that
  expired and triggered a fresh lookup. It rises during normal lease renewal
  too (not just failures), so treat a spike relative to baseline, not the
  absolute count, as the signal; an adoption with no prior lease and an
  explicit invalidation don't count toward it.

`capacity_waiting > 0` (or a `pressured` lease) is the direct "add a node"
signal. Scale down only when every remaining node reports `memory_headroom` and
a small `restoring` backlog — a drain into a fleet with no spare capacity parks
cells dormant on survivors and every later activation must shed to run.

## Graceful shutdown & rollout

celld drains on `SIGTERM` / `SIGINT` (what `systemctl stop`, `docker stop`, a
k8s pod delete send). Health goes 503, new public requests get 503 with a
connection close, accepted requests finish, cells hand off in batches (reserve
a batch, stop new local routes, cancel firing alarms — the alarm runtime
records a retry so the successor runs it ≥once, prove the batch durable,
publish an L9 snapshot + verify a restore object exists, release ownership,
ask a peer to acquire — kept dormant). A fleet **drain token** in the bucket
serializes concurrent donors (one node hands off at a time).

**Set an orchestrator stop grace longer than `CELLD_SHUTDOWN_TOTAL_MS`**
(default 40000) — systemd `TimeoutStopSec`, k8s
`terminationGracePeriodSeconds` — or SIGKILL can land mid-handoff.

| Var | Default | Meaning |
| --- | --- | --- |
| `CELLD_RELEASES` | 128 | max complete handoffs in progress |
| `CELLD_ACTIVATIONS` | 8/CPU, ≥16 ≤128 | concurrent cold-cell activations / restore work |
| `CELLD_SHUTDOWN_TOTAL_MS` | 40000 | complete-process-stop bound; the orchestrator stop grace must exceed it |

celld now derives every other shutdown wait from `CELLD_SHUTDOWN_TOTAL_MS`
instead of taking them as separate settings: the fleet drain-token wait is ¾ of
the total bound (30000 ms at the default), and the handoff no-progress
interval (each successor acknowledgement restarts it) is ⅝ of the total bound
(25000 ms at the default, minimum 1 ms after rounding). A same-node preserve
uses the no-progress interval as its limit, since it has no successor acks.
`CELLD_SHUTDOWN_DRAIN_MS`, `CELLD_DRAIN_TOKEN_WAIT_MS`, and `CELLD_PACED_HANDOFF`
are **removed** — celld rejects them at startup (including an empty value or
one matching the old default). Set only `CELLD_SHUTDOWN_TOTAL_MS` if the
derived defaults don't fit your orchestrator grace.

A fresh process holds its first healthy response until the fleet is settled
(live node lease, no active donor, memory below every pressure low watermark,
acceptable ownership distribution). `CELLD_READY_FLEET_GATE_MS` (120000, `0`
disables) bounds the observation; celld emits `ready_gate_expired` at the
deadline but readiness stays closed — set an orchestrator rollout deadline so a
persistent capacity problem fails the rollout.

**Rolling update:** stop each node with SIGTERM, wait for its replacement to
report healthy (and `restoring=0` on every node via `celld diagnose`), then the
next. celld paces cell handoffs inside each shutdown and the first-readiness
gate paces against fleet recovery.

### Upgrades that are NOT rolling-safe — check before every upgrade

| Upgrade | Rolling? | Why |
| --- | --- | --- |
| v0.1.0 → v0.2.0 | **No** — stop all, then start all | v0.2.0 advertises the internal listener; v0.1.0-written ownership records name an address v0.1.0 peers can't follow. v0.2.0 also compacts replicated data into block objects a v0.1.0 reader can't restore. Never mix. |
| v0.2.1 → v0.3.0 | Yes, one at a time | Default durability moves `bucket` → `fleet`. A v0.3.0 node can't replicate to a v0.2.x peer (acks via the bucket until the peer upgrades). Do not start a v0.2.x binary after a node ran v0.3.0 unless its shutdown log has `node-log close: sealed epoch` — otherwise this downgrade can lose acked writes. |
| v0.3.0 → v0.4.0 | **No** — stop all, then start all | v0.4.0 moves every proxied cell call onto one tunneled plain-HTTP connection and the peer protocol refuses a different version. It also stores large KV values under the ownership epoch with an epoch-qualified row reference a v0.3.0 node can't read — a mixed fleet can make a committed KV value unavailable. |
| v0.4.0 → v0.4.1 | Yes, one at a time | v0.4.1 restores a large cell by paging; a paged epoch continues its predecessor's chain and a v0.4.0 node can't restore it. Each node publishes the bucket format it reads in its lease and only pages a takeover while every live lease reads that format; a mixed fleet clones like v0.4.0. Do not start a v0.4.0 binary after paging begins. |
| v0.4.1 → v0.5.0 | **No** — stop all, then start all | v0.5.0 replaces the alarm-discovery/wake bucket layout (format 2 under `wake/entries/` + `wake/retired/`) and a v0.4.1 node can't read it. See *Alarm/wake-format migration* below — the upgrade requires a full stop, a backup, and revoking the old binaries' write access before restart. |
| within a fleet, L1 compaction | — | `CELLD_LTX_COMPACTION=0` on every node of a mixed fleet until all can read v0.5.2 block objects — an old reader can't take a cell over after its first L1 publication. |

### Alarm/wake-format migration (v0.4.1 → v0.5.0)

v0.5.0 gives each alarm installation its own durable object under
`wake/entries/`/`wake/retired/` (format 2) instead of the v0.4.1 layout, so a
mixed fleet can't serve — v0.4.1 can't read the new entries and stale
`wake-format.json`/`wake-v2/`/`wake-retired-v2/` names are treated as
unsupported data:

1. Stop application traffic and deployment writers, then stop every v0.4.1
   node and its supervisor and wait for every node lease to expire.
2. Back up the stopped fleet's bucket and each node's local data directory —
   a follower disk can hold acknowledged writes the bucket doesn't have yet.
3. Revoke the old binaries' bucket write access (credential rotation or a
   deployment-system change) so they can't restart into the fleet; the format
   marker alone doesn't stop them.
4. Start v0.5.0 on every node with the same configuration, data, and
   addresses, and wait for the fleet to report healthy before resuming
   traffic. Startup migrates the format automatically (paged inventory,
   discovery seed per cell); a live v0.4.1 lease blocks the migration, and if
   a node stops mid-migration another starting node resumes it — the fleet
   serves nothing until the inventory completes. Alarms can fire late during
   the outage.

Do not roll back by starting a v0.4.1 binary against an upgraded bucket —
restore the pre-upgrade backup instead, which loses any writes made after it.

## Telemetry

`CELLD_OTEL=1` (off by default, costs nothing off). Records a span per Worker
request / cell event (fetch, alarm, RPC, WS message) / outbound `fetch()` /
cell start, and a log record per `console.log` (carries the trace + span id,
survives `await`). Reads W3C `traceparent` in, sends it out; a Worker→DO call
stays one trace; a malformed header starts a new trace.

| Var | Default | Effect |
| --- | --- | --- |
| `CELLD_OTEL` | `0` | `0` disables. `1` writes Parquet to the fleet bucket. A full HTTP(S) collector base URL (no query/fragment — celld appends `/v1/traces` and `/v1/logs`) sends OTLP/HTTP instead. There's no separate sink switch and celld does not read `OTEL_EXPORTER_OTLP_ENDPOINT`. |
| `CELLD_OTEL_BUCKET` | fleet bucket | alternate bucket, same endpoint/credentials |
| `CELLD_OTEL_RETENTION` | `30d` | `none` keeps forever (use your own lifecycle rules); the sweep runs at startup and every 6h after |
| `CELLD_OTEL_FLUSH_MS` / `CELLD_OTEL_FLUSH_BYTES` | `300000` / `5242880` | write a Parquet file at whichever limit is hit first (the byte limit is an estimate, so a batch can run slightly over); `5000` ms for a five-second batching interval (needs the compaction job) |
| `OTEL_TRACES_SAMPLER` (+ `_ARG`) | `parentbased_always_on` | standard sampler names; `traceidratio` for a fraction |
| `OTEL_EXPORTER_OTLP_HEADERS` / `OTEL_EXPORTER_OTLP_TIMEOUT` | unset / `10000` | comma-separated `name=value` headers / request timeout (ms) for the collector `CELLD_OTEL` names |
| `OTEL_SERVICE_NAME` | `celld` | resource service name |

`CELLD_OTEL_SINK` is **removed** — celld rejects it at startup. Copy a
collector's full base URL into `CELLD_OTEL` directly; keep `CELLD_OTEL=1` for
the fleet bucket.

Bucket sink layout: `telemetry/traces/<node>/<yyyy>/<mm>/<dd>/<hh>/<id>.parquet`
(logs under `telemetry/logs/...`). Schema `v0-unstable` — column names can
change; the version is in each file's object metadata. No metrics yet (spans
carry durations + queue waits). Query with DuckDB:

```sql
INSTALL httpfs; LOAD httpfs;
CREATE SECRET celld_telemetry (TYPE s3, KEY_ID '...', SECRET '...',
  ENDPOINT 's3.example.com', URL_STYLE 'path');   -- plain-HTTP endpoint also needs USE_SSL false
CREATE VIEW traces AS SELECT * FROM
  read_parquet('s3://YOUR-BUCKET/telemetry/traces/*/*/*/*/*/*.parquet');
SELECT name, duration_us, trace_id FROM traces ORDER BY duration_us DESC LIMIT 20;
```

Run compaction on a maintenance node (not a celld node): rewrite each past
hour's small files into one `compacted.parquet` sorted by `start_unix_us`,
delete the sources, never touch the current hour.

## Containers (experimental)

App-level details (config keys, `ctx.container`, fencing, package support) are
in `SKILL.md`. Operationally:

- Each node that serves a `containers` class needs its own Docker or Podman
  daemon reachable at `DOCKER_HOST` or the engine's default socket. A node
  without one refuses `start()` for that class but keeps serving every other
  class — there's no fleet-wide fallback.
- `celld deploy` uploads each built/pulled image once to
  `deploy/images/<key>.tar` (keyed by layer + config hash), plus the
  `celld-fence` image every container deployment needs; a node loads an image
  into its own engine the first time a cell of the class starts, so no node
  ever contacts a registry, and a redeploy of an unchanged image uploads
  nothing.
- `max_instances` is enforced from the shared node sample (the same one
  ownership balancing reads), not a coordinator — a node sums the fleet's
  published running-container counts and its own live count before a start.
  Two nodes racing a start at once can briefly exceed the cap and converge on
  the next sample refresh (`CELLD_REBALANCE_INTERVAL_MS`).
- A node stop destroys every container on it (disk included); plan container
  workloads around that the same way you'd plan around a Cloudflare container
  restart.

## Environment variables (primary set)

Run `celld -h` for the complete list including advanced tuning switches.
Booleans accept only `0` / `1`; an invalid value stops startup; an unset
variable takes its documented default.

**Storage / identity:** `CELLD_BUCKET` (`--bucket`, optional `/PREFIX`),
`S3_ENDPOINT` (`--endpoint`), `AWS_REGION` / `AWS_DEFAULT_REGION`,
`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN`,
`GOOGLE_APPLICATION_CREDENTIALS` / `GOOGLE_SERVICE_ACCOUNT_KEY` (gs://),
`AZURE_STORAGE_ACCOUNT_NAME` / `AZURE_STORAGE_ACCOUNT_KEY` /
`AZURE_AUTHORITY_HOST` / `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` /
`AZURE_FEDERATED_TOKEN_FILE` / `AZURE_STORAGE_USE_EMULATOR` (az://),
`CELLD_NODE` (node-session id, 1–128 ASCII `[A-Za-z0-9._-]`, not `.`/`..`),
`CELLD_WATCH` (local SQLite/replication dir), `CELLD_ESBUILD`.

**Listeners:** `CELLD_ADDR`, `CELLD_INTERNAL_ADDR`, `CELLD_ADVERTISE`,
`CELLD_UNSAFE_PUBLIC_ADVERTISE`, `CELLD_TRUST_FORWARDED_HEADERS`.

**Durability:** `CELLD_DURABILITY` (`fleet` default / `bucket`),
`CELLD_LTX_DURABILITY_TIMEOUT_SECS` (10; one proof's budget, from upload start;
≤6 budgets when queued behind other cells), `CELLD_OPERATION_DEADLINE_MS`
(15000). `CELLD_OUTPUT_GATE` is **removed** — celld always waits for the
configured durability proof; celld rejects the variable, including `1` or an
empty value.

**Placement / capacity:** `CELLD_MAX_RESIDENT_CELLS`, `CELLD_IDLE_EVICT_S`,
`CELLD_PLACEMENT_WEIGHT`, `CELLD_REBALANCE_INTERVAL_MS` (5000, `0` disables),
`CELLD_PRESSURE_OWNERSHIP` (`release` / `sticky`), `CELLD_ACTIVATIONS`,
`CELLD_MAX_CELL_REQUESTS` (64), `CELLD_MAX_REQUEST_BODY_BYTES` (1 GiB).
`CELLD_REBALANCE_BATCH_CELLS` (fixed at 32) and `CELLD_EVICTIONS` (fixed at 4
concurrent evictions) are **removed**.

**Memory:** `CELLD_MAX_RSS_MB` (80%, `0` disables threshold + cap),
`CELLD_V8_HEAP_LIMIT_MB` (128), `CELLD_LOCAL_CACHE_MAX_BYTES` (2 GiB, `0`
disables), `CELLD_ASSET_CACHE_DIR`, `CELLD_ASSET_CACHE_BYTES` (512 MiB).

**Deploy / lease:** `CELLD_DEPLOY_POLL_S` (30), `CELLD_DEPLOY_MAX_AGE_S` (60),
`CELLD_TTL_MS` (10000). `CELLD_VARS_FILE` and `CELLD_VAR_*` are **removed** in
v0.5.0 — see *Worker vars and secrets*. `CELLD_STORAGE_PROBE` is **removed** —
the startup storage-contract test can no longer be skipped.

**Recovery / shutdown:** `CELLD_RECOVERY_RETRY_MS` (1000),
`CELLD_RECOVERY_RETRIES` (240), `CELLD_RELEASES` (128),
`CELLD_SHUTDOWN_TOTAL_MS` (40000), `CELLD_READY_FLEET_GATE_MS` (120000).
`CELLD_SHUTDOWN_DRAIN_MS`, `CELLD_DRAIN_TOKEN_WAIT_MS`, and
`CELLD_PACED_HANDOFF` are **removed** — see *Graceful shutdown & rollout*.

**LTX replication:** `CELLD_LTX_TRUNCATE_PAGES` (128, 512 KiB WAL cap; Queues
never truncate; `0` disables), `CELLD_LTX_COMPACTION` (`1`; `0` on a mixed
fleet until all nodes read block objects), `CELLD_LTX_COMPACTION_MIN_TXIDS`
(256), `CELLD_LTX_COMPACTION_MIN_MB` (32, ≤64), `CELLD_LTX_COMPACTIONS` (2),
`CELLD_LTX_PAGED` (`1`; `0` on a mixed fleet until all nodes run ≥v0.4.1),
`CELLD_LTX_PAGED_MIN_MB` (256), `CELLD_LTX_HYDRATE_MBPS` (16). Each L1
compaction merges ≤256 source objects with a 64 MiB buffer budget, spilling to
temporary files in the cell's local LTX directory for an oversized source.

**Logs / queues:** `CELLD_LOG_PIPELINE` (4), `CELLD_LOG_HEDGE_MS` (adaptive;
`0` disables), `RUST_LOG` (default `info`). `CELLD_LOG_CAPTURE_WORKERS` (fixed
at 8), `CELLD_LOG_GROUP_COMMIT_MS` (fixed at 1 ms), and
`CELLD_QUEUE_PRODUCER_GROUP_MS` (fixed at 4 ms) are **removed**.
`CELLD_LOG_BUNDLE` is also **removed** — fleet durability now always tiers
multiple cells' changes into shared bucket objects (a node falls back to
per-cell uploads only when the bucket itself must prove a write); this is
transparent to `CELLD_DURABILITY=bucket`, which still needs its own bucket
proof per acknowledged write, and recovery reads both old per-cell objects and
the new bundles.

**Alarms / timers:** `CELLD_ALARM_RESIDENT_MS`, `CELLD_WAKER_TICK_MS`,
`CELLD_FETCH_TIMEOUT_S`, `CELLD_HANDLER_BUDGET_S`, `CELLD_TOKIO_THREADS`.

**Containers (experimental):** `CELLD_DOCKER` (engine CLI, default `docker`),
`CELLD_CONTAINER_PLATFORM` (build target for `celld deploy`, default
`linux/amd64`), `CELLD_CONTAINER_RUNTIME` (node-wide OCI runtime, e.g. `runsc`
/ `kata`; a `containers` entry's own `runtime` key overrides it),
`CELLD_CONTAINER_DNS` (extra public resolvers for a container under a named
runtime; `1.1.1.1` is always included). See *Containers* above.

`CELLD_WORKER_LOADER`, `CELLD_MAX_LOADED_WORKERS`, `CELLD_AI_BINDING`, and
`CELLD_AI_URL` are **removed**. Declare a Worker Loader with the
`worker_loaders` config key in `wrangler.json` instead of an env var — Dynamic
Workers is no longer experimental. The `env.AI` HTTP adapter is gone entirely;
call an AI provider directly from application code.

## Fleet limitations (alpha)

- One application per fleet — no account service, multi-tenant scheduler, or
  managed ingress. Not safe for hostile multi-tenant use.
- Durable state only in an S3-compatible / GCS / Azure bucket (`celld dev` uses
  a local store that a node/operator command can't select).
- Balancing counts cells by node weight — it doesn't measure per-cell CPU/memory
  and moves only hibernated cells.
- No TLS termination on either listener. Peer traffic is plaintext HTTP.
- Azure identity support is public-cloud only; a managed identity on Azure App
  Service / Container Apps doesn't work (use a workload identity or account key).
- Installer binaries: Linux x86-64, Linux ARM64, Apple Silicon. **No Windows.**
- An outbound Durable Object WebSocket keeps its cell resident and closes if the
  cell moves — store the connection intent and reconnect.
- Containers are experimental and need a Docker/Podman daemon per node that
  serves the class; `max_instances` is a fleet-wide cap enforced from a
  sampled node count, not a coordinator, so it can be briefly exceeded.
