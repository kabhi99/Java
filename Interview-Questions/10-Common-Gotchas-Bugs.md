# **COMMON GOTCHAS & BUGS**

10 questions on the bugs that come up in EVERY senior Spring Boot interview. **Read this file the day before your interview.**

---

### Q01: Self-invocation breaks `@Transactional`, `@Async`, `@Retryable`, `@Cacheable`. Why? ⭐⭐⭐

**30-Second Answer:** All four are implemented via **Spring AOP proxies**. The proxy wraps the bean — external callers go through the proxy (annotations triggered), but **`this.method()` from within the bean bypasses the proxy** (annotations IGNORED).

**Deep Dive:**

```java
@Service
public class MyService {

    public void outer() {
        inner();   //  Direct call — bypasses proxy → @Transactional ignored!
    }

    @Transactional
    public void inner() { ... }
}

// What actually happens:
// Client → Proxy.outer() → MyService.outer() → this.inner() (raw object, no proxy)
//                                              ↑
//                                       Annotation interceptor missed
```

**Fixes:**
```java
//  FIX 1: Move to different bean (best)
@Service class OuterService {
    @Autowired InnerService innerService;
    public void outer() { innerService.inner(); }
}

//  FIX 2: Self-injection
@Service
public class MyService {
    @Autowired @Lazy private MyService self;
    public void outer() { self.inner(); }  // proxy → annotation works
    @Transactional public void inner() { ... }
}

//  FIX 3: AspectJ load-time weaving (heavy, last resort)
```

**Detection trick**: Enable `org.springframework.transaction.interceptor=TRACE`. If you don't see "Getting transaction for [...]" for a method, the proxy was bypassed.

**Common Follow-Ups:**
- "Why is this default behavior so dangerous?" → No warning, no error — just silently doesn't work
- "Why can't Spring fix this?" → Proxies wrap on outside; Spring would need bytecode manipulation (AspectJ) to handle internal calls

**Red Flag If You Say:** "It works in tests so it's fine." (Until production scales and the missing rollback bites you.)

---

### Q02: Why does `@Transactional` not roll back on checked exceptions? ⭐

**30-Second Answer:** Spring's default rollback only triggers on **`RuntimeException` and `Error`**. **Checked exceptions** (like `IOException`, `SQLException`) do NOT roll back. Fix: `@Transactional(rollbackFor = Exception.class)`.

**Deep Dive:**

```java
@Transactional
public void transferMoney() throws IOException {
    accountRepo.debit(...);
    if (somethingBad) throw new IOException("checked!");   //  NO ROLLBACK
    accountRepo.credit(...);
}
// Debit happened, IOException thrown, NO ROLLBACK → money disappeared

//  Fix
@Transactional(rollbackFor = Exception.class)
public void transferMoney() throws IOException { ... }

//  Or always use RuntimeException for business errors
@Transactional
public void transferMoney() {
    if (somethingBad) throw new BusinessException("...");   //  RuntimeException, rolls back
}
```

**Why this default?** EJB legacy — checked exceptions were "business exceptions" (recoverable), runtime were "system errors" (catastrophic).

**Common Follow-Ups:**
- "What about errors?" → `Error` (like `OutOfMemoryError`) DOES trigger rollback
- "Best practice?" → Use `RuntimeException` for business errors; or `@Transactional(rollbackFor = Exception.class)` everywhere

---

### Q03: Why might `@Transactional` not work at all?

**30-Second Answer:** Common reasons:
1. **Method is `private`** — proxy can't override
2. **Method is `final`** — CGLIB proxy can't override
3. **Self-invocation** (see Q01)
4. **No `@EnableTransactionManagement`** (Spring Boot enables by default; vanilla Spring doesn't)
5. **Exception caught and swallowed** inside the method
6. **Wrong propagation** — calling `@Transactional` from a `@Async` method (different thread, new context)

**Deep Dive:**

```java
//  Private — proxy can't intercept
@Transactional
private void save() { ... }

//  Final — CGLIB can't subclass
@Transactional
public final void save() { ... }

//  Exception swallowed — no rollback
@Transactional
public void save() {
    try {
        repo.save(...);
        throw new RuntimeException();
    } catch (Exception e) {
        log.error("oops", e);   //  Swallowed → no rollback
    }
}

//  Async wraps in new thread → new transaction context
@Async
@Transactional   //  Doesn't apply how you think — runs in new thread
public void saveAsync() { ... }
```

**Common Follow-Ups:**
- "Why does final not work with CGLIB?" → CGLIB creates subclass; final methods can't be overridden
- "JDK proxy vs CGLIB?" → JDK proxy for interfaces, CGLIB for concrete classes. Spring Boot defaults to CGLIB

---

### Q04: Spring Boot defaults `spring.jpa.open-in-view=true`. Why is this bad? ⭐

**30-Second Answer:** **Open Session In View (OSIV)** keeps the Hibernate session OPEN for the entire HTTP request — including the view rendering. **Hides N+1 problems** (lazy loads silently triggered in controllers), holds DB connections longer than needed, makes performance unpredictable. **Always set `spring.jpa.open-in-view=false` in production**.

**Deep Dive:**

```java
// With OSIV (default) — works "magically"
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) {
    return orderRepo.findById(id).get();
    //  Returns entity directly
    //  Jackson serializes → accesses order.items (lazy)
    //  Session still open → silently loads items + customer + everything
    //  N+1 queries happen INVISIBLY
}

// With OSIV disabled (correct)
//  Same code — throws LazyInitializationException
//  Forces you to fetch eagerly via DTO or JOIN FETCH
```

**Fix the right way:**
```yaml
spring:
  jpa:
    open-in-view: false   #  Disable in all environments
```

```java
//  Explicit DTO + JOIN FETCH
@GetMapping("/orders/{id}")
public OrderDto get(@PathVariable Long id) {
    Order o = orderRepo.findByIdWithItems(id).orElseThrow();
    return OrderDto.from(o);
}

@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

**Common Follow-Ups:**
- "Why does Spring Boot enable it by default?" → Backwards compatibility + ease of getting started. Logs a warning at startup
- "What about read-only operations?" → Use `@Transactional(readOnly = true)` on the service method

**Red Flag If You Say:** "OSIV is great because lazy loading just works." (You're hiding N+1 + connection-hold bugs.)

---

### Q05: Why does `equals()` and `hashCode()` on JPA entities cause subtle bugs? ⭐

**30-Second Answer:** Using **`@Id` (auto-generated) in `equals()`** breaks `Set`/`Map` semantics — entities without IDs (new, transient) all equal each other, then change identity when persisted (ID assigned). **Best practice**: use a **business key** or **only `id != null` comparison**.

**Deep Dive:**

```java
//  Lombok @EqualsAndHashCode on JPA entity — DANGEROUS
@Entity
@EqualsAndHashCode   //  Uses all fields, INCLUDING lazy collections!
public class Order { ... }

// Bugs:
// - Lazy collection access in equals() → unexpected DB queries / LazyInit exception
// - Two new entities (id=null) are equal → can't add both to a Set

//  Better: equals on ID with null safety
@Entity
public class Order {
    @Id @GeneratedValue Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order other = (Order) o;
        return id != null && id.equals(other.id);   //  null id never equals
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();   //  Constant — bad for HashMap perf, but correct
        //  Use a business key instead if you have one
    }
}

//  Best: business key
@Entity
public class Order {
    @NaturalId(mutable = false)
    String orderNumber;   //  immutable business identity

    public boolean equals(Object o) {
        return o instanceof Order other && Objects.equals(orderNumber, other.orderNumber);
    }
    public int hashCode() { return Objects.hash(orderNumber); }
}
```

**Common Follow-Ups:**
- "Why does `hashCode()` returning a constant work?" → Correctness-wise OK (all in same bucket); perf is `O(n)` not `O(1)`
- "What if I have no business key?" → Use `UUID` assigned in constructor (not DB-generated)

---

### Q06: Race condition: `findById` then update — what goes wrong? ⭐

**30-Second Answer:** Two transactions read same row (stock=10), each subtract 1, each save (stock=9). One decrement is **lost**. Fix: **atomic SQL**, **optimistic lock (`@Version`)**, or **pessimistic lock (`SELECT FOR UPDATE`)**.

**Deep Dive:**

```java
//  BUG — lost update
@Transactional
public void decrementStock(Long id) {
    Product p = repo.findById(id).get();   // both T1 and T2 read stock=10
    p.setStock(p.getStock() - 1);           // both compute stock=9
    repo.save(p);                            // both write stock=9 (one decrement lost!)
}

//  FIX 1: Atomic SQL (simplest, fastest)
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - 1 WHERE p.id = :id AND p.stock >= 1")
int decrementStock(@Param("id") Long id);
// Returns 0 if stock was 0 (out of stock); 1 if decremented

//  FIX 2: Optimistic lock
@Entity
public class Product { @Version Long version; }
// Failed updates → OptimisticLockException → retry

//  FIX 3: Pessimistic lock
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findByIdForUpdate(@Param("id") Long id);
// SELECT ... FOR UPDATE — blocks other transactions until commit
```

> See `Java/01-Concurrency-And-Race-Conditions.md` and `Java/04-Scaling-Spring-Boot-APIs.md` for more.

**Common Follow-Ups:**
- "Why does atomic SQL work?" → DB handles concurrency natively via row locks during UPDATE
- "When is optimistic better than pessimistic?" → Low contention (typical CRUD)

---

### Q07: Why does `LocalDateTime` cause timezone bugs? ⭐

**30-Second Answer:** `LocalDateTime` has **no timezone** — it's a wall-clock value. Two servers in different TZs interpret it differently. **Always use `Instant` or `ZonedDateTime` in APIs and DB columns; convert to `LocalDateTime` only for display**.

**Deep Dive:**

```java
//  BUG — timezone-naive
public class Order {
    LocalDateTime createdAt;   // "2026-06-01 12:00:00" — but in what TZ?!
}

// Server A (UTC): saves "2026-06-01 12:00:00"
// Server B (IST, UTC+5:30): reads as "2026-06-01 12:00:00 IST" → off by 5.5h

//  Use Instant (always UTC)
public class Order {
    Instant createdAt;   // unambiguous moment in time
}

//  Use ZonedDateTime if TZ matters (e.g., scheduling in user's TZ)
public class Meeting {
    ZonedDateTime startsAt;   // "2026-06-01T15:00:00+05:30[Asia/Kolkata]"
}

//  application.yml — force JVM TZ
spring:
  jackson:
    time-zone: UTC
```

```sql
-- DB type
-- Postgres: TIMESTAMP WITH TIME ZONE (TIMESTAMPTZ) — stores UTC, displays in session TZ
-- MySQL: DATETIME stores no TZ; TIMESTAMP stores UTC
```

**Common Follow-Ups:**
- "How to set JVM timezone?" → `-Duser.timezone=UTC` (all servers in UTC; convert at edges)
- "Why does `new Date()` cause bugs?" → Mutable, not timezone-aware, deprecated for most use cases

---

### Q08: Why does `@PostConstruct` not work in some classes?

**30-Second Answer:** Common reasons:
1. **JPMS / Java 11+** — `@PostConstruct` is in `jakarta.annotation`; needs `jakarta.annotation-api` dependency
2. **Method is private static**
3. **Bean is `prototype` scope** — Spring doesn't track lifecycle
4. **Exception thrown in `@PostConstruct`** — bean creation fails, app won't start

**Deep Dive:**

```java
//  PostConstruct on prototype bean — works on creation but no destroy
@Component
@Scope("prototype")
public class MyBean {
    @PostConstruct void init() { ... }     // called on each getBean()
    @PreDestroy void cleanup() { ... }     // NEVER called (Spring doesn't track)
}

//  PostConstruct that depends on bean (use ApplicationReadyEvent instead)
@Component
public class StartupTask {
    @Autowired SomeService service;

    @PostConstruct
    void init() {
        service.callOther();   //  Other bean might not be fully initialized yet
    }
}

//  Better: ApplicationReadyEvent (entire context is ready)
@Component
public class StartupTask {
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        service.callOther();   //  Everything initialized
    }
}
```

**Common Follow-Ups:**
- "When use `@PostConstruct` vs `ApplicationReadyEvent`?" → PostConstruct = per-bean init; ApplicationReadyEvent = whole app ready

---

### Q09: Why does `application.yml` not load `${ENV_VAR}` correctly?

**30-Second Answer:** Spring Boot uses **`${VAR}` or `${VAR:default}`** syntax. Bash-style `$VAR` won't work. Check OS env: env vars in Spring property names use `.` → `_` (so `SPRING_DATASOURCE_URL` overrides `spring.datasource.url`). **Beware env var precedence** (overrides yml).

**Deep Dive:**

```yaml
spring:
  datasource:
    url: ${DB_URL}                          # env var DB_URL
    username: ${DB_USER:postgres}           # with default
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:20}
```

**Env var name conversion:**
| yml property | env var |
|---|---|
| `spring.datasource.url` | `SPRING_DATASOURCE_URL` |
| `server.port` | `SERVER_PORT` |
| `app.feature.x-enabled` | `APP_FEATURE_X_ENABLED` |

**Common gotchas:**
```yaml
#  Old syntax for placeholders (works but ugly)
url: jdbc:postgresql://${DB_HOST}:${DB_PORT:5432}/${DB_NAME}

#  Nested defaults
url: ${DB_URL:jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:app}}

#  Profile-specific files override defaults
# application-prod.yml takes precedence over application.yml
```

**Common Follow-Ups:**
- "How to debug 'which property won?'" → Hit `/actuator/env` endpoint — shows source of every property
- "Difference between `${X}` and `#{...}`?" → `${X}` is property placeholder; `#{...}` is SpEL expression

---

### Q10: Why does Spring Boot startup get slow? How do you debug it?

**30-Second Answer:** Common causes:
1. **Too many auto-configurations evaluating** (each conditional check costs)
2. **Slow `@PostConstruct` methods** (DB calls, network)
3. **Eager bean initialization with complex graph**
4. **Component scan covering too many packages**
5. **Schema validation/Flyway migrations**

**Diagnose with `--debug` flag** to see auto-config report; profile with **Spring Boot Startup Tracker** or **JFR**.

**Deep Dive:**

```java
//  Lazy initialization — only create beans when accessed
@SpringBootApplication
@EnableLazyInitialization   // global
public class App { ... }

//  Or per-bean
@Component
@Lazy
public class HeavyBean { ... }
```

```yaml
spring:
  main:
    lazy-initialization: true   #  Speeds startup but risks runtime failures
```

```bash
# Debug auto-config
java -jar app.jar --debug   # see Positive/Negative matches

# Profile startup time
java -XX:StartFlightRecording=filename=startup.jfr -jar app.jar
# Open in JDK Mission Control
```

**Common offenders:**
```java
//  @PostConstruct doing DB calls
@PostConstruct
void warmup() { for (int i = 0; i < 1000000; i++) cache.put(loadFromDB(i)); }
//  Move to @EventListener(ApplicationReadyEvent) + run async

//  Component scan too broad
@SpringBootApplication(scanBasePackages = "com")   //  Scans everything
//  Restrict
@SpringBootApplication(scanBasePackages = "com.mycompany.app")
```

**Common Follow-Ups:**
- "What's GraalVM native image?" → AOT compile to native binary. Startup ~100x faster, lower memory. Trade-off: build complexity + no runtime reflection
- "Are virtual threads useful at startup?" → Yes for parallel bean init (Spring Boot 3.2+)

---

## Top 5 from This File (must-know — bugs that get juniors filtered out)

1. **Q01** — Self-invocation breaks AOP
2. **Q02** — `@Transactional` doesn't rollback on checked exceptions
3. **Q04** — Open-Session-In-View disable in prod
4. **Q06** — Lost update race condition
5. **Q07** — `LocalDateTime` timezone bugs

## The "Will You Notice This in a Code Review?" List

If you can spot these in code without thinking, you're a senior:

```
+----------------------------------+-----------------------------------+
| Code pattern                      | Bug                               |
+----------------------------------+-----------------------------------+
| this.someAsyncMethod()            | Self-invocation — @Async ignored  |
| catch(Exception e) { log... }     | Swallowed exception — no rollback |
| @Transactional throws IOException | No rollback for checked exception |
| @EqualsAndHashCode on @Entity     | Lazy load in equals + null ID bug |
| LocalDateTime in API DTO          | Timezone ambiguity                |
| public final method + @Transactional | Final → CGLIB can't override   |
| findById().set().save()           | Lost update race                  |
| @Autowired field                  | Not testable; circular dep silent |
| Returning Entity from controller  | OSIV bug, lazy load surprises     |
| @Scheduled in multi-pod deploy    | Runs N times                      |
| HashMap shared in @Service        | Race condition in singleton       |
| @PostConstruct calling other bean | Other bean might not be ready     |
| jdbc.batch_size unset             | saveAll = N inserts, not batched  |
| spring.jpa.open-in-view=true      | Hidden N+1, long-held connections |
+----------------------------------+-----------------------------------+
```
