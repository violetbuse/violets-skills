# celld ↔ Cloudflare Workers compatibility

Distilled from celld v0.4.1: `docs/cloudflare-compat.md` and `docs/wasm.md` in
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
| Workers | Yes | No custom domain / TLS termination — do it at ingress. Outbound `fetch` honors a caller `AbortSignal`; an incoming request's signal is not inherited by subrequests. |
| Durable Objects | Yes | SQLite storage, alarms, hibernating WebSockets. See the storage-API notes below. |
| Static assets | Yes | No edge cache (512 MiB local disk cache, `CELLD_ASSET_CACHE_BYTES`), no compression (put a compressing proxy in front). `_headers` / `_redirects` supported but can't set `connection`/`content-length`/`transfer-encoding`; ≤100 `_headers` rules, ≤2000 static + 100 dynamic `_redirects` rules, 100 KiB each. ≤20,000 assets, 25 MiB each, 1 GiB total. `.assetsignore` needs Wrangler. Each response carries an `etag` + `cache-control: public, max-age=0, must-revalidate`. |
| Cron Triggers | Yes | One handler per occurrence across the whole fleet. One missed occurrence re-run after downtime. One handler at a time per script; retries a failed handler until the next occurrence unless it calls `noRetry()`. Rejects descending ranges (`SAT-SUN`), `*` inside a list (`1,*`). A service-binding target can't run its own crons. |
| KV | Yes | No edge cache (`cacheTtl` ignored, `cacheStatus` null). Values > 1 MiB need a fleet bucket. **One writer per namespace** — add namespaces to scale writes. Namespace id = Cloudflare hex form or any stable string. |
| Queues | Yes | **One writer per queue**; one consumer script (which can't also export `fetch()`). Owner admits ≤256 concurrent producer calls (refuses extra — producer retries). Message id = UUIDv7 (sorts in enqueue order per owner). Messages retained 4 days (not configurable). No pull consumers, no Queues HTTP API, no dashboard controls / R2 event notifications / event subscriptions. |
| D1 | Yes | Binding result ≤ 100,000 rows or 32 MiB. `TEXT` rejects invalid UTF-8 — use `BLOB`. |
| Workflows | Yes | `create()` replaces a terminal instance with the same id (Cloudflare refuses duplicates). Replays `run()` from the start — code outside a step runs again; a crash after a step side effect can re-run the callback. Non-step work can't stay pending > 60 s. Step result / event payload / params ≤ 1 MiB each. `pause()`/`resume()`/`restart(from)` supported; `retention`, `locationHint`, `delete()`, `deleteBatch()`, rollback, sensitive/`ReadableStream` step results are not. |
| R2 | Yes | Uses the fleet bucket under `r2/<bucket_name>/`. `version` = content ETag (identical bytes ⇒ one version). No `ssecKey`, no `jurisdiction`. Conditional write can't use a streamed body > 8 MiB. Multipart: no checksum on `createMultipartUpload()`, can't resume on another node or after a restart, can't replace a stored part, out-of-order parts ≤ 256 MiB memory. |
| Workers AI | No (native) | Experimental HTTP adapter: with `CELLD_AI_URL` set, `env.AI.run()` POSTs `{model, input}` as JSON. Third options arg: `returnRawResponse: true` returns the upstream `Response` (also for error status); default parses JSON and throws on non-2xx. `signal` cancels. |
| Vectorize / Hyperdrive / Browser Rendering / Email Workers / Python Workers | No | — |
| Dynamic Workers (Worker Loader / Code Mode) | Experimental | `CELLD_WORKER_LOADER=LOADER` exposes `env.LOADER`. A call from a DO into a loaded Worker (or a service binding) waits for the DO's durability proof. No `globalOutbound` Fetcher, no capability stub in `env`, no awaitable/pipelined properties. `CELLD_MAX_LOADED_WORKERS` default 256. |
| Durable Object Facets | Experimental | Requires Dynamic Workers. `ctx.facets` → `get()`, `abort()`, `delete()`. A facet has an isolated SQLite DB replicated with the root DO; a facet's outbound effect / `storage.sync()` waits for the **root** object's durability proof. No `ctx.exports`/DO-binding facet class, no `clone()`. |

## Durable Object storage API — celld specifics

- `storage.sync()` resolves only after the object store or the fleet ensemble
  holds every write committed before the call. Rejects (and celld resets the
  object, restoring the proven history) when it can't prove durability within
  the shorter of `CELLD_LTX_DURABILITY_TIMEOUT_SECS` (10, from upload start)
  and `CELLD_OPERATION_DEADLINE_MS` (15000). The reset can stop the object
  before the handler sees the rejection — don't depend on the rejection to
  recover. Also rejects while a `transaction()` is open, and after
  `ctx.abort()` or a failed `blockConcurrencyWhile()`. With
  `CELLD_OUTPUT_GATE=0` or no object store, it resolves after the local commit.
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
  A remote DO call streams the request body and can't retry after transmission
  starts.
- **RPC:** a cross-isolate named service binding supports a single method call
  only — no `fetch()`, awaitable properties, or pipelined paths. `ctx.exports`
  has only declared entrypoints. A remote RPC retries only when the failed peer
  attempt didn't start the method — use a stable operation id for other retries.
- **Streams:** an unclaimed/inactive HTTP stream expires after 60 s.
- **WebSockets:** must `accept()` from a subrequest upgrade; celld rejects an
  upgrade unless the response status is 101; 1 MiB byte budget on non-terminal
  frames per input queue (a bigger message blocks further reads until consumed);
  a close frame with an invalid status ⇒ close code 1002, `wasClean` false;
  transport can't move owner — client reconnects with the same operation id;
  `acceptWebSocket()` throws above 90% V8 heap.
- **Web Crypto:** HMAC hashes MD5/SHA-1/SHA-224/256/384/512; ECDSA P-256 +
  SHA-256 only; AES-GCM tags 96–128 bits in 8-bit steps; RSA-OAEP
  SHA-1/256/384/512; no `jwk` for a secret key in `exportKey()`/`wrapKey()`.
- **Web standards:** `AbortSignal` doesn't abort an RPC call; `signal.onabort`
  has no effect (use `addEventListener("abort", …)`); `structuredClone()`
  doesn't clone an `AbortSignal`.
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
  `queues`, `workflows`, `r2_buckets`.
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
