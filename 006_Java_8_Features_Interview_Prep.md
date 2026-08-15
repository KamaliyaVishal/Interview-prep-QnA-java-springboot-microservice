# 6. Java 8 Features — Interview Prep — Senior Java Developer (8+ YOE)

*Covers: Lambda Expressions, Functional Interfaces, Stream API, Optional, Method References, Default/Static Interface Methods, Date/Time API.*

---

## SECTION 1: LAMBDA EXPRESSIONS

### Q1. What are Lambda Expressions? Why were they introduced in Java 8?
**Answer:** A **lambda expression** is a concise way to represent an **anonymous function** — a block of code that can be passed as an argument and executed later. Syntax: `(parameters) -> expression` or `(parameters) -> { statements; }`.

```java
// Before Java 8 — anonymous inner class
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// Java 8 — lambda
Runnable r2 = () -> System.out.println("Hello");
```

**Why introduced:** Java needed first-class support for **functional programming** to:
1. Reduce boilerplate (anonymous inner classes for single-method callbacks).
2. Enable the **Stream API** and **parallel processing** with readable, composable code.
3. Allow **behavior parameterization** — passing logic as data (e.g., sorting/filtering strategies).

**Senior angle:** Lambdas are **not** new threads of execution or new objects in the traditional sense — they're compiled to **`invokedynamic`** + **`LambdaMetafactory`** (not plain anonymous inner classes), which avoids generating a separate `.class` file per lambda and enables JVM optimizations. Mentioning `invokedynamic` signals you understand the bytecode, not just the syntax.

---

### Q2. Explain lambda expression syntax and type inference.
**Answer:**

| Form | Example |
|---|---|
| No params | `() -> System.out.println("Hi")` |
| One param (parens optional) | `x -> x * 2` or `(x) -> x * 2` |
| Multiple params | `(a, b) -> a + b` |
| Block body | `(s) -> { System.out.println(s); return s.length(); }` |
| Explicit types | `(String s, Integer i) -> s.length() + i` |

**Target typing:** The compiler infers the lambda's type from the **context** (the functional interface it's assigned to or passed as). The lambda doesn't declare its own return type — the compiler derives it from the single abstract method (SAM) of the target interface.

```java
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());
// Same lambda, different target type — different SAM signature
Function<String, Integer> lengthFn = s -> s.length();
```

**Follow-up trap:** "Can a lambda be assigned to `Object`?" → No — a lambda must match a **functional interface** type. `Object` is not a functional interface, so `( ) -> {}` cannot be assigned to `Object` directly.

---

### Q3. What is "effectively final"? Why does it matter for lambdas?
**Answer:** A local variable is **effectively final** if it is never reassigned after initialization (even without the `final` keyword). Lambdas and anonymous inner classes can **capture** local variables only if they are effectively final.

```java
int multiplier = 3;   // effectively final
list.forEach(n -> System.out.println(n * multiplier));
// multiplier = 5;   // COMPILE ERROR — breaks effective finality for the lambda above
```

**Why:** Captured variables are copied into the lambda's closure at creation time. If reassignment were allowed, the lambda would see a stale/inconsistent value — Java prevents this at compile time.

**Senior detail:** Instance/static fields **can** be modified from inside a lambda (no effectively-final restriction) because they're accessed via `this`/class reference, not copied. But doing so in concurrent/parallel streams is a **thread-safety bug** — a common interview follow-up.

**Follow-up:** "Can a lambda modify a local variable?" → No — only read effectively-final locals. To accumulate, use wrappers (`AtomicInteger`), collect via stream terminal ops, or use mutable objects whose *reference* is effectively final (e.g., `StringBuilder sb = new StringBuilder(); list.forEach(sb::append)` — mutating the object is fine; reassigning `sb` is not).

---

### Q4. Lambda vs Anonymous Inner Class — key differences?
| Aspect | Anonymous Inner Class | Lambda |
|---|---|---|
| Syntax | Verbose (`new Interface() { ... }`) | Concise (`() -> ...`) |
| `this` binding | Refers to the anonymous class instance | Refers to the **enclosing** class (no new `this`) |
| Generated class | Separate `.class` file per AIC | `invokedynamic` — no separate class per lambda |
| Applicable to | Any interface or class (single abstract method not required) | **Functional interfaces only** (single abstract method) |
| Shadowing | Can shadow enclosing variables | Cannot declare shadow variables like AIC |
| `@Override` | Can explicitly override | N/A — implements SAM implicitly |

```java
class Outer {
    void demo() {
        Runnable aic = new Runnable() {
            public void run() {
                // this == anonymous Runnable instance
            }
        };
        Runnable lambda = () -> {
            // this == Outer instance (same as demo()'s this)
        };
    }
}
```

**Senior answer:** "I use lambdas whenever the target is a functional interface — cleaner and better JVM optimization. I still use anonymous classes when I need to implement an abstract class, override multiple methods, or access specific `this` semantics — though that's rare in modern Java."

---

### Q5. Can lambdas throw checked exceptions? How do you handle them?
**Answer:** A lambda's body must be compatible with the SAM's declared exceptions. If the SAM doesn't declare `throws`, the lambda **cannot throw checked exceptions** unless caught/wrapped inside the body.

```java
// SAM: void accept(T t) — no throws
list.forEach(s -> {
    try {
        processFile(s);   // throws IOException
    } catch (IOException e) {
        throw new UncheckedIOException(e);   // wrap
    }
});

// Or define a custom @FunctionalInterface that declares throws
@FunctionalInterface
interface ThrowingConsumer<T> {
    void accept(T t) throws Exception;
}
```

**Production pattern:** Libraries like Vavr or small utility wrappers (`ThrowingFunction`) exist, but in enterprise code the idiomatic approach is either wrap in unchecked exceptions or handle within the lambda — interviewers often ask this because `Stream.forEach()` with IO operations is a very common real-world pain point.

---

## SECTION 2: FUNCTIONAL INTERFACES (Predicate, Function, Consumer, Supplier)

### Q6. What is a Functional Interface?
**Answer:** An interface with **exactly one abstract method** (SAM — Single Abstract Method). Java 8 added `@FunctionalInterface` — a compile-time annotation (optional but recommended) that enforces the single-abstract-method rule.

```java
@FunctionalInterface
interface Validator {
    boolean validate(String input);
    // default and static methods are allowed — they don't count toward SAM
    default void log(String msg) { System.out.println(msg); }
    static Validator alwaysTrue() { return s -> true; }
}
```

**Key rule:** `@FunctionalInterface` interfaces can have **default methods** and **static methods** (Java 8+) and **private methods** (Java 9+) — only **abstract instance methods** count. If you add a second abstract method, compilation fails.

**Built-in package:** `java.util.function` — `Predicate`, `Function`, `Consumer`, `Supplier`, `BiFunction`, `UnaryOperator`, `BinaryOperator`, etc.

---

### Q7. Explain Predicate, Function, Consumer, and Supplier with examples.
**Answer — the "core four" from `java.util.function`:**

| Interface | Signature | Purpose | Example |
|---|---|---|---|
| **Predicate\<T\>** | `boolean test(T t)` | Test/filter a condition | `s -> s.startsWith("A")` |
| **Function\<T,R\>** | `R apply(T t)` | Transform T → R | `s -> s.length()` |
| **Consumer\<T\>** | `void accept(T t)` | Side effect, no return | `System.out::println` |
| **Supplier\<T\>** | `T get()` | Produce/provide a value | `() -> UUID.randomUUID()` |

```java
List<String> names = List.of("Alice", "Bob", "Anna");

// Predicate — filter
names.stream()
     .filter(s -> s.startsWith("A"))          // Predicate<String>
     .forEach(System.out::println);            // Consumer<String>

// Function — map/transform
List<Integer> lengths = names.stream()
     .map(String::length)                       // Function<String, Integer>
     .toList();

// Supplier — lazy creation
Supplier<LocalDateTime> now = LocalDateTime::now;
LocalDateTime timestamp = now.get();

// Predicate composition
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> longName = s -> s.length() > 4;
Predicate<String> combined = startsWithA.and(longName);   // short-circuit AND
```

**Follow-up:** "Predicate vs Function returning Boolean?" → `Predicate` is specialized for `boolean` with composition methods (`and`, `or`, `negate`) and `isEqual()` factory — use `Predicate` for filtering; don't use `Function<T, Boolean>` (boxed, no composition API).

---

### Q8. What are UnaryOperator and BinaryOperator?
**Answer:** Specializations of `Function` where input and output types are the same:

- **`UnaryOperator<T>`** extends `Function<T, T>` — one arg, same type in/out. Example: `s -> s.toUpperCase()`.
- **`BinaryOperator<T>`** extends `BiFunction<T, T, T>` — two args, same type in/out. Example: `(a, b) -> a + b` for integers.

```java
UnaryOperator<String> upper = String::toUpperCase;
BinaryOperator<Integer> sum = Integer::sum;

List<String> uppercased = names.stream().map(upper).toList();
int total = List.of(1, 2, 3).stream().reduce(0, sum);
```

**Interview tip:** `Integer::sum`, `Integer::max`, `String::concat` are common method-reference answers for `BinaryOperator` examples.

---

### Q9. What are BiFunction, BiConsumer, and BiPredicate?
**Answer:** Two-argument variants:

| Interface | Signature | Use case |
|---|---|---|
| **BiPredicate\<T,U\>** | `boolean test(T t, U u)` | Test pair: `(key, value) -> key.equals("id")` |
| **BiFunction\<T,U,R\>** | `R apply(T t, U u)` | Combine two inputs: `(a, b) -> a + b` |
| **BiConsumer\<T,U\>** | `void accept(T t, U u)` | Side effect on pair: `Map.forEach((k,v) -> ...)` |

```java
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);
scores.forEach((name, score) -> System.out.println(name + ": " + score));  // BiConsumer

BiFunction<String, String, String> concat = String::concat;
String result = concat.apply("Hello, ", "World");
```

---

### Q10. When would you create a custom functional interface vs use a built-in one?
**Answer:**
- Use **built-in** (`Predicate`, `Function`, etc.) for generic, reusable patterns — streams, filters, maps.
- Create **custom** when the SAM has **domain meaning** or a **non-standard signature** (multiple checked exceptions, domain-specific method name):

```java
@FunctionalInterface
interface OrderProcessor {
    OrderResult process(Order order) throws OrderValidationException;
}
```

**Senior angle:** Custom functional interfaces improve **readability** in APIs (`processOrder(OrderProcessor processor)` is clearer than `Function<Order, OrderResult>`). But avoid proliferating interfaces that duplicate `Function`/`Consumer` — use built-ins unless the name adds real semantic value.

---

## SECTION 3: STREAM API (map, filter, reduce, collect)

### Q11. What is the Stream API? How is a Stream different from a Collection?
**Answer:** The **Stream API** (`java.util.stream`) provides a **declarative, functional-style** way to process sequences of elements — filter, map, reduce, collect — often in a pipeline, with optional parallel execution.

| Aspect | Collection | Stream |
|---|---|---|
| Storage | Holds elements | **Does not store** — computes on demand |
| Iteration | External (`for`, `iterator`) | **Internal** (pipeline drives iteration) |
| Reusability | Can iterate multiple times | **Single-use** — consumed after terminal op |
| Lazy evaluation | Eager (data always there) | **Lazy** — intermediate ops deferred until terminal op |
| Modification | Can add/remove (most) | **Cannot modify** source |

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

names.stream()                          // source
     .filter(n -> n.length() > 3)      // intermediate (lazy)
     .map(String::toUpperCase)          // intermediate (lazy)
     .forEach(System.out::println);     // terminal — triggers pipeline
```

**Follow-up:** "Can you reuse a Stream?" → No — calling a terminal operation closes the stream; further operations throw `IllegalStateException`. Create a new stream from the source for another pass.

---

### Q12. Intermediate vs Terminal operations — explain laziness.
**Answer:**
- **Intermediate operations** (`filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`, `peek`) return a new Stream — **lazy**, nothing executes until a terminal op runs.
- **Terminal operations** (`collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `toList`) produce a result or side effect — **trigger** the entire pipeline.

```java
List<String> data = List.of("a", "b", "c");

Stream<String> stream = data.stream()
    .filter(s -> {
        System.out.println("filter: " + s);   // NOT printed yet
        return s.length() > 0;
    });

System.out.println("Pipeline defined, not executed");
long count = stream.count();   // NOW filter runs
```

**Short-circuiting:** Operations like `findFirst()`, `anyMatch()`, `limit(n)` can stop processing early — the stream doesn't necessarily traverse the entire source.

**Senior detail:** `peek()` is for debugging only — don't use it for business logic side effects in production pipelines; use `forEach` at the terminal stage or refactor to explicit steps.

---

### Q13. Explain map, filter, reduce, and collect with examples.
**Answer:**

**filter** — keeps elements matching a `Predicate`:
```java
List<String> longNames = names.stream()
    .filter(n -> n.length() > 4)
    .toList();
```

**map** — transforms each element via `Function<T, R>`:
```java
List<Integer> lengths = names.stream()
    .map(String::length)
    .toList();
```

**reduce** — combines elements to a single result (associative accumulation):
```java
// Optional return — empty stream → Optional.empty()
Optional<Integer> sum = List.of(1, 2, 3).stream()
    .reduce(Integer::sum);                    // 6

// With identity — always returns T (identity used if empty)
int total = List.of(1, 2, 3).stream()
    .reduce(0, Integer::sum);                 // 6

// With identity + accumulator + combiner (required for parallel)
int parallelSum = List.of(1, 2, 3).parallelStream()
    .reduce(0, Integer::sum, Integer::sum);
```

**collect** — mutable reduction into a collection or custom structure via `Collector`:
```java
// Group by first letter
Map<Character, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(s -> s.charAt(0)));

// Join strings
String joined = names.stream()
    .collect(Collectors.joining(", "));

// To specific collection type
Set<String> unique = names.stream()
    .collect(Collectors.toCollection(LinkedHashSet::new));

// Downstream collectors
Map<Character, Long> countByLetter = names.stream()
    .collect(Collectors.groupingBy(s -> s.charAt(0), Collectors.counting()));
```

**Follow-up:** "reduce vs collect?" → `reduce` produces one **immutable** combined value (associative/commutative ideal). `collect` uses a **mutable container** and `Collector` strategies — better for building Lists, Maps, Sets, or complex aggregations. Prefer `collect` when the result is a collection.

---

### Q14. What is flatMap? When do you use it instead of map?
**Answer:** `map` transforms one element → one element. **`flatMap`** transforms one element → a **Stream** (or Optional), then **flattens** all resulting streams into one.

```java
List<List<String>> nested = List.of(
    List.of("a", "b"),
    List.of("c", "d")
);

// map → Stream<Stream<String>> — wrong shape
// flatMap → Stream<String> — flattened
List<String> flat = nested.stream()
    .flatMap(List::stream)
    .toList();   // [a, b, c, d]

// Real-world: orders with line items
List<OrderLine> allLines = orders.stream()
    .flatMap(order -> order.getLines().stream())
    .toList();
```

**Optional flatMap:** Avoids nested `Optional<Optional<T>>`:
```java
Optional<String> city = getUser(id)
    .flatMap(User::getAddress)      // Optional<Address> → Optional<String> via flatMap
    .map(Address::getCity);
```

---

### Q15. Parallel streams — how do they work? When should you use them?
**Answer:** `collection.parallelStream()` or `stream.parallel()` splits the source into chunks processed by the **Fork/Join pool** (`ForkJoinPool.commonPool()` by default).

**Use when:**
- Large data sets (thousands+ elements).
- CPU-bound, **stateless**, **associative** operations (map, filter, reduce).
- No shared mutable state.

**Avoid when:**
- Small collections (overhead exceeds benefit).
- IO-bound work (blocks common pool threads — use dedicated `ExecutorService` instead).
- Operations with **order dependencies** or **non-associative** reductions.
- Shared mutable state (race conditions).

```java
// Safe — stateless, associative
long count = hugeList.parallelStream()
    .filter(s -> s.length() > 5)
    .count();

// DANGEROUS — shared mutable counter
AtomicInteger counter = new AtomicInteger(0);
list.parallelStream().forEach(s -> counter.incrementAndGet());  // OK with AtomicInteger
// list.parallelStream().forEach(s -> sharedList.add(s));       // NOT thread-safe
```

**Senior trap follow-up:** "Does parallel stream always use all CPU cores?" → It uses the **common ForkJoinPool** (by default `Runtime.getRuntime().availableProcessors() - 1` threads). Blocking tasks in parallel streams can **starve** other code using the same pool — a real production issue with JDBC/HTTP inside `parallelStream()`.

---

### Q16. collect vs forEach — when to use which?
**Answer:**
- **`forEach(Consumer)`** — terminal op for **side effects** only (printing, logging, sending events). Returns `void`. Does not produce a new collection.
- **`collect(Collector)`** — terminal op that **accumulates** into a result (List, Map, String, custom). Returns the accumulated result. Preferred when you need output data.

```java
// Side effect — OK for logging, not for building a list
names.stream().forEach(System.out::println);

// Building result — use collect
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))
    .collect(Collectors.toList());
```

**Anti-pattern:** Using `forEach` to add to an external `ArrayList` — not thread-safe in parallel streams, and violates functional style. Use `collect(toList())`.

**Follow-up:** "forEach vs for-each loop?" → `forEach` on stream is internal iteration; you lose `break`/`continue`. For simple iteration with early exit, a loop or `takeWhile` (Java 9+) may be clearer.

---

### Q17. What are common Collectors you should know?
**Answer:**

| Collector | Result |
|---|---|
| `toList()`, `toSet()` | Mutable List/Set (Java 16+: `toList()` returns unmodifiable) |
| `toCollection(Supplier)` | Specific collection type |
| `joining(delimiter)` | Concatenated String |
| `groupingBy(classifier)` | `Map<K, List<T>>` |
| `groupingBy(classifier, downstream)` | Nested aggregation |
| `partitioningBy(Predicate)` | `Map<Boolean, List<T>>` — exactly two groups |
| `counting()`, `summingInt()`, `averagingInt()` | Numeric aggregations |
| `mapping(mapper, downstream)` | Transform before downstream collect |
| `collectingAndThen(downstream, finisher)` | Post-process result (e.g., unmodifiable wrapper) |

```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
```

---

## SECTION 4: OPTIONAL CLASS

### Q18. What is Optional? Why was it introduced?
**Answer:** `Optional<T>` is a **container object** that may or may not contain a non-null value — introduced to make **absence of value explicit** in APIs, reducing `NullPointerException` risk and forcing callers to handle the "no value" case.

```java
// Before — null ambiguity
public User findUser(String id) { return null; }   // caller may forget null check

// After — explicit
public Optional<User> findUser(String id) {
    return userRepository.lookup(id);   // returns Optional.empty() or Optional.of(user)
}

User user = findUser("123")
    .orElseThrow(() -> new UserNotFoundException("123"));
```

**Important:** Optional was designed for **return types**, not fields or method parameters (see Q21).

**Follow-up:** "Does Optional eliminate NPE?" → No — it reduces NPE when used correctly, but `optional.get()` without checking still throws `NoSuchElementException`, and `Optional.of(null)` throws NPE immediately.

---

### Q19. Optional.of vs Optional.ofNullable vs Optional.empty?
| Factory | Behavior |
|---|---|
| `Optional.of(value)` | Wraps value; **throws NPE** if value is null |
| `Optional.ofNullable(value)` | Wraps value, or `Optional.empty()` if null |
| `Optional.empty()` | Shared singleton empty Optional |

```java
Optional<String> a = Optional.of("hello");        // OK
Optional<String> b = Optional.ofNullable(null);   // Optional.empty()
Optional<String> c = Optional.empty();            // Optional.empty()
// Optional<String> d = Optional.of(null);        // NPE!
```

**Rule of thumb:** Use `of()` when you're **certain** the value is non-null. Use `ofNullable()` when the source may be null (DB lookup, map.get, chained calls).

---

### Q20. orElse vs orElseGet vs orElseThrow?
**Answer:**

| Method | When fallback is evaluated | Use when |
|---|---|---|
| `orElse(T other)` | **Always** — fallback computed eagerly | Fallback is a constant/cheap value |
| `orElseGet(Supplier<T>)` | **Only if empty** — lazy | Fallback is expensive to compute |
| `orElseThrow(Supplier<X>)` | **Only if empty** — throws | Absence is an exceptional/error condition |

```java
// BAD — expensiveDefault() runs even when optional is present
User user = findUser(id).orElse(loadDefaultUserFromDatabase());

// GOOD — loadDefaultUserFromDatabase() only if empty
User user = findUser(id).orElseGet(() -> loadDefaultUserFromDatabase());

User user = findUser(id)
    .orElseThrow(() -> new UserNotFoundException(id));
```

**Classic interview trap:** `orElse(computeExpensive())` vs `orElseGet(() -> computeExpensive())` — the first always calls `computeExpensive()` even when not needed.

---

### Q21. Optional best practices and anti-patterns?
**Best practices:**
- Use as **return type** for methods that may not find a result.
- Chain with `map`, `flatMap`, `filter` instead of imperative `if (optional.isPresent())`.
- Use `ifPresent(Consumer)` or `ifPresentOrElse` (Java 9+) for side effects.

```java
// Idiomatic chaining
String cityUpper = getUser(id)
    .flatMap(User::getAddress)
    .map(Address::getCity)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");
```

**Anti-patterns (senior interview gold):**
1. **Optional as method parameter** — overloading or `@Nullable` is clearer; callers shouldn't wrap args in Optional.
2. **Optional as field** — adds overhead; use null with clear contract or lazy initialization.
3. **Optional in collections** — `List<Optional<T>>` is a code smell; filter nulls instead.
4. **`optional.get()` without check** — use `orElse*` or `ifPresent`.
5. **Serializing Optional** — not designed for JSON/JPA fields (though Jackson has modules).

**Follow-up:** "Is Optional serializable?" → Yes (since Java 9), but still avoid using it as a DTO/entity field.

---

### Q22. Explain map and flatMap on Optional.
**Answer:**
- **`map(Function)`** — if present, apply function; if function returns null, result is `Optional.empty()` (wrapped).
- **`flatMap(Function<T, Optional<R>>)`** — if present, apply function that returns Optional; **flattens** nested Optional (no `Optional<Optional<R>>`).

```java
Optional<String> name = Optional.of("alice");

Optional<Integer> len = name.map(String::length);           // Optional[5]

Optional<String> upper = name.flatMap(this::toUpperIfValid);
// flatMap used when the mapping function itself returns Optional

Optional<String> bad = name.map(s -> null);                 // Optional.empty() — not Optional[null]
```

**flatMap vs map for method returning Optional:**
```java
// map → Optional<Optional<Address>>
optionalUser.map(User::getAddress);

// flatMap → Optional<Address>
optionalUser.flatMap(User::getAddress);   // where getAddress() returns Optional<Address>
```

---

## SECTION 5: METHOD REFERENCES

### Q23. What are Method References? What are the four types?
**Answer:** A **method reference** is shorthand for a lambda that **only calls an existing method** — syntax: `ClassName::methodName` or `instance::methodName`. Requires a compatible functional interface.

| Type | Syntax | Lambda equivalent | Example |
|---|---|---|---|
| **Static** | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` | `Integer::parseInt` |
| **Instance on particular object** | `instance::method` | `(args) -> instance.method(args)` | `System.out::println` |
| **Instance on arbitrary object** | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` | `String::compareToIgnoreCase` |
| **Constructor** | `ClassName::new` | `(args) -> new ClassName(args)` | `ArrayList::new` |

```java
// Static
Function<String, Integer> parser = Integer::parseInt;

// Bound instance
Consumer<String> printer = System.out::println;

// Unbound instance — first param becomes the receiver
Comparator<String> cmp = String::compareToIgnoreCase;

// Constructor
Supplier<List<String>> listFactory = ArrayList::new;
Function<String, User> userFactory = User::new;
```

---

### Q24. When to use method reference vs lambda?
**Answer:**
- Use **method reference** when the lambda **only delegates** to an existing method with matching signature — cleaner, more readable.
- Use **lambda** when you need **additional logic**, multiple statements, or parameter transformation.

```java
// Method reference — direct delegation
list.forEach(System.out::println);
list.stream().map(String::toUpperCase);

// Lambda — extra logic needed
list.forEach(s -> System.out.println("Name: " + s));
list.stream().map(s -> s.trim().toUpperCase());
```

**Follow-up:** "Is `String::length` instance or static?" → Unbound instance method reference — for `Function<String, Integer>`, the Stream element becomes the receiver: `s -> s.length()`.

---

### Q25. Constructor method references and array constructor references?
```java
// Zero-arg constructor
Supplier<ArrayList<String>> supplier = ArrayList::new;

// One-arg constructor
Function<Integer, int[]> arrayCreator = int[]::new;   // size -> new int[size]

// Collect to custom type
List<String> names = stream.collect(ArrayList::new, List::add, List::addAll);
// Or more idiomatically:
List<String> names = stream.collect(Collectors.toCollection(ArrayList::new));
```

**Senior note:** `ArrayList::new` as `Supplier` vs `int[]::new` as `IntFunction<int[]>` — constructor references adapt to the target functional interface's arity.

---

## SECTION 6: DEFAULT & STATIC METHODS IN INTERFACES

### Q26. Why were default methods introduced in Java 8?
**Answer:** To **evolve interfaces without breaking existing implementations**. Before Java 8, adding a method to an interface broke every class that implemented it. Default methods provide a **default implementation** that implementors inherit unless they override.

**Primary motivator:** Adding methods to core interfaces like **`Collection`**, **`List`**, **`Iterable`** (e.g., `forEach`, `stream`, `removeIf`) without breaking the entire Java ecosystem.

```java
public interface Iterable<T> {
    default void forEach(Consumer<? super T> action) {
        for (T t : this) {
            action.accept(t);
        }
    }
}
```

**Also enables:** Interface **multiple inheritance of behavior** (not state) — a class can implement multiple interfaces with defaults.

---

### Q27. How does Java resolve conflicts when two interfaces provide the same default method?
**Answer — conflict resolution rules:**

1. **Class wins over interface** — if a superclass has the method, it's used (default ignored).
2. **More specific interface wins** — if one interface extends another and overrides the default, the sub-interface version wins.
3. **Two unrelated interfaces with same default** — implementing class must **explicitly override** and choose:

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();   // explicitly delegate to A's default
    }
}
```

**Follow-up trap:** "Does this cause the diamond problem like C++ multiple inheritance?" → Java allows **multiple inheritance of type and behavior (defaults)** but **not state** — interfaces can't have instance fields (only public static final constants). Conflicts are resolved at compile time with explicit overrides — no ambiguity at runtime like C++ virtual inheritance.

---

### Q28. Static methods in interfaces — why and how?
**Answer:** Java 8 allows **static methods** in interfaces — utility methods scoped to the interface, not inherited by implementors.

```java
public interface PaymentGateway {
    static PaymentGateway create(String type) {
        return switch (type) {
            case "stripe" -> new StripeGateway();
            case "paypal" -> new PayPalGateway();
            default -> throw new IllegalArgumentException(type);
        };
    }

    void charge(Money amount);
}

// Usage — via interface name, not instance
PaymentGateway gw = PaymentGateway.create("stripe");
```

**vs default method:** Static methods are **not** overridable and **not** inherited. Default methods **are** inherited and can be overridden.

**Follow-up (Java 9+):** Interfaces can also have **private instance methods** (to share code between default methods) and **private static methods** — worth mentioning to show currency beyond Java 8.

---

### Q29. Can you override a default method? What about abstract class vs interface defaults?
**Answer:** Yes — implementors can **override** a default method like any instance method. If not overridden, the default implementation is inherited.

```java
interface Logger {
    default void log(String msg) { System.out.println(msg); }
}

class FileLogger implements Logger {
    @Override
    public void log(String msg) { writeToFile(msg); }   // replaces default
}
```

**Abstract class vs interface defaults (common comparison):**
| | Abstract Class | Interface + Default |
|---|---|---|
| State (fields) | Can have instance fields | Only constants |
| Inheritance | Single inheritance | Multiple interfaces |
| Constructor | Yes | No |
| Purpose | IS-A with shared state/implementation | Capability contract + optional behavior |

**Senior answer:** "I use abstract classes when subclasses share **state and core lifecycle**. I use interfaces with defaults for **capability mix-ins** and API evolution — e.g., `Repository` interface adding a default `findAllPaged()` without breaking 50 existing implementations."

---

## SECTION 7: DATE/TIME API (java.time)

### Q30. Why did Java 8 introduce a new Date/Time API? Problems with old java.util.Date/Calendar?
**Answer:** Legacy API (`Date`, `Calendar`, `SimpleDateFormat`) was widely criticized:

| Problem | Legacy | java.time (Java 8) |
|---|---|---|
| Mutability | `Date` is mutable — accidental modification | **Immutable** — thread-safe by design |
| Clarity | `Date` includes time but name suggests date only | Separate types: `LocalDate`, `LocalTime`, `LocalDateTime` |
| API design | Poor — month 0-based in Calendar, confusing methods | Clean, fluent API |
| Thread safety | `SimpleDateFormat` **not thread-safe** | `DateTimeFormatter` is **immutable and thread-safe** |
| Time zones | Confusing, error-prone | Explicit `ZonedDateTime`, `ZoneId` |
| Period/duration | Manual calculation | `Period`, `Duration` types |

```java
// Legacy — mutable, confusing
Date d = new Date();
d.setYear(125);   // deprecated, bug-prone (year 1900+)

// java.time — immutable, clear
LocalDate date = LocalDate.of(2026, 8, 15);
LocalDate nextWeek = date.plusWeeks(1);   // returns new instance, date unchanged
```

---

### Q31. Explain the key classes: LocalDate, LocalTime, LocalDateTime, ZonedDateTime, Instant.
**Answer:**

| Class | Represents | Time zone | Use case |
|---|---|---|---|
| **LocalDate** | Date only (YYYY-MM-DD) | None | Birthdays, due dates |
| **LocalTime** | Time only (HH:mm:ss.ns) | None | Store opening hours |
| **LocalDateTime** | Date + time | None | Local events without zone context |
| **ZonedDateTime** | Date + time + zone | Yes (`ZoneId`) | Global apps, scheduling across zones |
| **Instant** | Point on UTC timeline | UTC | Timestamps, event logs, DB storage |

```java
LocalDate today = LocalDate.now();
LocalTime now = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.of(2026, 8, 15, 14, 30);
ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
Instant timestamp = Instant.now();   // always UTC — best for persistence
```

**Follow-up:** "What should you store in a database?" → **`Instant`** or **`OffsetDateTime`** for absolute moments; **`LocalDateTime`** only if the zone is implicit/known by business context (e.g., "store closes at 9 PM local" — but be careful with DST).

---

### Q32. Duration vs Period?
**Answer:**
- **`Duration`** — amount of time in **time-based units** (seconds, nanos). For `LocalTime`, `Instant`, `LocalDateTime` (time-component diffs).
- **`Period`** — amount of time in **date-based units** (years, months, days). For `LocalDate`.

```java
Duration betweenTimes = Duration.between(
    LocalTime.of(9, 0),
    LocalTime.of(17, 30)
);   // PT8H30M

Period betweenDates = Period.between(
    LocalDate.of(2026, 1, 1),
    LocalDate.of(2026, 8, 15)
);   // P7M14D

LocalDate deadline = LocalDate.now().plus(Period.ofDays(30));
Instant timeout = Instant.now().plus(Duration.ofMinutes(30));
```

**Trap:** Don't use `Duration.between` on `LocalDate` — it won't work (use `Period`). Mixing them incorrectly is a common bug in interview coding exercises.

---

### Q33. DateTimeFormatter — formatting and parsing?
**Answer:** `DateTimeFormatter` is **immutable and thread-safe** (unlike `SimpleDateFormat`).

```java
LocalDate date = LocalDate.of(2026, 8, 15);

// Predefined
String iso = date.format(DateTimeFormatter.ISO_LOCAL_DATE);   // 2026-08-15

// Custom pattern
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MMM-yyyy", Locale.ENGLISH);
String formatted = date.format(formatter);           // 15-Aug-2026
LocalDate parsed = LocalDate.parse("15-Aug-2026", formatter);
```

**Senior tip:** Always pass **`Locale`** explicitly for month names/day names in user-facing formats — default locale varies by JVM/deployment environment and causes production bugs in international apps.

**Follow-up:** "Can DateTimeFormatter parse ZonedDateTime?" → Yes — use patterns with zone offset (`XXX`, `Z`) and parse with `ZonedDateTime.parse(text, formatter)`.

---

### Q34. How do you convert between legacy Date/Calendar and java.time?
**Answer:**

```java
// Date → Instant → LocalDateTime
Date legacy = new Date();
Instant instant = legacy.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());

// LocalDateTime → Instant → Date
Instant inst = ldt.atZone(ZoneId.systemDefault()).toInstant();
Date converted = Date.from(inst);

// java.sql.Timestamp ↔ Instant
Timestamp ts = Timestamp.from(Instant.now());
Instant fromTs = ts.toInstant();
```

**Senior angle:** In modern Spring/JPA (Hibernate 5.2+), map entity fields directly to `Instant`, `LocalDate`, or `ZonedDateTime` — avoid legacy `Date` in new code entirely.

---

### Q35. How does java.time handle time zones and daylight saving time (DST)?
**Answer:** `ZonedDateTime` uses **`ZoneId`** (region rules, e.g., `"America/New_York"`) which encapsulates DST transitions. `ZoneOffset` is a fixed offset (e.g., `-05:00`) with no DST rules.

```java
ZoneId ny = ZoneId.of("America/New_York");
ZonedDateTime beforeDST = ZonedDateTime.of(2026, 3, 8, 1, 30, 0, 0, ny);
ZonedDateTime afterDST  = beforeDST.plusHours(2);
// JVM applies DST rules — offset may change from -05:00 to -04:00
```

**Gap/overlap traps during DST transitions:**
- **Gap** (spring forward): non-existent local times — `ZonedDateTime` adjusts forward.
- **Overlap** (fall back): ambiguous local times — two valid offsets; `ZonedDateTime` defaults to earlier offset unless specified.

**Best practice:** Store **`Instant`** in DB/logs (absolute point in time); convert to `ZonedDateTime` with user's `ZoneId` only at presentation layer.

---

## SECTION 8: CROSS-CUTTING / SCENARIO QUESTIONS (HIGH FREQUENCY)

### Q36. Write a Stream pipeline: filter employees earning > 100k, group by department, average salary per department.
```java
Map<String, Double> avgByDept = employees.stream()
    .filter(e -> e.getSalary() > 100_000)
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
```
**Follow-up:** "How would you handle null department?" → `filter(Objects::nonNull)` on department, or `groupingBy(e -> Optional.ofNullable(e.getDepartment()).orElse("UNKNOWN"))`.

---

### Q37. Find the second highest distinct salary using Streams.
```java
Optional<BigDecimal> secondHighest = employees.stream()
    .map(Employee::getSalary)
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
```
**Alternative:** `reduce` with top-two tracking — discuss trade-offs (sorted approach O(n log n) vs single-pass O(n) — interviewer may ask for optimization).

---

### Q38. What Java 8 features improved Collection operations?
**Answer:**
- **`default void forEach(Consumer)`** on `Iterable`.
- **`default boolean removeIf(Predicate)`** — safe, readable bulk removal.
- **`default Stream<E> stream()` / `parallelStream()`** on `Collection`.
- **`List.sort(Comparator)`** / **`Collections.sort`** with lambdas.
- **`Map.forEach`, `replaceAll`, `putIfAbsent`, `computeIfAbsent`, `merge`** — functional-style map operations.

```java
list.removeIf(s -> s.isBlank());
Map<String, User> cache = new HashMap<>();
cache.computeIfAbsent(userId, id -> userRepository.load(id));   // atomic, concise
cache.merge(key, 1, Integer::sum);   // atomic increment pattern
```

---

### Q39. What is the difference between Stream and parallelStream performance characteristics?
**Answer:** `parallelStream()` uses the **ForkJoin common pool** — benefits CPU-bound, large, stateless pipelines. Costs include task-splitting overhead, potential contention on the spliterator/source, and **ordering non-guarantee** (unless using `forOrdered` patterns). For IO-bound or small lists, sequential `stream()` is faster and safer.

**Production story:** A team used `parallelStream()` inside a REST controller for DB-heavy lookups — saturated the common pool, causing unrelated parts of the app to slow down. Fix: dedicated `ExecutorService` with async `CompletableFuture` composition instead.

---

### Q40. Quick-fire: Java 8 feature summary table
| Feature | Package / Location | One-line purpose |
|---|---|---|
| Lambda expressions | Language feature | Anonymous functions for behavior parameterization |
| Functional interfaces | `java.util.function` | SAM types that lambdas/method refs implement |
| Stream API | `java.util.stream` | Declarative, lazy, composable data processing |
| Optional | `java.util` | Explicit optional return values, reduce null bugs |
| Method references | Language feature | Shorthand lambda delegating to existing method |
| Default interface methods | Language feature | Interface evolution without breaking implementors |
| Static interface methods | Language feature | Interface-scoped utilities |
| Date/Time API | `java.time` | Immutable, thread-safe, clear date/time model |
| Nashorn JavaScript | `jdk.nashorn` | JS engine on JVM (deprecated/removal in later JDKs — mention awareness) |

---

*End of Java 8 Features Interview Prep — Topic 6*

# SENIOR 8+ YOE EXPANSION — STREAMS, OPTIONAL & COMPLETABLEFUTURE

> The original Java 8 module is preserved above. This section adds execution-model reasoning, performance trade-offs, custom collector concepts, async composition, and production interview scenarios.

## Q41. What actually happens when a Stream pipeline runs?
**Answer:** A stream is a computation pipeline: source → intermediate operations → terminal operation. Intermediate operations are lazy; the terminal operation triggers traversal. Streams do not generally store a second copy of the source data.

```java
employees.stream()
    .filter(Employee::isActive)
    .map(Employee::getSalary)
    .findFirst();
```

Because `findFirst()` short-circuits, the pipeline may stop early.

## Q42. Stateless vs stateful intermediate operations?
**Answer:** `filter()` and `map()` can generally process an element without retaining the whole stream. `sorted()` and `distinct()` are stateful because they may need information from multiple elements. Stateful operations can require buffering and can reduce parallel efficiency.

## Q43. Why is Stream laziness useful?
**Answer:** It enables deferred execution, short-circuiting and pipeline fusion. For:

```java
users.stream()
    .filter(User::isActive)
    .filter(this::isPreferred)
    .findFirst();
```

the stream can stop once a suitable element is found.

## Q44. What is a Spliterator?
**Answer:** `Spliterator` supports traversal and partitioning of elements. Its `trySplit()` operation allows a source to be divided for parallel processing. Characteristics such as `ORDERED`, `SIZED`, `SORTED`, `DISTINCT`, `CONCURRENT` and `SUBSIZED` communicate useful properties to the stream implementation.

## Q45. Why can parallelStream() be slower?
**Answer:** Parallelism has splitting, scheduling, coordination and merging overhead. Small collections and cheap operations often do not have enough work to justify that overhead. Blocking I/O, shared mutable state, ordering requirements and poor source splitting can make it worse.

## Q46. What is the danger of side effects in streams?
**Answer:**
```java
List<Result> result = new ArrayList<>();
items.parallelStream().forEach(result::add);
```
This is unsafe because multiple workers mutate a non-thread-safe list. Prefer collectors or thread-safe designs. More broadly, side effects make stream pipelines harder to reason about and parallelize.

## Q47. forEach() vs forEachOrdered()
**Answer:** On a parallel ordered stream, `forEach()` does not guarantee encounter order. `forEachOrdered()` preserves encounter order but can reduce parallelism. Use ordering only when the business requirement needs it.

## Q48. map() vs flatMap() — senior explanation
**Answer:** `map()` performs one-to-one transformation. `flatMap()` transforms each input into a stream and flattens the nested streams.

```java
orders.stream()
    .flatMap(o -> o.getItems().stream())
    .toList();
```

Use `flatMap()` when the domain contains nested collections and the result should be one logical sequence.

## Q49. reduce() vs collect()
**Answer:** `reduce()` is for combining values into a result, especially immutable/associative reductions. `collect()` is designed for mutable result containers and collector strategies such as grouping and partitioning.

```java
int total = numbers.stream().reduce(0, Integer::sum);
Map<String,List<Employee>> byDept =
    employees.stream().collect(Collectors.groupingBy(Employee::getDepartment));
```

## Q50. What makes a reduction safe for parallel execution?
**Answer:** The reduction needs a correct identity and an associative accumulation/combination operation. If the operation depends on encounter order or has non-associative behavior, a parallel reduction can produce different or incorrect results.

## Q51. How does a custom Collector work?
**Answer:** Conceptually it supplies:
- a result container
- an accumulator
- a combiner for partial results
- an optional finisher
- characteristics

The `combiner` is especially important in parallel execution because independently accumulated partial results must be merged correctly.

## Q52. groupingBy() vs groupingByConcurrent()
**Answer:** `groupingBy()` produces normal grouped results. `groupingByConcurrent()` targets concurrent grouping and can be useful with parallel streams when its semantics match the workload. Do not choose it simply because "concurrent" sounds faster; measure.

## Q53. Optional: what problem does it solve?
**Answer:** It makes the possibility of absence explicit in an API return type and encourages callers to handle that case. It is not a universal replacement for null and is normally most useful at API boundaries/return values.

## Q54. orElse() vs orElseGet()
**Answer:**
```java
optional.orElse(expensiveFallback());
```
evaluates the fallback eagerly.

```java
optional.orElseGet(this::expensiveFallback);
```
evaluates it lazily. This matters for expensive operations or side effects.

## Q55. Optional anti-patterns?
**Answer:** Avoid unguarded `get()`, Optional fields solely for style, Optional parameters as a blanket rule, and expensive eager `orElse()` fallbacks. Prefer clear domain/API semantics rather than wrapping every nullable value.

## Q56. map() vs flatMap() on Optional
**Answer:** `map()` wraps a normal result:

```java
user.map(User::getName);
```

If the function already returns Optional:

```java
user.flatMap(User::findAddress);
```

`flatMap()` prevents `Optional<Optional<T>>`.

# COMPLETABLEFUTURE

## Q57. thenApply() vs thenCompose()
**Answer:** `thenApply()` transforms a completed value. `thenCompose()` chains a function that itself returns a future.

```text
A → B
thenApply

A → Future<B>
thenCompose → flattened Future<B>
```

## Q58. thenCombine() vs thenCompose()
**Answer:** `thenCompose()` represents dependency: start B after A completes. `thenCombine()` combines independent futures.

```text
A ─┐
   ├─ combine → result
B ─┘
```

This is useful when two remote calls can execute concurrently and their results are needed for one response.

## Q59. What does allOf() return?
**Answer:** `CompletableFuture.allOf(...)` returns `CompletableFuture<Void>`. It represents completion of all supplied futures but does not directly provide a typed list of results. Keep the original futures and collect their results after `allOf()` completes.

## Q60. exceptionally(), handle(), whenComplete()?
**Answer:**
- `exceptionally()` is primarily recovery from failure.
- `handle()` receives success or failure and can transform either.
- `whenComplete()` observes completion and is useful for logging/metrics without being a normal recovery operation.

Choose based on whether you want to recover, transform, or merely observe.

## Q61. Which executor runs CompletableFuture tasks?
**Answer:** Async methods without an explicit executor commonly use the common ForkJoinPool. This can be reasonable for CPU-oriented work but can be problematic for blocking I/O. For explicit control:

```java
future.thenApplyAsync(this::call, executor);
```

Choose an executor appropriate for the workload.

## Q62. How do you handle timeouts in asynchronous service composition?
**Answer:** Use explicit timeouts and failure policy. Modern Java provides timeout-oriented CompletableFuture methods such as `orTimeout()` and `completeOnTimeout()`. Also distinguish a timeout from a successful fallback and propagate cancellation/deadline semantics where the underlying operation supports them.

## Q63. CompletableFuture vs parallelStream()
**Answer:** Streams are primarily for data transformation. CompletableFuture is for asynchronous workflow composition.

```text
Stream            → collection/data processing
CompletableFuture → async service workflow
```

Do not use either simply to "make code faster" without understanding workload and bottlenecks.

# PRODUCTION SCENARIOS

## Scenario 1: Three independent REST calls
**Question:** Customer, account and recommendation calls are independent. How would you design the aggregation?
**Answer:** Start them concurrently, combine results with `allOf()`/`thenCombine()`, apply deadlines and failure policy, and use an executor model appropriate to the calls. Avoid serial `get()` calls that destroy the intended concurrency.

## Scenario 2: A parallel stream calls another service
**Question:** Is this a good idea?
**Answer:** Usually treat it with caution. Parallel streams are not a general-purpose async I/O framework. External calls introduce latency, connection-pool limits, timeouts and backpressure concerns. Use an explicit concurrency model appropriate to the service architecture.

## Scenario 3: Stream pipeline has become unreadable
**Question:** Should everything be converted into one stream expression?
**Answer:** No. Break complex transformations into named methods or intermediate variables when that improves readability, debugging and testability. Java 8 functional style is a tool, not a requirement to eliminate every loop.

# SENIOR QUICK-FIRE
1. Streams are lazy until a terminal operation.
2. `sorted()` and `distinct()` are stateful.
3. `findFirst()` can short-circuit.
4. `findAny()` can be useful for parallel streams when encounter order is irrelevant.
5. `parallelStream()` is not automatically faster.
6. Avoid shared mutable state inside parallel pipelines.
7. `reduce()` needs correct identity/associative semantics.
8. Collector combiners matter for parallel execution.
9. `orElse()` is eager; `orElseGet()` is lazy.
10. Optional is not a universal null replacement.
11. `thenCompose()` flattens asynchronous dependencies.
12. `thenCombine()` combines independent futures.
13. `allOf()` returns `CompletableFuture<Void>`.
14. Know which executor runs async stages.
15. Async code still needs timeouts, cancellation and failure policy.
