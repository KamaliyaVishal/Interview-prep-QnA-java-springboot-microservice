# Core Java Fundamentals — Interview Prep — Senior Java Developer

---

## SECTION 1: JDK vs JRE vs JVM ARCHITECTURE

### Q1. Difference between JDK, JRE, and JVM?
**Answer:**
- **JVM (Java Virtual Machine):** The abstract runtime engine that executes bytecode. Platform-specific implementation, platform-independent bytecode — this is *why* Java is "write once, run anywhere." Contains the class loader subsystem, runtime data areas (heap/stack/metaspace), and execution engine (interpreter + JIT compiler).
- **JRE (Java Runtime Environment):** JVM + core libraries (`java.lang`, `java.util`, etc.) + supporting files needed to **run** Java applications. No compiler.
- **JDK (Java Development Kit):** JRE + development tools (`javac`, `javadoc`, `jar`, `jdb`, `jshell`, profilers). Needed to **develop** Java applications.

**Relationship:** `JDK ⊃ JRE ⊃ JVM`

**Senior angle:** Since Java 11, Oracle stopped shipping JRE as a separate standalone download — you now build custom runtime images with `jlink` (modular runtime, only include modules you need, shrinking deployment footprint) — worth mentioning to show you're current beyond Java 8 knowledge.

**Follow-up:** "Is JVM platform-independent?" → No — JVM itself is platform-*dependent* (different native binaries for Windows/Linux/Mac); it's the **bytecode** that's platform-independent, which is the actual source of Java's portability.

---

### Q2. Explain JVM architecture / internal components.
**Answer — three major subsystems:**

1. **Class Loader Subsystem** — loads, links, and initializes `.class` files (see Section 3 for depth).
2. **Runtime Data Areas:**
   - **Method Area / Metaspace** — class metadata, static variables, constant pool (shared across all threads).
   - **Heap** — all objects and arrays (shared across all threads).
   - **Stack** — one per thread; stores stack frames (local variables, operand stack, partial results) per method call.
   - **PC (Program Counter) Register** — one per thread; tracks the current executing instruction.
   - **Native Method Stack** — for native (JNI/C) method calls.
3. **Execution Engine:**
   - **Interpreter** — executes bytecode line by line (slow startup but immediate execution).
   - **JIT (Just-In-Time) Compiler** — compiles "hot" bytecode paths to native machine code at runtime for speed (C1 client compiler for fast startup, C2 server compiler for peak throughput; tiered compilation combines both).
   - **Garbage Collector** — reclaims heap memory (Section 2).

**Senior talking point:** Mention **tiered compilation** (`-XX:+TieredCompilation`, default since Java 8) — JVM starts with interpreter/C1 for fast startup, then promotes hot methods to C2-optimized native code as they're called repeatedly. This nuance often distinguishes a senior candidate from someone who just memorized "interpreter + JIT."

---

### Q3. How does `javac` differ from `java` command? What happens when you run `java HelloWorld`?
**Answer:** `javac` compiles `.java` source → `.class` bytecode. `java` launches the JVM, which:
1. Class loader loads `HelloWorld.class` (loading → linking [verify, prepare, resolve] → initialization).
2. JVM locates `main(String[] args)` method.
3. Execution engine starts interpreting/JIT-compiling bytecode.
4. JVM allocates thread stack for `main` thread, begins execution.
5. On completion (or uncaught exception), JVM triggers shutdown hooks, then exits.

**Follow-up trap:** "Can a `.class` file run on a different OS than it was compiled on?" → Yes — that's the entire point of bytecode; only the JVM implementation is OS-specific, not the bytecode itself.

---

### Q4. What's the difference between JIT compilation and AOT (Ahead-of-Time) compilation? Have you heard of GraalVM?
**Answer:** JIT compiles hot code paths **at runtime**, adapting to actual usage patterns (profile-guided optimization) — but incurs a "warm-up" period. AOT (e.g., **GraalVM Native Image**) compiles the entire application to a native binary **before** runtime — near-instant startup, lower memory footprint, no warm-up, but loses runtime profile-guided optimizations and has stricter reflection/dynamic-class-loading constraints.

**Real relevance:** GraalVM Native Image is increasingly used with **Spring Boot 3+ / Spring Native** for serverless/Kubernetes cold-start-sensitive workloads (e.g., AWS Lambda) — mentioning this shows awareness of modern JVM ecosystem trends beyond Java 8.

---

## SECTION 2: MEMORY MANAGEMENT (Heap, Stack, Metaspace)

### Q5. Explain the Java memory areas: Heap vs Stack vs Metaspace.
| Area | Stores | Shared? | Managed by GC? |
|---|---|---|---|
| **Heap** | All objects, arrays, instance variables | Shared across threads | Yes |
| **Stack** | Method call frames, local (primitive) variables, object references | Per-thread, private | No (auto-popped on method return) |
| **Metaspace** (Java 8+) | Class metadata, method bytecode, constant pool | Shared | Yes (but grows into native/OS memory, not heap) |

**Senior detail — PermGen → Metaspace migration (very commonly asked):** Before Java 8, class metadata lived in **PermGen**, a fixed-size region **within the heap**, causing frequent `OutOfMemoryError: PermGen space` in apps with heavy classloading (e.g., app servers doing hot redeployment, frameworks generating proxy classes dynamically like Spring AOP/Hibernate). Java 8 replaced PermGen with **Metaspace**, allocated in **native (off-heap) memory**, growing dynamically by default (bounded only by `-XX:MaxMetaspaceSize` if set) — largely eliminating this class of OOM, though it can still occur if you have severe classloader leaks (e.g., repeated hot-redeploys without unloading old classloaders).

**Follow-up:** "What causes StackOverflowError vs OutOfMemoryError?" → `StackOverflowError` = stack frame exceeds thread stack size (usually deep/infinite recursion). `OutOfMemoryError: Java heap space` = heap exhausted, GC can't reclaim enough (memory leak, oversized cache, insufficient `-Xmx`).

---

### Q6. Explain heap generational structure — Young Gen, Old Gen, Eden, Survivor spaces.
**Answer:** Heap is divided based on the **generational hypothesis**: most objects die young, so segregating them lets GC scan young objects cheaply and frequently, without repeatedly scanning long-lived objects.

- **Young Generation** = Eden + 2 Survivor spaces (S0, S1).
  - New objects allocated in **Eden**.
  - When Eden fills → **Minor GC** triggers: surviving objects copied to a Survivor space; dead objects reclaimed.
  - Objects surviving multiple Minor GCs (age threshold, `-XX:MaxTenuringThreshold`) are **promoted** to Old Gen.
- **Old Generation (Tenured):** long-lived objects. Cleaned by **Major/Full GC**, which is far more expensive (scans/compacts larger region, often stop-the-world).

**Real production example:** A service caching large objects in a `static Map` without eviction causes those objects to survive Minor GCs and get promoted to Old Gen, where they accumulate — eventually triggering frequent, long Full GC pauses (visible as latency spikes in production, often the actual root cause behind "random slowdowns every few minutes").

**Follow-up:** "What's a Minor GC vs Major GC vs Full GC?" → Minor = Young Gen only, fast, frequent. Major = Old Gen collection. Full GC = both Young + Old (+ Metaspace in some collectors), stop-the-world, most expensive — a Full GC pause is the classic cause of latency SLA breaches interviewers want you to have debugged.

---

### Q7. Stack memory — how does it work? What's stored per stack frame?
**Answer:** Each thread gets its own stack (`-Xss` controls size, default ~512KB–1MB depending on OS/JVM). Every method call pushes a new **stack frame** containing:
- Local variables (primitives, and object *references* — not the objects themselves, which live on heap).
- Operand stack (for intermediate computation).
- Reference to the runtime constant pool for the current method.
- Return address.

When the method returns, the frame is popped — **automatic, no GC involved for stack memory.**

**Common mistake in interviews:** Saying "objects are stored on the stack." **Wrong** — objects always live on the heap in Java; only **primitive local variables and references to heap objects** live on the stack. (Java has no stack-allocated objects, unlike C++; this is a frequent trip-up even for experienced devs.)

---

### Q8. What causes a memory leak in Java if it has automatic GC?
**Answer:** A "leak" in Java means objects are **still reachable** (referenced) but never actually needed again — GC can't reclaim what's still referenced, even if the app logically doesn't need it anymore.

**Common real-world causes (high-value senior answer):**
1. **Static collections** growing unbounded (e.g., a static `Map` used as a cache with no eviction).
2. **Unclosed resources** (DB connections, streams, HTTP clients) holding references indirectly.
3. **Listener/callback registration without deregistration** (classic Swing/observer pattern leak, also common in Spring event listeners).
4. **ThreadLocal not cleaned up** in pooled-thread environments (Tomcat worker threads live forever; if `ThreadLocal.remove()` isn't called, the value — and everything it references — leaks for the *lifetime of the pooled thread*, not the request).
5. **Inner class holding implicit reference to outer class** (non-static inner classes hold an implicit `this$0` reference — if the inner class instance outlives intended scope, it keeps the outer instance alive too).
6. Long-lived caches without proper eviction policy (should use **Caffeine/Guava Cache** with size/TTL-based eviction, not a raw `HashMap`).

**Debugging approach (interviewer wants this):** Take a heap dump (`jmap -dump:live,format=b,file=heap.hprof <pid>`), analyze with **Eclipse MAT** (Memory Analyzer Tool), look at the "Dominator Tree" / "Leak Suspects" report to find what's retaining large object graphs, trace back the GC roots.

---

### Q9. -Xms vs -Xmx vs -Xss — what do they control?
- `-Xms` — initial heap size.
- `-Xmx` — maximum heap size.
- `-Xss` — thread stack size (per-thread).

**Best practice:** In production, set `-Xms == -Xmx` to avoid runtime heap resizing overhead (avoids repeated OS memory allocation calls as heap grows) — this is a well-known senior-level tuning tip interviewers listen for.

---

## SECTION 3: GARBAGE COLLECTION (GC Algorithms, G1, ZGC)

### Q10. How does Garbage Collection work at a high level? Mark-Sweep-Compact.
**Answer:** GC identifies objects unreachable from **GC Roots** (local variables on active thread stacks, static fields, JNI references) and reclaims their memory. The classic algorithm:
1. **Mark** — traverse the object graph from GC roots, mark all reachable objects as "live."
2. **Sweep** — reclaim memory occupied by unmarked (unreachable) objects.
3. **Compact** — move surviving objects together to eliminate fragmentation (not all collectors compact every cycle).

**Follow-up:** "Is GC deterministic — can you force it to run?" → `System.gc()` is only a **hint/suggestion** to the JVM, not a guarantee — the JVM may ignore it. Relying on `System.gc()` in production code is considered an anti-pattern.

---

### Q11. What are the different GC algorithms/collectors in the JVM, and when would you choose each?
| Collector | Approach | Best For |
|---|---|---|
| **Serial GC** | Single-threaded, stop-the-world | Small apps, single-core, client-side |
| **Parallel GC** (Throughput collector) | Multi-threaded stop-the-world | Batch jobs, throughput > latency |
| **CMS (Concurrent Mark Sweep)** | Concurrent marking, minimal pauses | **Deprecated/removed in Java 14+** — was common pre-G1 |
| **G1 (Garbage First)** | Region-based, concurrent + incremental compaction | **Default since Java 9** — balanced throughput/latency for most enterprise apps |
| **ZGC** | Region-based, fully concurrent, sub-millisecond pauses | Very large heaps (multi-GB to TB), ultra-low-latency requirements |
| **Shenandoah** | Similar goals to ZGC (concurrent compaction) | Low-latency alternative (Red Hat-developed) |

**Senior answer for "which would you use":** "For most Spring Boot microservices, G1 (the default) is fine out of the box. If I'm running a latency-critical service (e.g., trading systems, real-time bidding) with large heaps where even G1's pause times (typically tens of ms) are unacceptable, I'd evaluate **ZGC**, which targets sub-millisecond pauses regardless of heap size, at some throughput cost. I'd always validate the choice with actual load testing and GC logging (`-Xlog:gc*`) rather than assuming."

---

### Q12. Explain G1 GC in more depth — how is it different from older collectors?
**Answer:** G1 divides the heap into many equal-sized **regions** (not fixed contiguous Young/Old spaces) — each region can dynamically serve as Eden, Survivor, or Old. G1:
- Prioritizes collecting regions with the **most garbage first** (hence "Garbage First") for efficiency.
- Performs **concurrent marking** (mostly alongside application threads) to identify live objects in Old regions.
- Compacts incrementally, avoiding the fragmentation problem CMS had (CMS never compacted, leading to eventual Full GC fallback under fragmentation).
- Lets you set a **pause-time goal** (`-XX:MaxGCPauseMillis=200`) — G1 tries to meet it by choosing how many regions to collect per cycle (soft goal, not a hard guarantee).

**Follow-up:** "What's a mixed GC in G1?" → After concurrent marking, G1 does "mixed" collections — cleaning Young regions **plus** some Old regions identified as garbage-heavy — distinguishing it from a pure Young-only Minor GC.

---

### Q13. What is ZGC and why does it achieve sub-millisecond pauses?
**Answer:** ZGC uses **colored pointers** and **load barriers** to do almost all GC work (marking, relocating, remapping references) **concurrently with application threads**, rather than stopping the world. It reserves metadata bits in the object pointer itself to track object state (marked/relocated), letting it perform reference updates on-the-fly without a global pause. This makes GC pause times **independent of heap size** — a 2GB heap and a 2TB heap both see sub-millisecond pauses, which is why it's chosen for very large-heap, latency-sensitive systems.

**Trade-off to mention:** Slightly higher CPU/throughput overhead than G1 due to more concurrent work and barrier checks — not "free," it's a throughput-for-latency trade.

---

### Q14. What is a "stop-the-world" pause? Why can't we eliminate it entirely?
**Answer:** A stop-the-world (STW) pause freezes **all application threads** while GC performs certain operations that require a consistent view of the heap (e.g., initial marking phase, some compaction operations) — without this, objects could move or references could change mid-scan, corrupting the GC's object graph traversal. Even ZGC/Shenandoah have *very brief* STW phases (root scanning) — true zero-pause GC doesn't fully exist yet; the goal is minimizing STW duration and frequency, not eliminating it entirely.

---

### Q15. How do you tune/troubleshoot GC in a production Spring Boot application?
**Senior answer (this is what interviewers want, not theory):**
1. Enable GC logging: `-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=50M`.
2. Analyze with **GCEasy** or **GCViewer** — look at pause frequency, pause duration, throughput %, and whether Full GCs are occurring (red flag if frequent).
3. Correlate GC pause spikes with app latency spikes (APM tool timeline).
4. Check for **premature promotion** (objects promoted to Old Gen too early due to undersized Young Gen) — a common cause of frequent Old Gen pressure; fix by tuning `-Xmn` (Young Gen size) or survivor ratio.
5. Check for large object allocation patterns causing Eden to fill rapidly (e.g., excessive `String` concatenation in loops, large JSON payload parsing without streaming).
6. Consider switching collectors only after confirming default (G1) tuning isn't sufficient — collector switch is a last resort, not first.

---

## SECTION 4: CLASSLOADERS (Bootstrap, Extension, Application)

### Q16. Explain the ClassLoader hierarchy in Java.
**Answer — three built-in loaders, parent-delegation model:**
1. **Bootstrap ClassLoader** — loads core JDK classes (`java.lang.*`, `java.util.*`) from `rt.jar` (pre-Java 9) or the modular runtime (`java.base` module, Java 9+). Written in native code, has no Java-visible parent (`getClassLoader()` returns `null`).
2. **Extension/Platform ClassLoader** — loads classes from `ext` directory (pre-Java 9) or platform modules (Java 9+, renamed to **Platform ClassLoader**).
3. **Application/System ClassLoader** — loads classes from the application classpath (your own `.jar`/`.class` files). This is the default loader for classes you write.

**Custom ClassLoaders** can be written by extending `ClassLoader` — used by app servers (Tomcat per-webapp isolation), OSGi, plugin frameworks, and frameworks like Spring Boot's executable JAR loader (which uses a custom `LaunchedURLClassLoader` to load nested JARs from a fat JAR).

---

### Q17. What is the parent delegation model? Why does it exist?
**Answer:** When a class needs loading, a class loader **first delegates the request to its parent** before trying to load it itself. Only if the parent can't find the class does the child attempt to load it.

**Why:** Security and consistency — prevents user code from defining a fake `java.lang.String` (or `java.lang.Object`, etc.) and injecting it to override core JDK behavior. Since `Bootstrap` always gets first chance and already has the real `java.lang.String` loaded, any attempted override by application code is simply ignored/rejected (loaded from a different, "untrusted" loader would even yield a `LinkageError` if it collides with an already-loaded class name in a security-sensitive package).

**Follow-up:** "Can you break the parent delegation model?" → Yes, by writing a custom class loader that overrides `loadClass()` to check itself *first* — this is exactly what app servers do for classloader isolation (each deployed WAR gets its own classloader so two apps can use different versions of the same library without conflict) — Tomcat, OSGi, and Spring Boot's layered JARs all break/customize strict delegation intentionally.

---

### Q18. Bootstrap vs Extension vs Application ClassLoader — what happens if two of them try to load the same class name?
**Answer:** Because of parent delegation, this normally can't create a runtime conflict — the *first* loader up the hierarchy (closest to Bootstrap) that can resolve the class wins, and it's cached; children never get to override it. But — **two sibling classloaders (not in a parent-child relationship)** *can* each load their own copy of the same-named class independently — this is exactly how app servers isolate different web apps' dependency versions from each other (two WARs each with their own `commons-lang-2.x` vs `commons-lang-3.x`, no collision, because each has its own isolated classloader).

**Follow-up trap:** "If class `com.foo.Bar` is loaded by two different classloaders, are the resulting `Class` objects equal?" → **No.** JVM identity of a class = (fully qualified class name + defining ClassLoader). Same source, two loaders → two distinct `Class` objects → `instanceof` checks/casts between them **fail**, causing the notorious `ClassCastException: com.foo.Bar cannot be cast to com.foo.Bar` (same name printed, different loader) — a classic, very confusing production bug in app-server/plugin environments. This is a strong senior-differentiator question.

---

### Q19. What are the phases of class loading — Loading, Linking, Initialization?
**Answer:**
1. **Loading:** Class loader reads `.class` bytecode, creates a `Class` object in the Metaspace.
2. **Linking:**
   - **Verification** — bytecode verifier checks structural correctness/safety (prevents malformed/malicious bytecode from crashing the JVM or bypassing access checks).
   - **Preparation** — allocates memory for static fields, sets **default** values (not the actual assigned values yet — e.g., `static int x = 5` gets `x = 0` at this stage).
   - **Resolution** — symbolic references (e.g., method/field names as strings in the constant pool) resolved to direct references (optional — can be lazy, resolved on first use).
3. **Initialization:** Static initializer blocks and static field assignments actually execute, in source-code order — this is when `x = 5` actually happens. Triggered by first active use (instantiation, static method call, static field access — but **not** by simply declaring a reference variable of that type).

**Follow-up:** "When exactly does a class get initialized?" → Lazily, on first active use — e.g., `MyClass obj;` does NOT trigger initialization, but `new MyClass()`, `MyClass.staticMethod()`, or accessing a non-constant static field does.

---

### Q20. How does Spring Boot's executable "fat JAR" classloading work differently from a normal JAR?
**Answer:** A standard JVM classloader can't load nested JARs-within-a-JAR directly. Spring Boot ships a custom `JarLauncher` + `LaunchedURLClassLoader` that knows how to read `BOOT-INF/classes` and `BOOT-INF/lib/*.jar` (nested jars) from within the single fat JAR, constructing the classpath at startup. This is a good example to cite if asked "have you customized/understood classloading beyond textbook JDK behavior" — shows real framework-internals awareness.

---

## SECTION 5: DATA TYPES & TYPE CASTING

### Q21. Primitive types vs Reference types — key differences?
**Answer:**
- **Primitives** (`byte, short, int, long, float, double, char, boolean`): fixed size, stored directly (on stack if local variable), default values exist, **not objects**, no methods, compared with `==` by value.
- **Reference types** (objects, arrays, `String`, wrapper classes, custom classes): variable holds a **reference (pointer)** to heap-allocated data; default value is `null`; compared with `==` by reference identity (unless overridden via `.equals()`).

**Size table (senior-level precision matters here):**
| Type | Size | Range |
|---|---|---|
| byte | 1 byte | -128 to 127 |
| short | 2 bytes | -32,768 to 32,767 |
| int | 4 bytes | -2^31 to 2^31-1 |
| long | 8 bytes | -2^63 to 2^63-1 |
| float | 4 bytes | ~7 decimal digits precision |
| double | 8 bytes | ~15-16 decimal digits precision |
| char | 2 bytes | 0 to 65,535 (UTF-16) |
| boolean | JVM-dependent (not precisely specified) | true/false |

---

### Q22. Autoboxing/Unboxing — what's the performance/correctness trap?
**Answer:** Autoboxing auto-converts primitive ↔ wrapper (`int` ↔ `Integer`). 

**Trap 1 — NullPointerException:** 
```java
Integer count = null;
int x = count;  // NPE at unboxing, easy to miss in refactors from Integer to int
```
**Trap 2 — Integer caching (`==` comparison bug):**
```java
Integer a = 127, b = 127;
System.out.println(a == b);   // true — cached (-128 to 127 pool)
Integer c = 200, d = 200;
System.out.println(c == d);   // false — outside cache range, different objects!
```
**Why:** `Integer` caches values -128 to 127 (`IntegerCache`, JLS-mandated minimum range) for autoboxing performance. This creates a classic interview trap and a **real production bug source** — always use `.equals()` for wrapper comparison, never `==`.

**Trap 3 — Performance in loops:**
```java
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i;   // unboxes sum, adds, reboxes — creates a new Long object EVERY iteration!
}
```
This silently creates a million `Long` objects — should use primitive `long sum` instead. A classic senior-level performance-review catch.

---

### Q23. Implicit vs Explicit type casting (widening vs narrowing conversion)
**Answer:**
- **Widening (implicit):** smaller type → larger type, automatic, no data loss. `int → long → float → double`.
- **Narrowing (explicit):** larger type → smaller type, requires explicit cast, **can lose data/precision**.

```java
int i = 100;
long l = i;              // widening, implicit — fine

double d = 100.99;
int x = (int) d;         // narrowing, explicit — x = 100 (fraction truncated, not rounded)

long big = 3_000_000_000L;
int overflowed = (int) big;   // narrowing — overflow, produces garbage value due to bit truncation
```

**Follow-up:** "What happens when you cast a `double` outside `int` range to `int`?" → Doesn't throw — silently truncates/overflows to `Integer.MAX_VALUE`/`MIN_VALUE` boundary behavior or wraps depending on the value — this "silent failure" is exactly why interviewers ask it: it's a real bug class (unchecked overflow), unlike languages that throw on overflow.

---

### Q24. What's the difference between casting an object reference vs casting a primitive?
**Answer:** Casting a primitive **converts the actual value** (bit-level reinterpretation/truncation). Casting an object reference **doesn't change the object at all** — it only changes the **compile-time type** the reference is treated as, enabling you to call subclass-specific methods; the underlying object's actual runtime type is unchanged. An invalid object cast (casting to an unrelated type) throws `ClassCastException` **at runtime**, not compile time (unless the compiler can statically prove it's impossible).

```java
Object obj = "hello";
String s = (String) obj;       // valid — actual object IS a String

Object obj2 = new ArrayList<>();
String bad = (String) obj2;    // compiles fine, throws ClassCastException at RUNTIME
```

---

### Q25. instanceof operator and pattern matching (Java 16+ enhancement — mention if relevant)
```java
if (obj instanceof String) {
    String s = (String) obj;   // traditional
}
// Java 16+ pattern matching for instanceof — reduces boilerplate
if (obj instanceof String s) {
    System.out.println(s.length());   // s already cast, scoped to the if-block
}
```
Worth mentioning briefly to show you're aware of modern Java even while primarily working in Java 8 — shows growth mindset.

---

## SECTION 6: PASS BY VALUE vs PASS BY REFERENCE

### Q26. Is Java pass-by-value or pass-by-reference? (One of the MOST asked "gotcha" questions at every level)
**Answer: Java is ALWAYS strictly pass-by-value. There is no pass-by-reference in Java, period.**

The confusion arises because for objects, the **value being passed is a copy of the reference (the memory address/pointer)** — not the object itself, and not a reference-to-the-reference. So:
- You **can** mutate the object's internal state through that copied reference (because both the original and copied reference point to the *same* heap object).
- You **cannot** reassign the caller's original reference to point elsewhere, because the method only has a *copy* of that reference — reassigning the copy doesn't affect the original variable.

```java
public static void modifyValue(int x) {
    x = 100;   // only changes the local copy
}
public static void modifyObject(StringBuilder sb) {
    sb.append(" world");   // mutates the SAME object both references point to — visible to caller
}
public static void reassignObject(StringBuilder sb) {
    sb = new StringBuilder("new object");   // reassigns the LOCAL copy of the reference only
}

public static void main(String[] args) {
    int num = 5;
    modifyValue(num);
    System.out.println(num);   // 5 — unchanged

    StringBuilder original = new StringBuilder("hello");
    modifyObject(original);
    System.out.println(original);   // "hello world" — mutated, visible!

    reassignObject(original);
    System.out.println(original);   // still "hello world" — reassignment inside method didn't leak out
}
```

**How to phrase this precisely in an interview (this exact wording gets senior candidates strong marks):**
> "Java passes everything by value. For primitives, that value is the data itself. For objects, that value is a copy of the *reference* — so the method can mutate the object the reference points to, but reassigning the parameter inside the method never affects the caller's original variable. This is fundamentally different from true pass-by-reference (like C++ `&` references or `ref`/`out` in C#), where reassignment inside the callee *would* be visible to the caller."

**Common mistake to explicitly call out:** Saying "objects are passed by reference" without the nuance is the single most common wrong/incomplete answer — interviewers specifically probe this to filter shallow understanding.

---

### Q27. Given this code, what's the output? (Classic live-coding trap based on Q26)
```java
public static void swap(StringBuilder a, StringBuilder b) {
    StringBuilder temp = a;
    a = b;
    b = temp;
}
public static void main(String[] args) {
    StringBuilder x = new StringBuilder("X");
    StringBuilder y = new StringBuilder("Y");
    swap(x, y);
    System.out.println(x + " " + y);   // ?
}
```
**Answer:** Prints `X Y` — unchanged. The swap only exchanges the **local copies** of the references inside the method; `x` and `y` in `main` still point to their original objects. This is the definitive proof-by-example that Java has no true pass-by-reference — a very popular whiteboard/live-coding question.

---

### Q28. If arrays are objects, does passing an array to a method allow the method to change its contents? Can it replace the whole array?
```java
public static void modifyArray(int[] arr) {
    arr[0] = 999;              // mutates existing array — VISIBLE to caller (same object)
}
public static void reassignArray(int[] arr) {
    arr = new int[]{1, 2, 3};  // reassigns local copy of reference — NOT visible to caller
}
```
**Answer:** Same rule as any object — arrays are reference types, so element mutation is visible to the caller, but reassigning the array reference inside the method is not. Confirms arrays follow the identical pass-by-value-of-reference semantics as any other object in Java.

---

### Q29. Why does this matter for real production code? (Senior framing)
**Answer:** This principle directly explains behavior developers hit constantly:
- Why mutating a `List`/`Map` parameter inside a service method is visible to the caller (a common **unintended side-effect bug** if you mutate an input collection you were only supposed to read).
- Why Java's `String` being **immutable** matters — since `String` mutation methods like `concat()` always return a *new* `String` rather than mutating in place, `String` parameters behave "as if" pass-by-value from the caller's perspective, even though technically it's still reference-passing under the hood — this is *why* `String` immutability specifically prevents a whole class of aliasing bugs that mutable objects (like `StringBuilder`, custom DTOs) don't protect you from.
- Defensive copying (`new ArrayList<>(inputList)`) is a deliberate technique senior developers use specifically **because** of this semantics — to avoid unintentionally exposing/mutating caller-owned state, especially across module/service boundaries.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Why was PermGen removed in favor of Metaspace?" → Fixed-size PermGen caused frequent OOM under dynamic classloading (proxies, hot-redeploys); Metaspace grows in native memory, avoiding that specific bottleneck.
- "What's the default GC in Java 8 vs Java 11 vs Java 17?" → Java 8: Parallel GC. Java 9–16(ish): G1 became default from Java 9 onward. Java 17: G1 still default, ZGC/Shenandoah production-ready as opt-in.
- "Can you have a `ClassLoader` load two different versions of the same class simultaneously?" → Yes — see Q18, sibling classloader isolation, exactly how app servers/OSGi handle multi-version dependency conflicts.
- "Why is `float`/`double` risky for financial calculations, and what should you use instead?" → Binary floating-point can't represent decimals like 0.1 exactly (base-2 vs base-10 mismatch) → rounding errors compound. Use `BigDecimal` (with `String` constructor, not `double` constructor, to avoid inheriting the imprecision) for currency/financial data — a very common fintech-interview (JPMC/Goldman/Visa) follow-up given your target companies.
- "Does casting a primitive `float` to `int` round or truncate?" → Truncates toward zero, doesn't round (`(int) 4.9 == 4`, not 5).
- "What's the difference between `Integer.valueOf()` and `new Integer()`?" → `valueOf()` uses the integer cache (returns cached instance for -128–127), `new Integer()` always creates a new object (also — `new Integer(int)` is **deprecated since Java 9** for exactly this reason: unnecessary object creation).

---

*Study tip: The pass-by-value question (Q26–Q29) and the Integer-caching `==` trap (Q22) are asked in almost every single Java interview regardless of company tier — make sure you can explain and live-code both without hesitation, since fumbling these signals weaker fundamentals even if your framework knowledge (Spring/microservices) is strong.*
