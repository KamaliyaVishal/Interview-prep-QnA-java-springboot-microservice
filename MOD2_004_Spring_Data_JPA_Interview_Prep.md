# Spring Data JPA — Interview Prep
---

## SECTION 1: JPA vs Hibernate vs Spring Data JPA

### Q1. JPA vs Hibernate vs Spring Data JPA — how do these three actually relate to each other?
**Answer:**
```
JPA (Jakarta Persistence API)
 └── a SPECIFICATION — interfaces/annotations only, no implementation (EntityManager, @Entity, JPQL, etc.)
      └── Hibernate
           └── the most popular IMPLEMENTATION of the JPA spec — the actual ORM engine doing the work
                └── Spring Data JPA
                     └── a Spring abstraction layer ON TOP of a JPA provider (usually Hibernate),
                         eliminating boilerplate DAO/repository code
```
- **JPA** — a Java **specification** (like `Serializable` or the Servlet API) defining a standard set of annotations and interfaces for ORM: `@Entity`, `@Id`, `EntityManager`, JPQL. It has **no runtime behavior of its own** — it's a contract.
- **Hibernate** — the dominant **implementation** of that contract. When you use JPA annotations, at runtime it's Hibernate (or another provider like EclipseLink) actually generating SQL, managing the persistence context, caching, dirty checking.
- **Spring Data JPA** — sits **on top of** JPA/Hibernate, providing `JpaRepository` and friends so you don't hand-write boilerplate DAO classes (`EntityManager.persist()`, `find()`, manual JPQL for basic CRUD) — it generates the implementation for you at runtime via dynamic proxies.

**Senior-level answer:**
> "This layering matters because it explains what's actually swappable and what isn't. I can switch Hibernate for EclipseLink without touching my `@Entity` classes, because they only depend on the JPA spec. I could also swap Spring Data JPA out entirely and write plain `EntityManager` code against the same entities and the same underlying Hibernate — Spring Data JPA is a **convenience layer**, not a different persistence technology. Understanding this layering is what lets you debug 'is this a Spring Data problem or a Hibernate problem' correctly — most subtle bugs (N+1, dirty checking surprises, lazy-loading exceptions) are actually **Hibernate/JPA-level** behavior, not Spring Data JPA's."

---

### Q2. Can you use Hibernate-specific features that go beyond the JPA spec? Give an example, and what's the trade-off?
**Answer:** Yes — Hibernate has proprietary extensions beyond plain JPA: `@DynamicUpdate`, `@Where`, `@Formula`, Hibernate's native `Criteria`/`Session` API, `@NaturalId`, second-level cache-specific annotations. Example:
```java
@Entity
@DynamicUpdate   // Hibernate-specific: generates UPDATE statements with ONLY changed columns, not all columns
public class Order { ... }
```
**Trade-off (senior talking point):** Using Hibernate-specific features **breaks the "swap the JPA provider freely" theoretical portability** — in practice, though, almost nobody actually swaps JPA providers in a real production codebase once Hibernate is chosen, so I don't treat portability as a hard constraint. I use Hibernate-specific features pragmatically when they solve a real problem (like `@DynamicUpdate` reducing unnecessary column writes on wide tables), while defaulting to plain JPA annotations for the common 90% case.

---

## SECTION 2: ENTITY MAPPING

### Q3. Walk through a basic @Entity mapping — @Entity, @Table, @Id, @Column.
```java
@Entity
@Table(name = "orders", indexes = @Index(name = "idx_customer_id", columnList = "customer_id"))
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false, unique = true, length = 20)
    private String orderNumber;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Enumerated(EnumType.STRING)   // stores enum NAME, not ordinal — critical (Q4)
    private OrderStatus status;

    @Column(precision = 19, scale = 2)
    private BigDecimal totalAmount;

    // getters/setters/constructors
}
```
- **`@Entity`** — marks the class as a JPA-managed persistent entity (must have a no-arg constructor, not be `final`, have an identifier field).
- **`@Table`** — maps to a specific DB table name/schema; **optional** — without it, Hibernate defaults to the class name.
- **`@Id`** — marks the primary key field.
- **`@Column`** — customizes column mapping (name, nullability, length, precision); **also optional** for basic mapping — without it, Hibernate infers the column name from the field name.

---

### Q4. Trap: @Enumerated(EnumType.ORDINAL) vs STRING — why is ORDINAL dangerous in production?
**Answer:** `EnumType.ORDINAL` stores the enum's **position index** (0, 1, 2...) in the database; `EnumType.STRING` stores the enum's **name** as text.

```java
public enum OrderStatus { PENDING, SHIPPED, DELIVERED, CANCELLED }
// ORDINAL: PENDING=0, SHIPPED=1, DELIVERED=2, CANCELLED=3
```

**Why ORDINAL is a genuine production landmine:** If a developer later **reorders, inserts, or removes** a value in the enum — e.g., adding a new `REFUNDED` status **between** `PENDING` and `SHIPPED` — every ordinal shifts, and **existing rows in the database now silently map to the wrong status** with zero compile-time or runtime warning. This is one of the most cited "I've seen this cause a real production incident" answers senior candidates give.

**Senior best practice:** Always use `@Enumerated(EnumType.STRING)` in production code — yes, it uses slightly more storage and is technically a bit slower to compare, but it's **human-readable in the database** (invaluable for debugging/manual queries) and **immune to reordering bugs**. The only time `ORDINAL` (the JPA default if `@Enumerated` is omitted entirely) is defensible is a very storage-sensitive legacy system — and even then it's rarely worth the risk.

---

### Q5. What's the difference between @GeneratedValue strategies — IDENTITY, SEQUENCE, TABLE, AUTO?
| Strategy | Mechanism | Trade-off |
|---|---|---|
| `IDENTITY` | DB auto-increment column (`SERIAL`/`AUTO_INCREMENT`) | Simple, but **disables JDBC batch inserts** in Hibernate — the ID isn't known until after the INSERT executes, so Hibernate can't batch multiple inserts together |
| `SEQUENCE` | DB sequence object, Hibernate pre-fetches IDs | **Supports batching** — Hibernate can grab a block of sequence values upfront and batch inserts efficiently; generally preferred for high-throughput inserts on Postgres/Oracle |
| `TABLE` | A separate table simulates a sequence | Portable across all DBs but **slowest** — adds extra table locking/contention; rarely used today |
| `AUTO` | Hibernate picks a strategy based on the DB dialect | Convenient default, but **implicit** — for senior/production code, I prefer being explicit about which strategy is actually in use |

**Senior-level answer:**
> "For a high-throughput service doing bulk inserts, I specifically choose `SEQUENCE` over `IDENTITY` because of the batching implication — it's a subtle performance difference that only shows up under load, and I've seen teams debug 'why are my batch inserts still issuing individual INSERT statements' and trace it back to `GenerationType.IDENTITY` disabling batching entirely, silently."

---

## SECTION 3: RELATIONSHIPS

### Q6. @OneToMany / @ManyToOne — how do you correctly map a bidirectional relationship, and what's the "owning side" concept?
```java
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}

@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")   // this side OWNS the relationship — holds the actual FK column
    private Customer customer;
}
```
**Answer:** In a bidirectional relationship, exactly **one side owns it** — the owning side has the actual foreign-key column (`@JoinColumn`) and is what Hibernate uses to determine what SQL to issue on save; the **inverse** side uses `mappedBy` to reference the owning side's field name and is purely for **object-graph navigation** — changes made only on the inverse side (e.g., adding to `customer.getOrders()` without also setting `order.setCustomer(customer)`) are **silently NOT persisted**, because Hibernate only looks at the owning side to decide what to write to the database.

**Senior trap to flag proactively:** This is one of the most common real bugs junior/mid developers hit — "I added an order to the customer's list but it's not showing up in the database" — because they only updated the inverse (`mappedBy`) side. **Best practice: always provide a convenience method that keeps both sides in sync:**
```java
public void addOrder(Order order) {
    orders.add(order);
    order.setCustomer(this);   // keep the owning side in sync — this is what actually gets persisted
}
```

---

### Q7. @ManyToMany — how do you map it, and why do many senior developers avoid it in favor of two @OneToMany relationships?
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}
```
**Answer:** `@ManyToMany` auto-manages a join table for you — convenient for the simplest case where the join table has **no extra columns of its own**.

**Senior-level answer (the real interview-differentiating point):**
> "In practice, I almost always avoid `@ManyToMany` in production schemas, because real-world many-to-many relationships almost always need **extra attributes on the join itself** eventually — an enrollment date, a status, an assigned-by user — and a plain `@ManyToMany` join table can't hold that. I model it explicitly instead: two `@OneToMany`/`@ManyToOne` relationships around an explicit join **entity** (e.g., `Enrollment` with its own `@Id`, `enrolledDate`, `grade` fields). This is more verbose upfront but avoids a painful, disruptive migration later when the 'simple' many-to-many inevitably needs extra data. It also makes cascade/orphan-removal behavior far more predictable and explicit than `@ManyToMany`'s implicit join-table management."

---

### Q8. @OneToOne — what's a common pitfall with fetch type here specifically?
**Answer:** `@OneToOne`'s **default fetch type is `EAGER`** (unlike `@OneToMany`/`@ManyToMany`, which default to `LAZY`) — this is a frequently-missed JPA spec detail. If left as `EAGER` on a relationship that isn't always needed, you silently pay for an extra join/query on **every single load** of the owning entity, even when the related entity is never accessed.

```java
@Entity
public class User {
    @OneToOne(fetch = FetchType.LAZY, mappedBy = "user")   // explicitly override the EAGER default
    private UserProfile profile;
}
```
**Additional trap worth mentioning:** Even with `fetch = LAZY` explicitly set, Hibernate historically couldn't truly lazy-load a `@OneToOne` on the **non-owning** side without either bytecode enhancement or restructuring the mapping, because there's no foreign key on that side to check for `null` cheaply — Hibernate needs to query the other table anyway to know if the association exists at all. Modern Hibernate versions with bytecode enhancement handle this better, but it's a nuance worth mentioning as a "gotcha I'm aware of" in a senior interview.

---

### Q9. What does CascadeType actually control, and what's the danger of CascadeType.ALL used carelessly?
| CascadeType | Propagates |
|---|---|
| `PERSIST` | Saving the parent also saves new/unsaved children |
| `MERGE` | Merging the parent also merges children |
| `REMOVE` | **Deleting the parent also deletes children** |
| `REFRESH` | Refreshing parent state also refreshes children from DB |
| `DETACH` | Detaching the parent also detaches children from the persistence context |
| `ALL` | All of the above |

**Senior-level danger to flag proactively:**
> "`CascadeType.ALL` — specifically the `REMOVE` cascade bundled inside it — is genuinely dangerous on relationships where the child entity has meaning **independent** of the parent. A classic real mistake: cascading `REMOVE` from `Order` to `Customer` (or similar) would delete a customer's entire order history just because one order got deleted through an unrelated code path, or worse, cascading in the wrong direction entirely and deleting a shared/referenced entity. I only use `CascadeType.ALL` (or `orphanRemoval = true`, which is even more aggressive — deletes children removed from the collection even without an explicit delete call) on relationships that are genuinely a true **composition** — the child truly has no independent existence without the parent, like `Order` → `OrderLineItem`. For anything else, I cascade only the specific operations I actually intend (usually just `PERSIST`/`MERGE`), never a blanket `ALL`."

---

## SECTION 4: REPOSITORY HIERARCHY

### Q10. JpaRepository vs CrudRepository vs PagingAndSortingRepository — how do they relate?
```
Repository (marker interface, no methods)
 └── CrudRepository<T, ID>              — basic CRUD: save, findById, findAll, delete, count
      └── PagingAndSortingRepository<T, ID>   — adds findAll(Pageable), findAll(Sort)
           └── JpaRepository<T, ID>          — adds JPA-specific: flush(), saveAndFlush(), 
                                                deleteInBatch(), getReferenceById(), batch operations
```
**Answer:** Each extends the previous, adding capability. `JpaRepository` is what nearly everyone actually uses in Spring Data JPA projects, since it's a strict superset providing everything below it plus JPA-specific conveniences.

**Senior-level answer (when would you use a narrower interface?):**
> "I default to `JpaRepository` for the vast majority of repositories. I'd only deliberately narrow to `CrudRepository` in a rare scenario where I want to **signal via the type system** that a particular repository shouldn't support pagination/JPA-specific batch operations — e.g., a repository over a very small, fixed reference-data table where pagination genuinely makes no sense and I want that constraint visible in the interface itself. In practice this is uncommon; it's more of a 'know the hierarchy exists' interview point than something I apply constantly."

---

### Q11. What actually happens when you define an interface extending JpaRepository — where does the implementation come from?
**Answer:** You never write an implementation class — Spring Data JPA generates one **dynamically at runtime** using a JDK/CGLIB **proxy**, backed by `SimpleJpaRepository` (the default base implementation providing all the standard CRUD methods via `EntityManager` calls) plus a **query-derivation engine** that parses your custom method names/`@Query` annotations and generates the corresponding JPQL/SQL on the fly.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatusAndCustomerId(OrderStatus status, Long customerId);   // no implementation written — ever
}
```
**Senior detail:** This is registered as a Spring bean via `@EnableJpaRepositories` (auto-configured by Spring Boot when `spring-boot-starter-data-jpa` is on the classpath) — component scanning finds repository **interfaces**, and Spring Data's proxy factory creates the actual bean instance backing them.

---

## SECTION 5: QUERY METHODS

### Q12. What are derived query methods? Show a few examples and explain the naming convention.
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByStatus(OrderStatus status);
    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);
    List<Order> findByTotalAmountGreaterThan(BigDecimal amount);
    List<Order> findByCreatedDateBetween(Instant start, Instant end);
    Optional<Order> findFirstByCustomerIdOrderByCreatedDateDesc(Long customerId);
    boolean existsByOrderNumber(String orderNumber);
    long countByStatus(OrderStatus status);
    void deleteByStatus(OrderStatus status);
}
```
**Answer:** Spring Data JPA **parses the method name itself** — `findBy`, `And`/`Or`, comparison keywords (`GreaterThan`, `Between`, `Like`, `In`, `IsNull`), and property names matching entity fields — and generates the corresponding JPQL query automatically, with **zero implementation code**.

**Senior-level trade-off/opinion:**
> "Derived query methods are great for simple, single-condition or two-condition lookups — they're self-documenting and require no query maintenance. But I set a personal/team limit around **2-3 conditions**; beyond that, method names become genuinely unreadable (`findByStatusAndCustomerIdAndCreatedDateBetweenAndTotalAmountGreaterThan(...)`), and I switch to an explicit `@Query` instead, which is far more maintainable and debuggable even though it requires writing the query by hand."

---

### Q13. @Query with JPQL vs native SQL — differences, and when do you reach for native?
```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.totalAmount > :minAmount")
    List<Order> findHighValueOrders(@Param("status") OrderStatus status, @Param("minAmount") BigDecimal minAmount);

    @Query(value = "SELECT * FROM orders WHERE status = :status ORDER BY total_amount DESC LIMIT :limit",
           nativeQuery = true)
    List<Order> findTopOrdersNative(@Param("status") String status, @Param("limit") int limit);
}
```
| Aspect | JPQL (`@Query`) | Native SQL (`@Query(nativeQuery = true)`) |
|---|---|---|
| Operates on | **Entity/field names**, not table/column names — DB-agnostic | Actual **table/column names** — DB-specific SQL |
| Portability | Portable across DB vendors (Hibernate translates to the target dialect) | Tied to the specific database's SQL dialect |
| Access to DB-specific features | No — limited to JPQL's feature set | **Yes** — window functions, CTEs, DB-specific functions, query hints |
| Return type mapping | Automatic entity mapping | Requires explicit mapping if not returning a full entity shape (via `@SqlResultSetMapping` or DTO projections) |

**Senior-level answer:**
> "I default to JPQL for anything expressible in it — it stays portable and integrates cleanly with entity mapping. I reach for native SQL specifically when I need a DB-specific feature JPQL can't express — window functions, a recursive CTE, a vendor-specific full-text search function, or when I need to hand-tune a query for a proven performance problem that the ORM's generated SQL wasn't handling well. Native queries are a deliberate escape hatch, not a default."

---

### Q14. How do you write an @Query for an UPDATE or DELETE (bulk operations)?
```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Modifying
    @Transactional
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus")
    int bulkUpdateStatus(@Param("oldStatus") OrderStatus oldStatus, @Param("newStatus") OrderStatus newStatus);
}
```
**Answer:** `@Modifying` is **required** to tell Spring Data JPA the query isn't a `SELECT` — without it, Spring Data JPA throws an exception trying to treat an `UPDATE`/`DELETE` as a result-returning query. `@Transactional` is required because modifying queries must execute within a transaction. The method returns `int` — the number of affected rows.

**Senior trap to flag proactively:** Bulk `@Modifying` updates execute **directly against the database**, bypassing the persistence context entirely — meaning any already-loaded entities in the current `EntityManager`'s first-level cache become **stale**, silently out of sync with the database, unless you explicitly `clearAutomatically = true` (or manually `entityManager.clear()`) afterward:
```java
@Modifying(clearAutomatically = true)   // clears the persistence context after the bulk update
@Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus")
int bulkUpdateStatus(...);
```

---

### Q15. How do you avoid the classic "unnecessary SELECT * with all columns" problem for read-heavy endpoints — what are projections?
```java
public interface OrderSummary {
    Long getId();
    String getOrderNumber();
    BigDecimal getTotalAmount();
}

public interface OrderRepository extends JpaRepository<Order, Long> {
    List<OrderSummary> findByCustomerId(Long customerId);   // interface-based projection — only fetches needed columns
}

// Or a class-based (DTO) projection:
public record OrderSummaryDto(Long id, String orderNumber, BigDecimal totalAmount) { }

@Query("SELECT new com.example.OrderSummaryDto(o.id, o.orderNumber, o.totalAmount) FROM Order o WHERE o.customerId = :customerId")
List<OrderSummaryDto> findSummariesByCustomerId(@Param("customerId") Long customerId);
```
**Answer:** Projections let a query method return a **subset of an entity's fields** (interface-based, Spring Data JPA proxies it at runtime) or a dedicated DTO (class-based, via a JPQL constructor expression) instead of always fetching and hydrating the **entire entity**. This directly reduces the SQL column list and avoids the memory/serialization overhead of loading full entity graphs when a read endpoint only actually needs 2-3 fields — a genuinely impactful, easy performance win for read-heavy, high-traffic endpoints.

---

## SECTION 6: PAGINATION & SORTING

### Q16. How does Pageable/Page work in Spring Data JPA? Show a controller-to-repository flow.
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}

@GetMapping("/orders")
public Page<Order> getOrders(
        @RequestParam OrderStatus status,
        @PageableDefault(size = 20, sort = "createdDate", direction = Sort.Direction.DESC) Pageable pageable) {
    return orderRepository.findByStatus(status, pageable);
}
```
**Answer:** `Pageable` (typically built automatically by Spring Data Web support from `?page=0&size=20&sort=createdDate,desc` query params) carries pagination + sort instructions into the repository method; `Page<T>` in the response wraps the result content **plus** metadata (`totalElements`, `totalPages`, `hasNext`, current page number) — everything a client needs to render pagination controls, without a separate count query written by hand.

**Senior detail on what happens internally:** Spring Data JPA issues **two queries** for a `Page<T>` return type — the actual data query (`LIMIT`/`OFFSET`) and a **separate `COUNT` query** to populate `totalElements`. This is a real, non-trivial performance cost on large tables — worth knowing for Q17.

---

### Q17. Page vs Slice — what's the difference, and when would you choose Slice for a performance reason?
| Type | Behavior |
|---|---|
| `Page<T>` | Runs an extra `COUNT` query to know the **total** number of results/pages |
| `Slice<T>` | **No COUNT query** — only knows if there's a **next** slice (fetches `size + 1` records internally, and if the extra one exists, `hasNext()` returns true) |

**Senior-level answer:**
> "For a classic paginated UI with page numbers ('page 3 of 47'), you need the total count, so `Page<T>` is correct despite its extra query cost. But for **infinite-scroll** or 'load more' style UIs — which is an increasingly common pattern — you never actually need the total count, only 'is there more.' In that case I use `Slice<T>` specifically to **avoid the extra COUNT query** entirely, which can be a meaningful performance win on large tables where `COUNT(*)` itself is expensive (especially with complex WHERE clauses or on tables without a good covering index for the count)."

---

## SECTION 7: LAZY vs EAGER LOADING — THE N+1 PROBLEM

### Q18. Lazy vs Eager fetch — what's the default for each relationship type, and why do those defaults exist?
| Relationship | Default fetch type |
|---|---|
| `@ManyToOne` | `EAGER` |
| `@OneToOne` | `EAGER` (Q8) |
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |

**Answer:** The rationale: "to-one" relationships (`@ManyToOne`, `@OneToOne`) default to eager because they're typically a **single extra join** — relatively cheap. "To-many" relationships default to lazy because eagerly loading a **collection** could mean pulling in an unbounded, potentially huge number of rows just to load one parent entity — far riskier to default to eager.

**Senior best practice (a strong, commonly-expected answer):**
> "In practice, I override virtually everything to `LAZY` explicitly, including `@ManyToOne`/`@OneToOne` — I don't rely on JPA's defaults at all. I want fetching to be a **deliberate, visible decision** made per-query (via `JOIN FETCH` or entity graphs, Q20), not an implicit side effect of the entity mapping. Relying on `EAGER` defaults is exactly how N+1 problems and unexpectedly large query graphs sneak into a codebase silently."

---

### Q19. What is the N+1 select problem? Walk through a concrete example.
```java
List<Order> orders = orderRepository.findAll();      // 1 query — fetches N orders
for (Order order : orders) {
    System.out.println(order.getCustomer().getName());   // N additional queries — one per order, if customer is LAZY
                                                            // and this is the FIRST access triggering a separate SELECT
}
// Total: 1 + N queries, instead of a single JOIN
```
**Answer:** The N+1 problem occurs when fetching a list of **N** parent entities triggers **one initial query** for the parents, then **N additional individual queries** — one per parent — to lazily load an associated entity/collection as it's accessed in a loop, instead of a single efficient `JOIN`. This is one of the most commonly asked, most commonly **mis-diagnosed-in-production** JPA problems — it often doesn't show up in local testing with a handful of rows, but becomes a severe, very real performance cliff at production scale (1000 orders = 1001 queries instead of 1-2).

---

### Q20. How do you actually fix/avoid the N+1 problem? Name multiple techniques.
**Answer (structure this as a list — reads well live, and shows breadth):**
1. **`JOIN FETCH` in JPQL** — explicitly force eager loading for that specific query only:
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findByStatusWithCustomer(@Param("status") OrderStatus status);
```
2. **`@EntityGraph`** — declaratively specify which associations to eagerly fetch, without writing custom JPQL:
```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus status);
```
3. **Batch fetching** (`@BatchSize` on the entity/collection, or `hibernate.default_batch_fetch_size` globally) — instead of N individual queries, Hibernate issues queries in **batches** (e.g., `WHERE customer_id IN (?, ?, ?, ... up to batch size)`), turning N+1 into roughly `N/batchSize + 1` queries — a good pragmatic middle ground when `JOIN FETCH` isn't easily applicable everywhere.
4. **DTO projections** (Q15) — if you don't actually need the full entity graph, project only the fields you need in a single query, sidestepping the lazy-association issue entirely.
5. **Enable SQL logging in development** (`spring.jpa.show-sql=true` + a query-counting tool like `p6spy` or Hibernate's statistics) to **catch N+1 issues before production**, since they're notoriously invisible with small local datasets.

**Senior talking point:** "N+1 is rarely caught by functional tests since they pass either way — it's caught by **query-count assertions** in tests, or by production APM tooling flagging unusually high query counts per request. I've made query-count checks part of integration test suites specifically to catch regressions here."

---

### Q21. Trap: Does adding a JOIN FETCH for a collection (@OneToMany) risk anything, like duplicate rows or pagination bugs?
**Answer: Yes — this is a genuinely tricky, senior-differentiating detail.** `JOIN FETCH`-ing a **collection** association produces a Cartesian-product-style result set — one row **per child row**, meaning the parent entity appears duplicated in the raw SQL result (Hibernate de-duplicates them in the returned Java `List` via its identity map, but the **JDBC-level row count** is inflated). Two real consequences:
1. **In-memory pagination trap:** If you combine `JOIN FETCH` on a `@OneToMany` with `Pageable`, Hibernate historically couldn't apply `LIMIT`/`OFFSET` at the SQL level safely (it would paginate the wrong thing — the joined/duplicated rows, not the distinct parent entities) — Hibernate would fall back to **fetching the entire result set into memory and paginating there**, silently destroying the performance benefit of pagination and potentially loading way more data than intended. (Modern Hibernate versions have improved this in some cases, but it remains a well-known trap worth naming.)
2. Fix: fetch the **collection separately** via a second query/`@EntityGraph` scoped appropriately, or use **`@BatchSize`** instead of `JOIN FETCH` when pagination is also required on the same query.

---

## SECTION 8: ENTITY LIFECYCLE & CALLBACKS

### Q22. What are the JPA entity states (lifecycle), and how do they transition?
```
NEW/TRANSIENT  →  (persist())  →  MANAGED  →  (remove())  →  REMOVED
                                     ↕ (detach() / close EntityManager)
                                  DETACHED  →  (merge())  →  MANAGED
```
- **Transient/New** — a plain Java object, not associated with any persistence context, no DB row.
- **Managed** — associated with an active `EntityManager`'s persistence context; changes are automatically tracked (**dirty checking**) and flushed to the DB.
- **Detached** — was managed, but the persistence context closed/cleared, or `entityManager.detach()` was called explicitly — changes are **no longer tracked**.
- **Removed** — scheduled for deletion; still in the persistence context until the transaction commits/flushes.

**Senior detail (dirty checking, frequently asked as a follow-up):** For a **managed** entity, you don't need to explicitly call `save()`/`update()` to persist a field change — simply mutating a field on a managed entity within an active transaction is enough; Hibernate detects the change (**dirty checking**, comparing current state against a snapshot taken at load time) and issues an `UPDATE` automatically at flush/commit time. This surprises developers coming from a plain-JDBC/manual-SQL background.

---

### Q23. What are JPA entity lifecycle callbacks (@PrePersist, @PostLoad, etc.)? Give a real use case.
```java
@Entity
public class Order {
    @Column(updatable = false)
    private Instant createdDate;
    private Instant updatedDate;

    @PrePersist
    void onCreate() {
        createdDate = Instant.now();
        updatedDate = createdDate;
    }

    @PreUpdate
    void onUpdate() {
        updatedDate = Instant.now();
    }
}
```
**Answer:** JPA callback annotations hook into specific lifecycle transitions: `@PrePersist`/`@PostPersist` (before/after insert), `@PreUpdate`/`@PostUpdate`, `@PreRemove`/`@PostRemove`, `@PostLoad` (after an entity is loaded from the DB). Common real use cases: auto-populating timestamps (shown above — though Spring Data's `@CreatedDate`/`@LastModifiedDate` auditing, Q24, is usually preferred over hand-rolling this), computing/normalizing a derived field before save (e.g., uppercasing a code field), or triggering validation logic that must run specifically at the persistence boundary.

**Senior caution worth mentioning:** Callback methods should stay **lightweight and side-effect-free** with respect to the persistence context — avoid triggering additional DB queries or complex business logic inside them; that logic belongs in the service layer, not scattered across entity lifecycle hooks, to keep entities focused on data shape rather than behavior.

---

## SECTION 9: AUDITING

### Q24. How do you enable automatic @CreatedDate/@LastModifiedDate auditing in Spring Data JPA?
```java
@SpringBootApplication
@EnableJpaAuditing   // required to activate auditing infrastructure
public class MyApp { }

@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass   // fields inherited by entities, not itself a table
public abstract class Auditable {
    @CreatedDate
    @Column(updatable = false)
    private Instant createdDate;

    @LastModifiedDate
    private Instant lastModifiedDate;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}

@Entity
public class Order extends Auditable { ... }
```
**Answer:** `@EnableJpaAuditing` activates the auditing infrastructure; `@EntityListeners(AuditingEntityListener.class)` on the entity (or a shared `@MappedSuperclass` base class, the cleaner pattern for applying it consistently across many entities) hooks into the `@PrePersist`/`@PreUpdate` lifecycle internally to auto-populate the annotated fields — no manual timestamp-setting code needed anywhere in the service layer.

---

### Q25. How does @CreatedBy/@LastModifiedBy know which user to populate? What do you need to configure?
```java
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName);   // pulls the current authenticated username
}
```
**Answer:** You must provide a bean implementing `AuditorAware<T>`, returning the "current user" for the auditing framework to populate `@CreatedBy`/`@LastModifiedBy`. In a typical Spring Security-secured application, this is implemented by reading the current `Authentication` from `SecurityContextHolder` — this ties auditing directly into whatever authentication mechanism is already in place, without any custom plumbing at each entity save call site.

**Senior production value to mention:** "Having reliable `createdBy`/`lastModifiedBy` fields populated automatically, consistently, across every entity has been genuinely valuable for production incident investigation and compliance/audit requirements — especially at fintech-style companies where 'who changed this record and when' is a real, frequently-asked question, sometimes from regulators."

---

## SECTION 10: TRANSACTIONS — @Transactional

### Q26. What does @Transactional actually do under the hood?
**Answer:** `@Transactional` is implemented via **Spring AOP proxying** (same underlying mechanism as any other Spring aspect) — when a bean method annotated `@Transactional` is called from **outside** the bean, the proxy intercepts the call, starts a transaction (via the configured `PlatformTransactionManager`, e.g., `JpaTransactionManager`), executes the actual method, and **commits** on normal return or **rolls back** on a matching exception (by default, only unchecked exceptions — `RuntimeException`/`Error` — trigger rollback; checked exceptions do **not**, unless explicitly configured via `rollbackFor`).

```java
@Transactional
public void placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    inventoryService.reserveStock(order);   // if this throws a RuntimeException, the order save above ROLLS BACK too
}
```

---

### Q27. Trap: Why doesn't @Transactional work when a method calls another @Transactional method on the SAME class (self-invocation)?
**Answer:** This directly parallels the AOP self-invocation trap already covered for `@Transactional`/general AOP in the Spring Core prep — the proxy wraps the bean from **outside**; calling `this.someTransactionalMethod()` from within the same class **bypasses the proxy entirely**, invoking the raw, un-intercepted method directly. No new transaction is started, and if the "outer" method also isn't transactional, **there's no transaction at all** for that call.

```java
@Service
public class OrderService {
    public void processOrder(OrderRequest request) {
        saveOrder(request);   // BUG: calls the RAW method directly — @Transactional on saveOrder is SILENTLY IGNORED
    }

    @Transactional
    public void saveOrder(OrderRequest request) { ... }
}
```
**Fix:** Restructure so the transactional method is called from a **different** bean (so the call goes through the proxy), or, less cleanly, self-inject a proxy reference to the bean itself. This is a very frequently asked trap question specifically because it's such a common, genuinely confusing real bug.

---

### Q28. What are the common @Transactional propagation types, and give a real scenario for REQUIRES_NEW?
| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Joins an existing transaction if one is active; creates a new one if not |
| `REQUIRES_NEW` | **Always** suspends any existing transaction and starts a brand-new, independent one |
| `NESTED` | Starts a nested transaction (savepoint) within the existing one — can roll back independently without rolling back the outer transaction |
| `SUPPORTS` | Joins an existing transaction if present; runs non-transactionally if not |
| `MANDATORY` | Requires an existing transaction; throws an exception if none is active |
| `NEVER` | Throws an exception if called within an active transaction |

**Real `REQUIRES_NEW` scenario (strong senior answer):**
> "A common real use case: logging an audit trail entry or a failure record that **must persist even if the main business transaction rolls back**. If `placeOrder()` fails and rolls back, I still want a record that the attempt was made and why it failed — if that audit-logging call shared the same transaction, it would roll back along with everything else, and I'd lose the failure record entirely. Marking the audit-write method `REQUIRES_NEW` ensures it commits **independently**, regardless of what happens to the outer transaction."

```java
@Transactional
public void placeOrder(OrderRequest request) {
    try {
        // main business logic
    } catch (Exception e) {
        auditService.recordFailure(request, e);   // REQUIRES_NEW — persists even if this whole transaction rolls back
        throw e;
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordFailure(OrderRequest request, Exception e) { ... }
```

---

### Q29. Should you mark a read-only method @Transactional(readOnly = true)? What does it actually do?
**Answer:** Yes, on any method that only reads data — it's a genuinely valuable, low-effort optimization hint:
1. **Hibernate skips dirty-checking** for entities loaded in that session, since it knows nothing will be modified — a real performance win, especially for large object graphs.
2. Some JPA providers/connection pools can route to a **read replica** based on this flag (with the right infrastructure setup).
3. It's **self-documenting** — signals clear intent to future readers of the code that the method has no write side effects.

```java
@Transactional(readOnly = true)
public List<Order> getOrderHistory(Long customerId) {
    return orderRepository.findByCustomerId(customerId);
}
```
**Common mistake to flag:** Forgetting this on read-heavy service methods is a low-cost missed optimization that adds up meaningfully across a high-traffic read path — a good, easy thing to proactively mention as a code-review habit.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does @Transactional work on a private method?" → No — Spring AOP proxying requires the method to be intercepted from outside the class, which needs at minimum a `protected`/`public` method (and even then, self-invocation still bypasses it per Q27).
- "What's the default rollback behavior for checked exceptions?" → No rollback, by default — only unchecked exceptions trigger rollback unless `rollbackFor` is explicitly specified.
- "Does calling save() on a JpaRepository always issue an immediate INSERT/UPDATE?" → Not necessarily — Hibernate may batch/defer the actual SQL until flush time (end of transaction, or an explicit `flush()`/query execution that forces a flush), which is why `saveAndFlush()` exists for when you need the SQL to run immediately.
- "What is the first-level cache?" → The persistence context itself — every managed entity within a single `EntityManager`/transaction is cached in memory; repeated `find()` calls for the same ID within that scope hit the cache, not the DB.
- "Is the first-level cache shared across requests?" → No — it's scoped to a single `EntityManager` (effectively, a single transaction/request in typical Spring usage), unlike the optional second-level cache, which is shared across the whole `EntityManagerFactory`/application.
- "Can you use Optional<T> as a repository return type?" → Yes — `Optional<Order> findById(Long id)` is standard and preferred over returning `null` directly.
- "What happens if you call getReferenceById() (formerly getOne()) on a non-existent ID?" → No exception immediately — it returns a lazy proxy without hitting the DB; the exception (`EntityNotFoundException`) only fires when a field is actually accessed, which is a subtle trap if not understood.

---

*Study tip: The bidirectional relationship "owning side" trap (Q6), the N+1 problem and its fixes (Q19–Q21), and the @Transactional self-invocation trap (Q27) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY these behaviors exist at the Hibernate/JPA level, not just WHAT the annotation does.*