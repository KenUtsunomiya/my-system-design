# Interview meta — procedural rules

Cross-problem reference: habits and procedural moves that compound across all system design interviews, independent of the specific problem. Consult before any new interview prep session as a quick warm-up.

---

## Name your load-bearing operational assumptions

If an architectural decision **depends on** operational maturity (CI/CD, managed services, team size, etc.), state the assumption explicitly out loud — once, briefly. The interviewer can then accept it or redirect ("actually, assume X is not available"). Silent assumptions are how candidates lose senior signal: a beautiful design quietly depending on capabilities the team doesn't have.

**Rule of thumb:** if the *negation* of an assumption would change your design, name it. If it wouldn't, skip it.

**One-sentence form:**

> "I'm assuming [assumption], so I'm choosing [design decision]. If that's not the case, I'd instead [alternative]."

### Common load-bearing assumptions to surface

| Assumption | When it's load-bearing |
|---|---|
| Modern CI/CD (canary deploys, automatic rollback, feature flags) | Whenever you appeal to deployment safety to justify a topology choice (e.g., merging services for simplicity, deploying frequently) |
| Cloud / managed services available | Whenever you say "use DynamoDB / SQS / Kafka / Cloudflare" — implies you don't have to operate them yourself |
| Existing platform services | Auth, rate limiting, observability, secrets — they exist and you're not designing them |
| Team size / on-call capacity | Whenever you justify operational complexity (e.g., "we can run a sharded multi-region store") |
| Engineering org structure | Whenever Conway's law affects splitting (one team → merge; many teams → split paths to teams) |
| Network constraints | mTLS, VPC boundaries, cross-region pipes — relevant in regulated environments |
| Compliance / data residency | Whenever the design touches user data subject to GDPR / HIPAA / regional storage rules |

### What NOT to do

- **Don't design the pipeline.** CI/CD is not a deliverable in a system design interview unless the interviewer specifically asks.
- **Don't dwell.** One sentence per assumption. If the interviewer doesn't push, drop it and move on.
- **Don't list them all.** Surface only the ones whose negation would change *this design*. Listing every possible assumption is padding.

---

## Push back on the interviewer when you have an argument

Interviewers reward candidates who push back with substance. Pushing back is not adversarial — it's how you demonstrate you can defend trade-offs and aren't just executing instructions.

**The form that works:**

1. **State the principle** you're applying (cite it explicitly if it has a name — e.g., "premature distribution," "add complexity only when forced").
2. **Apply it to the specific case** with concrete numbers / characteristics.
3. **Acknowledge the cost** of your position so the interviewer sees you've considered the trade-off, not just one side.

**The form that doesn't work:**

- Pushing back on style ("I prefer X") without a principle behind it.
- Pushing back without acknowledging the counter-argument.
- Pushing back twice on the same point after the principled answer is given.

When the interviewer concedes, it's a strong positive signal. When you concede after a substantive exchange, that's *also* a strong signal — flexibility under good arguments is senior behavior, stubbornness is not.

---

## State trade-offs, don't dodge them

Every architectural choice has a cost. Beginners describe choices as if they were obviously correct; seniors describe them as the better of two named alternatives.

**Junior phrasing:** "I'll use Postgres."
**Senior phrasing:** "I'll use Postgres because we need transactional updates on the alias-collision path. The cost is that horizontal write-scaling is harder than with Cassandra; I'm betting we won't need it at this scale."

The pattern: *choice + named alternative + reason for the choice + cost paid.*

Apply this to every load-bearing decision: SQL vs NoSQL, sync vs async, push vs pull, strong vs eventual consistency, sharding strategy, cache eviction, etc.

---

## Treat clarifying-question answers as a checklist

After the clarification phase, every confirmed in-scope capability must show up later as:

- A **functional requirement** (§2) — if it's user-observable
- A **non-functional target** (§3) — if it's about scale/latency/availability/consistency
- A **component** (§6) or a **deep-dive** (§8) — if it's an architectural choice

Common failure mode: clarifying questions surface a requirement, the candidate writes it down on the whiteboard, and then forgets to thread it through the design. Senior interviewers will notice.

**Habit:** when drafting a section, scroll back to §1 and tick each clarification answer against the section you're writing. Anything that doesn't fit there should fit *somewhere*.

---

## Section purposes — keep them separate

Each section in the standard arc has one job. Mixing them is sloppy:

| Section | Job | Doesn't contain |
|---|---|---|
| §1 Clarification | Scope and assumptions | Design choices |
| §2 Functional reqs | What the user observes | Implementation details, internals |
| §3 Non-functional reqs | Targets (numbers, -ilities) | Math; design |
| §4 Capacity estimation | Math that justifies §3 | Design choices; new requirements |
| §5 API design | External interface | Internals; storage; auth implementation |
| §6 High-level architecture | Components + request flow | Field-level data model; deep algorithm specifics |
| §7 Data model | Schema, keys, indexes, partition key | Component topology |
| §8 Deep dives | The 1–2 hard subsystems | Re-litigating earlier sections |
| §9 Trade-offs / bottlenecks | What breaks at 10×, what was traded away | New design |
| §10 Follow-ups | Honest list of gaps and deferrals | New requirements |

If you find yourself doing §4's job in §3, or §7's job in §6, stop and move it. Auditability suffers when sections aren't crisp.
