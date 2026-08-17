# Distributed Transactions & Consensus — Interview Prep
---

## SECTION 1: LOCAL VS DISTRIBUTED TRANSACTIONS

### Q1. What's the difference between a local transaction and a distributed transaction, and why is the distributed case fundamentally harder?
**Answer:** A **local transaction** is scoped to a **single database/resource manager** — the DB's own ACID engine handles atomicity via its write-ahead log and locking, all in one process, with no network involved.

A **distributed transaction** spans **multiple independent resources** — two databases, or a database and a message queue, or two separate microservices each with their own database — and needs **all of them** to commit or **all of them** to roll back, as a single atomic unit, despite each being a separate process that can fail or become unreachable independently.

```
Local transaction:                  Distributed transaction:
BEGIN;                              Service A's DB   Service B's DB
  UPDATE accounts ...;                    |                |
  UPDATE orders ...;                 both must commit together,
COMMIT;  -- one WAL, one engine      but neither knows the other's outcome
                                      without explicit coordination
```

**Senior-level answer:**
> "The core difficulty isn't the 'commit' step — it's that in a distributed setting you lose the single point of truth a local transaction relies on. Each participant can independently succeed, fail, or become unreachable, and there's no shared memory to atomically flip a single bit that says 'committed.' Every distributed transaction protocol — 2PC, Sagas — is really just a different answer to 'how do independent parties agree on an outcome despite that.'"

---

### Q2. Why do most modern microservice architectures deliberately avoid distributed ACID transactions across services?
**Answer:** A few converging reasons:

- **Availability cost** — distributed transaction protocols like 2PC require **all participants to be up and responsive** for a transaction to complete (Q5) — this directly works against the "each service should fail independently" goal of microservices.
- **Coupling** — a distributed transaction spanning Service A and Service B's databases means A and B can no longer deploy, scale, or fail independently — it re-introduces the tight coupling microservices are meant to remove.
- **Vendor/technology lock-in** — XA (the standard distributed transaction protocol) requires **every participant** to support the XA interface — many modern data stores (Kafka, DynamoDB, most NoSQL stores) simply don't support it.
- **Latency** — a 2PC-style transaction requires multiple network round trips **while holding locks**, so it scales far worse than either purely local transactions or asynchronous alternatives.

**Senior-level answer:**
> "This is exactly why the Saga pattern (Q6) exists and has become the default answer in interviews — the industry's collective experience with distributed 2PC in microservice architectures (particularly in the 2010s as SOA/microservices scaled) was that it doesn't survive contact with real availability and independent-deployability requirements. I'd frame my answer as: 2PC isn't wrong, it's just optimized for a different environment — tightly coupled, same-datacenter, few participants (think a single service's own DB + a local message broker) — not for cross-service transactions at internet scale."

---

## SECTION 2: TWO-PHASE COMMIT VS SAGA

### Q3. Walk through the Two-Phase Commit (2PC) protocol step by step.
**Answer:** 2PC coordinates a **transaction coordinator** and multiple **participants** (resource managers) through two phases:

```
Phase 1: PREPARE (voting)                Phase 2: COMMIT (or ROLLBACK)
Coordinator -> "prepare?" -> Participant A   Coordinator -> "commit!" -> Participant A
Coordinator -> "prepare?" -> Participant B   Coordinator -> "commit!" -> Participant B
   Participant A: locks resources,             Participants: durably commit,
   writes to its own log, votes YES/NO         release locks, ack
   Participant B: votes YES/NO
   Coordinator collects ALL votes:
     - ALL YES → send COMMIT to all
     - ANY NO (or timeout) → send ROLLBACK to all
```

1. **Prepare phase** — the coordinator asks every participant "can you commit?" Each participant does everything short of actually committing — acquires locks, writes changes to a durable log — and replies **YES** (ready) or **NO** (abort).
2. **Commit phase** — if **all** participants voted YES, the coordinator tells everyone to **commit**; if any voted NO (or didn't respond in time), it tells everyone to **rollback**.

**Senior-level answer:**
> "The key property is that once a participant votes YES, it's made a **binding promise** — it must be able to commit later even if it crashes and restarts in between, which is why the prepare phase requires a durable log write, not just an in-memory lock. That durability requirement is exactly what makes 2PC expensive."

---

### Q4. What are the major weaknesses of 2PC?
**Answer:**
1. **Blocking on coordinator failure** — if the coordinator crashes **after** participants have voted YES but **before** sending the commit/rollback decision, participants are stuck holding locks indefinitely — they can't unilaterally decide to commit or abort because they don't know what the other participants voted. This is the protocol's most cited flaw.
2. **Synchronous, lock-holding, multi-round-trip** — every participant holds locks on its resources for the **entire duration** of both phases, across the network — this kills throughput and availability under load or latency.
3. **All-or-nothing availability** — the transaction cannot complete unless **every single participant** is reachable and responsive — one slow or down participant blocks the whole transaction, directly working against microservice availability goals (Q2).
4. **Not partition-tolerant** — a network partition between the coordinator and a participant mid-protocol leaves that participant blocked, unable to safely resolve on its own.

**Senior note:** "Three-Phase Commit (3PC) was proposed to fix the blocking problem by adding a 'pre-commit' phase and timeouts so participants can make progress independently — but it doesn't handle network partitions correctly either, and it's rarely used in practice. I'd mention it exists but explain why the industry moved to Sagas rather than 3PC as the practical answer."

---

### Q5. What is the Saga pattern? Explain choreography vs orchestration.
**Answer:** A Saga breaks a distributed transaction into a **sequence of local transactions**, each committed independently in its own service, with a **compensating transaction** defined for each step to undo its effect if a later step fails. There's no locking across services and no coordinator holding global locks — consistency is achieved **eventually**, through forward progress or compensation, not through an atomic all-or-nothing commit.

**Two coordination styles:**

| | Choreography | Orchestration |
|---|---|---|
| **How steps are triggered** | Each service publishes an event; the next service **reacts** to it — no central controller | A central **orchestrator** explicitly calls each service in sequence and tracks saga state |
| **Coupling** | Loose — services only know about events, not each other | Orchestrator knows about all participants; participants are simpler |
| **Best for** | Simple sagas, few steps | Complex sagas, many steps, need for visibility/monitoring of saga state |
| **Downside** | Hard to see/debug overall saga state — "callback hell" across services via events | Orchestrator becomes a critical component / potential single point of coordination logic |

```java
// Orchestration-style saga step (simplified) — orchestrator drives explicit steps + compensations
public class OrderSagaOrchestrator {
    public void execute(OrderSagaContext ctx) {
        try {
            paymentService.charge(ctx.getPaymentDetails());
            ctx.markStepComplete("PAYMENT");

            inventoryService.reserve(ctx.getOrderItems());
            ctx.markStepComplete("INVENTORY");

            shippingService.scheduleShipment(ctx.getOrderId());
        } catch (Exception e) {
            compensate(ctx);   // roll back completed steps, in reverse order
        }
    }
}
```

**Senior-level answer:**
> "I default to orchestration for anything beyond 2-3 steps, purely for observability — with choreography, when something goes wrong at 2am, you're reconstructing the saga's state by grepping event logs across five services; with orchestration, the orchestrator's persisted state tells you exactly where the saga stopped and what's already been compensated. That operational debuggability difference matters more in practice than the coupling argument most articles lead with."

---

### Q6. When would you actually choose 2PC over a Saga, if ever?
**Answer:** 2PC is still legitimate in narrower scopes:

- **Within a single trust/availability domain** — e.g., a service coordinating a write to its own database **and** a local message broker (like the transactional outbox alternative) where all participants are co-located, same datacenter, and expected to have correlated (not independent) availability.
- **Few participants, short-lived transactions** — 2PC's blocking risk and lock-holding cost scale badly with more participants and longer transaction duration; with 2 participants and a millisecond-scale transaction it's often fine.
- **When strict atomicity is non-negotiable and you control all participants** — some financial ledger systems still use XA/2PC internally for this reason.

**Senior-level answer:**
> "I wouldn't reach for 2PC across microservice boundaries in a 2026-era architecture — the industry consensus is pretty settled there. But I'd push back if an interviewer implies 2PC is simply 'wrong' — inside a bounded, single-team, same-infrastructure scope, it's a legitimate, simpler-to-reason-about tool than building saga compensation logic for a transaction that's genuinely a single atomic unit. The decision is about **trust and availability domain size**, not protocol age."

---

## SECTION 3: COMPENSATION TRANSACTIONS

### Q7. What is a compensating transaction, and what makes designing one harder than it sounds?
**Answer:** A compensating transaction **semantically undoes** the effect of a previously committed local transaction — it's not a database rollback (the original transaction already committed), it's a **new, forward-moving transaction** that reverses the business effect.

**What makes it hard:**
1. **Not everything is cleanly reversible** — "send confirmation email" or "call an external non-refundable payment API" can't be truly undone, only mitigated (send a follow-up cancellation email; issue a refund transaction rather than "un-charging").
2. **Compensations must themselves be idempotent and retryable** — the compensation is itself a distributed operation subject to the same network failure problems (Module 5's Network Failures topic) — it can be retried, so it must be safe to run more than once.
3. **Visibility to other systems/users in between** — if "reserve inventory" made a "3 left in stock" badge visible to other users before the saga later fails and compensates, those users saw a state that's now being retracted — compensations undo the **system's** state but can't undo **externally observed side effects**.
4. **Ordering matters** — compensations generally must run in the **reverse order** of the original steps, and must correctly handle "this step never actually completed" vs "this step completed and needs undoing."

**Senior-level answer:**
> "The mental model I use: a compensating transaction is not a rollback, it's an **apology** — a new transaction whose job is to make the business state consistent again, not to pretend the original action never happened. That framing is what makes people design compensations correctly — e.g., 'cancel and refund' instead of trying to literally un-charge a card, or 'release reservation' instead of trying to pretend the reservation window never showed reduced stock to another user."

---

### Q8. Walk through a real saga example — an order placement flow — including its compensations.
**Answer:**

| Step | Forward Action | Compensating Action |
|---|---|---|
| 1 | `PaymentService.charge(card, amount)` | `PaymentService.refund(transactionId)` |
| 2 | `InventoryService.reserve(sku, qty)` | `InventoryService.release(reservationId)` |
| 3 | `ShippingService.createShipment(orderId)` | `ShippingService.cancelShipment(shipmentId)` |

```
Happy path:      Charge -> Reserve -> Ship  ✓ ✓ ✓
Failure at step 3: Charge -> Reserve -> Ship FAILS
Compensation (reverse order):    Release reservation <- Refund payment
```

```java
private void compensate(OrderSagaContext ctx) {
    // reverse order — undo the most recent completed step first
    if (ctx.isStepComplete("INVENTORY")) {
        inventoryService.release(ctx.getReservationId());
    }
    if (ctx.isStepComplete("PAYMENT")) {
        paymentService.refund(ctx.getPaymentTransactionId());
    }
}
```

**Senior-level answer:**
> "The detail interviewers actually want here is the **reverse-order execution** and the fact that each compensation must be looked up by an **ID captured from the forward step's result** (the reservation ID, the transaction ID) — you can't compensate a step you don't have a durable reference to. I'd also flag: the saga's state (which steps completed) needs to be **persisted**, not held in memory, or a crash mid-saga leaves you with no way to know what to compensate — this is usually backed by a saga state table with a status per step."

---

## SECTION 4: LEADER ELECTION & REPLICATION

### Q9. Why do distributed systems need leader election, and what problem does a single leader solve?
**Answer:** A leader (single-writer) simplifies **ordering and conflict resolution** — all writes go through one node, so there's a single, unambiguous order of operations, without needing to coordinate conflicting concurrent writes across multiple nodes. Leader election is the process of the cluster **agreeing on which node holds that role**, and **re-agreeing** automatically when the current leader fails.

**Senior-level answer:**
> "A leader isn't strictly required — leaderless systems like Dynamo/Cassandra exist and handle writes on any replica — but a leader trades some availability (writes can't proceed if the leader's unreachable, until a new one's elected) for much simpler consistency reasoning. This maps directly back to the CP/AP conversation — single-leader replication is a natural fit for CP-leaning systems."

---

### Q10. Compare single-leader, multi-leader, and leaderless replication.
| Model | How Writes Work | Strength | Weakness |
|---|---|---|---|
| **Single-leader** | All writes go to one leader; replicated to followers | Simple conflict resolution, strong consistency achievable | Leader is a bottleneck/single point of failure until re-elected; failover has a gap |
| **Multi-leader** | Multiple nodes (often per-datacenter) accept writes, replicate to each other | Good for multi-region write availability, lower write latency per region | Write conflicts **can** occur between leaders and must be resolved (LWW, CRDTs, app-level) |
| **Leaderless** (Dynamo-style) | Any replica accepts writes; client (or coordinator) writes to N replicas directly, uses quorum reads/writes | Highest availability, no failover gap | Conflict resolution pushed to the client/read-path (vector clocks, read-repair) |

**Senior-level answer:**
> "I pick based on the **write pattern's geography and conflict tolerance**. Single-leader for anything needing strict ordering with a single region of truth (most OLTP systems). Multi-leader when you genuinely need low-latency writes in multiple regions and can tolerate/resolve conflicts — collaborative editing tools are the textbook case. Leaderless when maximum availability matters more than avoiding conflicts entirely, and you're willing to push resolution to the application, like Dynamo-style shopping carts."

---

### Q11. Sync vs. async replication — what's the trade-off?
**Answer:**
- **Synchronous replication** — the leader waits for one or more followers to **acknowledge** the write before confirming success to the client. Guarantees the data survives leader failure (a follower has it), but adds latency and reduces availability if a follower is slow/down.
- **Asynchronous replication** — the leader confirms the write immediately, replication happens in the background. Lower latency, higher availability, but a **leader crash before replication completes loses that write** — this is exactly the failover data-loss scenario (Q9's "gap").

**Senior note:** "Most production systems use **semi-synchronous** replication in practice — synchronous to at least one follower (so at least one durable copy exists beyond the leader), asynchronous to the rest — this is a direct real-world application of the quorum (N/W/R) tuning discussed in the CAP/consistency topic, and I'd connect the two explicitly if the interview allows it."

---

## SECTION 5: RAFT BASICS & CONSENSUS CONCEPTS

### Q12. What is the "consensus problem" in distributed systems, and why is it considered provably hard?
**Answer:** Consensus is the problem of getting a set of nodes to **agree on a single value** despite failures — e.g., "who is the leader," or "what is the next entry in the replicated log." The **FLP impossibility result** (Fischer, Lynch, Paterson, 1985) proved that in a **fully asynchronous** system (no bound on message delay), **no deterministic consensus algorithm can guarantee both safety and termination** if even one node can fail — you cannot distinguish a slow node from a dead one with certainty (this directly echoes the fail-stop vs fail-slow problem).

**Senior-level answer:**
> "The practical resolution isn't that consensus is impossible — Raft and Paxos both work in production every day — it's that real systems aren't purely asynchronous: they use **timeouts** as a practical (if imperfect) way to distinguish 'probably dead' from 'just slow,' accepting a small risk of getting it wrong rather than requiring theoretical certainty. Being able to name FLP and explain that nuance — rather than either overclaiming 'consensus is impossible' or not knowing it exists — is a good senior signal."

---

### Q13. Explain the basics of Raft — roles, terms, and how log replication works.
**Answer:** Raft is a consensus algorithm designed explicitly to be **more understandable** than Paxos, built around a **replicated log** and a single elected leader.

**Roles:**
- **Leader** — handles all client writes, replicates log entries to followers.
- **Follower** — passive; replicates the leader's log, responds to the leader's/candidates' RPCs.
- **Candidate** — a follower that hasn't heard from a leader within an election timeout, and starts an election.

**Terms:** Raft divides time into numbered **terms**, each starting with an election. At most one leader exists per term. Every message carries the sender's term number, so nodes can detect and reject messages from a stale, outdated leader.

```
Term 1: [Leader A] --- heartbeats ---> Followers
   (A crashes / partitioned)
Term 2: Followers time out -> election -> [Leader B] elected (majority vote)
   (A recovers, still thinks it's leader in Term 1)
   A's messages carry term=1 < current term=2 -> rejected, A steps down to follower
```

**Log replication:**
1. Client sends a write to the leader.
2. Leader appends it to its local log, sends `AppendEntries` RPCs to followers in parallel.
3. Once a **majority** of nodes have durably stored the entry, the leader considers it **committed** and applies it to its state machine, then acknowledges the client.
4. The leader includes the commit index in future heartbeats so followers eventually apply it too.

**Senior-level answer:**
> "The property that matters most in an interview is: an entry is only considered committed once a **majority** has it durably — this is the same quorum principle from Q11/the CAP module, applied specifically to log entries instead of arbitrary key-value writes. That's what guarantees no committed entry is ever lost even if the leader crashes immediately after — a majority already has it, and any future leader must also be elected by a majority, which guarantees overlap with at least one node that has the committed entry."

---

### Q14. How does Raft's leader election specifically prevent split-brain (two leaders in the same term)?
**Answer:**
1. A candidate must receive votes from a **strict majority** of the cluster to become leader — with an odd cluster size, only one candidate can possibly achieve a majority in a given term, since two disjoint majorities can't both exist.
2. Each node votes **at most once per term**, and votes only for a candidate whose log is **at least as up-to-date** as its own (this "election restriction" also prevents electing a leader that's missing already-committed entries).
3. Every RPC carries a **term number**; any node that sees a higher term than its own **immediately steps down** to follower and updates its term — this is exactly how a stale leader (like Node A in Q13's diagram) gets demoted once it can communicate with the majority again.

**Senior-level answer:**
> "This directly answers the split-brain prevention question from the Network Failures topic, but with the specific mechanism named — Raft doesn't need a separate fencing-token layer bolted on for leader election itself, because the term number **is** effectively a built-in fencing token: any message from an old term is provably stale and gets rejected by construction. That's a big part of why Raft is preferred over hand-rolled leader-election schemes — the safety argument is built into the core protocol rather than needing extra infrastructure."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is Saga eventually consistent or strongly consistent?" → Eventually consistent — there's a window during saga execution where the system is in an intermediate, partially-completed state visible to observers.
- "What's the 'outbox pattern' and how does it relate to distributed transactions?" → A way to atomically commit a local DB write and a "to be published" event in the **same local transaction**, then reliably publish it asynchronously — avoids needing 2PC between a DB and a message broker.
- "Can Paxos and Raft both solve the same problem?" → Yes, both solve consensus/replicated log agreement; Raft was explicitly designed to be more understandable/implementable than Paxos while providing equivalent safety guarantees.
- "What happens if a Raft leader is partitioned from the majority but not from a client?" → The client's writes to that leader can never commit (no majority ack) and will eventually time out — the leader can't make progress alone, which is exactly what prevents divergent state.
- "Is 2PC a consensus algorithm?" → Not quite in the formal sense — 2PC assumes a non-faulty coordinator and blocks on coordinator failure (Q4); Raft/Paxos are designed to make progress despite node failures, which is the actual consensus problem.
- "How many node failures can a 5-node Raft cluster tolerate and still make progress?" → 2 — it needs a majority (3 of 5) to elect a leader and commit entries.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) explaining **why** 2PC fell out of favor for cross-service transactions — availability and coupling, not "it's old" (Q2, Q4), (2) being able to walk through a saga's compensation logic with correct **reverse-order** execution and durable step tracking, not just naming the pattern (Q7–Q8), and (3) connecting Raft's majority-commit rule back to the same quorum principle used in replication and consistency tuning elsewhere — interviewers notice when a candidate treats these as one coherent idea rather than isolated facts (Q13–Q14). Be ready to sketch the term-number-as-fencing-token argument from memory — it's a strong, differentiated answer to "how does Raft prevent split-brain."*