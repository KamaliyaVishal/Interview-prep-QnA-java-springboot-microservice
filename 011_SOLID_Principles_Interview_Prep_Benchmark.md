# 011. SOLID Principles — Interview Questions & Answers — Senior Java Developer (8+ YOE)

**Focus:** SRP, OCP, LSP, ISP, and the missing fifth SOLID principle—DIP—plus practical Java/Spring Boot examples, violations, refactoring approaches, trade-offs, and senior-level interview scenarios.

> **Interview goal:** Do not memorize only the definitions. Be able to explain **what problem each principle solves, what bad design looks like, how to refactor it, and what trade-offs the principle introduces.**

---

# 1. What are SOLID principles?

### Answer

SOLID is a set of five object-oriented design principles that help create software that is:

- easier to maintain
- easier to test
- less tightly coupled
- easier to extend
- safer to change

SOLID stands for:

| Letter | Principle | Main problem it addresses |
|---|---|---|
| S | Single Responsibility Principle | Too many reasons for a class to change |
| O | Open/Closed Principle | Changes require modifying stable code |
| L | Liskov Substitution Principle | Subtypes cannot safely replace their base types |
| I | Interface Segregation Principle | Clients depend on methods they do not need |
| D | Dependency Inversion Principle | High-level code depends directly on low-level details |

### Senior-level view

> SOLID is not about creating more interfaces and classes. It is about controlling **coupling, change, responsibility, and dependency direction**.

---

# 2. Why are SOLID principles important in real projects?

### Answer

Without good design principles, a codebase can become difficult to change.

A small requirement may cause:

```text
One change
   ↓
Modify large class
   ↓
Break unrelated behavior
   ↓
Update many tests
   ↓
Regression risk
```

SOLID tries to make change more localized:

```text
Requirement
   ↓
Small focused component
   ↓
Existing stable behavior remains unchanged
```

For a senior developer, the important question is not:

> "Did I use all five principles?"

It is:

> "Does the design make the expected changes easier and safer?"

---

# 3. SINGLE RESPONSIBILITY PRINCIPLE (SRP)

## Concept

### What problem does SRP solve?

SRP addresses classes that contain multiple unrelated responsibilities.

A common misconception is:

> "A class should have only one method."

That is not SRP.

The better definition is:

> **A class should have one reason to change.**

A "reason to change" usually means a responsibility owned by a particular actor, business concern, or requirement.

---

## Q1. What is Single Responsibility Principle?

### Answer

A class should have a focused responsibility and therefore a limited set of reasons to change.

Bad example:

```java
class InvoiceService {

    void calculateTotal(Invoice invoice) {
        // business calculation
    }

    void saveToDatabase(Invoice invoice) {
        // persistence
    }

    void generatePdf(Invoice invoice) {
        // presentation/document generation
    }

    void sendEmail(Invoice invoice) {
        // notification
    }
}
```

This class has several independent responsibilities:

```text
InvoiceService
 ├── Business calculation
 ├── Persistence
 ├── PDF generation
 └── Email notification
```

A better design:

```text
InvoiceCalculator
InvoiceRepository
InvoicePdfGenerator
InvoiceNotificationService
```

---

## Q2. What does "one reason to change" actually mean?

### Answer

Suppose:

```java
class InvoiceService {
    calculateInvoice();
    saveInvoice();
    generatePdf();
}
```

Potential changes come from different stakeholders:

- business team changes calculation rules
- database team changes persistence
- document team changes PDF format

That means the class has multiple reasons to change.

SRP asks us to separate those independent axes of change.

### Senior interview answer

> "SRP is about cohesion and change boundaries, not simply the number of methods in a class."

---

## Q3. How do you identify an SRP violation?

Look for:

- large classes
- unrelated fields
- unrelated dependencies
- methods serving different business concerns
- frequent changes to the same class for unrelated requirements
- difficult unit testing
- class names containing `And`, `Manager`, `Processor`, `Service` for many unrelated jobs

Example smell:

```java
UserService
    ├── createUser()
    ├── validateTax()
    ├── generatePdf()
    ├── sendEmail()
    ├── uploadImage()
    └── saveToDatabase()
```

---

## Q4. Does SRP mean every class should be tiny?

### Answer

No.

Over-splitting can be just as harmful.

This is unnecessary:

```text
UserNameValidator
UserEmailValidator
UserAgeValidator
UserCountryValidator
```

if those validations are tightly coupled and naturally belong together.

The goal is **high cohesion**, not minimum class size.

---

## Q5. How would you refactor an SRP violation?

### Answer

First identify the independent responsibilities.

For example:

```java
class OrderService {
    createOrder();
    calculatePrice();
    saveOrder();
    sendEmail();
}
```

Refactor:

```java
class OrderService {
    private final PriceCalculator priceCalculator;
    private final OrderRepository repository;
    private final NotificationService notificationService;
}
```

Now:

```text
OrderService
   ↓
coordinates use case

PriceCalculator
   ↓
business calculation

OrderRepository
   ↓
persistence

NotificationService
   ↓
notification
```

This also improves testability.

---

# 4. OPEN/CLOSED PRINCIPLE (OCP)

## Concept

### What problem does OCP solve?

OCP addresses code that must repeatedly be modified whenever a new behavior is introduced.

Typical smell:

```java
if (type.equals("CARD")) {
    ...
} else if (type.equals("UPI")) {
    ...
} else if (type.equals("BANK")) {
    ...
}
```

Every new payment type requires modifying the existing class.

OCP encourages a design where new behavior can be added through new implementations rather than repeatedly changing stable code.

---

## Q6. What is Open/Closed Principle?

### Answer

> **Software entities should be open for extension but closed for modification.**

Meaning:

- **Open for extension:** new behavior can be added.
- **Closed for modification:** stable existing code should not need repeated changes for every new variation.

Example:

```java
interface PaymentProcessor {
    void process(Payment payment);
}
```

Implementations:

```java
class CardPaymentProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // card logic
    }
}

class UpiPaymentProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // UPI logic
    }
}
```

Adding:

```java
class WalletPaymentProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // wallet logic
    }
}
```

does not require changing the existing processor implementations.

---

## Q7. Does OCP mean we should never modify existing code?

### Answer

No.

That would be unrealistic.

OCP means that **expected variations should be designed so they can be introduced with minimal modification to stable code**.

If a business requirement changes the existing business rule itself, modifying existing code is completely normal.

### Senior answer

> "OCP is about protecting stable abstractions from recurring extensions, not about making every class immutable or impossible to modify."

---

## Q8. How does Strategy help implement OCP?

Suppose:

```java
class DiscountService {

    double calculate(String type, double amount) {

        if ("REGULAR".equals(type)) {
            return amount * 0.05;
        }

        if ("PREMIUM".equals(type)) {
            return amount * 0.10;
        }

        return 0;
    }
}
```

Adding a discount type requires modifying the class.

Using Strategy:

```java
interface DiscountStrategy {
    double calculate(double amount);
}
```

Then:

```java
class RegularDiscount implements DiscountStrategy {
    public double calculate(double amount) {
        return amount * 0.05;
    }
}

class PremiumDiscount implements DiscountStrategy {
    public double calculate(double amount) {
        return amount * 0.10;
    }
}
```

New behavior becomes a new implementation.

This is a common combination:

```text
OCP principle
      +
Strategy pattern
      +
Dependency Injection
```

---

## Q9. Is a switch statement always an OCP violation?

### Answer

No.

A small, stable switch may be perfectly reasonable.

For example:

```java
switch (day) {
    case MONDAY -> ...
    case TUESDAY -> ...
}
```

The question is whether the conditional represents **expected, frequently growing variation**.

### Senior rule

> Do not replace every switch with Strategy or Factory. Introduce an abstraction when the variation is real and likely to evolve.

---

## Q10. What is the relationship between OCP and polymorphism?

### Answer

Polymorphism allows behavior to vary behind an abstraction.

```text
PaymentProcessor
       |
       +-- CardProcessor
       +-- UpiProcessor
       +-- WalletProcessor
```

The caller depends on:

```java
PaymentProcessor
```

rather than concrete implementations.

This allows new implementations to be added without changing the consumer.

---

# 5. LISKOV SUBSTITUTION PRINCIPLE (LSP)

## Concept

### What problem does LSP solve?

LSP addresses inheritance where a subclass technically extends a parent class but cannot actually behave as a valid replacement for it.

The key question is:

> **Can code written for the parent safely use the child without unexpected behavior?**

---

## Q11. What is Liskov Substitution Principle?

### Answer

> Objects of a subtype should be usable wherever objects of the base type are expected without breaking the correctness of the program.

Example of a bad hierarchy:

```java
class Bird {
    void fly() {
        System.out.println("Flying");
    }
}

class Penguin extends Bird {
    @Override
    void fly() {
        throw new UnsupportedOperationException();
    }
}
```

The parent contract implies:

```java
bird.fly();
```

is valid.

But:

```java
Bird bird = new Penguin();
bird.fly();
```

fails.

The subtype violates the expected contract.

---

## Q12. Is LSP only about inheritance?

### Answer

The principle is most visible with inheritance/subtyping, but the deeper concept is **behavioral substitutability**.

It also matters with:

- interfaces
- implementations
- abstractions
- API contracts

If an implementation violates the expectations of its interface, substitutability is broken.

---

## Q13. What are common signs of an LSP violation?

Look for:

- subclass throws `UnsupportedOperationException` for expected parent behavior
- subclass weakens guarantees
- subclass unexpectedly rejects valid parent inputs
- subclass changes important semantics
- callers need `instanceof` checks
- callers need special cases for a particular implementation

Example smell:

```java
if (shape instanceof Rectangle) {
    ...
}
```

or:

```java
if (service instanceof SpecialService) {
    ...
}
```

may indicate the abstraction is wrong.

---

## Q14. What is the classic Rectangle/Square LSP problem?

### Answer

Suppose:

```java
class Rectangle {
    void setWidth(int width) { ... }
    void setHeight(int height) { ... }
}
```

and:

```java
class Square extends Rectangle {
    @Override
    void setWidth(int width) {
        super.setWidth(width);
        super.setHeight(width);
    }
}
```

A client expecting independent width and height behavior can break when given a Square.

The mathematical "is-a" relationship does not automatically imply a valid behavioral subtype.

### Senior lesson

> Inheritance should be based on **behavioral contracts**, not merely conceptual relationships.

---

## Q15. How do you fix an LSP violation?

### Answer

Redesign the abstraction around behavior that all implementations can genuinely support.

Instead of:

```java
Bird
 ├── Sparrow
 └── Penguin
```

with `fly()` in the base class:

```java
Bird
 ├── Sparrow
 └── Penguin

Flyable
 └── Sparrow
```

Now:

```java
interface Flyable {
    void fly();
}
```

Only birds that can fly implement `Flyable`.

---

## Q16. What does LSP imply about exceptions?

### Answer

A subtype should not unexpectedly violate the contract by introducing stronger restrictions.

For example, if the base abstraction promises that an operation succeeds for a valid input, a subtype should not arbitrarily reject that same valid input.

### Senior answer

> "LSP is fundamentally about preserving the behavioral contract expected by clients, including valid inputs, outputs, side effects, and failure behavior."

---

# 6. INTERFACE SEGREGATION PRINCIPLE (ISP)

## Concept

### What problem does ISP solve?

ISP addresses "fat interfaces" where clients are forced to depend on methods they do not use.

Bad design:

```java
interface Worker {
    void work();
    void eat();
    void sleep();
}
```

A robot:

```java
class Robot implements Worker {

    public void work() {}

    public void eat() {
        throw new UnsupportedOperationException();
    }

    public void sleep() {
        throw new UnsupportedOperationException();
    }
}
```

This is a strong signal that the interface is too broad.

---

## Q17. What is Interface Segregation Principle?

### Answer

> **Clients should not be forced to depend on methods they do not use.**

Instead of one large interface:

```java
interface Worker {
    void work();
    void eat();
    void sleep();
}
```

split it:

```java
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}
```

Now a robot can implement only:

```java
class Robot implements Workable {
    public void work() {}
}
```

---

## Q18. Why is ISP useful in microservices and enterprise applications?

### Answer

Large interfaces can create unnecessary coupling.

For example:

```java
interface UserService {
    createUser();
    deleteUser();
    generateReport();
    exportCsv();
    sendNotification();
    authenticate();
    ...
}
```

Different clients may need only a small subset.

Focused interfaces make:

- testing easier
- implementations simpler
- dependencies clearer
- API evolution safer

---

## Q19. ISP vs SRP — what is the difference?

### Answer

They are related but focus on different things.

**SRP:**

> How many reasons does this class have to change?

**ISP:**

> How many unrelated methods are clients forced to depend on?

Example:

```text
SRP → class responsibility

ISP → interface/client dependency
```

---

## Q20. Does ISP mean every interface should contain exactly one method?

### Answer

No.

That would lead to unnecessary fragmentation.

A cohesive interface can contain multiple methods when they naturally belong to the same client responsibility.

The goal is:

> **Client-specific, cohesive contracts.**

Not:

> "One method per interface."

---

# 7. DEPENDENCY INVERSION PRINCIPLE (DIP)

> **Important:** DIP is the fifth SOLID principle and should be included even though it was not explicitly listed in the module outline.

## Concept

### What problem does DIP solve?

High-level business logic should not be tightly coupled to low-level implementation details.

Bad:

```java
class OrderService {

    private final MySqlOrderRepository repository =
        new MySqlOrderRepository();
}
```

Now business logic knows the database implementation.

Better:

```java
interface OrderRepository {
    void save(Order order);
}
```

Then:

```java
class OrderService {

    private final OrderRepository repository;

    OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

Concrete implementation:

```java
class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) {
        // MySQL
    }
}
```

Dependency direction:

```text
High-level policy
       |
       v
   Abstraction
       ^
       |
Low-level detail
```

---

## Q21. What is Dependency Inversion Principle?

### Answer

There are two important parts:

1. High-level modules should not depend directly on low-level modules; both should depend on abstractions.
2. Abstractions should not depend on details; details should depend on abstractions.

Example:

```text
OrderService
     |
     v
OrderRepository
     ^
     |
MySqlOrderRepository
```

The business service depends on the abstraction.

---

## Q22. Is Dependency Inversion the same as Dependency Injection?

### Answer

No.

**Dependency Inversion** is a design principle.

**Dependency Injection** is a technique for supplying dependencies.

```text
DIP
 ↓
Depend on abstractions

DI
 ↓
How dependencies are provided
```

Spring provides Dependency Injection, which can help implement DIP.

---

## Q23. Why is DIP important for testing?

Without DIP:

```java
OrderService
    ↓
new MySqlRepository()
```

Unit tests may require a real database.

With DIP:

```java
OrderService
    ↓
OrderRepository
    ↑
MockOrderRepository
```

Tests can provide a fake/mock implementation.

This reduces infrastructure coupling.

---

# 8. HOW THE FIVE PRINCIPLES WORK TOGETHER

Consider a payment system.

Bad design:

```java
class PaymentService {

    void pay(String type, Payment payment) {

        if ("CARD".equals(type)) {
            // card
        } else if ("UPI".equals(type)) {
            // UPI
        }

        // save to DB
        // send email
        // generate report
    }
}
```

Problems:

- SRP violation
- OCP pressure
- DIP violation
- difficult testing
- high coupling

A better design:

```text
PaymentService
      |
PaymentProcessor
      |
      +-- CardPaymentProcessor
      +-- UpiPaymentProcessor
      +-- WalletPaymentProcessor

PaymentRepository
NotificationService
```

Possible principles:

```text
SRP
 ↓
Separate responsibilities

OCP
 ↓
Add payment processors without changing stable service

LSP
 ↓
Every processor honors PaymentProcessor contract

ISP
 ↓
Use focused interfaces

DIP
 ↓
PaymentService depends on abstractions
```

---

# 9. COMMON INTERVIEW QUESTIONS

## Q24. What is the difference between SRP and OCP?

### Answer

**SRP** controls responsibility:

> A class should have one primary reason to change.

**OCP** controls extensibility:

> Expected new variations should be addable without repeatedly modifying stable code.

Example:

```text
SRP → "Who should own this behavior?"

OCP → "How can I add another variation safely?"
```

---

## Q25. OCP vs DIP?

### Answer

They often work together.

**OCP:**

Design for extension.

**DIP:**

Make dependencies point toward abstractions.

Example:

```text
PaymentService
      |
   interface
      ^
      |
CardPayment
```

DIP creates the abstraction boundary.

OCP allows new implementations to be added behind it.

---

## Q26. SRP vs ISP?

### Answer

```text
SRP → class responsibility

ISP → interface/client dependency
```

SRP asks:

> Does this class have unrelated reasons to change?

ISP asks:

> Is this client forced to depend on methods it does not need?

---

## Q27. LSP vs ISP?

### Answer

**LSP:**

Can an implementation safely substitute for its abstraction?

**ISP:**

Is the abstraction itself too broad for its clients?

They can interact:

```text
Fat Interface
     ↓
Implementations cannot meaningfully support all methods
     ↓
LSP problems
```

ISP can therefore help prevent certain LSP violations.

---

## Q28. Which SOLID principle is most important?

### Answer

There is no universally "most important" principle.

In practice:

- SRP helps establish cohesive components.
- OCP helps manage expected variation.
- LSP protects abstraction correctness.
- ISP keeps contracts focused.
- DIP reduces coupling.

For enterprise applications, **DIP + SRP + OCP** are particularly visible because they strongly affect testability and changeability.

---

# 10. SOLID AND DESIGN PATTERNS

SOLID principles and design patterns are complementary.

### Strategy

Often supports:

- OCP
- DIP
- SRP

### Factory

Often supports:

- OCP
- DIP

### Adapter

Supports:

- DIP
- separation of external details

### Decorator

Supports:

- OCP
- composition

### Template Method

Can support:

- OCP
- controlled variation

### Facade

Can support:

- SRP
- reduced coupling

### Dependency Injection

Strongly supports:

- DIP
- testability

---

# 11. SPRING BOOT EXAMPLES OF SOLID

## Q29. How does Spring support DIP?

### Answer

Instead of:

```java
class OrderService {
    private MySqlOrderRepository repository =
        new MySqlOrderRepository();
}
```

Spring can inject:

```java
interface OrderRepository {
    void save(Order order);
}
```

Implementation:

```java
@Repository
class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) {
        // ...
    }
}
```

Service:

```java
@Service
class OrderService {

    private final OrderRepository repository;

    OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

The service depends on the abstraction.

---

## Q30. How does Spring help with OCP?

A common approach is multiple implementations behind an interface.

```java
interface NotificationSender {
    void send(Notification notification);
}
```

Implementations:

```java
@Component
class EmailNotificationSender implements NotificationSender {
    public void send(Notification notification) {}
}

@Component
class SmsNotificationSender implements NotificationSender {
    public void send(Notification notification) {}
}
```

New notification mechanisms can be added as implementations, while the core abstraction remains stable.

---

# 12. SENIOR PRODUCTION SCENARIOS

## Q31. You have a 2,000-line service class. Which SOLID principle is probably being violated?

### Answer

Likely **SRP**, but I would not conclude that based on line count alone.

I would inspect:

- number of responsibilities
- dependency count
- unrelated methods
- reasons for change
- test complexity
- coupling

A large class can be cohesive, although 2,000 lines is a strong smell worth investigating.

---

## Q32. A new payment type requires changing five existing classes. Which principle should you investigate?

### Answer

Likely **OCP**.

I would look for:

- switch/if chains
- concrete-type checks
- duplicated type-based logic

Then consider:

```text
Strategy
Factory/Registry
Polymorphism
Dependency Injection
```

---

## Q33. A subclass throws UnsupportedOperationException for a parent method. Which principle?

### Answer

Likely **LSP**.

The subclass cannot safely substitute for the parent abstraction.

I would reconsider the hierarchy or split the abstraction.

---

## Q34. An interface has 25 methods and most implementations use only 5. Which principle?

### Answer

Likely **ISP**.

Split the interface around meaningful client responsibilities rather than creating one huge contract.

---

## Q35. A service directly creates repositories, HTTP clients and database clients with `new`. Which principle?

### Answer

Likely **DIP**.

Move dependencies behind abstractions and inject them.

This improves:

- testability
- configurability
- replacement of implementations
- separation of business logic from infrastructure

---

## Q36. Can a class violate more than one SOLID principle?

### Answer

Absolutely.

For example:

```java
class PaymentService {

    void process(String type) {
        if ("CARD".equals(type)) {
            // ...
        } else if ("UPI".equals(type)) {
            // ...
        }

        saveToDatabase();
        sendEmail();
        generateReport();
    }
}
```

Potential issues:

```text
SRP → multiple responsibilities
OCP → new types require modification
DIP → direct infrastructure dependency
```

One design problem can therefore manifest as multiple SOLID violations.

---

# 13. SOLID TRADE-OFFS AND OVERENGINEERING

## Q37. Can following SOLID too strictly make code worse?

### Answer

Yes.

Over-application can create:

```text
One simple requirement
      ↓
10 interfaces
      ↓
15 classes
      ↓
Multiple layers of indirection
```

This increases cognitive load without meaningful benefit.

### Senior answer

> "SOLID is a set of heuristics for managing change and coupling. I apply the level of abstraction justified by the problem and expected evolution."

---

## Q38. Should every class depend on an interface?

### Answer

No.

Creating an interface for every class can become ceremony.

An abstraction is valuable when:

- multiple implementations exist or are expected
- dependency boundaries matter
- external systems need isolation
- testing benefits from substitution
- behavior varies

If there is one stable implementation and no meaningful boundary, a concrete dependency may be perfectly acceptable.

---

# 14. TOP 25 SOLID INTERVIEW QUESTIONS TO PRIORITIZE

1. What is SOLID?
2. Explain Single Responsibility Principle with a real example.
3. What does "one reason to change" mean?
4. How do you identify an SRP violation?
5. Does SRP mean one method per class?
6. Explain Open/Closed Principle.
7. Does OCP mean never modifying existing code?
8. How does Strategy help implement OCP?
9. Is every switch statement an OCP violation?
10. Explain Liskov Substitution Principle.
11. What is the Rectangle/Square LSP problem?
12. How do you identify an LSP violation?
13. What does LSP mean for API contracts and exceptions?
14. Explain Interface Segregation Principle.
15. Does ISP mean one method per interface?
16. SRP vs ISP?
17. Explain Dependency Inversion Principle.
18. DIP vs Dependency Injection?
19. How does Spring implement/support DIP?
20. OCP vs DIP?
21. LSP vs ISP?
22. How do SOLID principles work together?
23. Which design patterns commonly support SOLID?
24. Can a class violate multiple SOLID principles?
25. Can over-applying SOLID be harmful?

---

# 15. QUICK-FIRE INTERVIEW REVISION

### SRP
**Problem:** Too many unrelated responsibilities.

**Question:** "What causes this class to change?"

---

### OCP
**Problem:** Existing stable code repeatedly modified for new variations.

**Question:** "Can I add this behavior without changing stable logic?"

---

### LSP
**Problem:** Subtype cannot safely replace its abstraction.

**Question:** "Can clients use this implementation exactly as they use the abstraction?"

---

### ISP
**Problem:** Clients depend on methods they don't need.

**Question:** "Does this client need the whole interface?"

---

### DIP
**Problem:** High-level business logic depends directly on implementation details.

**Question:** "Can I depend on an abstraction instead of infrastructure details?"

---

# 16. SENIOR INTERVIEW ANSWER: "EXPLAIN SOLID IN ONE MINUTE"

> "SOLID is a group of object-oriented design principles that help control coupling and make code easier to change and test. SRP says a class should have a focused responsibility and one primary reason to change. OCP says stable behavior should be protected from frequent modifications by allowing new behavior through extension. LSP ensures implementations can safely substitute for their abstractions without violating client expectations. ISP says clients should depend only on the contracts they actually need. DIP says high-level business logic should depend on abstractions rather than low-level implementation details. In practice I use these as design guidelines rather than rigid rules, and I combine them with composition, dependency injection, and patterns such as Strategy and Factory where they genuinely reduce complexity."

---

# 17. FINAL INTERVIEW CHECKLIST

## Single Responsibility
- [ ] One reason to change
- [ ] Cohesion
- [ ] Identify multiple actors/reasons for change
- [ ] Avoid giant service classes
- [ ] Don't over-split classes

## Open/Closed
- [ ] Extension vs modification
- [ ] Polymorphism
- [ ] Strategy
- [ ] Factory/Registry
- [ ] Conditional logic
- [ ] Don't blindly eliminate switches

## Liskov Substitution
- [ ] Behavioral substitutability
- [ ] Contracts
- [ ] Preconditions/postconditions
- [ ] Unsupported operations
- [ ] Rectangle/Square example
- [ ] Avoid invalid inheritance

## Interface Segregation
- [ ] Focused interfaces
- [ ] Client-specific contracts
- [ ] Fat-interface smell
- [ ] UnsupportedOperationException smell
- [ ] Don't create one-method interfaces unnecessarily

## Dependency Inversion
- [ ] High-level vs low-level modules
- [ ] Abstractions
- [ ] Dependency Injection
- [ ] Spring constructor injection
- [ ] Testability
- [ ] Infrastructure isolation

## Senior-level thinking
- [ ] SOLID is not a checklist
- [ ] Design for expected change
- [ ] Prefer composition where appropriate
- [ ] Avoid unnecessary abstractions
- [ ] Explain trade-offs
- [ ] Connect SOLID to design patterns
- [ ] Connect SOLID to Spring Boot
- [ ] Use production scenarios
- [ ] Explain when NOT to apply a principle

---

# FINAL TAKEAWAY

The strongest senior-level answer is not:

> "SRP means one class should do one thing."

It is:

> "SRP is about giving a component a cohesive responsibility and limiting its reasons to change. If business rules, persistence, notification, and presentation change independently, putting them in one class creates coupling and regression risk. I would separate those responsibilities, but I would avoid over-fragmenting the code if the behaviors naturally belong together."

Apply the same reasoning to all five principles:

```text
SOLID
  |
  +-- SRP → Responsibility
  |
  +-- OCP → Change / Extension
  |
  +-- LSP → Behavioral Substitution
  |
  +-- ISP → Focused Contracts
  |
  +-- DIP → Dependency Direction
```

For an **8+ YOE Java/Spring Boot interview**, always connect the principle to:

**problem → design smell → refactoring → trade-off → production use case.**
