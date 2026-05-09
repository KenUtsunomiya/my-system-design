# URL Shortener — interviewer notes

Section-by-section feedback, corrections, and learnings. Cross-references `solution.md` by section number. The metacognitive layer: *why* the design ended up the way it did, *what mistakes were made along the way*, *what to internalize for next time*.

Legend: ✅ strong / ⚠️ weak or low-impact / ❌ missed / 🌟 senior-grade move / 🎯 takeaway for next time.

---

## Section 1 — Problem statement & clarifying questions

- ✅ Q1, Q2 are well-formed scoping questions. Good instinct to nail down "who uses it" and "what's in/out of scope" before anything else.
- ⚠️ Q3 ("what's in the URL") is a low-impact detail question — the answer didn't change anything in the design. In a real interview, prefer questions whose answer would *change the design.*
- ⚠️ "How many users" is a fine starting point, but **users is rarely the unit that drives architecture.** Always follow up with: write rate, read rate, read/write ratio, growth per unit time. Train yourself to think in those terms.
- ✅ Q5 (custom codes) was the right question to close on — it's the one whose answer most directly shapes the ID-generation deep dive later.
- 🎯 **Takeaway for next time:** when generating clarifying questions, rank them by "how much would the answer change my design?" and ask the highest-leverage 3–5.

---

## Section 2 — Functional requirements

- ⚠️ First draft mixed an implementation detail ("globally unique short URL") into the FR list. **Functional requirements describe user-observable behavior, not how the system achieves it.** Uniqueness is our problem, not the user's.
- ❌ First draft missed two FRs that the clarifying answers had already established (TTL/expiration; deletion by registered users). **Treat clarifying-question answers as a checklist when drafting FRs** — anything the interviewer confirmed in scope must appear here.
- ⚠️ First draft duplicated the entire flow for registered users. Prefer "core flow + delta for registered users" — easier to read, and shows you understand registration as an *additional capability*, not a separate product.
- ✅ Good instinct on Out-of-scope after prompting. Remember its two purposes: **defensive** (prevents interviewer ambushes mid-deep-dive) and **demonstrative** (shows you noticed adjacent problems and consciously deferred). Don't pad it — 3–5 items is enough.
- 🎯 **Takeaway for next time:** before writing FRs, glance at your clarifying-question answers and turn each user-facing capability into a bullet. Then strip anything that mentions implementation.

---

## Section 3 — Non-functional requirements

- ❌ First draft set **symmetric SLOs** for reads and writes. The single biggest senior signal in NFRs is **noticing the hot path** and giving it tighter numbers. URL shortener is 100:1 read-heavy, so **redirect is the hot path** and gets tighter latency / availability budgets than creation. Internalize this as a habit: *"is one path 10× more important than the other? say so explicitly."*
- ⚠️ Initial redirect latency budget (100 ms p99) was loose for a redirect. Users notice 100 ms before a link resolves; real shorteners are sub-50 ms. **Calibrate budgets against what users perceive on the critical path**, not against generic "fast web" numbers.
- ❌ Durability statement was self-contradictory: "replicated in real time" but "could lose data before replication". Sync vs async replication is a real, named trade-off — pick one and state the loss window explicitly. Mumbling between them is the kind of thing that loses senior signal fast.
- ⚠️ "At most N replicas" sounds like a cost cap; durability targets are stated as **minimums** ("at least N replicas"). Mind the direction.
- ⚠️ Capacity arithmetic was inlined into NFRs. Section 3 = *targets*; Section 4 = *math that justifies them*. Keep them separate so the reader can audit each layer independently.
- ✅ Good instinct on multi-region thinking and explicit cross-region staleness budget; that level of specificity is uncommon for a beginner.
- 🌟 The **strongly-consistent alias-collision check inside an otherwise eventually-consistent system** is exactly the kind of nuance senior interviewers reward. Most of the system tolerates eventual consistency, but *one specific operation* cannot. Steal this pattern — almost every system you design will have at least one operation like this.
- 🎯 **Takeaway for next time:** before writing NFRs, ask *"is there a hot path?"*. If yes, write SLOs in two columns (hot vs cold) and state the asymmetry as a deliberate choice.

---

## Section 4 — Capacity estimation

- ✅ QPS and storage math are clean and well-rounded. Strong instinct on aggressive rounding.
- ❌ **Bandwidth had a 1000× unit error** in the first draft (`3500 × 500B` written as 1750 MB/s instead of 1.75 MB/s). 1000× errors change architectural conclusions completely (5 MB/s = trivial; 5 GB/s = mandatory CDN). **Always sanity-check by converting to a familiar unit:** "is this the kind of number a single machine could do?" A modern NIC handles 1–10 Gbps; if your math says you need a CDN for 5 MB/s, redo the math.
- ❌ Cache sizing mixed two concepts: (1) **cache hit ratio** (target high — 80–95%) is not the same as (2) **fraction of data stored in cache**. The 80/20 rule is about *traffic*, not *storage* — 80% of reads hit ~20% of URLs, but the *hot working set* is much smaller than 20% of all data because URLs cool off fast. Real-world: cache the recent + trending tail, target hit rate, let LRU find the working set.
- ❌ The **Implications** line was skipped. This is the single most important line in §4 — numbers without implications are trivia. The whole point of capacity estimation is to *constrain the design that follows.* Always answer: do I need a cache? sharding? a CDN? what grows fastest?
- 🌟 Good follow-up question on CDN vs cache and "what other layers exist" — that conceptual curiosity will compound. Saved the layered cache model and the senior toolkit to `wisdom/`.
- 🎯 **Takeaway for next time:** end §4 with a 3–5 line "Implications" block that explicitly says what the numbers *force* and what they *don't* force. That block is what the interviewer reads to decide whether you're ready for §6.

---

## Section 5 — API design

- ❌ First draft had **endpoint proliferation**: three creation endpoints (`POST /codes`, `POST /codes/custom`, `POST /codes/expiration`). The senior reflex is **one endpoint per resource action; optional fields for variants** — this scales cleanly to new optional features ("custom alias + TTL"), keeps the API surface small, and matches how Stripe/bit.ly/etc. design real APIs. The cost paid by splitting is forever-cost (docs, monitors, rate limits, SDK methods, versioning); the cost saved is rounding-error CPU.
- ❌ DELETE-with-body shape (`DELETE /codes` + `{code}` in body) is technically allowed but considered bad practice — many proxies/CDNs strip DELETE bodies. The resource identifier belongs in the **path**: `DELETE /shortlinks/{code}`.
- ⚠️ Returned only `code`, not the full `short_url`. The user-facing artifact is the **shareable URL**, not the slug. Convention: return both `code` (canonical id) + `short_url` (what the user shares) — saves clients from constructing the URL and gives you flexibility to change the redirect host without breaking them.
- ❌ Idempotency was treated as a single concept. Two distinct concepts hide here: **product-level** (does dup-URL → dup-code?) and **retry-level** (can the client safely retry a timed-out POST?). The latter requires an `Idempotency-Key` header — Stripe-pattern. State both explicitly.
- 🌟 **Strongest moment of the section: pushing back twice on the merged-endpoint design with substantive arguments** (body-parse overhead, then AuthN-vs-controller separation of concerns). The second argument was sharp and forced an exact response. **Pushing back on the interviewer is exactly what senior signal looks like** — but always pair the pushback with awareness of the trade-off you're paying. The resolution: AuthN is identity (middleware-only); AuthZ is identity × intent (lives in a policy layer post-parse, not in the controller). Splitting endpoints "for AuthN cleanliness" actually moves AuthZ from policy code to routing config, which scales worse as the auth-required feature set grows.
- 🌟 Good instinct on JWT in `Authorization: Bearer` and using 401/403/409 distinctly — many candidates collapse them.
- 🎯 **Takeaway for next time:** when designing APIs, ask *"is this a new resource action or a variant of an existing one?"* before adding an endpoint. Variants → optional fields. New resource actions → new endpoints. The smell of three endpoints with overlapping behavior almost always means "merge."

---

## Section 6 — High-level architecture

- ❌ First component list **forgot the CDN** despite §4 implications explicitly flagging it as motivated by latency. **Whenever drafting §6, scan §3 NFRs and §4 Implications and verify each commitment maps to a component or a property of one.** This is the most common mistake in §6.
- ❌ First list had a **single Redis instance** — a SPOF that contradicts the 99.99% redirect SLO. *Single-instance anything in a 4-nines design is a smell.*
- ❌ First list had **compute in only one region** with cross-region DB replication. *Compute follows data*; placing data in many regions but compute in one defeats both the latency and availability story for users far from the compute region.
- 🌟 **Strongest move of the section:** pushing back on the read/write service split using `wisdom/capacity.md`'s "add complexity only when forced" principle by name. Citing the principle is the move that makes the level read jump in a real interview.
- 🌟 Asked *"do all regions have their own DB?"* — exactly the right question, surfaced a load-bearing topology choice (single-leader vs. multi-leader vs. sharded vs. globally-distributed) that was implicit in the diagram. Saved the four-option decision framework to `wisdom/multi-region-databases.md`.
- 🌟 Asked the **CI/CD assumption question** — exactly the kind of meta-procedural awareness senior interviewers reward. Saved the "name your operational assumptions" rule to `wisdom/interview-meta.md`.
- ⚠️ The first walkthrough for the write path silently assumed the writer was in the home region. **Cross-region write latency is a real cost of single-leader topology and should be visible in the walkthrough**, with an explicit "if traffic shifts, this is the upgrade path" note for §10.
- 🎯 **Takeaway for next time:** before drafting §6, do a 60-second audit: (a) re-read §3 and §4 Implications; (b) for each commitment, ask "what component or property enforces this?"; (c) any commitment without a component is a hole. Most §6 mistakes are commitments-not-threaded-through-the-design.

---

## Section 7 — Data model

- ❌ First draft skipped the **SQL-vs-NoSQL articulation**. Even when the answer is "obviously Postgres," the senior move is to state the choice + named alternative + why this won. Not stating it is the smell — the interviewer can't tell if you understood the trade-off or just defaulted.
- ❌ `code: varchar(8)` was a **§5 ↔ §7 consistency bug**: §5 allows custom aliases up to 32 chars, so an 8-char column truncates them. **Before committing the data model, scan the API spec for max field lengths and copy them to the schema.** Senior interviewers spot this instantly.
- ❌ "No effective partition available" silently committed to a single-node DB forever. **Even when not sharding on day one, name the partition key you'd use when forced.** For URL shortener: `hash(code)` — every read and write specifies `code`, so it's a perfect shard key.
- ⚠️ Datetime type was vague (`datetime`). In a global multi-region service, **always `timestamptz`** — it stores UTC under the hood and avoids DST and timezone bugs at the data layer.
- ⚠️ Nullability was not specified for `expiration` (most rows have no TTL → NULL) or `creator` (anonymous creates → NULL). Be explicit about NULL semantics — they're load-bearing.
- ⚠️ `creator` data type was a placeholder. Commit: `uuid` if Auth issues UUIDs (modern norm). Saved cross-cutting type wisdom to `wisdom/data-types.md` (binary UUID vs CHAR(36); UUIDv7 vs UUIDv4 for index health; the top-5 type gotchas).
- ✅ Excellent discipline on **single table, no speculative columns, indexes only for actual queries, hard-delete justified by concrete reasoning**. This is exactly the lean schema a senior interviewer wants — many candidates over-decompose into 4 tables for what's a 1-table problem.
- ✅ Idempotency-key store correctly placed in Redis (TTL'd KV is exactly the right primitive).
- 🎯 **Takeaway for next time:** before committing §7, run a 30-second audit: (a) does every column appear somewhere in §5 or §6? (b) does each `VARCHAR(n)` match the API max length? (c) have I named the shard key, even if deferred? (d) have I committed to `timestamptz` and explicit NULL/NOT NULL? Most §7 misses are caught by this checklist.

---

## Section 8.1 — Custom-alias collision under concurrent writers

- ✅ Got the core algorithm right on the first pass: INSERT relying on the PK unique index, no TOCTOU SELECT-then-INSERT, no application-level locking. Many candidates flounder here and reach for distributed locks or pessimistic gap locks; you didn't.
- ✅ Correctly identified `Idempotency-Key` as the mechanism for retry-safety on dropped responses. The connection between §6's design choice and §8's deep-dive is exactly what binds the writeup together.
- ⚠️ Mentioned **isolation level (READ-COMMITTED) as the reason uniqueness holds** — that's a misdirection. The unique-constraint mechanism is **independent of isolation level**; it operates at the B-tree storage layer below MVCC. Reaching for "transaction isolation" to explain uniqueness is a tell that you may not fully separate the two concerns. The senior framing: *"the unique index serializes concurrent INSERTs at the storage layer; this works at any isolation level."*
- ⚠️ "Safe to retry" on network-drop and failover was stated unconditionally — but it's safe **only with an `Idempotency-Key`**. Without one, the client can't distinguish "didn't happen" from "happened but ACK lost." Always pair retry-safety claims with the precondition that makes them true.
- ❌ Missed **case sensitivity** as an edge case. For user-facing identifiers, case-insensitive is almost always the right default (case-insensitive comparison + lowercase storage). Forgetting this leads to user-visible bugs ("MyLink exists but mylink appears available?").
- ❌ Missed **cross-keyspace collision** (custom alias vs system-generated code). Worth saying explicitly that they share the same `code` column and unique index — same primitive, both directions.
- ⚠️ Performance answer was generic ("queue grows, latency rises"). For deep-dives, engage with the *specific* contention shape: failed inserts are sub-ms, legitimate writes don't contend, contention only on same-key, bot squatting bounded by gateway rate limit, eventual ceiling is single-primary capacity. **Generic perf framing in a deep-dive is a missed opportunity** — this is the one section where being specific about your workload beats being general.
- 🌟 Identifying the reserved-words requirement at the validation layer (not as a DB constraint) was a senior touch — config-driven, no migration to update.
- 🎯 **Takeaway for next time:** in a deep-dive, after stating the algorithm, ask yourself *"what edge cases would a senior reviewer ask about?"* and walk through 4–6 of them explicitly. Case sensitivity, retry-safety preconditions, cross-keyspace overlap, performance under specific abuse patterns — these are the things that separate a clean answer from a complete one.

---

## Section 8.2 — Short-code generation

- ❌ First swing picked **hash-of-URL + truncate**, which contradicts §5's explicit decision that the same URL submitted twice should return *different* codes. **The §5 ↔ §8 consistency check is mandatory** — when picking an algorithm in a deep-dive, scroll back through earlier sections and verify it doesn't violate any committed product behavior. Hash-determinism would also have leaked URL-existence info to anyone who knows the URL (privacy concern beyond enumeration).
- ⚠️ The first collision-rate calculation labeled `0.0017` as "the collision rate" — numerically correct as a per-insert probability for the fill ratio, but the labeling was loose. In Step 2 a unit conversion error (`0.0017 → 0.017%` instead of `0.17%`) showed up. **Off-by-10x in unit conversions is the kind of mistake a senior spot-checks instinctively.** Practice the decimal-to-percent move (multiply by 100) until it's automatic.
- 🌟 **Pushed back when the interviewer demanded math that wasn't load-bearing.** Asking *"is the birthday paradox really required for this decision?"* was correct — the per-insert rate alone is sufficient to choose the algorithm; lifetime cumulative collision math is interesting but doesn't move the choice. **Senior calibration: when math doesn't change the decision, push back on being asked for it.** This was the strongest move of the section.
- ❌ Threshold for keyspace saturation set at "row count hits 3.5T" — far too late. By that point the system has been broken under retry pressure for years. **Trigger from the retry-rate metric (~1% threshold), not from row counts.** Monitoring is the load-bearing piece; it gives decades of lead time at our growth rate.
- ❌ Migration plan suggested *"adding a fixed char prefix to existing codes"* — this is catastrophic. Mutating old codes 404s every short URL ever shared in the wild, violating the implicit "links work forever" product contract. **The right migration is no migration**: feature-flag the new length, old codes stay forever, new codes use the larger keyspace, schema unchanged.
- ✅ Got the algorithm structure right (bounded retry, regenerate on `23505`) on Step 1.
- ✅ Got the predictability framing right on Step 3 — "attackers can guess the pattern" → enumeration. Sharpened to the senior framing: *"predictable codes turn an unenumerable namespace into an enumerable one."*
- ✅ Got the §8.1 composition right in one sentence — *"shared keyspace, no special handling."* That's the senior version of "I understood the design."
- 🎯 **Takeaway for next time:** for any algorithm choice in a deep-dive, run a four-step gate: (a) does it satisfy every committed property in earlier sections? (b) what does it leak that the alternative doesn't? (c) what's the operational signal that tells me to migrate, and how much lead time does it give? (d) does the migration preserve external contracts (URLs, IDs, references that have escaped the system)?

---

## Section 9 — Bottlenecks (part 1: what breaks first)

- ✅ **Got the right core answer:** single-primary write path is the architectural choke point, but at 1K writes/sec under 10× it's still in headroom — the honest senior take is "nothing breaks at 10×." Many candidates manufacture a bottleneck because they assume the question expects one; not doing that is senior signal.
- ❌ **Initial draft claimed "cache hit rate decreases because write QPS is 10×ed."** The causal chain is wrong: cache hit rate is governed by working-set size and access locality, not by write volume. Writes populate the cache (write-through) but don't evict useful entries — LRU pushes out cold ones. The mental model to internalize: **cache pressure tracks unique-key access patterns, not total write volume.**
- ❌ **Misconception about read replicas.** Initially assumed replicas are "just stand-by servers for failover" — meaning all reads (including cache misses) would route to the primary. This would have made the primary the read-path bottleneck. **Replicas serve TWO purposes simultaneously: HA/failover AND read scaling.** Sync vs async is about durability/staleness, not about whether replicas serve reads. Saved the corrected mental model to `wisdom/replication.md`.
- ⚠️ **Missed the cross-region write latency tail** as the *actual* discomfort at 10×. At 10× more writers cross 150–300 ms to reach the home-region primary; same per-writer cost but more of the distribution sits in the p99 tail, pressuring the 200 ms creation SLO. "Nothing breaks" is right; "and here's what gets uncomfortable" is the senior-grade follow-up.
- ⚠️ **Missed naming the architectural ceiling at ~50–100×.** Senior interviewers want *"and here's what would break it"* — names the load level where the topology pivots. For us, ~10K write QPS forces escalation off Option 1 (single-leader async) to sharded-by-code or globally-distributed.
- 🌟 **Best move of the section: pushed back on interview-time calibration.** Asking *"do you really need to compute every component? I don't think it's realistic in 45 minutes"* was correct — and it surfaced a deeper structural issue (interviewer feedback was conflating study-time depth with interview-time delivery), which led to splitting `solution.md` and `notes.md`.
- 🎯 **Takeaway for next time:** in a real interview, the §9 answer is **3 sentences**: what's mostly fine, where the real pressure is, and the architectural ceiling. The component-by-component table belongs in the writeup, not the interview transcript. Train both modes — depth for the writeup, terseness for the spoken answer.

---

## Section 9 — Trade-offs (part 2: what was given up)

- ✅ Got the **X/Y/Z form** right structurally on all three trade-offs (chose X / paid Y / accepted because Z). That's the senior shape; many candidates state just X without naming the cost.
- ❌ **Trade-off #1 (RDB vs NoSQL): cost was mislabeled as "latency."** Postgres is *not* slower than DynamoDB/Cassandra at our access pattern (PK lookup, single-row INSERT) — in fact, Postgres is often faster (< 1 ms vs 5–10 ms typical). The actual cost is **harder horizontal write-scaling** (single primary's ~10K-QPS ceiling forces sharding when crossed). The justification is **unique-constraint enforcement for alias collision**, not generic "strong consistency."
- ⚠️ **Trade-off #2 (cross-region eventual consistency): framing undersold the alternative.** "Covers most use cases" doesn't show why we made the trade. The senior version names what the *alternative* would have cost: synchronous cross-region replication adds ~100–300 ms to *every* write commit globally — making the 200 ms creation p99 unachievable. **Compare X to the alternative cost we avoided**, not to a generic "good enough" floor.
- ❌ **Trade-off #3 (merged service): cost was mislabeled as "scaling independently."** At our scale (100:1 read/write ratio), the merged service auto-scales on read load and writes ride along — there's no efficiency cost from co-location. The real cost is **deploy blast radius** (a write-path bug deploys onto the read fleet's hot path). The justification is that **modern CI/CD handles blast radius in software**, not in topology. Also: the 100:1 ratio is a *consequence* of choosing merged, not the *reason* for it.
- 🎯 **Pattern across all three:** **cost-naming is precision-load-bearing.** A trade-off that names a wrong cost looks superficially senior but actually reveals shallow understanding. The interviewer probes the cost — and the wrong cost falls apart under one follow-up question. Practice naming costs *specifically* (not "latency" but "harder horizontal write-scaling"; not "scaling independently" but "deploy blast radius"). Specific costs survive interviewer probing; generic costs don't.
- 🎯 **Takeaway for next time:** for every trade-off X/Y/Z, ask yourself *"would the interviewer's first follow-up question expose this Y as wrong?"* If yes, sharpen Y. The follow-up question for "we paid latency" is "Postgres PK lookup is sub-millisecond — what latency are you paying?" — and the cost-naming collapses.

---

## Section 10 — Follow-ups

- 🌟 **The user noticed that §10 was already pre-populated** by inline flags throughout earlier sections — *"We've touched most of them. What should I do further here?"* That observation is a senior signal: a well-structured design surfaces gaps as it goes, so §10 is consolidation, not invention. Many candidates write §10 from scratch and either repeat themselves or miss the items they already flagged.
- ✅ Once consolidated, the list ran 13 items — too long. Recommended cut to 7 was accepted. **The senior move is to rank by "likelihood of causing a real production incident if ignored"** and cut the rest, with the cut items briefly named so the interviewer knows you considered them.
- 🎯 **Takeaway for next time:** as you work through §1–§9, **proactively flag items "for §10"** when you defer or out-of-scope something. Then §10 is a 5-minute consolidation pass: collect the inline flags, add 1–2 truly new gaps, rank by operational risk, cut to 6–8. Don't draft §10 from a blank page.

---

## Meta — interview-time vs study-time

A late-emerging confusion: I was prompting with study-time depth ("walk through all 6 components and compute the load on each at 10×") in a frame that should have been interview-time ("name the bottleneck in 3 sentences"). The user pushed back during §9 — *"do you really think I need to calculate for all components? I don't think it's realistic to finish it within 45 minutes of interview"* — and was right.

Lesson: keep the two modes distinct.

- **Study-time** (`solution.md` and `notes.md`) — exhaustive, audit-style, every commitment threaded through. Optimized for re-reading months later.
- **Interview-time** (`interview-45min.md`, produced at the end) — terse, critical-path only, skip what's obviously fine. Optimized for being *spoken* in 45 minutes.

These are different deliverables for the same design. Don't conflate them.
