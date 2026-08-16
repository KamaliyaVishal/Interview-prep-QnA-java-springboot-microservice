# Spring MVC / REST API — Interview Prep
---

## SECTION 1: @Controller vs @RestController

### Q1. @RestController vs @Controller — what's the actual difference?
**Answer:** `@RestController` is a meta-annotation combining `@Controller` + `@ResponseBody`:

```java
@Target(ElementType.TYPE)
@Controller
@ResponseBody
public @interface RestController { }
```

| Aspect | `@Controller` | `@RestController` |
|---|---|---|
| Return value handling | Method return value is treated as a **view name** (resolved via `ViewResolver` to a JSP/Thymeleaf template) | Method return value is written **directly to the HTTP response body**, serialized via `HttpMessageConverter` (typically Jackson → JSON) |
| Needs `@ResponseBody` per method? | Yes, if you want raw data output instead of a view | No — implied on **every** handler method in the class |
| Typical use | Traditional server-rendered MVC apps (returning HTML views) | REST APIs (returning JSON/XML payloads) |

```java
@Controller
public class WebController {
    @GetMapping("/orders")
    public String orders(Model model) {
        model.addAttribute("orders", orderService.findAll());
        return "orders-view";   // resolved to /templates/orders-view.html
    }
}

@RestController
public class OrderApiController {
    @GetMapping("/api/orders")
    public List<Order> orders() {
        return orderService.findAll();   // serialized directly to JSON — no view resolution
    }
}
```

**Senior detail:** You *can* mix — use `@Controller` with `@ResponseBody` on individual methods if a class needs to serve **both** views and JSON endpoints, though in practice most teams cleanly separate REST controllers from view controllers into different classes for clarity.

---

## SECTION 2: REQUEST MAPPING ANNOTATIONS

### Q2. @RequestMapping vs @GetMapping/@PostMapping/etc. — what's the relationship?
**Answer:** `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` are all **specialized shortcuts** for `@RequestMapping(method = RequestMethod.XXX)`, introduced in Spring 4.3 for readability.

```java
// Before Spring 4.3 (still valid, more verbose)
@RequestMapping(value = "/orders/{id}", method = RequestMethod.GET)
public Order getOrder(@PathVariable Long id) { ... }

// Modern equivalent
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) { ... }
```

**Senior-level answer:**
> "I always use the HTTP-method-specific annotations (`@GetMapping`, `@PostMapping`, etc.) rather than raw `@RequestMapping` with an explicit `method` attribute — it's more concise, more readable at a glance, and the intent is immediately obvious in code review. I only reach for `@RequestMapping` at the **class level**, to define a shared base path (`@RequestMapping("/api/orders")`), with the method-specific annotations underneath for each endpoint."

```java
@RestController
@RequestMapping("/api/orders")   // class-level base path
public class OrderController {
    @GetMapping("/{id}")          // → GET /api/orders/{id}
    public Order get(@PathVariable Long id) { ... }

    @PostMapping                  // → POST /api/orders
    public Order create(@RequestBody Order order) { ... }
}
```

---

### Q3. Can two handler methods map to the same URL but different HTTP methods? What if two map to the exact same URL AND method?
**Answer:** Same URL, different HTTP methods — perfectly fine and the standard REST pattern (`GET /orders/{id}` for fetch, `PUT /orders/{id}` for update, `DELETE /orders/{id}` for delete, all on the same path). Same URL **and** same method on two different handler methods, however, causes Spring to throw `AmbiguousMappingException` at **startup** (context initialization fails fast) — Spring can't determine which handler should serve the request.

**Follow-up: "How do you disambiguate mappings that look identical on the surface?"** → Use additional mapping attributes: `params`, `headers`, `consumes`, `produces` — e.g., two `POST /orders` handlers can coexist if one specifies `consumes = "application/json"` and the other `consumes = "application/xml"`.

---

## SECTION 3: @PathVariable vs @RequestParam vs @RequestBody

### Q4. @PathVariable vs @RequestParam vs @RequestBody — differences and when to use each.
| Annotation | Source | Typical use |
|---|---|---|
| `@PathVariable` | URI **path segment** (`/orders/{id}`) | Identifying a **specific resource** — REST convention: resource identity belongs in the path |
| `@RequestParam` | Query string (`?status=SHIPPED`) or form data | **Filtering, sorting, pagination, optional parameters** — not identity |
| `@RequestBody` | HTTP **request body**, deserialized via `HttpMessageConverter` (Jackson) | The actual **payload** for `POST`/`PUT`/`PATCH` — creating or fully/partially updating a resource |

```java
@GetMapping("/orders/{id}")                                   // PathVariable — identity
public Order getOrder(@PathVariable Long id) { ... }

@GetMapping("/orders")                                        // RequestParam — filtering
public List<Order> searchOrders(
        @RequestParam(required = false) String status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) { ... }

@PostMapping("/orders")                                        // RequestBody — payload
public Order createOrder(@RequestBody @Valid OrderRequest request) { ... }
```

**Senior REST design principle worth stating proactively:**
> "I follow the convention that **identity goes in the path, filters/pagination go in query params, and the payload goes in the body.** Mixing these up — e.g., putting a resource ID in a query param instead of the path — works technically but violates REST conventions and makes the API less discoverable/cacheable, since URL-based caching and routing rely on resource identity being part of the URL itself."

---

### Q5. Trap: What happens if a @RequestParam is missing and you haven't set required=false or a default?
**Answer:** Spring throws `MissingServletRequestParameterException`, resulting in a **400 Bad Request** by default (assuming no custom exception handling overrides it). This is a deliberate fail-fast behavior — treating "required parameter missing" as a **client error**, not something to silently default or NPE on deep inside business logic. Best practice: always be explicit — either mark genuinely optional parameters with `required = false` (and handle the `null`/`Optional` case), or provide a sensible `defaultValue`, rather than leaving the required-ness ambiguous.

---

### Q6. Can @RequestBody be used with GET requests? Why or why not?
**Answer:** Technically the Spring/servlet stack doesn't hard-block it, but it's **strongly discouraged and non-standard** — per HTTP semantics, `GET` requests are not supposed to carry a meaningful body, and many intermediaries (proxies, caches, load balancers, some HTTP client libraries) **strip or ignore GET request bodies entirely**, making behavior unreliable across environments. For "GET with complex filter criteria" scenarios, the correct REST approach is either query parameters (for simple filters) or a `POST` to a dedicated search endpoint (e.g., `POST /orders/search`) if the filter criteria is too complex/structured for query params — never rely on a GET body reaching the server intact.

---

## SECTION 4: @ResponseBody, @ResponseStatus, ResponseEntity

### Q7. @ResponseBody vs ResponseEntity — when do you use which?
| Aspect | `@ResponseBody` (or implied via `@RestController`) | `ResponseEntity<T>` |
|---|---|---|
| Control over status code | Fixed (200 OK by default, or set once via `@ResponseStatus` on the method) | **Full control** — set any status per response, dynamically |
| Control over headers | Not directly — needs separate mechanisms | **Full control** — add any header per response |
| Typical use | Simple endpoints always returning 200 with a body | Endpoints needing **dynamic** status codes (201 on create, 404 on not-found, 204 on delete) or custom headers (`Location`, caching headers) |

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody @Valid OrderRequest request) {
    Order created = orderService.create(request);
    URI location = URI.create("/api/orders/" + created.getId());
    return ResponseEntity.created(location).body(created);   // 201 Created + Location header
}

@GetMapping("/orders/{id}")
public ResponseEntity<Order> getOrder(@PathVariable Long id) {
    return orderService.findById(id)
            .map(ResponseEntity::ok)                          // 200 OK
            .orElse(ResponseEntity.notFound().build());       // 404 Not Found
}
```

**Senior-level answer:**
> "I default to `ResponseEntity` for any endpoint where the status code genuinely varies based on outcome — which, honestly, is most real endpoints once you account for not-found, validation failures, and conflict cases. `@ResponseBody`-only methods returning a fixed 200 are fine for simple, always-succeeds-or-throws endpoints where exceptions (handled centrally via `@ControllerAdvice`, Section 5) take care of the non-2xx cases instead."

---

### Q8. What does @ResponseStatus do, and how is it different from setting status via ResponseEntity?
**Answer:** `@ResponseStatus` sets a **fixed** HTTP status for a handler method (or for an exception class, Q11) declaratively via annotation, rather than programmatically per-response.

```java
@PostMapping("/orders")
@ResponseStatus(HttpStatus.CREATED)   // always 201, regardless of body content
public Order createOrder(@RequestBody @Valid OrderRequest request) {
    return orderService.create(request);   // just return the object — status is fixed by the annotation
}
```

**Key difference:** `@ResponseStatus` is a **static, one-size-fits-all** declaration — every successful invocation of that method returns the same status. `ResponseEntity` gives **per-invocation, dynamic** control (Q7). If a method's outcome always maps to the same status when it succeeds (with different outcomes handled via thrown exceptions instead, Section 5), `@ResponseStatus` is simpler and cleaner; if the status genuinely varies within the same method body based on runtime logic, use `ResponseEntity`.

---

### Q9. Trap: Does @ResponseStatus work correctly combined with returning a ResponseEntity from the same method?
**Answer: No, and this is a real gotcha.** If a method returns `ResponseEntity`, Spring uses the status **set on the `ResponseEntity` object itself** — any `@ResponseStatus` annotation on the method is **ignored**, because `ResponseEntity` already fully controls the response. Mixing the two is a common source of confusion ("I set `@ResponseStatus(CREATED)` but it's still returning 200") — pick one mechanism per method, not both.

---

## SECTION 5: EXCEPTION HANDLING — @ExceptionHandler & @ControllerAdvice

### Q10. How do you centralize exception handling across all controllers in a Spring MVC/REST application?
```java
@RestControllerAdvice   // = @ControllerAdvice + @ResponseBody, applies globally across ALL @RestController classes
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        ErrorResponse body = new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)   // thrown by @Valid failures (Section 6)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_FAILED", message));
    }

    @ExceptionHandler(Exception.class)   // catch-all safety net — never leak internal details to clients
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);   // full detail logged server-side
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}
```
**Answer:** `@ControllerAdvice` (or `@RestControllerAdvice` for REST APIs, which adds implied `@ResponseBody`) is a **global exception-handling layer** applied across every controller — `@ExceptionHandler` methods inside it intercept exceptions thrown from any handler method, matched by **exception type**, and produce a consistent error response format instead of duplicating try-catch logic in every controller.

---

### Q11. What's the difference between using @ExceptionHandler inside a single controller vs inside a @ControllerAdvice class?
| Scope | Behavior |
|---|---|
| `@ExceptionHandler` inside a specific `@Controller`/`@RestController` | Only handles exceptions thrown by **that controller's own methods** |
| `@ExceptionHandler` inside a `@ControllerAdvice`/`@RestControllerAdvice` | Handles exceptions thrown by **any** controller in the application (global) — or a scoped subset via `@ControllerAdvice(basePackages = ...)` / `assignableTypes = ...` |

**Senior-level answer:**
> "I use a global `@RestControllerAdvice` as the default, single source of truth for exception-to-HTTP-status mapping across the whole API — this guarantees a **consistent error response contract** for every endpoint, which matters a lot to API consumers. I'd only add a controller-local `@ExceptionHandler` for a genuinely controller-specific exception that doesn't belong in the shared global mapping — that's the exception, not the rule."

---

### Q12. Can you attach @ResponseStatus directly to a custom exception class instead of writing an @ExceptionHandler for it?
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: " + id);
    }
}
```
**Answer:** Yes — if an exception class is annotated with `@ResponseStatus`, Spring automatically maps any uncaught instance of it to that HTTP status, **without needing an explicit `@ExceptionHandler`**. This is simpler for basic status mapping, but you **lose control over the response body structure** — Spring generates a default error body (or none, depending on configuration) rather than your custom `ErrorResponse` DTO.

**Senior-level trade-off:**
> "I use `@ResponseStatus` directly on the exception class only for very simple internal-tooling APIs where the default error format is fine. For any real production API — especially one consumed by external clients or other teams — I use `@ExceptionHandler`/`@ControllerAdvice` instead, because I need full control over the JSON error body's shape (error codes, field-level validation details, trace IDs) for a consistent, well-documented API contract — that level of control isn't available via `@ResponseStatus` alone."

---

### Q13. How would you design a consistent error response structure for a REST API used by multiple client teams?
**Answer (strong senior/architecture talking point):**
```java
public record ErrorResponse(
        String errorCode,        // machine-readable, e.g., "ORDER_NOT_FOUND" — for programmatic client handling
        String message,          // human-readable summary
        String traceId,          // correlates to distributed tracing/logs for support investigation
        Instant timestamp,
        List<FieldError> fieldErrors   // populated only for validation failures
) { }
```
> "I standardize on a single `ErrorResponse` shape returned by **every** error path in the API, populated centrally by the global exception handler — so no matter which endpoint fails or why, consuming teams can write **one** error-parsing code path instead of handling different shapes per endpoint. Including a `traceId` (correlated with the request's tracing/correlation ID, ties back to distributed tracing concepts) is something I push for specifically because it drastically speeds up cross-team production debugging — a client team can hand us a `traceId` from an error response and we can jump straight to the relevant logs."

---

## SECTION 6: VALIDATION — @Valid, @Validated, Custom Validators

### Q14. @Valid vs @Validated — what's the actual difference?
| Aspect | `@Valid` (`jakarta.validation`) | `@Validated` (Spring's own, `org.springframework.validation.annotation`) |
|---|---|---|
| Source | Standard Bean Validation (JSR 380) annotation | Spring-specific |
| Supports validation groups | No | **Yes** — `@Validated(CreateGroup.class)` |
| Triggers method-level validation on service beans | No | **Yes** — combined with `@Validated` at the class level, enables validating individual method parameters not wrapped in a DTO |
| Typical use | Validating `@RequestBody` DTOs in controllers | Validating DTOs **with group-specific rules**, or validating plain method parameters on `@Service` beans |

```java
@PostMapping("/orders")
public Order create(@Valid @RequestBody OrderRequest request) { ... }   // standard DTO validation

@Service
@Validated   // enables method-parameter-level validation
public class OrderService {
    public void cancel(@NotNull Long orderId) { ... }   // validated even though it's not inside a DTO
}
```

---

### Q15. What happens internally when @Valid validation fails on a @RequestBody? What exception is thrown?
**Answer:** Spring throws `MethodArgumentNotValidException`, containing a `BindingResult` with the full list of field-level violations (field name, rejected value, violation message). Left unhandled, this results in Spring Boot's default `400 Bad Request` error response; in practice, you intercept it centrally via `@ExceptionHandler(MethodArgumentNotValidException.class)` (Q10) to produce a structured, field-by-field error response instead of the default generic one.

```java
public record OrderRequest(
        @NotNull(message = "customerId is required")
        Long customerId,

        @NotEmpty(message = "items cannot be empty")
        List<@Valid OrderItemRequest> items,   // nested validation — each item validated too

        @Positive(message = "total must be positive")
        BigDecimal total
) { }
```

---

### Q16. How do you use validation groups to apply different rules for create vs update operations?
```java
public interface OnCreate { }
public interface OnUpdate { }

public record OrderRequest(
        @Null(groups = OnCreate.class, message = "id must not be set on create")
        @NotNull(groups = OnUpdate.class, message = "id is required on update")
        Long id,

        @NotBlank
        String customerName
) { }

@PostMapping
public Order create(@Validated(OnCreate.class) @RequestBody OrderRequest request) { ... }

@PutMapping("/{id}")
public Order update(@Validated(OnUpdate.class) @RequestBody OrderRequest request) { ... }
```
**Answer:** Validation groups (a Bean Validation feature, activated via `@Validated` rather than plain `@Valid`) let a **single DTO** apply different constraint subsets depending on context — avoiding the need to maintain near-duplicate `CreateOrderRequest`/`UpdateOrderRequest` DTOs just to vary a couple of rules. **Senior trade-off to mention:** groups add real complexity and are easy to misuse — for genuinely different shapes (not just a couple of differing rules), separate DTOs are often actually clearer and more maintainable than heavily group-annotated shared DTOs.

---

### Q17. How do you write a custom validator (beyond the built-in @NotNull/@Size/etc.)?
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidOrderStatusValidator.class)
public @interface ValidOrderStatus {
    String message() default "invalid order status";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class ValidOrderStatusValidator implements ConstraintValidator<ValidOrderStatus, String> {
    private static final Set<String> ALLOWED = Set.of("PENDING", "SHIPPED", "DELIVERED", "CANCELLED");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || ALLOWED.contains(value);   // let @NotNull handle nullness separately
    }
}

public record OrderRequest(
        @ValidOrderStatus
        String status
) { }
```
**Answer:** Implement `ConstraintValidator<AnnotationType, FieldType>`, define a custom annotation with `@Constraint(validatedBy = ...)`, and apply it like any built-in Bean Validation annotation. Common real use cases: validating enum-like string values, cross-field consistency checks (via **class-level** constraints, e.g., "endDate must be after startDate"), or business-rule checks that need a Spring-managed bean injected into the validator (e.g., checking uniqueness against the database — achievable by making the validator itself a Spring `@Component` so it can `@Autowired` a repository).

---

## SECTION 7: CONTENT NEGOTIATION

### Q18. What is content negotiation in Spring MVC? How does Spring decide which representation to return?
**Answer:** Content negotiation is how the same endpoint can return **different representations** (JSON, XML, etc.) of a resource depending on what the client requests, primarily via the `Accept` HTTP header.

```java
@GetMapping(value = "/orders/{id}", produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
public Order getOrder(@PathVariable Long id) { ... }
```
```
Accept: application/json   → JSON response (Jackson)
Accept: application/xml    → XML response (Jackson XML module, if on classpath)
```

**Resolution order (highest to lowest priority) Spring uses by default:**
1. URL path extension (`.json`/`.xml`) — **disabled by default since Spring Boot 2.6+** due to security concerns (RFD attacks) — generally shouldn't be re-enabled.
2. `Accept` HTTP header — the standard, recommended mechanism.
3. A configured default (`spring.mvc.contentnegotiation.default-content-type`) if no `Accept` header is present or no match is found.

**Senior-level answer:**
> "In practice, nearly every modern REST API I've worked on standardizes on JSON only and doesn't bother supporting content negotiation across multiple formats — it adds real complexity (extra converters, testing every format) for a use case (XML-consuming clients) that's increasingly rare outside of certain legacy enterprise/SOAP-adjacent integrations. I'd only implement multi-format content negotiation if there's a genuine, named client requirement for it."

---

### Q19. How do you use @RequestMapping's consumes attribute, and how is it different from produces?
| Attribute | Governs |
|---|---|
| `produces` | What content type(s) the **server can return**, matched against the client's `Accept` header |
| `consumes` | What content type(s) the **server accepts as input**, matched against the client's `Content-Type` header on the request |

```java
@PostMapping(value = "/orders", consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
public Order create(@RequestBody OrderRequest request) { ... }
```
If a client sends `Content-Type: application/xml` to an endpoint declaring `consumes = "application/json"`, Spring returns **415 Unsupported Media Type** before the handler method even executes — a clean, fail-fast rejection of malformed requests.

---

## SECTION 8: CORS CONFIGURATION

### Q20. What is CORS, and why does a browser block cross-origin requests by default?
**Answer:** CORS (Cross-Origin Resource Sharing) is a **browser-enforced security mechanism** preventing a web page served from one origin (e.g., `https://app.example.com`) from making requests to a different origin (e.g., `https://api.example.com`) unless the **server explicitly allows it** via response headers. This is a browser-side protection against malicious sites silently making authenticated requests to unrelated APIs on a user's behalf (leveraging the user's existing cookies/session) — it's enforced by the browser, **not** the server; server-to-server calls or tools like `curl`/Postman are entirely unaffected by CORS.

**Senior clarification worth stating (commonly misunderstood):** CORS is **not** a security feature that protects your API from unauthorized access in general — it only controls which **browser-based JavaScript origins** are allowed to read the response. A misconfigured CORS policy doesn't expose data to non-browser clients differently; it specifically governs browser same-origin-policy exceptions.

---

### Q21. How do you configure CORS globally in a Spring Boot application?
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);   // how long the browser caches the preflight response
    }
}
```
**Alternative — per-controller/method:**
```java
@CrossOrigin(origins = "https://app.example.com")
@RestController
public class OrderController { ... }
```
**Senior-level answer on which to prefer:**
> "I configure CORS **globally**, once, via `WebMvcConfigurer` (or Spring Security's `CorsConfigurationSource` if Security is on the classpath — see Q22) rather than scattering `@CrossOrigin` across individual controllers. A per-controller approach is easy to get inconsistent across a growing API surface, and CORS policy is fundamentally an **application-wide, security-relevant concern** that belongs in one centralized, auditable place."

---

### Q22. If Spring Security is on the classpath, does the WebMvcConfigurer CORS config above still work? What's the trap here?
**Answer: Not fully, and this is a very common real-world gotcha.** When Spring Security is present, it processes/filters requests **before** they reach `DispatcherServlet`'s CORS handling — so `WebMvcConfigurer`'s `addCorsMappings` alone isn't sufficient; Spring Security needs its **own** CORS configuration wired in, or it will reject/block cross-origin preflight (`OPTIONS`) requests before MVC-level CORS rules ever get a chance to apply.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
        // ... other security config
    ;
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```
**Senior debugging talking point:** "Requests work fine in Postman but fail with a CORS error only in the browser, and only after adding Spring Security" is one of the most common real support tickets tied to this exact interaction — worth naming proactively as a scenario you've diagnosed before.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does @RequestBody work with GET requests reliably?" → No — avoid it; use query params or a POST-based search endpoint instead (Q6).
- "What status code does Spring return by default for MethodArgumentNotValidException if unhandled?" → 400 Bad Request.
- "Can @PathVariable be optional?" → Yes, via `@PathVariable(required = false)` combined with a matching optional path pattern, though it's uncommon — usually cleaner to define a separate overloaded mapping for the with/without-ID cases.
- "What's the difference between 401 and 403?" → 401 Unauthorized = not authenticated (who are you?); 403 Forbidden = authenticated but not allowed (I know who you are, but you can't do this).
- "Is @ControllerAdvice global by default, or does it need configuration to be global?" → Global by default across all `@Controller`/`@RestController` beans unless scoped via `basePackages`/`basePackageClasses`/`assignableTypes`.
- "Does @Valid trigger validation on nested objects automatically?" → Only if the nested field is itself annotated with `@Valid` (cascading validation) — otherwise nested object constraints are silently skipped.
- "What HTTP method should a DELETE endpoint use, and should it accept a body?" → `DELETE`, and per REST convention it typically **should not** require/expect a request body — identity comes from the path (`DELETE /orders/{id}`).

---

*Study tip: The ResponseEntity vs @ResponseStatus behavior when both are present (Q9), the global `@ControllerAdvice` error-contract design (Q13), and the Spring Security + CORS interaction trap (Q22) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY these interactions happen, not just WHAT the annotations do.*
