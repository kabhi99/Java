# **JPA, TRANSACTIONS & PERSISTENCE**

12 questions on Spring Data JPA, `@Transactional`, locking, performance. **This is the #1 source of senior interview questions at high-paying companies.**

---

### Q01: Explain `@Transactional` propagation levels. ⭐⭐

**30-Second Answer:** Propagation controls **what happens when a transactional method calls another transactional method**. Seven levels; the 4 you must know:

| Propagation | If existing txn? | If no existing txn? |
|---|---|---|
| **`REQUIRED`** (default) | Join it | Create new |
| **`REQUIRES_NEW`** | **Suspend it**, create new (independent commit/rollback) | Create new |
| **`NESTED`** | Create nested savepoint | Create new |
| **`SUPPORTS`** | Join it | Run non-transactionally |

**Deep Dive:**

```java
//  REQUIRED — entire flow rolls back together
@Transactional   // default = REQUIRED
public void createOrder(Order o) {
    orderRepo.save(o);
    auditService.log(o);    // joins same txn — both commit or both rollback
}

//  REQUIRES_NEW — audit log committed even if order rollback
@Service
public class AuditService {
    @Transactional(propagation = REQUIRES_NEW)
    public void log(Order o) { ... }   // independent txn
}

@Transactional
public void createOrder(Order o) {
    orderRepo.save(o);
    auditService.log(o);    // SEPARATE txn — commits independently
    throw new RuntimeException("oops");   // outer rolls back, audit stays!
}
```

**Common interview gotcha:**
```java
@Transactional
public void outer() {
    inner();   //  Same class — @Transactional IGNORED (proxy bypass!)
}

@Transactional(propagation = REQUIRES_NEW)
public void inner() { ... }
```

**Common Follow-Ups:**
- "When would you use REQUIRES_NEW?" → Audit logs, sending notifications — things that should commit independently
- "What's NESTED?" → Single physical connection, savepoint allows partial rollback (rarely used; not all DBs support)

**Red Flag If You Say:** "REQUIRES_NEW is the safest." (Misses that it suspends the outer transaction — connections + locks held twice → deadlock risk.)

---

### Q02: Explain `@Transactional` isolation levels. ⭐

**30-Second Answer:** Controls what other concurrent transactions can see. Four levels (weakest → strongest):

| Level | Prevents | Issue |
|---|---|---|
| **`READ_UNCOMMITTED`** | Nothing | Dirty reads |
| **`READ_COMMITTED`** | Dirty reads | Non-repeatable reads |
| **`REPEATABLE_READ`** | Non-repeatable reads | Phantom reads |
| **`SERIALIZABLE`** | Everything | Lowest throughput |

**Deep Dive:**

```
Dirty read:       T1 reads T2's UNCOMMITTED data → T2 rolls back → T1 has wrong data
Non-repeatable:   T1 reads X=10, T2 commits X=20, T1 reads X=20 (same query, different result)
Phantom:          T1 reads "SELECT * WHERE x<10" (gets 3 rows), T2 inserts a 4th matching row,
                  T1 re-reads → 4 rows (phantom appeared)
```

**Postgres default = `READ_COMMITTED`. MySQL InnoDB default = `REPEATABLE_READ`.**

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferMoney(long from, long to, BigDecimal amt) {
    // Strongest isolation — DB enforces serial order
    // Use ONLY for critical financial operations
}
```

**Common Follow-Ups:**
- "Which level for an inventory decrement?" → `READ_COMMITTED` + optimistic lock; or `SERIALIZABLE` for absolute correctness
- "What's `SERIALIZABLE`'s cost?" → Postgres uses SSI (Serializable Snapshot Isolation) — may abort transactions, need retry logic

---

### Q03: What's the self-invocation problem with `@Transactional`? How do you fix it? ⭐⭐

**30-Second Answer:** Spring's `@Transactional` works via a **proxy** wrapping the bean. When method A calls method B in the **same class**, the call bypasses the proxy → `@Transactional` on B is **silently ignored**. Same for `@Async`, `@Retryable`, `@Cacheable`.

**Deep Dive:**

```java
@Service
public class OrderService {

    @Transactional
    public void outer() {
        inner();   //  Direct call — bypasses proxy → no new transaction!
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void inner() {
        // Expected to be in a separate txn — but it's in the SAME one as outer()
    }
}
```

**Fixes (in order of preference):**

```java
//  FIX 1: Move to another bean (cleanest)
@Service
public class OrderService {
    private final InnerService innerService;

    @Transactional
    public void outer() {
        innerService.inner();   //  Goes through proxy
    }
}

//  FIX 2: Self-injection (works but smells)
@Service
public class OrderService {
    @Autowired @Lazy private OrderService self;   //  Inject proxy of self

    @Transactional
    public void outer() {
        self.inner();   //  Proxy called
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void inner() { ... }
}

//  FIX 3: AspectJ weaving (most invasive)
// Use compile-time/load-time weaving instead of Spring proxies
```

**Common Follow-Ups:**
- "Why doesn't this happen with `private` methods?" → Private methods can't be overridden by proxy at all (always silent fail)
- "How would you debug a missing transaction?" → Enable `org.springframework.transaction.interceptor=TRACE` logging

**Red Flag If You Say:** "Just use `@Transactional` and trust Spring." (Misses the proxy mechanic.)

---

### Q04: What's the N+1 query problem and how do you fix it? ⭐⭐

**30-Second Answer:** N+1 = one query to fetch parents, then N queries to fetch each parent's children (lazy loading). Fix with `JOIN FETCH`, `@EntityGraph`, or DTO projections. **Single biggest perf killer in JPA.**

**Deep Dive:**

```java
@Entity
public class Order {
    @Id Long id;
    @OneToMany(mappedBy = "order", fetch = LAZY)   // default
    List<OrderItem> items;
}

//  N+1 — 1 query for orders + N queries for items
List<Order> orders = repo.findAll();   // 1 query
for (Order o : orders) {
    o.getItems().size();   // 1 query EACH → N more queries
}

//  FIX 1: JOIN FETCH
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items")
List<Order> findAllWithItems();

//  FIX 2: @EntityGraph (cleaner, declarative)
@EntityGraph(attributePaths = {"items", "customer"})
@Override
List<Order> findAll();

//  FIX 3: DTO projection (best for read-only)
@Query("""
    SELECT new com.example.OrderDto(o.id, o.total, COUNT(i))
    FROM Order o LEFT JOIN o.items i
    GROUP BY o.id, o.total
""")
List<OrderDto> findAllAsDto();
```

**Detection:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true   # logs query count
logging:
  level:
    org.hibernate.SQL: DEBUG         # see all queries
```

Or use **p6spy** / **Datasource-proxy** / **Hibernate Statistics**.

**Common Follow-Ups:**
- "Why does `@Transactional` make N+1 worse?" → Open session lets lazy proxies trigger; without txn, you'd get `LazyInitializationException` (which exposes the bug earlier!)
- "What's MultipleBagFetchException?" → JOIN FETCH on multiple `List<T>` collections — Hibernate can't deduplicate; use `Set` or split queries

**Red Flag If You Say:** "Just use EAGER fetching." (Then EVERY query loads everything — worse problem.)

---

### Q05: Optimistic vs Pessimistic locking — when to use which? ⭐

**30-Second Answer:** **Optimistic** (default for low conflict): use `@Version`, retry on conflict. **Pessimistic** (`SELECT FOR UPDATE`, for high conflict): blocks other readers/writers; lower throughput but no retry needed.

**Deep Dive:**

```java
//  OPTIMISTIC — version column
@Entity
public class Product {
    @Id Long id;
    int stock;
    @Version Long version;   // auto-incremented on update
}

@Transactional
public void decrementStock(Long id, int qty) {
    Product p = repo.findById(id).orElseThrow();
    p.setStock(p.getStock() - qty);
    repo.save(p);
    // UPDATE product SET stock=?, version=? WHERE id=? AND version=?
    // If 0 rows updated → OptimisticLockException → retry
}

//  PESSIMISTIC — locks row in DB
@Query("SELECT p FROM Product p WHERE p.id = :id")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Product> findByIdWithLock(@Param("id") Long id);

@Transactional
public void decrementStock(Long id, int qty) {
    Product p = repo.findByIdWithLock(id).orElseThrow();
    p.setStock(p.getStock() - qty);
    repo.save(p);
    // Other transactions wait until this one commits
}
```

**Which to use:**
| Pattern | Use case |
|---|---|
| **Optimistic** | Read-heavy, low contention (most CRUD) |
| **Pessimistic** | High contention (flash sales, payment) |
| **Atomic SQL** | Best of both: `UPDATE products SET stock = stock - 1 WHERE id = ? AND stock >= 1` |

**Common Follow-Ups:**
- "What if optimistic lock retries 10 times and still fails?" → Probably high contention → switch to pessimistic or atomic SQL
- "Difference between PESSIMISTIC_READ and PESSIMISTIC_WRITE?" → READ = shared lock (others can read, not write); WRITE = exclusive lock

---

### Q06: When does Hibernate flush? What's the flush mode?

**30-Second Answer:** Hibernate flushes (sends pending SQL to DB) **on transaction commit**, **before queries** (to keep them consistent), or **on explicit `flush()`**. Default flush mode = `AUTO`. **Bug magnet: changes aren't visible to other transactions until commit.**

**Deep Dive:**

```java
@Transactional
public void example() {
    Order o = new Order();
    repo.save(o);   //  NOT yet sent to DB — just in persistence context

    // Hibernate flushes here because next query might depend on saved data
    long count = repo.count();

    // STILL not committed — other txns don't see it
}
// COMMIT happens here → all pending SQL sent + committed
```

**Flush modes:**
- `AUTO` (default): flush before query, on commit, on `flush()`
- `COMMIT`: only on commit (faster, but queries may see stale data)
- `MANUAL`: only on explicit `flush()`

**Common Follow-Ups:**
- "Why might I call `flush()` manually?" → To get generated ID before transaction commits (for sending events), or to catch constraint violations early
- "What's dirty checking?" → On flush, Hibernate compares entity snapshot to current state — generates UPDATE only for changed fields

---

### Q07: Explain Hibernate's first-level and second-level cache.

**30-Second Answer:** **L1 cache** = persistence context (per transaction/session) — always on, can't disable. **L2 cache** = shared across sessions (per JVM or distributed) — off by default, opt-in per entity. L2 needs careful invalidation; often replaced by Redis for cross-pod sharing.

**Deep Dive:**

```java
@Transactional
public void example() {
    Order o1 = repo.findById(1L).get();   // DB hit + cached in L1
    Order o2 = repo.findById(1L).get();   // L1 hit — NO DB query
    assert o1 == o2;                       // same instance!
}
//  L1 cleared at end of transaction
```

**L2 cache (opt-in):**
```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }   // Reference data — perfect for L2
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

**Common Follow-Ups:**
- "When is L2 cache useful vs Redis?" → L2 for read-mostly reference data (countries, settings). Redis for cross-pod sharing, eviction control
- "What invalidates L2 cache?" → JPA writes auto-invalidate; direct SQL doesn't (silent stale data!)

**Red Flag If You Say:** "Enable L2 cache everywhere for performance." (Cache stampede, invalidation hell.)

---

### Q08: `save()` vs `saveAndFlush()` vs `persist()` vs `merge()`?

**30-Second Answer:**
- **`persist()`** — JPA — entity must be transient (no ID yet)
- **`merge()`** — JPA — copies state of detached entity into managed; returns NEW managed instance
- **`save()`** — Spring Data — calls `persist()` if new, `merge()` if detached
- **`saveAndFlush()`** — Spring Data — `save()` + immediate `flush()` (visible to subsequent queries in same txn)

**Deep Dive:**

```java
//  persist — for NEW entities
@Transactional
public void create(Order o) { em.persist(o); }   // fails if o has ID set

//  merge — for DETACHED entities (came from outside session)
@Transactional
public Order update(Order detached) {
    return em.merge(detached);   //  Returns the managed version
    //  detached is NOT updated — use the return value
}

//  Spring Data save — convenient but ambiguous
public Order updateOrCreate(Order o) {
    return repo.save(o);   // persist if new (no ID), merge if existing
}

//  saveAndFlush — when you need ID immediately
@Transactional
public void createAndSendEvent(Order o) {
    Order saved = repo.saveAndFlush(o);   //  Forces INSERT now
    kafkaTemplate.send("orders", saved.getId());   //  ID available
    // Without saveAndFlush, save() might be deferred until txn commit
}
```

**Common Follow-Ups:**
- "Why does `merge()` return a new object?" → Original is detached; managed version is what's in session
- "Why is `saveAll(...)` slow?" → Without batching, it's one INSERT per entity. Enable: `hibernate.jdbc.batch_size=50`

---

### Q09: What's the difference between `JpaRepository`, `PagingAndSortingRepository`, `CrudRepository`?

**30-Second Answer:** Inheritance chain: `Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository`.

| Interface | Adds |
|---|---|
| `Repository<T, ID>` | Marker only |
| `CrudRepository` | `save`, `findById`, `findAll`, `count`, `delete` |
| `PagingAndSortingRepository` | `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository` | `flush`, `saveAndFlush`, `deleteAllInBatch`, `findAll` returns `List` not `Iterable` |

**Deep Dive:**

```java
//  Most common — full power
public interface OrderRepository extends JpaRepository<Order, Long> { ... }

//  Minimal — read-only
public interface CountryRepository extends Repository<Country, String> {
    Optional<Country> findById(String code);
    List<Country> findAll();
}
```

**Common Follow-Ups:**
- "Why might you NOT extend `JpaRepository`?" → To restrict the surface area (e.g., expose only `findById` to enforce immutability)

---

### Q10: How do derived queries work? When do you use `@Query`?

**30-Second Answer:** Derived queries parse method names (`findByEmailAndStatus`) into SQL. Use `@Query` when:
- Complex joins
- Custom projections
- Performance (you want exact SQL)
- Native queries (database-specific features)

**Deep Dive:**

```java
//  Derived — Spring parses method name
List<User> findByEmailAndStatus(String email, Status status);
List<User> findByCreatedAtAfterOrderByCreatedAtDesc(LocalDateTime since);
long countByActiveTrue();

//  @Query — JPQL
@Query("SELECT u FROM User u WHERE u.email = :email AND u.status = 'ACTIVE'")
Optional<User> findActiveByEmail(@Param("email") String email);

//  Native query
@Query(value = "SELECT * FROM users WHERE email = ?1 AND deleted_at IS NULL",
       nativeQuery = true)
Optional<User> findActiveByEmailNative(String email);

//  Modifying query (for UPDATE/DELETE)
@Modifying
@Transactional
@Query("UPDATE User u SET u.lastSeen = :now WHERE u.id = :id")
int updateLastSeen(@Param("id") Long id, @Param("now") LocalDateTime now);
```

**Common Follow-Ups:**
- "When does derived query become unreadable?" → ~3+ conditions → switch to `@Query` or Specification
- "Why `@Modifying`?" → Required for UPDATE/DELETE; otherwise Spring expects a SELECT result

---

### Q11: How do you handle large bulk inserts efficiently?

**30-Second Answer:** Three techniques:
1. **JDBC batching** — `hibernate.jdbc.batch_size=50` + `repo.saveAll(chunks)`
2. **Native batch insert** — single multi-row INSERT
3. **`COPY` (Postgres) / `LOAD DATA` (MySQL)** — fastest for huge loads

**Deep Dive:**

```yaml
# Enable Hibernate batching
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          order_inserts: true
          order_updates: true
          batch_versioned_data: true
```

```java
//  Batched via JPA
@Transactional
public void bulkInsert(List<User> users) {
    for (int i = 0; i < users.size(); i++) {
        em.persist(users.get(i));
        if (i % 50 == 0) {           // flush + clear every 50
            em.flush();
            em.clear();               // prevent OOM
        }
    }
}

//  Native batch (fastest in JPA)
@Modifying
@Query(value = """
    INSERT INTO users (id, name, email) VALUES
    (:id, :name, :email)
    ON CONFLICT (id) DO NOTHING
""", nativeQuery = true)
int batchInsert(@Param("id") Long id, @Param("name") String n, @Param("email") String e);

//  COPY (Postgres) — millions of rows
jdbcTemplate.execute(conn -> {
    CopyManager cm = ((PGConnection) conn.unwrap(PGConnection.class)).getCopyAPI();
    return cm.copyIn("COPY users(id, name, email) FROM STDIN WITH CSV",
                     new StringReader(csvData));
});
```

**Common Follow-Ups:**
- "Why `em.clear()` in the loop?" → Without it, persistence context grows unbounded → OOM
- "Why `IDENTITY` IDs kill batching?" → IDENTITY requires DB roundtrip per INSERT → batching disabled. Use `SEQUENCE` instead

---

### Q12: What's the difference between LAZY and EAGER loading? Which should you default to? ⭐

**30-Second Answer:** **LAZY** — fetch only when accessed (proxy). **EAGER** — fetch immediately with parent. **Always default to LAZY**, then opt-in to fetching specific associations via `@EntityGraph` or `JOIN FETCH` per query.

**Deep Dive:**

```java
@Entity
public class Order {
    @ManyToOne(fetch = LAZY)     //  Default to LAZY
    Customer customer;

    @OneToMany(mappedBy = "order", fetch = LAZY)   // already default for @OneToMany
    List<OrderItem> items;

    @ManyToOne(fetch = EAGER)    //  Default for @ManyToOne / @OneToOne
    Address shipping;
}
```

**Why EAGER is dangerous:**
- Hidden cost on every load
- Cascading EAGER loads → entire object graph fetched
- `LazyInitializationException` becomes "everything loaded everywhere"

**Common interview question**: "Where does `LazyInitializationException` come from?"
**Answer**: Accessing a lazy association OUTSIDE the persistence context (after txn closed). Common pattern: lazy load → return to controller → Jackson serializes → tries to access `items` → no session → boom.

**Fix**: Use DTOs in the controller layer; never serialize entities directly.

```java
//  Returns entity — serialization triggers lazy loads in Jackson
@GetMapping("/{id}")
public Order get(@PathVariable Long id) { return repo.findById(id).get(); }

//  DTO — fetch what you need, return immutable
@GetMapping("/{id}")
public OrderDto get(@PathVariable Long id) {
    Order o = repo.findByIdWithItems(id).get();
    return OrderDto.from(o);   // explicit, type-safe
}
```

**Common Follow-Ups:**
- "Open Session In View (OSIV) — what is it?" → Spring keeps session open during HTTP request → lazy loads work in controller. **Spring Boot enables by default — BAD practice in prod**. Disable: `spring.jpa.open-in-view=false`
- "What's `@DynamicUpdate`?" → Hibernate generates UPDATE only for changed columns (vs all columns)

**Red Flag If You Say:** "Make everything EAGER for simplicity." (Performance killer; loads entire object graph on every query.)

---

## Top 5 from This File (must-know — asked in nearly every senior Spring interview)

1. **Q01** — Propagation levels (esp. REQUIRES_NEW)
2. **Q03** — Self-invocation problem
3. **Q04** — N+1 query problem
4. **Q05** — Optimistic vs Pessimistic locking
5. **Q12** — LAZY vs EAGER + OSIV

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| Propagation REQUIRED              | Default — joins or creates txn   |
| Propagation REQUIRES_NEW          | Suspends outer, creates new      |
| Isolation READ_COMMITTED          | Postgres default                 |
| Isolation REPEATABLE_READ         | MySQL InnoDB default             |
| Self-invocation                   | Same-class call bypasses proxy   |
| N+1 fix                           | JOIN FETCH / @EntityGraph / DTO  |
| Optimistic lock                   | @Version + retry on conflict      |
| Pessimistic lock                  | @Lock(PESSIMISTIC_WRITE)         |
| Atomic SQL                        | UPDATE ... WHERE stock >= qty    |
| LAZY default                      | Use DTOs to avoid LazyInit issues|
| OSIV                              | DISABLE in production            |
| Bulk insert                       | batch_size + flush/clear loop    |
| L1 cache                          | Per-transaction, always on       |
| L2 cache                          | Off by default; use for ref data |
| @Modifying                        | Required for UPDATE/DELETE query |
| save() vs saveAndFlush()          | Latter forces INSERT immediately |
| persist() vs merge()              | merge for detached entities      |
+----------------------------------+----------------------------------+
```
