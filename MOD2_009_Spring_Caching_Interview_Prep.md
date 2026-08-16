# Spring Caching — Interview Prep
---

## SECTION 1: SPRING CACHE ABSTRACTION: @CACHEABLE, @CACHEPUT, @CACHEEVICT

### Q1. What problem does Spring's Cache abstraction solve, and how does it relate to Spring AOP under the hood?
**Answer:** Spring's caching abstraction (`@EnableCaching` + `@Cacheable`/`@CachePut`/`@CacheEvict`) lets you add caching behavior to a method **declaratively**, without hand-writing cache-lookup/store logic inside the method body, and without coupling business code to a specific caching provider (Caffeine, Redis, EhCache, etc.) — the provider is swapped via a `CacheManager` bean, the annotated code doesn't change.

Mechanically, it's implemented exactly like `@Transactional` (see the Spring Transactions/AOP prep docs) — **proxy-based AOP**. A `CacheInterceptor` wraps the method call: for `@Cacheable`, it checks the cache *before* the method body runs and can short-circuit the call entirely; for `@CacheEvict`/`@CachePut`, it runs cache logic around the invocation.

```java
@Service
public class ProductService {

    @Cacheable("products")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();   // only runs on a cache miss
    }
}
```

**Senior-level answer:**
> "The fact that this is proxy-based AOP under the hood matters practically, not just academically — it inherits the exact same **self-invocation problem** as `@Transactional`. If `findById()` is called from another method in the same class via `this.findById()`, the proxy is bypassed and caching silently never happens. I've debugged this exact symptom — 'why isn't my cache working' — and it traced straight back to an internal call that skipped the proxy, same root cause as the classic `@Transactional` self-invocation bug."

---

### Q2. @Cacheable vs @CachePut vs @CacheEvict — what does each do, and when do you use which?
**Answer:**

| Annotation | Behavior | Typical use |
|---|---|---|
| `@Cacheable` | Checks cache first; runs method **only on a miss**, then caches the result | Read paths — `findById`, `findByX` lookups |
| `@CachePut` | **Always** runs the method, then updates the cache with the result | Writes that should refresh the cache — `update()` |
| `@CacheEvict` | Removes an entry (or all entries) from the cache; doesn't touch the method's return value | Deletes, or writes where refreshing is unnecessary/expensive |

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);   // always executes, then refreshes the cache entry
    }

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);          // removes the stale entry
    }

    @CacheEvict(value = "products", allEntries = true)
    public void clearAllProducts() {                // bulk invalidation — e.g., after a batch import
    }
}
```

**Senior-level answer:**
> "The distinction I make sure people internalize: `@Cacheable` is **conditional** execution (skip the method on a hit), while `@CachePut` and `@CacheEvict` are **unconditional** — they always run the underlying method and only affect the cache as a side effect. A mistake I've seen more than once is putting `@Cacheable` on an `update()` method expecting it to refresh the cache — it doesn't; on a cache hit, `@Cacheable` would return the **stale** cached value without ever calling the method at all, silently preventing the update's return value from being reflected. `update()` needs `@CachePut`, specifically because you want the write to always execute and the fresh result to always overwrite the cache."

---

### Q3. How do you construct cache keys — default key generation vs custom SpEL keys — and what's the risk of getting this wrong?
**Answer:** By default, Spring generates a key from all method parameters via `SimpleKeyGenerator` — zero args → a fixed `SimpleKey.EMPTY`, one arg → the arg itself, multiple args → a composite `SimpleKey` wrapping all of them. Custom keys are expressed via SpEL in the `key` attribute.

```java
@Cacheable(value = "orders", key = "#customerId + '-' + #status")
public List<Order> findOrders(Long customerId, String status) { ... }

@Cacheable(value = "products", key = "#root.methodName + '-' + #id")
public Product findById(Long id) { ... }

@Cacheable(value = "search", key = "#criteria.toCacheKey()")   // delegate to a method on the argument
public List<Product> search(SearchCriteria criteria) { ... }
```

**Senior-level answer:**
> "The risk with default key generation is real once a method has multiple overloads or optional parameters — two different call signatures can silently collide or, more commonly, a method with an object parameter caches on the object's `equals()`/`hashCode()`, which may not reflect the fields that actually make the result unique. My rule: any method with more than one parameter, or a parameter that's a complex object, gets an **explicit** SpEL key — I don't rely on the default generator past the simplest single-primitive-argument case, because a key collision or over-broad key is a correctness bug that's genuinely hard to spot in code review, since the annotation looks fine at a glance."

---

### Q4. What does the condition and unless attributes do on @Cacheable, and give a practical example of each.
**Answer:** `condition` is evaluated **before** the method runs and decides whether caching applies at all (both read and write side); `unless` is evaluated **after** the method returns and can veto **caching the result** specifically, based on the return value — the method still executes either way.

```java
@Cacheable(value = "products", key = "#id", condition = "#id > 0")   // never cache for invalid/sentinel ids
public Product findById(Long id) { ... }

@Cacheable(value = "products", key = "#id", unless = "#result == null || #result.outOfStock")
public Product findById(Long id) {
    return productRepository.findById(id).orElse(null);   // avoid caching nulls or stale-looking results
}
```

**Senior-level answer:**
> "`unless` is the one I reach for constantly and think is under-used — caching `null` results (or an empty/error-shaped object) is a subtle trap: the cache then 'correctly' keeps returning null for an id that becomes valid moments later, and nobody notices until a support ticket comes in about 'this product doesn't show up.' I make it a standing rule that any `@Cacheable` method whose return type can be null or represent a not-found/error state gets an explicit `unless` guard — it's cheap insurance against caching negative or transient results as if they were permanent."

---

## SECTION 2: CAFFEINE VS REDIS; LOCAL VS DISTRIBUTED CACHING

### Q5. Caffeine vs Redis — what's the actual architectural difference, and how do you decide between them?
**Answer:**

| Aspect | Caffeine | Redis |
|---|---|---|
| Location | In-process (same JVM heap) | External, networked service |
| Speed | Nanoseconds — no network hop | Sub-millisecond, but still network round-trip |
| Shared across instances? | No — each app instance has its own cache | Yes — one shared cache visible to every instance |
| Data survives restart? | No | Yes (with persistence configured) |
| Consistency across a cluster | Each instance can diverge (stale on some nodes) | Single source of truth |
| Typical use | Hot, small, per-instance data (config, lookup tables) | Shared state across a horizontally-scaled service, session data, larger datasets |

```java
@Bean
public CacheManager caffeineCacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager("products");
    manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10)));
    return manager;
}
```

```java
@Bean
public CacheManager redisCacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));
    return RedisCacheManager.builder(factory).cacheDefaults(config).build();
}
```

**Senior-level answer:**
> "The deciding question I ask first is: **does this data need to be consistent across instances, or is per-instance staleness acceptable?** For something like a rarely-changing reference/lookup table (currency codes, feature flags with a short TTL), Caffeine's in-process speed is hard to beat and the small window of cross-instance inconsistency after an update is a non-issue. For anything where different instances returning different answers is a real bug — user session data, inventory counts, anything with write-heavy invalidation that all instances must see immediately — it has to be Redis, or a genuinely distributed cache. I've also used both together in a two-tier setup: Caffeine as a very-short-TTL L1 cache in front of Redis as L2, to cut down on network round-trips for extremely hot keys, while still getting Redis's cross-instance consistency as the source of truth."

---

### Q6. What's the "thundering herd across instances" risk specific to local (Caffeine) caching in a horizontally-scaled app, and how do you mitigate it?
**Answer:** With a local cache, **every instance** independently experiences its own cache misses on startup, deploy, or expiry — if 10 instances all restart together (a rolling deploy) and all immediately receive traffic for the same hot key, all 10 independently hit the database at roughly the same time, since there's no shared cache state to prevent it.

**Senior-level answer:**
> "This is a real operational risk I plan for specifically during deploys — a rolling restart of N instances, each with a cold local cache, hitting the DB simultaneously for the same hot keys can produce a real load spike right when the system is already in a more fragile state (mid-deploy). Mitigations I've used: staggering the rollout so instances don't all cold-start together, warming the cache proactively on startup for known-hot keys before accepting traffic, or — if the data genuinely needs cross-instance consistency and shared warm state — moving that specific cache to Redis instead, which doesn't have this problem since all instances share the same already-warm cache."

---

## SECTION 3: CACHE-ASIDE PATTERN; CACHE KEYS, TTL & EVICTION

### Q7. Explain the cache-aside (lazy-loading) pattern, and how it maps to what @Cacheable does automatically.
**Answer:** Cache-aside is the pattern where the **application**, not the cache itself, is responsible for keeping the cache and the database in sync: on a read, check the cache first; on a miss, read from the DB and populate the cache; on a write, update the DB and then evict (or update) the corresponding cache entry.

```
READ:  check cache → hit? return it
                   → miss? read DB → write to cache → return it

WRITE: write DB → evict (or update) cache entry
```

`@Cacheable` implements exactly the read side of this automatically; `@CacheEvict`/`@CachePut` implement the write side.

**Senior-level answer:**
> "Cache-aside is the default pattern I reach for precisely because it's simple and resilient to cache failure — if the cache is down or evicted, the app just falls through to the database, degraded but correct. The alternative patterns — write-through (writes go to the cache, which writes through to the DB) and write-behind (writes go to the cache, DB is updated asynchronously) — trade that simplicity for either tighter coupling to the cache being available, or genuine risk of data loss if the cache crashes before the async write-behind flushes. I've only reached for write-behind in genuinely high-throughput scenarios where DB write latency was the actual bottleneck, and even then, only for data where losing the last few seconds of writes on a crash was an acceptable risk."

---

### Q8. How do you decide on a TTL for a given cache entry, and what are the trade-offs of setting it too short vs too long?
**Answer:**

| TTL choice | Risk |
|---|---|
| Too short | Cache barely helps — high miss rate, most of the load-reduction benefit is lost, and you approach thundering-herd risk on every expiry |
| Too long | Stale data served for longer; increases blast radius of any cache-consistency bug; wastes memory on data that may never be re-read |

**Senior-level answer:**
> "I set TTL based on the actual **staleness tolerance of the data**, not a blanket default — a product catalog entry that changes rarely can tolerate a 30-minute or even hour-long TTL, while an inventory count that directly affects whether an order can be placed needs a TTL measured in seconds, or explicit eviction on write rather than relying on TTL expiry at all. I also deliberately **jitter** TTLs (e.g., `baseTtl + random(0, 60s)`) for large sets of keys that get populated around the same time, specifically to avoid a mass-simultaneous-expiry event causing a thundering herd — that's a small detail that's saved me from a real production incident before."

---

### Q9. What eviction policies exist for a bounded local cache (like Caffeine), and how do you choose one?
**Answer:**

| Policy | Behavior |
|---|---|
| **LRU** (Least Recently Used) | Evicts the entry that hasn't been *accessed* in the longest time |
| **LFU** (Least Frequently Used) | Evicts the entry with the fewest total accesses |
| **FIFO** | Evicts the oldest entry by insertion time, regardless of access pattern |
| **Size-based** | Evicts once a maximum entry count or weighted size is exceeded (Caffeine's `maximumSize`/`maximumWeight`) |
| **Time-based** | `expireAfterWrite` / `expireAfterAccess` — evicts purely on elapsed time |

```java
Caffeine.newBuilder()
        .maximumSize(50_000)
        .expireAfterAccess(Duration.ofMinutes(15))   // sliding expiry — resets on each read
        .recordStats()                                // enables hit/miss rate metrics
        .build();
```

**Senior-level answer:**
> "Caffeine actually uses a hybrid **Window TinyLFU** algorithm by default under `maximumSize`, which in practice out-performs plain LRU for most real-world access patterns by better resisting cache pollution from one-off scans — I don't usually need to hand-pick a policy, just set a sane size bound and let Caffeine's default algorithm do its job. The setting I actually think carefully about is `expireAfterWrite` vs `expireAfterAccess` — write-based expiry gives a predictable staleness ceiling regardless of traffic, while access-based (sliding) expiry can let a hot key live indefinitely, which is usually what you want for genuinely hot data but is the wrong choice if you need a hard guarantee that data is never older than X minutes, like a price that must reflect a recent update within a bounded time."

---

## SECTION 4: CACHE CONSISTENCY, STAMPEDE, PENETRATION & AVALANCHE

### Q10. What is cache stampede (thundering herd), and how do you prevent it at the code level?
**Answer:** Cache stampede happens when a **single popular key** expires (or is evicted) and a large burst of concurrent requests for that same key all miss simultaneously — all of them independently fall through to the database/origin at once, spiking load right at the moment the cache should have been protecting it.

```java
@Cacheable(value = "products", key = "#id", sync = true)   // only one thread computes on a miss; others wait
public Product findById(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

**Senior-level answer:**
> "Spring's `@Cacheable(sync = true)` is the simplest fix for a single-instance stampede — it makes concurrent callers block on the first thread's DB call instead of all independently hitting the DB, which is exactly the right trade-off for a genuinely hot, single key. For a distributed setup with Redis, `sync = true` doesn't help across instances, so I've used a **distributed lock** pattern instead — the first instance to miss acquires a short-lived lock (e.g., `SET key NX PX`), recomputes and repopulates the cache, while other instances either wait briefly and retry the cache read, or fall back to serving a slightly-stale value if one is available (stale-while-revalidate), rather than all hammering the DB in parallel."

---

### Q11. What is cache penetration, and how is it different from a normal cache miss?
**Answer:** Cache penetration is repeated querying for a key that **doesn't exist in the database at all** — since there's nothing to cache (a normal `@Cacheable` miss caches the found value), every request for that non-existent key falls through to the DB every single time, which an attacker (or a buggy client retrying an invalid id) can exploit to bypass the cache entirely and hammer the database directly.

**Senior-level answer:**
> "This is exactly why I pointed to `unless` guards in Q4 as insufficient by themselves — if you *never* cache a not-found result, penetration is trivially easy to trigger, deliberately or accidentally. The fix is to deliberately cache the **negative result too**, typically as a sentinel value (an empty object, or a dedicated 'not found' marker) with a **short** TTL — short specifically because if the record is created moments later, you don't want to serve a stale 'not found' for too long. I've also used a Bloom filter in front of the cache for very large keyspaces prone to penetration — a probabilistic check that can definitively say 'this key was never inserted' before even attempting a cache or DB lookup, which is a heavier tool I'd only reach for at genuine scale, not by default."

---

### Q12. What is cache avalanche, and how is it different from a stampede — and how do you defend against it?
**Answer:** Cache avalanche is when a **large number of different keys** expire (or the entire cache goes down) at roughly the same time, causing a broad spike in DB load across many queries at once — distinct from a stampede, which is concurrent load on **one** hot key. A common cause is populating many entries with the exact same fixed TTL, so they all expire in lockstep later.

**Senior-level answer:**
> "The two defenses I always pair together: **TTL jitter** (mentioned in Q8) so keys populated around the same time don't all expire at the exact same moment, and designing for **graceful degradation if the cache layer itself goes down entirely** — meaning the database and connection pool need to be able to survive a full cache outage without falling over, via things like a circuit breaker/rate limiter in front of the DB, or serving degraded/stale responses rather than let every request fail. I treat 'what happens if Redis is completely unreachable right now' as a question I want a concrete, tested answer to before shipping any cache-dependent feature — not an assumption that the cache will just always be up."

---

### Q13. How do you keep a Redis cache consistent with the database when the underlying data changes from multiple write paths (not just through your service)?
**Answer:** The hard case isn't a single service doing cache-aside correctly — it's when data can change via a path that bypasses your caching code entirely: a batch job, a direct DB migration, another microservice writing to a shared table, or an admin running a manual SQL fix.

**Senior-level answer:**
> "This is where I stop trusting explicit `@CacheEvict` calls as the *only* consistency mechanism, because they only fire for writes that go through your annotated methods — anything else silently leaves a stale cache entry with no way to know it's stale. My practical answers, layered: (1) always keep a bounded TTL as a safety net even on data that's also explicitly evicted on write, so staleness has a hard ceiling regardless of which write path caused it; (2) for systems where this is a frequent problem, use **event-driven invalidation** — the write path (wherever it lives) publishes a domain event (e.g., via Kafka or a DB change-data-capture stream) that any interested service consumes to evict its own cache, decoupling invalidation from any single service's code path; (3) for genuinely critical consistency needs, question whether that specific piece of data should be cached at all versus read live, since some data's correctness requirements just don't tolerate any staleness window."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "Does @CacheEvict run before or after the method executes by default?" → After, by default (`beforeInvocation = false`) — set `beforeInvocation = true` if you need eviction to happen even if the method throws an exception.
- "Can you cache a void method?" → Not meaningfully with `@Cacheable` (there's no return value to cache), but `@CacheEvict` on a void method is a completely normal and common pattern (e.g., a `delete()` method).
- "What happens if two @Cacheable methods use the same cache name but different key structures?" → They share the same underlying cache/keyspace, so key collisions become possible if the key generation isn't distinct enough — a good reason to use explicit, well-namespaced SpEL keys per method.
- "Is Spring's cache abstraction synchronous or does it support reactive/async caching?" → The classic annotations are synchronous; for reactive (`Mono`/`Flux`) return types, `@Cacheable` support is limited/version-dependent, and many teams instead cache manually inside the reactive chain (e.g., via a reactive Redis client) for full control.
- "What's the difference between expireAfterWrite and expireAfterAccess in Caffeine?" → `expireAfterWrite` is a fixed TTL from creation/update regardless of reads; `expireAfterAccess` is a sliding TTL that resets on every read — see Q9.
- "How would you monitor whether your cache is actually effective in production?" → Track hit/miss ratio (Caffeine's `recordStats()`, Redis's `INFO stats`/keyspace hit metrics) — a persistently low hit ratio signals either a TTL that's too short, a key strategy that's too fragmented, or data that simply isn't a good caching candidate.

---

*Study tip: The self-invocation proxy trap (Q1, same root cause as `@Transactional`'s), the `@Cacheable` vs `@CachePut` distinction and the stale-update bug it prevents (Q2), and being able to clearly articulate stampede vs penetration vs avalanche with a distinct mitigation for each (Q10–Q12) are the areas most likely to separate a senior candidate — interviewers use these specifically to check whether caching was learned from a real production incident or just from the annotation reference docs.*
