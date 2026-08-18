# Deployment Strategies — Interview Prep
---

## SECTION 1: ROLLING DEPLOYMENT

### Q1. What is a Rolling Deployment, and what are its actual trade-offs compared to just replacing everything at once?
**Answer:** A Rolling Deployment incrementally replaces old-version instances with new-version instances, a few at a time, rather than taking the whole fleet down and back up together — this is Kubernetes' **default** Deployment strategy (MOD7_003 Q4), controlled via `maxSurge`/`maxUnavailable`.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # up to 2 extra pods beyond desired count during rollout
      maxUnavailable: 1    # at most 1 pod below desired count unavailable at a time
```

| Pros | Cons |
|---|---|
| No extra infrastructure cost — reuses existing capacity | **Two versions run simultaneously** during rollout — old and new code both serving live traffic at once |
| Built into Kubernetes natively, no extra tooling | Rollback isn't instant — it's another rolling update in reverse, taking real time |
| Gradual — a bad readiness probe (MOD7_003 Q1) limits blast radius somewhat | Hard to test the new version against real traffic in isolation before it's already serving a portion of users |
| Simple to configure and reason about | If the new version has a subtle bug, some fraction of real users hit it before anyone notices |

**Senior-level answer:** "The trade-off I always flag explicitly: Rolling Deployment's biggest hidden cost is that **old and new versions coexist and both serve real traffic simultaneously** for the duration of the rollout — which means your API and database schema **must be backward/forward compatible during that window** (Q6), or you'll get real production errors from version-mismatch failures, not just a theoretical concern. It's the right default for most routine, low-risk deployments, but I wouldn't rely on it alone for a genuinely risky change — that's when Canary (Q3) or Blue-Green (Q2) earn their added complexity."

---

## SECTION 2: BLUE-GREEN DEPLOYMENT

### Q2. What is Blue-Green Deployment, how does it actually work in practice, and what does it cost you that Rolling doesn't?
**Answer:** Blue-Green maintains **two complete, independent environments** — "Blue" (currently live) and "Green" (the new version, fully deployed but not yet receiving production traffic). Once Green is verified healthy, traffic is **switched all at once** (via a load balancer/router/DNS change) from Blue to Green. Blue is kept running, idle, as an instant rollback target.

```
Before switch:
  Load Balancer → Blue (v1.0, LIVE) ✓
                   Green (v1.1, deployed, NOT receiving traffic — smoke-tested here)

After switch (instant, at the router level):
  Load Balancer → Green (v1.1, LIVE) ✓
                   Blue (v1.0, kept running, idle — instant rollback target)
```

```yaml
# Kubernetes Service selector flip — the simplest way to implement Blue-Green on K8s
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: green   # flip this single label from "blue" to "green" to cut traffic over
  ports:
    - port: 80
```

**Key trade-offs, stated directly:**
- **Rollback is genuinely instant** — flip the router back to Blue, no waiting for a reverse rolling update; this is Blue-Green's single biggest advantage over Rolling.
- **Costs roughly double the infrastructure** during the transition window — you're running two full production-sized environments simultaneously, even if briefly.
- **No gradual exposure** — the cutover is all-or-nothing; a subtle bug that only manifests under a slice of real traffic hits **100% of users** the instant you switch, unlike Canary's gradual exposure.
- **Database compatibility still matters** — even though app-level cutover is instant, if Green needs schema changes, both Blue and Green are pointed at the **same database** during the window Blue could still be rolled back to, so the same backward-compatibility discipline (Q6) applies regardless of strategy.

**Senior-level answer:** "I reach for Blue-Green specifically when **rollback speed** is the priority — a high-stakes release where I want a proven, instant escape hatch, not a rolling reversal that takes minutes. The cost is real (double infrastructure, momentarily) and it doesn't give you gradual, real-traffic validation the way Canary does — those are genuinely different guarantees, and picking between them should be a deliberate choice based on what you're actually optimizing for, not just 'which sounds more sophisticated.'"

---

## SECTION 3: CANARY DEPLOYMENT

### Q3. What is Canary Deployment, and how is it meaningfully different from both Rolling and Blue-Green?
**Answer:** Canary deploys the new version alongside the old one and **routes a small, controlled percentage of real production traffic** to it (e.g., 5%), monitors key metrics closely, and **gradually increases** that percentage as confidence grows — versus Rolling's uncontrolled/no traffic-percentage-awareness replacement, and versus Blue-Green's instant all-or-nothing cutover.

```yaml
# Istio VirtualService — traffic-split based canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts: ["order-service"]
  http:
    - route:
        - destination: { host: order-service, subset: v1 }
          weight: 95     # 95% of traffic still on the stable version
        - destination: { host: order-service, subset: v2 }
          weight: 5      # 5% canary — gradually increased: 5% → 25% → 50% → 100%
```

**What makes Canary genuinely different, not just "Rolling with extra steps":**
- **Deliberate, monitored, gradual traffic percentage** — not just "some pods are old, some are new" (Rolling's implicit, uncontrolled mix) but an explicit, controlled split you can hold, advance, or abort at will.
- **Real production traffic validates the release before full exposure** — catches issues that only manifest under real user behavior/data patterns, which staging environments often can't replicate.
- **Automated or manual promotion gate** between each traffic-percentage step, typically tied to specific health/business metrics (error rate, latency, even business metrics like checkout success — MOD6_001 Q6) — if the canary's metrics degrade, the rollout **halts or auto-reverts** before most users are ever affected.

**Senior-level answer, since interviewers often want the decision framework, not just definitions:** "I think of these three strategies on a spectrum of **risk mitigation vs. complexity/cost**: Rolling is the cheap default for routine, low-risk changes. Blue-Green buys you instant rollback for high-stakes releases where speed of reversal matters most. Canary buys you the **smallest possible blast radius** for genuinely risky changes — a new payment provider integration, a major algorithm change — where I want real production signal before committing to full exposure, and I'm willing to pay for the added tooling (service mesh or a canary-aware gateway) and the slower rollout to get it. I've used all three in the same organization, chosen per-change based on the actual risk profile of what's being deployed, not a single fixed house style."

---

## SECTION 4: FEATURE FLAGS

### Q4. How do feature flags relate to deployment strategy, and why would you use them alongside (not instead of) something like Canary or Blue-Green?
**Answer:** A feature flag **decouples deployment from release** — the code for a new feature can be deployed to production (fully rolled out at the infrastructure level) while remaining **dark/inactive** for users until a flag is explicitly flipped, independent of any deployment event. This is a fundamentally different lever than the deployment strategies above, which control *which version of the binary* serves traffic — feature flags control *which code paths are active* within a single deployed version.

```java
@Service
public class CheckoutService {
    private final FeatureFlagService flags;

    public CheckoutResult checkout(Cart cart, String userId) {
        if (flags.isEnabled("new-payment-provider", userId)) {
            return newPaymentProvider.process(cart);   // dark-launched, controlled independently of deployment
        }
        return legacyPaymentProvider.process(cart);
    }
}
```

**Why combine flags with a deployment strategy rather than picking one or the other:** "Deployment strategy controls **infrastructure risk** (is the new binary healthy, performant, not crash-looping); feature flags control **business/product risk** (is this specific feature actually working correctly for users, can we turn it off in one second without any redeploy at all if something looks wrong). I've used them together deliberately: deploy the new binary via Rolling or Canary (validating it's infrastructurally sound), with the risky new feature **behind a flag, defaulted off** — then separately, gradually enable the flag for an increasing percentage of users, completely decoupled from any further deployment. If the feature itself misbehaves, flipping the flag off is **instant** — no rollback, no redeploy, no waiting on Kubernetes rollout mechanics at all."

**Real trade-off to name proactively:** "Feature flags add genuine code complexity — conditional branches that live in the codebase, sometimes for a long time, and **stale flags** left in code long after a feature has fully shipped become their own form of tech debt and confusion. I push for a discipline of removing flags (and the old code path) once a feature is fully rolled out and stable — a flag is meant to be temporary risk-mitigation scaffolding, not a permanent architectural feature."

---

## SECTION 5: ZERO-DOWNTIME DEPLOYMENT

### Q5. What does "zero-downtime deployment" actually require end-to-end — not just "use Rolling Update" — walk through everything that has to be true?
**Answer:** Zero-downtime isn't a single feature you turn on — it's the **outcome of several things working correctly together**, and missing any one of them reintroduces downtime even with a "zero-downtime" deployment strategy nominally in place.

**What actually has to be true:**
1. **Accurate readiness probes** (MOD7_003 Q1) — traffic must only be routed to instances that are genuinely ready to serve it; a shallow or missing readiness check means new pods receive traffic before they're actually ready, producing real errors during rollout.
2. **Graceful shutdown, not abrupt termination** — when an old instance is being terminated, it needs to **finish in-flight requests** before actually dying, not be killed mid-request.
```java
@PreDestroy
public void gracefulShutdown() {
    // stop accepting NEW work, but let in-flight requests complete
}
```
```yaml
spec:
  terminationGracePeriodSeconds: 30   # give the pod time to finish in-flight work after SIGTERM before SIGKILL
```
"This is the piece I see most commonly missed — Kubernetes sends SIGTERM and starts routing traffic away from the terminating pod, but if the application doesn't handle SIGTERM gracefully (draining connections, finishing active requests), and `terminationGracePeriodSeconds` isn't tuned to the app's actual longest-typical request duration, in-flight requests get abruptly cut off mid-response during every single deployment — a very real, very common source of 'random' 5xx errors that only happen during deploys."

3. **Backward-compatible API and database changes** (Q6) — during any rolling-style deployment, old and new versions coexist briefly; if a change isn't compatible both ways, that window itself causes failures, independent of anything else being configured correctly.
4. **Connection draining at the load balancer level** — the LB/ingress needs to stop sending *new* connections to a terminating instance while still letting *existing* connections complete, working in concert with #2.
5. **Idempotent startup** — a new instance shouldn't do something destructive or stateful-and-non-idempotent purely by starting up (relevant for services that run migrations or bootstrap logic on startup — see Q6 for why this connects to migration ordering).

**Senior-level closing answer:** "'Zero-downtime' is really a claim about the **whole chain** — probe accuracy, graceful shutdown, LB connection draining, and API/schema compatibility all have to hold simultaneously. I've seen teams call a rollout 'zero-downtime' because they used a rolling strategy, without ever actually verifying graceful shutdown was implemented correctly — and then be genuinely surprised by a small, consistent trickle of errors on every single deploy that nobody had connected back to deployment at all, because it looked like generic background noise in the error rate rather than a deployment-specific pattern."

---

## SECTION 6: BACKWARD-COMPATIBLE DATABASE/API CHANGES

### Q6. Walk through how you'd actually make a breaking-looking database schema change (e.g., renaming a column) backward-compatible during a rolling deployment.
**Answer:** The core constraint: during a rolling/canary/blue-green transition window, **old and new application code may both be querying the same database simultaneously** — so the schema itself has to remain valid for both versions at once, for the duration of that window. The standard technique is the **expand-and-contract pattern**, breaking one "breaking" change into several individually-safe deployment steps.

**Concrete example — renaming a column `email` to `email_address`:**
```sql
-- Step 1 (deploy N): EXPAND — add the new column, keep the old one, backfill, keep both in sync
ALTER TABLE customers ADD COLUMN email_address VARCHAR(255);
UPDATE customers SET email_address = email;  -- backfill existing rows
-- App code at this point still reads/writes ONLY the old "email" column — no behavior change yet

-- Step 2 (deploy N+1): app code updated to WRITE to both columns, READ from the new one
--   (old app instances, if still running during rollout, still work fine — they only touch "email")

-- Step 3 (deploy N+2, after full rollout confirmed stable): stop writing to the old column entirely
--   app code now only touches "email_address"

-- Step 4 (deploy N+3, a separate later cleanup deploy, once fully confident): CONTRACT — drop the old column
ALTER TABLE customers DROP COLUMN email;
```

"The rule I apply: **never combine a schema change and the application code change that depends on it in the same deploy** — always expand the schema first (additive, harmless to old code), deploy the app change that uses it, confirm it's stable, and only then contract/remove the old structure in a later, separate deploy. This is exactly what avoids the failure mode where a rolling update briefly has old-version pods (still expecting the old column) running against a database that's already had it dropped."

**API compatibility follows the same additive-only discipline as event schema evolution (MOD5_007 Q10) and REST versioning (MOD4_004 Q6):** add new optional fields, never remove/rename/retype existing ones in the same release that also removes the code depending on the old shape — consumers (including your own old, still-running instances during a rollout) need a full deploy cycle to migrate off the old contract before it's safe to remove.

**Follow-up: "What about a change that genuinely can't be made additive — like a fundamental data type change?"** → Same expand-contract principle, just with more steps: add the new-typed column alongside the old, dual-write during a transition period, backfill and validate data integrity between the two, cut reads over once fully validated, then drop the old column in a later cleanup deploy — the pattern scales to more complex changes, it just takes more deploys to get there safely.

---

## SECTION 7: ROLLBACK STRATEGIES

### Q7. What are your actual rollback options, and how do you decide which one to use when something goes wrong in production?
**Answer:** Rollback options aren't one-size-fits-all — the right choice depends on **what kind of change actually broke**, and reaching for the wrong one wastes precious time during an active incident.

| Rollback method | When to use | Speed |
|---|---|---|
| **Feature flag off** | The problem is in a flagged feature, not the core binary | Instant — no deploy needed at all |
| **Traffic shift back (Canary/Blue-Green)** | Still mid-rollout, old version still running alongside new | Fast — just a routing change |
| **`kubectl rollout undo` / `helm rollback`** | Fully rolled out, need to revert the deployed version | Minutes — triggers a real rolling update in reverse |
| **Database rollback/restore** | Data corruption occurred, not just bad application code | Slow, risky — often the last resort, not the first |

**Senior-level decision framework:** "My first question during an incident is always: **is this a code problem or a data problem?** If it's a code problem, rollback (via whichever mechanism is fastest available for that deployment strategy) is usually safe and should happen immediately, without waiting for full root-cause analysis — restore service first, investigate after. If it's a **data** problem — the new version wrote bad data, or a migration corrupted something — rolling back the application code does **not** undo that; the data damage persists regardless of which binary is now running, and a naive app-only rollback can even make things worse if the old code doesn't expect the new (or now-corrupted) data shape. That's exactly why the expand-contract pattern (Q6) matters so much — schema changes that are correctly staged as backward-compatible mean an app-level rollback stays safe, because the old code was never actually broken by the new schema in the first place."

**A rollback readiness practice worth naming proactively:** "I treat 'can we roll back this specific change safely' as a question to answer **before** deploying, not during an incident — for any change touching the database, I explicitly verify the previous app version would still function correctly against the current (post-migration) schema. If the answer is no, that's a sign the change wasn't actually staged as backward-compatible, and it needs to be split using expand-contract before it ships, not rushed out with 'we'll just roll back if it breaks' as the safety net."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can you combine Canary and Blue-Green?" → Yes — some platforms implement "canary within Blue-Green," gradually shifting traffic percentage to Green rather than an instant 100% cutover, combining Blue-Green's clean environment separation with Canary's gradual, monitored exposure.
- "What's a 'dark launch'?" → Deploying and even exercising new code in production (sometimes shadowing real traffic to it without using the response) without exposing it to real users at all — closely related to feature flags (Q4), used to validate performance/behavior under real load before any user-facing risk.
- "Does Canary require a service mesh?" → Not strictly — Kubernetes-native tools (Argo Rollouts, Flagger) can implement traffic-percentage canary without a full service mesh, but a mesh (MOD4_003 Q13) makes fine-grained traffic splitting and automated metric-based promotion significantly easier to implement well.
- "How long should a canary stage sit at each traffic percentage before promoting?" → Long enough to capture a representative sample of traffic patterns and to observe key metrics stabilize — often minutes to hours depending on traffic volume; too short risks promoting before a rare-but-real issue surfaces, too long unnecessarily delays full rollout benefit.
- "What's the risk of feature-flag evaluation itself failing?" → If the flag service/SDK is unreachable, the application needs a sane, explicitly-chosen default behavior (fail-open vs fail-closed, decided per feature's risk profile) rather than crashing — a feature-flagging system becoming a single point of failure defeats its own safety purpose.
- "Is Blue-Green compatible with stateful services holding in-memory session data?" → It requires care — if Green doesn't share session state with Blue, users mid-session at cutover could be disrupted; typically solved by externalizing session state (Redis) so either environment can serve any user's session, independent of which one they originally connected to.

---

*Study tip: The expand-contract pattern for backward-compatible schema changes (Q6) is the single most consistently tested deep-dive in this module — have the exact multi-step example ready, not just the name of the pattern. Being able to state the decision framework for choosing Rolling vs Blue-Green vs Canary based on what's actually being optimized (cost vs rollback speed vs blast radius) — rather than reciting definitions — is what separates a senior answer here.*
