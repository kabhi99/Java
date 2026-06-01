# **JAVA CONCURRENCY & RACE CONDITIONS — COMPLETE GUIDE**

### TABLE OF CONTENTS

1. What is a Race Condition (with classic bugs)
2. Synchronization Primitives (`synchronized`, `Lock`, `volatile`, atomics)
3. Concurrent Collections (industry standard)
4. Deadlocks, Livelocks, Starvation
5. Thread Pools & Executors
6. Patterns: Producer-Consumer, Read-Write, Double-Checked Locking
7. Common Interview Bugs & Fixes
8. Golden Rules & Interview Cheat Codes

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

**RULE**: Use `volatile` for **single read or single write** of a flag/reference. For **read-modify-write**, use atomics or locks.

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
