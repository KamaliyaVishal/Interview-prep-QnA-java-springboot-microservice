# Resilience & Fault Tolerance — Interview Prep
---

## SECTION 1: CIRCUIT BREAKER, RETRY & EXPONENTIAL BACKOFF

### Q1. What problem does a Circuit Breaker solve that a simple try/catch around a remote call doesn't?
**Answer:** A try/catch handles a **single failed call**, but it does nothing to protect the caller when a downstream dependency is **consistently failing or slow** — every subsequent request still pays the full timeout cost waiting to fail again, and threads pile up waiting on a dependency that isn't coming back soon. A **Circuit Breaker** tracks the failure rate over a rolling window and, once it crosses a threshold, **stops calling the downstream service entirely for a cooldown period**, failing fast instead.

```
CLOSED (normal, calls pass through, failures counted)
   --failure rate > threshold-->  OPEN (calls fail immediately, no network call made)
   --after wait duration-->        HALF_OPEN (a limited number of trial calls allowed through)
        --trial calls succeed--> CLOSED
        --trial calls fail-->    OPEN again
```

**Senior-level answer:**
> "The real value of a circuit breaker isn't preventing the first failure — it's preventing the **pile-up**: threads/connections held open waiting on a dead dependency, which is exactly how one slow downstream service cascades into exhausting the caller's own thread pool and taking down a service that was otherwise healthy. Tripping the breaker converts 'many slow failures' into 'fast, cheap failures,' which protects the caller's own resources."

**Trap:** Candidates often describe only the CLOSED→OPEN transition and forget **HALF_OPEN** — without it, a breaker that opens would either need a fixed timer to blindly close again (risking a full traffic flood onto a still-broken service) or require a manual reset. HALF_OPEN is what lets the system **safely self-test recovery** with limited exposure. (Implementation specifics — Resilience4j state config, sliding window types — are covered in the Resilience4j module.)

---

### Q2. How is Retry different from a Circuit Breaker, and why can combining them naively make things worse?
**Answer:** **Retry** re-attempts a **single failed call** on the assumption the failure was transient (network blip, momentary overload). A Circuit Breaker instead **stops calling** a dependency that's failing persistently. They solve different failure durations — retry for transient, circuit breaker for sustained.

**Why naive combination backfires:** Retrying immediately and repeatedly against an already-struggling downstream service **adds more load to a service that's failing partly *because* of load** — this is the classic **retry storm**, where retries from many callers synchronize and amplify an outage instead of recovering from it.

**Fix: exponential backoff with jitter.**
```java
Retry retry = Retry.of("paymentService", RetryConfig.custom()
    .maxAttempts(4)
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(200), 2.0, 0.5))  // base 200ms, x2 each attempt, +/-50% jitter
    .retryOnException(e -> e instanceof TimeoutException)  // only retry transient failures
    .build();
```
- **Exponential**: each retry waits longer than the last (200ms → 400ms → 800ms...), giving the dependency room to recover instead of hammering it at a fixed rate.
- **Jitter**: randomizing the wait prevents thousands of callers from retrying in **synchronized bursts**, which is what actually causes the thundering-herd effect even with pure exponential backoff.

**Senior-level answer:**
> "The ordering matters: retry belongs *inside* the circuit breaker's CLOSED state for individual transient blips, but the moment the breaker trips OPEN, retries should stop immediately — you don't want to keep retrying against a dependency you've already determined is down. I also always cap retries to **idempotent operations only** (Q11); retrying a non-idempotent write on a timeout — where you don't actually know if the first attempt succeeded — is a correctness bug waiting to happen, not just a performance one."

---

## SECTION 2: TIMEOUT, BULKHEAD & RATE LIMITING

### Q3. Why is an explicit timeout non-negotiable on every remote call, even ones that "almost never" hang?
**Answer:** Without an explicit timeout, a caller is at the mercy of the downstream service's (or the network's) worst-case behavior — a hung connection can block a thread **indefinitely**. In a thread-per-request model (like a standard Tomcat servlet container), enough hung calls exhaust the entire thread pool, and the service becomes unresponsive to **all** requests, not just the ones hitting the slow dependency.

```java
WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create().responseTimeout(Duration.ofSeconds(2))))
    .build();
```

**Senior-level answer:**
> "I treat 'no timeout configured' as equivalent to 'timeout = infinity,' which is never actually the intended behavior — it's just an oversight that happens to work fine until the one day the dependency hangs instead of erroring cleanly. Every remote call — HTTP, DB, message broker — gets an explicit, deliberately-chosen timeout; the actual value should be based on realistic p99 latency for that dependency plus margin, not a copy-pasted default across every service."

**Trap:** Setting the timeout too aggressively (e.g., copying a 500ms timeout meant for a fast internal call onto a genuinely slower downstream like a third-party payment gateway) causes **false-positive failures** under normal load spikes, which then trips the circuit breaker unnecessarily. Timeout tuning should be dependency-specific, informed by actual observed latency distributions.

---

### Q4. What is the Bulkhead pattern, and what specific failure mode does it prevent?
**Answer:** Named after ship compartments that stop one flooded section from sinking the whole vessel — **Bulkhead isolates the resources (usually thread pools/connection pools) used to call different dependencies**, so that one dependency behaving badly can't starve the resources needed to call a completely unrelated, healthy dependency.

**Without bulkhead:** all outbound calls share one thread pool → a slow `RecommendationService` call consumes all available threads → the `CheckoutService` call (unrelated, healthy) can't get a thread either → the *entire application* degrades because of *one* slow dependency.

```java
Bulkhead bulkhead = Bulkhead.of("recommendationService", BulkheadConfig.custom()
    .maxConcurrentCalls(10)      // this dependency can never hold more than 10 concurrent calls
    .maxWaitDuration(Duration.ofMillis(500))
    .build());
```

| Bulkhead type | Isolation mechanism |
|---|---|
| **Semaphore bulkhead** | Limits concurrent calls via a counting semaphore — lightweight, no extra threads |
| **Thread-pool bulkhead** | Each protected call runs in its own dedicated, bounded thread pool — stronger isolation (a stuck call can't block the caller's own request thread) but more resource overhead |

**Senior-level answer:**
> "Bulkhead is the pattern that directly answers 'why did an outage in a non-critical dependency take down checkout?' — a question I've had to answer for real in a postmortem. Circuit breakers protect you from *repeated* failure over time; bulkheads protect you from **resource contention in the moment**, even during a single, ongoing incident. I treat them as complementary, not either/or — bulkhead limits blast radius, circuit breaker limits duration."

---

### Q5. How does Rate Limiting differ from Bulkhead, given they both "restrict how much gets through"?
**Answer:** They restrict different things for different reasons:

| | Bulkhead | Rate Limiter |
|---|---|---|
| **Restricts** | Concurrent in-flight calls to a dependency | Number of calls allowed per time window |
| **Protects** | The **caller's own resources** from being exhausted by one dependency | The **downstream/self service** from being overwhelmed by call *volume*, regardless of concurrency |
| **Typical use** | Isolating blast radius between multiple downstream dependencies | Enforcing an API contract/SLA, protecting a service from traffic spikes, respecting a third-party's rate limits |

```java
RateLimiter rateLimiter = RateLimiter.of("externalApi", RateLimiterConfig.custom()
    .limitForPeriod(100)                  // 100 calls
    .limitRefreshPeriod(Duration.ofSeconds(1))  // per second
    .timeoutDuration(Duration.ofMillis(100))    // max wait for a permit before failing
    .build());
```

**Senior-level answer:**
> "A concrete case where the distinction matters: calling a third-party API with a contractual '100 requests/second' limit. Bulkhead alone won't stop you from breaching that — you could have only 5 concurrent calls but still fire 500 requests/second if they're all fast. Rate limiting is specifically about **throughput over time**, independent of concurrency, which is why I apply it at the API Gateway level for inbound protection and at the client level for respecting outbound third-party limits."

---

## SECTION 3: FAIL FAST & FALLBACK

### Q6. What does "fail fast" actually mean in a resilience context, and why is it considered a *good* thing?
**Answer:** Fail fast means detecting and **rejecting a request immediately** when you already know it can't succeed (an open circuit, an exhausted bulkhead, a rate limit hit) — **rather than letting it queue, wait, and eventually time out**. Counter-intuitively, failing faster **improves overall system resilience**, because it frees up resources (threads, connections) instantly instead of holding them hostage on a request destined to fail anyway.

```java
CircuitBreaker breaker = CircuitBreaker.ofDefaults("inventoryService");
Supplier<Inventory> decorated = CircuitBreaker.decorateSupplier(breaker, inventoryClient::getStock);
// When OPEN: decorated.get() throws CallNotPermittedException immediately —
// no network call attempted, no thread held waiting on a timeout.
```

**Senior-level answer:**
> "New engineers often think of 'fail fast' as pessimistic or defeatist, but it's actually the opposite — it's what keeps the rest of the system healthy while one dependency recovers. The alternative — letting every request wait out the full timeout before failing — is strictly worse: same end result (failure), but you've also burned a thread/connection for the full timeout duration, times however many requests came in during that window. Fail fast turns a slow failure into a cheap one."

---

### Q7. What makes a good Fallback, and what's a common mistake teams make when implementing one?
**Answer:** A fallback is the **alternative response returned when the primary call fails** (via circuit breaker, timeout, or rate limiter) — instead of propagating the failure to the end user. Good fallbacks return something **genuinely useful and clearly degraded**, not a silent lie.

```java
@CircuitBreaker(name = "recommendationService", fallbackMethod = "fallbackRecommendations")
public List<Product> getRecommendations(String userId) {
    return recommendationClient.getPersonalized(userId);
}

private List<Product> fallbackRecommendations(String userId, Throwable t) {
    return productRepository.findTopSellingProducts(); // generic but useful, not personalized
}
```

**Good fallback candidates:** cached/stale data with an explicit "last updated" indicator, a generic non-personalized default, a queued-for-later response ("we'll email your receipt"), a degraded feature (search without filters).
**Bad fallback:** silently returning an empty list/zero/null with no signal that anything went wrong — this hides a real failure as if it were valid data, which is worse than failing visibly because it corrupts downstream logic or metrics silently.

**Senior-level answer:**
> "The mistake I see most is fallback logic that's just as likely to fail as the primary call — e.g., falling back to *another* remote service call. A fallback should be **strictly simpler and more reliable** than the path it's replacing: return from a local cache, an in-memory default, or a static response — never another network hop with its own failure modes. I also always make sure a fallback is **observable** — increment a metric/log every time it's used — a silent fallback path is invisible technical debt until the day the fallback itself breaks and nobody notices."

**Trap:** Fallback for a **write** operation (e.g., "payment failed, so fallback to logging it locally and pretending it succeeded") is fundamentally different and much riskier than fallback for a **read** — you generally can't fabricate a successful outcome for a write. The correct pattern there is usually queue-and-retry (outbox) or a clear failure to the user, not a fake success.

---

## SECTION 4: GRACEFUL DEGRADATION

### Q8. What is Graceful Degradation, and how is it different from just having fallbacks on individual calls?
**Answer:** **Graceful Degradation** is a **system/product-level strategy**: deliberately designing which features can be shed or simplified under partial failure or overload so the **core functionality keeps working**, even if the full experience doesn't. Fallbacks (Q7) are the *mechanism*; graceful degradation is the *design decision* of what's essential vs. optional.

**Example — e-commerce checkout under a recommendation-service outage:**
| Feature | Under normal load | Degraded mode |
|---|---|---|
| Add to cart / checkout / payment | Full functionality | **Unaffected — this is core, must always work** |
| "You might also like" recommendations | Personalized ML results | Hidden entirely, or replaced with generic bestsellers |
| Order history search | Full-text search | Falls back to simple date-sorted list if search service is down |

**Senior-level answer:**
> "This is fundamentally a product and architecture conversation, not just an engineering one — you have to explicitly rank features by criticality *before* an incident, not during one. I push teams to define this upfront: 'if service X is down, which of our features are allowed to break, and which absolutely cannot?' Checkout and payment are almost always non-negotiable; recommendations, reviews, and 'related items' are almost always gracefully degradable. Having that list defined ahead of time turns a 3am incident into 'disable feature flag X' instead of a scramble."

**Trap:** Don't conflate graceful degradation with high availability generally — it's specifically about **maintaining core value with reduced functionality**, not about preventing the failure in the first place. A system can have zero redundancy for recommendations (so it *will* go down) but still degrade gracefully by simply hiding that feature when it does.

---

## SECTION 5: BACKPRESSURE

### Q9. What is Backpressure, and why do timeouts/circuit breakers alone not solve it?
**Answer:** Backpressure is a mechanism for a **downstream consumer to signal to an upstream producer that it's being overwhelmed and needs the producer to slow down** — rather than the consumer either dropping work silently or crashing from unbounded queue/buffer growth. Timeouts and circuit breakers protect a **caller** from a **slow downstream**; backpressure protects a **downstream consumer** from a **fast/bursty upstream** — it's the same problem from the opposite direction.

**Without backpressure:** a producer keeps pushing messages faster than the consumer can process them → messages queue up in an unbounded buffer → memory grows unbounded → the consumer eventually OOMs, which is a much worse failure than simply processing slower.

```java
// Reactive Streams / Project Reactor — consumer explicitly requests capacity
Flux.range(1, 1_000_000)
    .onBackpressureBuffer(1000)              // bounded buffer, not unbounded
    .publishOn(Schedulers.boundedElastic(), 32) // only pull 32 at a time
    .subscribe(this::process);
```

**Senior-level answer:**
> "Backpressure is the piece people most often skip because it requires the *producer* to cooperate, not just the consumer to defend itself — which is why it shows up naturally in reactive systems (Project Reactor, RxJava) where 'request N items' is baked into the protocol, but has to be deliberately engineered on top of something like a raw message queue via consumer-side rate limiting or partition-level consumer lag monitoring that triggers producer throttling."

---

### Q10. In a message-queue-based system (e.g., Kafka), how do you actually implement backpressure since the broker doesn't "push" faster than the consumer pulls?
**Answer:** Kafka consumers are **pull-based by design**, which already gives natural backpressure at the consumer-poll level — a consumer only processes as fast as it calls `poll()`. The real backpressure concern in Kafka systems is usually **upstream of the broker** or **within the consumer's own processing pipeline**:

- **Consumer lag monitoring** — track `records-lag-max`; if lag grows unbounded, that's a signal to scale out consumers (more partitions/instances) or slow the producer, not just "let it queue forever" — unbounded lag eventually risks retention-based data loss.
- **Bounded internal queues** — if a Kafka listener hands records off to an internal async processing pool, that hand-off queue must be **bounded** with a rejection/backoff policy, or you've just moved the unbounded-buffer problem in-process.
- **`max.poll.records` / concurrency tuning** — explicitly cap how much work a single poll pulls in, so the consumer doesn't grab more than it can process before the next `poll()` (which risks a rebalance from missing `max.poll.interval.ms`).

**Senior-level answer:**
> "A subtle failure mode I've seen: a Kafka consumer configured with a large `max.poll.records`, handing every batch off to an unbounded internal thread pool queue to 'process fast.' Under a traffic spike, that internal queue grows unbounded — same OOM risk as no backpressure at all, just moved one layer deeper. Backpressure isn't really about the broker in Kafka; it's about making sure **every hand-off point in your own pipeline is bounded**, end to end."

---

## SECTION 6: FAULT ISOLATION & HEALTH CHECKS

### Q11. What is Fault Isolation as a design principle, and how do Bulkhead/Circuit Breaker relate to it versus being separate concepts?
**Answer:** **Fault isolation** is the umbrella design goal: **contain the blast radius of any single failure so it doesn't cascade into unrelated parts of the system.** Bulkhead, Circuit Breaker, Timeout, and Rate Limiting are all specific **mechanisms** that implement fault isolation at the call/resource level — they're tactics; fault isolation is the strategy.

**Fault isolation also applies above the individual-call level:**
- **Service-level isolation** — deploying/scaling services independently so one service's resource exhaustion (CPU, memory) doesn't starve co-located services (relevant to container/pod resource limits and requests in Kubernetes).
- **Data-level isolation** — database-per-service (see Distributed Data Management module) prevents one service's runaway query from degrading another's DB performance.
- **Zone/region isolation** — deploying across availability zones so an infrastructure failure in one zone doesn't take down the whole service.

**Senior-level answer:**
> "I think of resilience patterns in layers: timeout stops a single call from hanging forever; retry handles transient blips; circuit breaker stops sustained failure from being retried into a storm; bulkhead stops one dependency's failure from starving resources needed by another; and fault isolation at the infrastructure level (K8s resource limits, AZ spread, DB-per-service) stops failures from crossing service boundaries entirely. None of these individually is 'the' resilience strategy — production resilience is the composition of all of them at the right layers."

---

### Q12. What's the difference between a liveness check and a readiness check, and what's the most common mistake teams make configuring them?
**Answer:**
| | Liveness | Readiness |
|---|---|---|
| **Question answered** | "Is this process fundamentally broken and should be restarted?" | "Is this instance currently able to handle traffic?" |
| **Action on failure** | Container/process is **killed and restarted** | Instance is **pulled out of load-balancer rotation**, not restarted |
| **Should check** | Internal deadlock/hung-thread detection, basic app responsiveness | Downstream dependency health (DB, message broker, critical upstream services) |

```yaml
# Spring Boot Actuator health groups mapped to k8s probes
management:
  endpoint:
    health:
      probes:
        enabled: true   # exposes /actuator/health/liveness and /actuator/health/readiness separately
```

**The most common mistake:** wiring a **downstream dependency check (e.g., database connectivity) into liveness instead of readiness**. If the database goes down temporarily, every pod fails its liveness check and Kubernetes **restarts all of them in a crash-loop** — which does nothing to fix the database and actively makes recovery worse by adding restart churn on top of an already-degraded dependency. Downstream checks belong in **readiness**: the pod simply stops receiving new traffic, without restarting, until the dependency recovers.

**Senior-level answer:**
> "This exact misconfiguration is one of the highest-value things to call out proactively in an interview, because it's a real, repeatable production incident pattern — I've both caused and fixed this one. The rule of thumb I give teams: liveness answers 'should this specific process be replaced,' readiness answers 'should this instance receive traffic right now' — and a downstream outage is almost always a readiness concern, never a liveness one."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between semaphore and thread-pool bulkheads?" → Semaphore limits concurrent calls on the *caller's own thread* (lightweight, no isolation from a stuck call blocking that thread); thread-pool bulkhead runs the call on a separate, bounded pool so a stuck call can't block the original request thread at all.
- "Should retries be enabled by default on every remote call?" → No — only on operations confirmed idempotent (Q11 of Distributed Data Management module covers idempotent consumers); retrying a non-idempotent write blindly risks duplicate side effects.
- "Is exponential backoff enough to prevent a retry storm on its own?" → No — jitter is required too; pure exponential backoff without jitter still lets synchronized clients retry in lockstep bursts.
- "Can a circuit breaker and a rate limiter be applied to the same call?" → Yes, and commonly are — Resilience4j supports chaining decorators (RateLimiter → CircuitBreaker → Retry → TimeLimiter) around the same call, each protecting against a different failure mode.
- "What HTTP status should a fallback/degraded response return?" → Depends on semantics — a successful degraded response (e.g., generic recommendations) can return 200; a rejected request due to rate limiting/circuit open should return 429/503 so clients and load balancers can react appropriately, not silently retry blind.
- "Does Kubernetes have a startup probe, and why would you need one separate from liveness?" → Yes — `startupProbe` gives slow-starting apps (large caches, JVM warm-up) time to initialize before liveness checks begin, preventing Kubernetes from killing a healthy-but-still-starting pod for failing early liveness checks.

---

*Study tip: The failure-mode reasoning behind fail-fast (Q6) and the liveness-vs-readiness misconfiguration (Q12) are the two most reliably asked "have you actually run this in production" questions in this module — along with clearly separating bulkhead's blast-radius isolation (Q4) from circuit breaker's duration-of-failure protection (Q1), since interviewers frequently probe whether candidates understand these as complementary, not redundant, patterns.*
