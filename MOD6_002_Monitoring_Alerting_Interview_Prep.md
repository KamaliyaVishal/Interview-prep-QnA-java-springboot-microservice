# Monitoring & Alerting — Interview Prep
---

## SECTION 1: PROMETHEUS, GRAFANA & MICROMETER

### Q1. Explain Prometheus's architecture — specifically, why does it use a pull model instead of applications pushing metrics to it?
**Answer:** Prometheus **scrapes** (pulls) metrics by periodically issuing an HTTP GET to each target's `/metrics` endpoint, rather than having applications push metrics to it.

```
Prometheus Server                    Target Apps
   |-- scrape every 15s -->  GET /actuator/prometheus  (order-service)
   |-- scrape every 15s -->  GET /actuator/prometheus  (payment-service)
   |-- scrape every 15s -->  GET /actuator/prometheus  (inventory-service)
Service Discovery (k8s/Consul) tells Prometheus WHICH targets exist
```

**Why pull, deliberately:**
- **Centralized control of scrape frequency/load** — Prometheus decides the cadence; a misbehaving app can't flood the monitoring system with a push storm.
- **Built-in liveness signal** — if a scrape fails, Prometheus immediately knows the target is unreachable (`up == 0`), without needing a separate heartbeat mechanism.
- **Simpler service discovery integration** — Prometheus queries Kubernetes/Consul for the current set of targets and scrapes whatever exists **now** — no need for every new instance to know where to push to.

**Senior-level answer:**
> "The trade-off is that pull doesn't work well for **short-lived, batch, or ephemeral jobs** that finish before a scrape can happen — that's exactly why Prometheus has the **Pushgateway** as an explicit exception (Q4), not a general-purpose replacement for the pull model. I'd frame this as: pull is right for the common case of long-running services, push is the deliberate escape hatch for the minority case of jobs that don't fit that shape."

---

### Q2. What is Micrometer, and how does it relate to Prometheus in a Spring Boot application?
**Answer:** **Micrometer** is a **vendor-neutral metrics facade** for the JVM — think "SLF4J, but for metrics" — application code instruments against Micrometer's API, and a **registry implementation** translates that into the format a specific backend (Prometheus, Datadog, CloudWatch, etc.) expects. Switching monitoring backends means swapping the registry dependency, not rewriting instrumentation code.

```java
// Application code — instrumented against Micrometer, backend-agnostic
@RestController
public class OrderController {
    private final Counter orderCounter;

    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed")
                .tag("channel", "web")
                .register(registry);   // registry impl determines the actual backend
    }

    @PostMapping("/orders")
    public OrderResponse placeOrder(@RequestBody OrderRequest req) {
        orderCounter.increment();
        // ...
    }
}
```

```xml
<!-- Adding this dependency alone makes Spring Boot Actuator expose /actuator/prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Senior-level answer:**
> "This is the same abstraction pattern as SLF4J-over-Logback/Log4j2, applied to metrics — and it's a strong Spring Boot–specific answer to give, because it directly explains why Spring Boot apps can switch between Prometheus, Datadog, and CloudWatch with essentially a dependency swap and a config change, not an instrumentation rewrite. I'd cite this as a concrete example of the value of coding against an abstraction rather than a vendor SDK directly."

---

### Q3. What role does Grafana play, and what does a basic PromQL query look like?
**Answer:** Grafana is a **visualization and dashboarding layer** — it doesn't store metrics itself; it **queries** a data source (Prometheus, Datadog, Loki, etc.) and renders the results as graphs, tables, and alerting rules. Prometheus and Grafana are complementary, not competing — Prometheus is the time-series database and query engine, Grafana is the UI on top of it (and on top of other data sources too).

```promql
# p99 latency for the checkout endpoint over the last 5 minutes
histogram_quantile(0.99, 
  sum(rate(http_server_requests_seconds_bucket{uri="/checkout"}[5m])) by (le)
)

# error rate as a percentage
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m])) * 100
```

**Senior-level answer:**
> "A distinction worth making explicit: PromQL's `rate()` function is doing more than it looks like — it computes a **per-second average rate of increase** over the given time window from a monotonically increasing counter, correctly handling counter resets (e.g., on app restart) — using raw counter values directly instead of `rate()` is a very common mistake that produces meaningless graphs with sharp negative spikes every time a pod restarts."

---

### Q4. When would you need the Prometheus Pushgateway, and what's the risk of overusing it?
**Answer:** The **Pushgateway** exists for **short-lived batch jobs** that complete and exit before Prometheus's next scheduled scrape could ever reach them — the job pushes its final metrics to the Pushgateway, and Prometheus scrapes the **Pushgateway** (a long-running intermediary) instead of the job directly.

```java
// Batch job pushes its result before exiting
PushGateway pg = new PushGateway("pushgateway:9091");
Gauge lastSuccess = Gauge.build().name("batch_job_last_success_timestamp").help("...").create();
lastSuccess.setToCurrentTime();
pg.pushAdd(registry, "nightly_reconciliation_job");
```

**The overuse risk:** Using the Pushgateway for regular long-running services defeats Prometheus's built-in target-liveness detection (`up`) — the Pushgateway will happily keep serving the **last pushed value forever**, even if the job that pushed it has been dead for days, silently masking failures instead of surfacing them.

**Senior-level answer:**
> "This is explicitly called out in Prometheus's own documentation as an anti-pattern when misused, and it's a good thing to volunteer in an interview — 'the Pushgateway is for batch jobs, not a general push-based alternative to scraping, because you lose the automatic staleness/liveness detection that's one of Prometheus's core value propositions.' Knowing the exception **and** its specific failure mode if misapplied is a stronger answer than just knowing the Pushgateway exists."

---

## SECTION 2: DATADOG, SPLUNK & ELK

### Q5. When would you choose a managed platform like Datadog over a self-hosted Prometheus/Grafana stack?
**Answer:**

| | Prometheus + Grafana (self-hosted) | Datadog (managed) |
|---|---|---|
| **Cost model** | Infrastructure + engineering time to run/scale it | Per-host/per-metric SaaS pricing — can get expensive at scale |
| **Operational burden** | You own scaling, storage retention, HA setup | Fully managed — no infra to run |
| **Integration breadth** | Requires exporters/instrumentation per tech | Very broad out-of-box integrations (APM, logs, RUM, infra, security in one platform) |
| **Data correlation** | Requires stitching Prometheus + Loki + Tempo yourself | Logs/metrics/traces unified in one UI natively |
| **Vendor lock-in** | Low — open standards (PromQL, OTLP) | Higher — proprietary query language/UI |

**Senior-level answer:**
> "I'd frame the decision around **team size and observability maturity**, not just cost. A small-to-mid team without dedicated SRE/platform capacity generally gets to a good outcome faster with Datadog — the setup cost of getting logs, metrics, and traces correlated well in a self-hosted stack (Prometheus + Loki + Tempo + Grafana, all wired together correctly) is nontrivial. A larger org with platform engineering capacity, or one sensitive to the recurring SaaS cost at scale, benefits from owning the stack — and the open standards (Prometheus's exposition format, OpenTelemetry/OTLP) mean you're not locked in either way if instrumentation is done through Micrometer/OTel from the start, regardless of which backend you point it at."

---

### Q6. What is the ELK stack, and what role does each component play?
**Answer:** ELK = **Elasticsearch, Logstash, Kibana** — a log aggregation and search stack (often paired with **Beats** or **Filebeat** for log shipping, sometimes called "Elastic Stack" now).

- **Elasticsearch** — the search/storage engine; stores logs as indexed, searchable documents (a search index, not a relational DB) — this is what makes free-text and structured-field log queries fast at scale.
- **Logstash** (or lighter-weight **Filebeat**) — the ingestion/parsing pipeline; reads raw logs, parses/transforms them (e.g., extracting structured fields from a log line via grok patterns), and ships them into Elasticsearch.
- **Kibana** — the visualization/query UI on top of Elasticsearch — dashboards, saved searches, log exploration.

```
App logs (JSON, stdout) -> Filebeat (ships) -> Logstash (parse/enrich, optional) -> Elasticsearch (index/store) -> Kibana (query/visualize)
```

**Senior-level answer:**
> "The detail worth knowing: if your application already emits **structured JSON logs** (Section 2 of the Observability Fundamentals topic), you often don't need Logstash's parsing step at all — Filebeat can ship pre-structured JSON directly to Elasticsearch, which is both simpler and less brittle than writing grok patterns to parse unstructured text after the fact. This is a good example of how structured logging upstream simplifies the entire downstream pipeline."

---

### Q7. Splunk vs. the ELK stack — what's the practical trade-off?
**Answer:**

| | Splunk | ELK / Elastic Stack |
|---|---|---|
| **Model** | Commercial, managed or self-hosted; very mature enterprise features | Open-source core (with a paid tier for advanced features), more setup/ops burden |
| **Cost** | Historically priced by data ingest volume — can get very expensive at scale | Generally cheaper at scale if self-hosted and operated well |
| **Query language** | SPL (Search Processing Language) — powerful, but proprietary | Elasticsearch Query DSL / KQL (Kibana Query Language) |
| **Ease of use / polish** | Generally considered more turnkey, better out-of-box UX for non-engineers | More flexible but requires more tuning/expertise |

**Senior-level answer:**
> "In my experience the decision usually comes down to **budget and existing organizational investment** more than raw technical superiority — both are mature, capable log platforms. Splunk's ingest-volume-based pricing is the thing I'd flag as a real operational risk to watch: log volume tends to grow faster than teams expect (especially with verbose structured logging), and Splunk costs scale directly with that, which has driven a lot of migrations toward ELK or a metrics-first approach that reduces reliance on high-volume raw log search for routine monitoring."

---

## SECTION 3: SLIs, SLOs & SLAs

### Q8. Define SLI, SLO, and SLA precisely, with a concrete example.
**Answer:**
- **SLI (Service Level Indicator)** — an actual **measured** value: "the percentage of HTTP requests that returned a 2xx/3xx status in the last 28 days" = **99.92%**.
- **SLO (Service Level Objective)** — an **internal target** for that SLI: "we aim for the success-rate SLI to be ≥ **99.9%**." This drives engineering priorities and error-budget decisions (Q10).
- **SLA (Service Level Agreement)** — an **external, often contractual** commitment to customers, typically with a **looser** threshold than the internal SLO (to leave margin for error) and usually tied to **consequences** (service credits, penalties) if breached: "we guarantee ≥ **99.5%** uptime, or you receive a service credit."

```
SLI (measured):    99.92% success rate this month
SLO (internal target): >= 99.9%   -> currently meeting it, small error budget remaining
SLA (external promise): >= 99.5%   -> comfortable margin below the SLO, protects against SLA breach
```

**Senior-level answer:**
> "The relationship that matters: SLA should always be **looser** than SLO, which should be **achievable relative to** what the SLI actually measures — if your SLA promises the same number as your internal SLO target, you have **zero margin**, and any month you merely meet (not beat) your internal target technically breaches the external contractual commitment. This gap between SLO and SLA is a deliberate buffer, not an oversight, and being able to explain *why* the gap exists is a stronger answer than just reciting the three definitions."

---

### Q9. How do you choose good SLIs for a service? What makes an SLI "good"?
**Answer:** Good SLIs are:
- **User-centric** — measure what the **user actually experiences**, not an internal implementation detail. "Checkout success rate" is a good SLI; "CPU usage" generally isn't (it's a cause, not a symptom the user feels).
- **Measurable at the point closest to the user** — e.g., measuring latency at the load balancer/edge reflects real user experience better than measuring it inside one internal service, which misses network hops, retries, and downstream delays.
- **Aligned to the RED method** for request-driven services (Q11) — availability (successful request rate), latency, and often a domain-specific one (e.g., "search results returned within 1s").

**Senior-level answer:**
> "I push back when a team proposes an SLI like 'database CPU below 80%' — that's a **cause-oriented, internal metric**, not a symptom the user experiences; a database can be at 95% CPU and every user request is still succeeding fine within latency budget. Good SLIs are almost always **request-outcome-based**: was the request successful, was it fast enough. Internal resource metrics belong in dashboards and cause-based alerting (Q14), not as the SLI itself — conflating the two is one of the most common SLO-design mistakes I've seen teams make."

---

### Q10. What is an error budget, and how does it change how a team operates?
**Answer:** If your SLO is 99.9% success rate over 30 days, your **error budget** is the allowed 0.1% of requests that can fail without breaching the SLO — a concrete, spendable quantity ("we can have roughly 43 minutes of full downtime this month, or an equivalent partial-degradation amount, and still be within SLO").

```
30-day window, SLO = 99.9% availability
Error budget = 0.1% of 30 days ≈ 43.2 minutes of allowed downtime
If 30 minutes already consumed this month by an incident -> ~13 minutes of budget remain
```

**How it changes team behavior:**
- **Budget remaining → team can ship features, take calculated risks** (a risky deploy, a new experimental feature flag rollout).
- **Budget exhausted → team shifts focus to reliability work** — freezes risky releases, prioritizes fixing the root causes that burned the budget, until it recovers.

**Senior-level answer:**
> "The real value of an error budget is that it turns 'reliability vs. feature velocity' from a subjective, often political argument into a **quantitative, pre-agreed policy** — this is Google's SRE book's central contribution on this topic. Rather than an engineer having to convince a PM 'we should slow down and fix reliability,' the error budget being exhausted **automatically** triggers that conversation with an agreed-upon number everyone signed off on ahead of time, which removes a lot of the friction and politics from that trade-off decision."

---

## SECTION 4: ERROR RATE, LATENCY, THROUGHPUT & SATURATION

### Q11. Compare the RED method and the USE method — when do you use each?
**Answer:**
- **RED** (Rate, Errors, Duration) — designed for **request-driven services**: how many requests, how many failed, how long did they take. Best for **application-level** monitoring (APIs, microservices).
- **USE** (Utilization, Saturation, Errors) — designed for **resources** (CPU, memory, disk, network, connection pools, thread pools): how busy is it, how much queued/backed-up work exists, how many errors. Best for **infrastructure/resource-level** monitoring.

| | RED | USE |
|---|---|---|
| Focus | Request-driven services | Resources (CPU, disk, pools, queues) |
| Answers | "Is my service handling requests well?" | "Is this resource a bottleneck?" |
| Example metrics | `http_requests_total`, `http_errors_total`, `http_request_duration_seconds` | `cpu_utilization`, `thread_pool_queue_size`, `disk_io_errors` |

**Senior-level answer:**
> "I use them **together and at different layers** — RED at the service/API boundary (what SLIs are typically built from, Q9), USE for the underlying infrastructure that service depends on. A concrete example that ties them together: RED tells you `checkout` p99 latency spiked; USE on the DB connection pool tells you *why* — the pool was **saturated** (all connections in use, requests queuing for one), which is the actual root cause the RED-level symptom was pointing at."

---

### Q12. Why do teams monitor p99 (or p95) latency instead of average latency?
**Answer:** **Average latency hides the tail** — a service where 99% of requests return in 10ms and 1% take 5 seconds has an average of roughly 60ms, which looks totally fine, while 1% of your users are having a terrible, potentially timeout-triggering experience.

```
100 requests: 99 requests @ 10ms, 1 request @ 5000ms
Average = (99*10 + 5000) / 100 = 59.9ms   <- looks healthy
p99 = 5000ms                               <- reveals the actual problem
```

**Senior-level answer:**
> "At scale, tail latency compounds in a way averages completely obscure — if a single user-facing request fans out to 20 backend calls, and each has a 1% chance of hitting that slow p99 tail, the probability that **at least one** of those 20 calls is slow is roughly 1 - 0.99^20 ≈ **18%**, not 1%. This is a well-known effect (sometimes called 'the tail at scale,' from a Google/Dean & Barroso paper) and it's exactly why high-fan-out systems obsess over p99/p999 latency specifically — average latency would make a genuinely bad tail-latency problem look invisible."

---

### Q13. What does "saturation" mean, and give a concrete example of measuring it.
**Answer:** Saturation measures **how much queued/backed-up demand exists relative to capacity** — it's the leading indicator that a resource is about to become a bottleneck, **before** it starts producing outright errors. High utilization alone (e.g., 90% CPU) isn't necessarily a problem; saturation (requests actively queuing/waiting because capacity is exhausted) is where user-facing pain begins.

```java
// Thread pool saturation — queue size and active count relative to max
ThreadPoolTaskExecutor executor = ...;
int active = executor.getActiveCount();
int queueSize = executor.getThreadPoolExecutor().getQueue().size();
int maxPoolSize = executor.getMaxPoolSize();
// saturation signal: queueSize growing and active == maxPoolSize -> requests are now queuing, not just busy
```

```promql
# HikariCP connection pool saturation in Prometheus — connections waiting for a free connection
hikaricp_connections_pending{pool="orderServicePool"} > 0
```

**Senior-level answer:**
> "Saturation is the metric I'd prioritize as an **early-warning** signal ahead of latency/error alerts, because by the time latency or error-rate alerts fire, users are already affected — a growing queue depth or pending-connections count tells you the system is heading toward trouble **before** it manifests as user-visible symptoms. In practice, connection pool exhaustion (HikariCP `pending` connections, or thread pool queue depth) is one of the most common root causes I've traced a latency spike back to, and it's very directly measurable if you instrument for it deliberately, which many teams don't do by default."

---

## SECTION 5: ALERTING AND DASHBOARDS

### Q14. What are the principles of good alerting, and how do you avoid alert fatigue?
**Answer:**

1. **Alert on symptoms, not causes** — page on "checkout success rate below SLO," not on "CPU above 80%." A cause-based alert can fire without any actual user impact (CPU high but latency/errors fine), training the on-call engineer to ignore pages — which is exactly how real incidents get missed.
2. **Every page should be actionable** — if there's nothing a human can/should do right now, it shouldn't page; it should be a lower-severity notification or a dashboard signal instead.
3. **Alert on rate-of-change/duration thresholds, not single data points** — "error rate > 5% for 5 consecutive minutes," not "one 500 response just happened," to avoid noisy, self-resolving blips paging someone unnecessarily.
4. **Tie alerts to SLO burn rate where possible** — a "multi-window, multi-burn-rate" alert (fast burn over a short window AND a slower burn over a longer window, both required) balances catching real incidents fast while avoiding false pages on brief noise — this is the approach Google's SRE workbook recommends specifically to reduce alert fatigue while preserving fast detection.

**Senior-level answer:**
> "Alert fatigue is one of the most damaging things that can happen to an on-call rotation — once engineers learn that pages are frequently noise, they start being slower to react even to real incidents, which is worse than having no alerting at all in some ways. My rule of thumb: every alert should be reviewed periodically for its **precision** — if a particular alert has fired 10 times and only 1 was a real actionable incident, that alert needs to be fixed or removed, not tolerated as background noise. I'd bring up burn-rate alerting specifically as the more sophisticated answer here — it directly ties paging urgency to actual SLO risk rather than to arbitrary static thresholds."

---

### Q15. What makes a good dashboard, as opposed to a dashboard nobody actually looks at during an incident?
**Answer:**

- **Top-down structure** — start with the RED/SLI-level view (is the service healthy, from a user's perspective) at the top, drill down into USE-level resource metrics and dependency health below — so an engineer's eye naturally goes from symptom to cause.
- **Consistent time ranges and aligned panels** — all panels on the same dashboard should default to the same time window, so a spike in one panel can be visually correlated with another without manually adjusting each one.
- **Avoid "wall of graphs" dashboards** — a 40-panel dashboard with no hierarchy is unusable during a 2am incident; a good dashboard has a small number of **headline** panels (RED for the service) plus links/drill-downs into more detailed ones, not everything flattened onto one screen.
- **Include the SLO line directly on the relevant graph** — plotting the actual SLI value against the SLO threshold on the same chart makes "are we breaching" an immediate visual read rather than a mental calculation.

**Senior-level answer:**
> "The test I apply to any dashboard: could someone **unfamiliar with this service, at 2am, mid-incident** use it to get oriented in under a minute? If a dashboard requires tribal knowledge of which of 40 panels actually matters, it's failed its actual purpose. I try to design the top of every service dashboard around the same RED-based structure across all our services, specifically so that on-call engineers — who might be covering a service they don't own day-to-day — already know where to look, regardless of which service is paging."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between monitoring and alerting?" → Monitoring is the ongoing collection/visualization of system state (dashboards, metrics); alerting is the subset of monitoring that proactively notifies a human when a defined condition is met — you can have monitoring without alerting, but not the reverse.
- "Why is `rate()` important in PromQL when working with counters?" → Raw counters only ever increase (or reset to 0 on restart); `rate()` converts that into a meaningful per-second rate over a window and correctly handles counter resets, which raw counter graphs do not.
- "What's a 'burn rate' alert?" → An alert based on how fast the error budget is being consumed relative to the SLO window — a fast burn rate triggers urgent paging even early in the window, while a slow, sustained burn triggers a lower-urgency alert over a longer observation period.
- "Should you alert on every log ERROR line?" → Generally no — that couples alerting to code-level logging decisions and produces noisy, low-precision alerts; alert on aggregated symptom-level metrics (error rate over a threshold, SLO burn) instead, and use logs for investigation, not triggering.
- "What's the difference between Grafana and Kibana?" → Both are visualization layers; Grafana is primarily metrics/time-series-focused (commonly paired with Prometheus, though it supports many data sources including Elasticsearch), while Kibana is Elasticsearch-native and log/search-focused.
- "Is 100% uptime a reasonable SLO?" → No — pursuing 100% is both practically unachievable and economically irrational (Google's SRE book makes this argument explicitly) — the cost/engineering effort to go from 99.9% to 99.99% is nonlinear, and a 100% target leaves zero error budget for any planned risk-taking (deploys, experiments) at all.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) explaining the SLO-vs-SLA gap as a deliberate buffer rather than reciting definitions (Q8), (2) being able to justify **why** p99 matters over average with the tail-at-scale/fan-out math, not just "it shows outliers" (Q12), and (3) articulating symptom-based vs cause-based alerting with a real example of a bad alert you've seen or fixed (Q14) — this is one of the most commonly asked "tell me about a time" style follow-ups in SRE-adjacent senior interviews, so have a concrete story ready, not just the principle.*
