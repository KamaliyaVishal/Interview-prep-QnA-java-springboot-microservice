# Resilience4j — Interview Prep
---

## SECTION 1: CIRCUIT BREAKER AND ITS STATES

### Q1. What problem does a circuit breaker solve that a plain timeout doesn't?
**Answer:** A timeout limits how long you'll wait for **one** call, but if a downstream service is genuinely struggling, every subsequent call still pays the full timeout cost while continuing to hammer an already-failing dependency — wasting caller resources (threads, connections) and making the downstream's recovery *harder*, not easier. A circuit breaker adds **memory**: it tracks recent call outcomes, and once failures cross a threshold, it **stops attempting calls entirely** for a cooldown period, failing fast (near-zero cost) instead of repeatedly timing out.

**Senior detail:** The circuit breaker's real value is bidirectional — it protects the **caller** from wasting resources on doomed calls, and it protects the **struggling downstream** from an unrelenting flood of retries/traffic exactly when it's least able to handle load, giving it breathing room to recover.

---

### Q2. Explain the three core circuit breaker states and the transitions between them.
**Answer:**
| State | Behavior |
|---|---|
| **CLOSED** | Normal operation — all calls go through; failures/successes are recorded in a sliding window |
| **OPEN** | Calls are **rejected immediately** (no network call attempted) — a `CallNotPermittedException` is thrown, or the fallback runs |
| **HALF_OPEN** | A limited number of **trial calls** are permitted through to test if the downstream has recovered |

**Transitions:**
```
CLOSED --(failure rate ≥ threshold)--> OPEN
OPEN --(wait duration elapses)--> HALF_OPEN
HALF_OPEN --(trial calls succeed enough)--> CLOSED
HALF_OPEN --(trial calls still fail)--> OPEN
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderService:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 5
```

**Trap:** Assuming OPEN → CLOSED is a direct transition — it always passes through HALF_OPEN first. Skipping that verification step would mean flipping straight back to full traffic on an unverified assumption that the downstream recovered.

---

### Q3. Count-based vs time-based sliding windows — what's the difference, and which would you pick for a low-traffic service?
**Answer:** `sliding-window-type` controls how the breaker's failure-rate window is measured:
- **COUNT_BASED** (default) — the window holds the last **N calls**, regardless of how long they took to accumulate.
- **TIME_BASED** — the window holds calls from the last **N seconds**, regardless of how many calls that was.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      lowTrafficService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 60   # last 60 seconds
```

**Senior-level answer:**
> "For a low-traffic endpoint — say, an internal admin API called a handful of times an hour — I use TIME_BASED. With COUNT_BASED and a low-traffic service, a single stale failure from hours ago could still be sitting in a window of only 10-20 calls and disproportionately skew the failure rate. TIME_BASED naturally 'forgets' old data as time passes regardless of call volume. For high-traffic services, COUNT_BASED is usually the more intuitive and stable choice — I know exactly how many recent calls are being evaluated."

---

### Q4. What's the difference between `failure-rate-threshold` and `slow-call-rate-threshold`?
**Answer:** Both open the circuit, but for different reasons:
- **`failure-rate-threshold`** — percentage of calls that **errored** (exception thrown).
- **`slow-call-rate-threshold`** — percentage of calls that **exceeded** `slow-call-duration-threshold`, even if they eventually succeeded.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderService:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 2s
```

**Senior detail:** This distinction matters a lot in practice — a downstream service that's technically "succeeding" but taking 10 seconds per call (e.g., due to a DB connection pool bottleneck) never trips a pure `failure-rate-threshold`, but is arguably just as damaging to overall system latency and thread occupancy as one that's outright erroring. `slow-call-rate-threshold` catches exactly this "alive but crawling" failure mode.

---

### Q5. `minimum-number-of-calls` — why is this setting important, and what happens if it's misconfigured?
**Answer:** `minimum-number-of-calls` sets the floor of calls that must occur within the sliding window before the breaker will even **evaluate** the failure rate and potentially open. Without this, a service that's only received 2 calls, both of which failed, would show a 100% failure rate and trip the breaker — a statistically meaningless sample size driving a real production decision.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderService:
        minimum-number-of-calls: 10   # need at least 10 calls before evaluating failure rate
        sliding-window-size: 20
```

**Trap:** Setting `minimum-number-of-calls` higher than `sliding-window-size` — the breaker can never accumulate enough evaluated calls within the window to trip at all, effectively **disabling** the circuit breaker without anyone noticing, since it silently never opens.

---

## SECTION 2: RETRY, EXPONENTIAL BACKOFF & TIMEOUT

### Q6. How do you configure Resilience4j's `@Retry` annotation, and what's the difference between a fixed and exponential backoff?
**Answer:**
- **Fixed backoff** waits the **same** interval between every retry attempt.
- **Exponential backoff** increases the wait interval **each attempt** (typically by a multiplier), spreading retries out further as failures persist — this avoids a "thundering herd" of many clients all retrying at the exact same fixed interval and re-overwhelming an already-struggling downstream at the same moment.

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        max-attempts: 4
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2   # 500ms, 1000ms, 2000ms between attempts
```
```java
@Retry(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(String id) {
    return orderClient.getOrder(id);
}

public Order fallback(String id, Exception ex) {
    return Order.unavailablePlaceholder(id);
}
```

**Senior-level answer:**
> "I default to exponential backoff for any retry against a shared downstream dependency — fixed-interval retry across many concurrent callers tends to synchronize into repeated traffic spikes exactly when the downstream is already degraded, which is the opposite of what you want. I also add a small random jitter on top of exponential backoff where retry volume is high, so retries from different callers don't stay lock-step even after the delay grows."

---

### Q7. Why is retrying dangerous for non-idempotent operations, and how do you retry safely?
**Answer:** If a `POST /api/orders` request actually succeeds on the server, but the response is lost in transit (network blip, timeout on the way back), a naive retry sends **a second, functionally-identical creation request** — and if the operation isn't idempotent, that means a duplicate order, not just a duplicate network call. Retry is only automatically safe for genuinely **idempotent** operations (GET, well-defined PUT, DELETE); non-idempotent operations need either:
1. An **idempotency key** the server can use to de-duplicate (client generates a UUID per logical operation, sends it as a header, server checks/stores it before executing).
2. No automatic retry — surface the ambiguity to the caller/user instead.

```java
@PostMapping("/api/orders")
public Order createOrder(@RequestHeader("Idempotency-Key") String key, @RequestBody OrderRequest req) {
    return orderService.createIfAbsent(key, req);   // server-side de-dup using the key
}
```

**Trap:** This is one of the most reliably-asked resilience questions at senior level — a candidate who says "I just add `@Retry` to every outbound call" without qualifying idempotency is showing a real gap.

---

### Q8. How do `@Retry` and `@CircuitBreaker` interact when stacked on the same method — what order do they apply in?
**Answer:** Order matters, and Resilience4j applies annotations in the order they're declared on the method (outermost to innermost, matching the order you list them) — but the conventional/recommended composition is:

```java
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
@Retry(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(String id) {
    return orderClient.getOrder(id);
}
```
Here, `@Retry` is the **inner** decorator — it retries individual call attempts — and `@CircuitBreaker` is the **outer** decorator, observing the *overall* outcome (after retries are exhausted) to decide whether to open. This way, individual transient blips get retried quietly, while the circuit breaker's failure tracking reflects genuinely persistent failures rather than being skewed by retries it doesn't know about.

**Trap:** Getting the order backwards (circuit breaker inside retry) can cause **retry to keep attempting calls even while the circuit is open**, each attempt hitting `CallNotPermittedException` and being retried pointlessly — defeating the entire purpose of the breaker's fail-fast behavior.

---

### Q9. How do you configure a call timeout with Resilience4j, and how does it interact with the circuit breaker's slow-call detection?
**Answer:** `TimeLimiter` enforces a hard timeout on a call, typically used together with an async call wrapped in `CompletableFuture`:

```yaml
resilience4j:
  timelimiter:
    instances:
      orderService:
        timeout-duration: 3s
        cancel-running-future: true
```
```java
@TimeLimiter(name = "orderService", fallbackMethod = "fallback")
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
public CompletableFuture<Order> getOrder(String id) {
    return CompletableFuture.supplyAsync(() -> orderClient.getOrder(id));
}
```

**Senior detail:** A call that hits the `TimeLimiter`'s timeout is recorded as a **failure** (or a slow call, depending on configuration) by the circuit breaker's sliding window — so `TimeLimiter` and `slow-call-rate-threshold` (Q4) work together: the timeout defines the hard ceiling on how long any single call is allowed to take, while the breaker's slow-call tracking observes the pattern across many calls to decide whether to open.

---

## SECTION 3: BULKHEAD, RATE LIMITER & FALLBACK

### Q10. What is the Bulkhead pattern, and what real production problem does it prevent?
**Answer:** Named after ship bulkheads that contain flooding to one compartment, the Bulkhead pattern **isolates resources** (typically thread pool or concurrent-call capacity) **per dependency**, so a slow/hung call to one downstream service can't exhaust the resources needed to call **other, unrelated** downstream services from the same application.

**Production scenario:** A service calls both `order-service` and `notification-service` using a **shared** thread pool for all outbound calls. `order-service` starts hanging under load; every request thread piles up waiting on it, and soon there are **no threads left** to call `notification-service` either — even though `notification-service` is perfectly healthy. A bulkhead around each dependency prevents this cross-contamination by giving each its own bounded pool of concurrent call capacity.

---

### Q11. Semaphore-based vs thread-pool-based bulkhead in Resilience4j — what's the difference, and when do you use each?
**Answer:**
| Aspect | `SemaphoreBulkhead` | `ThreadPoolBulkhead` |
|---|---|---|
| Mechanism | Limits **concurrent calls** via a semaphore permit count — calls run on the **caller's own thread** | Runs calls on a **separate, dedicated thread pool**, decoupled from the caller's thread |
| Overhead | Low — no extra thread pool/context switch | Higher — thread pool management, context switching |
| Timeout enforcement | Can't forcibly interrupt a call once it's running (no separate thread to cancel) | Can be combined with a genuine timeout that actually reclaims the thread |
| Typical use | Reactive/non-blocking code, or where you just want to cap concurrency | Blocking calls where you want true isolation and the ability to abandon a hung call |

```yaml
resilience4j:
  bulkhead:
    instances:
      orderService:
        max-concurrent-calls: 20
  thread-pool-bulkhead:
    instances:
      notificationService:
        core-thread-pool-size: 10
        max-thread-pool-size: 20
        queue-capacity: 50
```

**Senior-level answer:**
> "For blocking service-to-service calls, I lean toward `ThreadPoolBulkhead` when true isolation matters — it genuinely decouples the caller thread from the downstream call, so a hung call can be abandoned without the calling thread itself being stuck. For reactive/WebFlux code, or when the overhead of a dedicated pool isn't justified, `SemaphoreBulkhead` is lighter-weight and sufficient — it just caps concurrency without adding thread pool machinery."

---

### Q12. How does Resilience4j's `RateLimiter` differ from a Bulkhead, and when would you use one over the other?
**Answer:** Both limit "how much is allowed through," but on different axes:
- **Bulkhead** limits **concurrent, in-flight** calls at any instant (a capacity ceiling).
- **RateLimiter** limits the **rate** of calls **over time** (e.g., N calls per second), regardless of how many are concurrently in-flight.

```yaml
resilience4j:
  ratelimiter:
    instances:
      orderService:
        limit-for-period: 50
        limit-refresh-period: 1s
        timeout-duration: 0
```

**When to use which:** RateLimiter is appropriate when a downstream has a **known hard throughput cap** you must respect (e.g., a third-party API with a contractual rate limit) — it's about *pacing*, not resource protection per se. Bulkhead is about protecting **your own application's** resources (threads/connections) from being monopolized by one dependency. In practice, they're often used together: RateLimiter to respect the downstream's contract, Bulkhead to protect your own service's stability regardless of what the downstream is doing.

---

### Q13. What's the difference between a `fallbackMethod` on `@CircuitBreaker` and one on `@Retry` — do you need separate fallback logic for each?
**Answer:** Practically, no — when patterns are **stacked** (Q8), only the **outermost** decorator's fallback actually fires for a given failure, since the exception propagates outward through the chain until something either recovers it or reaches the outermost fallback. Resilience4j allows specifying the same `fallbackMethod` name on multiple annotations for convenience/clarity, but only one fallback invocation happens per failed call chain.

```java
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
@Retry(name = "orderService", fallbackMethod = "fallback")
@Bulkhead(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(String id) {
    return orderClient.getOrder(id);
}

public Order fallback(String id, Exception ex) {
    log.warn("getOrder fallback triggered for {}: {}", id, ex.getClass().getSimpleName());
    return Order.unavailablePlaceholder(id);
}
```

**Senior detail:** The fallback method signature must match the original method's parameters **plus a `Throwable`/`Exception`** parameter at the end — this is a common compile-time/runtime mismatch bug when a method's fallback signature drifts out of sync after refactoring the original method's parameters.

---

## SECTION 4: COMBINING RESILIENCE PATTERNS

### Q14. What is the recommended order for stacking multiple Resilience4j annotations on one method, and why does order matter?
**Answer:** The generally recommended composition, outermost to innermost:
```
Bulkhead → RateLimiter → CircuitBreaker → TimeLimiter/Retry
```
```java
@Bulkhead(name = "orderService")
@RateLimiter(name = "orderService")
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
@Retry(name = "orderService", fallbackMethod = "fallback")
public Order getOrder(String id) {
    return orderClient.getOrder(id);
}
```

**Why this order:**
- **Bulkhead outermost** — reject excess concurrent calls before they consume any other resource at all.
- **RateLimiter next** — reject calls exceeding the allowed rate before they reach the circuit breaker's accounting.
- **CircuitBreaker** — decides whether the call should even be attempted based on recent history.
- **Retry innermost** — retries individual attempts, with the circuit breaker observing the *overall* outcome (Q8), not each individual retry attempt.

**Trap:** Placing `Retry` outside `CircuitBreaker` causes retries to keep attempting calls the breaker has already rejected (`CallNotPermittedException`), wasting retry budget on calls that were never going to be attempted in the first place.

---

### Q15. How do you unit test a method decorated with Resilience4j annotations without waiting for real timeouts/backoff delays?
**Answer:** Two common approaches:
1. **Programmatic API** — instead of relying purely on annotations, build the decorators programmatically in tests with short, test-friendly durations (`CircuitBreakerConfig.custom().waitDurationInOpenState(Duration.ofMillis(50))...`), and manually invoke `CircuitBreaker.decorateSupplier(...)`.
2. **Integration test with a mocked downstream** — use WireMock (or an equivalent) to simulate downstream failures/delays, and run with the *actual* configured YAML values but a much shorter `wait-duration`/backoff overridden specifically for the test profile (`application-test.yml`).

```java
CircuitBreaker cb = CircuitBreaker.of("test", CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofMillis(50))
    .slidingWindowSize(4)
    .build());

Supplier<String> decorated = CircuitBreaker.decorateSupplier(cb, () -> { throw new RuntimeException("fail"); });
```

**Senior-level answer:**
> "I test the decision logic (does the breaker open when it should, does retry stop at max attempts) using the programmatic API with compressed durations — waiting real seconds/minutes in a test suite isn't practical. For the actual integration between my service and a real downstream, I use WireMock to simulate specific failure scenarios (500s, timeouts, slow responses) and verify the observed behavior end-to-end, including that fallback methods actually return what I expect."

---

### Q16. How do you observe/monitor circuit breaker state transitions and metrics in production?
**Answer:** Resilience4j exposes metrics via **Micrometer**, which integrates with Actuator and any metrics backend (Prometheus, Datadog). Key metrics per instance: current state, failure rate, slow-call rate, number of calls in each outcome bucket.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, circuitbreakers, metrics, prometheus
  health:
    circuitbreakers:
      enabled: true
```
Actuator's `/actuator/circuitbreakers` and `/actuator/circuitbreakerevents` endpoints expose live state and a recent event log (state transitions, individual call outcomes) directly, useful for debugging without needing a full metrics pipeline.

**Senior detail:** State transitions specifically (`CLOSED → OPEN`, `OPEN → HALF_OPEN`) are exactly the kind of event worth **alerting on**, not just graphing — a circuit opening for a critical downstream dependency is an incident-worthy signal that should page someone, not just sit in a dashboard nobody's watching.

---

## SECTION 5: RESILIENCE4J vs HYSTRIX (COMPARISON ONLY — HYSTRIX IS LEGACY)

### Q17. Why did the ecosystem move from Netflix Hystrix to Resilience4j?
**Answer:** Netflix put **Hystrix into maintenance mode** in 2018 as part of winding down several Netflix OSS projects, and officially recommended Resilience4j (along with other alternatives) as the path forward. Resilience4j was designed as a lightweight, modular successor with a few concrete advantages:

| Aspect | Hystrix | Resilience4j |
|---|---|---|
| Status | Maintenance mode / effectively legacy | Actively maintained |
| Dependencies | Pulls in RxJava as a hard dependency | No mandatory reactive library dependency — works with plain Java functional interfaces, or integrates optionally with Reactor/RxJava |
| Design | Monolithic — one library doing circuit breaking, thread isolation, and metrics together | **Modular** — separate, composable modules (`CircuitBreaker`, `Retry`, `Bulkhead`, `RateLimiter`, `TimeLimiter`), pull in only what you need |
| Concurrency model | Thread-pool isolation was central and somewhat heavyweight | Offers both lightweight semaphore-based and thread-pool-based bulkheads (Q11), letting you choose the right cost/isolation trade-off |
| Spring Cloud integration | `spring-cloud-netflix-hystrix` — deprecated | `spring-cloud-circuitbreaker-resilience4j` — current standard, also abstracts over other implementations |

**Interview trap:** Describing Hystrix as the current standard, or not being able to name *why* the ecosystem moved away from it, is a clear outdated-knowledge signal in a senior interview — this is one of the most commonly asked "do you know current Spring Cloud state" questions.

---

### Q18. Functionally, is there anything Hystrix did that Resilience4j doesn't directly replicate?
**Answer:** The most notable difference is Hystrix's **built-in request collapsing/caching within a single request context** and its tightly-integrated **dashboard** (Hystrix Dashboard + Turbine for aggregated stream views) — Resilience4j doesn't ship an equivalent built-in dashboard; instead it leans on the broader observability ecosystem via Micrometer (Prometheus/Grafana), which is arguably a stronger long-term choice since it doesn't lock you into a Netflix-specific visualization tool, but does mean there's no batteries-included dashboard out of the box.

**Senior-level answer:**
> "I don't consider the missing built-in dashboard a real loss — Micrometer plus Prometheus/Grafana is a far more standard, flexible observability stack than Hystrix Dashboard ever was, and it means circuit breaker metrics live alongside all my other application metrics rather than in a separate, Netflix-specific tool. The bigger practical win with Resilience4j is the modularity — I'm not forced to pull in a full reactive stack (RxJava) just to get circuit breaking on a simple blocking service."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What state does a circuit breaker start in?" → CLOSED.
- "Does a circuit breaker prevent a call from being attempted, or just fail it after attempting?" → In the OPEN state, it prevents the call from being attempted at all — fails immediately with `CallNotPermittedException`.
- "What's evaluated during HALF_OPEN — all traffic, or a limited sample?" → A limited number of trial calls (`permitted-number-of-calls-in-half-open-state`).
- "Does Resilience4j retry require RxJava?" → No — unlike Hystrix, Resilience4j has no mandatory reactive library dependency.
- "What does `minimum-number-of-calls` protect against?" → A statistically meaningless failure rate from too small a sample tripping the breaker prematurely.
- "Should you retry a timed-out POST automatically?" → Only if it's idempotent or protected by an idempotency key — otherwise no.
- "Which pattern limits concurrent calls, and which limits calls per unit time?" → Bulkhead = concurrency; RateLimiter = throughput over time.
- "Is Hystrix still recommended for new projects?" → No — it's in maintenance mode; Resilience4j is the current standard.

---

*Study tip: The failure-rate vs slow-call-rate distinction (Q4), correct decorator stacking order and why (Q8, Q14), and the idempotency risk in blind retry (Q7) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY the ordering and thresholds matter, not just naming the annotations.*
