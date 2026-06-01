# **ASYNC, CONCURRENCY & REACTIVE**

8 questions on `@Async`, `CompletableFuture`, WebFlux, virtual threads.

---

### Q01: How does `@Async` work and what are the gotchas? ⭐

**30-Second Answer:** `@Async` makes a method run on a different thread (returns immediately). Spring wraps the bean in a proxy that submits the call to a `TaskExecutor`. **Gotchas**: needs `@EnableAsync`; **doesn't work on self-invocation** (same-class call bypasses proxy); return type must be `void`, `Future`, or `CompletableFuture`.

**Deep Dive:**

```java
@Configuration
@EnableAsync   //  REQUIRED
public class AsyncConfig {

    @Bean(name = "emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(5);
        ex.setMaxPoolSize(20);
        ex.setQueueCapacity(500);                     //  bounded
        ex.setThreadNamePrefix("email-");             //  debugging
        ex.setRejectedExecutionHandler(new CallerRunsPolicy());   //  backpressure
        return ex;
    }
}

@Service
public class EmailService {
    @Async("emailExecutor")
    public CompletableFuture<Void> send(String to, String body) {
        // runs on emailExecutor thread, returns immediately to caller
        emailClient.send(to, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

**Gotchas:**
- **Self-invocation**: `this.asyncMethod()` bypasses proxy — runs synchronously, silently
- **`SecurityContext` lost**: Use `DelegatingSecurityContextExecutor`
- **Exceptions lost in `void` async methods**: Set `AsyncUncaughtExceptionHandler`
- **Default executor is `SimpleAsyncTaskExecutor`** — creates new thread every time → thread explosion

```java
//  Custom exception handler for void async methods
@Override
public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) -> log.error("Async error in {}", method.getName(), ex);
}
```

**Common Follow-Ups:**
- "What's the default executor?" → `SimpleAsyncTaskExecutor` — never use in prod (no thread reuse)
- "How do you wait for multiple async calls?" → `CompletableFuture.allOf(f1, f2, f3).join()`

**Red Flag If You Say:** "Just add `@Async` and Spring handles everything." (Missing executor config → SimpleAsyncTaskExecutor → unbounded threads → OOM)

---

### Q02: `CompletableFuture` deep dive — `thenApply` vs `thenCompose` vs `thenCombine`? ⭐

**30-Second Answer:**
- **`thenApply(Function)`** — transform result (`A → B`)
- **`thenCompose(Function)`** — chain ANOTHER async call (`A → CompletableFuture<B>`); avoids `Future<Future<B>>` nesting
- **`thenCombine(other, BiFunction)`** — combine result of THIS with ANOTHER independent future

**Deep Dive:**

```java
//  thenApply — synchronous transformation
CompletableFuture<User> userF = fetchUser(id);
CompletableFuture<String> nameF = userF.thenApply(User::getName);

//  thenCompose — chain another async (like flatMap)
CompletableFuture<List<Order>> ordersF = fetchUser(id)
    .thenCompose(user -> fetchOrders(user.getId()));   //  returns CF<List<Order>>, not CF<CF<List<Order>>>

//  thenCombine — combine two independent futures
CompletableFuture<UserProfile> profileF = fetchUser(id)
    .thenCombine(fetchPreferences(id),
        (user, prefs) -> new UserProfile(user, prefs));

//  Error handling
fetchUser(id)
    .exceptionally(ex -> { log.error("oops", ex); return User.unknown(); })   // recover
    .handle((res, ex) -> ex != null ? handleErr(ex) : process(res))            // both

//  Timeout (Java 9+)
fetchUser(id).orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> User.unknown());

//  Wait for all
CompletableFuture.allOf(f1, f2, f3).join();
// (allOf returns CF<Void>; collect results separately)

//  Race
CompletableFuture.anyOf(f1, f2).thenAccept(first -> ...);
```

**Common Follow-Ups:**
- "Async vs sync versions?" → `.thenApplyAsync(...)` runs on executor; `.thenApply(...)` runs on the completing thread (could be calling thread!)
- "When to provide an executor?" → Always — never trust default `ForkJoinPool.commonPool` for I/O

---

### Q03: When should you use reactive (WebFlux) vs imperative (MVC)? ⭐

**30-Second Answer:** **WebFlux** when you need to handle **many concurrent I/O-bound requests** with limited threads (10K+ concurrent connections, streaming, server-sent events). **MVC** for everything else — simpler model, easier to debug, blocking JDBC drivers fine. **Don't mix freely — calling blocking code in reactive flow blocks the event loop.**

**Deep Dive:**

| Aspect | MVC (Servlet) | WebFlux (Reactive) |
|---|---|---|
| Threading | 1 thread per request | Event loop (few threads) |
| Programming model | Imperative (easy) | Reactive (steep curve) |
| Debugging | Easy (stack traces) | Hard (no traditional stack) |
| Throughput at high concurrency | OK | Much better |
| Database | JDBC (blocking) | R2DBC (reactive) — limited |
| Stack | Tomcat + Servlet | Netty + Reactor |
| Use case | Most APIs | Streaming, SSE, high fan-out, BFF |

```java
//  MVC
@GetMapping("/users/{id}")
public User get(@PathVariable Long id) { return service.findById(id); }

//  WebFlux
@GetMapping("/users/{id}")
public Mono<User> get(@PathVariable Long id) { return service.findById(id); }
//  service.findById returns Mono<User>, all the way down — never .block()
```

**THE BIG TRAP — mixing:**
```java
//  Calls blocking JDBC inside reactive flow
@GetMapping("/users/{id}")
public Mono<User> get(@PathVariable Long id) {
    return Mono.fromCallable(() -> jpaRepo.findById(id))   //  blocks event loop!
        .subscribeOn(Schedulers.boundedElastic());          //  fix: move to bounded elastic
}
```

**Common Follow-Ups:**
- "Why is Java 21 virtual threads a game-changer here?" → You get MVC's simplicity with WebFlux-like concurrency. May make WebFlux less attractive for most use cases
- "When does WebFlux still win?" → Streaming, SSE, back-pressure, functional composition

**Red Flag If You Say:** "Reactive is faster, so always use it." (Reactive is for **concurrency**, not raw speed. For low-traffic APIs, MVC is simpler and just as fast.)

---

### Q04: Virtual threads (Java 21 / Project Loom) — what's the big deal? ⭐

**30-Second Answer:** Virtual threads are lightweight threads (~1KB each, millions per JVM) managed by JVM, not OS. **You get the simplicity of blocking code with the concurrency of reactive.** Just enable in Spring Boot 3.2+ and Tomcat + `@Async` use virtual threads automatically. No code changes.

**Deep Dive:**

```yaml
spring:
  threads:
    virtual:
      enabled: true       # Spring Boot 3.2+
```

```java
//  Old way — limited by OS threads
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public Order get(@PathVariable Long id) {
        return orderService.fetchFromSlowApi(id);   // blocks 1 of ~200 Tomcat threads
    }
}

//  Same code with virtual threads — handles 100K+ concurrent requests
```

**How it works:**
- Virtual thread "parks" on blocking call (file, network, lock)
- JVM mounts another virtual thread on the OS thread
- When blocking call returns, JVM remounts the original virtual thread
- Result: millions of "blocked" virtual threads, sharing few OS threads

**Pinning trap:**
```java
synchronized (lock) {
    httpCall();   //  Pins virtual thread to OS thread — defeats the purpose
}
//  Use ReentrantLock instead
ReentrantLock lock = new ReentrantLock();
lock.lock();
try { httpCall(); }
finally { lock.unlock(); }
```

**Common Follow-Ups:**
- "When NOT to use virtual threads?" → CPU-bound work (no I/O wait) — no benefit; just use ForkJoinPool
- "What about WebFlux?" → WebFlux still wins for streaming + back-pressure. For most CRUD APIs, virtual threads + MVC is simpler

---

### Q05: How do you handle concurrent updates to a Spring `@Service` singleton?

**30-Second Answer:** **Make it stateless** (no mutable instance fields) — that's the idiomatic Spring way. If state is needed: use thread-safe types (`AtomicInteger`, `ConcurrentHashMap`), explicit locks, or change scope to `prototype`/`request`.

**Deep Dive:**

```java
//  Race condition — singleton scope, shared state
@Service
public class CounterService {
    private int count = 0;
    public void increment() { count++; }   //  Not atomic
}

//  Atomic
@Service
public class CounterService {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); }
    public int get() { return count.get(); }
}

//  ConcurrentHashMap for per-key updates
@Service
public class PerUserCounter {
    private final ConcurrentMap<String, AtomicLong> counts = new ConcurrentHashMap<>();
    public void increment(String userId) {
        counts.computeIfAbsent(userId, k -> new AtomicLong()).incrementAndGet();
    }
}

//  Stateless — preferred Spring pattern
@Service
public class CounterService {
    private final CounterRepository repo;   //  state in DB, not in service
    public void increment(String userId) {
        repo.incrementCount(userId);        // atomic SQL: UPDATE ... SET count = count + 1
    }
}
```

**Common Follow-Ups:**
- "When would you use `prototype` scope?" → Stateful workflow holders (rare); usually a code smell
- "Where to put thread-safety logic?" → Inside the singleton; or DB-level constraint/atomic SQL

> See `Java/01-Concurrency-And-Race-Conditions.md` for the full thread-safety deep dive.

---

### Q06: What's `@Scheduled` and how do you handle distributed scheduling? ⭐

**30-Second Answer:** `@Scheduled` runs methods periodically. **`@Scheduled` runs on EVERY pod** by default — so a 3-pod deployment runs the same cron 3 times. Fix with **ShedLock** (DB-based distributed lock) or use a **scheduler service** (Quartz cluster, Temporal, K8s CronJob).

**Deep Dive:**

```java
@Component
@EnableScheduling
public class ReportGenerator {

    @Scheduled(cron = "0 0 2 * * *")   // 2 AM daily
    public void generateReports() { ... }   //  Runs on ALL pods!

    @Scheduled(fixedDelay = 60_000)    // 60s after last finish
    public void cleanupTempFiles() { ... }

    @Scheduled(fixedRate = 30_000)     // every 30s (regardless of duration)
    public void healthCheck() { ... }
}
```

**ShedLock — distributed lock:**

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
</dependency>
```

```sql
CREATE TABLE shedlock(
    name VARCHAR(64) PRIMARY KEY,
    lock_until TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NOT NULL,
    locked_by VARCHAR(255) NOT NULL
);
```

```java
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulerConfig {
    @Bean
    public LockProvider lockProvider(DataSource ds) {
        return new JdbcTemplateLockProvider(ds);
    }
}

@Component
public class ReportGenerator {
    @Scheduled(cron = "0 0 2 * * *")
    @SchedulerLock(name = "generateReports", lockAtMostFor = "30m", lockAtLeastFor = "5m")
    public void generateReports() {
        //  Only ONE pod runs this; others skip if lock held
    }
}
```

**Common Follow-Ups:**
- "Why `lockAtLeastFor`?" → Prevents another pod from running if first pod's clock is slightly off (avoids double-run)
- "Alternative to ShedLock?" → Quartz with JDBC store (more features), K8s CronJob (move out of app entirely)

**Red Flag If You Say:** "`@Scheduled` only runs on one pod automatically." (False — runs on every replica!)

---

### Q07: How do you make a thread-safe singleton in Spring?

**30-Second Answer:** Spring `@Service`/`@Component` is **already a singleton in the container** — you don't need DCL or Holder pattern. Just make the class **stateless** (no mutable fields). If state needed, use thread-safe types.

**Deep Dive:**

```java
//  Spring already does this for you
@Service
public class UserService { ... }   // ONE instance per Spring container

//  Pure Java singleton (when you need lazy init outside Spring)
public class LegacyService {
    private LegacyService() {}
    private static class Holder {
        static final LegacyService INSTANCE = new LegacyService();
    }
    public static LegacyService getInstance() { return Holder.INSTANCE; }
    //  Initialization-on-demand holder idiom — thread-safe by JVM class loader semantics
}

//  Old-school DCL (rarely needed)
public class LegacyService {
    private static volatile LegacyService INSTANCE;
    public static LegacyService getInstance() {
        if (INSTANCE == null) {
            synchronized (LegacyService.class) {
                if (INSTANCE == null) {
                    INSTANCE = new LegacyService();   //  volatile is REQUIRED here
                }
            }
        }
        return INSTANCE;
    }
}
```

**Common Follow-Ups:**
- "Why is the Holder idiom preferred?" → Lazy + thread-safe + no synchronization cost on every call
- "Why does DCL need `volatile`?" → Without it, partially-constructed object may be visible to another thread (publication race)

---

### Q08: How do `Mono` and `Flux` work in WebFlux?

**30-Second Answer:** `Mono<T>` = 0 or 1 element; `Flux<T>` = 0 to N elements. Both are **publishers** (cold streams) — nothing happens until subscribed. Operators (`map`, `flatMap`, `filter`, `zip`) build a pipeline; `.subscribe()` or returning from controller triggers execution.

**Deep Dive:**

```java
//  Mono — single async result
Mono<User> userMono = userRepo.findById(id);                     // not executed yet
Mono<String> nameMono = userMono.map(User::getName);             // still not executed
Mono<String> upperMono = nameMono.map(String::toUpperCase);      // still not executed
//  Spring subscribes when returning from controller — pipeline runs

//  Flux — stream of results
Flux<Order> orders = orderRepo.findByCustomer(id);
Flux<OrderDto> dtos = orders.filter(o -> o.getTotal() > 100)
    .map(OrderDto::from)
    .take(20);
//  Server-Sent Events
@GetMapping(value = "/orders/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<OrderEvent> stream() {
    return orderEventService.eventStream();   // pushes events as they happen
}

//  Common operators
mono.flatMap(u -> fetchOrders(u.getId()))    // chain async
   .zipWith(fetchPreferences(id))             // combine two
   .timeout(Duration.ofSeconds(5))            // timeout
   .retry(3)                                  // retry on error
   .onErrorResume(ex -> Mono.just(fallback))  // recover
   .doOnNext(x -> log.info("got {}", x))      // side-effect
   .subscribeOn(Schedulers.boundedElastic()); // run on bounded elastic
```

**Common Follow-Ups:**
- "Cold vs hot stream?" → Cold = each subscriber gets own data (DB query re-runs); Hot = shared stream (Kafka topic)
- "What's `.block()` and why is it bad?" → Blocks the reactive thread, defeating the purpose

---

## Top 3 from This File (must-know)

1. **Q01** — `@Async` gotchas (self-invocation, executor config)
2. **Q04** — Virtual threads (the hot 2024-2026 topic)
3. **Q06** — Distributed scheduling with ShedLock

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| @Async                            | Needs @EnableAsync; configure    |
|                                   | executor; self-invocation breaks |
| Default async executor            | SimpleAsyncTaskExecutor (BAD)    |
| CompletableFuture.thenApply       | Transform result (sync)          |
| CompletableFuture.thenCompose     | Chain another async (flatMap)    |
| CompletableFuture.thenCombine     | Merge two independent futures    |
| .orTimeout(5, SECONDS)            | Avoid hanging futures            |
| MVC vs WebFlux                    | WebFlux for high concurrency I/O |
| Virtual threads                   | Spring Boot 3.2 + Java 21        |
| Pinning                           | synchronized blocks pin VT to OS |
| Singleton service + state         | AtomicX or stateless             |
| @Scheduled                        | Runs on EVERY pod                |
| ShedLock                          | DB lock for distributed schedule |
| Mono                              | 0-1 result, cold publisher       |
| Flux                              | 0-N stream, cold publisher       |
| .block()                          | NEVER in reactive code           |
+----------------------------------+----------------------------------+
```
