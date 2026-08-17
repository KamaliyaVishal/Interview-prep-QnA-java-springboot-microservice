# Distributed Data Management — Interview Prep
---

## SECTION 1: DATABASE PER SERVICE vs SHARED DATABASE

### Q1. Why does the microservices pattern insist on "database per service"? What breaks if you share a database?
**Answer:** **Database-per-service** means each microservice owns its own database (schema, or even a separate DB instance), and **no other service is allowed to access it directly** — all access goes through that service's API.

**What breaks with a shared database:**
- **Tight coupling at the schema level** — any service can change a column and silently break five other services reading that table. This defeats the entire point of independently deployable services.
- **No independent scaling/tech choice** — you can't give the order service Postgres and the search service Elasticsearch if everyone reads from one shared schema.
- **Unclear ownership** — nobody "owns" a shared table, so nobody can safely evolve it.
- **Hidden transactional coupling** — teams start wrapping cross-service writes in a single local ACID transaction against the shared DB, which reintroduces a distributed monolith with none of the deployment independence.

```
[Order Service] --> [Order DB]
[Inventory Service] --> [Inventory DB]
[Payment Service] --> [Payment DB]
-- No service reaches into another service's DB. Only via API/events. --
```

**Senior-level answer:**
> "Shared databases are the single most common reason 'microservices' projects fail to actually decouple anything — you've split the code but not the data, so a schema migration in one team's table still requires coordinating a release with three other teams. Database-per-service is what actually buys you independent deployability; everything else in this module — sagas, CQRS, outbox — exists specifically to deal with the consequences of that decision."

**Trap:** Don't say "database per service" means every service must have a *physically separate* database server — it just means a separate schema/logical database with **no direct cross-service access**. Physical isolation is a stronger, optional variant usually reserved for regulatory/compliance boundaries or noisy-neighbor performance isolation.

---

### Q2. If services can't share a database, how do you enforce data consistency across services (e.g., an Order and a Payment that must both succeed or both fail)?
**Answer:** You give up a single ACID transaction across services and instead use the **Saga pattern** — a sequence of local transactions, each committed independently, where each step publishes an event/command that triggers the next step, and **compensating transactions** undo prior steps on failure.

| Approach | Coordination | Failure handling |
|---|---|---|
| **Choreography** | Each service reacts to events published by others; no central coordinator | Each service must know its own compensating action |
| **Orchestration** | A central saga orchestrator explicitly calls each service and tracks state | Orchestrator issues compensations on failure — easier to reason about for complex flows |

```java
// Choreography-style saga step (Order Service)
@KafkaListener(topics = "payment-events")
public void onPaymentResult(PaymentEvent event) {
    if (event.getStatus() == PaymentStatus.FAILED) {
        orderService.cancelOrder(event.getOrderId()); // compensating action
    } else {
        orderService.confirmOrder(event.getOrderId());
    }
}
```

**Senior-level answer:**
> "This is the trade-off that trips up people coming from a monolith background: you're exchanging strong, immediate consistency for eventual consistency plus explicit compensation logic. That's a legitimate engineering trade-off, but it has to be a conscious one — I always push teams to explicitly define what 'undo' means for every step (e.g., a completed payment can't be un-charged the same way an unconfirmed order can be cancelled; you may need a refund flow instead), because compensating transactions are rarely a perfect mirror of the forward action."

---

## SECTION 2: DATA DUPLICATION & SYNCHRONIZATION

### Q3. If every service owns its own data, how do services access data that "belongs" to another service without constant synchronous calls?
**Answer:** By **deliberately duplicating a read-optimized copy** of the data they need locally, kept in sync via **asynchronous events** — instead of calling the owning service synchronously on every read.

Example: the Order service needs product name/price at order time. Rather than calling Product Service on every order, it keeps a local, denormalized `product_snapshot` table updated via events.

```java
@KafkaListener(topics = "product-events")
public void onProductUpdated(ProductUpdatedEvent event) {
    productSnapshotRepository.save(new ProductSnapshot(
        event.getProductId(), event.getName(), event.getPrice()
    ));
}
```

- **Owning service** = source of truth, publishes change events on every write.
- **Consuming services** = subscribe and maintain their own local, purpose-built copy — eventually consistent, not real-time.

**Senior-level answer:**
> "Duplication looks wasteful coming from a normalized-database mindset, but in distributed systems it's the standard trade: you accept storage duplication and eventual consistency in exchange for removing a runtime dependency and its associated latency/availability coupling. The Order service being unable to take orders because the Product service is down is a much worse outage than the Order service showing a product name that's a few seconds stale."

**Trap:** Candidates often propose "just call the other service synchronously via REST/Feign whenever you need that data." That's fine for low-volume, non-critical-path lookups, but at scale it creates a **cascading failure risk** — if Product Service is slow/down, every Order request now blocks or fails too. This is exactly the coupling database-per-service was meant to avoid, just re-introduced at the API layer instead of the DB layer.

---

### Q4. How do you handle the fact that the local copy can become stale or out of sync with the source of truth?
**Answer:** A few concrete techniques, usually combined:
- **Event versioning** — include a version/timestamp on each event; consumers discard out-of-order/stale updates (`if (incoming.version <= local.version) return;`).
- **Idempotent event handlers** (Q11–Q12) — safe to reprocess the same event without corrupting state.
- **Periodic reconciliation jobs** — a scheduled job that compares a sample or full snapshot against the source of truth and self-heals drift caused by missed/lost events.
- **TTL-based fallback** — treat the local copy as a cache with an expiry; on cache miss/expiry, fall back to a (rate-limited) synchronous call to the source.

```java
@Scheduled(cron = "0 0 3 * * *")
public void reconcileProductSnapshots() {
    List<ProductDto> source = productClient.getAllProducts(); // batch reconciliation call
    source.forEach(p -> productSnapshotRepository.upsertIfNewer(p));
}
```

**Senior-level answer:**
> "I treat eventual consistency as a spectrum, not a binary. Most business flows can tolerate seconds of staleness fine — but I always ask 'what's the actual business impact if this is 10 minutes stale?' for each field, because that answer tells you whether you need pure event-driven sync, a reconciliation job as a safety net, or a stronger read-through pattern for the handful of fields that genuinely can't drift, like available inventory count at checkout."

---

## SECTION 3: DISTRIBUTED QUERIES

### Q5. How do you implement a query that needs to join data owned by three different services (e.g., "orders with customer name and product details")?
**Answer:** There's no distributed JOIN across service databases — you have to compose the result at the **application/API layer**, not the database layer. Common patterns:

| Pattern | How it works | Best for |
|---|---|---|
| **API Composition** | A composer (often an API Gateway or a dedicated aggregator service) calls each service, then joins results in memory | Simple, low-fan-out queries; real-time accuracy needed |
| **CQRS read model** | A separate service maintains a pre-joined, denormalized read table built from events (Q6–Q7) | High-volume, complex, or frequently-repeated queries |
| **GraphQL federation** | A gateway composes a single GraphQL schema stitched from multiple service subgraphs | Client-facing APIs needing flexible field selection |

```java
// API Composition example
public OrderDetailsDto getOrderDetails(String orderId) {
    Order order = orderClient.getOrder(orderId);
    Customer customer = customerClient.getCustomer(order.getCustomerId());     // parallel calls
    List<Product> products = productClient.getProducts(order.getProductIds()); // via WebClient/CompletableFuture
    return new OrderDetailsDto(order, customer, products);
}
```

**Senior-level answer:**
> "API composition is the right default for simple, low-cardinality joins — it's simple and always reflects current data. It falls apart once you need aggregation across a large dataset, e.g. 'total revenue per customer across all orders and refunds' — fanning that out to N services per request doesn't scale and turns one slow downstream service into a slow query for everyone. That's the signal to move to a CQRS read model instead."

**Trap:** Don't suggest "just give the reporting team direct read access to each service's production database." Even read-only, that couples your internal schema to external consumers permanently — you can never change that schema without breaking their reports. Expose a proper read API or a purpose-built read replica/data warehouse instead.

---

### Q6. What's the practical difference between API Composition and building a dedicated CQRS read model for cross-service queries?
**Answer:**
| | API Composition | CQRS Read Model |
|---|---|---|
| **Data freshness** | Always current (real-time calls) | Eventually consistent (built from events, lag of ms–seconds) |
| **Latency** | Bound by slowest downstream call, done at query time | Fast — pre-joined, indexed, queried directly |
| **Load on source services** | High — every query hits N services | Low — source services just publish events once |
| **Complexity** | Simple to build initially | Requires event pipeline + a new service/store to maintain |
| **Good for** | Low-volume, simple joins, admin tools | High-volume reads, complex aggregations, reporting/search |

**Senior-level answer:**
> "I use API composition by default because it's the simplest thing that works, and I only invest in a CQRS read model once there's a proven, recurring query pattern that either can't tolerate the fan-out latency or is hit frequently enough that hammering three services per request becomes the bottleneck. Building the read model prematurely is over-engineering; not building it once you've got a dashboard doing 12 downstream calls per page load is under-engineering."

---

## SECTION 4: CQRS & EVENT SOURCING

### Q7. What is CQRS, and what problem does it actually solve?
**Answer:** **CQRS (Command Query Responsibility Segregation)** splits the **write model** (handles commands, enforces business rules/invariants) from the **read model** (optimized purely for querying), instead of using one model for both.

```
Write side:  Command -> Order Aggregate (validates, applies business rules) -> writes to Write DB -> publishes OrderPlaced event
Read side:   OrderPlaced event -> Read Model Updater -> writes denormalized row to Read DB (optimized for the exact queries the UI needs)
```

- The **write model** stays normalized and focused on correctness/consistency.
- The **read model** can be denormalized, duplicated across multiple shapes, and even use a different data store (e.g., Elasticsearch for search, Redis for a leaderboard) per query need — each shaped exactly for the query it serves, with no joins required at read time.

**Senior-level answer:**
> "The core insight is that read and write workloads have fundamentally different requirements — writes need strong consistency and business-rule enforcement; reads need speed and flexible shaping for the UI. Forcing one model to serve both usually means the read side is full of DTOs and read-only projections bolted onto a write-optimized schema anyway. CQRS just makes that split explicit and lets each side scale and evolve independently."

**Trap:** CQRS does **not** require Event Sourcing — they're frequently used together but are independent decisions. You can do CQRS with a plain write DB that publishes change events (via outbox, Q9) to update a read DB, with no event log as the system of record at all. Conflating the two is a very common interview mistake.

---

### Q8. What is Event Sourcing, and how is it different from just publishing events after each write?
**Answer:** In **Event Sourcing**, the **sequence of domain events IS the system of record** — you don't store current state directly; you store every state-changing event ever applied to an entity, and current state is derived by **replaying** those events.

```java
// Traditional: store current state
Order { id, status: "CONFIRMED", total: 150.00 }

// Event Sourced: store the event log; state is derived
OrderCreated{orderId, items}
ItemAdded{orderId, item}
OrderConfirmed{orderId}
// current state = fold/replay all events for orderId
```

```java
public Order rebuildState(String orderId) {
    List<DomainEvent> events = eventStore.getEvents(orderId);
    Order order = new Order();
    events.forEach(order::apply);   // each event mutates state incrementally
    return order;
}
```

**Key benefits:** complete audit trail for free, ability to rebuild any past state, ability to derive new read models retroactively by replaying history.
**Key costs:** replaying long event streams is slow without **snapshots** (periodic state checkpoints); querying current state directly is awkward (hence pairing with CQRS read models); schema evolution of old events (**event versioning/upcasting**) is a real long-term maintenance burden.

**Senior-level answer:**
> "I reserve full Event Sourcing for domains where the history itself has business value — financial ledgers, audit-heavy domains, inventory movements — not as a default architecture. For most services, 'store current state + publish domain events via outbox' gets you 90% of the integration benefit with far less operational complexity than maintaining an event store, snapshotting strategy, and event schema versioning long-term."

---

## SECTION 5: TRANSACTIONAL OUTBOX

### Q9. What problem does the Transactional Outbox pattern solve?
**Answer:** It solves the **dual-write problem**: a service needs to (1) update its own database AND (2) publish an event to a message broker, as a single atomic unit — but a local DB transaction and a broker publish **cannot** be committed together atomically. If the DB commit succeeds but the broker publish fails (network blip, broker down), you've silently lost an event; if you publish first and the DB commit then fails, you've published an event for something that never actually happened.

**The fix:** write the event to an **outbox table in the same local database transaction** as the business data change — so it's atomic by definition (same DB, same transaction). A separate poller/relay process then reads unpublished rows from the outbox and publishes them to the broker, marking them sent.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);                                // business write
    outboxRepository.save(new OutboxEvent(                      // event write — SAME transaction, SAME DB
        "Order", order.getId(), "OrderPlaced", toJson(order)
    ));
}   // both commit together, or both roll back together
```

```java
@Scheduled(fixedDelay = 500)
public void relayOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findUnpublished();
    pending.forEach(e -> {
        kafkaTemplate.send(e.getTopic(), e.getPayload());
        outboxRepository.markPublished(e.getId());
    });
}
```

**Senior-level answer:**
> "The dual-write problem is one of those things that looks like a non-issue in a demo and becomes a real production data-loss bug under partial failure — a broker timeout during a deploy, a network partition, whatever. Transactional Outbox is the standard fix because it reduces the problem to 'can I write two rows in one local ACID transaction,' which every relational database already guarantees, instead of trying to coordinate atomicity across two completely different systems."

---

### Q10. How does the outbox relay process actually read and publish the events — polling the table directly, or something else?
**Answer:** Two common implementations:

| Approach | How it works | Trade-off |
|---|---|---|
| **Polling publisher** | A scheduled job periodically `SELECT`s unpublished rows and publishes them | Simple to implement; adds polling latency and DB read load |
| **CDC (Change Data Capture)** | A tool like **Debezium** tails the database's transaction/binlog and streams outbox inserts to Kafka directly | Near-zero latency, no polling load on the DB; adds an operational dependency (Debezium/Kafka Connect) |

```yaml
# Debezium outbox event router config (conceptual)
transforms: outbox
transforms.outbox.type: io.debezium.transforms.outbox.EventRouter
transforms.outbox.table.field.event.id: id
transforms.outbox.table.field.event.payload: payload
```

**Senior-level answer:**
> "I default to a simple polling relay for most teams — it's trivial to reason about and debug, and sub-second polling is fast enough for the vast majority of business events. I only reach for Debezium/CDC once there's a genuine near-real-time requirement or the polling load itself becomes measurable on a high-write table — it's strictly more powerful but it's also another piece of infrastructure your team now owns and has to keep healthy."

**Trap:** Whichever mechanism you use, the relay **must** delete or mark-published the row only *after* a confirmed broker acknowledgment — not before. Marking it published optimistically before the broker ack risks losing the event on a crash between the two steps, which reintroduces the exact problem outbox was built to solve.

---

## SECTION 6: IDEMPOTENT CONSUMER & INBOX PATTERN

### Q11. Why does an event consumer need to be idempotent even if the producer only sends each event once?
**Answer:** Because **"exactly-once delivery" doesn't really exist end-to-end** across a network + broker — brokers (Kafka included) provide **at-least-once delivery** in practice: a consumer can process a message, crash/fail *before* committing its offset, and receive the **same message again** on redelivery/rebalance. If processing has a side effect (charging a card, decrementing inventory), reprocessing it naively double-applies that side effect.

**Idempotent Consumer** = designing the message handler so that processing the same message **multiple times produces the same end result as processing it once.**

```java
@KafkaListener(topics = "payment-events")
@Transactional
public void handlePaymentEvent(PaymentEvent event) {
    if (processedEventRepository.existsById(event.getEventId())) {
        return; // already processed — safe no-op
    }
    walletService.debit(event.getAccountId(), event.getAmount());
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

**Senior-level answer:**
> "At-least-once delivery is the realistic default you should design for in any distributed messaging system — treating 'my broker guarantees exactly-once' as a reason to skip idempotency is a mistake I've seen bite teams hard, because even Kafka's exactly-once semantics only cover producer-to-broker and broker-to-broker atomicity within Kafka itself, not the consumer's own side-effecting business logic. Idempotency at the consumer is what actually protects you."

**Trap:** A common but broken "idempotency" attempt: checking `existsById` and inserting the processed-event record in **two separate statements/transactions**. If the process crashes between them, you can still double-process on retry. The dedup check and the business-side-effect write need to be in the **same local transaction** — this is exactly what the Inbox pattern formalizes (Q12).

---

### Q12. What is the Inbox pattern, and how does it relate to the Transactional Outbox on the producer side?
**Answer:** The **Inbox pattern** is the consumer-side mirror of Outbox: an `inbox` table records every message ID already processed, and the **check for "have I seen this before" plus the business state change happen in one local database transaction** — guaranteeing idempotency even across consumer crashes/redeliveries.

```java
@Transactional
public void handleOrderPlaced(OrderPlacedEvent event) {
    boolean alreadyProcessed = inboxRepository.existsByMessageId(event.getMessageId());
    if (alreadyProcessed) return;

    inventoryService.reserveStock(event.getItems());     // business effect
    inboxRepository.save(new InboxRecord(event.getMessageId(), Instant.now())); // dedup marker
}   // both committed atomically — dedup record can never get out of sync with the effect
```

| | Transactional Outbox (producer) | Inbox (consumer) |
|---|---|---|
| **Guarantees** | Event is published if and only if the local DB write committed | Event is applied if and only if it hasn't already been applied |
| **Mechanism** | Outbox table written in the same TX as the business write | Inbox table checked/written in the same TX as the business write |
| **Solves** | Dual-write problem (DB write + publish) | Duplicate delivery problem (at-least-once redelivery) |

**Senior-level answer:**
> "Outbox and Inbox are really the same idea applied to opposite ends of the pipe: 'make the messaging concern atomic with the local database transaction.' On the producer side that guarantees you never lose an event; on the consumer side it guarantees you never double-apply one. I think of end-to-end reliable event-driven integration as needing *both* — an outbox alone doesn't protect the downstream consumer, and an idempotent consumer alone doesn't protect the producer from dual-write loss."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can two services share a database schema temporarily during a migration?" → Sometimes used as a deliberate, time-boxed strangler-fig transition step, but it should be treated as technical debt with an explicit exit plan, not a long-term architecture.
- "Is eventual consistency the same as 'eventually correct'?" → Not automatically — you also need idempotent handlers and ordering guarantees (or version checks) per entity, or "eventual" consistency can converge on the wrong final state if events are applied out of order.
- "Does Kafka guarantee message ordering across partitions?" → No — ordering is only guaranteed **within a single partition**; events for the same entity should share a partition key (e.g., orderId) to preserve per-entity ordering.
- "What happens if the outbox relay publishes an event twice?" → That's expected and fine — it's exactly why the consumer side needs the Inbox/idempotency pattern; outbox gives at-least-once delivery, not exactly-once.
- "Is Event Sourcing required to implement CQRS?" → No, they're independent patterns commonly paired together but neither requires the other.
- "How do you handle a saga step that has no natural compensating action (e.g., an email already sent)?" → Accept it as a semantic compensation (e.g., send a follow-up correction/cancellation email) rather than a true undo — not everything can be rolled back technically.

---

*Study tip: The dual-write problem and why Transactional Outbox solves it (Q9), the distinction between at-least-once delivery and idempotent consumers via the Inbox pattern (Q11–Q12), and knowing precisely when CQRS/Event Sourcing earn their added complexity versus when simple event-driven sync suffices (Q7–Q8) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining the failure mode each pattern prevents, not just its mechanics.*
