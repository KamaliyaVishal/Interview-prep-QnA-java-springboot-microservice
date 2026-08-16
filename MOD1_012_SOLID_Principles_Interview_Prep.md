# 011. SOLID Principles — Interview Questions & Answers 

**Focus:** SRP, OCP, LSP, ISP, and the missing fifth SOLID principle—DIP—plus practical Java/Spring Boot examples, violations, refactoring approaches, trade-offs, and senior-level interview scenarios.

> **Interview goal:** Do not memorize only the definitions. Be able to explain **what problem each principle solves, what bad design looks like, how to refactor it, and what trade-offs the principle introduces.**

---

# 1. What are SOLID principles?

**Answer:** SOLID is a set of five object-oriented design principles that make software easier to maintain, test, extend, and change safely, while keeping components loosely coupled.

SOLID stands for:

| Letter | Principle | Main problem it addresses |
|---|---|---|
| S | Single Responsibility Principle | Too many reasons for a class to change |
| O | Open/Closed Principle | Changes require modifying stable code |
| L | Liskov Substitution Principle | Subtypes cannot safely replace their base types |
| I | Interface Segregation Principle | Clients depend on methods they do not need |
| D | Dependency Inversion Principle | High-level code depends directly on low-level details |

SOLID isn't about creating more interfaces and classes — it's about controlling **coupling, change, responsibility, and dependency direction**.

---

# 2. Why are SOLID principles important in real projects?

**Answer:** Without good design principles, a small requirement change can ripple through a codebase — you modify a large class, break unrelated behavior, update many tests, and take on regression risk. SOLID tries to localize change:

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

vs.

```text
Requirement
   ↓
Small focused component
   ↓
Existing stable behavior remains unchanged
```

For a senior developer, the real question isn't "did I use all five principles?" — it's "does the design make the expected changes easier and safer?"

---

# 3. SINGLE RESPONSIBILITY PRINCIPLE (SRP)

## Concept

### What problem does SRP solve?

SRP addresses classes that bundle multiple unrelated responsibilities together. A common misconception is "a class should have only one method" — that's not SRP. The better definition: **a class should have one reason to change**, where a "reason to change" usually maps to a responsibility owned by a particular actor, business concern, or requirement.

---

## Q1. What is Single Responsibility Principle?

**Answer:** A class should have a focused responsibility and therefore a limited set of reasons to change. Consider this bad example:

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

This class mixes several independent concerns:

```text
InvoiceService
 ├── Business calculation
 ├── Persistence
 ├── PDF generation
 └── Email notification
```

A better design splits these out:

```text
InvoiceCalculator
InvoiceRepository
InvoicePdfGenerator
InvoiceNotificationService
```

---

## Q2. What does "one reason to change" actually mean?

**Answer:** Take this class:

```java
class InvoiceService {
    calculateInvoice();
    saveInvoice();
    generatePdf();
}
```

Different stakeholders drive different changes here — the business team changes calculation rules, the database team changes persistence, the document team changes PDF format. That means the class has multiple reasons to change, and SRP asks us to separate those independent axes of change. It's fundamentally about cohesion and change boundaries, not the number of methods in a class.

---

## Q3. How do you identify an SRP violation?

**Answer:** I look for large classes, unrelated fields, unrelated dependencies, methods serving different business concerns, frequent changes to the same class for unrelated requirements, difficult unit testing, and class names with `And`, `Manager`, `Processor`, `Service` covering many unrelated jobs. A typical smell:

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

**Answer:** No — over-splitting is just as harmful as under-splitting. This is unnecessary fragmentation:

```text
UserNameValidator
UserEmailValidator
UserAgeValidator
UserCountryValidator
```

if those validations are tightly coupled and naturally belong together. The goal is **high cohesion**, not minimum class size.

---

## Q5. How would you refactor an SRP violation?

**Answer:** First identify the independent responsibilities. Given:

```java
class OrderService {
    createOrder();
    calculatePrice();
    saveOrder();
    sendEmail();
}
```

I'd extract collaborators:

```java
class OrderService {
    private final PriceCalculator priceCalculator;
    private final OrderRepository repository;
    private final NotificationService notificationService;
}
```

Now `OrderService` coordinates the use case, `PriceCalculator` owns the business calculation, `OrderRepository` owns persistence, and `NotificationService` owns notification. This also makes each piece independently testable.

---

# 4. OPEN/CLOSED PRINCIPLE (OCP)

## Concept

### What problem does OCP solve?

OCP addresses code that must be repeatedly modified whenever new behavior is introduced. A typical smell:

```java
if (type.equals("CARD")) {
    ...
} else if (type.equals("UPI")) {
    ...
} else if (type.equals("BANK")) {
    ...
}
```

Every new payment type forces a change to this existing class. OCP encourages a design where new behavior is added through new implementations rather than repeatedly editing stable code.

---

## Q6. What is Open/Closed Principle?

**Answer:** Software entities should be **open for extension but closed for modification** — new behavior can be added, but stable existing code shouldn't need repeated changes for every new variation. For example:

```java
interface PaymentProcessor {
    void process(Payment payment);
}
```

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

Adding a new type:

```java
class WalletPaymentProcessor implements PaymentProcessor {
    public void process(Payment payment) {
        // wallet logic
    }
}
```

doesn't require touching the existing processor implementations at all.

---

## Q7. Does OCP mean we should never modify existing code?

**Answer:** No, that's not realistic. OCP means expected variations should be designed so they can be introduced with minimal modification to stable code. If a business requirement changes the existing business rule itself, modifying that code is completely normal. OCP protects stable abstractions from recurring extensions — it's not about making every class immutable or unmodifiable.

---

## Q8. How does Strategy help implement OCP?

**Answer:** Given:

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

adding a discount type means modifying this class. With Strategy:

```java
interface DiscountStrategy {
    double calculate(double amount);
}

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

New behavior becomes a new implementation, not a code change. This combination — OCP + Strategy + Dependency Injection — is one of the most common patterns in enterprise Java.

---

## Q9. Is a switch statement always an OCP violation?

**Answer:** No, a small, stable switch is often perfectly reasonable:

```java
switch (day) {
    case MONDAY -> ...
    case TUESDAY -> ...
}
```

The real question is whether the conditional represents expected, frequently growing variation. I wouldn't replace every switch with Strategy or Factory — I'd introduce an abstraction only when the variation is real and likely to evolve.

---

## Q10. What is the relationship between OCP and polymorphism?

**Answer:** Polymorphism lets behavior vary behind an abstraction:

```text
PaymentProcessor
       |
       +-- CardProcessor
       +-- UpiProcessor
       +-- WalletProcessor
```

The caller depends on `PaymentProcessor`, not the concrete implementations, so new implementations can be added without touching the consumer — that's the mechanism OCP relies on.

---

# 5. LISKOV SUBSTITUTION PRINCIPLE (LSP)

## Concept

### What problem does LSP solve?

LSP addresses inheritance where a subclass technically extends a parent class but can't actually behave as a valid replacement for it. The key question: **can code written for the parent safely use the child without unexpected behavior?**

---

## Q11. What is Liskov Substitution Principle?

**Answer:** Objects of a subtype should be usable wherever objects of the base type are expected, without breaking the correctness of the program. Here's a bad hierarchy:

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

The parent contract implies `bird.fly()` is always valid, but:

```java
Bird bird = new Penguin();
bird.fly();
```

fails at runtime — the subtype violates the expected contract.

---

## Q12. Is LSP only about inheritance?

**Answer:** It's most visible with inheritance, but the deeper idea is **behavioral substitutability**, and it applies just as much to interfaces, implementations, and API contracts. If an implementation violates what its interface promises, substitutability breaks regardless of whether inheritance is involved.

---

## Q13. What are common signs of an LSP violation?

**Answer:** I look for a subclass throwing `UnsupportedOperationException` for behavior the parent promises, a subclass weakening guarantees, unexpectedly rejecting valid parent inputs, changing important semantics, or callers needing `instanceof` checks or special-casing for a particular implementation:

```java
if (shape instanceof Rectangle) {
    ...
}
```

```java
if (service instanceof SpecialService) {
    ...
}
```

Either of these usually means the abstraction itself is wrong.

---

## Q14. What is the classic Rectangle/Square LSP problem?

**Answer:** Given:

```java
class Rectangle {
    void setWidth(int width) { ... }
    void setHeight(int height) { ... }
}

class Square extends Rectangle {
    @Override
    void setWidth(int width) {
        super.setWidth(width);
        super.setHeight(width);
    }
}
```

A client expecting width and height to change independently breaks when handed a `Square`. The mathematical "is-a" relationship doesn't automatically translate into a valid behavioral subtype — inheritance should be based on **behavioral contracts**, not conceptual relationships.

---

## Q15. How do you fix an LSP violation?

**Answer:** Redesign the abstraction around behavior every implementation can genuinely support. Instead of putting `fly()` in a shared `Bird` base class:

```java
Bird
 ├── Sparrow
 └── Penguin
```

split flight into its own contract:

```java
interface Flyable {
    void fly();
}
```

so only birds that can actually fly implement `Flyable`.

---

## Q16. What does LSP imply about exceptions?

**Answer:** A subtype shouldn't unexpectedly tighten the contract by rejecting inputs the base type promises to accept. If the base abstraction says an operation succeeds for a valid input, a subtype shouldn't arbitrarily reject that same input — LSP is fundamentally about preserving the behavioral contract clients rely on: valid inputs, outputs, side effects, and failure behavior.

---

# 6. INTERFACE SEGREGATION PRINCIPLE (ISP)

## Concept

### What problem does ISP solve?

ISP addresses "fat interfaces" that force clients to depend on methods they don't use. Bad design:

```java
interface Worker {
    void work();
    void eat();
    void sleep();
}
```

A robot implementation:

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

Throwing `UnsupportedOperationException` for methods you're forced to implement is a strong signal the interface is too broad.

---

## Q17. What is Interface Segregation Principle?

**Answer:** Clients shouldn't be forced to depend on methods they don't use. Instead of one large interface:

```java
interface Worker {
    void work();
    void eat();
    void sleep();
}
```

split it into focused contracts:

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

Now a robot only needs to implement what applies to it:

```java
class Robot implements Workable {
    public void work() {}
}
```

---

## Q18. Why is ISP useful in microservices and enterprise applications?

**Answer:** Large interfaces create unnecessary coupling. Take:

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

Different clients typically need only a small subset of this. Focused interfaces make testing easier, implementations simpler, dependencies clearer, and API evolution safer.

---

## Q19. ISP vs SRP — what is the difference?

**Answer:** They're related but focus on different things. SRP asks how many reasons a class has to change; ISP asks how many unrelated methods clients are forced to depend on.

```text
SRP → class responsibility

ISP → interface/client dependency
```

---

## Q20. Does ISP mean every interface should contain exactly one method?

**Answer:** No, that leads to unnecessary fragmentation. A cohesive interface can contain multiple methods when they naturally belong to the same client responsibility — the goal is **client-specific, cohesive contracts**, not "one method per interface."

---

# 7. DEPENDENCY INVERSION PRINCIPLE (DIP)

> **Important:** DIP is the fifth SOLID principle and should be included even though it was not explicitly listed in the module outline.

## Concept

### What problem does DIP solve?

High-level business logic shouldn't be tightly coupled to low-level implementation details. Bad:

```java
class OrderService {

    private final MySqlOrderRepository repository =
        new MySqlOrderRepository();
}
```

Now the business logic directly knows about the database implementation. Better:

```java
interface OrderRepository {
    void save(Order order);
}

class OrderService {

    private final OrderRepository repository;

    OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}

class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) {
        // MySQL
    }
}
```

Dependency direction now points toward the abstraction:

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

**Answer:** DIP has two parts: high-level modules shouldn't depend directly on low-level modules — both should depend on abstractions — and abstractions shouldn't depend on details; details should depend on abstractions.

```text
OrderService
     |
     v
OrderRepository
     ^
     |
MySqlOrderRepository
```

The business service depends on `OrderRepository`, the abstraction — not on `MySqlOrderRepository` directly.

---

## Q22. Is Dependency Inversion the same as Dependency Injection?

**Answer:** No. Dependency Inversion is a design **principle** — depend on abstractions. Dependency Injection is a **technique** for supplying those dependencies. Spring gives you DI, and DI is one of the main ways you actually achieve DIP in practice.

---

## Q23. Why is DIP important for testing?

**Answer:** Without DIP:

```java
OrderService
    ↓
new MySqlRepository()
```

unit tests may need a real database. With DIP, `OrderService` depends on `OrderRepository`, so tests can supply a mock or fake implementation instead — reducing infrastructure coupling and making tests fast and isolated.

---

# 8. HOW THE FIVE PRINCIPLES WORK TOGETHER

**Answer:** Consider a payment system. Bad design:

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

This single class has an SRP violation (too many responsibilities), OCP pressure (new payment types force edits), a DIP violation (direct infrastructure coupling), and it's hard to test. A better design:

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

Here SRP separates responsibilities, OCP lets me add payment processors without touching the stable service, LSP means every processor honors the `PaymentProcessor` contract, ISP keeps interfaces focused, and DIP means `PaymentService` depends on abstractions rather than concrete classes.

---

# 9. COMMON INTERVIEW QUESTIONS

## Q24. What is the difference between SRP and OCP?

**Answer:** SRP controls responsibility — a class should have one primary reason to change. OCP controls extensibility — expected new variations should be addable without repeatedly modifying stable code.

```text
SRP → "Who should own this behavior?"

OCP → "How can I add another variation safely?"
```

---

## Q25. OCP vs DIP?

**Answer:** They often work together. OCP is about designing for extension; DIP is about making dependencies point toward abstractions.

```text
PaymentService
      |
   interface
      ^
      |
CardPayment
```

DIP creates the abstraction boundary, and OCP is what lets new implementations be added behind it.

---

## Q26. SRP vs ISP?

**Answer:**

```text
SRP → class responsibility

ISP → interface/client dependency
```

SRP asks whether a class has unrelated reasons to change; ISP asks whether a client is forced to depend on methods it doesn't need.

---

## Q27. LSP vs ISP?

**Answer:** LSP asks whether an implementation can safely substitute for its abstraction. ISP asks whether the abstraction itself is too broad for its clients. They can interact — a fat interface that implementations can't meaningfully support tends to produce LSP problems too, so ISP can help prevent certain LSP violations.

```text
Fat Interface
     ↓
Implementations cannot meaningfully support all methods
     ↓
LSP problems
```

---

## Q28. Which SOLID principle is most important?

**Answer:** There's no universally "most important" one. SRP builds cohesive components, OCP manages expected variation, LSP protects abstraction correctness, ISP keeps contracts focused, and DIP reduces coupling. In enterprise applications, **DIP + SRP + OCP** tend to be the most visible because they strongly affect testability and changeability.

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

**Answer:** Instead of:

```java
class OrderService {
    private MySqlOrderRepository repository =
        new MySqlOrderRepository();
}
```

Spring lets me depend on an abstraction:

```java
interface OrderRepository {
    void save(Order order);
}

@Repository
class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) {
        // ...
    }
}

@Service
class OrderService {

    private final OrderRepository repository;

    OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

The service depends on the abstraction, and Spring wires in the concrete bean at runtime.

---

## Q30. How does Spring help with OCP?

**Answer:** A common approach is multiple implementations behind one interface:

```java
interface NotificationSender {
    void send(Notification notification);
}

@Component
class EmailNotificationSender implements NotificationSender {
    public void send(Notification notification) {}
}

@Component
class SmsNotificationSender implements NotificationSender {
    public void send(Notification notification) {}
}
```

New notification mechanisms are added as new implementations, and the core abstraction stays stable.

---

# 12. SENIOR PRODUCTION SCENARIOS

## Q31. You have a 2,000-line service class. Which SOLID principle is probably being violated?

**Answer:** Most likely SRP, but I wouldn't conclude that from line count alone. I'd look at the number of responsibilities, dependency count, unrelated methods, reasons for change, and test complexity. A large class can still be cohesive, but 2,000 lines is a strong smell worth investigating.

---

## Q32. A new payment type requires changing five existing classes. Which principle should you investigate?

**Answer:** Likely OCP. I'd look for switch/if chains, concrete-type checks, and duplicated type-based logic, then consider Strategy, a Factory/Registry, polymorphism, and dependency injection to remove the need for repeated edits.

---

## Q33. A subclass throws UnsupportedOperationException for a parent method. Which principle?

**Answer:** Likely LSP — the subclass can't safely substitute for the parent abstraction. I'd reconsider the hierarchy or split the abstraction so implementations only commit to behavior they can actually support.

---

## Q34. An interface has 25 methods and most implementations use only 5. Which principle?

**Answer:** Likely ISP. I'd split the interface around meaningful client responsibilities instead of keeping one huge contract.

---

## Q35. A service directly creates repositories, HTTP clients and database clients with `new`. Which principle?

**Answer:** Likely DIP. I'd move those dependencies behind abstractions and inject them, which improves testability, configurability, replaceability of implementations, and separation of business logic from infrastructure.

---

## Q36. Can a class violate more than one SOLID principle?

**Answer:** Absolutely. For example:

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

This hits SRP (multiple responsibilities), OCP (new types require modification), and DIP (direct infrastructure dependency) all at once — one design problem can manifest as several SOLID violations simultaneously.

---

# 13. SOLID TRADE-OFFS AND OVERENGINEERING

## Q37. Can following SOLID too strictly make code worse?

**Answer:** Yes. Over-applying it can turn one simple requirement into 10 interfaces, 15 classes, and multiple layers of indirection — that increases cognitive load without real benefit. I treat SOLID as a set of heuristics for managing change and coupling, and apply the level of abstraction the problem and its expected evolution actually justify.

---

## Q38. Should every class depend on an interface?

**Answer:** No — an interface for every class becomes ceremony. An abstraction earns its place when multiple implementations exist or are expected, dependency boundaries matter, external systems need isolation, testing benefits from substitution, or behavior genuinely varies. If there's one stable implementation and no meaningful boundary, a concrete dependency is perfectly fine.

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

For an **Experience Java/Spring Boot interview**, always connect the principle to:

**problem → design smell → refactoring → trade-off → production use case.**
