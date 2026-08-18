# Scalability & Availability — Interview Prep
---

## SECTION 1: HORIZONTAL vs VERTICAL SCALING

### Q1. What's the practical difference between horizontal and vertical scaling, and why does horizontal scaling dominate modern cloud-native architecture despite being more complex?
**Answer:**
| | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| **How** | Add more CPU/RAM/disk to an existing single machine | Add more machines/instances running the same application |
| **Ceiling** | Hard physical limit — a single machine can only get so big | Effectively unbounded — keep adding instances |
| **Downtime to scale** | Often requires a restart/reboot to apply | New instances join without interrupting existing ones |
| **Single point of failure** | Yes — one machine, one failure takes everything down | No — losing one instance still leaves others serving traffic |
| **Complexity** | Simple — no architectural changes needed | Requires the app to support running multiple instances (statelessness, Q3) |

```
Vertical:    [Small Server] --> [Bigger Server] --> [Even Bigger Server]  (one box, growing)
Horizontal:  [Server] --> [Server] [Server] --> [Server] [Server] [Server] [Server]  (more boxes)
```

**Senior-level answer:**
> "Vertical scaling is the easy first move and I don't dismiss it — for a lot of workloads, just moving to a bigger instance type solves the problem for a long time with zero code changes. But it has a hard ceiling and a single point of failure baked in: the biggest cloud instance still eventually maxes out, and until it does, you have exactly one machine that can go down and take the whole service with it. Horizontal scaling trades that simplicity for near-unlimited capacity and, just as importantly, **redundancy as a side effect** — multiple instances means no single instance failure is fatal, which is why it's the default assumption in cloud-native design even before raw capacity becomes the driving concern."

**Trap:** Don't present this as an either/or choice — real systems do both. You still pick a reasonably-sized instance type (some vertical scaling) *and* run multiple instances of it (horizontal scaling); they're complementary levers, not competing strategies.

---

## SECTION 2: STATELESS vs STATEFUL SERVICES

### Q2. Why does horizontal scaling require services to be stateless, and what does "stateless" actually mean in practice?
**Answer:** A **stateless** service holds **no client-specific data in memory between requests** — every request contains (or can retrieve) everything needed to process it, so **any instance can handle any request**, with no dependency on which instance handled a previous request from that same client. A **stateful** service keeps something in memory (a session, a WebSocket connection, an in-progress upload) that only *that specific instance* knows about — which breaks horizontal scaling, because a load balancer routing a follow-up request to a *different* instance loses access to that state entirely.

```java
// STATEFUL — breaks under horizontal scaling
@Service
class CartService {
    private Map<String, Cart> inMemoryCarts = new HashMap<>();  // lives only on this one instance
}

// STATELESS — safe to scale horizontally
@Service
class CartService {
    private final RedisCartRepository cartRepository; // external, shared store — any instance can serve any request
}
```

**Senior-level answer:**
> "The practical test I use: if you kill the instance handling a user's request mid-session and route their next request to a totally different, freshly-started instance, does anything break? If the answer is yes, there's hidden state somewhere that needs to move to a shared external store — Redis, a database, a distributed cache. This matters beyond just 'can we add more instances' — it's also what makes rolling deployments and auto-scaling safe, since instances need to be freely interchangeable and disposable for either to work without disrupting active users."

---

### Q3. Sessions are a classic example of state that complicates horizontal scaling. What are the actual solutions, and what are their trade-offs?
**Answer:**
| Solution | How it works | Trade-off |
|---|---|---|
| **Sticky sessions** | Load balancer routes a given client to the *same* instance every time (via a cookie/IP hash) | Simple, but breaks if that instance dies (session lost), and unevenly distributes load if some clients are much more active than others |
| **Centralized session store** | Session data lives in Redis/a database, not in any instance's memory; any instance can read it | Adds a network hop per request needing session data; requires the shared store itself to be highly available |
| **Client-side/token-based state (JWT)** | The session data (or a reference to it) is encoded in a signed token the client holds and sends with every request | No server-side lookup needed at all; but token size grows with data, and revocation becomes harder (can't simply "delete" a session server-side) |

**Senior-level answer:**
> "I default to a centralized session store (Redis) for most systems — it keeps the actual server instances genuinely stateless and interchangeable, which is worth the extra network hop. Sticky sessions are a tempting shortcut because they require zero application changes, but I'm wary of them specifically because they quietly reintroduce a single point of failure per user — if that one 'sticky' instance goes down, that user's session is just gone, which defeats a big part of why you scaled horizontally in the first place. JWTs solve it differently by not needing server-side state lookup at all, which is great for stateless auth, but revocation (logging a user out immediately, force-expiring a compromised token) needs its own solution — usually a short-lived token plus a blocklist — since you can't just delete something the client is holding."

---

## SECTION 3: LOAD BALANCING

### Q4. What are the common load balancing algorithms, and how do you choose between them?
**Answer:**
| Algorithm | How it works | Best for |
|---|---|---|
| **Round Robin** | Requests distributed sequentially across instances in a fixed rotation | Simple, works well when instances are equally sized and requests are roughly uniform in cost |
| **Least Connections** | Routes to the instance currently handling the fewest active connections | Workloads with variable request duration — prevents a slow instance from still getting an equal share of new traffic |
| **Weighted Round Robin/Least Connections** | Same as above, but instances get proportionally more/less traffic based on assigned capacity weight | Mixed instance sizes (e.g., during a gradual migration to bigger instance types) |
| **IP Hash / Consistent Hashing** | Routes based on a hash of the client IP (or another key), so the same client consistently reaches the same instance | Needed for sticky sessions (Q3), or for cache-locality optimization |

**Senior-level answer:**
> "Round robin is the reasonable default and what most managed load balancers use out of the box, but I switch to least-connections the moment request processing times vary meaningfully — round robin doesn't know or care that one instance is currently stuck processing a slow request; it'll happily send it more work anyway. Least-connections actively routes away from an instance that's already backed up, which is a real, practical difference under uneven load, not just a theoretical one."

---

### Q5. What's the difference between a Layer 4 and a Layer 7 load balancer, and when does the distinction actually matter?
**Answer:**
| | Layer 4 (Transport) | Layer 7 (Application) |
|---|---|---|
| **Operates on** | IP address + TCP/UDP port only — no visibility into the actual request content | Full HTTP request — path, headers, cookies, method, body |
| **Routing decisions** | Basic — which backend gets this connection | Content-aware — route `/api/orders/*` to the Orders service, `/api/users/*` to the Users service, based on the actual URL |
| **Performance** | Faster — less processing per request | Slightly more overhead — has to parse the HTTP layer |
| **SSL termination** | Typically passes through encrypted | Commonly terminates TLS at the load balancer |

**Senior-level answer:**
> "The distinction matters the moment you need routing decisions based on anything in the request itself, not just where it's going at a network level — path-based routing to different microservices, header-based canary routing for a gradual rollout, or cookie-based session affinity all require Layer 7 visibility. I use Layer 4 when I just need fast, dumb TCP-level distribution to a homogeneous pool of backends, and reach for Layer 7 (an API Gateway or an application load balancer) the moment routing needs to be content-aware — which, in a microservices architecture, is almost always."

---

## SECTION 4: REPLICATION

### Q6. What's the difference between synchronous and asynchronous replication, and what's the actual trade-off you're making between them?
**Answer:**
| | Synchronous Replication | Asynchronous Replication |
|---|---|---|
| **Write acknowledged when** | The primary AND at least one replica have confirmed the write | The primary alone has confirmed the write; replicas catch up afterward |
| **Data loss risk on primary failure** | None — a replica is guaranteed to have every acknowledged write | Possible — writes acknowledged to the client but not yet replicated are lost if the primary fails before replicating |
| **Write latency** | Higher — must wait for network round-trip to replica(s) | Lower — doesn't wait for replicas at all |
| **Availability impact** | A slow/unreachable replica can block writes entirely (or force a fallback policy) | Primary keeps accepting writes regardless of replica health |

**Senior-level answer:**
> "This is a direct, explicit trade between **durability guarantee** and **write latency/availability**, and the right choice depends entirely on what a given piece of data is worth. For something like a financial transaction, I lean toward synchronous replication (or at minimum, a quorum-based semi-synchronous approach) because losing an acknowledged write is unacceptable — the client was told it succeeded. For something like an activity log or a metrics event, asynchronous replication's small window of potential data loss on a rare primary failure is a completely acceptable trade for meaningfully lower write latency on every single write, which matters at volume."

---

### Q7. What's replication lag, and what real problems does it cause in a typical read-replica setup?
**Answer:** Replication lag is the **delay between a write landing on the primary and that write becoming visible on a replica** — under async replication (Q6), this is never zero, and under load it can grow from milliseconds to seconds or more. The classic resulting bug: a user **writes data, then immediately reads it back**, but the read is routed to a replica that hasn't caught up yet — the user sees their own change appear to have "not worked."

```java
// Classic read-your-writes bug
orderService.createOrder(order);              // writes to PRIMARY
Order fetched = orderService.getOrder(order.getId()); // reads from a REPLICA — might not have propagated yet!
// fetched could be null or stale, even though the write just succeeded
```

**Mitigation approaches:**
- **Read-your-writes consistency** — route reads for a user back to the primary for a short window after that user's own write, or track a "read your own write" token/timestamp.
- **Route critical, freshness-sensitive reads to the primary** always, and only route tolerant, high-volume reads (browsing, search, analytics) to replicas.
- **Monitor lag actively** and remove a replica from the read pool if its lag exceeds an acceptable threshold.

**Senior-level answer:**
> "Replication lag is one of those things that's invisible in a demo — low traffic means near-instant replication — and becomes a real, user-facing bug only under production load, which is exactly why it needs to be a deliberate design decision, not discovered in an incident. My default pattern: writes and any read that must reflect that exact write immediately go to the primary; everything else — dashboards, search, general browsing — reads from replicas, since a few hundred milliseconds of staleness there is genuinely unnoticeable and buys real read-scaling headroom."

---

## SECTION 5: PARTITIONING & SHARDING

### Q8. What is sharding, and what makes choosing a shard key the most consequential decision in the whole design?
**Answer:** Sharding splits a dataset **horizontally across multiple independent database instances**, each holding a subset of the total data (as opposed to replication, where each replica holds the *full* dataset) — done specifically to scale write throughput and total storage capacity beyond what a single database instance can handle. The **shard key** determines which shard a given row lives on, and a poor choice causes lasting, hard-to-fix problems:

- **Hot shard** — a shard key with uneven distribution (e.g., sharding by `country` when 80% of users are in one country) concentrates load on one shard while others sit idle, defeating the entire point of sharding.
- **Cross-shard queries become expensive** — a query that needs data spanning multiple shards (e.g., "all orders across all customers in the last hour" when sharded by `customer_id`) requires fanning out to every shard and merging results in the application, which is slow and complex compared to a single-database query.
- **Re-sharding is extremely costly** — changing the shard key after the fact means physically migrating large volumes of data between shards, usually with real downtime or complex online-migration tooling.

```
Sharded by customer_id (hash-based):
  hash(customer_id) % 4 == 0 -> Shard A
  hash(customer_id) % 4 == 1 -> Shard B
  hash(customer_id) % 4 == 2 -> Shard C
  hash(customer_id) % 4 == 3 -> Shard D
```

**Senior-level answer:**
> "I treat the shard key decision as close to irreversible in practice, even though it's technically changeable, because re-sharding a large production dataset is one of the most operationally dangerous migrations a team can undertake — real downtime risk, real data-consistency risk during the migration window. I always push to choose a shard key based on **actual query and access patterns**, not just an obvious-looking column: the right question isn't 'what's a natural unique identifier' but 'what's the field most of our high-volume queries filter by, and does distributing on it actually spread load evenly across our real traffic, not just in theory.'"

---

### Q9. What's the difference between sharding and partitioning, since the terms are often used interchangeably?
**Answer:** In practice, the terms overlap significantly and usage varies by community, but the distinction most commonly drawn:
- **Partitioning** — splitting a large table into smaller pieces, which may or may not live on **separate physical database instances**; a single database can partition a huge table internally (e.g., PostgreSQL table partitioning by date range) purely for manageability/query performance, with no distributed-systems implications at all.
- **Sharding** — specifically refers to partitioning **across separate database instances/servers**, explicitly for horizontal scaling of write throughput and storage beyond one machine — sharding always implies distribution; partitioning doesn't necessarily.

```sql
-- Partitioning WITHOUT sharding — still one database instance, split internally for manageability
CREATE TABLE orders (
    id BIGINT, order_date DATE, ...
) PARTITION BY RANGE (order_date);
-- orders_2025, orders_2026 partitions -- same DB server, faster queries/maintenance, not distributed
```

**Senior-level answer:**
> "I don't get too rigid about the terminology distinction in an interview, since usage genuinely varies, but I do make sure to demonstrate I understand the underlying difference: you can partition a table for query performance and maintenance convenience without any distributed-systems complexity at all — it's still one database. Sharding specifically means you've taken on the operational complexity of a distributed dataset — cross-shard query limitations, no single-instance ACID transactions across shards, the shard-key design problem (Q8) — and that's a meaningfully bigger architectural commitment than table partitioning within one instance."

---

## SECTION 6: CACHING

### Q10. How does caching contribute to both scalability and availability, beyond just "making things faster"?
**Answer:**
- **Scalability**: caching absorbs read load that would otherwise hit the database directly — a well-cached system can serve orders of magnitude more read traffic than the underlying database could handle alone, because most requests never reach it.
- **Availability**: a cache can act as a **fallback during a downstream outage** — if the database or a dependency is temporarily unavailable, serving slightly stale cached data (a deliberate degraded mode) can keep the system partially functional instead of failing completely, directly connecting to the graceful degradation pattern (see Resilience & Fault Tolerance module).

**Senior-level answer:**
> "The availability angle is the one that's easy to overlook when caching is framed purely as a performance optimization. I've specifically designed cache layers with a 'serve stale on backend failure' fallback — if the primary data source times out or errors, return the last-known cached value with a flag indicating it might be stale, rather than propagating the failure to the end user. For a lot of read paths, slightly stale data is a far better user experience than an error page, and that's a deliberate availability decision, not just a caching performance detail." (Full caching pattern depth — cache-aside, stampede/penetration/avalanche, distributed locking — is covered in the Distributed Caching module.)

---

## SECTION 7: HIGH AVAILABILITY

### Q11. What does "High Availability" (HA) actually mean quantitatively, and how do the common "nines" translate to real downtime?
**Answer:** High Availability is usually expressed as a **percentage of uptime over a period**, commonly called "the nines" — each additional nine represents a roughly 10x reduction in acceptable downtime.

| Availability | Downtime per year | Downtime per month |
|---|---|---|
| 99% ("two nines") | ~3.65 days | ~7.3 hours |
| 99.9% ("three nines") | ~8.76 hours | ~43.2 minutes |
| 99.99% ("four nines") | ~52.6 minutes | ~4.3 minutes |
| 99.999% ("five nines") | ~5.26 minutes | ~26 seconds |

**Senior-level answer:**
> "The number itself is meaningless without context — 99.99% sounds impressively close to 99.9%, but it's the difference between roughly 4 minutes of downtime a month versus 43, and each additional nine typically requires a disproportionately larger investment: redundancy across availability zones, automated failover, extensive chaos/failure testing. I always push teams to define the *actual* required availability based on real business impact — an internal admin tool genuinely does not need five-nines engineering investment, and treating every system as if it does is a waste of engineering effort that could go toward the systems that actually need it."

---

### Q12. What are the core architectural techniques for achieving High Availability, beyond "just add more servers"?
**Answer:**
- **Redundancy with no single point of failure** — every critical component (app servers, database, load balancer itself) has at least one healthy standby/replica ready to take over.
- **Multi-AZ / multi-region deployment** — spreading instances across physically separate data centers/availability zones so a single facility's failure (power outage, network issue) doesn't take down the whole service.
- **Automated health checks and failover** — detecting an unhealthy instance/replica and automatically routing around it or promoting a standby, without requiring a human to notice and react (see readiness/liveness probes — Resilience & Fault Tolerance module).
- **No manual, human-in-the-loop steps in the failover path** — anything requiring a human to notice and act adds minutes-to-hours of downtime versus an automated failover measured in seconds.

**Senior-level answer:**
> "The detail that separates a genuinely highly-available design from one that just looks redundant on a diagram is **automated failover with no human in the loop**. I've reviewed 'HA' architectures that had a standby database ready to go, but promoting it required someone to notice the primary was down, log in, and manually run a promotion script — that's not high availability in any meaningful sense if the actual downtime is bounded by how fast an on-call engineer wakes up and responds. True HA means the system detects and routes around failure automatically, on the order of seconds, with human involvement only for the post-incident cleanup, not the immediate recovery."

---

## SECTION 8: DISASTER RECOVERY

### Q13. What's the difference between High Availability and Disaster Recovery — aren't they the same thing?
**Answer:** They address different failure scales and are **not** the same thing, though they're complementary:
| | High Availability | Disaster Recovery |
|---|---|---|
| **Failure scope** | Individual component failure (one server, one AZ) | Catastrophic, large-scale failure (an entire region outage, major data corruption, a successful ransomware attack) |
| **Goal** | Continue operating with minimal/no interruption | Recover operations after a major event, potentially with some data loss and downtime |
| **Typical mechanism** | Redundancy + automated failover within a region | Cross-region backups/replication, a documented recovery plan, periodically tested restoration procedures |
| **Key metrics** | Uptime percentage | RTO and RPO (Q14) |

**Senior-level answer:**
> "HA handles 'a server died' automatically and near-instantly; DR handles 'the entire region is gone' or 'our data got corrupted and we need to restore from a known-good backup,' which is a fundamentally different, much larger-scale scenario that HA mechanisms alone don't cover — multi-AZ redundancy within a region doesn't help you if the whole region has an outage. I always push for DR planning to be treated as a real, tested capability, not just 'we take backups' — a backup nobody has ever practiced restoring from is not a disaster recovery plan, it's a hope."

---

### Q14. What are RTO and RPO, and how do they concretely drive DR architecture decisions?
**Answer:**
- **RTO (Recovery Time Objective)** — how long the system is allowed to be **down** before it must be restored — "we must be back online within 4 hours of a disaster."
- **RPO (Recovery Point Objective)** — how much **data loss** is acceptable, measured in time — "we can tolerate losing up to 15 minutes of the most recent data."

**How these numbers drive real architecture:**
| Requirement | Implication |
|---|---|
| **RPO near zero** (can't lose any data) | Requires synchronous or near-real-time cross-region replication (Q6) — expensive, adds write latency |
| **RPO = hours** | Periodic backups (e.g., hourly snapshots) are sufficient — much cheaper |
| **RTO near zero** (must fail over instantly) | Requires a warm/hot standby in another region, already running and ready to take traffic |
| **RTO = hours** | A cold standby — infrastructure provisioned from backups/IaC only when a disaster actually occurs — is acceptable and far cheaper |

**Senior-level answer:**
> "RTO and RPO are the two numbers that should drive every DR spending decision, and they need to come from an actual business conversation about acceptable risk and cost — not be picked by engineering in isolation. A near-zero RPO and RTO requirement effectively mandates an expensive, always-on multi-region active-active or active-passive setup; a business that can tolerate 4 hours of downtime and 30 minutes of data loss can get away with a dramatically cheaper cold-standby-plus-backups approach. I always make sure these targets are explicitly documented and agreed with the business, because 'we should be highly available' without concrete RTO/RPO numbers gives engineering no actual target to design against."

---

## SECTION 9: FAULT TOLERANCE

### Q15. How does Fault Tolerance differ from High Availability, and what's a concrete example that shows the distinction?
**Answer:** High Availability is about **minimizing downtime** when something fails; **Fault Tolerance** is about the system **continuing to function correctly, often without the end user even noticing**, *while* a failure is actively occurring — a stronger, more specific property than HA. A highly-available system might have a brief blip (seconds) during failover; a fully fault-tolerant system is architected so that specific failure produces **zero visible interruption** at all.

**Concrete example:** A payment processing system with two redundant payment gateway integrations. If the primary gateway starts timing out:
- **HA-only approach**: an alert fires, an on-call engineer manually fails over to the secondary gateway — some number of payments fail or are delayed during that window.
- **Fault-tolerant approach**: a circuit breaker automatically detects the primary gateway's failure and routes new payment requests to the secondary gateway **within milliseconds, with no human involvement and no failed customer-facing requests** — the failure happened, but it was fully absorbed by the system's design.

**Senior-level answer:**
> "I think of fault tolerance as the more ambitious, harder-to-achieve subset of high availability — HA can tolerate some brief interruption during recovery; true fault tolerance means the specific failure mode was anticipated and designed around well enough that it produces no visible interruption at all. In practice, most systems are fault-tolerant against *some* failure modes (a single instance dying, handled by redundancy + load balancing) but only highly-available (with brief interruption) against others (a full database failover). Being explicit about which failures are fully tolerated versus which just trigger a fast, automated recovery is a more precise and more useful conversation than treating 'resilience' as one undifferentiated property." (Circuit breakers, bulkheads, retries, and the specific mechanisms behind this are covered in depth in the Resilience & Fault Tolerance module.)

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can a system be highly available but not scalable, or vice versa?" → Yes — a single, very robust, well-monitored server with automatic restart-on-failure could technically be "available" but isn't horizontally scalable; conversely, a system that scales to handle huge load but has a single point of failure (an unreplicated primary database) is scalable but not highly available. They're independent properties that both need deliberate design.
- "What's the CAP theorem's relevance to this module?" → CAP theorem (covered in depth in the Distributed Systems module) directly explains *why* the synchronous-vs-asynchronous replication trade-off (Q6) exists — you can't have perfect consistency and perfect availability during a network partition, so replication strategy is fundamentally a CAP trade-off made concrete.
- "How does auto-scaling relate to horizontal scaling?" → Auto-scaling is horizontal scaling's automated form — instances are added/removed dynamically based on real-time load metrics (CPU, request count, queue depth), rather than a fixed number of instances always running; it requires the same statelessness prerequisite (Q2) to work safely.
- "Should you always design for multi-region from day one?" → No — multi-region adds significant complexity and cost (cross-region replication latency, data residency/compliance considerations, more complex deployment); it should be driven by actual RTO/RPO/business requirements (Q14), not adopted reflexively for a system that doesn't yet need that level of resilience.
- "What's a split-brain scenario, and how does it relate to replication/failover?" → A failure mode where a network partition causes two nodes to each believe they're the primary/leader simultaneously, both accepting writes independently — this is why leader election typically requires a quorum-based consensus mechanism (Raft/Paxos, covered in the Distributed Transactions & Consensus module), not a naive "promote on connection loss" policy.
- "Is more redundancy always better for availability?" → No — redundancy has real cost (infrastructure spend, operational complexity, more moving parts that can themselves fail or drift out of sync) and diminishing returns past a certain point; the right amount of redundancy is driven by the actual availability target (Q11), not maximized unconditionally.

---

*Study tip: Being able to explain why statelessness is the prerequisite that makes horizontal scaling actually work (Q2–Q3), clearly distinguishing High Availability from Disaster Recovery with concrete RTO/RPO numbers rather than treating them as synonyms (Q13–Q14), and demonstrating that shard key selection is a near-irreversible decision worth real design time rather than an afterthought (Q8) are the three areas where interviewers most reliably separate candidates who've operated real production systems at scale from candidates repeating textbook definitions.*
