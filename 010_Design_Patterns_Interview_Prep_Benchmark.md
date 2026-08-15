# 010. Design Patterns — Interview Questions & Answers — Senior Java Developer

**Focus:** Creational, Structural, Behavioral design patterns; what problem each solves, when to use/not use, Java/Spring examples, trade-offs, interview traps, and production scenarios.

---

# HOW TO APPROACH DESIGN PATTERNS IN AN INTERVIEW

A senior-level design-pattern answer should not start with memorized definitions.

Use this structure:

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

A good senior developer should be able to explain both:

- **Why this pattern helps**
- **Why introducing this pattern may be unnecessary**

---

# SECTION 1: CREATIONAL DESIGN PATTERNS

Creational patterns focus on **how objects are created**.

They help when object creation is:

- complex
- conditional
- expensive
- coupled to concrete classes
- required to be controlled or reused

Patterns covered:

1. Singleton
2. Factory
3. Builder
4. Prototype

Also important in industry:

- Abstract Factory
- Dependency Injection

---

# 1. SINGLETON

## Concept

### What problem does Singleton solve?

Sometimes an application needs exactly one shared instance of a component and a controlled way to access it.

Examples can include:

- configuration objects
- application-wide registries
- certain caches
- stateless shared services

The core idea is:

```text
Many callers
     |
     v
 Single shared instance
```

---

### Q1. What is the Singleton design pattern?

**Answer:**

Singleton ensures that a class has only one instance and provides a controlled access point to that instance.

A basic implementation:

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

The initialization-on-demand holder idiom is:

- lazy
- thread-safe
- avoids explicit synchronization on every access

---

### Q2. Why would you use Singleton?

**Answer:**

Use it when the application genuinely requires a single shared instance with controlled lifecycle.

For example:

```text
Application
   |
   +-- Configuration
   |
   +-- Registry
   |
   +-- Shared stateless component
```

But in modern Spring applications, manually implementing Singleton is often unnecessary because the Spring container manages singleton-scoped beans by default.

---

### Q3. What are the different ways to implement Singleton in Java?

**Answer:**

Common approaches:

### Eager initialization

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

Simple and thread-safe, but initialization happens when the class is initialized.

### Lazy synchronized method

```java
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

Thread-safe but synchronization occurs on every call.

### Double-checked locking

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

`volatile` is required to guarantee correct publication.

### Holder idiom

Usually a clean choice:

```java
private static class Holder {
    private static final Singleton INSTANCE = new Singleton();
}
```

### Enum Singleton

```java
public enum Singleton {
    INSTANCE;
}
```

Enum-based Singleton provides strong serialization and reflection guarantees.

---

### Q4. Why is volatile required in double-checked locking?

**Answer:**

Object creation is not conceptually a single CPU/JVM step.

Without correct publication, another thread could observe a non-null reference before the object's construction is safely visible.

`volatile` provides the required memory-visibility and ordering guarantees.

**Senior answer:**

> "In double-checked locking, volatile prevents unsafe publication and ensures that once another thread observes the reference, the object's initialization is correctly visible."

---

### Q5. How can Singleton be broken?

**Answer:**

Potential mechanisms include:

- Reflection
- Serialization
- Cloning
- Multiple class loaders

Enum Singleton is particularly robust against several of these issues.

**Interview trap:**

A private constructor alone does not make a class universally impossible to instantiate through reflection.

---

### Q6. Is Singleton always a good design?

**Answer:**

No.

Singleton can introduce:

- Global mutable state
- Hidden dependencies
- Difficult unit testing
- Tight coupling
- Lifecycle complexity

**Senior answer:**

> "I avoid Singleton as a default design choice. If an object is a dependency, dependency injection is usually clearer because dependencies remain explicit and testable."

---

# 2. FACTORY

## Concept

### What problem does Factory solve?

Suppose client code directly creates many concrete implementations:

```java
Payment payment = new CreditCardPayment();
```

As the number of implementations grows:

```text
CreditCardPayment
UPIPayment
PayPalPayment
BankTransferPayment
...
```

client code becomes coupled to concrete classes.

Factory moves creation logic behind an abstraction.

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

**Answer:**

Factory encapsulates object creation so the client depends on an abstraction instead of knowing which concrete implementation to instantiate.

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

Client:

```java
Payment payment = PaymentFactory.create("UPI");
payment.pay();
```

---

### Q8. What problem does Factory solve?

**Answer:**

It reduces:

- coupling to concrete classes
- duplicated creation logic
- complex conditional construction
- impact of implementation changes

The client only knows:

```java
Payment
```

instead of:

```java
UpiPayment
CardPayment
BankTransferPayment
```

---

### Q9. Factory Method vs Simple Factory — are they the same?

**Answer:**

Not exactly.

A **Simple Factory** is commonly an application-level technique where one factory class chooses an implementation.

**Factory Method** is a GoF pattern where object creation is delegated to subclasses/overridable factory methods.

Example concept:

```text
Creator
   |
   +-- createProduct()
   |
ConcreteCreator
   |
   +-- ConcreteProduct
```

**Senior point:**

> "Many codebases call a switch-based factory a Factory Pattern, but strictly speaking it is often a Simple Factory rather than the GoF Factory Method."

---

### Q10. When should you use Factory?

Use it when:

- creation depends on runtime information
- there are multiple implementations
- creation logic is complex
- you want to isolate concrete types
- new implementations are expected

Avoid it when:

```java
new UserService()
```

is already simple and there is no meaningful variation.

---

### Q11. How would you avoid a huge switch statement in a Factory?

**Answer:**

Use a registry/map.

```java
Map<String, Supplier<Payment>> factories = Map.of(
    "UPI", UpiPayment::new,
    "CARD", CardPayment::new
);

Payment payment = factories.get(type).get();
```

For larger Spring applications, dependency injection can make this even cleaner:

```java
Map<String, PaymentProcessor> processors;
```

Then select the processor by key.

---

# 3. BUILDER

## Concept

### What problem does Builder solve?

Builder is useful when an object has:

- many optional parameters
- combinations of fields
- validation rules
- immutable state
- constructors that would otherwise become difficult to read

Without Builder:

```java
new User("Vishal", null, null, true, "IN", null, ...);
```

This is difficult to read and maintain.

With Builder:

```java
User user = User.builder()
        .name("Vishal")
        .country("IN")
        .active(true)
        .build();
```

---

### Q12. What is the Builder pattern?

**Answer:**

Builder separates complex object construction from the final object representation.

Typical implementation:

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

**Answer:**

Builder provides:

- readable construction
- optional parameters
- validation in one place
- fewer parameter-order mistakes
- easier evolution
- immutable final objects

Compare:

```java
new User("A", "IN", true, false, null, 10, ...);
```

with:

```java
User.builder()
    .name("A")
    .country("IN")
    .active(true)
    .build();
```

---

### Q14. Is Builder always better?

**Answer:**

No.

For a simple object:

```java
new Point(10, 20);
```

Builder adds unnecessary code and indirection.

Use Builder when construction complexity justifies it.

---

### Q15. How is Builder related to immutability?

**Answer:**

Builder can collect mutable construction state while producing an immutable final object.

```text
Mutable Builder
      |
      | build()
      v
Immutable Object
```

This is a common and useful combination.

---

### Q16. What are common Builder mistakes?

**Answer:**

- Forgetting validation
- Exposing mutable internal collections
- Reusing builders incorrectly
- Allowing invalid combinations
- Making builder state shared between threads

**Senior point:**

> "Builders are generally intended to be local construction objects, not shared mutable state."

---

# 4. PROTOTYPE

## Concept

### What problem does Prototype solve?

Sometimes creating an object from scratch is expensive or complex, while an existing object is already close to what we need.

Prototype creates a new object by copying an existing object.

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

**Answer:**

Prototype creates new objects by copying an existing prototype instead of constructing them from scratch.

Java provides `Cloneable`/`clone()`, but the built-in mechanism is often awkward.

A safer approach is often an explicit copy constructor:

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

**Answer:**

Shallow copy copies field references.

```text
Object A
  |
  +---- Address ----+
                    |
Object B -----------+
```

Both objects refer to the same nested object.

Deep copy creates independent nested objects.

```text
Object A -> Address A
Object B -> Address B
```

**Senior point:**

> "The right copying strategy depends on ownership and mutability of the object graph. Blind deep copying can be expensive."

---

### Q19. Why is Object.clone() often avoided?

**Answer:**

`clone()` has several design issues:

- `Cloneable` is only a marker interface
- `Object.clone()` is protected
- shallow-copy semantics can be surprising
- inheritance complicates correctness
- mutable nested state requires extra work

For many domain objects, copy constructors or explicit copy methods are clearer.

---

# SECTION 2: STRUCTURAL DESIGN PATTERNS

Structural patterns focus on **how classes and objects are composed**.

Patterns:

1. Adapter
2. Decorator
3. Proxy
4. Facade

Also important:

- Composite
- Bridge

---

# 5. ADAPTER

## Concept

### What problem does Adapter solve?

Two components have compatible responsibilities but incompatible interfaces.

Example:

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

**Answer:**

Adapter converts the interface of an existing class into an interface expected by the client.

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

Typical examples:

- integrating legacy APIs
- wrapping third-party libraries
- migrating old interfaces
- standardizing multiple providers

Example:

```text
PaymentService
    |
PaymentGateway
    |
    +-- StripeAdapter
    +-- LegacyBankAdapter
    +-- AnotherProviderAdapter
```

The business layer stays independent of vendor-specific APIs.

---

### Q22. Adapter vs Facade?

**Answer:**

**Adapter:**

> Changes an interface to make two components compatible.

**Facade:**

> Provides a simpler interface over a complex subsystem.

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

Example:

```text
Basic Coffee
   ↓
+ Milk
   ↓
+ Sugar
   ↓
+ Whipped Cream
```

---

### Q23. What is Decorator pattern?

**Answer:**

Decorator wraps an object and adds behavior while preserving the same interface.

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

**Answer:**

Inheritance adds behavior statically through the class hierarchy.

Decorator adds behavior dynamically through composition.

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

Decorator avoids creating many subclasses for combinations of features.

---

### Q25. What are real-world Java examples of Decorator?

Examples include Java I/O:

```java
new BufferedInputStream(
    new FileInputStream("data.txt"));
```

Conceptually:

```text
FileInputStream
      ↓
BufferedInputStream
```

Additional examples include wrapping streams for:

- buffering
- compression
- encryption
- counting
- logging

---

### Q26. What is a drawback of Decorator?

Too many wrappers can make debugging and object construction difficult.

If a chain becomes:

```text
A(B(C(D(E(F(...))))))
```

the behavior can become difficult to reason about.

---

# 7. PROXY

## Concept

### What problem does Proxy solve?

Proxy provides a substitute object that controls access to another object.

```text
Client
  |
Proxy
  |
Real Object
```

Possible reasons:

- security
- lazy loading
- remote access
- caching
- logging
- transaction handling

---

### Q27. What is Proxy pattern?

**Answer:**

Proxy controls access to a target object while exposing a compatible interface.

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

**Answer:**

They look structurally similar because both wrap objects.

The intent differs:

**Proxy:**
Controls access to an object.

**Decorator:**
Adds responsibilities/behavior.

Examples:

```text
Proxy:
Security → Real Service

Decorator:
Logging → Metrics → Real Service
```

---

### Q29. Where do you see Proxy in Spring?

**Answer:**

Spring heavily uses proxies for cross-cutting behavior such as:

- `@Transactional`
- method security
- caching
- AOP advice

Conceptually:

```text
Client
   |
Spring Proxy
   |
Target Bean
```

The proxy can execute logic before/after delegating to the target.

**Senior interview point:**

This is why Spring AOP has important proxy-related limitations, including self-invocation behavior.

---

### Q30. What is the self-invocation problem in Spring proxy-based AOP?

**Answer:**

Suppose:

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

Calling:

```java
orderService.outer();
```

does not normally pass the `inner()` call through the Spring proxy because it is an internal `this.inner()` call.

Therefore proxy-based advice may not be applied as expected.

**Senior answer:**

> "The issue is not that @Transactional is broken; the internal call bypasses the proxy."

---

# 8. FACADE

## Concept

### What problem does Facade solve?

A subsystem may contain many complex classes.

Instead of forcing clients to understand all of them:

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

**Answer:**

Facade provides a simplified interface to a complex subsystem.

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

The caller only needs:

```java
facade.placeOrder(order);
```

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

**Answer:**

Yes.

If a facade starts containing business logic for every subsystem, it can become a maintenance bottleneck.

A good facade should primarily coordinate/delegate rather than absorb unrelated domain responsibilities.

---

# SECTION 3: BEHAVIORAL DESIGN PATTERNS

Behavioral patterns focus on **how objects communicate and distribute responsibilities**.

Patterns:

1. Strategy
2. Observer
3. Template Method
4. Command

Also important:

- Chain of Responsibility
- State
- Mediator
- Iterator

---

# 9. STRATEGY

## Concept

### What problem does Strategy solve?

Suppose business logic has many interchangeable algorithms:

```java
if (type.equals("CARD")) ...
else if (type.equals("UPI")) ...
else if (type.equals("BANK")) ...
```

As algorithms grow, conditional logic becomes difficult to maintain.

Strategy moves each algorithm behind a common interface.

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

**Answer:**

Strategy defines a family of algorithms, encapsulates each algorithm, and makes them interchangeable.

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

**Answer:**

They solve different problems.

**Factory:**

> Which object should I create?

**Strategy:**

> Which algorithm/behavior should I execute?

They can be used together:

```text
Factory
  ↓
select Strategy
  ↓
Strategy executes behavior
```

---

### Q36. Strategy vs if/else?

**Answer:**

A small number of stable conditions may be clearer as `if`/`switch`.

Strategy becomes valuable when:

- algorithms grow
- behavior changes independently
- new strategies are added frequently
- testing each algorithm separately is useful
- runtime selection is required

**Senior principle:**

> "Do not replace every switch with a design pattern. The abstraction should pay for itself."

---

### Q37. How would you implement Strategy in Spring?

**Answer:**

Define an interface and inject implementations.

```java
public interface PaymentStrategy {
    String type();
    void pay(Order order);
}
```

Then implementations:

```java
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

A registry can be built:

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

This is a common enterprise replacement for large conditional blocks.

---

# 10. OBSERVER

## Concept

### What problem does Observer solve?

One object changes state and multiple interested objects need to be notified without tightly coupling the producer to every consumer.

```text
Publisher
   |
   +---- Observer A
   +---- Observer B
   +---- Observer C
```

---

### Q38. What is Observer pattern?

**Answer:**

Observer defines a one-to-many dependency where observers are notified when the subject changes.

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

Examples:

- application events
- UI event systems
- domain events
- messaging systems
- reactive streams

In Spring:

```java
@EventListener
public void handle(OrderCreatedEvent event) {
    // ...
}
```

This provides an event-driven form of observer-style communication.

---

### Q40. What are the problems with Observer?

Potential issues:

- unexpected notification chains
- ordering complexity
- synchronous observers blocking publisher
- memory leaks if subscriptions are not removed
- difficult debugging
- cascading failures

**Senior point:**

> "For distributed systems, I would not treat an in-process Observer as equivalent to a durable message broker. Delivery, retries, persistence, ordering, and failure semantics are different."

---

### Q41. Observer vs Pub/Sub?

**Answer:**

Observer is typically an in-process object relationship.

Pub/Sub generally uses a broker or messaging infrastructure.

```text
Observer:
Subject → Objects

Pub/Sub:
Producer → Broker → Consumers
```

Pub/Sub can provide additional distributed-system capabilities such as:

- persistence
- retries
- consumer groups
- scaling
- decoupling across services

---

# 11. TEMPLATE METHOD

## Concept

### What problem does Template Method solve?

Several algorithms share the same overall workflow, but some individual steps differ.

Instead of duplicating the workflow:

```text
validate
  ↓
process
  ↓
persist
  ↓
notify
```

Template Method defines the invariant algorithm structure and allows subclasses to customize specific steps.

---

### Q42. What is Template Method pattern?

**Answer:**

Template Method defines the skeleton of an algorithm in a base class while allowing subclasses to override selected steps.

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

**Answer:**

If the overall workflow is an invariant business process, making the template method `final` prevents subclasses from changing the algorithm sequence.

```java
public final void process() {
    read();
    validate();
    transform();
    save();
}
```

Subclasses customize steps, not the workflow.

---

### Q44. Template Method vs Strategy?

**Answer:**

**Template Method:**

- inheritance
- algorithm skeleton in base class
- subclasses customize steps

**Strategy:**

- composition
- entire algorithm can be replaced
- usually more flexible at runtime

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

**Senior answer:**

> "Prefer Strategy when I need runtime composition and flexibility. Template Method can be appropriate when the algorithm skeleton is stable and inheritance represents a meaningful relationship."

---

# 12. COMMAND

## Concept

### What problem does Command solve?

Command encapsulates a request as an object.

This allows requests to be:

- queued
- logged
- retried
- scheduled
- undone
- composed

```text
Invoker
   |
Command
   |
Receiver
```

---

### Q45. What is Command pattern?

**Answer:**

A command object encapsulates an operation and its parameters.

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

Examples:

- job queues
- task scheduling
- audit logs
- undo/redo
- workflow engines
- retryable operations
- asynchronous processing

Example:

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

**Answer:**

**Strategy** encapsulates an algorithm.

**Command** encapsulates a request/action.

```text
Strategy:
"How should this calculation/payment be performed?"

Command:
"What operation should be executed?"
```

Command often carries operation data and can have a lifecycle independent of the caller.

---

# SECTION 4: ADDITIONAL HIGH-IMPORTANCE PATTERNS

The listed patterns are important, but for senior Java interviews you should also know the following.

---

# 13. ABSTRACT FACTORY

### What problem does it solve?

Creates families of related objects without exposing their concrete classes.

Example:

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

### Factory vs Abstract Factory

**Factory:**

Creates one product/type of object.

**Abstract Factory:**

Creates a related family of objects.

---

# 14. CHAIN OF RESPONSIBILITY

### What problem does it solve?

Passes a request through a chain of handlers until one handles it or the chain finishes.

```text
Request
  ↓
Handler A
  ↓
Handler B
  ↓
Handler C
```

Common enterprise examples:

- authentication filters
- authorization
- validation pipelines
- servlet filters
- Spring Security filter chains
- logging pipelines

---

# 15. STATE

### What problem does State solve?

When an object's behavior changes significantly based on its current state.

Instead of:

```java
if (state == NEW) ...
else if (state == PAID) ...
else if (state == CANCELLED) ...
```

Use state-specific behavior.

```text
Order
 |
 +-- NewState
 +-- PaidState
 +-- ShippedState
 +-- CancelledState
```

**Strategy vs State:**

Strategy is generally selected to choose an algorithm.

State represents an object's current condition and can change transitions over time.

---

# 16. COMPOSITE

### What problem does Composite solve?

Treat individual objects and groups of objects uniformly.

```text
File
Folder
  ├── File
  ├── File
  └── Folder
       └── File
```

Useful for hierarchical structures:

- filesystem trees
- organization structures
- UI component trees
- expression trees

---

# 17. BRIDGE

### What problem does Bridge solve?

Separates abstraction from implementation so both can evolve independently.

```text
Abstraction
     |
Implementor
     |
Concrete Implementor
```

Useful when two dimensions of variation would otherwise create a large inheritance hierarchy.

---

# SECTION 5: DESIGN PATTERNS IN SPRING

### Q48. Which design patterns are commonly used internally by Spring?

**Answer:**

Spring uses many classic design ideas.

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

**Important:**

Spring is not simply "an implementation of the GoF patterns." It combines patterns with dependency injection, inversion of control, AOP, and framework infrastructure.

---

### Q49. Why is JdbcTemplate an example of Template Method-like design?

**Answer:**

The framework controls the common JDBC workflow:

```text
Acquire resources
     ↓
Execute operation
     ↓
Handle common infrastructure
     ↓
Clean up
```

Application code supplies the variable behavior.

This is the essence of Template Method-style design:

> Stable workflow + customizable operation.

---

### Q50. How does Spring use the Proxy pattern?

**Answer:**

Spring can create a proxy around a target bean.

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

This allows cross-cutting concerns without putting all of them directly into business methods.

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

The structures may look nearly identical; **intent matters**.

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

**Answer:**

I would avoid a large `if/else` or `switch`.

Use:

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

A registry/factory can select the strategy:

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

This combines:

- Strategy
- Factory/Registry
- Adapter

The important senior point is that patterns can be **combined** when each addresses a different problem.

---

### Q58. You need to integrate three third-party payment APIs with different interfaces. Which pattern?

**Answer:**

Use **Adapter**.

```text
PaymentService
      |
PaymentGateway
      |
      +-- ProviderAAdapter
      +-- ProviderBAdapter
      +-- ProviderCAdapter
```

The business layer depends on `PaymentGateway`, not provider-specific APIs.

---

### Q59. You need logging, metrics, authorization and caching around a service. Which pattern?

**Answer:**

Conceptually this is a **Decorator/Proxy** use case.

In Spring, proxy-based AOP commonly provides these cross-cutting concerns.

For example:

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

The exact implementation depends on framework infrastructure and ordering requirements.

---

### Q60. Your order workflow has 15 steps but only 3 vary by order type. Which pattern?

**Answer:**

Consider **Template Method** if the workflow is stable and inheritance is appropriate.

If the varying behaviors need runtime composition or independent replacement, prefer **Strategy**.

The key question is:

> "Is the workflow fixed, or is the entire behavior interchangeable?"

---

### Q61. You need to queue user operations and execute them asynchronously. Which pattern?

**Answer:**

**Command** is a strong fit.

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

It makes the request independently representable and allows:

- delayed execution
- retries
- scheduling
- auditing

For distributed systems, combine this with appropriate messaging and delivery semantics rather than treating Command itself as a message broker.

---

### Q62. Your code contains a 500-line switch based on order status. What would you consider?

**Answer:**

First identify whether the conditions represent **state-dependent behavior**.

If they do, consider State.

If they represent interchangeable algorithms, consider Strategy.

If they simply select an object implementation, consider Factory/Registry.

I would not blindly replace a switch with a pattern.

---

# SECTION 8: DESIGN PRINCIPLES BEHIND PATTERNS

### Q63. Which SOLID principles are commonly supported by design patterns?

**Answer:**

Patterns often help implement SOLID principles.

### Single Responsibility

Separate responsibilities into focused classes.

### Open/Closed

Add new implementations without modifying existing core logic.

Strategy and Factory can help.

### Liskov Substitution

Abstractions should remain safely substitutable.

### Interface Segregation

Use focused interfaces.

### Dependency Inversion

Depend on abstractions rather than concrete implementations.

Many patterns become more useful when combined with dependency inversion.

---

### Q64. What is "composition over inheritance"?

**Answer:**

Composition means building behavior by combining objects instead of creating deep inheritance hierarchies.

Decorator and Strategy are strong examples.

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

Composition usually provides more runtime flexibility and reduces coupling.

---

### Q65. Can too many design patterns be a bad thing?

**Answer:**

Absolutely.

Overengineering can cause:

- unnecessary classes
- indirection
- difficult debugging
- slower onboarding
- abstraction without real value

**Senior answer:**

> "I introduce a pattern when the underlying problem is recurring and the pattern reduces complexity. I don't introduce a pattern simply because the pattern exists."

---

# SECTION 9: INTERVIEW TRAPS

### Q66. Is Singleton the same as Spring singleton scope?

**Answer:**

No.

A Spring singleton means the Spring container normally maintains one bean instance per application context.

The GoF Singleton pattern is a class-level mechanism enforcing a single instance.

Spring singleton scope does not require a private constructor or static `getInstance()` method.

---

### Q67. Are Factory and Dependency Injection competing patterns?

**Answer:**

Not necessarily.

A factory controls object selection/creation.

Dependency Injection delegates dependency creation and wiring to a container/framework.

They can coexist.

For example:

```text
DI Container
   ↓
Factory/Registry
   ↓
Selected implementation
```

---

### Q68. Is Decorator always better than inheritance?

**Answer:**

No.

Use the simplest design that fits the problem.

Decorator is particularly useful when:

- combinations are dynamic
- behavior should be composed
- subclass explosion is becoming a problem

Simple inheritance may be perfectly reasonable for stable specialization.

---

### Q69. Is Proxy and Decorator the same pattern?

**Answer:**

Structurally they can look almost identical.

The distinction is **intent**:

- Proxy → control access
- Decorator → add responsibilities

This is a classic interview question.

---

### Q70. Can one design use multiple patterns?

**Answer:**

Yes, and real enterprise systems commonly do.

Example payment architecture:

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

Each pattern solves a different problem:

- Facade → simplify subsystem access
- Factory/Registry → select implementation
- Strategy → encapsulate payment behavior
- Adapter → integrate incompatible external APIs

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

**Answer:**

> "I don't start by choosing a pattern. I first identify the design problem—whether it is object creation, structural integration, or behavior variation. Then I look at the coupling, expected change points, lifecycle, testability, and runtime requirements. If object creation varies, I may use Factory or Builder. If behavior needs to be interchangeable, Strategy is often appropriate. If I need to integrate an incompatible API, I use Adapter. For cross-cutting access control or framework interception, Proxy is common. For complex subsystem simplification, I consider Facade. I also look for simpler alternatives such as dependency injection, composition, or a straightforward class. The goal is to reduce complexity and make future change safer—not to maximize the number of patterns in the codebase."

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

For an **Experience Java/Spring Boot interview**, do not memorize:

> "Singleton is Creational, Adapter is Structural, Strategy is Behavioral."

Instead, be able to explain:

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

That reasoning is much more valuable in a senior interview than pattern definitions alone.
