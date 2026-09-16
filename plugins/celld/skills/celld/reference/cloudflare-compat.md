# celld ↔ Cloudflare Workers compatibility

Distilled from celld v0.5.0: `docs/cloudflare-compat.md` and `docs/wasm.md` in
<https://github.com/denoland/celld> (fetch those for the primary text).
celld implements the Cloudflare Workers APIs and **must reject an
unsupported configuration or API at deployment or first use** — an unsupported
feature that doesn't error is a defect. The notes below are only the celld gaps
and differences.

- **Yes** — an app that uses the API runs on celld.
- **Partial** — a large part is missing; an app depending on that part won't run.
- **Experimental** — needs an opt-in setting and can change without notice.
- **No** — not implemented.

## Services

| Service | Status | Key notes |
| --- | --- | --- |
| Workers | Yes | No custom domain / TLS termination — do it at ingress. celld has no Workers AI, and rejects `CELLD_AI_BINDING`/`CELLD_AI_URL` and an `ai` config declaration (both removed in v0.5.0 — call a provider directly). |
| Durable Objects | Yes | SQLite storage, alarms, hibernating WebSockets. Pending I/O (a timer, a subrequest) stays active after the handler returns with no `ctx.waitUntil()` needed. See the storage-API notes below. |
| Containers | Experimental | A `containers` entry (`class_name`, `image`, `name`, `instance_type`, `max_instances`) binds a Docker/Podman image to a SQLite-backed DO class — see *Containers* below. Config keys, `ctx.container`, and the security boundary can change without notice. |
| Static assets | Yes | No edge cache (512 MiB local disk cache, `CELLD_ASSET_CACHE_BYTES`), no compression (put a compressing proxy in front). `_headers` / `_redirects` supported but can't set `connection`/`content-length`/`transfer-encoding`; ≤100 `_headers` rules, ≤2000 static + 100 dynamic `_redirects` rules, 100 KiB each. ≤20,000 assets, 25 MiB each, 1 GiB total. `.assetsignore` needs Wrangler. Each response carries an `etag` + `cache-control: public, max-age=0, must-revalidate`. |
| Cron Triggers | Yes | One handler per occurrence across the whole fleet. One missed occurrence re-run after downtime. One handler at a time per script; retries a failed handler until the next occurrence unless it calls `noRetry()`. Rejects descending ranges (`SAT-SUN`) and a list that contains `*`. A service-binding target can't run its own crons. |
| KV | Yes | No edge cache (`cacheTtl` ignored, `cacheStatus` null). Values > 1 MiB need a fleet bucket. **One writer per namespace** — add namespaces to scale writes. |
| Queues | Yes | **One writer per queue**; one consumer script (which can't also export `fetch()`). Owner admits ≤256 concurrent producer calls (refuses extra — producer retries). Messages retained 4 days (not configurable). No pull consumers, no Queues HTTP API, no dashboard controls / R2 event notifications / event subscriptions. |
| D1 | Yes | Binding result ≤ 100,000 rows or 32 MiB. `TEXT` rejects invalid UTF-8 — use `BLOB`. |
| Workflows | Yes | celld retains a terminal instance 30 days by default; `retention` can request up to 30 days. `locationHint` is accepted (Cloudflare's values) but fleet ownership picks the actual location. Replays `run()` from the start — code outside a step runs again; a crash after a step side effect can re-run the callback. Non-step work can't stay pending > 60 s. Step result / event payload / params ≤ 1 MiB each. `pause()`/`resume()`/`restart(from)` supported; rollback, sensitive/`ReadableStream` step results are not. |
| R2 | Yes | Uses the fleet bucket under `r2/<bucket_name>/`. `version` = content ETag (identical bytes ⇒ one version). No `ssecKey`, no `jurisdiction`. Conditional write can't use a streamed body > 8 MiB. Multipart: no checksum on `createMultipartUpload()`, can't resume on another node or after a restart, can't replace a stored part, out-of-order parts ≤ 256 MiB memory. |
| Workers AI / Vectorize / Hyperdrive / Browser Rendering / Email Workers / Python Workers | No | — |
| Dynamic Workers (Worker Loader / Code Mode) | Yes | Declared with the `worker_loaders` config key in `wrangler.json` (no env var, and no longer experimental — `CELLD_WORKER_LOADER`/`CELLD_MAX_LOADED_WORKERS` are removed). Process-wide limit: 256 live loaded Workers, ≤255 slots per script generation. `getEntrypoint()`/`getDurableObjectClass()` support only the `props` option (≤1 MiB structured-clone encoded); `WorkerCode.env` takes structured-clone values and Service Binding capabilities (≤1 MiB total). No `limits`/`tails`/`allowExperimental` (rejected); a `globalOutbound` Fetcher can't `connect()` or use a WebSocket; a loaded entrypoint can't transfer to another Worker; no awaitable/pipelined properties. |
| Durable Object Facets | Yes | Requires Dynamic Workers and a class from a Worker Loader binding (no `ctx.exports`/DO-binding facet class). `ctx.facets` → `get()`, `abort()`, `delete()`; no `clone()`. Each facet has an isolated SQLite DB replicated with the root DO; celld rejects an outbound effect from a facet while a root storage transaction holds an uncommitted facet image. An explicit transaction's writes are visible only in the facet until commit, when they become available for root replication; a rollback discards them. |

## Durable Object storage API — celld specifics

- `storage.sync()` resolves only after the object store or the fleet ensemble
  holds every write committed before the call. Rejects (and celld resets the
  object, restoring the proven history) when it can't prove durability within
  the shorter of `CELLD_LTX_DURABILITY_TIMEOUT_SECS` (10, from upload start)
  and `CELLD_OPERATION_DEADLINE_MS` (15000). The reset can stop the object
  before the handler sees the rejection — don't depend on the rejection to
  recover. Also rejects while a `transaction()` is open, and after
  `ctx.abort()` or a failed `blockConcurrencyWhile()`. Without an object store,
  it resolves after the local commit — there is no `CELLD_OUTPUT_GATE=0`
  opt-out anymore (removed in v0.5.0; celld always waits for the configured
  durability proof).
- Outside an explicit transaction, a SQL write cursor must finish (be fully
  read) before a response, an outbound effect, or `storage.sync()` — an
  unfinished `RETURNING` cursor holds uncommitted writes, and celld rejects
  that output with an error. A read cursor can stay open.
- **A handler that fails after it writes answers only after the write is
  durable** — celld holds the error behind the same output gate as a success
  (a later request could read that write). Applies to `fetch`/RPC, all three
  `webSocket*` handlers, and `alarm`. A handler that fails without a write of
  its own waits for every write the object holds to be durable (its thrown
  message can carry a value it read); a failure celld raises itself (over
  budget, waiting on nothing) carries no object value and leaves at once.
- `transaction()` callback runs under the input gate (no other event starts);
  a callback running > 30 s resets the object and rolls back (the
  `blockConcurrencyWhile()` limit). A throwing callback rolls back and the
  object continues. A rollback cancels un-awaited writes in that transaction; a
  nested rollback preserves the outer transaction's writes.
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

- **Fetch:** no `cache` option; `redirect` accepts `follow`/`manual`/`error`
  only; celld strips `Content-Length` from a Worker response (kept for `HEAD`).
  An inbound `Request`'s `cf` object has no Cloudflare edge fields (no
  geolocation/colo/TLS metadata). A remote DO call streams the request body
  and can't retry after transmission starts.
- **RPC:** an RPC stub still can't cross an isolate boundary, but a named
  service binding now supports `fetch()` on a named entrypoint across
  isolates, not just a single method call (v0.5.0). An `AbortSignal` in a DO
  RPC call passes through only **on the same node** — it does not cross a node
  boundary. A remote RPC retries only when the failed peer attempt didn't
  start the method — use a stable operation id for other retries.
- **Streams:** an unclaimed/inactive HTTP stream expires after 60 s (an
  expired/unknown stream reports an error, not EOF).
- **WebSockets:** must `accept()` from a subrequest upgrade; celld rejects an
  upgrade unless the response status is 101; 1 MiB byte budget on non-terminal
  frames per input queue (a bigger message blocks further reads until consumed);
  a close frame with an invalid status ⇒ close code 1002, `wasClean` false;
  transport can't move owner — client reconnects with the same operation id;
  `acceptWebSocket()` throws above 90% V8 heap. A tunneled connection forwards
  the owner's Close frame as-is; if the owner connection fails between frames
  before sending a Close, the ingress sends code 1012, and an incomplete frame
  closes the transport instead.
- **Web Crypto:** HMAC hashes MD5/SHA-1/SHA-224/256/384/512; ECDSA P-256 +
  SHA-256 only; AES-GCM tags 96–128 bits in 8-bit steps; RSA-OAEP
  SHA-1/256/384/512; no `jwk` for a secret key in `exportKey()`/`wrapKey()`.
- **Web standards:** an `AbortSignal` now crosses a DO RPC call, but only on
  the same node (see RPC above); it still doesn't cross a node boundary.
- **Performance:** `performance.timeOrigin` = 0, `performance.now()` ==
  `Date.now()`; both advance only at an I/O boundary.
- **Node.js compat — implemented:** `node:assert`, `node:async_hooks`,
  `node:buffer`, `node:diagnostics_channel`, `node:events`, `node:fs`,
  `node:os`, `node:path`, `node:stream`, `node:timers/promises`, `node:util`.
  `node:crypto` lacks Diffie-Hellman, streaming signatures, ciphers, RSA-PSS,
  DSA. `node:zlib` = synchronous gzip/deflate only. `node:fs` = empty
  memory-backed `/tmp` per request + read-only `/bundle` (one file per Worker
  module, entry `/bundle/worker.js`); implements `access`, `mkdir`, `realpath`,
  `stat`, `lstat`, `readFile`. `require()` of a Node built-in works
  (synchronous CommonJS); a raw ESM Worker gets no global `require()`. Import
  of any other Node module succeeds; the first call into it throws.
- **TCP sockets:** a socket can't outlive the event that created it — a DO must
  reconnect in a later event. TLS verified against the bundled Mozilla root
  store. celld blocks no destination port (operator controls egress).
- **Cache:** `caches.default` / `caches.open()` always-miss; `put()` validates
  and drains the body but stores nothing; `match()` → `undefined`, `delete()`
  → `false`. `ctx.passThroughOnException()` has no effect.

## Compatibility flags celld honors

`delete_all_deletes_alarm`, `js_rpc`, `fetcher_no_get_put_delete`,
`sqlite_vec`, `websocket_standard_binary_type`, and the static-assets
navigation flags. Every other flag is accepted without effect;
`Cloudflare.compatibilityFlags` reports only the honored ones.

## Wrangler configuration

- `wrangler.jsonc` or `wrangler.json` only — **not `wrangler.toml`**.
- `name`: 1–63 lowercase ASCII letters / digits / internal hyphens, no
  leading/trailing hyphen.
- Accepted top-level keys: `$schema`, `name`, `main`, `no_bundle`,
  `compatibility_date`, `compatibility_flags`, `durable_objects`, `migrations`,
  `assets`, `services`, `triggers`, `vars`, `d1_databases`, `kv_namespaces`,
  `queues`, `workflows`, `r2_buckets`, `worker_loaders`, `containers` (the
  last two added in v0.5.0 — see the Dynamic Workers / Containers rows above).
- **Any other top-level key — including `routes` and `rules` — stops the
  deploy.** Configure routing in your ingress. There is no equivalent of
  wrangler's module `rules`: the bundler runs esbuild with only
  `--loader:.wasm=copy`, so non-JS imports (`.sql`/`.txt` as Text modules, etc.)
  aren't supported — inline that content into JS before deploying. This bites
  drizzle's `durable-sqlite` migration bundle; see the `drizzle-orm` skill.
- An asset-only project can omit `main`. celld refuses a symlink / special file
  / non-UTF-8 name / path > 1024 bytes / path that changes under
  percent-decoding / a `_worker.js` entry in an asset dir.
- Deployment manifest records each JS/wasm module's full SHA-256; a node
  verifies bytes before building, so changed bytes can't become active (a
  legacy 16-char digest is also accepted).

## WebAssembly

A Worker bundle can `import mod from "./x.wasm"` — the import is the compiled
`WebAssembly.Module` (Wrangler's `CompiledWasm` rule), not bytes. `celld
deploy` uploads each wasm file beside the bundle and marks the deployment
`wasm-v1` (an older node refuses it at deploy time). Each module is compiled
once per process and reused by every isolate. Rust: `worker-build --release`
(from `workers-rs`) → point `main` at `build/worker/shim.mjs` → `celld deploy`.

**Prebuilt Workers (`no_bundle: true`, since v0.5.0):** celld preserves the
entry JS byte-for-byte and applies Wrangler's default `**/*.wasm` /
`**/*.wasm?module` scan below the directory containing `main`, so a prebuilt
`main` can import a `.wasm` at a relative path under it. The scan finds every
matching file, imported or not — use a dedicated build output directory so
celld doesn't upload unrelated wasm, and copy (don't symlink) a wasm file into
that directory. This mode doesn't discover extra JS modules and doesn't
accept `rules` / `base_dir` / `find_additional_modules`; it needs no `esbuild`
on `PATH`. See `docs/wasm.md` § *Prebuilt Workers*.

## Containers

See `SKILL.md` § *Containers* for the config shape, the node-side engine
requirement, network fencing, the `ctx.container` surface, and
`@cloudflare/containers` / `@cloudflare/sandbox` support — condensed here:

- `image` is a Dockerfile path or an image reference; `celld deploy` builds
  (`linux/amd64` by default) or pulls it with the `docker` CLI (or
  `CELLD_DOCKER`) and uploads it once to `deploy/images/<key>.tar`, so a node
  never contacts a registry.
- A node needs a Docker/Podman daemon to run the class at all; `CELLD_CONTAINER_RUNTIME`
  (or a per-class `runtime` key) selects the OCI runtime, e.g. `runsc`/`kata`
  for real kernel isolation — the default runtime is only a namespace boundary.
- celld fences every container's network with nftables via a one-shot
  `celld-fence` image before any container starts: `enableInternet: true`
  reaches the Internet but not the node, other nodes, private ranges, or
  link-local addresses; `enableInternet: false` gets no route out.
- `ctx.container` supports `start()`, `monitor()`, `destroy()`, `signal()`,
  `getTcpPort()`, `exec()`, `setInactivityTimeout()`; no `inspect()`, no
  snapshot methods, no outbound interception. A container's disk survives an
  idle eviction but not a node move/restart/reset.
- `@cloudflare/containers` and `@cloudflare/sandbox` (on `cloudflare/sandbox`)
  run as published; a move to another node stops the container, so the first
  post-move call can raise the SDK's `OperationInterruptedError`, same as a
  Cloudflare container restart.
