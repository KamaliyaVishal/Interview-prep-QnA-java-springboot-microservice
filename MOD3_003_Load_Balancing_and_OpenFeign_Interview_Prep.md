# Load Balancing & OpenFeign — Interview Prep
---

## SECTION 1: SPRING CLOUD LOADBALANCER & CLIENT-SIDE LOAD BALANCING

### Q1. What is client-side load balancing, and why did Spring Cloud move from Ribbon to Spring Cloud LoadBalancer?
**Answer:** Client-side load balancing means the **calling service itself** decides which instance of a downstream service to call, using a locally-cached list of available instances (from a discovery client like Eureka) rather than routing through a central load balancer/proxy. Spring Cloud originally used **Netflix Ribbon** for this; Ribbon entered maintenance mode as part of Netflix's broader OSS wind-down, so Spring Cloud replaced it with its own **Spring Cloud LoadBalancer**, which is reactive-friendly (works cleanly with WebClient, not just the blocking `RestTemplate`) and has a simpler, pluggable strategy model.

**Senior detail:** This migration is a common "do you keep up with the ecosystem" interview signal — candidates who still describe Ribbon as the current default reveal outdated knowledge, since Ribbon has been the default since Spring Cloud 2020.0 (Ilford) release train.

---

### Q2. How does Spring Cloud LoadBalancer choose which instance to call by default, and what strategies are available?
**Answer:** The default strategy is **Round Robin** (`RoundRobinLoadBalancer`) — instances are cycled through in order, spreading load evenly assuming instances have roughly equal capacity. Spring Cloud LoadBalancer also ships a `RandomLoadBalancer`, and supports writing a fully **custom `ReactorServiceInstanceLoadBalancer`** for more advanced strategies (weighted, zone-aware, least-connections).

```java
@Configuration
public class LoadBalancerConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment, LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

**Trap:** Assuming load balancing accounts for actual instance health/latency out of the box — the default round-robin strategy is **purely positional**, with no awareness of response time or error rate unless you layer in something like a circuit breaker or a custom weighted strategy.

---

### Q3. How does `@LoadBalanced` change the behavior of a `RestTemplate` or `WebClient` bean?
**Answer:** `@LoadBalanced` is a qualifier annotation that tells Spring Cloud to **intercept** calls made through that specific bean and resolve the hostname portion of the URL against the discovery client + load balancer, instead of treating it as a literal DNS name. Without it, `http://order-service/...` would just fail (or resolve incorrectly) since `order-service` isn't a real DNS entry — it's a **logical** service name known only to the discovery registry.

```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

// usage — "order-service" resolved via Eureka + load balancer, not literal DNS
restTemplate.getForObject("http://order-service/api/orders/{id}", Order.class, orderId);
```

**Senior detail:** Under the hood, `@LoadBalanced` registers a `LoadBalancerInterceptor` (for `RestTemplate`) or an equivalent `ExchangeFilterFunction` (for `WebClient`) that rewrites the request's target host at call time by consulting the `ReactorLoadBalancer` for that service name.

---

### Q4. Can you have both a load-balanced and a non-load-balanced `RestTemplate`/`WebClient` in the same application? Why would you need to?
**Answer:** Yes — and it's a common real-world need. A **load-balanced** client is for calling other services **registered in the same discovery registry** (internal microservices). A **plain, non-load-balanced** client is needed for calling **external third-party APIs** (payment gateways, external SaaS APIs) that aren't in your service registry and have a real, literal hostname.

```java
@Configuration
public class ClientConfig {
    @Bean
    @LoadBalanced
    public RestTemplate internalRestTemplate() { return new RestTemplate(); }

    @Bean
    public RestTemplate externalRestTemplate() { return new RestTemplate(); }  // no @LoadBalanced — plain hostname resolution
}
```

**Trap:** Accidentally injecting the `@LoadBalanced` bean for an external API call — Spring Cloud LoadBalancer will try to resolve the external hostname against the discovery registry, fail to find a matching service, and the call errors out with a confusing "no instances available" style exception rather than a normal connection error.

---

## SECTION 2: @FEIGNCLIENT & SERVICE-TO-SERVICE COMMUNICATION

### Q5. What is OpenFeign, and what problem does it solve compared to hand-writing REST calls with RestTemplate?
**Answer:** OpenFeign is a **declarative HTTP client** — instead of writing imperative code to build a request, call it, and parse the response, you define a Java **interface** annotated with HTTP mapping annotations, and Feign generates the implementation at runtime (proxy-based), wiring in load balancing, serialization, and error handling automatically.

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/api/orders/{id}")
    Order getOrder(@PathVariable("id") String id);

    @PostMapping("/api/orders")
    Order createOrder(@RequestBody OrderRequest request);
}

// usage elsewhere — just inject and call, no manual HTTP plumbing
@Service
class CheckoutService {
    private final OrderClient orderClient;
    Order order = orderClient.getOrder("123");
}
```

**Senior detail:** Feign integrates with Spring Cloud LoadBalancer **automatically** — a `@FeignClient(name = "order-service")` is load-balanced by default, no `@LoadBalanced` annotation needed, since Feign clients are specifically designed for internal service-to-service calls against the discovery registry.

---

### Q6. How does `@FeignClient` resolve the target service, and what's the role of the `name`/`contextId` attributes?
**Answer:** `name` (or the older `value` attribute) is the logical service name looked up via the discovery client — same resolution mechanism as `lb://` in Spring Cloud Gateway. `contextId` disambiguates **multiple Feign clients targeting the same service name** but configured differently (e.g., different timeout settings, different fallback), since Spring needs a unique bean-context identity for each client configuration.

```java
@FeignClient(name = "order-service", contextId = "orderClientFastTimeout",
             configuration = FastTimeoutConfig.class)
public interface FastOrderClient {
    @GetMapping("/api/orders/{id}/status")
    OrderStatus getStatus(@PathVariable String id);
}
```

**Trap:** Defining two `@FeignClient` interfaces with the same `name` and no `contextId` differentiation when they need different per-client configuration — Spring will complain about bean definition conflicts, since each Feign client normally gets its own isolated Spring child context keyed by name.

---

### Q7. How do you configure a per-client custom `RequestInterceptor`, encoder, or error decoder for a Feign client?
**Answer:** Via a `configuration` class referenced on the `@FeignClient` annotation — this class is **not** a regular `@Configuration`-scanned bean (deliberately, so it doesn't leak into the main application context and affect unrelated beans); it's registered into a Feign-specific child context.

```java
public class OrderClientConfig {
    @Bean
    public RequestInterceptor authHeaderInterceptor() {
        return requestTemplate -> requestTemplate.header(
            "Authorization", "Bearer " + SecurityContextHolder.getContext()
                .getAuthentication().getCredentials());
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomFeignErrorDecoder();
    }
}

@FeignClient(name = "order-service", configuration = OrderClientConfig.class)
public interface OrderClient { /* ... */ }
```

**Interview trap:** Annotating the Feign configuration class itself with `@Configuration` and letting component scanning pick it up — this can cause it to be applied **globally** rather than scoped to just that one client, which is almost never the intent. Feign configuration classes should be plain classes referenced explicitly, kept outside the main `@ComponentScan` path (or explicitly excluded).

---

### Q8. What does Feign do automatically when a downstream service returns a non-2xx response, and how do you customize that behavior?
**Answer:** By default, Feign throws a `FeignException` (with subtypes like `FeignException.NotFound`, `FeignException.BadRequest` mapped from the actual status code) whenever the response isn't 2xx. To customize this — e.g., translate specific downstream error codes into your own domain exceptions — implement a custom `ErrorDecoder`:

```java
public class CustomFeignErrorDecoder implements ErrorDecoder {
    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new OrderNotFoundException("Order not found");
            case 409 -> new OrderConflictException("Order already processed");
            default -> defaultDecoder.decode(methodKey, response);
        };
    }
}
```

**Senior-level answer:**
> "I always write a custom `ErrorDecoder` for any Feign client calling a service whose error responses I need to react to differently — a generic `FeignException` forces every caller to inspect status codes manually, which scatters magic numbers across the codebase. A decoder lets me translate a `404` into a proper `OrderNotFoundException` once, in one place, and every caller just catches a meaningful domain exception."

---

## SECTION 3: TIMEOUTS, RETRY & FALLBACK

### Q9. How do you configure connect and read timeouts for a Feign client, and what happens if you don't?
**Answer:** By default, Feign's timeouts are relatively generous (framework defaults, historically quite long depending on the underlying HTTP client), which is dangerous in production — an un-timed-out call to a hanging downstream can hold a thread indefinitely. Always configure explicit timeouts:

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          order-service:                    # per-client config, matches @FeignClient name
            connect-timeout: 2000
            read-timeout: 5000
          default:                          # applies to all Feign clients unless overridden
            connect-timeout: 2000
            read-timeout: 5000
```

**Trap:** Assuming a global `default` timeout config automatically applies even when a specific client name is also configured — per-client config **overrides** `default` for that client entirely (it doesn't merge field-by-field), so a partially-specified per-client block can silently drop settings you thought you inherited from `default`.

---

### Q10. How do you enable retry on a Feign client, and what's the danger of naive retry configuration?
**Answer:** Feign supports a `Retryer` (a legacy, simpler mechanism) or, more commonly today, layering **Spring Retry** or **Resilience4j's retry module** on top via `spring.cloud.openfeign.circuitbreaker` integration.

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          order-service:
            retryer: com.acme.CustomRetryer
```
```java
public class CustomRetryer implements Retryer {
    private int attempt = 1;
    private final int maxAttempts = 3;
    @Override
    public void continueOrPropagate(RetryableException e) {
        if (attempt++ >= maxAttempts) throw e;
        try { Thread.sleep(200L * attempt); } catch (InterruptedException ignored) {}
    }
    @Override
    public Retryer clone() { return new CustomRetryer(); }
}
```

**Danger — the classic senior trap:** Retrying a **non-idempotent** call (e.g., `POST /api/orders` to create an order) blindly can create **duplicate side effects** — two orders instead of one — if the first attempt actually succeeded on the server but the response was lost/timed out on the way back. Retry should only be applied unconditionally to **idempotent** operations (GET, PUT with a well-defined idempotency key, DELETE); POST-style creates need either an idempotency key the server de-duplicates on, or no automatic retry at all.

---

### Q11. What's the difference between a retry and a fallback, and how do you implement a fallback for a Feign client?
**Answer:** **Retry** re-attempts the *same* call, hoping a transient failure clears up. **Fallback** is what happens when retries are exhausted (or a circuit breaker is open) — instead of propagating the failure to the caller, you return a **degraded but usable** response.

```java
@FeignClient(name = "order-service", fallback = OrderClientFallback.class)
public interface OrderClient {
    @GetMapping("/api/orders/{id}")
    Order getOrder(@PathVariable String id);
}

@Component
public class OrderClientFallback implements OrderClient {
    @Override
    public Order getOrder(String id) {
        return Order.unavailablePlaceholder(id);   // degraded response, not an exception
    }
}
```

**Senior detail:** `fallback` gives you **no visibility into why** the call failed inside the fallback method itself. Use `fallbackFactory` instead when you need the actual exception (e.g., to log it, or return a different fallback response depending on whether it was a timeout vs a 404):

```java
@FeignClient(name = "order-service", fallbackFactory = OrderClientFallbackFactory.class)
public interface OrderClient { /* ... */ }

@Component
public class OrderClientFallbackFactory implements FallbackFactory<OrderClient> {
    @Override
    public OrderClient create(Throwable cause) {
        return id -> {
            log.warn("order-service call failed for id {}: {}", id, cause.getMessage());
            return Order.unavailablePlaceholder(id);
        };
    }
}
```

**Trap:** Feign fallbacks require **circuit breaker integration to be enabled** (`spring.cloud.openfeign.circuitbreaker.enabled=true`) — without it, `fallback`/`fallbackFactory` attributes on `@FeignClient` are silently ignored, and exceptions propagate normally. This is a frequently-missed config flag in real setups.

---

### Q12. How does Feign integrate with Resilience4j circuit breakers, and what does "open circuit" mean for the caller?
**Answer:** With `spring.cloud.openfeign.circuitbreaker.enabled=true`, every Feign client call is automatically wrapped in a Resilience4j `CircuitBreaker`. The breaker tracks failure rate over a sliding window; once it exceeds a configured threshold, the circuit **opens** — subsequent calls **fail immediately** (calling the fallback, or throwing, without even attempting the network call) for a configured wait duration, before transitioning to **half-open** to test if the downstream has recovered.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      order-service:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 5
```

**Senior-level answer:**
> "The value of the open state isn't just protecting the caller — it protects the **struggling downstream service** from a continuous hammering of retries while it's already failing, giving it room to recover. I always pair circuit breakers with a meaningful fallback response rather than just letting the exception propagate — a `503`-with-explanation to the end user is a much better experience than an unhandled 500 stack trace."

---

## SECTION 4: FEIGN vs RESTTEMPLATE vs WEBCLIENT

### Q13. Compare Feign, RestTemplate, and WebClient — when would you choose each?
**Answer:**
| Aspect | RestTemplate | WebClient | Feign |
|---|---|---|---|
| Style | Imperative, blocking | Reactive/non-blocking (or blocking via `.block()`) | Declarative, interface-based |
| Status | **Maintenance mode** — Spring team recommends WebClient for new code | Modern standard for both reactive and blocking use | Built on top of an HTTP client (RestTemplate or WebClient under the hood, configurable) |
| Boilerplate | Moderate — manual request building | Moderate — fluent builder API | Minimal — just declare an interface |
| Reactive support | No — purely blocking | Yes — native `Mono`/`Flux` | Historically blocking-oriented, though newer Feign supports reactive-style clients |
| Best fit | Legacy codebases already using it | New reactive applications, or any new code in general | Many internal service-to-service calls where reducing boilerplate matters |

**Senior-level answer:**
> "For new code, I default to Feign for internal service-to-service calls — the declarative style keeps the codebase clean, especially once you have many clients to many services, and it integrates cleanly with load balancing, circuit breakers, and per-client config out of the box. For anything genuinely reactive end-to-end (e.g., a WebFlux application composing multiple non-blocking calls with `Mono.zip`), I use WebClient directly, since Feign's abstraction can get in the way of fine-grained reactive composition. I avoid RestTemplate in new code entirely — it's in maintenance mode, and there's no reason to introduce a blocking-only client when WebClient covers both blocking (`.block()`) and reactive use cases."

---

### Q14. Is Feign inherently reactive or blocking? Does using Feign inside a WebFlux application cause problems?
**Answer:** Feign is, by default, built on a **blocking** HTTP client under the hood (traditionally `RestTemplate`/`URLConnection`/`OkHttp`/`ApacheHttpClient`, pluggable). Calling a blocking Feign client from inside a **reactive** WebFlux request-handling thread (the Netty event loop) is a real problem — it **blocks an event-loop thread**, which is exactly the resource a reactive application depends on staying non-blocking to handle high concurrency.

**Correct handling in a reactive application:**
```java
// WRONG in a WebFlux app — blocks the event loop thread directly
Order order = orderClient.getOrder(id);

// Better — push the blocking call onto a dedicated bounded elastic scheduler
Mono<Order> orderMono = Mono.fromCallable(() -> orderClient.getOrder(id))
    .subscribeOn(Schedulers.boundedElastic());
```

**Senior detail:** This is exactly why Feign is most at home in a traditional Spring MVC (Servlet, thread-per-request) application. In a genuinely reactive codebase, `WebClient` is the more natural fit end-to-end, since it never introduces a blocking call into the event loop in the first place — recent Spring Cloud OpenFeign versions do offer a reactive Feign variant, but it's less mature/widely adopted than plain WebClient for that use case.

---

### Q15. Does using `@LoadBalanced WebClient` give you feature parity with Feign, or are there meaningful differences?
**Answer:** `@LoadBalanced WebClient` gives you the same underlying load-balancing/discovery resolution as Feign, but you lose Feign's **declarative interface generation** — with WebClient you're still writing the request-building code by hand (URI, headers, body serialization) for every call, just reactively instead of imperatively. Feign trades a small amount of flexibility for significantly less boilerplate across many service clients; `WebClient` trades more boilerplate for finer-grained control (streaming responses, custom backpressure handling, complex reactive composition) that Feign's abstraction doesn't expose as naturally.

```java
@Bean
@LoadBalanced
WebClient.Builder loadBalancedWebClientBuilder() { return WebClient.builder(); }

// still hand-written per call
Mono<Order> order = webClientBuilder.build()
    .get()
    .uri("http://order-service/api/orders/{id}", id)
    .retrieve()
    .bodyToMono(Order.class);
```

**Trap:** Assuming you must choose one client type application-wide — in practice, mixed usage is common: Feign for the bulk of straightforward internal calls, WebClient for the specific cases needing reactive streaming or fine-grained control, and both can coexist in the same Spring Boot application without conflict.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is Ribbon still the default load balancer in current Spring Cloud?" → No — Spring Cloud LoadBalancer replaced it; Ribbon is legacy/deprecated.
- "What's the default load-balancing strategy in Spring Cloud LoadBalancer?" → Round robin.
- "Does `@FeignClient` need `@LoadBalanced` to get load balancing?" → No — it's load-balanced automatically via the discovery client integration, unlike `RestTemplate`/`WebClient` which need the explicit qualifier.
- "What exception type does Feign throw on a non-2xx response by default?" → `FeignException` (with status-specific subtypes).
- "Does Feign fallback work without any extra config?" → No — requires `spring.cloud.openfeign.circuitbreaker.enabled=true`; otherwise `fallback`/`fallbackFactory` are ignored.
- "Should you retry a POST request automatically?" → Generally no, unless it's provably idempotent (e.g., protected by an idempotency key) — blind retry on non-idempotent operations risks duplicate side effects.
- "Is RestTemplate deprecated?" → Not formally deprecated, but in maintenance mode — Spring recommends WebClient for new code.
- "Can Feign and WebClient coexist in the same application?" → Yes — commonly used together, Feign for most internal calls, WebClient where reactive composition is genuinely needed.

---

*Study tip: The idempotency risk of blind retry (Q10), the missing-config trap for Feign fallbacks (Q11), and the blocking-Feign-in-a-reactive-app problem (Q14) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY these failure modes happen, not just WHAT annotation or config flag fixes them.*
