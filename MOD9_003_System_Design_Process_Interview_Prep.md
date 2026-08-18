# System Design Process — Interview Prep
---

## SECTION 1: FUNCTIONAL vs NON-FUNCTIONAL REQUIREMENTS

### Q1. In a system design interview, why does spending the first several minutes on requirements gathering matter more than jumping straight to a diagram?
**Answer:** **Functional requirements** define *what* the system does — the features, user actions, and business rules (e.g., "users can post a tweet," "a tweet can be liked and retweeted"). **Non-functional requirements (NFRs)** define *how well* it needs to do it — scale, latency, availability, consistency needs — and NFRs are almost always what actually determine the architecture. Two systems with identical functional requirements can need completely different designs depending on their NFRs: a "post and read messages" feature for a 10-person team Slack clone versus for Twitter-scale are functionally similar but architecturally worlds apart.

**A practical requirements-gathering checklist:**
```
Functional:
- Core user actions (what can a user do?)
- What's explicitly OUT of scope (just as important — bounds the problem)

Non-functional:
- Scale: how many users, how many requests/sec, read:write ratio?
- Latency: what's an acceptable response time?
- Consistency: strong or eventual consistency acceptable?
- Availability: what's the tolerance for downtime?
```

**Senior-level answer:**
> "Jumping straight to a diagram without clarifying requirements is the single most common mistake I see in system design interviews, because it signals that the candidate is pattern-matching to a memorized architecture rather than actually designing for the problem in front of them. I always spend the first several minutes asking pointed questions — read-heavy or write-heavy, how many users, what's the acceptable staleness — because the answers directly determine whether I reach for a single database with a read replica or a fully sharded, eventually-consistent, multi-region system. The requirements conversation isn't a formality before the 'real' design work — it *is* the design work, since it's what the rest of the design is a response to."

**Trap:** Don't treat this as a one-time upfront step — real requirements gathering continues throughout the interview. If a proposed design reveals a new question ("wait, do likes need to be counted in real-time, or is a few-second delay okay?"), asking it mid-design is expected and a good sign, not a failure to have asked everything upfront.

---

### Q2. How do you prioritize which non-functional requirements to actually design around, when you can't optimize for everything simultaneously?
**Answer:** Most NFRs trade off against each other directly (strong consistency vs. low latency, cost vs. redundancy), so the practical approach is to **identify the 1–2 NFRs that are most critical for this specific system's purpose**, and explicitly treat the rest as secondary — stated as a conscious trade-off, not left unaddressed.

**Example — designing a URL shortener vs. a payment system:**
| | URL Shortener | Payment System |
|---|---|---|
| **Primary NFR** | Low latency on redirect (most critical path), high read throughput | Strong consistency (never double-charge), durability |
| **Secondary/relaxed** | Eventual consistency is fine (a new short URL being slightly delayed to propagate is harmless) | Some added latency for consistency guarantees is an acceptable cost |

**Senior-level answer:**
> "I explicitly rank NFRs out loud rather than trying to design a system that's simultaneously maximally consistent, available, low-latency, and cheap — because that system doesn't exist; CAP theorem alone rules out having everything at once during a partition. For a URL shortener, I'd say directly: 'redirects need to be fast and this system needs to handle huge read volume; a few seconds of delay before a brand-new short link is globally available is a totally acceptable trade for that.' Stating the prioritization explicitly, and explaining *why* given the system's actual purpose, is what turns a generic architecture into one that's actually justified for the problem at hand."

---

## SECTION 2: CAPACITY ESTIMATION

### Q3. Walk through how you'd do back-of-the-envelope capacity estimation for a system, and why interviewers care about this step at all.
**Answer:** Capacity estimation translates requirements into **concrete numbers** — requests per second, storage growth, bandwidth — that directly justify specific architecture decisions (how many servers, whether sharding is needed yet, what caching hit rate is required). Interviewers care because it's what separates "we should use a cache because caches are good" from "we need to serve roughly 50,000 reads/sec and our database alone caps out around 5,000, so we need something absorbing the other 45,000."

**Example — estimating for a social media feed feature:**
```
Given: 100M daily active users, each checks their feed ~10 times/day average

Total daily feed reads = 100M x 10 = 1 billion reads/day
Reads per second (average) = 1,000,000,000 / 86,400 ≈ ~11,600 reads/sec
Peak traffic (assume 3x average for peak hours) ≈ ~35,000 reads/sec

Storage: assume 500 bytes/post, 100M users post ~1x/day average
Daily storage growth = 100M x 500 bytes ≈ 50 GB/day ≈ ~18 TB/year
```

**Senior-level answer:**
> "I keep these estimates deliberately rough — round numbers, powers of ten — because the goal isn't precision, it's directional confidence about *scale of magnitude*, which is what actually drives architecture decisions. Knowing it's roughly 35,000 reads/sec at peak, not exactly 34,872, is what tells me 'a single database read replica setup won't cut it, we need aggressive caching and probably read replicas at minimum, possibly sharding' — versus a system doing 50 reads/sec, where a single well-indexed database is completely sufficient and any more elaborate design would be over-engineering. The number itself is less important than what decision it justifies."

---

### Q4. What's a common mistake candidates make during capacity estimation, and how do you avoid it?
**Answer:** The most common mistake is **estimating average load and designing only for that** — real systems need to handle **peak load**, which is often several times the average, and ignoring this leads to a design that looks fine on paper but falls over during real traffic spikes (a product launch, a viral moment, a predictable daily peak like lunch-hour for a food delivery app).

**A second common mistake:** getting lost in excessive precision — spending five minutes calculating an exact number instead of using a reasonable round estimate and moving on to what that number implies for the design.

**Senior-level answer:**
> "I always explicitly call out the average-versus-peak distinction, because it's exactly the kind of detail that separates a design that survives a real launch from one that doesn't. My habit is to estimate average load, then apply a peak multiplier — 2x to 5x depending on how spiky the domain is (a flash sale or a live sports event is far spikier than a general social app) — and design capacity around the peak number, not the average. I also intentionally keep the arithmetic fast and rough; spending real interview time on precise multiplication is time not spent on the architecture decisions the number is supposed to inform."

---

## SECTION 3: API DESIGN

### Q5. What should you cover when designing the API surface for a system in an interview, beyond just listing endpoints?
**Answer:** A strong API design pass covers:
- **Core resources and operations** — the primary nouns (e.g., `/posts`, `/users`) and the actions on them (REST verbs, or RPC-style methods).
- **Request/response shape** — what fields are needed, what's returned, pagination for list endpoints.
- **Explicit protocol choice and why** — REST for a public, resource-oriented API; gRPC for high-performance internal service-to-service calls; GraphQL if clients need flexible field selection.
- **Versioning strategy** — how the API evolves without breaking existing clients.
- **Rate limiting/auth touchpoints** — at least acknowledging where these apply, even briefly.

```
POST   /v1/posts                  { userId, content } -> 201 { postId, createdAt }
GET    /v1/posts/{postId}         -> 200 { postId, userId, content, likeCount, createdAt }
GET    /v1/users/{userId}/feed?cursor={cursor}&limit=20  -> 200 { posts: [...], nextCursor }
POST   /v1/posts/{postId}/likes   -> 204
```

**Senior-level answer:**
> "I treat the API design step as a forcing function for clarifying the data model before diving into database schema — writing out `POST /posts { userId, content }` immediately raises the question of what a Post actually needs to store, and writing the feed endpoint's pagination raises the fan-out question (Q9 in messaging) of how a feed actually gets assembled at read time. I also always justify protocol choice rather than defaulting to REST reflexively — for a real-time chat feature, I'd explicitly bring up WebSockets or gRPC streaming instead of long-polling a REST endpoint, because the requirement (real-time, bidirectional) doesn't fit REST's request-response model well."

**Trap:** Don't spend excessive interview time bikeshedding exact field names or HTTP status codes — the goal is demonstrating you can define a clean, sensible contract quickly, not producing a complete OpenAPI spec. Move on once the core resources and operations are clear.

---

### Q6. How do you handle API pagination for a large, frequently-changing dataset (e.g., a social feed), and why does offset-based pagination break down at scale?
**Answer:** **Offset-based pagination** (`?page=5&size=20`, translating to `OFFSET 100 LIMIT 20`) has two real problems at scale: (1) the database still has to **scan and discard** all the skipped rows, so performance degrades linearly as the offset grows — page 10,000 is much slower than page 1; and (2) if rows are inserted/deleted between page loads (very likely on a live feed), users see **duplicated or skipped items** as the underlying offsets shift under them.

**Cursor-based pagination** solves both: instead of a page number, the client passes an opaque cursor (typically an encoded ID/timestamp of the last item seen), and the query fetches items **after that specific point** — no scanning/discarding, and stable under concurrent inserts.

```java
// Offset — degrades with page depth, unstable under concurrent writes
SELECT * FROM posts ORDER BY created_at DESC OFFSET 10000 LIMIT 20;

// Cursor — consistent performance, stable under inserts
SELECT * FROM posts WHERE created_at < :cursorTimestamp ORDER BY created_at DESC LIMIT 20;
```

**Senior-level answer:**
> "Cursor-based pagination is my default recommendation the moment a dataset is large or frequently changing, and I make sure to explain *why*, not just state the pattern — the performance argument (no scan-and-discard) matters, but the correctness argument matters just as much: on a live feed, offset pagination genuinely shows users duplicate or missing posts as new items get inserted ahead of their current page, which is a real, noticeable bug, not just a performance nitpick. Offset pagination is still fine for smaller, mostly-static datasets — an admin dashboard listing a few hundred infrequently-changing records — where the trade-offs simply don't apply at that scale."

---

## SECTION 4: DATA MODEL & DATABASE SELECTION

### Q7. How do you decide between a relational (SQL) and a NoSQL database for a given system component, beyond "SQL for structured data, NoSQL for unstructured"?
**Answer:** That oversimplified rule of thumb misses the actual decision drivers:

| Consideration | Favors SQL | Favors NoSQL |
|---|---|---|
| **Relationships/joins** | Data is naturally relational, queries need joins across entities | Data is naturally denormalized/document-shaped, or queries are always by a single known key |
| **Consistency needs** | Strong consistency, ACID transactions required (financial data, inventory counts) | Eventual consistency acceptable |
| **Schema stability** | Schema is well-understood and relatively stable | Schema varies significantly per record, or evolves frequently |
| **Write/read pattern at scale** | Moderate scale, complex queries | Very high write throughput, simple key-based access patterns, horizontal scaling is a primary concern |
| **Example use case** | Order/payment records, user account data | Session storage, activity feeds, IoT sensor data, product catalogs with wildly varying attributes |

**Senior-level answer:**
> "I explicitly reject 'structured vs. unstructured' as the deciding factor, because plenty of NoSQL use cases involve perfectly structured data — the real drivers are consistency requirements, access patterns, and scale characteristics. I also don't treat this as a single system-wide decision — polyglot persistence, using different databases for different components of the same system based on each component's actual needs, is the norm in real architectures. A typical e-commerce system might use a relational database for orders and payments (need ACID transactions), a document store for the product catalog (highly variable attributes per product category), and Redis for session/cart data (fast, ephemeral, key-based access)."

**Trap:** Don't present NoSQL databases as inherently "more scalable" than SQL as a blanket statement — modern relational databases scale substantially with proper indexing, read replicas, and sharding; the real distinction is about the *shape* of the consistency/query trade-offs each is optimized for, not a fixed scalability ceiling.

---

### Q8. How do you approach designing the data model for a new feature in an interview — what's the actual process, step by step?
**Answer:**
1. **Identify the core entities** from the functional requirements (e.g., for a feed: `User`, `Post`, `Follow`, `Like`).
2. **Define relationships between them** — one-to-many, many-to-many (e.g., `Follow` is a many-to-many relationship between users).
3. **Identify the dominant access patterns** from the API design (Q5) — what queries will actually be run most often? This often matters more than normalized "correctness" for NoSQL modeling specifically.
4. **Decide normalization level based on those access patterns** — a relational schema normalized for data integrity, or a denormalized NoSQL document shaped around how it'll actually be read (e.g., embedding a user's display name directly in a Post document to avoid a join/lookup on every feed read, accepting the sync cost of Q7 in the Distributed Data Management module).

```sql
-- Relational, normalized
users(id, username, email)
posts(id, user_id FK, content, created_at)
follows(follower_id FK, followee_id FK)
likes(user_id FK, post_id FK)
```
```json
// NoSQL, denormalized for the feed's dominant read pattern
{
  "postId": "123",
  "authorId": "u1",
  "authorDisplayName": "Alice",  // denormalized — avoids a join on every feed read
  "content": "...",
  "likeCount": 42
}
```

**Senior-level answer:**
> "The step people skip is tying the data model explicitly back to access patterns established during API design — I don't model data in a vacuum. If the feed endpoint needs to display the author's name on every post without an extra lookup, that's a concrete argument for denormalizing it directly into the post record, accepting that an eventual username change needs to be synced (or accepted as historically accurate) rather than always freshly joined. I make this trade-off explicit rather than silently choosing one — 'we're denormalizing here because the read pattern is extremely high-volume and username changes are rare' is the kind of reasoning that shows real design judgment."

---

## SECTION 5: CACHING STRATEGY

### Q9. When designing a new system, how do you decide what to cache and where in the request path to put the cache — walk through the reasoning, not just naming patterns.
**Answer:** The decision process:
1. **Identify hot, expensive, and reused data** from the access patterns already established — data that's read far more often than it's written, and expensive to compute/fetch (a complex join, an aggregate, a call to a slow downstream service).
2. **Decide the cache's position** — client-side (browser/CDN, for static or rarely-changing public content), a shared distributed cache (Redis, for data needing consistency across instances), or a local in-process cache (for extremely hot, tolerant-of-slight-staleness data, see the Distributed Caching module's local-vs-distributed comparison).
3. **Decide the caching pattern** (cache-aside is the sensible default for most read paths — see the Distributed Caching module for the full pattern comparison) and a TTL appropriate to how often that specific data actually changes.

**Senior-level answer:**
> "I walk through this reasoning out loud rather than just declaring 'we'll add a Redis cache here,' because the *why* is what interviewers are actually evaluating. For a feed system, I'd identify that the read:write ratio is heavily skewed toward reads, and that assembling a feed (potentially fanning out across many followed users) is expensive enough to be worth caching the assembled result rather than recomputing it on every request — that's a concrete, access-pattern-driven justification, not caching applied reflexively because 'caching is generally good.' I'd also explicitly flag what happens on a cache miss and what the fallback behavior is under cache unavailability, since a caching strategy that doesn't account for cache failure isn't actually complete." (Full depth on cache invalidation, stampede prevention, and distributed locking is covered in the Distributed Caching module.)

---

## SECTION 6: MESSAGING STRATEGY

### Q10. When does a system design call for asynchronous messaging (a queue/broker) instead of a direct synchronous API call, and how do you justify that choice in an interview?
**Answer:** Reach for asynchronous messaging when:
- **The producer doesn't need an immediate response** — e.g., "send a welcome email after signup" doesn't need to block the signup response waiting for the email to actually send.
- **Decoupling matters** — the producer shouldn't need to know or care about every consumer (see the Distributed Messaging module for the full coupling argument).
- **Traffic needs to be smoothed/buffered** — a sudden burst of writes (e.g., a flash sale) can be queued and processed at a sustainable rate by downstream workers, rather than overwhelming a downstream system synchronously.
- **The workflow spans multiple services with eventual-consistency tolerance** — an order-placement flow triggering inventory updates, notifications, and analytics, none of which need to complete before the user gets an "order placed" response.

**Example — designing the "post creation" flow for a social feed:**
```
POST /posts -> writes the post to the DB -> publishes "PostCreated" event -> returns 201 immediately
                                                    |
                    [Feed fan-out worker]  [Notification worker]  [Analytics worker]
                    (all process asynchronously, independently, don't block the response)
```

**Senior-level answer:**
> "I explicitly justify the sync-vs-async decision per interaction rather than picking one style for the whole system — the write to the post itself needs to be synchronous and confirmed before returning 201, because the user needs to know their post was actually saved. But everything that happens *as a result* of that post existing — fanning it out to followers' feeds, sending notifications, updating analytics — doesn't need to complete before responding, and forcing it to be synchronous would mean the user's request latency is bounded by the slowest of those downstream concerns, which is an unnecessary and fragile coupling." (Delivery semantics, exactly-once handling, and broker selection are covered in depth in the Distributed Messaging module.)

---

## SECTION 7: SCALABILITY & AVAILABILITY

### Q11. During a system design interview, at what point should you introduce scaling concerns like sharding, replication, or caching — and what's the risk of introducing them too early?
**Answer:** Scaling techniques should be introduced **after** establishing a working baseline design and **in direct response to the capacity numbers** from estimation (Q3–Q4) — not reflexively, as if every design needs sharding and multi-region replication regardless of actual scale. Introducing them too early is a real, commonly-cited red flag: it signals reciting memorized "scalable architecture" components rather than reasoning from the specific problem's numbers.

```
Bad pattern: "We'll shard the database, add a message queue, and deploy across 3 regions"
             — stated before any discussion of actual expected scale

Good pattern: "Based on our ~35,000 reads/sec peak estimate, a single primary database
             would be a bottleneck, so I'd add read replicas here — and since our
             estimate shows [X], sharding isn't yet justified at this scale, though I'd
             flag it as the next lever if traffic grows another order of magnitude."
```

**Senior-level answer:**
> "I treat the order of introduction as a genuine signal I'm being evaluated on — starting with a single application server and a single database, explaining what would actually break first as load increases (usually database read capacity), and introducing exactly the scaling technique that addresses *that specific bottleneck* is a much stronger answer than front-loading every scaling pattern I know. It also gives me room to demonstrate judgment about what's NOT needed yet — explicitly saying 'sharding would be premature at this scale, read replicas plus caching gets us well past our estimated peak' shows I understand the cost side of these techniques, not just their benefit." (Full depth on horizontal/vertical scaling, replication, sharding, and HA/DR is covered in the Scalability & Availability module.)

---

## SECTION 8: SECURITY

### Q12. How much security discussion is actually expected in a system design interview, and what should you make sure to cover even briefly?
**Answer:** Security usually isn't the deep focus of a system design interview, but skipping it entirely is a real gap interviewers notice. A solid baseline to touch on:
- **AuthN/AuthZ** — how users authenticate (session, JWT, OAuth), and how authorization is enforced (can this user access this specific resource — the IDOR concern from the security modules).
- **Data protection** — encryption in transit (TLS) as a given; encryption at rest for genuinely sensitive fields.
- **Rate limiting/abuse prevention** — especially for any public-facing write endpoint (signup, posting, commenting) that could be abused/spammed.
- **Input validation** at the API boundary — never trusting client-supplied data.

**Senior-level answer:**
> "I keep this section proportionate — a few sentences, not a deep dive, unless the interviewer signals they want to go deeper — but I make sure to touch on authentication/authorization explicitly, because skipping it entirely reads as an oversight rather than a deliberate scoping choice. For most designs I'll say something like: 'Users authenticate via JWT issued at login; every write endpoint validates the token and checks resource ownership before allowing the action; public endpoints are rate-limited per-IP and per-account to prevent abuse.' That's enough to show the concern was considered, without derailing the time budget from the core architecture the interview is actually testing." (Full depth on authentication, authorization, and application security is covered in Module 8.)

---

## SECTION 9: OBSERVABILITY

### Q13. Why does mentioning observability matter in a system design interview, even though it's rarely the main focus, and what's the minimum worth covering?
**Answer:** A design with no mention of observability implicitly assumes the system will run perfectly forever, which is never realistic — interviewers are checking whether the candidate thinks about **operating** the system, not just building it. The minimum worth covering:
- **Metrics** — key operational numbers to track (request rate, error rate, latency percentiles — p50/p95/p99, not just averages, since averages hide tail latency problems) and what would trigger an alert.
- **Logging** — structured, centralized logs with correlation/trace IDs so a request can be followed across services.
- **Health checks** — how the system detects and reacts to an unhealthy component (connects directly to the failure handling section, Q14).

**Senior-level answer:**
> "I usually mention this in one or two sentences near the end unless it becomes a deeper focus: 'I'd track p95/p99 latency and error rate per endpoint, with alerting on threshold breaches, and every request would carry a correlation ID through the system for tracing issues across services.' The p95/p99-over-average point specifically is worth stating explicitly if there's any room — a system can have a perfectly fine average latency while a meaningful fraction of users have a genuinely bad experience, and average-only monitoring would never surface that." (Full depth on observability, metrics, tracing, and monitoring/alerting is covered in Module 6.)

---

## SECTION 10: FAILURE HANDLING

### Q14. What failure scenarios should you proactively raise in a system design interview, without being prompted?
**Answer:** A strong candidate proactively identifies at least a few realistic failure points in their own design and states how the system handles them, rather than waiting for the interviewer to ask "what if X goes down":
- **A downstream dependency becomes slow/unavailable** — timeouts, circuit breakers, and a defined fallback (serve stale cached data, degrade a non-critical feature) rather than letting the failure cascade.
- **A database/cache node fails** — what's the failover behavior, and is there a risk of data loss depending on replication strategy (Q6 in Scalability & Availability).
- **A message gets processed twice, or is lost** — is the consumer idempotent, does the producer guarantee at-least-once delivery, what's acceptable (see the Distributed Data Management and Distributed Messaging modules).
- **A single component becomes a bottleneck/single point of failure** — explicitly identify it in the design and state the mitigation, or explicitly accept it as a known trade-off for a system at this scale.

**Senior-level answer:**
> "This is one of the highest-signal moments in the whole interview — I make a point of pausing after sketching the core design and saying something like, 'let me walk through what happens if the cache goes down, if the message broker backs up, or if the database primary fails,' rather than waiting to be asked. It demonstrates the design wasn't just built for the happy path, and it's usually where the most interesting conversation in the interview actually happens, because it reveals whether I understand the *consequences* of the architectural choices I made earlier — a queue-based design that gets confidently drawn but has no answer for 'what if a message is processed twice' hasn't actually been thought through completely." (Full depth on circuit breakers, retries, bulkheads, and graceful degradation is covered in the Resilience & Fault Tolerance module.)

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "How much time should each phase of a system design interview take?" → Roughly: requirements/estimation (5–10 min), high-level design/API/data model (15–20 min), deep dive into 1–2 specific components the interviewer steers toward (15–20 min), failure handling/wrap-up (5–10 min) — flexible, but having a rough internal budget prevents spending 30 minutes on requirements and rushing the actual design.
- "What if the interviewer doesn't specify scale — should you assume it or ask?" → Always ask; if pressed to just proceed, state a reasonable explicit assumption out loud ("I'll assume ~10M daily active users unless told otherwise") so the rest of the design is anchored to a stated number, not left ambiguous.
- "Is it okay to say 'I'd use X' without deeply justifying every single technology choice?" → For less central choices, a brief justification is fine ("Postgres, since we need relational integrity for orders"); for the choices most load-bearing to the design's core trade-offs, deeper justification is expected — calibrate depth to how consequential the decision is.
- "Should you always draw a diagram?" → Yes, if the interview format allows it — a visual reduces ambiguity and gives the interviewer a shared reference point to ask follow-up questions against; a purely verbal design is harder for both sides to track as it grows in complexity.
- "What if you don't know a specific technology mentioned in the prompt?" → Say so directly and reason from first principles about what properties are needed, rather than guessing confidently about specifics — interviewers are generally more impressed by honest reasoning than by fabricated familiarity.
- "Is it a red flag to change your design mid-interview after realizing a flaw?" → No — the opposite; catching your own design flaw and explaining the fix demonstrates real engineering judgment, and is generally viewed far more favorably than defending a flawed initial choice out of stubbornness.

---

*Study tip: Treating requirements gathering and capacity estimation as the steps that justify every later decision rather than a formality to rush through (Q1–Q4), proactively raising failure scenarios before being asked rather than only in the happy path (Q14), and calibrating how much depth to give security/observability/scalability relative to the interview's actual focus (Q11–Q13) are the three areas where interviewers most reliably separate candidates who've actually run the full system design process from candidates who've memorized individual component patterns without the connective reasoning between them.*
