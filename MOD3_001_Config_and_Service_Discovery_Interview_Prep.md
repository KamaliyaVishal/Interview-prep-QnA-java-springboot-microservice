# Config & Service Discovery — Interview Prep
---

## SECTION 1: SPRING CLOUD CONFIG SERVER & CLIENT FUNDAMENTALS

### Q1. What problem does Spring Cloud Config solve? Why not just use per-service `application.yml` files?
**Answer:** In a microservices system with dozens of services, each shipping its own `application.yml` means configuration is **scattered, unversioned as a group, and requires a redeploy for every property change**. Spring Cloud Config **centralizes** configuration in one place — typically a Git repository — served by a **Config Server**, and pulled by every **Config Client** at startup (and optionally refreshed at runtime).

```
Git repo (config-repo)
 ├── application.yml            # shared defaults, all services
 ├── order-service.yml          # order-service, all profiles
 ├── order-service-prod.yml     # order-service, prod profile only
 └── payment-service.yml
```

**Senior detail:** This isn't just convenience — it gives you **audit trail** (every config change is a Git commit, reviewable via PR), **environment parity** (same artifact, different config per profile — true externalized config per 12-factor principles), and **no rebuild required** for a config-only change.

---

### Q2. How does a Config Client fetch its configuration, and what determines which file it gets?
**Answer:** The client identifies itself via `spring.application.name`, and the Config Server resolves a file using the pattern `{application}-{profile}.yml`, matched against a Git branch/tag via `{label}`. Historically this happened via a separate **bootstrap context** (`bootstrap.yml`); since **Spring Boot 2.4+**, the modern approach uses `spring.config.import`, folding remote config into normal `Environment` resolution — no separate context needed.

```yaml
# order-service — application.yml
spring:
  application:
    name: order-service
  config:
    import: "optional:configserver:http://localhost:8888"
  profiles:
    active: prod
```
This resolves, in order of increasing priority: `application.yml` (shared) → `order-service.yml` → `order-service-prod.yml`.

**Senior-level answer:**
> "I always use `spring.config.import` with the `optional:` prefix for local development so the app doesn't fail to start if Config Server isn't running locally. I avoid `bootstrap.yml` in new projects — it's the legacy two-context mechanism that Spring Cloud's Config Data API replaced; using it today is a sign of an outdated Spring Cloud setup."

---

### Q3. What's the property source precedence when both Config Server and local config define the same key?
**Answer:** Highest to lowest priority:
| Priority | Source |
|---|---|
| 1 (highest) | Command-line args / OS environment variables |
| 2 | `{application}-{profile}.yml` on Config Server |
| 3 | `{application}.yml` on Config Server |
| 4 | Shared `application.yml` on Config Server |
| 5 (lowest) | Local `application.yml` bundled in the client's own jar |

**Trap:** Candidates often assume the local `application.yml` wins because "it's closest to the app" — it's actually the **lowest**-priority source here. This matters operationally: an env var override (e.g., set via a Kubernetes `env:` block) always beats whatever's in Git, which is exactly what you want for last-mile overrides without touching the shared repo.

**Follow-up:** Use `/actuator/env` on a running client to inspect the resolved `PropertySource` list and confirm exactly where a value came from.

---

### Q4. How do you secure sensitive values (DB passwords, API keys) stored in the Config Server's Git repo?
**Answer:** Never commit plaintext secrets to Git, even a private repo. Two standard approaches:
1. **Config Server encryption** — `POST /encrypt` with a symmetric key (`encrypt.key`) or an asymmetric keystore returns a `{cipher}...`-prefixed value; commit that ciphertext to Git. Config Server automatically decrypts it when serving the property to clients, so the client never even sees the encrypted form.
2. **Vault backend** — swap the Git backend for HashiCorp Vault (`spring.cloud.config.server.vault.*`), so secrets never live in Git at all, and get dynamic short-lived credentials where supported.

```yaml
# committed to Git — safe, ciphertext only
payment:
  api-key: '{cipher}AQBv3E9f...'
```

**Senior detail:** Approach 1 is simpler to bootstrap but the encryption key itself becomes a secret you must manage carefully (ideally in a KMS, not in the Config Server's own config file). Approach 2 (Vault) is the stronger pattern beyond a small team, since it also gives you audit logging and rotation.

---

## SECTION 2: EXTERNALIZED CONFIGURATION & CONFIGURATION REFRESH

### Q5. What does `@RefreshScope` actually do?
**Answer:** `@RefreshScope` wraps a bean in a special scope where, instead of being a fixed singleton, the bean is **discarded and lazily re-created** the next time it's accessed after a refresh event — picking up new values from the `Environment`. Without it, a `@Value`-injected field is fixed at bean-creation time forever, and a Git config change has **zero effect** on a running instance until restart.

```java
@RefreshScope
@RestController
public class PricingController {
    @Value("${pricing.discount-rate}")
    private double discountRate;   // re-read from Environment after a refresh event
}
```

**Trap:** `@RefreshScope` only affects the **specific beans annotated** — it doesn't refresh static fields, values already copied out into local variables at startup, or `@Scheduled` tasks/listeners holding a direct reference to the old bean instance rather than going through the refresh-scope proxy.

---

### Q6. How do you trigger a refresh — manually and cluster-wide?
**Answer:**
- **Single instance, manual:** `POST /actuator/refresh` — re-evaluates that one instance's `@RefreshScope` beans immediately.
- **Cluster-wide, automatic:** `spring-cloud-bus` (backed by RabbitMQ or Kafka) links Config Server and every client instance on a shared event bus. A Git webhook hits Config Server's `/monitor` endpoint on push → Config Server publishes a `RefreshRemoteApplicationEvent` on the bus → every subscribed instance refreshes itself within seconds, no manual per-instance calls needed.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: refresh, bus-refresh
```

**Senior-level answer:**
> "I never call `/actuator/refresh` on individual instances in production — with 30+ pods behind a service, that doesn't scale operationally. `spring-cloud-bus` wired to a Git webhook is the standard pattern: one Git push, one event, fanned out cluster-wide in seconds. I make sure `/actuator/refresh` and `/bus-refresh` are locked down, not publicly exposed, since they're effectively an unauthenticated way to force config re-evaluation if left open."

---

### Q7. Are `@ConfigurationProperties` beans refreshed the same way as `@Value` fields?
**Answer:** `@ConfigurationProperties` beans **are refresh-aware by default** in modern Spring Cloud — they don't need an explicit `@RefreshScope` annotation the way a hand-rolled `@Value`-based bean does, because the properties-binding mechanism itself re-binds against the current `Environment` on a refresh event. The practical difference candidates should know: `@ConfigurationProperties` is generally the **preferred** style for groups of related settings (type-safe, validated, IDE-autocomplete-friendly) versus scattering individual `@Value("${...}")` fields across many classes.

```java
@ConfigurationProperties(prefix = "pricing")
@Component
public class PricingProperties {
    private double discountRate;
    private int maxItemsPerOrder;
    // getters/setters
}
```

---

### Q8. What kinds of configuration should NOT be marked refreshable, and why?
**Answer:** Refresh is safe for pure **policy** values — feature flags, thresholds, rate limits, log levels. It's **not safe** for values tied to infrastructure identity, where a live swap can leave a bean in an inconsistent mid-transition state:
- Datasource URL/credentials — a connection pool mid-refresh can hold a mix of old and new connections.
- Thread pool core sizing tied to OS-level thread allocation.
- Security/auth provider configuration.

**Production scenario:** A team live-refreshed a `DataSource` bean's JDBC URL via `@RefreshScope`; the pool held a mix of old and new connections during the swap window, causing intermittent `SQLException`s under load. Post-incident, that property was moved to require a rolling restart via the deployment pipeline instead of a live refresh.

**Trap:** Assuming "we use Config Server" means everything is safely hot-swappable — a senior candidate should be able to name which values are refresh-safe and which require a controlled restart.

---

## SECTION 3: EUREKA REGISTRATION & HEARTBEAT

### Q9. How does a service register itself with Eureka, and what's in the registration payload?
**Answer:** Eureka uses a **client-side registration** model — each instance's `DiscoveryClient` sends its own `InstanceInfo` (hostname, IP, port, status, health-check URL, metadata) to the Eureka Server on startup, and the server stores it in an in-memory registry.

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    lease-renewal-interval-in-seconds: 30    # heartbeat interval
    lease-expiration-duration-in-seconds: 90 # eviction timeout if heartbeats stop
```

**Senior detail:** This is fundamentally different from a server-side/proxy discovery model (e.g., Kubernetes Service + kube-proxy) — with Eureka, the *client* owns the registration and the *consuming* services own the lookup and load-balancing decision (Q13), not a central router.

---

### Q10. Explain the Eureka heartbeat mechanism and what "lease renewal" means.
**Answer:** Each registered instance sends a periodic **heartbeat** (`PUT /eureka/apps/{appName}/{instanceId}`) every `lease-renewal-interval-in-seconds` (default 30s) to prove it's still alive. If Eureka Server doesn't receive a renewal within `lease-expiration-duration-in-seconds` (default 90s), it **evicts** that instance from the registry.

**Trap:** Candidates often assume eviction is instant on failure — it isn't. Worst case, a dead instance can remain in the registry (and keep receiving some traffic from stale client caches) for up to the full expiration window.

---

### Q11. What is Eureka's self-preservation mode, and why does it exist?
**Answer:** Self-preservation is a safety mechanism: Eureka Server tracks the expected heartbeat renewal rate across **all** registered instances. If the actual rate drops below a threshold (default 85%) — typically the signature of a **network partition** between clients and server, not mass instance death — Eureka **stops evicting instances** entirely and keeps serving its last-known registry, favoring availability over strict correctness.

```yaml
eureka:
  server:
    enable-self-preservation: false   # generally only disabled in small/dev environments
```

**Senior-level answer:**
> "Self-preservation is a deliberate CAP-theorem trade-off — Eureka is an **AP** system, and self-preservation is exactly where that choice becomes visible: during a suspected partition, it would rather risk routing to some stale/dead instances than mass-evict potentially-healthy ones because the server briefly lost visibility. I've seen it trip up small (3–5 instance) staging environments during rolling deploys, where a big fraction of instances stop renewing at once and looks like a partition — that's a reasonable place to disable it, but I'd never disable it in a production-scale cluster."

---

### Q12. Peer-to-peer replication — what happens if one Eureka Server node goes down?
**Answer:** Production Eureka deployments run a **cluster of Eureka Server nodes** that replicate registry state to each other peer-to-peer. Clients are configured with multiple zone URLs and transparently fail over between them.

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-1:8761/eureka/,http://eureka-2:8761/eureka/,http://eureka-3:8761/eureka/
```

**Trap:** A single-node Eureka Server in production is a well-known anti-pattern and single point of failure — this is a common "spot the design flaw" interview question.

---

## SECTION 4: HEALTH CHECKING & CLIENT-SIDE SERVICE DISCOVERY

### Q13. What is client-side service discovery, and how does it differ from server-side discovery?
**Answer:** In **client-side discovery** (Eureka's model), the calling service pulls the registry and picks an instance itself, using a client-side load balancer. In **server-side discovery** (e.g., a reverse proxy, Kubernetes Service + kube-proxy, or a service mesh sidecar), the caller just addresses a stable logical name/VIP, and routing to a specific instance happens transparently on the server/infrastructure side.

| Aspect | Client-side (Eureka) | Server-side (K8s Service, Envoy) |
|---|---|---|
| Where routing decision is made | In the calling app's process | Infrastructure layer (kube-proxy, sidecar, LB) |
| Registry lookups per call | None on the hot path — resolved from a local cache | Handled transparently, app is unaware |
| Coupling | App needs the discovery client dependency | App just makes a normal network call |
| Resilience during registry outage | App keeps working off its last cached registry | Depends entirely on infra layer's own resilience |

---

### Q14. How does the Eureka Client cache the registry, and what's the staleness trade-off?
**Answer:** Rather than querying Eureka Server on every call, the client **pulls the full registry periodically** (default every 30s) into a local in-memory cache. When code calls a logical service name, a client-side load balancer (Spring Cloud LoadBalancer, round-robin by default) picks an instance from that cache — zero network calls to Eureka Server on the actual request hot path.

```java
@LoadBalanced
@Bean
RestTemplate restTemplate() { return new RestTemplate(); }

// "order-service" resolved against the local Eureka cache, not a live registry lookup
restTemplate.getForObject("http://order-service/api/orders/123", Order.class);
```

**Trade-off:** There's up to a ~30s window where a newly registered instance isn't yet known, or a newly dead instance is still cached as available. The upside: if Eureka Server itself is briefly unreachable, calling services keep working off their last-known-good cache instead of failing immediately.

---

### Q15. How does `eureka.client.healthcheck.enabled=true` change what "UP" means in the registry?
**Answer:** By default, Eureka's status reflects only whether the **process** is alive and heartbeating — an instance can be `UP` in the registry while its database connection is dead and every request it serves fails. Enabling Actuator-backed health checks (`eureka.client.healthcheck.enabled=true`) makes the instance's Eureka status reflect its actual `/actuator/health` result, so an instance with a failing dependency can **proactively mark itself `DOWN`** and stop receiving new traffic, rather than waiting for heartbeat expiry (up to 90s away).

```yaml
eureka:
  client:
    healthcheck:
      enabled: true
```

**Senior detail:** This closes a real gap — "registered" and "healthy" are not the same thing, and interviewers use this distinction to check whether a candidate conflates registry presence with actual instance health.

---

## SECTION 5: SERVICE DISCOVERY FAILURE HANDLING

### Q16. A downstream instance dies mid-traffic — walk through the full failure and recovery timeline.
**Answer:**
1. Instance crashes, stops sending heartbeats.
2. Up to `lease-expiration-duration-in-seconds` (default 90s) later, Eureka Server evicts it — unless self-preservation is active (Q11), in which case it may linger longer.
3. Independently, calling clients may keep routing to the dead instance until **their own** local cache refreshes (~30s cycle) — worst case, the blind spot before full recovery can exceed two minutes.
4. This gap is exactly why discovery alone isn't a resilience strategy — call-level patterns fill it:
   - **Timeout** — fail fast rather than hang on a stuck instance.
   - **Circuit breaker** (Resilience4j) — stop hammering a known-bad instance/service after a failure threshold.
   - **Retry** — ideally routed to a *different* instance on retry, not the same dead one.

**Trap:** Treating "service discovery" and "fault tolerance" as the same concern. Discovery answers *what instances exist*; it does nothing to protect against an instance that's technically alive but slow/hanging — that's the circuit breaker/timeout's job entirely.

---

### Q17. Production scenario: a downstream instance is alive but hangs on every request (e.g., DB pool exhausted). What breaks, and how do you fix it?
**Answer:** Because the instance is still heartbeating, Eureka has **no reason to evict it** — this failure mode is invisible to the registry entirely. Without a client-side timeout, every caller's request thread blocks waiting on the hung instance; under load, this **cascades** — calling services exhaust their own thread/connection pools waiting on the one bad downstream, taking otherwise-healthy services down with it.

**Fix:** Aggressive per-call **timeout** + **circuit breaker** so a hanging instance is isolated in milliseconds, not tens of seconds, plus a **bulkhead** (isolated thread/connection pool per dependency) so one bad downstream can't starve calls to unrelated services.

```java
@CircuitBreaker(name = "orderService", fallbackMethod = "fallback")
@TimeLimiter(name = "orderService")
public CompletableFuture<Order> getOrder(String id) {
    return CompletableFuture.supplyAsync(() -> restTemplate.getForObject(
        "http://order-service/api/orders/" + id, Order.class));
}
```

**Senior-level answer:**
> "I treat this as the single most important lesson about service discovery: the registry tells you an instance *exists*, not that it's *fast* or *healthy in the way that matters right now*. I always pair discovery with a timeout + circuit breaker on every outbound call, sized deliberately shorter than Eureka's eviction window — otherwise you're relying on a 90-second-plus safety net for a failure mode that can cascade in seconds."

---

### Q18. Eureka Server itself goes down — what's the actual blast radius?
**Answer:** Because discovery is client-cache-based (Q14), **existing traffic between already-running services is largely unaffected** — clients keep using their last-known-good registry snapshot. What *does* break: any instance that starts up **while** Eureka Server is unreachable can't register, so it stays undiscoverable by others until the server recovers and it successfully re-registers.

**Trap:** Assuming Eureka Server going down means the whole system goes down immediately — a well-designed system degrades gracefully (existing calls keep flowing) rather than failing hard, precisely because discovery isn't a live dependency on every request.

**Follow-up — how do you validate this before it's tested by a real outage?** Chaos engineering / game days: deliberately kill Eureka Server (or a cluster node) under load in staging and verify existing service-to-service calls continue while new registrations queue and recover once the registry returns.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does `/actuator/refresh` restart the application?" → No — it discards and lazily re-creates `@RefreshScope` beans only; the JVM process itself never restarts.
- "Does Config Server push changes to clients, or do clients pull?" → Pull by default (client fetches on startup); push-like behavior only happens via `spring-cloud-bus` reacting to a webhook event.
- "What's the default Eureka heartbeat interval and eviction timeout?" → 30s renewal interval, 90s expiration.
- "Is Eureka CP or AP?" → AP — self-preservation is the clearest evidence of this design choice.
- "Does a client-side load balancer retry a failed call automatically?" → Not by default — retry behavior must be explicitly configured (e.g., Spring Retry or Resilience4j's retry module) and should target a different instance where possible.
- "Can `@ConfigurationProperties` beans be refreshed without `@RefreshScope`?" → Yes — they re-bind against the current `Environment` on a refresh event without needing the explicit annotation.
- "What happens to config precedence if the same key exists in the shared `application.yml` and the service-specific file on Config Server?" → The service-specific file wins.
- "Why might a service be `UP` in Eureka but receiving no traffic?" → Health-check integration (Q15) may have marked it `DOWN` at the app level, or client-side caches (Q14) simply haven't picked up a recent change yet.

---

*Study tip: The refresh-scope vs full-restart distinction (Q5, Q8), self-preservation as a deliberate AP trade-off (Q11), and the "registered ≠ healthy ≠ fast" distinction (Q15–Q17) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY these systems are designed this way, not just WHAT the config flags do.*
