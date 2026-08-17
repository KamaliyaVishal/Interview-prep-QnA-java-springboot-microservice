# CAP Theorem & Consistency Models — Interview Prep
---

## SECTION 1: CAP THEOREM FUNDAMENTALS

### Q1. What is the CAP theorem? Explain each of the three properties.
**Answer:** CAP theorem states that a **distributed data store** can only guarantee **two of three** properties simultaneously **during a network partition**:

- **Consistency (C)** — every read receives the **most recent write** or an error. All nodes see the same data at the same time (this is *linearizability*, not the "C" from ACID).
- **Availability (A)** — every request to a non-failing node receives a **response** (success or failure of the operation itself, but the system doesn't just hang/time out).
- **Partition Tolerance (P)** — the system **continues to operate** despite network partitions — messages between nodes being dropped or delayed.

```
        Consistency
           /  \
          /    \
         /      \
        /   CAP  \
       /  (pick 2) \
      /______________\
Availability    Partition Tolerance
```

**Senior trap to flag proactively:** People often say "pick 2 of 3" as if it's a free choice at all times — it isn't. Partitions **will** happen in any real distributed system (network cables get cut, switches fail, nodes get GC-paused long enough to look dead). So P isn't really optional — the actual choice you're making is **C vs A, specifically when a partition is happening**. This distinction is the single most common thing interviewers use to separate candidates who memorized the theorem from candidates who understand it.

---

### Q2. Why can't a distributed system guarantee all three properties at once? Walk through the proof intuition.
**Answer:** Consider two nodes, N1 and N2, replicating the same data, and a network partition occurs between them.

```
   Client A                          Client B
      |                                 |
      v                                 v
   [ N1 ] ---X (partition) X--- [ N2 ]
   write(x=5)                   read(x)?
```

- Client A writes `x = 5` to N1.
- The network between N1 and N2 is partitioned — N2 never receives the replication update.
- Client B reads `x` from N2. N2 must now choose:
  - **Return the stale value** (x = old value) → system stays **available**, but **not consistent**.
  - **Refuse to answer / block until the partition heals** → system stays **consistent**, but **not available**.

There is no third option — N2 physically cannot get the new value without communicating with N1, and by definition the partition prevents that communication. This is why **P is not a design choice** — it's a fact of distributed systems — and the real trade-off surfaces as **CP vs AP** the moment a partition occurs.

---

### Q3. Is "pick 2 of 3" actually how CAP works in practice, or is that an oversimplification?
**Answer:** It's an oversimplification, and I'd say that explicitly in an interview:

- CAP only constrains behavior **during an actual partition**. When the network is healthy, a well-designed system can offer both C and A simultaneously — there's no trade-off to make in the happy path.
- CAP says nothing about **latency**, which is often the trade-off that actually matters in production far more often than full network partitions (which are relatively rare vs. everyday slow/degraded links).
- This gap is exactly what **PACELC** (Q11) extends CAP to address — "if Partitioned, choose A or C; Else (normal operation), choose Latency or Consistency."

**Senior-level answer:**
> "CAP is a useful mental model for the failure-mode conversation, but I don't design systems around it directly — I design around PACELC, because in the 99.9% of the time you're not partitioned, the real trade-off is consistency vs latency, and CAP is silent on that."

---

### Q4. Give real examples of CP systems vs AP systems.
| Category | Example Systems | Behavior During Partition |
|---|---|---|
| **CP** (Consistency + Partition Tolerance) | ZooKeeper, etcd, Consul, HBase, MongoDB (default, single-primary) | Minority-side nodes **refuse writes/reads** (or return errors) rather than serve stale data — they sacrifice availability |
| **AP** (Availability + Partition Tolerance) | Cassandra, DynamoDB, Riak, CouchDB | All nodes **keep serving reads/writes** even on the minority side, accepting that some responses may be stale — reconciled later |

**Practical framing I'd give in an interview:** "CP systems are what you reach for when correctness matters more than uptime — leader election, distributed locks, configuration/metadata stores (ZooKeeper, etcd) — because two nodes disagreeing about who's the leader is catastrophic. AP systems are what you reach for when uptime matters more than perfect freshness — shopping carts, social media feeds, session stores — a slightly stale "like count" is fine; the site going down is not."

---

## SECTION 2: CONSISTENCY MODELS

### Q5. What's the difference between strong consistency and eventual consistency?
**Answer:**
- **Strong consistency** — after a write completes, **every subsequent read from any node** returns that write's value (or a newer one). Achieved via synchronous replication / consensus (e.g., Raft, Paxos) — the write isn't acknowledged to the client until enough replicas confirm it.
- **Eventual consistency** — after a write, replicas **converge to the same value over time**, but there's no guarantee about *when*. A read immediately after a write on a different replica may return a stale value.

```java
// Strong consistency: write blocks until quorum acknowledges
Response write = coordinator.write(key, value, ConsistencyLevel.QUORUM);
// any subsequent QUORUM read is guaranteed to see this write

// Eventual consistency: write returns immediately, replication is async
Response write = coordinator.write(key, value, ConsistencyLevel.ONE);
// a read on a different node moments later might still see the old value
```

**Senior-level answer:**
> "The trade-off is latency and availability vs. freshness guarantees. Strong consistency means every write pays the cost of coordinating with a quorum before it's acknowledged — that's higher write latency and reduced availability if a quorum can't be reached. Eventual consistency decouples the write acknowledgment from full replication, so it's faster and more available, but pushes the complexity of 'stale reads are possible' onto the application layer. I pick based on what a stale read actually costs the business — for a bank balance, unacceptable; for a product view counter, irrelevant."

---

### Q6. What are the different consistency models on the spectrum between strong and eventual? (High-value senior question)
**Answer:** Consistency isn't binary — there's a well-known spectrum, strongest to weakest:

| Model | Guarantee | Example |
|---|---|---|
| **Linearizability (strict/strong)** | Every operation appears to take effect instantaneously at some point between its start and end; a global real-time order exists | Distributed locks, leader election (etcd, ZooKeeper) |
| **Sequential consistency** | All nodes see operations in **the same order**, but not necessarily the real-time order they occurred | Distributed logs |
| **Causal consistency** | Operations that are **causally related** (a reply to a comment) are seen by everyone in the same order; unrelated operations may be seen in different orders | Comment threads, chat apps |
| **Read-your-writes** | A client always sees its **own** writes on subsequent reads, even if other clients see stale data | Session data, profile updates ("why don't I see my own edit?") |
| **Monotonic reads** | Once a client has seen a value, it will never see an **older** value on a later read | Avoids "time travel" glitches in a UI |
| **Eventual consistency** | No ordering guarantee at all — only that replicas converge **eventually** | DNS, S3 (historically), Cassandra default |

**Senior talking point:**
> "In real systems I rarely pick pure strong or pure eventual — I pick the **weakest model that still satisfies the product requirement**, because weaker models are cheaper and more available. A common real pattern is 'read-your-writes' for a user's own profile page combined with plain eventual consistency for everyone else's view of it — DynamoDB and Cosmos DB both expose this as a tunable per-request consistency level rather than a single global setting, which is the right way to think about it."

---

### Q7. Give a real-world example of an eventual-consistency bug, and how you'd mitigate it.
**Answer:** Classic case: a user updates their profile picture, immediately refreshes the page, and **still sees the old picture** because the read was routed to a replica that hadn't received the update yet.

**Mitigation strategies (know 2-3 well):**
1. **Read-your-writes via sticky sessions / read-from-primary-after-write** — route a user's reads to the same node/replica they just wrote to, or to the primary, for a short window after a write.
2. **Session tokens / version vectors** — client carries a "read at least version X" token from the write response; the read layer waits for a replica to catch up to that version before responding.
3. **Client-side optimistic update** — update the UI immediately from the write payload rather than re-reading from the (possibly stale) backend.

```java
// Version-token pattern: client presents the write's version, read waits for it
WriteResult result = repo.save(profile);
long writeVersion = result.getVersion();

// later, on read:
Profile p = repo.findByIdWithMinVersion(userId, writeVersion); // blocks/retries until replica catches up
```

**Senior note:** "This is exactly the kind of bug that looks like 'flaky' behavior in bug reports — 'sometimes my edit doesn't show, then it shows a minute later' — and recognizing it as an eventual-consistency symptom rather than an application bug is a good signal in an interview."

---

## SECTION 3: BASE

### Q8. What is BASE, and how does it contrast with ACID?
**Answer:** BASE is the consistency philosophy underlying most AP/NoSQL systems, explicitly trading strict correctness for availability and scalability:

- **Basically Available** — the system guarantees availability, in the CAP sense — it always responds, even if that response is degraded/stale.
- **Soft state** — the state of the system **may change over time, even without new input**, as replicas converge (background repair, anti-entropy).
- **Eventual consistency** — given enough time with no new writes, all replicas converge to the same value.

| | ACID | BASE |
|---|---|---|
| **Priority** | Correctness, strong consistency | Availability, scalability |
| **Consistency** | Immediate, transactional | Eventual |
| **Typical systems** | RDBMS (PostgreSQL, Oracle) | Cassandra, DynamoDB, Riak |
| **Failure handling** | Transaction rolls back, all-or-nothing | Conflicting writes reconciled later (LWW, vector clocks, CRDTs) |
| **Best fit** | Financial transactions, inventory counts | Feeds, caches, session stores, analytics |

**Senior-level answer:**
> "BASE isn't 'ACID but worse' — it's a deliberate design philosophy for a different problem shape. ACID assumes a single source of truth you can lock and coordinate around; BASE assumes many replicas that need to keep serving traffic even when some of them can't talk to each other, and accepts that correctness is restored asynchronously rather than instantly. The mistake I see junior engineers make is applying BASE-style eventual consistency to a domain that actually needed ACID guarantees, like account balances — that's how you get double-spends."

---

### Q9. How do you implement BASE-style conflict resolution in practice when two replicas diverge?
**Answer:** A few standard techniques, worth naming concretely:

1. **Last-Write-Wins (LWW)** — each write carries a timestamp; on conflict, the higher timestamp wins. Simple but **can silently lose data** if clocks skew or two writes race.
2. **Vector clocks** — track causality per replica; conflicting (concurrent, non-causally-related) writes are surfaced to the application to resolve (classic Dynamo/Riak approach — "sibling" values returned to the client).
3. **CRDTs (Conflict-free Replicated Data Types)** — data structures (counters, sets, sorted maps) mathematically designed so **concurrent updates merge deterministically without conflict**, no coordination needed.

```java
// CRDT-style grow-only counter merge — associative, commutative, idempotent
Map<String, Long> nodeCounts = Map.of("node1", 5L, "node2", 3L);
long mergedTotal = nodeCounts.values().stream().mapToLong(Long::longValue).sum();
// merging two replicas' counters is just summing per-node contributions — order-independent
```

**Senior note:** "LWW is the pragmatic default and what most people reach for first, but I'd flag in an interview that it silently drops data on true concurrent writes — for anything where losing a write is unacceptable (e.g., a shopping cart), CRDTs or explicit conflict surfacing to the user, like Dynamo's sibling resolution, is the more correct answer."

---

## SECTION 4: TRADE-OFFS & SYSTEM DESIGN APPLICATION

### Q10. In a system design interview, how do you decide whether a component should be CP or AP?
**Answer:** I ask: **"What's worse — serving stale/wrong data, or being unavailable?"**

| Choose **CP** when... | Choose **AP** when... |
|---|---|
| Data must be correct or the system shouldn't respond at all | Staleness is acceptable and uptime matters more |
| Examples: leader election, distributed locks, inventory/stock counts, payment processing, config/metadata stores | Examples: product catalogs, social feeds, view/like counters, session caches, DNS |
| Cost of a wrong answer > cost of downtime | Cost of downtime > cost of a slightly stale answer |

**Senior-level answer:**
> "I don't decide this at the whole-system level — I decide it **per component**. A typical e-commerce system is CP for the payment and inventory-decrement path (you cannot oversell the last unit or double-charge a card), but AP for the product catalog and recommendations (a stale 'in stock' badge on a listing page is annoying, not catastrophic). Treating the whole system as one CAP choice is a common mistake — the right granularity is per-service, sometimes per-endpoint."

---

### Q11. What is PACELC, and why does it matter more than CAP for most real-world decisions?
**Answer:** PACELC extends CAP to cover the (much more common) non-partitioned case:

> **P**artition — then choose **A**vailability or **C**onsistency; **E**lse (normal operation) — choose **L**atency or **C**onsistency.

| System | During Partition | During Normal Operation |
|---|---|---|
| DynamoDB / Cassandra | **PA** (favor availability) | **EL** (favor low latency) |
| MongoDB (default) | **PC** (favor consistency) | **EC** (favor consistency, waits for replication) |
| Google Spanner | **PC** | **EC** (uses TrueTime for strong consistency at cost of latency) |

**Senior-level answer:**
> "CAP gets all the interview attention, but partitions are actually rare compared to the everyday latency-vs-consistency trade-off a system makes on **every single request**. PACELC is why Cassandra can tune consistency **per query** via `ConsistencyLevel` — `ONE` for low latency, `QUORUM` or `ALL` for stronger guarantees — you're choosing your point on the EL/EC axis at the call site, not just once at design time."

---

### Q12. Explain quorum-based consistency (N, W, R) — how systems like Cassandra/DynamoDB make consistency tunable.
**Answer:** Quorum systems let you tune the consistency/availability trade-off **per operation** using three numbers:

- **N** — total number of replicas holding the data.
- **W** — number of replicas that must acknowledge a **write** before it's considered successful.
- **R** — number of replicas that must respond to a **read** before returning a result.

**The key rule:** if **W + R > N**, reads and writes are guaranteed to overlap on at least one replica → **strong consistency**. If `W + R <= N`, it's possible to read from a replica that missed the latest write → **eventual consistency**, but higher availability/lower latency.

```
N = 3 replicas
Strong consistency:  W = 2, R = 2   → W + R = 4 > N(3)  ✔ overlap guaranteed
Eventual (fast):     W = 1, R = 1   → W + R = 2 ≤ N(3)  ✘ no overlap guaranteed, but low latency
```

```java
// Example: Cassandra-style tunable consistency per query
session.execute(
    SimpleStatement.newInstance("SELECT * FROM orders WHERE id = ?", orderId)
        .setConsistencyLevel(ConsistencyLevel.QUORUM)   // R = majority of N
);
```

**Senior note:** "This is the single most practical piece of CAP-adjacent knowledge for real systems — most modern distributed databases don't force a global CP-or-AP choice at all, they expose N/W/R (or an equivalent consistency-level enum) so **each individual query** can choose its point on the trade-off. Knowing this distinguishes candidates who've actually operated Cassandra/Dynamo from those who've only read about CAP theoretically."

---

### Q13. Walk through a real design scenario applying these concepts.
**Answer:** *Example: designing a flash-sale/limited-inventory checkout system.*

- **Inventory decrement (CP)** — must use a strongly consistent store (or a single-writer pattern with a distributed lock/optimistic locking + version column) to prevent overselling the last N units. Latency cost here is acceptable — it's a low-frequency, high-value operation.
- **Product page views / "X people are viewing this" counter (AP)** — backed by an eventually consistent counter (even a CRDT counter) — being off by a few is irrelevant, and this path needs to survive massive read traffic without becoming a bottleneck.
- **Order confirmation email trigger (AP, at-least-once)** — fine to be eventually consistent / retried; duplicate sends are a minor annoyance, not sending one at all is worse — so I'd bias toward availability with idempotency keys to dedupe on the client/consumer side.

**Senior-level answer:**
> "I'd walk the interviewer through this component-by-component rather than declaring the whole system 'AP' or 'CP' — that's usually what they're actually listening for: can you apply the theorem as a design tool per data path, not just recite the definitions."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is MongoDB CP or AP?" → CP by default (single primary, majority write concern) — but it can be tuned toward AP-like behavior with weaker read/write concerns.
- "What's the difference between eventual consistency and BASE?" → Eventual consistency is one property; BASE is the broader philosophy (basically available + soft state + eventual consistency) that eventual consistency is part of.
- "Can a single-node database violate CAP?" → No — CAP only applies to distributed systems where partitions between nodes are possible; a single node has nothing to partition from.
- "What does 'consistency' in CAP have to do with ACID's 'C'?" → Different concepts sharing a name — CAP's C means linearizability (all nodes agree on the latest value); ACID's C means the database's integrity constraints (e.g., foreign keys) are never violated.
- "Is Kafka CP or AP?" → Configurable — `acks=all` with `min.insync.replicas` tuned appropriately leans CP; `acks=1` leans AP/lower-latency.
- "What's a split-brain scenario?" → When a partition causes **two nodes to both believe they're the primary/leader** and accept writes independently — a classic CP failure to guard against via quorum-based leader election (Raft/Paxos), not simple heartbeats.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates from mid-level ones on this topic are (1) correctly framing CAP as "P is mandatory, the real choice is C vs A during a partition" rather than "pick any 2 of 3" (Q1–Q3), (2) knowing PACELC and being able to explain why it matters more day-to-day than CAP itself (Q11), and (3) applying the CP/AP choice **per component** in a system design scenario rather than to the whole system (Q10, Q13). Practice walking through Q12's N/W/R quorum math out loud — it's the concept most candidates can recite but few can actually compute on the spot.*
