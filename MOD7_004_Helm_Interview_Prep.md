# Helm — Interview Prep
---

## SECTION 1: HELM CHARTS, TEMPLATES & CHART STRUCTURE

### Q1. What is Helm, what problem does it actually solve, and what's the structure of a Helm chart?
**Answer:** Helm is the **package manager for Kubernetes** — it solves the problem of managing complex, multi-resource Kubernetes applications (a Deployment + Service + ConfigMap + Ingress + HPA, all needing to work together and be versioned/deployed as one unit) without hand-maintaining a pile of loosely-related YAML files applied via `kubectl apply -f`. A **Chart** is the packaged, templated definition of that application; a **Release** is a specific deployed instance of a chart (with a given set of values) running in a cluster.

**Standard chart structure:**
```
order-service-chart/
├── Chart.yaml              # chart metadata: name, version, appVersion, dependencies
├── values.yaml              # default configuration values (Q3)
├── templates/                # Kubernetes manifest templates, rendered using values
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl          # reusable named template snippets (helper functions)
│   └── NOTES.txt              # post-install usage message shown to the user
├── charts/                    # bundled sub-charts (dependencies)
└── values.schema.json         # (optional) JSON Schema to validate values.yaml input
```

**Why this beats plain `kubectl apply -f` for anything non-trivial:** "With plain manifests, deploying to dev/staging/prod means either maintaining near-duplicate YAML files per environment (drift-prone, tedious to keep in sync) or scripting substitutions yourself. Helm templates the manifests **once** and parameterizes everything that legitimately differs per environment — replica count, resource limits, image tag — through `values.yaml` (Q3), so the actual Kubernetes resource *shape* stays identical across environments, and only the values that should differ, do. It also gives you **versioned releases** with built-in rollback (Q4), which plain `kubectl apply` doesn't track at all."

---

### Q2. Walk through how Helm templating actually works — show me a real template using variables, conditionals, and loops.
**Answer:** Helm templates are standard Kubernetes YAML with **Go template syntax** embedded — values from `values.yaml`, `Chart.yaml`, and built-in objects get substituted in at `helm install`/`upgrade` time, producing the final rendered manifest that's actually applied to the cluster.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-order-service
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: order-service
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range .Values.extraEnv }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: 8080
          {{- end }}
```

**Breaking down what's actually happening here, since this is where candidates often just wave their hands:**
- `{{ .Values.replicaCount }}` — pulls a value from `values.yaml` (or an override, Q3).
- `{{ .Release.Name }}` / `{{ .Chart.Name }}` — **built-in objects** Helm provides automatically — `.Release` carries release-specific info (name, namespace, revision), `.Chart` carries chart metadata.
- `{{- range .Values.extraEnv }}...{{- end }}` — a **loop**, iterating over a list defined in values, letting the number of env vars be entirely data-driven rather than hardcoded.
- `{{- if .Values.livenessProbe.enabled }}...{{- end }}` — **conditional rendering** — whole blocks of YAML can be included or omitted based on a value, letting one template serve very different configurations.
- `{{- toYaml .Values.resources | nindent 12 }}` — takes an entire YAML block from values and re-indents it correctly into the template — the standard pattern for embedding structured, multi-line config (like `resources.requests`/`resources.limits`) without hand-flattening every field.
- The leading `-` in `{{-` trims whitespace/newlines — necessary because Go templating is whitespace-literal, and without it you'd get extra blank lines or broken YAML indentation in the rendered output.

**Debugging tip worth stating proactively:** "`helm template <chart>` renders the manifests locally without touching the cluster at all, and `helm install --dry-run --debug` renders **and** validates against the target cluster's API — I always run one of these before a real install/upgrade, especially after editing templates, because template syntax errors or bad indentation only surface at render time, not when you save the file."

---

## SECTION 2: VALUES.YAML

### Q3. How does `values.yaml` work, and what's the actual precedence order when values come from multiple sources — this is a very common practical question?
**Answer:** `values.yaml` defines the **default configuration** for the chart — every `{{ .Values.x }}` reference in templates resolves against this (or an override). Helm supports overriding values from several sources, and knowing the **precedence order** is exactly the kind of practical detail that separates someone who's actually run Helm in a real pipeline from someone who's only read the docs.

```yaml
# values.yaml (chart defaults)
replicaCount: 2
image:
  repository: myregistry/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"
livenessProbe:
  enabled: true
  path: /actuator/health/liveness
extraEnv: []
```

**Precedence, from LOWEST to HIGHEST (later overrides earlier):**
1. Chart's own `values.yaml` (the defaults, as above)
2. Parent chart's values, for a sub-chart dependency
3. A values file passed via `-f`/`--values` (can pass multiple, later files override earlier ones)
4. Individual overrides via `--set` on the command line (highest precedence)

```bash
helm install order-service ./order-service-chart \
  -f values-prod.yaml \
  --set replicaCount=5 \
  --set image.tag=1.2.3
# --set here wins over anything in values-prod.yaml, which wins over the chart's own values.yaml
```

**Senior-level answer:** "In practice, I treat `--set` as something for **quick, one-off overrides or CI pipeline injection** (like setting `image.tag` to the exact commit SHA being deployed) — not as the primary way to manage environment configuration, because `--set` values aren't captured anywhere durable/reviewable the way a values file checked into git is. For anything structural or meaningfully different per environment, it goes in a proper `values-<env>.yaml` file (Q4) that's version-controlled, reviewable in a PR, and reproducible — `--set` is for the handful of values (like a build-time image tag) that are genuinely dynamic per deployment, not for hand-managing an entire environment's config on the command line."

---

## SECTION 3: ENVIRONMENT-SPECIFIC CONFIGURATION

### Q4. How do you manage configuration differences across dev/staging/prod with Helm without duplicating the whole chart?
**Answer:** The chart (templates) stays **identical** across every environment — only the **values** differ. The standard pattern is one base `values.yaml` with sensible defaults, plus a **thin override file per environment** containing only what's genuinely different for that environment.

```yaml
# values.yaml (base defaults — shared baseline)
replicaCount: 2
resources:
  requests: { memory: "512Mi", cpu: "250m" }
  limits: { memory: "1Gi", cpu: "500m" }
autoscaling:
  enabled: false

# values-prod.yaml (ONLY the overrides — not a full duplicate)
replicaCount: 5
resources:
  requests: { memory: "1Gi", cpu: "500m" }
  limits: { memory: "2Gi", cpu: "1" }
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20

# values-dev.yaml (ONLY the overrides for dev)
replicaCount: 1
resources:
  requests: { memory: "256Mi", cpu: "100m" }
```

```bash
helm install order-service ./order-service-chart -f values.yaml -f values-dev.yaml
helm install order-service ./order-service-chart -f values.yaml -f values-prod.yaml
```

**The design principle that actually matters here, worth stating explicitly:** "The environment-specific file should contain **only the deltas** — replica count, resource sizing, feature flags, external endpoint URLs — never a full copy-pasted duplicate of the whole values structure. The moment `values-prod.yaml` and `values-dev.yaml` both redefine every single key, you've recreated the exact drift problem Helm was supposed to solve — a change to a shared default now has to be manually applied to every environment file separately, and they inevitably fall out of sync over time."

**Secrets — a common, important follow-up:** "Plaintext secrets should never sit directly in a `values-prod.yaml` committed to git. In practice I've used **Helm Secrets** (SOPS-encrypted values files) or, more commonly, kept secret values **out of Helm entirely** — referencing a Kubernetes `Secret` that's provisioned separately (via Sealed Secrets, External Secrets Operator pulling from AWS Secrets Manager/Vault, or a GitOps secret-management tool) and just pointing the chart at that secret's name, rather than having Helm template the actual secret values itself."

---

## SECTION 4: INSTALL, UPGRADE & ROLLBACK

### Q5. Walk through the Helm release lifecycle — install, upgrade, rollback — and what actually happens under the hood at each step?
**Answer:**

```bash
# INSTALL — first-time deployment, creates a new Release
helm install order-service ./order-service-chart -f values-prod.yaml
# → renders templates + values into final manifests, applies them, creates Release revision 1

# UPGRADE — deploy a new chart version or changed values to an EXISTING release
helm upgrade order-service ./order-service-chart -f values-prod.yaml --set image.tag=1.2.3
# → renders new manifests, diffs against current cluster state, applies changes, creates revision 2

# install-or-upgrade in one command — the pattern actually used in most CI/CD pipelines
helm upgrade --install order-service ./order-service-chart -f values-prod.yaml

# ROLLBACK — revert to a previous revision
helm rollback order-service 1          # back to revision 1 specifically
helm history order-service              # see all revisions and their status

# Always dry-run first for anything going to prod
helm upgrade order-service ./order-service-chart -f values-prod.yaml --dry-run --debug
```

**What's actually happening under the hood, since this is where the real depth lives:** "Helm stores each release's rendered manifest and metadata as a **Secret** (by default, in modern Helm 3) in the target namespace — that's what makes `helm history` and `helm rollback` possible: it's not re-running your local template files against some remembered git commit, it's replaying the **exact previously-rendered manifest** that was captured at that revision. This matters practically — if I've since changed `values.yaml` or edited a template, `helm rollback` still reverts to what was **actually deployed** at that earlier revision, not to whatever the chart files currently look like on disk."

**How upgrade safely handles a failure — the part that separates confident answers from shaky ones:**
```bash
helm upgrade order-service ./order-service-chart --atomic --timeout 5m
```
"`--atomic` is what I always add for production upgrades — if the upgrade fails or the new pods don't become healthy within the timeout, Helm **automatically rolls back** to the previous working release, rather than leaving the cluster in a half-upgraded, broken intermediate state. Without `--atomic`, a failed upgrade just... stops wherever it failed, and you have to notice and manually run `helm rollback` yourself — I've seen that manual gap turn a routine deployment into an extended incident purely because nobody was watching closely enough to catch it immediately."

**Rollback's dependency on the underlying Kubernetes rollout mechanics:** "It's worth being explicit that `helm rollback` doesn't magically undo a bad deployment instantly — it triggers a normal Kubernetes rolling update (MOD7_003 Q4) back to the previous manifest, meaning it's just as dependent on accurate readiness probes to be safe as any other deployment. A rollback with a broken readiness probe can be just as unsafe as the forward deployment that triggered it."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between `helm template` and `helm install --dry-run`?" → `helm template` renders locally with zero cluster interaction at all (useful even offline, for CI validation); `--dry-run` (via `install` or `upgrade`) renders **and** submits to the cluster's API server for server-side validation (catches things like invalid API versions or admission-webhook rejections) without actually persisting anything.
- "What is a Helm sub-chart / dependency?" → A chart can declare other charts as dependencies in `Chart.yaml` (`dependencies:` block) and bundle them under `charts/` — common for a shared component like a Redis or PostgreSQL chart that your application chart depends on, avoiding hand-writing those manifests yourself.
- "How do you uninstall a release, and does it delete PersistentVolumeClaims?" → `helm uninstall <release>` removes all resources the release created; by default this typically **does** delete Deployments/Services/etc., but PVCs' behavior depends on the chart's design and the StorageClass's reclaim policy — always verify this explicitly for stateful charts before uninstalling in prod, since data loss from an uninstall is a real, avoidable mistake.
- "What's `Chart.yaml`'s `version` vs `appVersion`?" → `version` is the **chart's own** version (increments when the chart's templates/structure change); `appVersion` is the version of the **application** the chart deploys (e.g., the Docker image tag lineage) — they're independent and often diverge, since the chart can be updated without the app changing and vice versa.
- "Can you test a chart's rendered output before ever touching a cluster?" → Yes — `helm lint` checks the chart for structural/syntax issues, `helm template` renders manifests locally, and `helm unittest` (a community plugin) allows writing actual unit tests against expected rendered output — all runnable in CI before any real deployment is attempted.
- "What happens if two `helm upgrade` commands run concurrently against the same release?" → Helm takes out a release-level lock; a second concurrent operation against the same release will typically fail with an error about another operation in progress, rather than silently corrupting release state — still, CI/CD pipelines should avoid triggering concurrent deploys to the same release as a matter of design, not rely on this as a safety net.

---

*Study tip: The values precedence order (Q3) and exactly what `--atomic` protects against during a failed upgrade (Q5) are the two areas where interviewers most reliably separate candidates who've only run `helm install` locally from those who've actually operated Helm in a real CI/CD pipeline deploying to production. Being able to explain that Helm stores rendered manifests as Secrets — not live template re-evaluation — is a strong, concrete signal of genuine hands-on depth.*
