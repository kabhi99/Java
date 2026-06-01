# **READ REPLICAS, REPLICATION LAG & CONSISTENCY**

How to use read replicas without breaking your users — specifically, how to guarantee a user always sees their own writes.

### TABLE OF CONTENTS

1. Why Read Replicas Cause Bugs (the trade-off you bought)
2. Replication: Synchronous vs Asynchronous
3. Consistency Models You Need to Know
4. **PATTERN 1: Read-from-Primary After Write (time-based sticky)**
5. **PATTERN 2: LSN-Aware Reads (wait for replica to catch up)**
6. **PATTERN 3: Session Stickiness (route a user briefly)**
7. **PATTERN 4: Write-Through Cache (return the just-written value)**
8. **PATTERN 5: Synchronous Replication (when correctness > latency)**
9. **PATTERN 6: Causal Consistency via Read Tokens**
10. **FEATURED PROBLEM: "Why Does My API Return Stale Data" (LinkedIn Q)**
11. Cache + Replicas: The Combined Trap
12. Golden Rules & Interview Cheat Codes

---

## PART 1: WHY READ REPLICAS CAUSE BUGS

> When you added read replicas, you traded **strong consistency** for **read throughput**. Most engineers don't realize they made the trade.

```
BEFORE (single DB):
  Write    → Primary
  Read     → Primary       ← always sees latest
  ✓ strong consistency, but throughput-limited

AFTER (with replicas):
  Write    → Primary
  Read     → Replica       ← may lag 100ms-1s+
  ✓ 5-10x more read throughput
  ✗ replica is BEHIND primary — your reads can return stale data
```

### The "It Looked Healthy" Trap

All your standard metrics will be GREEN during this bug:
- Write latency: fine
- API latency: fine
- Error rate: 0% (HTTP 200 with stale data!)
- Replica lag: "only 250ms"

> **CORRECTNESS bugs don't show up in availability metrics.** You need explicit **staleness alerts** and **correctness tests**.

---

## PART 2: REPLICATION — SYNC vs ASYNC

| Mode | Behavior | Pros | Cons |
|---|---|---|---|
| **Async (default)** | Primary commits, then **eventually** sends to replicas | Fast writes, high throughput | Replicas lag → stale reads possible |
| **Semi-sync** | Primary waits for **at least 1** replica to ack | Better durability, modest lag | Slower writes |
| **Sync** | Primary waits for **all** replicas to ack | Replicas always current | Slowest writes; one slow replica blocks all writes |

### **Postgres specifically**
```sql
-- Async (default)
synchronous_commit = off    -- or 'local'

-- Sync (writes wait for replica)
synchronous_commit = on     -- + synchronous_standby_names = 'replica1'

-- Per-transaction (best of both)
BEGIN;
SET LOCAL synchronous_commit = on;        -- this one txn waits
UPDATE users SET name='Suraj' WHERE id=123;
COMMIT;
```

> **Production pattern**: keep async by default, but use sync **per-transaction** for critical writes (e.g., financial commits) or use **PATTERNS 1-6 below** to handle async correctly.

---

## PART 3: CONSISTENCY MODELS

```
STRONGEST ↑
+----------------------------------+--------------------------------+
| Strong consistency                | All reads see latest write     |
| Linearizable                      | Real-time order preserved      |
| Sequential                        | Single global order            |
| Read-your-own-writes (RYW)        | You see YOUR writes (most APIs)|
| Monotonic reads                   | Never see "old then new" go back|
| Causal consistency                | If A→B happened, all see A→B   |
| Eventual consistency              | Eventually convergent          |
+----------------------------------+--------------------------------+
↓ WEAKEST
```

### **What most user-facing APIs need: Read-Your-Own-Writes (RYW)**

> Users tolerate other people's data being stale for a second. They do NOT tolerate their own update vanishing.

The 6 patterns below all solve RYW with different trade-offs.

---

## PART 4: PATTERN 1 — Read-from-Primary After Write (time-based sticky)

> **Simplest, most common fix.** After a write, route the same user's reads to **primary** for N seconds (longer than max replica lag).

### Idea
- User does `PUT /users/123` → write to primary
- Mark "user 123 wrote at time T" in a fast store (Redis, ThreadLocal+session)
- For next `T + 2 seconds`, route reads of user 123 to **primary**
- After 2 seconds, replica is caught up → safe to route to replica again

### Code

```java
@Component
public class WriteAwareRouter {

    private final RedisTemplate<String, Long> redis;
    private static final Duration STICKY_WINDOW = Duration.ofSeconds(2);

    public void markWrote(String userId) {
        redis.opsForValue().set(
            "wrote:" + userId,
            System.currentTimeMillis(),
            STICKY_WINDOW
        );
    }

    public boolean shouldReadFromPrimary(String userId) {
        return redis.hasKey("wrote:" + userId);
    }
}

// Wire it into the routing data source
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Autowired private WriteAwareRouter router;

    protected Object determineCurrentLookupKey() {
        String userId = RequestContext.currentUserId();
        if (router.shouldReadFromPrimary(userId)) return DbRole.PRIMARY;
        return DbContext.getRole();  // honors @ReadOnly otherwise
    }
}

// Service layer
@Service
public class UserService {
    private final UserRepository repo;
    private final WriteAwareRouter router;

    public User update(String userId, UpdateRequest req) {
        User u = repo.findById(userId).orElseThrow();
        u.applyUpdate(req);
        User saved = repo.save(u);          // writes to PRIMARY
        router.markWrote(userId);            // stick reads to primary for 2s
        return saved;
    }

    @ReadOnly
    public User get(String userId) {
        return repo.findById(userId).orElseThrow();
        //  Will hit PRIMARY if recently wrote, REPLICA otherwise
    }
}
```

### Trade-offs

| Pro | Con |
|---|---|
| Simple, works for 99% of cases | Bursts of writes hit primary (some load) |
| No application changes besides router | Window is a guess (need to be > p99 replica lag) |
| Per-user — minimal extra primary load | If window too short, bug returns |

> **Sizing**: set window = `p99 replica lag × 2`. If p99 lag = 500ms, use 1-2 seconds.

---

## PART 5: PATTERN 2 — LSN-Aware Reads (wait for replica to catch up)

> **More precise than time-based.** Track the exact log position (LSN) of the write; only read from a replica that has caught up to that LSN.

### Idea
- After a write, capture Postgres's WAL LSN (`pg_current_wal_lsn()`)
- Store LSN in the user's session/cookie/cache
- On read, check the replica's `pg_last_wal_replay_lsn()`
- If replica LSN ≥ user's LSN → safe to read; else read from primary (or wait)

### Code

```java
@Service
public class UserService {

    @Transactional
    public User update(String userId, UpdateRequest req) {
        User saved = repo.save(...);
        // Capture LSN of THIS write
        String lsn = jdbcTemplate.queryForObject(
            "SELECT pg_current_wal_lsn()::text", String.class);
        sessionStore.saveLsn(userId, lsn);
        return saved;
    }

    public User get(String userId) {
        String requiredLsn = sessionStore.getLsn(userId);
        if (requiredLsn == null) return readFromReplica(userId);

        String replicaLsn = jdbcTemplate.queryForObject(
            "SELECT pg_last_wal_replay_lsn()::text", String.class);

        if (lsnGreaterOrEqual(replicaLsn, requiredLsn)) {
            return readFromReplica(userId);     // safe!
        }
        return readFromPrimary(userId);         // replica behind → use primary
    }
}
```

### Trade-offs

| Pro | Con |
|---|---|
| Precise — exact RYW guarantee | DB-specific (Postgres LSN, MySQL GTID) |
| Reads switch back to replica ASAP | Extra query per read to check replica LSN |
| Works across multiple replicas | More plumbing |

> **Used by**: AWS Aurora has a similar mechanism via "session pinning"; companies like Discord and Notion use LSN-aware routing.

---

## PART 6: PATTERN 3 — Session Stickiness

> Route an entire user's session through the same path. Useful when you also want consistent caching.

### Variants

1. **Sticky to primary for K minutes after login** — heavy-handed; defeats replica purpose
2. **Sticky to a specific replica** — user always reads from replica 1; if they wrote, replica 1 may lag. NOT a fix by itself.
3. **Sticky session + LSN check** — combine with PATTERN 2

> Generally, PATTERN 1 (time-based) or PATTERN 2 (LSN-based) is better than full session stickiness.

---

## PART 7: PATTERN 4 — Write-Through Cache (return the just-written value)

> Avoid the read entirely. The PUT response should contain the new state — let the client trust its own write.

### Idea

```java
@PutMapping("/users/{id}")
public ResponseEntity<User> update(@PathVariable String id, @RequestBody UpdateRequest req) {
    User updated = userService.update(id, req);
    // Return FULL updated object. Client SHOULD NOT need to GET again.
    return ResponseEntity.ok(updated);
}
```

### Client-side
```javascript
// Mobile/web client
const updated = await fetch('PUT /users/123', { body: ... }).json();
setLocalState(updated);   //  Use the response, don't refetch
```

### Trade-offs

| Pro | Con |
|---|---|
| Zero replica lag impact | Client must respect the contract |
| Saves a round-trip | Doesn't help when ANOTHER client refetches |

> **Required**: REST API best practice anyway. Always return the full updated resource on PUT/POST.

---

## PART 8: PATTERN 5 — Synchronous Replication

> When correctness matters more than latency, make the write wait for the replica.

```sql
-- Per-transaction (recommended)
BEGIN;
SET LOCAL synchronous_commit = remote_apply;   -- wait until replica APPLIES, not just receives
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### Postgres `synchronous_commit` modes
- `off` — Don't wait even for local flush (fastest, can lose data on crash)
- `local` — Wait for local WAL flush (default, durable but async to replicas)
- `remote_write` — Wait until replica writes to its WAL (still in memory)
- `remote_apply` — Wait until replica APPLIES the change (full RYW guarantee)

### Trade-offs

| Pro | Con |
|---|---|
| Strong consistency guarantee | Write latency +50-200ms |
| Simple — no app changes | One slow replica blocks all sync writes |
| Per-transaction control | Not all DB engines support |

> **Use for**: financial commits, password changes, anything where stale = legal risk. Use **async + PATTERN 1** for everything else.

---

## PART 9: PATTERN 6 — Causal Consistency via Read Tokens

> Used at hyperscale (Facebook, MongoDB, FoundationDB). Client carries an opaque token; server enforces "this read is after that write".

### Idea
```
1. Client writes        → server returns Token (e.g., LSN, vector clock)
2. Client stores token
3. Client reads         → sends Token with request
4. Server: "Replica is at version V; token requires version V'. V >= V' → serve; else wait or route to primary"
```

### Pseudo-implementation

```java
// On write
@PutMapping("/users/{id}")
public User update(...) {
    User saved = service.update(...);
    String token = service.captureToken();
    return ResponseEntity.ok().header("X-Read-Token", token).body(saved);
}

// On read
@GetMapping("/users/{id}")
public User get(@PathVariable String id, @RequestHeader(value="X-Read-Token", required=false) String token) {
    if (token != null && !service.replicaCaughtUp(token)) {
        return service.readFromPrimary(id);
    }
    return service.readFromReplica(id);
}
```

> Powerful, but adds protocol complexity. Use only when **strict causal consistency** is required across multiple unrelated entities.

---

## PART 10: FEATURED — "WHY DOES MY API RETURN STALE DATA"

### **THE PROBLEM** (LinkedIn interview question)

> Spring Boot app, recently scaled with Postgres read replicas.
> - Writes → Primary
> - Reads → Replica
>
> User flow:
> ```
> PUT /users/123  {"displayName": "Suraj"}
> immediately followed by
> GET /users/123  → sometimes returns "Old Name"
> ```
>
> **Monitoring:**
> - Primary write latency: healthy
> - **Replica lag: 250-800ms**
> - API latency: healthy
> - Error rate: 0%
>
> "Yet users think the system is broken."

### **DIAGNOSIS**

This is a textbook **read-your-own-writes (RYW) violation**.

| Signal | Inference |
|---|---|
| PUT then immediate GET fails | Classic RYW |
| Replica lag 250-800ms | The exact window where bug exists |
| Error rate 0% | Silent correctness bug (worst kind) |
| Healthy metrics everywhere | Standard SLOs miss correctness |

### **WHY IT HAPPENS**

```
T=0ms:   PUT /users/123 → write to PRIMARY (commits at T=10ms)
T=10ms:  Replication lag starts; replica is behind
T=12ms:  GET /users/123 → routed to REPLICA (lag = 300ms)
T=12ms:  Replica still has old data → returns "Old Name"
T=310ms: Replica catches up → from now on, GET returns "Suraj"
```

The whole **300ms window** is when reads return stale. With 1000 RPS, that's 300 affected requests per write.

### **THE FIX — Layered Solution**

#### **Layer 1: Return the updated resource on PUT (free win)**

```java
@PutMapping("/users/{id}")
public ResponseEntity<UserDto> update(@PathVariable String id, @RequestBody UpdateRequest req) {
    User updated = userService.update(id, req);
    return ResponseEntity.ok(UserDto.from(updated));   //  full updated object
}
```
The client should not need to GET again. **This alone fixes most cases.**

#### **Layer 2: Write-aware routing (PATTERN 1)**

For cases where another component refetches the user, route reads to primary for 2 seconds after a write.

```java
@Service
public class UserService {
    private final UserRepository repo;
    private final WriteAwareRouter router;

    @Transactional
    public User update(String userId, UpdateRequest req) {
        User saved = repo.save(applyUpdate(userId, req));
        router.markWrote(userId);     // sticky to primary for 2s
        return saved;
    }

    @ReadOnly
    public User get(String userId) {
        return repo.findById(userId).orElseThrow();
        // routing data source: primary if recently wrote, else replica
    }
}
```

#### **Layer 3: LSN-aware reads (PATTERN 2) — for stricter guarantees**

Used by Discord, Notion, Aurora. More plumbing, exact RYW.

#### **Layer 4: Synchronous replication for CRITICAL writes**

```java
@Transactional
public void changePassword(String userId, String newHash) {
    jdbcTemplate.execute("SET LOCAL synchronous_commit = remote_apply");
    repo.updatePassword(userId, newHash);
    // Now even an immediate read from any replica will see the new password.
}
```

### **EXPECTED OUTCOME**

| Before | After |
|---|---|
| Random "Old Name" responses for 250-800ms after every write | 100% RYW — user always sees their own update |
| Customer complaints | Silent fix |
| Error rate metric: 0% (lying!) | Add staleness/RYW correctness probe to monitoring |

### **CRITICAL: Add correctness monitoring**

```java
@Scheduled(fixedRate = 30_000)
public void rywCorrectnessProbe() {
    String marker = UUID.randomUUID().toString();
    userService.update("probe-user", new UpdateRequest(marker));
    for (int i = 0; i < 10; i++) {
        String got = userService.get("probe-user").getDisplayName();
        if (!marker.equals(got)) {
            meterRegistry.counter("ryw.violation").increment();
            log.error("RYW violation: wrote {} but read {}", marker, got);
        }
    }
}
```
This catches the bug *during the next deploy*, not from a customer ticket.

### **INTERVIEW NARRATIVE**

> "Every metric being green is the giveaway — this is a **silent correctness bug**, not an availability bug. It's a textbook **read-your-own-writes violation** caused by read replica lag.
>
> The architecture trades strong consistency for read throughput. The 300ms replication lag window is when reads can return stale data.
>
> I'd fix in layers:
>
> **First** — **return the full updated resource on PUT**. The client shouldn't need to GET again. This is REST best practice and fixes most cases for free.
>
> **Second** — **write-aware routing**: after a user writes, route their reads to primary for a window slightly longer than p99 replica lag (say 2 seconds). I'd use a Redis-backed marker so it works across all pods.
>
> **Third** — for stricter guarantees, **LSN-aware reads**: capture `pg_current_wal_lsn()` after the write, store it in the user's session, and only read from a replica whose `pg_last_wal_replay_lsn()` has caught up.
>
> **Fourth** — for critical writes like password changes, use `synchronous_commit = remote_apply` per-transaction so the write physically blocks until replicas have applied it.
>
> **Finally** — add a **RYW correctness probe** to monitoring. The lesson here is that standard SLOs don't catch correctness bugs. You need explicit probes that write and immediately read, alerting on mismatches."

---

## PART 11: CACHE + REPLICAS — THE COMBINED TRAP

When you have BOTH a cache AND read replicas, you have **3 sources of truth** in 3 states of staleness:

```
              fresh ↓
          Primary DB ────write─── User writes
              │
              │ async replication (lag: 100-1000ms)
              ↓
          Replica DB ───read── App reads
              │
              │ stored in cache
              ↓
          Redis / Caffeine ──read── App reads
              │
                stale ↑ (could be 5-60 minutes!)
```

### **The "Refresh Refresh Refresh" Bug** (real production case)

User updates name → refresh → old name (replica lag) → refresh → new name (cache miss, replica caught up) → refresh → old name (cache hit on entry written from old replica response).

### **Fix order**

1. **Invalidate cache on write** — write-through or explicit invalidation
2. **Read-from-primary after write** — for both the write response AND for cache repopulation
3. **Short cache TTL** — never trust cache for longer than ~replica lag + safety buffer

```java
@Transactional
public User update(String id, UpdateRequest req) {
    User saved = repo.save(...);             // PRIMARY
    router.markWrote(id);                     // route reads to primary for 2s
    cache.invalidate("user:" + id);           // invalidate cache
    pubsub.publish("user-invalidate", id);    // tell other pods' L1 caches
    return saved;
}

public User get(String id) {
    User cached = cache.get(id);
    if (cached != null) return cached;

    User u = repo.findById(id).orElseThrow();  // hits primary if marker present
    cache.put(id, u, Duration.ofMinutes(5));
    return u;
}
```

---

## PART 12: GOLDEN RULES

```
+----------------------------------------------------------------------+
|  1. Read replicas trade STRONG CONSISTENCY for THROUGHPUT — own it   |
|  2. Default user expectation = Read-Your-Own-Writes (RYW)            |
|  3. ALWAYS return the updated resource on PUT/POST (saves a refetch) |
|  4. After a write, route same user's reads to PRIMARY for ~2 sec     |
|  5. Window = max(p99 replica lag) × 2                                |
|  6. For strict RYW: use LSN-aware reads (Postgres LSN, MySQL GTID)   |
|  7. For critical writes: SET LOCAL synchronous_commit = remote_apply |
|  8. Standard metrics LIE about correctness — add RYW probes          |
|  9. Cache + replicas = compounded staleness — invalidate aggressively |
| 10. NEVER read your own write from a replica (always route primary)  |
| 11. Replication lag is a SLO — monitor p99 lag, alert at 1s+         |
| 12. Multi-master replication ≠ free consistency (need quorum reads)  |
| 13. Sync replication blocks writes if any replica is slow            |
| 14. Eventual consistency is fine for OTHER users' data, not own      |
| 15. Document the consistency contract in your API docs               |
+----------------------------------------------------------------------+
```

---

## PART 13: INTERVIEW CHEAT CODES

> "When you add read replicas, you trade **strong consistency** for **throughput**. The default user expectation is **read-your-own-writes** — they tolerate other people's data being stale, never their own."

> "The clue here is that every metric is healthy but users see broken behavior. That's a **silent correctness bug** — error rate is 0% because we're returning HTTP 200 with stale data. Standard SLOs don't catch correctness."

> "Simplest fix: **return the full updated resource on PUT**. Clients shouldn't need to GET again. REST best practice and free RYW for the writer."

> "For the cases where another component refetches: **write-aware routing**. After a write, route that user's reads to primary for a window longer than p99 replica lag — usually 1-2 seconds via a Redis marker."

> "For exact RYW: **LSN-aware reads** — capture `pg_current_wal_lsn()` on write, store in session, only read from replicas where `pg_last_wal_replay_lsn()` has caught up. Aurora and Discord do this."

> "For critical writes — passwords, money — use `synchronous_commit = remote_apply` per-transaction. The write physically blocks until replicas have applied. Slower but correct."

> "I add a **correctness probe** to monitoring: write a marker, immediately read it back from multiple paths, count mismatches. This catches the bug during the next deploy, not from a customer ticket."

> "Cache plus replicas = compounded staleness across three sources of truth. Cache must be invalidated on write, and the cache repopulation must read from primary — otherwise you cache stale data and lock the bug in for the TTL."

---

## QUICK REVISION TABLE

```
+----------------------------------+--------------------------------------+
| Scenario                          | Fix                                  |
+----------------------------------+--------------------------------------+
| PUT then GET same user           | Return updated obj + route to primary|
| Critical write (money, auth)     | synchronous_commit = remote_apply    |
| Multiple replicas, want fastest  | LSN-aware routing                    |
| Cross-service causal consistency | Read tokens (vector clock / LSN)     |
| Cache + replicas combined        | Invalidate cache + cache from primary|
| One slow replica blocks sync     | Use semi-sync (at least 1 ack)       |
| Need to detect the bug           | RYW correctness probe                |
| Multiple users, same query       | Replica fine (other users tolerate)  |
| Window of staleness too long     | Investigate replica lag spike (WAL)  |
| Read-after-deletion case         | Same patterns — route to primary     |
+----------------------------------+--------------------------------------+
```

---

## RELATED PATTERNS

- See `02-Spring-Boot-Resilience-Patterns.md` Part 4 — timeouts and circuit breaker for replica health
- See `04-Scaling-Spring-Boot-APIs.md` Part 5 — `AbstractRoutingDataSource` setup
- See `04-Scaling-Spring-Boot-APIs.md` Part 3 — cache-aside pattern (combine with routing carefully)
