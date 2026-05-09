# Caching layers

Cross-problem reference: every cache that can sit between a user and a source of truth, with rough latencies and what each absorbs. Real systems stack these — each layer absorbs traffic the next layer never sees. Senior engineers reason explicitly about which layer catches what fraction of traffic, at what cost.

---

## The full caching stack (top to bottom)

| Layer | Where | Typical hit latency | Notes |
|---|---|---|---|
| **Browser cache** | User's device | ~0 ms (no network) | Controlled by HTTP `Cache-Control` / `ETag`. Closest possible cache. |
| **DNS cache** | Client + recursive resolvers | saves 10–50 ms DNS lookup | Often forgotten; matters for TTFB on first request. |
| **CDN edge** | Edge PoPs near user | 10–50 ms | Caches HTTP responses keyed by URL. Absorbs wide-area traffic. |
| **Reverse proxy / API gateway cache** | Your infra, in front of app servers | sub-ms | Nginx / Varnish / Envoy. In-region edge for response caching. |
| **App-local in-memory cache** | Inside each app process | microseconds | Caffeine / Guava / process-local LRU. **Per-instance — no sharing across replicas.** Use for ultra-hot keys to skip the network hop to Redis. |
| **Distributed cache** | Shared Redis / Memcached cluster | ~1–2 ms | Shared across all app servers; survives app restarts. The standard "the cache" most diagrams refer to. |
| **DB query / result cache** | Inside the DB process | sub-ms | Some DBs maintain this (varies by engine; often disabled in modern OLTP). |
| **DB buffer pool / page cache** | Inside the DB process | sub-ms | Postgres `shared_buffers`, MySQL InnoDB buffer pool. Transparent to your app, but real and load-bearing. |
| **OS page cache** | Kernel | sub-ms | Filesystem-level; transparent. |
| **Disk** | Source of truth | 0.1–10 ms (SSD) | Where caching ends. |

---

## CDN vs application cache — the distinction

Both are caches, but at different layers and with different concerns:

| | **App cache (Redis / Memcached)** | **CDN (Cloudflare / Fastly / CloudFront)** |
|---|---|---|
| Lives where | Your data center / region, next to app servers | At the **edge** — 100s of PoPs near end users |
| Caches what | Anything (DB rows, computed results, sessions) | HTTP responses (full URL → response body + headers) |
| Reduces which latency | App → DB hop (5–20 ms → <1 ms) | User → origin hop (cross-continent 150 ms → 10–20 ms) |
| Reduces which load | DB load | Origin server load + egress bandwidth |
| Cache key | Whatever you choose (e.g., `url:abc123`) | The HTTP URL/path of the request |
| Invalidation | You control it | Mostly TTL-based; explicit purge APIs available |

**Mental model:**
- **DB** = warehouse (source of truth, far, slow, complete)
- **App cache (Redis)** = your kitchen pantry (close to the cook, fits common things)
- **CDN** = corner stores in every neighborhood (close to the customer)

---

## How layers stack in a real read path

A read for a popular item might be served by:

1. The user's **browser cache** (no network at all) — if miss
2. The **CDN edge** (~20 ms) — if miss
3. Your **reverse proxy** (~1 ms) — if miss
4. The **app-local cache** (microseconds) — if miss
5. **Redis** (~1 ms) — if miss
6. The **DB**, with its own buffer pool likely serving the row from RAM anyway

Each layer absorbs traffic the next one never sees. Good design = stacking the right layers for your traffic shape, not picking just one "cache."

---

## Picking layers for a given problem

| Layer | Worth introducing when |
|---|---|
| Browser cache | Responses are cacheable by HTTP semantics (immutable URLs, idempotent GETs) |
| CDN | Users are geographically dispersed AND content is cacheable AND wide-area latency matters |
| Reverse proxy cache | You want response caching without external CDN cost / dependency |
| App-local in-memory cache | A small, very hot working set exists; the ~1 ms Redis hop matters |
| Distributed cache (Redis) | Read QPS in the thousands AND repeated reads of the same keys; shared state across app replicas |
| DB-level caches | These are mostly automatic; tune buffer pool sizes when DB is the hot path |

---

## Two specific lessons worth memorizing

### 301 vs 302 for redirect-heavy systems

For systems that issue HTTP redirects (URL shorteners, link expanders, OAuth flows):

- **301 Moved Permanently** → browsers cache the redirect *indefinitely* (until cache cleared). The user's *next* click never reaches your servers — the browser redirects locally. Massive load reduction.
- **302 Found** (temporary) → browsers don't cache. Every click round-trips to you.

The trade-off: 301 makes deletion / URL revocation effectively impossible for users who already cached the redirect, and you lose visibility into per-click traffic. Most production shorteners use **302** for control + analytics, paying the latency / load cost as a deliberate choice.

### App-local cache as a Redis pre-tier

Even with Redis, every read costs ~1 ms of network. For ultra-hot keys, a process-local LRU (~10K entries, ~10 MB per instance) catches them before Redis:

- Cuts ~1 ms off the hottest requests.
- Adds per-instance staleness — fine for short TTLs.
- Effectively free in CPU/memory.

Mention this in a deep-dive when latency-on-the-hot-path is in the SLO.
