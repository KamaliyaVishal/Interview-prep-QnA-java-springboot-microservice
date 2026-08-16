# Spring Cloud Gateway — Interview Prep
---

## SECTION 1: GATEWAY FUNDAMENTALS & ROUTES

### Q1. What is Spring Cloud Gateway, and why do microservices architectures need an API gateway at all?
**Answer:** Spring Cloud Gateway is a **reactive, non-blocking** API gateway built on **Spring WebFlux / Project Reactor** (Netty under the hood, not the Servlet stack) that sits in front of your microservices and handles routing, cross-cutting concerns (auth, rate limiting, logging), and request/response manipulation in one place — so individual services don't each have to reimplement them.

**Why it exists / problem solved:** Without a gateway, every client (mobile app, SPA, third party) has to know about and directly call N different services, each service has to implement its own auth/rate-limiting/CORS, and there's no single choke point for cross-cutting policy changes. A gateway gives you **one entry point**, centralizing routing, security, observability, and resilience.

**Senior detail:** Being reactive matters specifically for a gateway: it sits on the hot path of **every** request in the system, so it needs to handle high concurrency with a small number of threads (event-loop model) rather than blocking a thread per in-flight request the way a traditional Servlet-based proxy would under load.

---

### Q2. What are the three core building blocks of Spring Cloud Gateway — Route, Predicate, Filter — and how do they fit together?
**Answer:**
| Component | Role |
|---|---|
| **Route** | The basic unit of routing — an ID, a destination URI, a list of predicates, and a list of filters |
| **Predicate** | A condition (Java 8 `Predicate<ServerWebExchange>`) that must match for the route to be used — e.g., path, header, method |
| **Filter** | Modifies the request before it's forwarded, and/or the response before it's returned — e.g., add a header, strip a path prefix, rewrite the body |

**How they fit together:** For every incoming request, the gateway evaluates each configured route's predicates in order; the **first route whose predicates all match** is selected, its filters run (pre-filters on the way in, post-filters on the way out), and the request is proxied to the destination URI.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service                # lb:// = resolve via load balancer/discovery
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

---

### Q3. `uri: lb://order-service` — what does the `lb://` scheme actually do?
**Answer:** `lb://` tells the gateway to resolve `order-service` through the configured **service discovery + client-side load balancer** (Spring Cloud LoadBalancer, typically backed by Eureka) rather than treating it as a literal hostname. The gateway looks up available instances of `order-service` from the discovery client's cache and picks one per the load-balancing strategy — exactly the same discovery mechanism a normal service-to-service call would use.

**Trap:** Using a plain `http://order-service` URI instead of `lb://order-service` silently breaks load balancing and discovery — the gateway will try to resolve `order-service` as a literal DNS name instead of going through the discovery client, which typically fails unless you happen to have DNS-based resolution set up separately.

---

### Q4. How do you configure routes in Java (functional/RouteLocator) vs YAML — which do experienced teams prefer and why?
**Answer:** Both are first-class options:

```java
@Bean
public RouteLocator customRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-service-route", r -> r
            .path("/api/orders/**")
            .filters(f -> f.stripPrefix(1)
                           .addRequestHeader("X-Gateway", "true"))
            .uri("lb://order-service"))
        .build();
}
```

**Senior-level answer:**
> "I default to YAML for straightforward routes — it's declarative, diffable in code review, and doesn't require a redeploy if paired with Config Server + refresh. I reach for the Java `RouteLocator` DSL when routing logic needs actual **conditional logic** that YAML can't express cleanly — e.g., picking a destination based on a custom header value computed at runtime, or building routes dynamically from an external source (a database of tenant-to-service mappings). For a large gateway with 50+ routes, YAML also tends to be easier for non-Java teams (ops/platform) to review and modify safely."

---

## SECTION 2: PREDICATES & FILTERS

### Q5. Name the built-in predicate types and give a realistic example combining a few.
**Answer:** Common built-in predicates: `Path`, `Method`, `Header`, `Query`, `Cookie`, `Host`, `After`/`Before`/`Between` (time-based), `RemoteAddr` (IP-based).

```yaml
routes:
  - id: admin-only-route
    uri: lb://admin-service
    predicates:
      - Path=/api/admin/**
      - Method=GET,POST
      - Header=X-Tenant-Id, \d+          # regex — must be a numeric tenant ID
```

**Senior detail:** Predicates on a single route are **ANDed together** — all must match for the route to apply. There's no built-in OR between predicates on one route; if you need OR-like behavior, you typically define **multiple routes** with the same destination and different predicate sets.

---

### Q6. Global filters vs Gateway filters (per-route) — what's the difference, and give an example of each?
**Answer:**
| Type | Scope | Example |
|---|---|---|
| **GlobalFilter** | Applies to **every** route automatically, no per-route config needed | Logging, metrics, a global auth check |
| **GatewayFilter** | Applied **per-route**, explicitly listed in that route's `filters:` | `StripPrefix`, `AddRequestHeader`, `RewritePath` |

```java
@Component
public class LoggingGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.currentTimeMillis();
        return chain.filter(exchange).then(Mono.fromRunnable(() ->
            log.info("{} took {}ms", exchange.getRequest().getPath(), System.currentTimeMillis() - start)));
    }
    @Override
    public int getOrder() { return -1; }   // lower = runs earlier
}
```

**Interview trap:** Assuming `GlobalFilter`s and per-route `GatewayFilter`s run in a single unified, easily-predictable order by default — in reality both implement `Ordered`, and understanding the `getOrder()` value is essential to reason about execution sequence, especially when a global auth filter must run **before** a route-specific transformation filter.

---

### Q7. What's the difference between a "pre" filter and a "post" filter, and why does it matter for something like adding a response header?
**Answer:** Every filter wraps the call to `chain.filter(exchange)`:
- Code **before** `chain.filter(exchange)` runs as the request flows **toward** the downstream service ("pre" logic) — e.g., adding a request header, validating a JWT.
- Code **after** (typically inside a `.then(...)` or `Mono.fromRunnable(...)` chained onto the result) runs as the response flows **back** to the client ("post" logic) — e.g., adding a response header, logging the final status code.

```java
public GatewayFilter customFilter() {
    return (exchange, chain) -> {
        exchange.getRequest().mutate().header("X-Request-Start", Instant.now().toString());  // PRE
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            exchange.getResponse().getHeaders().add("X-Response-Time", "..."); // POST
        }));
    };
}
```

**Trap:** Trying to mutate the **request** after calling `chain.filter()` has no effect — the request has already been forwarded downstream by that point. Request mutation must happen strictly before the `chain.filter()` call.

---

### Q8. How does `StripPrefix` differ from `RewritePath`, and when would you use each?
**Answer:**
- **`StripPrefix=N`** removes the first N path segments before forwarding — simple, positional.
- **`RewritePath`** uses a regex + replacement to transform the path arbitrarily — more powerful, needed when the transformation isn't just "chop off the front."

```yaml
filters:
  - StripPrefix=1                                    # /api/orders/123 -> /orders/123
  # or, for non-trivial rewrites:
  - RewritePath=/api/orders/(?<segment>.*), /v2/orders/${segment}
```

**When to use which:** `StripPrefix` for the common case of removing a versioning/service-name prefix; `RewritePath` when the downstream path structure genuinely differs from the public-facing path (e.g., exposing `/api/orders/**` publicly while the backend actually expects `/v2/order-service/orders/**`).

---

## SECTION 3: AUTHENTICATION / AUTHORIZATION AT THE GATEWAY

### Q9. Where should authentication live — at the gateway, or in each downstream service? What's the trade-off?
**Answer:** In practice, most production systems do **both**, for different reasons:
- **Gateway-level auth** — validates that a request has a legitimate, non-expired token/session **before** it's allowed anywhere near internal services; centralizes the logic (one place to update auth policy) and blocks unauthenticated traffic as early and cheaply as possible.
- **Service-level auth** — each service still validates the token itself (and enforces its own fine-grained authorization, e.g., "does this user own this specific order") rather than blindly trusting that "it came through the gateway, so it's safe."

**Senior-level answer:**
> "I treat gateway-level auth as **coarse-grained perimeter defense** — is this a valid, authenticated caller at all — and push **fine-grained authorization** (resource ownership, role-to-action mapping) down into each service, which has the actual domain context to make that call correctly. Relying solely on the gateway creates a **confused deputy** risk: if an internal network boundary is ever crossed by a compromised service or a misconfigured internal client, downstream services with zero auth of their own are wide open."

---

### Q10. How do you validate a JWT at the gateway using a `GatewayFilter`?
**Answer:** A custom `GatewayFilterFactory` (or a global filter) intercepts the request, extracts the `Authorization: Bearer <token>` header, validates the signature/expiry (typically via Spring Security's `ReactiveJwtDecoder`), and either lets the request continue or short-circuits with `401`.

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    private final ReactiveJwtDecoder jwtDecoder;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        String token = authHeader.substring(7);
        return jwtDecoder.decode(token)
            .flatMap(jwt -> chain.filter(exchange))
            .onErrorResume(JwtException.class, e -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }
    @Override public int getOrder() { return -100; }   // run very early, before other filters
}
```

**Senior detail:** In practice, most teams don't hand-roll this — Spring Cloud Gateway integrates directly with **Spring Security's reactive OAuth2 Resource Server** support (`spring-boot-starter-oauth2-resource-server` + WebFlux security config), which gives you JWT validation, issuer/audience checks, and role extraction declaratively instead of a manual filter.

---

### Q11. How does the gateway propagate the authenticated user's identity to downstream services after validating the token?
**Answer:** After validating the JWT, the gateway typically **re-attaches claims as headers** (or forwards the original token as-is) so downstream services don't have to re-parse the JWT themselves for basic identity info — though downstream services should still independently validate the token/signature rather than blindly trusting gateway-added headers (Q9's confused-deputy point).

```java
return jwtDecoder.decode(token).flatMap(jwt -> {
    ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
        .header("X-User-Id", jwt.getSubject())
        .header("X-User-Roles", String.join(",", jwt.getClaimAsStringList("roles")))
        .build();
    return chain.filter(exchange.mutate().request(mutatedRequest).build());
});
```

**Trap:** If downstream services trust `X-User-Id`/`X-User-Roles` headers **without also verifying they only entered the network at the gateway** (e.g., via network policy preventing direct external access to internal services), a caller who can reach a service directly could forge those headers and impersonate any user. This is exactly why network-level isolation (only the gateway is externally reachable) is a prerequisite for header-based identity propagation to be safe.

---

## SECTION 4: RATE LIMITING & THROTTLING

### Q12. How does Spring Cloud Gateway's built-in `RequestRateLimiter` filter work?
**Answer:** It uses the **token bucket algorithm**, backed by **Redis** by default (`RedisRateLimiter`), tracking request counts per a configurable key (user ID, IP, API key, etc.) via a Lua script executed atomically in Redis for correctness under concurrent requests.

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 10   # tokens added per second (sustained rate)
      redis-rate-limiter.burstCapacity: 20   # max tokens in the bucket (burst allowance)
      key-resolver: "#{@userKeyResolver}"
```
```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id"));
}
```

**Senior detail:** `replenishRate` is the **steady-state** allowed rate; `burstCapacity` allows short spikes above that rate up to the bucket size before requests start getting `429`d. Setting `burstCapacity` equal to `replenishRate` effectively disables bursting entirely — a common tuning mistake when teams don't understand the two parameters are independent.

---

### Q13. Why is rate limiting backed by Redis rather than in-memory counters, given the gateway is stateless?
**Answer:** A gateway typically runs as **multiple replicas** behind a load balancer for its own scalability/availability. If rate limiting used in-memory, per-instance counters, a client could get **N× the intended limit** simply by having requests spread across N gateway instances — each instance would independently think the client is well under its own local limit. A shared store (Redis) gives every gateway instance a **consistent, cluster-wide view** of how many tokens a given key has consumed.

**Trap:** Candidates sometimes propose "just use a local in-memory rate limiter, it's faster" — which is true for latency, but breaks the actual rate-limiting guarantee the moment the gateway scales beyond one instance, which it almost always does in production.

---

### Q14. What HTTP status code does the rate limiter return, and how do you customize the response for a rate-limited client?
**Answer:** By default, `429 Too Many Requests`. You can customize headers/body via `setDenyEmptyKey`/`setEmptyKeyStatus` config, or more commonly by writing a custom response in a filter that wraps the built-in limiter's decision, adding headers like `X-RateLimit-Remaining` and `Retry-After` so well-behaved clients know when to retry.

```yaml
spring:
  cloud:
    gateway:
      redis-rate-limiter:
        include-headers: true   # adds X-RateLimit-Remaining, X-RateLimit-Burst-Capacity, X-RateLimit-Replenish-Rate
```

**Senior-level answer:**
> "I always enable `include-headers` so clients — especially internal/partner API consumers — get explicit rate-limit visibility (`X-RateLimit-Remaining`) rather than discovering the limit only by hitting `429`. It's a small config flag with outsized developer-experience value for anyone building against the gateway."

---

### Q15. How would you design *different* rate limits for different tiers of API consumers (e.g., free vs paid)?
**Answer:** The `KeyResolver` is the extension point — instead of resolving purely by user ID, resolve a **composite key or a per-tier route**:

```java
@Bean
KeyResolver tieredKeyResolver() {
    return exchange -> {
        String tier = exchange.getRequest().getHeaders().getFirst("X-Api-Tier");
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        return Mono.just(tier + ":" + userId);
    };
}
```
Then either (a) apply different `replenishRate`/`burstCapacity` values on **separate routes** matched by an `X-Api-Tier` header predicate, or (b) look up the tier's limit dynamically inside a custom filter that calls `RedisRateLimiter` programmatically with tier-specific arguments rather than static YAML values.

**Production scenario:** A SaaS platform defines two routes with identical `Path` predicates but different `Header=X-Api-Tier` predicates, each pointing to the same backend but configured with different `RequestRateLimiter` args — free tier gets `replenishRate: 5`, paid tier gets `replenishRate: 100` — with no code duplication beyond the route definition.

---

## SECTION 5: REQUEST/RESPONSE TRANSFORMATION & AGGREGATION

### Q16. How do you modify a request or response body at the gateway (e.g., wrapping a legacy backend's response into a new shape)?
**Answer:** `ModifyRequestBody` / `ModifyResponseBody` filters let you intercept and rewrite the body reactively:

```java
@Bean
public GlobalFilter modifyResponseFilter() {
    return (exchange, chain) -> chain.filter(exchange);   // simplified — real usage below
}

// Route-level usage via the ModifyResponseBodyGatewayFilterFactory
filters:
  - name: ModifyResponseBody
    args:
      inClass: String
      outClass: String
```
```java
public GatewayFilter wrapLegacyResponse() {
    return new ModifyResponseBodyGatewayFilterFactory().apply(config -> config
        .setRewriteFunction(String.class, String.class, (exchange, body) ->
            Mono.just("{\"data\":" + body + ",\"apiVersion\":\"v2\"}")));
}
```

**Senior detail:** Because Gateway is reactive/streaming by nature, body-modifying filters have to buffer the full body into memory to transform it (you can't stream-transform a JSON restructuring in general) — this is a real cost for large payloads, and a common reason teams push heavier transformation logic down into a dedicated backend-for-frontend (BFF) service instead of the gateway itself.

---

### Q17. Can Spring Cloud Gateway aggregate responses from multiple downstream services into a single response? How would you approach it?
**Answer:** Not out of the box as a declarative filter — Gateway's routing model is fundamentally **one request in, one route, one response out**. For true fan-out/aggregation (e.g., a single `/api/dashboard` call that needs data from `order-service`, `user-service`, and `inventory-service` combined), the standard patterns are:
1. **Custom `GlobalFilter`/handler** that uses a reactive `WebClient` to call multiple downstream services in parallel (`Mono.zip`) and combines results before writing the response — technically possible but pushes real business/orchestration logic into the gateway, which most teams try to avoid.
2. **Dedicated aggregation/BFF service** behind the gateway — the gateway just routes `/api/dashboard` to this service, which owns the fan-out logic. This is the far more common and maintainable pattern.

```java
Mono<DashboardResponse> aggregate(String userId) {
    Mono<Orders> orders = webClient.get().uri("http://order-service/orders?user=" + userId)
        .retrieve().bodyToMono(Orders.class);
    Mono<UserProfile> profile = webClient.get().uri("http://user-service/users/" + userId)
        .retrieve().bodyToMono(UserProfile.class);
    return Mono.zip(orders, profile, DashboardResponse::new);
}
```

**Senior-level answer:**
> "I keep the gateway focused on routing and cross-cutting concerns — auth, rate limiting, logging — and push actual response aggregation into a dedicated BFF layer. Gateway-level aggregation is possible but blurs the gateway's responsibility, makes it harder to test the aggregation logic in isolation, and couples gateway deploys to business-logic changes that have nothing to do with routing."

---

### Q18. How do you add or remove headers, and rewrite query parameters, without writing custom filter code?
**Answer:** Gateway ships built-in filters for exactly this — no custom code needed for the common cases:

```yaml
filters:
  - AddRequestHeader=X-Gateway-Source, spring-cloud-gateway
  - RemoveRequestHeader=Cookie                 # e.g., strip cookies before forwarding to internal services
  - AddResponseHeader=X-Response-Source, gateway
  - SetRequestHeader=X-Trace-Id, ${traceId}    # overwrite rather than append
```

**Interview trap:** Confusing `AddRequestHeader` (appends, keeping any existing value with the same name) with `SetRequestHeader` (overwrites) — using `Add` when you actually need `Set` can result in duplicate headers reaching the backend, which some HTTP clients handle unpredictably.

---

## SECTION 6: GATEWAY ERROR HANDLING

### Q19. What happens by default when a downstream service is unreachable or returns a 5xx — how does Gateway surface that to the client?
**Answer:** By default, Gateway's `GlobalFilter` chain includes error-handling that maps connection failures (service unreachable, timeout) to a `503 Service Unavailable`, and simply **proxies through** whatever status code the downstream service itself returned for application-level errors (a downstream `404` stays a `404` at the gateway, for example) — Gateway doesn't rewrite 4xx/5xx from the backend unless you explicitly configure it to.

**Senior detail:** The default error response body is Spring Boot's generic `ErrorWebFluxAutoConfiguration`-driven JSON — often not what you want to expose externally (can leak stack traces/internal details in non-prod profiles). Production gateways almost always define a **custom global error handler**.

---

### Q20. How do you implement a custom, unified error response format at the gateway (e.g., wrapping every error as `{ "error": { "code": ..., "message": ... } }`)?
**Answer:** Implement `ErrorWebExceptionHandler` (WebFlux's reactive equivalent of a `@ControllerAdvice`) and register it with a higher precedence than the default handler:

```java
@Component
@Order(-2)   // run before Spring Boot's default error handler
public class GlobalErrorHandler implements ErrorWebExceptionHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        HttpStatus status = (ex instanceof ResponseStatusException rse)
            ? HttpStatus.valueOf(rse.getStatusCode().value())
            : HttpStatus.INTERNAL_SERVER_ERROR;

        exchange.getResponse().setStatusCode(status);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = """
            {"error":{"code":"%s","message":"%s"}}
            """.formatted(status.value(), ex.getMessage());

        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body.getBytes());
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

**Trap:** Forgetting `@Order(-2)` (or otherwise ensuring correct precedence) means Spring Boot's default `DefaultErrorWebExceptionHandler` handles the exception first, and the custom handler never actually runs — a subtle bug that "works in a toy test" but silently does nothing once other error-handling auto-configuration is present.

---

### Q21. A downstream service is slow/hanging rather than fully down — how do you prevent that from taking down the gateway itself?
**Answer:** This is the same fundamental resilience problem as any service-to-service call (see the Config & Service Discovery module's failure-handling section) — a gateway that proxies to a hung backend without protection will pile up in-flight requests and threads/connections waiting on it, risking resource exhaustion for **every other route** sharing the gateway process. The fix is the same layered approach:
- **Per-route timeout** — `spring.cloud.gateway.httpclient.connect-timeout` / `response-timeout`, or per-route `metadata.response-timeout`.
- **Circuit breaker filter** — the `CircuitBreaker` GatewayFilterFactory (Resilience4j-backed) wraps a route, opening after a failure threshold and returning a fast fallback instead of continuing to hammer a bad backend.

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: orderServiceCB
      fallbackUri: forward:/fallback/orders
```
```java
@GetMapping("/fallback/orders")
public Mono<ResponseEntity<Map<String, String>>> fallback() {
    return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
        .body(Map.of("message", "Order service is temporarily unavailable")));
}
```

**Senior-level answer:**
> "Because the gateway is a single choke point for traffic to potentially dozens of services, I treat per-route timeouts and circuit breakers as non-negotiable, not optional hardening — a hung downstream without gateway-level protection is a single point of failure for the *entire* system, not just the one bad service. I size route timeouts deliberately shorter than any upstream client's own timeout, so the gateway fails fast and returns a clear error before the calling client's own timeout fires blindly."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is Spring Cloud Gateway built on Spring MVC or WebFlux?" → WebFlux — reactive, non-blocking, Netty-based by default.
- "Can you run Spring Cloud Gateway on the Servlet stack?" → There's a separate Servlet-based variant (`spring-cloud-gateway-server-webmvc`), but the original/primary implementation is reactive; mixing reactive gateway code with blocking calls anywhere in a filter defeats the point.
- "What does `lb://` resolve against?" → The configured service discovery client (e.g., Eureka) via Spring Cloud LoadBalancer.
- "Do predicates on one route support OR logic?" → Not natively — predicates on a route are ANDed; OR-like behavior needs multiple routes.
- "What backs the default rate limiter?" → Redis, using a token-bucket algorithm via an atomic Lua script.
- "What status code does the built-in rate limiter return when the limit is exceeded?" → 429 Too Many Requests.
- "Does Gateway rewrite a downstream 404 into something else by default?" → No — application-level status codes from the backend are proxied through as-is unless you explicitly intercept them.
- "Where should fine-grained authorization (e.g., resource ownership) live — gateway or service?" → The service — the gateway is for coarse-grained perimeter auth, not domain-aware authorization decisions.

---

*Study tip: The `lb://` vs `http://` distinction (Q3), the difference between coarse-grained gateway auth and fine-grained service auth (Q9), and treating per-route timeouts/circuit breakers as mandatory rather than optional (Q21) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY the gateway should stay a thin, resilient routing layer rather than accumulating business logic.*
