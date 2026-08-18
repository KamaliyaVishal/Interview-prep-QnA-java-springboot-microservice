# Cloud & Managed Services — Interview Prep
---

## SECTION 1: COMPUTE, STORAGE, NETWORKING & LOAD BALANCERS

### Q1. Walk through the core cloud building blocks — compute, storage, networking, load balancing — and how you'd actually choose between the compute options for a given workload.
**Answer:** Every cloud architecture is built on the same four fundamental categories, and the interview signal here is knowing **when to use which option**, not just naming them.

**Compute spectrum, from most to least operational control:**
| Option | You manage | Cloud manages | Best fit |
|---|---|---|---|
| **VMs (EC2/Azure VM)** | OS, runtime, scaling, patching | Physical hardware | Full control needed, legacy/lift-and-shift workloads |
| **Container orchestration (EKS/AKS)** | App packaging, deployment config, cluster-level policy | Underlying node provisioning (with managed node groups), control plane | Microservices needing orchestration, portability across clouds |
| **Serverless containers (ECS Fargate)** | App packaging, task definition | Servers/nodes entirely — no node management at all | Container workloads without wanting to run/patch nodes |
| **FaaS (Lambda/Azure Functions)** | Just the function code | Everything else — runtime, scaling, servers | Event-driven, short-lived, bursty workloads |

**Storage — matched to access pattern, not just "which is cheapest":**
- **Block storage** (EBS) — attached to a single compute instance, low-latency, for a running application's disk (e.g., a database's data volume).
- **Object storage** (S3) — durable, highly available, accessed over HTTP, for unstructured data (images, backups, logs, static assets) — not mountable as a traditional filesystem.
- **File storage** (EFS) — shared, POSIX-compliant filesystem multiple instances can mount simultaneously — needed when several compute instances genuinely need to share the same files.

**Networking — the core building blocks worth naming explicitly:** VPC (isolated virtual network), subnets (public — internet-facing; private — internal only), security groups (stateful, instance-level firewall), NACLs (stateless, subnet-level firewall), and NAT Gateway (lets private-subnet resources reach the internet outbound without being reachable inbound).

**Load Balancers — the distinction interviewers actually probe:**
| Type | Layer | Use case |
|---|---|---|
| **Application Load Balancer (ALB)** | L7 (HTTP/HTTPS) | Path/host-based routing, needs to inspect request content |
| **Network Load Balancer (NLB)** | L4 (TCP/UDP) | Extreme throughput/low latency, static IP requirement, non-HTTP protocols |

**Senior-level answer on compute choice:** "My decision framework is really about **who I want managing what**, weighed against actual operational maturity and workload shape. For a steady-state, long-running microservice with real scaling and orchestration needs, I go straight to EKS/AKS — the Kubernetes patterns (MOD7_003) already cover HPA, rolling updates, service discovery. For something genuinely event-driven and bursty — image processing triggered by an S3 upload, a lightweight webhook handler — Lambda's pay-per-invocation model is a much better cost and operational fit than keeping a container running 24/7 to handle occasional traffic. I explicitly avoid defaulting to 'everything is a microservice on Kubernetes' — that's over-engineering for genuinely event-driven, low-frequency workloads."

---

## SECTION 2: IAM, SECRETS & KEY MANAGEMENT

### Q2. What is the Principle of Least Privilege in the context of cloud IAM, and how would you actually implement it for a microservice needing to read from S3 and write to RDS?
**Answer:** Least Privilege means every identity (a user, or critically, a **service/application**) gets **only the specific permissions it needs to do its job — nothing broader** — no wildcard `*` permissions "just in case," no reusing one overly-broad role across many services.

```json
// AWS IAM Policy — scoped precisely to what THIS service actually needs, nothing more
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::order-invoices-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": ["rds-db:connect"],
      "Resource": "arn:aws:rds-db:us-east-1:123456789:dbuser:order-db/order-service-user"
    }
  ]
}
```

**How I'd actually wire this to a running service — the part that shows real depth:** "Rather than embedding long-lived AWS access keys as environment variables (a real, common anti-pattern I actively push back on), I attach this policy to an **IAM Role** and associate that role with the compute the service actually runs on — an **EC2 instance profile**, an **ECS task role**, or for EKS specifically, **IRSA (IAM Roles for Service Accounts)**, which lets a specific Kubernetes ServiceAccount assume a specific IAM role via OIDC federation, scoped to exactly that one microservice's pods. The application code just uses the AWS SDK's default credential chain — it never sees or manages a static credential at all, and the underlying temporary credentials are automatically rotated by AWS."

```yaml
# EKS IRSA — Kubernetes ServiceAccount annotated to assume a specific IAM role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
```

**Follow-up: "What's the risk of one shared, overly broad IAM role across all services?"** → A compromise of *any one* service using that role grants the attacker the **combined permissions of every service** sharing it — least privilege's whole point is limiting blast radius; one shared role defeats that entirely, turning a single service compromise into a much broader account compromise.

---

### Q3. How do you actually manage application secrets (DB passwords, API keys) in a cloud-native architecture — and why is a dedicated secrets manager better than environment variables or Kubernetes Secrets alone?
**Answer:** Plain Kubernetes `Secret` objects are only **base64-encoded, not encrypted**, by default — anyone with API access to read that Secret (or etcd access) can trivially decode it; they also have no rotation, versioning, or audit trail built in. A dedicated secrets manager (**AWS Secrets Manager**, **Azure Key Vault**) solves these gaps specifically.

```java
// Fetching a secret at runtime via AWS SDK — no hardcoded credential in code or config
@Bean
public DataSource dataSource(SecretsManagerClient client) {
    GetSecretValueResponse secret = client.getSecretValue(
        GetSecretValueRequest.builder().secretId("prod/order-service/db-credentials").build());
    DbCredentials creds = parseJson(secret.secretString());
    return DataSourceBuilder.create()
        .url(creds.jdbcUrl()).username(creds.username()).password(creds.password()).build();
}
```

```yaml
# Better: External Secrets Operator — syncs the cloud secret INTO a real K8s Secret automatically,
# so app code stays simple (reads a normal K8s Secret/env var) but the source of truth is Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-db-secret
spec:
  secretStoreRef: { name: aws-secrets-manager, kind: ClusterSecretStore }
  target: { name: order-service-db-secret }
  data:
    - secretKey: password
      remoteRef: { key: prod/order-service/db-credentials, property: password }
```

**What a real secrets manager gives you that raw K8s Secrets don't:**
- **Automatic rotation** — Secrets Manager can rotate an RDS password on a schedule and update the secret value, with a Lambda function coordinating the actual DB-side password change — no manual credential rotation process to forget.
- **Fine-grained IAM-based access control and audit trail** — exactly which role/service accessed which secret, when, is logged (CloudTrail) — critical for compliance in fintech/regulated environments.
- **Encryption at rest by default**, backed by a proper key management service (KMS), not just base64 encoding.

**Senior-level answer:** "My default pattern is: secrets live in AWS Secrets Manager or Azure Key Vault as the **single source of truth**, synced into Kubernetes via External Secrets Operator so application code doesn't need cloud-SDK-specific logic just to fetch a password — it reads a normal env var/mounted file the same way it always would. That gives us rotation, audit, and centralized access control at the source of truth, while keeping the actual application code simple and not tightly coupled to a specific cloud provider's secrets API."

**Follow-up on Key Management specifically: "What's KMS/Key Vault's actual role, distinct from secrets management?"** → A **Key Management Service** manages **encryption keys** themselves (used to encrypt data at rest — EBS volumes, S3 buckets, RDS databases) — it's the cryptographic key layer *underneath* both secrets managers and general data encryption; Secrets Manager stores and rotates *credentials*, KMS manages the *encryption keys* that protect data (and often the secrets themselves) at rest — related but distinct responsibilities.

---

## SECTION 3: AWS — EC2, ECS, EKS, LAMBDA, SQS, SNS, RDS

### Q4. Compare EC2, ECS, and EKS — when have you actually chosen one over the others for a real workload?
**Answer:**
| Service | What it is | Orchestration | Best fit |
|---|---|---|---|
| **EC2** | Raw virtual machines | None built-in — you manage everything | Legacy apps, workloads needing full OS-level control, non-containerized apps |
| **ECS** | AWS's own container orchestrator | AWS-native, simpler than Kubernetes, tightly integrated with AWS services | Teams fully committed to AWS, wanting simpler container orchestration without Kubernetes' full complexity |
| **EKS** | Managed Kubernetes | Full Kubernetes — same APIs/patterns as any K8s cluster | Multi-cloud portability desired, team already has Kubernetes expertise, needing the broader K8s ecosystem (Helm, service mesh, etc.) |

```yaml
# ECS Task Definition — AWS-native, simpler mental model than a full K8s Deployment
{
  "family": "order-service",
  "containerDefinitions": [{
    "name": "order-service",
    "image": "myregistry/order-service:1.0",
    "cpu": 256, "memory": 512,
    "portMappings": [{"containerPort": 8080}]
  }],
  "requiresCompatibilities": ["FARGATE"]   # serverless — no EC2 node management at all
}
```

**Real decision I've made:** "For a team already deep in Kubernetes (using Helm, MOD7_003/004 patterns, potentially planning a multi-cloud or hybrid strategy), EKS is the natural fit — you get the exact same K8s primitives and tooling we'd use anywhere else. For a smaller team fully committed to AWS with no multi-cloud ambition, wanting to minimize the genuinely real operational complexity of running Kubernetes (even managed), ECS with Fargate is a legitimate, simpler choice — you give up some ecosystem breadth (no native Helm, smaller third-party tooling ecosystem) in exchange for a notably shallower learning curve and less cluster-level operational surface area to own."

---

### Q5. Lambda — what's it actually good for, and what are the real limitations that determine when NOT to use it?
**Answer:** Lambda runs your code **only in response to an event**, scales automatically to zero when idle, and you pay strictly **per invocation and execution duration** — no server ever sits idle waiting for traffic.

```java
public class OrderNotificationHandler implements RequestHandler<SQSEvent, Void> {
    public Void handleRequest(SQSEvent event, Context context) {
        for (SQSMessage message : event.getRecords()) {
            OrderPlacedEvent order = parseJson(message.getBody());
            emailService.sendOrderConfirmation(order);
        }
        return null;
    }
}
```

**Genuinely good fits:** event-driven glue logic (S3 upload triggers image resize), lightweight webhook handlers, scheduled/cron-style jobs, processing messages off an SQS queue at variable/bursty volume, infrequent batch jobs.

**Real limitations that should drive the "not this" decision, not just theoretical concerns:**
- **Cold starts** — an idle function's first invocation pays a startup latency penalty (JVM cold start is notably worse than Node/Python) — a real problem for latency-sensitive, user-facing synchronous request paths.
- **Execution time limit** (15 minutes max) — genuinely long-running batch processing doesn't fit.
- **Stateless by design** — no in-memory state persists reliably between invocations; anything needing to hold state (session data, in-memory caches) needs external storage (Redis, DynamoDB).
- **Vendor lock-in risk** — Lambda-specific handler signatures and event source integrations make migrating away from AWS a genuinely non-trivial rewrite, unlike a portable containerized service.
- **Harder local testing/debugging** compared to a normal running service — though tools like SAM/LocalStack narrow this gap somewhat.

**Senior-level answer:** "I use Lambda specifically for **event-driven, bursty, or infrequent** workloads where the pay-per-invocation model and zero idle cost are a genuine win — not as a default compute choice. For a core, high-traffic, latency-sensitive synchronous API (our main checkout flow, say), I'd keep that on always-warm compute (ECS/EKS) specifically to avoid cold-start latency variance hitting real users — mixing the two deliberately based on each workload's actual shape, rather than picking one paradigm for the whole system."

---

### Q6. SQS vs SNS — what's the actual difference, and how do they combine in a real fan-out architecture?
**Answer:** **SQS (Simple Queue Service)** is a **point-to-point message queue** — one message, consumed by (typically) one consumer, removed once processed; this is AWS's managed equivalent of RabbitMQ-style queuing (MOD5_007 Q1). **SNS (Simple Notification Service)** is a **pub/sub topic** — one published message, delivered to **every subscriber** of that topic.

**The classic, very commonly asked combination — SNS fan-out to multiple SQS queues:**
```
OrderService → publishes to SNS Topic "order-events"
                       │
        ┌──────────────┼──────────────┐
        ▼               ▼               ▼
  SQS: fraud-queue  SQS: inventory-queue  SQS: notification-queue
  (Fraud-Service      (Inventory-Service    (Notification-Service
   consumes)            consumes)             consumes)
```

"SNS alone can push directly to some subscriber types (Lambda, HTTP endpoints), but fanning out to **SQS queues** specifically is the standard pattern when you want each downstream consumer to have its **own durable, independently-consumable queue** — if the Inventory-Service is down for 20 minutes, its messages just sit safely in its own SQS queue waiting, completely unaffected by whether Fraud-Service or Notification-Service are healthy. This gives you the same **decoupled, multi-consumer** benefit Kafka gives you (MOD5_007 Q1, Q5), using AWS-native managed services rather than running/operating Kafka yourself."

**Follow-up: "When would you actually choose SQS/SNS over Kafka?"** → SQS/SNS requires **zero operational overhead** (fully managed, no brokers/partitions to size or tune) and is often cheaper and simpler for moderate-throughput, straightforward pub/sub or queuing needs; Kafka's advantage is **replay/retention** (SQS messages are gone once consumed, no history to replay) and **much higher sustained throughput** for genuine event-streaming/event-sourcing use cases (MOD5_007 Q1, MOD4_003 Q9) — the choice mirrors the same Kafka-vs-RabbitMQ trade-off, just AWS-managed vs. self-run.

---

### Q7. What does RDS actually manage for you, and what do you still need to handle yourself even with a fully managed database?
**Answer:** RDS is a **managed relational database service** (Postgres, MySQL, Oracle, SQL Server) — AWS handles the underlying **infrastructure**: automated backups, patching, storage scaling, and (critically) **Multi-AZ failover**.

**What RDS genuinely takes off your plate:**
- Automated daily backups + point-in-time recovery, without you scripting `pg_dump` cron jobs yourself.
- OS/engine patching on a schedule you control, without manual patch management.
- **Multi-AZ deployment** — a synchronously-replicated standby in a different Availability Zone, with **automatic failover** if the primary fails, typically under a minute, without manual intervention.
- **Read Replicas** — asynchronously-replicated read-only copies you can point read-heavy traffic at, offloading the primary.

**What you're still fully responsible for, even on a managed database — this is exactly the follow-up depth interviewers want:**
- **Schema design and query performance** — RDS doesn't stop you from writing a bad query or missing an index; that's still entirely on you (SQL performance tuning, MOD... database module).
- **Connection pool management** — a burst of new connections from a scaling fleet of app pods can still exhaust the database's max connection limit; you still need proper connection pooling (HikariCP) and potentially **RDS Proxy** to pool/multiplex connections in front of the database itself.
- **Capacity planning** — choosing the right instance size/storage type for actual load, and monitoring for when you've outgrown it, is still an ongoing operational responsibility, not something RDS decides for you.
- **Application-level resilience to failover** — even with automatic Multi-AZ failover, there's a brief window (seconds to under a minute) where the database is unavailable during the actual failover — the application still needs sane connection-retry behavior (MOD4_004 Q7-Q8's timeout/retry discipline) to ride that out gracefully rather than surfacing hard errors to users.

**Senior talking point:** "'Managed' significantly reduces **operational** burden — I'm not patching an OS or scripting backups — but it doesn't remove **application-level responsibility** for how the database is actually used. I've seen teams treat 'it's RDS, it's managed' as license to skip connection pool tuning entirely, and then be surprised when a traffic spike exhausts `max_connections` well before RDS's own infrastructure was ever close to its limits."

---

## SECTION 4: AZURE — AKS, SERVICE BUS, KEY VAULT, API MANAGEMENT

### Q8. Give the Azure equivalents to the AWS services above, and note anything genuinely different in behavior, not just naming.
**Answer:**
| AWS | Azure | Notable difference worth knowing |
|---|---|---|
| EKS | **AKS** (Azure Kubernetes Service) | Same core Kubernetes; AKS's control plane is free (you only pay for worker nodes) — a real cost difference from EKS's per-cluster control plane fee |
| SQS + SNS | **Azure Service Bus** | Service Bus natively supports **both** queue and pub/sub (Topics/Subscriptions) in **one service**, rather than AWS's two separate services (SQS + SNS) combined for fan-out |
| Secrets Manager + KMS | **Azure Key Vault** | Key Vault combines secrets, keys, AND certificate management in one service, where AWS splits this across Secrets Manager, KMS, and ACM |
| API Gateway | **Azure API Management (APIM)** | APIM leans more heavily into full API lifecycle/developer-portal features (API versioning, a self-service developer portal, built-in analytics) out of the box, vs. AWS API Gateway's more minimal, routing-focused default feature set |

```java
// Azure Service Bus — Topics/Subscriptions give you SNS+SQS fan-out in ONE service
@ServiceBusListener(topicName = "order-events", subscriptionName = "fraud-service-sub")
public void handleOrderEvent(OrderPlacedEvent event) {
    fraudDetectionService.evaluate(event);
}
```

```java
// Azure Key Vault — fetching a secret, analogous to AWS Secrets Manager (Q3)
@Value("${db-password}")   // Spring Cloud Azure auto-resolves this from Key Vault via configured property source
private String dbPassword;
```

**Senior-level answer on the genuinely meaningful differences, not just name-matching:** "The one difference I'd actually call out unprompted in an interview is Service Bus's **unified queue+topic model** — in AWS you deliberately combine two separate services (SNS fan-out into multiple SQS queues) to get the same decoupled multi-consumer pattern; Service Bus gives you Topics-with-Subscriptions as one native concept, which is a genuinely simpler mental model for the same architectural pattern (MOD5_007 Q1's pub/sub vs queue distinction, unified in one Azure service). Beyond naming, the underlying **architectural decisions** — least privilege IAM, database-per-service, resilience patterns — stay identical regardless of cloud provider; the provider choice changes *which specific managed service* implements a pattern, not whether the pattern itself applies."

**Follow-up: "Is choosing between AWS and Azure usually a technical decision?"** → Rarely, in my experience — it's far more often driven by **existing organizational investment** (enterprise agreements, existing Active Directory/identity integration, existing team expertise) than by a genuine feature gap between the two for standard microservices workloads; both provide broadly equivalent managed-service coverage for the patterns that actually matter architecturally.

---

## SECTION 5: MANAGED SERVICES & CLOUD-NATIVE ARCHITECTURE

### Q9. What does "cloud-native" actually mean as an architectural philosophy, and how does preferring managed services change how you design a system?
**Answer:** Cloud-native isn't just "runs in the cloud" — it's an architectural approach that **deliberately leans on managed services and cloud-provider primitives** (rather than self-hosting and operating equivalent infrastructure yourself) specifically to offload undifferentiated operational work, so engineering effort concentrates on the business logic that actually differentiates the product (echoing the Generic-subdomain framing from MOD4_001 Q1 — undifferentiated infrastructure work is a textbook Generic subdomain, worth buying/using managed rather than building).

**The trade-off, stated honestly, since "always use managed services" isn't a complete answer:**
| Favor managed services when... | Favor self-hosted/rolled-your-own when... |
|---|---|
| The service is genuinely undifferentiated infra (message queue, database, secrets store) | You need capabilities the managed offering genuinely doesn't support |
| Team wants to minimize operational burden, focus engineering time on the product | Multi-cloud portability is a hard organizational requirement (managed services are often provider-specific) |
| Team lacks deep operational expertise in that specific infra (e.g., running Kafka well is genuinely hard) | Cost at very large scale tips in favor of self-hosting (managed services carry a real markup at high volume) |
| Built-in HA/backup/patching is worth the (often modest) cost premium | Regulatory/compliance requirements mandate specific control over infrastructure the managed offering can't satisfy |

**Senior-level answer:** "My default bias is genuinely toward managed services — running a highly-available Kafka cluster well, or a Postgres cluster with proper failover, is real, ongoing, specialized operational work, and unless that operational expertise is itself something the business needs to own (rare), I'd rather pay AWS/Azure's markup and redirect that engineering time toward the actual product. That said, I don't treat it as an unconditional rule — I've been in a specific regulated environment where data residency and infrastructure-control compliance requirements genuinely ruled out certain managed offerings, and self-hosting (or a more constrained managed option) was the only compliant choice. The decision should be an explicit, stated trade-off (ops burden vs. cost vs. control vs. portability) — not a reflexive default in either direction."

**Cloud-native architecture principles worth naming as a closing summary, tying the whole module together:** stateless, horizontally-scalable services (so any instance can be replaced/scaled freely); externalized configuration and secrets (never baked into the image, MOD2_002-style profiles/config, Q3 here); managed data stores instead of self-hosted databases where practical (Q7); infrastructure as code for reproducibility (not covered in depth here, but the natural companion to everything in this module); and designing explicitly for failure (MOD4_004 Q7-Q8's timeout/retry/circuit-breaker discipline) since managed services still have their own availability SLAs, not 100% uptime guarantees.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between an IAM Role and an IAM User?" → A User represents a long-lived human/application identity with (often static) credentials; a Role is assumed **temporarily** (by a service, a user, or federated identity) and issues **short-lived, auto-rotated** credentials — roles are strongly preferred for anything a service assumes programmatically (Q2), specifically because there's no long-lived secret to leak.
- "What's the difference between Multi-AZ and Multi-Region for RDS?" → Multi-AZ (Q7) protects against a **single data-center-level failure** within one region, with synchronous replication and automatic failover; Multi-Region (via Read Replicas across regions, or Aurora Global Database) protects against an **entire region** being unavailable, but typically involves asynchronous replication and NOT automatic failover — a meaningfully bigger, usually manually-triggered, DR event.
- "Does Fargate remove the need to think about resource requests/limits?" → No — you still specify CPU/memory for the task (similar in spirit to Kubernetes resource requests, MOD7_003 Q2), you just don't manage the underlying EC2 instances those tasks run on; right-sizing still matters for cost and performance.
- "What's the 'shared responsibility model'?" → The cloud provider secures the underlying infrastructure (physical security, hypervisor, network fabric); the customer remains responsible for securing what they configure on top of it — IAM policies, security group rules, data encryption choices, patching their own OS/application layer where applicable — a managed service does NOT mean security is entirely the provider's problem.
- "Can you use SQS for exactly-once processing?" → SQS Standard queues are at-least-once by default (occasional duplicates possible); **SQS FIFO queues** provide exactly-once processing and strict ordering within a message group, at some throughput cost — same idempotent-consumer discipline (MOD4_004 Q5, MOD5_007 Q6) still applies as the practical, robust default regardless of which queue type is used.
- "What's a VPC endpoint, and why does it matter for security?" → Lets resources in a private subnet reach an AWS service (S3, DynamoDB, Secrets Manager) **without traversing the public internet at all** — traffic stays entirely within AWS's network — a meaningful security improvement over routing that traffic out through a NAT Gateway to the public internet and back.

---

*Study tip: The "what does managed still leave YOU responsible for" framing (Q3's rotation/audit gap in raw K8s Secrets, Q7's connection pooling and query performance on RDS) is where interviewers most reliably separate candidates who've clicked through the AWS console from those who've actually operated production workloads on managed cloud infrastructure and hit the boundaries of what "managed" really covers. Being able to state the honest trade-off for managed-vs-self-hosted (Q9), rather than reflexively saying "always use managed services," is the clearest senior signal in this entire module.*
