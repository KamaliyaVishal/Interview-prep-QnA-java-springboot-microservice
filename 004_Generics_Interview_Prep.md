# 009. Generics — Interview Preparation — Senior Java Developer (experienced)

**Topics:** Type Parameters & Bounds, Wildcards (`?`, `extends`, `super`), Type Erasure, PECS Principle

> **Interview benchmark:** For each topic, be ready to explain **what it is → what problem it solves → how it works → example → trade-offs → production use → interview trap → senior follow-up**.

---

# 1. Generics Fundamentals

## Q1. What are Generics in Java? Why were they introduced?

**Answer:** Generics allow classes, interfaces, and methods to operate on **types as parameters**.

Without generics:

```java
class Box {
    private Object value;

    public void set(Object value) {
        this.value = value;
    }

    public Object get() {
        return value;
    }
}
```

The caller must cast:

```java
String value = (String) box.get();
```

With generics:

```java
class Box<T> {
    private T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}
```

Usage:

```java
Box<String> box = new Box<>();
box.set("Java");

String value = box.get();
```

**Problems Generics solve:** compile-time type safety, reduced explicit casting, reusable algorithms and data structures, stronger API contracts, fewer runtime `ClassCastException` problems.

**Senior answer:** Generics move many type errors from runtime to compile time and allow reusable, type-safe APIs without scattering casts throughout the code.

---

## Q2. What problem do Generics solve?

**Answer:** Primarily type safety and avoiding casts.

Type safety:

```java
List<String> names = new ArrayList<>();

names.add("Java");
// names.add(100); // compile-time error
```

Avoiding casts — without generics:

```java
String name = (String) list.get(0);
```

With generics:

```java
String name = names.get(0);
```

Generics are heavily used in Collections, repository abstractions, DTO wrappers, API responses, utility methods, framework APIs, and result types.

---

# 2. Type Parameters & Type Arguments

## Q3. What is a type parameter?

**Answer:** A type parameter is a placeholder for a type.

```java
class Box<T> {
    private T value;
}
```

Here `T` is the **type parameter**. When we write `Box<String>`, `String` is supplied as the actual type.

---

## Q4. What is the difference between a type parameter and a type argument?

**Answer:**

```java
class Box<T> { }
```

`T` is the **type parameter**.

```java
Box<String> box;
```

`String` is the **type argument**.

| Concept | Example |
|---|---|
| Type parameter | `T` |
| Type argument | `String` |

---

## Q5. What are common generic type parameter naming conventions?

**Answer:**

| Parameter | Common meaning |
|---|---|
| `T` | Type |
| `E` | Element |
| `K` | Key |
| `V` | Value |
| `N` | Number |
| `R` | Result |
| `S`, `U` | Additional types |

Example:

```java
interface Map<K, V> {
}
```

These are conventions, not language requirements.

---

# 3. Generic Classes and Methods

## Q6. How do you create a generic class?

**Answer:**

```java
class Pair<K, V> {

    private final K key;
    private final V value;

    Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() {
        return key;
    }

    public V getValue() {
        return value;
    }
}
```

Usage:

```java
Pair<String, Integer> pair =
        new Pair<>("age", 30);
```

---

## Q7. Can a generic method have its own type parameter?

**Answer:** Yes.

```java
public static <T> T identity(T value) {
    return value;
}
```

Usage:

```java
String s = identity("Java");
Integer n = identity(10);
```

The `<T>` before the return type declares the method's type parameter.

---

## Q8. What is the difference between a generic class and a generic method?

**Answer:** In a generic class the type parameter belongs to the class:

```java
class Box<T> {
    T get();
}
```

In a generic method the type parameter belongs to the method:

```java
static <T> T identity(T value) {
    return value;
}
```

A non-generic class can contain generic methods:

```java
class Utility {

    static <T> T first(List<T> list) {
        return list.get(0);
    }
}
```

---

# 4. Type Parameter Bounds

## Q9. What are bounded type parameters?

**Answer:** Bounds restrict which types may be used as a generic argument.

```java
class Calculator<T extends Number> {
}
```

Valid:

```java
Calculator<Integer> c1 = new Calculator<>();
Calculator<Double> c2 = new Calculator<>();
```

Invalid:

```java
// Calculator<String> c = new Calculator<>();
```

because `String` is not a subtype of `Number`.

---

## Q10. What does `<T extends Number>` mean?

**Answer:** It means `T` must be `Number` or a subtype of `Number`.

Despite the keyword `extends`, bounds can be used with interfaces as well:

```java
<T extends Comparable<T>>
```

means `T` must satisfy the specified `Comparable<T>` contract.

---

## Q11. Can a generic parameter have multiple bounds?

**Answer:** Yes.

```java
<T extends Number & Comparable<T>>
```

The first bound may be a class; additional bounds must be interfaces.

Valid: `<T extends Number & Comparable<T>>`

Invalid: `<T extends Number & Integer>` — because two class bounds are not allowed.

---

## Q12. Why are bounded type parameters useful?

**Answer:** They allow generic code to use operations guaranteed by the bound.

```java
static <T extends Number> double doubleValue(T value) {
    return value.doubleValue();
}
```

Because `T` is known to be a `Number`, the compiler allows `value.doubleValue()`. Without the bound, that operation could not be assumed.

---

# 5. Wildcards

## Q13. What is a wildcard in Java Generics?

**Answer:** A wildcard, written `?`, means "some unknown type."

```java
List<?> list;
```

The exact element type is unknown. You can safely read values as `Object`:

```java
Object value = list.get(0);
```

but cannot normally add an arbitrary non-null value.

---

## Q14. What is the difference between `List<Object>` and `List<?>`?

**Answer:** They are **not equivalent**.

`List<Object>` means the list is specifically parameterized with `Object`:

```java
List<Object> list = new ArrayList<>();

list.add("Java");
list.add(100);
```

`List<?>` means the list contains some unknown type:

```java
List<?> list = new ArrayList<String>();
```

Reading `Object value = list.get(0);` is safe. Adding `list.add("Java");` is a compile error, because the actual list could be `List<Integer>`. The exception is `list.add(null);`, since `null` is compatible with every reference type.

```text
List<Object> → specifically Object
List<?>      → unknown type
```

---

# 6. Unbounded Wildcard

## Q15. When should you use `<?>`?

**Answer:** Use it when the method works with a collection of **any type** but does not need to add values of that unknown type.

```java
static void printAll(List<?> list) {

    for (Object value : list) {
        System.out.println(value);
    }
}
```

This works with `List<String>`, `List<Integer>`, `List<Employee>`. The API communicates: I don't care what the element type is.

---

# 7. Upper-Bounded Wildcards

## Q16. What is `? extends T`?

**Answer:** It means "some unknown type that is `T` or a subtype of `T`."

```java
List<? extends Number>
```

could represent `List<Integer>`, `List<Double>`, `List<Float>`, `List<Number>`. The exact type is unknown.

---

## Q17. Why can't you add an Integer to `List<? extends Number>`?

**Answer:** Consider `List<? extends Number> numbers;` — the actual list could be `List<Double>`. If Java allowed `numbers.add(10);` we could put an `Integer` into a `List<Double>`. Therefore arbitrary non-null additions are rejected. Reading is safe: `Number n = numbers.get(0);` because every possible element type extends `Number`.

---

## Q18. What can you add to `List<? extends Number>`?

**Answer:** You cannot add a specific non-null value (`numbers.add(10)` and `numbers.add(10.5)` both fail to compile). You can add `numbers.add(null);`, because `null` is valid for every reference type.

---

# 8. Lower-Bounded Wildcards

## Q19. What is `? super T`?

**Answer:** It means "some unknown type that is `T` or a supertype of `T`."

```java
List<? super Integer>
```

could be `List<Integer>`, `List<Number>`, `List<Object>`.

---

## Q20. Why can you add Integer to `List<? super Integer>`?

**Answer:** Every possible actual list can safely contain an `Integer` (`List<Integer>`, `List<Number>`, `List<Object>`). Therefore `list.add(10);` is safe.

---

## Q21. What can you read from `List<? super Integer>`?

**Answer:** Only safely as `Object`: `Object value = list.get(0);`. `Integer value = list.get(0);` is a compile error, because the actual list might be `List<Object>` and the stored value could be a `String`.

---

# 9. PECS Principle

## Q22. What is the PECS principle?

**Answer:** PECS stands for **Producer Extends, Consumer Super**. If a structure **produces values** for your code to read, use `? extends T`. If a structure **consumes values** that your code wants to put into it, use `? super T`.

---

## Q23. Explain PECS with an example.

**Answer:** Producer:

```java
static double sum(List<? extends Number> numbers) {

    double total = 0;

    for (Number number : numbers) {
        total += number.doubleValue();
    }

    return total;
}
```

The list produces `Number` values. It accepts `List<Integer>`, `List<Double>`, `List<Long>`.

Consumer:

```java
static void addDefaults(
        List<? super Integer> numbers) {

    numbers.add(10);
    numbers.add(20);
}
```

The list consumes `Integer` values. It accepts `List<Integer>`, `List<Number>`, `List<Object>`.

---

## Q24. Why is PECS important?

**Answer:** PECS helps design flexible generic APIs. A copy operation can be expressed as:

```java
static <T> void copy(
        List<? extends T> source,
        List<? super T> destination) {

    for (T item : source) {
        destination.add(item);
    }
}
```

Now this is valid:

```java
List<Integer> source = ...;
List<Number> destination = ...;

copy(source, destination);
```

because source is the producer (`extends`) and destination is the consumer (`super`). This is the same fundamental design used by APIs such as `Collections.copy()`.

---

# 10. `extends` vs `super`

## Q25. What is the difference between `? extends T` and `? super T`?

**Answer:**

| | `? extends T` | `? super T` |
|---|---|---|
| Meaning | Unknown subtype of T | Unknown supertype of T |
| Main use | Producer | Consumer |
| Read safely as T | Yes | No |
| Add T | No | Yes |
| Example | `List<? extends Number>` | `List<? super Integer>` |

Memory rule: Producer → extends, Consumer → super.

---

## Q26. Why does `extends` allow reading but `super` allow writing?

**Answer:** For `List<? extends Number>`, the actual type might be `List<Integer>` or `List<Double>`, so every element is guaranteed to be at least a `Number` — safe to read as `Number n = list.get(0);`. But the compiler can't know which exact subtype the list requires when adding, so writes are blocked. For `List<? super Integer>`, the actual type might be `List<Integer>`, `List<Number>`, or `List<Object>` — all can safely accept an `Integer`, so `list.add(10);` is allowed, but reading only guarantees `Object`.

---

# 11. Type Erasure

## Q27. What is Type Erasure?

**Answer:** Java implements generics primarily through compile-time type checking and type erasure. `List<String>` and `List<Integer>` use the same runtime `ArrayList` class. The compiler uses generic information for type checking and inserts casts where required.

```java
List<String> names = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();
```

---

## Q28. Why does Java use Type Erasure?

**Answer:** A major reason is backward compatibility. Generics were introduced in Java 5 while preserving compatibility with existing bytecode and pre-generics APIs, so existing code using raw `List` continues to work alongside newer code using `List<String>`, with compiler warnings where safety can't be guaranteed.

**Senior answer:** Erasure allowed Java to add generic type checking without requiring a completely separate runtime representation for every parameterized type, preserving compatibility with the existing JVM ecosystem.

---

## Q29. Are generic type parameters available at runtime?

**Answer:** Not in the same way they are available to the compiler.

```java
List<String> list = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
```

Both use the runtime class `java.util.ArrayList`. Therefore `if (list instanceof List<String>) { }` is invalid, but `if (list instanceof List<?>) { }` is valid.

---

## Q30. Why can't you do `new T()`?

**Answer:** Because the runtime does not know which concrete type `T` represents after erasure.

```java
class Factory<T> {
    // T create() {
    //     return new T();
    // }
}
```

Better approach — Supplier:

```java
class Factory<T> {

    private final Supplier<T> supplier;

    Factory(Supplier<T> supplier) {
        this.supplier = supplier;
    }

    T create() {
        return supplier.get();
    }
}

Factory<StringBuilder> factory = new Factory<>(StringBuilder::new);
```

Another approach — class token:

```java
class Factory<T> {

    private final Class<T> type;

    Factory(Class<T> type) {
        this.type = type;
    }

    T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}
```

---

# 12. Generic Arrays and Reification

## Q31. Why can't you create `new T[10]`?

**Answer:** Arrays are **reified** — the runtime knows their component type — while generic type parameters are subject to erasure. Therefore `T[] values = new T[10];` is a compile error, because the runtime cannot determine whether the array should be `String[]`, `Integer[]`, or `Employee[]`. A collection is usually preferable: `List<T> values = new ArrayList<>();`.

---

## Q32. Why are arrays and generics different?

**Answer:** Arrays are reified: `String[] names = new String[10];` — the runtime knows it's a `String[]`, enabling runtime array-store checks (`Object[] values = names; values[0] = 100;` throws `ArrayStoreException`). Generics are mostly erased: `List<String>` does not have a separate runtime class from `List<Integer>`. This difference explains the restrictions around generic arrays.

---

# 13. Invariance

## Q33. Is `List<Integer>` a subtype of `List<Number>`?

**Answer:** No. Although `Integer extends Number`, this does not mean `List<Integer> extends List<Number>`. Java generics are **invariant** by default.

```java
List<Integer> integers = new ArrayList<>();
// List<Number> numbers = integers; // invalid
```

---

## Q34. Why are Java Generics invariant?

**Answer:** Because covariance for mutable collections would break type safety. If `List<Number> numbers = integers;` were allowed, then `numbers.add(10.5);` would put a `Double` into a list intended to contain `Integer`. Wildcards provide controlled flexibility instead: `List<? extends Number>` or `List<? super Integer>`.

---

# 14. Raw Types and Heap Pollution

## Q35. What is a raw type?

**Answer:** A raw type uses a generic class without specifying its type argument: `List list = new ArrayList();` instead of `List<String> list = new ArrayList<>();`. Raw types mainly exist for backward compatibility — avoid them in new code.

---

## Q36. What problems do raw types cause?

**Answer:** They weaken compile-time type safety.

```java
List list = new ArrayList();

list.add("Java");
list.add(100);

String value = (String) list.get(1); // compiles with warnings, fails at runtime
```

Problems include unchecked warnings, explicit casts, possible `ClassCastException`, weaker IDE support, unclear API contracts, and greater refactoring risk.

---

## Q37. What is heap pollution?

**Answer:** Heap pollution occurs when a variable of a parameterized type refers to an object that is not actually of the expected parameterized type.

```java
List<String> strings = new ArrayList<>();

List raw = strings;
raw.add(100);
```

The raw reference bypasses generic type checking. Later, `String value = strings.get(0);` can fail depending on the element retrieved. Heap pollution can arise from raw types, unchecked casts, unsafe generic varargs, or legacy APIs.

---

# 15. Generic Varargs

## Q38. Why can generic varargs be dangerous?

**Answer:** Arrays are reified, while generic type arguments are erased.

```java
static <T> void process(List<T>... lists) {
}
```

This can cause an unchecked warning because the runtime cannot fully represent the parameterized array type. `@SafeVarargs` can suppress the warning when the method is genuinely safe:

```java
@SafeVarargs
static <T> void process(List<T>... lists) {
    // safe implementation
}
```

Important: `@SafeVarargs` does not make unsafe code safe — it tells the compiler the implementation doesn't perform unsafe operations involving the varargs parameter.

---

# 16. Type Inference

## Q39. What is type inference?

**Answer:** The compiler can infer generic type arguments from context.

```java
List<String> names = new ArrayList<>();
```

The compiler infers `new ArrayList<String>()`. This is the diamond operator `<>`.

---

## Q40. What is the diamond operator?

**Answer:** Instead of `Map<String, List<Integer>> map = new HashMap<String, List<Integer>>();`, we write:

```java
Map<String, List<Integer>> map =
        new HashMap<>();
```

The compiler infers the type arguments from the target context.

---

## Q41. Can generic methods infer their type parameters?

**Answer:** Yes.

```java
static <T> T first(List<T> list) {
    return list.get(0);
}

String s = first(List.of("A", "B"));
Integer i = first(List.of(1, 2));
```

The compiler infers `T`.

---

# 17. Recursive Type Bounds

## Q42. What is a recursive type bound?

**Answer:** A recursive bound references the type parameter itself.

```java
<T extends Comparable<T>>
```

means "`T` must be comparable with another `T`."

```java
static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

Integer result = max(10, 20);
String result2 = max("A", "B");
```

---

## Q43. Why do we use `Comparable<T>` instead of raw `Comparable`?

**Answer:** Because we want a type-safe relationship — `<T extends Comparable<T>>` means "T can compare itself with T" instead of relying on the raw `Comparable` type.

---

# 18. Wildcard vs Type Parameter

## Q44. What is the difference between `List<?>` and `<T> List<T>`?

**Answer:** A wildcard, `void print(List<?> list)`, means "give me a list of some unknown type — I don't need to name that type." A type parameter, `<T> void copy(List<T> source, List<T> destination)`, means "there is a specific type `T`, and these parameters are related through that same type." Use a type parameter when the type relationship needs to be expressed across parameters or a return value.

---

## Q45. When should you prefer a wildcard over a type parameter?

**Answer:** Use a wildcard when the type is unknown and no relationship involving that type needs to be expressed (`void print(List<?> list)`). Use a type parameter when you need a relationship (`<T> T first(List<T> values)` or `<T> void copy(List<? extends T> source, List<? super T> destination)`).

**Senior rule:** Use a wildcard when the type is unknown and irrelevant to the relationship; use a type parameter when the type itself participates in a relationship.

---

# 19. Generics and Collections APIs

## Q46. Why does `Collections.copy()` use `extends` and `super`?

**Answer:** Its generic contract follows the PECS model:

```java
public static <T> void copy(
        List<? super T> dest,
        List<? extends T> src)
```

`src` is the producer (`extends`), `dest` is the consumer (`super`). The source may contain a subtype of `T`, while the destination can consume `T` or a supertype. This is an excellent real-world example of PECS.

---

# 20. Generics in Spring Boot / Enterprise Applications

## Q47. Where do you use Generics in Spring Boot applications?

**Answer:** Generics are common in enterprise abstractions.

Generic API response:

```java
class ApiResponse<T> {

    private T data;
    private String message;
}
```

Usage: `ApiResponse<UserDto>`, `ApiResponse<OrderDto>`, `ApiResponse<List<UserDto>>`.

Generic repository abstraction:

```java
interface BaseRepository<T, ID> {

    T findById(ID id);

    void save(T entity);
}
```

Generic mapper:

```java
interface Mapper<S, T> {
    T map(S source);
}
```

Generic page result:

```java
class PageResult<T> {

    private List<T> content;
    private long totalElements;
}
```

Generics make these contracts explicit and type-safe.

---

## Q48. How can Generics improve API design?

**Answer:** Instead of `ApiResponse response; Object data = response.getData();`, use `ApiResponse<UserDto> response; UserDto user = response.getData();`. Benefits: compile-time safety, clearer contracts, better IDE support, safer refactoring, fewer casts, better readability.

---

# 21. Senior Production Scenarios

## Q49. You receive `List<?>`. How would you process it?

**Answer:** If I only need to read:

```java
static void logValues(List<?> values) {

    for (Object value : values) {
        log.info("{}", value);
    }
}
```

If I need to add values, I'd identify the actual type relationship and use a suitable type parameter or lower-bounded wildcard. The key is to design the generic contract around the operation rather than adding wildcards blindly.

---

## Q50. A developer changes `List<T>` to `List<Object>` to make an API more flexible. Is that correct?

**Answer:** Usually no. `List<Object>` is not a supertype of every parameterized `List` — `List<String>` cannot be passed to a `List<Object>` parameter. If the method only needs to inspect values, `List<?>` is usually more appropriate. If it consumes a specific type, `List<? super String>` may be appropriate.

---

## Q51. A production API uses raw `List` everywhere. What problems would you expect?

**Answer:** Unchecked warnings, runtime casts, possible `ClassCastException`, weak contracts, reduced IDE assistance, greater refactoring risk, possible heap pollution. I would migrate toward parameterized types, prioritizing public APIs and high-risk code paths first.

---

## Q52. You need to write a utility that copies values from one collection to another. How would you design it?

**Answer:** Use PECS:

```java
static <T> void copy(
        List<? extends T> source,
        List<? super T> destination) {

    for (T item : source) {
        destination.add(item);
    }
}
```

`source` produces T (`extends`), `destination` consumes T (`super`).

---

# 22. Common Interview Traps

## Q53. Can you overload methods only by changing generic type arguments?

**Answer:** No.

```java
void process(List<String> list) { }
void process(List<Integer> list) { }
```

After erasure both have approximately `void process(List list)`, causing a name clash. Generic type arguments cannot distinguish these overloads.

---

## Q54. Can a static field use a class's type parameter?

**Answer:** No.

```java
class Box<T> {
    // static T value; // invalid
}
```

`T` belongs to each parameterized instance/type use, while a static field belongs to the class.

---

## Q55. Can a class implement the same generic interface twice with different type arguments?

**Answer:** No. A class cannot implement both `Comparable<String>` and `Comparable<Integer>`, because that would create conflicting parameterizations of the same generic interface.

---

## Q56. Are Generics a replacement for polymorphism?

**Answer:** No — they solve different problems. Generics primarily provide type parameterization, compile-time type safety, and reusable type relationships. Polymorphism primarily provides behavioral substitution, dynamic dispatch, and runtime implementation selection. They are frequently used together.

---

# 23. Senior Design Questions

## Q57. Is PECS a rule that should always be applied literally?

**Answer:** No. PECS is a guideline for designing flexible APIs. Adding wildcards everywhere can make APIs harder to understand — `List<T>` may be clearer when the exact type relationship matters. The goal is to maximize useful flexibility without making the API unnecessarily complex.

---

## Q58. Can too many generic bounds make an API worse?

**Answer:** Yes.

```java
<T extends A & B & C & D>
```

may be technically valid but difficult to understand and use. Introduce bounds when they provide meaningful operations or enforce an important contract — avoid abstraction for abstraction's sake.

---

# 24. Most Asked Generics Interview Questions — Priority List

### Must Know

1. What are Generics and why were they introduced?
2. What problem do Generics solve?
3. Type parameter vs type argument?
4. What is a generic class?
5. What is a generic method?
6. What are bounded type parameters?
7. What does `<T extends Number>` mean?
8. What is a wildcard?
9. `List<Object>` vs `List<?>`?
10. What is `? extends T`?
11. What is `? super T`?
12. Why can't you add to `List<? extends Number>`?
13. Why can you add to `List<? super Integer>`?
14. What can you safely read from `? super T`?
15. What is PECS?
16. Explain Producer Extends.
17. Explain Consumer Super.
18. Why are Java Generics invariant?
19. Is `List<Integer>` a subtype of `List<Number>`?
20. What is Type Erasure?
21. Why does Java use Type Erasure?
22. Why can't you use `new T()`?
23. Why can't you create `new T[]`?
24. What are raw types?
25. What is heap pollution?

### Senior-Level

26. Generic wildcard vs type parameter?
27. `? extends T` vs `<T extends X>`?
28. Why does `Collections.copy()` use PECS?
29. What are recursive bounds?
30. Why use `Comparable<T>`?
31. Why can generic varargs be unsafe?
32. What is a reifiable type?
33. Why does erasure affect method overloading?
34. How do Generics improve API design?
35. How would you design a generic repository/API response?
36. When should you avoid wildcards?
37. When can Generics make an API over-engineered?
38. How do raw types create heap pollution?
39. How would you migrate a legacy raw-type API?
40. Explain Generics to an interviewer using a production example.

---

# 25. Quick-Fire Revision

### Generic type parameter

```java
<T>
```

Named type placeholder.

### Type argument

```java
List<String>
```

`String` is the type argument.

### Upper type bound

```java
<T extends Number>
```

`T` must be `Number` or a subtype.

### Unbounded wildcard

```java
<?>
```

Unknown type.

### Upper-bounded wildcard

```java
<? extends Number>
```

Unknown subtype of `Number`. Safe to read as `Number`; cannot add arbitrary non-null values.

### Lower-bounded wildcard

```java
<? super Integer>
```

Unknown supertype of `Integer`. Can add `Integer`; safe reads are `Object`.

### PECS

```text
Producer → extends
Consumer → super
```

### Invariance

```text
Integer extends Number

but

List<Integer> is NOT a List<Number>
```

### Type Erasure

Generic type information is primarily used by the compiler and is erased from ordinary runtime representation.

### Raw type

```java
List
```

Avoid in new code.

### Generic method

```java
static <T> T identity(T value)
```

### Recursive bound

```java
<T extends Comparable<T>>
```

---

# 26. One-Minute Senior Interview Answer

> Java Generics provide compile-time type safety and allow us to build reusable APIs without explicit casts. We can define generic classes and methods using type parameters such as `T`, and restrict them with bounds such as `T extends Number`. Wildcards represent unknown types: `? extends T` is normally used for producers, while `? super T` is used for consumers, which gives us the PECS principle—Producer Extends, Consumer Super. Java Generics are invariant, so `List<Integer>` is not a `List<Number>`. At runtime, Java primarily relies on type erasure, which is why we cannot directly create `new T()`, create generic arrays, or overload methods only by their generic type arguments. In production, I use Generics for collection APIs, repository abstractions, API response wrappers, mappers, and reusable utilities, while avoiding raw types, unnecessary wildcards, heap-pollution risks, and overly complex generic bounds.

---

# 27. Final Interview Checklist

## Type Parameters & Bounds

- [ ] Generic class
- [ ] Generic method
- [ ] Type parameter vs type argument
- [ ] `T`, `E`, `K`, `V`
- [ ] Bounded type parameters
- [ ] Multiple bounds
- [ ] Recursive bounds
- [ ] Type inference

## Wildcards

- [ ] `?`
- [ ] `? extends`
- [ ] `? super`
- [ ] `List<Object>` vs `List<?>`
- [ ] `List<T>` vs `List<?>`
- [ ] Reading vs writing
- [ ] Invariance

## Type Erasure

- [ ] Why erasure exists
- [ ] Runtime implications
- [ ] `new T()` restriction
- [ ] Generic array restriction
- [ ] Method signature/name clash
- [ ] Reifiable types
- [ ] Raw types
- [ ] Heap pollution
- [ ] Generic varargs

## PECS

- [ ] Producer Extends
- [ ] Consumer Super
- [ ] Copy example
- [ ] `Collections.copy()`
- [ ] API design

## Senior Reasoning

- [ ] Explain why, not only syntax
- [ ] Understand invariance
- [ ] Know when wildcard is preferable
- [ ] Know when a type parameter is preferable
- [ ] Explain compiler vs runtime behavior
- [ ] Explain trade-offs
- [ ] Connect Generics to Spring Boot/API design
- [ ] Recognize heap-pollution risks
- [ ] Avoid unnecessary abstraction

---

# Final Takeaway

For an **experienced Senior Java/Spring Boot interview**, the four Generics concepts that should be completely clear are:

```text
                    GENERICS
                       |
        +--------------+--------------+
        |              |              |
   Type Bounds     Wildcards      Type Erasure
        |              |              |
   T extends X     ? extends T     Compile-time
   T extends A&B   ? super T       type safety
        |              |              |
        +--------------+--------------+
                       |
                      PECS
                       |
          Producer Extends
          Consumer Super
```

The strongest interview explanation follows:

> **Definition → Problem solved → Compiler behavior → Example → Why it is safe → Trade-off → Production use case → Interview trap → Senior follow-up.**