# Spring Boot Fundamentals — Interview Prep
---

## SECTION 1: @SpringBootApplication & AUTO-CONFIGURATION

### Q1. What does @SpringBootApplication actually do? Break it down.
**Answer:** `@SpringBootApplication` is a **convenience meta-annotation** combining three separate annotations:

```java
@SpringBootConfiguration   // = @Configuration — this class is a source of bean definitions
@EnableAutoConfiguration   // triggers Spring Boot's auto-configuration mechanism
@ComponentScan             // scans the current package + sub-packages for @Component-annotated classes
public @interface SpringBootApplication { }
```

- **`@SpringBootConfiguration`** — a specialized `@Configuration`, marking the class as a bean-definition source (Q24 of Spring Core prep applies here too).
- **`@EnableAutoConfiguration`** — the real "magic" (Q2) — tells Spring Boot to automatically configure beans based on what's on the classpath.
- **`@ComponentScan`** — scans for `@Component`/`@Service`/`@Repository`/`@Controller` classes, **starting from the package of the annotated class and all sub-packages** — this is exactly why the main application class must sit at the **root package**, above every other package in the project.

**Senior trap to flag proactively:** If you place `@SpringBootApplication` in a sub-package or a package that doesn't sit above all your `@Component` classes, component scanning silently misses beans outside its scan path — one of the most common "why isn't my bean being picked up" production bugs for developers new to Spring Boot's convention-over-configuration philosophy.

---

### Q2. What is auto-configuration? How does Spring Boot decide what to configure automatically?
**Answer:** Auto-configuration is Spring Boot's mechanism for **automatically creating and wiring beans based on what's present on the classpath**, environment properties, and existing bean definitions — without you writing explicit `@Configuration`/`@Bean` boilerplate. Example: if `spring-boot-starter-data-jpa` and an H2/Postgres driver are on the classpath, Spring Boot auto-configures a `DataSource`, `EntityManagerFactory`, and `TransactionManager` for you.

**How it works internally:**
1. `@EnableAutoConfiguration` triggers Spring Boot to read `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+; older versions used `spring.factories`) from every JAR on the classpath — a plain text file listing candidate `@Configuration` classes.
2. Each candidate auto-configuration class is **conditionally evaluated** using `@Conditional`-family annotations (Q3).
3. Only the auto-configurations whose conditions are satisfied actually register their beans.

```java
@Configuration
@ConditionalOnClass(DataSource.class)                          // only if DataSource is on the classpath
@ConditionalOnMissingBean(DataSource.class)                    // only if the user hasn't defined their own
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration { /* ... */ }
```

**Senior-level answer:**
> "Auto-configuration is really just a big collection of ordinary `@Configuration` classes, shipped inside Spring Boot's `spring-boot-autoconfigure` JAR, each guarded by `@Conditional` checks. It's not magic — it's convention-driven wiring that backs off the moment you define your own bean of the same type, via `@ConditionalOnMissingBean`. Understanding that gives you the tools to override or disable any piece of it deliberately, rather than treating it as an opaque black box."

---

### Q3. What are the key @Conditional annotations Spring Boot uses to drive auto-configuration?
| Annotation | Condition |
|---|---|
| `@ConditionalOnClass` | A specific class is present on the classpath |
| `@ConditionalOnMissingClass` | A specific class is **absent** from the classpath |
| `@ConditionalOnBean` | A bean of a given type **already exists** in the context |
| `@ConditionalOnMissingBean` | A bean of a given type does **not** already exist — lets user-defined beans override auto-configuration |
| `@ConditionalOnProperty` | A given property is set (optionally to a specific value) in `application.properties`/`.yml` |
| `@ConditionalOnWebApplication` | The application is a web application (servlet or reactive) |
| `@ConditionalOnExpression` | A SpEL expression evaluates to `true` |

```java
@Bean
@ConditionalOnMissingBean(PaymentGateway.class)   // only registers the default if user hasn't defined their own
public PaymentGateway defaultPaymentGateway() {
    return new StripePaymentGateway();
}
```

**Follow-up:** "How do you override an auto-configured bean?" → Simply define your **own** `@Bean` of the same type in your own `@Configuration` — `@ConditionalOnMissingBean` on the auto-configuration means it backs off automatically. No need to explicitly exclude anything unless you want to prevent the auto-configuration class from even being evaluated (Q4).

---

### Q4. How do you exclude a specific auto-configuration class entirely?
**Answer:** Two common approaches:
```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApp { }
```
or via properties (preferred when the exclusion should vary by environment/profile):
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
**When you'd actually do this:** A common real scenario is a service that has a JDBC driver on the classpath (transitively, via another dependency) but doesn't actually need a `DataSource` — without excluding it, Spring Boot might try to auto-configure a datasource and fail startup because no connection URL is configured. Explicitly excluding it avoids the failure cleanly rather than fighting the auto-configuration with dummy properties.

---

### Q5. How would you debug which auto-configurations were applied (or not) in a running Spring Boot app?
**Answer:** Two practical techniques:
1. **`--debug` flag** (or `debug=true` in properties) — on startup, Spring Boot prints a full **auto-configuration report**: "Positive matches" (applied) and "Negative matches" (skipped, with the exact reason — e.g., "did not find required class").
2. **Actuator's `/actuator/conditions` endpoint** (Q22) — same report, queryable at runtime via HTTP, useful in environments where you can't easily restart with `--debug`.

**Senior talking point:** This is genuinely one of the most useful day-to-day debugging techniques for "why is my bean not being created / why is this datasource configuring itself unexpectedly" issues — I use it far more often than stepping through auto-configuration source code.

---

## SECTION 2: STARTERS & DEPENDENCY MANAGEMENT

### Q6. What is a Spring Boot "starter"? Why do they exist?
**Answer:** A starter is a **curated dependency descriptor POM** — a single artifact (e.g., `spring-boot-starter-web`) that transitively pulls in a **coherent, version-compatible set of libraries** for a given use case, so you don't have to manually pick and version-match dozens of individual dependencies yourself.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- transitively brings in: spring-webmvc, embedded Tomcat, Jackson, validation-api, spring-boot-starter, spring-boot-starter-logging, etc. -->
```

**Why this matters for senior interviews:** Starters solve the classic **"dependency hell"** problem — before Boot, you'd manually pick a Spring MVC version, a Jackson version, a servlet container version, and hope they were mutually compatible. `spring-boot-starter-parent` (or `spring-boot-dependencies` BOM) centrally manages **all** these versions, tested together as a compatible set, so you typically never specify a version for a Spring-managed dependency yourself — you just declare the starter and inherit a known-good version.

---

### Q7. What's the difference between spring-boot-starter-parent and importing the spring-boot-dependencies BOM directly?
| Approach | What it does |
|---|---|
| `spring-boot-starter-parent` (Maven `<parent>`) | Inherits **dependency version management** *and* plugin configuration, resource filtering, default encoding, Java version, etc. — an all-in-one parent POM |
| `spring-boot-dependencies` (imported as a BOM via `<dependencyManagement>`) | **Only** dependency version management — used when your project already has its own parent POM (e.g., a corporate/organizational parent) and can't also inherit from Spring Boot's |

```xml
<!-- Option 2: use when you already have a different required parent POM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
**Senior detail:** Most real enterprise projects at larger companies (fintech especially) **can't** use `spring-boot-starter-parent` because they already inherit from a corporate parent POM enforcing internal standards — knowing the BOM-import alternative is a genuinely practical, frequently-needed piece of knowledge.

---

### Q8. Name a few commonly used Spring Boot starters and what they bring in.
| Starter | Purpose |
|---|---|
| `spring-boot-starter-web` | Spring MVC + embedded Tomcat + Jackson (REST APIs) |
| `spring-boot-starter-webflux` | Reactive stack (Project Reactor + Netty by default) |
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate + JDBC |
| `spring-boot-starter-security` | Spring Security auto-configuration |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Spring Test — the standard testing baseline |
| `spring-boot-starter-actuator` | Production-readiness endpoints (Q19–Q22) |
| `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) for `@Valid`/`@Validated` |
| `spring-boot-starter-cache` | Caching abstraction (`@Cacheable`, etc.) |

---

## SECTION 3: CONFIGURATION — application.properties vs application.yml

### Q9. application.properties vs application.yml — differences, and which do you prefer?
| Aspect | `.properties` | `.yml` |
|---|---|---|
| Syntax | Flat key-value pairs | Hierarchical, indentation-based |
| Readability for nested config | Repetitive prefixes (`app.db.host`, `app.db.port`) | Naturally nested, less repetition |
| Lists | Indexed keys (`app.servers[0]=a`, `app.servers[1]=b`) | Native YAML list syntax (`- a`, `- b`) |
| Multiple profiles in one file | Not supported natively (needs separate `-{profile}` files) | Supported via `---` document separators in a single file |
| Comment support | `#` | `#` |
| Common pitfalls | Verbose for deeply nested config | **Whitespace-sensitive** — a misplaced indent silently breaks parsing or creates the wrong structure; tabs are not allowed |

```properties
# application.properties
spring.datasource.url=jdbc:postgresql://localhost:5432/orders
spring.datasource.username=admin
app.feature.new-checkout.enabled=true
```
```yaml
# application.yml — equivalent, nested
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: admin
app:
  feature:
    new-checkout:
      enabled: true
```

**Senior-level answer:**
> "I lean toward YAML for larger configuration files because the hierarchical structure mirrors the actual configuration model and multiple profiles can live in a single file with `---` separators, which keeps environment-specific overrides visually close together. That said, YAML's whitespace sensitivity is a real source of subtle bugs — a wrong indent doesn't always fail loudly, it can silently produce an unintended structure — so for smaller, flatter configs `.properties` is arguably safer and just as readable. Either is fine; the key is team consistency, not which one is objectively 'better'."

---

### Q10. What is the configuration property precedence order in Spring Boot? (High-value senior question)
**Answer (highest to lowest priority — this ordering is exactly what interviewers test):**
1. Command-line arguments (`--server.port=8081`)
2. `SPRING_APPLICATION_JSON` (inline JSON in an env var)
3. JNDI attributes
4. Java System properties (`-Dserver.port=8081`)
5. OS environment variables
6. Profile-specific properties **outside** the packaged jar (`application-{profile}.yml`)
7. Profile-specific properties **inside** the packaged jar
8. Application properties **outside** the packaged jar (`application.yml`)
9. Application properties **inside** the packaged jar
10. `@PropertySource` annotations
11. Default properties (`SpringApplication.setDefaultProperties`)

**Senior-level answer:**
> "The rule of thumb I actually use day to day: **externalized configuration always wins over packaged configuration**, and **more specific always wins over more general** — command-line args and environment variables (used heavily in containerized/Kubernetes deployments via ConfigMaps and Secrets) override anything baked into the JAR. This is exactly why you can build one JAR and deploy it identically across dev/staging/prod, overriding only what differs via environment variables or a mounted config file — you never rebuild the artifact per environment."

---

### Q11. How do you bind a group of related properties to a strongly-typed Java object instead of injecting individual values with @Value?
```yaml
app:
  feature:
    new-checkout:
      enabled: true
      max-retry: 3
```
```java
@ConfigurationProperties(prefix = "app.feature.new-checkout")
public class CheckoutFeatureProperties {
    private boolean enabled;
    private int maxRetry;
    // getters/setters (or use a record in Boot 3.x — constructor binding)
}

@Configuration
@EnableConfigurationProperties(CheckoutFeatureProperties.class)
public class FeatureConfig { }
```
**Answer:** `@ConfigurationProperties` binds an entire configuration **tree** to a POJO/record, rather than injecting scattered individual `@Value("${...}")` fields.

**Senior-level answer (why this is preferred):**
> "I prefer `@ConfigurationProperties` over scattering `@Value` annotations across the codebase for anything beyond a couple of standalone values. It's **type-safe** (Boot validates and converts types at binding time, failing fast on misconfiguration), supports **relaxed binding** (`max-retry`, `maxRetry`, `MAX_RETRY` all bind to the same field), integrates with **Bean Validation** (`@Validated` + `@NotNull`/`@Min` on fields), and groups related config into one cohesive, IDE-autocomplete-friendly object instead of magic strings scattered everywhere. `@Value` is fine for a single, truly standalone property; anything structured belongs in a `@ConfigurationProperties` class."

---

## SECTION 4: PROFILES

### Q12. What are Spring Profiles? How do you activate one?
**Answer:** Profiles let you define **environment-specific beans and configuration** (dev, test, prod, etc.) that are only active when that profile is selected.

```java
@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() { return productionPooledDataSource(); }
}

@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() { return new EmbeddedH2DataSource(); }
}
```

**Activating a profile (in order of how commonly used):**
```bash
# Command line
java -jar app.jar --spring.profiles.active=prod

# Environment variable (typical in containers/Kubernetes)
SPRING_PROFILES_ACTIVE=prod

# In application.yml
spring:
  profiles:
    active: dev
```

---

### Q13. How does application-{profile}.yml relate to the base application.yml? Does it replace or merge?
**Answer:** It **merges** — `application-{profile}.yml` (or `.properties`) is layered **on top of** the base `application.yml`, overriding only the specific keys it defines; everything else from the base file still applies. This is precisely why the standard pattern is: put **common/shared** config in `application.yml`, and put only the **environment-specific overrides** (DB URLs, log levels, feature flags) in each `application-{profile}.yml`.

```yaml
# application.yml (shared/common — always loaded)
server:
  port: 8080
logging:
  level:
    root: INFO
---
# application-dev.yml (only the dev-specific override)
logging:
  level:
    root: DEBUG   # overrides root level only — server.port: 8080 still applies from base
```

---

### Q14. Can multiple profiles be active at once? Give a real-world use case.
**Answer:** Yes — `spring.profiles.active=prod,eu-region` activates **both** profiles simultaneously; beans/configuration from both are applied (if there's a conflicting key, the **later-listed** profile in the active list wins).

**Real production use case (strong senior talking point):**
> "A pattern I've used: one profile dimension for the **environment** (`dev`/`staging`/`prod`) and a separate, orthogonal profile for **region** or **deployment variant** (`eu-region`/`us-region`, or `kafka-enabled`/`kafka-disabled` for a service that can run with or without event publishing in certain deployments). Combining `prod,eu-region` lets you compose configuration axes independently instead of creating a combinatorial explosion of profile files like `prod-eu`, `prod-us`, `staging-eu`, etc."

**Related feature worth mentioning:** `spring.profiles.group` (Boot 2.4+) lets you define a profile that automatically activates several others — e.g., activating `production` auto-includes `prod-db,prod-logging,prod-security` without listing them all explicitly every time.

---

## SECTION 5: EMBEDDED SERVERS

### Q15. Why does Spring Boot use an embedded server instead of deploying a WAR to an external Tomcat?
**Answer:** The embedded server model means the application is a **self-contained, runnable JAR** with the servlet container packaged inside it (`java -jar app.jar` starts everything) — no separate Tomcat installation, no WAR deployment step, no version mismatch between "what I tested locally" and "what's installed on the server."

**Senior-level answer (this is the real "why", not just "what"):**
> "This is foundational to Spring Boot's whole philosophy of **'production-ready, self-contained artifacts.'** It aligns perfectly with container-based deployment (Docker/Kubernetes) — the JAR *is* the deployable unit, with everything it needs to run baked in, which is exactly the immutable-artifact model containers are built around. It also means dev and prod run the **exact same server version** since it's a versioned dependency in your build, not something separately installed and potentially drifting on a shared app server."

---

### Q16. Tomcat vs Jetty vs Undertow — differences, and how do you switch?
| Server | Characteristics |
|---|---|
| **Tomcat** (default) | Most widely used, thread-per-request model, mature ecosystem, default choice unless you have a specific reason to change |
| **Jetty** | Lighter-weight, historically favored for high-concurrency/long-lived-connection scenarios (WebSockets), smaller memory footprint |
| **Undertow** | Non-blocking I/O core (by Red Hat/WildFly), generally the best raw throughput/lowest memory footprint of the three, used by both `spring-boot-starter-web` (blocking) and `spring-boot-starter-webflux` (reactive) |

**Switching servers:** Exclude the default Tomcat starter, then add the alternative:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

**Senior-level answer:**
> "In practice, I keep the default Tomcat for most standard Spring MVC (blocking) services — it's the best-tested, most widely documented path, and switching servers is rarely where real performance problems come from. I'd only reach for Undertow if profiling showed the servlet container itself was a genuine bottleneck under very high concurrency, which is uncommon compared to database/downstream-call latency being the actual constraint."

---

### Q17. How do you customize the embedded server (port, connection timeout, thread pool)?
```yaml
server:
  port: 8443
  tomcat:
    threads:
      max: 200
      min-spare: 20
    connection-timeout: 20000
    max-connections: 10000
```
**Answer:** Most common tuning is done declaratively via `server.*` properties — no code needed for standard adjustments. For programmatic customization (e.g., conditional logic), implement `WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>`:
```java
@Component
public class ServerCustomizer implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
    @Override
    public void customize(ConfigurableServletWebServerFactory factory) {
        factory.setPort(8443);
    }
}
```

---

## SECTION 6: SPRING BOOT ACTUATOR

### Q18. What is Spring Boot Actuator? Why is it critical for production systems?
**Answer:** Actuator exposes a set of **production-readiness HTTP (or JMX) endpoints** — health checks, metrics, environment details, thread dumps, and more — that operational tooling (load balancers, Kubernetes probes, monitoring systems like Prometheus/Grafana, incident responders) rely on to observe a running application without needing to SSH in or attach a debugger.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Senior-level answer:**
> "At every company I've worked at with any real production maturity, Actuator's `/health` endpoint is wired directly into Kubernetes liveness/readiness probes, and `/metrics` (via Micrometer) feeds Prometheus for dashboards and alerting. I consider Actuator a non-negotiable baseline for any production Spring Boot service, not an optional add-on — you genuinely cannot operate a service reliably at scale without it."

---

### Q19. What are the most important Actuator endpoints, and what does each expose?
| Endpoint | Exposes |
|---|---|
| `/actuator/health` | Overall application health (UP/DOWN), aggregated from individual `HealthIndicator`s (DB connectivity, disk space, custom checks) |
| `/actuator/info` | Arbitrary static/build info (app version, git commit — populated via `info.*` properties or the `git.properties`/`build-info.properties` files) |
| `/actuator/metrics` | Runtime metrics (JVM memory, GC pauses, HTTP request counts/latencies, custom `Counter`/`Timer` via Micrometer) |
| `/actuator/env` | All active configuration properties and their resolved source (useful for debugging "which value actually won" per Q10's precedence order) |
| `/actuator/loggers` | View **and dynamically change** log levels at runtime, without a restart — extremely valuable for live production debugging |
| `/actuator/threaddump` | Full JVM thread dump — critical for diagnosing deadlocks/thread starvation live |
| `/actuator/heapdump` | Downloadable heap dump for memory-leak analysis |
| `/actuator/conditions` | Auto-configuration positive/negative match report (Q5) |
| `/actuator/mappings` | All registered `@RequestMapping` endpoints — useful for verifying routing configuration |

---

### Q20. By default, are all Actuator endpoints exposed over HTTP? What's the security implication?
**Answer:** No — by default, **only `/health` and `/info`** are exposed over HTTP; the rest require explicit opt-in:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
        # exclude: env, heapdump   # or explicitly exclude sensitive ones
  endpoint:
    health:
      show-details: when-authorized   # avoid leaking internal health details to unauthenticated callers
```
**Senior security talking point (highly relevant for fintech interviews):**
> "Actuator endpoints like `/env`, `/heapdump`, and `/threaddump` can leak **sensitive data** — connection strings, credentials embedded in environment variables, or raw memory contents — if exposed publicly. I always lock down Actuator behind a **separate management port** (`management.server.port`), require authentication via Spring Security for anything beyond `/health`, and only ever expose the specific endpoints actually needed rather than `include: '*'`. This has been a real, repeatedly-flagged finding in security audits at companies handling financial data — treating Actuator as 'internal-only, authenticated' by default is a strong practice to state proactively."

---

### Q21. How do you write a custom health indicator?
```java
@Component
public class DownstreamServiceHealthIndicator implements HealthIndicator {
    private final RestTemplate restTemplate;

    @Override
    public Health health() {
        try {
            restTemplate.getForEntity("http://payment-service/health", String.class);
            return Health.up().withDetail("payment-service", "reachable").build();
        } catch (Exception e) {
            return Health.down(e).withDetail("payment-service", "unreachable").build();
        }
    }
}
```
**Answer:** Implement `HealthIndicator` (or extend `AbstractHealthIndicator`), and Spring Boot automatically **aggregates it into the overall `/health` status** — if any indicator reports `DOWN`, the aggregate health becomes `DOWN`. This is how you make Kubernetes readiness probes actually reflect real downstream dependency health, not just "the JVM process is running."

---

### Q22. How do Actuator health checks map to Kubernetes liveness vs readiness probes? (High-value senior/DevOps-adjacent question)
**Answer:**
| Kubernetes probe | Purpose | Actuator equivalent |
|---|---|---|
| **Liveness** | "Is the process healthy enough to keep running, or should it be restarted?" | `/actuator/health/liveness` — should reflect only whether the app itself is deadlocked/broken, **not** downstream dependency failures |
| **Readiness** | "Is the app ready to receive traffic right now?" | `/actuator/health/readiness` — should reflect downstream dependency health (DB, message broker) — if a critical dependency is down, the pod should stop receiving traffic without being killed and restarted |

**Senior-level nuance (a genuinely common production mistake to call out):**
> "A mistake I've seen repeatedly: wiring a downstream DB health check into the **liveness** probe. If the database goes down temporarily, Kubernetes would restart every pod in a crash-loop — pointless and actively harmful, since restarting the app does nothing to fix a downed database. Downstream dependency checks belong in **readiness**, so the pod is simply pulled out of the load balancer rotation until the dependency recovers, while liveness stays focused purely on 'is this process itself stuck/broken.' Spring Boot's built-in **liveness/readiness groups** (`management.endpoint.health.probes.enabled=true`) exist specifically to make this separation easy to configure correctly."

---

## SECTION 7: DEVTOOLS

### Q23. What does spring-boot-devtools provide, and should it ship to production?
**Answer:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>   <!-- doesn't propagate to consumers of this artifact -->
</dependency>
```
Key features:
- **Automatic restart** — watches the classpath and triggers a fast application restart on code changes (uses **two classloaders** internally — a base loader for unchanging third-party JARs, and a restart loader for your own classes — so restarts are much faster than a full cold JVM start).
- **LiveReload** — auto-refreshes the browser on static resource changes.
- **Disabled caching** for template engines (Thymeleaf, etc.) in dev, so template edits are visible immediately without a restart.
- **Remote debug tunneling** support (`spring.devtools.remote.secret`) — rarely used, mostly historical.

**Answer to "should it ship to production": No, and by default it doesn't have to be manually excluded** — Spring Boot **automatically disables DevTools' restart/live-reload features when the application is run from a repackaged, fully executable JAR** (detected via the classloader), which is exactly how production artifacts are typically run. Still, best practice is to keep the dependency scoped `optional`/`provided`-equivalent (or exclude it in the production build profile) so it isn't unnecessarily bundled at all, and to never rely solely on Boot's auto-detection as the only safety net.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can you have both application.properties and application.yml in the same project?" → Technically yes, but Boot only loads one style consistently in practice — mixing them is confusing and strongly discouraged; pick one per project.
- "What's the difference between @Value and @ConfigurationProperties for lists?" → `@Value` requires awkward syntax for collections (`${app.servers}` with a comma-separated string, manually split); `@ConfigurationProperties` binds YAML/properties lists natively to `List<String>`/`List<SomeType>` fields.
- "How do you set a default profile if none is specified?" → `spring.profiles.default=dev` in `application.yml` — used only if `spring.profiles.active` isn't set at all.
- "Does excluding an auto-configuration class throw an error if that class isn't on the classpath?" → Yes — use `excludeName` (string class name) instead of `exclude` (class reference) if the class might not always be present, to avoid a `ClassNotFoundException` at startup.
- "What port does Actuator run on by default?" → Same port as the main application (`server.port`) unless `management.server.port` is explicitly set to a separate port.
- "Is DevTools' restart the same as a full JVM restart?" → No — it's an in-JVM restart using a dedicated restart classloader for your own classes, which is why it's noticeably faster than stopping and re-launching the JVM entirely.

---

*Study tip: Auto-configuration's @Conditional mechanism (Q2–Q3), configuration property precedence order (Q10), and the liveness-vs-readiness health check distinction (Q22) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining WHY Spring Boot behaves this way, not just WHAT the annotation or property does.*
