# **JAVA / SPRING BOOT INTERVIEW CHEAT SHEET**

One-page lookup. Given a problem signal → which pattern → which file/section.

> **How to use**: Read top-to-bottom in 10 min for a refresh. During an interview, use as a decision tree to never miss the obvious lever.

---

## THE MASTER DECISION TREE

```
What is the interviewer testing?
│
├── CORRECTNESS (data is wrong, not just slow)
│   ├── Same write happening twice / double-charge       → 02 §9  (Idempotency)
│   ├── Lost update on shared row                        → 01 §1  (Race conditions)
│   ├── User can't see their own write                   → 06     (RYW + replica lag)
│   ├── Old data after refresh                           → 06 §11 (Cache + replica trap)
│   ├── HashMap blew up under threads                    → 01 §3  (ConcurrentHashMap)
│   └── Deadlock between two operations                  → 01 §4  (Lock ordering)
│
├── PERFORMANCE (slow, high latency)
│   ├── DB CPU > 80%                                     → 04 §3  (Caching) + §5 (Read replicas)
│   ├── Many lookups for non-existent items              → 04 §4  (Bloom filter)
│   ├── LIKE '%x%' too slow                              → 04 §6  (pg_trgm / Elasticsearch)
│   ├── Pool starvation / "too many connections"         → 04 §7  (HikariCP + PgBouncer)
│   ├── N+1 query                                        → 04 §1 (diagnose) (will add full deep-dive)
│   ├── 800ms latency at peak                            → 04 §1  (diagnosis ladder)
│   └── GC pauses / OOM                                  → (Q10 in roadmap)
│
├── AVAILABILITY / SURVIVING FAILURE
│   ├── Downstream API hangs                             → 02 §4  (Timeouts)
│   ├── Transient 5xx / timeout from partner             → 02 §2-3 (Retry with backoff)
│   ├── Cascading failure (one slow API kills you)       → 02 §5  (Circuit breaker)
│   ├── Slow downstream eating threads                   → 02 §6  (Bulkhead)
│   ├── Retry after SSL handshake / 4xx                  → 02 §8  (Selective retry)
│   └── Partner retry bug → 100× spike                   → 05     (Rate limit + idempotency)
│
├── ABUSE / TRAFFIC SURGE
│   ├── One tenant dominates traffic                     → 05 §3  (Per-tenant rate limit)
│   ├── Need to back off clients                         → 05 §4  (429 + Retry-After)
│   ├── Daily/monthly fair use                           → 05 §5  (Quotas)
│   ├── Healthy customers starving                       → 05 §6-7 (Bulkhead + priority)
│   ├── Emergency stop a bad client                      → 05 §11 (Kill-switch)
│   └── Block at the edge                                → 05 §9  (API Gateway)
│
├── CODE QUALITY (SonarQube, complexity, OOP)
│   ├── Cyclomatic complexity > 10                       → 03 §2  (Guard clauses first)
│   ├── Per-type if-else ladder                          → 03 §3  (Strategy + Factory)
│   ├── Pipeline of validations                          → 03 §3  (Chain of Responsibility)
│   ├── Composable boolean rules                         → 03 §3  (Specification)
│   ├── 50+ rules, business edits                        → 03 §3  (Rule Engine / Drools)
│   └── Deeply nested ifs                                → 03 §2  (Guard clauses)
│
└── CONCURRENCY (multi-thread / async)
    ├── count++ wrong                                    → 01 §1  (Atomic / synchronized)
    ├── volatile vs atomic confusion                     → 01 §2.2-2.3
    ├── ReadWriteLock for cache                          → 01 §2.5
    ├── Producer-Consumer                                → 01 §6.1 (BlockingQueue)
    ├── Safe singleton                                   → 01 §6.2 (DCL or Holder)
    ├── Singleflight (one DB load, many waiters)         → 01 §6.3 (computeIfAbsent)
    ├── Thread pool sizing                               → 01 §5
    └── Deadlock / livelock / starvation                 → 01 §4
```

---

## THE 6 FILES AT A GLANCE

| # | File | Mental Anchor | LinkedIn Q Solved |
|---|------|---------------|-------------------|
| 01 | Concurrency-And-Race-Conditions | "Shared mutable state + threads = synchronize" | — (foundation) |
| 02 | Spring-Boot-Resilience-Patterns | "NEVER retry without a timeout" | Selective Retry on Network Errors |
| 03 | Refactoring-Cyclomatic-Complexity | "New rule = new class, not new branch" | processPayment() SonarQube refactor |
| 04 | Scaling-Spring-Boot-APIs | "Measure → Cache → Bloom → Replicas → Async" | User Search @ 50M users |
| 05 | Rate-Limiting-And-Abuse-Protection | "Stop abuse at the cheapest layer" | API Went Viral Overnight |
| 06 | Read-Replica-Lag-And-Consistency | "Standard metrics LIE about correctness" | Why Does My API Return Stale Data |

---

## SYMPTOM → PATTERN — THE 30-SECOND LOOKUP

| If you see... | Reach for... | Where |
|---|---|---|
| "Retry storm" | Exponential backoff + jitter | 02 §3 |
| "Hanging threads" | Explicit timeouts + bulkhead | 02 §4, §6 |
| "Cascading failure" | Circuit breaker | 02 §5 |
| "Should retry this 5xx but not 4xx" | Selective retry by exception | 02 §8 |
| "Charged twice" | Idempotency key | 02 §9 |
| "CC > 10, SonarQube blocks merge" | CoR + Strategy + extract method | 03 |
| "Switch on type" | Polymorphism | 03 §2 |
| "Pipeline of checks" | Chain of Responsibility | 03 §3 |
| "Repeated reads" | Caffeine + Redis 2-level cache | 04 §3 |
| "40% lookups for non-existent" | Bloom filter | 04 §4 |
| "DB CPU 80%+" | Read replicas | 04 §5 |
| "LIKE '%x%' is slow" | pg_trgm GIN or Elasticsearch | 04 §6 |
| "Pool exhausted" | HikariCP tuning + PgBouncer | 04 §7 |
| "Need millions of concurrent requests" | Virtual threads / WebFlux | 04 §8 |
| "One tenant ruining it for all" | Per-tenant rate limit (Bucket4j) | 05 §3 |
| "Need to tell client to back off" | 429 + Retry-After + X-RateLimit-* | 05 §4 |
| "Daily fair use cap" | Quotas | 05 §5 |
| "Priority customers starving" | Per-tenant bulkhead + priority shedding | 05 §6-7 |
| "Need to block a client NOW" | Kill-switch at API Gateway | 05 §11 |
| "User sees old data after PUT" | Return updated obj + route to primary 2s | 06 §4, §7 |
| "Strict RYW guarantee" | LSN-aware reads | 06 §5 |
| "Password / money critical write" | synchronous_commit = remote_apply | 06 §8 |
| "Need to catch correctness bugs in prod" | RYW correctness probe | 06 §10 |
| "Cache returns stale after replica catchup" | Invalidate on write + cache from primary | 06 §11 |
| "count++ wrong" | AtomicInteger | 01 §1 |
| "Singleton thread-safe" | Holder idiom (better than DCL) | 01 §6.2 |
| "Cache stampede" | computeIfAbsent (singleflight) | 01 §6.3 |
| "Two threads acquire locks in different order" | Global lock ordering | 01 §4 |

---

## THE 7 INTERVIEW CHEAT CODES (memorize these)

Drop these phrases verbatim — they signal seniority instantly.

> **1. "Measure first."**
> "Before optimizing I'd add Micrometer + Grafana to see DB CPU, p99 latency, and cache hit rate. Optimizing without measurement is gambling."

> **2. "Stop the bleeding at the cheapest layer."**
> "Block abuse at the API Gateway — costs ~$0. Blocking at the DB costs everything. This is why I never scale up to absorb abuse — that just costs 10× to fail the same way."

> **3. "Standard metrics lie about correctness."**
> "Error rate = 0% with users seeing wrong data is the worst kind of bug. I add correctness probes — write a marker, read it back, count mismatches — and alert on those."

> **4. "Retry only idempotent operations."**
> "Without idempotency, retry is a double-charge waiting to happen. Idempotency keys make retries free instead of catastrophic."

> **5. "NEVER retry without a timeout and backoff with jitter."**
> "Without jitter, 1000 clients retry at exactly 1s, 2s, 4s — a synchronized retry storm that destroys the recovering service."

> **6. "New rule = new class, not new branch."**
> "Adding a new currency or validator should be adding a Spring @Component, not modifying processPayment(). That's Open/Closed Principle in practice."

> **7. "Read-your-own-writes is the default user expectation."**
> "Users tolerate other people's data being stale for a second. They don't tolerate their own update vanishing."

---

## THE UNIVERSAL DIAGNOSIS LADDER

When given a "scale this system" or "this is slow" problem, walk this ladder in order:

```
1. WHO is using it?          → per-tenant rate distribution
2. WHAT is the bottleneck?    → DB CPU, pool, GC, network, downstream
3. CAN WE CACHE?              → hot key + cache-aside (Caffeine + Redis)
4. CAN WE SKIP THE LOOKUP?    → Bloom filter for negative cases
5. CAN WE PARALLELIZE READS?  → read replicas (with RYW guards!)
6. CAN WE FAIL FAST?          → timeout + circuit breaker on downstream
7. CAN WE LIMIT INPUT?        → rate limit per tenant, pagination
8. CAN WE ISOLATE?            → bulkhead per tenant, priority shed
9. CAN WE RETRY SAFELY?       → idempotency + exponential backoff + jitter
10. CAN WE SCALE OUT?         → HPA, but ONLY for healthy demand
```

**The order matters.** Engineers who reach for "scale out" before "cache" or "rate limit" lose the interview.

---

## "WHAT WOULD YOU DO FIRST?" — THE INCIDENT PLAYBOOK

```
EVERY PRODUCTION INCIDENT, IN ORDER:

1. STOP THE BLEEDING       (5 min)  → kill switch / feature flag / rollback
2. MITIGATE                (15 min) → block abuser, scale up, drain pool
3. DIAGNOSE                (30 min) → logs, traces, metrics, dashboards
4. PATCH                   (1-4 hr) → smallest fix that addresses root cause
5. POSTMORTEM              (next day) → why didn't tests / monitoring catch it?
6. PREVENT                 (next week) → make this class of bug impossible
```

> A senior engineer's answer to "production is down" is NEVER "let me read the code". It's "what's our kill-switch?"

---

## TEMPLATE FOR ANSWERING ANY "SCALING / RELIABILITY" QUESTION

Use this 5-step structure in interviews:

```
1. RESTATE & CLARIFY (30s)
   "Just to confirm: you have X RPS, Y constraint, Z SLA. Is read-heavy
   or write-heavy? What's the current bottleneck?"

2. DIAGNOSE (1 min)
   "The killer signal here is [specific observation]. That tells me
   this isn't a [generic problem] — it's a [specific problem]."

3. SOLUTION IN LAYERS (3 min)
   "I'd fix in layers, cheapest first:
    - First: [smallest, lowest-risk fix]
    - Second: [next layer of defense]
    - Third: [longer-term structural fix]
    - Fourth: [observability so we catch the next one]"

4. TRADE-OFFS (1 min)
   "The trade-off here is [X]. I'd choose [option] because [reason
   tied to constraints from step 1]."

5. WHAT WOULD GO WRONG (1 min)
   "Two things I'd watch out for: [pitfall 1] and [pitfall 2].
   I'd add specific alerts for both."
```

---

## RELATED NOTES OUTSIDE THIS FOLDER

| Topic | Where |
|---|---|
| System design (microservices, kafka, etc.) | `System Design/md/Notes-Detailed/` |
| DSA patterns | `DSA/notes/md/` |
| Two pointers conditions (< vs <=) | `DSA/notes/md/Two Pointers.md` (PART 2.6) |
| Production issues (SD level) | `System Design/.../Fundamentals/24-Production-Issues-And-Solutions.md` |
| Microservices resilience (SD level) | `System Design/.../Microservices-Architecture/03-Resilience-And-Deployment.md` |
| Idempotency deep-dive | `System Design/.../Fundamentals/22-Idempotent-API-Design.md` |
| Distributed concurrency control | `System Design/.../Fundamentals/21-Distributed-Concurrency-Control.md` |

---

## QUICK SELF-TEST (5 min)

If you can answer these in <30s each, you're interview-ready on this folder:

1. What's the difference between `volatile` and `AtomicInteger`?
2. When do you use `left < right` vs `left <= right`?
3. Why is `Executors.newFixedThreadPool(n)` dangerous?
4. What's the order of decorators in Resilience4j?
5. Why does `@Retryable` silently fail when called from the same class?
6. What are the four conditions for deadlock (Coffman)?
7. How does a Bloom filter save 40% of DB lookups?
8. Why is `synchronous_commit = remote_apply` slow?
9. What headers MUST a 429 response carry?
10. What's the cache stampede pattern and how do you prevent it?

If any of these is shaky, jump to the relevant file/section above and refresh.

---

## NEXT TOPICS TO ADD (when LinkedIn screenshots arrive)

These are the gaps I'd fill next:

| Priority | Topic | Why it matters |
|---|---|---|
| HIGH | Transactional Outbox + dual-write problem | Most asked microservices pattern 2024-2026 |
| HIGH | Kafka consumer lag + head-of-line blocking | Universal at Kafka-using companies |
| HIGH | OOM debugging in production | Separates senior from staff |
| MED | N+1 + JPA EntityGraph | Most asked Spring Data question |
| MED | Zero-downtime DB migrations | Expand-contract pattern |
| MED | OAuth token refresh race | Singleflight for token APIs |
| MED | Username enumeration / timing attacks | Security questions are increasing |
| LOW | LLD problems (Parking Lot, Rate Limiter, Cache) | When asked specifically |
| LOW | Java Streams deep-dive | Trivia, but common |
| LOW | JVM internals (GC, memory model) | Senior+ only |
