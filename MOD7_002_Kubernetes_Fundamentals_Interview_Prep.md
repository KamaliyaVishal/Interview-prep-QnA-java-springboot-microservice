# Kubernetes Fundamentals — Interview Prep
---

## SECTION 1: CLUSTER, NODE & POD

### Q1. Explain Kubernetes' architecture — control plane vs. node components, and what each piece does.
**Answer:**
```
CONTROL PLANE (the "brain")              WORKER NODE(S) (where workloads run)
┌─────────────────────────┐              ┌─────────────────────────┐
│ kube-apiserver           │◄────────────►│ kubelet                  │
│ (front door — all reqs   │              │ (talks to API server,    │
│  go through here)        │              │  manages pods on node)   │
│                          │              │                          │
│ etcd                     │              │ kube-proxy               │
│ (cluster state — the     │              │ (network rules for       │
│  single source of truth) │              │  Service routing)        │
│                          │              │                          │
│ scheduler                │              │ container runtime        │
│ (decides WHICH node a    │              │ (containerd/CRI-O —      │
│  new pod runs on)        │              │  actually runs containers)│
│                          │              └─────────────────────────┘
│ controller-manager        │
│ (reconciliation loops —   │
│  Deployment, ReplicaSet,  │
│  Node controllers, etc.)  │
└─────────────────────────┘
```

- **`kube-apiserver`** — the only component that talks to `etcd` directly; every `kubectl` command, every controller, every kubelet goes through it. It's a stateless REST front door.
- **`etcd`** — a distributed key-value store (Raft-based, from the Distributed Transactions & Consensus topic) holding the entire cluster's desired and observed state.
- **`scheduler`** — watches for unscheduled pods and assigns each to a node based on resource requests, constraints, and affinity rules.
- **`controller-manager`** — runs the **reconciliation control loops** (Q4) that continuously drive actual state toward desired state.
- **`kubelet`** — the node-level agent; ensures the containers described in pod specs assigned to its node are actually running, and reports node/pod status back to the API server.
- **`kube-proxy`** — maintains network rules (via iptables/IPVS) on each node so traffic to a Service's virtual IP gets routed to the right backend pods.

**Senior-level answer:**
> "The design principle worth calling out explicitly: nearly everything in Kubernetes is a **reconciliation control loop** — a controller continuously watches the actual state (via the API server), compares it to the declared desired state, and takes action to close the gap. That's true for Deployments driving ReplicaSets, for the scheduler assigning pods, for kubelet ensuring containers stay running — it's one consistent pattern applied repeatedly, not a collection of unrelated mechanisms, and once you see that, most of Kubernetes' behavior becomes predictable rather than memorized."

---

### Q2. What is a Pod, and why does Kubernetes use pods as the deployable unit instead of deploying containers directly?
**Answer:** A **Pod** is the smallest deployable unit in Kubernetes — one or more containers that are **always scheduled together on the same node**, share the same network namespace (same IP, can reach each other via `localhost`), and can share storage volumes.

**Why not just deploy containers directly:** some workloads genuinely need **tightly coupled co-located processes** that should live and die together — the classic example is the **sidecar pattern**: a main application container plus a helper container (a log shipper, a service mesh proxy like Envoy/Istio, a config-reloader) that must share the same network/filesystem and be scheduled/scaled as one atomic unit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
    - name: app
      image: myapp:1.0
    - name: log-shipper
      image: fluent-bit:latest
      # shares the same network namespace and can mount the same volume as 'app'
```

**Senior-level answer:**
> "The mental model I use: a **container** is 'one process,' a **Pod** is 'one deployable unit that might contain several tightly-coupled processes that must live, scale, and network together.' The sidecar pattern is the concrete justification — a service mesh proxy needs to intercept every network call the app makes, which only works cleanly if they share a network namespace; running the proxy as a totally separate deployment with its own IP wouldn't give you transparent traffic interception at all. Multi-container pods should be the exception, not the default, though — most workloads are correctly a single-container pod, and reaching for a sidecar should be a deliberate choice for a specific coupling need, not a default habit."

---

### Q3. Walk through the lifecycle phases of a Pod.
**Answer:**
```
Pending -> Running -> Succeeded / Failed
             |
          (or) Unknown  (node unreachable, kubelet not reporting status)
```
- **Pending** — the pod has been accepted by the API server but isn't running yet — could be waiting to be scheduled, or waiting on an image pull.
- **Running** — the pod has been scheduled to a node and at least one container is running (or starting/restarting).
- **Succeeded** — all containers terminated successfully (exit code 0) and won't be restarted — typical for a completed `Job`, not a long-running service.
- **Failed** — all containers terminated, and at least one terminated with a non-zero exit code / was terminated by the system.
- **Unknown** — the pod's state can't be determined, typically because the node it's on is unreachable.

**Senior-level answer:**
> "In practice, I spend most of my diagnostic time distinguishing **why** a pod is stuck in `Pending` versus **why** a `Running` pod is unhealthy — those point to completely different problem classes. A stuck `Pending` pod usually means the scheduler can't place it (insufficient node resources, an unsatisfiable affinity/taint rule, or a PVC that can't be bound) and `kubectl describe pod`'s Events section is the right tool; a `Running`-but-unhealthy pod is where readiness/liveness probes (Observability topic) and container logs come in instead. Conflating these two categories is a common early mistake when triaging a pod issue under pressure."

---

## SECTION 2: DEPLOYMENT & REPLICASET

### Q4. Explain the relationship between Deployment, ReplicaSet, and Pod.
**Answer:** A layered ownership chain, each layer adding a capability the one below doesn't have:

```
Deployment  (manages rollout history, rolling updates, rollback)
     |  owns/creates
ReplicaSet  (ensures N identical pod replicas exist, replaces dead ones)
     |  owns/creates
Pod(s)      (the actual running containers)
```

- A **ReplicaSet**'s only job is: "ensure exactly N pods matching this template exist" — if a pod dies, the ReplicaSet's controller notices (via the reconciliation loop from Q1) and creates a replacement.
- A **Deployment** manages ReplicaSets on your behalf — when you update a Deployment's pod template (e.g., a new image version), it creates a **new** ReplicaSet and orchestrates a rolling transition from the old ReplicaSet to the new one, while keeping the old ReplicaSet around (scaled to 0) for **rollback** history.

```bash
kubectl get deployments,replicasets,pods -l app=myapp
# Deployment "myapp" -> owns ReplicaSet "myapp-7d9f4b" (old, scaled to 0) 
#                     -> owns ReplicaSet "myapp-8c2a1d" (current, scaled to 3) -> 3 Pods
```

**Senior-level answer:**
> "You almost never create or manage a ReplicaSet directly in practice — you work with Deployments, and the ReplicaSet is the mechanism underneath that makes rolling updates and rollback possible by letting two ReplicaSet generations briefly coexist during a transition. Knowing this layering matters practically when debugging: if pods aren't updating as expected after a Deployment change, checking `kubectl get replicasets` often reveals the real story — e.g., a new ReplicaSet stuck at 0/3 ready because the new pod template has a bug, which the Deployment-level view alone doesn't always make obvious."

---

### Q5. Explain the rolling update strategy — what do `maxSurge` and `maxUnavailable` control?
**Answer:** During a rolling update, Kubernetes gradually replaces old-version pods with new-version pods, controlled by two parameters:

- **`maxUnavailable`** — the maximum number/percentage of pods that can be **unavailable** (below the desired replica count) at any point during the rollout.
- **`maxSurge`** — the maximum number/percentage of **extra** pods (above the desired replica count) that can be created temporarily during the rollout.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # can briefly run 1 extra pod above the desired count
      maxUnavailable: 0    # never drop below the desired count — zero-downtime-oriented
  replicas: 3
```
```
Desired: 3 replicas, maxSurge=1, maxUnavailable=0
Step 1: [v1][v1][v1]         -> start 1 new pod: [v1][v1][v1][v2 starting]     (4 total, surge=1)
Step 2: v2 becomes Ready     -> terminate 1 old:  [v1][v1][v2]                 (back to 3, one v1 replaced)
Step 3: repeat until all v1 -> [v2][v2][v2]
```

**Senior-level answer:**
> "The combination `maxSurge: 1, maxUnavailable: 0` is the config I'd default to for a genuinely zero-downtime rollout — it guarantees capacity never drops below the desired replica count, at the cost of briefly running one extra pod (and consuming its resources) during the transition. The opposite extreme, `maxSurge: 0, maxUnavailable: 1`, avoids ever exceeding the resource footprint but **does** briefly reduce capacity — appropriate when you're resource-constrained and can tolerate momentarily reduced capacity but not extra cost. This is a genuine trade-off to reason through per-service, not just a default to leave untouched."

---

### Q6. How does a rolling update actually achieve zero-downtime, and what role do readiness probes play?
**Answer:** The rolling update mechanism alone (Q5) only controls **how many** pods are up/extra at a time — it does **not**, by itself, know whether a new pod is actually able to serve traffic. That's what the **readiness probe** (Observability Fundamentals topic, Q10) provides: a new pod is only counted as "available" (and only then does the Deployment proceed to terminate an old pod) once its readiness probe **passes**.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

**Senior-level answer:**
> "This is the piece that actually closes the loop for zero-downtime deploys, and it's worth stating explicitly because it's a common gap — without a correctly configured readiness probe, Kubernetes considers a pod 'available' the moment its container **starts**, not when the application inside has actually finished initializing (DB connection pool warmup, cache loading, Spring context startup). Without readiness gating, a rolling update can route live traffic to a pod that's technically running but not actually ready to serve requests, causing a burst of errors right during the deploy — which is exactly the kind of subtle gap that only shows up under real production traffic, not in a quick manual test."

---

## SECTION 3: SERVICE & INGRESS

### Q7. Compare the Service types — ClusterIP, NodePort, LoadBalancer, ExternalName.
**Answer:**
| Type | Exposes | Typical Use |
|---|---|---|
| **ClusterIP** (default) | A stable virtual IP, reachable **only from inside the cluster** | Internal service-to-service communication |
| **NodePort** | Opens a static port (30000-32767 range) on **every node's** IP | Simple external access without a cloud load balancer; often used underneath other approaches, or for local/dev clusters |
| **LoadBalancer** | Provisions an actual **cloud provider load balancer** (AWS ELB, GCP LB) pointing at the service | Standard way to expose a service externally in a cloud-managed cluster |
| **ExternalName** | Maps the service name to an **external DNS name** via a CNAME — no proxying, just DNS-level redirection | Referencing an external system (a legacy on-prem DB, a third-party API) using in-cluster service-discovery-style naming |

**Senior-level answer:**
> "These build on each other, which is a good structural way to explain it: `LoadBalancer` is implemented **on top of** `NodePort`, which is implemented **on top of** `ClusterIP` — a `LoadBalancer` service still gets a ClusterIP and a NodePort under the hood, the cloud LB just adds external routing to that NodePort across nodes. Understanding that layering explains why, in a cloud environment without a load balancer integration configured (e.g., a bare-metal or local cluster), a `LoadBalancer` service can get stuck with its `EXTERNAL-IP` shown as `<pending>` forever — there's no cloud controller to actually provision the LB resource, so it falls back to behaving like a NodePort with no external IP assigned."

---

### Q8. How does a Service actually route traffic to the correct backend pods?
**Answer:**
1. A Service selects backend pods via a **label selector** (`selector: app: myapp`) — any pod matching those labels becomes a backend.
2. The **Endpoints** (or `EndpointSlice`) object is continuously updated by a controller to list the current IPs of all matching, **ready** pods.
3. **`kube-proxy`** on every node watches Endpoints/EndpointSlice and programs local networking rules (iptables or IPVS) so that traffic sent to the Service's stable virtual IP gets load-balanced (typically round-robin or random) across the actual pod IPs — this happens at the kernel/network level, not via an actual proxying process in the traffic path for iptables mode.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service     # matches pods with label app=order-service
  ports:
    - port: 80
      targetPort: 8080
```

**Senior-level answer:**
> "The detail worth knowing: only pods that are currently **Ready** (readiness probe passing) are included in the Endpoints list — this is the exact mechanism connecting readiness probes to actual traffic routing, not just a Deployment-level rollout concept. A pod that's `Running` but failing its readiness check is automatically **removed from the Service's routing pool** without needing to be restarted, which is precisely the graceful-degradation behavior discussed in the Observability topic's liveness-vs-readiness distinction — readiness failure means 'stop sending traffic here,' and this Endpoints mechanism is exactly how that's enforced network-wide."

---

### Q9. What problem does Ingress solve that a Service alone doesn't, and what is an Ingress Controller?
**Answer:** A `Service` of type `LoadBalancer` gets its **own** cloud load balancer per service — for a cluster with dozens of externally-exposed services, that means dozens of expensive cloud load balancers, and no shared layer for host/path-based routing, TLS termination, or other HTTP-level concerns.

**Ingress** is a **single entry point** with rules for routing HTTP(S) traffic to different backend Services based on **hostname and/or path** — one load balancer, many services behind it.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service: { name: order-service, port: { number: 80 } }
          - path: /payments
            pathType: Prefix
            backend:
              service: { name: payment-service, port: { number: 80 } }
```

**Ingress Controller** — Ingress resources are just **rules**; something has to actually implement them. The **Ingress Controller** (NGINX Ingress Controller, Traefik, cloud-provider-specific controllers) is the actual running component (typically itself a pod behind a single `LoadBalancer` Service) that watches Ingress objects and configures itself (e.g., generates an NGINX config) to route traffic accordingly.

**Senior-level answer:**
> "This is a commonly confused point worth clarifying directly: Ingress **without** an Ingress Controller installed in the cluster does **nothing** — creating an Ingress resource is declaring intent, not implementing behavior, and a cluster needs an actual controller deployed to make Ingress objects functional at all, similar to how a Deployment resource needs the controller-manager running to actually reconcile it. I'd also mention that many teams are now migrating toward the newer **Gateway API**, which addresses some of Ingress's limitations (more expressive routing, better multi-team ownership model) — worth knowing it exists as the direction the ecosystem is moving, even if Ingress remains far more widely deployed today."

---

## SECTION 4: CONFIGMAP & SECRET

### Q10. What's the difference between a ConfigMap and a Secret, and is a Secret actually secure by default?
**Answer:** Structurally, **ConfigMap and Secret are nearly identical** — both store key-value data that can be injected into pods as environment variables or mounted as files. The difference is intent and (partial) handling:
- **ConfigMap** — for non-sensitive configuration (feature flags, URLs, non-secret settings).
- **Secret** — for sensitive data (passwords, API keys, TLS certs) — values are **base64-encoded**, not encrypted, by default.

**Is a Secret secure by default? No, not fully:**
- Base64 is **encoding, not encryption** — anyone with API access to read the Secret object can trivially decode it; it provides no confidentiality on its own.
- **`etcd` at rest**: unless **encryption at rest** is explicitly enabled for the cluster, Secrets are stored in `etcd` in this same base64 (effectively plaintext-equivalent) form.
- Secrets **are** somewhat better protected than ConfigMaps in a few specific ways — kubelet avoids writing them to node disk unencrypted when using `tmpfs`-backed volume mounts, and RBAC can (and should) restrict `get`/`list` access on Secret objects specifically.

```bash
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
# trivially reversible — anyone with RBAC read access sees the plaintext instantly
```

**Senior-level answer:**
> "This is a genuinely important point to get right in an interview, because 'Secrets are encrypted' is a common and dangerous misconception — out of the box, a Secret's confidentiality is really enforced by **RBAC** (who can read the object) and, if enabled, **etcd encryption at rest**, not by the Secret mechanism itself. For anything genuinely sensitive in a production environment, I'd reach for an external secrets manager (Vault, AWS Secrets Manager, or the External Secrets Operator syncing from one of those into Kubernetes Secrets) rather than treating native Kubernetes Secrets as sufficient security on their own — they're a **plumbing** mechanism for getting sensitive values into pods, not a security boundary by themselves."

---

### Q11. How do you consume a ConfigMap/Secret in a pod, and what's the difference in update behavior between environment variables and volume mounts?
**Answer:**
```yaml
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef: { name: app-config }   # as environment variables
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets              # as mounted files
  volumes:
    - name: secret-vol
      secret: { secretName: db-credentials }
```

**Update behavior — the key difference:**
- **Environment variables** — set **once**, at container start. If the underlying ConfigMap/Secret changes, the running container's environment variables **do not update** — the pod must be restarted (a new rollout) to pick up the change.
- **Volume mounts** — Kubernetes **does** update the mounted file's content when the ConfigMap/Secret changes (via kubelet's periodic sync, typically within about a minute), **without** restarting the pod — but the **application itself** must be watching the file for changes and reload it; Kubernetes doesn't restart or notify the app automatically.

**Senior-level answer:**
> "This distinction genuinely surprises people who assume volume-mounted config 'just works' for dynamic updates — Kubernetes updates the file on disk, but if the application read the file once at startup into memory and never re-reads it, the app is still running on stale config despite the file itself being current. That's why patterns like Spring Cloud Config's refresh scope, or a file-watcher that triggers `@RefreshScope`-style reloading, or a sidecar that restarts the process on file change, exist specifically to close this gap — the platform-level update and the application-level reload are two genuinely separate concerns, and conflating them is a common source of 'I updated the ConfigMap but nothing changed' confusion."

---

## SECTION 5: STATEFULSET & DAEMONSET

### Q12. What does a StatefulSet provide that a Deployment doesn't, and when do you actually need one?
**Answer:** StatefulSet is for workloads that need **stable, unique identity** across restarts and rescheduling — three specific guarantees a Deployment's identical, interchangeable pods don't provide:

1. **Stable, predictable network identity** — pods get ordinal names (`mysql-0`, `mysql-1`, `mysql-2`), not random hashes, and each gets a stable DNS name that persists across restarts/rescheduling.
2. **Ordered, sequential deployment and scaling** — pods are created/deleted **one at a time, in order** (`mysql-0` before `mysql-1`), not all-at-once like a Deployment — important for clustered systems with a bootstrap/leader-election dependency.
3. **Stable, per-pod persistent storage** — each pod gets its **own** PersistentVolumeClaim that follows it across rescheduling — `mysql-1` always reattaches to the **same** volume it had before, not a fresh/shared one.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql          # headless service, gives each pod a stable DNS name
  replicas: 3
  volumeClaimTemplates:        # each replica gets its OWN PVC, not shared
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 10Gi } }
```

**Senior-level answer:**
> "The concrete rule I use: if your workload's replicas are truly **interchangeable** (any pod can serve any request, and losing one and getting a fresh replacement is fine — the typical stateless API service case), use a Deployment. If replicas have **individual identity that matters** — a database node that needs to always reattach to its own data volume, a Kafka broker that needs a stable identity for partition assignment, anything doing peer-to-peer clustering with per-node state — that's a StatefulSet. I'd also flag that in practice, most teams don't hand-roll StatefulSets for something like Postgres or Kafka anymore — they use a purpose-built Kubernetes Operator (a controller with deep knowledge of that specific system's operational needs) that manages the StatefulSet and surrounding resources on their behalf, since correctly operating a clustered stateful system involves more than the StatefulSet primitive alone provides."

---

### Q13. What is a DaemonSet, and what are typical real-world use cases?
**Answer:** A DaemonSet ensures **exactly one copy of a pod runs on every node** (or every node matching a selector) in the cluster — as nodes are added, the DaemonSet controller automatically schedules a copy onto the new node; as nodes are removed, that pod is garbage collected with it.

**Typical use cases** — almost always **node-level infrastructure concerns**, not application workloads:
- **Log collection agents** (Fluentd, Filebeat) — need to run on every node to collect logs from every pod on that node.
- **Node monitoring/metrics agents** (Prometheus Node Exporter, Datadog agent) — need per-node system metrics.
- **Networking/CNI components** — some cluster networking plugins run as DaemonSets to configure networking on every node.
- **Security/compliance agents** — node-level security scanning agents.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  selector:
    matchLabels: { app: fluent-bit }
  template:
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit
```

**Senior-level answer:**
> "The pattern that unifies these use cases: a DaemonSet is right whenever the thing you're deploying is conceptually **'per-node infrastructure'** rather than 'an instance of my application' — you don't want 'N replicas load-balanced across the cluster' (that's a Deployment's job), you want 'exactly one, on every single node, because each node individually needs this thing.' I'd contrast this directly with a Deployment in an interview to make the distinction concrete: a logging agent needs to see every pod's logs **on its own node specifically** — running 3 replicas of it via a Deployment wouldn't guarantee coverage of every node at all, which is exactly the gap DaemonSet closes."

---

## SECTION 6: JOB & CRONJOB

### Q14. What's the difference between a Job and a Deployment, and how do `completions` and `parallelism` work?
**Answer:** A **Job** creates pod(s) that are expected to **run to completion and exit** (exit code 0 = success), unlike a Deployment's pods, which are expected to run **indefinitely** and get restarted if they exit. A Job tracks successful completions and retries failed pods up to a configurable limit.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: monthly-report
spec:
  completions: 5      # needs 5 total successful pod completions
  parallelism: 2       # runs at most 2 pods concurrently to get there
  backoffLimit: 3       # retry a failed pod up to 3 times before giving up
  template:
    spec:
      restartPolicy: Never   # or OnFailure — Job pods must NOT use "Always"
      containers:
        - name: report
          image: report-generator:1.0
```

**Senior-level answer:**
> "The `restartPolicy: Never` (or `OnFailure`) requirement is a small but important detail — Job pods **cannot** use `restartPolicy: Always`, because that's the Deployment-style 'keep this running forever' semantic, which fundamentally conflicts with a Job's 'run to completion' model. I'd also flag `backoffLimit` specifically — without a sensible limit, a Job whose pod keeps failing (a genuine bug, not a transient issue) will keep retrying with exponential backoff indefinitely by default, silently consuming cluster resources on a task that's never going to succeed, rather than surfacing as a clearly failed Job that alerts someone."

---

### Q15. How does a CronJob work, and what does `concurrencyPolicy` control?
**Answer:** A **CronJob** creates a new **Job** on a cron schedule — it's a Job-generator, not a workload itself.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 2 * * *"           # 2 AM daily, standard cron syntax
  concurrencyPolicy: Forbid        # don't start a new run if the previous one is still going
  startingDeadlineSeconds: 300     # if a scheduled run is missed by more than this, skip it rather than run late
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: cleanup-job:1.0
```

**`concurrencyPolicy` options:**
| Value | Behavior |
|---|---|
| **`Allow`** (default) | Multiple runs can overlap and execute concurrently |
| **`Forbid`** | Skip a new scheduled run entirely if the previous run hasn't finished yet |
| **`Replace`** | Cancel the still-running previous execution and start the new one |

**Senior-level answer:**
> "`concurrencyPolicy: Allow` (the default) is a common footgun for anything that isn't idempotent-safe to run concurrently — a nightly reconciliation job that takes longer than 24 hours on a bad day (large data volume, a slow downstream dependency) would, under `Allow`, start overlapping with itself, potentially corrupting shared state or doubling load on a downstream system exactly when it's already struggling. I default to `Forbid` for anything with side effects unless I've specifically verified the job is safe to run concurrently with itself — that's a much safer default assumption than the platform's own default of `Allow`."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What happens if a node fails — what happens to its pods?" → The node controller marks the node `NotReady` after a timeout; pods on it are eventually evicted/rescheduled onto healthy nodes by their owning controller (ReplicaSet/Deployment) — but this only applies to pods managed by a controller; a bare unmanaged pod is simply lost.
- "What's a headless Service, and why does StatefulSet need one?" → A Service with `clusterIP: None` — instead of load-balancing to a single virtual IP, DNS returns the individual pod IPs directly, which is what gives each StatefulSet pod its own addressable, stable DNS name (`mysql-0.mysql.default.svc.cluster.local`).
- "What's the difference between a `Recreate` and `RollingUpdate` deployment strategy?" → `Recreate` terminates all old pods before creating any new ones (causes downtime, but guarantees no two versions ever run simultaneously — sometimes required for apps that can't tolerate mixed versions); `RollingUpdate` (Q5) replaces gradually with no downtime, at the cost of a brief window with both versions live.
- "Can a ConfigMap be too large?" → Yes — ConfigMaps (and Secrets) are stored in `etcd` and have a practical size limit (~1MB); very large configuration should generally be handled differently (e.g., mounted from external storage, or split into multiple ConfigMaps) rather than treating it as unlimited.
- "What's the difference between `kubectl apply` and `kubectl create`?" → `create` fails if the resource already exists; `apply` performs a declarative create-or-update, computing a diff against the last-applied state — `apply` is the standard for GitOps-style, idempotent, repeatable deployments.
- "Why might a StatefulSet pod get stuck in `Pending` when a Deployment pod with the same resource requests doesn't?" → Its PersistentVolumeClaim might be unable to bind (no available PersistentVolume matching the requested storage class/size/access mode) — StatefulSet pods have a hard dependency on their PVC being bindable before they can even be scheduled, which Deployment pods (typically without per-pod PVCs) don't share.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) explaining Kubernetes as fundamentally built on **reconciliation control loops**, since that framing correctly predicts behavior across Deployments, ReplicaSets, and even readiness-driven Endpoints updates, rather than memorizing each resource type in isolation (Q1, Q4), (2) knowing that Secrets are **not encrypted by default** and being able to explain what actually protects them (RBAC, optional etcd encryption at rest) — this is one of the most common and consequential misconceptions in real production Kubernetes usage (Q10), and (3) correctly matching workload type to resource kind — StatefulSet vs. Deployment vs. DaemonSet vs. Job — using the "does this need individual identity / does this need to run on every node / does this run to completion" decision framework rather than by rote memorization (Q12–Q15).*