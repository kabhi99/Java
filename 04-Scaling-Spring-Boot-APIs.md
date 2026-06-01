# **SCALING SPRING BOOT APIs — PRODUCTION PATTERNS**

How to take a Spring Boot REST API from 1K RPS / 800ms latency → 100K RPS / <50ms latency. Patterns, code, and the "why" behind each.

### TABLE OF CONTENTS

1. The Scaling Diagnosis Checklist (always start here)
2. The 7 Levers of API Performance
3. **PATTERN 1: Multi-Level Caching (Caffeine + Redis)**
4. **PATTERN 2: Bloom Filter (Negative Lookup Optimization)**
5. **PATTERN 3: Read Replicas (Read/Write Splitting)**
6. **PATTERN 4: Search Index (Elasticsearch / pg_trgm)**
7. **PATTERN 5: Connection Pool Tuning (HikariCP)**
8. **PATTERN 6: Async / Non-Blocking (CompletableFuture, WebFlux)**
9. **PATTERN 7: Rate Limiting + Pagination**
10. **FEATURED PROBLEM: User Search Endpoint @ 50M users (LinkedIn Q)**
11. Golden Rules & Interview Cheat Codes

---

## PART 1: THE SCALING DIAGNOSIS CHECKLIST

Before optimizing, **measure**. Symptoms → likely cause:

```
+----------------------------------+----------------------------------------+
| Symptom                          | Likely Cause                           |
+----------------------------------+----------------------------------------+
| DB CPU > 80%                     | Missing index / no cache / N+1 query   |
| App CPU > 80%                    | Bad algo / serialization / GC pressure |
| High p99 latency                 | DB slow query / cache miss storm       |
| Many duplicate requests          | No cache                               |
| Many 404 / not-found responses   | No negative cache / no Bloom filter    |
| Connection pool exhausted        | Pool too small / slow downstream       |
| OOM / GC pauses                  | Unbounded queue / cache / response     |
| Latency = network + DB           | No connection pool, no keep-alive      |
| Spiky latency at intervals       | GC, cache invalidation, or batch jobs  |
| Reads >> Writes                  | Need read replicas + caching           |
| Hot keys                         | Need consistent hashing / partitioning |
+----------------------------------+----------------------------------------+
```

 **GOLDEN RULE**: "Measure twice, optimize once." Use Micrometer + Grafana before guessing.

---

## PART 2: THE 7 LEVERS OF API PERFORMANCE

```
+--------------------------+--------------------------------------------------+
|  Lever                   | Typical gain                                     |
+--------------------------+--------------------------------------------------+
| 1. Cache (in-mem + dist) | 100x-1000x (skip DB entirely)                    |
| 2. Bloom filter          | Save 30-50% of cache/DB hits on misses           |
| 3. Read replicas         | 5-10x read throughput                            |
| 4. Index / search engine | 100x on filter/search queries                    |
| 5. Connection pool       | Stable latency under load (no pool starvation)   |
| 6. Async / non-blocking  | 5-10x throughput per thread                      |
| 7. Horizontal scale + LB | Linear scaling (until DB becomes bottleneck)     |
+--------------------------+--------------------------------------------------+
```

**Apply in this order** (don't skip ahead — measure after each).

---

## PART 3: MULTI-LEVEL CACHING (Caffeine + Redis)

> The single biggest performance win for any read-heavy API. Skip the DB entirely for hot data.

### **Why 2 levels?**

| Cache | Latency | Capacity | Shared across pods? |
|---|---|---|---|
| **L1: Caffeine** (JVM heap) | ~100ns | MB (limited by RAM) | No — local |
| **L2: Redis** (network) | 0.5-2ms | GB-TB | Yes — distributed |
| **L3: PostgreSQL** | 10-100ms | TB | Yes |

> L1 absorbs the hottest keys (e.g., Justin Bieber's profile); L2 absorbs the long tail.

### **Setup**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### **Cache-Aside Pattern (most common)**

```java
@Service
public class UserService {

    private final UserRepository repo;
    private final Cache<String, User> l1;            // Caffeine
    private final RedisTemplate<String, User> l2;    // Redis

    public Optional<User> findByUsername(String username) {
        // L1 — JVM local
        User u = l1.getIfPresent(username);
        if (u != null) return Optional.of(u);

        // L2 — Redis
        u = l2.opsForValue().get("user:" + username);
        if (u != null) {
            l1.put(username, u);                     // backfill L1
            return Optional.of(u);
        }

        // L3 — DB
        Optional<User> dbUser = repo.findByUsername(username);
        dbUser.ifPresent(user -> {
            l1.put(username, user);
            l2.opsForValue().set("user:" + username, user, Duration.ofMinutes(10));
        });
        return dbUser;
    }
}
```

### **Caffeine config**

```java
@Bean
public Cache<String, User> userL1Cache() {
    return Caffeine.newBuilder()
        .maximumSize(10_000)                    // bounded! prevents OOM
        .expireAfterWrite(Duration.ofMinutes(5))
        .recordStats()                          // Micrometer metrics
        .build();
}
```

### **Critical gotchas**

| Gotcha | Fix |
|---|---|
| **Cache stampede** (10K threads miss → all hit DB) | `computeIfAbsent` (atomic) or `singleflight` pattern |
| **Stale data on update** | Invalidate L1 + L2 on write, OR use short TTL |
| **L1 inconsistent across pods** | Short L1 TTL (1-5 min) + pub/sub invalidation |
| **Unbounded cache → OOM** | ALWAYS set `maximumSize` |
| **Caching `null` to avoid repeated lookups** | Yes! Cache the negative result too (or use Bloom filter) |

### **Cache Stampede Protection**

```java
// computeIfAbsent ensures only ONE thread does the DB call per key
public User findByUsername(String username) {
    return l1.get(username, key -> {
        User cached = l2.opsForValue().get("user:" + key);
        if (cached != null) return cached;
        User u = repo.findByUsername(key).orElse(null);
        if (u != null) l2.opsForValue().set("user:" + key, u, Duration.ofMinutes(10));
        return u;
    });
}
```

---

## PART 4: BLOOM FILTER (Negative Lookup Optimization)

> **The killer feature for "40% queries are for non-existent items".** Probabilistic data structure that says "definitely doesn't exist" or "maybe exists" — in O(1) memory.

### **Why it works**

- **No false negatives** — if Bloom says "doesn't exist", it really doesn't (skip DB, return 404 immediately)
- **False positives possible** — if Bloom says "maybe", you still check the cache/DB (so correctness is preserved)
- **Tiny memory** — 50M usernames at 1% false positive = ~60MB (vs ~5GB to cache them all)

### **Setup (Guava)**

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
</dependency>
```

```java
@Component
public class UsernameBloomFilter {

    private BloomFilter<String> filter;

    @PostConstruct
    public void init() {
        // Sized for 50M users with 1% false positive rate
        filter = BloomFilter.create(
            Funnels.stringFunnel(StandardCharsets.UTF_8),
            50_000_000,
            0.01
        );
        // Warm-up: load all existing usernames at startup
        userRepo.streamAllUsernames().forEach(filter::put);
    }

    public boolean mightExist(String username) {
        return filter.mightContain(username);
    }

    public void add(String username) {
        filter.put(username);                       // call on user creation
    }
}
```

### **Use in the service**

```java
public Optional<User> findByUsername(String username) {
    // FAST FAIL — skip cache + DB entirely
    if (!bloomFilter.mightExist(username)) {
        return Optional.empty();                    // saves ~3-4 ms per request
    }
    // existing cache-aside flow...
}
```

### **For distributed setups: Redis Bloom (RedisBloom module)**

```java
// All pods share the same Bloom filter
redisTemplate.execute((RedisCallback<Object>) conn ->
    conn.execute("BF.ADD", "usernames".getBytes(), username.getBytes()));

boolean mightExist = (Long) redisTemplate.execute((RedisCallback<Object>) conn ->
    conn.execute("BF.EXISTS", "usernames".getBytes(), username.getBytes())) == 1;
```

### **Gotchas**

| Gotcha | Fix |
|---|---|
| **Can't remove items** from standard Bloom filter | Use **Counting Bloom Filter** or **Cuckoo Filter** |
| **Filter rebuild on restart** is expensive | Persist to Redis, or use RedisBloom |
| **Forgot to update on new user creation** | Bloom returns false → 404 for real user! Always `.put()` on insert |
| **Sizing wrong** → high false positive rate | Plan for 2x current data |

---

## PART 5: READ REPLICAS (Read/Write Splitting)

> When DB CPU > 80% and reads >> writes, add replicas. Writes go to primary, reads to replicas.

### **Spring AbstractRoutingDataSource**

```java
public enum DbRole { PRIMARY, REPLICA }

public class RoutingDataSource extends AbstractRoutingDataSource {
    protected Object determineCurrentLookupKey() {
        return DbContext.getRole();
    }
}

public class DbContext {
    private static final ThreadLocal<DbRole> ctx = ThreadLocal.withInitial(() -> DbRole.PRIMARY);
    public static void setRole(DbRole r) { ctx.set(r); }
    public static DbRole getRole() { return ctx.get(); }
    public static void clear() { ctx.remove(); }
}

// Annotation + AOP
@Retention(RUNTIME) @Target(METHOD)
public @interface ReadOnly {}

@Aspect @Component
public class ReadOnlyAspect {
    @Around("@annotation(ReadOnly)")
    public Object route(ProceedingJoinPoint pjp) throws Throwable {
        DbContext.setRole(DbRole.REPLICA);
        try { return pjp.proceed(); }
        finally { DbContext.clear(); }
    }
}

// Usage
@ReadOnly
public Optional<User> findByUsername(String u) { return repo.findByUsername(u); }
```

### **Easier alternative — Spring's `@Transactional(readOnly = true)`**

Combined with a routing data source that reads the `readOnly` flag — many ORMs (Hibernate) and connection pools (HikariCP) support this.

### **Gotcha: Read-After-Write Consistency**

If you write, then immediately read the same data from a replica, you may see stale data (replication lag).
**Fix**: For "read your own writes", route to primary for N seconds after a write (sticky reads).

---

## PART 6: SEARCH INDEX (when SQL `LIKE` isn't enough)

> Username search is often **prefix** (`LIKE 'just%'`) or **fuzzy** (`Juztin` → `Justin`). PostgreSQL `LIKE` doesn't use an index for `%X%` patterns.

### **Option A: PostgreSQL `pg_trgm` (good enough for most cases)**

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX users_username_trgm_idx ON users USING gin (username gin_trgm_ops);

-- Now fuzzy/prefix search uses the index
SELECT * FROM users WHERE username ILIKE '%just%' LIMIT 20;
SELECT * FROM users WHERE username % 'juztin' ORDER BY similarity(username, 'juztin') DESC LIMIT 10;
```

### **Option B: Elasticsearch (Twitter/Instagram scale)**

```java
@Document(indexName = "users")
public class UserDoc {
    @Id private String id;
    @Field(type = FieldType.Text, analyzer = "edge_ngram")  // for prefix
    private String username;
    @Field(type = FieldType.Keyword)
    private String region;
}

@Service
public class UserSearchService {
    private final ElasticsearchOperations es;

    public List<UserDoc> searchByPrefix(String prefix, int limit) {
        Query q = NativeQuery.builder()
            .withQuery(QueryBuilders.matchPhrasePrefix(m -> m.field("username").query(prefix)))
            .withPageable(PageRequest.of(0, limit))
            .build();
        return es.search(q, UserDoc.class).stream().map(SearchHit::getContent).toList();
    }
}
```

**Keep DB as source of truth, sync to ES via**:
- Spring `@PostPersist` listener (simple, but lossy on failure)
- Debezium CDC from Postgres WAL (robust, eventually consistent)
- Outbox pattern (transactional)

---

## PART 7: CONNECTION POOL TUNING (HikariCP)

> The connection pool is often the silent bottleneck. Default `maximum-pool-size=10` is wrong for 10K RPS.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50              # ~ CPU cores * 2-4 on DB
      minimum-idle: 10
      connection-timeout: 2000           # ms to wait for a conn
      idle-timeout: 600000
      max-lifetime: 1800000              # rotate before LB closes
      leak-detection-threshold: 60000    # log if held > 60s
```

### **Sizing formula** (rule of thumb)
```
pool_size = (cores_on_DB * 2) + effective_spindle_count
```
- 8-core DB, SSD → ~16-20 connections per app pod
- Total across all pods should not exceed DB's `max_connections`

### **The "too many connections" gotcha**
50 pods × 50 conns = 2500 conns to Postgres → blown.
**Fix**: Use **PgBouncer** (transaction pooler) between app and DB to multiplex.

---

## PART 8: ASYNC / NON-BLOCKING

> Each blocked thread waiting on DB/HTTP holds ~1MB of stack. 200 threads = 200MB just for waiting.

### **`@Async` (simple)**

```java
@EnableAsync
@Configuration
public class AsyncConfig {
    @Bean(name = "ioExecutor")
    public Executor ioExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(20);
        ex.setMaxPoolSize(100);
        ex.setQueueCapacity(500);
        ex.setThreadNamePrefix("io-");
        return ex;
    }
}

@Async("ioExecutor")
public CompletableFuture<User> fetchAsync(String id) { ... }
```

### **WebFlux (full non-blocking)**

```java
@RestController
public class UserController {
    @GetMapping("/users/{username}")
    public Mono<User> get(@PathVariable String username) {
        return userService.findByUsername(username);   // returns Mono<User>
    }
}
```

WebFlux = 1 Netty event-loop thread per CPU handles 100K+ concurrent connections.

### **Java 21 Virtual Threads (game changer)**

```yaml
spring:
  threads:
    virtual:
      enabled: true        # Spring Boot 3.2+
```

Now `@Async` and Tomcat threads are virtual — millions of concurrent threads, 1KB each. **Use this if on Java 21+ and Spring Boot 3.2+.**

---

## PART 9: RATE LIMITING + PAGINATION

### **Rate limiting (Resilience4j)**

```yaml
resilience4j:
  ratelimiter:
    instances:
      userSearch:
        limitForPeriod: 100         # 100 req
        limitRefreshPeriod: 1s      # per sec per user/IP
        timeoutDuration: 100ms
```

Better: enforce at **API Gateway** (Kong, Spring Cloud Gateway, AWS API Gateway) so backend pods never even see the rejected traffic.

### **Mandatory pagination**

```java
@GetMapping("/users/search")
public Page<UserDto> search(
    @RequestParam String q,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") @Max(100) int size
) {
    return userService.search(q, PageRequest.of(page, size));
}
```

NEVER allow unbounded results. Always cap `size` (e.g., 100). For deep pagination, use **cursor-based** (`after=lastId`) instead of `offset/limit`.

---

## PART 10: FEATURED PROBLEM — USER SEARCH @ 50M USERS

### **THE PROBLEM** (LinkedIn interview question)

> Social media platform, 50M users. Search-by-username endpoint backed by Postgres.
>
> **Production metrics after launch:**
> - 10K RPS at peak
> - DB CPU consistently > 85%
> - Avg latency ~800ms (target <100ms)
> - **40% searches are for usernames that don't exist**
> - Many searches repeat the same popular usernames
>
> Scale this for production traffic.

### **DIAGNOSIS** (read the symptoms!)

| Signal | Inference |
|---|---|
| 40% don't-exist searches | → **Bloom filter** is THE killer fix |
| Repeated popular searches | → **Caching** (Caffeine + Redis) |
| DB CPU 85% | → Reduce DB load via cache + replicas |
| 800ms latency | → DB-bound; cache hits return in <10ms |
| Username search | → Need proper index (`pg_trgm`) or Elasticsearch for prefix/fuzzy |
| 10K RPS | → Need horizontal scale + connection pooling |

### **THE FULL SOLUTION (layered)**

```
                    [API Gateway: rate limit, auth]
                              ↓
        [Load Balancer]  →  [App Pod 1 ... App Pod N]
                              ↓
              ┌──────────────────────────────────┐
              │  Bloom Filter (in-memory or       │  ← skip 40% of misses
              │   RedisBloom across pods)          │
              └──────────────────────────────────┘
                              ↓
              ┌──────────────────────────────────┐
              │  L1: Caffeine (per-pod, 10K hot)   │  ← <1ms
              └──────────────────────────────────┘
                              ↓
              ┌──────────────────────────────────┐
              │  L2: Redis (shared, 1M hot)        │  ← ~1ms
              └──────────────────────────────────┘
                              ↓
              ┌──────────────────────────────────┐
              │  Postgres Read Replicas             │  ← ~10ms
              │   (with pg_trgm GIN index)          │
              └──────────────────────────────────┘
                              ↓
                  [Postgres Primary]  (writes only)
                              │
                              │  CDC (Debezium)
                              ↓
                  [Elasticsearch]  (for fuzzy/prefix search)
```

### **CODE**

```java
@Service
public class UserSearchService {

    private final UsernameBloomFilter bloom;
    private final Cache<String, User> l1;
    private final RedisTemplate<String, User> redis;
    private final UserRepository repo;
    private final MeterRegistry metrics;

    public Optional<User> findByUsername(String username) {
        // 1. Bloom filter — eliminates 40% of misses for FREE
        if (!bloom.mightExist(username)) {
            metrics.counter("user.lookup.bloom.miss").increment();
            return Optional.empty();
        }

        // 2. L1 cache (Caffeine, ~100ns)
        User u = l1.getIfPresent(username);
        if (u != null) {
            metrics.counter("user.lookup.l1.hit").increment();
            return Optional.of(u);
        }

        // 3. L2 cache (Redis, ~1ms)
        u = redis.opsForValue().get("user:" + username);
        if (u != null) {
            l1.put(username, u);
            metrics.counter("user.lookup.l2.hit").increment();
            return Optional.of(u);
        }

        // 4. DB (read replica)
        return loadFromDb(username);
    }

    @ReadOnly  // routes to replica via routing data source
    private Optional<User> loadFromDb(String username) {
        Optional<User> result = repo.findByUsername(username);
        result.ifPresent(u -> {
            l1.put(username, u);
            redis.opsForValue().set("user:" + username, u, Duration.ofMinutes(10));
        });
        metrics.counter("user.lookup.db.hit").increment();
        return result;
    }

    // Keep Bloom + caches consistent on writes
    @EventListener
    public void onUserCreated(UserCreatedEvent e) {
        bloom.add(e.username());
    }

    @EventListener
    public void onUserUpdated(UserUpdatedEvent e) {
        l1.invalidate(e.username());
        redis.delete("user:" + e.username());
    }
}
```

### **EXPECTED IMPROVEMENT**

| Metric | Before | After |
|---|---|---|
| Avg latency | 800ms | ~5-50ms |
| DB CPU | 85% | <30% |
| DB QPS | 10K | ~500 (cache absorbs 95%) |
| 40% nonexistent searches | Hit DB | Bloom filter, <1ms |
| Repeated popular searches | Hit DB | Caffeine, <1µs |

### **INTERVIEW NARRATIVE**

> "Three signals stand out: 40% non-existent lookups, repeated popular searches, and DB CPU pinned at 85%. So I'd attack in this order:
>
> **First** — a **Bloom filter** to short-circuit the 40% non-existent lookups before they even touch Redis or the DB. For 50M users at 1% false positive that's ~60MB of memory per pod.
>
> **Second** — a **two-level cache** (Caffeine in-JVM for the hottest few thousand keys, Redis for the long tail) using a cache-aside pattern with `computeIfAbsent` to prevent stampedes.
>
> **Third** — **read/write splitting** with a routing data source so all reads hit Postgres replicas; primary only sees writes.
>
> **Fourth** — for the actual search, swap `LIKE` for a `pg_trgm` GIN index, or move to Elasticsearch if the product wants typo-tolerant search.
>
> **Fifth** — tune **HikariCP** (size ~ 2 × DB cores per pod), put **PgBouncer** in front to multiplex, enforce **rate limiting** at the gateway, and **paginate** with a hard size cap.
>
> If running Java 21 + Spring Boot 3.2, enable **virtual threads** to absorb concurrency cheaply.
>
> After this, the typical latency should drop from 800ms to <50ms, DB CPU well under 30%, and the architecture scales horizontally."

---

## PART 11: GOLDEN RULES

```
+----------------------------------------------------------------------+
|  1. Measure first — don't guess where the bottleneck is              |
|  2. Cache is the #1 lever for read-heavy APIs                        |
|  3. Bloom filter when ≥ 20% of lookups are for non-existent items    |
|  4. 2-level cache (L1 local + L2 distributed) for hot keys           |
|  5. ALWAYS bound caches (maximumSize) → no OOM                       |
|  6. Use computeIfAbsent / singleflight to prevent cache stampedes    |
|  7. Reads >> writes? Read replicas + routing data source             |
|  8. LIKE '%x%' kills RDBMS → use pg_trgm or Elasticsearch            |
|  9. Tune HikariCP — default size 10 is wrong for any real workload   |
| 10. Use PgBouncer when total app pods × pool > DB max_connections    |
| 11. Hard cap pagination size (e.g., 100); prefer cursor over offset  |
| 12. Rate limit at the API Gateway, not at every backend pod          |
| 13. Async / virtual threads to scale concurrency without thread count|
| 14. Invalidate caches on writes (or use short TTLs)                  |
| 15. Emit Micrometer metrics for every cache hit/miss                 |
+----------------------------------------------------------------------+
```

---

## PART 12: INTERVIEW CHEAT CODES

> "I always **measure first** — Micrometer + Grafana — before optimizing. Three signals tell me everything: DB CPU, p99 latency, and cache hit rate."

> "When ≥ 20% of lookups are for non-existent items, a **Bloom filter** is the killer fix — it's the only data structure that lets you skip the entire cache+DB lookup for a 'maybe' answer."

> "I prefer a **two-level cache**: Caffeine in-JVM for the hottest thousand keys (sub-microsecond) and Redis as the shared L2 (sub-millisecond). L1 absorbs the celebrity-effect hot keys, L2 absorbs the long tail."

> "Cache-aside has a classic gotcha — **cache stampede** when 1000 threads simultaneously miss the same key. I use `computeIfAbsent` (Caffeine) or the singleflight pattern to ensure only one thread computes."

> "Read replicas in Spring are clean with `AbstractRoutingDataSource` + `@ReadOnly` AOP — annotate read methods and Spring routes them to replicas."

> "For username search, `LIKE '%x%'` skips indexes. Either add a **`pg_trgm` GIN index** (good to ~100M rows) or move search to **Elasticsearch** synced via Debezium CDC."

> "The default HikariCP `maximum-pool-size=10` is wrong for any real workload. Rule of thumb: **2 × DB cores per pod**. And put **PgBouncer** in front when total connections approach Postgres's `max_connections`."

> "If on Java 21 + Spring Boot 3.2+, I enable **virtual threads** — Tomcat and `@Async` get massive concurrency for free, no WebFlux rewrite needed."

---

## QUICK REVISION TABLE

```
+-------------------------+---------------------------+-------------------+
| Problem                 | Fix                       | Tool              |
+-------------------------+---------------------------+-------------------+
| Repeated reads          | Cache                     | Caffeine + Redis  |
| Lookups for non-existent| Bloom filter              | Guava / RedisBloom|
| DB CPU saturated        | Read replicas             | RoutingDataSource |
| Slow LIKE search        | Trigram index / ES        | pg_trgm / Elastic |
| Pool starvation         | Tune HikariCP + PgBouncer | HikariCP/PgBouncer|
| Thread starvation       | Async / virtual threads   | @Async / Java 21  |
| Abusive clients         | Rate limit                | API Gateway / R4j |
| Unbounded result sets   | Pagination + size cap     | Pageable          |
| Cache stampede          | computeIfAbsent           | Caffeine          |
| Stale cache after write | Invalidate on event       | @EventListener    |
| Hot key (celebrity)     | L1 + request coalescing   | Caffeine          |
| Connection limit blown  | PgBouncer in front of PG  | PgBouncer         |
+-------------------------+---------------------------+-------------------+
```
