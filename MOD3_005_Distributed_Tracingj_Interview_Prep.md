# Distributed Tracing — Interview Prep
---

## SECTION 1: TRACE ID, SPAN ID & CORRELATION ID FUNDAMENTALS

### Q1. What problem does distributed tracing solve that centralized logging alone doesn't?
**Answer:** In a microservices system, a single user request might touch 5–10 services. With plain per-service logging, each service's logs are isolated — there's no built-in way to answer "show me everything that happened for **this one request**, across every service it touched, in the order it happened." Distributed tracing solves this by attaching a **shared identifier** to a request as it flows through the system, so every service's logs/spans for that request can be correlated and reassembled into a single, ordered timeline — showing exactly where time was spent and where something failed.

**Senior detail:** Tracing isn't a replacement for logging or metrics — it's the third pillar of observability alongside them (logs = discrete events, metrics = aggregated numbers over time, traces = the causal, timed structure of one request's journey). Interviewers often check whether a candidate understands tracing complements, rather than replaces, the other two.

---

### Q2. What is a Trace ID, and what is a Span ID — how do they relate to each other?
**Answer:**
- **Trace ID** — a single identifier assigned to an **entire request's journey**, shared by every service and every operation involved in handling it, from the moment it enters the system until the final response.
- **Span ID** — identifies **one unit of work** within that trace — e.g., "order-service handling this HTTP request," or "a specific DB query inside that handling." A trace is composed of **many spans**, each with its own Span ID, but all sharing the same Trace ID.

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
 ├── Span: gateway [order-service call]         Span ID: 00f067aa0ba902b7
 │    └── Span: order-service [handle request]  Span ID: 1a2b3c4d5e6f7890   parentSpanId: 00f067aa0ba902b7
 │         └── Span: order-service [DB query]   Span ID: 9f8e7d6c5b4a3210   parentSpanId: 1a2b3c4d5e6f7890
```

**Trap:** Confusing Trace ID and Span ID as interchangeable — a common wrong answer treats "trace" and "span" as synonyms. The relationship is strictly hierarchical: one trace, many spans, each span (except the root) has a `parentSpanId` linking it back into the tree.

---

### Q3. What is a Correlation ID, and how does it differ from a Trace ID?
**Answer:** In practice, **"Correlation ID" is often used as a looser, pre-standardization synonym for Trace ID** — both describe an identifier that ties related events together across a request's lifecycle. Historically (pre-OpenTelemetry-standardization era, e.g., early Spring Cloud Sleuth or custom logging setups), teams would generate their own ad-hoc "correlation ID" header purely for **log correlation**, without the richer structured span/parent-child model that a proper tracing system (Trace ID + Span ID + span metadata) provides.

**Senior-level answer:**
> "I treat 'correlation ID' as the general concept — some identifier threading a request through logs — and 'Trace ID' as the specific, standardized implementation of that concept within a real tracing system like OpenTelemetry. If a team says 'we use correlation IDs' but can't describe span parent-child relationships or a trace visualization tool, that's usually a sign they've built basic log correlation, not actual distributed tracing."

---

### Q4. How does a Trace ID/Span ID propagate from one service to the next over an HTTP call?
**Answer:** Via HTTP headers, injected by the calling service's instrumentation before the request goes out, and read by the receiving service's instrumentation before it starts processing. The current standard is the **W3C Trace Context** format (`traceparent` header), which OpenTelemetry and modern Micrometer Tracing use by default:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └────────── trace-id ──────────┘ └── parent-id ──┘ │
           version                                              trace-flags
```

**Senior detail:** This propagation is automatic for common instrumented clients (RestTemplate, WebClient, Feign, Kafka producers/consumers) once the tracing library is on the classpath and auto-configured — you generally don't manually add these headers yourself. The trap: if a call is made through an **uninstrumented** client (e.g., a raw `HttpURLConnection`, or a message broker without tracing instrumentation), propagation silently breaks and that hop becomes untraceable — the trace effectively splits into two disconnected traces.

---

### Q5. What's the difference between a "root span" and a "child span," and what makes a span "the root" of a trace?
**Answer:** The **root span** is the very first span created — typically at the system's entry point (e.g., the API gateway, or the first service receiving an external client request with no incoming trace context header). Every other span in that trace is a **child span**, directly or transitively descended from the root via `parentSpanId` links. A new trace begins (a new root span is created, with a brand-new Trace ID) whenever a request arrives **without** an existing trace context to continue.

**Trap:** Assuming every incoming request automatically continues an existing trace — if the incoming request has no `traceparent` header at all (e.g., a fresh external client call, or a batch job kicking things off internally), the receiving service starts a **new** trace with itself as the root, not a continuation of anything.

---

## SECTION 2: MICROMETER TRACING & OPENTELEMETRY

### Q6. What is Micrometer Tracing, and how does it relate to Micrometer (metrics) and OpenTelemetry?
**Answer:** **Micrometer** is Spring's vendor-neutral **metrics** facade (analogous to SLF4J for logging) — write instrumentation once, plug in any supported metrics backend (Prometheus, Datadog, etc.) without changing application code. **Micrometer Tracing** is the equivalent **facade for distributed tracing** — a vendor-neutral API for creating spans, propagating context, and adding tags, decoupled from which underlying tracing implementation actually does the work.

```
Application code
      │
Micrometer Tracing API   (vendor-neutral facade)
      │
      ├── OpenTelemetry bridge  → OTLP exporter → Jaeger / Zipkin / any OTel-compatible backend
      └── (or) Brave bridge     → Zipkin reporter → Zipkin
```

**Senior detail:** Micrometer Tracing doesn't do the actual span creation/export itself — it delegates to a **bridge** implementation, most commonly either **OpenTelemetry** or **Brave** (Zipkin's native instrumentation library). This is exactly analogous to SLF4J delegating to Logback/Log4j2 — the facade lets application code stay implementation-agnostic.

---

### Q7. What is OpenTelemetry, and why has it become the industry-standard approach over vendor-specific tracing libraries?
**Answer:** OpenTelemetry (OTel) is a **CNCF-hosted, vendor-neutral standard and toolset** for traces, metrics, and logs — a merger of the earlier OpenTracing and OpenCensus projects. It defines a standard data model (spans, trace context propagation format), SDKs for instrumenting applications, and the **OTLP** (OpenTelemetry Protocol) for exporting telemetry data to any compatible backend (Jaeger, Zipkin, Prometheus, commercial APM vendors) without changing instrumentation code.

**Why it won out:** Before OTel, teams instrumenting for Zipkin vs Jaeger vs a commercial APM tool often needed **different instrumentation libraries**, making it painful to switch backends later or support multiple simultaneously. OTel standardizes the instrumentation layer so the **backend becomes a swappable export target**, not a decision baked into your application code.

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

---

### Q8. What is sampling, and why wouldn't you trace 100% of requests in a high-traffic production system?
**Answer:** **Sampling** decides what fraction of traces are actually recorded and exported, versus discarded to keep overhead manageable. Tracing every span adds real CPU, memory, network, and storage cost — at high request volume (e.g., tens of thousands of requests/second), tracing **100%** of traffic can become a meaningful tax on both the application and the tracing backend's storage costs.

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # trace roughly 10% of requests
```

**Senior-level answer:**
> "I use 100% sampling in lower environments (dev/staging) where traffic is low and I want maximum visibility while debugging, and a much lower probability — often 1-10% — in production, tuned based on actual traffic volume and what the tracing backend's storage can sustain. For critical paths I care about disproportionately (payment processing, checkout), I'll sometimes use **head-based sampling overrides** or tail-based sampling in the collector to guarantee those specific traces are always kept, even while the general sampling rate stays low."

---

### Q9. Head-based vs tail-based sampling — what's the difference, and why does it matter for catching errors?
**Answer:**
- **Head-based sampling** — the decision to keep or discard a trace is made **at the very start** (the root span), before knowing how the request will turn out. Simple, low-overhead, but you might discard a trace that would have turned out to contain an error, purely by random chance.
- **Tail-based sampling** — the decision is deferred until the **entire trace completes**, so you can preferentially **always keep traces containing errors or unusually high latency**, and only apply probability-based discarding to the "boring," fully-successful traces.

**Trade-off:** Tail-based sampling gives you much better error visibility, but requires buffering the full trace (all its spans, across all services) somewhere — typically in a dedicated **OpenTelemetry Collector** configured for tail sampling — before the keep/discard decision is made, adding infrastructure complexity and some latency to when a trace is finally exported. Micrometer Tracing/Spring Boot's built-in `sampling.probability` is head-based; tail-based sampling is a collector-level concern, not something configured directly in the application.

---

### Q10. How do you add custom tags/attributes to a span to make it more useful for debugging (e.g., a customer ID or order ID)?
**Answer:** Inject the current `Span` (or `Tracer`) and tag it with business-relevant context — this makes traces searchable/filterable in the tracing UI by fields that actually matter to your domain, not just technical HTTP details.

```java
@Service
public class OrderService {
    private final Tracer tracer;

    public Order processOrder(String orderId, String customerId) {
        Span span = tracer.currentSpan();
        if (span != null) {
            span.tag("order.id", orderId);
            span.tag("customer.id", customerId);
        }
        return doProcessing(orderId);
    }
}
```

**Senior-level answer:**
> "I always tag spans with domain identifiers relevant to the business — order ID, customer ID, tenant ID for multi-tenant systems — because the default HTTP-level tags (method, status, URI) tell you *what request* failed, but not *whose* request or *which business entity* was involved. When investigating a specific customer's reported issue, being able to search the tracing backend by `customer.id` directly, instead of correlating timestamps by hand, is the difference between a five-minute investigation and a much longer one."

---

## SECTION 3: ZIPKIN & JAEGER

### Q11. What are Zipkin and Jaeger, and what role do they play relative to OpenTelemetry/Micrometer Tracing?
**Answer:** Both are **tracing backends** — systems that **receive, store, and visualize** span data collected by instrumentation, letting you search traces and view the request timeline/waterfall in a UI. They don't do the instrumentation themselves; that's OpenTelemetry/Micrometer Tracing's job. Think of it as: **instrumentation** (what generates and propagates trace data) is a separate concern from **storage/visualization** (what you query and look at afterward).

| Aspect | Zipkin | Jaeger |
|---|---|---|
| Origin | Twitter | Uber, now a CNCF graduated project |
| Native protocol | Zipkin's own span format (Micrometer's Brave bridge speaks this natively) | OpenTelemetry-native, also accepts Zipkin/Jaeger-format spans |
| Storage backends | In-memory (dev), Elasticsearch, Cassandra, MySQL | Elasticsearch, Cassandra, or Badger (embedded) |
| Typical fit today | Simpler setups, teams already invested in the Brave/Zipkin ecosystem | Increasingly the default choice for greenfield OTel-based systems, given its native OTel/CNCF alignment |

---

### Q12. How does a Spring Boot application send trace data to Zipkin vs Jaeger — what changes in configuration?
**Answer:** With Micrometer Tracing as the facade, switching backends is largely a **configuration/dependency change**, not an instrumentation rewrite — this is the core value proposition of the facade pattern (Q6).

```yaml
# Sending to Zipkin (via Brave bridge)
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

# Sending to Jaeger (via OTLP, since Jaeger natively speaks OTLP)
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://jaeger-collector:4318/v1/traces
```
```xml
<!-- Zipkin path -->
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-brave</artifactId></dependency>
<dependency><groupId>io.zipkin.reporter2</groupId><artifactId>zipkin-reporter-brave</artifactId></dependency>

<!-- Jaeger/OTLP path -->
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-otlp</artifactId></dependency>
```

**Senior detail:** The actual `@Service`/`@RestController` application code doesn't change at all between these two setups — only the bridge dependency and export endpoint config differ, exactly demonstrating why the facade pattern (Micrometer Tracing) is valuable: application code is written once against a stable API, and the backend is an infrastructure/ops decision, not a development-time one.

---

### Q13. What does the trace "waterfall view" in Zipkin/Jaeger actually show, and how do you use it to diagnose a slow request?
**Answer:** The waterfall view lays out every span in a trace **horizontally by time**, nested by parent-child relationship, showing exactly how long each operation took and — critically — whether operations happened **sequentially** (one waiting on the previous) or **in parallel**. This immediately reveals where the actual time went: is one specific downstream call dominating the total latency, or is the total latency the sum of many small sequential calls that could have been parallelized?

**Production scenario:** A `/api/dashboard` endpoint takes 2.5 seconds end-to-end. The waterfall view shows three downstream calls — `order-service` (200ms), `user-service` (180ms), `inventory-service` (2.1s) — all running **sequentially** back-to-back rather than concurrently. The fix isn't optimizing any single call; it's parallelizing the three independent calls (e.g., via `Mono.zip` or `CompletableFuture.allOf`), which would drop total latency to roughly the slowest single call (~2.1s) instead of their sum.

**Trap:** Assuming the trace only tells you *what* is slow — the waterfall's real diagnostic power is revealing *why* the total is slow: serialized calls that should be parallel, N+1-style repeated child spans (e.g., a loop making 50 individual DB queries instead of one batched query, visible as 50 nearly-identical sibling spans), or a single genuinely slow operation.

---

### Q14. Both Zipkin and Jaeger are running — a specific request is definitely failing, but you can't find its trace. What do you check?
**Answer:** A structured debugging checklist:
1. **Sampling** — if `probability` is less than 1.0, the specific trace you're looking for may simply have been dropped by the sampler and never exported at all.
2. **Broken propagation** — check whether every hop in the request path uses instrumented clients; a call through an uninstrumented library silently breaks the chain, producing two disconnected traces instead of one continuous one, so searching by the "wrong half's" trace ID finds nothing relevant.
3. **Export failures** — the application may be generating spans correctly but failing to actually ship them to the collector (network issue, wrong endpoint config, collector down) — check the application's own tracing-related logs/metrics for export errors.
4. **Time window in the UI** — a surprisingly common false alarm: the tracing UI's default search window (e.g., "last 15 minutes") doesn't cover when the request actually happened.

**Senior-level answer:**
> "For anything customer-reported or urgent, I don't rely purely on chance sampling — I either bump the sampling probability temporarily while investigating, or make sure error-triggering traces are always kept via tail-based sampling in the collector (Q9), so I'm not debugging a production incident with a coin-flip chance the relevant trace even exists."

---

## SECTION 4: SLEUTH → MICROMETER TRACING (MIGRATION AWARENESS)

### Q15. What was Spring Cloud Sleuth, and why was it replaced by Micrometer Tracing?
**Answer:** **Spring Cloud Sleuth** was the original Spring Cloud project providing distributed tracing instrumentation — automatic trace/span creation, header propagation, and Zipkin integration for Spring Boot applications. As part of the broader Spring ecosystem's move toward **Micrometer as the unified observability facade** (metrics *and* tracing sharing one consistent, vendor-neutral model), **Sleuth's tracing responsibilities were absorbed into Micrometer Tracing** starting with Spring Boot 3 / Spring Cloud 2022.0 (Kilburn), and Sleuth itself is now in maintenance mode / effectively superseded for new projects.

**Senior detail:** This is directly analogous to the earlier Ribbon → Spring Cloud LoadBalancer migration and Hystrix → Resilience4j migration — all part of a broader pattern where Spring Cloud has been replacing Netflix-OSS-derived (or Sleuth-specific) components with more modular, standards-aligned successors over the past several release trains. Recognizing this pattern across multiple modules is itself a strong senior signal.

---

### Q16. What are the practical, code-level differences a team hits when migrating from Sleuth to Micrometer Tracing?
**Answer:**
| Aspect | Spring Cloud Sleuth | Micrometer Tracing |
|---|---|---|
| Dependency | `spring-cloud-starter-sleuth` | `micrometer-tracing-bridge-brave` or `-otel`, plus an exporter dependency |
| API for manual span creation | Sleuth's own `Tracer`/`Span` API | Micrometer's `Tracer`/`Span` API (similar concepts, different package/API surface) |
| Default underlying implementation | Brave (Zipkin's library), tightly coupled | Brave **or** OpenTelemetry — your choice via bridge dependency |
| Spring Boot version | Spring Boot 2.x | Spring Boot 3.x |
| Log correlation (trace/span ID in log lines) | Automatic via Sleuth's Slf4jSpanLogger | Automatic via Micrometer's logging integration, same practical outcome, different underlying wiring |

**Trap:** A candidate confidently describing Sleuth-specific APIs/annotations as the current approach in a Spring Boot 3 context is showing the same kind of dated knowledge as describing Ribbon or Hystrix as current — interviewers use this specific migration as a quick "how current is your Spring Cloud knowledge" check.

---

### Q17. If you inherited a Spring Boot 2.x application using Sleuth, would you migrate it to Micrometer Tracing immediately? What would you weigh?
**Answer:** Not automatically, and not in isolation — Micrometer Tracing requires **Spring Boot 3.x**, so adopting it means the application must first go through (or be part of) a broader Spring Boot 2 → 3 migration, which itself carries other breaking changes (`javax.*` → `jakarta.*` namespace migration being the biggest one). Migrating tracing alone isn't possible without that larger upgrade.

**Senior-level answer:**
> "I wouldn't treat 'Sleuth is legacy' as a reason to rush a migration in isolation — I'd fold it into the team's broader Spring Boot 3 upgrade plan, since that's a prerequisite anyway. What I would flag as a genuine reason to prioritize the overall upgrade sooner rather than later is that Sleuth itself stops receiving meaningful updates, so any new tracing capability (e.g., newer OTel semantic conventions, better sampling strategies) is only landing in Micrometer Tracing going forward — staying on Sleuth means slowly falling behind on tracing capabilities, not just missing a rename."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "How many Trace IDs exist across all spans in one request's journey?" → Exactly one — that's the definition of a trace; many Span IDs share it.
- "What header format does modern tracing propagation use by default?" → W3C Trace Context (`traceparent`).
- "Does Micrometer Tracing create spans itself, or delegate?" → Delegates, via a bridge (Brave or OpenTelemetry).
- "Is 100% sampling a good default for production?" → Generally no at meaningful traffic volume — tune probability based on cost/visibility trade-offs, or use tail-based sampling for error visibility.
- "Can you switch from Zipkin to Jaeger without changing application code?" → Yes, largely — swap the bridge/exporter dependency and config; instrumentation code is unaffected.
- "Is Spring Cloud Sleuth the current recommendation for Spring Boot 3?" → No — Micrometer Tracing is current; Sleuth is effectively superseded.
- "What does a broken propagation hop look like in the tracing backend?" → Two disconnected traces instead of one continuous trace across that boundary.
- "Head-based or tail-based sampling for guaranteeing error traces are kept?" → Tail-based — the keep/discard decision is deferred until the outcome (error or not) is known.

---

*Study tip: The facade-pattern relationship between Micrometer Tracing and its bridges (Q6, Q12), head-based vs tail-based sampling trade-offs (Q9), and the Sleuth → Micrometer Tracing migration context tied to the Spring Boot 3 upgrade (Q17) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY the architecture is layered this way, not just naming the tools.*