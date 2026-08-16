# 📘 Java Full Stack Interview Prep — Master Index
> Senior Java Full Stack Developer (8+ Years) — Interview Preparation
> Legend: ✅ Complete · 🔲 Planned

---

## MODULE 1 — CORE JAVA ✅

| File | Topic | Status |
|---|---|---|
| `001_Core_Java_Fundamentals_Interview_Prep.md` | JVM · Memory · GC · ClassLoaders · Types · Pass-by-Value | ✅ |
| `002_OOP_Interview_Prep.md` | Encapsulation · Inheritance · Polymorphism · Abstract · Cloning | ✅ |
| `003_String_Immutability_Interview_Prep.md` | String Pool · `intern()` · StringBuilder · Compact Strings | ✅ |
| `004_Generics_Interview_Prep.md` | Type Erasure · Wildcards · PECS · Bounded Types | ✅ |
| `005_Collections_Framework_Interview_Prep.md` | List/Set/Map · HashMap Internals · Iterators · equals/hashCode | ✅ |
| `006_Exception_Handling_Interview_Prep.md` | Checked/Unchecked · try-catch · Custom · Chaining | ✅ |
| `007_Java_IO_and_NIO_Interview_Prep.md` | IO Streams · NIO · Channels · Buffers · File API | ✅ |
| `008_Multithreading_and_Concurrency_Interview_Prep.md` | Threads · Locks · Executor · CompletableFuture · ThreadLocal · Fork/Join | ✅ |
| `009_Java_8_Features_Interview_Prep.md` | Lambdas · Streams · Collectors · Optional · Method Refs · Date API | ✅ |
| `010_Java_8_Programming_Problems_Interview_Prep.md` | Stream coding problems · Lambda exercises | ✅ |
| `011_Java_9_26_Features_Interview_Prep.md` | Records · Sealed · Pattern Matching · Modules · Virtual Threads | ✅ |
| `012_SOLID_Principles_Interview_Prep.md` | SRP · OCP · LSP · ISP · DIP · Real examples | ✅ |
| `013_Design_Patterns_Interview_Prep.md` | Creational · Structural · Behavioral · Patterns in Spring | ✅ |
| `014_Reflection_Serialization_Interview_Prep.md` | Reflection API · Serialization · serialVersionUID · transient | ✅ |

**Module 1 status: 14 of 14 files complete.**

---

## MODULE 2 — SPRING 🔲

| File | Topic | Status |
|---|---|---|
| `015_Spring_Core_Interview_Prep.md` | IoC · DI · Bean Lifecycle · Scopes · Autowired | 🔲 |
| `016_Spring_Boot_Interview_Prep.md` | Auto-config · Starters · Profiles · Properties | 🔲 |
| `017_Spring_MVC_Interview_Prep.md` | REST APIs · Validation · Exception Handling | 🔲 |
| `018_Spring_Security_Interview_Prep.md` | JWT · OAuth2 · Security Filters | 🔲 |
| `019_Hibernate_JPA_Interview_Prep.md` | ORM · Caching · Transactions · N+1 Problem | 🔲 |
| `020_Spring_AOP_Actuator_Cloud_Interview_Prep.md` | AOP · Actuator · Spring Cloud · Feign · Gateway · Config Server · Eureka · Resilience4j | 🔲 |

---

## MODULE 3 — MICROSERVICES 🔲

| File | Topic | Status |
|---|---|---|
| `021_Microservices_Architecture_Interview_Prep.md` | Design Principles · API Gateway · Service Discovery | 🔲 |
| `022_Fault_Tolerance_Interview_Prep.md` | Circuit Breaker · Bulkhead · Retry · Rate Limiting | 🔲 |
| `023_Kafka_EventDriven_Interview_Prep.md` | Kafka · RabbitMQ · Outbox · Exactly-Once · DLQ | 🔲 |
| `024_Microservices_Patterns_Interview_Prep.md` | Saga · CQRS · Idempotency · Event Ordering | 🔲 |
| `025_Distributed_Logging_Tracing_Interview_Prep.md` | ELK · Zipkin · Sleuth · Correlation IDs | 🔲 |

---

## MODULE 4 — DATABASE 🔲

| File | Topic | Status |
|---|---|---|
| `026_SQL_Advanced_Interview_Prep.md` | Indexes · Joins · Window Functions · CTE · Execution Plan | 🔲 |
| `027_DB_Transactions_Interview_Prep.md` | Isolation Levels · Optimistic/Pessimistic Locking · ACID | 🔲 |
| `028_DB_Performance_Interview_Prep.md` | Partitioning · Normalization · Denormalization · Tuning | 🔲 |

---

## MODULE 5 — ANGULAR 🔲

| File | Topic | Status |
|---|---|---|
| `029_Angular_Core_Interview_Prep.md` | Components · Lifecycle · DI · Change Detection | 🔲 |
| `030_Angular_RxJS_Interview_Prep.md` | Observables · Subjects · Signals · Operators | 🔲 |
| `031_Angular_Advanced_Interview_Prep.md` | Routing · Lazy Loading · Forms · State Management | 🔲 |

---

## MODULE 6 — SYSTEM DESIGN 🔲

| File | Topic | Status |
|---|---|---|
| `032_System_Design_HLD_Interview_Prep.md` | Scalability · Caching · Redis · Load Balancer · CDN · Rate Limiting | 🔲 |
| `033_System_Design_LLD_Interview_Prep.md` | SOLID · Design Patterns · API Design · DB Design | 🔲 |
| `034_System_Design_Real_Systems_Interview_Prep.md` | Uber · WhatsApp · Netflix · Payment · Order Management · Inventory · Notification | 🔲 |

---

## MODULE 7 — DEVOPS 🔲

| File | Topic | Status |
|---|---|---|
| `035_Docker_Interview_Prep.md` | Dockerfile · Docker Compose · Networking | 🔲 |
| `036_Kubernetes_Interview_Prep.md` | Pods · Deployments · Services · Ingress · ConfigMaps · Secrets | 🔲 |
| `037_CICD_AWS_Interview_Prep.md` | Jenkins · GitHub Actions · EC2 · S3 · IAM · ECS · EKS · CloudWatch | 🔲 |

---

## MODULE 8 — BEHAVIORAL 🔲

| File | Topic | Status |
|---|---|---|
| `038_Behavioral_Interview_Prep.md` | STAR · Leadership · Conflict · Production Issues · Ownership · Mentoring | 🔲 |

---

## 🔥 Must-Revise Before Any Interview

| Priority | Question | File |
|---|---|---|
| 🔴 | HashMap internal working — hashing, treeification, resize | `005` → §4 Q9 |
| 🔴 | ThreadLocal leak in pooled threads | `008` → §11 Q36 |
| 🔴 | Why NOT to use `Executors` factory methods | `008` → §6 Q18 |
| 🔴 | Pass by value — swap method proof | `001` → §6 Q27 |
| 🔴 | equals()/hashCode() contract & HashSet identity bug | `005` → §9 Q23 |
| 🔴 | Static/instance block + constructor execution order | `002` → §4 Q16 |
| 🔴 | Exception chaining — never swallow the cause | `006` → §5 Q17 |
| 🟠 | Static method hiding vs overriding output trap | `002` → §3 Q13 |
| 🟠 | CompletableFuture silent exception swallow | `008` → §7 Q23 |
| 🟠 | Fail-fast ConcurrentModificationException fix | `005` → §8 Q21 |
| 🟠 | Fork/Join work-stealing mechanism | `008` → §12 Q42 |
| 🟠 | LRU cache using LinkedHashMap | `005` → §6 Q17 |
| 🟠 | String Pool vs `new String()` vs `intern()` | `003` → §3–4 |
| 🟠 | PECS — Producer Extends, Consumer Super | `004` → §9 Q22–24 |
| 🟠 | SOLID — DIP vs Dependency Injection | `012` → §7 Q22 |
| 🟠 | Singleton double-checked locking + `volatile` | `013` → §1 Q4 |
| 🟠 | `serialVersionUID` and why to declare it explicitly | `014` → §13 Q37 |
| 🟠 | Virtual Threads vs Platform Threads | `011` → Q32 |

---

## Recommended Learning Order

```text
001 Core Java Fundamentals
       ↓
002 OOP
       ↓
003 String Immutability
       ↓
004 Generics
       ↓
005 Collections Framework
       ↓
006 Exception Handling
       ↓
007 Java I/O & NIO
       ↓
008 Multithreading & Concurrency
       ↓
009 Java 8 Features
       ↓
010 Java 8 Programming Problems
       ↓
011 Java 9–26 Features
       ↓
012 SOLID Principles
       ↓
013 Design Patterns
       ↓
014 Reflection & Serialization
       ↓
015–020 Spring / Spring Boot / Spring Security / Hibernate / Spring Cloud
       ↓
021–025 Microservices / Fault Tolerance / Kafka / Patterns / Observability
       ↓
026–028 Database (SQL / Transactions / Performance)
       ↓
029–031 Angular (Core / RxJS / Advanced)
       ↓
032–034 System Design (HLD / LLD / Real Systems)
       ↓
035–037 DevOps (Docker / Kubernetes / CI-CD & AWS)
       ↓
038 Behavioral
```

## Interview Preparation Benchmark

For each topic, aim to answer beyond a definition:

> **What is it? → Why does it exist? → What problem does it solve? → How does it work? → Example → Trade-offs → When to use → When not to use → Production scenario → Interview trap → Senior follow-up**

For an **experienced Senior Java / Spring Boot / Microservices** interview, prioritize understanding and trade-offs over memorizing definitions.

## Suggested Study Method

### Pass 1 — Understand
Read each module in order and make sure you can explain the core concepts without looking at the answer.

### Pass 2 — Interview
For every major question, practice a **60–90 second spoken answer**.

### Pass 3 — Senior Depth
For important topics, be able to discuss internal working, performance implications, memory behavior, thread-safety, failure modes, trade-offs, production use cases, common mistakes, and alternatives.

### Pass 4 — Coding
Use the Java 8 Programming Problems module (010) after completing Java 8 Features (009).

### Pass 5 — Design
Study SOLID (012) before Design Patterns (013) so that patterns are understood as design tools rather than memorized templates.

---

## Module Checklist

- [x] 001 Core Java Fundamentals
- [x] 002 OOP
- [x] 003 String Immutability
- [x] 004 Generics
- [x] 005 Collections Framework
- [x] 006 Exception Handling
- [x] 007 Java I/O & NIO
- [x] 008 Multithreading & Concurrency
- [x] 009 Java 8 Features
- [x] 010 Java 8 Programming Problems
- [x] 011 Java 9–26 Features
- [x] 012 SOLID Principles
- [x] 013 Design Patterns
- [x] 014 Reflection & Serialization
- [ ] 015–020 Spring / Spring Boot / Security / Hibernate / Cloud
- [ ] 021–025 Microservices
- [ ] 026–028 Database
- [ ] 029–031 Angular
- [ ] 032–034 System Design
- [ ] 035–037 DevOps
- [ ] 038 Behavioral

## Senior Interview Priority

For your target senior-level Java interviews, give extra attention to:

1. Collections internals and concurrency
2. Multithreading and Java Memory Model
3. Streams and CompletableFuture
4. Modern Java, especially Java 17/21/25-era features
5. Immutability and thread-safe design
6. SOLID and practical design patterns
7. I/O/NIO and resource management
8. Generics, type erasure, wildcards, and PECS
9. Reflection and serialization trade-offs/security
10. Production troubleshooting and performance reasoning

---

**Status:** Module 1 (Core Java) — 14 of 14 files complete. Modules 2–8 — planned.
