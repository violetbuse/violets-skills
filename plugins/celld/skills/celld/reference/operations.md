# Operating a celld fleet

Distilled from celld v0.4.1: `docs/README.md`, `docs/security.md`,
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

celld has no encrypted secret store (no `wrangler secret`, no `.dev.vars`).
Worker vars (`plain_text` bindings, read as `env.NAME`) resolve on each node at
deployment-build time from three sources, later wins:

1. `vars` in `wrangler.json` — string values only; become `plain_text` bindings
   in `deploy/<name>/<version>/manifest.json`.
2. `CELLD_VARS_FILE` — dotenv-style file on the node: `NAME=value` per line,
   `#` comments and blank lines skipped, one surrounding `'`/`"` pair stripped,
   no escapes or multi-line values.
3. `CELLD_VAR_<NAME>` env vars on the node.

`celld deploy` uploads the manifest to the bucket **unencrypted**, and past
versions are immutable — so anything in `wrangler.json` `vars` is readable by
any bucket reader forever. To pass a secret without writing it to the bucket,
keep it out of `vars` and set it per node via `CELLD_VARS_FILE` / `CELLD_VAR_*`;
those are never uploaded. Deliver the file/env to **every** node out of band
(systemd `EnvironmentFile=`, mounted secret, config manager) — a var on only
some nodes makes behaviour depend on which node served the request. Edit the
file and `POST /reload` (rebuilds unchanged code) to apply with no restart.

## Ownership balancing

- Every node reads a shared fleet sample every 5 s
  (`CELLD_REBALANCE_INTERVAL_MS`; `0` disables balancing but the bucket-format
  gate still reads the sample). One node refreshes it from the node leases
  (`fleet/capacity-v1.json`); others read the result.
- Each node has an ownership target = fleet owned cells ÷ node weights
  (`CELLD_PLACEMENT_WEIGHT`, default CPU count). The node with the most owned
  cells per unit weight hands ≤ `CELLD_REBALANCE_BATCH_CELLS` (32) **hibernated**
  cells per sample to the peer furthest below its target (receiver fills to 2%
  below target so a stale sample can't cause a trade-back).
- Only a hibernated cell moves — one ownership-record write + one signed
  acquire; it stays hibernated on the new owner; parked hibernatable WebSockets
  close 1012 so clients reconnect. A **resident** cell moves only after idle
  eviction (`CELLD_IDLE_EVICT_S`) hibernates it — so a fleet without idle
  eviction balances only cells that hibernate on their own.
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
- **`GET /state`** on the internal listener — the live counters above.

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
| `CELLD_SHUTDOWN_DRAIN_MS` | 25000 | max interval with no completed handoff (each ack restarts it) |
| `CELLD_SHUTDOWN_TOTAL_MS` | 40000 | complete-process-stop bound |
| `CELLD_DRAIN_TOKEN_WAIT_MS` | 30000 (`0` disables) | wait for the fleet drain token before proceeding without it; must be ≤ ¾ of the total bound |

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
| within a fleet, L1 compaction | — | `CELLD_LTX_COMPACTION=0` on every node of a mixed fleet until all can read v0.5.2 block objects — an old reader can't take a cell over after its first L1 publication. |

## Telemetry

`CELLD_OTEL=1` (off by default, costs nothing off). Records a span per Worker
request / cell event (fetch, alarm, RPC, WS message) / outbound `fetch()` /
cell start, and a log record per `console.log` (carries the trace + span id,
survives `await`). Reads W3C `traceparent` in, sends it out; a Worker→DO call
stays one trace; a malformed header starts a new trace.

| Var | Default | Effect |
| --- | --- | --- |
| `CELLD_OTEL` | `0` | `1` enables |
| `CELLD_OTEL_SINK` | `bucket` | `bucket` = Parquet under `telemetry/`; `otlp` = OTLP/HTTP protobuf to a collector |
| `CELLD_OTEL_BUCKET` | fleet bucket | alternate bucket, same endpoint/credentials |
| `CELLD_OTEL_RETENTION` | `30d` | `none` keeps forever (use your own lifecycle rules) |
| `CELLD_OTEL_FLUSH_MS` / `CELLD_OTEL_FLUSH_BYTES` | `300000` / `5242880` | write a Parquet file at whichever limit is hit first; `10000` ms for near-live (needs the compaction job) |
| `OTEL_TRACES_SAMPLER` (+ `_ARG`) | `parentbased_always_on` | standard sampler names; `traceidratio` for a fraction |
| `OTEL_EXPORTER_OTLP_ENDPOINT` (+ `_HEADERS`, `_TIMEOUT`) | `http://localhost:4318` | for `otlp` |
| `OTEL_SERVICE_NAME` | `celld` | resource service name |

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
`CELLD_OUTPUT_GATE` (`0` removes the durability wait — accepts loss),
`CELLD_LTX_DURABILITY_TIMEOUT_SECS` (10; one proof's budget, from upload start;
≤6 budgets when queued behind other cells), `CELLD_OPERATION_DEADLINE_MS`
(15000).

**Placement / capacity:** `CELLD_MAX_RESIDENT_CELLS`, `CELLD_IDLE_EVICT_S`,
`CELLD_PLACEMENT_WEIGHT`, `CELLD_REBALANCE_INTERVAL_MS` (5000, `0` disables),
`CELLD_REBALANCE_BATCH_CELLS` (32), `CELLD_PRESSURE_OWNERSHIP`
(`release` / `sticky`), `CELLD_ACTIVATIONS`, `CELLD_EVICTIONS` (4),
`CELLD_MAX_CELL_REQUESTS` (64), `CELLD_MAX_REQUEST_BODY_BYTES` (1 GiB).

**Memory:** `CELLD_MAX_RSS_MB` (80%, `0` disables threshold + cap),
`CELLD_V8_HEAP_LIMIT_MB` (128), `CELLD_LOCAL_CACHE_MAX_BYTES` (2 GiB, `0`
disables), `CELLD_ASSET_CACHE_DIR`, `CELLD_ASSET_CACHE_BYTES` (512 MiB).

**Deploy / lease:** `CELLD_DEPLOY_POLL_S` (30), `CELLD_DEPLOY_MAX_AGE_S` (60),
`CELLD_TTL_MS` (10000), `CELLD_VARS_FILE` / `CELLD_VAR_*` (per-node Worker var
overrides, never uploaded to the bucket — see *Worker vars and secrets*),
`CELLD_STORAGE_PROBE` (`0` skips startup storage test).

**Recovery / shutdown:** `CELLD_RECOVERY_RETRY_MS` (1000),
`CELLD_RECOVERY_RETRIES` (240), `CELLD_RELEASES` (128),
`CELLD_SHUTDOWN_DRAIN_MS` (25000), `CELLD_SHUTDOWN_TOTAL_MS` (40000),
`CELLD_DRAIN_TOKEN_WAIT_MS` (30000), `CELLD_READY_FLEET_GATE_MS` (120000).

**LTX replication:** `CELLD_LTX_TRUNCATE_PAGES` (128, 512 KiB WAL cap; Queues
never truncate; `0` disables), `CELLD_LTX_COMPACTION` (`1`; `0` on a mixed
fleet until all nodes read block objects), `CELLD_LTX_COMPACTION_MIN_TXIDS`
(256), `CELLD_LTX_COMPACTION_MIN_MB` (32, ≤64), `CELLD_LTX_COMPACTIONS` (2),
`CELLD_LTX_PAGED` (`1`; `0` on a mixed fleet until all nodes run ≥v0.4.1),
`CELLD_LTX_PAGED_MIN_MB` (256), `CELLD_LTX_HYDRATE_MBPS` (16).

**Logs / queues:** `CELLD_LOG_CAPTURE_WORKERS` (8), `CELLD_LOG_PIPELINE` (4),
`CELLD_LOG_GROUP_COMMIT_MS` (1), `CELLD_QUEUE_PRODUCER_GROUP_MS` (4),
`CELLD_LOG_HEDGE_MS` (adaptive; `0` disables), `RUST_LOG` (default `info`).

**Alarms / timers:** `CELLD_ALARM_RESIDENT_MS`, `CELLD_WAKER_TICK_MS`,
`CELLD_FETCH_TIMEOUT_S`, `CELLD_HANDLER_BUDGET_S`, `CELLD_TOKIO_THREADS`.

**Experimental:** `CELLD_WORKER_LOADER` (Worker Loader / Code Mode binding
name), `CELLD_MAX_LOADED_WORKERS` (256), `CELLD_AI_BINDING` / `CELLD_AI_URL`
(env.AI HTTP adapter).

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
