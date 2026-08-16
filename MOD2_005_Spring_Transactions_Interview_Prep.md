# Spring Transactions — Interview Prep
---

## SECTION 1: @TRANSACTIONAL, TRANSACTION BOUNDARIES & PROXY MECHANICS

### Q1. What does @Transactional actually do, and how does Spring implement it mechanically?
**Answer:** `@Transactional` declaratively marks a method (or class) as needing to run inside a database transaction — begin before entry, commit on normal return, roll back on a qualifying exception — without hand-writing `begin`/`commit`/`rollback` calls. Under the hood, it's implemented as an **AOP concern**: Spring wraps the bean in a proxy, and a `TransactionInterceptor` (`@Around`-style advice) surrounds the actual method call.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        inventoryService.reserve(request.getItems());   // participates in same transaction
        orderRepository.save(new Order(request));         // participates in same transaction
    }                                                       // commit here, or rollback on exception
}
```

**What the proxy does, conceptually:**
```
1. Caller invokes proxy.placeOrder()
2. TransactionInterceptor: begin transaction, bind connection to current thread
3. Actual placeOrder() body executes
4. No exception → commit; matching exception → rollback
5. Unbind connection, return control to caller
```

**Senior-level answer:**
> "The single most important consequence of the proxy-based implementation is: **the transaction boundary is the outermost `@Transactional` method entry point on the proxy**, not the method body itself. Everything called *within* that method — other service methods, repository calls — participates in the same transaction by default (`REQUIRED` propagation), as long as those calls go **through** the proxy. That last clause is what causes the self-invocation bug (Q10) and is the detail I always verify first when a transaction isn't behaving as expected."

---

### Q2. Where should @Transactional be placed — service layer, repository layer, or controller layer? Why?
**Answer:** **Service layer**, almost always — this is the layer that expresses a business *use case*, which is the natural unit of atomicity.

| Layer | Should it hold @Transactional? | Why |
|---|---|---|
| Controller | No | Mixes HTTP concerns with transaction boundaries; also risks holding a connection open during slow I/O like external API calls made in the same method |
| **Service** | **Yes — primary placement** | Encapsulates a business operation; naturally the atomic unit ("place an order," "transfer funds") |
| Repository | Rarely directly | Individual repository methods are usually too fine-grained to be the *whole* transaction — a service method often calls several |

**Senior-level answer:**
> "I treat the service layer as the transaction boundary almost as a hard rule, and the reasoning is really about *scope of atomicity*: a single business operation — like `placeOrder()` — often spans multiple repository calls (reserve inventory, charge payment, save the order), and those need to succeed or fail **together**. Putting `@Transactional` on individual repository methods would make each database call its own atomic unit, which defeats the purpose entirely — a failure in step 3 wouldn't roll back steps 1 and 2. Controllers, meanwhile, are the wrong place because I don't want a slow external HTTP call or view-rendering logic holding a database connection open for the duration."

---

## SECTION 2: ACID, PROPAGATION & ISOLATION LEVELS

### Q3. What does ACID mean, and how does each property map to something Spring/the database actually enforces?
**Answer:**

| Property | Meaning | Enforced by |
|---|---|---|
| **Atomicity** | All operations in a transaction succeed, or none do | Commit/rollback mechanics — Spring's `@Transactional` rollback rules |
| **Consistency** | A transaction moves the DB from one valid state to another (constraints, triggers not violated) | Database constraints (FK, unique, check) + application logic |
| **Isolation** | Concurrent transactions don't interfere with each other's intermediate state | Isolation level setting (Q4) |
| **Durability** | Once committed, changes survive a crash | Database's write-ahead log / storage engine — not a Spring concern |

**Senior-level answer:**
> "Atomicity and Isolation are the two properties I actually make explicit decisions about day-to-day as an application developer — Consistency is largely a schema/constraint design concern, and Durability is entirely the database engine's job, nothing Spring touches. When I'm reviewing a `@Transactional` method, I'm really asking two questions: 'what's the correct rollback behavior here' (atomicity, tied to rollback rules — Q6) and 'what concurrency anomalies can this specific operation tolerate' (isolation — Q4)."

---

### Q4. What are the four standard isolation levels, what anomaly does each prevent, and what's Spring's default?
**Answer:** Spring's default is `DEFAULT` — meaning **defer to the underlying database's own default** (which is `READ_COMMITTED` for PostgreSQL and Oracle, but `REPEATABLE_READ` for MySQL/InnoDB — a detail worth knowing explicitly since it differs by vendor).

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Notes |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible | Weakest; rarely used in practice |
| `READ_COMMITTED` | Prevented | Possible | Possible | Common default (Postgres, Oracle); good balance |
| `REPEATABLE_READ` | Prevented | Prevented | Possible* | MySQL/InnoDB default; *InnoDB actually also prevents most phantom reads via gap locking |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | Strongest, effectively serial execution; highest contention/lowest throughput |

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public BigDecimal getAccountBalance(Long accountId) {
    return accountRepository.findBalance(accountId);
}
```

**Senior-level answer:**
> "In practice I rarely override the isolation level explicitly — the database's default is usually the right engineering tradeoff, and jumping straight to `SERIALIZABLE` because 'it sounds safest' is a common overcorrection that tanks throughput under load via lock contention and serialization failures you then have to retry around. The one place I *have* explicitly raised isolation is a financial balance-check-then-debit sequence prone to non-repeatable reads under concurrent load — and even there, I first considered whether **optimistic locking** (a `@Version` column) or a **pessimistic `SELECT ... FOR UPDATE`** solved it more cheaply than raising the isolation level for the whole transaction."

---

### Q5. What is transaction propagation, and walk through the difference between REQUIRED, REQUIRES_NEW, and NESTED with a concrete scenario.
**Answer:** Propagation determines what happens when a `@Transactional` method is called **from within an already-active transaction** — does it join the existing one, suspend it and start fresh, or something else?

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join the existing transaction if one exists; otherwise start a new one |
| `REQUIRES_NEW` | Always suspend any existing transaction and start a brand-new, independent one |
| `NESTED` | Start a savepoint within the existing transaction (rollback of the nested part only, if the DB/driver supports savepoints — JDBC only, not JPA in general) |
| `SUPPORTS` | Join if a transaction exists; otherwise run non-transactionally |
| `MANDATORY` | Must run within an existing transaction; throws if none exists |
| `NEVER` | Must run **without** a transaction; throws if one exists |
| `NOT_SUPPORTED` | Suspends any existing transaction and runs non-transactionally |

**Concrete scenario — audit logging that must persist even if the main operation rolls back:**
```java
@Service
public class OrderService {

    @Autowired private AuditService auditService;

    @Transactional
    public void placeOrder(OrderRequest request) {
        orderRepository.save(new Order(request));
        auditService.logAttempt(request);          // must survive even if placeOrder rolls back later
        paymentService.charge(request);              // if this throws, order save rolls back...
    }                                                   // ...but the audit log must NOT roll back
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAttempt(OrderRequest request) {
        auditRepository.save(new AuditEntry(request));  // commits independently, immediately
    }
}
```

**Senior-level answer:**
> "`REQUIRES_NEW` is the propagation type I reach for most deliberately, and always for the same reason: I need a piece of work to commit **independently** of the outer transaction's outcome — audit trails, notification records, or idempotency-key writes that must persist as evidence even if the main business operation fails and rolls back. The cost is real, though: it opens a second physical database connection/transaction, so overusing it can exhaust a connection pool under load or, worse, create subtle deadlocks if the outer and inner transactions touch overlapping rows. `NESTED` is the one I use least — savepoint support is inconsistent across JPA providers, and I've found `REQUIRES_NEW` in a separate method is usually more predictable and easier to reason about than debugging savepoint rollback semantics."

---

## SECTION 3: ROLLBACK RULES; CHECKED VS UNCHECKED EXCEPTIONS

### Q6. What's Spring's default rollback behavior, and why does it treat checked and unchecked exceptions differently?
**Answer:** By default, Spring's declarative transaction management rolls back **only on unchecked exceptions** — `RuntimeException` and its subclasses, plus `Error` — and **commits** despite a checked exception being thrown, unless explicitly told otherwise.

```java
@Transactional
public void placeOrder(OrderRequest request) throws InsufficientStockException {
    orderRepository.save(new Order(request));
    if (!inventoryService.hasStock(request)) {
        throw new InsufficientStockException("Out of stock");  // checked exception
    }
    // ⚠️ Without rollbackFor, the order save above still COMMITS despite this throw!
}
```

**Why this default exists:** It mirrors an EJB-era convention treating checked exceptions as **expected, recoverable business outcomes** (e.g., "insufficient funds" as part of a normal business flow the caller is expected to handle) versus unchecked exceptions as **unexpected failures** that should trigger a safety-net rollback.

**Senior-level answer:**
> "This default is, in my experience, the single most common source of silent data-integrity bugs in Spring codebases — a checked exception gets introduced later by a teammate (often from a library call, like `IOException`), gets declared on a `@Transactional` method, and the transaction quietly commits partial work instead of rolling back, with no error at all to signal it. My team's standing rule: **never rely on the default** — every `@Transactional` method either only throws unchecked exceptions (my strong preference, since checked exceptions in service-layer code are broadly an anti-pattern I try to avoid regardless of transactions), or explicitly declares `rollbackFor`/`noRollbackFor` so the intent is unambiguous and visible in code review, rather than implicit and dependent on someone remembering this specific default."

---

### Q7. How do you customize rollback rules with rollbackFor and noRollbackFor, and when would you use noRollbackFor?
**Answer:**
```java
@Transactional(rollbackFor = Exception.class)   // rolls back on ANY exception, checked or unchecked
public void placeOrder(OrderRequest request) throws InsufficientStockException {
    ...
}

@Transactional(noRollbackFor = ItemBackorderedException.class)   // commits despite this specific exception
public void placeOrder(OrderRequest request) {
    orderRepository.save(new Order(request));
    if (!inventoryService.hasStock(request)) {
        throw new ItemBackorderedException();  // expected business outcome — order is still saved as "pending"
    }
}
```

**Senior-level answer:**
> "`noRollbackFor` is the less commonly needed of the two, but it's genuinely useful for exactly one pattern: an exception that represents an **expected business state**, not a failure, where the transaction's work up to that point is still meant to be saved — like flagging an order as 'backordered' rather than aborting the whole placement. I use it sparingly and always pair it with a code comment explaining why, since seeing `noRollbackFor` in review without context reads exactly like a bug. My default posture is still `rollbackFor = Exception.class` on essentially every `@Transactional` method, purely to eliminate the checked-exception trap from Q6 as a category of bug."

---

### Q8. Does a caught exception trigger a rollback? Walk through why or why not.
**Answer:** No — if an exception is **caught and handled inside the `@Transactional` method** (i.e., it never propagates back out to the proxy), Spring's `TransactionInterceptor` never sees it, so no rollback is triggered, regardless of exception type.

```java
@Transactional
public void placeOrder(OrderRequest request) {
    orderRepository.save(new Order(request));
    try {
        paymentService.charge(request);
    } catch (PaymentException ex) {
        log.error("Payment failed", ex);
        // swallowed — the transaction proxy never sees this exception → COMMITS the order save above
    }
}
```

**Senior-level answer:**
> "This trips people up because the mental model of 'unchecked exceptions cause rollback' is only half the story — it's really 'unchecked exceptions that **propagate out of the proxied method** cause rollback.' If you genuinely need to catch and handle an exception but still force a rollback, the correct tool is `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` inside the catch block — this explicitly marks the transaction for rollback without rethrowing, which is useful when you want to log/handle the error locally but still guarantee the DB changes don't commit."

```java
} catch (PaymentException ex) {
    log.error("Payment failed", ex);
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
}
```

---

## SECTION 4: SELF-INVOCATION PROBLEM; READ-ONLY TRANSACTIONS

### Q9. Explain the self-invocation problem in detail, and give at least two ways to fix it.
**Answer:** Because `@Transactional` is proxy-based AOP, calling a `@Transactional` method **from another method in the same class** (`this.method()`) is a direct JVM call on the raw object — it never goes through the proxy, so no transaction is started (or, if already inside a transaction, propagation rules like `REQUIRES_NEW` are silently ignored).

```java
@Service
public class OrderService {

    public void placeOrder(OrderRequest request) {
        saveOrder(request);   // ⚠️ direct call on 'this' — bypasses the proxy entirely
    }

    @Transactional
    public void saveOrder(OrderRequest request) {
        orderRepository.save(new Order(request));   // NOT actually running in a transaction!
    }
}
```

**Fix 1 — inject a self-reference bean:**
```java
@Service
public class OrderService {

    @Autowired
    @Lazy
    private OrderService self;   // proxy reference to itself

    public void placeOrder(OrderRequest request) {
        self.saveOrder(request);   // goes through the proxy — transaction applies correctly
    }

    @Transactional
    public void saveOrder(OrderRequest request) { ... }
}
```

**Fix 2 — split into a separate bean (usually the cleaner fix):**
```java
@Service
public class OrderService {
    @Autowired private OrderPersistenceService persistenceService;

    public void placeOrder(OrderRequest request) {
        persistenceService.saveOrder(request);   // different bean → goes through its own proxy
    }
}

@Service
public class OrderPersistenceService {
    @Transactional
    public void saveOrder(OrderRequest request) { ... }
}
```

**Senior-level answer:**
> "I default to the second fix — extracting the `@Transactional` method into its own collaborator bean — over the self-injection trick, because self-injection reads as unusual/surprising to anyone unfamiliar with the pattern and can trip up static analysis or testing setups. Splitting into two beans is more code but is unambiguous and self-documenting: it makes the transaction boundary a real architectural seam rather than a workaround. The other tool worth knowing is `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)`, but I treat that as a last resort — it couples the class directly to AOP proxy internals, which is exactly the kind of implementation detail I'd rather keep out of business logic."

---

### Q10. What does @Transactional(readOnly = true) actually do, and why bother using it if the database enforces read-only at the SQL level anyway?
**Answer:** `readOnly = true` is a **hint**, not an enforced constraint at the Spring level — it doesn't prevent write statements from executing (some drivers/databases will reject writes, but this isn't guaranteed or portable). Its real value is enabling optimizations further down the stack:

- **JPA/Hibernate:** can skip dirty-checking on loaded entities (no need to track changes for a flush that will never happen), reducing memory/CPU overhead — especially valuable for large result sets.
- **JDBC driver / connection pool:** some drivers can route read-only transactions to a **read replica** in a primary/replica database setup.
- **Database engine:** some databases (e.g., certain PostgreSQL configurations) can take a lighter-weight locking/snapshot strategy for read-only transactions.

```java
@Transactional(readOnly = true)
public List<OrderSummary> getOrderHistory(Long customerId) {
    return orderRepository.findSummariesByCustomerId(customerId);   // no dirty-checking overhead
}
```

**Senior-level answer:**
> "I mark essentially every pure-query service method `readOnly = true` as a matter of habit — the Hibernate dirty-checking savings alone are worth it on any method returning a non-trivial result set, and it also serves as a form of **executable documentation**: it tells the next engineer, unambiguously, 'this method must never write.' The detail I make sure junior engineers don't miss: it's a hint, not a guarantee — if you actually need to *prevent* accidental writes, that has to be enforced through code review discipline or a genuinely read-only DB user/connection, not just this flag. I've also used it as the trigger for routing to a read-replica connection pool in a primary/replica setup via a custom `AbstractRoutingDataSource`, keyed off `TransactionSynchronizationManager.isCurrentTransactionReadOnly()`."

---

## SECTION 5: TRANSACTION MANAGEMENT BEST PRACTICES

### Q11. What are the most common @Transactional mistakes you watch for in code review?
**Answer:**

| Mistake | Why it's a problem |
|---|---|
| `@Transactional` on a `private` method | Silently ignored — proxies can't intercept private methods (no override possible) |
| Relying on default rollback for checked exceptions | Silently commits partial work — see Q6 |
| Self-invocation of a `@Transactional` method | Bypasses the proxy entirely — see Q9 |
| Long-running I/O (external HTTP calls, file uploads) inside a transactional method | Holds a DB connection open far longer than necessary, starving the connection pool under load |
| `@Transactional` on a class with mixed read/write methods, all called generically without `readOnly` tuning | Missed optimization opportunity at scale — see Q10 |
| Catching an exception and not rethrowing or explicitly calling `setRollbackOnly()` | Silent partial commit — see Q8 |

**Senior-level answer:**
> "The one I flag most often in review is external I/O inside a transactional method — a call to a payment gateway, an email service, or an external API sitting between two database writes inside the same `@Transactional` boundary. That's holding a database connection open for the entire duration of a network call that could take seconds, which is a direct path to connection-pool exhaustion under any real load. My usual fix is to restructure: do the external call **outside** the transaction (often via an outbox pattern — persist an event/intent transactionally, then process the external call asynchronously and update status afterward), rather than trying to keep everything inside one atomic unit just because it's conceptually one business operation."

---

### Q12. How do you unit-test transactional behavior — e.g., verifying a method actually rolls back on failure?
**Answer:**
```java
@SpringBootTest
class OrderServiceTransactionTest {

    @Autowired private OrderService orderService;
    @Autowired private OrderRepository orderRepository;

    @Test
    void placeOrder_rollsBackOnPaymentFailure() {
        assertThrows(PaymentException.class, () ->
                orderService.placeOrder(failingPaymentRequest()));

        // verify nothing was actually committed to the DB
        assertThat(orderRepository.findAll()).isEmpty();
    }
}
```

**Senior-level answer:**
> "I deliberately avoid `@Transactional` on the *test* method itself for this kind of test — Spring Test's `@Transactional` wraps each test in a transaction that auto-rolls-back at the end regardless of what the code under test does, which would mask a real bug where the production code *fails* to roll back correctly. I want the test to genuinely commit (or not) exactly as production would, then assert on the actual persisted state afterward — which means either a plain `@SpringBootTest` without test-level `@Transactional`, or explicitly using `TestTransaction.flagForCommit()` if I do need the surrounding scaffolding. I've caught real `rollbackFor` misconfigurations this way that a transactional test would have silently hidden."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does @Transactional work on a method called via a static call or a constructor?" → No — proxy-based, so it only intercepts calls made through the Spring-managed proxy on a bean reference, not static methods or object construction.
- "What isolation level does READ_COMMITTED still allow?" → Non-repeatable reads and phantom reads — it only prevents dirty reads (seeing another transaction's uncommitted changes).
- "Can @Transactional be placed at the class level?" → Yes — it applies as the default for every public method in the class, individually overridable per-method.
- "What happens if you nest REQUIRES_NEW inside REQUIRES_NEW?" → The outer transaction is suspended, a second fully independent transaction runs and commits/rolls back on its own, then the outer resumes — they don't affect each other's outcome.
- "Is @Transactional required on a method that only does a single repository.save() call?" → Not strictly, since Spring Data JPA repository methods are individually transactional by default (`SimpleJpaRepository` is itself annotated `@Transactional`) — but wrapping the calling service method explicitly is still good practice once more than one operation is involved.
- "What's the difference between PlatformTransactionManager and @Transactional?" → `PlatformTransactionManager` (or its modern `TransactionManager` interface) is the underlying SPI that actually begins/commits/rolls back transactions against a specific resource (JDBC, JPA, JMS); `@Transactional` is the declarative annotation that AOP-wraps a method to delegate to whichever `TransactionManager` bean is configured.

---

*Study tip: The default-rollback-only-on-unchecked-exceptions trap (Q6) and the self-invocation proxy bypass (Q9) are, by a wide margin, the two most frequently asked "gotcha" questions on Spring Transactions — be ready to explain not just *what* happens but *why* the proxy mechanics cause it, and have the fix (extract to a collaborator bean) ready without hesitation.*
