# 📘 Java Full Stack Interview Prep — Master Index
> `D:\Vishal\Interview\prep\`
> Legend: ✅ Complete · 🔲 Planned

---

## MODULE 1 — CORE JAVA ✅🔄

| File | Topic | Q Count | Status |
|---|---|---|---|
| `001_Core_Java_Fundamentals_Interview_Prep.md` | JVM · Memory · GC · ClassLoaders · Types · Pass-by-Value | 29 | ✅ |
| `002_OOP_Interview_Prep.md` | Encapsulation · Inheritance · Polymorphism · Abstract · Cloning | 26 | ✅ |
| `003_Exception_Handling_Interview_Prep.md` | Checked/Unchecked · try-catch · Custom · Chaining | 21 | ✅ |
| `004_Collections_Framework_Interview_Prep.md` | List/Set/Map · HashMap Internals · Iterators · equals/hashCode | 32 | ✅ |
| `005_Multithreading_and_Concurrency_Interview_Prep.md` | Threads · Locks · Executor · CompletableFuture · ThreadLocal · Fork/Join | 42 | ✅ |
| `006_Java_8_Features_Interview_Prep.md` | Lambdas · Streams · Collectors · Optional · Method Refs · Date API | — | ✅ |
| `007_Java_8_Programming_Problems_Interview_Prep.md` | Stream coding problems · Lambda exercises | — | ✅ |
| `008_Java_9_26_Features_Interview_Prep.md` | Records · Sealed · Pattern Matching · Modules · Text Blocks | — | ✅ |
| `009_Java_IO_and_NIO_Interview_Prep.md` | IO Streams · NIO · Channels · Buffers · File API | — | ✅ |
| `010_Design_Patterns_Interview_Prep_Benchmark.md` | Creational · Structural · Behavioral · Patterns in Spring | — | ✅ |
| `011_SOLID_Principles_Interview_Prep_Benchmark.md` | SRP · OCP · LSP · ISP · DIP · Real examples | — | ✅ |
| `012_Generics_Interview_Prep.md` | Type Erasure · Wildcards · PECS · Bounded Types | — | 🔲 |
| `013_Reflection_Serialization_Interview_Prep.md` | Reflection API · Serialization · serialVersionUID · transient | — | 🔲 |
| `014_String_Immutability_Interview_Prep.md` | String Pool · StringBuilder · Compact Strings | — | 🔲 |

---

## MODULE 2 — SPRING BOOT 🔲

| File | Topic | Status |
|---|---|---|
| `015_Spring_Core_Interview_Prep.md` | IoC · DI · Bean Lifecycle · Scopes · Autowired | 🔲 |
| `016_Spring_Boot_Interview_Prep.md` | Auto-config · Starters · Profiles · Properties | 🔲 |
| `017_Spring_MVC_Interview_Prep.md` | REST APIs · Validation · Exception Handling | 🔲 |
| `018_Spring_Security_Interview_Prep.md` | JWT · OAuth2 · Security Filters | 🔲 |
| `019_Hibernate_JPA_Interview_Prep.md` | ORM · Caching · Transactions · N+1 Problem | 🔲 |
| `020_Spring_AOP_Actuator_Interview_Prep.md` | AOP · Actuator · Spring Cloud · Feign · Gateway | 🔲 |

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
| `028_DB_Performance_Interview_Prep.md` | Partitioning · Normalization · Tuning | 🔲 |

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
| `032_System_Design_HLD_Interview_Prep.md` | Scalability · Caching · Load Balancer · CDN · Rate Limiting | 🔲 |
| `033_System_Design_LLD_Interview_Prep.md` | SOLID · Design Patterns · API Design · DB Design | 🔲 |
| `034_System_Design_Real_Systems_Interview_Prep.md` | Uber · WhatsApp · Netflix · Payment · Order Management | 🔲 |

---

## MODULE 7 — DEVOPS 🔲

| File | Topic | Status |
|---|---|---|
| `035_Docker_Interview_Prep.md` | Dockerfile · Docker Compose · Networking | 🔲 |
| `036_Kubernetes_Interview_Prep.md` | Pods · Deployments · Services · Ingress · ConfigMaps | 🔲 |
| `037_CICD_AWS_Interview_Prep.md` | Jenkins · GitHub Actions · EC2 · S3 · ECS · EKS | 🔲 |

---

## MODULE 8 — BEHAVIORAL 🔲

| File | Topic | Status |
|---|---|---|
| `038_Behavioral_Interview_Prep.md` | STAR · Leadership · Conflict · Production Issues · Ownership | 🔲 |

---

## 🔥 Must-Revise Before Any Interview

| Priority | Question | File |
|---|---|---|
| 🔴 | HashMap internal working — hashing, treeification, resize | `004` → §4 Q9 |
| 🔴 | ThreadLocal leak in pooled threads | `005` → §11 Q36 |
| 🔴 | Why NOT to use `Executors` factory methods | `005` → §6 Q18 |
| 🔴 | Pass by value — swap method proof | `001` → §6 Q26 |
| 🔴 | equals()/hashCode() contract & HashSet identity bug | `004` → §9 Q23 |
| 🔴 | Static/instance block + constructor execution order | `002` → §4 Q16 |
| 🔴 | Exception chaining — never swallow the cause | `003` → §5 Q17 |
| 🟠 | Static method hiding vs overriding output trap | `002` → §3 Q13 |
| 🟠 | CompletableFuture silent exception swallow | `005` → §7 Q23 |
| 🟠 | Fail-fast ConcurrentModificationException fix | `004` → §8 Q21 |
| 🟠 | Fork/Join work-stealing mechanism | `005` → §12 Q42 |
| 🟠 | LRU cache using LinkedHashMap | `004` → §6 Q17 |

---

*Module 1: 11 of 14 files on disk · Module 2–8: Planned*
