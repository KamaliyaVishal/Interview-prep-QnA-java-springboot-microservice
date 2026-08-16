# Service Communication — Interview Prep
---

## SECTION 1: SYNCHRONOUS vs ASYNCHRONOUS COMMUNICATION

### Q1. Synchronous vs Asynchronous inter-service communication — compare them, and how do you actually decide which one to use for a given call?
**Answer:** **Synchronous** communication means the caller blocks and waits for a response before proceeding (REST/gRPC over HTTP) — the caller's availability becomes coupled to the callee's availability *at that moment*. **Asynchronous** communication means the caller sends a message/event and moves on without waiting (Kafka/RabbitMQ) — the two services are decoupled in time; the callee can process it whenever it's ready.

| Aspect | Synchronous | Asynchronous |
|---|---|---|
| Coupling | Temporal coupling — both services must be up **at the same time** | Decoupled in time — producer and consumer don't need to be simultaneously available |
| Response | Immediate — caller gets a result (or error) right away | Delayed/eventual — caller doesn't get an immediate result |
| Failure impact | Callee down → caller's request fails or blocks | Callee down → message queues up, processed later, caller unaffected |
| Use case fit | Need an immediate answer to proceed (e.g., "is this payment authorized?") | Fire-and-forget notifications, workflows tolerant of eventual consistency (e.g., "send a confirmation email") |
| Complexity | Simpler to reason about, easier to debug (request → response) | Harder to trace (message → eventual side effect), needs a broker |
| Consistency model | Can enforce request-time validation before proceeding | Naturally fits eventual consistency (Sagas, CQRS) |

**How I actually decide:** "I ask: does the caller **need the result to make its next decision right now**? Checkout needing to know 'was the card charged' is a hard synchronous dependency — the flow can't sensibly continue without that answer. But 'send the customer a shipping confirmation email' has no reason to block the order-fulfillment flow — that should be a published event, consumed asynchronously by a notification service. Mixing this up is one of the most common real design mistakes I see: services making synchronous calls for things that were never actually time-critical, just because REST was the default."

**Senior trap to flag proactively:** Overusing synchronous calls for internal service-to-service chains creates the **chatty call chain** problem (a single user request may trigger 5+ blocking hops) and directly increases the risk of cascading failures — a slow downstream service can exhaust the caller's thread pool and take down services that have nothing to do with the actual failure (this is exactly the failure mode Resilience4j patterns exist to contain, Q6).

---

## SECTION 2: REST vs gRPC

### Q2. REST vs gRPC — compare them properly, and when have you actually chosen one over the other?
**Answer:**
| Aspect | REST (typically JSON over HTTP/1.1) | gRPC (Protocol Buffers over HTTP/2) |
|---|---|---|
| Payload format | JSON — human-readable, larger, text-based | Protobuf — binary, compact, requires a `.proto` schema |
| Performance | Slower serialization, larger payloads | Faster serialization, smaller payloads — meaningful at high throughput |
| Contract | Loose — OpenAPI/Swagger is optional documentation, not enforced | Strict — `.proto` file is the enforced contract; client/server stubs generated from it |
| Streaming | Not natively (typically request/response only, or hacks like SSE) | Native support for **client, server, and bidirectional streaming** over a single HTTP/2 connection |
| Browser support | Native — every browser/client speaks HTTP/JSON | Requires **gRPC-Web** + a proxy for browser clients — not natively browser-friendly |
| Tooling/debugging | Easy — curl, Postman, browser dev tools all work directly | Needs specialized tooling (`grpcurl`, BloomRPC) — binary payloads aren't human-readable on the wire |
| Best fit | Public APIs, browser-facing APIs, external partner integrations | Internal service-to-service calls, especially high-throughput/low-latency paths, streaming use cases |

```proto
// gRPC contract — the .proto file IS the enforced API contract
service PaymentService {
  rpc AuthorizePayment (PaymentRequest) returns (PaymentResponse);
  rpc StreamTransactionUpdates (AccountId) returns (stream TransactionUpdate);  // native streaming
}
message PaymentRequest {
  string order_id = 1;
  double amount = 2;
}
```

**Real decision I've made:** "For our public-facing partner API, we kept REST/JSON — external partners need something they can hit with curl/Postman without generating client stubs, and browser-based partner dashboards need direct compatibility. But for **internal, high-volume service-to-service calls** — specifically a pricing service getting called on nearly every product page render — we moved that one path to gRPC and measured a meaningful drop in p99 latency and CPU spent on serialization, purely from Protobuf's compactness and HTTP/2 multiplexing. I wouldn't default to gRPC everywhere though — the loss of easy human-debuggability and browser support is a real cost for anything customer/partner-facing."

**Follow-up: "Can you mix both in the same system?"** → Yes, and it's common — REST/GraphQL at the edge (BFF/API Gateway, MOD4_003 Q11) facing external clients, gRPC for internal service-to-service calls behind that boundary.

---

## SECTION 3: EVENT-DRIVEN COMMUNICATION

### Q3. What is event-driven communication, and what are the two main event styles (event notification vs event-carried state transfer) — why does the distinction matter?
**Answer:** Event-driven communication means services communicate by **publishing and subscribing to events** through a broker (Kafka/RabbitMQ) rather than calling each other directly — producers don't know or care who (if anyone) consumes their events.

**Two distinct styles, and the distinction is a common interview trap:**

**1. Event Notification (thin event)** — the event just says "something happened," with minimal data (often just an ID); consumers must **call back** to the producer's API to get full details.
```json
{ "eventType": "OrderPlaced", "orderId": "ORD-123" }
```
- Pro: small payload, producer's data never goes stale in the event itself.
- Con: **reintroduces synchronous coupling** — every consumer still has to call back to Order-service to get details, which partly defeats the purpose of decoupling via events.

**2. Event-Carried State Transfer (fat event)** — the event carries **all the data a consumer would plausibly need**, so consumers never need to call back.
```json
{ "eventType": "OrderPlaced", "orderId": "ORD-123", "customerId": "CUST-9",
  "items": [{"sku": "ABC", "qty": 2, "price": 19.99}], "totalAmount": 39.98 }
```
- Pro: true decoupling — consumers build their own local read models (this is exactly how CQRS read models get populated, MOD4_003 Q8) without ever calling the producer synchronously.
- Con: payload duplication across services, and **eventual staleness** — a consumer's copy of "customer name" could lag behind the source of truth until the next event.

**Senior-level answer:** "I default to **event-carried state transfer** for anything where I want genuine decoupling and the consumer needs to build its own local view — that's the whole point of going async in the first place. I only use thin notification events when the payload would be prohibitively large or highly sensitive (e.g., an event just says 'PaymentDetailsUpdated' rather than carrying card data), where the callback's synchronous dependency is an acceptable, deliberate trade-off."

---

### Q4. Kafka vs RabbitMQ for event-driven communication — what's the actual difference in how you'd use them?
**Answer:**
| Aspect | Kafka | RabbitMQ |
|---|---|---|
| Model | **Distributed log** — messages persist for a configured retention period; consumers track their own offset | **Traditional message broker/queue** — message is typically removed once acknowledged/consumed |
| Replay | Consumers can **re-read history** by resetting their offset — new consumers can be added later and replay from the beginning | No native replay — once consumed and acked, it's gone (unless explicitly designed otherwise) |
| Throughput | Very high — built for high-volume event streaming | High, but generally lower ceiling than Kafka for pure throughput |
| Routing | Simple — topic + partition key | Rich routing — exchanges (direct/topic/fanout/headers) for complex routing logic |
| Ordering | Guaranteed **within a partition** (same key → same partition → ordered) | FIFO per queue, but less natural fit for "ordered by entity ID" semantics |
| Best fit | Event sourcing, stream processing, audit logs, high-throughput event pipelines, feeding multiple independent consumers | Task queues, complex routing needs, RPC-style request/reply over messaging, lower-throughput enterprise integration |

**Real decision:** "We used Kafka for our order-events pipeline specifically because we needed **multiple independent consumers** (fraud detection, analytics, notifications, inventory) all reading the same event stream **at their own pace**, plus the ability to replay history when we onboarded a new consumer team months later — RabbitMQ's consume-and-remove model doesn't naturally support that. For a simpler background job queue — 'process this image resize task, exactly once, then discard it' — RabbitMQ's traditional queue semantics were the simpler, more appropriate fit."

---

## SECTION 4: IDEMPOTENCY & DUPLICATE REQUEST HANDLING

### Q5. Why does idempotency matter so much in distributed/microservices communication, and how do you actually implement it?
**Answer:** In a distributed system, **at-least-once delivery is the realistic default** — retries (Q6), message broker redelivery, and network timeouts where the caller doesn't know if the request actually succeeded, all mean the **same logical operation can arrive more than once**. Without idempotency, a retried "charge $50" request could charge the customer twice. **Idempotency** means processing the same request multiple times produces the **same result as processing it once** — the operation is safe to retry blindly.

**Implementation — Idempotency Key pattern (the standard approach):**
```java
@PostMapping("/payments")
public ResponseEntity<PaymentResult> charge(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    Optional<PaymentResult> existing = idempotencyStore.find(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());   // already processed — return the SAME stored result, don't re-charge
    }

    PaymentResult result = paymentProcessor.charge(request);
    idempotencyStore.save(idempotencyKey, result);  // store atomically with the charge, ideally same transaction
    return ResponseEntity.ok(result);
}
```

**Key design points to state proactively:**
- The **client generates the idempotency key** (typically a UUID) once per logical operation, and sends the **same key on every retry** of that same logical request — the server never generates it itself.
- The idempotency-key lookup and the actual business operation should be **atomic together** (same DB transaction) — otherwise a crash between "process payment" and "record the key" reopens the exact race condition idempotency was meant to close.
- For **event consumers** (Kafka), idempotency is achieved differently — track processed **event IDs** (or use the event's natural key + a dedupe table) so re-delivery of the same event (which at-least-once delivery guarantees will eventually happen) doesn't double-apply the effect.

```java
@KafkaListener(topics = "order-events")
public void handle(OrderPlacedEvent event) {
    if (processedEventRepository.existsById(event.getEventId())) {
        return;   // already handled this exact event — skip, don't reprocess
    }
    inventoryService.reserve(event.getItems());
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

**Senior talking point:** "I treat idempotency as **mandatory**, not optional, for any operation with a side effect that's triggered over the network — because retries aren't a rare edge case in distributed systems, they're the expected behavior of a correctly-implemented resilient client. The question isn't 'will a duplicate request happen,' it's 'when it happens, is my system safe' — and naturally idempotent operations (e.g., `setStatus(SHIPPED)` instead of `incrementShipCount()`) are the cheapest way to get this for free where the domain allows it."

---

## SECTION 5: API VERSIONING

### Q6. What are the different approaches to API versioning, and which do you actually recommend?
**Answer:**
| Strategy | Example | Trade-off |
|---|---|---|
| **URI versioning** | `/api/v1/orders`, `/api/v2/orders` | Most explicit and cache-friendly; clutters the URI; "is the resource really different, or just the representation?" purists object |
| **Header versioning** | `Accept: application/vnd.company.order.v2+json` | Keeps URIs clean; less discoverable, harder to test with a browser/curl casually |
| **Query parameter** | `/api/orders?version=2` | Simple, but easy to forget/omit, easy to accidentally cache incorrectly |
| **No versioning — additive-only changes** | Always add optional fields, never remove/rename existing ones | Avoids versioning entirely for a long time; requires strict API evolution discipline |

**My actual recommendation:** "I default to **URI versioning** (`/v1/`, `/v2/`) for public/partner-facing APIs — it's the most explicit, most cache-friendly, and easiest for external consumers to reason about and pin to. Internally, between our own services, I push much harder for **backward-compatible, additive-only evolution** — add new optional fields, never remove or repurpose existing ones, and use consumer-driven contract tests (Pact) to catch a breaking change at CI time before it ever reaches a consumer — so we rarely need a `v2` internal contract at all except for genuinely breaking domain-model changes."

**What actually counts as a breaking change (a common interview follow-up):**
- Breaking: removing a field, renaming a field, changing a field's type, tightening validation on an existing field, changing an enum's existing values.
- Non-breaking: adding a new optional field, adding a new endpoint, adding a new enum value **if consumers are contractually required to handle unknown values gracefully** (a discipline worth calling out explicitly).

**Senior talking point on deprecation:** "Versioning isn't just about adding v2 — it's equally about **retiring v1** deliberately: announce a deprecation window, monitor actual v1 traffic (Actuator/metrics tell you who's still calling it), and only remove it once usage is genuinely zero or the deprecation deadline is enforced — silently breaking a partner because 'v1 was old' is a trust-destroying mistake I've seen happen."

---

## SECTION 6: TIMEOUT, RETRY & COMMUNICATION FAILURE HANDLING

### Q7. Why is setting a timeout on every inter-service call non-negotiable, and what happens in production when it's missing?
**Answer:** Without an explicit timeout, a caller will **wait indefinitely** (or until the underlying TCP/socket default, which is often far too long) for a downstream service that's slow or hung — during that wait, the caller's thread (or connection pool slot) is **held**, not released. Under load, this **exhausts the caller's own thread pool/connection pool**, meaning the caller becomes unresponsive **to everyone**, not just to requests depending on the slow downstream — a single slow dependency cascades into a much broader outage.

**Real production example:** "We had an inventory service go into GC-pause-induced slowness — not fully down, just very slow to respond. Because the calling checkout service had no explicit timeout configured (just relying on the HTTP client's default, which was effectively unbounded), every checkout request calling inventory piled up holding a connection-pool slot, the pool exhausted within seconds, and checkout became completely unresponsive **for every user**, including the ones whose carts didn't even touch the slow inventory path. A single configured timeout (and a circuit breaker, see below) would have contained the blast radius to just the inventory-dependent requests failing fast, instead of taking the whole service down."

**What a properly resilient call actually needs (all four working together):**
```java
@Retry(name = "inventoryService", fallbackMethod = "fallbackStock")
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackStock")
@TimeLimiter(name = "inventoryService")
public CompletableFuture<StockLevel> getStock(String productId) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.getStock(productId));
}

public CompletableFuture<StockLevel> fallbackStock(String productId, Throwable t) {
    return CompletableFuture.completedFuture(StockLevel.unknown());  // graceful degradation, not a hard failure
}
```
```yaml
resilience4j:
  timelimiter:
    instances:
      inventoryService:
        timeout-duration: 2s              # fail fast — don't wait forever
  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true   # avoid hammering an already-struggling service
        retry-exceptions: java.io.IOException
  circuitbreaker:
    instances:
      inventoryService:
        sliding-window-size: 20
        failure-rate-threshold: 50         # trip open after 50% of the last 20 calls fail
        wait-duration-in-open-state: 10s   # stop calling the failing service entirely for a cooldown period
```

---

### Q8. Walk me through how Timeout, Retry, and Circuit Breaker fit together — what happens if you get the combination wrong?
**Answer:** They solve **different, complementary problems**, and misconfiguring how they interact is one of the most common real production incidents in distributed systems:

- **Timeout** — bounds how long you'll wait for a single call before giving up (fail fast instead of hanging).
- **Retry** — handles **transient** failures (a momentary network blip, a brief GC pause) by trying again, ideally with **exponential backoff + jitter** to avoid synchronized retry storms across many callers hitting the same struggling service at once.
- **Circuit Breaker** — protects against **sustained** failures — once a downstream service is failing consistently, the breaker **trips open** and stops sending traffic to it entirely for a cooldown period, instead of continuing to retry against something that's clearly not recovering, giving the failing service room to actually recover instead of being hammered by everyone's retries.

**What goes wrong if you get the combination wrong (a genuinely common mistake):**
> "Retrying without a circuit breaker, against a downstream service that's failing under load, is actively harmful — every caller's retries **compound the load** on the already-struggling service, making recovery less likely, not more. I've seen this exact 'retry storm' pattern turn a brief blip into a prolonged outage. The correct order of defenses is: **timeout** (fail fast per call) → **retry with backoff** (handle brief transient blips only, capped attempts) → **circuit breaker** (stop entirely once failures are sustained, give the dependency room to recover) → **fallback** (degrade gracefully — cached data, default value, or a clear error — instead of cascading the failure upstream)."

**Bulkhead, the pattern that's often asked as a follow-up here:** isolate the **thread pool/connection pool per downstream dependency**, so a slow inventory-service call can only exhaust *its own* dedicated pool of threads, not the shared pool also used for calls to payment-service or the database — containing the blast radius exactly the way a ship's bulkheads contain flooding to one compartment.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is gRPC always faster than REST?" → Generally yes for internal high-throughput calls due to Protobuf's compact binary format and HTTP/2 multiplexing, but the difference is often negligible for low-volume calls, and REST's simplicity/tooling can outweigh the raw performance gain for many services.
- "Can an operation be naturally idempotent without an idempotency key?" → Yes — `PUT /orders/123 {status: SHIPPED}` is naturally idempotent (same result no matter how many times it's applied); `POST /orders/123/increment-ship-count` is not — prefer designing operations to be naturally idempotent where the domain allows it, before reaching for an idempotency-key mechanism.
- "Should retries be used for 4xx errors?" → No — 4xx (client errors, e.g., 400 Bad Request, 404 Not Found) generally indicate the request itself is wrong and retrying won't help; only retry on 5xx/transient network errors (with the notable exception of 429 Too Many Requests, which explicitly signals "retry later," often with a `Retry-After` header).
- "What's 'jitter' in retry backoff, and why does it matter?" → Adding a small random delay on top of exponential backoff, so that many callers who all failed at the same moment don't all retry at exactly the same synchronized instant and cause a new spike — jitter spreads retries out over time.
- "Does event-driven communication remove the need for timeouts/retries?" → No — a message broker still needs consumer-side idempotency (Q5) and its own failure handling (dead-letter queues for messages that repeatedly fail processing); async communication changes *where* failure handling lives, it doesn't eliminate the need for it.
- "How do you version an event schema, not just a REST API?" → Same additive-only discipline applies — add new optional fields to the event, never remove/rename existing ones; a **schema registry** (e.g., Confluent Schema Registry with Avro) enforces this compatibility rule automatically at publish time rather than relying on developer discipline alone.

---

*Study tip: The interplay between Timeout, Retry, and Circuit Breaker (Q7-Q8) — and specifically what breaks when they're combined incorrectly — is where interviewers most reliably separate candidates who know the pattern names from those who've actually been paged for the outage those patterns are designed to prevent. Idempotency (Q5) is the other area worth over-preparing — it comes up in nearly every senior microservices interview in some form.*
