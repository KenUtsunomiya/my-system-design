# Multi-region database topology

Cross-problem reference: how to compose a database cluster that spans regions, the four standard topologies, and how to pick between them. Use this whenever a problem requires multi-region availability or low latency for globally distributed users.

---

## The four standard topologies

| Topology | What it means | Pros | Cons |
|---|---|---|---|
| **1. Single-leader, async cross-region replicas** | One region holds the write primary. Other regions hold read replicas, kept up via async replication. All writes route to the primary; reads are served locally. | Simple. Strong consistency on writes. Unique constraints / transactions "just work" via the DB's normal mechanisms. Mature replication tooling (Postgres logical / streaming, MySQL binlog, etc.). | Writes from non-primary regions pay cross-region RTT (50–200 ms). Primary region outage = global write outage until failover. Reader replicas may serve stale data within the async-replication lag (typically 100 ms – seconds). |
| **2. Multi-leader (active-active)** | Each region can accept writes locally; replication is bidirectional. | Low write latency everywhere. No single write-point SPOF. Survives any one region's outage transparently. | **Conflict resolution is brutal.** Two regions writing the same key → conflict; resolution requires CRDTs, last-write-wins, or app-level merging. **Unique constraints become very hard** (two regions could both grant the same alias / username / slug). |
| **3. Sharded by region (data residency)** | Each row "lives" in one region; that region is authoritative. The shard key is regional. | Clean ownership boundary. Natural data-residency story (GDPR / HIPAA-friendly). No cross-region coordination on writes within a shard. | Reads of foreign-region data cross regions. Routing logic at the application layer (or via a smart proxy) must know which region owns which key. Rebalancing shards across regions is operationally expensive. |
| **4. Globally-distributed DB** (Spanner, CockroachDB, FoundationDB, DynamoDB Global Tables) | The DB itself transparently spans regions. Presents as a single logical store. Some products give strong consistency globally (Spanner) via TrueTime / consensus; others give eventual or session consistency (DynamoDB Global Tables). | Clean abstraction — application code stays simple. Strong consistency available even across regions for some products. Survives regional outages without manual failover. | Cost is significant. Cross-region writes still pay synchronization latency (often a Paxos / Raft round trip = inter-region RTT). Vendor lock-in is real. Operational expertise is rare on the team. |

---

## Decision criteria — what to ask first

The choice is forced by a small number of questions. Walk them in order:

1. **Is there any operation that must be strongly consistent globally?** (e.g., uniqueness on a username / alias / slug; payment idempotency; "first writer wins"-type guarantees)
   - **Yes** → rules out Option 2 (multi-leader can't easily express unique constraints across regions). You're choosing between 1, 3, and 4.
   - **No** → all four are in play.

2. **What is the write rate, and what fraction of writes originate far from the primary region?**
   - Low write rate (≤ a few hundred QPS globally) and most writes near one region → Option 1 is fine; cross-region write latency for the long tail is acceptable.
   - High write rate, geographically distributed → Option 1's write latency penalty becomes painful. Option 4 (Spanner) or Option 3 (sharded by region) gets attractive.

3. **What's the durability vs. latency trade-off on writes?**
   - "Lose ≤ 0 bytes after ack" → synchronous replication required → either single-leader with sync intra-region (and async cross-region, accepting cross-region loss window on regional outage), or globally-distributed DB with sync cross-region replication (slower writes but no loss window).
   - "Tolerate ≤ X seconds of loss after ack" → async replication is OK; Option 1 fits.

4. **Are there data residency / regulatory constraints?**
   - "EU user data must stay in EU; US data in US" → Option 3 (sharded by region) is strongly favored, possibly required.

5. **What's the team's operational maturity?**
   - Strong DBA/SRE muscle → any option is on the table.
   - Small team → Option 1 (vanilla Postgres replication) is dramatically simpler than 2/4. Option 4's managed variants (DynamoDB Global Tables, Spanner) lower this barrier in exchange for cost and lock-in.

---

## When each shines

- **Option 1 (single-leader)** — Default for most CRUD systems with global users where writes are modest and reads dominate. URL shorteners, content management, blogging platforms, most B2B SaaS. Mature, cheap, well-understood.
- **Option 2 (multi-leader)** — Mostly only chosen when writes are **commutative or idempotent** (chat with last-write-wins per message; counter increments) or when you're willing to invest in CRDTs. Riak, Cassandra (with appropriate consistency levels), some collaborative-edit systems.
- **Option 3 (sharded by region)** — Strong fit when data residency is a regulatory requirement (GDPR), or when users naturally cluster regionally and rarely access foreign data (regional social platforms, regional e-commerce).
- **Option 4 (globally-distributed)** — Right when you genuinely need globally-consistent transactions and can absorb the cost. Financial systems, inventory across regions, ad auction systems. Spanner is the textbook case.

---

## Common gotchas

- **The "no SPOF" fallacy on Option 1.** People assume "we have replicas in 3 regions, so we're safe." But if all writes go through one primary, a primary-region outage *still* takes down writes globally until failover completes. Failover is a real, drilled, rehearsed operation, not a magic button.
- **Async replication lag is not zero.** Reads from a replica may return data 100 ms – several seconds older than the primary. If the user does *write then read*, they may not see their own write unless you stick them to the primary or read-after-write through the primary.
- **Multi-leader unique constraints don't work.** If two regions both insert with the same key during a partition, you've got two "winners" when partition heals. CRDTs don't fix this case; you need a globally-coordinated arbiter, which defeats the point of multi-leader.
- **Globally-distributed DBs still pay physics.** Spanner's `commit_wait` is bounded by inter-region RTT for cross-region transactions. Don't expect millisecond writes across continents.
- **Sharded-by-region cross-shard queries are nightmares.** "Find all URLs created by user X" where X has URLs in 3 regions → fan-out query, slow, hard to paginate consistently.

---

## Senior heuristic

> **Start with single-leader, async cross-region replicas (Option 1). Move off it only when forced by:**
> - **Write latency** that single-leader can't meet for non-primary-region users (escalate to Option 4 or carefully-shaped Option 3)
> - **Regulatory data residency** (escalate to Option 3)
> - **Hot writes from many regions to the same keys with conflict semantics that fit CRDTs** (escalate to Option 2)

For most problems, Option 1 satisfies the requirements with the simplest operational model. The senior move is to know **what would force you off it** and to flag that as a §10 follow-up rather than over-engineering on day one.

---

## How this connects to other choices

- The **strong-consistency-on-one-operation pattern** (URL shortener's alias collision; usernames; payment idempotency keys; slug generation) almost always pulls you toward Option 1 or Option 4 — and Option 1 is cheaper unless write latency forces the upgrade.
- **Read-your-writes consistency** is independent of topology choice but interacts with it: in single-leader, reads from a replica may not see the user's own recent write → fix with session stickiness to primary, or "read from primary for N seconds after write."
- **Cross-region failover automation** matters more in single-leader than in any other topology, because it's the recovery mechanism for write availability. Test it; don't trust it without drills.
