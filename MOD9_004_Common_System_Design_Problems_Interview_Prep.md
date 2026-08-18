# Common System Design Problems — Interview Prep
---

## SECTION 1: URL SHORTENER

### Q1. Design a URL shortener (like bit.ly). Walk through the core design and the key decision that shapes everything else.
**Answer:** The core flow is simple — `POST /shorten { longUrl }` returns a short code; `GET /{shortCode}` redirects to the original URL — but the design is entirely shaped by one decision: **how the short code is generated**, and the system is overwhelmingly **read-heavy** (redirects vastly outnumber creations), which drives every scaling choice.

**Short code generation approaches:**
| Approach | How it works | Trade-off |
|---|---|---|
| **Base62 encode an auto-incrementing ID** | DB assigns a sequential ID, encode it (0-9a-zA-Z) into a short string | Simple, guaranteed unique, but a single auto-increment counter can become a write bottleneck/single point of contention at very high scale |
| **Pre-generated key pool** | A background service generates and stores batches of unique random codes ahead of time; each app instance claims a batch to hand out | Avoids the single-counter bottleneck; codes are unpredictable (can't be enumerated sequentially by scraping) |
| **Hash the long URL (MD5/SHA) and truncate** | Deterministic, no coordination needed | Truncated hashes collide at scale, requiring collision-handling logic — extra complexity for a marginal benefit over the pool approach |

```
POST /shorten {longUrl: "https://example.com/very/long/path"}
  -> assign ID from pre-claimed pool, e.g. 125 -> base62 encode -> "cb"
  -> store {shortCode: "cb", longUrl: "...", createdAt}
  -> return "https://short.ly/cb"

GET /cb -> lookup longUrl by shortCode -> 301/302 redirect
```

**Senior-level answer:**
> "The read:write ratio is the single number that should drive the whole design conversation — for a typical URL shortener it's easily 100:1 or higher, so I spend most of the design effort on making the redirect path fast and cheap, not the creation path. That means aggressive caching (Redis in front of the DB, since redirects are simple key lookups — ideal cache-aside candidates) and choosing a permanent, non-expiring **301 redirect only if we don't need click analytics**; if click tracking matters, a **302** is required, since a 301 gets cached by the browser itself, and subsequent clicks never even reach our server to be counted. That's a small detail that's easy to miss but has a real product implication."

**Trap:** Don't default to MD5-hash-and-truncate without acknowledging the collision problem — interviewers commonly probe "what happens when two different URLs hash to the same truncated code," and having no answer is a red flag. The pre-generated key pool approach sidesteps this cleanly and is generally the stronger answer.

---

## SECTION 2: PAYMENT SYSTEM

### Q2. Design a payment processing system. What's the single most important property this system must guarantee, and how do you actually achieve it?
**Answer:** The most important property is **exactly-once effect, never double-charging or double-crediting**, even under retries, network failures, or duplicate requests — payment systems are the canonical case where correctness matters far more than raw throughput or latency. This is achieved primarily through **idempotency keys**: the client generates a unique key per logical payment attempt and includes it on every request/retry; the server guarantees that processing the same idempotency key twice produces the **same result without charging twice**.

```java
@PostMapping("/payments")
public PaymentResult processPayment(@RequestHeader("Idempotency-Key") String idempotencyKey,
                                      @RequestBody PaymentRequest request) {
    PaymentResult existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing != null) return existing;   // already processed — return the same result, don't re-charge

    PaymentResult result = paymentGateway.charge(request);
    paymentRepository.save(idempotencyKey, result);   // atomic with the charge, via transactional outbox
    return result;
}
```

**Senior-level answer:**
> "I always explicitly walk through the failure scenario that makes idempotency non-negotiable: a client calls 'charge $50,' the charge succeeds on our end, but the response is lost to a network blip before the client receives it — the client, seeing a timeout, retries. Without an idempotency key, that's a double charge; with one, the retry is recognized as the same logical attempt and returns the already-computed result. I also make sure to bring up the **Transactional Outbox pattern** (Distributed Data Management module) for publishing a 'PaymentCompleted' event atomically with the DB write — a payment succeeding in the database but the confirmation event never being published due to a dual-write failure is exactly the kind of subtle bug that's catastrophic specifically in a payments context."

**Trap:** Don't propose "just make the payment API idempotent by checking if this exact request body was seen before" — that's insufficient, since a legitimately different request (a retry with a slightly different timestamp field, for example) wouldn't match. The idempotency key must be a **client-generated, stable identifier for the logical attempt**, independent of incidental request field variation.

---

### Q3. How would you design the payment system to handle the reality that a downstream payment gateway (Stripe, a bank) is itself unreliable or slow?
**Answer:** Treat the payment gateway exactly like any unreliable external dependency — **timeouts, circuit breakers, and a defined strategy for the ambiguous "did it actually succeed" case**, which is the specific hard problem payments add on top of standard resilience patterns: a timeout talking to the gateway doesn't tell you whether the charge went through or not on their end.

```
1. Call gateway with a timeout + the idempotency key passed through to the gateway itself
   (most payment gateways, e.g., Stripe, support idempotency keys on their own API)
2. On timeout/ambiguous failure: do NOT assume failure and retry blindly —
   query the gateway's status endpoint for that idempotency key to find the true outcome
3. Record the payment as PENDING until a definitive outcome is confirmed, not FAILED or SUCCESS
4. A reconciliation job periodically resolves any PENDING payments left ambiguous for too long
```

**Senior-level answer:**
> "The detail I make sure to raise explicitly: a timeout calling an external payment gateway is fundamentally ambiguous — it does not mean the charge failed, it means **we don't know**. Treating a timeout as a failure and retrying blindly risks a double-charge on the gateway's side even with our own idempotency key, if the gateway itself doesn't dedupe reliably. The correct handling is to record an explicit PENDING/UNKNOWN state and actively reconcile it — either by querying the gateway's own idempotent status-check endpoint, or via a webhook the gateway sends asynchronously once it resolves the charge — rather than guessing. This is the payments-specific twist on standard resilience patterns (Resilience & Fault Tolerance module) that's worth calling out unprompted, since it's exactly the kind of nuance that separates a generic answer from one that reflects real payments experience."

---

## SECTION 3: NOTIFICATION SYSTEM

### Q4. Design a notification system that supports email, SMS, and push notifications across multiple triggering services. What's the core architectural decision?
**Answer:** The core decision is **decoupling notification-sending from the services that trigger notifications**, via a central **Notification Service** consumed asynchronously through a message queue — so the Order Service, the Comment Service, and every other feature that needs to notify a user don't each need their own email/SMS/push integration logic, retry handling, and provider credentials.

```
Order Service --publishes--> "OrderShipped" event -----> [Notification Service]
Comment Service --publishes--> "NewComment" event -/            |
                                                          [determines channel(s) per user preference]
                                                          [Email Adapter] [SMS Adapter] [Push Adapter]
```

```java
// Notification Service consumes generic domain events, maps to user-specific delivery
@KafkaListener(topics = {"order-events", "comment-events"})
public void handleEvent(NotificationTriggerEvent event) {
    UserPreferences prefs = preferencesService.get(event.getUserId());
    if (prefs.emailEnabled()) emailAdapter.send(event);
    if (prefs.pushEnabled())  pushAdapter.send(event);
}
```

**Senior-level answer:**
> "The design decision I'd emphasize is treating this explicitly as an **event-driven, fan-out problem** (Event-Driven Architecture module) — every feature team publishes a domain event when something notification-worthy happens, and the Notification Service alone owns the complexity of channel selection, user preferences, provider integration, and retry logic. Without this separation, every team ends up re-implementing 'call SendGrid, handle its failures, respect unsubscribe preferences' independently, which is both duplicated effort and a compliance risk if even one team gets unsubscribe handling wrong."

---

### Q5. How do you handle notification delivery reliability — ensuring a user gets notified without spamming them with duplicates on retry?
**Answer:** The same **Idempotent Consumer** pattern used elsewhere in event-driven systems: each notification event carries a unique ID, and the Notification Service records which notification IDs have already been sent **before** attempting delivery, in the same transaction, so a redelivered event (from at-least-once queue semantics) doesn't trigger a duplicate send.

**Additional reliability considerations specific to notifications:**
- **Per-channel retry with backoff** for transient provider failures (an SMS gateway briefly down), distinct from the message-queue-level redelivery.
- **Dead-letter handling** for notifications that repeatedly fail (an invalid phone number) — routed to a DLQ for investigation rather than retried forever.
- **Rate limiting/batching** for high-frequency triggers (e.g., don't send 50 separate "someone liked your post" notifications in a minute — batch into a digest: "12 people liked your post").

**Senior-level answer:**
> "The digest/batching point is the one I'd bring up as a practical, product-level detail beyond the pure infrastructure reliability question — a naive implementation that fires one push notification per like on a popular post creates a genuinely bad, spammy user experience even if every individual delivery is technically correct and non-duplicated. I'd design a short buffering window (e.g., batch like-notifications for a post over a 5-minute window into a single summarized notification) specifically to solve that, which is a UX-driven design decision on top of the pure reliability mechanics."

---

## SECTION 4: CHAT/MESSAGING SYSTEM

### Q6. Design a real-time chat system (like WhatsApp or Slack). Why is a standard REST API insufficient here, and what replaces it?
**Answer:** REST's request-response model requires the **client to initiate** every interaction — it has no mechanism for the **server** to proactively push a new incoming message to a client instantly. Real-time chat needs a **persistent, bidirectional connection**, most commonly **WebSockets** (or long-polling as a fallback for constrained environments).

```
Client A --WebSocket connect--> [Connection/Gateway Server] <--WebSocket connect-- Client B
Client A sends message -> Gateway Server -> [Message Service: persist + route]
                                                    |
                                          Client B's Gateway Server pushes over
                                          Client B's open WebSocket connection
```

**Key architectural pieces:**
- **Connection management layer** — tracks which server instance holds which user's active WebSocket connection (since with multiple server instances, a message for User B might arrive at a different instance than the one holding User B's actual connection — requiring a pub/sub layer like Redis Pub/Sub or Kafka to route the message to the correct instance).
- **Message persistence** — messages are stored (for history, offline delivery) independent of the real-time delivery path.
- **Presence/online status** — a separate concern, typically tracked via connection heartbeats.

**Senior-level answer:**
> "The detail that catches people off guard in this design: with multiple stateless-seeming application servers, WebSocket connections are actually **stateful** — a specific user's connection lives on one specific server instance at a time, which reintroduces exactly the state-management problem the Scalability module covers. The fix is a routing layer: when Client A sends a message to Client B, the server handling Client A's connection doesn't know which server instance holds Client B's connection, so it publishes to a shared pub/sub channel (Redis Pub/Sub, or a Kafka topic per user/shard) that every gateway server instance subscribes to, and whichever instance is actually holding Client B's live connection picks it up and pushes it through. This connection-routing layer is the piece that's easy to leave out of a first-pass design and worth raising proactively."

---

### Q7. How do you guarantee message ordering and handle offline delivery in a chat system?
**Answer:**
- **Ordering** — each message gets a **monotonically increasing sequence number per conversation** (not a wall-clock timestamp alone, which can be ambiguous/skewed across servers); clients render messages sorted by this sequence number, and can detect gaps (missing sequence numbers) to know a message needs to be re-fetched.
- **Offline delivery** — messages are always persisted first (write to the Message Service's database), independent of whether the recipient is currently connected; on reconnect, the client fetches any messages with a sequence number greater than the last one it has locally — the WebSocket push is an optimization for the online case, not the source of truth for delivery.

```java
// Persist first, push second — WebSocket delivery is a notification, not the record of truth
public void sendMessage(Message msg) {
    long sequenceNum = conversationSequencer.next(msg.getConversationId());
    messageRepository.save(msg.withSequence(sequenceNum));  // durable, source of truth
    pushIfRecipientOnline(msg);                               // best-effort real-time delivery
}
```

**Senior-level answer:**
> "The principle I make explicit: the WebSocket push is purely a **real-time delivery optimization**, never the actual guarantee of delivery — the database write is the durable source of truth, and a client reconnecting after being offline (or a client that never received the WebSocket push due to a dropped connection) always has a reliable path to catch up by querying for messages after its last known sequence number. This separation is what makes the system resilient to a flaky mobile connection — a very real, common condition for a chat app — without needing complicated at-delivery-time acknowledgment tracking on the WebSocket layer itself."

---

## SECTION 5: BOOKING/RIDE-SHARING SYSTEM

### Q8. Design a ride-sharing system's core matching flow (rider requests a ride, gets matched with a nearby driver). What's the hardest part of this design?
**Answer:** The hardest part is **efficiently finding nearby available drivers** from potentially millions of drivers' constantly-updating locations, and doing so **without race conditions** — two riders shouldn't be matched to the same driver simultaneously.

**Geospatial indexing** is the core building block — rather than scanning every driver's location on every match request, drivers' locations are indexed in a structure that supports fast "who's near this point" queries:
- **Geohashing** — encode lat/long into a string where nearby locations share string prefixes, enabling range queries on a standard index.
- **Redis Geo commands** (`GEOADD`, `GEORADIUS`) — a purpose-built, commonly used solution that handles this indexing natively and is fast enough for real-time driver-location updates.

```
GEOADD drivers:live -122.41 37.77 "driver_123"          # driver location update, frequent
GEORADIUS drivers:live -122.42 37.78 2 km ASC COUNT 10   # find nearest 10 available drivers within 2km
```

**Avoiding double-matching:** once a driver is selected as a match candidate, **atomically claim** them (e.g., a Redis `SETNX`-based lock on that driver ID, or a DB-level conditional update `WHERE status = 'AVAILABLE'`) before notifying the rider — the same distributed-locking concern covered in the Distributed Caching module, applied here to prevent two concurrent match requests from both claiming the same driver.

**Senior-level answer:**
> "I always bring up the double-matching race condition proactively, because it's the kind of correctness bug that's easy to miss in a first pass but is a real, embarrassing bug in production — two riders both see 'driver found' for the same driver. The fix is the same atomic compare-and-set primitive used in distributed locking generally: claim the driver with an atomic conditional update, and only then notify the rider; if the atomic claim fails (someone else got there first), fall back to the next-nearest candidate. Redis Geo commands plus an atomic claim step is the combination I'd walk through, since it directly demonstrates both the geospatial and the concurrency-correctness halves of the problem."

---

### Q9. How would you extend the design to handle surge pricing and estimated arrival time (ETA) calculation?
**Answer:**
- **Surge pricing** — computed from the **real-time ratio of ride requests to available drivers within a geographic zone**; a background aggregation job (or a streaming aggregation over ride-request/driver-availability events) continuously recomputes a surge multiplier per zone, cached and served to the pricing calculation on each ride request — this needs to be **fast to read** (every ride quote depends on it) even though it changes frequently.
- **ETA calculation** — typically delegated to a specialized routing/mapping service (Google Maps Directions API or an internal equivalent) rather than built from scratch, since accurate ETA requires real road network, traffic, and routing data that's a substantial specialized system on its own — a system design interview answer should recognize this as "integrate with a routing provider," not attempt to design a full routing engine unless the interviewer specifically steers there.

**Senior-level answer:**
> "For surge pricing specifically, I'd emphasize that this is a **read-heavy, write-tolerant-of-staleness** problem — every single ride quote needs the current surge multiplier for its zone, but the multiplier itself only needs to update every few seconds to a minute, not on every individual request. That asymmetry is exactly what justifies caching the computed multiplier per zone rather than recalculating supply/demand from scratch on every quote request — a classic case of the caching-strategy reasoning from the System Design Process module applied to a specific, concrete number that directly affects what the user is charged."

---

## SECTION 6: E-COMMERCE / ORDER MANAGEMENT

### Q10. Design the order placement flow for an e-commerce system, specifically handling inventory correctly under concurrent purchases of the same limited-stock item.
**Answer:** The critical correctness problem: **two customers simultaneously buying the last unit of an item** must not both succeed — this is a classic race condition requiring either **pessimistic locking** or an **atomic conditional decrement** at the database level, not a naive read-then-write.

```sql
-- WRONG — race condition: two concurrent requests can both read stock=1, both proceed to decrement
SELECT stock FROM inventory WHERE product_id = 'X';   -- both read stock = 1
-- both then separately: UPDATE inventory SET stock = stock - 1 WHERE product_id = 'X';
-- result: stock = -1, both orders "succeeded" for an item with only 1 unit

-- RIGHT — atomic conditional update, only one request succeeds
UPDATE inventory SET stock = stock - 1
WHERE product_id = 'X' AND stock > 0;
-- check rows-affected: 1 = success, 0 = out of stock, atomically, no race window
```

**The broader flow** then follows the **Saga pattern** (Distributed Data Management module) across services — Order Service reserves inventory, Payment Service charges, and if payment fails, a compensating action releases the reserved inventory back:
```
1. Order Service: create order (PENDING), atomically decrement inventory (reserve)
2. Payment Service: attempt charge
   -> success: Order Service marks order CONFIRMED
   -> failure: Order Service marks order CANCELLED, compensating action releases inventory back (+1)
```

**Senior-level answer:**
> "I lead with the atomic-decrement detail because it's the single highest-signal correctness question in this entire design — a read-then-write inventory check is a textbook race condition, and I've seen it cause real overselling incidents. Beyond that single-service correctness fix, the interesting distributed-systems question is what happens when inventory is correctly reserved but the *payment* step fails afterward — that's not a bug, that's an expected outcome requiring an explicit compensating transaction to release the reservation, which is exactly the Saga pattern's reason for existing. I make sure to walk through both the single-database race condition and the cross-service compensation explicitly, since they're different problems at different layers."

---

### Q11. How do you design the order status/tracking system so a customer sees accurate, real-time order status without hammering the database on every page load?
**Answer:** Order status changes are **infrequent relative to how often customers check them** (a customer might refresh a tracking page dozens of times while the order status changes maybe 5 times total over its lifecycle) — a strong cache-aside candidate (Distributed Caching module), invalidated specifically on each status transition rather than left to a blind TTL.

```java
public OrderStatus getStatus(String orderId) {
    OrderStatus cached = redisTemplate.opsForValue().get("order-status:" + orderId);
    if (cached != null) return cached;
    OrderStatus fromDb = orderRepository.getStatus(orderId);
    redisTemplate.opsForValue().set("order-status:" + orderId, fromDb, Duration.ofMinutes(30));
    return fromDb;
}

// On each status transition (e.g., SHIPPED), invalidate immediately — don't wait on TTL
public void updateStatus(String orderId, OrderStatus newStatus) {
    orderRepository.updateStatus(orderId, newStatus);
    redisTemplate.delete("order-status:" + orderId);   // Q6 pattern from Distributed Caching module
}
```

**For genuinely real-time push (optional, if the interviewer wants more):** WebSockets or Server-Sent Events for customers actively viewing the tracking page, publishing a status-change event that pushes to any connected client watching that order — same connection-routing consideration as the chat system design (Q6).

**Senior-level answer:**
> "This is a direct application of the read-heavy, infrequently-changing data pattern — same reasoning as the URL shortener's redirect path. I'd default to cache-aside with explicit invalidation on write rather than a short TTL, since order status changes are meaningful, discrete events that we always control and know about, so there's no reason to accept staleness up to a TTL window when we can invalidate precisely at the moment it actually changes. I'd only add WebSocket/SSE push if the interviewer specifically wants a 'live updating' experience without a page refresh — otherwise, cache-aside with fast invalidation already gives a snappy, mostly-real-time-feeling experience with far less complexity."

---

## SECTION 7: SOCIAL MEDIA FEED

### Q12. Design a social media feed (like Twitter's home timeline). What's the fundamental architectural choice, and what's the trade-off?
**Answer:** The fundamental choice is **fan-out-on-write vs. fan-out-on-read**:

| | Fan-out-on-write (push) | Fan-out-on-read (pull) |
|---|---|---|
| **How it works** | When a user posts, immediately write that post into every follower's precomputed feed | When a user opens their feed, query and merge recent posts from everyone they follow, on the fly |
| **Read performance** | Very fast — feed is precomputed, just fetch it | Slower — real-time fan-in query across potentially thousands of followed accounts |
| **Write cost** | Expensive for users with huge follower counts (a celebrity's post fans out to millions of feed writes) | Cheap — a post is just one write regardless of follower count |
| **Best for** | The vast majority of users (normal follower counts) | High-profile/celebrity accounts, where write fan-out would be enormous |

**Senior-level answer:**
> "The celebrity problem is exactly why real systems (this is publicly documented as Twitter's actual approach) use a **hybrid**: fan-out-on-write for the overwhelming majority of users, since their follower counts are small enough that pushing a new post to every follower's feed is cheap and gives fast reads — but for accounts above some follower-count threshold, switch to fan-out-on-read, merging their posts into a requester's feed at read time instead of writing to millions of individual feed caches on every single post. I always bring up this hybrid as the answer, because a pure fan-out-on-write design has a well-known, real breaking point, and recognizing it (rather than proposing a naive pure-push design) is a strong signal in this specific, very commonly-asked problem."

---

### Q13. How would you rank/order posts in a feed beyond simple reverse-chronological order, and how does that change the caching/precomputation strategy?
**Answer:** A ranked (algorithmic) feed scores each candidate post using signals — recency, engagement (likes/comments), affinity between the viewer and the poster, content type preferences — and orders by that score rather than pure timestamp. This changes the architecture meaningfully:
- **Precomputed feeds (fan-out-on-write) become harder to maintain** — a purely chronological feed just needs new posts prepended; a ranked feed's order can change based on engagement *after* the post was already written into followers' feeds (a post gaining likes hours later should potentially re-rank), which a simple precomputed list doesn't handle well.
- **Common real-world approach**: fan-out-on-write still populates a **candidate pool** per user (a reasonably-sized set of recent posts from followed accounts), but a **ranking service scores and re-orders that candidate pool at read time**, combining the cheap-write benefit of fan-out with fresh, up-to-date ranking at read time.

**Senior-level answer:**
> "I present this as fan-out and ranking being **separable concerns** — fan-out-on-write solves 'which posts are even candidates for this user's feed,' cheaply, while ranking is a separate, read-time scoring pass over that already-narrowed candidate set, not something that needs to be baked into the write path. This separation is what lets the ranking algorithm evolve independently (new signals, a machine-learning model update) without needing to re-fan-out historical data every time the ranking logic changes — the candidate pool stays the same, only the read-time scoring changes."

---

## SECTION 8: FILE STORAGE SYSTEM

### Q14. Design a file storage/sharing system (like Dropbox or Google Drive). What are the two fundamentally different concerns this system needs to separate?
**Answer:** File storage systems need to cleanly separate **metadata** (file name, folder structure, owner, sharing permissions, version history — small, relational, frequently queried) from **actual file content/blob data** (potentially huge, rarely needs complex querying, just needs durable storage and fast retrieval by key). Conflating these — e.g., storing file bytes directly in a relational database — is a common design mistake that doesn't scale.

```
Metadata (relational DB — Postgres/MySQL):
  files(id, name, folder_id, owner_id, size, mime_type, current_version_id, ...)
  permissions(file_id, user_id, access_level)

Blob storage (object store — S3/GCS, NOT the relational DB):
  actual file bytes, addressed by a content hash or generated key,
  the metadata DB stores a REFERENCE (the object store key), not the bytes themselves
```

**Senior-level answer:**
> "The metadata/blob separation is the single most important architectural decision here, and it maps directly onto the earlier data-model reasoning about choosing storage based on access patterns — file metadata benefits from a relational database's querying strength (list a folder, check permissions, search by name), while raw file bytes have completely different needs: massive potential size, no need for complex queries, and durability/availability requirements that a dedicated object store (S3) is purpose-built for and dramatically cheaper at scale than storing blobs in a relational database ever would be."

---

### Q15. How do you handle large file uploads efficiently and support features like resumable uploads and deduplication?
**Answer:**
- **Chunked/multipart upload** — large files are split into chunks client-side and uploaded independently (often directly to the object store via pre-signed URLs, bypassing the application server entirely for the actual bytes) — this both parallelizes upload speed and enables **resumability**: if the connection drops mid-upload, only the failed chunk needs to be retried, not the entire file.
- **Deduplication** — content-addressable storage: compute a hash (e.g., SHA-256) of the file content, and use that hash as the storage key. If two users upload byte-identical files, the second upload detects the hash already exists in the object store and simply creates a new metadata reference to the existing blob, **without storing the bytes a second time**.

```
Client splits file into 5MB chunks -> requests a pre-signed upload URL per chunk from the app server
                                    -> uploads each chunk directly to S3 (app server never touches the bytes)
                                    -> app server notified on completion, assembles metadata + finalizes object

Dedup: SHA-256(file bytes) = "abc123..."
  if blob "abc123..." already exists in object store -> just add a new metadata row pointing to it
  else -> store the new blob, then add the metadata row
```

**Senior-level answer:**
> "The pre-signed URL detail is worth calling out specifically, because it's a meaningful architectural choice, not just an implementation nicety — having the client upload chunks **directly** to the object store rather than routing large file bytes through the application server avoids making the app server a bandwidth bottleneck for something it doesn't actually need to process. Content-addressable deduplication is the other detail I'd bring up unprompted, since it's both a genuine storage-cost optimization at scale (a viral shared file uploaded independently by thousands of users only needs to be stored once) and a nice, concrete example of hashing being used for a purpose beyond security — pure content identification."

---

## QUICK-FIRE FOLLOW-UPS TO EXPECT (Rapid Round)

- "For the URL shortener, how would you handle custom/user-chosen short codes alongside auto-generated ones?" → Check availability of the requested custom code atomically (a conditional insert that fails on conflict) before falling back to auto-generation if taken — same atomic-uniqueness concern as any unique-key allocation.
- "For payments, why not just use a database transaction across the payment gateway call?" → You cannot include an external HTTP call to a third-party gateway inside a local ACID database transaction — this is exactly the dual-write problem the Transactional Outbox pattern exists to solve (Distributed Data Management module).
- "For chat, how would you show 'typing...' indicators without persisting every keystroke?" → Ephemeral, not persisted — sent directly over the WebSocket connection with no database write at all, since it's transient UI state with no need for durability or history.
- "For ride-sharing, what happens if the matched driver doesn't respond in time?" → A timeout on the match offer releases the atomic claim (Q8) back to available, and the system falls back to the next-nearest candidate driver — same pattern as a distributed lock's TTL expiring.
- "For e-commerce, how do you handle an item added to cart but never purchased, holding up inventory?" → Cart reservations should have a short TTL (e.g., 15 minutes) after which the reservation is automatically released — never hold inventory indefinitely just because it's sitting in someone's cart.
- "For the social feed, how do you handle a user who follows thousands of accounts efficiently at read time?" → This is exactly the scenario fan-out-on-read is bad at and fan-out-on-write handles well — reinforces why the hybrid approach (Q12) is the standard real-world answer, not a pure strategy in either direction.

---

*Study tip: For every one of these problems, the recurring skill being tested is the same — identify the single hardest correctness or scale problem specific to that domain (double-charging in payments, overselling in e-commerce, celebrity fan-out in feeds, connection routing in chat) and design directly around it, rather than describing a generic layered architecture that could apply to any system. Interviewers reuse these exact problems specifically because each has a well-known "gotcha," and reliably surfacing it unprompted is the clearest signal of real system design experience.*