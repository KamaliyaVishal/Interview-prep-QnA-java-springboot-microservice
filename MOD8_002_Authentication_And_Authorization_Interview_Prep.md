# Authentication & Authorization — Interview Prep
---

## SECTION 1: AUTHENTICATION vs AUTHORIZATION

### Q1. Authentication vs Authorization — give the precise distinction, and walk through where each actually happens in a Spring Security request.
**Answer:** **Authentication** answers "who are you?" — verifying an identity claim (username/password, a token's signature) and establishing a trusted `Principal`. **Authorization** answers "what are you allowed to do?" — given that established identity, deciding whether this specific request against this specific resource is permitted. They're sequential and distinct: a request can be **fully authenticated** and still be **denied** at the authorization step (MOD8_001 Q1's Broken Access Control is specifically an authorization failure, not an authentication one).

```java
// AUTHENTICATION — happens in the Spring Security filter chain, establishes WHO the caller is
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String token = extractToken(req);
        if (jwtValidator.isValid(token)) {
            Authentication auth = new UsernamePasswordAuthenticationToken(
                jwtValidator.extractUsername(token), null, jwtValidator.extractAuthorities(token));
            SecurityContextHolder.getContext().setAuthentication(auth);  // identity established
        }
        chain.doFilter(req, res);
    }
}

// AUTHORIZATION — happens AFTER authentication, decides if THIS identity can do THIS thing
@PreAuthorize("hasRole('ADMIN') or #orderId == authentication.principal.currentOrderId")
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable String orderId) { ... }
```

**Senior-level answer:** "The mistake I see most often — and specifically call out in code review — is a team implementing solid authentication (JWT validation, proper token handling) and then treating that as 'security done,' without a **separate, resource-specific authorization check**. Authentication proves you're a legitimate, logged-in user; it says nothing about whether **this** user should see **this specific order**. That gap is exactly how IDOR vulnerabilities (MOD8_001 Q1/Q4) happen — the request passes authentication cleanly, and nobody checks ownership at the authorization layer."

---

## SECTION 2: JWT

### Q2. What is a JWT, what's actually inside one, and what are the real security pitfalls teams run into using it?
**Answer:** A JSON Web Token is a **self-contained, cryptographically signed token** carrying claims (identity, roles, expiry) — three base64url-encoded parts separated by dots: `header.payload.signature`. The signature lets a receiving service verify the token hasn't been tampered with, **without needing to call back to the issuer** for every request.

```
Header:    {"alg": "RS256", "typ": "JWT"}
Payload:   {"sub": "user123", "roles": ["ADMIN"], "iat": 1700000000, "exp": 1700003600}
Signature: RSASHA256(base64(header) + "." + base64(payload), privateKey)
```

```java
// Issuing
String token = Jwts.builder()
    .subject("user123")
    .claim("roles", List.of("ADMIN"))
    .issuedAt(new Date())
    .expiration(Date.from(Instant.now().plus(1, ChronoUnit.HOURS)))
    .signWith(privateKey, Jwts.SIG.RS256)
    .compact();

// Validating — MUST verify signature, not just decode
Jws<Claims> claims = Jwts.parser().verifyWith(publicKey).build().parseSignedClaims(token);
```

**Real security pitfalls — this is where the depth actually is:**
- **"none" algorithm attack** — older/misconfigured JWT libraries would accept a token with `alg: none` and **skip signature verification entirely**; an attacker crafts a token with arbitrary claims (`roles: [ADMIN]`) and no signature, and a vulnerable parser accepts it as valid. Modern libraries default to rejecting this, but it's a real historical CVE pattern worth naming.
- **Algorithm confusion (RS256 → HS256)** — if a server is configured to accept **either** algorithm, an attacker who knows the RSA **public** key (often not actually secret — it's meant to be shared for verification) can forge a token signed with HS256, using the public key as the HMAC secret. **Fix:** explicitly pin the expected algorithm when parsing, never trust the algorithm declared in the token's own header.
- **No revocation mechanism** — a JWT is valid until it expires; there's **no built-in way to invalidate one early** (user logs out, account compromised, permissions changed) without extra infrastructure — this is JWT's single biggest practical trade-off, addressed via short expiry + a refresh-token pattern, or a server-side denylist for critical revocation needs.
- **Storing sensitive data in the payload** — the payload is only base64-**encoded**, not encrypted; anyone can decode and read it. Never put secrets/PII you wouldn't want visible into JWT claims.
- **Storage on the client side** — storing a JWT in `localStorage` is vulnerable to theft via XSS (MOD8_001 Q2); an `httpOnly`, `Secure`, `SameSite` cookie is generally safer, though it reintroduces CSRF considerations (MOD8_001 Q2) that need their own explicit handling.

**Senior-level answer on the revocation trade-off specifically:** "The lack of built-in revocation is the JWT trade-off I make sure to flag proactively in any design discussion — it's the direct cost of JWT's main benefit (stateless, no database lookup needed to validate every request). My standard pattern: **short-lived access tokens** (minutes, not hours) paired with a **longer-lived refresh token** that IS checked against a server-side store on each use — that bounds the exposure window of a compromised access token to just the short expiry, while still keeping most request validation stateless and fast."

---

## SECTION 3: OAUTH2 & OPENID CONNECT

### Q3. OAuth2 vs OpenID Connect — these get conflated constantly; what's the actual relationship, and walk through the Authorization Code flow.
**Answer:** **OAuth2** is an **authorization** framework — it lets a user grant a third-party application **limited access to their resources** on another service, without sharing their password (e.g., "let this app read my Google Calendar"). It was never actually designed to answer "who is this user" in a standardized way. **OpenID Connect (OIDC)** is an **authentication** layer built directly on top of OAuth2 — it adds a standardized **ID Token** (a JWT, Q2) containing the user's verified identity, plus a standard `/userinfo` endpoint — solving the "who is this user" problem OAuth2 alone doesn't address.

**The distinction stated plainly:** "OAuth2 answers 'is this app allowed to access this resource on the user's behalf' (authorization); OIDC answers 'who is this user, verified by the identity provider' (authentication) — OIDC literally extends OAuth2's protocol with an extra token type and endpoint specifically to close that gap, rather than being a wholly separate protocol."

**Authorization Code flow (the standard, most secure flow for a server-side web app):**
```
1. User clicks "Login with Google" → app redirects to Google's /authorize endpoint
2. User authenticates directly with Google, consents to requested scopes
3. Google redirects back to the app with a short-lived AUTHORIZATION CODE (not a token yet)
4. App's BACKEND exchanges that code + client secret for tokens via a server-to-server call:
       POST /token  { code, client_id, client_secret, redirect_uri }
5. Google returns: access_token (OAuth2 — for calling Google APIs)
                    id_token (OIDC — a JWT proving who the user is)
                    refresh_token (for getting new access tokens later)
```

```java
// Spring Security OAuth2 Client — configuring an OIDC login
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: [openid, profile, email]   # "openid" scope is what turns this into an OIDC flow
```

**Why the Authorization Code flow specifically, and not simpler alternatives:** "The key security property is that the actual **access token and ID token are never exposed to the browser/front-channel** — only a short-lived, single-use authorization code passes through the redirect, and the real token exchange happens **server-to-server** using a client secret the browser never sees. The older **Implicit flow** (tokens returned directly in the redirect URL fragment) is now explicitly discouraged specifically because it exposes tokens to the browser history, referrer leakage, and any XSS on the page — modern guidance is Authorization Code flow (with PKCE for public clients like SPAs/mobile apps that can't safely hold a client secret) for essentially everything."

**Follow-up: "What's PKCE, and why does a mobile app or SPA need it?"** → Proof Key for Code Exchange — since a public client (SPA, mobile app) can't safely store a client secret (it's shipped in client-side code, extractable), PKCE has the client generate a random `code_verifier`, send its hash (`code_challenge`) in the initial auth request, and present the original `code_verifier` during token exchange — preventing an attacker who intercepts the authorization code from completing the exchange themselves without also having generated the matching verifier.

---

## SECTION 4: RBAC vs ABAC

### Q4. RBAC vs ABAC — compare them, and give a real example of an authorization requirement that RBAC genuinely can't express well.
**Answer:** **RBAC (Role-Based Access Control)** grants permissions based on a user's **assigned role** — "ADMIN can delete orders," "USER can view their own orders." **ABAC (Attribute-Based Access Control)** grants permissions based on evaluating **attributes of the user, the resource, and the context together** at request time — a much more expressive, fine-grained model.

```java
// RBAC — simple, role-based, works great for coarse permissions
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/orders/{id}")
public void deleteOrder(@PathVariable String id) { ... }
```

```java
// ABAC — evaluates attributes of user + resource + context together, not just a role name
@PreAuthorize("""
    hasRole('MANAGER') 
    and #order.department == authentication.principal.department
    and #order.amount <= authentication.principal.approvalLimit
    and T(java.time.LocalTime).now().isBefore(T(java.time.LocalTime).parse('18:00'))
""")
@PostMapping("/orders/{orderId}/approve")
public void approveOrder(@PathVariable Order order) { ... }
// A pure role check ("MANAGER can approve") can't express: same department, within THIS manager's
// specific approval limit, AND only during business hours — that requires evaluating actual attributes
```

**A real example where RBAC genuinely breaks down:** "'A manager can approve expense reports from their own department, up to their individually assigned approval limit, but not their own submitted expenses' — RBAC's `hasRole('MANAGER')` can't express **any** of the department-matching, dollar-limit, or self-approval-exclusion parts; you'd end up bolting ad-hoc `if` checks onto every endpoint to compensate, which is exactly the kind of scattered, easy-to-miss authorization logic that causes real Broken Access Control bugs (MOD8_001 Q1). ABAC, or a proper policy engine, expresses that whole rule declaratively in one place."

**Senior-level answer on the actual trade-off:** "RBAC is simpler to reason about, simpler to audit ('who has the ADMIN role'), and is the right default for coarse-grained permission needs — most systems genuinely don't need more. I reach for ABAC (or a hybrid — RBAC for coarse gating, with additional attribute checks layered on for the genuinely context-dependent rules, exactly like the example above) specifically when authorization decisions depend on **relationships between the user and the specific resource**, not just 'what role does this user generically have.' For anything beyond a handful of these context-dependent rules, I'd also consider a dedicated policy engine (OPA/Open Policy Agent) rather than letting complex authorization logic sprawl across scattered `@PreAuthorize` SpEL expressions — those get genuinely hard to audit and test once they grow past a certain complexity."

---

## SECTION 5: SESSION-BASED vs TOKEN-BASED AUTHENTICATION

### Q5. Session-based vs token-based authentication — compare them, and specifically explain why token-based fits microservices better.
**Answer:**
| Aspect | Session-based | Token-based (JWT) |
|---|---|---|
| Where state lives | **Server-side** — session store (in-memory, Redis) holds the actual session data; client just holds an opaque session ID cookie | **Client-side** — the token itself carries the claims; server doesn't need to store anything to validate it |
| Scalability across instances | Requires a **shared session store** (sticky sessions or Redis-backed session replication) — a raw in-memory session breaks the moment a request lands on a different instance | Naturally stateless — **any** instance can validate the token independently, no shared store needed for validation itself |
| Revocation | Instant — delete the session server-side, immediately invalidated | Not immediate (Q2) — valid until expiry unless extra infrastructure (denylist) is added |
| Cross-service auth | Awkward — every service would need access to the shared session store | Natural fit — the token itself is passed along and independently verified by each service (Q6) |
| CSRF exposure | Yes, since it's cookie-based (MOD8_001 Q2) | Generally no, if carried in an `Authorization` header rather than a cookie |

**Why token-based genuinely fits microservices better — the core architectural reasoning:** "In a monolith, a shared in-process session (or even a single shared session store) is a perfectly reasonable choice. In microservices, session-based auth means **every service** needs access to that shared session store just to answer 'is this request even authenticated' — that's a hidden, unwanted coupling exactly like the shared-database problem (MOD4_003 Q3), just for auth state instead of business data. A signed JWT lets **every service independently verify** the token's signature and read its claims **without any network call back to an auth service or shared store at all** — which is exactly the kind of decoupling microservices are meant to achieve (MOD4_001 Q4)."

**Senior-level nuance on revocation, since it's the natural pushback:** "The trade-off I'm explicit about: token-based auth trades **instant revocation** for **decoupled, stateless validation** — I mitigate that with short-lived access tokens plus a refresh-token flow (Q2), which bounds exposure without reintroducing a shared-state dependency on every single request. For genuinely critical, must-revoke-instantly scenarios (a confirmed account compromise), a lightweight server-side denylist checked only for that edge case is a reasonable, deliberately-scoped compromise — not a reason to abandon stateless tokens as the default."

---

## SECTION 6: SERVICE-TO-SERVICE AUTHENTICATION & mTLS

### Q6. How do services authenticate to each other internally — walk through the main approaches, and where mTLS fits.
**Answer:** User-facing auth (JWT/OAuth2, Q2-Q3) answers "which human is making this request"; **service-to-service authentication** answers a different question — "is this internal service call actually coming from a legitimate service, and which one." Several approaches, often layered together:

**1. Token propagation / pass-through** — the user's original token (or a derived internal token) is forwarded along the call chain so downstream services know both **which service** is calling and **on behalf of which user**.
```java
// Downstream call forwards the caller's identity — services can enforce "this user, via this call chain"
restTemplate.getForObject(inventoryServiceUrl, StockResponse.class,
    Map.of("Authorization", "Bearer " + originalToken));
```

**2. Service-specific credentials (Client Credentials OAuth2 flow)** — for genuine machine-to-machine calls with no user context at all, a service authenticates using its **own** client ID/secret to obtain a token representing *the service itself*, not any user.
```java
// Client Credentials grant — service authenticates AS ITSELF, no user involved
spring:
  security:
    oauth2:
      client:
        registration:
          inventory-service-client:
            client-id: order-service
            client-secret: ${ORDER_SERVICE_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: inventory.read
```

**3. mTLS** — as covered in depth in MOD7_007 Q3, mutual TLS authenticates **both** sides of a connection at the **network/transport layer**, independent of any application-level token at all — every service proves its identity via a certificate (typically tied to its Kubernetes service account identity), issued and rotated automatically by a service mesh's control plane.

**How these actually layer together in a real system — the answer that shows real depth:** "I think of it as **two independent layers that compose**: mTLS operates at the transport layer, answering 'is this literally the `payment-service` workload, cryptographically proven, and is this connection encrypted' — completely orthogonal to any user identity. JWT/OAuth2 token propagation operates at the application layer, answering 'which user's request is this, and did they have permission to trigger this call chain.' A request in a mature mesh-based architecture typically has **both**: the sidecar-to-sidecar hop is mTLS-authenticated and encrypted regardless of what's inside, and the payload still carries the propagated user JWT for the receiving service's own business-level authorization checks. Relying on mTLS alone would mean any service inside the mesh's trust boundary could act as any user by simply forwarding a stale or forged claim — mTLS proves the caller's **service identity**, not the **end-user's** identity or permissions; you still need both layers doing their own job."

**Follow-up: "Why not just trust internal network traffic without mTLS at all, since it's 'inside our own cluster'?"** → This is the "trusted internal network" assumption MOD7_007 Q3 already pushes back on — zero-trust security posture assumes a compromised pod/node is a realistic threat model, not a hypothetical; mTLS ensures a compromised or malicious workload inside the cluster **still can't** impersonate another service's identity or read another service's traffic just by being on the same internal network.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can you use both session-based and token-based auth in the same system?" → Yes — a common pattern is session-based auth for the browser-facing web app (simpler, built-in CSRF protection) and token-based (JWT/OAuth2 Client Credentials) for the internal service-to-service and public API surface — they're not mutually exclusive architectural choices.
- "What's the difference between an access token and a refresh token?" → An access token is short-lived and sent with every API request to prove authorization; a refresh token is longer-lived, sent only to the auth server to obtain a **new** access token without re-prompting the user to log in — refresh tokens should be stored more securely and checked against revocation, since compromising one has a much longer exposure window.
- "Is Basic Auth (username:password in a header) ever appropriate?" → Rarely for production user-facing APIs — credentials are sent (base64-encoded, not encrypted) on **every single request**, meaning any leak exposes the actual password, not just a scoped, expiring token; it still shows up for simple internal service-to-service calls over mTLS-secured internal networks, or basic API testing, but isn't the standard for anything user-facing.
- "What does 'stateless authentication doesn't mean stateless authorization' mean?" → Even with a fully stateless JWT for authentication, authorization decisions (Q4) can still require a database/cache lookup — e.g., checking a user's current department or approval limit, which may have changed since the token was issued — the token proves identity, it doesn't necessarily carry every current, potentially-changing permission attribute.
- "How do you handle authorization consistently across many microservices without duplicating logic in every service?" → Common approaches: a shared library encapsulating common authorization checks (a form of shared kernel, MOD4_002 Q10, used deliberately), or a centralized policy engine (OPA) queried by each service, or coarse authorization enforced at the gateway (MOD4_005 Q2) with fine-grained, resource-specific checks still done in the owning service — a genuine architectural decision, not a one-size-fits-all answer.
- "What's token introspection, and when do you need it instead of just validating a JWT locally?" → An endpoint on the authorization server a resource server calls to check if a token is still valid/unrevoked in real time — needed for **opaque tokens** (not self-contained JWTs) or when you need guaranteed real-time revocation awareness that local JWT signature validation alone can't provide, at the cost of a network call per validation.

---

*Study tip: The OAuth2-vs-OIDC distinction (Q3) and correctly explaining why token-based auth fits microservices architecturally, not just "it's more modern" (Q5), are the two areas where interviewers most reliably separate candidates who've configured Spring Security once from those who actually understand the distributed-systems reasoning behind the choice. Being able to layer mTLS (transport-level service identity) and JWT (application-level user identity) as complementary, not competing, controls (Q6) is a strong, specific senior signal.*