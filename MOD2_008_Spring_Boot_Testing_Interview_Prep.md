# Spring Boot Testing — Interview Prep
---

## SECTION 1: JUNIT 5 & MOCKITO; UNIT VS INTEGRATION TESTING

### Q1. What's the practical difference between a unit test and an integration test in a Spring Boot app, and how do you decide which one to write?
**Answer:**

| Aspect | Unit test | Integration test |
|---|---|---|
| Scope | A single class, in isolation | Multiple collaborating components, real or near-real infrastructure |
| Spring context loaded? | No | Yes (full or sliced) |
| Dependencies | Mocked (Mockito) | Real beans, real (or containerized) DB, real HTTP layer |
| Speed | Milliseconds | Hundreds of ms to seconds |
| Typical tool | JUnit 5 + Mockito | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, Testcontainers |
| What it catches | Logic bugs in one class | Wiring bugs, query correctness, HTTP contract, transaction behavior |

**Senior-level answer:**
> "I think about this as a pyramid, not a binary choice: the bulk of coverage should be fast, isolated unit tests on service-layer business logic — the actual decision-making code — with a much smaller number of integration tests verifying that the pieces are wired together correctly and that things Spring/JPA/HTTP handle for you (queries, serialization, validation, transaction boundaries) actually work as expected. A common anti-pattern I push back on in review is writing `@SpringBootTest` for logic that has no Spring dependency at all — it's needlessly slow and it also tends to hide the fact that a class has too many collaborators, which a fast unit test would surface immediately as 'this constructor takes seven mocks, maybe this class is doing too much.'"

---

### Q2. What are the key JUnit 5 building blocks you use day-to-day — @Test, @BeforeEach, @ParameterizedTest, @Nested — and what changed from JUnit 4?
**Answer:**
```java
class OrderCalculatorTest {

    private OrderCalculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new OrderCalculator();   // fresh instance per test — avoids shared mutable state
    }

    @Test
    void appliesDiscountForOrdersAboveThreshold() {
        BigDecimal total = calculator.calculateTotal(new Order(BigDecimal.valueOf(150)));
        assertThat(total).isEqualByComparingTo("135.00");   // AssertJ fluent assertions
    }

    @ParameterizedTest
    @ValueSource(ints = {0, -1, -100})
    void rejectsNonPositiveQuantities(int quantity) {
        assertThrows(IllegalArgumentException.class, () -> calculator.validateQuantity(quantity));
    }

    @Nested
    class WhenCustomerIsPremium {
        @Test
        void appliesExtraDiscount() { ... }
    }
}
```

**Key JUnit 4 → 5 changes:** package moved from `org.junit` to `org.junit.jupiter.api`; `@Before`/`@After` → `@BeforeEach`/`@AfterEach`; `@BeforeClass`/`@AfterClass` → `@BeforeAll`/`@AfterAll` (must be `static` unless `@TestInstance(PER_CLASS)`); built-in `assertThrows` replaces the old `@Test(expected = ...)`; extension model (`@ExtendWith`) replaces JUnit 4 `@RunWith`/rules.

**Senior-level answer:**
> "`@Nested` is the one JUnit 5 feature I actively push teams to adopt but that's still under-used — grouping tests by scenario (`WhenCustomerIsPremium`, `WhenInventoryIsLow`) produces test-report output that reads like living documentation of the business rules, versus a flat list of 40 loosely-related `@Test` methods where the intent has to be inferred from the method name alone. `@ParameterizedTest` is the other one I lean on heavily to eliminate copy-pasted near-duplicate test methods that differ only in input values."

---

### Q3. Mockito basics: what's the difference between mock(), when()/thenReturn(), and verify() — and what's a common Mockito mistake you watch for?
**Answer:**
```java
class OrderServiceTest {

    @Test
    void placesOrderAndChargesPayment() {
        InventoryService inventoryService = mock(InventoryService.class);
        PaymentService paymentService = mock(PaymentService.class);
        OrderService orderService = new OrderService(inventoryService, paymentService);

        when(inventoryService.hasStock(any())).thenReturn(true);      // stub a return value
        when(paymentService.charge(any())).thenReturn(PaymentResult.success());

        orderService.placeOrder(new OrderRequest("SKU-1", 2));

        verify(paymentService).charge(any());                          // verify interaction happened
        verify(paymentService, times(1)).charge(any());                 // verify exact call count
        verify(inventoryService, never()).cancelReservation(any());     // verify something did NOT happen
    }
}
```

**Common mistake — stubbing a method that's never actually called ("unnecessary stubbing"):** Mockito's strict stubbing (default since Mockito 2+, enforced via `MockitoExtension`) throws `UnnecessaryStubbingException` at the end of the test if a `when()` stub is never invoked — a deliberate signal that the test setup doesn't match what the code path actually exercises.

**Senior-level answer:**
> "The mistake I see most often from engineers newer to Mockito is over-verifying — asserting on *every* mock interaction, including incidental ones, which produces tests so brittle that any harmless refactor breaks half the test suite. My rule: verify the interactions that are the actual **point** of the test (did we charge the customer, did we send the notification), and don't verify implementation details that could reasonably change without the behavior being wrong. I also always run with `@ExtendWith(MockitoExtension.class)` rather than manually calling `MockitoAnnotations.openMocks()`, specifically to get strict-stubbing checks for free — it's caught real bugs where a code change silently stopped calling a dependency the test still assumed was being exercised."

---

## SECTION 2: @SPRINGBOOTTEST, @WEBMVCTEST, @DATAJPATEST

### Q4. What does @SpringBootTest do, and why is it the "heaviest" test annotation — when do you actually need it?
**Answer:** `@SpringBootTest` boots the **entire** Spring application context — all `@Component`, `@Service`, `@Repository`, `@Configuration` beans, the full auto-configuration — the same as running the real application, optionally with a real (or random) embedded servlet container via `webEnvironment`.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiIntegrationTest {

    @Autowired private TestRestTemplate restTemplate;

    @Test
    void placeOrder_persistsOrderEndToEnd() {
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
                "/api/orders", new OrderRequest("SKU-1", 2), OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

**Senior-level answer:**
> "I reserve `@SpringBootTest` specifically for **true end-to-end scenarios** — verifying the whole stack is wired correctly, a smoke test before a release, or a handful of critical business-flow tests that intentionally exercise controller → service → repository → database together. It's expensive: the context load alone can take seconds, and a test suite full of `@SpringBootTest` classes (especially ones each loading a slightly different context configuration, which defeats Spring's context caching between tests) is the single most common reason I've seen a CI test suite balloon from 2 minutes to 20. My default is always a narrower **test slice** (`@WebMvcTest`, `@DataJpaTest`) unless the test's actual purpose requires the full context."

---

### Q5. What is @WebMvcTest, what does it load vs exclude, and how do you handle the service-layer dependency it won't autowire?
**Answer:** `@WebMvcTest` loads only the **web layer** — controllers, `@ControllerAdvice`, filters, `HandlerMethodArgumentResolver`s, Jackson configuration — and explicitly **excludes** `@Service`, `@Repository`, and other business/persistence beans. This makes it fast and focused purely on HTTP-contract correctness (status codes, JSON shape, validation, exception mapping).

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired private MockMvc mockMvc;

    @MockBean                          // required — OrderService isn't loaded by @WebMvcTest
    private OrderService orderService;

    @Test
    void placeOrder_returns201WithLocationHeader() throws Exception {
        when(orderService.placeOrder(any())).thenReturn(new Order(1L, "SKU-1"));

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                            {"sku": "SKU-1", "quantity": 2}
                        """))
                .andExpect(status().isCreated())
                .andExpect(header().string("Location", "/api/orders/1"))
                .andExpect(jsonPath("$.sku").value("SKU-1"));
    }

    @Test
    void placeOrder_invalidPayload_returns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isBadRequest());
    }
}
```

**Senior-level answer:**
> "`@WebMvcTest` is where I put the bulk of my controller testing effort, precisely *because* it forces the service layer to be mocked — that's a feature, not a limitation. It keeps the test honestly scoped to what a controller test should verify: request mapping, validation annotations firing correctly, JSON serialization shape, status codes, and exception-to-response mapping via `@ControllerAdvice` — none of which requires a real service or database to exercise. I've seen teams try to avoid `@MockBean`-ing the service by reaching for `@SpringBootTest` 'just to make it easier,' which quietly turns every controller test into a slow, DB-dependent integration test and loses the fast-feedback benefit entirely."

---

### Q6. What is @DataJpaTest, and how does it handle test data isolation and the embedded/real database question?
**Answer:** `@DataJpaTest` loads only JPA-related beans — `@Repository` interfaces, `EntityManager`, the configured `DataSource` — and by default wraps **each test method in a transaction that's rolled back automatically at the end**, so tests never leak data into each other. By default it also swaps in an **in-memory embedded database** (H2, if on the classpath) unless explicitly configured otherwise.

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired private OrderRepository orderRepository;
    @Autowired private TestEntityManager entityManager;   // lower-level JPA test helper

    @Test
    void findByCustomerId_returnsOnlyThatCustomersOrders() {
        entityManager.persist(new Order("CUST-1", "SKU-1"));
        entityManager.persist(new Order("CUST-2", "SKU-2"));
        entityManager.flush();

        List<Order> results = orderRepository.findByCustomerId("CUST-1");

        assertThat(results).hasSize(1).extracting(Order::getSku).containsExactly("SKU-1");
    }
}
```

**Senior-level answer:**
> "The automatic rollback-per-test is genuinely convenient, but the embedded-H2-by-default part is something I actively override in most real projects — H2's SQL dialect and behavior diverge from production databases (Postgres, MySQL) in enough subtle ways (case sensitivity, specific function support, constraint error messages) that a repository test passing against H2 doesn't guarantee it'll pass against production. I use `@AutoConfigureTestDatabase(replace = Replace.NONE)` combined with a Testcontainers-backed real Postgres instance instead — slower per test, but it catches real dialect-specific query bugs that H2 would silently let through, and I've hit that gap in production before, which is why I don't compromise on it anymore."

---

## SECTION 3: @MOCK, @INJECTMOCKS, @SPY; MOCKMVC

### Q7. @Mock vs @Spy vs @InjectMocks — what does each actually do, and when do you reach for @Spy specifically?
**Answer:**

| Annotation | Behavior |
|---|---|
| `@Mock` | Fully fake object — every method returns a default (null/0/empty) unless explicitly stubbed |
| `@Spy` | Wraps a **real object** — calls go to the real implementation unless explicitly stubbed for specific methods |
| `@InjectMocks` | Creates the real class under test and injects the `@Mock`/`@Spy` fields into it (constructor injection preferred, falls back to field injection) |

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private InventoryService inventoryService;
    @Spy  private DiscountCalculator discountCalculator = new DiscountCalculator();  // real logic runs
    @InjectMocks private OrderService orderService;   // OrderService(inventoryService, discountCalculator)

    @Test
    void appliesRealDiscountLogicWhileMockingInventory() {
        when(inventoryService.hasStock(any())).thenReturn(true);
        // discountCalculator's real calculateDiscount() logic actually executes — not stubbed

        Order order = orderService.placeOrder(new OrderRequest("SKU-1", 10));

        assertThat(order.getDiscount()).isEqualByComparingTo("15.00");  // real calculation verified
    }
}
```

**Senior-level answer:**
> "I reach for `@Spy` specifically when I want to verify interactions on an object while still exercising its **real logic** — a common case is a utility/calculator class with non-trivial logic that I don't want to re-mock and re-assert piece by piece, but I still want to `verify()` it was called correctly from the class under test. It's a narrower tool than `@Mock` and I use it sparingly — reaching for `@Spy` everywhere is usually a sign the test is trying to be both a unit test and an integration test at once, and I'd rather pick one clearly. One gotcha worth knowing: stubbing a `@Spy` method requires `doReturn().when()` syntax rather than `when().thenReturn()` in some cases, because `when(spy.method())` actually invokes the real method first unless you use the `doReturn` form."

---

### Q8. What is MockMvc, and how is it different from actually starting an embedded server and hitting it over HTTP?
**Answer:** `MockMvc` simulates the Spring MVC request-handling pipeline **in-process, without an actual network call or running servlet container** — it dispatches a constructed `HttpServletRequest` directly through `DispatcherServlet` and its full chain of filters/interceptors/handler mappings, then lets you assert on the resulting `HttpServletResponse`.

```java
mockMvc.perform(get("/api/orders/1")
                .accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.id").value(1))
        .andExpect(jsonPath("$.status").value("PENDING"))
        .andDo(print());   // prints full request/response for debugging
```

**Senior-level answer:**
> "The practical benefit of `MockMvc` over `TestRestTemplate`/`WebTestClient` against a real running server is speed and precision — no actual socket, no port binding, no serialization-over-the-wire overhead, and the failure messages are much more specific (you get the exact deserialized response body in the assertion failure, not just 'connection refused' style errors if something's misconfigured). I use `MockMvc` for essentially all `@WebMvcTest`-level controller tests, and reserve `TestRestTemplate`/`WebTestClient` against a real embedded server specifically for the smaller set of true end-to-end `@SpringBootTest` scenarios where I actually want to validate the real HTTP stack, including things like actual serialization over a real connection or filter behavior tied to a real container."

---

## SECTION 4: TESTCONTAINERS, WIREMOCK & TEST PROFILES

### Q9. What problem does Testcontainers solve, and how do you wire it into a Spring Boot integration test?
**Answer:** Testcontainers spins up **real, disposable Docker containers** (Postgres, Kafka, Redis, etc.) for the duration of a test run — solving the exact H2-vs-production-dialect gap called out in Q6, by testing against the *actual* database engine used in production, without needing a shared, manually-managed test environment.

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private OrderRepository orderRepository;

    @Test
    void queryUsesPostgresSpecificJsonbFiltering() {
        // exercises a real Postgres-specific query (e.g., JSONB containment) — meaningless against H2
        ...
    }
}
```

**Senior-level answer:**
> "Testcontainers is what actually made me comfortable dropping H2 entirely from most of my repository tests — the container startup cost (a few seconds, shared across a test class via a static `@Container` field, and often across the whole suite via a singleton-container pattern to avoid repeated startup) is a completely reasonable price for tests that catch real production-dialect bugs. The other place I lean on it heavily is testing Kafka producer/consumer wiring or Redis-backed caching logic against the real thing rather than an embedded fake, since those in-memory fakes have historically diverged from real broker/cache behavior in ways that cost me debugging time in production before I switched."

---

### Q10. What is WireMock used for, and how does it differ from just mocking a RestTemplate/WebClient bean directly?
**Answer:** WireMock stands up an actual **HTTP server** (in-process or as a separate process) that returns pre-programmed responses to real HTTP calls — used to test integration with **external third-party services** (payment gateways, external APIs) without hitting the real service, while still exercising the *real* HTTP client configuration (timeouts, serialization, retry/interceptor logic).

```java
@SpringBootTest
class PaymentClientIntegrationTest {

    static WireMockServer wireMockServer = new WireMockServer(8089);

    @BeforeAll
    static void setup() {
        wireMockServer.start();
        WireMock.configureFor("localhost", 8089);
    }

    @Test
    void chargePayment_handles503WithRetry() {
        stubFor(post(urlEqualTo("/charge"))
                .inScenario("retry")
                .whenScenarioStateIs(STARTED)
                .willReturn(aResponse().withStatus(503))
                .willSetStateTo("retried"));

        stubFor(post(urlEqualTo("/charge"))
                .inScenario("retry")
                .whenScenarioStateIs("retried")
                .willReturn(aResponse().withStatus(200).withBody("""
                    {"status": "SUCCESS"}
                """)));

        PaymentResult result = paymentClient.charge(new PaymentRequest(...));

        assertThat(result.isSuccess()).isTrue();
        verify(2, postRequestedFor(urlEqualTo("/charge")));   // confirms the retry actually happened
    }
}
```

**Senior-level answer:**
> "Mocking `RestTemplate`/`WebClient` at the bean level only verifies *your* code called the client correctly — it says nothing about whether your actual HTTP configuration (connection timeout, retry policy, request/response serialization, error-handling interceptors) behaves correctly against real HTTP semantics. WireMock closes that gap: I use it specifically to test resilience behavior — retries on 5xx, circuit-breaker trip thresholds, timeout handling — which is exactly the kind of logic that a bean-level mock can't meaningfully exercise, since there's no real network round-trip for a timeout or connection-reset scenario to actually occur against."

---

### Q11. How do you use test profiles (@ActiveProfiles, application-test.yml) to isolate test configuration from production config?
**Answer:**
```java
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceIntegrationTest { ... }
```

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:tc:postgresql:16:///testdb    # Testcontainers JDBC URL
  jpa:
    hibernate:
      ddl-auto: create-drop
external:
  payment-service:
    base-url: http://localhost:8089          # points at WireMock, not the real gateway
```

**Senior-level answer:**
> "I keep a dedicated `application-test.yml` that never inherits properties I don't want tests silently depending on — pointing every external integration (payment gateway URL, email provider, feature flags) at either a WireMock stub or an explicitly disabled/no-op implementation. The failure mode I'm actively guarding against is a test suite that *happens* to work because it's inheriting a real external URL from the default `application.yml` — that's a landmine that only goes off when someone runs the tests in an environment where that URL is actually reachable, at which point CI might start making real calls to a third-party sandbox or, worse, production."

---

## SECTION 5: CONTROLLER, SERVICE, REPOSITORY & INTEGRATION TESTING

### Q12. Design the testing strategy for a typical CRUD feature — order placement — across all four layers. What does each layer's test actually verify, and what doesn't it verify?
**Answer:**

| Layer | Annotation/tool | Verifies | Explicitly does NOT verify |
|---|---|---|---|
| **Repository** | `@DataJpaTest` + Testcontainers | Query correctness, entity mapping, constraints | Business logic, HTTP layer |
| **Service** | Plain JUnit 5 + Mockito (`@ExtendWith(MockitoExtension.class)`) | Business logic, branching, calculation correctness | Real DB behavior, HTTP contract |
| **Controller** | `@WebMvcTest` + `MockMvc` | Request mapping, validation, status codes, JSON shape, exception mapping | Business logic (service is mocked), real persistence |
| **End-to-end** | `@SpringBootTest` (small number) | Full wiring across all layers actually works together | — (this is the "everything" layer, kept intentionally small) |

```java
// SERVICE layer — pure unit test, no Spring context
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private InventoryService inventoryService;
    @Mock private OrderRepository orderRepository;
    @InjectMocks private OrderService orderService;

    @Test
    void placeOrder_throwsWhenOutOfStock() {
        when(inventoryService.hasStock(any())).thenReturn(false);
        assertThrows(InsufficientStockException.class,
                () -> orderService.placeOrder(new OrderRequest("SKU-1", 5)));
        verify(orderRepository, never()).save(any());   // confirms no partial save happened
    }
}
```

**Senior-level answer:**
> "The discipline I care most about here is **not duplicating the same assertion at every layer** — each layer's tests should verify what's uniquely *its* responsibility. I don't re-verify discount-calculation business rules in the controller test (that belongs in the service unit test, mocked away at the controller level), and I don't re-verify HTTP status-code mapping in the service test (that's meaningless outside the web layer). When I review a PR and see the exact same scenario tested three times at three layers with near-identical assertions, that's a sign the test suite is going to be expensive to maintain without actually buying extra confidence — the goal is a pyramid where each layer catches a *different class* of bug, not the same bug three times."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does @WebMvcTest load @Service beans automatically?" → No — only web-layer beans; any `@Service`/`@Repository` dependency of the controller must be provided via `@MockBean`.
- "What's the difference between @Mock and @MockBean?" → `@Mock` (plain Mockito) creates a standalone mock with no Spring context involvement; `@MockBean` registers the mock **into the Spring application context**, replacing the real bean — needed specifically when the class under test is wired via Spring DI (e.g., inside a `@WebMvcTest` or `@SpringBootTest`).
- "Why does @DataJpaTest roll back after each test by default?" → To guarantee test isolation — no test's inserted data leaks into another test's assertions, without needing manual cleanup code.
- "What's the risk of using @DirtiesContext too often?" → It forces Spring to discard and rebuild the entire application context after the test, defeating Spring Test's context-caching optimization and significantly slowing down the whole suite if overused.
- "How do you test a method annotated @Async?" → Either use `Awaitility` to poll for the expected async side effect within a timeout, or make the async executor synchronous in the test profile (e.g., a `SyncTaskExecutor` test bean) so the call completes before the assertion runs.
- "Should you use real time (Instant.now()) in tests?" → No — inject a `Clock` bean and use a fixed `Clock.fixed(...)` in tests, so time-dependent logic (expiry, scheduling) is deterministic and doesn't intermittently fail near a time boundary.

---

*Study tip: Being able to clearly draw the line between `@WebMvcTest` (Q5), `@DataJpaTest` (Q6), and `@SpringBootTest` (Q4) — what each loads, what it deliberately excludes, and why using the heaviest tool everywhere is an anti-pattern — is the fastest way interviewers gauge whether a candidate has actually maintained a real test suite at scale versus only having used these annotations in isolated tutorials.*
