# Distributed Caching — Interview Prep
---

## SECTION 1: REDIS; LOCAL vs DISTRIBUTED CACHE

### Q1. What's the practical difference between a local (in-process) cache and a distributed cache like Redis? When would you choose each?
**Answer:**
| | Local Cache (e.g., Caffeine) | Distributed Cache (e.g., Redis) |
|---|---|---|
| **Location** | In the same JVM/process as the application | Separate networked service, shared across instances |
| **Latency** | Nanoseconds — no network hop | Sub-millisecond to a few ms — network round trip |
| **Consistency across instances** | None — each instance has its own copy, can diverge | Single shared source — all instances see the same value |
| **Capacity** | Bounded by that instance's heap | Bounded by the Redis cluster's memory, independently scalable |
| **Survives instance restart/scale-out** | No — cold on every new instance | Yes — persists independently of app instances |

```java
// Local cache — Caffeine
Cache<String, Product> localCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build();

// Distributed cache — Redis via Spring Data Redis
@Cacheable(value = "products", key = "#id")
public Product getProduct(String id) {
    return productRepository.findById(id);
}
```

**Senior-level answer:**
> "The deciding factor is almost always **consistency requirements versus latency budget**. If every service instance can tolerate having a slightly different, independently-expiring view of the data — like a rarely-changing config or reference lookup — local cache wins on raw speed since there's no network hop at all. The moment two instances *must* see the same value at the same time — a rate-limit counter, a distributed lock, a session shared across pods behind a load balancer — you need Redis, because local caches by definition can't be kept in sync with each other without becoming a distributed cache themselves."

**Trap:** A very common real bug: caching data locally in a multi-instance deployment, then wondering why a write on instance A isn't visible on instance B — because there was never a mechanism to invalidate B's local copy. This is exactly why teams often use a **two-tier cache** — local as an L1 for speed, Redis as an L2 shared source with a pub/sub-based invalidation signal to clear local caches on write (Q6).

---

### Q2. Why is Redis so commonly chosen over other caching stores? What are its actual differentiators beyond "it's fast"?
**Answer:** Redis being an **in-memory data store** explains the speed, but the differentiators that matter in interviews are:
- **Rich data structures** — not just key/value strings; native `Hash`, `List`, `Set`, `Sorted Set`, `Stream`, `HyperLogLog` — meaning Redis can implement leaderboards (sorted sets), rate limiters (with TTLs + INCR), pub/sub messaging, and simple queues natively, not just caching.
- **TTL support built in** — `EXPIRE key seconds` — native, atomic expiration without application-side cleanup logic.
- **Atomic operations** — `INCR`, `SETNX`, and Lua scripting for multi-step atomic operations — critical for correctness under concurrent access (Q10 distributed locking).
- **Persistence options** (RDB snapshots, AOF log) — unlike a pure cache, Redis *can* survive a restart without a full cold cache, though it's still typically treated as a cache, not a system of record.
- **Clustering** (Redis Cluster) for horizontal scaling, plus replication for read scaling and failover.

```bash
SET user:1001 "..." EX 300          # set with 5-minute TTL, atomic
INCR pageviews:homepage             # atomic counter, no read-modify-write race
SETNX lock:order:555 "worker-A"     # atomic set-if-not-exists — building block for locks
```

**Senior-level answer:**
> "I position Redis less as 'a cache' and more as 'a fast, purpose-built data structure server that's commonly used as a cache.' That distinction matters in interviews because a lot of Redis's real value in production systems comes from those atomic primitives — rate limiting, distributed locks, leaderboards, pub/sub — not just key-value GET/SET, and I've seen teams reach for a heavier dedicated tool for something Redis already does natively and atomically."

---

## SECTION 2: CACHE-ASIDE, READ-THROUGH, WRITE-THROUGH & WRITE-BEHIND

### Q3. Explain Cache-Aside (Lazy Loading) — how it works, and its most common failure mode.
**Answer:** The **application code itself** is responsible for checking the cache first, and on a miss, loading from the database and populating the cache — the cache has no knowledge of the database at all.

```java
public Product getProduct(String id) {
    Product cached = redisTemplate.opsForValue().get("product:" + id);
    if (cached != null) return cached;                      // cache hit

    Product fromDb = productRepository.findById(id);         // cache miss — go to DB
    redisTemplate.opsForValue().set("product:" + id, fromDb, Duration.ofMinutes(10));
    return fromDb;
}

public void updateProduct(Product product) {
    productRepository.save(product);                         // write to DB
    redisTemplate.delete("product:" + product.getId());       // invalidate, don't update — Q6
}
```

**Most common failure mode:** on write, updating the cache **directly** with the new value instead of **deleting** it, combined with concurrent writes, can leave a **stale value permanently cached** if writes race (see Q6 for the full race condition).

**Senior-level answer:**
> "Cache-aside is my default pattern for the vast majority of read-heavy workloads because it's simple, and — critically — it's **resilient to cache failure**: if Redis goes down entirely, every request just falls through to the database and the application keeps working, just slower. That fail-open property is a big deal operationally compared to read-through, where the caching layer is often architecturally load-bearing."

---

### Q4. What is Read-Through, and how is it actually different from Cache-Aside from the application's point of view?
**Answer:** In **Read-Through**, the **cache provider itself** sits in front of the data store and handles the miss-then-load logic — the application only ever talks to the cache; it never contains fallback logic to query the database directly.

```
Cache-Aside:   App --miss--> queries DB itself --> populates cache itself
Read-Through:  App --> Cache --miss--> Cache queries DB internally --> Cache populates itself --> returns to App
```

- Cache-aside = the **application** owns the miss-handling logic.
- Read-through = the **caching layer/library** owns it (e.g., via a configured `CacheLoader` in Caffeine, or a caching provider integration).

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofMinutes(10))
    .build(id -> productRepository.findById(id));   // CacheLoader — cache handles the miss internally

Product p = cache.get("123"); // app never writes fallback logic — cache does it
```

**Senior-level answer:**
> "In practice, with Redis specifically, true read-through requires a plugin/module that lets Redis call back into your data source, which is less common than just implementing cache-aside in application code — so when people say 'read-through with Redis' colloquially, they often actually mean cache-aside wrapped in a clean abstraction like Spring's `@Cacheable`. The distinction matters more with local, library-managed caches like Caffeine's `LoadingCache`, where read-through is a first-class, well-supported feature."

---

### Q5. Compare Write-Through and Write-Behind (Write-Back) — what's the trade-off, and when would Write-Behind actually be worth the risk?
**Answer:**
| | Write-Through | Write-Behind (Write-Back) |
|---|---|---|
| **Write path** | App writes to cache → cache **synchronously** writes to DB before returning | App writes to cache → cache returns immediately → DB write happens **asynchronously**, batched/delayed |
| **Latency** | Write latency = cache write + DB write | Write latency = cache write only — DB write is off the critical path |
| **Durability risk** | Low — DB is always in sync at write-completion time | Real risk — a crash before the async flush means **data loss** for writes that never reached the DB |
| **DB load** | One DB write per cache write | Can batch/coalesce multiple writes into fewer DB operations — reduces DB load significantly |

```java
// Write-behind concept: buffer writes, flush on a schedule
private final Map<String, Product> writeBuffer = new ConcurrentHashMap<>();

public void updateProduct(Product product) {
    redisTemplate.opsForValue().set("product:" + product.getId(), product);
    writeBuffer.put(product.getId(), product);   // buffered, not yet persisted
}

@Scheduled(fixedDelay = 5000)
public void flushBuffer() {
    writeBuffer.values().forEach(productRepository::save);  // batched DB write
    writeBuffer.clear();
}
```

**Senior-level answer:**
> "Write-behind is worth the risk specifically when you have a **high write volume where DB write latency would otherwise become the bottleneck**, and the data has some tolerance for loss on a crash — think view counters, activity logs, telemetry — not financial transactions or order state. I've essentially never used write-behind for anything where losing a few seconds of unflushed writes would be a real business problem; write-through or cache-aside with synchronous writes is the safe default, and write-behind is a deliberate, narrow optimization, not a general-purpose pattern."

**Trap:** Don't present write-behind as a strictly-better performance upgrade over write-through — the durability trade-off is the entire point of the question, and glossing over "you can lose data on a crash" is a red flag in a senior interview.

---

## SECTION 3: CACHE INVALIDATION & CONSISTENCY

### Q6. On a write, should you update the cache with the new value, or delete the cache entry and let the next read repopulate it? Why does this matter?
**Answer:** **Delete (invalidate), don't update in place** — this is one of the most consequential, frequently-tested details in caching. Updating the cache directly on write is vulnerable to a **race condition** between a concurrent read and write:

```
T1: Write updates DB to value=NEW
T2: Concurrent read: cache miss (expired) -> reads DB, gets OLD stale value (read started before T1's commit was visible)
T3: Write updates cache to value=NEW
T4: Read (T2) finishes and writes value=OLD into cache  <-- stale value now cached, potentially for the full TTL
```

By instead **deleting** the cache key on write, the next read is forced to go to the database and repopulate — and while the same race can still theoretically leave a stale value cached in rare interleavings, deleting is strictly safer than updating because it removes the far more common and damaging failure mode: **a slow write's stale value silently overwriting a newer cached value with no way to detect it happened.**

```java
@Transactional
public void updateProduct(Product product) {
    productRepository.save(product);
    redisTemplate.delete("product:" + product.getId());  // invalidate — not redisTemplate.set(...)
}
```

**Senior-level answer:**
> "This is the pattern known as **Cache-Aside with invalidation**, and I always explain the reasoning, not just the rule — because interviewers are checking whether you actually understand the race condition versus having memorized 'delete not update.' For genuinely strict consistency requirements even invalidation isn't airtight; that's when you'd reach for a short TTL as a safety net (belt-and-suspenders) or accept the small window as a deliberate eventual-consistency trade-off, which is usually the right call for cache — perfect cache consistency generally isn't worth the complexity it costs."

---

### Q7. What role does TTL (Time To Live) play in cache consistency, and how do you choose a value?
**Answer:** TTL is the **safety net** that bounds how long a stale or missed-invalidation value can persist — even if an explicit invalidation is missed (a bug, a service crash between DB write and cache delete, a multi-step process failing partway), the entry **self-heals** once the TTL expires and the next read repopulates from the source of truth.

**Choosing a TTL is a trade-off, not a fixed number:**
- **Short TTL** → fresher data, but more cache misses → more DB load, less benefit from caching.
- **Long TTL** → better hit rate/less DB load, but staleness persists longer if invalidation is ever missed.
- Base the choice on **how often the data actually changes** and **how tolerable staleness is for that specific field** — not a single global default across every cache.

```java
@Cacheable(value = "products", key = "#id")
@CacheEvict(value = "products", key = "#result.id", condition = "false") // illustrative — explicit evict on write is the real mechanism
public Product getProduct(String id) { ... }
```
```yaml
spring.cache.redis.time-to-live: 600000   # 10 minutes — global default; override per-cache-name as needed
```

**Senior-level answer:**
> "I treat explicit invalidation as the primary consistency mechanism and TTL as the **backstop for the cases invalidation doesn't cover** — distributed systems fail partway through operations, and 'the cache-delete call never ran because the pod crashed right after the DB commit' is a real scenario. Relying on TTL *alone*, with no explicit invalidation, is the mistake I push back on most — it means every read is stale for up to the full TTL window after every write, which for frequently-updated data can mean users routinely seeing outdated information."

---

## SECTION 4: CACHE STAMPEDE, PENETRATION & AVALANCHE

### Q8. What is Cache Stampede (a.k.a. Thundering Herd), and how do you prevent it?
**Answer:** Cache Stampede happens when a **popular key expires**, and a large burst of **concurrent requests all miss the cache at the same moment** and all hit the database simultaneously trying to repopulate the same key — momentarily spiking DB load far beyond normal, sometimes enough to take the DB down.

**Prevention techniques:**
| Technique | How it works |
|---|---|
| **Mutex/lock on repopulation** | Only the first request that misses acquires a lock and queries the DB; others wait briefly and read the freshly-populated cache instead of also hitting the DB |
| **Probabilistic early expiration** | Recompute slightly *before* actual expiry, with jittered timing per key, so identical keys don't all expire at the exact same instant across many app instances |
| **Logical TTL, physical never-expire** | Store a "stale-after" timestamp in the value itself; serve stale data while one request refreshes in the background — no hard cache miss at all |

```java
public Product getProduct(String id) {
    Product cached = redisTemplate.opsForValue().get("product:" + id);
    if (cached != null) return cached;

    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent("lock:product:" + id, "1", Duration.ofSeconds(5));   // SETNX-based mutex
    if (Boolean.TRUE.equals(acquired)) {
        try {
            Product fromDb = productRepository.findById(id);
            redisTemplate.opsForValue().set("product:" + id, fromDb, Duration.ofMinutes(10));
            return fromDb;
        } finally {
            redisTemplate.delete("lock:product:" + id);
        }
    }
    Thread.sleep(50); // brief wait, then retry read — someone else is populating it
    return getProduct(id);
}
```

**Senior-level answer:**
> "Stampede is specifically a **single hot key** problem — it's what happens to your most popular product page or celebrity user profile at the moment its TTL expires, not a general cache-miss issue. The mutex approach is the most common fix I've implemented; the key insight is you're not trying to prevent the cache miss, you're trying to ensure **only one request pays the cost of repopulating**, while the rest wait a few milliseconds instead of all hammering the DB in parallel."

---

### Q9. What's the difference between Cache Penetration and Cache Avalanche, and how does prevention differ for each?
**Answer:**
| | Cache Penetration | Cache Avalanche |
|---|---|---|
| **What happens** | Requests for **keys that don't exist in the DB at all** — every such request is guaranteed to miss cache (nothing to cache) and hit the DB every time | **Many different keys** expire around the same time (or the cache goes down entirely), causing a broad spike in DB load across many keys at once, not just one |
| **Typical trigger** | Malicious/scripted requests probing for non-existent IDs, or a buggy client repeatedly querying a deleted/invalid ID | Keys all cached with the same TTL at the same time (e.g., a cache warm-up that set thousands of keys simultaneously), or a full Redis outage |
| **Prevention** | **Cache the "not found" result too** (with a short TTL), or a **Bloom filter** in front of the cache to cheaply reject requests for keys that provably don't exist before ever touching cache or DB | **Jitter/randomize TTLs** so keys don't all expire simultaneously; use Redis clustering/replication so a single node failure isn't a full outage; have a circuit breaker + fallback for total cache unavailability |

```java
public Product getProduct(String id) {
    String cached = redisTemplate.opsForValue().get("product:" + id);
    if ("__NULL__".equals(cached)) return null;              // cached negative result — Penetration fix
    if (cached != null) return deserialize(cached);

    Product fromDb = productRepository.findById(id).orElse(null);
    if (fromDb == null) {
        redisTemplate.opsForValue().set("product:" + id, "__NULL__", Duration.ofMinutes(2)); // short TTL
    } else {
        redisTemplate.opsForValue().set("product:" + id, serialize(fromDb),
            Duration.ofMinutes(10 + new Random().nextInt(5)));  // jittered TTL — Avalanche fix
    }
    return fromDb;
}
```

**Senior-level answer:**
> "The names sound similar but the failure shapes are opposite: penetration is 'the cache is being **bypassed** on every request for a specific, non-existent key' — an adversarial or edge-case problem; avalanche is 'the cache **stops helping across the board**, all at once' — a blast-radius/timing problem. I've seen penetration specifically weaponized in real attacks — scripted requests scanning through sequential non-existent IDs specifically to bypass the cache and load-test the DB directly — which is why the Bloom filter defense matters beyond just correctness."

**Trap:** Don't confuse Stampede (Q8, single hot key) with Avalanche (many keys) — interviewers frequently probe whether a candidate can articulate the precise distinction rather than treating all three "cache-breaking-under-load" scenarios as interchangeable.

---

## SECTION 5: DISTRIBUTED LOCKING

### Q10. How do you implement a simple distributed lock using Redis, and what's the classic bug in a naive implementation?
**Answer:** The building block is `SET key value NX PX ttl` — **atomically** set the key only if it doesn't already exist, with an expiry, in a single command (critical — doing the "check if exists" and "set" as two separate commands is a race condition).

```java
// Acquire
Boolean acquired = redisTemplate.opsForValue()
    .setIfAbsent("lock:order:555", requestId, Duration.ofSeconds(10));  // SET NX PX, atomic

if (Boolean.TRUE.equals(acquired)) {
    try {
        processOrder("555");
    } finally {
        // Release — MUST verify ownership before deleting (see below)
        releaseLockSafely("lock:order:555", requestId);
    }
}
```

**The classic naive bug:** releasing the lock with a plain `DEL key` **without checking who owns it**. If the lock's TTL expires while the holder is still working (a slow GC pause, a long DB call), **another process can acquire the lock**, and then the *original* holder finishes and blindly calls `DEL`, **deleting the second process's lock** — now two processes think they hold the lock simultaneously, which is exactly the correctness violation the lock was supposed to prevent.

**The fix — release only if you're still the owner, atomically via Lua script:**
```lua
-- Executed atomically in Redis (EVAL) — compare-and-delete
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Senior-level answer:**
> "This is the single most commonly botched distributed-locking detail, and it's a great senior-signal question because the naive version *looks* correct and even works fine in testing — it only breaks under real production timing conditions like GC pauses or network hiccups that make the lock outlive its intended holder. The fix — a unique token per acquisition, compare-and-delete via Lua for atomicity — is the standard approach, and it's exactly what's formalized in **Redlock** for multi-node Redis setups, though Redlock itself has known debated edge cases around clock drift that are worth knowing about but usually out of scope unless the interviewer pushes on it."

**Trap:** Setting the lock **without a TTL at all** is worse than the race condition above — if the holder crashes before releasing, the lock is held **forever**, deadlocking every future attempt to acquire it. A TTL is mandatory as a safety net even though it introduces the ownership-check subtlety described above.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Is Redis single-threaded? Doesn't that limit throughput?" → Redis's core command execution is single-threaded (avoiding lock contention on data structures), but it achieves high throughput via an efficient event loop and in-memory operations; I/O threading was added in Redis 6+ for network I/O, not command execution.
- "Can you use Redis as a primary database instead of just a cache?" → Technically yes with persistence (RDB/AOF) enabled, but it's generally treated as a cache/fast auxiliary store, not a system of record, due to weaker durability/query guarantees compared to a relational DB.
- "What's the difference between Redis Sentinel and Redis Cluster?" → Sentinel provides high availability/automatic failover for a single logical dataset (replica promotion on primary failure); Cluster provides horizontal sharding of data across multiple nodes for scale, with its own built-in failover per shard.
- "How would you cache a paginated/list query safely?" → Cache the ID list or page result under a key that includes the query parameters (page, filters, sort) in the cache key itself, with a shorter TTL than single-entity caches since list results change more often as underlying data is added/removed.
- "What eviction policy would you choose for a Redis cache instance?" → `allkeys-lru` is the common default for pure caching use cases (evict least-recently-used when memory is full); `volatile-lru` only evicts keys with a TTL set, useful when the same instance holds both cache data and data that must never be evicted.
- "Does @Cacheable in Spring handle cache stampede automatically?" → No — plain `@Cacheable` has no built-in mutex/locking for concurrent misses on the same key; stampede protection has to be implemented explicitly (Q8) if that's a real risk for a given cache.

---

*Study tip: The write-then-invalidate race condition (Q6), correctly distinguishing stampede vs. penetration vs. avalanche by failure shape (Q8–Q9), and the ownership-check bug in naive distributed lock release (Q10) are the three areas where interviewers most reliably separate senior candidates from mid-level ones — practice walking through the exact failure timeline for each, since that's what actually gets probed, not just naming the pattern.*