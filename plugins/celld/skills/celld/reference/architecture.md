# celld architecture & guarantees

Distilled from celld v0.4.1: `docs/guarantees.md`, `docs/README.md`,
`docs/testing.md`, and `crates/celld/protocol.rs` in
<https://github.com/denoland/celld> (fetch those files for the primary text).

## The two promises

1. **One writer per cell.** Exactly one node serves a cell at a time, so two
   machines never write the same SQLite database.
2. **RPO=0.** celld does not answer a write until it survives a failure, so no
   write you were told succeeded is ever lost.

Both rest on the object store. Nodes elect no leader and keep no membership
list.

## What the bucket must provide

- **Conditional create** — write fails if the object exists.
- **Conditional overwrite** — write fails if the object changed since the read.
- **Read-after-write consistency** — a read after a successful write returns it.
- **Ranged reads** — a range request returns exactly those bytes.

Qualified: Amazon S3, Cloudflare R2, Tigris, Google Cloud Storage, Azure Blob
Storage. Release tests run against R2; the S3 path uses the same client and
headers. Azure was qualified 2026-08-18 (account key, VM managed identity, AKS
workload identity, single-node).

Not correct on: Backblaze B2, Hetzner Object Storage, DigitalOcean Spaces — no
required conditional writes, so two nodes can own one cell. A store can also
accept the conditional headers and ignore them (fails late and silently) —
hence the storage test.

Request dialect per provider: S3 uses `If-None-Match: *` / `If-Match` (etag);
`gs://` uses the Cloud Storage XML API with `x-goog-if-generation-match` +
OAuth; `az://` uses the same `If-` headers via Put Blob. The adapter treats
`AlreadyExists` and `Precondition` (HTTP 412 on Azure) as clean rejections and
keeps every other error ambiguous.

## The storage test

`celld diagnose` sends four conditional writes; two of the four **must** fail.
A store that accepts either failing write can't fence a cell, so celld names it
as the fault and exits non-zero. Each node runs the same writes before it
serves, then requests part of a second object and verifies the range and bytes.
Startup test makes ≤3 attempts on an unclear failure (starts with a warning if
all fail — a transient outage can clear), but stops immediately on an
unsupported conditional write / ranged read or a store that ignores a
condition or returns wrong bytes. `CELLD_STORAGE_PROBE=0` disables it; or run
`celld diagnose --read-only` with a non-writing credential.

## The supervisor

Run celld under a supervisor that restarts the process with **no attempt
limit** and waits **≥1 lease lifetime** between attempts (systemd, Docker
restart policy, Kubernetes). A fenced process exits; without a restart the
fleet loses that capacity until an operator intervenes.

## The mechanism

### Ownership record

Each cell has one ownership record in the bucket naming the owner node's
session and a fencing **epoch**. A node acquires a cell with a conditional
write — create when no record exists, compare-and-swap on the previous record
otherwise — and the bucket accepts one, so two nodes can't acquire the same
cell. **Every activation advances the epoch** (takeover and local wake alike),
so an epoch never has two writers.

### Epoch prefix (the data-path fence)

The replicator copies each cell's SQLite data (as LTX) to
`cells/<cell>/ltx/e<epoch>/` with plain unconditional PUTs. A node that lost
ownership can keep writing, but its writes land in a **superseded prefix** and
a restore selects the current lineage. (A tiering path may first combine
segments from many cells into a node bundle and drain them into per-cell
prefixes later.)

### Acknowledgement rule (RPO=0)

A gate holds each write response until a durability proof covers the write. A
read-only response waits the same way when the object has an uncovered
committed write; an error answer waits too (a thrown handler's message can
carry a value it read); a streaming body gets the rule per chunk.

- **Bucket proof:** after the upload, celld reads the ownership record once and
  acks only if it still names this node at this epoch. A partitioned node
  commits locally into its superseded prefix, but the ownership read shows the
  new owner, so it does not ack. The check reads the record, not a clock, so a
  paused process or skewed clock can't pass it.
- **Fleet proof** (`CELLD_DURABILITY=fleet`, default): the owner sends each
  write to one or two **followers** (the *ensemble*), each of which must
  `fsync`. A takeover seals the prior node-log session before it restores, so a
  stale owner can't complete another fleet proof. No ownership read needed.

### The ensemble needs two nodes

A node picks followers from *other* nodes, never itself. One follower is
enough, so a fleet needs **2+ running nodes** before any node can complete a
fleet proof. A node recruits up to two followers (3+ nodes ⇒ three copies of an
acknowledged write) and keeps acknowledging while one follower remains. A node
with no ensemble stays correct — it falls back to a bucket proof; the cost is
latency (a storage round trip vs a follower `fsync`).

### Takeover recovery gate

Each process session creates a conditional node-log record before its first
fleet-durable ack. A cold activation checks the prior owner's log records
before reading the bucket: absent ⇒ the session never acked past the bucket;
sealed ⇒ recovery completed; open/recovering ⇒ the activation runs recovery
(fence the record with CAS, seal reachable followers, upload retained
segments/bundles into per-cell prefixes, mark sealed) and cannot restore until
that finishes. A large dead node can hold recovery open for minutes; waiting
requests retry with backoff (`CELLD_RECOVERY_RETRY_MS` default 1000) and fail
only after the budget (`CELLD_RECOVERY_RETRIES` default 240). Restarting nodes
serve authenticated follower seal/tail requests before finishing their own
predecessor recovery, so nodes that restart together can still recover.

### Epoch-chain restore

A restore composes epoch prefixes containing LTX into one chain, newest first.
An epoch that opened with a whole-database snapshot starts the chain at
transaction one. An epoch that **paged in** continues its predecessor: its
prefix holds a marker at the successor of the cut it paged from, then its own
deltas. A link must end exactly at its successor's cut. celld no longer writes
an epoch seal object; a legacy `e<epoch>.seal.json` doesn't limit the chain.

**Paged restore** (`CELLD_LTX_PAGED=1`, default): a cell whose chain is ≥
`CELLD_LTX_PAGED_MIN_MB` (default 256) opens over a sparse local file through a
fault-in VFS that reads each page from the bucket on first use, instead of
downloading the whole chain. The file then fills in the background at
`CELLD_LTX_HYDRATE_MBPS` (default 16, one cell at a time per node). A smaller
chain is cloned whole. A paged cell's local file is a cache — not preserved as
an eviction snapshot, not used for a handoff snapshot.

### Self-fencing

Each node holds a bucket lease with an expiry, renewed after ⅓ of the lifetime
(`CELLD_TTL_MS`, default 10000). A renewal that doesn't reach the bucket
doesn't fence the node (it retries while the published expiry hasn't passed).
When the published expiry passes, or the lease record is gone or no longer
matches what the node published, the node **fences itself**: stops every active
cell, fails every uncompleted request, writes nothing to the bucket, logs a
`SELF-FENCE:` line, and exits code 3. Peers already read the lease as
dead/replaced and acquire the cells through the ownership records. The fenced
state is terminal — only a restart returns the node, via the same
cold-activation path a peer failure uses.

Log events: `node_lease_watchdog_fence` (expired), `node_lease_record_missing_fence`,
`node_lease_record_mismatch_fence`. `RUST_LOG=celld=info,store=debug` adds
per-request `node_lease_read` / `node_lease_write` events with an `outcome`
(`found` / `missing` / `applied` / `rejected` / `error`).

## The bucket contract types

`crates/celld/protocol.rs` is the durable interface between deployment tools
and celld — "these objects are the interface; nothing else is exchanged":

- `Manifest` — `deploy/<script>/<version>/manifest.json`: `version`,
  `script_name`, `main_module` (absent for asset-only), `do_classes`,
  `sqlite_classes`, `modules` (each `ModuleRef` carries a full SHA-256; a
  legacy 16-char prefix digest is also accepted), `assets`, `crons`,
  `queue_consumers`, `required_features`, and `raw_metadata` (wrangler's raw
  metadata verbatim).
- `required_features` gate — a node rejects a manifest needing a feature it
  doesn't support, up front (not at request time). Values:
  `assets-v1`, `cron-v1`, `d1-v1`, `kv-v1`, `queues-v1`, `sqlite-vec-v1`,
  `r2-v1`, `wasm-v1`, `workflows-v1`.
- `DeployPointer` — `deploy/current.json`: `{ script_name?, version, prefix,
  rollout: { percent } }`. Changing it *is* a deploy; nodes converge to it.
- `QueueConsumerAttachment` — `deploy/queues/<queue>/consumer.json`: the one
  deployment allowed to consume a fleet-global queue.
- `AssetIndex` / `AssetEntry` — `deploy/<script>/<version>/assets.json`;
  bodies live at `deploy-blobs/assets/sha256/<xx>/<sha256>`.
- `deployment_version(modules, metadata_json, asset_index)` — deployment
  identity: SHA-256 of sorted module contents + serialized metadata (+ asset
  index), truncated to 16 hex chars. **Cron expressions are deliberately not an
  input** — a schedule is configuration layered on a version, as Cloudflare
  models it.

## How celld is tested (for trust calibration)

- **Conformance:** every fixture runs on both `workerd` and `celld` on
  identical bytes; outputs must be equal. Ports of workerd's own DO contract
  tests + Web Platform Tests.
- **Specification:** the coordination protocol is modeled in TLA+ (against
  v0.1.0; found 4 bugs + a split-brain that lost an acked write, all fixed).
  Exhaustive at small size, hand-synced (deliberately not in CI).
- **Simulation:** the protocol is a pure decision core
  (`crates/logic`) with no I/O; a seeded scheduler injects store latency, CAS
  races, lost responses, clock drift, and crashes at every await. Millions of
  schedules; broken-protocol variants must trip the checkers.
- **Live fleets:** a permanent lab on standard VMs + a real bucket. Fault
  injection: `SIGKILL` mid-write-stream + delete local DB; freeze/unfreeze an
  owner; cut a node from the bucket; throttle the bucket to 429; kill a whole
  host. Across every run: zero lost acked writes, zero committed-state damage.
- Numbers celld publishes: warm resident request p50 ~1.1 ms / p99 ~7 ms, zero
  bucket ops. Bucket proof ~600 ms vs fleet proof ~25 ms (one lab fleet, not
  region-local). 10 nodes (4 vCPU / 8 GB) held 10,000 resident cells + 20,000
  WebSockets; killing 2 of 10 restored every cell elsewhere in ~11 s at the
  tail (with reserve headroom). One 8 GB node ≈ 1,000 resident cells ≈ $0.05
  per resident cell per month.
