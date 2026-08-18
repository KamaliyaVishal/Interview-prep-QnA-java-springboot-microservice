# Kubernetes Production — Interview Prep
---

## SECTION 1: LIVENESS, READINESS & STARTUP PROBES

### Q1. Explain Liveness, Readiness, and Startup probes — what does each one actually control, and what breaks if you misconfigure them?
**Answer:** All three are health checks Kubernetes runs against a container, but they control **different kubelet actions** — conflating them is one of the most common real production mistakes.

| Probe | Question it answers | Action on failure | Should check |
|---|---|---|---|
| **Liveness** | "Is this process alive, or hung/deadlocked?" | **Kills and restarts** the container | Only the process's own internal health — NOT downstream dependencies |
| **Readiness** | "Is this instance ready to receive traffic right now?" | **Removes pod from Service endpoints** (stops routing traffic to it) — does NOT restart it | Downstream dependency health (DB, broker, cache warm-up) |
| **Startup** | "Has this slow-starting app finished initializing yet?" | Suspends liveness/readiness checks until startup succeeds, then hands off to them | Whatever indicates initialization is complete (e.g., app context loaded) |

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3          # 3 consecutive failures → kill & restart

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
  failureThreshold: 3          # pulled from Service rotation, NOT restarted

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30         # allows up to 30 * periodSeconds for slow startup
  periodSeconds: 10
```

**What actually breaks when these are misconfigured — the real production failure mode:** "The single most common mistake I've seen is making the **liveness probe check downstream dependencies** (DB connectivity) instead of just the process's own health. When the database has a brief blip, every pod's liveness probe fails simultaneously, Kubernetes **restarts every instance at once** — which doesn't fix the DB outage at all, and now you've also lost your warm caches/connection pools and added restart churn on top of an already-degraded dependency. The correct design: liveness stays shallow (is the JVM itself responsive), readiness checks the dependency and simply pulls the pod from traffic until it recovers on its own — no restart needed, no compounding the incident."

**Startup probe's specific purpose, often missed:** "Before startup probes existed, a slow-starting Spring Boot app (heavy context initialization, cache warm-up) could get killed by the liveness probe **before it even finished starting**, if `initialDelaySeconds` wasn't generous enough — and a delay generous enough for slow startup made liveness far too slow to detect a genuine hang later. Startup probe solves that split responsibility cleanly: give the app as long as it needs to start, then hand off to the normal, tightly-tuned liveness check once it's actually running."

---

## SECTION 2: RESOURCE REQUESTS/LIMITS

### Q2. What's the difference between resource requests and limits, and what real problems come from getting them wrong?
**Answer:** **Requests** are what the scheduler **guarantees is reserved** for the container when deciding which node to place it on — the container is guaranteed at least this much. **Limits** are the **hard ceiling** the container is not allowed to exceed at runtime.

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"        # 0.25 CPU core, used for scheduling decisions
  limits:
    memory: "1Gi"       # hard ceiling — exceeding this = OOMKilled (Q6)
    cpu: "500m"          # hard ceiling — exceeding this = THROTTLED, not killed
```

**Key asymmetry that's a common interview trap:** CPU and memory behave **differently** when a limit is hit:
- **Exceed CPU limit** → the container is **throttled** (slowed down), not killed — it keeps running, just slower.
- **Exceed memory limit** → the container is **OOMKilled** (Q6) — memory can't be "throttled" the way CPU can, so the kernel kills the process outright.

**What goes wrong when requests/limits are set poorly:**
- **No requests set at all** → the scheduler has no idea how much to reserve, pods can be packed too densely onto a node, and under real load the node runs out of resources — the kubelet then starts **evicting** pods somewhat unpredictably to relieve pressure.
- **Requests set far too high ("just to be safe")** → wastes cluster capacity; the scheduler reserves resources that sit mostly idle, inflating infrastructure cost for no benefit.
- **Limits set far too low relative to actual usage** → frequent OOMKills or CPU throttling under normal, expected load — not even a spike, just business-as-usual traffic.
- **requests ≠ limits (a "burstable" QoS pod)** → fine for tolerating occasional spikes, but under **node-wide** resource pressure, pods running above their request (even if under their limit) are the **first to be evicted** — worth knowing for the QoS-class follow-up below.

**Senior-level answer:** "I set requests based on **observed steady-state usage** (from actual metrics, not guesses) and limits based on **observed peak usage plus headroom** — and I explicitly avoid the common shortcut of setting `requests == limits` for every workload by default. That gives you `Guaranteed` QoS (safest from eviction) but it also means you're reserving peak capacity permanently, even when actual usage is far lower most of the time — for a bursty, non-critical workload I'd rather accept `Burstable` QoS and the small eviction risk than pay for permanently-reserved peak capacity across the whole fleet."

**Follow-up: "What are Kubernetes QoS classes?"** → `Guaranteed` (requests == limits for every container, evicted last), `Burstable` (requests < limits, evicted before Guaranteed under pressure), `BestEffort` (no requests/limits set at all, evicted first) — this ties directly into eviction order during node resource pressure.

---

## SECTION 3: HORIZONTAL POD AUTOSCALER

### Q3. How does the Horizontal Pod Autoscaler (HPA) actually work, and what's the relationship between HPA and the resource requests you set in Q2?
**Answer:** HPA automatically adjusts the **number of pod replicas** based on observed metrics (CPU/memory utilization by default, or custom/external metrics) — it polls metrics periodically, compares against a target, and scales replica count up or down accordingly.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # scale up when average CPU usage exceeds 70% of REQUESTED cpu
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60     # wait 60s of sustained high usage before scaling up
    scaleDown:
      stabilizationWindowSeconds: 300    # wait 5 min before scaling down — avoids flapping
```

**The critical dependency most candidates miss:** "HPA's CPU-utilization percentage is calculated **relative to the pod's CPU *request*, not its limit**. If requests aren't set accurately (Q2), HPA's scaling decisions are built on a meaningless number — I've seen a service with CPU requests set far too high 'because we weren't sure,' which meant actual utilization never appeared to cross the 70% threshold even under real load, and HPA simply never scaled up when it should have. Getting resource requests right isn't just about scheduling — it's a **prerequisite** for autoscaling to work correctly at all."

**Follow-up: "Why the stabilization windows / different scale-up vs scale-down timing?"** → Scaling up should react reasonably fast to protect against real load spikes, but scaling down should be conservative — scaling down too aggressively right after a brief dip in traffic, only to immediately need to scale back up when traffic returns, causes **flapping** (repeated scale up/down cycles) which is disruptive (new pods need startup time, Q1) and wasteful; a longer scale-down stabilization window smooths that out.

**Follow-up: "Can HPA scale based on something other than CPU/memory?"** → Yes — **custom metrics** (e.g., Kafka consumer lag, queue depth) via the custom/external metrics API and something like KEDA (Kubernetes Event-Driven Autoscaling) — genuinely valuable when the real scaling signal is business/application-specific (e.g., "scale based on messages waiting in the queue," not CPU, since a message-processing service can be CPU-idle while still badly backlogged).

---

## SECTION 4: ROLLING UPDATES & ROLLBACKS

### Q4. Walk through how a Rolling Update actually works, and what controls you have to make it safe.
**Answer:** A Rolling Update incrementally replaces old-version pods with new-version pods, a few at a time, so the service **never goes fully down** during a deployment — Kubernetes brings up new pods, waits for them to become **Ready** (readiness probe, Q1), then terminates old pods, repeating until the rollout completes.

```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # up to 2 EXTRA pods beyond desired count during rollout (12 total temporarily)
      maxUnavailable: 1    # at most 1 pod below desired count can be unavailable at a time
```

**Why `maxUnavailable`/`maxSurge` matter, and how they trade off against each other:** "`maxSurge` controls how many *new* pods can be created ahead of terminating old ones — higher surge means faster rollout but temporarily higher resource usage. `maxUnavailable` controls how many pods can be *missing* from capacity at once — for a latency-sensitive service, I keep this low (often 0-1) so capacity never meaningfully drops during deployment, accepting a slightly slower rollout in exchange."

**This is exactly where readiness probes matter most:** "A rolling update's safety is **entirely dependent on an accurate readiness probe** — Kubernetes only routes traffic to a new pod, and only proceeds to terminate the next old pod, once the new pod reports Ready. If the readiness probe is too shallow (just checks the port is open, not that the app actually finished initializing and can serve real requests), Kubernetes will happily route live traffic to a pod that isn't actually ready to serve it correctly — I've seen a rollout produce a wave of 5xx errors exactly because readiness was too permissive and let traffic hit pods before their cache/connection pool had warmed up."

**Rollback:**
```bash
kubectl rollout status deployment/order-service        # watch progress
kubectl rollout undo deployment/order-service            # roll back to the previous revision
kubectl rollout undo deployment/order-service --to-revision=3   # roll back to a specific earlier revision
kubectl rollout history deployment/order-service          # see revision history
```

**Senior talking point:** "A rollback is just a rolling update in reverse — same mechanics, same readiness-probe dependency, applied to the previous ReplicaSet's pod spec, which Kubernetes keeps around specifically to make this fast. In practice I pair rolling updates with a readiness gate tied to real health, and for anything higher-risk, I'll do a **canary** (route a small percentage of traffic to the new version first, via a service mesh or a separate low-replica Deployment) rather than trusting the rolling update alone to catch a bad release before it's fully out."

---

## SECTION 5: POD SCHEDULING BASICS

### Q5. How does the Kubernetes scheduler decide which node to place a pod on, and what tools do you have to influence that decision?
**Answer:** The scheduler's core job: given a pod's resource requests (Q2) and any placement constraints, find a node with enough **available** capacity (based on requests, not current actual usage) and that satisfies all constraints, then binds the pod to it.

**The main placement controls, in order of how often they actually come up in interviews:**

**1. Node Affinity/Anti-Affinity** — prefer or require scheduling onto nodes matching certain labels (e.g., a specific instance type, or a specific availability zone).
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: node-type
              operator: In
              values: ["high-memory"]
```

**2. Pod Affinity/Anti-Affinity** — prefer or require scheduling relative to *other pods*, not nodes — the most commonly asked-about one, for high availability.
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: order-service
        topologyKey: "kubernetes.io/hostname"   # never schedule 2 replicas of THIS app on the SAME node
```
"This is the pattern I use to guarantee real high availability — without pod anti-affinity, the scheduler is free to place all 3 replicas of a critical service on the **same physical node**, and a single node failure takes down 100% of that service's capacity, defeating the whole point of running multiple replicas."

**3. Taints and Tolerations** — a **node** repels pods unless the pod explicitly "tolerates" that taint; used to reserve nodes for specific workloads (e.g., GPU nodes only for ML workloads, or keeping a dedicated node pool free of general-purpose pods).
```yaml
# On the node: kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"    # only pods with this toleration can be scheduled onto that tainted node
```

**Key distinction to state clearly, since it's a common confusion:** "Affinity is something the **pod** expresses ('I want/need to run near/away from X') — it's opt-in attraction. Taints/tolerations work the other way: the **node** repels pods by default, and only pods that explicitly tolerate the taint are even considered — it's opt-in exception to an exclusion. I use pod anti-affinity for HA spread across nodes/zones, and taints/tolerations for reserving specialized infrastructure for specific workloads — mixing them up is a common interview slip worth avoiding."

---

## SECTION 6: CRASHLOOPBACKOFF / OOMKILLED TROUBLESHOOTING

### Q6. A pod is stuck in CrashLoopBackOff — walk me through exactly how you'd troubleshoot it, live, in production.
**Answer:** CrashLoopBackOff means the container keeps **starting, crashing, and Kubernetes is backing off** before retrying (exponential backoff, up to a cap) — it's a symptom, not a root cause; the actual reason could be anything from a bad config to an unhandled startup exception to a failed liveness probe.

**My actual troubleshooting sequence:**
```bash
kubectl get pods                                      # confirm the CrashLoopBackOff status, restart count
kubectl describe pod <pod-name>                        # Events section often shows the SPECIFIC reason
kubectl logs <pod-name>                                 # logs from the CURRENT (crashed) attempt
kubectl logs <pod-name> --previous                       # logs from the PREVIOUS crashed attempt — often more useful
kubectl get events --sort-by='.lastTimestamp'             # cluster-wide events for more context
```

**What `describe pod`'s Events section typically reveals, and what each points to:**
- `Back-off restarting failed container` + exit code **1** → application-level exception, usually visible in `logs --previous` (bad config, failed DB connection at startup, unhandled exception in a `@PostConstruct`/startup hook).
- Exit code **137** → the container was **killed** (SIGKILL) — almost always either **OOMKilled** (see below) or a failed liveness probe killing it before it could finish starting.
- `Liveness probe failed` repeatedly in Events → the probe itself may be misconfigured (checking too early, wrong port/path, or checking a downstream dependency it shouldn't — Q1), not necessarily that the app is actually broken.

**Senior-level triage habit:** "The first thing I check is `--previous` logs, not just current — by the time you notice CrashLoopBackOff, the pod may already be restarting again, and the container you're looking at might not have crashed yet or might not have any logs. The second thing I check is the **exit code** in `describe pod`, because it immediately tells me which of two very different investigation paths to take: 137 sends me toward resource limits/probes, while 1 (or another app-specific code) sends me straight into application logs."

---

### Q7. Specifically for OOMKilled — what causes it, and how do you actually diagnose whether it's a real memory leak vs just an undersized limit?
**Answer:** OOMKilled means the container exceeded its **memory limit** (Q2), and the kernel's OOM killer terminated it — `kubectl describe pod` shows `Reason: OOMKilled` under the container's last state, and `Exit Code: 137`.

```bash
kubectl describe pod <pod-name>
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

kubectl top pod <pod-name>          # current memory usage vs limit, quick sanity check
```

**How I distinguish "undersized limit" from "real leak" — this is the part interviewers actually want:**
1. **Check if OOMKills are recurring at a predictable interval** — a real leak typically shows **memory climbing steadily over hours/days** until it hits the limit, then crashes, restarts, and climbs again in roughly the same pattern — visible clearly in a Grafana panel of `container_memory_working_set_bytes` over time. A one-off OOMKill during a genuine traffic spike, with normal memory the rest of the time, points toward "limit is just a bit low for peak load," not a leak.
2. **For JVM-based services specifically**, cross-check the container memory limit against **JVM heap settings** — a classic, very common misconfiguration is setting `-Xmx` (max heap) close to or above the container's memory limit, forgetting that the JVM also needs room for **off-heap memory** (thread stacks, metaspace, direct buffers, GC overhead) — the container gets OOMKilled even though heap usage alone never looked excessive.
```yaml
resources:
  limits:
    memory: "1Gi"
env:
  - name: JAVA_OPTS
    value: "-Xmx700m"   # deliberately leave ~300Mi headroom for off-heap JVM memory, NOT -Xmx1000m
```
   "Since Java 10+, `-XX:+UseContainerSupport` (on by default in modern JDKs) makes the JVM auto-detect the container's memory limit and size heap defaults sensibly — but I still set `-Xmx` explicitly in production rather than trusting the default ergonomics, specifically to guarantee that headroom is preserved."
3. **Take a heap dump for a genuine suspected leak** — `jmap`/`jcmd` against the running container (or trigger one automatically via `-XX:+HeapDumpOnOutOfMemoryError`), then analyze with Eclipse MAT to find what's actually accumulating (unbounded caches, listeners never unregistered, connection leaks are the classic culprits).

**Senior-level closing answer:** "My default assumption is **not** 'this is a memory leak' — most OOMKilled incidents I've actually debugged turned out to be an undersized limit relative to real peak load, or a JVM heap setting too close to the container limit with no off-heap headroom, both fixable in minutes. I only reach for a full heap-dump-and-analyze investigation once the memory-over-time graph shows a genuine, sustained upward climb that doesn't correlate with traffic and doesn't reset after a GC cycle — that's the actual signature of a real leak, versus normal (if underprovisioned) memory behavior."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between a pod eviction and OOMKilled?" → OOMKilled is the **container's own** memory limit being exceeded (kernel kills that one container); eviction is the **kubelet** proactively removing an entire pod because the **node itself** is under resource pressure — different trigger, different scope.
- "Does readiness probe failure count toward CrashLoopBackOff?" → No — a failing readiness probe just removes the pod from Service endpoints (Q1); the pod keeps running and isn't restarted, so it won't show CrashLoopBackOff from readiness failures alone (only liveness failures trigger restarts).
- "What does `ImagePullBackOff` mean, as opposed to `CrashLoopBackOff`?" → A completely different, earlier-stage problem — Kubernetes can't even pull the container image (wrong tag, missing registry credentials, private registry auth issue) — the container never even starts, so check `describe pod` Events for the actual pull error rather than digging into application logs.
- "Can HPA and Cluster Autoscaler work together, and how do they differ?" → Yes, and they operate at different layers — HPA scales **pod replica count** within existing node capacity; Cluster Autoscaler scales the **number of nodes** in the cluster when there isn't enough room to schedule the pods HPA wants to add — HPA scaling up can trigger Cluster Autoscaler to add nodes if the cluster is full.
- "Why might maxSurge/maxUnavailable both being 0 be a problem?" → That configuration would prevent the rollout from making any progress at all — you need at least surge or availability room to create new pods or remove old ones; it's a common misconfiguration mistake that stalls a rollout indefinitely.
- "What's `terminationGracePeriodSeconds` and why does it matter alongside rolling updates?" → It controls how long Kubernetes waits after sending SIGTERM before force-killing (SIGKILL) a terminating pod — for a service handling in-flight requests, this needs to be long enough for graceful shutdown (finish current requests, close connections cleanly) or a rolling update can abruptly cut off in-flight work during otherwise-routine deployments.

---

*Study tip: The liveness-checking-downstream-dependencies mistake (Q1) and OOMKilled diagnosis — specifically distinguishing a real leak from an undersized limit or JVM heap misconfiguration (Q7) — are the two areas where interviewers most reliably separate candidates who've read Kubernetes docs from those who've actually been paged for a CrashLoopBackOff at 2am and had to fix it under pressure.*
