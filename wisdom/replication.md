# Database replication

Cross-problem reference: what replicas are *for*, the difference between sync and async replication, where reads should go, and the standard patterns. Consult whenever a design has more than one DB instance.

The single biggest misconception to defeat upfront:

> **Replicas are not just standby servers for failover.** In every well-designed production system, replicas serve reads in steady state. Treating them as cold standby is leaving most of their value on the table.

---

## The two purposes of replication

A single replica accomplishes **two** things at once:

1. **High availability / failover.** If the primary dies, a replica is promoted and writes resume after a brief outage. This is the durability/availability story.
2. **Read scalability.** Replicas serve read traffic, freeing the primary to handle writes. Read throughput scales linearly with the number of replicas (up to coordination limits).

A production deployment uses **both purposes simultaneously**. There's no such thing as "the read replicas" vs. "the failover replicas" — they're the same machines wearing two hats.

---

## Synchronous vs asynchronous replication

Sync vs async is about **durability**, not about whether replicas serve reads. **Both kinds of replicas can serve reads.** The difference is the staleness window and the durability story on primary failure.

| | Synchronous | Asynchronous |
|---|---|---|
| **What the primary does on commit** | Waits for ≥ N replicas to ACK before ACKing the client | ACKs the client immediately; replicas catch up in the background |
| **Write latency** | Higher — bound by slowest replica's network RTT | Lower — bound by local commit only |
| **Durability on primary failure** | No data loss (replicas have everything ACKed) | Data loss window equal to current replication lag |
| **Replica staleness when serving reads** | Microseconds (near-zero lag) | Milliseconds to seconds (sometimes more under load) |
| **Typical use** | Within a region (low-latency network) for HA + read scaling without staleness | Across regions (high-latency network) where sync would make writes too slow |

**Common pattern:** sync replication within the home region (≥ 2 replicas across ≥ 2 AZs); async replication across regions. This balances durability inside the failure domain you care about most (region) against the latency cost of cross-region sync.

---

## Read routing — where each kind of read goes

Different reads tolerate different staleness. The standard playbook:

| Read type | Where it goes | Why |
|---|---|---|
| **Strongly-consistent / read-your-writes** | Primary | The replica may be milliseconds behind; the read needs to see the very latest write. |
| **General reads tolerating ms-scale staleness** | Sync replicas (home region) | Near-zero lag; offloads from primary; scales linearly with replica count. |
| **Reads where seconds of staleness are fine** | Async replicas (any region, often local) | Lowest user latency; eventual consistency. |
| **Read-modify-write transactions** | Primary | Lock + write must serialize on the same node. |
| **Analytics / OLAP queries** | Dedicated read replica with looser lag tolerance, possibly in a separate cluster | Avoids OLTP contention on the primary's working pages. |

**Routing decisions are usually made in the application** (or via a smart proxy / connection pool that knows about replication topology). A common pattern:

```python
def get_short_link(code):
    return replica_pool.query("SELECT * FROM shortlinks WHERE code = $1", code)
    # safe — eventual consistency is fine for redirect lookup

def create_short_link(...):
    return primary.execute("INSERT INTO shortlinks ...")
    # must go to primary — only it accepts writes

def list_my_recent_links(user_id):
    # special: user just created one and wants to see it
    # → either pin to primary briefly, or use sticky session, or read primary
    return primary.query("SELECT ...")  # safest default for "show me my own data"
```

---

## Patterns by topology

### Single-leader (the most common)

- One primary accepts all writes.
- N replicas (sync within region, async across regions) serve reads.
- Failover: promote a replica when primary fails.
- **Best for:** OLTP systems with strong-consistency requirements on writes, modest write rate. Default choice for most CRUD systems.

### Multi-leader (active-active)

- Every replica is a leader — accepts both reads and writes locally.
- Bidirectional replication between leaders.
- Reads always local; writes always local (low latency everywhere).
- **Cost:** conflict resolution. Two leaders writing the same key concurrently → conflict; resolved by CRDTs, last-write-wins, or app-level merging. Unique constraints across leaders are very hard.
- **Best for:** Geographically distributed writes where conflicts are rare or the data shape supports CRDTs (counters, sets, last-write-wins fields).

### Leaderless (Cassandra / DynamoDB style)

- No dedicated leader; reads and writes go to any node, with a tunable quorum (e.g., write to ≥ W nodes, read from ≥ R nodes such that W + R > N).
- Replication via gossip / read-repair / hinted handoff.
- **Best for:** Massive horizontal scale where you can tolerate eventual consistency and design around it (timestamps for last-write-wins; CRDTs for counters; etc.).

See `wisdom/multi-region-databases.md` for how these interact with multi-region designs.

---

## Read-after-write — the consistency trap

Common bug: user creates a resource, immediately tries to view it, and gets a 404 because the read went to a replica that hasn't caught up yet.

Three standard fixes:

1. **Read from primary for the user's own recent writes.** Track "this user wrote at T; for the next N seconds, route their reads to the primary." Simple, common.
2. **Sticky sessions to the primary region** for users who recently wrote. Coarse-grained version of (1).
3. **Read your own writes via cache.** Write-through to a cache the user's reads also hit (avoids replica lag entirely for the just-written entry). This is what URL shortener does for the creator — the new code is populated in the local Redis on write, so the creator's next click is a cache hit regardless of replica lag.

The general principle: **eventual consistency is fine for everyone *except the writer*.** The writer expects to see their own write immediately, and that's where staleness bugs are most user-visible.

---

## Common gotchas

- **"Sync replication is slower because it goes to all replicas"** — partially true. Sync replication waits for the slowest replica in the *required* set (often N=2 of M), not all of them. Tail latency is still bounded by the slowest of those N.
- **Replica lag is not zero, even on "fast" sync replication** — measured in microseconds within a region, milliseconds-to-seconds across regions. Never assume zero.
- **Failover is not free.** Promotion takes seconds-to-minutes; in-flight writes during the failover window are lost (with async) or hung (with sync). Drill failover; don't trust the automation without rehearsal.
- **Cascading replica failure.** When the primary dies and a replica is promoted, the *other* replicas now point at the old primary. They need re-pointing. Many tools automate this; some don't.
- **Reads from a stale replica during failover.** A replica that's behind by minutes still serves reads happily. If the primary just promoted, that replica might serve very stale data until rebuilding. Health checks should consider replication lag, not just liveness.
- **Read replica throughput scales with replica count, but writes don't.** Adding replicas only scales reads. If writes are the bottleneck, replicas don't help — sharding does.

---

## Mental model summary

- **Primary** = the one node that accepts writes (in single-leader). All writes serialize here.
- **Replicas** = copies of the primary, kept in sync via the replication log.
- **Reads can go anywhere** — pick based on staleness tolerance.
- **Sync vs async** = durability/latency trade-off, not access pattern.
- **Replicas are working machines, not standby spare tires.** In steady state, they carry most of the read traffic.

When designing, always ask:

1. Which reads need to see the latest write? (→ primary)
2. Which reads tolerate ms staleness? (→ sync replicas)
3. Which reads tolerate sec staleness? (→ async replicas, possibly in another region)
4. What's the durability requirement on writes? (→ chooses sync vs async for the home-region replication)
5. What's the read-after-write strategy for the writer themselves?
