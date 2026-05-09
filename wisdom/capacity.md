# Capacity & scaling toolkit

Cross-problem reference: the techniques available to a system designer when capacity, latency, or availability becomes a constraint, and rough heuristics for *when* to introduce each. Consult this when working through Section 4 (capacity estimation) and Section 6 (high-level architecture) of any system design problem.

---

## The full toolkit

### Performance / scale

- **Cache (in-region)** — reduces read latency and DB load. Lives next to app servers (Redis, Memcached). Caches anything: DB rows, computed results, sessions.
- **CDN** — reduces wide-area latency and egress bandwidth. Lives at the edge (Cloudflare, Fastly, CloudFront). Caches HTTP responses, keyed by URL.
- **Read replicas** — scales read QPS and provides HA. Lag is the trade-off.
- **Sharding / partitioning** — splits data across nodes. Required when single-node storage or write throughput is the bottleneck.
- **Async processing (queues, workers)** — decouples slow work from the user-facing request path. Trades latency-on-the-hot-path for eventual completion.
- **Batching / bulk operations** — amortizes per-call overhead across many items.
- **Denormalization / materialized views** — precomputes joins or aggregates so reads don't pay the cost.
- **Indexes** — turns full scans into log-N lookups. Often the cheapest single win.
- **Compression** — saves network and storage at the cost of CPU.
- **Connection pooling** — eliminates per-request connection overhead (especially for DBs and HTTP backends).

### Reliability / availability

- **Multi-region** — active-active or active-passive. Survives a full regional outage. Expensive operationally.
- **Circuit breakers / bulkheads** — prevents cascading failures when a downstream is sick.
- **Rate limiting / load shedding** — protects the system from overload by rejecting excess load early.
- **Backpressure** — slows down producers when consumers can't keep up. Prevents unbounded queue growth.

---

## When to introduce each — rough heuristics

| Technique | Introduce when | Signal you needed it |
|---|---|---|
| **Bigger machine (vertical scale)** | Always try first | Cheapest; fewest operational risks |
| **Indexes / query tuning** | DB CPU > 50%, slow queries in p99 | Often free; do this before anything fancy |
| **App cache** | Read QPS in the thousands AND repeated reads of the same keys | Hit rate ≥ 80% post-deploy |
| **Read replicas** | Primary read CPU saturating; reads tolerate ms-scale lag | Primary breathes easier |
| **CDN** | Users geographically dispersed AND content is cacheable | TTFB drops for distant users; egress bill drops |
| **Async queue** | Synchronous path includes slow work the user shouldn't wait for | p99 of the user-facing endpoint drops |
| **Sharding** | Single-node storage > ~1–10 TB OR write QPS exceeds single-node throughput | Vertical scaling stops being cheap or possible |
| **Multi-region** | SLO requires it (e.g., 99.99% with regional outage budget) OR users span continents | Survives a regional outage |

---

## The senior principle: add complexity only when forced

Order of escalation, cheapest to most expensive (operationally):

> **vertical scale → indexes → app cache → read replicas → CDN → async pipelines → sharding → multi-region**

Each step adds operational burden (monitoring, debugging, failure modes, on-call surface area). Each step is justified only when the previous step has been tried and proven insufficient.

### Anti-pattern: premature distribution

Designing a sharded, multi-region, event-sourced system on day one for traffic a single Postgres on a beefy box would handle. Common in interview answers and a fast way to lose senior signal — it shows you reach for complexity before you've measured the actual constraint.

A senior engineer's instinct is the opposite: **start boring, prove the bottleneck, escalate.**
