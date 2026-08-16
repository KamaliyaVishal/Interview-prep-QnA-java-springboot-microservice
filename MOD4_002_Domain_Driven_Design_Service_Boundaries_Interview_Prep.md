# Domain-Driven Design & Service Boundaries — Interview Prep
---

## SECTION 1: DOMAIN, SUBDOMAIN & BUSINESS CAPABILITY

### Q1. What do "Domain" and "Subdomain" mean in DDD, and why should a Java developer designing microservices care?
**Answer:** The **Domain** is the overall business problem space your software exists to solve — for an e-commerce platform, the domain is "running an online retail business." A **Subdomain** is a natural partition of that domain into smaller, more focused problem areas — Order Management, Inventory, Payments, Shipping, Customer Support, Recommendations.

DDD classifies subdomains into three types, and this classification directly drives **where you invest engineering effort**:

| Subdomain type | Definition | Example (e-commerce) | Investment |
|---|---|---|---|
| **Core** | The thing that differentiates the business competitively — where you win or lose | Pricing/promotions engine, recommendation algorithm | Build in-house, best engineers, highest care |
| **Supporting** | Necessary for the business, but not a competitive differentiator | Order management, inventory tracking | Build in-house but pragmatically, less gold-plating |
| **Generic** | Solved problems every company has — no competitive value in reinventing them | Authentication, notifications/email, payment processing | Buy/use off-the-shelf (Auth0, Stripe, SendGrid) rather than build |

**Why this matters for microservices:** subdomains are the **business-driven** input that later map to service boundaries (Q9-10) — a Core subdomain often deserves its own dedicated, carefully-owned service and team, while several Generic subdomains might just be third-party integrations, not services you build at all.

**Business capability**, closely related, is simply "what the business *does*" expressed as a capability — "Process Orders," "Manage Inventory," "Authorize Payments" — independent of *how* it's currently implemented. Business capabilities are usually **more stable over time** than org charts or current team structure, which is exactly why they make a more durable basis for service boundaries than "whatever team currently exists."

---

## SECTION 2: BOUNDED CONTEXT & UBIQUITOUS LANGUAGE

### Q2. What is a Bounded Context, and how is it different from a subdomain?
**Answer:** A **Bounded Context** is the **explicit boundary within which a particular domain model — and its terminology — is valid and internally consistent**. Outside that boundary, the same word can mean something completely different, and that's fine and expected.

**Subdomain vs Bounded Context — the distinction interviewers actually probe for:** A subdomain is a **problem-space** concept (a piece of the business); a bounded context is a **solution-space** concept (a boundary in your software/model where a term has one consistent meaning). In an ideal design they align 1:1, but in real legacy systems they often don't — you might have one bounded context sloppily covering three subdomains, which is itself a design smell worth flagging.

**Classic concrete example — "Product":**
| Bounded Context | What "Product" means there |
|---|---|
| **Catalog context** | Name, description, images, category, SEO metadata |
| **Inventory context** | SKU, warehouse location, stock quantity, reorder threshold |
| **Pricing context** | Base price, discounts, tax rules, currency |
| **Shipping context** | Weight, dimensions, fragility flag, hazardous-material class |

Trying to build **one shared `Product` entity** with all these fields to satisfy every context is a well-known anti-pattern — it becomes a bloated god-object that every team fears touching, and it's exactly the kind of thing that produces a distributed monolith even *within* a single service. Each bounded context should have **its own model of Product**, shaped only by what that context actually needs.

---

### Q3. What is "Ubiquitous Language," and what's a real production example of what happens when it's missing?
**Answer:** Ubiquitous Language is the practice of using **one precise, shared vocabulary — agreed upon with domain experts — consistently across conversations, code, class names, method names, and even database tables**, *within a given bounded context*. The code should read like the business talks, not like a generic technical abstraction layered on top of business concepts translated by developers.

**In practice:**
```java
// Weak — generic, doesn't reflect actual domain language
public void updateStatus(Order order, String status) { ... }

// Strong — matches how the business actually talks about this operation
public void cancelOrder(Order order, CancellationReason reason) { ... }
public void shipOrder(Order order, TrackingNumber trackingNumber) { ... }
```

**Real-world consequence of skipping it:** "I've worked on a system where 'Customer' meant a *billing account* to the Finance team's service but meant an *individual end-user with a login* to the Auth service. Nobody had agreed on Ubiquitous Language up front, so a 'delete customer' request from a support agent — meaning 'delete this one user login' — got silently propagated to the billing service, which interpreted it as 'close this entire billing account,' cancelling active subscriptions for an entire company account. That's not a coding bug — it's a **missing bounded-context boundary and shared language** bug, and no amount of unit testing catches it because both services were individually 'correct' according to their own (undocumented, divergent) definition of Customer."

**Senior talking point:** "Ubiquitous Language isn't documentation fluff — it's a forcing function to surface these hidden ambiguities *before* they become integration bugs. I push for domain experts and engineers to literally use the same nouns and verbs in requirement discussions, class names, and API field names — any translation layer between 'what the business says' and 'what the code says' is where these bugs breed."

---

## SECTION 3: ENTITY, VALUE OBJECT, AGGREGATE & AGGREGATE ROOT

### Q4. Entity vs Value Object — what's the actual distinguishing rule, not just the definition?
**Answer:** The rule that actually matters: **does this object have a distinct identity that persists over time, independent of its attribute values?**

- **Entity** — has a unique, stable **identity** (usually an ID) that persists even as its attributes change. Two entities with identical attributes but different IDs are **different objects**. Equality is by **identity**, not attribute values.
- **Value Object** — has **no conceptual identity** — it's defined entirely by its attribute values, is typically **immutable**, and equality is by **value comparison**. Two Value Objects with the same attributes are **interchangeable/equal**.

```java
// Entity — identity matters, mutable state, equals() by ID
public class Order {
    private final OrderId id;         // identity
    private OrderStatus status;       // mutable
    private List<OrderLine> lines;

    @Override
    public boolean equals(Object o) {
        return o instanceof Order other && this.id.equals(other.id);
    }
}

// Value Object — no identity, immutable, equals() by value
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount.add(other.amount), this.currency);   // returns NEW instance — immutable
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof Money m && amount.equals(m.amount) && currency.equals(m.currency);
    }
    @Override
    public int hashCode() { return Objects.hash(amount, currency); }
}
```

**Interview trap:** "Is `Address` an Entity or a Value Object?" — The honest senior answer is **it depends on the bounded context**. In a shipping context, `Address` is usually a Value Object (two identical addresses are interchangeable — you don't care *which* address record it is, only where the package goes). In a context where you need to track "this specific address the customer edited on this date for audit purposes," it might need identity and become an Entity. This is exactly the kind of question where showing you understand it's context-dependent — not a fixed rule — signals seniority.

---

### Q5. What is an Aggregate, and what is an Aggregate Root? Why can't you just let any object be modified directly?
**Answer:** An **Aggregate** is a **cluster of related Entities and Value Objects treated as a single consistency boundary** — a transactional unit that must always be left in a valid state. The **Aggregate Root** is the **single Entity through which all external access to the aggregate must go** — nothing outside the aggregate is allowed to hold a reference to, or directly modify, an internal object of that aggregate.

```java
// Order is the Aggregate Root — the ONLY entry point into this aggregate
public class Order {
    private final OrderId id;
    private OrderStatus status;
    private final List<OrderLine> lines = new ArrayList<>();   // internal entities — NOT exposed for direct mutation

    public void addLine(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify a submitted order");   // invariant enforced HERE
        }
        lines.add(new OrderLine(product, quantity));
    }

    public void submit() {
        if (lines.isEmpty()) throw new IllegalStateException("Cannot submit an empty order");
        this.status = OrderStatus.SUBMITTED;
    }

    // Returns an unmodifiable view — callers cannot bypass addLine()'s invariant checks
    public List<OrderLine> getLines() {
        return Collections.unmodifiableList(lines);
    }
}
```

**Why the boundary exists:** `OrderLine` should never be modified directly from outside `Order` (e.g., no `orderLineRepository.save(orderLine)` from some unrelated service) — because that would let someone add a line to a `SUBMITTED` order and silently violate a business invariant that only `Order` itself is responsible for enforcing. The aggregate root is the **single point of invariant enforcement and transactional consistency**.

**Key rules to state proactively:**
- **One transaction = one aggregate.** Modifying two different aggregates in a single ACID transaction is a design smell in DDD — if Order and Inventory are separate aggregates (likely in separate services), consistency between them should be **eventual**, via domain events/Sagas, not a shared transaction.
- **Reference other aggregates by ID only**, never by object reference — e.g., `Order` holds a `CustomerId`, not a full `Customer` object — this keeps aggregates decoupled and independently loadable/persistable.
- **Keep aggregates small.** A large aggregate (e.g., one `Customer` aggregate holding every order they've ever placed) causes lock contention, poor performance, and unnecessarily wide consistency boundaries — a very common real-world mistake.

---

## SECTION 4: DOMAIN SERVICE & REPOSITORY

### Q6. When does behavior belong in a Domain Service instead of an Entity? Give a concrete example.
**Answer:** Behavior belongs on the **Entity/Aggregate itself** whenever it's naturally a responsibility of that single object (e.g., `order.submit()`, `money.add()`). It belongs in a **Domain Service** when the operation **doesn't naturally belong to any single entity** — typically because it involves **coordinating multiple aggregates**, or requires **external domain knowledge/policy** that doesn't fit cleanly inside one object.

```java
// Domain Service — coordinates two separate aggregates; doesn't naturally belong to either
@Service
public class FundsTransferService {

    public void transfer(Account from, Account to, Money amount) {
        if (from.getBalance().isLessThan(amount)) {
            throw new InsufficientFundsException(from.getId());
        }
        from.withdraw(amount);   // still delegates the actual mutation + invariant check to the aggregate itself
        to.deposit(amount);
    }
}
```

**Rule of thumb I actually apply:** "If I find myself writing `if (order != null && payment != null && ...)` logic that spans multiple aggregates inside a Controller or a generic 'Manager' class, that's usually a sign the coordination logic needs to be pulled into an explicit **Domain Service** — it keeps the coordination logic testable and named after what it actually represents in the business (`FundsTransferService`), rather than scattered across application-layer code as incidental glue."

**Important distinction to flag:** A Domain Service is **not** the same as an **Application Service** — the Domain Service (`FundsTransferService`) contains pure business/domain logic and has no knowledge of HTTP, transactions, or persistence framework details; the Application Service (typically the `@Service`-annotated class called by a `@RestController`) orchestrates transaction boundaries, security checks, and calls into the domain layer. Conflating the two is a common reason domain logic ends up leaking into controllers.

---

### Q7. What's the DDD-correct role of a Repository, and how is it different from just "a DAO"?
**Answer:** A Repository provides the **illusion of an in-memory collection of Aggregate Roots** — from the domain layer's perspective, you `save()` and `findById()` a whole aggregate as if it were sitting in a simple collection, with all the persistence mechanics (SQL, joins, ORM mapping) completely hidden behind that interface.

```java
// The domain layer only ever sees THIS interface — no JPA/Hibernate/SQL concepts leak through
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

// Infrastructure layer implements it — persistence detail, invisible to the domain
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final SpringDataOrderJpaRepository jpaRepo;
    private final OrderMapper mapper;   // maps between JPA entity and domain Aggregate

    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findById(id.value()).map(mapper::toDomain);
    }
    public void save(Order order) {
        jpaRepo.save(mapper.toJpaEntity(order));
    }
}
```

**Repository vs "plain DAO" — the distinction interviewers want to hear:**
| Repository (DDD) | DAO (typical CRUD) |
|---|---|
| Operates on **whole Aggregate Roots**, enforcing the aggregate consistency boundary | Often operates on individual tables/rows directly |
| Interface lives in the **domain layer**; implementation lives in infrastructure — dependency points *inward* (Dependency Inversion) | Usually just a data-access convenience with no domain-boundary concept |
| Hides ALL persistence details — the domain never sees `EntityManager`, SQL, or JPA annotations | Often leaks persistence-framework types (e.g., `@Entity` classes used directly as domain objects) |
| One repository **per Aggregate Root only** — never one per table | Commonly one DAO per table, regardless of aggregate boundaries |

**Common real mistake to flag proactively:** "A mistake I've seen repeatedly is exposing a `OrderLineRepository` alongside `OrderRepository` — since `OrderLine` isn't an Aggregate Root (it's internal to the `Order` aggregate, per Q5), giving it its own repository lets code elsewhere modify order lines directly, bypassing `Order`'s invariant checks entirely. In DDD, you should only ever have a Repository for an Aggregate Root."

---

## SECTION 5: IDENTIFYING SERVICE BOUNDARIES

### Q8. In practice, how do you go from "we have a big monolith/domain" to "here are our microservice boundaries"?
**Answer:** The practical, repeatable process I follow:

1. **Event Storming with domain experts** — a collaborative workshop mapping out all the significant **domain events** (`OrderPlaced`, `PaymentAuthorized`, `InventoryReserved`, `OrderShipped`) across the whole business process, using sticky notes/whiteboard, without worrying about technical implementation at all.
2. **Cluster events into Bounded Contexts** — group related events, commands, and the aggregates that produce them into cohesive contexts (Order Management, Payment, Inventory, Shipping) — this is where subdomains (Q1) and bounded contexts (Q2) get discovered organically rather than assumed up front.
3. **Draw a Context Map** — identify **how contexts relate** to each other (Q10) — which context is upstream/downstream of which, where translation is needed.
4. **Bounded Context → candidate microservice** — as a strong default heuristic, **one bounded context maps to one service**. This isn't an absolute law (a context can sometimes stay split across a couple of services for scaling reasons, or two very small/related contexts might start as one service), but it's the right starting default.
5. **Validate against team structure and change frequency (Conway's Law, applied deliberately)** — check whether one team can realistically own the proposed service end-to-end, and whether the boundary isolates the parts of the domain that change **for different reasons** at **different rates** (a strong signal you've found a real boundary vs. an arbitrary one).

---

### Q9. What are the concrete "smells" that tell you a proposed service boundary is wrong?
**Answer:**
- **Chatty synchronous communication** — if Service A must call Service B multiple times *within* a single business operation just to get basic data it needs constantly, the boundary likely cuts through a single cohesive concept that should've stayed together.
- **Frequent breaking API changes together** — if Order-service and Payment-service's APIs keep changing **in lockstep** release after release, they're probably not truly separate bounded contexts — this is the earliest warning sign of a distributed monolith (see MOD4_001 Q8-Q9).
- **Shared database tables** — if two "services" both need direct SQL access to the same table, the aggregate/bounded-context boundary was drawn in the wrong place.
- **Splitting along technical layers instead of business capability** — a "validation-service" or "database-service" that many other services must call for basic operations is a classic anti-pattern; boundaries should follow **what the business does**, not **CRUD/technical layering**.
- **God aggregate spanning multiple concerns** — e.g., one `Customer` aggregate carrying billing history, support tickets, AND login credentials signals the boundary should be split along the Q2 "Product" example pattern — each concern likely belongs to a different bounded context.

**Senior-level framing:** "I treat service boundaries as a hypothesis to validate, not a permanent decision made once in a whiteboard session. Event Storming gives a strong starting boundary, but the real confirmation comes from watching actual change patterns in production — services that keep needing to change together didn't have a real boundary between them, no matter how clean the original diagram looked."

---

### Q10. What's a "Context Map," and what are the common relationship patterns between Bounded Contexts?
**Answer:** A Context Map documents **how bounded contexts relate to and integrate with each other** — critical for deciding integration style (sync API, async events, shared library) and where translation logic needs to live.

| Pattern | Meaning | When you'd use it |
|---|---|---|
| **Shared Kernel** | Two contexts deliberately share a small, jointly-owned piece of the model (e.g., a common `Money` value object library) | Rare, high-trust — requires close coordination between teams since a change affects both |
| **Customer–Supplier** | Downstream context depends on upstream; upstream team considers downstream's needs in its own planning | Internal teams with a healthy working relationship (e.g., Inventory as supplier, Order as customer) |
| **Conformist** | Downstream simply accepts the upstream model as-is, no negotiation power | Consuming a third-party or another team's API you can't influence (e.g., a payment provider's API) |
| **Anti-Corruption Layer (ACL)** | Downstream builds a translation layer to convert the upstream's model into its own clean domain model, preventing upstream's concepts/quirks from leaking in | Integrating with a legacy system or messy external API — very commonly used and worth naming explicitly in interviews |
| **Open Host Service** | Upstream publishes a well-defined, stable public API/protocol specifically designed for multiple consumers | A shared platform service (e.g., a central Notification service used by many other services) |
| **Published Language** | A shared, well-documented data format/schema used for integration (e.g., a versioned Avro/JSON event schema) | Event-driven integration across many services (Kafka topics with a schema registry) |

**Real production example — Anti-Corruption Layer:** "Integrating with a legacy inventory mainframe that used cryptic field names and a totally different unit-of-measure convention, we built a thin ACL inside our Inventory bounded context that translated the mainframe's model into our own clean `StockLevel` domain concept. When the mainframe's format changed (which happened more than once), we only had to update the ACL — the rest of our clean domain model, and every consumer of it, was completely insulated from that churn. That containment is the entire point of drawing the boundary deliberately, rather than letting an external system's model bleed into your own."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can a Value Object ever be mutable?" → In principle it violates the pattern's intent; the standard, recommended implementation is always immutable — mutation should return a **new** instance (see `Money.add()` in Q4).
- "Can an Aggregate contain another Aggregate Root?" → No — reference other aggregates only by **ID**, never nest one Aggregate Root inside another; this keeps consistency boundaries and transactions clean.
- "Is Bounded Context the same thing as a microservice?" → Not by definition — it's a **modeling boundary**; in practice it's the strongest default candidate for a service boundary, but the mapping isn't mandatory 1:1 in every case (Q8).
- "Where do domain events get published — inside the Aggregate or the Application Service?" → The Aggregate **records** that a domain event occurred as part of its state change (e.g., `order.submit()` adds an `OrderSubmitted` event to an internal list); the Application Service/infrastructure layer is responsible for actually **publishing** it (e.g., to Kafka) after the transaction commits.
- "What's the difference between a Domain Service and a utility/helper class?" → A Domain Service is named after and expresses a real **business concept/operation** (`FundsTransferService`); a generic "Utils"/"Helper" class is a technical grab-bag with no ubiquitous-language meaning — a strong DDD codebase avoids the latter for business logic.
- "Does every microservice need its own bounded context?" → Ideally yes, one-to-one is the target; a service spanning multiple unrelated bounded contexts is itself a boundary-identification mistake worth flagging (Q9).

---

*Study tip: Aggregate/Aggregate Root invariant enforcement (Q5), Domain Service vs Application Service (Q6), and identifying service boundaries via Event Storming + Context Mapping (Q8-Q10) are where interviewers most reliably separate candidates who've read about DDD from those who've actually applied it to draw real microservice boundaries under production constraints.*
