# Spring AOP — Interview Prep
---

## SECTION 1: ASPECT, ADVICE, POINTCUT, JOINPOINT

### Q1. What problem does AOP solve, and what does each core term (Aspect, Advice, Pointcut, JoinPoint) actually mean?
**Answer:** AOP (Aspect-Oriented Programming) solves the problem of **cross-cutting concerns** — logic like logging, transaction management, security checks, or caching that would otherwise have to be duplicated across many unrelated classes/methods, tangling business logic with infrastructure concerns. AOP lets you define that logic **once**, in one place, and have it applied wherever needed, without touching the target code.

| Term | Meaning |
|---|---|
| **Aspect** | The module encapsulating a cross-cutting concern — a class annotated `@Aspect`, combining advice + pointcuts |
| **Advice** | The actual **action** taken at a matched point (the code that runs) — "what to do" |
| **Pointcut** | An **expression** that selects *which* join points the advice applies to — "where to do it" |
| **JoinPoint** | A specific point during execution where advice *could* run — in Spring AOP, always a **method execution** |

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")   // pointcut
    public void logMethodCall(JoinPoint joinPoint) {        // advice + joinpoint access
        System.out.println("Calling: " + joinPoint.getSignature().getName());
    }
}
```

**Senior-level answer:**
> "I explain this to junior engineers with a simple analogy: the **pointcut** is the *filter* (which methods), the **advice** is the *action* (what code runs), and the **aspect** is the *class* bundling related filters and actions together — e.g., all logging concerns in one `LoggingAspect`, all transaction concerns in another. The one thing worth calling out proactively: unlike full AspectJ, **Spring AOP only supports method-execution join points** — it can't intercept field access, constructor calls, or static initializers. If a requirement genuinely needs that, it requires full AspectJ compile-time/load-time weaving, not Spring's proxy-based AOP."

---

### Q2. How does Spring AOP actually implement aspects under the hood — proxies, not bytecode weaving?
**Answer:** Spring AOP is **proxy-based**, not compile-time bytecode weaving (unlike full AspectJ). At startup, Spring wraps any bean matched by a pointcut in a **dynamic proxy** that intercepts calls and runs the configured advice around the real method invocation.

- **JDK dynamic proxy** — used automatically when the target bean implements at least one interface; the proxy implements the same interface(s).
- **CGLIB proxy** — used when the target has no interface (or `proxyTargetClass=true` is forced); generates a runtime subclass of the target class.

```java
@Service
public class OrderService {           // no interface → Spring uses a CGLIB subclass proxy
    public void placeOrder() { ... }
}
```

**Senior-level answer:**
> "The proxy-based model is exactly why **self-invocation doesn't trigger advice** — this is the single most common Spring AOP bug I've debugged. If `OrderService.placeOrder()` internally calls `this.validate()`, that's a direct JVM method call on `this`, bypassing the proxy entirely, so any advice on `validate()` (like `@Transactional` or a custom security check) silently never fires. The fix is either injecting a self-reference bean (`@Autowired private OrderService self;` and calling `self.validate()`), splitting the method into a separate bean, or in some cases using `AopContext.currentProxy()` with `exposeProxy=true` — though I generally treat this as a sign the class needs restructuring rather than reaching for `AopContext` as a first resort."

---

## SECTION 2: ADVICE TYPES (@BEFORE, @AFTER, @AROUND, @AFTERRETURNING, @AFTERTHROWING)

### Q3. What are the five advice types, and when do you use each?
**Answer:**

| Advice | Runs | Typical use |
|---|---|---|
| `@Before` | Before the method executes | Validation, logging entry, security pre-checks |
| `@After` (finally) | Always, after the method — success or exception | Cleanup, resource release (like a `finally` block) |
| `@AfterReturning` | Only after **successful** completion | Logging results, auditing successful operations |
| `@AfterThrowing` | Only when the method **throws** | Error logging, alerting, exception auditing |
| `@Around` | Wraps the **entire** invocation — before and after, with full control | Timing/metrics, caching, transaction management, retry logic |

```java
@Aspect
@Component
public class OrderAspect {

    @Before("execution(* com.example.service.OrderService.placeOrder(..))")
    public void beforePlaceOrder(JoinPoint jp) {
        System.out.println("About to place order, args=" + Arrays.toString(jp.getArgs()));
    }

    @AfterReturning(pointcut = "execution(* com.example.service.OrderService.placeOrder(..))",
                     returning = "result")
    public void afterReturning(Object result) {
        System.out.println("Order placed successfully: " + result);
    }

    @AfterThrowing(pointcut = "execution(* com.example.service.OrderService.placeOrder(..))",
                    throwing = "ex")
    public void afterThrowing(Exception ex) {
        System.out.println("Order placement failed: " + ex.getMessage());
    }

    @After("execution(* com.example.service.OrderService.placeOrder(..))")
    public void afterAlways() {
        System.out.println("placeOrder() finished (success or failure)");
    }
}
```

**Senior-level answer:**
> "I pick the narrowest advice type that expresses the intent — `@AfterThrowing` for error alerting rather than `@Around` with a try/catch, `@AfterReturning` for auditing success rather than `@After`, because it's more self-documenting and there's less room to accidentally swallow or alter the method's behavior. I reach for `@Around` specifically when I need to **influence** the outcome — measure elapsed time around the call, retry on failure, short-circuit and return a cached value without invoking the target at all, or modify the return value — none of which the other four advice types can do, since they can only observe, not control, the invocation."

---

### Q4. Deep dive: How does @Around work, and what happens if you forget to call proceed()?
**Answer:** `@Around` advice receives a `ProceedingJoinPoint`, and is responsible for explicitly calling `.proceed()` to actually invoke the target method — this is what gives `@Around` its power (you control *if*, *when*, and *how many times* the real method runs) and its risk.

```java
@Aspect
@Component
public class TimingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();              // invokes the actual target method
            return result;                                // must return it — or the caller gets null
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{} took {}ms", pjp.getSignature().getName(), elapsed);
        }
    }
}
```

**Trap: forgetting `proceed()`** — the target method **never executes at all**, and the advice's return value (or `null`, if none is returned) is silently substituted as the result. This is a subtle bug: no exception is thrown, the call just silently does nothing, which can be a genuinely confusing production incident to trace back to an aspect.

**Senior-level answer:**
> "The `proceed()` call is also where retry logic lives naturally — I've written `@Around` advice that calls `pjp.proceed()` in a loop with backoff on a specific transient exception type, which isn't expressible with any of the other advice annotations. The discipline I hold myself to: always wrap `proceed()` in a `try/finally` (not just `try/catch`) when the advice does timing or resource cleanup, so the cleanup runs regardless of whether the target method threw — and always explicitly return `pjp.proceed()`'s result (or a deliberately substituted value) rather than letting a code path fall through and return `null` by accident."

---

## SECTION 3: POINTCUT EXPRESSIONS

### Q5. How does the execution() pointcut designator work, and what does each part of the expression mean?
**Answer:** `execution()` is the most commonly used pointcut designator — it matches method executions based on a structured pattern:

```
execution(modifiers-pattern? return-type-pattern declaring-type-pattern? method-name-pattern(param-pattern) throws-pattern?)
```

```java
execution(public * com.example.service.*.*(..))
```
| Segment | Meaning |
|---|---|
| `public` | Access modifier (optional — omit to match any) |
| `*` | Return type — `*` matches any return type |
| `com.example.service.*` | Declaring type — any class directly in this package |
| `*` | Method name — any method name |
| `(..)` | Parameters — `..` matches any number/type of arguments |

**More examples:**
```java
execution(* com.example.service.OrderService.placeOrder(..))       // one specific method, any args
execution(* com.example.service.*.*(..))                            // any method, any class, in this package
execution(* com.example.service..*.*(..))                           // .. after package = includes sub-packages
execution(* com.example.service.OrderService.find*(Long))           // methods starting with "find", taking one Long
execution(* *..*Repository.save(..))                                // save() on any class ending in "Repository"
```

**Senior-level answer:**
> "I favor package-scoped wildcards (`com.example.service..*.*(..)`) over overly broad `execution(* *.*(..))` expressions — an unbounded pointcut is both a performance concern (proxying more beans than necessary) and a correctness risk, since it can silently start matching classes you never intended to intercept as the codebase grows, like framework-internal beans."

---

### Q6. What's the difference between using an annotation-based pointcut (@annotation) vs execution(), and why prefer one for something like a custom @LogExecutionTime aspect?
**Answer:** `@annotation()` matches methods (or types, via `@within`/`@target`) carrying a specific annotation, regardless of package, class name, or signature — a fundamentally different, more **intent-driven** selection strategy than `execution()`'s structural pattern matching.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface LogExecutionTime { }

@Aspect
@Component
public class TimingAspect {
    @Around("@annotation(com.example.annotation.LogExecutionTime)")
    public Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        log.info("{} took {}ms", pjp.getSignature().getName(), System.currentTimeMillis() - start);
        return result;
    }
}

@Service
public class OrderService {
    @LogExecutionTime
    public Order placeOrder(OrderRequest request) { ... }   // opted-in explicitly
}
```

**Senior-level answer:**
> "This is my default choice whenever the cross-cutting concern is **opt-in per method** rather than blanket-applied to a whole layer — `execution()` patterns are fragile to refactoring (rename a package or move a class and the pointcut silently stops matching), while a custom annotation is self-documenting at the call site and survives structural refactors. I use `execution()`-based pointcuts for broad, structural concerns like 'every method in the service layer gets a transaction boundary,' and `@annotation()`-based pointcuts for selective, explicit concerns like custom auditing or rate-limiting that shouldn't apply universally."

---

### Q7. How do you combine multiple pointcuts, and how do you avoid repeating the same expression across many advice methods?
**Answer:** Pointcuts support Boolean composition (`&&`, `||`, `!`), and can be **extracted into a named, reusable `@Pointcut` method** referenced by signature from multiple advice annotations — avoiding copy-pasted expressions.

```java
@Aspect
@Component
public class ServiceLoggingAspect {

    @Pointcut("execution(* com.example.service..*(..))")
    public void serviceLayer() { }

    @Pointcut("@annotation(com.example.annotation.Auditable)")
    public void auditableMethods() { }

    @Before("serviceLayer() && auditableMethods()")   // combined, reused pointcuts
    public void logAuditableServiceCall(JoinPoint jp) {
        log.info("Auditable call: {}", jp.getSignature());
    }

    @Around("serviceLayer()")
    public Object timeServiceCall(ProceedingJoinPoint pjp) throws Throwable {
        // reuses the same serviceLayer() pointcut without repeating the expression
        ...
    }
}
```

**Senior-level answer:**
> "Once a team has more than two or three aspects, I always centralize shared pointcuts into a dedicated `CommonPointcuts` class with well-named `@Pointcut` methods (`serviceLayer()`, `repositoryLayer()`, `restControllers()`) — this turns pointcut expressions into a small internal vocabulary that's referenced by name everywhere else, so a structural change (like renaming a package) is a one-line fix instead of a grep-and-replace across every aspect in the codebase."

---

## SECTION 4: USE CASES (LOGGING, TRANSACTION, SECURITY)

### Q8. Show a practical logging aspect — how do you capture method name, arguments, execution time, and outcome without cluttering business code?
**Answer:**
```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    @Around("execution(* com.example.service..*(..))")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        String method = pjp.getSignature().toShortString();
        log.info("Entering {} with args={}", method, Arrays.toString(pjp.getArgs()));

        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();
            log.info("Exiting {} in {}ms, result={}", method, System.currentTimeMillis() - start, result);
            return result;
        } catch (Exception ex) {
            log.error("{} threw {} after {}ms", method, ex.getClass().getSimpleName(),
                    System.currentTimeMillis() - start);
            throw ex;   // never swallow — rethrow so normal exception handling still applies
        }
    }
}
```

**Senior-level answer:**
> "The one discipline I enforce here: logging aspects must **always rethrow** any caught exception. It's tempting to catch-and-log-and-swallow in a generic `@Around` aspect, but that silently changes the target method's contract — callers relying on that exception for control flow (like a `@Transactional` rollback trigger) would break in a way that's very hard to trace back to an unrelated logging aspect. I also avoid logging full argument objects at `info` level in production if they might contain PII — I'll mask or omit sensitive fields, or drop to `debug` level for full payloads."

---

### Q9. How does @Transactional actually relate to Spring AOP? Walk through what happens under the hood.
**Answer:** `@Transactional` is itself implemented as a Spring AOP concern — `TransactionInterceptor`, an `@Around`-style advice, wraps the annotated method in a proxy that begins a transaction before invocation, commits on normal return, and rolls back on a matching exception.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        inventoryService.reserve(request.getItems());   // same transaction
        paymentService.charge(request.getPayment());    // same transaction
        orderRepository.save(new Order(request));        // same transaction
    }                                                     // commits here if no exception
}
```

**Senior-level answer:**
> "Because `@Transactional` is AOP-proxy-based under the hood, it inherits the exact same self-invocation limitation covered in Q2 — calling a `@Transactional` method from another method **in the same class** bypasses the proxy, so no transaction boundary is created, which is a very common 'why isn't my rollback happening' bug. The other detail I always verify explicitly rather than assume: by default, Spring only rolls back on **unchecked exceptions** (`RuntimeException` and `Error`); a checked exception thrown from a `@Transactional` method does **not** trigger rollback unless you explicitly declare `@Transactional(rollbackFor = Exception.class)`. I've seen production bugs where a checked `IOException` from an external call was expected to roll back a transaction and silently didn't."

---

### Q10. How would you use AOP to enforce a cross-cutting security check — e.g., custom rate limiting or auditing — instead of Spring Security's built-in @PreAuthorize?
**Answer:** For concerns Spring Security doesn't cover out of the box (custom rate limiting, business-specific audit trails, feature-flag gating), a custom annotation + `@Around` aspect gives the same declarative, opt-in ergonomics as `@PreAuthorize` without forcing the logic into Spring Security's authorization model.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimited {
    int maxCallsPerMinute() default 60;
}

@Aspect
@Component
public class RateLimitAspect {

    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Around("@annotation(rateLimited)")
    public Object enforceLimit(ProceedingJoinPoint pjp, RateLimited rateLimited) throws Throwable {
        String key = pjp.getSignature().toShortString();
        RateLimiter limiter = limiters.computeIfAbsent(key,
                k -> RateLimiter.create(rateLimited.maxCallsPerMinute() / 60.0));

        if (!limiter.tryAcquire()) {
            throw new RateLimitExceededException("Rate limit exceeded for " + key);
        }
        return pjp.proceed();
    }
}

@RestController
public class PaymentController {
    @RateLimited(maxCallsPerMinute = 10)
    @PostMapping("/api/payments")
    public PaymentResult charge(@RequestBody PaymentRequest request) { ... }
}
```

**Senior-level answer:**
> "I draw a clear line between what belongs in Spring Security (`@PreAuthorize` — authentication-and-authorization decisions tied to the current principal) versus a custom AOP aspect (business-specific cross-cutting rules like rate limiting, feature flags, or domain-specific audit logging that don't fit Spring Security's model). Both are 'security-adjacent' but reusing `@PreAuthorize`'s SpEL machinery for non-authorization concerns tends to produce confusing, overloaded expressions — a dedicated annotation with a purpose-built aspect is more explicit and easier for the next engineer to reason about."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does Spring AOP work on private methods?" → No — proxy-based AOP can only intercept calls that go through the proxy, and private methods can't be overridden/proxied at all (CGLIB) nor are they exposed via an interface (JDK proxy).
- "What's the difference between Spring AOP and full AspectJ?" → Spring AOP is runtime, proxy-based, method-execution-only, and simpler to set up; AspectJ is compile-time/load-time bytecode weaving, supports far more join point types (field access, constructors, static initializers), and has a steeper setup cost (LTW agent or AspectJ compiler).
- "Can you have multiple @Around advices on the same method — what determines execution order?" → Yes — controlled via `@Order` on the aspect class (lower value = higher precedence, runs first on the way in, last on the way out — like nested try blocks).
- "What happens if a pointcut matches zero methods?" → No error — the aspect is simply never invoked; this is a common silent-bug source worth testing for explicitly (e.g., asserting the advice actually fired in a test).
- "Is @Aspect enough by itself, or do you need something else to activate it?" → `@Aspect` alone just marks the class; you also need `@EnableAspectJAutoProxy` (auto-enabled by Spring Boot autoconfiguration when `spring-boot-starter-aop` is on the classpath) plus the aspect being a registered Spring bean (`@Component`).
- "JDK dynamic proxy vs CGLIB — which does Spring Boot use by default?" → Spring Boot defaults `proxyTargetClass=true` since Spring Boot 2, meaning CGLIB (class-based) proxies are used by default even for beans that implement interfaces — for consistency across the app.

---

*Study tip: The self-invocation/proxy-bypass trap (Q2, and how it silently breaks `@Transactional` in Q9), the discipline of always rethrowing in `@Around` logging advice (Q8), and being able to explain the `execution()` expression syntax component-by-component (Q5) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice tracing *why* the proxy model causes each trap, not just memorizing that it does.*
