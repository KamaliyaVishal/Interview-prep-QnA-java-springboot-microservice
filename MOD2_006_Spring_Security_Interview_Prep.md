# Spring Security — Interview Prep
---

## SECTION 1: AUTHENTICATION VS AUTHORIZATION

### Q1. Authentication vs Authorization — what's the actual difference, and where does each happen in the Spring Security filter chain?
**Answer:** Authentication answers **"who are you?"** — verifying identity (username/password, JWT signature, OAuth2 token). Authorization answers **"what are you allowed to do?"** — deciding whether an already-identified principal can access a given resource.

| Aspect | Authentication | Authorization |
|---|---|---|
| Question answered | Who is the caller? | Is the caller allowed to do *this*? |
| Runs | Earlier in the filter chain (e.g., `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`) | After authentication succeeds — `AuthorizationFilter` / `@PreAuthorize` checks |
| Failure result | `401 Unauthorized` | `403 Forbidden` |
| Spring abstraction | `AuthenticationManager` / `AuthenticationProvider` | `AccessDecisionManager` (legacy) / `AuthorizationManager` (modern, Spring Security 6+) |

```java
http
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/public/**").permitAll()      // no authentication required
        .requestMatchers("/api/admin/**").hasRole("ADMIN")  // authenticated + authorized
        .anyRequest().authenticated()                       // authenticated, any role
    );
```

**Senior-level answer:**
> "I always frame this distinction explicitly in code reviews because it's the root cause of most security bugs I've seen: someone secures an endpoint with `authenticated()` and assumes that's enough, forgetting that being logged in doesn't mean being *allowed* — that second check (role/permission) has to be explicit. The 401-vs-403 distinction is also the fastest way to diagnose a security bug report: 401 means the token/credentials never got recognized; 403 means they were recognized but the account lacks permission."

---

## SECTION 2: SECURITYFILTERCHAIN

### Q2. What is SecurityFilterChain, and how did security configuration change from the old WebSecurityConfigurerAdapter approach?
**Answer:** `SecurityFilterChain` is a bean-based, functional-style way of declaring HTTP security rules, replacing the deprecated (removed since Spring Security 6 / Spring Boot 3) `WebSecurityConfigurerAdapter` subclassing approach. Instead of overriding a `configure(HttpSecurity http)` method, you now expose a `SecurityFilterChain` bean directly.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

**Senior-level answer:**
> "The move to a bean-based `SecurityFilterChain` isn't just a syntax change — it enables **multiple filter chains** in the same application, each matched by a `securityMatcher`, which the old subclassing model made awkward. I use this in practice to run a stateless, JWT-based chain for `/api/**` and a stateful, form-login chain for `/admin/**` in the same app, each with `@Order` controlling evaluation priority."

```java
@Bean
@Order(1)
public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain adminFilterChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/admin/**")
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

---

### Q3. Where does a custom filter (e.g., a JWT filter) fit into the filter chain, and why does ordering matter?
**Answer:** Spring Security's filter chain is an ordered list of `Filter`s, each with a specific responsibility (CSRF check, authentication, exception translation, authorization, etc.). Custom filters are inserted **relative to** an existing filter using `addFilterBefore`, `addFilterAfter`, or `addFilterAt` — not appended to the end.

```java
http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

A JWT filter is placed **before** `UsernamePasswordAuthenticationFilter` because it needs to populate the `SecurityContext` from the token *before* downstream authorization checks (`AuthorizationFilter`, near the end of the chain) run — if it ran too late, every request would be rejected as unauthenticated regardless of a valid token.

**Senior-level answer:**
> "Getting filter order wrong is one of the most common Spring Security bugs I debug — symptoms are usually 'my JWT is valid but I still get 401' or 'the filter runs but the SecurityContext is empty downstream.' I always verify order by printing `((FilterChainProxy) filterChain).getFilters()` in a debug test, rather than assuming `addFilterBefore` did what I think it did."

---

## SECTION 3: USERDETAILSSERVICE

### Q4. What is UserDetailsService, and how does it fit into the authentication flow?
**Answer:** `UserDetailsService` is a single-method functional interface — `loadUserByUsername(String username)` — that Spring Security calls during authentication to fetch user data (username, hashed password, authorities, account-status flags) from wherever it's stored (usually a database).

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User
                .withUsername(user.getUsername())
                .password(user.getPasswordHash())
                .authorities(user.getRoles().stream()
                        .map(SimpleGrantedAuthority::new)
                        .toList())
                .accountLocked(!user.isActive())
                .build();
    }
}
```

**Authentication flow:** `DaoAuthenticationProvider` calls `UserDetailsService.loadUserByUsername()` → gets the stored (hashed) password → compares it against the submitted raw password using the configured `PasswordEncoder` → on match, builds an authenticated `Authentication` object and places it in the `SecurityContext`.

**Senior-level answer:**
> "I keep `UserDetailsService` implementations thin — pure data-fetching plus mapping to `UserDetails` — and never put business logic (like login-attempt throttling) directly in it, since it's also invoked in contexts like `remember-me` token validation, not just interactive login. Throttling/lockout logic belongs in an `AuthenticationFailureHandler` or a dedicated `AuthenticationProvider` wrapper instead."

---

### Q5. Trap: If a user isn't found, should loadUserByUsername return null or throw an exception?
**Answer:** Always **throw** `UsernameNotFoundException` — never return `null`. Returning `null` causes a `NullPointerException` deep inside `DaoAuthenticationProvider`, an unhandled 500 error that leaks implementation detail, instead of the clean `BadCredentialsException` → 401 flow Spring Security expects. As a security best practice, the exception message itself (and the outward-facing error response) should **not** distinguish "user not found" from "wrong password" — both should surface identically to the client, to avoid **username enumeration** attacks.

---

## SECTION 4: PASSWORD ENCODING (BCRYPT)

### Q6. Why BCrypt instead of MD5/SHA-256 for password storage, and how does the salt work?
**Answer:** MD5 and raw SHA-256 are **fast** hash functions — great for checksums, terrible for passwords, because that same speed lets an attacker brute-force billions of guesses per second on stolen hash dumps (especially with GPU/ASIC hardware). BCrypt is deliberately **slow and tunable** via a configurable *work factor* (also called "strength" or "cost"), and it **auto-generates and embeds a unique salt per password**, so identical passwords never produce identical hashes — defeating precomputed rainbow-table attacks.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);   // work factor 12 (default is 10)
}
```

```java
String hashed = passwordEncoder.encode("myPassword123");
// $2a$12$N9qo8uLOickgx2ZMRZoMy.MrqzVOoP2LlwlJEXP5EDMxbwPXBfDNa
//  ^^  ^^ ^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//  |   |  |-- 22-char salt --| |------- 31-char hash --------|
//  |   +-- cost factor (work factor 2^12 rounds)
//  +-- algorithm version

boolean matches = passwordEncoder.matches("myPassword123", hashed);  // true — never decode/reverse
```

**Senior-level answer:**
> "The work factor is the real lever here — each increment doubles the computation cost, so I tune it based on measured login-endpoint latency (typically landing around 200–300ms per hash on production hardware, which is imperceptible to a real user but prohibitively expensive at brute-force scale). I also treat the work factor as something to periodically increase over time as hardware gets faster — `BCryptPasswordEncoder` transparently supports re-verifying old hashes at a lower factor while encoding new ones at the current, higher factor."

---

### Q7. What is DelegatingPasswordEncoder, and why does Spring Boot use it by default?
**Answer:** `PasswordEncoderFactories.createDelegatingPasswordEncoder()` — the default returned by `PasswordEncoder` autoconfiguration — stores an **algorithm identifier prefix** (e.g., `{bcrypt}`, `{noop}`, `{scrypt}`) alongside each hash, and delegates encoding/matching to the correct underlying encoder based on that prefix.

```
{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG
```

**Senior-level answer:**
> "This exists specifically to support **migrating hashing algorithms without a hard cutover** — legacy accounts hashed with an older algorithm keep working (identified by their prefix), while all new hashes use the current default (`bcrypt` unless explicitly configured otherwise). I've used this directly when migrating a legacy system off unsalted SHA-1: wrap the old verification logic in a custom `PasswordEncoder`, register it under its own id in the `DelegatingPasswordEncoder`, and re-hash each user's password with BCrypt transparently on their next successful login."

---

## SECTION 5: JWT AUTHENTICATION

### Q8. Walk through how stateless JWT authentication works end-to-end in a Spring Security setup.
**Answer:**
1. **Login:** Client `POST`s credentials to a login endpoint → server authenticates via `AuthenticationManager` → on success, generates a signed JWT (typically HS256/RS256) containing claims (`sub`, `roles`, `exp`) → returns it to the client.
2. **Subsequent requests:** Client sends `Authorization: Bearer <token>` on every request.
3. **Custom filter:** A `OncePerRequestFilter` intercepts each request, extracts the token, validates its signature and expiry, loads user details, and manually populates the `SecurityContextHolder`.
4. **Stateless session policy:** `SessionCreationPolicy.STATELESS` tells Spring Security never to create or read an `HttpSession` — every request is authenticated fresh from the token.

```java
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            if (jwtService.isValid(token)) {
                String username = jwtService.extractUsername(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                var authToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

**Senior-level answer:**
> "The key architectural commitment with JWT is **statelessness** — the server holds no session state, which is what makes JWT-based APIs trivially horizontally scalable behind a load balancer with no sticky sessions or shared session store needed. The trade-off is that you lose the ability to instantly revoke a token server-side; a compromised token remains valid until it expires. I mitigate that with **short-lived access tokens (5–15 min)** plus a **refresh-token rotation** flow, and for genuinely sensitive revocation needs, a server-side denylist (e.g., in Redis, keyed by token `jti`, checked on each request) as a pragmatic compromise on pure statelessness."

---

### Q9. Access token vs refresh token — why use two tokens instead of one long-lived token?
**Answer:**

| Token | Lifetime | Purpose | Where stored client-side |
|---|---|---|---|
| Access token | Short (minutes) | Sent on every API request; if stolen, damage window is small | Memory / short-lived storage |
| Refresh token | Long (days/weeks) | Used **only** to obtain a new access token via a dedicated `/refresh` endpoint; never sent to business API endpoints | HttpOnly, Secure cookie (preferred) |

**Senior-level answer:**
> "This is a direct blast-radius tradeoff. A single long-lived token is convenient but catastrophic if leaked — e.g., via an XSS bug or a logged request — since it stays valid for its entire lifetime with no revocation. Splitting into a short access token plus a refresh token means a leaked access token expires quickly, and the refresh token — the more sensitive, longer-lived credential — is scoped to a single low-surface-area endpoint and stored in an `HttpOnly` cookie specifically so client-side JavaScript, and therefore XSS payloads, can never read it. I also implement **refresh-token rotation**: each refresh call issues a new refresh token and invalidates the old one, so a stolen-but-unused refresh token gets silently invalidated the next time the legitimate client refreshes, which is a strong signal to detect token theft."

---

## SECTION 6: OAUTH2 & OPENID CONNECT

### Q10. OAuth2 vs OpenID Connect — what problem does each actually solve?
**Answer:** OAuth2 is an **authorization** framework — it lets a user grant a third-party application limited access to their resources on another service, **without sharing their password**, via an access token (e.g., "let this app read my Google Calendar"). It says nothing, by itself, about *who* the user is. OpenID Connect (OIDC) is a thin **identity layer built on top of OAuth2** — it adds a standardized `id_token` (a JWT containing identity claims: `sub`, `email`, `name`) specifically so applications can perform **authentication** ("who is this user?") using the same OAuth2 flow.

| | OAuth2 | OpenID Connect |
|---|---|---|
| Solves | Authorization (delegated access) | Authentication (identity) |
| Core artifact | Access token (opaque or JWT, meant for the resource server) | `id_token` (always a JWT, meant for the client app itself) |
| Typical question answered | "Can this app act on my behalf?" | "Who is logging in?" |

**Senior-level answer:**
> "The common mistake I still see is using a raw OAuth2 access token to identify the user in your own app — that's misusing a token that was never designed to prove identity to *you*, only to authorize calls to the resource server it was issued for. If you need to know who the user is (classic 'Login with Google' style flow), you need OIDC's `id_token`, not the access token."

---

### Q11. How does Spring Boot's OAuth2 Client support (spring-boot-starter-oauth2-client) simplify a "Login with X" integration?
**Answer:** Spring Boot autoconfigures the entire authorization-code flow from a handful of properties — redirect handling, state/PKCE generation, token exchange, and `OAuth2User`/`OidcUser` population — with almost no manual wiring.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
```

```java
http.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo.oidcUserService(customOidcUserService))
    .defaultSuccessUrl("/dashboard", true)
);
```

**Senior-level answer:**
> "The value here is that Spring handles the security-critical parts of the OAuth2 dance correctly by default — CSRF-protected `state` parameter, PKCE for public clients, token validation — which are exactly the details that are easy to get subtly wrong hand-rolling this. Where I still write custom code is in `OAuth2UserService`/`OidcUserService` — mapping the provider's claims onto my own application's `User` entity, handling first-login account provisioning, and merging authorities from my own database with whatever the identity provider returns."

---

## SECTION 7: METHOD SECURITY (@PREAUTHORIZE, @SECURED)

### Q12. @PreAuthorize vs @Secured vs @RolesAllowed — differences, and which should you actually use?
**Answer:**

| Annotation | Expression support | Standard | Typical use |
|---|---|---|---|
| `@Secured` | No — only literal role strings (`@Secured("ROLE_ADMIN")`) | Spring-specific | Legacy codebases; simple role checks |
| `@RolesAllowed` | No | JSR-250 (Java EE standard) | Portable across Jakarta EE-compliant frameworks |
| `@PreAuthorize` / `@PostAuthorize` | Yes — full **SpEL** expressions, method arguments, return values | Spring-specific | Modern default choice — nearly everything |

```java
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN') or #order.ownerId == authentication.principal.id")
    public void cancelOrder(Order order) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Order getOrder(Long id) { ... }

    @PreAuthorize("hasAuthority('SCOPE_orders:write')")
    public Order createOrder(OrderRequest request) { ... }
}
```

`@EnableMethodSecurity` (replacing the older `@EnableGlobalMethodSecurity` since Spring Security 6) must be present on a `@Configuration` class to activate these annotations.

**Senior-level answer:**
> "I default to `@PreAuthorize` almost exclusively — the SpEL support means I can express **ownership-based, data-driven authorization** (`#order.ownerId == authentication.principal.id`) directly at the service-method boundary, not just static role checks. `@PostAuthorize` is useful specifically when the authorization decision depends on data only available *after* the method runs — like verifying the returned entity actually belongs to the caller — but it has a real cost: the method's side effects have already executed before the check fails, so it's inappropriate for state-mutating methods; use `@PreAuthorize` there instead."

---

### Q13. Where should authorization checks live — controller layer, service layer, or both? Why?
**Answer:** **Service layer**, as the primary enforcement point — controller-layer checks alone are a common but fragile mistake.

**Senior-level answer:**
> "Controller-level `@PreAuthorize`/`.hasRole()` checks in `SecurityFilterChain` are good for coarse, URL-shape-based rules (`/api/admin/**` requires `ROLE_ADMIN`), but they're **bypassable** the moment that service method gets called from anywhere else — a scheduled job, another service, a message listener, a future internal API. Putting `@PreAuthorize` on the service method itself means the authorization rule travels with the business logic and can't be accidentally bypassed by a new caller. My rule of thumb: URL-pattern rules in the filter chain for broad, cheap-to-evaluate gating (keeps unauthenticated traffic out early, before it costs any real work), and fine-grained, data-aware `@PreAuthorize` rules at the service layer as the actual source of truth."

---

## SECTION 8: CSRF & CORS

### Q14. What is CSRF, and why does Spring Security enable CSRF protection by default for session-based apps but it's typically disabled for stateless JWT APIs?
**Answer:** CSRF (Cross-Site Request Forgery) exploits the fact that browsers automatically attach cookies (including session cookies) to requests, regardless of which site initiated them — a malicious page can silently trigger a state-changing request (`POST /transfer-funds`) to your app, and the browser will happily attach the victim's valid session cookie, making it look legitimate.

**Why it matters for session/cookie auth but not typically for stateless JWT APIs:** CSRF protection exists specifically to defend against the **browser automatically attaching credentials**. A stateless JWT API that requires the token in an `Authorization: Bearer` header (not a cookie) isn't vulnerable in the same way — an attacker's page has no mechanism to make the victim's browser attach a header the attacker doesn't know, since it's not auto-attached like a cookie. This is *why* — not just "convention" — CSRF is commonly disabled for such APIs.

```java
http.csrf(AbstractHttpConfigurer::disable);   // safe ONLY if you're not using cookie-based auth
```

**Senior-level answer:**
> "I never disable CSRF as a reflex — I disable it only after confirming the auth mechanism doesn't rely on browser-auto-attached credentials. The moment an app stores the JWT in a cookie instead of `localStorage`/memory (which some teams do specifically to mitigate XSS token theft), CSRF protection needs to come back, because now you're back to browser-auto-attached credentials on cross-origin requests. Security is about the actual threat model, not a checkbox."

---

### Q15. How do you configure CORS correctly alongside Spring Security, and what's the most common CORS + Security integration mistake?
**Answer:** When Spring Security is on the classpath, it evaluates and can reject requests (including CORS preflight `OPTIONS` requests) **before** they reach any `WebMvcConfigurer`-based CORS configuration — so CORS must be wired into the security chain itself via `.cors(...)`.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

**Common mistake:** Configuring CORS only via `WebMvcConfigurer.addCorsMappings()` and assuming it applies globally once Spring Security is added — it doesn't; Security's own `CorsFilter`/`.cors()` configuration takes precedence and must be explicitly set, or preflight `OPTIONS` requests get blocked before MVC ever sees them (classic symptom: "works in Postman, fails only in the browser").

---

## SECTION 9: ROLE-BASED ACCESS CONTROL (RBAC)

### Q16. Role vs Authority in Spring Security — is there an actual technical difference, or just convention?
**Answer:** Internally, Spring Security only has one concept — `GrantedAuthority`, a simple string. "Role" is a **convention layered on top**: a role is an authority prefixed with `ROLE_`, and methods like `hasRole("ADMIN")` are just sugar that automatically prepends the prefix and checks for `"ROLE_ADMIN"`.

```java
.requestMatchers("/api/admin/**").hasRole("ADMIN")     // checks for authority "ROLE_ADMIN"
.requestMatchers("/api/orders/**").hasAuthority("orders:read")   // checks literal string, no prefix added
```

**Senior-level answer:**
> "In practice I treat 'roles' as coarse-grained buckets (`ADMIN`, `USER`, `MANAGER`) and 'authorities'/'scopes' as fine-grained permissions (`orders:read`, `orders:write`, `users:delete`) — and I design real systems around **permission-based** checks (`hasAuthority`) rather than role-based ones wherever the access model has any nuance, because roles-only RBAC tends to explode into role sprawl (`ADMIN_BUT_NO_BILLING`, `ADMIN_READONLY`) once real business requirements arrive. A cleaner model: roles are just named collections of permissions, resolved into individual `GrantedAuthority` entries at login time, and authorization checks always test the fine-grained permission, never the role name directly."

---

### Q17. How would you model and enforce RBAC where permissions can change without requiring users to log out and back in?
**Answer:** The core tension: `Authentication.getAuthorities()` is normally captured **once, at login**, and baked into the session/token — so a permission change in the database doesn't retroactively affect an already-authenticated user until they re-authenticate.

**Senior-level answer:**
> "For session-based apps, I handle this by not fully trusting the cached authorities — either re-loading fresh authorities from the DB per request via a custom `AuthenticationProvider`/`SecurityContext` refresh strategy for highly sensitive operations, or, more commonly, keeping the access-token/session lifetime short enough (e.g., 15–30 minutes) that stale-permission windows are an acceptable business risk. For JWT specifically, since the token is immutable once issued, the practical fix is short-lived access tokens plus forcing a refresh-token exchange on privilege changes — I've also implemented a 'permission version' claim in the token, checked against a current version stored server-side per user, so a permission change can be enforced by simply bumping that version number and having the API reject any token carrying a stale one, without needing a full revocation list."

---

## SECTION 10: SESSION MANAGEMENT

### Q18. What session-management strategies does Spring Security offer, and when do you pick STATELESS vs a traditional session?
**Answer:**

| Policy | Behavior |
|---|---|
| `ALWAYS` | Creates a session if one doesn't exist |
| `IF_REQUIRED` (default) | Creates a session only when needed (e.g., after login) |
| `NEVER` | Never creates a session, but will use one if it already exists |
| `STATELESS` | Never creates **or** reads a session — every request must carry its own credentials (JWT) |

```java
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

**Senior-level answer:**
> "The decision point is almost always horizontal scalability and client type. Traditional server-rendered apps with browser cookie sessions are simpler to reason about and get CSRF protection for free, but sessions require sticky routing or a shared session store (Redis) once you scale past one instance. I go `STATELESS` + JWT by default for any API meant to be consumed by mobile clients, SPAs, or other services, specifically to avoid that shared-session-store operational overhead — the trade-off, as covered in Q8, is losing instant server-side revocation, which I address separately."

---

### Q19. How do you prevent session fixation attacks, and does Spring Security handle this automatically?
**Answer:** Session fixation is where an attacker sets/knows a victim's session ID *before* login, then hijacks the now-authenticated session once the victim logs in without the session ID changing. Spring Security defends against this **by default** — on successful authentication, it invalidates the pre-login session and issues a brand-new session ID, controllable via `sessionManagement().sessionFixation()`.

```java
http.sessionManagement(session -> session
    .sessionFixation(SessionFixationConfigurer::migrateSession)); // default: copies old session attrs to new session ID
```

Options: `migrateSession()` (default — new ID, old attributes preserved), `newSession()` (new ID, old session's attributes discarded), `changeSessionId()` (Servlet 3.1+ container-level ID change, no new `HttpSession` object), `none()` (disables protection — should essentially never be used).

---

### Q20. How do you limit concurrent sessions per user (e.g., "only one active login at a time")?
**Answer:**
```java
http.sessionManagement(session -> session
    .maximumSessions(1)
    .maxSessionsPreventsLogin(false)   // false = new login kicks out the old session; true = new login is rejected
    .expiredUrl("/login?expired"));
```
This requires a `HttpSessionEventPublisher` bean registered so Spring Security is notified when sessions actually expire/get invalidated at the servlet-container level, keeping its internal session registry accurate.

**Senior-level answer:**
> "This is inherently a **session-based** feature — it relies on the server tracking active sessions in a `SessionRegistry`, so it doesn't apply cleanly to a stateless JWT setup. If a 'single active login' requirement comes up on a JWT-based API, I implement it differently: store the currently-valid token's `jti` (or a session marker) per user server-side, and have each request check the presented token's `jti` against that stored value, invalidating on a new login — functionally the same guarantee, just re-implemented for a stateless architecture."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What status code for authentication failure vs authorization failure?" → 401 (not authenticated) vs 403 (authenticated but not permitted) — see Q1.
- "Is BCrypt hashing reversible?" → No — it's one-way; verification works by re-hashing the input and comparing, never by decoding the stored hash.
- "Can you validate a JWT without hitting the database?" → Yes, for signature/expiry checks (self-contained, stateless) — but authorization decisions needing current roles/permissions may still require a DB lookup unless claims are trusted as current.
- "What's the default password encoder in a fresh Spring Boot Security project?" → `DelegatingPasswordEncoder`, defaulting new hashes to BCrypt — see Q7.
- "Does @PreAuthorize work on private methods or self-invoked calls within the same class?" → No — like other Spring AOP-based features, it relies on proxying, so it's bypassed on internal (`this.method()`) calls within the same bean.
- "What's the risk of storing a JWT in localStorage vs an HttpOnly cookie?" → `localStorage` is readable by any JS on the page, so it's vulnerable to XSS-based token theft; `HttpOnly` cookies aren't readable by JS but reopen CSRF exposure — see Q14.
- "Does disabling CSRF affect GET requests?" → CSRF protection by default only applies to state-changing methods (`POST`/`PUT`/`DELETE`/`PATCH`); `GET` requests are exempt since they're expected to be side-effect-free (per HTTP semantics).

---

*Study tip: The Spring Security + CORS interaction trap (Q15), the service-layer-vs-controller-layer authorization placement argument (Q13), and the access-token/refresh-token blast-radius reasoning (Q9) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice explaining the underlying threat model, not just the annotation syntax.*
