# Design Trade-offs — Interview Prep
---
*This module is a synthesis/quick-reference for system design interviews — each topic here has deeper treatment elsewhere (cross-referenced below); this module's focus is the **decision framework**: what to actually weigh, and how to state the trade-off out loud under interview pressure.*

---

## SECTION 1: SQL vs NoSQL

### Q1. SQL vs NoSQL — how do you actually decide, beyond "NoSQL scales better"?
**Answer:** The real decision hinges on **data shape, consistency needs, and query patterns** — not raw scalability, which is a common oversimplification.

| Aspect | SQL (Postgres/MySQL/Oracle) | NoSQL (MongoDB/Cassandra/DynamoDB) |
|---|---|---|
| Schema | Fixed, enforced at write time | Flexible/schemaless — enforced (if at all) at the application layer |
| Relationships | Native JOINs, strong referential integrity | Denormalized by design — joins are expensive/avoided, data duplicated across documents |
| Consistency | Strong (ACID) by default | Often tunable — many NoSQL stores default to eventual consistency for availability (Q5) |
| Scaling | Vertical first, horizontal (sharding) is harder and often manual | Horizontal scaling built in from the ground up (partitioning is a first-class concept) |
| Query flexibility | Ad-hoc queries via SQL, indexes on any column | Query patterns often need to be known **upfront** — data modeled around access patterns (especially true for Cassandra/DynamoDB) |
| Best fit | Transactional data with real relationships and invariants (orders, payments, inventory) | High-write-throughput, flexible/evolving schema, massive horizontal scale (event logs, session data, product catalogs with varying attributes) |

**Senior-level answer:** "I default to SQL/relational unless there's a **specific, articulable reason** to reach for NoSQL — most business domains genuinely have relationships and invariants (an Order genuinely relates to a Customer and OrderLines, MOD4_002 Q5's Aggregate boundaries) that a relational model expresses naturally and enforces for free via constraints. I reach for NoSQL when the access pattern is genuinely different from what SQL is good at — e.g., a product catalog where every category has wildly different attributes (schema flexibility genuinely matters), or a write-heavy event/session store needing horizontal write scale beyond what a single Postgres primary can sustain. 'NoSQL is more scalable' isn't a complete argument on its own — Postgres with proper indexing, read replicas, and partitioning scales further than most teams ever actually need."

**Follow-up: "Can you use both in the same system?"** → Yes, and it's common — **polyglot persistence**: relational for the transactional core (orders, payments), a document store for a flexible catalog, Redis for caching/sessions (Q7), each service (MOD4_003 Q3's database-per-service) choosing the store that actually fits its own access pattern.

---

## SECTION 2: REST vs gRPC

### Q2. REST vs gRPC — state the trade-off in one breath, the way you would in an interview.
**Answer:** *(Full comparison in MOD4_004 Q2 — this is the compressed, interview-ready version.)* "REST/JSON is human-debuggable, universally browser-compatible, and the right default for anything public or partner-facing. gRPC/Protobuf is faster and more compact via binary serialization and HTTP/2 multiplexing, with a strictly enforced contract via `.proto` files, and native streaming support — the right choice for internal, high-throughput service-to-service calls where you control both ends and don't need browser compatibility. I've mixed both in the same system: REST at the public edge, gRPC internally behind the gateway."

---

## SECTION 3: SYNCHRONOUS vs ASYNCHRONOUS

### Q3. Sync vs async communication — what's the one question that actually decides this for a given call?
**Answer:** *(Full comparison in MOD4_004 Q1.)* "The deciding question: **does the caller need the result to make its next decision right now?** If yes — checkout needing to know 'was the card charged' — that's a genuine synchronous dependency. If no — 'send a confirmation email' — it should be an async, published event. The most common real mistake is defaulting to synchronous REST calls for things that were never actually time-critical, creating unnecessary temporal coupling (MOD4_004 Q1) and chatty call chains that increase cascading-failure risk (MOD4_004 Q7-Q8)."

---

## SECTION 4: MONOLITH vs MICROSERVICES

### Q4. Monolith vs Microservices — give the decision framework, not the definitions.
**Answer:** *(Full comparison in MOD4_001 Q1, Q6.)* "This isn't 'which is better' — it's a trade of development-time simplicity for operational complexity, made in exchange for independent scalability and deployability. I look for **concrete signals** before recommending microservices: genuinely different scaling needs across parts of the system, multiple teams blocking each other in one codebase's release train, or a real need for independent release cadence. Absent those signals, a well-modularized monolith (MOD4_001 Q2) gets most of the organizational benefit without the distributed-systems tax — network partial failures, eventual consistency, and the operational overhead of running dozens of independently deployed services."

---

## SECTION 5: STRONG vs EVENTUAL CONSISTENCY

### Q5. Strong vs eventual consistency — explain the trade-off, and give a real example of choosing eventual consistency deliberately.
**Answer:** **Strong consistency** guarantees that once a write completes, **every subsequent read, from any node, sees that write immediately** — simple to reason about, but requires coordination that costs latency and availability during partitions (this is the C in CAP theorem, MOD5_001). **Eventual consistency** allows reads to temporarily return stale data after a write, with the guarantee that **all replicas will converge** to the same value given enough time and no new writes — trading immediate correctness for availability and lower latency.

```java
// Strong consistency — synchronous replication/read from primary, always fresh but slower/less available
Order order = orderRepository.findById(id);  // reads from primary, guaranteed latest

// Eventual consistency — read from a replica/cache that may lag behind the write
Order order = orderReadReplicaRepository.findById(id);  // might be milliseconds-to-seconds stale
```

**Real example of deliberately choosing eventual consistency:** "A product's view/like count doesn't need to be strongly consistent — if a user sees a count that's a few seconds stale, there's zero real business impact, but requiring strong consistency on every single view-count increment across a high-traffic system would be a genuinely unnecessary coordination bottleneck. Compare that to an account balance during a funds transfer — that genuinely needs strong consistency (or at minimum, a carefully-designed Saga with compensating transactions, MOD4_003 Q6, if it's split across services) because stale reads there translate directly into real financial correctness bugs, not just a cosmetic UI lag."

**Senior-level answer:** "I treat this as a **per-use-case decision**, not a system-wide default — even within one system, I'll have strongly-consistent reads for financial state and eventually-consistent, cache-backed reads for high-traffic, low-stakes data like view counts or recommendation scores. The mistake is applying one consistency model uniformly across an entire system regardless of what each specific piece of data actually requires."

---

## SECTION 6: KAFKA vs RABBITMQ

### Q6. Kafka vs RabbitMQ — the one-breath version for an interview.
**Answer:** *(Full comparison in MOD5_007 Q1-Q3, MOD4_004 Q4.)* "RabbitMQ's architecture centers on **flexible routing** (exchanges, bindings, routing keys) with messages typically removed once consumed — optimized for smart routing to the right consumer, task-queue semantics. Kafka's architecture centers on a **durable, partitioned, replayable log** — optimized for high-throughput event streaming where multiple independent consumers need their own pace and new consumers can replay history. I pick Kafka when I need replay or many independent consumer groups reading the same stream (feeding both real-time fraud detection and nightly batch analytics off the same topic); I pick RabbitMQ for simpler task-queue needs or when I need its richer routing logic and don't need message history."

---

## SECTION 7: CACHE vs DATABASE

### Q7. Cache vs Database — walk through the actual trade-offs, and the failure modes a cache introduces that a database read doesn't have.
**Answer:** A database is the **system of record** — durable, consistent, the source of truth. A cache is a **fast, often in-memory, disposable copy** of a subset of that data, trading some consistency/durability for dramatically lower read latency and reduced load on the database.

| Aspect | Database (direct read) | Cache (Redis, etc.) |
|---|---|---|
| Latency | Higher (disk I/O, query planning) | Very low (in-memory) |
| Consistency | Always current (for strongly-consistent reads) | Can be **stale** — a genuine, real trade-off (Q5) |
| Durability | Persistent, survives restarts | Often ephemeral (unless explicitly persisted) — losing the cache should never lose data |
| Failure mode if unavailable | The request usually just fails/errors | Should **degrade gracefully** — fall through to the database, not fail the whole request |

**The failure mode that's a genuinely common real mistake:** "A cache should **never** be the only place a piece of data lives — if Redis is unavailable, the correct behavior is falling through to the database (slower, but correct), not returning an error or, worse, silently serving nothing. I've seen an outage where a cache cluster went down and the application had no fallback path coded at all — it had implicitly become a hard dependency rather than a performance optimization, which defeats the entire purpose of caching being an optional acceleration layer."

**Cache invalidation — the other classic problem, worth naming proactively:** "There's the old joke that cache invalidation is one of the two hard problems in computer science, and it's not really a joke — a **write-through** cache (write to cache and DB together, always consistent but no faster writes) is simpler to reason about than a **write-behind**/**cache-aside** pattern (write to DB, invalidate or update cache separately, async), which is faster but has a real window where cache and DB can briefly disagree. I pick the pattern based on how tolerant that specific data is of brief staleness — exactly the same reasoning as Q5's strong-vs-eventual consistency decision, just applied at the cache layer specifically."

---

## SECTION 8: API GATEWAY vs SERVICE MESH

### Q8. API Gateway vs Service Mesh — the one-breath version.
**Answer:** *(Full comparison in MOD7_007 Q7, MOD4_005 Q1.)* "API Gateway handles **north-south** traffic — external clients calling into the system at the edge: authentication of external/untrusted clients, coarse rate limiting, protocol translation. Service Mesh handles **east-west** traffic — internal service-to-service: mTLS, fine-grained internal traffic control, uniform observability, all via a sidecar transparent to application code. They're complementary, not competing — I've built systems where a request authenticates at the gateway, then flows through the mesh for every internal hop after that. I wouldn't introduce a full mesh for a handful of services, though — the operational overhead only pays off once service count and cross-cutting-concern inconsistency across teams become the bigger cost."

---

## SECTION 9: CENTRALIZED vs DISTRIBUTED TRANSACTIONS

### Q9. Centralized (single-database ACID) vs Distributed transactions (Saga) — walk through the actual trade-off, and when 2PC is (rarely) still the right call.
**Answer:** A **centralized transaction** — everything inside one database, one ACID transaction — gives you atomicity and consistency essentially for free, guaranteed by the database engine. The moment a business operation spans **multiple services with separate databases** (MOD4_003 Q3's database-per-service), you lose that free guarantee, and the choices become: **Two-Phase Commit (2PC)** — a distributed transaction protocol requiring all participants to lock resources and vote commit/abort together — or a **Saga** (MOD4_003 Q6-Q7) — a sequence of local transactions with explicit compensating actions.

| Aspect | Centralized (single DB, ACID) | 2PC (distributed transaction) | Saga (distributed, choreographed/orchestrated) |
|---|---|---|---|
| Atomicity guarantee | True, database-enforced | True, but at real cost (below) | None natively — **eventual consistency**, achieved via compensation |
| Availability | High — no cross-service coordination needed | Low — blocks/holds locks across all participants until every vote is in | High — each local transaction commits independently and quickly |
| Failure handling | Automatic rollback | Coordinator failure can leave participants blocked indefinitely (the "in-doubt" problem) | Explicit compensating transactions must be written for every step |
| Complexity | Lowest | Protocol complexity, and most modern brokers/stores don't support XA well at scale | Business logic complexity — every step needs a compensating action designed and tested |

**Why 2PC is essentially never the right call in modern microservices, stated directly:** "2PC requires every participant to **hold locks and block** until the coordinator collects all votes — that's fundamentally at odds with microservices' availability and independent-scaling goals, and directly conflicts with the CAP trade-offs (MOD5_001, MOD5_003) you've already implicitly accepted by going distributed in the first place. It also has a nasty failure mode: if the coordinator crashes after participants have voted but before the final commit/abort decision is broadcast, those participants are stuck **holding locks indefinitely**, unable to safely proceed either way — a genuine, well-known operational hazard, not a theoretical one."

**Senior-level answer on the actual decision:** "My default for any operation spanning services is Saga with explicit compensating transactions, accepting eventual consistency as a deliberate, documented business trade-off (MOD4_003 Q6's DRAFT-order window needing product sign-off, not a silent engineering decision). I'd only reach for 2PC in a genuinely narrow scenario — a small number of tightly-coupled, co-located systems where strict atomicity is non-negotiable and the availability/scalability cost is acceptable — and even then, I'd push hard to first ask whether the transaction boundary itself was drawn correctly (MOD4_002 Q5's 'one transaction, one aggregate' rule) before reaching for a heavier coordination protocol to paper over what might actually be a service-boundary design smell."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "If you had to pick just ONE of these trade-offs to get right in a system design interview, which would you emphasize?" → Consistency model (Q5) and transaction boundaries (Q9) — because getting these wrong isn't a performance problem, it's a **correctness** problem that surfaces as subtle, hard-to-reproduce production bugs (double charges, lost updates) rather than a clean failure.
- "Does choosing NoSQL mean giving up transactions entirely?" → No — many NoSQL stores support transactions within a single document/partition (MongoDB multi-document transactions exist too, with real performance cost); what you generally give up is **cross-partition, cross-shard** ACID transactions at the scale NoSQL is chosen for, which is exactly why Saga-style patterns (Q9) matter even more in NoSQL-heavy architectures.
- "Is 'it depends' actually an acceptable interview answer for these trade-off questions?" → Yes — **if immediately followed by the specific factors you'd weigh and a real example**. "It depends" alone signals indecision; "it depends on X, Y, Z, and here's a case where I chose one way because of X" signals genuine judgment — the factors are the actual answer.
- "How do these trade-offs interact with each other — pick two and show how one decision constrains another?" → Choosing Microservices (Q4) forces you off centralized transactions (Q9) toward Sagas, which forces you toward eventual consistency (Q5) for cross-service state, which often means leaning on async messaging (Q3) and event-carried state transfer (MOD4_004 Q3) to propagate that eventual state — a good answer traces this chain rather than treating each trade-off as independent.
- "What's the most common trade-off mistake you've seen made by a well-intentioned team?" → Applying a single consistency/communication model uniformly across an entire system "for simplicity" rather than deciding **per use case** — e.g., making every internal call synchronous "to keep it simple," which then creates chatty call chains and cascading failure risk (MOD4_004 Q7-Q8) for the majority of calls that never actually needed to be synchronous in the first place.

---

*Study tip: This module is meant to be your **rapid-recall layer** heading into a system design round — for each topic here, be ready to state the trade-off in under 30 seconds AND have one concrete example ready where you'd choose the less-popular option (NoSQL over SQL, RabbitMQ over Kafka, 2PC in a narrow case) — interviewers specifically probe for whether a candidate defaults reflexively to the trendier choice or can articulate a genuine, context-dependent decision.*
