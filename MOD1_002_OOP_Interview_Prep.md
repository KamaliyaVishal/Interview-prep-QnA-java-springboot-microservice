# Object-Oriented Programming (OOP) — Interview Prep 

---

## SECTION 1: ENCAPSULATION, INHERITANCE, POLYMORPHISM, ABSTRACTION

### Q1. Explain the four pillars of OOP with real examples.
**Answer:**

**Encapsulation** — bundling data (fields) and behavior (methods) together, restricting direct access to internal state via access modifiers, exposing controlled access through public methods (getters/setters, or better — behavior methods).

```java
public class Account {
    private BigDecimal balance;   // hidden internal state
    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) throw new InsufficientFundsException();
        balance = balance.subtract(amount);
    }
}
```
**Senior nuance:** Encapsulation isn't just "make fields private and add getters/setters" — that's often an anti-pattern (anemic domain model). True encapsulation means exposing **behavior**, not raw state — the `Account` class should validate and control *how* balance changes, not just let any caller directly set it.

**Inheritance** — a class (`subclass`) acquires fields/methods of another (`superclass`), modeling an **is-a** relationship. Enables code reuse and polymorphic substitution.
```java
class Vehicle { void start() { ... } }
class Car extends Vehicle { }
```

**Polymorphism** — the ability for the same interface/method call to behave differently depending on the actual runtime object. Two kinds:
- **Compile-time (static) polymorphism** — method overloading, resolved at compile time by signature.
- **Runtime (dynamic) polymorphism** — method overriding, resolved at runtime via the actual object type (dynamic dispatch / virtual method invocation).

**Abstraction** — hiding implementation complexity, exposing only essential behavior through a well-defined contract (abstract classes/interfaces). Answers "what" an object does, not "how."

**Real production example tying all four together:** A `PaymentProcessor` interface (abstraction) with `CreditCardProcessor`/`UpiProcessor`/`WalletProcessor` implementations (inheritance/polymorphism) — each encapsulates its own validation/state — and the calling `OrderService` never needs to know which concrete implementation it's using (polymorphic dispatch via Spring dependency injection).

---

### Q2. What's the difference between abstraction and encapsulation? (Frequently confused, favorite trap)
**Answer:** Abstraction is about **design/what is exposed** (hiding complexity at the interface/contract level — "what can this object do"). Encapsulation is about **implementation/how it's protected** (hiding internal state via access modifiers — "how is this object's data protected from misuse").

**Analogy that works well in interviews:** A car's steering wheel/pedals are *abstraction* — you don't need to know engine internals to drive. The engine being sealed under the hood, inaccessible directly, is *encapsulation*. Both work together but solve different problems: abstraction reduces complexity for the consumer; encapsulation protects integrity of internal state.

---

### Q3. Runtime polymorphism internal working — how does the JVM know which overridden method to call?
**Answer:** Via the **vtable (virtual method table)** mechanism. Each class has a vtable mapping method signatures to the actual method implementation to invoke. At runtime, the JVM looks up the method via the **actual object's class** (not the reference's declared type) — this is called **dynamic method dispatch**.

```java
Animal a = new Dog();
a.makeSound();   // calls Dog's makeSound(), NOT Animal's — resolved at RUNTIME via vtable, based on actual object type
```

**Follow-up:** "Is method overloading resolved at compile time or runtime?" → Compile time — the compiler picks the overload based on the **declared (static) type** and argument types at the call site, not the actual runtime object. This is the core distinction interviewers test between overloading and overriding resolution.

---

### Q4. Can constructors be polymorphic / overridden?
**Answer:** No — constructors are **not inherited and cannot be overridden** (they're tied to the specific class, not part of the vtable mechanism). Constructors **can be overloaded** within the same class. This is a common trick question: "Is constructor polymorphism a thing?" — Answer firmly no, and explain why (constructors aren't inherited members at all).

---

## SECTION 2: ABSTRACT CLASSES vs INTERFACES

### Q5. Abstract class vs Interface — full comparison (extremely high-frequency question)
| Aspect | Abstract Class | Interface |
|---|---|---|
| Multiple inheritance | No (single inheritance only) | Yes (a class can implement multiple interfaces) |
| Fields | Can have instance fields (any access modifier, mutable) | Only `public static final` constants |
| Constructors | Yes | No |
| Method implementation | Can mix abstract + concrete methods | Java 8+: can have `default`/`static` methods with bodies |
| Access modifiers on methods | Any (public/protected/private) | `public` by default (implicitly), `private` methods allowed since Java 9 |
| When to use | Shared state + partial implementation, "is-a" with common base behavior | Contract/capability definition, "can-do" behavior, especially across unrelated class hierarchies |

**Senior-level decision framework (this is what interviewers actually want to hear, not just the table):**
> "I use an **abstract class** when subclasses share common state and some common implementation logic — e.g., a `BaseRepository` with a shared `EntityManager` field and template-method-style CRUD logic. I use an **interface** when I'm defining a contract/capability that unrelated classes might implement — e.g., `Comparable`, `Serializable`, or a `NotificationSender` interface implemented by `EmailSender`, `SmsSender`, `PushNotificationSender`, which have no shared state or inheritance relationship, just a shared capability."

---

### Q6. Since Java 8 added default/static methods to interfaces, why do we still need abstract classes?
**Answer:** Interfaces still **cannot hold instance state** (no non-static, non-final fields) and **cannot have constructors** — so any implementation that needs to maintain and initialize shared mutable state across a family of subclasses still requires an abstract class. Also, a class can extend only **one** abstract class but implement **many** interfaces — abstract classes preserve the classic single-inheritance state-sharing model interfaces were never meant to replace.

**Follow-up:** "What problem do default methods solve?" → **Interface evolution without breaking existing implementers.** Before Java 8, adding a new method to a widely-implemented interface (e.g., `Collection`) would break every existing class that implements it. `default` methods let you add new methods with a default implementation, so old implementers keep compiling — this is literally why `Stream`-related default methods were added to `Collection`/`Iterable` in Java 8 without breaking the entire ecosystem.

---

### Q7. What happens if a class implements two interfaces with the same default method signature? (Diamond problem)
```java
interface A { default void greet() { System.out.println("A"); } }
interface B { default void greet() { System.out.println("B"); } }
class C implements A, B {
    // MUST override — compiler forces resolution, won't guess
    public void greet() {
        A.super.greet();   // explicitly choosing which interface's default to invoke
    }
}
```
**Answer:** The compiler **forces an explicit override** in the implementing class — Java refuses to silently pick one, avoiding the classic C++ diamond-inheritance ambiguity. You can call a specific interface's default implementation using `InterfaceName.super.methodName()`.

---

### Q8. Can an interface extend multiple interfaces? Can an abstract class implement an interface without implementing its methods?
**Answer:** Yes to both.
- An interface **can extend multiple interfaces** (`interface C extends A, B {}`) — this is allowed because interfaces don't carry implementation/state conflicts the same way classes do.
- An abstract class **can implement an interface without providing method bodies** — it can leave the methods abstract, deferring the obligation to whichever concrete subclass eventually implements it. Only a concrete (non-abstract) class **must** provide implementations for all inherited abstract methods.

---

### Q9. Marker interfaces (e.g., `Serializable`) — what are they, and are they still relevant given annotations exist?
**Answer:** A marker interface has **no methods** — it just "tags" a class with metadata the JVM/framework checks via `instanceof` (e.g., `Serializable`, `Cloneable`, `Remote`). Modern Java favors **annotations** for metadata (more flexible — can carry parameters, apply to fields/methods, processed via reflection or annotation processors) over marker interfaces for *new* APIs, but legacy core JDK APIs (serialization, cloning) still use the marker-interface pattern because changing them would break backward compatibility across the entire ecosystem.

---

## SECTION 3: METHOD OVERLOADING vs OVERRIDING

### Q10. Overloading vs Overriding — full comparison
| Aspect | Overloading | Overriding |
|---|---|---|
| Definition | Same method name, different parameter list, same class (or subclass adding new signatures) | Same method name + signature, subclass redefines superclass behavior |
| Resolution | Compile-time (static binding) | Runtime (dynamic binding / dynamic dispatch) |
| Return type | Can differ freely | Must be same or covariant (Section 6) |
| Access modifier | Can differ freely | Cannot be more restrictive than superclass method |
| Exceptions | Can throw different/new exceptions | Can't throw new/broader checked exceptions than the overridden method |
| `static`/`private`/`final` methods | Can be overloaded | **Cannot be overridden** (can be hidden for static, see Q13) |

---

### Q11. Rules for valid method overloading
- Must differ in **number, type, or order of parameters**.
- Return type alone is **not sufficient** to overload (compile error — return type isn't part of the method signature for overload resolution purposes).
- Overload resolution at compile time follows a **priority order**: (1) exact match → (2) widening primitive conversion → (3) autoboxing → (4) varargs (lowest priority) — a favorite tricky follow-up.

```java
void print(int x) { }
void print(long x) { }
void print(Integer x) { }
void print(int... x) { }

print(5);   // calls print(int) — exact match wins over widening, boxing, and varargs
```

**Follow-up trap:** "If you remove `print(int)`, which overload does `print(5)` call now?" → `print(long x)` — widening (int→long) is preferred over autoboxing (int→Integer), which is preferred over varargs. This exact ordering question is asked at product companies to test deep JLS understanding.

---

### Q12. Rules for valid method overriding
- Same method signature (name + parameter types) as superclass.
- Return type must be same or **covariant** (Section 6).
- Access modifier can be **same or less restrictive**, never more restrictive (can't override `public` with `protected`).
- Cannot throw new or broader **checked** exceptions than the overridden method (can throw fewer/narrower, or any unchecked exception).
- Method must not be `static`, `private`, or `final` in the superclass (those aren't override-eligible — see Q13).

```java
class Parent {
    protected Object process() throws IOException { ... }
}
class Child extends Parent {
    @Override
    public String process() throws FileNotFoundException {  // ✓ wider access, covariant return, narrower exception
        return "done";
    }
}
```

---

### Q13. Can you override a static method? What actually happens if you try?
**Answer: No — static methods are NOT polymorphic. Attempting to redefine one in a subclass is method HIDING, not overriding.**

```java
class Parent {
    static void greet() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void greet() { System.out.println("Child"); }   // hides, doesn't override
}

Parent p = new Child();
p.greet();   // prints "Parent" — resolved by REFERENCE TYPE (static binding), not object type!
```
**Why:** Static methods belong to the **class**, resolved at **compile time** based on the reference's declared type — there's no vtable/dynamic dispatch involved. This is one of the most commonly asked "gotcha" questions precisely because the output (`"Parent"`) surprises people who assume all method calls are polymorphic.

**Follow-up:** "What about instance methods called on a null reference cast to a type — does it still resolve statically for static methods?" → Yes — since static method resolution never actually needs the object (no dynamic dispatch), `((Parent) null).greet()` works fine and doesn't throw NPE, because it's resolved purely by compile-time type, never touching the object at runtime.

---

### Q14. What is method overloading resolution ambiguity, and when does the compiler reject it?
```java
void process(int x, long y) { }
void process(long x, int y) { }

process(5, 5);   // COMPILE ERROR — ambiguous, both need one widening conversion, compiler can't pick
```
**Answer:** When multiple overloads are equally applicable after the standard conversion rules (Q11) and none is strictly more specific, the compiler throws an **ambiguous method call** compile-time error rather than guessing — important to know this fails at compile time, not runtime.

---

## SECTION 4: CONSTRUCTORS & INITIALIZATION BLOCKS

### Q15. Constructor chaining — this() and super()
**Answer:** `this(...)` calls another constructor **in the same class**; `super(...)` calls the **parent class's constructor**. Either must be the **first statement** in a constructor if used (can't have both, and can't have other code before them).

```java
class Vehicle {
    Vehicle() { System.out.println("Vehicle()"); }
    Vehicle(String name) { this(); System.out.println("Vehicle(String): " + name); }
}
class Car extends Vehicle {
    Car() {
        super("Car");   // must be first statement
        System.out.println("Car()");
    }
}
// new Car() prints: Vehicle() → Vehicle(String): Car → Car()
```

**Follow-up:** "What happens if you don't explicitly call super()?" → Compiler **implicitly inserts `super()`** (no-arg) as the first line — if the parent class has **no** no-arg constructor available, this causes a **compile error**, forcing you to explicitly call the correct parameterized `super(...)`.

---

### Q16. Order of execution: static blocks, instance blocks, constructors — what's the exact sequence?
**Answer (extremely commonly asked, must be memorized precisely):**
1. **Static blocks/static field initializers** — run once, in source order, when the class is **first loaded** (Section 3/Class-loading topic — triggered on first active use).
2. **Instance initializer blocks + instance field initializers** — run in source order, every time an object is created, **before** the constructor body.
3. **Constructor body** — runs last.

For inheritance, the order is: **parent static → child static → parent instance blocks → parent constructor → child instance blocks → child constructor.**

```java
class Parent {
    static { System.out.println("Parent static block"); }
    { System.out.println("Parent instance block"); }
    Parent() { System.out.println("Parent constructor"); }
}
class Child extends Parent {
    static { System.out.println("Child static block"); }
    { System.out.println("Child instance block"); }
    Child() { System.out.println("Child constructor"); }
}
// new Child() prints:
// Parent static block
// Child static block
// Parent instance block
// Parent constructor
// Child instance block
// Child constructor
```
**This exact question with this exact code is asked at nearly every product-company interview** — practice writing it from memory.

---

### Q17. What's the purpose of an instance initializer block if you already have constructors?
**Answer:** Instance blocks run **before every constructor**, useful for logic shared across **multiple overloaded constructors** without duplicating code in each one, or for initializing anonymous class instances (which can't have named constructors at all — instance blocks are the *only* way to run init logic in an anonymous class). In practice, rarely used in modern code (constructors or field initializers cover most needs), but worth knowing for completeness and anonymous-class edge cases.

---

### Q18. Can a constructor call an overridable instance method? What's the danger?
```java
class Parent {
    Parent() { init(); }         // calling overridable method from constructor — DANGEROUS
    void init() { System.out.println("Parent init"); }
}
class Child extends Parent {
    private String name = "Child";
    @Override
    void init() {
        System.out.println("Child init, name=" + name);   // name is still NULL here!
    }
}
// new Child() prints: "Child init, name=null" — because Parent's constructor runs BEFORE Child's field initializers
```
**Answer:** This is a well-known anti-pattern. Because the **parent constructor runs before the child's field initializers**, calling an overridable method from the parent constructor invokes the **child's overridden version**, but the child's fields **aren't initialized yet** — silent, hard-to-debug bugs (nulls/zeros where you expect real values). **Best practice: never call overridable (non-private, non-final) instance methods from a constructor** — use `private` or `final` methods for any constructor-invoked logic instead.

---

## SECTION 5: static, final, this, super KEYWORDS

### Q19. static keyword — all its uses
**Answer:**
1. **Static variable** — one copy shared across all instances (class-level state).
2. **Static method** — belongs to class, no `this`, can't access instance members directly, resolved at compile time (no polymorphism, per Q13).
3. **Static block** — runs once at class loading, for static field initialization logic too complex for a single expression.
4. **Static nested class** — doesn't hold an implicit reference to the outer instance (unlike inner classes), so it can be instantiated independently: `new Outer.StaticNested()`.
5. **Static import** — imports static members directly (e.g., `import static java.lang.Math.PI;`), rarely used except for readability in test assertions/DSLs.

**Common production use case:** Utility classes (`Collections`, `Math`-style helper classes) with all-static methods and a private constructor to prevent instantiation.

---

### Q20. final keyword — all its uses, and why is immutability valuable in concurrent code?
1. **final variable** — value/reference can't be reassigned after initialization (for objects, the reference is fixed, but the object's internal state **can still mutate** unless the object itself is designed as immutable — a very common confusion point).
2. **final method** — cannot be overridden by subclasses.
3. **final class** — cannot be subclassed at all (e.g., `String`, `Integer` are final).

```java
final List<String> list = new ArrayList<>();
list.add("hello");     // ✓ fine — the LIST OBJECT is mutable, only the reference "list" is final
list = new ArrayList<>();  // ✗ compile error — can't reassign
```

**Senior tie-in to concurrency (ties back to Multithreading prep):** `final` fields participate in special JMM guarantees — a properly constructed object with all-`final` fields is guaranteed to be **safely published** (fully visible to other threads) without extra synchronization, **as long as the reference doesn't escape the constructor before it completes**. This is exactly why immutable classes (all fields `final`, no setters, defensive copies in constructor) are inherently thread-safe without needing locks — a strong link between OOP fundamentals and concurrency that senior interviewers love to see you connect.

---

### Q21. this vs super keywords — all uses
**`this`:**
- Refers to the current object instance.
- Disambiguates instance fields from constructor/method parameters with the same name (`this.name = name;`).
- Calls another constructor in the same class (`this(...)`, must be first statement).
- Can be passed as an argument to indicate "this specific object" (e.g., registering `this` as a listener).

**`super`:**
- Refers to the immediate parent class.
- Accesses a parent's field/method that's been hidden/overridden (`super.fieldName`, `super.methodName()`).
- Calls the parent's constructor (`super(...)`, must be first statement).

**Follow-up trap:** "Can you use `this` inside a static method?" → No — compile error. `this` refers to an instance, and static methods aren't tied to any instance (ties back to Q19/Q13 — static context has no object).

---

### Q22. Difference between this() and super() — can both appear in the same constructor?
**Answer: No — only one of `this(...)` or `super(...)` can appear in a constructor, and it must be the first line.** Logically this makes sense: if `this(...)` delegates to another constructor in the same class, that *other* constructor will itself call `super(...)` (explicitly or implicitly) — having both in the same constructor would mean calling the parent constructor twice, which the JVM disallows.

---

## SECTION 6: COVARIANT RETURN TYPES

### Q23. What are covariant return types? Why were they introduced?
**Answer:** Since Java 5, an overriding method can return a **subtype** of the return type declared in the overridden (superclass) method, instead of requiring an exact type match.

```java
class Animal {
    Animal reproduce() { return new Animal(); }
}
class Dog extends Animal {
    @Override
    Dog reproduce() { return new Dog(); }   // covariant return — Dog is a subtype of Animal, this is VALID
}
```

**Why introduced:** Before Java 5, overriding methods **had to return the exact same type**, forcing callers to manually downcast even when they knew the more specific type was guaranteed:
```java
Animal offspring = dog.reproduce();
Dog puppy = (Dog) offspring;   // ugly, error-prone manual cast — needed pre-Java-5
```
Covariant return types eliminate this unnecessary cast, improving type safety and API ergonomics — most commonly seen in the **`clone()` method override pattern** (Section 7) and in fluent builder APIs.

**Real-world example:** `Object.clone()` returns `Object`; a well-designed override returns the specific subtype directly:
```java
@Override
public Employee clone() {   // covariant — narrower than Object, no cast needed by caller
    return (Employee) super.clone();
}
```

**Follow-up:** "Does this work for method parameters too (contravariance)?" → **No** — Java does **not** support covariant/contravariant *parameter* types for overriding; parameter types must match exactly, or it's treated as **overloading**, not overriding, an important distinction interviewers test.

---

## SECTION 7: OBJECT CLONING (Shallow vs Deep Copy)

### Q24. How does object cloning work in Java? What is the Cloneable interface?
**Answer:** `Object.clone()` is a `protected native` method that performs a **field-by-field copy** of the object. To use it externally, a class must:
1. Implement the `Cloneable` **marker interface** (no methods — just a flag; without it, calling `clone()` throws `CloneNotSupportedException`).
2. Override `clone()` as `public`, typically calling `super.clone()` internally.

```java
class Employee implements Cloneable {
    private String name;
    private Address address;   // reference type — this matters for shallow vs deep!

    @Override
    public Employee clone() throws CloneNotSupportedException {
        return (Employee) super.clone();   // shallow copy by default
    }
}
```

**Follow-up:** "Why is `clone()` widely considered a design flaw in Java?" → It's **broken by design**: `Cloneable` has no methods (doesn't actually declare `clone()`), the mechanism relies on a fragile marker-interface + native-method combo, it bypasses constructors entirely (can violate invariants), and checked-exception handling (`CloneNotSupportedException`) is clunky. **Joshua Bloch (Effective Java) explicitly recommends avoiding `Cloneable`/`clone()`** in favor of **copy constructors or static factory methods** for object copying — a very strong senior-level answer to volunteer.

---

### Q25. Shallow copy vs Deep copy — explain with example, and how do you implement deep copy?
**Answer:**
- **Shallow copy** (`super.clone()` default behavior): copies primitive fields by value, but for **reference fields, copies only the reference** — both original and clone point to the **same underlying object**. Mutating a shared nested object through either reference affects both.
- **Deep copy:** recursively clones referenced objects too, so the clone is **fully independent** of the original — no shared mutable state at any level.

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}
class Employee implements Cloneable {
    String name;
    Address address;

    // SHALLOW clone (default)
    @Override
    public Employee clone() throws CloneNotSupportedException {
        return (Employee) super.clone();   // address reference is SHARED between original and clone
    }

    // DEEP clone (manual)
    public Employee deepClone() {
        Employee copy = new Employee();
        copy.name = this.name;
        copy.address = new Address(this.address.city);   // new independent Address object
        return copy;
    }
}

Employee e1 = new Employee();
e1.address = new Address("NYC");
Employee e2 = e1.clone();          // shallow
e2.address.city = "LA";
System.out.println(e1.address.city);   // "LA" — BUG! e1 was unintentionally affected too, because shallow copy shared the Address reference
```

**Ways to implement deep copy (senior should know multiple approaches and trade-offs):**
1. **Manual recursive cloning** — explicit, most control, most boilerplate (shown above).
2. **Copy constructor** — `new Employee(existingEmployee)` that internally deep-copies mutable fields — **Joshua Bloch's recommended approach**, avoids `Cloneable` entirely.
3. **Serialization-based deep copy** (serialize to byte stream, deserialize into a new object graph) — works generically for any `Serializable` graph but has real performance overhead and requires every referenced type to be `Serializable`; rarely used in modern production code.
4. **Third-party libraries** — Jackson (`objectMapper.readValue(objectMapper.writeValueAsString(obj), MyClass.class)`) or Apache Commons `SerializationUtils.clone()` — pragmatic in practice, but adds a dependency and JSON round-trip overhead.

**Best practice senior answer:** "I generally avoid `Cloneable` entirely per Effective Java's guidance and use copy constructors or static factory methods instead — they're type-safe, don't throw checked exceptions, work with `final` fields (which `clone()` can't set since it bypasses constructors), and make the deep-vs-shallow decision explicit and readable at the call site rather than hidden inside an overridden `clone()`."

---

### Q26. Real production bug caused by unintentional shallow copy — can you describe one?
**Answer (framing you should use in a real interview, tailored to your own experience):**
> "I've seen this bite teams in caching layers — a service caches a `List<OrderDTO>` and returns it directly (or via a shallow `clone()`/copy) to multiple callers. One caller mutates a nested field on an object in that list (e.g., updates `order.getLineItems().add(...)`), and because the 'copy' only copied the top-level list reference structure but shared the nested `OrderLineItem` objects, **every other caller's supposedly independent copy is silently corrupted too.** The fix was either returning fully immutable DTOs, or explicitly deep-copying (via defensive copy in the getter, or an immutable builder pattern) before handing data across a boundary where the caller might mutate it."

This ties Q24-26 directly to the "defensive copying" principle from the Core Java Pass-by-Value discussion — a good callback if asked in the same interview session.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can an abstract class have a constructor if it can never be instantiated directly?" → Yes — it's called via `super()` from a concrete subclass constructor, used to initialize shared fields.
- "Can an interface have a private method? What's it for?" → Yes, since Java 9 — used to share common code **between default methods** within the same interface, without exposing it as part of the public API.
- "Is it possible to overload the main method?" → Yes, you can have multiple `main` methods with different signatures, but the JVM only ever invokes `public static void main(String[] args)` as the entry point; others are just regular overloaded methods you'd have to call explicitly.
- "Why can't static methods be abstract?" → Abstract implies "must be overridden by subclass, resolved dynamically" — but static methods aren't part of dynamic dispatch at all (Q13), so `abstract static` is a contradiction and a compile error.
- "What's the difference between hiding and overriding?" → Overriding = dynamic dispatch, resolved by object's runtime type (applies to instance methods). Hiding = static resolution by reference's declared type (applies to static methods and to fields — fields are NEVER polymorphic in Java, only methods are).
- "Do fields support polymorphism/overriding in Java?" → **No** — fields are resolved by the **declared (reference) type**, always, at compile time. Only methods get dynamic dispatch. This is a subtle but very real trap: `Parent p = new Child(); p.someField` accesses `Parent`'s field even if `Child` redeclares a field with the same name (field hiding, not overriding).

---

*Study tip: The static-block/instance-block/constructor execution order (Q16), the static-method-hiding-vs-overriding gotcha (Q13), and the shallow-vs-deep-copy bug (Q25) are the three most commonly whiteboard-tested OOP questions at the senior level — be ready to write and trace through all three from memory without hesitation.*
