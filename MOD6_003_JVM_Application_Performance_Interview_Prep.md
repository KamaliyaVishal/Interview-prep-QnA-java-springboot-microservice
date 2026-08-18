# JVM & Application Performance — Interview Prep
---

## SECTION 1: HEAP DUMPS, THREAD DUMPS & GC LOGS

### Q1. What's the difference between a heap dump and a thread dump, and when would you take each?
**Answer:**
- **Heap dump** — a full snapshot of **every object on the heap** at a point in time (all live objects, their fields, and reference chains). Used for diagnosing **memory issues** — memory leaks, unexpectedly high memory usage, understanding what's actually consuming heap space.
- **Thread dump** — a snapshot of **every thread's current stack trace and state** (RUNNABLE, BLOCKED, WAITING, TIMED_WAITING) at a point in time. Used for diagnosing **concurrency/hang issues** — deadlocks, thread pool exhaustion, a request that's stuck/slow, high CPU from a spinning thread.

| | Heap Dump | Thread Dump |
|---|---|---|
| Captures | All objects on the heap | All threads' stacks and states |
| Diagnoses | Memory leaks, OOM, excessive retention | Deadlocks, hangs, contention, stuck requests |
| Typical trigger | Heap usage climbing / OOM | High CPU, request timeouts, app appears frozen |
| Size/cost | Can be very large (GBs), heavier to capture | Lightweight, fast, safe to take repeatedly |

**Senior-level answer:**
> "A useful rule of thumb I give junior engineers: if the symptom is 'memory keeps growing,' reach for a heap dump; if the symptom is 'requests are hanging / CPU is pegged / the app looks frozen but memory is fine,' reach for a thread dump — they're diagnosing fundamentally different failure classes, and taking the wrong one wastes the incident's most valuable minutes. In practice I often take **both** together during a live incident, since a thread stuck holding a huge object graph can show up in either."

---

### Q2. How do you generate a heap dump, and what are the different ways to trigger one?
**Answer:**
```bash
# On-demand, against a running JVM
jmap -dump:live,format=b,file=heap.hprof <pid>

# Automatically on OutOfMemoryError — the single most useful flag for production
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/dumps/ -jar app.jar

# Via JDK Mission Control / jcmd (modern, preferred over jmap in recent JDKs)
jcmd <pid> GC.heap_dump /var/dumps/heap.hprof
```

**Senior-level answer:**
> "`-XX:+HeapDumpOnOutOfMemoryError` should be a **default flag on every production JVM**, not something you remember to add after the first OOM incident — without it, an OOM crash gives you a stack trace but no heap snapshot, and you're stuck trying to reproduce a memory issue that might take hours or days to recur. I've seen teams lose the ability to root-cause a production OOM entirely because this flag wasn't set, and by the time they added it and waited for a recurrence, the underlying code had already changed. It's a near-zero-cost, high-value flag to always have on."

---

### Q3. How do you generate a thread dump, and what should you look for when reading one?
**Answer:**
```bash
# jstack against a running JVM
jstack <pid> > threaddump.txt

# kill -3 sends the dump to stdout/the app's log file — works without extra tooling
kill -3 <pid>

# jcmd (modern preferred approach)
jcmd <pid> Thread.print
```

**What to look for:**
- **`BLOCKED` threads** — waiting to acquire a monitor lock another thread holds; many threads `BLOCKED` on the same lock is a strong contention/bottleneck signal.
- **Threads stuck in the same stack trace across multiple dumps** (take 2-3 dumps a few seconds apart) — a thread that's genuinely stuck (vs. just momentarily busy) will show an **identical or near-identical** stack across samples.
- **`"Found one Java-level deadlock"`** — `jstack` explicitly detects and reports classic two-thread lock-ordering deadlocks at the bottom of the dump.
- **Thread pool queue depth indirectly** — a large number of threads with names like `pool-1-thread-N` all `WAITING` on the same executor's queue often signals pool exhaustion.

**Senior-level answer:**
> "Taking a **single** thread dump only tells you a snapshot; taking **2–3 dumps a few seconds apart** and diffing them is what actually distinguishes 'this thread is doing real, changing work' from 'this thread is truly stuck on the exact same line' — that comparison is the single most useful technique for diagnosing a hang, and it's something I'd want a senior candidate to volunteer without being prompted."

---

### Q4. How do you enable GC logging, and what are the key things to look for in the logs?
**Answer:**
```bash
# Modern unified GC logging (JDK 9+)
java -Xlog:gc*:file=gc.log:time,uptime,level,tags -jar app.jar
```

```
[2026-08-18T10:15:02.123+0000][3.421s] GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->128M(1024M) 18.421ms
                                          ^ GC type          ^before->after (heap)      ^pause duration
```

**Key things to look for:**
- **Pause duration** — how long the application was actually stopped (STW — stop-the-world).
- **Frequency** — how often GCs occur; frequent young-gen GCs are normal, frequent **full/major** GCs are a red flag.
- **Heap reclaimed per cycle** (`512M->128M`) — if a GC barely reclaims memory (e.g., `900M->880M`), that's a strong memory-pressure/leak signal — the collector is working hard for little gain.
- **Full GC occurrences specifically** — these are typically much longer STW pauses than young-gen collections and directly correlate with visible application latency spikes/timeouts.

**Senior-level answer:**
> "I always cross-reference GC logs against the application's own latency metrics/traces from the same time window — a p99 latency spike that lines up exactly with a full GC pause is a very different root cause (and fix) than one that lines up with a downstream dependency being slow, but from the outside they can look identical as 'the app was slow at 10:15am.' GC logs are what let you tell those apart definitively rather than guessing."

---

## SECTION 2: GARBAGE COLLECTION ANALYSIS

### Q5. Explain generational garbage collection — young gen, old gen, and minor vs. major/full GC.
**Answer:** The generational hypothesis: **most objects die young** (short-lived, like a request-scoped DTO), so the heap is split into regions collected at different frequencies/costs:

```
Heap:
[ Young Generation ]                        [ Old Generation ]
  Eden | Survivor S0 | Survivor S1              (tenured objects)
     |
  New objects allocated here
     |
  Minor GC: Eden filled -> live objects copied to Survivor,
            dead objects reclaimed (fast, frequent, short pause)
     |
  Objects surviving several minor GCs -> PROMOTED to Old Gen
     |
  Old Gen fills up -> Major/Full GC (slower, longer pause, scans much more memory)
```

- **Minor GC** — collects only the young generation; fast and frequent because most objects there are already dead.
- **Major/Full GC** — collects the old generation (and typically the whole heap); much more expensive because it scans a larger, longer-lived object set — this is usually where visible application pause-time problems come from.

**Senior-level answer:**
> "The generational hypothesis is why this design works so well in practice for typical request/response workloads — but it's also exactly why certain patterns cause GC problems: objects that **almost** survive to old gen but don't (a cache with a slightly-too-long TTL, a large per-request buffer) cause needless promotion and copying overhead, sometimes called 'premature promotion.' Recognizing that a GC problem often traces back to an **allocation pattern** in application code, not just a GC-tuning-flags problem, is a good senior distinction to draw."

---

### Q6. Compare the major GC algorithms — G1, Parallel, ZGC, CMS — and when you'd choose each.
**Answer:**

| Collector | Design Goal | Typical Pause Times | When to Use |
|---|---|---|---|
| **Parallel GC** | Maximize throughput | Longer pauses (STW for major GC) acceptable | Batch jobs, throughput-first workloads where occasional longer pauses are fine |
| **G1 (Garbage-First)** | Balance throughput and predictable pause times | Target-configurable (e.g., `-XX:MaxGCPauseMillis=200`) | **Default since JDK 9** — good general-purpose choice for most web services |
| **ZGC** | Ultra-low pause times, scales to very large heaps | Sub-millisecond to low-single-digit ms, largely independent of heap size | Latency-sensitive services with large heaps (tens/hundreds of GB) where even G1's pauses are too long |
| **CMS (Concurrent Mark Sweep)** | Low pause, concurrent old-gen collection | Low, but suffers fragmentation over time | **Deprecated/removed** in modern JDKs (removed in JDK 14) — superseded by G1/ZGC |

**Senior-level answer:**
> "For the vast majority of Spring Boot services I'd default to **G1** and only reach for ZGC when there's a specific, measured pause-time problem G1 can't hit — ZGC's sub-millisecond pauses come from a more complex concurrent algorithm, and it's not automatically the right default just because 'lower pause time' sounds strictly better; G1 has a much longer production track record and simpler tuning surface for typical CRUD/API workloads. I'd want to see the actual GC log data showing G1 pause times **before** justifying a switch to ZGC — that's the kind of measured, evidence-first answer that separates real production tuning experience from cargo-culting the newest collector."

---

### Q7. What GC tuning flags/approaches would you actually reach for on a Spring Boot service under memory pressure?
**Answer:**
```bash
# Explicit heap sizing — avoid letting the JVM guess (default is often too conservative in containers)
-Xms2g -Xmx2g            # setting Xms == Xmx avoids heap resize pauses during warmup

# G1 pause time target — a target, not a hard guarantee
-XX:MaxGCPauseMillis=200

# Container-awareness (JDK 10+, but always verify in containerized environments)
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0   # let the JVM heap scale relative to the container's memory limit, not the host's
```

**Senior-level answer:**
> "The single most common misconfiguration I see in containerized Spring Boot deployments isn't a GC algorithm choice at all — it's the JVM either not respecting the container's memory **limit** (pre-JDK 10 container awareness issues) or being given a fixed `-Xmx` that doesn't match the container's actual allocated memory, leading to either OOM-killed pods or wasted headroom. Before touching GC algorithm choice, I'd always verify heap sizing is correctly derived from the container's actual memory limit — `MaxRAMPercentage` is the right lever for that in a Kubernetes environment where memory limits can vary across environments."

---

## SECTION 3: JFR/JMC & MEMORY LEAK DETECTION

### Q8. What is JFR (Java Flight Recorder), and how does JMC (JDK Mission Control) use it?
**Answer:** **JFR** is a low-overhead, built-in JVM profiling and event-collection framework (production-safe, typically <1-2% overhead) that records detailed runtime data — GC events, thread activity, allocation profiles, lock contention, method sampling — directly from the JVM. **JMC** is the GUI tool for **analyzing** JFR recordings — flame graphs, allocation hot spots, thread timelines, GC pause visualization.

```bash
# Start a JFR recording on app startup, capped duration and file size
java -XX:StartFlightRecording=duration=300s,filename=recording.jfr -jar app.jar

# Trigger a recording on a running JVM without restarting it
jcmd <pid> JFR.start duration=60s filename=recording.jfr
```

**Senior-level answer:**
> "The key differentiator versus most external profilers: JFR is built into the JVM itself and is specifically engineered to be **safe to run continuously in production** with negligible overhead, unlike traditional bytecode-instrumentation profilers that can be too heavy to leave on. That's what makes it genuinely useful for diagnosing intermittent production issues that are hard to reproduce in a dev/staging environment — you can leave a rolling JFR recording running and grab it retroactively after an incident, rather than needing to reproduce the issue live with a heavyweight profiler attached."

---

### Q9. How do you detect and diagnose a memory leak in a Java application?
**Answer:**

1. **Confirm it's actually a leak, not just normal usage** — plot heap usage (specifically **old-gen usage after full GCs**, not just raw heap) over a long period; a genuine leak shows a **sawtooth pattern trending steadily upward** — each GC reclaims some memory, but the post-GC baseline keeps climbing.

```
Heap usage over time (leak signature):
|      /\      /\        /\          /\
|     /  \    /  \      /    \      /    \
|    /    \  /    \    /      \    /      \       <- baseline after each GC keeps rising
|___/______\/______\__/________\__/________\____
     (each GC reclaims less relative to the trend)
```

2. **Take a heap dump** (Q2) once the pattern is confirmed, ideally close to (but before) OOM.
3. **Analyze it with Eclipse MAT (Memory Analyzer Tool) or JMC** — look at the **dominator tree** and **"leak suspects" report** — these tools identify the objects retaining the most memory and the **reference chains keeping them alive** (what's actually holding onto them and preventing GC).

**Senior-level answer:**
> "MAT's 'leak suspects' report is genuinely good at auto-flagging the likely culprit, but the report only tells you **what** is retained — the real diagnostic work is tracing the **GC roots path**, i.e., the specific reference chain from a GC root down to the leaking objects, because that chain tells you **where in the code** to actually fix the retention. I'd want to walk an interviewer through a real example: e.g., a `static Map<String, Session>` that entries are added to on login but never removed on logout — the dominator tree shows the Map dominating a huge chunk of retained heap, and the GC-root path shows it's reachable via a `static` field, which immediately tells you it'll never be collected regardless of GC algorithm, because static references are GC roots by definition."

---

### Q10. What are the most common causes of memory leaks in a Spring Boot / Java application?
**Answer:**

| Cause | Why It Leaks |
|---|---|
| **Unbounded static collections** | A `static Map`/`List` that's only ever added to (caches, listener registries) — static fields are GC roots, so anything reachable from them is never collected, no matter how unused it actually is |
| **Unclosed resources** | Connections, streams, file handles not closed (missing `try-with-resources`) — leaks native/off-heap resources and often heap-referenced wrapper objects too |
| **`ThreadLocal` not cleared** | Especially dangerous in thread-pool-based servers (Tomcat) — a `ThreadLocal` set per-request but never removed persists for the **life of the pooled thread**, not the request, silently accumulating |
| **Listener/callback registration without deregistration** | Event listeners added but never removed keep the registering object alive via the listener reference, even after it's logically "done" |
| **Inner classes holding implicit outer-class references** | A non-static inner class (or anonymous class) implicitly holds a reference to its enclosing instance — if the inner object outlives the intended scope (e.g., stored in a cache), it keeps the entire outer object alive too |
| **Caches without eviction policy** | A cache that only grows (no TTL, no max size, no LRU eviction) is functionally a memory leak by design |

```java
// Classic ThreadLocal leak in a pooled-thread environment
private static final ThreadLocal<UserContext> CONTEXT = new ThreadLocal<>();

public void handleRequest(UserContext ctx) {
    CONTEXT.set(ctx);
    try {
        // ... process request
    } finally {
        CONTEXT.remove();   // REQUIRED — without this, ctx (and everything it references) 
    }                        // stays attached to this pooled thread indefinitely
}
```

**Senior-level answer:**
> "I'd call out the `ThreadLocal` case specifically because it's the one I've seen bite experienced engineers the most — it's counterintuitive that something scoped 'per-request' in your mental model is actually scoped 'per-thread' at the JVM level, and in a thread-pool-based server that distinction is the entire bug. This is the exact same underlying mechanism as the correlation-ID MDC-clearing issue from the observability topic — MDC is itself backed by a `ThreadLocal` — so it's a good example of a single root-cause pattern showing up across multiple areas of the stack."

---

## SECTION 4: OOM vs STACKOVERFLOW & DEADLOCK DETECTION

### Q11. What's the difference between `OutOfMemoryError` and `StackOverflowError`, and how do you fix each?
**Answer:**
- **`OutOfMemoryError`** — the JVM cannot allocate more memory in a given memory area (heap, metaspace, etc.) because it's genuinely exhausted, even after a full GC attempt. Root cause: too much live data retained (leak, or genuinely needing more heap), too many classes loaded (metaspace), or too many native/off-heap allocations.
- **`StackOverflowError`** — a single thread's **call stack** exceeds its configured size — almost always caused by **uncontrolled/infinite recursion** (missing or incorrect base case), occasionally by genuinely very deep legitimate recursion on a default-sized stack.

```java
// StackOverflowError — classic missing base case
public int factorial(int n) {
    return n * factorial(n - 1);   // never terminates -> unbounded stack growth
}

// Fix: correct base case, or convert to iteration for genuinely deep cases
public int factorial(int n) {
    if (n <= 1) return 1;          // base case
    return n * factorial(n - 1);
}
```

**Fixes:**
| | OutOfMemoryError | StackOverflowError |
|---|---|---|
| First step | Heap dump analysis (Q9) — is it a leak or genuine undersizing? | Check the stack trace for a **repeating pattern** — that's almost always the recursive call site |
| Fix | Fix the leak, or increase `-Xmx` if usage is legitimately that high | Fix the recursion's base case/termination condition; convert to iteration if the recursion depth is inherently unbounded by input size |

**Senior-level answer:**
> "The distinction I'd emphasize: `OutOfMemoryError` is almost never fixed by 'just increase the heap' as a first response — that can mask a genuine leak and just delay the crash to a less convenient time. `StackOverflowError` is almost never fixed by increasing `-Xss` (thread stack size) for the same reason — it's nearly always a logic bug in recursion, not a sizing problem, and increasing stack size just delays the crash and wastes memory per thread across potentially thousands of threads."

---

### Q12. What are the different types of `OutOfMemoryError`, and what does each specifically indicate?
**Answer:**

| Error Message | Meaning |
|---|---|
| `java.lang.OutOfMemoryError: Java heap space` | The heap itself is exhausted — classic leak or genuine undersizing |
| `java.lang.OutOfMemoryError: Metaspace` | Too many classes loaded (class metadata) — common with **classloader leaks**, e.g., repeated hot-redeployment in an app server, or dynamic proxy/bytecode-generation libraries creating classes without bound |
| `java.lang.OutOfMemoryError: GC overhead limit exceeded` | The JVM is spending **>98% of CPU time on GC** while reclaiming **<2% of heap** — it's not technically "full" yet, but GC is thrashing uselessly — a strong leak signal that fires *before* a literal heap-space OOM |
| `java.lang.OutOfMemoryError: Direct buffer memory` | Off-heap (native) memory used by `ByteBuffer.allocateDirect()` or NIO is exhausted — a heap dump won't show this, since it's not heap memory; needs native memory tracking (`-XX:NativeMemoryTracking`) instead |
| `java.lang.OutOfMemoryError: unable to create new native thread` | The **OS** can't create more threads for the JVM — usually a thread leak (pool misconfiguration, or code creating unbounded raw `Thread`s) rather than a heap issue at all |

**Senior-level answer:**
> "This is a good one to know cold because the **specific error text tells you which tool to reach for** — heap space and GC-overhead-limit point you to a heap dump; metaspace points you to classloader/class-generation analysis; direct buffer memory means a heap dump is actually the **wrong** tool and you need native memory tracking instead; and 'unable to create native thread' means the problem isn't memory at all, it's thread count, so you'd go straight to a thread dump. Misreading which OOM variant you have and reaching for the wrong diagnostic tool wastes real incident time."

---

### Q13. How do you detect and analyze a deadlock in a Java application?
**Answer:** `jstack` (or `jcmd Thread.print`) **automatically detects** classic two-or-more-thread lock-ordering deadlocks and prints an explicit section at the end of the dump:

```
Found one Java-level deadlock:
=============================
"Thread-A":
  waiting to lock monitor 0x00007f... (object 0x000000076..., a java.lang.Object),
  which is held by "Thread-B"
"Thread-B":
  waiting to lock monitor 0x00007f... (object 0x000000075..., a java.lang.Object),
  which is held by "Thread-A"

Java stack information for the threads listed above:
"Thread-A": at com.example.TransferService.transfer(TransferService.java:24)
            - waiting to lock <0x...> (a java.lang.Object)
            - locked <0x...> (a java.lang.Object)
```

```java
// Classic deadlock — two threads locking two objects in opposite order
// Thread A: transfer(accountX, accountY)   Thread B: transfer(accountY, accountX)
public void transfer(Account from, Account to, BigDecimal amount) {
    synchronized (from) {           // Thread A locks X, Thread B locks Y
        synchronized (to) {         // Thread A now wants Y (held by B), Thread B wants X (held by A) -> deadlock
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Senior-level answer:**
> "This exact 'transfer between two accounts' example is the textbook deadlock case, and I'd use it to explain the fix directly: **consistent lock ordering** — always acquire locks in a globally consistent order (e.g., by account ID) regardless of which direction the transfer logically goes, so two threads can never be simultaneously waiting on each other. `jstack`'s automatic deadlock detection only catches **classic mutual-monitor deadlocks** — it won't catch deadlock-like conditions involving `java.util.concurrent` locks in more complex ways or resource deadlocks (e.g., two threads each holding one of two DB connections from a pool sized too small) — those need reasoning through the thread dump's stack traces manually, since they don't get the automatic 'Found one Java-level deadlock' banner."

---

### Q14. How do you prevent deadlocks in application code?
**Answer:**
1. **Consistent lock ordering** — always acquire multiple locks in the same global order across all code paths (Q13's account-transfer fix — order by a stable ID).
2. **Lock timeouts** — use `Lock.tryLock(timeout)` instead of `synchronized` (which blocks indefinitely) where contention is a real risk, so a thread gives up and can retry/fail gracefully instead of deadlocking forever.
3. **Minimize lock scope** — hold locks for the shortest time possible and avoid calling out to unknown/external code while holding a lock, which risks unexpectedly reacquiring another lock deep in a call chain.
4. **Prefer higher-level concurrency utilities** over hand-rolled `synchronized` blocks — `java.util.concurrent` collections (`ConcurrentHashMap`), `ReentrantLock` with `tryLock`, or lock-free structures (`AtomicReference`, CAS-based approaches) sidestep a lot of manual lock-ordering risk entirely.

```java
// tryLock with timeout — fails fast instead of deadlocking indefinitely
if (lockA.tryLock(500, TimeUnit.MILLISECONDS)) {
    try {
        if (lockB.tryLock(500, TimeUnit.MILLISECONDS)) {
            try {
                // critical section
            } finally { lockB.unlock(); }
        }
    } finally { lockA.unlock(); }
}
```

**Senior-level answer:**
> "In practice, I try to design concurrent code to avoid needing to hold **two or more locks simultaneously at all**, since that's the actual precondition for a classic deadlock — restructuring the problem (e.g., a single lock protecting both accounts' balances via a coarser-grained lock, or a lock-free CAS-based balance update) is often a better fix than getting lock ordering exactly right everywhere and hoping every future contributor follows the same discipline. Lock ordering works, but it's a convention that has to be maintained correctly across the whole codebase forever — I treat it as a fallback when a simpler design isn't feasible, not the default first solution."

---

## SECTION 5: THREAD POOL & CONNECTION POOL TUNING

### Q15. How do you correctly size a thread pool?
**Answer:** Depends heavily on whether the work is **CPU-bound** or **I/O-bound**:

- **CPU-bound work** (heavy computation, no blocking) — optimal pool size is roughly **`number of CPU cores`** (or `cores + 1`) — more threads than cores just adds context-switching overhead without more actual throughput.
- **I/O-bound work** (DB calls, HTTP calls, blocking on network) — threads spend most of their time **waiting**, not computing, so a much larger pool size than core count is appropriate. A common formula: `threads = cores * (1 + wait_time/compute_time)`.

```java
// I/O-bound example: if each request spends ~90ms waiting on I/O and ~10ms on CPU work,
// on an 8-core machine: threads = 8 * (1 + 90/10) = 8 * 10 = 80
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
executor.setCorePoolSize(20);
executor.setMaxPoolSize(80);
executor.setQueueCapacity(100);   // bounded queue — see below
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

**Senior-level answer:**
> "The detail I'd flag beyond core sizing: an **unbounded queue** on a thread pool is a common production landmine — if the pool is undersized relative to incoming load, requests queue up indefinitely instead of the pool ever hitting `maxPoolSize`, which means you never get the backpressure signal (rejected execution) that would otherwise tell you the system is overloaded — instead, memory grows and latency silently balloons until something else breaks. I always set an explicit **bounded queue capacity** paired with a deliberate rejection policy (`CallerRunsPolicy` for graceful backpressure, or a custom handler that fails fast) rather than leaving the queue unbounded by default."

---

### Q16. How do you tune a database connection pool (e.g., HikariCP), and how do you detect a connection leak?
**Answer:**
```yaml
# application.yml — HikariCP (Spring Boot's default connection pool)
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # NOT "as high as possible" — see below
      minimum-idle: 5
      connection-timeout: 3000     # ms to wait for a connection before failing fast
      leak-detection-threshold: 60000   # ms a connection can be held before Hikari logs a WARN suspecting a leak
```

**Sizing guidance:** Counter-intuitively, connection pools should generally be **smaller** than people expect — HikariCP's own sizing guide (based on the formula `connections = ((core_count * 2) + effective_spindle_count)`) argues that a pool that's too large causes **more** contention (context switching, lock contention on the DB side) than a correctly-sized smaller pool, because the DB itself can only usefully execute a limited number of queries concurrently before it becomes CPU/IO bound on its own end.

**Detecting a leak:** `leak-detection-threshold` logs a warning with the **stack trace of where the connection was acquired** if it's held longer than the threshold without being returned — this is usually caused by a missing `close()` (or a missing `try-with-resources`) somewhere in the code.

```java
// Connection leak — missing try-with-resources / finally block
Connection conn = dataSource.getConnection();
PreparedStatement stmt = conn.prepareStatement("SELECT ...");
// if an exception is thrown here, conn is NEVER closed/returned to the pool
ResultSet rs = stmt.executeQuery();

// Fixed: try-with-resources guarantees close() even on exception
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement("SELECT ...");
     ResultSet rs = stmt.executeQuery()) {
    // ...
}
```

**Senior-level answer:**
> "The HikariCP sizing guide's core insight — that 'bigger pool = better' is usually **wrong** — is one of the more counterintuitive but well-evidenced pieces of guidance in this space, and it's a great thing to cite directly if the interviewer probes on pool sizing, since it shows you've actually read the reasoning rather than just picking a number that felt safe. I'd also connect this directly back to the saturation/USE-method discussion from the monitoring topic — `hikaricp_connections_pending > 0` is exactly the early-warning saturation signal that tells you the pool (correctly or incorrectly sized) is currently a bottleneck, before it manifests as request-level timeouts."

---

## SECTION 6: DATABASE / QUERY PERFORMANCE

### Q17. How do you diagnose a slow database query?
**Answer:**
1. **Get the execution plan** — `EXPLAIN ANALYZE` (Postgres) or `EXPLAIN` (MySQL) shows the actual query plan the optimizer chose — look specifically for **sequential/full table scans** on large tables where an index scan would be expected.
2. **Check for missing or unused indexes** — a `WHERE`, `JOIN`, or `ORDER BY` column without a supporting index is the most common root cause of a slow query on an otherwise reasonably-sized table.
3. **Check row estimates vs. actual rows** in the plan — a large mismatch between the optimizer's row estimate and the actual row count often indicates stale table statistics, causing the optimizer to pick a bad plan.
4. **Look at lock waits** — a query can be "slow" not because its own execution is slow, but because it's **waiting on a lock** held by another transaction.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42 AND status = 'PENDING';

-- Bad: Seq Scan on orders (cost=0.00..48291.00 rows=3 width=120) (actual time=812ms)
--       Filter: (customer_id = 42 AND status = 'PENDING')
--   -> full table scan on a large table, no supporting index

-- Good: Index Scan using idx_orders_customer_status (cost=0.42..8.44 rows=3 width=120) (actual time=0.05ms)
```

**Senior-level answer:**
> "I'd walk through this as a diagnostic funnel rather than jumping straight to 'add an index' — sometimes the plan reveals stale statistics (fixed with `ANALYZE`, not a schema change), sometimes it's genuinely a missing index, and sometimes the query is fast in isolation but slow under load because it's contending for a lock — those three root causes have completely different fixes, and treating every slow query as an 'add an index' problem is a common junior-level shortcut that doesn't always hold up."

---

### Q18. What is the N+1 query problem, and how do you fix it in a Spring/JPA application?
**Answer:** The N+1 problem: fetching a list of **N** parent entities triggers **1** initial query, but then **N additional queries** — one per parent — to lazily fetch each one's related child collection, instead of fetching everything in a small, fixed number of queries.

```java
// N+1 in action
List<Order> orders = orderRepository.findAll();          // 1 query
for (Order order : orders) {
    order.getLineItems().size();                          // N additional queries — one per order, lazy-loaded
}
```

**Fixes:**
```java
// Fix 1: JOIN FETCH — eagerly fetch the association in the SAME query
@Query("SELECT o FROM Order o JOIN FETCH o.lineItems WHERE o.status = :status")
List<Order> findByStatusWithLineItems(@Param("status") String status);

// Fix 2: @EntityGraph — declarative eager-fetch hint without hand-writing JPQL joins
@EntityGraph(attributePaths = {"lineItems"})
List<Order> findByStatus(String status);

// Fix 3: Batch fetching — fetches related entities in batches (e.g., IN (id1, id2, ...id20))
// instead of one query per parent — configured via @BatchSize or 
// hibernate.default_batch_fetch_size
```

**Senior-level answer:**
> "N+1 is one of the most common performance bugs I encounter in code review specifically because it's **invisible in local testing** — with 3 test orders in a dev DB, 4 queries instead of 1 is imperceptible; the same code against 10,000 orders in production is a very different story. This is exactly why I'd want query-count assertions or a slow-query threshold in integration tests, or at minimum enabling `hibernate.generate_statistics` in a staging environment, to catch N+1 patterns **before** they reach production rather than discovering them via a latency incident."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the difference between `jstack` and JFR for diagnosing a hang?" → `jstack` gives a point-in-time (or diffed multi-sample) snapshot, quick and sufficient for classic deadlocks; JFR gives a continuous timeline of thread state/lock contention over a window, better for intermittent or hard-to-catch issues.
- "What does `-Xms` equal to `-Xmx` actually buy you?" → Avoids the JVM incrementally resizing the heap during warmup (each resize is itself a pause-inducing operation) — predictable from the start at the cost of committing that memory upfront.
- "Can a memory leak happen even with a correctly configured GC?" → Yes — GC only reclaims **unreachable** objects; if application code keeps a reference alive (static collection, unclosed `ThreadLocal`), those objects are still reachable and GC is working exactly as designed while memory still grows.
- "What's the difference between `synchronized` and `ReentrantLock`?" → `ReentrantLock` supports `tryLock` with timeout, interruptible lock acquisition, and fairness policies — capabilities `synchronized` doesn't offer — at the cost of requiring manual `unlock()` in a `finally` block instead of automatic release.
- "Why would increasing thread pool size make throughput worse, not better?" → Beyond the point where CPU/downstream resources (like a DB connection pool) are saturated, more threads just adds context-switching overhead and increases contention on those shared downstream resources, without any usable increase in actual parallel work.
- "What's the first thing you'd check if a service suddenly has high CPU usage?" → A thread dump — a small number of threads in `RUNNABLE` state with an identical, repeating stack trace across multiple samples usually points directly at a spinning/busy-loop culprit.

---

*Study tip: The three areas interviewers most reliably use to separate senior candidates on this topic are (1) correctly matching the OOM error **variant** to the right diagnostic tool rather than reaching for a heap dump for everything (Q12), (2) being able to explain a memory leak all the way from symptom (sawtooth heap graph) to root cause (GC-roots reference chain) to a concrete, named pattern like the `ThreadLocal`/static-collection cases, not just "there was a leak" (Q9–Q10), and (3) treating thread pool and connection pool sizing as evidence-based (formulas, HikariCP's own guidance, measured wait/compute ratios) rather than "set it high to be safe" (Q15–Q16) — that instinct is one of the most common and costly production misconfigurations, and calling it out unprompted is a strong signal.*
