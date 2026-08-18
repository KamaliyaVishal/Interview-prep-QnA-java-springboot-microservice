# Architecture Patterns — Interview Prep
---

## SECTION 1: LAYERED ARCHITECTURE

### Q1. What is Layered (N-Tier) Architecture, and what's the core rule that makes it work?
**Answer:** Layered Architecture organizes code into **horizontal layers** — typically Presentation (Controllers), Business/Service, and Data Access (Repository) — where **each layer only depends on the layer directly below it**, and never on layers above it.

```
Presentation Layer  (Controllers, REST endpoints)
        |
Business Layer      (Services, business rules)
        |
Data Access Layer   (Repositories, DAOs)
        |
Database
```

```java
@RestController
class OrderController {
    private final OrderService orderService;   // depends downward only
}

@Service
class OrderService {
    private final OrderRepository orderRepository;   // depends downward only
}
```

**Senior-level answer:**
> "Layered architecture's entire value proposition is **separation of concerns by technical responsibility** — controllers handle HTTP, services handle business logic, repositories handle persistence. It's the default most Spring Boot applications start with because it maps naturally onto the framework's annotations, and it's genuinely fine for small-to-medium applications. Where it breaks down is at scale: business logic tends to leak upward into controllers or downward into repositories under time pressure, and because layers are organized by *technical* concern rather than *business* concern, a single feature change often touches every layer, which is the opposite of what you want for independent, low-risk changes."

**Trap:** Don't claim layered architecture inherently prevents a controller from calling a repository directly, skipping the service layer — that's a **discipline convention**, not something the pattern enforces structurally. Nothing in a typical layered Spring Boot setup stops that violation at compile time, which is a key limitation compared to Hexagonal/Clean architecture's explicit boundaries (Q3–Q5).

---

### Q2. What's the most common way Layered Architecture degrades in real, long-lived codebases?
**Answer:** The **"fat service" / "anemic domain model"** anti-pattern — business logic accumulates almost entirely in the Service layer, while domain entities become little more than getter/setter data bags with no behavior of their own. Over time, services grow enormous, testing requires standing up the whole layer stack, and the "business layer" becomes a dumping ground for every rule, regardless of which domain concept it actually belongs to.

```java
// Anemic model — Order has no behavior, all logic lives in the service
class Order {
    private String status;
    // just getters/setters
}

class OrderService {
    public void cancelOrder(Order order) {
        if (order.getStatus().equals("SHIPPED")) throw new IllegalStateException("Cannot cancel");
        order.setStatus("CANCELLED");   // business rule lives outside the entity it concerns
    }
}
```

**Senior-level answer:**
> "This is the practical, lived-experience answer versus the textbook one — I've inherited codebases where the 'business layer' was a 4,000-line `OrderService` class handling order creation, payment, shipping, and notification logic all in one place, purely because that's where 'business logic' was told to go. The fix isn't necessarily abandoning layered architecture entirely; it's often pushing behavior back into domain objects (a richer domain model) and splitting services by cohesive business capability rather than letting one service class absorb every rule that touches 'orders.'"

---

## SECTION 2: MODULAR MONOLITH

### Q3. What is a Modular Monolith, and how does it try to get the benefits of microservices without the operational cost?
**Answer:** A Modular Monolith is a **single deployable application** internally organized into **well-defined, loosely-coupled modules** (by business capability, not technical layer) with **explicit, enforced boundaries between modules** — modules communicate through defined interfaces/APIs, not by reaching into each other's internals, even though they all run in one process and deploy together.

```
com.example.app
 ├── orders/       (own package, own internal service/repo/domain, public API surface only)
 ├── inventory/    (cannot access orders' internal classes directly)
 ├── payments/
 └── shared-kernel/  (genuinely shared, minimal types)
```

```java
// orders module exposes only this — inventory module can't reach into OrderRepository directly
package com.example.app.orders.api;
public interface OrderQueries {
    OrderSummary getOrderSummary(String orderId);
}
```

**Senior-level answer:**
> "I position the modular monolith as the pragmatic middle ground for teams that aren't yet at the scale or organizational maturity that justifies microservices' operational overhead — distributed tracing, service discovery, network failure handling, independent deployment pipelines — but still want the *design discipline* of clear module boundaries. Done well, it's also the ideal precursor to microservices: if module boundaries are genuinely respected, extracting a module into its own deployable service later is a much smaller step, because the boundary already exists in code, not just in an org chart."

---

### Q4. How do you actually enforce module boundaries in a Modular Monolith so they don't silently erode over time, given it's all one codebase and one classpath?
**Answer:** A few concrete enforcement mechanisms, often layered together:
- **Package-private visibility** — internal classes within a module are package-private, not `public`, so other modules physically cannot import them; only explicitly `public` interface/API classes are usable from outside.
- **Architecture testing tools** — ArchUnit (Java) writes executable tests that fail the build if a forbidden dependency is introduced (e.g., "no class in `inventory` may import from `orders.internal`").
- **Build-tool module boundaries** — Java Platform Module System (JPMS), or separate Gradle/Maven modules per business module, which enforce boundaries at the build/compile level, not just convention.
- **Code review discipline** as the last line of defense — but this alone is the weakest and least reliable option.

```java
// ArchUnit — enforced as a test, fails the build on violation
@Test
void ordersModuleShouldNotDependOnInventoryInternals() {
    noClasses().that().resideInAPackage("..orders..")
        .should().dependOnClassesThat().resideInAPackage("..inventory.internal..")
        .check(importedClasses);
}
```

**Senior-level answer:**
> "This is exactly the question that separates 'we call it a modular monolith' from 'we actually have one' — without automated enforcement, module boundaries erode within a few sprints under deadline pressure, because nothing stops a developer from adding a quick import across modules 'just this once.' ArchUnit tests in CI are the single highest-leverage tool here — they turn an architectural intention into something the build literally cannot violate, which is a much stronger guarantee than a design doc or a code review comment."

---

## SECTION 3: HEXAGONAL ARCHITECTURE

### Q5. Explain Hexagonal Architecture (Ports & Adapters) — what problem is it actually solving?
**Answer:** Hexagonal Architecture puts the **business logic (the domain core) at the center**, completely isolated from external concerns — databases, web frameworks, message brokers, third-party APIs — via **Ports** (interfaces the core defines) and **Adapters** (implementations that plug into those ports from the outside). The core problem it solves: preventing infrastructure details from leaking into and coupling with business logic, so the domain can be tested and evolved **independently of any specific technology choice.**

```
        [REST Adapter]  [CLI Adapter]           <- driving/inbound adapters
                \            /
             [Port: OrderUseCase]  (interface)
                     |
             [Domain Core: Order business logic]  <- no framework/DB dependencies at all
                     |
             [Port: OrderRepository]  (interface, defined BY the core)
                /            \
      [JPA Adapter]   [MongoDB Adapter]          <- driven/outbound adapters
```

```java
// Port — defined by the domain, has zero knowledge of JPA/Mongo/anything concrete
interface OrderRepository {
    Order findById(String id);
    void save(Order order);
}

// Adapter — implements the port, lives OUTSIDE the domain core
@Repository
class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderJpaRepo jpaRepo;
    public Order findById(String id) { return jpaRepo.findById(id).map(this::toDomain).orElseThrow(); }
}
```

**Senior-level answer:**
> "The key inversion to articulate clearly: in a typical layered architecture, the business layer *depends on* the data access layer. In Hexagonal, that dependency is **inverted** — the domain core defines the `OrderRepository` interface it needs, and the persistence adapter depends on and implements that interface, not the other way around. That means you can swap Postgres for MongoDB, or add a CLI alongside a REST API, by writing new adapters, without touching a single line of business logic — and just as importantly, you can unit test the entire domain core with no database, no Spring context, and no web server running at all."

**Trap:** Don't describe Hexagonal Architecture as "just Layered Architecture with different names for the layers" — the fundamental difference is **dependency direction**. Layered architecture typically has business logic depending on data access; Hexagonal has the dependency arrow pointing the opposite way, into the domain core, via ports the core itself defines.

---

## SECTION 4: CLEAN ARCHITECTURE

### Q6. How does Clean Architecture relate to Hexagonal Architecture — are they the same thing, and what's the "Dependency Rule"?
**Answer:** Clean Architecture (Robert C. Martin) and Hexagonal Architecture share the **same core principle** — business logic isolated from infrastructure, dependencies pointing inward — but Clean Architecture is more **explicitly layered into concentric circles**, with a stricter, named rule: **the Dependency Rule** — source code dependencies can only point **inward**, toward higher-level policy; nothing in an inner circle can know anything about an outer circle.

```
 ┌─────────────────────────────────────┐
 │  Frameworks & Drivers (web, DB, UI)  │  outermost — most volatile, most detail
 │  ┌─────────────────────────────┐    │
 │  │  Interface Adapters          │    │  (controllers, presenters, gateways)
 │  │  ┌───────────────────┐      │    │
 │  │  │  Application/Use   │      │    │  (application-specific business rules)
 │  │  │  Case Layer         │      │    │
 │  │  │  ┌─────────────┐   │      │    │
 │  │  │  │  Entities    │   │      │    │  innermost — enterprise business rules,
 │  │  │  │  (Domain)    │   │      │    │  most stable, zero outward dependencies
 │  │  │  └─────────────┘   │      │    │
 │  │  └───────────────────┘      │    │
 │  └─────────────────────────────┘    │
 └─────────────────────────────────────┘
   Dependencies point INWARD only, always
```

**Senior-level answer:**
> "I treat Hexagonal and Clean Architecture as the same family of idea with different emphasis — Hexagonal is usually explained with two sides (inside the hexagon vs. outside, via ports/adapters), while Clean Architecture spells out **more explicit intermediate layers** — Entities, Use Cases, Interface Adapters — and gives the inward-dependency rule a formal name. In practice, when I say 'Clean Architecture' in a real codebase, I'm almost always also doing Hexagonal's ports-and-adapters mechanically; the distinction in interviews is more about vocabulary precision than a fundamentally different implementation."

---

### Q7. What's a concrete example of the Dependency Rule being violated, and why is it a problem even if the code "works fine"?
**Answer:** A common violation: an **Entity or Use Case class importing a JPA annotation** (`@Entity`, `@Column`) or a Spring-specific type, directly coupling the innermost, most stable layer to an outer-layer framework detail.

```java
// VIOLATION — domain entity now depends on JPA (an outer-layer framework detail)
@Entity
@Table(name = "orders")
class Order {
    @Id private String id;
    @Column private String status;
    // business logic mixed with persistence annotations
}

// CORRECT — pure domain entity, no framework dependency at all
class Order {
    private String id;
    private OrderStatus status;
    public void cancel() { /* pure business rule, testable with zero infrastructure */ }
}
// a separate persistence-layer class (in the outer ring) maps Order <-> a JPA entity
```

**Senior-level answer:**
> "It 'works fine' in the sense that the application runs, but the cost shows up later: the domain entity can no longer be unit tested without a JPA/Hibernate context on the classpath, and — more importantly — if the team ever needs to change persistence technology, or reuse the same domain logic in a different context (a batch job, a CLI tool, a different API), that JPA coupling has to be untangled from business rules it has nothing to do with. The Dependency Rule isn't about theoretical purity; it's specifically about keeping the *most expensive-to-change* part of the system (core business rules) free of dependencies on the *most likely-to-change* parts (frameworks, databases, UI)."

---

## SECTION 5: ONION ARCHITECTURE

### Q8. How does Onion Architecture differ from Hexagonal/Clean Architecture, if the core idea (inward dependencies) is the same?
**Answer:** Onion Architecture (Jeffrey Palermo) predates and heavily influenced both Hexagonal and Clean, sharing the same inward-dependency principle, with a specific emphasis on **the domain model and domain services sitting at the absolute center**, with **no dependency on the persistence/infrastructure layer at all** — even repository *interfaces* are defined in the domain layer, and infrastructure implements them, structurally identical to Hexagonal's ports.

**The practical distinction most interviewers actually care about:** these three patterns (Hexagonal, Clean, Onion) are **functionally near-identical in practice** — the same dependency-inversion principle, expressed with slightly different diagrams and layer names. What matters far more than memorizing the differences between them is being able to explain the **shared principle** clearly and apply it.

| | Hexagonal | Clean | Onion |
|---|---|---|---|
| **Core metaphor** | Ports & Adapters | Concentric circles, Dependency Rule | Concentric layers, domain at center |
| **Primary emphasis** | Symmetry between driving/driven adapters | Explicit use-case layer, named Dependency Rule | Domain services distinct from domain model |
| **Practical difference from the others** | Minimal — mostly vocabulary and diagram style | Minimal | Minimal |

**Senior-level answer:**
> "Honestly, in an interview I'd say directly that these three are best understood as **variations on one idea** rather than three distinct architectures to memorize separately — the interviewer is almost always testing whether you understand *why* inward dependencies and isolating the domain matter, not whether you can recite which one calls the outer ring 'Frameworks & Drivers' versus 'Infrastructure.' I'd rather spend the answer demonstrating the shared principle with a concrete example than trying to draw fine lines between three diagrams that, in real production codebases, end up looking nearly identical."

---

## SECTION 6: MICROSERVICES ARCHITECTURE

### Q9. When is Microservices Architecture actually the right call, versus when is it premature or unnecessary complexity?
**Answer:** Microservices earn their cost when:
- **Independent scaling is genuinely needed** — different parts of the system have wildly different load profiles (a search service needs 50 instances, a reporting service needs 2).
- **Independent deployability matters** — multiple teams need to ship changes to their own part of the system without coordinating a shared release.
- **Team structure already reflects service boundaries** (Conway's Law) — teams organized around business capabilities, each able to own a service end-to-end.
- **Different technology needs per component** — one service genuinely benefits from a different language/database than another.

**When it's premature:**
- A small team (or single team) building a new product with unclear domain boundaries — splitting into services *before* understanding where the real seams are tends to lock in the *wrong* boundaries, which are expensive to fix across a network boundary.
- No existing operational maturity — no CI/CD automation, no monitoring/observability investment, no experience with distributed systems failure modes — microservices multiply the operational surface area significantly.

**Senior-level answer:**
> "My honest default recommendation for a new product with an unclear domain and a small team is a **Modular Monolith** (Q3), not microservices — you get most of the boundary discipline without the distributed-systems tax, and you can extract true microservices later once the domain boundaries have proven themselves stable under real usage. I've seen more damage done by premature microservices — teams drowning in network calls, distributed transaction complexity, and cross-service debugging for a product that could have been one well-organized deployable — than I've seen from monoliths that grew too large. The decision should follow demonstrated need, not be a default architectural starting point."

---

### Q10. What's the single biggest architectural risk specific to Microservices that doesn't exist in a monolith, and how do you mitigate it?
**Answer:** **The distributed monolith** — a system split into separate services that are still **tightly coupled at runtime or data level**, so you've inherited all the operational cost of microservices (network calls, deployment coordination, distributed debugging) with **none of the actual independence benefit**. Symptoms: services that must be deployed together in a specific order, synchronous call chains where service A always calls B which always calls C for every request, or a shared database multiple services read/write to directly.

**Mitigation:**
- **Database per service**, strictly enforced (see Distributed Data Management module).
- **Asynchronous, event-driven communication** where possible instead of deep synchronous call chains (Q11–Q12).
- **Contract testing** (e.g., Pact) to catch breaking API changes between services without requiring full end-to-end integration testing for every change.
- **Service boundaries drawn around business capabilities**, not technical layers — a boundary drawn wrong is the root cause of most distributed-monolith situations.

**Senior-level answer:**
> "The distributed monolith is what I actually worry about most when reviewing a microservices migration plan, because it's the failure mode that gives you the worst of both worlds and is genuinely painful to unwind after the fact — you can't just 'merge the databases back together' once several teams have built independent deployment pipelines around a bad boundary. The root cause is almost always boundaries drawn along technical lines (a 'user service,' a 'notification service' as generic technical buckets) instead of genuine business capability boundaries with minimal cross-service chattiness — this is exactly why I push teams toward Domain-Driven Design bounded-context analysis *before* drawing service lines, not after."

---

## SECTION 7: EVENT-DRIVEN ARCHITECTURE

### Q11. What is Event-Driven Architecture, and what does it fundamentally change about how services are coupled to each other?
**Answer:** In Event-Driven Architecture (EDA), services communicate by **publishing and subscribing to events** through a broker (Kafka, RabbitMQ) rather than calling each other directly — a producer publishes "something happened" without knowing or caring which consumers, if any, react to it. This changes coupling from **synchronous, direct, and temporal** (the caller and callee must both be available *at the same moment*) to **asynchronous and decoupled** (the producer doesn't need the consumer to be available at all, and doesn't even need to know the consumer exists).

```
Synchronous (tight coupling):
  Order Service --REST call--> Inventory Service --REST call--> Notification Service
  (all three must be up and responsive, in sequence, for the request to succeed)

Event-Driven (loose coupling):
  Order Service --publishes--> "OrderPlaced" event --> [Inventory Service subscribes]
                                                    --> [Notification Service subscribes]
                                                    --> [Analytics Service subscribes]
  (Order Service doesn't know or care who's listening, or how many)
```

**Senior-level answer:**
> "The coupling shift is the real story here, not just 'it's async.' In a synchronous chain, adding a new consumer of an order-placed event means modifying the Order Service to add another outbound call — the producer has to change every time a new consumer cares about its events. In EDA, adding a new consumer (say, a new fraud-detection service) means that service simply subscribes to the existing `OrderPlaced` topic — **zero changes required in the Order Service**. That's a materially different scalability property for a system that needs to keep growing new capabilities without constantly modifying existing, stable services."

---

### Q12. What are the real trade-offs of Event-Driven Architecture that a team needs to consciously accept, not just the benefits?
**Answer:**
- **Eventual consistency, not immediate** — a consumer reacting to an event does so at some (usually short) delay after the fact, not instantaneously; this must be an acceptable trade for the specific use case.
- **Harder to trace/debug a business flow** — a request's effects are now scattered across multiple asynchronous consumers instead of one linear call stack; requires investment in distributed tracing/correlation IDs to reconstruct "what happened" for a given event.
- **At-least-once delivery realities** — consumers must be idempotent (see Distributed Data Management module's Inbox pattern), since events can be redelivered.
- **Schema evolution complexity** — changing an event's shape affects every consumer, potentially including ones the producing team doesn't know about, since the whole point of EDA is producers not knowing their consumers.
- **Operational complexity of the broker itself** — Kafka/RabbitMQ cluster management, partition strategy, consumer group rebalancing, retention policy — real ongoing operational investment.

**Senior-level answer:**
> "I always present EDA's trade-offs as seriously as its benefits, because 'event-driven' has become something of a buzzword that gets reached for reflexively. The schema evolution point is the one that bites teams hardest in practice — because producers deliberately don't know their consumers, a producing team can't just grep the codebase to find every place that'll break from a field rename; you need a real schema registry and a versioning discipline (backward-compatible changes only, or explicit versioned event types) from day one, or you'll break consumers you didn't even know existed."

---

## SECTION 8: SERVERLESS ARCHITECTURE

### Q13. What is Serverless Architecture, and what workloads is it genuinely well-suited for versus poorly suited for?
**Answer:** Serverless (e.g., AWS Lambda, Azure Functions) means the cloud provider **fully manages the underlying compute infrastructure** — no servers to provision, patch, or scale manually; you deploy individual functions that run **on-demand**, triggered by events (an HTTP request, a queue message, a file upload), and you're billed **per invocation/execution time**, not for idle capacity.

**Well-suited for:**
- **Spiky, unpredictable, or infrequent workloads** — a function that runs occasionally doesn't cost anything when idle, unlike a traditional server that runs (and costs money) 24/7 regardless of traffic.
- **Event-driven glue logic** — processing a file uploaded to S3, reacting to a queue message, a simple webhook handler.
- **Rapid scaling to handle traffic spikes** — the platform scales invocations automatically without any capacity planning.

**Poorly suited for:**
- **Long-running processes** — most serverless platforms impose execution time limits (e.g., 15 minutes on AWS Lambda), ruling out long batch jobs or persistent connections.
- **Consistently high, predictable, sustained traffic** — at high enough sustained volume, per-invocation serverless pricing often becomes **more expensive** than a traditional always-on server/container.
- **Workloads sensitive to cold-start latency** — a function that hasn't run recently incurs a startup delay (JVM-based runtimes especially) before serving the first request, which can be a real problem for latency-sensitive synchronous APIs.

**Senior-level answer:**
> "The framing I use: serverless is fundamentally a **cost and operational trade for spiky/infrequent workloads**, not a universally superior architecture. I've seen teams migrate a consistently busy, high-throughput API to Lambda expecting cost savings and instead see both a *higher* bill than an equivalent always-on container setup, and new latency problems from cold starts under a JVM-based runtime specifically — Java's JVM startup overhead makes it one of the worse-fit languages for latency-sensitive serverless functions compared to something like Go or a lightweight Node.js runtime. The decision has to be workload-shape-driven, not adopted because 'serverless' sounds modern."

---

### Q14. How does Serverless change how you think about state and connections, compared to a traditional long-running application?
**Answer:** Serverless functions are **stateless and ephemeral by design** — a function instance can be torn down after handling a request (or a handful of requests), so anything relying on **in-memory state persisting across invocations** (a local cache, an in-memory session store, a long-lived connection pool) either doesn't work reliably or requires deliberate workarounds.

**Practical implications:**
- **Database connection pooling becomes a real problem** — a traditional app maintains one connection pool for its lifetime; a serverless function that scales to hundreds of concurrent invocations can each try to open their own DB connections, potentially exhausting the database's connection limit. Mitigated with connection-pooling proxies (e.g., RDS Proxy) designed specifically for this pattern.
- **Any needed state must live externally** — session state in Redis/DynamoDB, not in function memory; any "warm" optimization (reusing a connection across invocations of the *same* warm instance) is an opportunistic bonus, never something to depend on for correctness.
- **Idempotency matters even more** — a function can be invoked more than once for the same trigger event under certain failure/retry conditions, same as the Idempotent Consumer pattern for message queues.

**Senior-level answer:**
> "The connection pool exhaustion problem is the one I'd bring up unprompted, because it's a very real, very common production surprise for teams new to serverless — a Lambda function connecting directly to a relational database works fine in testing at low concurrency, then falls over in production the moment traffic scales to hundreds of concurrent invocations, each opening its own connection against a database that might only support a few hundred total connections. This is exactly why AWS built RDS Proxy — it's a direct, purpose-built response to a problem serverless's execution model creates that traditional long-running applications never had to think about."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can you combine architecture patterns — e.g., Hexagonal within a Microservice?" → Yes, and it's common — each individual microservice can internally follow Hexagonal/Clean Architecture principles for its own domain isolation, while the overall system-level architecture is Microservices/Event-Driven; the patterns operate at different scopes (in-service vs. system-wide) and aren't mutually exclusive.
- "Is a Modular Monolith just a stepping stone to microservices, or a legitimate long-term architecture?" → It's legitimate long-term for many systems — not every application needs to end up as microservices; some well-organized modular monoliths never need to be split, and forcing an eventual microservices migration "because that's the natural next step" is itself a form of premature optimization.
- "What's the 'Big Ball of Mud' anti-pattern, and how does it relate to these patterns?" → An architecture (or lack thereof) with no clear structure, where every component can depend on every other component arbitrarily — it's the failure mode every pattern in this module is explicitly designed to prevent through enforced boundaries and dependency direction rules.
- "Does Event-Driven Architecture require microservices, or can a monolith be event-driven internally?" → No requirement — a monolith can use an internal event bus (e.g., Spring's `ApplicationEventPublisher`) for decoupling modules within a single process, which is a legitimate pattern in a Modular Monolith, separate from using an external broker like Kafka across service boundaries.
- "What's the difference between choreography and orchestration in Event-Driven systems?" → Choreography has each service reacting to events independently with no central coordinator; orchestration has a central process explicitly directing each step — this is the same distinction covered in the Saga pattern (Distributed Data Management module).
- "Would you recommend Clean Architecture for a small CRUD application?" → Generally no — the layering and indirection Clean Architecture introduces adds real overhead that isn't justified for a simple CRUD app with minimal business logic; it earns its cost in systems with substantial, evolving business rules worth protecting from infrastructure churn, not a thin API over a database.

---

*Study tip: Being able to clearly articulate the shared "inward dependency" principle across Hexagonal, Clean, and Onion architecture (Q5–Q8) without getting lost in their diagram differences, explaining the distributed monolith failure mode as the central risk of premature microservices adoption (Q9–Q10), and knowing serverless's genuine trade-offs — connection pooling and cold starts — rather than treating it as a universal win (Q13–Q14) are the three areas where interviewers most reliably separate candidates who've designed real systems from candidates repeating pattern names from a blog post.*
