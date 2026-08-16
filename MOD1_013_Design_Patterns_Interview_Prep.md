# 010. Design Patterns — Interview Questions & Answers 

**Focus:** Creational, Structural, Behavioral design patterns; what problem each solves, when to use/not use, Java/Spring examples, trade-offs, interview traps, and production scenarios.

---

# HOW TO APPROACH DESIGN PATTERNS IN AN INTERVIEW

A senior-level design-pattern answer should not start with memorized definitions. Use this structure:

```text
Problem
   ↓
Why the naive approach becomes difficult
   ↓
Pattern / design principle
   ↓
How the pattern solves the problem
   ↓
Small example
   ↓
Trade-offs
   ↓
When to use / when NOT to use
   ↓
Production example
```

### Important principle

> **A design pattern is not a rule that must be applied. It is a reusable design approach for a recurring problem.**

A good senior developer can explain both why a pattern helps, and why introducing it may be unnecessary.

---

# SECTION 1: CREATIONAL DESIGN PATTERNS

Creational patterns focus on **how objects are created**. They help when object creation is complex, conditional, expensive, coupled to concrete classes, or needs to be controlled/reused.

Patterns covered:

1. Singleton
2. Factory
3. Builder
4. Prototype

Also important in industry: Abstract Factory, Dependency Injection.

---

# 1. SINGLETON

## Concept

### What problem does Singleton solve?

Sometimes an application needs exactly one shared instance of a component and a controlled way to access it — configuration objects, application-wide registries, certain caches, or stateless shared services.

```text
Many callers
     |
     v
 Single shared instance
```

---

### Q1. What is the Singleton design pattern?

**Answer:** Singleton ensures a class has only one instance and provides a controlled access point to it. A basic implementation using the holder idiom:

```java
public final class Singleton {

    private Singleton() {
    }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

The initialization-on-demand holder idiom is lazy, thread-safe, and avoids explicit synchronization on every access — that's why it's my default choice when I actually need this pattern.

---

### Q2. Why would you use Singleton?

**Answer:** I'd use it when the application genuinely needs a single shared instance with a controlled lifecycle — configuration, a registry, or a shared stateless component. That said, in modern Spring applications manually implementing Singleton is usually unnecessary, because the Spring container already manages singleton-scoped beans by default.

---

### Q3. What are the different ways to implement Singleton in Java?

**Answer:** A few common approaches, each with trade-offs:

**Eager initialization** — simple and thread-safe, but the instance is created when the class loads, whether you need it yet or not:

```java
public final class Singleton {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

**Lazy synchronized method** — thread-safe, but pays synchronization cost on every call:

```java
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

**Double-checked locking** — avoids synchronizing on every call, but `volatile` is essential for correctness:

```java
private static volatile Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

**Holder idiom** — usually my clean default choice:

```java
private static class Holder {
    private static final Singleton INSTANCE = new Singleton();
}
```

**Enum Singleton** — gives strong serialization and reflection guarantees for free:

```java
public enum Singleton {
    INSTANCE;
}
```

---

### Q4. Why is volatile required in double-checked locking?

**Answer:** Object creation isn't a single atomic step. Without correct publication, another thread could observe a non-null reference before the object's construction is fully visible to it. `volatile` gives the memory-visibility and ordering guarantees needed so that once another thread sees the reference, the object's initialization is correctly visible too — that's exactly what prevents unsafe publication in double-checked locking.

---

### Q5. How can Singleton be broken?

**Answer:** Reflection, serialization, cloning, and multiple class loaders can all break a naive Singleton implementation. Enum Singleton is particularly robust against several of these. A common trap: a private constructor alone doesn't universally block instantiation through reflection.

---

### Q6. Is Singleton always a good design?

**Answer:** No. Singleton can introduce global mutable state, hidden dependencies, difficult unit testing, tight coupling, and lifecycle complexity. I avoid it as a default design choice — if an object is really just a dependency, dependency injection is usually clearer, since dependencies stay explicit and testable.

---

# 2. FACTORY

## Concept

### What problem does Factory solve?

Suppose client code directly creates many concrete implementations:

```java
Payment payment = new CreditCardPayment();
```

As implementations grow — `CreditCardPayment`, `UPIPayment`, `PayPalPayment`, `BankTransferPayment` — client code becomes coupled to concrete classes. Factory moves that creation logic behind an abstraction:

```text
Client
  |
  v
Factory
  |
  +---- CreditCardPayment
  +---- UPIPayment
  +---- BankTransferPayment
```

---

### Q7. What is the Factory pattern?

**Answer:** Factory encapsulates object creation so the client depends on an abstraction instead of knowing which concrete implementation to instantiate.

```java
interface Payment {
    void pay();
}

class UpiPayment implements Payment {
    public void pay() {
        System.out.println("UPI payment");
    }
}

class CardPayment implements Payment {
    public void pay() {
        System.out.println("Card payment");
    }
}

class PaymentFactory {

    static Payment create(String type) {
        return switch (type) {
            case "UPI" -> new UpiPayment();
            case "CARD" -> new CardPayment();
            default -> throw new IllegalArgumentException("Unknown type");
        };
    }
}
```

The client just does:

```java
Payment payment = PaymentFactory.create("UPI");
payment.pay();
```

---

### Q8. What problem does Factory solve?

**Answer:** It reduces coupling to concrete classes, duplicated creation logic, complex conditional construction, and the ripple effect of implementation changes. The client only needs to know `Payment`, not `UpiPayment`, `CardPayment`, or `BankTransferPayment`.

---

### Q9. Factory Method vs Simple Factory — are they the same?

**Answer:** Not exactly. A **Simple Factory** is typically an application-level technique where one factory class picks an implementation. **Factory Method** is the actual GoF pattern where object creation is delegated to subclasses or overridable factory methods:

```text
Creator
   |
   +-- createProduct()
   |
ConcreteCreator
   |
   +-- ConcreteProduct
```

A lot of codebases call a switch-based factory a "Factory Pattern," but strictly speaking that's usually a Simple Factory rather than GoF Factory Method.

---

### Q10. When should you use Factory?

**Answer:** When creation depends on runtime information, there are multiple implementations, the creation logic is complex, you want to isolate concrete types, or new implementations are expected over time. I'd avoid it when `new UserService()` is already simple with no meaningful variation.

---

### Q11. How would you avoid a huge switch statement in a Factory?

**Answer:** Use a registry/map instead:

```java
Map<String, Supplier<Payment>> factories = Map.of(
    "UPI", UpiPayment::new,
    "CARD", CardPayment::new
);

Payment payment = factories.get(type).get();
```

For larger Spring applications, dependency injection makes this even cleaner — inject a `Map<String, PaymentProcessor> processors` and select by key.

---

# 3. BUILDER

## Concept

### What problem does Builder solve?

Builder is useful when an object has many optional parameters, combinations of fields, validation rules, immutable state, or a constructor that would otherwise become unreadable. Without it:

```java
new User("Vishal", null, null, true, "IN", null, ...);
```

is hard to read and maintain. With Builder:

```java
User user = User.builder()
        .name("Vishal")
        .country("IN")
        .active(true)
        .build();
```

---

### Q12. What is the Builder pattern?

**Answer:** Builder separates complex object construction from the final object representation. A typical implementation:

```java
public final class User {

    private final String name;
    private final String country;
    private final boolean active;

    private User(Builder builder) {
        this.name = builder.name;
        this.country = builder.country;
        this.active = builder.active;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String name;
        private String country;
        private boolean active;

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        public Builder country(String country) {
            this.country = country;
            return this;
        }

        public Builder active(boolean active) {
            this.active = active;
            return this;
        }

        public User build() {
            if (name == null || name.isBlank()) {
                throw new IllegalArgumentException("name required");
            }
            return new User(this);
        }
    }
}
```

---

### Q13. Why is Builder better than a constructor with many parameters?

**Answer:** It gives readable construction, optional parameters, validation in one place, fewer parameter-order mistakes, easier evolution, and an immutable final object. Compare:

```java
new User("A", "IN", true, false, null, 10, ...);
```

against:

```java
User.builder()
    .name("A")
    .country("IN")
    .active(true)
    .build();
```

The builder version is self-documenting at the call site.

---

### Q14. Is Builder always better?

**Answer:** No. For a simple object like `new Point(10, 20)`, Builder just adds unnecessary code and indirection. I only reach for it when construction complexity actually justifies it.

---

### Q15. How is Builder related to immutability?

**Answer:** Builder collects mutable construction state and produces an immutable final object:

```text
Mutable Builder
      |
      | build()
      v
Immutable Object
```

That combination — mutable builder, immutable result — is common and genuinely useful.

---

### Q16. What are common Builder mistakes?

**Answer:** Forgetting validation, exposing mutable internal collections, reusing builders incorrectly, allowing invalid field combinations, and letting builder state be shared across threads. Builders are meant to be local construction objects, not shared mutable state.

---

# 4. PROTOTYPE

## Concept

### What problem does Prototype solve?

Sometimes creating an object from scratch is expensive or complex, while an existing object is already close to what's needed. Prototype creates a new object by copying an existing one:

```text
Existing Object
      |
    clone
      |
      v
New Object
```

---

### Q17. What is Prototype pattern?

**Answer:** Prototype creates new objects by copying an existing prototype rather than constructing from scratch. Java's `Cloneable`/`clone()` exists for this, but it's often awkward — I generally prefer an explicit copy constructor:

```java
class User {

    private String name;

    User(User other) {
        this.name = other.name;
    }
}
```

---

### Q18. Shallow copy vs deep copy?

**Answer:** A shallow copy copies field references — both objects end up pointing at the same nested object:

```text
Object A
  |
  +---- Address ----+
                    |
Object B -----------+
```

A deep copy creates independent nested objects:

```text
Object A -> Address A
Object B -> Address B
```

The right strategy depends on ownership and mutability of the object graph — blind deep copying can get expensive fast.

---

### Q19. Why is Object.clone() often avoided?

**Answer:** `clone()` has real design problems — `Cloneable` is only a marker interface, `Object.clone()` is protected, the default is shallow-copy which can surprise people, inheritance complicates correctness, and mutable nested state needs extra handling. For most domain objects, copy constructors or explicit copy methods are clearer and safer.

---

# SECTION 2: STRUCTURAL DESIGN PATTERNS

Structural patterns focus on **how classes and objects are composed**.

Patterns: Adapter, Decorator, Proxy, Facade. Also important: Composite, Bridge.

---

# 5. ADAPTER

## Concept

### What problem does Adapter solve?

Two components have compatible responsibilities but incompatible interfaces:

```text
Application
    |
ExpectedPaymentGateway
    |
 Adapter
    |
LegacyPaymentApi
```

The adapter translates one interface into another.

---

### Q20. What is Adapter pattern?

**Answer:** Adapter converts the interface of an existing class into the interface a client expects.

```java
interface PaymentGateway {
    void pay(double amount);
}

class LegacyPaymentApi {
    void makePayment(double amount) {
        System.out.println("Legacy payment");
    }
}

class PaymentAdapter implements PaymentGateway {

    private final LegacyPaymentApi legacy;

    PaymentAdapter(LegacyPaymentApi legacy) {
        this.legacy = legacy;
    }

    public void pay(double amount) {
        legacy.makePayment(amount);
    }
}
```

---

### Q21. When is Adapter useful in real projects?

**Answer:** Integrating legacy APIs, wrapping third-party libraries, migrating old interfaces, and standardizing multiple providers behind one contract:

```text
PaymentService
    |
PaymentGateway
    |
    +-- StripeAdapter
    +-- LegacyBankAdapter
    +-- AnotherProviderAdapter
```

This keeps the business layer independent of vendor-specific APIs.

---

### Q22. Adapter vs Facade?

**Answer:** Adapter changes an interface to make two components compatible. Facade provides a simpler interface over a complex subsystem.

```text
Adapter:
A interface → Adapter → B interface

Facade:
Client → Simple Facade → Complex subsystem
```

---

# 6. DECORATOR

## Concept

### What problem does Decorator solve?

You want to add behavior to an object dynamically without changing its original class.

```text
Component
   |
Decorator
   |
Decorator
   |
Concrete Component
```

Like layering a coffee: basic coffee, then milk, then sugar, then whipped cream.

---

### Q23. What is Decorator pattern?

**Answer:** Decorator wraps an object and adds behavior while preserving the same interface.

```java
interface Service {
    void execute();
}

class BasicService implements Service {
    public void execute() {
        System.out.println("Executing");
    }
}

class LoggingDecorator implements Service {

    private final Service delegate;

    LoggingDecorator(Service delegate) {
        this.delegate = delegate;
    }

    public void execute() {
        System.out.println("Before");
        delegate.execute();
        System.out.println("After");
    }
}
```

Usage:

```java
Service service =
    new LoggingDecorator(new BasicService());
```

---

### Q24. Decorator vs inheritance?

**Answer:** Inheritance adds behavior statically through the class hierarchy. Decorator adds behavior dynamically through composition:

```text
Inheritance:
Base
 └── FeatureA
      └── FeatureB

Decorator:
FeatureB(
   FeatureA(
      Base
   )
)
```

Decorator avoids exploding into subclasses for every combination of features.

---

### Q25. What are real-world Java examples of Decorator?

**Answer:** Java I/O is the classic example:

```java
new BufferedInputStream(
    new FileInputStream("data.txt"));
```

```text
FileInputStream
      ↓
BufferedInputStream
```

You see the same pattern for compression, encryption, counting, and logging wrappers.

---

### Q26. What is a drawback of Decorator?

**Answer:** Too many wrappers make debugging and construction harder to follow. A chain like `A(B(C(D(E(F(...))))))` can become genuinely difficult to reason about.

---

# 7. PROXY

## Concept

### What problem does Proxy solve?

Proxy provides a substitute object that controls access to another object:

```text
Client
  |
Proxy
  |
Real Object
```

Common reasons: security, lazy loading, remote access, caching, logging, transaction handling.

---

### Q27. What is Proxy pattern?

**Answer:** Proxy controls access to a target object while exposing a compatible interface.

```java
interface Service {
    void execute();
}

class RealService implements Service {
    public void execute() {
        System.out.println("Real work");
    }
}

class SecurityProxy implements Service {

    private final Service target;

    SecurityProxy(Service target) {
        this.target = target;
    }

    public void execute() {
        checkPermission();
        target.execute();
    }

    private void checkPermission() {
        System.out.println("Checking permission");
    }
}
```

---

### Q28. Proxy vs Decorator?

**Answer:** Structurally they look almost identical — both wrap objects — but the intent differs. Proxy controls access to an object; Decorator adds responsibilities/behavior.

```text
Proxy:
Security → Real Service

Decorator:
Logging → Metrics → Real Service
```

---

### Q29. Where do you see Proxy in Spring?

**Answer:** Spring heavily uses proxies for cross-cutting concerns like `@Transactional`, method security, caching, and AOP advice in general:

```text
Client
   |
Spring Proxy
   |
Target Bean
```

The proxy runs logic before/after delegating to the target — which is exactly why Spring AOP has some well-known proxy-related limitations, including self-invocation behavior.

---

### Q30. What is the self-invocation problem in Spring proxy-based AOP?

**Answer:** Given:

```java
@Service
class OrderService {

    public void outer() {
        inner();
    }

    @Transactional
    public void inner() {
        // ...
    }
}
```

calling `orderService.outer()` doesn't route the internal `inner()` call through the Spring proxy, because it's just `this.inner()` — so proxy-based advice like `@Transactional` may not apply as expected. The issue isn't that `@Transactional` is broken; it's that the internal call bypasses the proxy entirely.

---

# 8. FACADE

## Concept

### What problem does Facade solve?

A subsystem may contain many complex classes. Instead of forcing clients to understand all of them:

```text
Client
  |
  v
Facade
  |
  +-- Service A
  +-- Service B
  +-- Service C
  +-- Service D
```

Facade exposes a simpler entry point.

---

### Q31. What is Facade pattern?

**Answer:** Facade provides a simplified interface to a complex subsystem.

```java
class OrderFacade {

    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    void placeOrder(Order order) {
        inventory.reserve(order);
        payment.charge(order);
        shipping.ship(order);
    }
}
```

The caller just does `facade.placeOrder(order);`.

---

### Q32. Facade vs Adapter?

**Answer:**

| Adapter | Facade |
|---|---|
| Converts interface | Simplifies interface |
| Usually wraps one target | Often coordinates multiple classes |
| Compatibility problem | Complexity problem |
| Makes existing API fit expected API | Gives client an easier API |

---

### Q33. Can a Facade become a God class?

**Answer:** Yes — if it starts absorbing business logic for every subsystem it coordinates, it becomes a maintenance bottleneck. A good facade primarily coordinates and delegates rather than owning unrelated domain responsibilities.

---

# SECTION 3: BEHAVIORAL DESIGN PATTERNS

Behavioral patterns focus on **how objects communicate and distribute responsibilities**.

Patterns: Strategy, Observer, Template Method, Command. Also important: Chain of Responsibility, State, Mediator, Iterator.

---

# 9. STRATEGY

## Concept

### What problem does Strategy solve?

Business logic with many interchangeable algorithms:

```java
if (type.equals("CARD")) ...
else if (type.equals("UPI")) ...
else if (type.equals("BANK")) ...
```

As algorithms grow, this conditional logic becomes hard to maintain. Strategy moves each algorithm behind a common interface:

```text
Context
   |
Strategy
   |
   +-- CardStrategy
   +-- UpiStrategy
   +-- BankStrategy
```

---

### Q34. What is Strategy pattern?

**Answer:** Strategy defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```java
interface PaymentStrategy {
    void pay(double amount);
}

class CardStrategy implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("Card");
    }
}

class UpiStrategy implements PaymentStrategy {
    public void pay(double amount) {
        System.out.println("UPI");
    }
}

class PaymentService {

    private final PaymentStrategy strategy;

    PaymentService(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    void pay(double amount) {
        strategy.pay(amount);
    }
}
```

---

### Q35. Strategy vs Factory?

**Answer:** Different problems. Factory answers "which object should I create?" Strategy answers "which algorithm/behavior should I execute?" They're often used together — a Factory selects a Strategy, then the Strategy executes:

```text
Factory
  ↓
select Strategy
  ↓
Strategy executes behavior
```

---

### Q36. Strategy vs if/else?

**Answer:** A small number of stable conditions is often clearer as plain `if`/`switch`. Strategy earns its place when algorithms grow, behavior changes independently, new strategies are added frequently, each algorithm benefits from separate testing, or selection happens at runtime. I don't replace every switch with a design pattern — the abstraction has to pay for itself.

---

### Q37. How would you implement Strategy in Spring?

**Answer:** Define an interface and inject the implementations:

```java
public interface PaymentStrategy {
    String type();
    void pay(Order order);
}

@Component
class CardPaymentStrategy implements PaymentStrategy {
    public String type() {
        return "CARD";
    }

    public void pay(Order order) {
        // ...
    }
}
```

Then build a registry:

```java
@Component
class PaymentStrategyRegistry {

    private final Map<String, PaymentStrategy> strategies;

    PaymentStrategyRegistry(List<PaymentStrategy> strategies) {
        this.strategies = strategies.stream()
            .collect(Collectors.toMap(
                PaymentStrategy::type,
                Function.identity()));
    }

    PaymentStrategy get(String type) {
        return Optional.ofNullable(strategies.get(type))
            .orElseThrow();
    }
}
```

This is a very common enterprise replacement for large conditional blocks.

---

# 10. OBSERVER

## Concept

### What problem does Observer solve?

One object changes state and multiple interested objects need to be notified, without tightly coupling the producer to every consumer:

```text
Publisher
   |
   +---- Observer A
   +---- Observer B
   +---- Observer C
```

---

### Q38. What is Observer pattern?

**Answer:** Observer defines a one-to-many dependency where observers get notified when the subject changes.

```java
interface Observer {
    void update(String event);
}

class EventPublisher {

    private final List<Observer> observers = new ArrayList<>();

    void subscribe(Observer observer) {
        observers.add(observer);
    }

    void publish(String event) {
        for (Observer observer : observers) {
            observer.update(event);
        }
    }
}
```

---

### Q39. Where is Observer used in modern applications?

**Answer:** Application events, UI event systems, domain events, messaging systems, and reactive streams. Spring supports it directly:

```java
@EventListener
public void handle(OrderCreatedEvent event) {
    // ...
}
```

That's essentially an event-driven form of Observer.

---

### Q40. What are the problems with Observer?

**Answer:** Unexpected notification chains, ordering complexity, synchronous observers blocking the publisher, memory leaks from unremoved subscriptions, hard-to-debug behavior, and cascading failures. I wouldn't treat an in-process Observer as equivalent to a durable message broker — delivery, retries, persistence, ordering, and failure semantics are genuinely different.

---

### Q41. Observer vs Pub/Sub?

**Answer:** Observer is typically an in-process object relationship. Pub/Sub usually goes through a broker or messaging infrastructure.

```text
Observer:
Subject → Objects

Pub/Sub:
Producer → Broker → Consumers
```

Pub/Sub adds distributed-system capabilities like persistence, retries, consumer groups, scaling, and decoupling across services.

---

# 11. TEMPLATE METHOD

## Concept

### What problem does Template Method solve?

Several algorithms share the same overall workflow but differ in some individual steps. Instead of duplicating the workflow — read, validate, transform, persist, notify — Template Method fixes the invariant structure and lets subclasses customize specific steps.

---

### Q42. What is Template Method pattern?

**Answer:** It defines the skeleton of an algorithm in a base class while letting subclasses override selected steps.

```java
abstract class DataProcessor {

    public final void process() {
        read();
        validate();
        transform();
        save();
    }

    protected abstract void read();

    protected void validate() {
        // common validation
    }

    protected abstract void transform();

    protected abstract void save();
}
```

---

### Q43. Why should the template method often be final?

**Answer:** If the overall workflow is an invariant business process, marking it `final` stops subclasses from changing the sequence itself:

```java
public final void process() {
    read();
    validate();
    transform();
    save();
}
```

Subclasses customize steps, not the workflow order.

---

### Q44. Template Method vs Strategy?

**Answer:** Template Method uses inheritance — the algorithm skeleton lives in the base class, and subclasses customize steps. Strategy uses composition — the entire algorithm can be swapped out, usually with more runtime flexibility.

```text
Template:
Base class
   ↓
Subclass changes steps

Strategy:
Context
   ↓
Strategy object changes algorithm
```

I prefer Strategy when I need runtime composition and flexibility, and Template Method when the algorithm skeleton is genuinely stable and inheritance represents a meaningful relationship.

---

# 12. COMMAND

## Concept

### What problem does Command solve?

Command encapsulates a request as an object, which lets it be queued, logged, retried, scheduled, undone, or composed.

```text
Invoker
   |
Command
   |
Receiver
```

---

### Q45. What is Command pattern?

**Answer:** A command object encapsulates an operation and its parameters.

```java
interface Command {
    void execute();
}

class CreateOrderCommand implements Command {

    private final OrderService service;
    private final Order order;

    CreateOrderCommand(OrderService service, Order order) {
        this.service = service;
        this.order = order;
    }

    public void execute() {
        service.create(order);
    }
}
```

An invoker can execute it later:

```java
class CommandInvoker {

    void submit(Command command) {
        command.execute();
    }
}
```

---

### Q46. Where is Command useful in enterprise systems?

**Answer:** Job queues, task scheduling, audit logs, undo/redo, workflow engines, retryable operations, and asynchronous processing:

```text
API
 ↓
Command
 ↓
Queue
 ↓
Worker
 ↓
Receiver
```

---

### Q47. Command vs Strategy?

**Answer:** Strategy encapsulates an algorithm — "how should this calculation/payment be performed?" Command encapsulates a request/action — "what operation should be executed?" Command typically also carries its own data and can have a lifecycle independent of the caller.

---

# SECTION 4: ADDITIONAL HIGH-IMPORTANCE PATTERNS

The patterns above are the core set, but for senior Java interviews you should also know the following.

---

# 13. ABSTRACT FACTORY

**Answer:** Abstract Factory creates families of related objects without exposing their concrete classes.

```text
UIFactory
  |
  +-- createButton()
  +-- createCheckbox()

WindowsFactory
  ├── WindowsButton
  └── WindowsCheckbox

MacFactory
  ├── MacButton
  └── MacCheckbox
```

Factory creates one product/type of object; Abstract Factory creates a related family of objects.

---

# 14. CHAIN OF RESPONSIBILITY

**Answer:** Chain of Responsibility passes a request through a chain of handlers until one handles it or the chain finishes.

```text
Request
  ↓
Handler A
  ↓
Handler B
  ↓
Handler C
```

Common enterprise examples: authentication filters, authorization, validation pipelines, servlet filters, Spring Security filter chains, and logging pipelines.

---

# 15. STATE

**Answer:** State applies when an object's behavior changes significantly based on its current state. Instead of:

```java
if (state == NEW) ...
else if (state == PAID) ...
else if (state == CANCELLED) ...
```

you model state-specific behavior directly:

```text
Order
 |
 +-- NewState
 +-- PaidState
 +-- ShippedState
 +-- CancelledState
```

Strategy is generally selected to choose an algorithm; State represents an object's current condition and how its behavior transitions over time.

---

# 16. COMPOSITE

**Answer:** Composite lets you treat individual objects and groups of objects uniformly:

```text
File
Folder
  ├── File
  ├── File
  └── Folder
       └── File
```

Useful for hierarchical structures like filesystem trees, organization structures, UI component trees, and expression trees.

---

# 17. BRIDGE

**Answer:** Bridge separates abstraction from implementation so both can evolve independently:

```text
Abstraction
     |
Implementor
     |
Concrete Implementor
```

Useful when two dimensions of variation would otherwise create a huge inheritance hierarchy.

---

# SECTION 5: DESIGN PATTERNS IN SPRING

### Q48. Which design patterns are commonly used internally by Spring?

**Answer:**

| Pattern | Spring example |
|---|---|
| Singleton | Default singleton bean scope |
| Factory | `BeanFactory`, `FactoryBean` |
| Proxy | AOP, `@Transactional`, security |
| Template Method | `JdbcTemplate` |
| Strategy | Pluggable implementations |
| Observer | Application events |
| Adapter | Handler adapters |
| Decorator | Various wrapper-based abstractions |
| Facade | Higher-level service abstractions |

Spring isn't just "an implementation of the GoF patterns" — it combines these ideas with dependency injection, inversion of control, AOP, and framework infrastructure.

---

### Q49. Why is JdbcTemplate an example of Template Method-like design?

**Answer:** The framework owns the common JDBC workflow — acquire resources, execute the operation, handle common infrastructure, clean up — and application code only supplies the variable behavior. That's the essence of Template Method: a stable workflow plus a customizable operation.

---

### Q50. How does Spring use the Proxy pattern?

**Answer:** Spring creates a proxy around a target bean to layer in cross-cutting concerns without putting all of them directly into business methods:

```text
Caller
  |
  v
Proxy
  |
  +-- Transaction handling
  +-- Security
  +-- Caching
  +-- AOP advice
  |
  v
Target Bean
```

---

# SECTION 6: PATTERN COMPARISON QUESTIONS

### Q51. Factory vs Builder?

| Factory | Builder |
|---|---|
| Chooses/creates implementation | Builds complex object |
| Runtime type selection | Step-by-step construction |
| Hides concrete class creation | Handles many construction parameters |
| Often returns interface | Usually returns one final object |

---

### Q52. Adapter vs Decorator?

| Adapter | Decorator |
|---|---|
| Changes interface | Preserves interface |
| Makes incompatible APIs work together | Adds behavior |
| Compatibility | Responsibility extension |

---

### Q53. Decorator vs Proxy?

| Decorator | Proxy |
|---|---|
| Adds responsibilities | Controls access |
| Intent is behavior extension | Intent is access/control |
| Often composable | Often represents a target |
| Logging/metrics/features | Security/lazy/remote/caching |

The structures may look nearly identical — **intent** is what distinguishes them.

---

### Q54. Strategy vs State?

| Strategy | State |
|---|---|
| Select algorithm | Represents current state |
| Behavior usually selected by context | Behavior changes as state changes |
| Focuses on interchangeable behavior | Focuses on lifecycle/state transitions |

---

### Q55. Strategy vs Template Method?

| Strategy | Template Method |
|---|---|
| Composition | Inheritance |
| Runtime replacement | Subclass customization |
| Flexible | Strong workflow control |
| Usually preferred when composition fits | Useful when algorithm skeleton is stable |

---

### Q56. Observer vs Command?

| Observer | Command |
|---|---|
| Notification | Encapsulated action |
| One-to-many event relationship | Request represented as object |
| Pushes updates | Can queue/schedule/retry |
| Event-oriented | Action-oriented |

---

# SECTION 7: SENIOR PRODUCTION SCENARIOS

### Q57. You have 20 payment providers. How would you design the payment module?

**Answer:** I'd avoid a large `if/else` or `switch` entirely. I'd use:

```text
PaymentService
      |
PaymentStrategy
      |
      +-- Card
      +-- UPI
      +-- Bank
      +-- Wallet
```

with a registry/factory selecting the strategy:

```text
Request
  ↓
Factory/Registry
  ↓
PaymentStrategy
  ↓
Provider Adapter
  ↓
External API
```

That combines Strategy, Factory/Registry, and Adapter — each solving a distinct part of the problem. The important point is that patterns can be combined when each one addresses something different.

---

### Q58. You need to integrate three third-party payment APIs with different interfaces. Which pattern?

**Answer:** Adapter.

```text
PaymentService
      |
PaymentGateway
      |
      +-- ProviderAAdapter
      +-- ProviderBAdapter
      +-- ProviderCAdapter
```

The business layer depends on `PaymentGateway`, not any provider-specific API.

---

### Q59. You need logging, metrics, authorization and caching around a service. Which pattern?

**Answer:** Conceptually this is Decorator/Proxy territory. In Spring, proxy-based AOP commonly delivers these cross-cutting concerns:

```text
Client
 ↓
Security Proxy
 ↓
Transaction Proxy
 ↓
Caching Proxy
 ↓
Target Service
```

The exact wiring depends on the framework's infrastructure and ordering requirements.

---

### Q60. Your order workflow has 15 steps but only 3 vary by order type. Which pattern?

**Answer:** If the workflow itself is stable, Template Method is a good fit and inheritance makes sense here. If the varying behaviors need runtime composition or independent replacement, I'd lean toward Strategy instead. The key question: is the workflow fixed, or is the entire behavior interchangeable?

---

### Q61. You need to queue user operations and execute them asynchronously. Which pattern?

**Answer:** Command is a strong fit.

```text
User Request
   ↓
Command Object
   ↓
Queue
   ↓
Worker
   ↓
Receiver
```

It makes the request independently representable, enabling delayed execution, retries, scheduling, and auditing. For distributed systems, I'd combine this with proper messaging and delivery semantics rather than treating Command itself as a message broker.

---

### Q62. Your code contains a 500-line switch based on order status. What would you consider?

**Answer:** First I'd check whether the conditions represent state-dependent behavior — if so, consider State. If they represent interchangeable algorithms, consider Strategy. If they simply select an implementation, consider Factory/Registry. I wouldn't blindly replace a switch with a pattern without identifying which problem it actually is.

---

# SECTION 8: DESIGN PRINCIPLES BEHIND PATTERNS

### Q63. Which SOLID principles are commonly supported by design patterns?

**Answer:** Patterns often help implement SOLID directly. SRP is supported by separating responsibilities into focused classes. OCP is supported by Strategy and Factory, which let you add implementations without modifying existing core logic. LSP requires abstractions to stay safely substitutable. ISP is supported by focused interfaces. DIP is supported by depending on abstractions rather than concrete implementations — many patterns become far more useful once combined with dependency inversion.

---

### Q64. What is "composition over inheritance"?

**Answer:** It means building behavior by combining objects instead of creating deep inheritance hierarchies. Decorator and Strategy are the strongest examples.

```text
Inheritance:
Base → A → B → C

Composition:
Object
  |
  +-- Strategy
  +-- Decorator
  +-- Dependency
```

Composition usually gives more runtime flexibility and reduces coupling.

---

### Q65. Can too many design patterns be a bad thing?

**Answer:** Absolutely — overengineering leads to unnecessary classes, indirection, harder debugging, slower onboarding, and abstraction without real value. I introduce a pattern when the underlying problem is recurring and the pattern genuinely reduces complexity, not simply because the pattern exists.

---

# SECTION 9: INTERVIEW TRAPS

### Q66. Is Singleton the same as Spring singleton scope?

**Answer:** No. A Spring singleton means the container maintains one bean instance per application context — that's a container-managed lifecycle. The GoF Singleton pattern is a class-level mechanism enforcing a single instance via a private constructor and static access point. Spring singleton scope doesn't require either of those.

---

### Q67. Are Factory and Dependency Injection competing patterns?

**Answer:** Not necessarily. A factory controls object selection/creation; DI delegates dependency creation and wiring to a container/framework. They coexist fine:

```text
DI Container
   ↓
Factory/Registry
   ↓
Selected implementation
```

---

### Q68. Is Decorator always better than inheritance?

**Answer:** No — use the simplest design that fits. Decorator earns its place when combinations are dynamic, behavior should be composed, or subclass explosion is becoming a real problem. Simple inheritance is perfectly reasonable for stable, straightforward specialization.

---

### Q69. Is Proxy and Decorator the same pattern?

**Answer:** Structurally they can look nearly identical, but the distinction is intent — Proxy controls access, Decorator adds responsibilities. This is a classic interview trap question.

---

### Q70. Can one design use multiple patterns?

**Answer:** Yes, and real enterprise systems commonly do. A typical payment architecture:

```text
Controller
   ↓
Facade
   ↓
Factory/Registry
   ↓
Strategy
   ↓
Adapter
   ↓
External Provider
```

Each pattern solves a different problem — Facade simplifies subsystem access, Factory/Registry selects the implementation, Strategy encapsulates payment behavior, and Adapter integrates the incompatible external API.

---

# SECTION 10: TOP 20 DESIGN PATTERN QUESTIONS TO PRIORITIZE

1. **What problem do design patterns solve?**
2. **What is Singleton and when should you avoid it?**
3. **How do you implement a thread-safe Singleton?**
4. **Why is volatile required in double-checked locking?**
5. **Factory vs Factory Method vs Abstract Factory**
6. **Factory vs Dependency Injection**
7. **When would you use Builder?**
8. **Why is Builder useful for immutable objects?**
9. **Prototype and shallow vs deep copy**
10. **Adapter vs Facade**
11. **Decorator vs Proxy**
12. **Where does Spring use Proxy?**
13. **What is the Spring self-invocation problem?**
14. **What is Strategy and how does it replace large conditionals?**
15. **Strategy vs State**
16. **Observer vs Pub/Sub**
17. **Template Method vs Strategy**
18. **What is Command and when is it useful?**
19. **How do design patterns relate to SOLID?**
20. **Can multiple design patterns be combined in one architecture?**

---

# QUICK-FIRE INTERVIEW QUESTIONS

- What is a design pattern? → Reusable solution approach to a recurring design problem.
- Main GoF categories? → Creational, Structural, Behavioral.
- Singleton problem? → Controlled single shared instance.
- Factory problem? → Encapsulated object creation/selection.
- Builder problem? → Complex object construction.
- Prototype problem? → Create objects by copying an existing prototype.
- Adapter problem? → Incompatible interfaces.
- Decorator problem? → Add behavior dynamically.
- Proxy problem? → Control access to an object.
- Facade problem? → Simplify a complex subsystem.
- Strategy problem? → Interchangeable algorithms.
- Observer problem? → One-to-many notification.
- Template Method problem? → Stable algorithm skeleton with variable steps.
- Command problem? → Encapsulate a request/action.
- Strategy uses? → Composition.
- Template Method uses? → Inheritance.
- Adapter changes interface? → Yes.
- Decorator normally preserves interface? → Yes.
- Proxy and Decorator structurally similar? → Yes.
- Is Singleton always recommended? → No.
- Is Factory always necessary? → No.
- Does Spring use proxies? → Yes, extensively.
- Can patterns be combined? → Yes.
- Can patterns cause overengineering? → Yes.
- Best alternative to many hard-coded dependencies? → Dependency Injection.
- Best principle for choosing patterns? → Solve the problem, not the pattern.

---

# SENIOR DEVELOPER CLOSING ANSWER

### If the interviewer asks: "How do you decide which design pattern to use?"

**Answer:** I don't start by picking a pattern — I first identify the actual design problem: object creation, structural integration, or behavior variation. Then I look at coupling, expected change points, lifecycle, testability, and runtime requirements. If object creation varies, I might reach for Factory or Builder. If behavior needs to be interchangeable, Strategy usually fits. If I'm integrating an incompatible API, I use Adapter. For cross-cutting access control or framework interception, Proxy is common. For complex subsystem simplification, I consider Facade. I also stay open to simpler alternatives — dependency injection, composition, or just a plain class. The goal is reducing complexity and making future change safer, not maximizing the number of patterns in the codebase.

---

# FINAL INTERVIEW CHECKLIST

### Creational
- [ ] Singleton
- [ ] Thread-safe Singleton
- [ ] Double-checked locking
- [ ] `volatile`
- [ ] Enum Singleton
- [ ] Factory
- [ ] Factory Method
- [ ] Abstract Factory
- [ ] Builder
- [ ] Prototype
- [ ] Shallow vs deep copy
- [ ] Dependency Injection

### Structural
- [ ] Adapter
- [ ] Decorator
- [ ] Proxy
- [ ] Facade
- [ ] Composite
- [ ] Bridge

### Behavioral
- [ ] Strategy
- [ ] Observer
- [ ] Template Method
- [ ] Command
- [ ] Chain of Responsibility
- [ ] State

### Spring
- [ ] Singleton bean scope
- [ ] FactoryBean / BeanFactory
- [ ] Proxy
- [ ] `@Transactional`
- [ ] Spring AOP
- [ ] Self-invocation
- [ ] Template-style abstractions
- [ ] Application events
- [ ] Strategy via dependency injection

### Senior-level thinking
- [ ] Problem before pattern
- [ ] Composition over inheritance
- [ ] SOLID
- [ ] Trade-offs
- [ ] Testability
- [ ] Coupling
- [ ] Runtime flexibility
- [ ] Production scenarios
- [ ] Avoid overengineering
- [ ] Combine patterns when each solves a distinct problem

---

# FINAL TAKEAWAY

For an **Experience Java/Spring Boot interview**, don't just memorize:

> "Singleton is Creational, Adapter is Structural, Strategy is Behavioral."

Instead, be able to walk through:

```text
What problem exists?
        ↓
What happens without the pattern?
        ↓
What design does the pattern introduce?
        ↓
What trade-offs does it create?
        ↓
Where would I use it in production?
        ↓
What alternative could I use?
```

That reasoning is far more valuable in a senior interview than pattern definitions alone.
