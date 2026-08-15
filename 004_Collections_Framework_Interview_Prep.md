# Collections Framework — Interview Prep — Senior Java Developer

---

## SECTION 1: LIST, SET, QUEUE, MAP HIERARCHY

### Q1. Draw/explain the Java Collections Framework hierarchy.
**Answer:**
```
Iterable
 └── Collection
      ├── List        (ordered, indexed, duplicates allowed)
      │     ├── ArrayList
      │     ├── LinkedList
      │     ├── Vector (legacy, synchronized)
      │     │     └── Stack (legacy)
      │     └── CopyOnWriteArrayList
      ├── Set         (no duplicates)
      │     ├── HashSet
      │     │     └── LinkedHashSet
      │     ├── SortedSet → NavigableSet → TreeSet
      │     └── CopyOnWriteArraySet
      └── Queue       (FIFO by default, ordering may vary)
            ├── PriorityQueue
            ├── ArrayDeque
            └── BlockingQueue (interface) → ArrayBlockingQueue, LinkedBlockingQueue, etc.
                  └── Deque (interface) → ArrayDeque, LinkedList

Map (NOT a Collection — separate root, key point!)
 ├── HashMap
 │     └── LinkedHashMap
 ├── SortedMap → NavigableMap → TreeMap
 ├── Hashtable (legacy, synchronized)
 └── ConcurrentHashMap
```

**Key trap to state explicitly:** `Map` does **not** extend `Collection` — it's a completely separate interface hierarchy, because a `Map` operates on key-value pairs, not single elements, and its iteration semantics (`entrySet()`, `keySet()`, `values()`) don't fit the single-element `Iterable` contract naturally. This is asked constantly as a quick trap question.

---

### Q2. What's the core contract difference between List, Set, and Queue?
- **List:** ordered, allows duplicates, indexed access (`get(int index)`).
- **Set:** no duplicates (uniqueness enforced via `equals()`/`hashCode()` or `compareTo()`), generally no indexed access.
- **Queue:** designed for **holding elements for processing**, typically FIFO (though `PriorityQueue` reorders by priority, and `Deque` supports both ends) — offers specialized methods (`offer`, `poll`, `peek`) that return `null`/`false` on failure instead of throwing, unlike `Collection`'s `add`/`remove` which throw on failure.

**Follow-up:** "Why does `Queue` have both `add()`/`offer()` and `remove()`/`poll()` pairs that seem to do the same thing?" → `add()`/`remove()`/`element()` **throw exceptions** on failure (e.g., `add()` on a full bounded queue, `remove()`/`element()` on an empty queue); `offer()`/`poll()`/`peek()` return a **sentinel value** (`false`/`null`) instead — giving you a choice between exception-based and null/boolean-based failure handling depending on whether the condition is "exceptional" or "expected" in your use case (ties directly back to the Exception Handling best-practices discussion).

---

## SECTION 2: ArrayList vs LinkedList vs Vector

### Q3. ArrayList vs LinkedList — internal structure and performance comparison
| Operation | ArrayList | LinkedList |
|---|---|---|
| Internal structure | Dynamic resizable array | Doubly linked list (nodes with prev/next pointers) |
| `get(index)` | O(1) — direct index access | O(n) — must traverse from head or tail |
| `add(element)` at end | O(1) amortized (occasional resize) | O(1) |
| `add(index, element)` middle | O(n) — shifts elements | O(n) to find position + O(1) to link, but still O(n) overall |
| `remove(index)` | O(n) — shifts elements | O(n) to find + O(1) to unlink |
| Memory overhead | Lower (just the array + unused capacity) | Higher (each node has 2 pointers + object header overhead) |
| Cache locality | Good (contiguous memory) | Poor (nodes scattered in heap, pointer-chasing) |

**Senior-level nuance interviewers want to hear:** "In practice, `LinkedList`'s theoretical O(1) insert/delete-in-middle advantage rarely wins in real benchmarks, because **cache locality** dominates modern CPU performance — `ArrayList`'s contiguous memory layout means the CPU cache prefetcher works in its favor, while `LinkedList`'s node-per-element pointer-chasing causes frequent cache misses. Unless I specifically need frequent insertions/deletions at both ends (where I'd actually reach for `ArrayDeque` over `LinkedList` anyway) or true O(1) known-position insert/delete via an existing `ListIterator`, I default to `ArrayList` almost always."

**Follow-up:** "Is `LinkedList` still useful for anything?" → It implements `Deque`, so it can be used as a stack/queue, but `ArrayDeque` is now the generally preferred choice for that too (better cache performance, no per-node overhead) — `LinkedList` has become a fairly rare choice in modern production code.

---

### Q4. ArrayList internal resizing — how does dynamic growth work?
**Answer:** `ArrayList` starts with a default capacity (historically 10, though since Java 7+ an empty `ArrayList()` actually starts with an internal empty array and only allocates capacity 10 on the **first** `add()`). When capacity is exceeded, it creates a **new array of larger size** (growth factor is **1.5x** the old capacity: `newCapacity = oldCapacity + (oldCapacity >> 1)`) and copies all elements via `Arrays.copyOf()` — an O(n) operation, but **amortized O(1)** across many `add()` calls since resizes happen exponentially less often as size grows.

**Senior production tip:** If you know the approximate final size upfront, **always use the sized constructor** — `new ArrayList<>(expectedSize)` — to avoid repeated resize/copy operations, a small but real performance win frequently flagged in code review for hot paths building large lists.

---

### Q5. Vector vs ArrayList — why is Vector considered legacy?
**Answer:** `Vector` is functionally almost identical to `ArrayList` (dynamic array), but **every method is `synchronized`** — meaning even single-threaded usage pays lock-acquisition overhead unnecessarily. Its growth factor also differs (doubles by default, `2x`, vs `ArrayList`'s `1.5x`). It predates the Collections Framework (Java 1.0) and is retained only for backward compatibility.

**Why it's a poor choice even for concurrent use:** `Vector`'s synchronization is **per-method only**, not compound-operation safe — e.g., `if (!vector.contains(x)) vector.add(x)` is still a race condition even with a synchronized `Vector`, because the check-then-act isn't atomic as a whole. For genuine thread-safe list needs, prefer `Collections.synchronizedList(new ArrayList<>())` (still has the same compound-operation caveat, but is at least explicit about it) or, better, `CopyOnWriteArrayList` for read-heavy scenarios.

---

## SECTION 3: HashSet vs TreeSet vs LinkedHashSet

### Q6. HashSet vs TreeSet vs LinkedHashSet — comparison
| Aspect | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Ordering | No guaranteed order | Insertion order preserved | Sorted order (natural or via Comparator) |
| Backing structure | `HashMap` internally | `LinkedHashMap` internally | `TreeMap` (Red-Black tree) internally |
| `add`/`remove`/`contains` | O(1) average | O(1) average | O(log n) |
| Null elements | One `null` allowed | One `null` allowed | **No `null` allowed** (NPE on comparison) |
| Use case | Fast uniqueness check, order doesn't matter | Fast uniqueness + predictable iteration order | Sorted iteration, range queries (`headSet`, `tailSet`, `ceiling`, `floor`) |

**Senior detail:** `HashSet` is literally implemented as a `HashMap<E, Object>` internally, where every element is stored as a **key**, mapped to a shared dummy constant `PRESENT` object as the value — this is a favorite "how is it actually implemented under the hood" follow-up.

```java
// Simplified real JDK implementation detail
private transient HashMap<E,Object> map;
private static final Object PRESENT = new Object();
public boolean add(E e) {
    return map.put(e, PRESENT) == null;
}
```

**Follow-up:** "Why does `TreeSet` throw NPE when adding `null`, but `HashSet` allows it?" → `TreeSet` must **compare** elements to maintain sort order (`compareTo()`/`Comparator.compare()`), and comparing against `null` throws `NullPointerException` by contract — `HashSet` only needs `hashCode()`/`equals()`, and `null.hashCode()` is specially handled (bucket index 0) without issue.

---

### Q7. How does TreeSet maintain sorted order internally? What's its time complexity for range queries?
**Answer:** `TreeSet` is backed by a `TreeMap`, which uses a **Red-Black Tree** (a self-balancing binary search tree) — guarantees O(log n) for `add`, `remove`, `contains`, and enables efficient **`NavigableSet`** operations:
```java
TreeSet<Integer> set = new TreeSet<>(List.of(10, 20, 30, 40, 50));
set.ceiling(25);   // 30 — smallest element >= 25
set.floor(25);     // 20 — largest element <= 25
set.headSet(30);   // [10, 20] — elements < 30
set.tailSet(30);   // [30, 40, 50] — elements >= 30
```
These range operations are also O(log n) (tree traversal to the boundary), far better than sorting an `ArrayList` on every query (O(n log n) per query) — a strong point to raise when discussing "when would you choose TreeSet."

---

### Q8. How would you sort a HashSet, and how would you maintain custom sort order in a Set?
**Answer:** `HashSet` itself has no order — to get sorted output, either copy into a `TreeSet` (natural order or custom `Comparator`), or copy into a `List` and call `Collections.sort()`. For a `Set` that needs to **maintain** a custom sort order continuously (not just a one-time sort), use `TreeSet` with a `Comparator` passed to its constructor:
```java
TreeSet<Employee> byExperience = new TreeSet<>(Comparator.comparing(Employee::getYearsExperience).reversed());
```
Covered in depth in Section 7 (Comparable vs Comparator).

---

## SECTION 4: HashMap INTERNAL WORKING (Buckets, Hashing)

### Q9. Explain how HashMap works internally — buckets, hashing, collision resolution. (THE single most-asked Collections question at senior level — expect deep follow-ups)

**Answer:**

**Structure:** `HashMap` internally maintains an array of **buckets** (`Node<K,V>[] table`), default initial capacity 16, default **load factor 0.75**. Each bucket is a **linked list** of entries that hash to the same index (collision chain) — or, since **Java 8**, a **Red-Black tree** if a bucket's chain grows too long (treeification, Q11).

**Put operation flow:**
1. Compute `hash(key)` — JDK applies a **hash spreading function**: `(h = key.hashCode()) ^ (h >>> 16)` — XORs the high 16 bits into the low 16 bits, to better distribute hash codes across buckets and reduce collisions from poor-quality `hashCode()` implementations (especially relevant since capacity is always a power of 2, and using just the low bits directly would waste the high-bit entropy).
2. Compute bucket index: `(capacity - 1) & hash` — a bitwise AND, equivalent to `hash % capacity` but faster, only works correctly because capacity is always a power of 2.
3. If the bucket is empty, insert directly.
4. If not empty (collision), traverse the chain: if a key with `equals()` match is found, **replace** the value; otherwise, **append** to the chain (Java 8+: appends to the **end** of the list, Java 7 prepended to the front — a subtle but real behavioral change, relevant to the classic Java 7 concurrent-modification infinite-loop bug, Q13).
5. If chain length exceeds **8** (`TREEIFY_THRESHOLD`) **and** total table capacity is at least 64, the bucket converts from linked list to a red-black tree (O(log n) worst case instead of O(n)).
6. If `size > capacity * loadFactor` (i.e., > 12 at default 16/0.75), the table **resizes** — doubles capacity, and **rehashes** all entries into the new, larger table.

```java
// Simplified conceptual view
int hash = (h = key.hashCode()) ^ (h >>> 16);
int index = (table.length - 1) & hash;
```

**Follow-up:** "Why is default capacity 16 and load factor 0.75 specifically?" → Powers of 2 for capacity enable the fast bitmask-based index calculation (step 2). 0.75 is a JDK-chosen balance — lower load factor = more space used but fewer collisions (faster lookups); higher = less space but more collisions (slower). 0.75 is empirically a good time/space trade-off for general-purpose use, per the JDK documentation itself.

---

### Q10. What happens during HashMap resizing? Why is it expensive, and what production issue can it cause?
**Answer:** When the resize threshold is crossed, `HashMap` allocates a **new array double the size**, then **rehashes and redistributes every existing entry** into the new table (each entry's bucket index is recalculated against the new capacity) — an O(n) operation. Because this can happen unpredictably mid-execution when the map grows past a threshold, it causes a **latency spike** on whichever `put()` call triggers it.

**Production tip (senior-level, high value):** If you know the expected size upfront, **initialize `HashMap` with a sized capacity** to avoid resize-triggered rehashing during hot-path operations:
```java
// Avoids multiple resizes if you expect ~1000 entries
Map<String, Order> cache = new HashMap<>((int) (1000 / 0.75) + 1);
```

---

### Q11. What is treeification in HashMap (Java 8+)? Explain the constants involved.
**Answer:** When a single bucket's collision chain grows to **8 or more nodes** (`TREEIFY_THRESHOLD = 8`) **and** the table's total capacity is **at least 64** (`MIN_TREEIFY_CAPACITY = 64` — below this, the JDK prefers to just resize the table instead of treeifying, since a small table with a long chain likely just needs more buckets, not a tree), the bucket converts from a **linked list to a red-black tree**, changing worst-case lookup from **O(n) to O(log n)**.

**Why this was added:** To defend against **hash-flooding attacks** — if an attacker can predict/control keys with intentionally colliding hash codes (e.g., certain crafted `String` keys in a web app accepting user-controlled map keys), a linked-list-based `HashMap` degrades to O(n) per lookup, letting an attacker cause a denial-of-service via algorithmic complexity attack. Treeification caps the worst case at O(log n), mitigating this.

**Untreeification:** If entries are removed and the bucket shrinks back below **6** (`UNTREEIFY_THRESHOLD = 6`), it converts back to a linked list (a small gap between 6 and 8 avoids constant flapping back and forth at the boundary — a classic hysteresis design pattern, worth mentioning if you want to show real depth).

---

### Q12. Why must you override both `equals()` and `hashCode()` together for a class used as a HashMap key? (Deep dive with Section 9 below)
**Short answer here (full contract detail in Section 9):** `HashMap` uses `hashCode()` to locate the correct bucket, then `equals()` to find the exact matching key within that bucket's collision chain. If you override `equals()` without `hashCode()` (or with an inconsistent one), two "equal" objects can land in **different buckets**, so `map.get(key)` can fail to find an entry that logically should match — a very common, subtle real bug.

---

### Q13. What was the infamous Java 7 HashMap infinite-loop bug under concurrent modification, and how did Java 8 fix it?
**Answer:** In Java 7, `HashMap`'s resize operation **rehashed entries by prepending them to the new bucket's chain**, which — during **concurrent, unsynchronized** resizes by multiple threads — could reverse the order of a chain's links in a way that created a **circular reference (cycle)** in the linked list. Any subsequent `get()` on that bucket would then **loop forever**, pegging CPU at 100% — a real, notorious production incident class at companies running unsynchronized `HashMap` under concurrent load.

**Java 8's fix:** Changed the resize algorithm to **append to the end of the chain** (preserving relative order) rather than prepend, which structurally eliminates the specific cycle-creation bug — but **this does NOT make `HashMap` thread-safe in Java 8+** — concurrent modification can still cause **lost updates, data corruption, or `ConcurrentModificationException`** during iteration; it just no longer causes the specific infinite-loop pathology. **The correct fix was always, and remains, to never share a plain `HashMap` across threads without external synchronization — use `ConcurrentHashMap` instead.** This is a very strong, specific answer that signals real depth if asked "have you seen HashMap cause a production incident."

---

## SECTION 5: HashMap vs ConcurrentHashMap vs Hashtable

### Q14. HashMap vs ConcurrentHashMap vs Hashtable — full comparison
| Aspect | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Thread safety | None | Fully synchronized (every method) | Fine-grained internal locking (CAS + per-bin sync) |
| Null keys/values | 1 null key, multiple null values allowed | **No nulls allowed at all** (throws NPE) | **No nulls allowed** (deliberately, see Q15) |
| Performance under concurrency | N/A (unsafe) | Poor — single lock for entire map | Good — concurrent reads always, concurrent writes to different bins |
| Iterator behavior | Fail-fast (Section 8) | Fail-fast | **Fail-safe / weakly consistent** (Section 8) |
| Era | Modern (JCF) | Legacy (Java 1.0) | Modern concurrent (java.util.concurrent) |

**Internal mechanism recap (deeper detail already covered in Multithreading prep, worth summarizing here too):** Java 7 `ConcurrentHashMap` used **segment-based locking** (16 segments by default); Java 8+ dropped segments in favor of **CAS for empty-bucket insertion** and **synchronized blocks scoped to individual bin nodes** only on actual collision — far more granular, higher throughput than segment locking, and much higher than `Hashtable`'s single whole-map lock.

---

### Q15. Why does ConcurrentHashMap disallow null keys/values, while HashMap allows them?
**Answer (a genuinely excellent, high-signal senior answer, cite Doug Lea directly if you can recall):** In a **single-threaded** `HashMap`, if `map.get(key)` returns `null`, you can safely follow up with `map.containsKey(key)` to disambiguate "key maps to null" from "key isn't present" — no race condition, since nothing else is mutating the map concurrently. In a **concurrent** map, that two-step check-then-act is **inherently racy** — between your `get()` returning `null` and your subsequent `containsKey()` call, another thread could have inserted or removed the key, making the disambiguation **unreliable and misleading**. Doug Lea (the author of `java.util.concurrent`) made the deliberate design decision to **disallow nulls entirely** in `ConcurrentHashMap` to eliminate this entire class of ambiguity/race — forcing you to use `Optional` or a sentinel value instead if you need to represent "no value" explicitly.

---

### Q16. Is `ConcurrentHashMap.size()` guaranteed to be exact at the moment you call it?
**Answer:** No — under concurrent modification, `size()` (and similar aggregate methods like `isEmpty()`, iteration counts) returns a **best-effort approximate** value, since the map could be mutated by other threads during or immediately after the call completes. This is an accepted, deliberate trade-off (**weak consistency**, Section 8) in exchange for not requiring a global lock just to compute size — a good follow-up to mention proactively when discussing `ConcurrentHashMap` semantics.

---

## SECTION 6: TreeMap vs LinkedHashMap

### Q17. TreeMap vs LinkedHashMap vs HashMap — ordering and use cases
| Aspect | HashMap | LinkedHashMap | TreeMap |
|---|---|---|---|
| Ordering | None (implementation-defined, bucket order) | Insertion order (or access order, configurable) | Sorted (natural order or Comparator) |
| Backing structure | Hash table (buckets) | Hash table + doubly-linked list threading entries | Red-Black tree |
| get/put complexity | O(1) average | O(1) average | O(log n) |
| Null keys | 1 allowed | 1 allowed | **Not allowed** (NPE on comparison, same reasoning as TreeSet) |

**Real use case for `LinkedHashMap`:** Building an **LRU (Least Recently Used) cache** — `LinkedHashMap` has a constructor supporting **access-order** mode, and an overridable `removeEldestEntry()` hook, making it a near-perfect, minimal-code building block for LRU eviction:

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    public LRUCache(int maxSize) {
        super(16, 0.75f, true);   // true = access-order (most-recently-used moves to end)
        this.maxSize = maxSize;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;   // auto-evict the least-recently-used entry once over capacity
    }
}
```
This is a **very commonly asked live-coding question** — "implement an LRU cache" — and knowing this `LinkedHashMap` shortcut (vs. hand-rolling a `HashMap` + `Deque` combo) is a strong senior-level signal, though interviewers may also want you to implement it manually to test your understanding of the underlying mechanics — be ready for both versions.

**Real use case for `TreeMap`:** Any scenario needing sorted iteration or range queries on keys — e.g., a time-series map keyed by `LocalDateTime` where you need `tailMap(fromTime)` to efficiently get all entries after a certain point, without scanning the whole map.

---

### Q18. How does LinkedHashMap maintain insertion order internally?
**Answer:** It extends `HashMap` and adds a **doubly linked list** threading through all entries in insertion (or access) order, in addition to the normal bucket-array structure. Each `LinkedHashMap.Entry` extends `HashMap.Node` with additional `before`/`after` pointers. Lookups (`get`/`put`) still use the standard `HashMap` bucket mechanism (O(1)); the linked list is purely for **maintaining iteration order**, adding a small memory/bookkeeping overhead over plain `HashMap` in exchange for predictable iteration.

---

## SECTION 7: COMPARABLE vs COMPARATOR

### Q19. Comparable vs Comparator — full comparison
| Aspect | Comparable | Comparator |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `int compareTo(T other)` | `int compare(T o1, T o2)` |
| Where defined | Inside the class itself (**natural ordering**) | External, separate class/lambda (**custom ordering**) |
| How many orderings | Only one (the class defines a single natural order) | Unlimited — as many `Comparator`s as you want |
| Used by | `Collections.sort(list)`, `TreeSet`/`TreeMap` with no explicit comparator | `Collections.sort(list, comparator)`, `TreeSet`/`TreeMap` constructor, `Stream.sorted(comparator)` |

```java
class Employee implements Comparable<Employee> {
    int salary;
    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.salary, other.salary);   // natural order: by salary
    }
}

// External, flexible orderings via Comparator — as many as needed, without touching the Employee class
Comparator<Employee> byNameThenSalary = Comparator
    .comparing(Employee::getName)
    .thenComparing(Employee::getSalary, Comparator.reverseOrder());

employees.sort(byNameThenSalary);
```

**Senior decision framework:** "I implement `Comparable` when there's a single, obvious, intrinsic natural ordering for the type (e.g., `BigDecimal` naturally orders by value). I use `Comparator` — especially the Java 8 fluent builder methods (`comparing`, `thenComparing`, `reversed`) — for anything context-dependent or where multiple orderings are needed, since it keeps sorting logic decoupled from the domain class itself and composable at the call site."

---

### Q20. What's the contract for `compareTo()`/`compare()`? What happens if it's inconsistent?
**Answer:** Must return negative/zero/positive consistently, and must be:
- **Consistent with itself** — `sgn(compare(x, y)) == -sgn(compare(y, x))`.
- **Transitive** — if `compare(x, y) > 0` and `compare(y, z) > 0`, then `compare(x, z) > 0`.
- **Recommended (not strictly required) to be consistent with `equals()`** — if `x.equals(y)`, ideally `compare(x, y) == 0`, though the JDK explicitly permits exceptions "if a class has a natural ordering inconsistent with equals" — worth naming explicitly since it's a genuinely subtle JDK-documented caveat.

**What breaks if the contract is violated:** Sorting algorithms can throw `IllegalArgumentException: Comparison method violates its general contract!` (Java 7+'s TimSort added this **explicit runtime validation**, whereas older merge sort implementations might just silently produce incorrectly sorted/corrupted results) — and `TreeMap`/`TreeSet` can silently lose or "swallow" entries that appear "equal" per a broken comparator even though they're logically distinct objects, since `TreeSet`/`TreeMap` use `compareTo()`/`compare()` — **not `equals()`** — for uniqueness when a `Comparator` is present. This last point is a genuinely tricky, high-value trap:

```java
TreeSet<String> set = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
set.add("Apple");
set.add("apple");   // compare() returns 0 (case-insensitive) — treated as a DUPLICATE, silently NOT added!
System.out.println(set.size());   // 1, not 2 — because TreeSet uses compare()==0 for equality, not equals()
```

---

## SECTION 8: FAIL-FAST vs FAIL-SAFE ITERATORS

### Q21. Fail-fast vs fail-safe iterators — explain with the underlying mechanism
**Answer:**

**Fail-fast** (`ArrayList`, `HashMap`, `HashSet`, and most standard JCF collections): Detects **structural modification** (add/remove, not just element value change via `set()`) during iteration by tracking a `modCount` field, incremented on every structural change. The iterator checks `modCount` against an internally cached `expectedModCount` on every `next()` call; a mismatch throws `ConcurrentModificationException` **immediately** — "fails fast" rather than allowing undefined/corrupted iteration behavior to continue silently.

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) list.remove(s);   // throws ConcurrentModificationException on next iteration!
}
```

**Correct fix:** use `Iterator.remove()` (which updates `expectedModCount` correctly, since the iterator itself knows about its own removal), or `removeIf()`, or iterate over a copy:
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();   // safe — Iterator's own remove() keeps modCount in sync
}
// or, cleaner in modern code:
list.removeIf(s -> s.equals("b"));
```

**Fail-safe / weakly consistent** (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `CopyOnWriteArraySet`): Iterators operate on either a **snapshot** (`CopyOnWriteArrayList` — iterates the array as it was at iterator-creation time, since writes create an entirely new array) or provide **weak consistency** (`ConcurrentHashMap` — iterator reflects the state at some point during iteration, may or may not see concurrent updates, but never throws `ConcurrentModificationException` and never corrupts). No exception is thrown even if the underlying collection changes during iteration.

**Important nuance to state explicitly (interviewers love this precision):** "Fail-safe" doesn't mean "sees all the latest data" — it means "won't throw/corrupt," but the iteration might not reflect concurrent modifications made after the iterator started (particularly for `CopyOnWriteArrayList`'s true snapshot semantics) — an important distinction from "always up to date."

**Follow-up:** "Is fail-fast behavior a guarantee you should rely on for correctness?" → **No** — the JDK docs explicitly state `ConcurrentModificationException` detection is **best-effort**, not guaranteed (`modCount` checks can theoretically miss some race conditions) — it exists purely to help catch bugs during development, **not** as a reliable concurrency-safety mechanism. Never rely on it as an actual thread-safety guarantee.

---

### Q22. Why doesn't ConcurrentHashMap throw ConcurrentModificationException, but a synchronized HashMap wrapped via Collections.synchronizedMap() still can?
**Answer:** `Collections.synchronizedMap(new HashMap<>())` just wraps a plain `HashMap` with synchronized method calls — the underlying `HashMap`'s **iterator is still the same fail-fast iterator**, tracking `modCount` exactly as before; synchronizing individual method calls doesn't change the iterator's fundamental fail-fast detection mechanism, and the JDK docs explicitly require you to manually synchronize on the map **during iteration** to avoid `ConcurrentModificationException` even with the synchronized wrapper. `ConcurrentHashMap`, by contrast, was **purpose-built from the ground up** with a genuinely different, weakly-consistent iterator design that doesn't use `modCount`-based fail-fast detection at all — a structurally different guarantee, not just "more locking."

---

## SECTION 9: equals() AND hashCode() CONTRACT

### Q23. What is the equals()/hashCode() contract? Why must they be overridden together?
**Answer — the formal contract (know this precisely, it's asked verbatim very often):**
1. **Reflexive:** `x.equals(x)` must be `true`.
2. **Symmetric:** `x.equals(y)` must equal `y.equals(x)`.
3. **Transitive:** if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)` must be `true`.
4. **Consistent:** multiple invocations of `x.equals(y)` must consistently return the same result, provided no fields used in the comparison change.
5. **Non-null:** `x.equals(null)` must return `false`.

**The critical `hashCode()` link (the actual crux of "why override together"):**
> **If two objects are equal according to `equals()`, they MUST have the same `hashCode()`.** (The reverse is NOT required — unequal objects *can* share a hash code, that's just a collision, which hash-based collections handle via chaining/treeification.)

**Why this matters concretely:** Hash-based collections (`HashMap`, `HashSet`) use `hashCode()` to locate the bucket **first**, then `equals()` **only within that bucket** to find the exact match. If you override `equals()` but leave the default `Object.hashCode()` (identity-based, essentially memory-address-derived), two logically "equal" objects can compute **different hash codes**, land in **different buckets**, and a hash-based collection will **fail to recognize them as the same entry** — `contains()`/`get()` silently return false/null even though an "equal" object is technically present elsewhere in the map. This is one of the most common, subtle real-world bugs in Java codebases.

```java
class Employee {
    String id;
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        return id.equals(((Employee) o).id);
    }
    // BUG: hashCode() NOT overridden — uses default identity hash!
}

Set<Employee> set = new HashSet<>();
set.add(new Employee("E101"));
System.out.println(set.contains(new Employee("E101")));   // FALSE! Different object, different default hashCode,
                                                            // lands in a different bucket — equals() never even gets checked
```

---

### Q24. How would you correctly implement equals() and hashCode() for a class? What tools/utilities help?
```java
class Employee {
    private final String id;
    private final String department;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                          // reflexive shortcut, also handles identity fast-path
        if (o == null || getClass() != o.getClass()) return false;   // exact class match (see Q25 for getClass() vs instanceof debate)
        Employee employee = (Employee) o;
        return Objects.equals(id, employee.id) &&
               Objects.equals(department, employee.department);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, department);   // combines field hash codes correctly, handles nulls safely
    }
}
```
**Best practice:** Use `java.util.Objects.equals()` (null-safe field comparison) and `Objects.hash()` (varargs, null-safe hash combination) rather than hand-rolling null checks and multiplication-based hash combining — cleaner, less error-prone, standard since Java 7. Modern IDEs (IntelliJ, Eclipse) also auto-generate correct implementations — worth mentioning that in real projects you'd rarely hand-write this from scratch, but you absolutely need to understand *why* the generated code looks the way it does.

**Immutability tie-in (senior-level point worth raising proactively):** Only include **immutable fields** (or fields that won't change after the object is inserted into a hash-based collection) in `hashCode()` — if a field used in `hashCode()` mutates after insertion, the object's bucket location becomes "stale," and `get()`/`contains()` can silently fail to find it anymore, since the object is now in the wrong bucket relative to its current (changed) hash code. This is exactly why **mutable objects make risky/fragile `HashMap` keys** — a very real production gotcha.

---

### Q25. `getClass() == o.getClass()` vs `instanceof` in equals() — which is correct, and why does it matter?
**Answer:** This is a genuinely debated, nuanced point that shows real depth if you can articulate both sides:
- **`instanceof`** allows a subclass to be considered equal to its superclass instance (if fields match) — but this can **violate the symmetry contract** in tricky subclassing scenarios (`parent.equals(child)` might be true while `child.equals(parent)` is false, or vice versa, if the subclass adds extra fields to the equality check).
- **`getClass() == o.getClass()`** is stricter — only allows equality between objects of the **exact same runtime class**, which guarantees symmetry/transitivity cleanly, but breaks the **Liskov Substitution Principle** somewhat (a subclass instance can never `.equals()` a superclass instance even conceptually).

**Effective Java's (Joshua Bloch) guidance, worth citing directly:** Prefer **composition over inheritance** for value-like classes specifically to sidestep this problem entirely, and when you do need inheritance with `equals()`, `getClass()` comparison is generally safer for maintaining the contract correctly — this is a strong, citable, senior-level answer if this follow-up comes up.

---

### Q26. What happens if you override equals() but the class is used as a key in a TreeMap/TreeSet instead of HashMap/HashSet — does the equals()/hashCode() contract still apply?
**Answer:** **No — this is the trap already previewed in Q20.** `TreeMap`/`TreeSet` use `compareTo()`/`compare()` exclusively for both ordering **and** uniqueness determination — `equals()`/`hashCode()` are **not consulted at all** by tree-based collections for structural correctness (though `TreeSet.equals()` as a whole-collection comparison, inherited from `AbstractSet`, does still use element `equals()` — a subtle distinction). This means a `TreeSet`'s notion of "duplicate" can genuinely diverge from a `HashSet`'s notion of "duplicate" for the exact same class, if `compareTo()` and `equals()`/`hashCode()` aren't kept logically consistent — reinforcing why the JDK explicitly warns that natural ordering "should generally be consistent with equals."

---

## SECTION 10: SPECIALIZED COLLECTIONS (WeakHashMap, IdentityHashMap, PriorityQueue, Deque, CopyOnWriteArrayList)

### Q27. What is WeakHashMap? How is it different from HashMap, and what's a real use case?
**Answer:** `WeakHashMap` holds its **keys** via `WeakReference`s. If a key is no longer strongly referenced **anywhere else** in the application, the garbage collector is free to reclaim it — and once that happens, the corresponding entry is **automatically removed** from the map on a subsequent GC cycle (via a `ReferenceQueue` mechanism internally). A regular `HashMap` holds strong references to its keys, so entries live forever until explicitly removed, even if nothing else in the app still needs that key.

**Real use case:** Caches where you want entries to be **automatically evicted once the key object is no longer used elsewhere** — e.g., associating metadata with `Class` objects (`ClassLoader`-scoped caches, exactly how some frameworks like `ThreadLocal` internals and certain ORM metadata caches work), or a listener registry where you don't want to accidentally keep a listener's associated object alive forever just because it's a map key.

```java
Map<Key, Value> cache = new WeakHashMap<>();
Key key = new Key("session-123");
cache.put(key, someValue);
key = null;           // no more strong references to the original Key object
System.gc();          // eligible for collection — entry may be removed from the map after this
```

**Critical caveat interviewers want you to state:** Only the **key** is weakly referenced — if the **value** strongly references the key back (a common mistake, e.g., storing the key inside the value object), you create a reference cycle that **prevents collection entirely**, defeating the whole purpose. Also, eviction timing is **not deterministic** — it depends on GC running, so `WeakHashMap` is unsuitable for any use case needing predictable/immediate eviction (use `Caffeine`/Guava's explicit TTL-based caches for that instead).

---

### Q28. What is IdentityHashMap? How does it differ semantically from HashMap?
**Answer:** `IdentityHashMap` uses **reference equality (`==`)** instead of `equals()`/`hashCode()` to determine key (and value) uniqueness — internally, it uses `System.identityHashCode()` rather than the key's own `hashCode()`. Two keys that are `.equals()` to each other but are **different object instances** are treated as **distinct keys** in an `IdentityHashMap`.

```java
Map<String, String> hashMap = new HashMap<>();
String a = new String("key");
String b = new String("key");
hashMap.put(a, "value1");
System.out.println(hashMap.containsKey(b));   // true — equals()-based, "key".equals("key")

Map<String, String> identityMap = new IdentityHashMap<>();
identityMap.put(a, "value1");
System.out.println(identityMap.containsKey(b));   // false — different object references, even though content is equal!
```

**Real use case:** Framework/infrastructure code that needs to track objects by **exact identity**, deliberately bypassing custom `equals()` overrides — e.g., serialization frameworks detecting object graph cycles (tracking "have I already visited this exact object instance"), or some dependency-injection containers tracking bean instances internally. **Rarely used in typical business/application code** — mostly a framework-internals tool, worth knowing exists but not something you'd reach for in a service layer.

**Follow-up trap:** "Does `IdentityHashMap` violate the general `Map` interface contract?" → Yes, explicitly — the JDK docs state this openly; it deliberately breaks the `Map` contract (which specifies `equals()`-based behavior) in exchange for identity semantics, so it should be documented clearly wherever used to avoid confusing other developers.

---

### Q29. PriorityQueue — internal structure and use cases
**Answer:** `PriorityQueue` is backed by a **binary heap** (array-based, not a linked structure) — by default a **min-heap** (smallest element has highest priority, retrieved first via `poll()`), orderable via natural ordering (`Comparable`) or a supplied `Comparator`. It is **not FIFO** despite implementing `Queue` — insertion order is irrelevant; only priority order matters for retrieval.

| Operation | Complexity |
|---|---|
| `offer()`/`add()` | O(log n) — sift-up |
| `poll()`/`remove()` (head) | O(log n) — sift-down |
| `peek()` | O(1) |
| Arbitrary element removal | O(n) — must search first |

**Real use case:** Task scheduling by priority (e.g., a job queue where higher-priority orders/tickets should be processed first), Dijkstra's/A* pathfinding algorithms, "top-K elements" problems (a very common coding-round pattern — maintain a fixed-size min-heap of size K while scanning a stream, giving O(n log k) instead of O(n log n) full sort).

```java
// Classic "top K largest elements" pattern using a min-heap of size K
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
for (int num : stream) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll();   // evict the smallest, keep only the K largest seen so far
}
```
**Follow-up:** "Is `PriorityQueue` thread-safe? What's the concurrent equivalent?" → No — use `PriorityBlockingQueue` for a thread-safe, blocking priority queue (already covered in the Multithreading prep, Q38).

---

### Q30. Deque — what makes it different from Queue and Stack, and why is ArrayDeque generally preferred over both LinkedList and Stack?
**Answer:** `Deque` (**D**ouble-**E**nded **Que**ue) supports insertion/removal at **both ends** in O(1) — it can function as **either a Queue (FIFO) or a Stack (LIFO)** through the same interface, using `addFirst()/addLast()/removeFirst()/removeLast()` or the more idiomatic `push()/pop()` (stack-style) and `offer()/poll()` (queue-style) method aliases.

**Why `ArrayDeque` is now preferred over legacy `Stack` and often over `LinkedList` for both stack and queue use cases:**
- `Stack` (legacy, extends `Vector`) is **synchronized** (unnecessary overhead for single-threaded use) and part of the awkward pre-Collections-Framework legacy hierarchy — the JDK docs themselves recommend `Deque` over `Stack` for new code.
- `ArrayDeque` is backed by a **resizable circular array** (not linked nodes) — better cache locality than `LinkedList`, no per-node pointer overhead, generally faster in benchmarks for both stack-style and queue-style access patterns.
- `ArrayDeque` **disallows `null` elements** (unlike `LinkedList`, which permits them) — a deliberate design choice to allow `null` to unambiguously signal "empty" from `peek()`/`poll()`.

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
stack.pop();     // 2 — LIFO, stack-style

Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.offer(2);
queue.poll();    // 1 — FIFO, queue-style
```
**Senior recommendation to state directly:** "For any new stack or FIFO-queue need in modern Java code, I default to `ArrayDeque` over both legacy `Stack` and `LinkedList` — better performance characteristics and it's the JDK-recommended choice."

---

### Q31. CopyOnWriteArrayList — internal mechanism deep dive (expanding on the Multithreading-prep mention)
**Answer:** On every mutating operation (`add`, `remove`, `set`), `CopyOnWriteArrayList` **copies the entire backing array** into a new array, applies the change to the new array, then atomically swaps the internal reference to point to the new array (under a lock, to serialize writers against each other) — reads (`get`, iteration) **never lock at all**, always operating against whichever array snapshot was current at the moment they started.

**Why iteration never throws `ConcurrentModificationException`:** The iterator holds a reference to the **array snapshot at iterator-creation time** — even if the underlying list is mutated concurrently (creating new array versions), the iterator keeps reading its own frozen snapshot, so it's fundamentally safe by construction, not by exception-throwing detection.

**Performance trade-off to state explicitly:** O(n) memory allocation + copy cost **per write**, regardless of how small the change — this makes it a poor choice for write-heavy or very large lists. Ideal for **small, read-dominant, infrequently-mutated** collections — the textbook example is a list of event listeners/observers, registered rarely, iterated frequently.

**Follow-up:** "If Thread A is iterating a `CopyOnWriteArrayList` and Thread B adds an element mid-iteration, does Thread A's iterator see the new element?" → **No** — Thread A's iterator is bound to the array snapshot that existed when the iterator was created; Thread B's addition creates a *new* array version that Thread A's already-in-progress iterator never sees. This is the precise meaning of "snapshot" semantics, distinct from `ConcurrentHashMap`'s weakly-consistent (may-or-may-not-see-updates) iteration.

---

### Q32. Arrays vs Collections utility classes — what do you actually use them for, and any common gotchas?
**Answer:** Both are static utility classes (private constructors, all-static methods) providing helper operations.

**`java.util.Arrays`:** `Arrays.sort()`, `Arrays.binarySearch()`, `Arrays.equals()` (deep/shallow), `Arrays.fill()`, `Arrays.asList()` (Q-trap from earlier — fixed-size view backed by the array), `Arrays.copyOf()`/`copyOfRange()`.

**`java.util.Collections`:** `Collections.sort()`, `Collections.unmodifiableXxx()` (view wrappers, Q-trap from earlier — reflects changes to the underlying original), `Collections.synchronizedXxx()`, `Collections.emptyList()`/`singletonList()` (immutable, memory-efficient constants), `Collections.max()/min()`, `Collections.frequency()`.

**Common gotcha worth raising proactively:** `Arrays.sort()` on an array of **primitives** uses a **dual-pivot Quicksort** (O(n log n) average, but **not stable**, and technically O(n²) worst case, though practically rare with the dual-pivot scheme); `Arrays.sort()` on an array of **objects** (and `Collections.sort()`, which always deals with objects) uses **TimSort** (a stable, adaptive merge-sort variant) — **stability matters** if you're sorting by one key and need ties broken by original relative order (e.g., sorting orders by amount while preserving original timestamp order for equal amounts) — a good detail to volunteer since it shows awareness most candidates miss.

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "What's the time complexity of `contains()` on an `ArrayList` vs a `HashSet`?" → `ArrayList`: O(n), linear scan using `equals()`. `HashSet`: O(1) average, via hashing.
- "Can you modify a List's elements (not structure) during a for-each loop without ConcurrentModificationException?" → Yes — `list.get(i).setSomeField(x)` (mutating an existing element's internal state) doesn't touch `modCount` at all, only structural changes (add/remove) do; only structural modification triggers fail-fast detection.
- "Why is `Arrays.asList()` a fixed-size list — what happens if you call `.add()` on it?" → It's backed directly by the underlying array; `add()`/`remove()` throw `UnsupportedOperationException` since the list can't structurally resize an array — only `set()` (element replacement) is permitted. A very common gotcha when developers assume it returns a full mutable `ArrayList`.
- "Difference between `Collections.unmodifiableList()` and `List.of()` (Java 9+)?" → `unmodifiableList()` wraps an existing list — a **view**, so changes to the underlying original list are still visible through it (a common surprise/bug); `List.of()` creates a genuinely **immutable** copy, throws `UnsupportedOperationException` on any mutation attempt, and additionally disallows `null` elements entirely (fails fast on construction).
- "What's the initial capacity and load factor default for HashSet vs HashMap?" → Same defaults (16, 0.75), since `HashSet` is backed by `HashMap` internally (Q6).
- "Is `Iterator.remove()` safe to call twice in a row without calling `next()` in between?" → No — throws `IllegalStateException`; you must call `next()` before each `remove()` call, since `remove()` operates on "the last element returned by `next()`."

---

*Study tip: HashMap internals (Q9-Q13) is asked in nearly every single senior Java interview and often gets 10-15 minutes of deep follow-up alone — be ready to draw the bucket array + collision chain + treeification diagram from memory. The equals()/hashCode() bug (Q23) and the fail-fast ConcurrentModificationException trap (Q21) are the two most common "write code that breaks, then explain why" live-coding exercises for this topic.*

# SENIOR EXPANSION — PRODUCTION & INTERNALS

> The original module is preserved above. This section adds senior-level reasoning, production scenarios, trade-offs, and interview follow-ups.

## Q33. How do you choose a Collection in production?
**Answer:** Start with access patterns and invariants, not with habit. Ask: Do I need ordering? uniqueness? sorting? random access? concurrent mutation? bounded memory? frequent writes or reads?

| Requirement | Typical choice | Key reason |
|---|---|---|
| Key lookup | `HashMap` | Average O(1) |
| Insertion/access order | `LinkedHashMap` | Predictable order |
| Sorted keys | `TreeMap` | O(log n) ordered operations |
| Unique values | `HashSet` | Hash-based membership |
| Queue/deque | `ArrayDeque` | Efficient ends |
| Concurrent map | `ConcurrentHashMap` | Concurrent access |
| Read-heavy list | `CopyOnWriteArrayList` | Snapshot-style iteration |

Senior answer: also evaluate memory, cardinality, mutation frequency, contention, and API semantics.

## Q34. Explain the HashMap `put()` path.
**Answer:** HashMap derives/spreads the key hash, calculates a bucket, checks the bucket contents, compares hashes and keys, inserts or replaces the value, and may resize when the threshold is crossed. In Java 8+, heavily-colliding bins can become tree bins. The exact implementation details are JDK-specific, so explain the algorithm rather than claiming a particular internal layout for every future JDK.

## Q35. Why are power-of-two capacities important in HashMap?
**Answer:** HashMap's bucket-index calculation and resize strategy are optimized around power-of-two table sizes. This permits efficient bit-based indexing and predictable redistribution during resize.

## Q36. Why can a mutable HashMap key make an entry "disappear"?
**Answer:** If fields participating in `equals()`/`hashCode()` change after insertion, the key may hash to a different bucket. The entry can remain physically present but normal lookup may no longer find it. Prefer immutable keys.

```java
Map<UserKey, String> map = new HashMap<>();
map.put(key, "V");
// Do not mutate fields used by equals/hashCode while key is in the map.
```

## Q37. Why is `containsKey()` followed by `put()` unsafe with ConcurrentHashMap?
**Answer:** It is a check-then-act race.

```java
if (!map.containsKey(id)) {
    map.put(id, user);
}
```

Two threads can both observe absence. Prefer an atomic map operation:

```java
map.putIfAbsent(id, user);
```

or:

```java
map.computeIfAbsent(id, this::loadUser);
```

Important: an atomic map operation does not automatically make an entire business workflow transactional.

## Q38. What does `computeIfAbsent()` solve?
**Answer:** It expresses conditional initialization atomically at the map level.

```java
cache.computeIfAbsent(id, this::loadUser);
```

It is useful for caches and memoization, but consider loader cost, failures, side effects, recursion, and whether a local map is appropriate for a distributed service.

## Q39. HashMap vs ConcurrentHashMap: what is the senior-level distinction?
**Answer:** `HashMap` provides no concurrent mutation safety. `ConcurrentHashMap` is designed for concurrent access and provides useful atomic map operations. However, it does not make arbitrary multi-step application logic atomic.

```text
map.get(A)
map.get(B)
calculate()
map.put(A)
map.put(B)
```

If correctness depends on the whole sequence, the collection alone is not enough.

## Q40. Fail-fast vs weakly consistent iteration?
**Answer:** A fail-fast iterator attempts to detect structural modification and may throw `ConcurrentModificationException`; this is a debugging aid, not a concurrency guarantee. Concurrent collections commonly provide weakly consistent iterators that tolerate concurrent changes and may reflect some changes. Weakly consistent is not the same as a snapshot.

## Q41. Why can synchronized collections still require synchronization during iteration?
**Answer:** `Collections.synchronizedList()` synchronizes individual methods, but iteration is a multi-step operation. Follow the wrapper's documented synchronization protocol and hold the collection lock while iterating when required.

## Q42. CopyOnWriteArrayList — when is it an excellent choice?
**Answer:** When reads/iteration vastly outnumber writes, the list is relatively small, and readers benefit from stable snapshots. Typical examples include listener lists and configuration snapshots. Every write copies the backing array, so frequent writes can be very expensive.

## Q43. ArrayList vs LinkedList — which is your default?
**Answer:** Usually `ArrayList`. It offers efficient indexed access, good cache locality, and lower per-element overhead. `LinkedList` can provide efficient insertion/removal when you already have the node position, but finding that position is often O(n). For stack/deque behavior, prefer `ArrayDeque`.

## Q44. Why is ArrayDeque generally preferred over Stack?
**Answer:** `Stack` is a legacy `Vector`-based class. `ArrayDeque` is a modern deque and supports both stack and queue semantics.

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(10);
stack.push(20);
int top = stack.pop();
```

`ArrayDeque` is not thread-safe; choose a concurrent structure when required.

## Q45. TreeSet/TreeMap and Comparator consistency
**Answer:** Sorted collections use their ordering to decide placement and, for sets/maps, effective key uniqueness. If `compare(a,b) == 0` while `a.equals(b)` is false, the sorted collection can treat distinct objects as the same key/element. Prefer orderings consistent with equality when the domain requires normal set/map semantics.

## Q46. What is WeakHashMap NOT suitable for?
**Answer:** It is not a deterministic cache. Garbage collection determines when weakly reachable keys disappear. Use it for metadata associated with keys where the metadata should not keep those keys alive; use a real cache when you need size/TTL/eviction guarantees.

## Q47. How would you implement a simple LRU cache with JDK collections?
**Answer:**
```java
Map<String, User> cache =
    new LinkedHashMap<>(16, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(
                Map.Entry<String, User> eldest) {
            return size() > 1000;
        }
    };
```
This demonstrates the idea, but production code must address concurrency, expiration, memory limits, observability, and whether a dedicated cache is more appropriate.

## Q48. How do you make a collection immutable?
**Answer:** `List.of(...)` creates an immutable list. `Collections.unmodifiableList(existing)` creates a read-only view over an existing list. The view can reflect changes made through another reference to `existing`.

## Q49. What is the Arrays.asList() trap?
**Answer:**
```java
List<String> list = Arrays.asList("A", "B");
list.set(0, "X"); // allowed
list.add("C");   // UnsupportedOperationException
```
It is fixed-size and backed by the array. For a resizable list:

```java
new ArrayList<>(Arrays.asList("A", "B"));
```

## Q50. How do you troubleshoot a production memory issue caused by a collection?
**Answer:** Look for unbounded maps/caches, static collections, queues growing faster than consumers, duplicate copies during transformations, and values retaining large object graphs. Use heap dumps, retaining paths, allocation profiling, collection-size metrics, and traffic/load correlation.

## Q51. A queue grows continuously. Is changing ArrayList to LinkedList the solution?
**Answer:** No. The problem is usually throughput imbalance or missing backpressure. Investigate producer/consumer rates, queue bounds, rejection/throttling policy, downstream latency, and memory pressure. An unbounded queue can convert a throughput problem into an OOM.

## Q52. What is the senior rule for collection optimization?
**Answer:** Measure first. Choose the structure based on workload, memory, ordering, concurrency and complexity requirements. Do not optimize based only on textbook Big-O; allocation rate, cache locality, contention, and object overhead matter in real JVM workloads.

### Senior scenario: concurrent cache
**Question:** 100 threads request the same missing key. What would you use?
**Answer:** `ConcurrentHashMap.computeIfAbsent()` can be appropriate, but validate loader cost and failure semantics. If loading is remote/expensive, also consider stampede control, timeouts, TTL, negative caching, and a proper cache implementation.

### Senior scenario: ordered API response
**Question:** The API contract requires insertion order. Can you use HashMap?
**Answer:** Do not rely on incidental iteration behavior. Use an explicitly ordered structure such as `LinkedHashMap`, or sort explicitly if the contract is sorted order.

### Senior rapid revision
- Mutable keys + HashMap = danger.
- `containsKey()` + `put()` = check-then-act race.
- ConcurrentHashMap does not make arbitrary workflows atomic.
- Fail-fast is not thread safety.
- Weakly consistent is not snapshot semantics.
- CopyOnWriteArrayList = read-heavy/write-light.
- ArrayDeque = strong default for deque/stack behavior.
- TreeSet uniqueness follows ordering semantics.
- WeakHashMap is not a TTL cache.
- Bound caches and queues in production.
