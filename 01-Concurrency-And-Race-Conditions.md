# **JAVA CONCURRENCY & RACE CONDITIONS — COMPLETE GUIDE**

### TABLE OF CONTENTS

1. What is a Race Condition (with classic bugs)
2. Synchronization Primitives (`synchronized`, `Lock`, `volatile`, atomics)
3. Concurrent Collections (industry standard)
4. Deadlocks, Livelocks, Starvation
5. Thread Pools & Executors
6. Patterns: Producer-Consumer, Read-Write, Double-Checked Locking
7. Common Interview Bugs & Fixes
8. Golden Rules
9. Interview Cheat Codes + Immutability
10. **Concurrency in Distributed Systems** (JVM locks vs DB locks vs distributed locks)

---

## PART 1: WHAT IS A RACE CONDITION

 **DEFINITION**: When the outcome depends on the **interleaving** of thread execution. Two threads access shared state, at least one writes, and there's no synchronization.

### **The Classic Counter Bug**

```java
class Counter {
    private int count = 0;
    public void increment() { count++; }   //  RACE CONDITION
    public int get() { return count; }
}
```

**WHY IT'S BROKEN:**
`count++` is **3 operations**, not 1:
```
1. Read count       (load from memory)
2. Add 1            (in CPU register)
3. Write count      (store to memory)
```

**INTERLEAVING:**
```
Thread A: read count=5
Thread B: read count=5       ← both read same value!
Thread A: add 1 → 6
Thread B: add 1 → 6
Thread A: write count=6
Thread B: write count=6
Result: count=6 (should be 7!) → LOST UPDATE
```

### **The Three Fixes (Choose Based on Need)**

```java
//  FIX 1: synchronized (simple, locks the whole object)
class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

//  FIX 2: AtomicInteger (lock-free, faster for simple ops)
class Counter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); }
    public int get() { return count.get(); }
}

//  FIX 3: ReentrantLock (more control: tryLock, fair, interruptible)
class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();
    public void increment() {
        lock.lock();
        try { count++; }
        finally { lock.unlock(); }   // ALWAYS in finally!
    }
}
```

 **MEMORY AID**: "Read-Modify-Write needs Atomic OR Lock"

---

## PART 2: SYNCHRONIZATION PRIMITIVES

### **2.1 `synchronized`**

Two flavors:

```java
//  Method-level: locks on 'this'
public synchronized void method() { ... }

//  Block-level: locks on any object (finer control)
public void method() {
    synchronized (lockObject) { ... }
}

//  Static methods: locks on the CLASS object
public static synchronized void method() { ... }   // locks ClassName.class
```

**KEY INSIGHT (REENTRANT):**
A thread holding a lock can re-acquire the same lock without deadlocking:
```java
synchronized void a() { b(); }   // OK — same thread, same lock
synchronized void b() { ... }
```

### **2.2 `volatile`**

 **What it does**: Guarantees **visibility** across threads (write by one thread is immediately visible to others). **Does NOT** guarantee atomicity.

> Think of it as: "always read fresh from main memory, always write through to main memory" — no thread-local CPU caching. **NOT** "this whole operation is one atomic step."

```java
//  CORRECT use: flag for stopping a thread
class Worker implements Runnable {
    private volatile boolean running = true;
    public void run() {
        while (running) { doWork(); }
    }
    public void stop() { running = false; }   // visible to worker thread immediately
}

//  WRONG use: count++ is still a race even if volatile!
private volatile int count = 0;
count++;   //  Read-modify-write — volatile doesn't help
```

#### **GOLDEN RULE: `volatile` vs Atomic / Lock**

> Use `volatile` for **single read or single write** of a flag/reference.
> For **read-modify-write**, use atomics or locks.

**"Single read" means**: just read the variable, no computation based on it:
```java
if (shutdown) return;   //  read once, decide
```

**"Single write" means**: just assign the variable, no logic depending on current value:
```java
shutdown = true;         //  write once, done
```

**"Read-modify-write" means**: 3 separate steps (read → compute → write) that can be interleaved:
```java
count++;                 //  1) READ count  2) ADD 1  3) WRITE count
balance -= amount;       //  same — 3 steps
if (x == null) x = ...;  //  same — check, decide, write
```

**Why `volatile` fails for read-modify-write (count=10, both threads call increment()):**
```
Time | Thread A             | Thread B             | count in memory
-----|----------------------|----------------------|----------------
  1  | reads count = 10     |                      | 10
  2  |                      | reads count = 10     | 10
  3  | computes 10 + 1 = 11 |                      | 10
  4  |                      | computes 10 + 1 = 11 | 10
  5  | writes 11            |                      | 11
  6  |                      | writes 11            | 11   ← BUG! lost +1
```

Expected: `count = 12`. Actual: `count = 11`. `volatile` didn't help — both threads correctly read the latest value (10) and correctly wrote their new value (11). The 3 steps weren't atomic as a whole.

**Decision table:**
```
+----------------------------------+----------+--------------+
| Operation                         | volatile | atomic/lock  |
+----------------------------------+----------+--------------+
| boolean shutdown = true;          |  ENOUGH  | not needed   |
| if (shutdown) return;             |  ENOUGH  | not needed   |
| config = new Config(...);         |  ENOUGH  | not needed   |
| (replace immutable object ref)    |          |              |
+----------------------------------+----------+--------------+
| count++;                          |  FAILS   |  needed     |
| count = count + 1;                |  FAILS   |  needed     |
| if (x == null) x = new X();       |  FAILS   |  needed     |
| balance -= amount;                |  FAILS   |  needed     |
+----------------------------------+----------+--------------+
```

#### **The 4 classic patterns where `volatile` IS the right tool**

```java
// 1. SHUTDOWN FLAG — single read in loop, single write to stop
class Worker {
    private volatile boolean running = true;
    public void run() { while (running) doWork(); }   // single read
    public void stop() { running = false; }            // single write
}

// 2. PUBLISHING AN IMMUTABLE OBJECT — replace whole reference atomically
class ConfigHolder {
    private volatile Config config;                    // Config is immutable
    public Config get() { return config; }             // single read
    public void update(Config c) { this.config = c; }  // single write
}

// 3. DOUBLE-CHECKED LOCKING (DCL) — volatile prevents partial-construction visibility
class Singleton {
    private static volatile Singleton INSTANCE;        //  volatile REQUIRED
    public static Singleton get() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) INSTANCE = new Singleton();
            }
        }
        return INSTANCE;
    }
}

// 4. HAPPENS-BEFORE PUBLICATION — volatile write also publishes prior writes
class Result {
    int value;                                          // not volatile!
    volatile boolean ready;                             // single volatile flag

    void produce() {
        value = heavyCompute();                         // ordinary write
        ready = true;                                   // volatile write → publishes 'value' too
    }
    int consume() {
        while (!ready) {}                               // single volatile read
        return value;                                   //  safe — sees producer's value
    }
}
```

#### **One-line summary**

> **`volatile` = visibility only. The moment your operation depends on the current value, you need atomics or locks.**

This is the #1 interview trick: `volatile int count = 0; count++;` looks correct, compiles fine, even works in single-threaded tests — silently breaks under contention.

### **2.3 Atomic Classes (`java.util.concurrent.atomic`)**

Lock-free, use CAS (Compare-And-Swap) under the hood.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();              // ++counter
counter.getAndIncrement();              // counter++
counter.compareAndSet(expected, new);   // CAS
counter.updateAndGet(x -> x * 2);       // functional update

AtomicReference<User> userRef = new AtomicReference<>(user);
userRef.compareAndSet(oldUser, newUser);

//  ADDER (for high contention — faster than AtomicInteger)
LongAdder hits = new LongAdder();
hits.increment();   // each thread updates its own cell, sum at the end
long total = hits.sum();
```

**WHEN TO USE WHICH:**
- Single value, low contention → `AtomicInteger/Long/Reference`
- Counters under high contention → `LongAdder` / `DoubleAdder`
- Need block of code atomic → `synchronized` / `Lock`

### **2.4 `ReentrantLock` (when `synchronized` isn't enough)**

```java
ReentrantLock lock = new ReentrantLock();   // or new ReentrantLock(true) for FAIR

if (lock.tryLock(2, TimeUnit.SECONDS)) {       // timed acquire — avoid deadlock
    try { /* critical section */ }
    finally { lock.unlock(); }
} else {
    // couldn't get lock — degrade gracefully
}

lock.lockInterruptibly();   // responds to thread.interrupt()
```

**WHY USE `ReentrantLock` OVER `synchronized`?**
| Need | Use |
|---|---|
| Simple mutual exclusion | `synchronized` |
| **Timeout** on acquire | `ReentrantLock.tryLock(timeout)` |
| **Interruptible** wait | `lockInterruptibly()` |
| **Fair** ordering (FIFO) | `new ReentrantLock(true)` |
| Multiple **condition** variables | `lock.newCondition()` |

#### **DEEP DIVE: The 3 ReentrantLock Superpowers**

##### **A. Interruptible wait — `lockInterruptibly()`**

**The problem with `synchronized` / `lock.lock()`**: Once a thread is **waiting** for a lock, it CANNOT be cancelled. Even calling `thread.interrupt()` does nothing — if the lock holder is stuck forever, the waiter is stuck forever too.

```java
//  synchronized — UNINTERRUPTIBLE wait
public void process() {
    synchronized (lock) { doWork(); }    //  if blocked here, interrupt does NOTHING
}

//  ReentrantLock — interruptible
public void process() throws InterruptedException {
    lock.lockInterruptibly();             //  can bail out while waiting
    try { doWork(); }
    finally { lock.unlock(); }
}
```

**Real-world: graceful shutdown**
```java
@Service
public class OrderProcessor {
    private final ReentrantLock lock = new ReentrantLock();

    public void process(Order o) throws InterruptedException {
        lock.lockInterruptibly();        // SIGTERM can cancel mid-wait
        try { saveToDb(o); }
        finally { lock.unlock(); }
    }

    @PreDestroy
    void shutdown() { workers.forEach(Thread::interrupt); }   // waiters bail out
}
```

Without `lockInterruptibly()`, `kubectl delete pod` would wait the full `terminationGracePeriodSeconds` because waiters can't be cancelled.

> **RULE**: Use `lockInterruptibly()` for anything that should respect cancellation — shutdown, request timeout, user-cancelled job.

##### **B. Fair ordering (FIFO) — `new ReentrantLock(true)`**

**Default locks are UNFAIR**: the JVM grants the lock to any waiting thread, often the **newest** one (it's already running on the CPU — cheap to wake). This causes **starvation** — an early waiter sits there for minutes while late arrivals jump the queue.

```java
ReentrantLock unfair = new ReentrantLock();        // default = unfair (fast)
ReentrantLock fair   = new ReentrantLock(true);    // FIFO (slower, no starvation)
```

**How unfairness causes starvation:**
```
Time | Queue            | Released | JVM grants to
-----|------------------|----------|-----------------
  1  | [T1, T2, T3]     | yes      | T3  ← newest jumps queue
  2  | [T1, T2, T4]     | yes      | T4
  3  | [T1, T2, T5]     | yes      | T5  ← T1 STILL waiting!
```

**With fair lock:**
```
Time | Queue            | Released | Granted to
-----|------------------|----------|-----------------
  1  | [T1, T2, T3]     | yes      | T1  ← FIFO
  2  | [T2, T3, T4]     | yes      | T2
```

**Trade-off**: Fair locks are **~10× slower** under contention (must coordinate ordered queue, no opportunistic grabs). Only use fair when you've **observed actual starvation** in metrics.

> **RULE**: Default unfair = faster. Use `new ReentrantLock(true)` only when starvation is a real problem.

##### **C. Multiple condition variables — `lock.newCondition()`**

**The problem with `Object.wait()/notify()`**: Only **ONE wait queue per object**. So `notifyAll()` wakes everyone, even threads waiting for a different condition — they recheck, find their condition still false, go back to sleep. **Wasted CPU + lost-wakeup bugs.**

`Condition` lets you have **multiple independent wait queues** under the same lock.

**Classic example: Bounded Buffer (Producer-Consumer)** — producers block when **full**, consumers block when **empty**. Different conditions.

**BAD — one shared wait queue (`synchronized`):**
```java
public synchronized void put(T item) throws InterruptedException {
    while (q.size() == capacity) wait();   // one shared queue
    q.add(item);
    notifyAll();   //  wakes producers AND consumers — producers go right back to sleep
}

public synchronized T take() throws InterruptedException {
    while (q.isEmpty()) wait();
    T item = q.poll();
    notifyAll();   //  wakes everyone again
    return item;
}
```

50 producers + 50 consumers → every `notifyAll()` wakes all 100; most go right back to sleep.

**GOOD — two separate Conditions:**
```java
private final ReentrantLock lock     = new ReentrantLock();
private final Condition     notFull  = lock.newCondition();   //  producer queue
private final Condition     notEmpty = lock.newCondition();   //  consumer queue

public void put(T item) throws InterruptedException {
    lock.lock();
    try {
        while (q.size() == capacity) notFull.await();   // producers wait here
        q.add(item);
        notEmpty.signal();                              //  wake ONE consumer only
    } finally { lock.unlock(); }
}

public T take() throws InterruptedException {
    lock.lock();
    try {
        while (q.isEmpty()) notEmpty.await();           // consumers wait here
        T item = q.poll();
        notFull.signal();                                //  wake ONE producer only
        return item;
    } finally { lock.unlock(); }
}
```

**What changed:**
```
+----------------------------+--------------------+----------------------+
| Aspect                      | synchronized       | Condition            |
+----------------------------+--------------------+----------------------+
| Wait queues                 | 1 (shared)         | 2 (separate)         |
| notifyAll() wakes           | Everyone           | Only matching group  |
| signal() wakes              | n/a                | Exactly ONE thread   |
| CPU waste                   | High (false wakes) | None                 |
+----------------------------+--------------------+----------------------+
```

**Mental model:**
```
synchronized:                       Condition (2 of them):
+-----------+                       +----------+  +-----------+
| Everyone  |                       | Producers|  | Consumers |
|  waiting  |                       |  waiting |  |  waiting  |
+-----------+                       +----------+  +-----------+
     ↑                                   ↑              ↑
 notifyAll() wakes ALL          notFull.signal()  notEmpty.signal()
                                (wakes 1 producer) (wakes 1 consumer)
```

> **RULE**: Use multiple `Condition`s whenever threads are waiting for **different** events under the same lock. Otherwise you wake everyone needlessly.

##### **Quick decision: `synchronized` vs `ReentrantLock`**

```
+-----------------------------------------+-----------------+-----------------+
| Feature                                  | synchronized    | ReentrantLock   |
+-----------------------------------------+-----------------+-----------------+
| Auto-release on exit                     |  yes            | manual unlock() |
| Interruptible wait                       |  no             |  yes            |
| Try-acquire with timeout                 |  no             |  yes            |
| Fair ordering                            |  no             |  optional       |
| Multiple wait queues (Condition)         |  no             |  yes            |
| Lock across method boundaries            | hard            | easy            |
| Performance (uncontended)                | great           | great           |
| Simplicity                               | best            | more code       |
+-----------------------------------------+-----------------+-----------------+
```

**Default**: `synchronized` for simple cases. Switch to `ReentrantLock` when you need ANY of the 3 superpowers above.

### **2.5 `ReadWriteLock` (many readers, few writers)**

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// Read (multiple threads can hold simultaneously)
readLock.lock();
try { return cache.get(key); }
finally { readLock.unlock(); }

// Write (exclusive)
writeLock.lock();
try { cache.put(key, value); }
finally { writeLock.unlock(); }
```

**USE WHEN**: Reads >> Writes (e.g., config cache, lookup table)

### **2.6 `StampedLock` (Java 8+, even faster for read-heavy)**

```java
StampedLock sl = new StampedLock();

// OPTIMISTIC READ — no lock at all!
long stamp = sl.tryOptimisticRead();
int value = sharedValue;
if (!sl.validate(stamp)) {              // someone wrote during our read
    stamp = sl.readLock();              // fall back to real read lock
    try { value = sharedValue; }
    finally { sl.unlockRead(stamp); }
}
```

---

## PART 3: CONCURRENT COLLECTIONS

 **GOLDEN RULE**: Never use `HashMap`, `ArrayList`, `HashSet` from multiple threads — they will **silently corrupt** or throw `ConcurrentModificationException`.

### **3.1 `ConcurrentHashMap` (THE workhorse)**

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("a", 1);
map.get("a");

//  ATOMIC OPERATIONS (use these, not get-then-put!)
map.putIfAbsent("a", 1);
map.compute("a", (k, v) -> (v == null) ? 1 : v + 1);     // safe counter
map.computeIfAbsent("a", k -> loadFromDB(k));            // safe cache load
map.merge("a", 1, Integer::sum);                          // safe accumulator

//  COMMON BUG (race condition):
if (!map.containsKey(k)) map.put(k, v);   //  check-then-act, not atomic!
// Use: map.putIfAbsent(k, v);            //  atomic
```

### **3.2 Other Concurrent Collections**

```java
CopyOnWriteArrayList<T>      // read-mostly list (e.g., listener registry)
ConcurrentLinkedQueue<T>     // unbounded, lock-free queue
LinkedBlockingQueue<T>       // bounded, blocking — producer/consumer
ArrayBlockingQueue<T>        // bounded, fixed-size, blocking
ConcurrentSkipListMap<K,V>   // sorted concurrent map (instead of TreeMap)
ConcurrentSkipListSet<T>     // sorted concurrent set
```

### **3.3 Synchronized Wrappers vs Concurrent Collections**

```java
//  AVOID — coarse lock, slow
Map<K,V> m = Collections.synchronizedMap(new HashMap<>());

//  PREFER — fine-grained, scalable
Map<K,V> m = new ConcurrentHashMap<>();
```

---

## PART 4: DEADLOCKS, LIVELOCKS, STARVATION

### **4.1 Deadlock — Threads waiting forever**

**CLASSIC EXAMPLE (lock ordering bug):**
```java
//  DEADLOCK
void transfer(Account a, Account b, int amt) {
    synchronized (a) {
        synchronized (b) {           // T1: lock A→B, T2: lock B→A → DEADLOCK
            a.debit(amt); b.credit(amt);
        }
    }
}
```

**FIX 1 — Global lock ordering** (always acquire in consistent order):
```java
void transfer(Account a, Account b, int amt) {
    Account first  = a.id < b.id ? a : b;
    Account second = a.id < b.id ? b : a;
    synchronized (first) {
        synchronized (second) {
            a.debit(amt); b.credit(amt);
        }
    }
}
```

**FIX 2 — `tryLock` with timeout** (avoid waiting forever):
```java
if (a.lock.tryLock(1, SECONDS)) {
    try {
        if (b.lock.tryLock(1, SECONDS)) {
            try { /* transfer */ }
            finally { b.lock.unlock(); }
        }
    } finally { a.lock.unlock(); }
}
```

**FOUR COFFMAN CONDITIONS** for deadlock (break any one to prevent):
1. **Mutual exclusion** — resource held exclusively
2. **Hold and wait** — hold one, wait for another
3. **No preemption** — can't force release
4. **Circular wait** — A waits for B, B waits for A

### **4.2 Livelock — Threads keep changing state without progress**
Two threads keep yielding to each other politely → no work gets done. Fix: introduce randomness/backoff.

### **4.3 Starvation — A thread never gets CPU/lock**
Caused by unfair locks, priority inversion. Fix: use **fair locks** (`new ReentrantLock(true)`).

---

## PART 5: THREAD POOLS & EXECUTORS

 **GOLDEN RULE**: NEVER use `new Thread(...).start()` in production. ALWAYS use thread pools.

```java
//  INDUSTRY STANDARD — define explicit pool
ExecutorService executor = new ThreadPoolExecutor(
    4,                                       // corePoolSize
    10,                                      // maxPoolSize
    60L, TimeUnit.SECONDS,                   // keepAlive for idle threads
    new LinkedBlockingQueue<>(100),          // bounded queue (BACKPRESSURE!)
    new ThreadFactoryBuilder()
        .setNameFormat("payment-worker-%d")  // named threads (debugging!)
        .setUncaughtExceptionHandler(...)
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure: caller does the work
);

Future<String> f = executor.submit(() -> doWork());
String result = f.get(5, TimeUnit.SECONDS);   // timed get!

executor.shutdown();                            // graceful
executor.awaitTermination(30, TimeUnit.SECONDS);
```

### **The Four Built-in Pools (and why most are dangerous)**

```java
Executors.newFixedThreadPool(n)        //  Unbounded queue → OOM under load
Executors.newCachedThreadPool()        //  Unbounded threads → thread explosion
Executors.newSingleThreadExecutor()    //  OK for serialized work
Executors.newScheduledThreadPool(n)    //  OK for scheduled tasks
```

**ALWAYS** prefer `new ThreadPoolExecutor(...)` with explicit bounds for production.

### **`CompletableFuture` (modern async)**

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(id), executor)
    .thenApply(user -> enrich(user))
    .thenCompose(user -> fetchOrders(user))    // chain async
    .thenCombine(otherFuture, (a, b) -> merge(a, b))
    .exceptionally(ex -> fallback(ex))
    .orTimeout(2, TimeUnit.SECONDS)            // Java 9+
    .whenComplete((res, ex) -> log(res, ex));
```

---

## PART 6: KEY PATTERNS

### **6.1 Producer-Consumer (BlockingQueue)**

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);   // bounded!

// Producer
queue.put(task);                                // blocks if full

// Consumer
while (true) {
    Task t = queue.take();                      // blocks if empty
    process(t);
}
```

### **6.2 Double-Checked Locking (Singleton)**

```java
class Singleton {
    private static volatile Singleton instance;   // volatile is REQUIRED

    public static Singleton getInstance() {
        if (instance == null) {                   // 1st check (no lock — fast path)
            synchronized (Singleton.class) {
                if (instance == null) {           // 2nd check (inside lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**WHY `volatile`?** Without it, a partially-constructed object may be visible to other threads (publication race).

 **MODERN BETTER WAY** — initialization-on-demand (no locks needed):
```java
class Singleton {
    private Singleton() {}
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

### **6.3 Safe Lazy Cache**

```java
//  BAD — race condition (two threads both compute & put)
V get(K key) {
    V v = cache.get(key);
    if (v == null) { v = expensiveLoad(key); cache.put(key, v); }
    return v;
}

//  GOOD — atomic, only one thread computes
V get(K key) {
    return cache.computeIfAbsent(key, this::expensiveLoad);
}
```

---

## PART 7: COMMON INTERVIEW BUGS

| Bug | Symptom | Fix |
|---|---|---|
| `count++` without sync | Lost updates | `AtomicInteger` or `synchronized` |
| `if-then-put` on map | Duplicate writes | `putIfAbsent` / `computeIfAbsent` |
| Iterating `HashMap` while modifying | `ConcurrentModificationException` | Use `ConcurrentHashMap` |
| Forgetting `unlock()` in `try-catch` | Permanent lock leak → deadlock | Always `unlock()` in `finally` |
| `volatile` on `count++` | Still races | Use atomics or locks |
| Locking on `String` literal / `Integer` | Hidden shared lock across classes | Lock on private `final Object` |
| `Executors.newFixedThreadPool(n)` | OOM (unbounded queue) | `new ThreadPoolExecutor(...)` with bounded queue |
| `new Thread().start()` in loop | Thread explosion | Use a thread pool |
| Holding lock during slow I/O | Throughput tanks | Release lock before I/O; copy data out |
| Different lock for read vs write | Visibility issues | Same lock for both, or `volatile` |

---

## PART 8: GOLDEN RULES

```
+---------------------------------------------------------------------+
|  1. Shared mutable state + multiple threads = MUST synchronize      |
|  2. ALWAYS unlock() in finally{} block                              |
|  3. volatile = visibility only, NOT atomicity                       |
|  4. Read-modify-write needs Atomic or Lock (not volatile)           |
|  5. Prefer ConcurrentHashMap over synchronized HashMap              |
|  6. Use putIfAbsent / computeIfAbsent, NOT containsKey-then-put     |
|  7. Lock acquisition: always SAME ORDER to prevent deadlock         |
|  8. Use tryLock(timeout) when deadlock is possible                  |
|  9. NEVER new Thread() in production — use ThreadPoolExecutor       |
| 10. Bounded queues + sensible reject policy = backpressure          |
| 11. Lock the SMALLEST scope possible (release ASAP)                 |
| 12. NEVER do I/O inside a lock if you can avoid it                  |
| 13. Name your threads (ThreadFactoryBuilder) for debugging          |
| 14. Use CompletableFuture for async chains, not raw Future          |
| 15. Make objects IMMUTABLE — no concurrency bugs by construction    |
+---------------------------------------------------------------------+
```

---

## PART 9: INTERVIEW CHEAT CODES

> "Since `count++` is a read-modify-write, I'll use `AtomicInteger.incrementAndGet()` for lock-free atomicity."

> "I'm using `ConcurrentHashMap.computeIfAbsent` to ensure only one thread loads the value, avoiding the classic check-then-act race."

> "I'll acquire locks in a globally consistent order (by account ID) to prevent the circular-wait condition for deadlock."

> "I'm bounding the queue at 1000 with a `CallerRunsPolicy` reject handler — this gives natural backpressure to upstream producers."

> "For this read-heavy cache, I'll use `ReadWriteLock` to allow concurrent readers while serializing writers — or `StampedLock` for optimistic reads on the hot path."

> "I avoid `synchronized` on shared library objects like `String` literals or boxed `Integer` — I lock on a private `final Object` to keep the lock scope contained."

> "I'm using `CompletableFuture.orTimeout(2, SECONDS)` so a hanging downstream call can't block the calling thread indefinitely."

---

## IMMUTABILITY (THE BEST CONCURRENCY)

```java
//  Immutable — thread-safe by construction
public final class Money {
    private final long cents;
    private final String currency;
    public Money(long cents, String currency) { this.cents = cents; this.currency = currency; }
    public Money add(Money other) { return new Money(cents + other.cents, currency); }   // returns NEW instance
}
```

**Rules for immutability:**
1. Class `final` (no subclassing)
2. All fields `private final`
3. No setters
4. Defensive copies for mutable fields (e.g., `List`, `Date`)
5. Operations return **new** instances

> If you can make it immutable, you don't need any other concurrency primitive.

---

## PART 10: CONCURRENCY IN DISTRIBUTED SYSTEMS

> "Do we still use `synchronized` / `ReentrantLock` in modern microservices?" — common interview question.
>
> **Short answer**: Yes, but for different problems than people think. `synchronized` solves in-JVM thread safety. In distributed systems, contention usually lives in the **DB or Redis**, not in app memory. So JVM locks are RARELY the right tool across pods.

### **10.1 The 3 Different "Concurrency Problems"**

```
+-----------------------------------------+------------------------------+
| Problem                                  | Tool                         |
+-----------------------------------------+------------------------------+
| 1. Multiple THREADS in 1 JVM             | synchronized /                |
|    (singleton bean with state, batch)    | ReentrantLock / Atomic       |
+-----------------------------------------+------------------------------+
| 2. Multiple PODS sharing DB rows         | DB (optimistic/pessimistic   |
|    (most common in microservices)        | lock) OR atomic SQL          |
+-----------------------------------------+------------------------------+
| 3. Multiple PODS coordinating WITHOUT DB | Distributed lock:            |
|    (rare — e.g., schedule on 1 pod only) | ShedLock / Redisson / etcd   |
+-----------------------------------------+------------------------------+
```

**The big insight**: In a stateless microservice, **problem #1 barely exists** — there's no shared in-memory state worth locking. That's why `synchronized` / `ReentrantLock` usage has dropped dramatically.

### **10.2 Where `synchronized` / `ReentrantLock` STILL Make Sense (2026)**

####  In-process state in a singleton bean
```java
@Service
public class MetricsAggregator {
    private final LongAdder requests = new LongAdder();
    public void record() { requests.increment(); }   // hot path, lock-free
}
```

####  Coordinating threads within a batch job
```java
public class BatchProcessor {
    private final Semaphore parallelism = new Semaphore(10);
    private final ReentrantLock outputLock = new ReentrantLock();

    public void process(File f) throws InterruptedException {
        parallelism.acquire();
        try {
            byte[] result = doWork(f);
            outputLock.lock();
            try { writer.write(result); }
            finally { outputLock.unlock(); }
        } finally { parallelism.release(); }
    }
}
```

####  Library / framework code
Spring's `DefaultSingletonBeanRegistry`, Tomcat's connection pool, Jackson's caches — all use `synchronized` internally. It's not dead, just specialized.

### **10.3 Where `synchronized` is the WRONG Tool**

####  Coordinating across pods (the #1 mistake)
```java
//  Each pod has its OWN copy of this map → "lock" only blocks threads in ONE pod
@Service
public class OrderService {
    private final Set<String> processingIds = ConcurrentHashMap.newKeySet();

    public synchronized void process(String orderId) {
        if (processingIds.contains(orderId)) return;   //  Other pods don't see this!
        processingIds.add(orderId);
        // ...
    }
}
//  10 pods → same orderId can still be processed 10 times in parallel
```

####  Preventing double-charge across replicas
```java
//  Wrong tool
@Transactional
public synchronized void charge(Order o) { ... }   //  synchronized doesn't span pods
```

** Right tool — DB-level idempotency:**
```java
@Transactional
public Payment charge(Order o) {
    try {
        return paymentRepo.save(new Payment(o.getIdempotencyKey(), ...));
    } catch (DataIntegrityViolationException e) {
        return paymentRepo.findByIdempotencyKey(o.getIdempotencyKey());   // already done
    }
}
// ALTER TABLE payments ADD CONSTRAINT uq_idempotency UNIQUE (idempotency_key);
```

####  Inventory decrement across pods
```java
//  synchronized only blocks threads in same JVM
public synchronized void decrementStock(Long id) { ... }

//  atomic SQL handles cross-pod concurrency naturally
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - 1 WHERE p.id = :id AND p.stock >= 1")
int decrement(@Param("id") Long id);
```

### **10.4 What We Actually Use in Distributed Systems**

####  PATTERN 1: Push concurrency to the DB (90% of cases)

The DB already has locking — use it.

```java
//  Optimistic — DB-managed via @Version
@Entity public class Product { @Version Long version; }

//  Pessimistic — DB-managed via SELECT FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product lockForUpdate(@Param("id") Long id);

//  Atomic SQL — no app lock needed
@Modifying
@Query("UPDATE Account SET balance = balance - :amt WHERE id = :id AND balance >= :amt")
int debit(@Param("id") Long id, @Param("amt") BigDecimal amt);

//  Unique constraint as a "lock"
ALTER TABLE payments ADD CONSTRAINT uq_idempotency UNIQUE (idempotency_key);
```

> **First instinct in distributed systems: "Can the DB do this for me?" — usually yes.**

####  PATTERN 2: Distributed locks (only when truly needed)

When state is NOT in the DB but you still need cross-pod coordination:
- "Only ONE pod should run this cron job"
- "Only ONE pod should rebuild the search index"
- "Only ONE pod should send the welcome email"

```java
//  ShedLock — for scheduled jobs (DB-backed, simple)
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "nightlyReport", lockAtMostFor = "30m", lockAtLeastFor = "5m")
public void generateReport() { ... }   //  Only one pod runs this

//  Redisson — general distributed lock
@Autowired RedissonClient redisson;

public void rebuildIndex() {
    RLock lock = redisson.getLock("index-rebuild");
    if (!lock.tryLock(0, 10, MINUTES)) return;   // another pod has it
    try { searchIndex.rebuild(); }
    finally { lock.unlock(); }
}
```

####  PATTERN 3: Idempotency keys (don't lock — design out the contention)

```java
@PostMapping
public Order create(@RequestHeader("Idempotency-Key") String key,
                    @RequestBody CreateOrderRequest req) {
    return orderService.createIdempotent(key, req);   // safe across pods + retries
}
```

####  PATTERN 4: Single-writer per partition (Kafka, CQRS)

Kafka with a partition key = `orderId` → all events for one order go to the same consumer → **no cross-pod contention by design**.

```
Order 123 events ──┐
Order 124 events ──┼─► Kafka partition 0 ──► Consumer A   (single writer for orders 123, 124)
                   │
Order 125 events ──┼─► Kafka partition 1 ──► Consumer B   (single writer for order 125)
```

No locks needed — partitioning eliminates contention at the architectural level.

### **10.5 The DANGER: Distributed Locks Done Wrong**

Martin Kleppmann's "[How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)" — Redis-based locks (RedLock) can break under network partitions, GC pauses, or clock drift.

**Real failure mode:**
```
Pod A: acquires lock, starts work
Pod A: hits a 30-second GC pause
Lock TTL expires (30s)
Pod B: acquires lock, starts the same work
Pod A: wakes up, "still holding lock", finishes work
RESULT: Both pods committed → split-brain
```

**Mitigations:**
1. **Fencing tokens** — DB rejects writes with stale (lower) tokens
2. **Idempotency** — even if both pods run, the result is the same
3. **Don't use distributed locks for CORRECTNESS** — use them only for OPTIMIZATION
   ("avoid duplicate work" is OK; "guarantee uniqueness" is NOT)

> **Distributed locks for correctness = scary. Use DB constraints / idempotency / single-writer partitioning instead.**

### **10.6 Decision Tree (use this in interviews)**

```
Need to coordinate concurrent access?
│
├── Within ONE JVM (singleton bean state)?
│   ├── Simple counter/flag?              → AtomicInteger / volatile
│   ├── Map access?                       → ConcurrentHashMap.compute*
│   ├── Block of code?                    → synchronized
│   └── Need timeout/interrupt/fair/      → ReentrantLock
│       multiple Conditions?
│
├── Across PODS, state in DB?
│   ├── Low contention?                   → @Version (optimistic)
│   ├── High contention?                  → SELECT FOR UPDATE
│   ├── Just decrementing/updating?       → Atomic UPDATE WHERE ...
│   ├── Preventing duplicate inserts?     → UNIQUE constraint
│   └── Idempotency?                      → idempotency_key UNIQUE
│
├── Across PODS, state NOT in DB?
│   ├── Scheduled job once cluster-wide?  → ShedLock
│   ├── Coordinator pattern?              → Zookeeper / etcd leader election
│   ├── "Just avoid duplicate work"?      → Redisson RLock (+ idempotency!)
│   └── Need strict correctness?          → Move state to DB and use DB locks
│
└── Avoiding contention entirely?
    ├── Partition by key (Kafka)
    ├── Single-writer per aggregate (CQRS)
    └── Immutable data + event sourcing
```

### **10.7 Frequency in a Real Spring Boot Microservice (2026)**

```
+--------------------------------------------------------------+------+
| Mechanism                                                     | Use  |
+--------------------------------------------------------------+------+
| DB-level locking (@Version, SELECT FOR UPDATE, atomic SQL)    | ~70% |
| Idempotency keys + unique constraints                         | ~15% |
| Stateless design / Kafka partitioning                         | ~10% |
| ShedLock for scheduled jobs                                   |  ~3% |
| ConcurrentHashMap.compute* in app                             |  ~1% |
| synchronized / ReentrantLock                                  |  ~1% | ← rarely
| Redisson distributed lock                                     | <1%  |
+--------------------------------------------------------------+------+
```

### **10.8 TL;DR — Senior Answer for "Do you use synchronized?"**

| Question | Answer |
|---|---|
| Is `synchronized` dead? | **No** — still used for in-JVM thread safety in library code, caches, and batch jobs |
| Do I use it in microservices? | **Rarely** — services are stateless, contention lives in DB/Redis |
| What replaced it for cross-pod? | **DB constraints, atomic SQL, optimistic locking, idempotency keys** |
| When do I need a distributed lock? | When state isn't in DB AND you need cross-pod coordination (scheduled jobs, leader election) |
| Best distributed lock for Spring? | **ShedLock** (jobs) or **Redisson** (general) — but always pair with idempotency |
| Single best advice? | **Push concurrency to the DB.** Locks at the app layer are usually a sign of bad design |

### **10.9 Interview Cheat Codes**

> "In a stateless microservice the in-memory state isn't shared across pods, so `synchronized` only protects one JVM. I push concurrency to the DB layer — `@Version` for optimistic locking, `SELECT FOR UPDATE` for hot rows, or an atomic SQL UPDATE with a WHERE guard for inventory decrement."

> "For cross-pod scheduled jobs I use ShedLock — it's a thin DB-row-based lock that ensures only one pod runs the cron at a time. For general distributed locks I'd use Redisson, but I always pair it with an idempotency mechanism because distributed locks aren't safe for correctness on their own (GC pauses can break the TTL guarantee)."

> "When designing for high write contention I'd avoid locks altogether — partition by key in Kafka so one aggregate maps to one consumer, giving single-writer semantics by design. That's how Uber and Stripe scale order/payment processing."

> "`synchronized` and `ReentrantLock` are still useful inside one JVM — caches, metrics aggregators, in-process queues — but they shouldn't appear in your service-layer methods unless you've truly got JVM-local mutable state."

>  **"In a well-designed distributed system, you should rarely need application-level locks. If you do, that's a hint your state is in the wrong place."**
