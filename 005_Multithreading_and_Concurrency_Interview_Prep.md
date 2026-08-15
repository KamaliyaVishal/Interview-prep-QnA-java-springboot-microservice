# 5. Multithreading & Concurrency — Interview Prep — Senior Java Developer

*Note: This consolidates and reorganizes content from earlier multithreading prep into the curriculum's Topic 5 structure, with expanded depth on ThreadLocal and Fork/Join, which weren't previously covered.*

---

## SECTION 1: THREAD CREATION (Thread, Runnable, Callable)

### Q1. What are the ways to create a thread in Java? Which do you prefer in production?
**Answer:**
1. Extend `Thread`, override `run()`.
2. Implement `Runnable`, pass to a `Thread` or `ExecutorService`.
3. Implement `Callable<V>` (returns a value, can throw checked exceptions), submit to `ExecutorService`.
4. Use `CompletableFuture.supplyAsync()`/`runAsync()` for composable async work.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
Future<Order> future = executor.submit(() -> processOrder(orderId));   // Callable via lambda
```

**Senior answer:** "I never create raw `Thread` objects in production — I always go through a managed `ExecutorService` (or `CompletableFuture` for composition). Extending `Thread` wastes single inheritance and couples business logic to thread mechanics; `Runnable` can't return values or throw checked exceptions. `Callable` + `ExecutorService` gives lifecycle management, pooling, and backpressure — all essential at production scale."

**Follow-up:** "Why does `Runnable.run()` not throw checked exceptions but `Callable.call()` does?" → `Runnable` predates generics/checked-exception-friendly design (Java 1.0); `Callable` was introduced in Java 5 specifically to fix this limitation alongside the Executor framework.

---

### Q2. Runnable vs Callable — full comparison
| | Runnable | Callable\<V\> |
|---|---|---|
| Method | `void run()` | `V call() throws Exception` |
| Return value | None | Yes, typed |
| Checked exceptions | Cannot declare/throw | Can throw |
| Get result | N/A | Via `Future<V>` from `executor.submit()` |
| Since | Java 1.0 | Java 5 |

**Follow-up:** "Can you submit a `Runnable` to an `ExecutorService` and still get a `Future`?" → Yes — `executor.submit(Runnable)` returns a `Future<?>` whose `get()` returns `null` on success (useful only for tracking completion/exception, not a result value). There's also an overload `submit(Runnable, T result)` letting you supply a fixed result value to return upon completion.

---

### Q3. What happens if you call run() instead of start()?
**Answer:** `run()` executes as a **plain synchronous method call on the current thread** — no new thread is created, no concurrency occurs at all. `start()` is what actually asks the JVM/OS to allocate a new thread of execution, which then invokes `run()` on that new thread.

**Follow-up trap:** "Can `start()` be called twice on the same `Thread` object?" → No — throws `IllegalThreadStateException`. A `Thread` object is single-use; this is precisely why thread pools reuse **worker threads** internally rather than restarting dead `Thread` objects.

---

## SECTION 2: THREAD LIFECYCLE & STATES

### Q4. Explain the Thread states in Java, and what triggers each transition.
**Answer:**
```
NEW → RUNNABLE ⇄ (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```
- **NEW** — `Thread` object created, `start()` not yet called.
- **RUNNABLE** — eligible to run (JVM doesn't distinguish "actually running on CPU" vs "ready and waiting for CPU time" — both are RUNNABLE).
- **BLOCKED** — waiting to acquire an intrinsic (`synchronized`) lock currently held by another thread.
- **WAITING** — waiting indefinitely: `Object.wait()` (no timeout), `Thread.join()` (no timeout), `LockSupport.park()`.
- **TIMED_WAITING** — `Thread.sleep(ms)`, `Object.wait(ms)`, `Thread.join(ms)`, `LockSupport.parkNanos()`.
- **TERMINATED** — `run()` completed normally or via uncaught exception.

**Senior trap follow-up:** "Is a RUNNABLE thread guaranteed to be executing on a CPU core right now?" → No — RUNNABLE only means "eligible"; the OS scheduler decides actual CPU time allocation. This distinction trips up many candidates who conflate Java's `Thread.State` with OS-level scheduling states.

**Follow-up:** "What's the difference between BLOCKED and WAITING?" → `BLOCKED` is specifically about contending for a **monitor lock** (`synchronized`); `WAITING`/`TIMED_WAITING` cover a broader set of voluntary waits (`wait()`, `join()`, `park()`) that don't necessarily involve lock contention.

---

### Q5. How do you check a thread's current state, and how would you use this in production debugging?
```java
Thread.State state = someThread.getState();
```
**Production use case:** Combined with a thread dump (`jstack`), examining thread states across many threads reveals systemic issues — e.g., a large cluster of threads stuck in `BLOCKED` on the same lock signals contention; many threads `WAITING` on a `CountDownLatch`/connection pool signals a slow/stuck downstream dependency (Section on Executor Framework covers debugging thread pool exhaustion further).

---

## SECTION 3: SYNCHRONIZATION & LOCKS

### Q6. How does the synchronized keyword work internally?
**Answer:** Every Java object has an intrinsic lock/monitor (tracked via the object's header/mark word). `synchronized` compiles to `monitorenter`/`monitorexit` bytecode instructions — only one thread can hold the monitor at a time; contending threads enter `BLOCKED` state.

**JVM optimization (senior detail):** Modern JVMs escalate locks based on observed contention — from **thin/lightweight locking** (CAS-based, no OS mutex, used when there's no real contention) up to **heavyweight (inflated) locking** (OS-level mutex, used under real contention) — this escalation is invisible to the developer but explains why `synchronized` performance varies so much with actual contention levels. (Biased locking, once part of this ladder, was disabled by default from JDK 15 onward.)

---

### Q7. synchronized method vs synchronized block — and why prefer a dedicated lock object over synchronized(this)?
```java
public void updateBalance(BigDecimal amount) {
    validate(amount);                         // non-critical work outside the lock
    synchronized (balanceLock) {               // narrow, dedicated lock object
        balance = balance.add(amount);
    }
}
```
**Answer:** Synchronized instance methods lock on `this`; synchronized static methods lock on the `Class` object; synchronized blocks let you scope the lock to exactly the critical section and to a **specific, private lock object**.

**Why avoid `synchronized(this)`:** `this` is a **publicly accessible object** — any external code holding a reference to your object can also `synchronized` on it, creating unexpected contention or even deadlock you don't control. A `private final Object lock = new Object();` field keeps the lock entirely internal to your class.

---

### Q8. ReentrantLock vs synchronized — when would you choose ReentrantLock?
| | synchronized | ReentrantLock |
|---|---|---|
| Acquire/release | Automatic (JVM, even on exception) | Manual (`lock()`/`unlock()` — **must** be in `finally`) |
| Try without blocking | No | `tryLock()` |
| Timed attempt | No | `tryLock(timeout, unit)` |
| Interruptible wait | No | `lockInterruptibly()` |
| Fairness policy | No | `new ReentrantLock(true)` |
| Multiple wait-conditions | No (one implicit) | Yes, via `newCondition()` |

```java
private final ReentrantLock lock = new ReentrantLock();
public void process() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();   // mandatory — a common interview trap is forgetting this
    }
}
```
**Senior answer:** "I reach for `ReentrantLock` specifically when I need `tryLock()` with a timeout (to avoid deadlocking on a distributed resource), fairness guarantees, or multiple condition variables on one lock — e.g., a bounded buffer needing separate 'not full'/'not empty' conditions. Otherwise `synchronized` is simpler and JVM-optimized, so I default to it."

---

### Q9. What is "reentrant" and why does it matter?
**Answer:** A thread already holding a lock can re-acquire it (e.g., calling another synchronized method on the same object from within a synchronized method) without deadlocking itself — the lock tracks a **hold count**, released only when it returns to zero.
```java
public synchronized void outer() { inner(); }   // same thread, same lock — fine, reentrant
public synchronized void inner() { }
```
Without reentrancy, this common pattern (one synchronized method calling another on the same object) would deadlock the thread against its own held lock.

---

### Q10. ReadWriteLock — real use case
**Answer:** `ReentrantReadWriteLock` allows multiple concurrent **readers** OR one exclusive **writer**, never both — ideal for read-heavy, write-rare shared state (e.g., a config cache refreshed every 5 minutes but read by every request thread).
```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
public Config getConfig() {
    rwLock.readLock().lock();
    try { return cachedConfig; } finally { rwLock.readLock().unlock(); }
}
```

---

## SECTION 4: wait(), notify(), notifyAll()

### Q11. Why are wait()/notify()/notifyAll() defined on Object, not Thread?
**Answer:** Every object (not every thread) has an intrinsic monitor lock — `wait()` releases the monitor **of the specific object it's called on**, so the mechanism must be tied to the object, not the thread. Since any object can serve as a lock, the API lives on `Object`.

**Follow-up trap:** "What happens if you call wait() without holding the lock?" → Throws `IllegalMonitorStateException` — must be inside a `synchronized` block on that same object.

---

### Q12. notify() vs notifyAll() — when would using notify() alone cause a bug?
**Answer:** `notify()` wakes **one arbitrary waiting thread**; `notifyAll()` wakes **all** waiting threads (only one re-acquires the lock and proceeds at a time, others re-block until the lock is free again).

**Bug scenario:** In a producer-consumer setup with **multiple consumer types** waiting on different conditions on the same monitor, `notify()` might wake a thread that **still can't proceed** (wrong condition for it), while a thread that actually could proceed stays asleep — effectively a **missed signal / lost wakeup** deadlock-like scenario. `notifyAll()` avoids this by waking everyone to re-check their own condition, at the cost of some wasted wake-ups (all threads except the "right" one just re-block).

**Best practice:** Default to `notifyAll()` unless you can prove all waiting threads are interchangeable (identical wait condition) — a genuinely common, subtle production bug source when `notify()` is used prematurely as a "safe-looking" optimization.

---

### Q13. Why must wait() always be called inside a while loop, not an if statement?
```java
synchronized void put(T item) throws InterruptedException {
    while (queue.size() == capacity) wait();   // MUST be while, not if
    queue.add(item);
    notifyAll();
}
```
**Answer:** **Spurious wakeups** are explicitly permitted by the JVM spec — a thread can return from `wait()` without any actual `notify()` call having occurred. Additionally, even with a genuine `notify()`, another thread might "steal" the resource between the notify and this thread actually resuming and re-acquiring the lock. A `while` loop **re-checks the condition** after waking, looping back into `wait()` if the condition still doesn't hold — an `if` would proceed incorrectly on a spurious or stale wakeup.

---

### Q14. Implement Producer-Consumer using wait/notify.
```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    BoundedBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) wait();
        queue.add(item);
        notifyAll();
    }
    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) wait();
        T item = queue.poll();
        notifyAll();
        return item;
    }
}
```
**Senior answer to volunteer:** "In real production code I'd use `BlockingQueue` instead of hand-rolling this — but understanding this pattern is essential because it's the foundation `BlockingQueue` itself is built on, and interviewers use it to test JMM fundamentals directly."

---

## SECTION 5: DEADLOCK, LIVELOCK, STARVATION

### Q15. Deadlock — definition, example, detection, and prevention
**Answer:** Two or more threads each hold a lock the other needs, and each waits forever — circular wait.
```java
// Thread A: synchronized(lockA) { synchronized(lockB) { ... } }
// Thread B: synchronized(lockB) { synchronized(lockA) { ... } }
```
**Prevention:**
1. **Consistent lock ordering** — always acquire locks in the same global order (e.g., by object ID) across every thread.
2. **`tryLock(timeout)`** instead of blocking indefinitely — back off and retry rather than wait forever.
3. Minimize nested lock scope; keep critical sections small.
4. Prefer higher-level concurrency utilities (`ConcurrentHashMap`, `BlockingQueue`) over manual lock nesting.

**Detection:** Take a thread dump (`jstack <pid>`) — the JVM explicitly reports `"Found one Java-level deadlock"` with the exact threads and locks involved.

---

### Q16. Deadlock vs Livelock vs Starvation — differentiate clearly
- **Deadlock:** threads permanently blocked, waiting on each other — zero progress, forever.
- **Livelock:** threads actively keep changing state in response to each other but still make **no real progress** (e.g., two threads repeatedly backing off and retrying in a way that keeps colliding — busy but stuck).
- **Starvation:** a thread **never** gets CPU time or lock access because other threads (often higher-priority or simply greedy/long-holding) continuously get preference.

**Follow-up:** "Can `ReentrantLock`'s fairness setting prevent starvation? What's the trade-off?" → `new ReentrantLock(true)` grants the lock to the **longest-waiting** thread (FIFO order), preventing starvation — but at a real throughput cost, since fair locks have higher overhead than the default (unfair, which allows barging for better raw throughput). A senior answer should name this explicit throughput-vs-fairness trade-off.

---

### Q17. Fix this deadlock-prone bank transfer code.
```java
// BEFORE — deadlock-prone
void transfer(Account from, Account to, BigDecimal amt) {
    synchronized (from) {
        synchronized (to) { /* transfer logic */ }
    }
}

// AFTER — consistent lock ordering by a stable identity (e.g., account ID)
void transfer(Account from, Account to, BigDecimal amt) {
    Account first  = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) { /* transfer logic */ }
    }
}
```
This is a very common fintech-interview coding question (JPMC/Goldman/Visa/Morgan Stanley) — practice writing it from memory.

---

## SECTION 6: EXECUTOR FRAMEWORK & THREAD POOLS

### Q18. Why avoid Executors factory methods (newFixedThreadPool, etc.) in production?
**Answer:** `newFixedThreadPool`/`newSingleThreadExecutor` use an **unbounded `LinkedBlockingQueue`** — under sustained load, tasks queue indefinitely instead of failing fast, risking `OutOfMemoryError` rather than graceful degradation. `newCachedThreadPool` has an **unbounded max pool size** — can spawn unlimited threads under load, causing thread exhaustion.

**Senior answer:** Always construct `ThreadPoolExecutor` explicitly with a **bounded queue** and explicit `RejectedExecutionHandler`, so the system fails predictably under load instead of silently exhausting memory — directly cited from Brian Goetz's *Java Concurrency in Practice* guidance.

```java
new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(500),         // bounded
    new ThreadPoolExecutor.CallerRunsPolicy()   // graceful backpressure
);
```

---

### Q19. Explain all ThreadPoolExecutor parameters and the exact task-submission decision flow.
**Answer:** `corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, rejectedExecutionHandler`.

**Execution order (frequently whiteboard-tested):**
1. If active threads < `corePoolSize` → spin up a new thread.
2. Else if the queue has room → enqueue the task.
3. Else if active threads < `maximumPoolSize` → spin up an additional (non-core) thread.
4. Else → invoke the `RejectedExecutionHandler`.

**RejectedExecutionHandler policies:** `AbortPolicy` (default, throws), `CallerRunsPolicy` (runs on caller's thread — natural backpressure), `DiscardPolicy` (silently drops), `DiscardOldestPolicy` (drops oldest queued task, retries).

---

### Q20. How do you size a thread pool correctly?
**Answer:**
- **CPU-bound:** `threads ≈ N_cpu + 1`.
- **I/O-bound:** `threads ≈ N_cpu × (1 + wait_time/compute_time)` (Brian Goetz's formula, Little's-Law-based).

**Senior answer:** "It's not a fixed formula I apply blindly — I load test with realistic traffic, monitor queue depth and rejection rate via Micrometer/Actuator, and tune iteratively based on observed behavior, not theoretical numbers alone."

---

### Q21. Graceful shutdown of an ExecutorService — show the correct pattern.
```java
executor.shutdown();
try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
        if (!executor.awaitTermination(10, TimeUnit.SECONDS))
            log.error("Executor did not terminate");
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();   // restore interrupt status — commonly missed
}
```
**Follow-up:** "shutdown() vs shutdownNow()?" → `shutdown()` stops accepting new tasks, lets running/queued tasks finish. `shutdownNow()` attempts to interrupt running tasks and returns the list of tasks that never started.

---

## SECTION 7: FUTURE vs COMPLETABLEFUTURE

### Q22. Future vs CompletableFuture — why was CompletableFuture introduced?
**Answer:** `Future.get()` **blocks** with no way to register a completion callback, chain dependent async operations, or combine multiple futures. `CompletableFuture` (Java 8) is composable and can be driven in a non-blocking, callback style:
- `thenApply/thenAccept/thenRun` — chaining.
- `thenCompose` — chaining a **dependent** async call (flatMap-like, avoids nested futures).
- `thenCombine` — combining two **independent** futures.
- `allOf`/`anyOf` — waiting on many futures.

```java
CompletableFuture<User> userFuture = getUserAsync(id);
CompletableFuture<Orders> ordersFuture = getOrdersAsync(id);
CompletableFuture<Profile> combined = userFuture.thenCombine(ordersFuture,
    (user, orders) -> new Profile(user, orders));
```
**Real use case:** Aggregating 3 downstream microservice calls in parallel for a dashboard endpoint — `allOf()` cuts total latency from sum-of-latencies to max-of-latencies.

---

### Q23. Exception handling in CompletableFuture — what's the common bug?
```java
CompletableFuture.supplyAsync(() -> riskyCall())
    .exceptionally(ex -> { log.error("failed", ex); return fallback; });
```
**Common mistake:** An exception inside a `CompletableFuture` chain is **swallowed silently** unless you call `.get()`/`.join()` (which then throws) or attach `.exceptionally()`/`.handle()`. Forgetting this is one of the most common real production bugs — async failures that vanish without a trace in monitoring.

---

### Q24. thenApply vs thenApplyAsync — what's the actual difference, and why does it matter?
**Answer:** `thenApply()` runs the continuation on **whichever thread completed the previous stage** (could be the calling thread if already complete, or the async worker thread) — `thenApplyAsync()` (without an explicit executor) always submits the continuation to the **common `ForkJoinPool`**, and the overload accepting an `Executor` lets you control exactly which pool runs it.

**Why it matters:** If a continuation does blocking I/O and you don't pass a dedicated executor, `thenApplyAsync()` defaults to the shared common pool — the same pool parallel streams use — risking the same pool-starvation issue covered in Section 11 (Fork/Join).

---

## SECTION 8: CONCURRENT COLLECTIONS

### Q25. ConcurrentHashMap internal working
**Answer:** Java 8+ drops Java 7's segment-locking design in favor of **CAS** for inserting into an empty bucket and **synchronized blocks scoped to individual bin nodes** only on actual collision — far more granular than a single lock (`Hashtable`) or 16-segment locking (Java 7). Reads are lock-free. Buckets treeify into red-black trees past 8 collisions (same mechanism as `HashMap`).

---

### Q26. Why does ConcurrentHashMap disallow null keys/values?
**Answer:** In a single-threaded `HashMap`, `get(key) == null` followed by `containsKey(key)` safely disambiguates "maps to null" from "absent," since nothing else mutates concurrently. In a **concurrent** map, that two-step check is inherently racy — another thread could insert/remove between the two calls. Doug Lea deliberately disallowed nulls entirely to eliminate this ambiguity, forcing explicit `Optional`/sentinel patterns instead.

---

### Q27. CopyOnWriteArrayList — mechanism and correct use case
**Answer:** Every write (`add`/`remove`/`set`) copies the **entire backing array**, applies the change, then atomically swaps the reference. Reads/iteration never lock, and iterators hold a **fixed snapshot** from iterator-creation time — never throw `ConcurrentModificationException`, but also never see concurrent updates made after iteration began.

**Use when:** read-heavy, write-rare (e.g., listener/observer lists). **Avoid when:** write-heavy — O(n) copy cost per write.

---

### Q28. BlockingQueue implementations — quick comparison
- `ArrayBlockingQueue` — bounded, array-backed.
- `LinkedBlockingQueue` — optionally bounded (defaults **unbounded** — same OOM risk as `Executors.newFixedThreadPool`!).
- `PriorityBlockingQueue` — unbounded, priority-ordered, not FIFO.
- `SynchronousQueue` — zero capacity, direct hand-off (used internally by `newCachedThreadPool`).

---

## SECTION 9: ATOMIC VARIABLES

### Q29. How do Atomic classes achieve thread safety without locks?
**Answer:** Via **CAS (Compare-And-Swap)** — a hardware-level atomic instruction (`cmpxchg` on x86): "if current value equals expected, set to new value," done atomically by the CPU. If another thread changed the value first, CAS fails and the JVM retries in a loop — no OS-level lock/context switch, making it faster than `synchronized` under low-to-moderate contention.
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // internally: loop { old = get(); CAS(old, old+1) }
```

---

### Q30. What is the ABA problem, and how does AtomicStampedReference solve it?
**Answer:** Thread reads value A; another thread changes A→B→A before the first thread's CAS executes. CAS succeeds (value *looks* unchanged) even though it was modified in between — dangerous for lock-free structures like stacks where a freed reference could be reused. `AtomicStampedReference<T>` pairs the value with a version **stamp**; CAS checks both, so a same-value-different-history case is correctly detected as changed.

---

### Q31. AtomicLong vs LongAdder — when would you choose LongAdder?
**Answer:** Under **high contention**, all threads CAS-ing a single `AtomicLong` cell retry heavily. `LongAdder` internally stripes across **multiple cells**; each thread updates a different cell, and `sum()` aggregates them only when read — much faster for high-write, infrequent-read counters (e.g., request-count metrics). Trade-off: more memory, and `sum()` isn't perfectly atomic at a single instant — fine for metrics, wrong for values needing strict point-in-time consistency (e.g., account balance, where `AtomicLong`/proper locking is still correct).

---

## SECTION 10: volatile vs synchronized

### Q32. What does volatile actually guarantee? Does it give atomicity?
**Answer:** `volatile` guarantees:
1. **Visibility** — writes are immediately visible to other threads (reads/writes bypass CPU-cache/register staleness, go to/from main memory).
2. **Ordering** — prevents instruction reordering around the variable via memory barriers.

**It does NOT guarantee atomicity** — `count++` on a `volatile int` is still a race condition (read-modify-write, three separate steps).
```java
private volatile boolean running = true;
public void stop() { running = false; }
public void run() { while (running) { doWork(); } }   // without volatile, may never see the update — infinite loop risk
```

---

### Q33. volatile vs synchronized — full comparison and when to use each
| | volatile | synchronized |
|---|---|---|
| Visibility | Yes | Yes |
| Ordering | Yes | Yes |
| Atomicity of compound ops | No | Yes |
| Mutual exclusion | No | Yes |
| Blocking | Never blocks | Blocks contending threads |

**Senior answer:** "I use `volatile` for simple flags or single-writer/many-reader variables where I only need visibility, not compound atomicity. I use `synchronized`/`Atomic*`/`Lock` when I need to guard a compound state transition or mutual exclusion. Using `volatile` where atomicity is actually needed is a very common, subtle bug — the code compiles and often 'seems to work' under light load, then fails under real concurrency."

---

## SECTION 11: THREADLOCAL

### Q34. What is ThreadLocal? How does it work internally?
**Answer:** `ThreadLocal<T>` gives **each thread its own independent copy** of a variable — no thread sees another thread's value, and no synchronization is needed since there's no actual sharing.

**Internal mechanism (senior-level depth, often asked directly):** Each `Thread` object internally holds a `ThreadLocal.ThreadLocalMap` — **not** the other way around. When you call `threadLocal.get()`/`set()`, it actually looks up/stores the value in the **current thread's own map**, keyed by the `ThreadLocal` instance itself (using a `WeakReference` to the key, discussed in Q37).

```java
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
// each thread gets its own SimpleDateFormat instance — SimpleDateFormat is notoriously NOT thread-safe,
// this is a classic real-world use case for ThreadLocal
```

---

### Q35. What are real production use cases for ThreadLocal?
**Answer:**
1. **Avoiding synchronization for non-thread-safe objects** — e.g., `SimpleDateFormat` (mutable, not thread-safe) — give each thread its own instance instead of sharing one with locking (though in modern Java, `DateTimeFormatter` is immutable/thread-safe and preferred, making this specific use case less common than it used to be — worth mentioning you know the modern alternative).
2. **Per-request context propagation** — MDC (Mapped Diagnostic Context) in logging frameworks (Log4j/Logback) uses `ThreadLocal` to carry a request ID/user ID/trace ID through an entire request's processing, so every log line in that thread automatically includes the context without passing it as a parameter through every method call.
3. **Transaction context** — Spring's `TransactionSynchronizationManager` uses `ThreadLocal` internally to bind the current transaction/connection to the executing thread, so nested repository calls can access "the current transaction" without explicit parameter passing.
4. **Security context** — Spring Security's `SecurityContextHolder` (default strategy) stores the authenticated user's context in a `ThreadLocal`, accessible anywhere in that request's call stack.

---

### Q36. What's the danger of ThreadLocal in a thread-pooled environment (e.g., Tomcat, ExecutorService)? (Very high-value senior question)
**Answer:** Pooled worker threads are **reused across many requests/tasks** — they don't die after one unit of work. If you `set()` a `ThreadLocal` value and **don't explicitly `remove()` it**, the value **persists in that thread's `ThreadLocalMap` for the thread's entire lifetime**, potentially leaking into the **next, completely unrelated request** handled by that same pooled thread.

**Real production bug this causes:** A `ThreadLocal` holding a user's security context or session data doesn't get cleared after request A finishes; the pool reuses that same worker thread for request B; request B's code — which never set its own value — reads request A's **stale, leftover data**. This has caused real, serious security incidents (one user seeing another user's data) in production systems using thread pools with `ThreadLocal`-based context propagation.

**Correct pattern:**
```java
public void handleRequest() {
    contextHolder.set(buildContext());
    try {
        processRequest();
    } finally {
        contextHolder.remove();   // MANDATORY in pooled-thread environments — never skip this
    }
}
```
**Senior answer to state explicitly:** "Any time I use `ThreadLocal` in a Spring Boot service (which runs on pooled Tomcat threads), I treat `.remove()` in a `finally` block as non-negotiable — this is one of the most common subtle memory-leak-and-data-leak sources in production Java systems, and I specifically look for it in code review whenever I see `ThreadLocal.set()`."

---

### Q37. Why does ThreadLocal use a WeakReference for its keys? Can ThreadLocal still leak memory?
**Answer:** `ThreadLocalMap` stores entries keyed by a `WeakReference<ThreadLocal<?>>` — so if the `ThreadLocal` instance itself becomes otherwise unreferenced (e.g., it was a local variable, now out of scope), the **key** can be garbage collected even if the entry technically still exists in the map, allowing the entry to eventually be cleaned up (lazily, on subsequent `get`/`set`/`remove` calls that happen to sweep stale entries).

**But this does NOT fully prevent leaks:** The **value** is still held by a **strong reference** from the entry. If the key (the `ThreadLocal` object) gets collected but the thread lives on (pooled thread) and no further `ThreadLocalMap` operations happen to trigger the stale-entry cleanup, the **value object can still leak** for the life of the pooled thread — which is precisely why explicit `.remove()` remains essential and isn't something you can rely on the weak-reference design to fully solve for you.

---

### Q38. InheritableThreadLocal — what problem does it solve, and what's its limitation with thread pools?
**Answer:** `InheritableThreadLocal` automatically propagates a parent thread's value to any **child threads it creates** (at child-thread creation time, via a copy). Useful for passing context to explicitly spawned child threads.

**Limitation:** It only copies the value at the moment a **new** `Thread` is created — it does **not** work with pooled `ExecutorService` worker threads, since those threads already exist and are reused, not freshly spawned per task. For context propagation across `ExecutorService` task submission, you need to manually capture and re-set the context inside the submitted task (or use a library like Spring's `TaskDecorator`, or MDC-aware executor wrappers) — a good nuance to raise if this comes up, since it's a real limitation many candidates aren't aware of.

---

## SECTION 12: FORK/JOIN FRAMEWORK

### Q39. What is the Fork/Join framework, and what problem does it solve differently from ThreadPoolExecutor?
**Answer:** `ForkJoinPool` (Java 7) is designed for **divide-and-conquer, recursive** workloads — a task splits itself into smaller subtasks (`fork()`), those may split further, and results are combined (`join()`). It uses **work-stealing**: each worker thread has its own double-ended queue (deque) of subtasks; when a worker runs out of work, it **steals** tasks from the **back** of another busy worker's deque (while the owning thread takes from the front) — this keeps all CPU cores busy even when subtasks are unevenly sized, which a standard shared-queue `ThreadPoolExecutor` doesn't handle as efficiently for recursive workloads.

```java
class SumTask extends RecursiveTask<Long> {
    private final long[] array; private final int start, end;
    private static final int THRESHOLD = 10_000;

    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left  = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();                          // asynchronously execute left
        long rightResult = right.compute();   // compute right in current thread
        long leftResult = left.join();        // wait for left's result
        return leftResult + rightResult;
    }
}
// ForkJoinPool.commonPool().invoke(new SumTask(array, 0, array.length));
```

**Key classes:** `RecursiveTask<V>` (returns a value), `RecursiveAction` (no return value), both extend `ForkJoinTask`.

---

### Q40. RecursiveTask vs RecursiveAction — difference?
**Answer:** `RecursiveTask<V>`'s `compute()` returns a value `V` (used when the divide-and-conquer computation produces a result, e.g., a parallel sum/reduction). `RecursiveAction`'s `compute()` returns `void` (used for side-effecting recursive work, e.g., parallel in-place array sorting where no value needs returning, just the mutation).

---

### Q41. How does parallelStream() relate to ForkJoinPool, and what's the danger?
**Answer:** `parallelStream()` uses the **shared common `ForkJoinPool`** (default size `N_cpu - 1`) — the **same pool used application-wide**, including by unrelated library code and other parallel streams. 

**Danger:** Running a **blocking I/O call** (DB query, REST call, `Thread.sleep`) inside a parallel stream **starves the shared pool for the entire application**, not just your code — since there's no isolation between different parallel-stream usages across the app. 

**Best practice:** Never do blocking I/O inside `parallelStream()`. Reserve parallel streams for **pure, CPU-bound computation**; use a dedicated `ExecutorService` + `CompletableFuture` (with an explicit executor passed to `supplyAsync`/`thenApplyAsync`) for I/O-bound parallel work instead.

**Follow-up:** "Can you use a custom ForkJoinPool for a parallel stream instead of the common one?" → Yes:
```java
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() -> list.parallelStream().map(this::process).collect(Collectors.toList())).get();
```
This isolates the parallel stream's resource usage from the shared common pool — worth mentioning as the fix when this limitation comes up.

---

### Q42. Work-stealing — explain the exact mechanism and why it improves throughput for uneven workloads.
**Answer:** Each worker thread in a `ForkJoinPool` maintains its own **double-ended queue** of tasks. The owning thread pushes/pops from **its own front** (LIFO, cache-friendly — most recently forked task is likely still hot in cache). When a worker's queue empties (it's run out of work), it "steals" from the **tail (back)** of a **different, busy** worker's queue (FIFO from the stealer's perspective) — stealing from the opposite end minimizes contention between the owner and the stealer accessing the same queue simultaneously. This keeps all cores productively busy even when the recursive task tree splits unevenly, unlike a single shared queue where one slow/large task can create a bottleneck others can't route around.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Can you interrupt a thread that's blocked in `wait()`?" → Yes — `wait()` throws `InterruptedException` if the waiting thread is interrupted, allowing graceful cancellation.
- "What's the difference between `Thread.sleep()` and `TimeUnit.SECONDS.sleep()`?" → Functionally identical (both pause the current thread); `TimeUnit.sleep()` is just a more readable wrapper, avoiding manual millisecond math.
- "Does `synchronized` guarantee ordering between unrelated locks?" → No — synchronization only establishes happens-before relationships **between threads synchronizing on the same monitor**; unrelated locks provide no ordering guarantee relative to each other.
- "Is `ConcurrentHashMap.size()` always exact?" → No — under concurrent modification it's best-effort/approximate, a deliberate trade-off to avoid a global lock just for counting.
- "What happens to a ThreadLocal value when the thread itself terminates (not pooled, dies normally)?" → It's garbage collected along with the `Thread` object's `ThreadLocalMap` — no leak risk for genuinely short-lived, non-pooled threads; the danger is specifically about **long-lived pooled threads**.
- "Can two threads execute the same synchronized method on two DIFFERENT object instances simultaneously?" → Yes — each instance has its own monitor; locking is per-object, not per-method/per-class (for instance-synchronized methods).

---

*Study tip: ThreadLocal's leak-in-a-thread-pool scenario (Q36) and the work-stealing mechanism (Q42) are the two topics in this file most likely to be genuinely new ground for candidates who've only studied basic multithreading — both come up frequently at product companies specifically to differentiate senior candidates who've operated real pooled-thread production systems from those who've only studied theory.*
