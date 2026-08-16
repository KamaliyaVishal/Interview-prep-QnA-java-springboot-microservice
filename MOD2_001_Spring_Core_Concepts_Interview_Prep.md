# Spring Core Concepts — Interview Prep — Senior Java Developer
---

## SECTION 1: IoC & DEPENDENCY INJECTION FUNDAMENTALS

### Q1. What is Inversion of Control (IoC)? Why does Spring exist because of it?
**Answer:** IoC is a design principle where the **control of object creation and wiring is transferred from the application code to a container/framework**. Instead of a class doing `new SomeDependency()` itself, it declares what it needs, and something external (the Spring container) creates and injects it.

```java
// Without IoC — class controls its own dependency creation (tight coupling)
class OrderService {
    private PaymentGateway gateway = new StripePaymentGateway();   // hardcoded
}

// With IoC — dependency is handed to the class from outside
class OrderService {
    private final PaymentGateway gateway;
    OrderService(PaymentGateway gateway) {   // Spring provides this
        this.gateway = gateway;
    }
}
```

**Senior detail:** IoC is the broader principle; **Dependency Injection (DI)** is the specific implementation pattern Spring uses to achieve it. Other IoC mechanisms exist (Service Locator, template method callbacks, event listeners) — DI just happens to be the dominant one because it keeps dependencies explicit and testable. Spring exists to be a production-grade **IoC container**: it manages object lifecycles, wiring, configuration, and cross-cutting concerns (transactions, security, AOP) so application code stays focused on business logic instead of plumbing.

**Why this matters practically:** IoC is what makes unit testing painless — since `OrderService` receives `PaymentGateway` externally, you can inject a mock in tests without touching production wiring.

---

### Q2. What is Dependency Injection? How does it relate to IoC?
**Answer:** DI is the pattern where an object's dependencies are **provided to it** (via constructor, setter, or field) rather than the object creating or looking them up itself. It's the concrete mechanism Spring uses to implement IoC.

**The three DI types:**
| Type | How | Spring annotation |
|---|---|---|
| Constructor injection | Dependencies passed via constructor | `@Autowired` on constructor (optional if single constructor since Spring 4.3) |
| Setter injection | Dependencies set via setter methods | `@Autowired` on setter |
| Field injection | Dependencies injected directly into fields via reflection | `@Autowired` on field |

Covered in detail in Q3–Q4.

---

### Q3. Constructor vs Setter vs Field injection — compare, which do you prefer and why?
| Aspect | Constructor | Setter | Field |
|---|---|---|---|
| Immutability | Supports `final` fields — dependency set once, immutable | Fields must be mutable (non-final) | Fields must be mutable (non-final) |
| Mandatory vs optional | Best for **mandatory** dependencies — object can't exist in an invalid state | Best for **optional**/reconfigurable dependencies | No clear signal of mandatory vs optional |
| Testability | Easiest — plain `new OrderService(mockGateway)` in unit tests, no Spring/reflection needed | Requires calling setters after construction | Requires reflection or a Spring test context to inject — awkward in plain unit tests |
| Circular dependency detection | Fails fast at startup (Q5) | Tolerates circular dependencies (resolved in two passes) | Tolerates circular dependencies |
| Visibility of dependencies | All dependencies visible in one place — the constructor signature | Scattered across multiple setters | Completely hidden — you must inspect the class internals to know what it depends on |

**Senior-level answer (this is what interviewers want to hear):**
> "I default to **constructor injection** for all mandatory dependencies. It lets me make fields `final`, guarantees the object is never in a partially-constructed/invalid state, makes dependencies obvious from the constructor signature, and makes unit testing trivial since I can instantiate the class directly with mocks — no Spring container needed. I avoid field injection entirely in production code — it hides dependencies, prevents immutability, and makes classes harder to unit test in isolation. Setter injection I reserve for genuinely optional dependencies, or when reconfiguring a bean after construction (rare)."

```java
@Service
public class OrderService {
    private final PaymentGateway gateway;
    private final OrderRepository repository;

    // Since Spring 4.3, @Autowired is OPTIONAL if there's exactly one constructor
    public OrderService(PaymentGateway gateway, OrderRepository repository) {
        this.gateway = gateway;
        this.repository = repository;
    }
}
```

---

### Q4. Why does the Spring team recommend constructor injection as the default?
**Answer:** Directly from Spring's own documentation and framework design philosophy:
1. **Immutability** — `final` fields, thread-safe by construction, no risk of a dependency being reassigned mid-lifecycle.
2. **Fail-fast** — a bean genuinely cannot be constructed without its required dependencies; you get a startup-time `NoSuchBeanDefinitionException` instead of a runtime `NullPointerException` deep in production traffic.
3. **No hidden framework magic** — the class is a plain, testable POJO; you can `new` it up directly in a unit test without Spring at all.
4. **Prevents circular dependency design smells** — constructor injection **fails at startup** if there's a circular dependency (Q5), forcing you to fix a genuine design problem instead of silently working around it.

---

### Q5. Can you have circular dependencies with constructor injection? How do you resolve them?
**Answer:** **No** — constructor injection with a circular dependency (Bean A needs Bean B in its constructor, Bean B needs Bean A in its constructor) throws `BeanCurrentlyInCreationException` at startup, because Spring can't fully construct either bean first.

```java
@Service
class ServiceA {
    ServiceA(ServiceB b) { }   // needs B
}
@Service
class ServiceB {
    ServiceB(ServiceA a) { }   // needs A → circular → fails at startup
}
```

**Setter/field injection tolerates this** — Spring creates both bean instances first (uninitialized), then wires the dependencies in a second pass. This is exactly why some teams historically leaned on field injection: it "works around" circular dependencies rather than surfacing them.

**Senior talking point (strong answer):**
> "A circular dependency is almost always a **design smell** — it usually means two services are too tightly coupled and share responsibilities that should be extracted into a third component, or the interaction should go through an event/callback instead of a direct call. I treat Spring's constructor-injection failure here as a **feature, not a limitation** — it forces me to fix the actual design problem rather than paper over it. If truly unavoidable (rare, e.g., some legacy refactors), Spring does offer `@Lazy` on one of the constructor parameters to defer resolution, but I only use that as a last resort with a TODO to refactor."

```java
@Service
class ServiceA {
    ServiceA(@Lazy ServiceB b) { }   // defers B's resolution until first use — workaround, not a fix
}
```

---

## SECTION 2: THE SPRING CONTAINER — ApplicationContext vs BeanFactory

### Q6. ApplicationContext vs BeanFactory — what's the difference?
| Aspect | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Nature | Root interface — basic DI container | Sub-interface of `BeanFactory` — full-featured container |
| Bean instantiation | **Lazy** by default — beans created on first `getBean()` call | **Eager** by default for singletons — all singleton beans created at startup |
| Internationalization | Not supported | `MessageSource` support (i18n) |
| Event publishing | Not supported | `ApplicationEventPublisher` — publish/listen to application events |
| AOP integration | Manual | Auto-registers `BeanPostProcessor`/`BeanFactoryPostProcessor` beans automatically |
| Environment abstraction | No | `Environment` (profiles, property sources) |
| Typical use today | Rarely used directly — legacy/low-level, minimal memory footprint scenarios | **Standard choice for virtually all modern Spring/Spring Boot applications** |

**Senior-level answer:**
> "In practice, I've never had to reach for `BeanFactory` directly in a modern Spring Boot application — `ApplicationContext` is what Spring Boot auto-configures and what `@SpringBootApplication` bootstraps under the hood (`AnnotationConfigApplicationContext` / `AnnotationConfigServletWebServerApplicationContext`). `BeanFactory` still exists as the foundational interface `ApplicationContext` extends, and understanding it matters for explaining **eager vs lazy singleton instantiation** — a common trap question — but I wouldn't instantiate a raw `BeanFactory` in production code."

---

### Q7. Why does ApplicationContext eagerly instantiate singleton beans at startup? Isn't that wasteful?
**Answer:** It's a **deliberate fail-fast design choice** — if a bean's dependencies are misconfigured, missing, or a bean's constructor/`@PostConstruct` throws, you find out **immediately at application startup**, not later at 2 AM in production when that specific code path first executes. This trades a slightly longer startup time for much stronger reliability guarantees. You can override this per-bean with `@Lazy` if a specific bean is expensive to create and rarely used.

```java
@Component
@Lazy   // only instantiated on first actual use, not at context startup
public class ExpensiveReportGenerator { }
```

---

## SECTION 3: BEAN LIFECYCLE

### Q8. Explain the full Spring bean lifecycle, step by step.
**Answer:**
```
1. Bean definitions loaded (from @Component scan, @Bean methods, XML)
2. BeanFactoryPostProcessor runs — can modify bean DEFINITIONS before instantiation
3. Bean instantiated (constructor called → constructor injection happens here)
4. Dependencies injected (setter/field injection happens here, if used)
5. BeanPostProcessor.postProcessBeforeInitialization() — runs on every bean
6. Initialization callbacks, IN THIS ORDER:
      a. @PostConstruct method
      b. InitializingBean.afterPropertiesSet()
      c. custom init-method (if configured via @Bean(initMethod=...))
7. BeanPostProcessor.postProcessAfterInitialization() — runs on every bean
      (this is where Spring AOP proxies get created — e.g., @Transactional wrapping)
8. Bean is READY — lives in the container, available via getBean() / injection
9. On container shutdown, IN THIS ORDER:
      a. @PreDestroy method
      b. DisposableBean.destroy()
      c. custom destroy-method
```

**Senior detail worth stating proactively:** Step 7 (`postProcessAfterInitialization`) is exactly where Spring creates **AOP proxies** — this is why a bean calling its own `@Transactional` method internally (`this.someTransactionalMethod()`) **doesn't get intercepted**: the proxy wraps the bean *after* it's constructed, so self-invocation bypasses the proxy entirely. This is one of the most common Spring AOP interview traps.

---

### Q9. @PostConstruct/@PreDestroy vs InitializingBean/DisposableBean vs init-method/destroy-method — differences, and which do you use?
| Approach | Mechanism | Coupling to Spring |
|---|---|---|
| `@PostConstruct` / `@PreDestroy` | JSR-250 standard annotations | **None** — not Spring-specific, portable to any DI container (Jakarta EE, CDI) |
| `InitializingBean.afterPropertiesSet()` / `DisposableBean.destroy()` | Spring-specific interfaces | Couples your class directly to Spring's API |
| `@Bean(initMethod=..., destroyMethod=...)` | Configured externally in `@Configuration` | None — useful for **third-party classes** you can't annotate |

**Senior-level answer:**
> "I use `@PostConstruct`/`@PreDestroy` almost exclusively for my own classes — they're a JSR-250 standard, not Spring-specific, so the class stays portable and isn't coupled to Spring's `InitializingBean`/`DisposableBean` interfaces. I only reach for the `@Bean(initMethod=..., destroyMethod=...)` configuration style when wiring a **third-party class** I don't own and can't annotate directly — e.g., a legacy connection pool with a `start()`/`shutdown()` method pair."

```java
@Component
public class CacheWarmer {
    @PostConstruct
    public void warmCache() { /* runs after dependency injection completes */ }

    @PreDestroy
    public void flushCache() { /* runs during graceful shutdown */ }
}
```

---

### Q10. What's the difference between `BeanPostProcessor` and `BeanFactoryPostProcessor`?
**Answer:**
- **`BeanFactoryPostProcessor`** runs **before** any bean is instantiated — it operates on **bean definitions** (metadata: class, scope, property values), not actual bean instances. Classic example: `PropertySourcesPlaceholderConfigurer`, which resolves `${...}` placeholders in bean definitions before objects are created.
- **`BeanPostProcessor`** runs **after** each individual bean is instantiated (both before and after its init callbacks) — it operates on actual **bean instances**, and can wrap/replace them entirely. This is the extension point Spring's own AOP proxying and `@Autowired` processing (`AutowiredAnnotationBeanPostProcessor`) are built on.

**Senior detail:** Almost every "magic" annotation in Spring (`@Autowired`, `@Transactional`, `@Async`, `@Value`) is implemented as a `BeanPostProcessor` under the hood — understanding this demystifies a lot of "how does Spring actually do this" questions.

---

### Q11. Trap: If a `prototype`-scoped bean has a `@PreDestroy` method, does Spring call it?
**Answer: No.** Spring manages the **complete lifecycle** (including destruction callbacks) only for **singleton**-scoped beans. For `prototype` beans, Spring hands off the instance after creation and **does not track it further** — no destroy callback is ever invoked automatically, since the container has no way of knowing when the application is truly done with a given prototype instance. If cleanup is required for prototype beans, the application code must call the destroy logic manually (or use a custom `DestructionAwareBeanPostProcessor`, an advanced/rare technique). This is a well-known trap interviewers use to check whether you actually understand scope semantics, not just memorize the annotation list.

---

## SECTION 4: BEAN SCOPES

### Q12. What are the Spring bean scopes? Explain each.
| Scope | Lifetime | Typical use |
|---|---|---|
| `singleton` (default) | **One instance per Spring container**, shared everywhere it's injected | Stateless services, repositories, controllers — the vast majority of beans |
| `prototype` | A **new instance every time** it's requested from the container | Stateful, non-thread-safe objects (e.g., a builder-like helper with mutable state) |
| `request` (web-aware) | One instance **per HTTP request** | Request-scoped data holders in web apps |
| `session` (web-aware) | One instance **per HTTP session** | Per-user session state (e.g., shopping cart) |
| `application` (web-aware) | One instance per `ServletContext` | Effectively singleton-like, scoped to the whole web app |
| `websocket` (web-aware) | One instance per WebSocket session | Less common, WebSocket-specific state |

```java
@Component
@Scope("prototype")
public class ReportBuilder { /* mutable, stateful, not safe to share */ }
```

---

### Q13. Singleton scope — is it really "one instance per JVM"? Clarify this common misconception.
**Answer:** No — **singleton in Spring means one instance per `ApplicationContext` (per Spring container)**, not one instance per JVM/classloader like the classic GoF Singleton pattern. If you spin up **two separate `ApplicationContext`s in the same JVM** (uncommon but possible — e.g., parent/child contexts, or multiple contexts in tests), each gets its **own** singleton instance of the same class. This distinction is a frequent interview trap: "singleton" in Spring is a **container-scoped** concept, not a JVM-wide guarantee.

---

### Q14. How do you correctly inject a `prototype`-scoped bean into a `singleton`-scoped bean? (Classic trap question)
**Answer:** Naive field/constructor injection **doesn't work as expected** — because the singleton bean is created **once**, the prototype dependency is also only resolved **once** at that point and then reused forever, defeating the entire purpose of prototype scope.

```java
@Component
@Scope("singleton")
public class ReportService {
    @Autowired
    private ReportBuilder builder;   // WRONG — injected ONCE, same instance reused every call
}
```

**Correct solutions:**
1. **Scoped proxy** (`proxyMode`) — Spring injects a proxy that resolves a fresh prototype instance on **every method call**:
```java
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ReportBuilder { }
```
2. **`ObjectProvider`/`Provider<T>`** — explicitly request a new instance on demand:
```java
@Component
public class ReportService {
    @Autowired
    private ObjectProvider<ReportBuilder> builderProvider;

    public void generate() {
        ReportBuilder builder = builderProvider.getObject();   // fresh instance every call
    }
}
```
3. **`ApplicationContext.getBean()` lookup method injection** — older/more verbose approach, generally superseded by the two options above.

**Senior-level answer:**
> "I default to `ObjectProvider<T>` — it's explicit about the fact that a new instance is being requested, doesn't rely on CGLIB proxy magic, and is easy to reason about and test. Scoped proxies work too but are 'invisible' in the code, which can confuse teammates unfamiliar with the pattern."

---

### Q15. What is a scoped proxy (`proxyMode`) and why does it matter for `request`/`session` scope specifically?
**Answer:** Web-aware scopes (`request`, `session`) have the **same fundamental problem** as prototype-in-singleton (Q14): you typically want to inject a request/session-scoped bean into a **singleton** controller/service, but that singleton is created once at startup — **before any HTTP request even exists**. A scoped proxy solves this by injecting a **lightweight proxy object** into the singleton; the proxy defers to the **actual current** request/session-scoped bean on every method call, looked up fresh from the current thread's request/session context.

```java
@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart { /* per-user session state */ }

@Service   // singleton — created once at startup, long before any user session exists
public class CheckoutService {
    private final ShoppingCart cart;   // actually a proxy — resolves the REAL cart per current session at call time
    CheckoutService(ShoppingCart cart) { this.cart = cart; }
}
```

---

## SECTION 5: STEREOTYPE ANNOTATIONS

### Q16. @Component vs @Service vs @Repository vs @Controller — what's actually different between them?
**Answer:** All four are **stereotype annotations**, and `@Service`, `@Repository`, and `@Controller` are all **meta-annotated with `@Component`** — meaning component-scanning treats them identically for the purpose of bean registration. The differences are primarily **semantic** (documenting intent/layer) with **one functional exception** (`@Repository`, covered in Q17).

| Annotation | Layer | Extra behavior beyond @Component |
|---|---|---|
| `@Component` | Generic — any Spring-managed bean | None — the base annotation |
| `@Service` | Business/service layer | None functionally — purely semantic/documentation |
| `@Repository` | Persistence/DAO layer | **Exception translation** — converts persistence-specific exceptions into Spring's unchecked `DataAccessException` hierarchy (Q17) |
| `@Controller` | Web/presentation layer (MVC) | Enables request-handling behavior when combined with `@RequestMapping`/`@GetMapping` etc.; `@RestController` = `@Controller` + `@ResponseBody` |

---

### Q17. What extra behavior does @Repository actually provide beyond being a marker annotation?
**Answer:** `@Repository` enables **`PersistenceExceptionTranslationPostProcessor`** — a `BeanPostProcessor` that wraps the bean with an AOP advisor translating platform-specific persistence exceptions (JDBC's checked `SQLException`, JPA's `PersistenceException`, Hibernate's native exceptions) into Spring's **unified, unchecked `DataAccessException` hierarchy** (`DuplicateKeyException`, `DataIntegrityViolationException`, `OptimisticLockingFailureException`, etc.). This ties directly back to the checked-vs-unchecked design philosophy (Exception Handling prep, Q3): it lets your service layer catch **consistent, technology-agnostic exceptions** regardless of whether the DAO underneath uses plain JDBC, Hibernate, or JPA.

---

### Q18. Can you use @Component instead of @Service/@Repository/@Controller? What would you lose?
**Answer:** Functionally, `@Component` on a repository class would work for basic DI, but you'd **lose the automatic exception translation** described in Q17 for `@Repository`, and you'd lose the semantic clarity that helps both humans and tools (Spring's own AOP pointcuts, documentation generators, architecture-linting tools like ArchUnit) reason about layering. Using the correct stereotype is a low-cost, high-value convention — always use the most specific one that applies.

---

## SECTION 6: AUTOWIRING & AMBIGUITY RESOLUTION

### Q19. How does @Autowired resolve which bean to inject — by type or by name?
**Answer:** **By type first.** If exactly one bean of the required type exists, it's injected directly. If **multiple** beans of the same type exist, Spring then falls back to matching **by field/parameter name** against bean names — and if that still doesn't resolve unambiguously, it throws `NoUniqueBeanDefinitionException` unless disambiguated via `@Qualifier` or `@Primary` (Q20–Q21).

---

### Q20. What happens when multiple beans of the same type exist? How do @Qualifier and @Primary resolve the ambiguity?
```java
public interface PaymentGateway { }

@Component("stripeGateway")
public class StripeGateway implements PaymentGateway { }

@Component("paypalGateway")
public class PaypalGateway implements PaymentGateway { }

@Service
public class OrderService {
    // AMBIGUOUS without disambiguation — NoUniqueBeanDefinitionException at startup
    private final PaymentGateway gateway;

    public OrderService(@Qualifier("stripeGateway") PaymentGateway gateway) {   // explicit disambiguation
        this.gateway = gateway;
    }
}
```
**Answer:** Without disambiguation, Spring can't decide which `PaymentGateway` implementation to inject and fails fast at startup — a fail-fast design consistent with constructor injection's philosophy (Q4). `@Qualifier("beanName")` explicitly names which bean to use at the **injection point**. Alternatively, `@Primary` on **one** of the candidate beans marks it as the **default choice** whenever the type is ambiguous elsewhere, without requiring every injection point to specify a qualifier.

---

### Q21. @Qualifier vs @Primary — when do you use which?
| Aspect | `@Qualifier` | `@Primary` |
|---|---|---|
| Where declared | At the **injection point** (constructor param, field) | On the **bean definition** itself |
| Granularity | Per-injection-point choice — different callers can choose different implementations | One global default for **all** ambiguous injection points |
| Typical use | You genuinely need **different** implementations in different places | One implementation is the "usual"/default choice, with occasional overrides via `@Qualifier` |

**Senior-level answer:**
> "I use `@Primary` when there's a clear default implementation most of the codebase should use — e.g., a `StripeGateway` as the primary payment provider — and reach for `@Qualifier` only at the specific injection points that genuinely need the non-default implementation (e.g., a specific integration test or a fallback service). Using `@Qualifier` everywhere, for every injection point, is a smell that suggests the abstraction itself might not be well-defined."

---

### Q22. Is field injection with @Autowired considered a bad practice? Why does Spring itself discourage it?
**Answer:** Yes — widely considered an anti-pattern in modern Spring development, for the reasons already summarized in Q3:
1. **Prevents `final` fields** — the class becomes mutable when it conceptually shouldn't be.
2. **Hides dependencies** — you can't tell what a class needs without opening it up; a constructor signature is self-documenting, a pile of `@Autowired` fields isn't.
3. **Requires reflection to test** — you can't `new` the class up with mocks in a plain unit test; you need Spring's test context or reflection-based mock injection (e.g., Mockito's `@InjectMocks`, which works but is a workaround, not a clean design).
4. **Silently tolerates circular dependencies** (Q5) — hiding a design problem instead of surfacing it at startup.

IDEs like IntelliJ even flag `@Autowired` field injection with a warning ("Field injection is not recommended") for exactly these reasons.

---

### Q23. Can @Autowired be used on constructors? When is it required vs optional?
**Answer:** Yes, and it's the **preferred** usage. Since **Spring 4.3**, `@Autowired` on the constructor is **optional if the class has exactly one constructor** — Spring implicitly uses it for injection. It becomes **required** (must be explicitly annotated) only if the class has **multiple constructors** and you need to tell Spring which one to use for autowiring.

```java
@Service
public class OrderService {
    private final PaymentGateway gateway;

    // @Autowired is OPTIONAL here — only one constructor exists
    public OrderService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
}

@Service
public class NotificationService {
    private final EmailClient emailClient;

    @Autowired   // REQUIRED here — multiple constructors exist, must specify which one Spring should use
    public NotificationService(EmailClient emailClient) {
        this.emailClient = emailClient;
    }

    public NotificationService(EmailClient emailClient, boolean debugMode) {
        this(emailClient);
    }
}
```

**Follow-up:** `@Autowired(required = false)` marks a dependency as **optional** — if no matching bean exists, injection is simply skipped rather than throwing at startup. Generally discouraged in favor of `Optional<T>` or `ObjectProvider<T>` as parameter types, which express optionality more explicitly and type-safely.

---

## SECTION 7: JAVA-BASED CONFIGURATION

### Q24. @Configuration vs @Component — what's the actual difference, and why does @Configuration use CGLIB proxying?
**Answer:** `@Configuration` is meta-annotated with `@Component`, so a `@Configuration` class is itself a Spring bean — but it has one crucial extra behavior: Spring **CGLIB-enhances (subclasses) the configuration class at runtime**, intercepting calls to its `@Bean` methods so that **calling one `@Bean` method from another within the same class returns the same singleton instance** from the container, rather than creating a brand-new object each time.

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());   // CGLIB intercepts this call — returns the SAME singleton dataSource() bean
    }
}
```

**If you used `@Component` instead of `@Configuration`** (a "lite" `@Bean`-method container, allowed but semantically different), that interception **doesn't happen** — calling `dataSource()` directly inside `jdbcTemplate()` would create a **second, separate** `HikariDataSource` instance outside container management, silently breaking the singleton contract. This distinction is a genuinely tricky, high-value senior interview question.

---

### Q25. What does @Bean do, and how is it different from @Component?
| Aspect | `@Component` | `@Bean` |
|---|---|---|
| Applied to | A class (annotated directly on your own class) | A **method** inside a `@Configuration` (or `@Component`) class |
| Use case | Your own classes, discoverable via component-scanning | **Third-party classes you don't own** and can't annotate directly (e.g., configuring a `RestTemplate`, `ObjectMapper`, or a legacy library class) |
| Control over instantiation | Spring calls the constructor directly | You write the actual instantiation logic yourself inside the method — full manual control |

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();   // full manual control — can't annotate RestTemplate itself, it's a third-party class
    }
}
```

**Senior-level rule of thumb:** Use `@Component` (+ stereotype annotations) for classes you own and want auto-discovered; use `@Bean` inside `@Configuration` for third-party classes, or when the bean's construction genuinely needs custom logic/parameters that component-scanning alone can't express.

---

### Q26. Is component scanning enough, or do real projects always need explicit @Configuration classes too?
**Answer (senior-level, practical):** Both, in practice. Component scanning (`@ComponentScan`, implicit via `@SpringBootApplication`) handles the vast majority of your own application classes automatically. `@Configuration` classes remain necessary for:
- Wiring **third-party beans** you don't own (`@Bean` methods, Q25).
- **Conditional bean registration** (`@ConditionalOnProperty`, `@Profile`) where a bean should only exist under certain conditions.
- Centralizing infrastructure wiring (security filter chains, `RestTemplate`/`WebClient` configuration, custom `ObjectMapper` settings) in one discoverable, explicit place rather than scattering `@Component`-annotated wrapper classes everywhere.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is `@Autowired` required on a constructor?" → No, if the class has exactly one constructor (Spring 4.3+); required only with multiple constructors.
- "What happens if two beans have `@Primary`?" → `NoUniqueBeanDefinitionException` at startup — `@Primary` only resolves ambiguity if exactly one candidate is marked primary.
- "Does `@Qualifier` work with `@Primary` present?" → Yes — `@Qualifier` at an injection point always **overrides** `@Primary`'s default choice for that specific injection.
- "Can a `@Bean` method have parameters?" → Yes — parameters are resolved from the container just like constructor injection, letting you compose beans cleanly.
- "Is `ApplicationContext` thread-safe for `getBean()` calls?" → Yes, safe for concurrent reads after startup; the container itself isn't meant to be reconfigured concurrently at runtime.
- "What's the default scope if you don't specify `@Scope`?" → `singleton`.
- "Does `@Component` alone make a class request-mappable like `@Controller`?" → No — `@Component` only registers it as a bean; request mapping requires `@Controller`/`@RestController` plus `@RequestMapping`-family annotations, handled by `DispatcherServlet`'s handler mapping infrastructure.
- "Why can't self-invocation trigger AOP proxies (e.g., `@Transactional`)?" → The proxy wraps the bean from outside; calling `this.method()` bypasses the proxy entirely and calls the raw object directly (ties back to Q8's lifecycle explanation).

---

*Study tip: The constructor-vs-field-injection justification (Q3–Q4), the prototype-in-singleton scoped-proxy trap (Q14–Q15), and the @Configuration CGLIB self-invocation behavior (Q24) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY Spring behaves this way, not just WHAT the annotation does.*
