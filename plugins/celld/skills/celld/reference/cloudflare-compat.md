# celld ↔ Cloudflare Workers compatibility

Distilled from celld v0.6.2: `docs/cloudflare-compat.md`, `docs/services/*.md`
(one page per service), and `docs/wasm.md` in
<https://github.com/denoland/celld> (fetch those for the primary text).
celld implements the Cloudflare Workers APIs and **must reject an
unsupported configuration or API at deployment or first use** — an unsupported
feature that doesn't error is a defect. The notes below are only the celld gaps
and differences.

- **Yes** — an app that uses the API runs on celld.
- **Partial** — a large part is missing; an app depending on that part won't run.
- **Experimental** — can change without notice.
- **No** — not implemented.

## Services

| Service | Status | Key notes |
| --- | --- | --- |
| Workers | Yes | No custom domain / TLS termination — do it at ingress. Every export of the main module must be a handler object or a class, as in workerd — `export const X = "..."` fails the Worker's start (v0.6.0). Keep state in bindings, not module scope (isolates retire at any time). No Workers AI: celld rejects `CELLD_AI_BINDING`/`CELLD_AI_URL` and an `ai` config declaration — call a provider directly. |
| Python Workers | Partial | v0.6.1+: `fetch` handlers on Pyodide 0.28.3 / CPython 3.13.2. No Python Durable Objects / Workflows / Cron / Queue consumers, no RPC to Python, no compiled-extension packages. Needs `python_workers` flag and a `compatibility_date` in [2026-04-21, 2026-09-08) (or the equivalent flags / `no_python_workers_314`). See `SKILL.md` § *Python Workers*. |
| Durable Objects | Yes | SQLite storage, alarms, hibernating WebSockets. Pending I/O (a timer, a subrequest) stays active after the handler returns with no `ctx.waitUntil()` needed. A `migrations` entry accepts only `tag` + `new_sqlite_classes` — a class rename/delete/transfer stops the deploy. No jurisdiction (`newUniqueId({ jurisdiction })` / `namespace.jurisdiction()` throw). `idFromName()` keys include the **script name**, so renaming the Worker reaches new, empty objects. See the storage-API notes below. |
| Durable Object Facets | Yes | See *Facets* below. |
| Containers | Experimental | A `containers` entry (`class_name`, `image`, `name`, `instance_type`, `max_instances`, plus celld-only `runtime`) binds a Docker/Podman image to a SQLite-backed DO class — see *Containers* below. |
| Static assets | Yes | No edge cache (512 MiB local disk cache, `CELLD_ASSET_CACHE_BYTES`), no compression (put a compressing proxy in front). `assets` accepts only `directory`, `binding`, `html_handling`, `not_found_handling`, `run_worker_first` (≤100 patterns; the exact `/` pattern is accepted since v0.6.0 and matches only the root). `_headers` / `_redirects` supported but can't set `connection`/`content-length`/`transfer-encoding`; 100 KiB each. ≤20,000 assets, 1 GiB total, 25 MiB per file — `CELLD_MAX_ASSET_FILE_BYTES` (v0.6.1) raises it but must match on the deploying machine, the managed deployment agent, and every node (read once per process). `.assetsignore` is refused. `ETag` = strong SHA-256; `cache-control: public, max-age=0, must-revalidate`; no `Last-Modified` / `If-Modified-Since`. |
| Cron Triggers | Yes | One handler per occurrence across the whole fleet. One missed occurrence re-run after downtime. One handler at a time per script; a failure retries after 4 s, doubling, up to six failures or the next occurrence, unless it calls `noRetry()`; an ownership move loses a pending retry. Rejects descending ranges (`SAT-SUN`), a list containing `*`, and a step wider than its field. Weekdays are 1–7 from Sunday. A service-binding target can't run its own crons. |
| KV | Yes | Reads are strongly consistent (they reach the namespace's owner cell). No edge cache (`cacheTtl` ignored, `cacheStatus` null). `put()` accepts a `ReadableStream` (v0.5.1). Values > 1 MiB need a fleet bucket. **One writer per namespace** — add namespaces to scale writes; each read is one cell dispatch. |
| Queues | Yes | **One writer per queue**; one consumer script. Owner admits ≤256 concurrent producer calls (refuses extra — producer retries). At-least-once, no ordering guarantee; `message.id` is stable across redelivery. No automatic backoff — compute the delay from `message.attempts`. Messages retained 4 days (not configurable). No pull consumers, no Queues HTTP API, no dashboard controls / R2 event notifications / event subscriptions. |
| D1 | Yes | Binding result ≤ 100,000 rows or 32 MiB. Invalid UTF-8 in `TEXT` decodes as U+FFFD (v0.6.0; stored bytes unchanged) — use `BLOB` for arbitrary bytes. One writer per database. `prepare()` takes exactly one statement. No `BEGIN`/`SAVEPOINT`/`ATTACH`/`VACUUM`/temp tables in app SQL; `dump()` throws; no read replicas (`withSession()` → primary, bookmark `celld:primary`); no REST API / Time Travel. A typo'd database name silently opens an empty DB. |
| Workflows | Yes | celld retains a terminal instance 30 days by default; `retention` can request up to 30 days. `locationHint` is accepted but fleet ownership picks the actual location. Replays `run()` from the start — code outside a step runs again; a crash after a step side effect can re-run the callback. Non-step work can't stay pending > 60 s. Step result / event payload / params ≤ 1 MiB each. `pause()`/`resume()`/`restart()` supported; rollback, sensitive/`ReadableStream` step results, `schedules`/`limits`/cross-script `script_name`, the REST API, and `wrangler workflows` are not. |
| R2 | Yes | Uses the fleet bucket under `r2/<bucket_name>/`. **Since v0.6.0 empty key segments are kept** — `a/b`, `/a/b`, `a//b`, `a/b/` are four objects (an object ≤v0.5.1 wrote as `photos/` stays at `photos`). Some keys are stored percent-encoded, so `list()` sorts / compares `startAfter` by the encoded form (a celld `cursor` is fine). `version` = content ETag. No `ssecKey`, no `jurisdiction`, no public/presigned URLs. Conditional write can't use a streamed body > 8 MiB. Multipart: no checksum on `createMultipartUpload()`, can't resume on another node or after a restart, can't replace a stored part, out-of-order parts ≤ 256 MiB memory. Objects written by other tools are readable (user metadata → `customMetadata`). Operate with `celld r2` (no running node needed). |
| Dynamic Workers (Worker Loader / Code Mode) | Yes | Declared with the `worker_loaders` config key (`binding` only — `tails`/`limits` there stop the deploy; set them in `WorkerCode`). See *Dynamic Workers* below. |
| Workers AI / Vectorize / Hyperdrive / Browser Rendering / Email Workers | No | — |

### Dynamic Workers

- `WorkerCode` **requires `compatibilityDate`** (v0.6.0, as workerd). A wasm
  entry in `modules` must be `{ wasm: bytes }` — bare bytes are refused.
  Relative imports resolve from the importing module's name (so `dir/a.js` can
  import `./b.js`). Module sources total ≤ 64 MiB by default
  (`CELLD_MAX_DYNAMIC_WORKER_CODE_BYTES`, v0.6.1, per node).
- `WorkerCode.limits` / `getEntrypoint(name, { limits })` enforce `cpuMs` and
  `subRequests` (v0.5.1; the lower value wins); `allowExperimental` is
  rejected. `WorkerCode.tails` accepts Service Binding Fetchers, each receiving
  one event per fetch invocation (request metadata, response status, console
  logs ≤256 KiB, uncaught exception, outcome); a Tail Worker failure doesn't
  change the response.
- Lifetime: `get(id, getCode)` memoizes per loader binding, per isolate, per
  node — not guaranteed reuse. Since v0.6.1 a loaded Worker is released once
  no stub/entrypoint/class/running facet references it (GC timing, so a later
  `get()` may re-run `getCode`); `stub.dispose()` releases it explicitly.
  A fetch to a loaded Worker honors the caller's `AbortSignal` (v0.6.2).
- Process-wide limit: 256 live loaded Workers, ≤255 slots per script
  generation. `getDurableObjectClass()` supports only `props` (≤1 MiB
  structured-clone); `WorkerCode.env` takes structured-clone values and
  Service Binding capabilities (≤1 MiB total). A `globalOutbound` Fetcher
  can't `connect()` or use a WebSocket; a loaded entrypoint can't transfer to
  another Worker; no awaitable/pipelined properties.
- Isolation: the loaded isolate shares the loader's process — it's a scoping
  boundary, not a V8-escape-proof one (use a container under `runsc`/`kata`
  for a kernel boundary). It gets no deployment bindings/vars and no Worker
  Loader. celld's host functions are passed to internal scripts as
  parameters, never on `globalThis`, so with `globalOutbound: null` the
  loaded Worker reaches the host only through capabilities in its `env`.

### Facets

- Each facet has **its own SQLite file and replication stream** under the root
  DO's bucket prefix (v0.6.0; v0.5.1 facets migrate on first open). A facet
  write commits in the facet's database, so a root-transaction rollback does
  **not** undo a facet call inside it — facet and root never commit
  atomically. A facet's outbound effects / replies wait for the facet stream's
  proof and pass the root cell's output gate.
- Class source: `worker.getDurableObjectClass("App")` from a Worker Loader, or
  (v0.6.0) `ctx.exports.App` for an exported `DurableObject` class **without**
  a storage migration (runs in the root's isolate). A DO-binding class, or a
  class with a storage migration, can't be a facet class. A loaded class may
  be a plain class with a `(state, env)` constructor (v0.6.2); its RPC methods
  need `js_rpc`. `ctx.exports.App({ props })` passes startup props (v0.6.1).
- `ctx.facets.get(name, cb)` / `abort()` / `delete()` (recursive); no
  `clone()`. Name ≤ 256 bytes and alone selects the database; depth ≤ 4
  including the root. `id` sets `ctx.id` (a named id keeps its name).
- A facet **cannot set an alarm** (`setAlarm()` throws — schedule in the root).
- WebSockets work in facets (v0.6.2), accepted or outbound, but **don't
  hibernate** — an open facet socket keeps the root cell resident. abort/delete
  closes them with 1001; drain/move/deploy swap with 1012.
- `ctx.exports` holds `default` and each entrypoint; `fetch()` on its stubs
  sends an HTTP request (v0.6.0).

## Durable Object storage API — celld specifics

- `storage.sync()` resolves only after the object store or the fleet ensemble
  holds every write committed before the call. Rejects (and celld resets the
  object, restoring the proven history) when it can't prove durability within
  the shorter of `CELLD_LTX_DURABILITY_TIMEOUT_SECS` (10, from upload start)
  and `CELLD_OPERATION_DEADLINE_MS` (15000). The reset can stop the object
  before the handler sees the rejection — don't depend on the rejection to
  recover. Also rejects while a `transaction()` is open, and after
  `ctx.abort()` or a failed `blockConcurrencyWhile()`. Without an object store,
  it resolves after the local commit (there's no `CELLD_OUTPUT_GATE` opt-out).
- Outside an explicit transaction, a SQL write cursor must finish (be fully
  read) before a response, an outbound effect, or `storage.sync()` — an
  unfinished `RETURNING` cursor holds uncommitted writes, and celld rejects
  that output with an error. A read cursor can stay open.
- **A handler that fails after it writes answers only after the write is
  durable** — celld holds the error behind the same output gate as a success
  (a later request could read that write). A handler that fails without a
  write of its own waits for every write the object holds to be durable (its
  thrown message can carry a value it read); a failure celld raises itself
  carries no object value and leaves at once.
- A DO `stub.fetch()` whose handler throws **rejects** on the caller — on the
  owner node and, since v0.6.2, also when forwarded from another node (it used
  to arrive as a 500 response), and the handler doesn't run again.
- `transaction()` callback runs under the input gate; a callback running
  > 30 s resets the object and rolls back (the `blockConcurrencyWhile()`
  limit). A throwing callback rolls back and the object continues.
  `transactionSync()`'s callback receives **no argument** (v0.6.0, as
  workerd), and nested `transactionSync()` works (a failed nested transaction
  discards only its own writes).
- The synchronous `ctx.storage.kv.list()` iterator reads one entry per step
  and no longer blocks later writes (v0.6.1); it can observe changes to keys
  it hasn't returned yet, and a new `kv.list()` call invalidates the previous
  iterator.
- WebSocket handlers on one socket **start in arrival order without waiting
  for the previous handler to finish** (v0.6.1) — an incoming message can
  cancel work an earlier handler awaits. Frames are delivered while
  `webSocketMessage()` runs (v0.6.0). Hibernatable sockets keep send order
  across handlers and RPC methods.
- A re-armed alarm fires on time even if the handler left timers pending
  (v0.6.0). `setAlarm()` succeeds only once a durable wake entry covers it.
- `SqlStorage.Cursor.toArray()` errors near the V8 heap limit.
- An RPC stub can't cross an isolate boundary. An outbound WebSocket doesn't
  survive the object moving nodes.

## Runtime APIs

| API | Status |
| --- | --- |
| Fetch / Request / Response / Headers, Bindings, Context, Handlers, RPC, Streams, Encoding, WebSockets, Web Crypto, Web standards, WebAssembly, Performance & timers, Console, HTMLRewriter, TCP sockets, EventSource, MessageChannel | Yes |
| Node.js compatibility | Partial |
| Cache | Partial (always-miss) |
| BroadcastChannel | No (constructor throws; class is defined so a bundle can reference it) |

Selected differences:

- **Fetch:** no `cache` option; celld strips `Content-Length` from a Worker
  response (kept for `HEAD`). An inbound `Request`'s `cf` object has no
  Cloudflare edge fields (no geolocation/colo/TLS metadata). A remote DO call
  streams the request body and can't retry after transmission starts; it waits
  at most `CELLD_OPERATION_DEADLINE_MS` for a rejected owner generation to
  change. `Headers` accepts values above U+00FF and decodes response values as
  UTF-8 (v0.6.0).
- **Context / handlers:** `passThroughOnException()` has no effect;
  `ctx.facets` exists only inside a DO; no `tail` or `email` handlers. `self`
  is defined in Worker, DO, and Worker Loader isolates (v0.6.2).
- **RPC:** an RPC stub can't cross an isolate boundary. An `AbortSignal` in a
  DO RPC call passes through only **on the same node**. A remote RPC retries
  only when the failed peer attempt didn't start the method — use a stable
  operation id for other retries.
- **Streams:** an unclaimed/inactive HTTP stream expires after 60 s (each
  successful operation restarts the 60 s); an expired/unknown stream reports
  an error, not EOF.
- **WebSockets:** an outbound Worker socket closes when its event and
  `waitUntil` work end (one returned in the response stays open). 1 MiB budget
  on non-terminal frames per isolate-polled input queue; unread frames are
  discarded if the isolate stops polling. Transport can't move owner — client
  reconnects with the same operation id. `acceptWebSocket()` throws above 90%
  V8 heap. A tunneled connection forwards the owner's Close as-is; owner
  failure between frames ⇒ ingress sends 1012, mid-frame ⇒ transport closed.
  `wasClean` is `true` whenever the peer sent a close frame (v0.6.0). A
  WebSocket response to a request **without `Upgrade: websocket`** fails as in
  workerd (v0.6.2): `stub.fetch()` rejects, an HTTP client gets 500, the
  server end sees close 1006.
- **Web Crypto:** HMAC hashes MD5/SHA-1/SHA-224/256/384/512; ECDSA P-256 +
  SHA-256 only; AES-GCM tags 96–128 bits in 8-bit steps; RSA-OAEP
  SHA-1/256/384/512 (a non-empty label must be UTF-8); no `jwk` for a secret
  key in `exportKey()`/`wrapKey()`. **Ed25519** (also as `NODE-ED25519`) and
  **X25519** since v0.6.0, with `raw` import/export of the 32-byte point;
  X25519 `deriveBits()` rejects a low-order peer key.
- **Performance:** `performance.timeOrigin` = 0, `performance.now()` ==
  `Date.now()`; both advance only at an I/O boundary.
- **Node.js compat — implemented:** `node:assert`, `node:async_hooks`,
  `node:buffer`, `node:diagnostics_channel` (not exported to a tail Worker),
  `node:events`, `node:fs`, `node:os`, `node:path`, `node:stream`,
  `node:timers/promises`, `node:util`. `node:crypto` lacks Diffie-Hellman,
  streaming signatures, ciphers, RSA-PSS, DSA; `KeyObject.toCryptoKey()`
  honors algorithm/usages/extractability (so `jose` RSA signing works,
  v0.5.1). `node:zlib` = synchronous gzip/deflate only. `node:fs` = empty
  request-local `/tmp` + read-only `/bundle`; implements `access`, `mkdir`,
  `realpath`, `stat`, `lstat`, `readFile`. Built-in module objects are
  writable (e.g. `graceful-fs` patching works, per isolate). The global
  `process` matches workerd's fields; others (e.g. `process.kill`) are
  undefined. `require()` of a Node built-in works (synchronous CommonJS); a
  raw ESM Worker gets no global `require()`. Import of any other Node module
  succeeds; the first call into it throws.
- **TCP sockets:** a socket can't outlive the event that created it — a DO must
  reconnect in a later event. TLS verified against the bundled Mozilla root
  store. celld blocks no destination port (operator controls egress).
- **Cache:** `caches.default` / `caches.open()` always-miss; `put()` validates
  and drains the body but stores nothing; `match()` → `undefined`, `delete()`
  → `false`.

## Compatibility flags celld honors

`delete_all_deletes_alarm`, `js_rpc`, `fetcher_no_get_put_delete`,
`sqlite_vec`, `websocket_standard_binary_type`, and the static-assets
navigation flags (plus `python_workers` and the Python date flags for a `.py`
`main`). Every other flag is accepted without effect;
`Cloudflare.compatibilityFlags` reports only the honored ones.

## Wrangler configuration

- `wrangler.jsonc` or `wrangler.json` only — **not `wrangler.toml`**.
- `name`: 1–63 lowercase ASCII letters / digits / internal hyphens, no
  leading/trailing hyphen.
- Accepted top-level keys: `$schema`, `name`, `main`, `no_bundle`,
  `compatibility_date`, `compatibility_flags`, `durable_objects`, `migrations`,
  `assets`, `services`, `triggers`, `vars`, `d1_databases`, `kv_namespaces`,
  `queues`, `workflows`, `r2_buckets`, `worker_loaders`, `containers`,
  `define`, `rules` (the last two since v0.5.1).
- **Any other top-level key — including `routes` — stops the deploy.**
  Configure routing in your ingress.
- `define` and `rules` go to esbuild. Each `define` value is a JS expression,
  as in Wrangler. A rule is exactly `{ "type", "globs" }` — `type` is `Text`,
  `Data`, or `CompiledWasm`; each glob must be `**/*.ext` or `*.ext` (esbuild
  picks a loader by extension); **any other key, e.g. `fallthrough`, stops the
  deploy**; two rules giving one extension different types stop it;
  `CompiledWasm` is already applied to `**/*.wasm`. So a Wrangler rule like
  `{ "type": "Text", "globs": ["**/*.sql"], "fallthrough": true }` must drop
  `fallthrough` to deploy on celld (it then makes `import x from "./a.sql"`
  work — relevant for drizzle's `durable-sqlite` migrations). `no_bundle` with
  `define` or `rules` stops the deploy.
- An asset-only project can omit `main`. celld refuses a symlink / special file
  / non-UTF-8 name / unsafe decoded path / a `_worker.js` entry in an asset dir.
- Deployment manifest records each module's full SHA-256; a node verifies bytes
  before building, so changed bytes can't become active.

## WebAssembly

A Worker bundle can `import mod from "./x.wasm"` — the import is the compiled
`WebAssembly.Module` (Wrangler's `CompiledWasm` rule), not bytes. `celld
deploy` uploads each wasm file beside the bundle and marks the deployment
`wasm-v1` (an older node refuses it at deploy time). Each module is compiled
once per process and reused by every isolate. Rust: `worker-build --release`
(from `workers-rs`) → point `main` at `build/worker/shim.mjs` → `celld deploy`.

**Prebuilt Workers (`no_bundle: true`):** celld preserves the entry JS
byte-for-byte and applies Wrangler's default `**/*.wasm` /
`**/*.wasm?module` scan below the directory containing `main`, so a prebuilt
`main` can import a `.wasm` at a relative path under it. The scan uploads
every matching file, imported or not — use a dedicated build output directory,
and copy (don't symlink) a wasm file into it. This mode doesn't discover extra
JS modules and doesn't accept `rules` / `base_dir` /
`find_additional_modules`; it needs no `esbuild` on `PATH`.

**Dynamic Workers:** pass wasm as `{ wasm: bytes }` in `modules`, with a
`compatibilityDate` (both required since v0.6.0).

## Containers

See `SKILL.md` § *Containers* for the config shape, the node-side engine
requirement, network fencing, the `ctx.container` surface, and
`@cloudflare/containers` / `@cloudflare/sandbox` support — condensed here:

- `image` is a Dockerfile path or an image reference; `celld deploy` builds
  (`linux/amd64` by default) or pulls it with the `docker` CLI (or
  `CELLD_DOCKER`; Podman works) and uploads it once to
  `deploy/images/<key>.tar`, so a node never contacts a registry. `celld dev`
  builds for the local machine.
- A node needs a Docker/Podman daemon (Unix socket via `DOCKER_HOST` or the
  default socket) to activate the class at all; `CELLD_CONTAINER_RUNTIME` (or
  a per-class `runtime` key) selects the OCI runtime, e.g. `runsc`/`kata` for
  real kernel isolation — the default runtime is only a namespace boundary
  (all caps dropped, `no-new-privileges`, ≤1024 processes).
- celld fences every container's network with nftables via a one-shot
  `celld-fence` image before any container starts: `enableInternet: true`
  reaches the Internet but not the node, other nodes, private ranges
  (10/8, 172.16/12, 192.168/16, 100.64/10), or link-local; `enableInternet:
  false` gets no route out (on macOS `celld dev` keeps egress on with a
  warning).
- `ctx.container` supports `start()`, `running`, `monitor()`, `destroy()`,
  `signal()`, `getTcpPort()` (`.fetch()` incl. WebSocket upgrade, and
  `.connect()`), `exec()`, `setInactivityTimeout()`; `inspect()`, snapshot
  methods, and outbound interception reject; `hardTimeout` is validated then
  ignored. Disk size of an instance type isn't enforced. A container's disk
  survives an idle eviction but not a node move/restart/reset/stop.
- `@cloudflare/containers` and `@cloudflare/sandbox` (on `cloudflare/sandbox`)
  run as published; a move to another node stops the container, so the first
  post-move call can raise the SDK's `OperationInterruptedError`, same as a
  Cloudflare container restart. Sandbox preview URLs need a wildcard hostname
  at the load balancer.
