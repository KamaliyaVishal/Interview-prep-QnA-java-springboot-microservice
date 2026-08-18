# Production Security — Interview Prep
---

## SECTION 1: ZERO TRUST

### Q1. What is Zero Trust, and how is it fundamentally different from traditional "perimeter" security?
**Answer:** Traditional perimeter security operates on **"trust the network, verify at the edge"** — once a request/user/service is inside the corporate network or VPC, it's largely trusted by everything else inside that boundary, on the assumption the perimeter (firewall, VPN) already kept attackers out. **Zero Trust** flips this to **"never trust, always verify"** — every request is authenticated and authorized **on its own merits**, regardless of where it originates, including requests from *inside* the network.

| | Perimeter Security | Zero Trust |
|---|---|---|
| **Trust assumption** | Inside network = trusted; outside = untrusted | Nothing is trusted by default, inside or outside |
| **Verification point** | At the network edge (firewall, VPN gateway) | At every single request, every hop |
| **Lateral movement risk** | High — once inside, an attacker can often move freely between services | Low — every service-to-service call still requires its own authentication/authorization |
| **Typical mechanism** | VPN, network segmentation, IP allowlisting | mTLS, short-lived tokens, per-request identity verification (Q2) |

**Senior-level answer:**
> "The threat model Zero Trust actually addresses is 'what happens *after* the perimeter is breached' — because it always eventually is, whether through a phished credential, a compromised laptop, or a supply-chain vulnerability. Perimeter-only security means that one breach gives an attacker broad lateral access to everything inside; Zero Trust means a compromised service or credential still only grants access to exactly what that specific identity was authorized for, which is a dramatically smaller blast radius. I think of Zero Trust less as a specific product and more as a design principle that shows up in mTLS between services, short-lived tokens instead of long-lived API keys, and per-service IAM roles instead of one shared network-level trust boundary."

**Trap:** Don't describe Zero Trust as "just use a VPN and a firewall, but stricter" — that's still perimeter thinking. The defining characteristic is that trust is **never** implicit based on network location; even two services in the same VPC/subnet must still authenticate to each other explicitly.

---

### Q2. How would you implement Zero Trust principles for service-to-service communication inside a Kubernetes cluster, concretely?
**Answer:**
- **mTLS (mutual TLS) between every service** — not just client-to-server TLS, but **both sides** present and verify certificates, so a service can cryptographically prove its identity to every other service it talks to, and vice versa — typically via a service mesh (Istio, Linkerd) that handles certificate issuance/rotation automatically.
- **Per-service identity, not shared network trust** — each service gets its own identity (SPIFFE/SPIRE identities, or a service mesh's built-in identity system), and authorization policies are written per-identity, not per-IP-range or per-subnet.
- **Explicit network policies** — Kubernetes `NetworkPolicy` resources that **deny all traffic by default** and explicitly allowlist only the specific service-to-service paths that are actually required, rather than allowing any pod to reach any other pod by default.
- **Short-lived tokens for service auth** — service accounts using short-lived, auto-rotated tokens (Kubernetes projected service account tokens) rather than long-lived static credentials.

```yaml
# Kubernetes NetworkPolicy — deny-by-default, explicit allowlist
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-payment
spec:
  podSelector:
    matchLabels: { app: payment-service }
  ingress:
  - from:
    - podSelector:
        matchLabels: { app: order-service }   # ONLY order-service may reach payment-service
    ports:
    - port: 8443
```

**Senior-level answer:**
> "The 'default allow' networking that Kubernetes ships with out of the box is actually the opposite of Zero Trust — any pod can talk to any other pod unless you explicitly lock it down. Rolling out `default-deny` NetworkPolicies plus a service mesh for mTLS is the concrete implementation I've actually done, and the practical lesson is to roll it out incrementally with **monitoring/audit mode first** — a mesh like Istio can log what *would* be denied before you actually start enforcing, because a naive default-deny rollout in a live cluster with undocumented service dependencies will absolutely cause an outage from a legitimate path nobody remembered to allowlist."

---

## SECTION 2: LEAST PRIVILEGE

### Q3. What does "least privilege" actually mean in practice for a production system, beyond the textbook definition?
**Answer:** Least privilege means every identity — human user, service account, or application — is granted **only the specific permissions it needs to do its job, and nothing more**, and those permissions are **scoped as narrowly as possible** (specific resources, specific actions, often time-bounded), not broad "just in case" access.

**Where it actually shows up in practice:**
- **IAM roles scoped per-service**, not one shared "app-role" with broad access used by every microservice.
- **Database permissions scoped to exact tables/operations** — a service that only reads from `orders` shouldn't have `DELETE` or `DROP` permission, and definitely shouldn't have access to unrelated tables like `payments` if it never touches them.
- **Time-bounded elevated access** — a human engineer needing temporary production DB access for an incident gets a time-limited grant that auto-expires, not standing permanent access "for convenience."
- **No shared/generic service accounts** — each service has its own identity and its own scoped permission set, so a compromise of one service's credentials doesn't grant access to everything.

```json
// AWS IAM policy — scoped to exactly what this service needs, nothing broader
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::order-invoices-bucket/*"
  // NOT "s3:*" on "*" — no write/delete, no other buckets
}
```

**Senior-level answer:**
> "The failure mode I actually see most often isn't malicious over-permissioning — it's **convenience-driven scope creep**: a service gets `s3:*` on a bucket instead of just `GetObject` because it was faster to set up and nobody circled back to tighten it once things worked. Least privilege has to be actively maintained, not just applied once at setup — I push for periodic access reviews specifically to catch permissions that were granted for a since-removed feature or a one-time migration and were simply never revoked."

**Trap:** Don't conflate least privilege with Zero Trust as the same concept — they're complementary but distinct. Zero Trust is about **verifying identity on every request** regardless of network location; least privilege is about **how much that verified identity is allowed to do** once authenticated. A system can verify identity perfectly (Zero Trust) while still over-granting permissions (violating least privilege).

---

### Q4. How do you retrofit least privilege into an existing system where services already have broad, over-permissioned access — what's the actual process?
**Answer:** You can't safely narrow permissions blind — the practical process is:
1. **Observe actual usage** — use access logs (cloud provider access logs, IAM Access Analyzer-style tooling, database query logs) over a representative period to determine what permissions a service/role **actually exercises** in practice, versus what it's granted.
2. **Generate a scoped policy from observed usage** — many cloud providers offer tooling to auto-generate a minimal policy based on real access patterns (e.g., AWS IAM Access Analyzer's policy generation).
3. **Roll out in audit/monitor mode first** — apply the narrower policy in a "would this have been denied" logging mode before actually enforcing, to catch legitimate-but-infrequent access patterns (a monthly batch job, an annual reporting query) that a shorter observation window would have missed.
4. **Enforce, then monitor for denials** — once live, alert on any unexpected denial, since it either reveals a legitimate access pattern that was missed, or an attempted access that shouldn't be happening at all — both are useful signals.

**Senior-level answer:**
> "This is genuinely risky to do blind, because the honest truth is most teams don't have full confidence in what permissions a long-running service actually needs — there's always some quarterly job or rarely-hit code path that only becomes visible in a full year of access logs. I always insist on the audit-mode step before actual enforcement, specifically because the alternative — tightening permissions and finding out what broke in production — is a self-inflicted incident. The goal is to make the *rollout* safe, not just the end state."

---

## SECTION 3: API GATEWAY SECURITY

### Q5. What security responsibilities should live at the API Gateway layer versus in each individual service?
**Answer:**
| Belongs at the Gateway | Belongs in each service |
|---|---|
| **Authentication** — validating the token/credential is legitimate (signature check, expiry) | **Authorization** — deciding whether *this specific* authenticated identity can perform *this specific* action on *this specific* resource (the IDOR check from the security-auditing module) |
| **Rate limiting / DDoS protection** at the edge | Business-logic-specific rate limits (e.g., "max 3 password reset attempts per hour per account") |
| **TLS termination** | — |
| **Coarse-grained routing/access control** (e.g., only internal IPs can reach `/admin/*`) | Fine-grained, data-level access control (e.g., "can this user see *this specific* order") |
| **Request validation/schema enforcement** — rejecting malformed requests early | Deep business-rule validation |

**Senior-level answer:**
> "The mistake I actively watch for — and it's exactly the kind of gap DAST is good at catching (see the security-operations module) — is teams treating gateway-level authentication as if it were sufficient authorization, and skipping the ownership/resource-level check in the service itself. The gateway can correctly confirm 'this is a valid, authenticated user,' but only the service has the business context to answer 'is this user allowed to see *this specific* resource.' I treat the gateway as the first line of defense, never the only one — defense in depth means every layer does its own job, and services can never assume the gateway already handled authorization for them."

---

### Q6. What are the most important things to get right when securing an API Gateway itself, since it's a single point that sees all traffic?
**Answer:**
- **Fail closed, not open** — if the gateway's auth check fails to execute for any reason (a downstream identity provider timeout, a config error), the default behavior must be to **deny the request**, not silently let it through.
- **Consistent enforcement across every route** — a single misconfigured route that skips the standard auth filter (the exact scenario from the SAST/DAST module's DAST example) undermines the security of every other correctly-configured route, since attackers specifically probe for the one gap.
- **Rate limiting and WAF rules at the edge** — protecting backend services from being directly overwhelmed, and filtering known attack patterns (SQL injection strings, known bad payloads) before they ever reach application code.
- **No sensitive data in gateway logs** — the gateway sees every request, including headers/bodies that may contain tokens or PII; logging configuration needs the same discipline as application-level logging (Q7).
- **The gateway itself needs to be kept patched and monitored** — it's a high-value target precisely because it's a single chokepoint that sees all traffic.

```yaml
# Example: explicit route-level auth requirement, avoiding the "forgot to protect this route" gap
routes:
  - path: /api/orders/**
    filters: [RequireAuth]        # explicit, not inherited implicitly
  - path: /api/admin/**
    filters: [RequireAuth, RequireAdminRole]
  - path: /api/public/**
    filters: []                    # explicitly, deliberately public — not an oversight
```

**Senior-level answer:**
> "The 'fail closed' principle is the one I'd emphasize hardest, because the failure mode of fail-open is silent and severe — if your identity provider has a brief outage and the gateway is configured to let requests through when it can't reach it, you've just turned an availability blip into a full authentication bypass for the duration of the outage. I also make every route's auth requirement explicit in configuration rather than relying on a global default, specifically because an implicit 'everything requires auth unless configured otherwise' setup is one accidental route definition away from an unintentionally public admin endpoint."

---

## SECTION 4: SECURITY LOGGING & AUDITING

### Q7. What should — and should never — appear in production security logs?
**Answer:**
**Should be logged:**
- Authentication events — successful and **failed** login attempts, with enough context to spot brute-force patterns (source IP, timestamp, account targeted).
- Authorization failures — access-denied events, which are exactly the signal that reveals both misconfiguration and active attack probing.
- Administrative/privileged actions — who changed a permission, deleted a resource, accessed sensitive data, and when.
- Security-relevant configuration changes — IAM policy changes, firewall rule changes, secret rotations.

**Should never appear in logs, ever:**
- Passwords, API keys, tokens, session IDs — in plaintext, in full, or even partially in some compliance contexts.
- Full credit card numbers, SSNs, and other regulated PII — often subject to specific compliance requirements (PCI-DSS explicitly prohibits logging full card numbers).
- Full request/response bodies logged indiscriminately, since they routinely contain the above without a developer specifically intending to log them.

```java
// WRONG — logs the full Authorization header, leaking the bearer token into log aggregation
log.info("Incoming request: {} {}", request.getMethod(), request.getHeader("Authorization"));

// RIGHT — log what's needed for security investigation, nothing sensitive
log.info("Auth attempt: user={}, result={}, sourceIp={}",
    maskUsername(username), result, sourceIp);
```

**Senior-level answer:**
> "This is a genuinely easy category of bug to introduce accidentally — a debug log statement that dumps a full request object 'to help troubleshoot' during development, that then ships to production and quietly logs every user's bearer token into a log aggregation system that dozens of engineers have read access to. I push for **structured logging with an explicit allowlist of fields**, not blanket object serialization, specifically because an allowlist fails safe — a new sensitive field added to a request object later doesn't automatically start appearing in logs unless someone deliberately adds it."

**Trap:** Don't say "we mask passwords in logs, so we're fine" as if that solves the whole category — session tokens and API keys are just as sensitive as passwords and are the ones most commonly missed, since teams often specifically remember to scrub `password` fields but forget `Authorization` headers, `X-Api-Key` headers, and embedded tokens in URL query parameters (which also leak into web server access logs, not just application logs).

---

### Q8. How do security logs actually get used after something goes wrong — walk through what "good" audit logging enables during an incident?
**Answer:** Good audit logging answers the core investigative questions during an incident: **who did what, to which resource, when, and from where** — and critically, it needs to be **tamper-resistant and centralized**, because if an attacker has compromised a system, they may also try to delete or alter the logs on that same system to cover their tracks.

**What "good" looks like:**
- **Centralized, write-once log aggregation** (e.g., shipped immediately to a separate logging service/SIEM) — logs shouldn't only live on the compromised host, or an attacker with access can simply delete them.
- **Correlation IDs/trace IDs** threading through a request across every service it touches, so an investigator can reconstruct the full path of a specific suspicious request across a distributed system, not just what one service saw.
- **Immutable retention** for a defined period, meeting whatever compliance requirement applies (often 90 days to a year, sometimes longer for regulated industries).
- **Alerting on suspicious patterns proactively**, not just availability for after-the-fact investigation — e.g., an alert on repeated authorization failures from one account, or access to sensitive data outside normal patterns.

**Senior-level answer:**
> "The 'logs must be centralized and tamper-resistant' point is the one that separates logging-for-debugging from logging-for-security — if your only copy of an authentication log lives on the very host that gets compromised, an attacker with root access can just delete it, and now you have no record of how they got in. I've been part of an incident review where exactly this happened — application logs on the compromised host were incomplete because the attacker had cleared them, and the only useful trail was in the centralized log aggregator that had already ingested a copy before the host was touched. That's the entire argument for shipping logs off-host immediately rather than treating local logs as sufficient."

---

## SECTION 5: SECURITY INCIDENT HANDLING

### Q9. Walk through the phases of handling a production security incident, and what's different about the very first few minutes compared to the rest of the process?
**Answer:** Standard incident response phases: **Detect → Contain → Eradicate → Recover → Post-incident review.** The first few minutes are specifically about **containment, not full resolution** — the priority is stopping active harm from continuing, even with an imperfect or temporary fix, rather than immediately pursuing a complete, well-reviewed permanent solution.

```
Detect:    Alert fires (unusual access pattern, a reported vulnerability, anomalous traffic)
Contain:   Stop the bleeding NOW — revoke a compromised credential, block an IP/account,
           disable an affected feature — imperfect but fast
Eradicate: Remove the actual root cause — patch the vulnerability, remove attacker persistence
Recover:   Restore normal operation, verify systems are clean, re-enable disabled features
Review:    Blameless post-incident analysis — what happened, why didn't we catch it sooner,
           what specific gate would prevent recurrence
```

**Senior-level answer:**
> "The mental shift I emphasize for the first few minutes is that **containment doesn't have to be elegant** — if a specific API key is being actively abused, revoking it immediately even though it breaks a legitimate integration temporarily is the right call, and you fix the collateral damage afterward. I've seen incident response slowed down by people trying to find the 'proper' fix first instead of just stopping the damage — that instinct is backwards during active containment; you contain fast and imperfectly, then eradicate and recover properly once the bleeding has stopped."

---

### Q10. Who needs to be involved in a security incident beyond the engineering team, and why does that matter for how engineers should communicate during one?
**Answer:** Depending on severity, an incident often needs: **legal/compliance** (breach notification obligations — GDPR, CCPA, industry-specific regulations often have strict, short disclosure timelines), **communications/PR** (customer-facing messaging, avoiding uncoordinated public statements), **leadership** (business impact decisions, resourcing), and sometimes **law enforcement** (for certain categories of attack).

**Why this matters for engineer communication:**
- **Avoid speculation in written channels** (Slack, tickets) until facts are confirmed — an engineer's early guess ("looks like attacker accessed the whole user table") written in a persistent, potentially discoverable channel can become a legally significant statement if it turns out to be wrong, especially once legal/compliance teams are involved.
- **Stick to factual, timestamped findings** — what was observed, when, with evidence — rather than conclusions or blame, until the investigation is complete.
- **Route external-facing statements through the designated channel** (comms/legal), not ad hoc engineer responses to customer inquiries during an active incident.

**Senior-level answer:**
> "This is the part of incident handling that's easy to underrate as 'not really engineering,' but I've watched a technically well-handled incident get complicated by an engineer's early, unconfirmed Slack message speculating about scope that then had to be walked back once legal got involved — that message doesn't just disappear, it's part of the record. My practical rule during an active incident: document observed facts with timestamps as you go, keep interpretation and scope conclusions out of the log until confirmed, and let legal/comms own anything customer- or public-facing."

---

## SECTION 6: VULNERABILITY REMEDIATION

### Q11. How do you decide what to fix first when you have a backlog of known vulnerabilities across many systems — what's the actual prioritization framework?
**Answer:** CVSS score alone is a **starting point, not the answer** — real prioritization weighs several factors together:
- **Exploitability in your actual context** — is the vulnerable code path reachable, is there a known public exploit in active use, is the affected system internet-facing or internal-only.
- **Data/business sensitivity of the affected system** — a critical CVE in a system holding customer PII outranks the same CVSS score in an internal, low-value admin tool.
- **Compensating controls already in place** — a vulnerability behind a WAF rule that already blocks the known exploit pattern, or requiring an unlikely authentication prerequisite, is lower urgency than the same CVE with no mitigating factors.
- **Remediation cost/complexity vs. risk reduction** — sometimes the fastest path to real risk reduction is a config change or WAF rule, not the "correct" full fix, especially under time pressure.

| Factor | High priority | Lower priority |
|---|---|---|
| Exposure | Internet-facing | Internal-only, behind auth |
| Exploit availability | Public exploit known/active | Theoretical, no known exploit |
| Data sensitivity | Handles PII/payment data | No sensitive data |
| Compensating controls | None | WAF rule/network isolation already mitigates |

**Senior-level answer:**
> "I explicitly push back on prioritizing purely by CVSS score, because it's context-blind by design — a 9.8 CVSS vulnerability in an internal tool with no path to sensitive data and an existing compensating control can genuinely be lower real-world risk than a 6.5 in a public-facing payment endpoint with no other protections. The framework I actually use is closer to 'CVSS as the starting signal, then ask exposure, exploitability, data sensitivity, and existing mitigations' — and I document that reasoning, because 'we deprioritized a critical CVE' needs a clear, defensible justification on record, not just a gut call."

---

### Q12. What's the difference between a hotfix/patch and proper remediation, and when is a "temporary" mitigation actually the right call?
**Answer:** A **patch/hotfix** addresses the immediate vulnerable code or configuration — often the fastest path to closing the specific exploit. **Proper remediation** often means something broader: fixing the *class* of vulnerability (e.g., adding input validation as a standard practice, not just fixing the one injection point that was found), updating dependency management policy, or adding a new automated gate (a SAST rule, a CI check) so the same category of bug is caught earlier next time.

**When a temporary mitigation is the right call:** when a full, proper fix requires more time/testing than the risk can tolerate, and a faster, narrower mitigation (a WAF rule blocking the specific exploit pattern, disabling a non-critical feature, rate-limiting a specific endpoint) meaningfully reduces risk in the interim — **provided it's tracked with a deadline for the real fix**, not treated as a permanent substitute.

**Senior-level answer:**
> "The trap I actively watch for is a 'temporary' WAF rule or mitigation that quietly becomes permanent because the underlying ticket for the real fix loses priority once the immediate pressure is off — six months later nobody remembers it was meant to be temporary, and the actual vulnerable code is still sitting there, just currently unexploited because of a mitigation that could be bypassed or accidentally removed at any time. I insist temporary mitigations get a tracked ticket with an owner and a deadline, reviewed in the same post-incident process (Q9) that closes out the incident — 'we mitigated it' and 'we fixed it' are different states, and only the second one should let an incident actually close."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is Zero Trust only about network security, or does it apply to application design too?" → It applies broadly — the same "verify explicitly, never assume trust" principle applies to service-to-service auth, API design (never trust client-supplied data even from your own frontend), and human access to internal tools, not just network segmentation.
- "What's the difference between least privilege and role-based access control (RBAC)?" → RBAC is a *mechanism* for assigning permissions via roles; least privilege is the *principle* that should guide how narrowly those roles are scoped — you can implement RBAC badly (broad, generic roles) and still violate least privilege.
- "Should a WAF replace input validation in application code?" → No — a WAF is a compensating control/defense-in-depth layer, not a substitute for proper input validation and parameterized queries in the application itself; relying on the WAF alone means a WAF misconfiguration or bypass leaves the application fully exposed.
- "What's a 'blameless' post-incident review, and why does that framing matter?" → A review focused on identifying process/system gaps rather than individual fault — it matters because a blame-focused culture makes engineers less likely to report near-misses or their own mistakes quickly, which directly slows down detection and containment of future incidents.
- "How quickly should a critical, actively-exploited vulnerability be remediated versus a low-severity one?" → Actively-exploited critical vulnerabilities typically demand same-day containment (even if full remediation takes longer); low-severity, non-exploited findings are usually handled on a normal sprint/release cadence with a documented SLA by severity tier.
- "Does having a bug bounty program replace the need for internal security auditing?" → No — it's a complementary, external discovery channel, not a substitute for internal SAST/DAST, dependency scanning, and manual audits (see the security-operations module); relying solely on external researchers to find issues means your internal baseline security posture is unverified.

---

*Study tip: Being able to explain Zero Trust and least privilege as distinct-but-complementary principles rather than synonyms (Q1–Q3), walking through the "contain fast, imperfectly" instinct for the first minutes of an incident (Q9), and demonstrating a context-aware vulnerability prioritization framework beyond raw CVSS score (Q11) are the three areas where interviewers most reliably separate candidates who've actually run production security operations from candidates who know the terminology but haven't lived through the trade-offs.*
