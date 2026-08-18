# Docker — Interview Prep
---

## SECTION 1: CONTAINER VS VM; DOCKER ARCHITECTURE

### Q1. What's the fundamental difference between a container and a virtual machine?
**Answer:** A **VM** virtualizes the **hardware** — each VM runs its own full guest OS kernel on top of a hypervisor, isolated from the host and other VMs at the kernel level. A **container** virtualizes the **operating system** — containers share the **host's kernel** and are isolated from each other using kernel features (namespaces, cgroups), while each packages only its own application and userspace dependencies.

```
VM:                                   Container:
[ App A ] [ App B ]                   [ App A ] [ App B ]
[ Guest OS A ] [ Guest OS B ]         [ minimal userspace, no separate OS ]
[     Hypervisor      ]               [        Container Runtime          ]
[    Host OS + Kernel  ]              [       Host OS + Kernel            ]
   (each VM: full kernel,                (all containers share ONE
    slow boot, GBs of overhead)           kernel, fast boot, MBs)
```

**Senior-level answer:**
> "The practical consequence of sharing the kernel is what actually matters day-to-day: containers start in milliseconds instead of minutes, have MB-scale images instead of GB-scale, and pack far denser on the same hardware — but the trade-off is a **weaker isolation boundary**. A kernel vulnerability can potentially be exploited to escape a container in a way that's structurally harder with a VM's full hardware-level isolation. This is exactly why security-sensitive multi-tenant platforms sometimes layer VM-level isolation **around** containers (e.g., Firecracker microVMs, gVisor) to get container-like density with closer-to-VM isolation guarantees."

---

### Q2. Explain Docker's architecture — what are the daemon, client, and containerd, and how do they interact?
**Answer:**
```
docker CLI (client)
     |  REST API over a Unix socket / TCP
     v
dockerd (Docker daemon)
     |
     v
containerd  (manages container lifecycle: start/stop/pause)
     |
     v
runc  (OCI runtime — actually creates the container: namespaces, cgroups)
     |
     v
Linux kernel (namespaces, cgroups, the actual isolation primitives)
```

- **`docker` CLI** — what the user types; sends commands to the daemon over its API.
- **`dockerd`** — the long-running background daemon that manages images, builds, networks, volumes, and delegates actual container execution.
- **`containerd`** — a separate, lower-level daemon (also a standalone CNCF project, used directly by Kubernetes too) that handles the container lifecycle.
- **`runc`** — the actual OCI-compliant runtime that creates the container process using Linux namespaces and cgroups.

**Senior-level answer:**
> "The layered architecture is worth knowing specifically because `containerd` and `runc` aren't Docker-proprietary — they're the pieces Kubernetes itself uses directly via the CRI (Container Runtime Interface), which is why 'Docker' as a build/CLI tool became decoupled from the actual container **runtime** in the Kubernetes ecosystem (the deprecation of `dockershim` in Kubernetes 1.24 was exactly this — Kubernetes moved to talk to `containerd` directly, cutting `dockerd` out of that path entirely). Understanding this layering explains why images built with `docker build` still run fine on a Kubernetes cluster with no Docker daemon involved at all — the OCI image format and runtime spec are the actual shared standard, not Docker specifically."

---

### Q3. What Linux kernel features actually provide container isolation, and what does each one isolate?
**Answer:**
| Feature | What It Isolates |
|---|---|
| **Namespaces** | The container's **view** of system resources — PID namespace (its own process tree, PID 1 inside the container), network namespace (its own network interfaces/IP), mount namespace (its own filesystem view), UTS namespace (hostname), IPC namespace, user namespace (UID mapping) |
| **cgroups** (control groups) | **Resource limits** — CPU shares/quota, memory limits, I/O bandwidth — cgroups don't provide isolation of *view*, they enforce *limits* on consumption |
| **Union filesystem** (overlayfs, etc.) | Layered image filesystem — lets multiple containers share common read-only base layers while each has its own thin writable layer on top |

**Senior-level answer:**
> "The distinction between namespaces and cgroups is a common interview gotcha — namespaces answer 'what can this process **see**' (isolation), cgroups answer 'how much can this process **use**' (resource limiting) — they're complementary, not the same mechanism. A container with no memory cgroup limit set, for instance, can still see 'its own' isolated view of the system via namespaces while simultaneously consuming unbounded host memory and starving every other container on the same host — that's exactly why setting explicit resource limits (`docker run --memory`, or Kubernetes resource limits) is a separate, necessary step from just running something 'in a container.'"

---

## SECTION 2: DOCKERFILE & MULTI-STAGE BUILDS

### Q4. Explain Docker image layers — how does the build cache work, and how should you order Dockerfile instructions to take advantage of it?
**Answer:** Each Dockerfile instruction (`RUN`, `COPY`, `ADD`) creates a new **immutable layer**. Docker caches each layer and, on rebuild, **reuses cached layers** for any instruction whose input (the instruction itself and, for `COPY`/`ADD`, the copied files' content) hasn't changed — but the moment one layer's cache is invalidated, **every subsequent layer** must be rebuilt too, even if their own inputs didn't change.

```dockerfile
# BAD — copying everything first invalidates the cache on ANY source file change,
# forcing a full dependency reinstall every single build
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# GOOD — dependency manifest copied and installed FIRST; only invalidated
# when package.json actually changes, not on every source code edit
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

**Senior-level answer:**
> "The ordering principle is: put instructions that **change least often** earliest, and instructions that **change most often** (usually copying application source code) as late as possible. I've seen build times on a Node/Java project drop from several minutes to a few seconds on incremental builds purely from reordering `COPY package.json` / dependency install ahead of `COPY . .` — this is one of the highest-ROI, lowest-effort Dockerfile optimizations, and I'd expect any experienced candidate to bring it up unprompted rather than needing to be asked."

---

### Q5. What is a multi-stage build, and what problem does it solve?
**Answer:** A multi-stage build uses **multiple `FROM` statements** in one Dockerfile — each stage can have its own base image and toolchain, and later stages can selectively **copy artifacts** from earlier stages via `COPY --from=<stage>`, without carrying forward the earlier stage's build tools, source code, or intermediate files into the final image.

```dockerfile
# Stage 1: build stage — has the full JDK, Maven, source code — large, but discarded later
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline           # cached separately from source changes
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: runtime stage — only the JRE and the built jar, nothing else
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Senior-level answer:**
> "Before multi-stage builds, the common workaround was either shipping a bloated image with the entire build toolchain (Maven, the full JDK, source code) in production, or maintaining two separate Dockerfiles and stitching artifacts together with an external build script — both worse than a single, self-contained multi-stage Dockerfile. The concrete win: this pattern routinely takes a Spring Boot image from 600-800MB (full JDK + Maven + source) down to under 200MB (JRE + jar only), which directly matters for pull time, attack surface, and storage costs at scale."

---

### Q6. What's the difference between `COPY` and `ADD`? Which should you default to, and why?
**Answer:**
| | `COPY` | `ADD` |
|---|---|---|
| Basic copy | Yes | Yes |
| Auto-extracts local `.tar` archives | No | Yes |
| Can fetch from a remote URL | No | Yes |
| Behavior transparency | Explicit — does exactly what it looks like | Implicit "magic" — a URL or tarball behaves very differently from a plain file copy |

**Senior-level answer:**
> "Docker's own official best practices explicitly recommend defaulting to `COPY` and reaching for `ADD` only for its specific extra behaviors (local tar auto-extraction) when genuinely needed — `ADD`'s ability to fetch from a **remote URL** is generally discouraged, because the layer then depends on external network availability and content that isn't reproducibly cached or verifiable at build time, and it's clearer to use an explicit `RUN curl` step where the fetch behavior is visible in the build log rather than hidden inside an instruction that looks like a simple file copy."

---

### Q7. What's the difference between `CMD` and `ENTRYPOINT`, and how do they interact when both are specified?
**Answer:**
- **`ENTRYPOINT`** — defines the **fixed executable** the container always runs; not easily overridden at `docker run` time (without `--entrypoint`).
- **`CMD`** — defines **default arguments**, which **are** easily overridden by anything passed after the image name in `docker run <image> <these args override CMD>`.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=default"]
```
```bash
docker run myimage                                    # runs: java -jar app.jar --spring.profiles.active=default
docker run myimage --spring.profiles.active=prod       # runs: java -jar app.jar --spring.profiles.active=prod (CMD overridden)
```

**Senior-level answer:**
> "The pattern I use most: `ENTRYPOINT` for the thing that should **never** change (the actual executable/interpreter), `CMD` for **default, easily-overridable arguments** — this gives you an image that behaves predictably as a fixed executable by default but stays flexible for one-off overrides (e.g., `docker run myimage --help` or `docker run myimage sh` to get a shell for debugging), without needing to rebuild or maintain multiple images for different argument variants."

---

## SECTION 3: DOCKER COMPOSE

### Q8. What problem does Docker Compose solve, and what's a typical use case vs. Kubernetes?
**Answer:** Compose defines and runs **multi-container applications on a single host** from one declarative YAML file — networking between containers, volumes, environment variables, and startup dependencies are all described together and brought up/down with one command.

```yaml
# docker-compose.yml — a Spring Boot app + Postgres + Redis, wired together
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_REDIS_HOST: cache
    depends_on: [db, cache]
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: mydb
    volumes: ["pgdata:/var/lib/postgresql/data"]
  cache:
    image: redis:7-alpine
volumes:
  pgdata:
```

**Senior-level answer:**
> "I position Compose specifically as a **local development and single-host** tool, not a production orchestrator — it has no built-in concept of multi-host scheduling, rolling deployments, self-healing, or auto-scaling, which is exactly the gap Kubernetes fills. A very common and effective workflow: use Compose to spin up a realistic local dev environment (app + DB + cache + message broker, wired together with one `docker compose up`), and use Kubernetes manifests (or Helm) for the actual production deployment — the two aren't competing, they solve different-scoped problems, and conflating them is a common junior mistake."

---

### Q9. What does `depends_on` actually guarantee in Docker Compose, and what's a common misconception about it?
**Answer:** By default, `depends_on` only guarantees **start order** — that the dependency container has been **started**, not that the service **inside** it is actually ready to accept connections. A database container can be "started" (the process has begun) while Postgres itself is still initializing and not yet accepting connections — an app that assumes `depends_on` means "the DB is ready" can fail on startup with connection refused errors, especially on a cold start.

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy   # waits for db's healthcheck to pass, not just "started"
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**Senior-level answer:**
> "This is a genuinely common source of flaky local-dev and CI startup failures, and the fix — `condition: service_healthy` paired with an actual `healthcheck` block — is often missing from Compose files people copy from tutorials that only demonstrate the simpler, misleading `depends_on: [db]` form. The more robust pattern, especially for CI pipelines, combines this with the application's own connection-retry logic (Network Failures topic — exponential backoff on the DB connection attempt) as defense in depth, rather than relying solely on Compose's startup ordering to guarantee true readiness."

---

## SECTION 4: NETWORKING & VOLUMES

### Q10. Explain Docker's default networking modes — bridge, host, and none.
**Answer:**
| Mode | Behavior |
|---|---|
| **bridge** (default) | Container gets its own network namespace and a private IP on a virtual bridge network; reaches the outside world via NAT through the host; other containers on the same bridge network can reach it by container name (via Docker's embedded DNS) |
| **host** | Container shares the **host's** network namespace directly — no isolation, no port mapping needed (`-p` is meaningless), but also no network-level isolation from the host |
| **none** | Container gets no network interface at all besides loopback — fully network-isolated |

```bash
docker run --network bridge -p 8080:8080 myapp   # default: isolated, port explicitly mapped
docker run --network host myapp                   # shares host network stack directly, no port mapping
```

**Senior-level answer:**
> "In a Docker Compose or multi-container setup, the detail that actually matters day-to-day is Compose's **automatic DNS-based service discovery** on the default bridge network it creates — services can reach each other by their **service name** as a hostname (`jdbc:postgresql://db:5432/...` in Q8's example) without any manual IP management, because Compose sets up an embedded DNS resolver scoped to that network. `host` networking is mostly reserved for specific performance-sensitive or low-level networking tools that need to bypass Docker's NAT/bridge overhead entirely — it's not a typical default for application containers."

---

### Q11. What's the difference between a Docker volume, a bind mount, and tmpfs, and when would you use each?
**Answer:**
| | Volume | Bind Mount | tmpfs |
|---|---|---|---|
| **Managed by** | Docker (stored under Docker's own managed directory) | The host filesystem directly, at a path you specify | In-memory only |
| **Persists after container removal** | Yes | Yes (it's just a host directory) | No — gone when container stops |
| **Typical use** | Production data persistence (DB data directories) | Local development (mounting source code for live-reload) | Sensitive/temporary data that shouldn't touch disk at all |

```bash
docker run -v pgdata:/var/lib/postgresql/data postgres        # named volume — Docker-managed, portable
docker run -v $(pwd)/src:/app/src myapp                       # bind mount — host path, great for local dev live-reload
docker run --tmpfs /app/tmp myapp                              # tmpfs — in-memory, never written to disk
```

**Senior-level answer:**
> "I default to **named volumes** for anything production/persistent, specifically because they're portable and Docker-managed — they don't hardcode a host filesystem path the way a bind mount does, which matters when the container might run on a different host than where it was developed. Bind mounts are great for local development specifically because of that host-path coupling — you *want* the container to see your actual working directory live, for fast edit-reload cycles — but that same property makes them the wrong choice for production persistence."

---

## SECTION 5: IMAGE OPTIMIZATION

### Q12. What techniques do you use to reduce Docker image size, beyond multi-stage builds?
**Answer:**
1. **Use minimal base images** — `alpine` (musl-based, ~5MB) or **distroless** images (Google's, containing only the application and its runtime dependencies, no shell, no package manager) instead of a full OS base like `ubuntu`.
2. **Combine `RUN` instructions and clean up in the same layer** — package manager caches removed in a **separate** later `RUN` don't actually shrink the image, because the earlier layer with the cache still exists in the image's layer history.
3. **Use `.dockerignore`** — exclude `node_modules`, `.git`, build artifacts, and test files from the build context so they're never even sent to the daemon or accidentally `COPY`'d in.
4. **Order layers for cache efficiency** (Q4) — not a size reduction per se, but directly affects build speed, which matters for iteration/CI cost.

```dockerfile
# BAD — apt cache is "removed" but still exists in the earlier layer, image stays large
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD — install and cleanup in the SAME layer/instruction
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

**Senior-level answer:**
> "The 'combine `RUN` and clean up in the same instruction' point is the one I'd specifically flag as commonly misunderstood — because each `RUN` is its own immutable layer, deleting a file in a **later** layer doesn't remove it from the image's total size, it just hides it from the final filesystem view while the bytes still exist in an earlier layer and get pulled/stored regardless. This is a genuinely common mistake even from experienced engineers who understand layers conceptually but don't think through the size implication of splitting install-and-cleanup across separate `RUN` instructions."

---

### Q13. How do you choose between `alpine`, `slim`, `distroless`, and a full base image?
**Answer:**
| Base | Size | Trade-off |
|---|---|---|
| **Full** (e.g., `ubuntu`, `debian`) | Large (100s of MB) | Full toolchain/shell available, most compatible, easiest debugging |
| **`-slim`** (Debian-based, trimmed) | Medium | Good middle ground — still glibc-based (fewer compatibility surprises than musl), smaller than full |
| **`alpine`** | Small (~5MB base) | Uses **musl libc**, not glibc — some native dependencies (certain JNI libraries, some Python C-extensions) can behave differently or fail to build/run without extra care |
| **distroless** | Small, no shell/package manager at all | Smallest attack surface — but harder to debug (no shell to `exec` into) |

**Senior-level answer:**
> "Alpine's musl-vs-glibc difference is the practical gotcha I'd bring up specifically — I've seen a Java or Python image behave subtly differently or fail outright on Alpine because a native dependency was compiled against glibc assumptions, which cost more debugging time than the image-size savings were worth for that particular service. My default starting point is actually `-slim` for most JVM/Python services, specifically to avoid musl-compatibility surprises, and I reach for distroless or alpine deliberately once I've verified compatibility, rather than defaulting to the smallest option purely for the size number."

---

## SECTION 6: CONTAINER REGISTRY & SECURITY SCANNING

### Q14. What's the difference between an image `latest` tag and a pinned, immutable tag, and why does it matter for production?
**Answer:** `latest` is just a **mutable label** that can be repointed to a different image at any time — it provides **no guarantee** about which actual image content you're getting, and pulling `latest` at different times (or on different hosts) can silently produce **different** running code. A pinned tag (a specific version, or better, a **digest** — `image@sha256:abc123...`) is **immutable** — it always resolves to the exact same image content.

```yaml
# BAD — "latest" gives no reproducibility guarantee; what runs today may differ from what runs tomorrow
image: myapp:latest

# BETTER — a specific version tag
image: myapp:1.4.2

# BEST — pinned by content digest, cryptographically guaranteed to be the exact same bytes every time
image: myapp@sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

**Senior-level answer:**
> "Using `latest` in production is a well-known anti-pattern for exactly this reason — it breaks reproducibility and makes rollbacks unreliable, because 'roll back to the previous deploy' assumes you know precisely what image that was, and `latest` by definition doesn't tell you that on its own. I'd tie this back to the deployment/rollback discussion from the Production Troubleshooting topic — a safe rollback strategy fundamentally depends on being able to unambiguously reference exactly what was previously running, which requires immutable, specific tags or digests, not `latest`."

---

### Q15. How do you integrate container image security scanning into a CI/CD pipeline, and what does a scanner actually check for?
**Answer:** Image scanners (Trivy, Grype, Snyk, or a registry's built-in scanning like ECR/GCR/Docker Hub) inspect an image's layers against known-vulnerability databases (CVE feeds) and check:
- **OS package vulnerabilities** — known CVEs in installed system packages (this is exactly why minimal base images, Q13, also reduce scan findings — fewer packages installed means fewer potential CVEs to begin with).
- **Application dependency vulnerabilities** — known CVEs in language-level dependencies baked into the image (npm packages, Maven/Gradle dependencies, Python packages).
- **Misconfigurations** — running as root, exposed secrets accidentally baked into a layer, overly permissive file permissions.

```yaml
# Simplified CI step — fail the build on high/critical vulnerabilities before the image is ever pushed
- name: Scan image
  run: trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:${{ github.sha }}
```

**Senior-level answer:**
> "The principle I'd emphasize: scanning should happen **in CI, before the image is pushed to a registry**, not just as a periodic scan of already-deployed images — catching a critical CVE after it's already running in production is strictly worse than blocking the build from ever reaching the registry in the first place. I'd also flag that scanning isn't a one-time check — base images accumulate **new** CVEs over time even if the Dockerfile never changes, since vulnerabilities get disclosed against packages already baked into an image that was scanned clean months ago — so a mature pipeline re-scans images on a schedule too, not just at build time, and has a process for rebuilding/patching images that were clean when built but have since accumulated newly-disclosed vulnerabilities."

---

### Q16. What does "run as non-root" mean in a Dockerfile, and why is it a security best practice?
**Answer:** By default, a container process runs as **root** inside the container unless explicitly told otherwise — and because the container shares the host's kernel (Q1/Q3), a **container-escape vulnerability** combined with a root-privileged container process gives an attacker root access to the underlying host, not just "root inside a sandboxed container." Running as a non-root user significantly limits the damage a compromised container process (or a successful escape) can do.

```dockerfile
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
USER appuser              # switch to non-root BEFORE the app runs
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Senior-level answer:**
> "This connects directly back to the namespaces-vs-cgroups discussion (Q3) — namespaces give a process an isolated **view**, but that isolation isn't an impenetrable security boundary, especially against kernel-level vulnerabilities; running as non-root is a defense-in-depth layer specifically for the scenario where container isolation itself fails or is bypassed. Kubernetes' `securityContext` (`runAsNonRoot: true`, and ideally a read-only root filesystem) enforces this at the orchestration level too, and a lot of organizations' security/compliance policies now mandate it outright rather than leaving it as a per-team judgment call — it's a good thing to mention as both a Dockerfile-level and cluster-policy-level control."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can a container run multiple processes?" → Technically yes, but it's considered an anti-pattern — the standard guidance is one primary process per container, with process supervision (if genuinely needed) handled by a minimal init system like `tini`, not by treating a container like a lightweight VM running many services.
- "What does `EXPOSE` in a Dockerfile actually do?" → It's documentation/metadata only — it does **not** actually publish the port; you still need `-p`/`--publish` (or Compose's `ports:`) at run time to actually map it to the host.
- "Why use `tini` or `--init` in a container?" → PID 1 in a container has special responsibilities (reaping zombie processes, correctly forwarding signals like SIGTERM); many application processes don't handle PID 1 responsibilities correctly on their own, causing containers to not shut down cleanly on `docker stop` — `tini` handles this correctly as a minimal init wrapper.
- "What's the difference between `docker stop` and `docker kill`?" → `stop` sends SIGTERM first (graceful shutdown, allowing cleanup) and waits a grace period before SIGKILL; `kill` sends SIGKILL (or a specified signal) immediately, with no graceful shutdown opportunity.
- "Should secrets ever be passed via `ENV` or build args in a Dockerfile?" → No — both end up baked into the image's layer history/metadata and are recoverable even after being "overwritten" in a later layer; use a secrets manager, runtime-injected environment variables outside the image, or Docker's BuildKit `--secret` mount, which avoids persisting the secret in any layer.
- "What's the difference between an image and a container?" → An image is the immutable, versioned template/blueprint (layers + metadata); a container is a **running (or stopped) instance** of that image, with its own writable layer and runtime state — the same image can be used to start many independent containers.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) explaining Dockerfile layer caching and multi-stage builds with a concrete before/after size or build-time number, not just naming the features (Q4–Q5), (2) knowing the `latest`-tag and secrets-in-layers anti-patterns cold, since both are common real-world production mistakes with direct security/reliability consequences (Q14, and the secrets quick-fire item), and (3) being able to connect container isolation mechanics (namespaces/cgroups) to **why** defense-in-depth practices like non-root users and minimal base images actually matter, rather than reciting them as disconnected checklist items (Q3, Q16) — that connection is what signals genuine operational security understanding rather than memorized best-practices.*
