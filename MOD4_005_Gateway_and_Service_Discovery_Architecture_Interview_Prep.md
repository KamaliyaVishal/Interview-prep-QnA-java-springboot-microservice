# Gateway & Service Discovery Architecture — Interview Prep
---

## SECTION 1: API GATEWAY PATTERN vs REVERSE PROXY

### Q1. What is the API Gateway pattern, and how is it actually different from a plain reverse proxy? This gets asked a lot and most candidates give a fuzzy answer.
**Answer:** A **reverse proxy** sits in front of one or more backend servers and forwards incoming requests to them — its core job is routing/load-balancing traffic, typically with minimal awareness of the application's business logic (e.g., nginx forwarding `/app/*` to an app server pool).

An **API Gateway** is a **reverse proxy with application-aware, cross-cutting API concerns built in** — it's the single entry point for a microservices system's north-south traffic (external clients calling in), and it actively participates in the request lifecycle rather than just forwarding bytes.

| Aspect | Reverse Proxy | API Gateway |
|---|---|---|
| Primary job | Route/load-balance traffic to backend servers | Route + enforce cross-cutting API concerns |
| Awareness | Mostly protocol/network-level (HTTP paths, headers) | Application/API-aware (auth, rate limits, per-client contracts) |
| Authentication | Not typically its job | Commonly centralizes authn/authz (Q2) |
| Aggregation | No | Can aggregate multiple backend calls into one response (Q2) |
| Protocol translation | Rare | Common — e.g., external REST/JSON → internal gRPC |
| Examples | nginx, HAProxy (in their basic form) | Spring Cloud Gateway, Kong, Amazon API Gateway, Apigee |

**Senior-level answer:** "I think of it as a spectrum, not a hard line — nginx *can* be configured to do gateway-like things, and an API Gateway product is often built on reverse-proxy foundations. The distinguishing factor I actually care about in an interview or design discussion is **scope of responsibility**: a reverse proxy's job ends at 'get this request to the right backend'; an API Gateway's job includes 'is this caller allowed to do this, are they within their rate limit, does this response need data from three services stitched together' — it's an active participant in the API contract, not just a pipe."

---

## SECTION 2: ROUTING, AUTHENTICATION, RATE LIMITING & REQUEST AGGREGATION

### Q2. Walk through the core responsibilities an API Gateway typically owns, with how you'd actually implement each — this is the "give me the full picture" question.
**Answer:**

**1. Routing** — map incoming request paths to the correct backend service, often based on path, header, or version.
```yaml
# Spring Cloud Gateway route config
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://ORDER-SERVICE          # lb:// = resolved via service discovery (Q4)
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

**2. Authentication/Authorization** — validate the caller's identity **once, at the edge**, so downstream services don't each re-implement token validation.
```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());
        if (!jwtValidator.isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // propagate validated identity downstream via a trusted header
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-User-Id", jwtValidator.extractUserId(token)).build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }
}
```
**Senior nuance:** downstream services should still **trust but verify** — either re-validate a signed, short-lived internal token, or run within a network boundary (service mesh mTLS, MOD4_003 Q13) where spoofing the gateway's identity isn't possible; blindly trusting an `X-User-Id` header from *any* caller is a real security gap if the internal network isn't locked down.

**3. Rate Limiting** — protect backend services from being overwhelmed, and enforce per-client fairness/quotas.
```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10   # sustained requests/sec allowed
      redis-rate-limiter.burstCapacity: 20    # short burst allowed above the sustained rate
      key-resolver: "#{@userKeyResolver}"     # rate-limit per-user, not just globally
```
**Why at the gateway, not per-service:** centralizing rate limiting means one consistent policy and one place to tune it, instead of every service re-implementing (and likely mis-implementing) its own limiter — and it protects services from **abusive or buggy clients** before that traffic even reaches internal infrastructure.

**4. Request Aggregation** — combine multiple backend calls into a single response for the client, reducing round trips (especially valuable for mobile clients — this overlaps directly with the BFF pattern, MOD4_003 Q11).
```java
public Mono<OrderDetailsResponse> getOrderDetails(String orderId) {
    Mono<Order> order = orderClient.getOrder(orderId);
    Mono<PaymentStatus> payment = paymentClient.getStatus(orderId);
    Mono<ShipmentInfo> shipment = shipmentClient.getShipment(orderId);

    return Mono.zip(order, payment, shipment)
        .map(tuple -> new OrderDetailsResponse(tuple.getT1(), tuple.getT2(), tuple.getT3()));
    // client makes ONE call instead of three separate round trips
}
```

**Senior talking point tying it together:** "The theme across all four responsibilities is: handle it **once, centrally, at the edge**, instead of every service re-implementing auth, rate limiting, and aggregation logic independently and inconsistently. The trade-off I make sure to name is that the gateway becomes a **critical, shared piece of infrastructure** — it needs to be highly available and kept lightweight, because if it goes down, every client-facing request goes down with it, and if it gets bloated with business logic, it turns back into the ESB bottleneck problem SOA had (MOD4_001 Q3)."

---

### Q3. What's the risk of putting too much logic into the API Gateway? Where's the line?
**Answer:** The gateway should stay limited to **cross-cutting, generic API concerns** — auth, routing, rate limiting, basic aggregation, protocol translation. The moment it starts containing **business logic** (e.g., "if this order total exceeds $500, apply a special discount rule before forwarding") it becomes a shared bottleneck that every team must coordinate changes through — functionally recreating the **ESB problem from SOA** (MOD4_001 Q3), just rebranded.

**Concrete line I draw:** "If a change requires touching the gateway to alter *business behavior*, that's a red flag — business logic belongs inside the owning service. The gateway aggregating three services' *existing* data into one response for a client is fine (that's presentation-layer composition); the gateway deciding *what that data should be* based on business rules is not — that erodes the service's ownership of its own domain and creates exactly the kind of centralized coupling microservices are meant to avoid."

---

## SECTION 3: SERVICE REGISTRY

### Q4. What is a Service Registry, and why is it necessary at all — why can't services just call each other by a fixed IP/hostname?
**Answer:** In a dynamic microservices environment — containers restarting, auto-scaling adding/removing instances, Kubernetes rescheduling pods to different nodes — **instance IP addresses and ports are constantly changing**. A hardcoded IP/hostname would break the moment that instance was replaced. A **Service Registry** is a **central, dynamically-updated directory** where service instances **register themselves on startup** (and deregister on shutdown/failure) so other services can look up **who's currently available** for a given logical service name, at call time.

```java
// Instance registers itself with Eureka on startup
@SpringBootApplication
@EnableEurekaClient
public class OrderServiceApplication { }
```
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
  instance:
    lease-renewal-interval-in-seconds: 10   # heartbeat frequency
```

**How lookups work at runtime:** instead of `orderServiceClient.call("http://10.0.4.12:8080/orders")` (a specific instance, guaranteed to eventually go stale), the caller asks the registry "give me a healthy instance of `ORDER-SERVICE`" — and gets back a **currently valid** instance to call, chosen via a load-balancing strategy.

**Common registry implementations to know:** **Eureka** (Netflix/Spring Cloud, self-registration model), **Consul** (also does health checking + KV config store), **Kubernetes' built-in service discovery** (via `kube-proxy` + DNS, Q7) — the last of these is increasingly the default choice for teams already on Kubernetes rather than running a separate registry like Eureka.

---

## SECTION 4: CLIENT-SIDE vs SERVER-SIDE DISCOVERY

### Q5. Client-side discovery vs server-side discovery — explain both, with a concrete example of each, and the trade-off between them.
**Answer:**

**Client-side discovery** — the **calling service itself** queries the registry and picks which instance to call (applying load balancing client-side).
```
OrderService  →  queries Eureka: "who's healthy for PAYMENT-SERVICE?"
              →  gets back [10.0.1.5:8080, 10.0.1.9:8080, 10.0.2.3:8080]
              →  client-side load balancer (e.g., Spring Cloud LoadBalancer) picks one
              →  calls that instance directly
```
```java
@LoadBalanced   // this RestTemplate/WebClient resolves "PAYMENT-SERVICE" via Eureka + picks an instance client-side
@Bean
public RestTemplate restTemplate() { return new RestTemplate(); }

restTemplate.getForObject("http://PAYMENT-SERVICE/authorize", PaymentResponse.class);
```

**Server-side discovery** — the caller sends the request to a **known, stable load balancer/router**, which itself queries the registry and forwards to a chosen instance — the caller never talks to the registry directly.
```
OrderService  →  calls a fixed address: http://payment-service (a K8s Service, or an API Gateway/LB)
                     → that router/LB looks up healthy instances and forwards
```
```yaml
# Kubernetes Service — this IS server-side discovery, transparently
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
# kube-proxy handles routing to a healthy pod; the caller just hits "payment-service" by DNS name
```

| Aspect | Client-side | Server-side |
|---|---|---|
| Who queries the registry | The calling service itself | A dedicated LB/router component |
| Client complexity | Higher — client needs a discovery-aware library | Lower — client just calls a fixed hostname |
| Extra network hop | No — client calls the instance directly | Yes — request passes through the LB/router first |
| Language/framework coupling | Discovery logic tied to client's language/library (e.g., Spring Cloud LoadBalancer is Java-specific) | Discovery logic centralized, language-agnostic — any client just does a normal HTTP call |
| Common in | Netflix OSS-style stacks (Eureka + Ribbon/Spring Cloud LoadBalancer) | Kubernetes (kube-proxy + Services), most cloud load balancers, API Gateways |

**Senior-level answer:** "Kubernetes has made server-side discovery the practical default for most teams today — you get discovery essentially for free via K8s Services and DNS, with **no client-side library or language coupling** at all, which matters a lot in a polyglot environment. Client-side discovery (Eureka-style) still shows up in Java-heavy Spring Cloud shops, or when you specifically want smarter, application-aware load-balancing decisions made by the caller itself — but for a new system today, I'd lean server-side/Kubernetes-native unless there's a specific reason not to."

---

## SECTION 5: HEALTH CHECKS & DNS-BASED DISCOVERY

### Q6. How do health checks actually keep service discovery accurate — what happens if they're missing or misconfigured?
**Answer:** A service registry is only useful if it reflects **currently healthy** instances — an entry for a crashed or hung instance that never gets removed means callers keep getting routed to something that will fail. Health checks are the mechanism that keeps the registry (or the K8s Service's endpoint list) accurate in near-real-time.

**Two distinct health-check types, and conflating them is a classic mistake (echoing MOD2_002 Q22's liveness/readiness distinction, now applied at the discovery layer):**
- **Liveness** — "is this process alive, or should it be restarted?" Should reflect only the process's own internal health, not downstream dependencies.
- **Readiness** — "is this instance ready to actually receive traffic right now?" Should reflect downstream dependency health (DB connection, message broker) — an instance can be alive but **not ready** (e.g., still warming up a cache, or its DB is temporarily unreachable), and should be **pulled out of the registry/load-balancer rotation** during that window without being killed.

```java
@Component
public class DownstreamHealthIndicator implements HealthIndicator {
    public Health health() {
        return dbConnectionPool.isHealthy()
            ? Health.up().build()
            : Health.down().withDetail("reason", "DB unreachable").build();
        // this feeds BOTH Eureka's self-preservation heartbeat AND K8s readiness probes
    }
}
```

**What actually happens when health checks are missing or too lenient (a real production failure mode):** "I've seen an instance stuck in a degraded state — accepting TCP connections (so a naive check passed) but unable to reach its database — stay registered as healthy because the health check only verified 'is the port open,' not 'can this instance actually do its job.' Traffic kept routing to it, and a percentage of requests failed continuously until someone manually pulled the instance. A proper readiness check that actually exercises the critical dependency (not just a shallow ping) would have removed it from rotation automatically the moment it degraded."

---

### Q7. What is DNS-based service discovery, and how does it compare to a dedicated registry like Eureka?
**Answer:** DNS-based discovery uses **standard DNS resolution** as the discovery mechanism — a logical service name (`payment-service`) resolves to an IP (or a set of IPs), and the underlying platform keeps that DNS record updated as instances come and go. This is exactly how **Kubernetes' built-in discovery** works: every K8s `Service` gets a stable DNS name (`payment-service.namespace.svc.cluster.local`), resolved via `kube-proxy` / **CoreDNS**, transparently pointing to currently-healthy pod IPs.

| Aspect | DNS-based (e.g., Kubernetes) | Dedicated registry (e.g., Eureka) |
|---|---|---|
| Client requirement | None — any language's standard HTTP client works, since DNS resolution is universal | Needs a discovery-aware client library (or at least an HTTP call to the registry's API) |
| Update propagation | Can have **DNS caching delays** — clients/OS DNS caches may hold a stale IP briefly after an instance disappears | Near-immediate — registry pushes/serves current state directly, no DNS TTL/caching layer in between |
| Metadata richness | Limited — just name → IP resolution | Richer — can carry metadata (version, zone, custom tags) usable for smarter routing decisions |
| Operational simplicity | Very simple if already on Kubernetes — no extra component to run | Requires running and maintaining the registry itself as another piece of infrastructure |

**Senior-level answer to close with:** "DNS-based discovery's biggest practical gotcha is **caching** — a client (or its OS resolver, or a connection pool holding a long-lived connection) can keep using a stale IP for a removed instance until the TTL expires or the connection is forcibly recycled. This is exactly why properly configured **connection pool max-lifetime settings** and reasonable DNS TTLs matter in Kubernetes environments — I've seen a service keep sending traffic to a terminated pod's IP for longer than expected purely because a long-lived HTTP connection was never recycled, independent of DNS itself being correct. It's not a flaw unique to DNS-based discovery, but it's a subtlety worth naming proactively since Kubernetes has made this the dominant discovery mechanism most candidates will actually encounter day-to-day."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does an API Gateway replace the need for a Service Registry?" → No — they're complementary; the gateway typically **uses** service discovery internally to resolve `lb://ORDER-SERVICE` to an actual healthy instance, it doesn't replace that lookup mechanism.
- "Is Eureka still relevant if a team is fully on Kubernetes?" → Generally no — Kubernetes' built-in Service/DNS discovery covers the same need natively; running Eureka *in addition* to K8s discovery is usually redundant unless there's a specific legacy/hybrid reason.
- "What's Eureka's 'self-preservation mode'?" → If Eureka detects it's losing heartbeats from an unusually large fraction of registered instances at once (often signaling a network partition, not real instance failure), it stops aggressively evicting instances — trading some staleness for avoiding a mass, likely-false deregistration during a network blip.
- "Can request aggregation at the gateway become a performance problem?" → Yes — if the aggregation fans out to many slow downstream calls without timeouts/parallelization (`Mono.zip` style concurrent calls, not sequential), the gateway's response time becomes bound by the **slowest** of the aggregated calls; apply the same timeout/circuit-breaker discipline here as any other inter-service call (MOD4_004 Q7-Q8).
- "Should rate limiting happen only at the gateway, or also per-service?" → Gateway-level rate limiting protects against external/client abuse; well-run systems often *also* keep lightweight per-service limits or bulkheads (MOD4_004 Q8) as defense-in-depth against internal service-to-service overload, since not all traffic to a service necessarily passes through the edge gateway.
- "What load-balancing strategies does client-side discovery typically support?" → Round-robin (default), weighted response-time, and zone-aware routing (prefer instances in the same availability zone to reduce cross-zone latency/cost) are the common ones worth naming.

---

*Study tip: The Client-side vs Server-side discovery trade-off (Q5) and the liveness-vs-readiness distinction applied to health checks (Q6) are where interviewers most reliably separate candidates who've configured Eureka/Kubernetes once from those who've actually debugged a stale-instance-in-rotation incident in production. Knowing exactly where the API Gateway's responsibility should stop (Q3) is the other high-value senior signal — it shows you understand the ESB-bottleneck lesson microservices architecture is explicitly designed around.*
