# Decomposition & Design Patterns — Interview Prep
---

## SECTION 1: DECOMPOSITION STRATEGY & SERVICE GRANULARITY

### Q1. What's the correct way to decompose a system into services — and why is "decompose by business capability" preferred over other approaches?
**Answer:** The DDD-aligned, industry-recommended approach is **decomposition by business capability / subdomain** (see MOD4_002 Q1, Q8) — each service maps to a cohesive piece of what the business *does* (Order Management, Inventory, Payments), not to a technical layer.

**Common decomposition strategies, ranked by how well they hold up in production:**
| Strategy | What it looks like | Why it works / fails |
|---|---|---|
| **By business capability** (preferred) | OrderService, InventoryService, PaymentService | High cohesion — each service is a complete vertical slice; teams own end-to-end business outcomes |
| **By subdomain (DDD)** | One service per bounded context identified via Event Storming | Same as above, formalized through DDD's process (MOD4_002 Q8) |
| **By technical layer** (anti-pattern) | ValidationService, DatabaseService, UIService | Low cohesion, high coupling — nearly every business operation needs to call multiple layer-services, producing chatty synchronous chains and a distributed monolith |
| **By verb/use-case** (over-granular) | CreateOrderService, CancelOrderService, UpdateOrderService | Excessive fragmentation — logically one aggregate's lifecycle gets scattered across services that must all stay in lockstep |

**Senior-level answer:** "I always decompose along business capability boundaries discovered through Event Storming, not around technical concerns. The test I apply is: does this service represent something a business stakeholder would recognize and describe in one sentence — 'this handles our orders' — or is it a technical artifact that only makes sense to an engineer? If it's the latter, it's usually the wrong boundary."

---

### Q2. How do you decide the right granularity for a service — too big vs too small, and what actually goes wrong at each extreme?
**Answer:** Granularity isn't about lines of code or a target service count — it's about whether the boundary matches a genuine bounded context (MOD4_002 Q2) and a real independent-scaling/team-ownership need (MOD4_001 Q6).

**Too coarse (service doing too much):**
- Multiple unrelated bounded contexts crammed into one service → different parts change for different reasons but must be redeployed together → essentially a mini-monolith with network overhead layered on top for no benefit.
- Team ownership becomes unclear — too many people touching one service, release coordination overhead creeps back in.

**Too fine (over-decomposition — the more common real-world mistake I see):**
- **Chatty inter-service communication** — a single business operation now requires 5+ synchronous network hops, multiplying latency and failure surface.
- **Distributed transactions everywhere** — splitting what should be one aggregate (Q5, MOD4_002) into two services forces you to solve consistency with Sagas (Q4) for something that didn't need it.
- **Operational tax outpaces the team** — more services than the team can realistically monitor, deploy, and be on-call for — each with its own pipeline, dashboards, and alerting.
- **"Nanoservices" anti-pattern** — a service so small it's just a thin wrapper around a single table/CRUD operation, with no real independent business meaning.

**Senior talking point:** "My practical heuristic: if two 'services' always change together, always deploy together, or one is meaningless without the other being available synchronously — merge them, or at minimum question whether the boundary was drawn correctly. Granularity should fall out of bounded contexts and team-ownership realities, not out of a stylistic preference for 'small services.'"

---

## SECTION 2: DATABASE PER SERVICE & SHARED DATABASE PROBLEMS

### Q3. Why is "Database per Service" a core microservices principle, and what specifically breaks when services share a database?
**Answer:** Database-per-Service means each service **owns its own schema/database exclusively**, and every other service must go through that service's API to read or write its data — never direct SQL access.

**What breaks with a shared database (real, recurring production issues):**
- **Hidden coupling via schema** — Service B silently depends on a column Service A's team didn't know was in use; A's team renames it during a routine refactor, B breaks in production with no compile-time warning anywhere.
- **No independent deployability** — a schema migration must be coordinated across every service touching that table, defeating the entire purpose of "independent" services.
- **Uncontrolled write conflicts** — two services writing to the same table under different assumptions about valid state (e.g., one service allows a status transition the other's business rules forbid) → an aggregate's invariants (MOD4_002 Q5) are enforced nowhere consistently.
- **Scaling coupling** — you can't scale/tune the database differently per service's actual load profile (e.g., a read-heavy catalog vs. a write-heavy order service) — they're stuck sharing one instance's characteristics.
- **Technology lock-in** — one service might genuinely benefit from a document store or a graph DB for its access patterns; a shared relational database forces a one-size-fits-all choice on every consumer.

```java
// ANTI-PATTERN — Inventory service reaching directly into Order's table
@Query(value = "SELECT * FROM orders WHERE product_id = :productId", nativeQuery = true)
List<OrderRow> findOrdersByProduct(@Param("productId") Long productId);
// Breaks the moment Order-service changes its schema — Inventory has no contract, just a table it doesn't own

// CORRECT — go through Order's API/contract instead
OrderSummary summary = orderServiceClient.getOrdersByProduct(productId);
```

**Follow-up: "What if two services genuinely need the same data constantly and an API call is too slow?"** → Options, from least to most complex: (1) have the owning service expose a purpose-built read API/cache, (2) publish domain events the consumer projects into its **own local read model** (this is effectively CQRS at the integration level, Q5), (3) as a last resort, a tightly-scoped **read replica** exposed read-only — but the write path must always stay owned by a single service.

---

## SECTION 3: STRANGLER FIG & ANTI-CORRUPTION LAYER

### Q4. What is the Strangler Fig pattern, and how would you actually use it to migrate a legacy monolith to microservices without a risky big-bang rewrite?
**Answer:** Named after the strangler fig vine that grows around a host tree and gradually replaces it — the pattern **incrementally routes traffic for specific functionality away from the legacy system to a new service**, one capability at a time, until the legacy system has nothing left to do and can be safely decommissioned. At no point is there a single high-risk cutover.

**Practical implementation steps:**
1. Put a **facade/routing layer** (often an API Gateway or reverse proxy) in front of the legacy monolith.
2. Pick **one bounded context** (e.g., "Order Search") to extract first — ideally something with clear boundaries and lower risk.
3. Build the new microservice implementing that capability; the routing layer sends **only that capability's traffic** to the new service, everything else still goes to the monolith.
4. Validate in production (often with a **shadow/dark-launch** phase — new service receives traffic and its output is compared against the legacy path without actually being used yet) before fully cutting over.
5. Repeat capability-by-capability until the monolith is "strangled" down to nothing, then retire it.

```
Client → [API Gateway / Router]
              ├── /api/orders/search → NEW Order-Search-Service
              ├── /api/orders/**     → Legacy Monolith (not yet migrated)
              └── /api/inventory/**  → Legacy Monolith (not yet migrated)
```

**Senior talking point:** "The value of Strangler Fig isn't just technical risk reduction — it lets the business keep shipping features on the legacy system while migration happens in parallel, and if a migrated capability turns out to have a problem, you can route traffic back to the legacy path instantly instead of rolling back a full rewrite. I've seen 'big bang rewrite' projects fail specifically because the business couldn't freeze feature development for the 12+ months the rewrite needed — Strangler Fig avoids that trap entirely."

---

### Q5. What is an Anti-Corruption Layer, and how is it different from Strangler Fig — don't they sound similar?
**Answer:** They're related but solve different problems. An **Anti-Corruption Layer (ACL)** is a **translation layer** that converts an external/legacy system's model into your own clean domain model, so that system's quirks, naming, or outdated concepts never leak into your bounded context. It's a **permanent architectural boundary**, not a temporary migration step.

**Strangler Fig vs ACL — the distinction interviewers want to hear:**
| | Strangler Fig | Anti-Corruption Layer |
|---|---|---|
| Purpose | **Migrate away from** a legacy system over time | **Coexist with** a legacy/external system indefinitely |
| Lifespan | Temporary — removed once migration completes | Often permanent, as long as the integration exists |
| Direction | Routes traffic progressively to a *replacement* | Translates data/calls between two systems that both continue to exist |

```java
// Anti-Corruption Layer — legacy mainframe uses cryptic codes; our domain uses clean concepts
@Component
public class LegacyInventoryAntiCorruptionLayer {

    public StockLevel toDomainModel(LegacyMainframeResponse legacyResponse) {
        // Translate legacy's field names, units, and status codes into our clean domain model
        Quantity qty = Quantity.fromLegacyUnits(legacyResponse.getQtyFld(), legacyResponse.getUomCode());
        StockStatus status = mapLegacyStatusCode(legacyResponse.getStatCd());  // "A1" -> IN_STOCK, "B7" -> BACKORDERED
        return new StockLevel(qty, status);
    }
}
```

**When you'd use both together:** during a Strangler Fig migration, the ACL is often the exact component sitting inside the new service that talks *back* to the still-not-yet-migrated legacy system for data it doesn't own yet — so the new service's domain model stays clean even while depending on the legacy system temporarily.

---

## SECTION 4: SAGA — CHOREOGRAPHY vs ORCHESTRATION

### Q6. Why can't you just use a distributed transaction (2PC) across microservices, and what does a Saga do instead?
**Answer:** Two-Phase Commit requires all participating services to **hold locks and block** until every participant votes commit/abort — this is fundamentally incompatible with microservices' goals of availability and independent scaling (it directly conflicts with the CAP theorem trade-offs you're already making), and most modern data stores/brokers don't support XA transactions reliably at scale anyway. A **Saga** achieves consistency across services using a **sequence of local transactions**, each service committing its own local transaction and publishing an event/command to trigger the next step — with **explicit compensating transactions** to undo prior steps if something later fails.

```
Happy path:  OrderService (create DRAFT order) 
                → PaymentService (charge card) 
                    → InventoryService (reserve stock) 
                        → OrderService (mark CONFIRMED)

Failure path (inventory reservation fails):
                → InventoryService (reservation FAILS)
                    → PaymentService (compensate: REFUND)
                        → OrderService (compensate: mark CANCELLED)
```

**Key trade-off to state proactively:** "A Saga gives up ACID atomicity for **eventual consistency** — there's a window where the order exists as DRAFT before payment is confirmed. That's usually an acceptable business trade-off, but it has to be an explicit decision made with product/business stakeholders, not something engineering quietly accepts — 'the order briefly appears unpaid' needs sign-off, not just an the implementation detail."

---

### Q7. Choreography vs Orchestration — compare them, and tell me which you'd pick and why.
**Answer:**
| Aspect | Choreography | Orchestration |
|---|---|---|
| How it works | Each service publishes events; other services **subscribe and react** independently — no central coordinator | A central **Saga Orchestrator** explicitly tells each service what to do next, step by step |
| Coupling | Services only know about events, not each other directly — **loosely coupled** | Orchestrator knows about every participant — coupling is centralized there |
| Visibility of the overall flow | Implicit — scattered across every service's event handlers; hard to see "the whole saga" in one place | Explicit — the orchestrator's state machine **is** the documentation of the flow |
| Complexity as steps grow | Grows messy fast — "event soup," hard to trace which service reacts to what | Stays manageable — one place to look, one place to add a step |
| Single point of failure | None architecturally — but debugging is harder without a full picture | Orchestrator itself must be made highly available |
| Best for | Simple sagas (2-3 steps), highly decoupled teams | Complex sagas (4+ steps), when the business explicitly needs to track/observe saga state (e.g., "show the customer their order's real-time progress") |

```java
// Orchestration — explicit, centralized coordinator (often built with a state machine / Camunda / Temporal)
@Component
public class OrderSagaOrchestrator {

    public void handle(OrderCreatedEvent event) {
        try {
            paymentClient.charge(event.getOrderId(), event.getAmount());
            inventoryClient.reserve(event.getOrderId(), event.getItems());
            orderClient.confirm(event.getOrderId());
        } catch (PaymentFailedException e) {
            orderClient.cancel(event.getOrderId());
        } catch (InventoryReservationFailedException e) {
            paymentClient.refund(event.getOrderId());
            orderClient.cancel(event.getOrderId());
        }
    }
}
```

**Senior-level answer:** "I default to **Choreography for short sagas** (2-3 steps) where the coupling stays manageable, and I move to **Orchestration** the moment the saga grows past 3-4 steps, needs conditional branching, or the business wants observable saga state/progress. I've personally seen a choreographed saga become nearly undebuggable once it grew to 6+ services all reacting to each other's events — nobody could answer 'what happens next if payment fails' without tracing through six separate codebases. That's the exact point orchestration should have been introduced instead."

---

## SECTION 5: CQRS & EVENT SOURCING

### Q8. What is CQRS, and what real problem does it solve that a standard CRUD service doesn't?
**Answer:** CQRS (Command Query Responsibility Segregation) **separates the write model (Commands) from the read model (Queries)** — instead of one model/schema serving both writes and reads, you maintain **independent models**, each optimized for its own purpose, kept in sync via events.

**Problem it solves:** in a standard CRUD model, the same schema has to satisfy both **transactional write correctness** (normalized, enforcing invariants) and **fast, flexible read patterns** (often denormalized, aggregated across multiple entities, filtered many different ways) — these two needs frequently pull schema design in opposite directions, especially at scale.

```java
// WRITE side — normalized, invariant-enforcing, goes through the Order aggregate
@Service
public class OrderCommandService {
    public void placeOrder(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.getCustomerId(), cmd.getItems());  // invariants enforced here
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlacedEvent(order.getId(), order.getItems()));
    }
}

// READ side — denormalized, purpose-built for a specific query pattern, updated asynchronously via events
@Entity
public class OrderSummaryReadModel {   // flat, pre-joined — no runtime joins needed
    private String orderId;
    private String customerName;   // denormalized from Customer service
    private BigDecimal totalAmount;
    private String status;
}

@EventListener
public void on(OrderPlacedEvent event) {
    // project the event into the read-optimized model
    readModelRepository.save(new OrderSummaryReadModel(event));
}
```

**Senior-level nuance:** "CQRS doesn't have to mean two separate databases — a lightweight form is just separate DTOs/read-models within the same service and database. I only reach for full CQRS with **separate physical stores** (e.g., Postgres for writes, Elasticsearch for reads) when the read and write scaling/query needs have genuinely diverged enough to justify the added synchronization complexity — it's a real trade-off, not a default pattern to apply everywhere."

---

### Q9. What is Event Sourcing, and how is it different from just "publishing events" for integration? What's the catch?
**Answer:** Event Sourcing means the **event log itself is the system of record** — instead of persisting current state (`status = SHIPPED`), you persist the **full sequence of domain events** that led to that state (`OrderPlaced → PaymentAuthorized → OrderShipped`), and current state is **derived by replaying those events**. This is fundamentally different from just publishing integration events for other services to consume — here, events aren't a side-effect of persistence, **they are the persistence.**

```java
// Instead of storing current state directly...
public class Order {
    private OrderStatus status;   // traditional: just the latest value
}

// ...Event Sourcing stores and replays the event stream
public class Order {
    public static Order rehydrate(List<DomainEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);   // replay each event to rebuild current state
        return order;
    }

    private void apply(OrderPlacedEvent e)   { this.status = OrderStatus.PLACED; this.items = e.getItems(); }
    private void apply(PaymentAuthorizedEvent e) { this.status = OrderStatus.PAID; }
    private void apply(OrderShippedEvent e)  { this.status = OrderStatus.SHIPPED; }
}
```

**Genuine benefits:** complete audit trail for free (every state change is preserved, not just the latest value — valuable in fintech/regulated domains), ability to **rebuild new read models retroactively** from history, and natural fit with CQRS.

**The catch — real costs to state proactively:**
- **Querying current state requires replaying events** (or maintaining a snapshot + read model to avoid replaying from the beginning every time) — non-trivial infrastructure.
- **Schema evolution of events is hard** — you can never delete or freely change an old event's shape once it's persisted; you need upcasting/versioning strategies for events written years ago.
- **Steep learning curve and debugging difficulty** — "what is this order's current state" is no longer a simple `SELECT`; engineers unfamiliar with the pattern find it genuinely harder to reason about.

**Senior answer:** "I treat Event Sourcing as a specialized tool for aggregates where the **history itself has business value** — financial ledgers, audit-heavy domains — not a default persistence strategy. For most aggregates, a normal state-store plus separately published integration events for other services gives most of the benefit without the full operational complexity of event sourcing."

---

## SECTION 6: TRANSACTIONAL OUTBOX

### Q10. What problem does the Transactional Outbox pattern solve, and why can't you just publish an event right after saving to the database?
**Answer:** The problem: if you `save()` to the database and then separately call `kafkaTemplate.send()`, those are **two independent operations with no shared transaction** — if the app crashes (or the broker call fails) after the DB commit but before the publish succeeds, you have a **committed order with no event ever published** — silent, permanent data inconsistency between services. Doing it in the other order (publish first, then save) risks the opposite: an event fired for a write that then fails to commit.

**The pattern:** write the domain change **and** the outgoing event to an **outbox table, in the same local database transaction** — then a separate, asynchronous process (polling, or **Change Data Capture** via Debezium reading the DB's transaction log) reliably publishes the outbox rows to the message broker, deleting/marking them once confirmed.

```java
@Transactional
public void placeOrder(PlaceOrderCommand cmd) {
    Order order = new Order(cmd);
    orderRepository.save(order);                          // 1. business write

    OutboxEvent event = new OutboxEvent(
        "Order", order.getId(), "OrderPlaced", toJson(order));
    outboxRepository.save(event);                          // 2. event write — SAME transaction, SAME commit
}
// A separate poller/Debezium-CDC process reads unpublished outbox rows
// and publishes them to Kafka, then marks them as sent — guaranteed at-least-once delivery
```

**Senior talking point:** "This is the standard, production-proven way to get **atomicity between a database write and a message publish** without a distributed transaction — the trick is that both writes go into the *same* local ACID transaction, and reliability shifts to the outbox-relay process instead, which just needs to guarantee at-least-once delivery. Consumers then need to be **idempotent** (Q11's neighbor topic in the DLQ/idempotency space) since at-least-once means occasional duplicate delivery is expected, not exceptional."

---

## SECTION 7: BFF, SIDECAR & AMBASSADOR PATTERNS

### Q11. What is the Backend-for-Frontend (BFF) pattern, and why not just have one generic API Gateway serve every client?
**Answer:** BFF means building a **dedicated backend layer per client type** (web, mobile, third-party partner) rather than forcing every client to consume the same one-size-fits-all API. Each BFF **aggregates and shapes data from multiple downstream microservices into exactly the payload its specific client needs.**

**Why not one generic gateway for everyone:** a mobile app typically wants a **small, aggregated payload** (limited bandwidth, battery, fewer round trips) while a web dashboard might want a **richer, more granular response**; forcing a single shared API to satisfy both leads to either over-fetching for mobile or multiple round-trips for web — and every client-specific tweak risks breaking the other client types sharing the same endpoint.

```
Mobile App  → Mobile-BFF   → aggregates: OrderService + InventoryService (minimal fields, 1 call)
Web Client  → Web-BFF      → aggregates: OrderService + InventoryService + RecommendationService (richer payload)
Partner API → Partner-BFF  → different auth model, different rate limits, different data shape entirely
```

**Senior-level trade-off:** "BFF adds another layer to maintain per client type, so I only introduce it once client needs have genuinely diverged — different data shapes, different performance constraints, different release cadences per client team. For a single web client with no mobile app, a plain API Gateway is enough; BFF earns its complexity once you have multiple, meaningfully different consumers each owned by different frontend teams."

---

### Q12. What are the Sidecar and Ambassador patterns, and how are they different from each other?
**Answer:** Both patterns run a **helper process alongside your main application container**, but for different purposes.

**Sidecar** — a **generic companion process** deployed in the same pod, handling **cross-cutting infrastructure concerns** so your application code doesn't have to: log shipping, metrics collection, TLS termination, service-mesh proxying (Q13). It shares the pod's lifecycle and network namespace.

**Ambassador** — a **specialized sidecar** specifically acting as an **outbound proxy** on behalf of the application — handling things like retries, circuit breaking, or service discovery for **outgoing** calls, so the application code just calls `localhost` and the ambassador handles the real network complexity to the actual downstream service.

```
Pod:
  ┌─────────────────────────────────────┐
  │  App Container                       │
  │   → calls localhost:9090 (ambassador)│
  ├───────────────────────────────────────┤
  │  Ambassador (outbound proxy) :9090   │  → handles retries/discovery to real downstream service
  ├───────────────────────────────────────┤
  │  Sidecar: log-shipper, metrics-agent │  → cross-cutting concerns, no app code changes needed
  └─────────────────────────────────────┘
```

**Practical example:** "Instead of every service implementing its own retry/TLS/mTLS logic in application code (and every team implementing it slightly differently), we deploy an **Envoy sidecar** per pod that transparently handles mTLS, retries, and load balancing for all outbound calls — the application code just makes a plain HTTP call to `localhost`, completely unaware the sidecar is doing all that work. This is precisely the foundation a **Service Mesh** is built on (Q13) — a service mesh is essentially 'sidecar pattern applied uniformly, with centralized control.'"

---

## SECTION 8: SERVICE MESH

### Q13. What is a Service Mesh, and what problems does it solve that you'd otherwise have to build into every service individually?
**Answer:** A Service Mesh is a **dedicated infrastructure layer that handles service-to-service communication concerns** — mTLS encryption, retries, timeouts, circuit breaking, load balancing, traffic shifting/canary routing, and observability (metrics/tracing) — **uniformly, outside application code**, typically implemented as a **sidecar proxy** (Q12) deployed alongside every service instance, all managed by a central control plane (e.g., Istio, Linkerd).

**Without a service mesh:** every team re-implements retries, circuit breakers (Resilience4j), mTLS certificate rotation, and tracing instrumentation **inside their own application code**, in whatever language/framework they use — inconsistently, and with real risk of some teams doing it wrong or skipping it entirely.

**With a service mesh:** all of that moves into the **sidecar proxy**, controlled centrally — application code just makes a plain network call and the mesh transparently handles the rest.

```
                    ┌── Control Plane (Istio) ──┐
                    │  policy, config, certs      │
                    └─────────────┬───────────────┘
                                   │ configures
      ┌────────────────┐   mTLS   ┌────────────────┐
      │ Service A       │ ───────▶│ Service B       │
      │  + sidecar proxy│         │  + sidecar proxy│
      └────────────────┘         └────────────────┘
      (retries, circuit breaking, tracing — all handled by the proxies, not app code)
```

**Key capabilities worth naming explicitly in an interview:**
- **mTLS everywhere by default** — encrypted, authenticated service-to-service traffic without any application code changes.
- **Traffic shifting / canary deployments** — route 5% of traffic to a new version, centrally controlled, no per-service implementation needed.
- **Uniform observability** — golden-signal metrics and distributed tracing collected consistently across every service, regardless of language/framework.
- **Centralized resilience policy** — retry budgets, timeouts, circuit-breaking thresholds configured once at the mesh level instead of per-service code.

**Senior-level trade-off to state proactively:** "A service mesh is genuinely powerful, but it's real added operational complexity and latency overhead (every call now passes through two proxies) — I wouldn't introduce one for a handful of services; the payoff shows up once you have dozens-to-hundreds of services and the inconsistency/duplication of handling these concerns per-service becomes the bigger cost. For a smaller service count, a well-configured API Gateway plus per-service Resilience4j is often the more pragmatic, lower-overhead choice."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can Strangler Fig and BFF be used together?" → Yes — a common combo is routing through a BFF that itself acts as the Strangler Fig's facade, gradually shifting which downstream (legacy vs. new service) actually serves each capability.
- "Is CQRS the same as having separate microservices for reads and writes?" → Not necessarily — CQRS is a **pattern within or across services**; it can be applied inside a single service (separate DTOs/models) without any service-boundary implications at all.
- "Does Event Sourcing require Kafka?" → No — Event Sourcing is about how you **persist state** (an event store, which could even be a relational table); Kafka is commonly used to *also* publish those events for other consumers, but they're separate concerns.
- "What happens if the outbox relay process itself fails mid-publish?" → Because delivery is at-least-once, it may re-publish an already-sent event on recovery — consumers must be **idempotent** (dedupe by event ID) to handle safe re-delivery.
- "Is a Service Mesh a replacement for an API Gateway?" → No — they solve different traffic directions: API Gateway handles **north-south** traffic (external clients into the system); Service Mesh handles **east-west** traffic (service-to-service, internal).
- "Orchestration creates a central coordinator — doesn't that reintroduce the ESB problem from SOA?" → A fair concern, but the key difference is scope: a Saga Orchestrator coordinates **one specific business process's steps**, not all inter-service communication and business logic platform-wide the way an ESB did — keeping orchestrators narrowly scoped avoids recreating that bottleneck.

---

*Study tip: Choreography vs Orchestration trade-offs (Q7), the Transactional Outbox's role in solving the dual-write problem (Q10), and knowing exactly when to reach for CQRS/Event Sourcing vs when they're overkill (Q8-Q9) are the areas where interviewers most reliably separate candidates who've read pattern names from those who've actually implemented and debugged them under real failure conditions.*
