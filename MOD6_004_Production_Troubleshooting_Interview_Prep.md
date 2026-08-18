# Production Troubleshooting — Interview Prep
---

## SECTION 1: SUDDEN APPLICATION SLOWNESS

### Q1. Production suddenly gets slow — walk me through your first five minutes.
**Answer:** I'd work top-down from symptom to cause, resisting the urge to guess and jump straight to a fix:

1. **Confirm scope** — is it every endpoint or one? Every instance/pod or one? All users or a subset (one region, one client version)? This alone eliminates whole categories of cause.
2. **Check the RED-method dashboard** (Observability topic) — rate, error rate, latency — is this a genuine slowdown or a traffic spike overwhelming normal capacity?
3. **Check recent changes** — was there a deploy, a config change, a feature flag flip, or a scaling event in the last hour? Correlated timing is the single strongest early signal.
4. **Check dependency health** — DB, cache, downstream APIs, message queues — is the slowness actually originating downstream and just surfacing here?
5. **Pull a couple of slow traces** (if tracing is set up) to see exactly where time is going in the request path, rather than guessing.

**Senior-level answer:**
> "The instinct I actively resist in the first five minutes is jumping to a fix before I've localized the problem — restarting pods or scaling up can mask a genuine issue (a leak, a bad deploy) and burn the most valuable early diagnostic window. My actual first move is almost always 'what changed recently' cross-referenced with 'what does the dashboard say the shape of the problem is' — most production slowness incidents I've resolved traced back to a deploy or config change in the preceding hour, so that correlation check has the highest hit rate for the least effort."

---

### Q2. How do you distinguish application-level slowness from infrastructure or downstream-dependency slowness?
**Answer:**

| Signal | Points To |
|---|---|
| CPU/memory on the app's own pods is elevated | Application-level (GC pressure, CPU-bound code, Section 2) |
| App's own resource usage is normal, but latency is still high | Likely a downstream dependency (DB, cache, third-party API) |
| A specific span in a distributed trace dominates the total request time | That span's service/call is the actual bottleneck — look there, not at the entry service |
| Latency spikes correlate with a specific dependency's own dashboard alerts | Confirms the dependency, not your service, is the root cause |

**Senior-level answer:**
> "This is exactly what distributed tracing is for, and it's the fastest way to answer this question definitively rather than guessing from symptoms — a trace waterfall immediately shows whether 90% of a slow request's time was spent in your own service's code or in a call to Postgres or a third-party payment gateway. Without tracing, you're stuck cross-referencing timestamps across separate dashboards, which is slower and more error-prone — this is a good moment in an interview to make the case for why tracing investment pays for itself during exactly this kind of incident."

---

## SECTION 2: HIGH CPU / HIGH MEMORY & FREQUENT GC

### Q3. How do you diagnose sudden high CPU usage on a production JVM instance?
**Answer:**
1. **Identify the hot OS thread** — `top -H -p <pid>` shows per-thread CPU usage at the OS level; note the thread ID (in hex) with the highest usage.
2. **Map it to a Java thread** — take a thread dump (`jstack`/`jcmd`) and find the thread whose **native thread ID (nid)** matches the hex-converted OS thread ID from step 1.
3. **Read that thread's stack trace** — it shows exactly which method is consuming the CPU.
4. **If it's not one obvious thread**, use a sampling profiler (async-profiler, or JFR's CPU profiling event) to get an aggregate flame graph across all threads over a window, rather than diagnosing one thread at a time.

```bash
top -H -p 12345                       # find hot native thread, e.g. nid 0x1a2b (PID column, in decimal — convert to hex)
jstack 12345 | grep -A 20 "nid=0x1a2b"   # find the matching Java thread and its stack
```

**Senior-level answer:**
> "The `top -H` + `jstack` combination is the classic manual technique and important to know cold, but in practice I reach for **async-profiler** or a continuous JFR recording first when the cause isn't obvious from a single hot thread — a flame graph aggregated across many samples over 30-60 seconds tells you the actual hot code path far more reliably than eyeballing one thread dump, especially when the CPU load is spread across many threads doing the same expensive operation rather than concentrated in one."

---

### Q4. Frequent GC pauses are showing up in production — how do you confirm GC is actually the cause of the symptoms, and what do you check next?
**Answer:**
1. **Correlate GC log timestamps with the latency/error spike timestamps** — if p99 latency spikes line up exactly with full GC pauses, that's strong confirmation; if they don't line up, GC is a red herring and the real cause is elsewhere.
2. **Check the GC log's reclaim-per-cycle** (JVM & Application Performance topic, Q4) — GCs that reclaim very little relative to heap size point to genuine memory pressure or a leak, not just normal churn.
3. **Rule out a recent deploy** introducing a new allocation-heavy code path (e.g., a new feature building large in-memory collections per request) before assuming it's a slow-building leak — a sudden GC change is often deploy-correlated, not gradual.

**Senior-level answer:**
> "I treat 'frequent GC' as a symptom that needs its own root-cause chain, not a diagnosis in itself — 'frequent GC' could mean a genuine leak (Q9–Q10 of the JVM topic), it could mean a recent deploy introduced heavier per-request allocation, or it could simply mean traffic genuinely increased and the heap is undersized for the new baseline load. I'd pull up heap usage trend, correlate with the deploy timeline, and only then decide whether the fix is a code fix, a heap-sizing change, or genuinely just scaling up — jumping straight to '`-Xmx` needs to be bigger' without that triage is a common shortcut that sometimes just delays the real incident."

---

## SECTION 3: DATABASE DEADLOCK & SATURATION

### Q5. The database reports a deadlock in production — how do you diagnose and resolve it live?
**Answer:**
1. **Most DBs auto-resolve deadlocks** by killing one of the two transactions (the "victim") and returning a deadlock error to it — so the immediate live impact is usually a failed transaction/exception in the app, not a hung database.
2. **Check the DB's own deadlock log** (Postgres logs deadlock details if `log_lock_waits`/`deadlock_timeout` is configured; MySQL's `SHOW ENGINE INNODB STATUS` shows the last detected deadlock) — this shows **which two queries/transactions** and **which two resources (rows/tables)** were involved.
3. **Identify the lock-acquisition order** in application code for both code paths involved — deadlocks between two transactions almost always come down to acquiring the same two resources in **opposite order** (the same root cause as the JVM-level deadlock pattern, just at the DB-transaction level instead of the in-process-lock level).
4. **Fix**: enforce a consistent locking/update order across all code paths touching the same resources (e.g., always update rows in ascending primary-key order within a transaction), or reduce transaction scope/duration to shrink the window where the conflict can occur.

**Senior-level answer:**
> "This is conceptually the exact same problem and fix as the in-process JVM deadlock from the JVM Performance topic — two 'threads' (here, DB transactions) acquiring two locks in opposite order — just manifesting at the database-transaction level instead of the `synchronized`-block level. I'd make that connection explicitly in an interview, because it shows the underlying pattern-recognition rather than treating DB deadlocks and code deadlocks as unrelated topics to memorize separately."

---

### Q6. The database connection pool is saturated in production and requests are timing out — how do you triage this live?
**Answer:**
1. **Confirm it's saturation, not a slow DB** — check `hikaricp_connections_pending` / `hikaricp_connections_active` (Monitoring topic) — pending > 0 and active == max means every connection is in use and requests are queuing for one.
2. **Check for a connection leak** (JVM Performance topic, Q16) — has active connection count crept up over hours without dropping back to baseline between traffic peaks? That's a leak, not just load.
3. **Check for long-running/blocked queries holding connections** — a single slow or lock-blocked query can hold a connection for far longer than normal, starving the pool for everyone else, even if traffic itself hasn't increased.
4. **Immediate mitigation vs. root cause** — restarting the app releases leaked connections as an immediate mitigation, but that's not a fix; the actual root cause (missing `try-with-resources`, or a genuinely slow query) needs to be found and fixed, or it recurs.

**Senior-level answer:**
> "I'd explicitly separate the **mitigation** from the **fix** when explaining this in an interview — restarting the affected instances to clear leaked connections stops the bleeding and buys time, but if I stop there without finding the actual leaking code path or the query that's holding connections too long, the same incident recurs on the same timeline as before. A good incident response includes both the immediate action and an explicit follow-up owner/ticket for the root cause, not just the mitigation."

---

## SECTION 4: KAFKA CONSUMER LAG & RABBITMQ BACKLOG

### Q7. Kafka consumer lag is growing — what are the possible causes, and how do you diagnose which one it is?
**Answer:**

| Cause | How to Confirm |
|---|---|
| **Consumer is too slow** (slow per-message processing, e.g., a slow DB call per message) | Check consumer processing time per message/batch metrics; CPU/thread dump on the consumer if processing looks stuck |
| **Producer throughput increased** beyond consumer capacity | Compare produce rate vs. consume rate over the same window — if produce rate spiked and lag started climbing at the same time, this is it |
| **Consumer group has fewer active consumers than partitions** (e.g., a consumer crashed/rebalanced out) | Check consumer group's active member count vs. partition count via `kafka-consumer-groups.sh --describe` |
| **Rebalancing storms** (consumers repeatedly joining/leaving, e.g., due to overly aggressive `max.poll.interval.ms` timeouts) | Check consumer group rebalance events in logs — frequent rebalances mean consumers keep getting evicted mid-processing |

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processing-group
# Shows CURRENT-OFFSET, LOG-END-OFFSET, and LAG per partition — LAG is the number of unconsumed messages
```

**Senior-level answer:**
> "The fix depends entirely on which cause it is, so I wouldn't jump straight to 'add more consumers' — if the bottleneck is genuinely slow per-message processing (e.g., a synchronous call to a slow downstream API inside the consumer loop), adding consumers just moves the bottleneck to whatever that downstream dependency is, or overwhelms it further. I'd check the per-message processing time metric first — if it's high, the fix is optimizing or parallelizing the processing logic itself (or moving slow work to an async follow-up step) rather than just scaling consumer count, which only helps if there's spare partition parallelism to actually use."

---

### Q8. A RabbitMQ queue has a growing backlog — how do you diagnose and remediate it?
**Answer:**
1. **Check consume rate vs. publish rate** in the RabbitMQ management UI — a backlog means publish rate is exceeding consume rate, and you need to determine why on the consumer side (same diagnostic angle as Kafka lag, Q7).
2. **Check for a stuck/crashed consumer** — an unacknowledged message with `prefetch` set high can cause one slow/stuck consumer to hold a large batch of messages without processing them, while appearing "connected" in the UI.
3. **Check `prefetch count`** — a high prefetch count means one slow consumer can accumulate a large unprocessed backlog locally instead of leaving messages available for other consumers to pick up; lowering prefetch improves fairness under partial-consumer-slowness scenarios.
4. **Dead-letter queue (DLQ) growth** — if messages are being rejected/nacked repeatedly (e.g., a poison message causing a processing exception every time), check the DLQ — a backlog might actually be a small number of malformed messages blocking a queue, not a genuine throughput problem.

**Senior-level answer:**
> "A detail worth raising proactively: a 'backlog' and a 'stuck queue because of a poison message' can look identical from a high-level dashboard (queue depth climbing) but have completely different fixes — one is a scaling/throughput problem, the other is a data/error-handling problem where the same broken message is being redelivered and failing in a loop. Checking whether the **same message ID** keeps reappearing in logs, versus a genuinely diverse stream of new messages backing up, is the fastest way to tell them apart, and I'd want a DLQ with alerting configured specifically so poison-message scenarios surface immediately rather than looking like a generic backlog."

---

## SECTION 5: REDIS OUTAGE / CACHE FAILURE

### Q9. Redis goes down in production — what happens to the application, and how should it be designed to handle this gracefully?
**Answer:** What happens depends entirely on how the application was designed to treat the cache — this is the crux of the question:

- **Cache-aside without a fallback** — every cache miss (now: every request, since Redis is down) falls through to the database; if the app wasn't designed for this, the sudden 100% cache-miss rate can overwhelm the database that was previously shielded by the cache — this is the classic **cache stampede/thundering herd onto the DB**.
- **Well-designed fallback** — the app catches the Redis connection failure explicitly, logs/alerts on it, and **falls through to the DB with appropriate safeguards** (e.g., a circuit breaker on Redis itself, request coalescing to avoid duplicate DB hits for the same key, possibly a short-lived in-process fallback cache).

```java
public Optional<Product> getProduct(String id) {
    try {
        String cached = redisTemplate.opsForValue().get("product:" + id);
        if (cached != null) return Optional.of(deserialize(cached));
    } catch (RedisConnectionFailureException e) {
        log.warn("Redis unavailable, falling back to DB for product {}", id);
        meterRegistry.counter("cache.fallback", "reason", "redis_down").increment();
    }
    return productRepository.findById(id);  // DB fallback — but see stampede risk below
}
```

**Senior-level answer:**
> "The failure mode I'd specifically flag is the **thundering herd onto the database** the moment Redis goes down — if the cache was absorbing, say, 95% of read traffic, its sudden absence means the DB instantly receives ~20x its normal load, which can take down the database too, turning a cache outage into a full-system outage. Mitigations I'd design for ahead of time: a circuit breaker around Redis calls so a down cache fails fast instead of adding connection-timeout latency to every request, and ideally a **short TTL local/in-memory fallback cache** or request coalescing so concurrent requests for the same key don't all independently hit the DB during the outage window."

---

### Q10. Beyond a full outage, what are common Redis-specific production issues, and how do you diagnose each?
**Answer:**

| Issue | Symptom | Diagnosis |
|---|---|---|
| **Eviction due to memory pressure** | Cache hit rate suddenly drops; `evicted_keys` metric climbing | `INFO memory` and `INFO stats` — check `used_memory` vs `maxmemory`, and the configured eviction policy (`allkeys-lru`, etc.) |
| **Hot key** | One Redis shard/instance has much higher CPU/latency than others in a cluster | `redis-cli --hotkeys` (with `maxmemory-policy` LFU) or monitoring per-key access patterns — a single very-frequently-accessed key can bottleneck one node |
| **Replication lag** | Reads from a replica return stale data | `INFO replication` shows `master_repl_offset` vs the replica's offset — a growing gap indicates lag |
| **Blocking slow commands** | Latency spikes across all clients momentarily | `redis-cli --latency` / slow log (`SLOWLOG GET`) — commands like `KEYS *` or large `SORT`/`SMEMBERS` on huge collections block Redis's single-threaded command loop |

**Senior-level answer:**
> "The 'blocking slow command' case is the one I'd specifically call out, because it's a subtle trap — Redis is fundamentally **single-threaded for command execution**, so one expensive command (`KEYS *` on a large keyspace, or a large `SORT`) blocks **every other client's requests** for its duration, not just the caller who issued it. I've seen a well-intentioned debugging `KEYS pattern*` run against production cause a brief but real latency spike across an entire service fleet — `SCAN` (which is cursor-based and non-blocking) is the correct replacement for `KEYS` in any production context, and that's a good concrete example to have ready."

---

## SECTION 6: THIRD-PARTY API FAILURE

### Q11. A critical third-party API (e.g., a payment gateway) starts failing — how do you design the system to degrade gracefully rather than cascade?
**Answer:** The standard resilience toolkit (Network Failures topic) applied specifically to third-party dependencies:

1. **Circuit breaker** around the third-party call — stop hammering an already-failing external API, fail fast instead.
2. **Bulkhead isolation** — give the third-party call its own dedicated thread pool/connection pool, separate from the pool used for other operations, so this one dependency failing can't exhaust threads needed for unrelated functionality.
3. **Fallback behavior**, defined per use case — e.g., queue the payment request for retry later and tell the user "processing," rather than a hard failure, if the business can tolerate asynchronous settlement; or fail the request cleanly with a clear error if it genuinely can't proceed without the third party.
4. **Timeout tuned specifically to that dependency's own SLA**, not a generic default — a slow but technically "up" third-party API without a tight timeout can tie up your own resources for far longer than acceptable.

```java
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "queueForRetry")
@Bulkhead(name = "paymentGateway")
public PaymentResult chargeCard(ChargeRequest request) {
    return paymentGatewayClient.charge(request);
}

private PaymentResult queueForRetry(ChargeRequest request, Exception e) {
    retryQueue.enqueue(request);
    return PaymentResult.pending("Payment is processing, we'll confirm shortly");
}
```

**Senior-level answer:**
> "The bulkhead piece is the one people forget relative to circuit breakers — without it, a slow-but-not-yet-tripped-breaker third-party call can exhaust the **shared** thread pool that other, unrelated request types also depend on, so a payment gateway degradation ends up taking down completely unrelated functionality like browsing the product catalog, purely because they shared a thread pool. Isolating dependencies into their own bounded resource pools is what actually contains the blast radius of a third-party failure to the feature that depends on it."

---

### Q12. How do you detect and handle a third-party API that's *slow* but not fully down — the partial degradation case?
**Answer:** Partial degradation is arguably harder than a clean outage, because standard health checks/status pages often still show "up."

1. **Monitor your own client-side latency and error rate to that specific dependency** (Section 4/RED-method style, tagged per external dependency), not just its public status page — a third party's own status page frequently lags real user-facing degradation.
2. **Set an explicit timeout tuned to that dependency's normal p99**, not a generous default — without this, a "slow but technically responding" API behaves exactly like Q4's "long timeout" problem, tying up resources without ever triggering a circuit breaker.
3. **Use the circuit breaker's error-rate/slow-call-rate threshold, not just hard failures** — Resilience4j and similar libraries can trip a circuit breaker based on a **slow call rate** threshold, not only outright exceptions, specifically to catch this "technically succeeding but too slow" case.

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .slowCallDurationThreshold(Duration.ofSeconds(2))   // calls over 2s count as "slow"
    .slowCallRateThreshold(50)                            // trip if >=50% of calls are slow
    .build();
```

**Senior-level answer:**
> "I'd specifically flag that a naive circuit breaker configured only on hard failure rate will **not** trip during this scenario — the calls are 'succeeding,' just slowly — so without a slow-call-rate threshold configured explicitly, the system keeps sending traffic to a degraded dependency and every caller pays the latency cost individually. This is a real gap I've seen in circuit breaker configs that were copy-pasted from a tutorial with only the failure-rate threshold set and the slow-call settings left at defaults or unconfigured."

---

## SECTION 7: KUBERNETES CRASHLOOPBACKOFF / OOMKILLED

### Q13. A pod is stuck in `CrashLoopBackOff` — walk through your diagnosis steps.
**Answer:**
```bash
kubectl describe pod <pod-name>        # Events section often shows the direct cause (OOMKilled, failed probe, etc.)
kubectl logs <pod-name> --previous     # logs from the CRASHED container instance, not the current restart attempt
kubectl get pod <pod-name> -o yaml     # check resource limits, restart count, exit code
```

**Common causes, roughly in order of frequency:**
1. **Application crash on startup** — a bad config, missing environment variable, or an unhandled exception in initialization code — `--previous` logs almost always show this directly.
2. **Failing readiness/liveness probe** causing Kubernetes to kill and restart the container repeatedly, even if the app itself would eventually become healthy given more time (Observability topic, Q10) — check probe `initialDelaySeconds`/timeout configuration against the app's actual startup time.
3. **OOMKilled** (Q14) — exit code `137`, visible in `kubectl describe pod`'s last state.
4. **Dependency unavailable at startup** — e.g., the app fails fast if it can't reach the DB/config server on boot, rather than retrying with backoff — restarts in a loop until the dependency itself recovers.

**Senior-level answer:**
> "The single most useful command here is `kubectl logs --previous` — the default `kubectl logs` on a crash-looping pod often shows the **current** (freshly restarted, possibly not-yet-crashed) container's logs, which can be nearly empty and misleading; `--previous` gets you the actual crash's output. This trips up even experienced engineers under incident pressure, so it's worth having as an automatic first reflex rather than something to remember mid-incident."

---

### Q14. A pod is `OOMKilled` — how is this different from a JVM `OutOfMemoryError`, and how do you diagnose/fix it?
**Answer:** `OOMKilled` is the **Linux kernel's OOM killer** terminating the container because it exceeded its **container memory limit** — this is entirely separate from (and can happen even without) a JVM-level `OutOfMemoryError`. The JVM can be perfectly healthy from its own heap's perspective while the **total container memory** (heap + metaspace + thread stacks + off-heap/direct buffers + native memory) exceeds the Kubernetes-configured limit, and the kernel kills the whole process abruptly with **no Java-level exception or heap dump at all**.

```yaml
resources:
  requests:
    memory: "1Gi"
  limits:
    memory: "1.5Gi"    # if total JVM process memory exceeds this, the kernel SIGKILLs the container
```

**Diagnosis/fix:**
1. **Check if `-Xmx` is set close to or above the container limit** — the JVM heap is only *part* of total process memory; if `-Xmx` is set to 1.4Gi inside a 1.5Gi container limit, there's no headroom left for metaspace, thread stacks, or native memory, and an OOMKill is nearly inevitable under any load.
2. **Use `-XX:MaxRAMPercentage`** (JVM Performance topic, Q7) instead of a fixed `-Xmx`, so the JVM heap is sized as a **percentage** of the container's actual limit, deliberately leaving headroom for non-heap memory.
3. **If genuinely undersized, raise the container's memory limit** — but only after confirming it's not masking a leak (JVM Performance topic, Q9), the same "don't just add memory" caution as with a JVM heap OOM.

**Senior-level answer:**
> "The critical distinction I'd lead with: `OOMKilled` gives you **no heap dump, no stack trace, nothing** — it's a hard kernel-level kill, unlike a JVM `OutOfMemoryError` which at least gives you a catchable event and, with the right flag, a heap dump. This means the standard JVM memory-leak diagnostic playbook (Q9 of the JVM Performance topic) doesn't directly apply until you've first ensured the container isn't killing the process before the JVM even gets a chance to react — sizing the JVM's heap with real headroom below the container limit via `MaxRAMPercentage` is what makes the JVM's own OOM handling machinery actually usable in a Kubernetes environment, rather than being pre-empted by the kernel."

---

## SECTION 8: DEPLOYMENT FAILURE & ROLLBACK

### Q15. How do you detect a bad deployment quickly, before it impacts the majority of users?
**Answer:**
1. **Canary/progressive rollout** — deploy the new version to a small percentage of traffic/pods first, compare its error rate/latency against the stable baseline **automatically**, and only proceed to full rollout if the canary's metrics stay within an acceptable delta.
2. **Readiness probes gating traffic** — a new pod shouldn't receive traffic until it passes its readiness check (Observability topic, Q10), preventing a broken new version from serving requests before it's confirmed minimally functional.
3. **Automated rollback triggers** — tie deployment tooling directly to error-rate/latency alerts during the rollout window, so a spike **automatically** halts/reverts the rollout rather than waiting for a human to notice a dashboard.

```yaml
# Simplified canary analysis concept (e.g., Argo Rollouts / Flagger style)
analysis:
  metrics:
    - name: error-rate
      threshold: 1%          # if new version's error rate exceeds this vs baseline, abort rollout
    - name: p99-latency
      threshold: 500ms
  interval: 1m
  steps: [10%, 25%, 50%, 100%]  # progressive traffic shift, pausing for analysis at each step
```

**Senior-level answer:**
> "The maturity curve I'd describe: manual dashboard-watching after a deploy, then alerting during a deploy window, then true automated canary analysis that halts a bad rollout without a human needing to notice in time. The value of the automated version isn't just speed — it's that it removes the dependency on someone actively watching a dashboard at 2am when a scheduled or auto-triggered deploy goes out, which is exactly when a slow human reaction time causes the most damage."

---

### Q16. A deployment needs to be rolled back — what makes a rollback safe, and what complications commonly arise?
**Answer:**
- **Stateless rollback (code only)** — generally straightforward: redeploy the previous container image/version.
- **The real complication is almost always the database** — if the new version ran a **schema migration** that the old version's code isn't compatible with, rolling back the application code alone can break it against the now-migrated schema.

**The standard safe pattern: backward-compatible migrations, deployed in stages:**
1. **Expand** — add the new column/table **without removing or renaming anything old**; deploy this migration first, independently of the code change.
2. **Deploy the new application code** that uses the new schema, while the old columns/tables still exist untouched — this version is safely rollback-able, because rolling back just means running the *old* code against a schema that still has everything the old code needs.
3. **Contract** — only after the new code has been running successfully for a while (and a rollback is no longer a realistic concern), remove the old, now-unused columns/tables in a **separate**, later migration.

```sql
-- Expand: add new column, don't touch the old one yet
ALTER TABLE orders ADD COLUMN shipping_status VARCHAR(20);

-- (deploy new app code that reads/writes shipping_status, old code ignores it — both versions work)

-- Contract (later, separate deploy, only once rollback risk has passed):
ALTER TABLE orders DROP COLUMN legacy_status;
```

**Senior-level answer:**
> "This 'expand/contract' pattern is the single most important piece of practical knowledge in this area, and I'd bring it up unprompted — the naive assumption that 'rollback just means redeploying the old container image' quietly breaks the moment a schema migration is involved, and I've seen exactly this cause a rollback attempt to make an incident **worse**, not better, because the old code hit a schema it wasn't written for. Any migration that isn't backward-compatible with the currently-running previous version should be treated as a rollback risk and split into expand/contract stages rather than done as one atomic change alongside the code deploy."

---

## SECTION 9: ROOT CAUSE ANALYSIS

### Q17. Walk me through how you'd run a root cause analysis after a production incident.
**Answer:**
1. **Build an accurate timeline first**, before theorizing about cause — what was observed, and when, from monitoring/alerts/logs/deploy history — resist jumping to a root cause before the sequence of events is actually established.
2. **Distinguish the trigger from the root cause** — the trigger is often "a deploy went out" or "traffic spiked"; the root cause is *why* that trigger caused an incident (e.g., "the deploy introduced a query missing an index" or "the system had no backpressure/autoscaling for traffic spikes") — the trigger alone usually isn't the actionable finding.
3. **Use "5 Whys" (or similar) to go past the surface symptom** — e.g., "the service went down" → why? "OOMKilled" → why? "heap grew unbounded" → why? "a cache had no eviction policy" → why? "the TTL config was never set for this cache" → why? "no code review checklist item catches missing cache TTLs" — that last "why" is usually where the actually fixable, systemic issue lives.
4. **Identify contributing factors, not just the single root cause** — most real incidents have several contributing factors (a bug, a missing alert that would've caught it earlier, a runbook gap that slowed response) — a good RCA addresses the ones that matter, not just the first one found.
5. **Produce specific, owned, trackable action items** — "improve monitoring" is not an action item; "add an alert on cache eviction-policy-missing configuration, owned by X, due by Y" is.

**Senior-level answer:**
> "The discipline I care about most in an RCA is **blamelessness combined with specificity** — blameless in the sense that the goal is finding systemic gaps (missing alerting, missing safeguards, missing review checks), not finding a person to blame, because blame-focused RCAs make people hide information in the next incident instead of surfacing it. But blameless doesn't mean vague — a good RCA is specific enough that a stranger reading it six months later understands exactly what happened, why, and what concretely changed as a result, not just 'we've taken steps to prevent this in the future.'"

---

### Q18. What does a good incident postmortem document actually contain?
**Answer:**
- **Impact summary** — what broke, for how long, how many users/requests affected, in plain terms a non-engineer stakeholder can understand.
- **Timeline** — timestamped sequence of detection, escalation, key diagnostic findings, mitigation actions, and resolution — built from actual logs/alerts/chat history, not reconstructed from memory afterward.
- **Root cause** (and contributing factors) — the actual "why," from the 5-whys-style analysis (Q17), not just the immediate trigger.
- **What went well / what didn't** — including honest gaps in detection/response time, not just the technical root cause — e.g., "the relevant alert existed but wasn't tuned to fire fast enough" is a legitimate and useful finding.
- **Action items** — specific, owned, with due dates, and ideally tracked to actual completion in a follow-up review, not just written down and forgotten.

**Senior-level answer:**
> "The part most postmortems skip, and the part I think matters most: actually **following up** on whether the action items got done. A postmortem with great analysis but action items that quietly never got implemented just means the same incident recurs later — I've seen this happen literally with the same root cause hitting twice, a year apart, because the original fix was documented but never prioritized against other work. I'd want a lightweight recurring review (even just checking a tracker) to confirm postmortem action items actually close out, not just that the document was well-written."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the first command you'd run against a crash-looping pod?" → `kubectl describe pod` for the Events section, immediately followed by `kubectl logs --previous` for the actual crash output.
- "Is restarting the affected service ever the right first move?" → Sometimes — as an immediate mitigation to restore service while root cause investigation continues in parallel, but it should never be treated as the resolution on its own, and the root cause investigation shouldn't stop just because the symptom went away.
- "What's the difference between a canary deployment and a blue-green deployment?" → Canary shifts a small, gradually increasing percentage of live traffic to the new version with automated analysis at each step; blue-green runs both versions fully in parallel and switches all traffic at once, with the old version kept running for a fast full rollback if needed.
- "What's a poison message, and how do you stop it from blocking a queue?" → A message that consistently fails processing and gets redelivered/retried indefinitely; a dead-letter queue (DLQ) with a bounded retry count is the standard fix — after N failed attempts, route it to the DLQ instead of retrying forever.
- "Why check `kubectl describe pod` before logs?" → It surfaces the Kubernetes-level Events (OOMKilled, failed probe, image pull failure, scheduling issues) that explain *why* the container was killed/restarted — information that isn't in the application's own logs at all.
- "What's the risk of `KEYS *` in production Redis?" → Redis is single-threaded for command execution; `KEYS *` on a large keyspace blocks all other client requests for its duration — `SCAN` should be used instead, since it's cursor-based and non-blocking.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) demonstrating a **triage-before-fix discipline** — confirming scope and correlating with recent changes before touching anything (Q1, Q17) — rather than jumping straight to a remembered fix, (2) knowing the **expand/contract migration pattern** for safe rollbacks, since it's the single most common thing that makes an "easy" rollback actually break something (Q16), and (3) being able to walk through at least one of these scenarios (Redis stampede, Kafka lag, OOMKilled) **end-to-end with a specific real or plausible example**, not just the abstract theory — interviewers consistently probe for "tell me about a time" follow-ups on this exact topic, so have one or two real incident stories ready to adapt to whichever specific scenario comes up.*
