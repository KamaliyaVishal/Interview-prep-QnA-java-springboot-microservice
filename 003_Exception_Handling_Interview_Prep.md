# Exception Handling — Interview Prep — Senior Java Developer (8+ YOE)

---

## SECTION 1: CHECKED vs UNCHECKED EXCEPTIONS

### Q1. Explain the Java exception hierarchy.
**Answer:**
```
Throwable
 ├── Error                          (JVM-level, not meant to be caught: OutOfMemoryError, StackOverflowError)
 └── Exception
      ├── Checked Exceptions        (IOException, SQLException — compiler-enforced)
      └── RuntimeException          (Unchecked: NullPointerException, IllegalArgumentException, ArithmeticException)
```

- **`Throwable`** — root of everything catchable.
- **`Error`** — represents serious problems the application generally **shouldn't try to catch/recover from** (JVM internal failures, resource exhaustion).
- **`Exception`** — represents conditions applications should handle. Split into checked and unchecked.
- **`RuntimeException`** (and its subclasses) — unchecked; everything else under `Exception` is checked.

**Senior detail:** All exceptions in Java **are objects**, and the entire hierarchy exists because `Throwable implements Serializable` — this matters when exceptions cross JVM boundaries (RMI, distributed systems) or need to be logged/persisted.

---

### Q2. Checked vs Unchecked exceptions — difference, and when do you use each?
| Aspect | Checked | Unchecked |
|---|---|---|
| Compiler enforcement | Must be caught or declared (`throws`) | No compiler enforcement |
| Examples | `IOException`, `SQLException`, `ClassNotFoundException` | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException` |
| Represents | Recoverable conditions **external** to the program (network failure, file not found, DB down) | Programming errors / bugs, or conditions where recovery isn't reasonably expected at the call site |
| Typical use | Force the caller to explicitly acknowledge and handle a foreseeable failure | Signal a bug or invalid usage that shouldn't normally be "handled," just fixed |

**Senior-level decision framework (this is what separates senior from mid-level answers):**
> "I use a checked exception when the caller has a **reasonable chance to recover** and the failure is expected as part of normal operation — e.g., a downstream service being temporarily unavailable. I use an unchecked exception (usually extending `RuntimeException` or a domain-specific subclass) when the failure represents a **programming error or a violated precondition** — e.g., passing a null where it's disallowed, or reaching a state that should be structurally impossible if the code is correct. In modern Spring-based services, I lean heavily toward unchecked exceptions even for many 'expected' failures, because checked exceptions don't compose well with lambdas/streams and clutter method signatures — Spring's own `DataAccessException` hierarchy deliberately converts all the checked `SQLException` subtypes into unchecked exceptions for exactly this reason."

---

### Q3. Why does the Java community (Spring, Hibernate) largely avoid checked exceptions in modern API design?
**Answer:**
1. **They don't compose with functional interfaces/lambdas** — a `Function<T,R>` can't declare `throws IOException`, so checked exceptions inside lambdas force awkward try-catch-wrap-in-RuntimeException boilerplate.
2. **They leak implementation details into abstractions** — a `Repository` interface having to declare `throws SQLException` ties an abstraction to a specific persistence technology.
3. **They encourage "swallow and ignore" anti-patterns** — many developers write `catch (Exception e) {}` just to satisfy the compiler, actively hiding bugs (Section 5 covers this).
4. **Signature pollution compounds up the call stack** — every intermediate method has to either catch or re-declare `throws`, cascading through layers even when most callers can't meaningfully recover anyway.

**This is exactly why Spring's `DataAccessException` hierarchy converts JDBC's checked `SQLException` into a rich unchecked hierarchy** (`DuplicateKeyException`, `DataIntegrityViolationException`, etc.) — you get semantic, typed exceptions without forcing every DAO method signature to declare `throws SQLException`.

---

### Q4. Is `NullPointerException` checked or unchecked? What about `OutOfMemoryError`?
**Answer:** `NullPointerException` extends `RuntimeException` → **unchecked**. `OutOfMemoryError` extends `Error`, **not** `Exception` at all → not "checked" or "unchecked" in the conventional sense, but by the compiler's rule (only checks subclasses of `Exception` excluding `RuntimeException`), it also doesn't require explicit handling. Generally you should **never try to catch and recover from an `Error`** — it signals the JVM itself is in trouble (though some frameworks do catch `Throwable` broadly at a top-level boundary purely for logging before crashing cleanly — worth mentioning as a nuance).

---

### Q5. Can you create a custom exception that extends `Error` instead of `Exception`? Should you?
**Answer:** Technically yes — `Error` is just another `Throwable` subclass, nothing stops you from extending it. **You should not.** Extending `Error` signals "this is a JVM-level catastrophic failure the application can't reasonably recover from," which is almost never true for application-level business logic. Application exceptions should always extend `Exception` (checked) or `RuntimeException` (unchecked) — never `Error`. Interviewers ask this to test whether you understand the *semantic contract*, not just the syntax.

---

## SECTION 2: try-catch-finally, try-with-resources

### Q6. Explain try-catch-finally execution order, including edge cases.
**Answer:** `finally` **always executes**, whether or not an exception was thrown, and even if the `try`/`catch` block contains a `return` statement — with one specific exception: if the JVM exits (`System.exit()`) or the thread is killed inside `try`/`catch`, `finally` does **not** run.

```java
public static int test() {
    try {
        return 1;
    } finally {
        System.out.println("finally runs");   // ALWAYS prints, even though try already has a return
    }
}
```

**Trap follow-up — what if BOTH try and finally have a return statement?**
```java
public static int trickyReturn() {
    try {
        return 1;
    } finally {
        return 2;    // finally's return SILENTLY OVERRIDES try's return — try's return value is discarded
    }
}
// trickyReturn() returns 2, not 1!
```
**Answer:** The `finally` block's `return` **swallows/overrides** any return (or even an exception!) from the `try`/`catch` block. This is a well-known anti-pattern — **never put a `return` (or `throw`) inside a `finally` block**, since it silently discards the original outcome, including silently swallowing an exception that was about to propagate.

**Another classic trap:**
```java
public static int mutationTrap() {
    int x = 1;
    try {
        return x;      // x's VALUE (1) is captured/computed here, for primitives
    } finally {
        x = 2;         // this does NOT change the already-computed return value for primitives
    }
}
// returns 1, not 2 — the return value was effectively "locked in" before finally ran (for primitive return types)
```

---

### Q7. What happens if an exception is thrown in both `try` and `finally`?
```java
try {
    throw new RuntimeException("from try");
} finally {
    throw new RuntimeException("from finally");   // this one WINS — the try's exception is LOST/suppressed
}
// Only "from finally" propagates; "from try" is completely discarded (not even as a suppressed exception here)
```
**Answer:** The exception from `finally` **replaces** the exception from `try` — the original exception is lost entirely (unlike try-with-resources' suppressed-exception mechanism, discussed in Q10, which explicitly preserves both). This is another strong reason to **never throw from a `finally` block** — you can silently lose the real root-cause exception in production, making debugging much harder.

---

### Q8. Can you have try without catch? Can you have multiple catch blocks? What's the ordering rule?
**Answer:**
- `try-finally` (no `catch`) is valid — used purely for guaranteed cleanup, letting exceptions propagate up.
- Multiple `catch` blocks are allowed; the **first matching** catch block executes (checked top to bottom), so **subclasses must be caught before superclasses**, or it's a compile error (unreachable catch block).

```java
try {
    riskyOperation();
} catch (FileNotFoundException e) {      // more specific — must come first
    // ...
} catch (IOException e) {                // more general — must come after
    // ...
} catch (Exception e) {                  // most general — last
    // ...
}
```

**Java 7+ multi-catch (worth mentioning for modern awareness):**
```java
catch (IOException | SQLException e) {   // handle multiple unrelated exception types identically
    log.error("Operation failed", e);
}
```
Note: in multi-catch, the caught variable is **implicitly final**, and the exception types must not be related by inheritance (can't multi-catch a type and its own superclass together — compiler rejects it as redundant).

---

### Q9. try-with-resources — how does it work internally?
**Answer:** Introduced in Java 7 to automatically close resources implementing `AutoCloseable` (or `Closeable`), eliminating manual `finally`-block cleanup boilerplate and its bugs (Q7's lost-exception problem, forgetting to close on early return, etc.).

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // use resources
}   // conn, stmt, rs automatically closed here, in REVERSE order of declaration, even if an exception occurs
```

**Internal mechanism:** The compiler desugars this into a try-finally where `close()` is called automatically in the `finally` block, **in reverse declaration order** (`rs` closed first, then `stmt`, then `conn` — mirroring how you'd manually nest cleanup). Resources must implement `java.lang.AutoCloseable` (interface has one method: `close() throws Exception`) — `Closeable` is a more specific sub-interface (narrows `close()` to `throws IOException`) used by I/O classes.

**Why this matters for production code:** Before Java 7, nested manual `finally` blocks to close multiple resources correctly (each close call itself might throw, potentially masking earlier ones or leaking resources if an early close fails) were extremely error-prone and verbose — try-with-resources is now the **mandatory standard** for any `Connection`/`Stream`/`InputStream`/etc. handling in senior-level code review.

---

### Q10. What are suppressed exceptions in try-with-resources?
```java
class MyResource implements AutoCloseable {
    public void doWork() { throw new RuntimeException("work failed"); }
    public void close() { throw new RuntimeException("close failed"); }
}

try (MyResource r = new MyResource()) {
    r.doWork();
}
// Exception in thread "main" java.lang.RuntimeException: work failed
//   ... 
//   Suppressed: java.lang.RuntimeException: close failed
//   ...
```
**Answer:** Unlike the classic try-finally trap (Q7) where the `finally` exception **silently discards** the try-block's exception, try-with-resources **preserves both** — the primary exception (from the `try` body) propagates normally, and any exception thrown during `close()` is attached to it as a **suppressed exception**, retrievable via `primaryException.getSuppressed()`. This is a genuine, deliberate improvement over manual try-finally and a strong point to raise when asked "why try-with-resources over manual finally."

---

### Q11. Can you use try-with-resources with a resource that doesn't implement AutoCloseable?
**Answer:** No — compile error. Only types implementing `AutoCloseable` (or its `Closeable` sub-interface) are eligible. If you have a legacy resource class that doesn't implement it, you have two options: wrap it (adapter pattern implementing `AutoCloseable`, delegating to the legacy cleanup method), or fall back to manual try-finally. Since Java 9, you can also use **effectively final variables declared outside** the try block directly in the parentheses (not just new declarations) — worth mentioning as a modern refinement:
```java
Connection conn = dataSource.getConnection();
try (conn) {   // Java 9+ — reuses an existing effectively-final variable
    // ...
}
```

---

## SECTION 3: throw vs throws

### Q12. throw vs throws — difference (frequently confused by name alone)
| Aspect | `throw` | `throws` |
|---|---|---|
| Purpose | Actually **throws** an exception instance | **Declares** that a method might throw certain exception types |
| Location | Inside a method body | In the method signature |
| Followed by | A single exception **instance** (`throw new IOException(...)`) | One or more exception **class names**, comma-separated |
| Applicable to | Any exception, checked or unchecked | Primarily meaningful for checked exceptions (compiler-enforced); optional/informational for unchecked |

```java
public void readFile(String path) throws IOException {   // throws — declares the contract
    if (path == null) {
        throw new IllegalArgumentException("path cannot be null");   // throw — actually throws
    }
    // ...
    if (!file.exists()) {
        throw new FileNotFoundException(path);    // throw — actually throws a checked exception
    }
}
```

**Follow-up:** "Is declaring `throws RuntimeException` in a method signature meaningful?" → Not compiler-enforced (unchecked exceptions can be thrown without declaring them), but it can serve as **documentation** for callers about what to expect — some teams do this deliberately for clarity, though it's a style choice, not a requirement.

---

### Q13. If a method overrides a superclass method, can it declare broader checked exceptions in `throws`?
**Answer: No** — an overriding method can only declare the **same, narrower, or no** checked exceptions compared to the overridden method (revisits the OOP overriding rules) — never broader/new checked exceptions, because that would break the Liskov substitution principle: code calling the method via the superclass reference type wouldn't be prepared to catch an exception type it never declared.

```java
class Parent {
    void process() throws IOException { }
}
class Child extends Parent {
    @Override
    void process() throws FileNotFoundException { }   // ✓ narrower (subtype of IOException) — OK
    // @Override
    // void process() throws Exception { }             // ✗ COMPILE ERROR — broader than IOException
}
```
Unchecked exceptions have no such restriction — a subclass can throw any `RuntimeException` regardless of the parent's declaration, since the compiler never enforces unchecked exception declarations anyway.

---

## SECTION 4: CUSTOM EXCEPTIONS

### Q14. How do you design a good custom exception? What should it extend?
**Answer:**
```java
public class InsufficientFundsException extends RuntimeException {
    private final String accountId;
    private final BigDecimal shortfall;

    public InsufficientFundsException(String accountId, BigDecimal shortfall) {
        super(String.format("Account %s short by %s", accountId, shortfall));
        this.accountId = accountId;
        this.shortfall = shortfall;
    }
    public String getAccountId() { return accountId; }
    public BigDecimal getShortfall() { return shortfall; }
}
```

**Senior design principles for custom exceptions:**
1. **Extend `RuntimeException`** (or a shared internal base exception) in modern service code unless there's a strong, deliberate reason for compiler-enforced handling (Q2/Q3) — extending checked `Exception` is increasingly rare in new API design.
2. **Always provide constructors** that accept a `message`, and a `(message, cause)` overload for exception chaining (Section 5).
3. **Carry structured, machine-readable context as fields** (account ID, order ID, error code) — not just a formatted string message — so callers/logging/monitoring can programmatically extract details instead of regex-parsing a message string.
4. **Name it precisely and domain-specifically** — `InsufficientFundsException`, not a generic `BusinessException` with an error-code field, if the distinction matters for callers to catch and handle differently.
5. **Build an exception hierarchy** for your domain (e.g., a base `PaymentException`, with `InsufficientFundsException`, `CardDeclinedException`, `FraudSuspectedException` extending it) — lets callers catch broadly (`catch (PaymentException e)`) or narrowly as needed.
6. Consider whether it needs to be **serializable-safe** if it crosses process boundaries (e.g., via Kafka error events or RMI) — avoid non-serializable fields.

---

### Q15. Should custom exceptions be checked or unchecked in a typical Spring Boot microservice? Justify.
**Answer (strong senior talking point):**
> "In a Spring Boot service, I almost always make custom business exceptions **unchecked**, extending `RuntimeException`. They typically get handled centrally via a `@ControllerAdvice`/`@ExceptionHandler` layer that maps exception types to HTTP status codes and structured error responses — so I don't want every intermediate service/repository method signature cluttered with `throws` declarations for exceptions that ultimately get handled generically at the API boundary, not locally at each call site. I reserve checked exceptions for rare cases where I specifically want to **force** an immediate caller to handle a condition inline rather than let it bubble — which is uncommon in typical layered service architectures."

---

### Q16. Show how you'd map a custom exception to an HTTP response in Spring Boot.
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException ex) {
        ErrorResponse body = new ErrorResponse("INSUFFICIENT_FUNDS", ex.getMessage(), ex.getAccountId());
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(body);
    }

    @ExceptionHandler(Exception.class)   // catch-all fallback — never leak stack traces to clients
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);   // log full detail server-side
        ErrorResponse body = new ErrorResponse("INTERNAL_ERROR", "Something went wrong", null);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }
}
```
**Senior best practice to mention:** Always have a generic `Exception`-level fallback handler as a safety net, log the **full exception with stack trace server-side**, but return a **sanitized, generic message to the client** — never leak internal exception messages/stack traces in API responses (information disclosure risk, frequently flagged in security reviews at fintech companies like Visa/JPMC/Goldman).

---

## SECTION 5: EXCEPTION CHAINING & BEST PRACTICES

### Q17. What is exception chaining? Why does it matter?
**Answer:** Exception chaining preserves the **original ("cause") exception** when wrapping/rethrowing a different exception type — via the `cause` field on `Throwable`, set through the constructor (`new MyException(message, cause)`) or `initCause()`.

```java
try {
    orderRepository.save(order);
} catch (SQLException e) {
    throw new OrderPersistenceException("Failed to save order " + order.getId(), e);   // chains the original cause
}
```

**Why it matters (this is the crux of the interview question):** Without chaining, you **lose the root cause entirely** — you'd only see "Failed to save order 123" in logs with no indication it was actually a connection timeout, a constraint violation, or a deadlock. The full `printStackTrace()` (or logging framework output) shows **both** stack traces: the wrapping exception's trace **and** the original cause's trace (`Caused by: ...`), which is essential for real production debugging.

**Common mistake (explicitly call this out — it's a huge production anti-pattern interviewers want you to name):**
```java
catch (SQLException e) {
    throw new OrderPersistenceException("Failed to save order");   // BUG: cause is dropped entirely!
}
```
This "swallows" the original exception — the stack trace in production logs will point to the wrapping exception's throw site, giving **zero information** about what actually failed underneath. This single mistake has cost countless engineering hours in real production debugging — a very strong point to raise proactively.

---

### Q18. What does `getCause()` and the "Caused by" chain in a stack trace actually represent?
**Answer:** `Throwable.getCause()` returns the exception that triggered the current one (or `null` if none was set). When printed, Java shows the full chain: the outermost exception first, then each `Caused by:` going deeper toward the root cause. For deeply nested wrapping (service → repository → JDBC → driver), you can have multiple levels of chaining — good production logging always prints the **full chain**, not just `e.getMessage()`, specifically so the root cause is visible.

**Follow-up:** "How do you programmatically walk the full cause chain?"
```java
Throwable t = ex;
while (t.getCause() != null) {
    t = t.getCause();
}
// t is now the ROOT cause
```

---

### Q19. Exception handling best practices — what would you flag in a code review? (High-value, very commonly asked open-ended senior question)
**Answer — structure this as a list, it reads well live:**

1. **Never catch `Exception` or `Throwable` broadly** unless at a well-justified top-level boundary (e.g., a global handler, a thread's `run()` method to prevent silent thread death) — catching too broadly hides bugs and catches things you didn't intend to handle (like `NullPointerException` masking a real code defect).
2. **Never swallow exceptions silently** (`catch (Exception e) {}`) — at minimum, log it; ideally, handle or rethrow it. A silently swallowed exception is one of the hardest bugs to ever diagnose in production.
3. **Always chain the cause** when wrapping exceptions (Q17).
4. **Don't use exceptions for normal control flow** — exceptions are expensive (stack trace capture has real cost) and semantically wrong for expected, frequent conditions (e.g., don't throw an exception to signal "record not found" in a hot loop — return `Optional.empty()` instead).
5. **Catch the most specific exception type possible** — don't catch `Exception` when you only actually handle `IOException`; broad catches accidentally swallow unrelated bugs (like an `NPE` from your own logic) and treat them the same as the expected failure.
6. **Don't throw generic exceptions** (`throw new Exception("something went wrong")`) — always use specific, meaningful types so callers can differentiate and handle appropriately.
7. **Clean up resources properly** — try-with-resources over manual finally (Q9-Q10).
8. **Never throw/return from a `finally` block** (Q6-Q7) — silently discards the real outcome.
9. **Fail fast with clear messages** — validate inputs early (`Objects.requireNonNull`, precondition checks) and throw immediately with a message stating exactly what was wrong and what was expected, rather than letting a bad value propagate deep into the call stack before failing with a confusing, disconnected error.
10. **Log at the right level, in the right place** — typically log once, at the point where you have enough context to act or where the exception is finally handled (not at every layer it passes through, which causes duplicate/noisy logs for the same root failure).

---

### Q20. What's the performance cost of exceptions? Why shouldn't they be used for control flow?
**Answer:** Creating an exception is relatively expensive primarily because of **stack trace capture** — `fillInStackTrace()` walks the entire call stack at the point of `throw new XxxException()`, which is costly compared to normal method returns, especially in hot paths/loops. This is why:
- Exceptions used for expected, frequent conditions (e.g., validation failures in a tight loop, or "not found" lookups) are a real, measurable performance anti-pattern in high-throughput services.
- Some performance-critical code paths **override `fillInStackTrace()`** to skip stack trace generation entirely for exceptions that don't need one (rare, advanced technique — worth mentioning if you've encountered it, e.g., in some high-frequency-trading or ultra-low-latency codebases).
- Prefer returning `Optional`, sentinel values, or result-wrapper types (`Either`/`Result` pattern) for **expected** "failure" paths, reserving exceptions for genuinely **exceptional**, non-hot-path conditions.

---

### Q21. How would you handle exceptions in a multi-threaded / async context (e.g., inside a Runnable, ExecutorService task, or CompletableFuture)?
**Answer (ties back to Multithreading prep — good to connect explicitly if both topics come up):**
- An uncaught exception inside a plain `Runnable` run via `Thread` invokes the thread's **uncaught exception handler** (default: prints to `stderr`); the thread simply dies — **other threads are unaffected**, but the exception is easy to miss if you're not specifically watching for it.
- Inside an `ExecutorService.submit(Callable)`, the exception is **captured inside the returned `Future`** and only surfaces when you call `future.get()` (which throws `ExecutionException` wrapping the original) — if you never call `get()`, the exception is **silently swallowed**, a very common production bug.
- Inside `CompletableFuture`, exceptions propagate through the chain and must be handled explicitly via `.exceptionally()`/`.handle()` (as covered in the Multithreading prep) — again, silently swallowed if the chain is never joined/handled.
- **Best practice:** Always set a `Thread.setDefaultUncaughtExceptionHandler()` for critical background threads, always call `.get()`/`.join()` or attach `.exceptionally()` for async tasks, and centralize logging so async failures are never silently invisible in production monitoring.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can `finally` be skipped?" → Yes, if `System.exit()` is called inside `try`/`catch`, or the JVM crashes, or the thread is forcibly killed.
- "What's the difference between `Exception` and `RuntimeException` in terms of what you should catch?" → You should generally catch specific known types; catching generic `Exception` broadly (as opposed to `RuntimeException` broadly) is even more dangerous because it also catches checked exceptions you may not have anticipated.
- "Is it legal to catch `Error`?" → Syntactically yes, semantically almost never advisable — `Error` subclasses signal conditions (`OutOfMemoryError`, `StackOverflowError`) the application generally can't safely recover from.
- "What happens if you rethrow the caught exception object itself vs creating a new one?" → Rethrowing the same instance (`throw e;`) preserves the **original stack trace** (where it was first thrown); creating a new exception resets the stack trace to the current rethrow location unless you explicitly chain the original as `cause` — an important distinction for debugging accuracy.
- "Can a `catch` block itself throw a different exception?" → Yes — very common (Q17's chaining pattern) — the `catch` block executes normally, and any statement inside it, including `throw`, behaves as it would anywhere else.
- "What is `Objects.requireNonNull()` and why prefer it over a manual null check?" → Throws a clear `NullPointerException` immediately with a descriptive message at the actual point of the violated precondition, rather than letting a `null` propagate and fail confusingly somewhere unrelated later — a "fail fast" best practice (Q19, point 9).

---

*Study tip: The finally-block traps (Q6-Q7), the checked-vs-unchecked design justification (Q2-Q3), and exception chaining (Q17, "never swallow the cause") are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY, not just WHAT, for each.*
