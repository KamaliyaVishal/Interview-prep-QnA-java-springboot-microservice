# CI/CD — Interview Prep
---

## SECTION 1: CI/CD PIPELINE STAGES

### Q1. Walk through a complete CI/CD pipeline for a Spring Boot microservice, stage by stage — what does each stage actually verify, and why does the order matter?
**Answer:** A well-designed pipeline is built around a core principle: **fail as fast and as cheaply as possible** — cheap, fast checks run first; expensive, slow checks run only after the cheap ones pass, so a broken build never wastes time/resources on stages that were never going to matter.

```
Checkout → Build → Static Analysis → Unit Tests → Package →
Security Scan → Push Artifact → Deploy (Dev) → Integration Tests →
Deploy (Staging) → Smoke Tests → Deploy (Prod) → Post-deploy verification
```

| Stage | What it verifies | Typical duration | Fails the build if... |
|---|---|---|---|
| **Checkout** | Source retrieved correctly | seconds | Repo/branch inaccessible |
| **Build/Compile** | Code actually compiles | seconds–minutes | Compilation errors |
| **Static Analysis** | Code quality, style, known bug patterns (Q3) | minutes | Quality gate thresholds violated (e.g., SonarQube quality gate) |
| **Unit Tests** | Business logic correctness in isolation | minutes | Any test fails, or coverage drops below threshold |
| **Package** | Build the deployable artifact (JAR + Docker image) | minutes | Packaging errors |
| **Security Scan** | Known vulnerabilities in dependencies/image (Q3) | minutes | Critical/high CVEs found (policy-dependent) |
| **Push Artifact** | Store the versioned, immutable build output (Q4) | seconds | Repository unreachable/auth failure |
| **Deploy + Integration Tests** | Service behaves correctly with real dependencies | minutes | Integration test failures |
| **Deploy to Prod** | Final rollout, typically gated (Q1 follow-up) | minutes | Health checks fail post-deploy |

**Senior-level answer on ordering:** "I deliberately run static analysis and unit tests **before** the expensive steps like building a Docker image or running a full integration test suite against real infrastructure — there's no point spinning up Testcontainers or pushing an image if the code doesn't even pass basic quality gates. This 'fail fast' ordering isn't just about tidiness — on a busy CI system, it directly affects how quickly a developer gets feedback and how much shared CI capacity a broken commit wastes before someone even notices."

**Follow-up: "What's the difference between CI and CD, precisely?"** → **Continuous Integration** is the practice of frequently merging code and automatically building/testing it — the goal is catching integration problems early, continuously, rather than in a painful merge at the end of a sprint. **Continuous Delivery** extends that so every passing build is **automatically deployable** to production (a human still approves the actual release). **Continuous Deployment** goes one step further — every passing build is **automatically deployed to production with no manual gate at all**. Most enterprise/fintech environments I've worked in stop at Continuous Delivery, keeping a manual approval gate before production specifically for compliance/audit reasons.

---

## SECTION 2: BUILD, TEST, PACKAGE & DEPLOY

### Q2. Go deeper on the Build → Test → Package → Deploy stages specifically — what does each look like concretely for a Java/Spring Boot service, and what are the common mistakes at each?
**Answer:**

**Build:**
```bash
mvn clean compile          # or: gradle build --exclude-task test  (compile without running tests yet)
```
Common mistake: running the full `mvn clean install` (which includes tests + packaging) as a single monolithic step, instead of splitting build/test/package into **distinct pipeline stages** — this loses the ability to fail fast at the cheapest stage and makes it harder to see exactly which phase failed at a glance in the CI UI.

**Test:**
```bash
mvn test                      # unit tests only — fast, no external dependencies
mvn verify -Pintegration-test  # integration tests — often via Testcontainers, genuinely spins up real dependencies
```
```java
@Testcontainers
@SpringBootTest
class OrderServiceIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // real DB, real behavior — catches issues a mocked repository never would
}
```
Common mistake: **not separating unit tests from integration tests** in the pipeline — running slow, infrastructure-dependent integration tests on every single commit/push (rather than unit tests on every push, integration tests on merge/nightly) unnecessarily slows down the fast feedback loop developers depend on.

**Package:**
```dockerfile
# Multi-stage build — the standard senior-level pattern, worth stating explicitly
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline                # cache dependencies in their own layer
COPY src ./src
RUN mvn clean package -DskipTests             # tests already ran in an earlier CI stage — don't re-run here

FROM eclipse-temurin:17-jre-alpine AS runtime  # small runtime-only base image — no build tooling in the final image
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN addgroup -S appgroup && adduser -S appuser -G appgroup   # don't run as root (Q5)
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```
"Multi-stage builds matter here for two real reasons: the **final image is dramatically smaller** (no Maven, no JDK build tools, no source code — just a JRE and the JAR), which speeds up pulls and reduces the attack surface for security scanning (Q3); and it keeps the Dockerfile itself as the single source of truth for how the artifact is built, rather than requiring a separate build script outside of Docker."

**Deploy:**
```bash
helm upgrade --install order-service ./chart -f values-${ENV}.yaml \
  --set image.tag=${GIT_COMMIT_SHA} --atomic --timeout 5m
```
Common mistake: deploying with a mutable tag like `:latest` instead of an **immutable, traceable tag** (commit SHA or semantic version) — with `:latest`, you genuinely cannot answer "which exact code is running in prod right now" without digging through deploy logs, and a rollback command can't reliably target a specific known-good version.

---

## SECTION 3: STATIC ANALYSIS & SECURITY SCANNING

### Q3. What's the difference between static analysis and security scanning, and what specific tools/checks would you wire into a pipeline?
**Answer:** They overlap but target different concerns: **static analysis** focuses on **code quality and maintainability** (bugs, code smells, complexity, style violations) — issues that make the code worse to work with or hint at latent bugs. **Security scanning** focuses specifically on **known vulnerabilities** — in your own code, your dependencies, and your container images.

| Category | Tool examples | Catches |
|---|---|---|
| **Static code analysis (SAST)** | SonarQube, Checkstyle, PMD, SpotBugs | Code smells, cyclomatic complexity, duplicated code, potential null-pointer bugs, style violations |
| **Dependency vulnerability scanning (SCA)** | OWASP Dependency-Check, Snyk, GitHub Dependabot | Known CVEs in third-party libraries your project pulls in transitively |
| **Container image scanning** | Trivy, Grype, Docker Scout | Known CVEs in the OS packages and layers baked into your Docker image |
| **Secret scanning** | GitLeaks, TruffleHog, GitHub secret scanning | Accidentally committed API keys, passwords, credentials in source/history |

```yaml
# Example: SonarQube quality gate step (GitHub Actions)
- name: SonarQube Scan
  run: mvn sonar:sonar -Dsonar.projectKey=order-service -Dsonar.qualitygate.wait=true
  # pipeline FAILS the build if the quality gate (coverage %, new bugs, duplication threshold) isn't met

# Example: Trivy image scan, failing the build on HIGH/CRITICAL CVEs
- name: Scan Docker image
  run: trivy image --severity HIGH,CRITICAL --exit-code 1 myregistry/order-service:${{ github.sha }}
```

**Senior-level nuance on where to draw the line, since "block on any finding" isn't realistic:** "I don't gate the build on **every** finding — that leads to a wall of low-severity noise that teams learn to ignore or bypass entirely. I set the quality gate/scan policy to fail the build specifically on **new** critical/high issues introduced by that change (not pre-existing tech debt the team hasn't had bandwidth to address yet), and track everything else as visible, trending metrics rather than hard blockers. The goal is to make the pipeline a **useful gate**, not a bureaucratic obstacle developers route around."

**Follow-up: "What do you do when a scan finds a CVE in a transitive dependency you don't control directly?"** → Check if a patched version of the direct dependency exists that pulls in the fixed transitive version first; if not, evaluate actual exploitability in your specific usage (many CVEs are in code paths you never invoke) and either apply a dependency override/exclusion with the fixed version pinned explicitly, or document and formally accept the risk with an expiry date for re-review — blindly suppressing the finding without a documented reason is the anti-pattern to avoid.

---

## SECTION 4: ARTIFACT REPOSITORY

### Q4. What role does an artifact repository play in the pipeline, and why not just build the artifact fresh at deploy time each time you need it?
**Answer:** An artifact repository (Nexus, JFrog Artifactory, GitHub Packages, Docker Hub/ECR/GCR for images) is where **built, versioned artifacts are stored once and reused** — JARs, Docker images, npm packages — rather than rebuilt from source every time they're needed downstream.

**Why "build once, deploy many times" matters, not just "build fresh each time":**
- **Immutability and traceability** — the exact artifact that passed testing and scanning in the pipeline is the **exact same bytes** deployed to staging and then prod — rebuilding at deploy time risks a subtly different build (different dependency resolution at a different point in time, a flaky build step) actually reaching production than what was tested.
- **Speed** — pulling a pre-built artifact is far faster than recompiling/repackaging on every deploy, especially for promotion between environments.
- **Rollback reliability** — rolling back means pulling a **previously published, still-available** artifact version, not attempting to rebuild an old commit and hoping the build environment/dependency versions haven't drifted since.

```bash
# Publishing a versioned artifact (Maven)
mvn deploy -DaltDeploymentRepository=nexus::default::https://nexus.company.com/repository/releases

# Docker image, tagged with an immutable identifier and pushed to a registry
docker build -t myregistry.company.com/order-service:${GIT_COMMIT_SHA} .
docker push myregistry.company.com/order-service:${GIT_COMMIT_SHA}
```

**Senior-level talking point on promotion:** "I treat environment promotion as **moving the same already-built artifact forward**, not rebuilding per environment — build once in CI, run it through dev, then **promote the identical image/JAR** to staging and prod (just changing which config/values it's deployed with, Q4/MOD7_004 Q4). If prod is running a different binary than what actually passed your test suite, your tests were validating something other than what's actually live — that gap is exactly what 'build once' eliminates."

**Follow-up: "How do you handle artifact retention/cleanup?"** → Define a retention policy (e.g., keep all release-tagged versions indefinitely, prune snapshot/dev-branch builds after N days or N versions) — unmanaged artifact repositories grow unbounded and become expensive/slow; most artifact repository tools (Nexus, Artifactory) support this natively via cleanup policies.

---

## SECTION 5: DOCKER IMAGE BUILD/PUSH

### Q5. What are the actual best practices for building and pushing Docker images inside a CI pipeline — beyond just `docker build && docker push`?
**Answer:**
- **Multi-stage builds** (Q2) — smaller final image, no build tooling shipped to production, smaller attack surface.
- **Layer caching, ordered deliberately** — copy dependency manifests (`pom.xml`/`package.json`) and run dependency resolution **before** copying source code, so the expensive dependency-download layer is cached and only invalidated when dependencies actually change, not on every source code edit.
- **Immutable, traceable tags** — tag with the git commit SHA (or a semantic version for releases), never rely on `:latest` alone for anything deployed (Q2, Q4).
- **Run as a non-root user** — a container running as root inside is a meaningfully larger blast radius if the application is ever compromised; explicitly create and switch to a non-root user in the Dockerfile.
- **Scan before push, or scan and gate the pipeline before deploy** (Q3) — don't let a vulnerable image quietly reach the registry (and downstream deploy stages) unchecked.
- **Use a minimal, trusted base image** — `eclipse-temurin:17-jre-alpine` or a distroless image, rather than a full general-purpose OS image with tools/shells you don't actually need at runtime, further shrinking both size and attack surface.

```yaml
# GitHub Actions — build, scan, and conditionally push
- name: Build image
  run: docker build -t order-service:${{ github.sha }} .

- name: Scan image
  run: trivy image --exit-code 1 --severity CRITICAL,HIGH order-service:${{ github.sha }}
  # pipeline stops HERE if critical vulnerabilities found — image never reaches the registry

- name: Push image
  if: success()
  run: |
    docker tag order-service:${{ github.sha }} myregistry.io/order-service:${{ github.sha }}
    docker push myregistry.io/order-service:${{ github.sha }}
```

**Senior-level answer on caching in CI specifically:** "Local Docker layer caching doesn't automatically carry over between CI runs on ephemeral build agents — each run can start with a cold cache unless you explicitly configure **registry-based caching** (`--cache-from` pulling a previous image as a cache source) or your CI platform's native Docker layer cache (e.g., GitHub Actions' `docker/build-push-action` with `cache-from`/`cache-to` pointing at GitHub's cache backend, or BuildKit's inline cache). Without that, teams are often surprised their builds are slow in CI despite being fast locally — the difference is almost always missing cross-run cache persistence, not the Dockerfile itself."

---

## SECTION 6: JENKINS, GITHUB ACTIONS, GITLAB CI & AZURE DEVOPS

### Q6. Compare Jenkins, GitHub Actions, GitLab CI, and Azure DevOps — what would actually drive your choice between them?
**Answer:**
| Aspect | Jenkins | GitHub Actions | GitLab CI | Azure DevOps |
|---|---|---|---|---|
| Hosting model | Self-hosted (full control, full ops burden) | SaaS, hosted runners (or self-hosted runners) | SaaS or self-hosted (GitLab Runner) | SaaS (hosted agents) or self-hosted |
| Config format | Groovy-based `Jenkinsfile` (scripted or declarative) | YAML workflow files (`.github/workflows/*.yml`) | YAML (`.gitlab-ci.yml`) | YAML (`azure-pipelines.yml`) or classic UI-based pipelines |
| Ecosystem/plugins | Enormous plugin ecosystem (oldest, most mature, but plugin quality/maintenance varies widely) | Marketplace of Actions, tightly integrated with GitHub itself | Built-in features (Container Registry, security scanning) tightly integrated with GitLab | Deep integration with Azure services, Boards, Repos |
| Operational overhead | High — you own the Jenkins server, plugin updates, scaling agents | Low — fully managed by GitHub | Low (SaaS) to Medium (self-hosted runners) | Low — fully managed by Microsoft |
| Best fit | Complex, highly customized pipelines; on-prem/regulated environments needing full infra control | Teams already on GitHub wanting fast setup with minimal ops | Teams already on GitLab wanting an integrated DevOps platform (repo + CI + registry + security in one) | Teams in the Microsoft/Azure ecosystem, enterprises using Azure Boards/Repos |

```groovy
// Jenkinsfile (declarative)
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'mvn clean compile' } }
        stage('Test')  { steps { sh 'mvn test' } }
        stage('Package') { steps { sh 'mvn package -DskipTests' } }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'helm upgrade --install order-service ./chart --atomic' }
        }
    }
}
```
```yaml
# GitHub Actions equivalent
name: CI/CD
on: [push]
jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn clean compile
      - run: mvn test
      - run: mvn package -DskipTests
      - if: github.ref == 'refs/heads/main'
        run: helm upgrade --install order-service ./chart --atomic
```

**Senior-level answer, since "which is best" isn't a real answer:** "I don't think there's a universally 'best' one — the decision is almost entirely driven by where the **source code already lives** and how much operational overhead the team is willing to own. If we're on GitHub, GitHub Actions removes an entire category of infrastructure-maintenance work compared to running our own Jenkins fleet, and I'd need a strong specific reason (a very complex, highly custom pipeline need, or an existing large investment in Jenkins plugins/infrastructure) to justify the added ops burden of self-hosting Jenkins instead. I've worked in a regulated fintech environment where Jenkins, self-hosted entirely within our own network with no external SaaS dependency, was actually a compliance requirement, not a preference — so 'best' really depends on constraints outside the tool itself."

**Follow-up: "What's a self-hosted runner, and when would you actually need one even on a SaaS platform like GitHub Actions?"** → A self-hosted runner lets **your own infrastructure** execute the pipeline jobs instead of the platform's shared cloud runners — needed when the pipeline requires access to **internal/private network resources** (an on-prem database, an internal artifact repository not exposed to the internet), specific hardware (GPU-based ML builds), or when compliance requires build execution to stay entirely within your own network boundary.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between a CI pipeline stage and a CI pipeline job/step?" → Terminology varies by tool, but generally a **stage** is a logical phase (Test, Build) that can contain multiple **jobs**, which run one or more **steps** (individual commands) — jobs within a stage can often run in parallel, while stages typically run sequentially.
- "Should database migrations run as part of the deploy pipeline?" → Yes, typically via a dedicated migration tool (Flyway/Liquibase) run as an explicit pipeline step **before** the new application version starts serving traffic — and migrations should be written to be backward-compatible with the *previous* app version too, since during a rolling update (MOD7_003 Q4) old and new pod versions briefly run against the same schema simultaneously.
- "What's a canary deployment, and how does it relate to the pipeline?" → A deployment strategy where the new version is rolled out to a **small percentage of traffic first**, monitored against key metrics, and only promoted to full rollout if healthy — often implemented as an explicit pipeline stage with an automated or manual gate between "canary" and "full rollout," sometimes using a service mesh (MOD4_003 Q13) for the traffic split.
- "Why use Testcontainers instead of mocking the database in integration tests?" → Mocks verify your code calls the mock correctly, but don't catch real SQL syntax errors, database-specific behavior differences, or actual constraint violations — Testcontainers spins up a **real** instance of the actual database in a container for the test, catching a category of bugs mocks structurally cannot.
- "What's 'shift-left' in the context of CI/CD security?" → Moving security/quality checks (SAST, SCA, secret scanning) as early as possible in the pipeline — ideally even to the developer's local pre-commit hook or IDE — rather than only catching issues at a late gate, so problems are found and fixed when they're cheapest to fix.
- "How do you handle pipeline secrets (DB passwords, API keys) safely?" → Never hardcode them in the pipeline YAML/Jenkinsfile itself — use the platform's built-in secrets store (GitHub Actions Secrets, Jenkins Credentials, GitLab CI/CD Variables masked+protected) injected as environment variables at runtime, and for production-grade needs, a dedicated secrets manager (Vault, AWS Secrets Manager) rather than the CI platform's own secret store alone.

---

*Study tip: The "build once, promote the same artifact" principle (Q4) and knowing exactly what multi-stage Docker builds actually buy you beyond "smaller image" (Q2, Q5) are where interviewers most reliably separate candidates who've configured a YAML pipeline once from those who've actually owned a production CI/CD system end-to-end. Be ready to name a real trade-off you made between pipeline speed and thoroughness — that's the natural follow-up to almost every question in this module.*
