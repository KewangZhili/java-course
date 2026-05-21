# Module 19 — Executors, Locks, Atomics, Concurrent Collections

> Goal: use the high-level concurrency toolkit. `ExecutorService` for managing pools of threads, `ReentrantLock` and `Condition` for finer control than `synchronized`, atomics for lock-free counters, and the concurrent collections that survive contention. After this module you have the tools to build correct multi-threaded code.

These are all in `java.util.concurrent` (often imported as `j.u.c`). The whole package is the standard library's blessed answer to the JMM hazards from Module 18.

---

## 1. `ExecutorService` — the right way to use threads

Don't `new Thread()` per task. Submit tasks to a *thread pool*; the pool reuses threads.

```java
import java.util.concurrent.*;

ExecutorService pool = Executors.newFixedThreadPool(8);

pool.submit(() -> System.out.println("hi from pool"));
Future<Integer> f = pool.submit(() -> 42);
int v = f.get();      // blocks until done

pool.shutdown();      // accept no more tasks; finish in-flight
pool.awaitTermination(10, TimeUnit.SECONDS);
```

### 1.1 Factories on `Executors`

```java
Executors.newFixedThreadPool(n);            // n threads, unbounded queue
Executors.newSingleThreadExecutor();        // 1 thread
Executors.newCachedThreadPool();            // grows as needed; threads idle 60s; UNBOUNDED
Executors.newScheduledThreadPool(n);        // scheduled / periodic tasks
Executors.newVirtualThreadPerTaskExecutor(); // Java 21+ — one virtual thread per task
```

`newCachedThreadPool` can spawn unbounded threads under load — risky in production. `newFixedThreadPool` is safer; tune `n` to your workload.

### 1.2 `Future` and `submit`

`submit(Callable<T>)` returns `Future<T>`:
```java
Future<Integer> f = pool.submit(() -> heavyComputation());
int result = f.get();                 // blocks; throws ExecutionException on task failure
int r2 = f.get(5, TimeUnit.SECONDS);  // throws TimeoutException
f.cancel(true);                       // attempt to cancel; true = interrupt the thread
f.isDone();
```

`Future.get()` throws `InterruptedException`, `ExecutionException` (wrapping the task's exception), and `TimeoutException`.

For composing async work, prefer `CompletableFuture` (Module 20).

### 1.3 `invokeAll` / `invokeAny`

```java
List<Callable<Integer>> tasks = ...;
List<Future<Integer>> results = pool.invokeAll(tasks);     // wait for all
Integer first = pool.invokeAny(tasks);                      // first to succeed
```

### 1.4 `ScheduledExecutorService`

```java
var sch = Executors.newScheduledThreadPool(2);
sch.schedule(() -> ping(), 1, TimeUnit.MINUTES);
sch.scheduleAtFixedRate(() -> heartbeat(), 0, 30, TimeUnit.SECONDS);
sch.scheduleWithFixedDelay(() -> poll(), 0, 30, TimeUnit.SECONDS);
```

`fixedRate` schedules the next run from the previous *start*; `fixedDelay` from the previous *end*. Use `fixedDelay` for "run no more often than every X seconds".

### 1.5 Shutdown

```java
pool.shutdown();                                   // graceful
boolean done = pool.awaitTermination(10, TimeUnit.SECONDS);
if (!done) pool.shutdownNow();                     // best-effort cancel
```

`shutdown()` returns immediately; the pool keeps running in-flight tasks. Always call shutdown before letting your `main` exit — otherwise non-daemon pool threads keep the JVM alive.

---

## 2. Locks — `ReentrantLock`, `ReadWriteLock`, `StampedLock`

`synchronized` is the simple form; `Lock` types give you finer control.

### 2.1 `ReentrantLock`

Drop-in replacement for `synchronized` with extras:

```java
import java.util.concurrent.locks.ReentrantLock;

private final ReentrantLock lock = new ReentrantLock();

void f() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();              // ALWAYS in finally
    }
}
```

Extras over `synchronized`:
- `tryLock()` / `tryLock(timeout, unit)` — non-blocking or bounded acquisition.
- `lockInterruptibly()` — give up if interrupted.
- Fair mode: `new ReentrantLock(true)` — first-come-first-served (slower, sometimes desired).
- `Condition` objects (next).

### 2.2 `Condition`

Replacement for `wait`/`notify`. Multiple conditions per lock = clear signaling.

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Condition notFull  = lock.newCondition();

void put(T item) throws InterruptedException {
    lock.lock();
    try {
        while (queue.size() == capacity) notFull.await();
        queue.add(item);
        notEmpty.signal();
    } finally { lock.unlock(); }
}

T take() throws InterruptedException {
    lock.lock();
    try {
        while (queue.isEmpty()) notEmpty.await();
        T item = queue.poll();
        notFull.signal();
        return item;
    } finally { lock.unlock(); }
}
```

Always `await()` inside a `while`, not `if` — wakeups can be spurious or due to other producers/consumers.

### 2.3 `ReadWriteLock`

Multiple readers concurrently, exclusive writer:

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
var rl = rw.readLock();
var wl = rw.writeLock();

V read(K key)            { rl.lock(); try { return map.get(key); } finally { rl.unlock(); } }
void write(K key, V val) { wl.lock(); try { map.put(key, val); }   finally { wl.unlock(); } }
```

Useful when reads dominate writes. For most map-shaped uses, `ConcurrentHashMap` (§4) is better.

### 2.4 `StampedLock`

Java 8+, supports an *optimistic read* mode that doesn't take a lock at all:

```java
StampedLock sl = new StampedLock();
long stamp = sl.tryOptimisticRead();
T   data = read();
if (!sl.validate(stamp)) {
    stamp = sl.readLock();
    try { data = read(); } finally { sl.unlockRead(stamp); }
}
```

Faster than `ReadWriteLock` for read-heavy workloads when you can validate after reading. Use when profiling shows you need it; complexity is real.

---

## 3. Atomics — lock-free building blocks

`java.util.concurrent.atomic` provides single-variable atomic operations using CPU `compare-and-swap` (CAS) instructions. No locks needed.

```java
import java.util.concurrent.atomic.*;

AtomicInteger ai = new AtomicInteger(0);
ai.incrementAndGet();           // ++n; returns new value
ai.getAndIncrement();           // n++; returns old value
ai.addAndGet(5);
ai.compareAndSet(0, 1);         // CAS — only sets if current is 0
ai.set(7);
ai.get();

AtomicLong al = new AtomicLong();
AtomicReference<String> ar = new AtomicReference<>("init");
ar.compareAndSet("init", "next");
ar.updateAndGet(s -> s + "!");

AtomicReferenceArray<String> arr = new AtomicReferenceArray<>(10);
```

For high-contention counters, prefer `LongAdder` / `DoubleAdder` — they spread updates across multiple cells, reducing contention:

```java
LongAdder count = new LongAdder();
count.increment();
long total = count.sum();    // approximate; serialized internally
```

Use atomics when:
- Single variable updates need to be atomic across threads.
- You want to avoid lock contention.

CAS-based code can suffer **ABA**: a value changes A→B→A and a CAS thinks nothing happened. `AtomicStampedReference` handles this with a version counter.

---

## 4. Concurrent collections

`java.util.concurrent` provides thread-safe collections far better than wrapping `HashMap` in `synchronized`.

### 4.1 `ConcurrentHashMap`

The workhorse. Lock-free reads, lock-striped writes. Reads always see a *consistent* snapshot of values.

```java
ConcurrentHashMap<String, Integer> m = new ConcurrentHashMap<>();
m.put("a", 1);
m.computeIfAbsent("b", k -> 2);
m.merge("a", 1, Integer::sum);     // atomic increment
m.compute("a", (k, v) -> (v == null ? 0 : v) + 1);
```

`compute*` and `merge` are *atomic* — the function runs holding the bucket's lock. Use them instead of `get` + `put` to avoid races.

Bulk operations:
```java
m.forEach(1, (k, v) -> ...);                 // 1 = parallelism threshold
m.search(1, (k, v) -> v > 10 ? v : null);
m.reduce(1, (k, v) -> v, Integer::sum);
```

`ConcurrentHashMap` does NOT allow `null` keys or values (unlike `HashMap`).

### 4.2 `CopyOnWriteArrayList` / `CopyOnWriteArraySet`

Mutations copy the underlying array; reads are lock-free. Excellent for **read-heavy, write-rare** lists (e.g. listener lists).

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();
listeners.add(...);
for (Listener l : listeners) l.onEvent();   // safe even if another thread adds/removes
```

Bad for write-heavy use (each write copies).

### 4.3 `BlockingQueue`

Thread-safe queues with blocking `put` / `take`:

```java
BlockingQueue<Job> queue = new LinkedBlockingQueue<>();   // unbounded
BlockingQueue<Job> bounded = new LinkedBlockingQueue<>(100);
BlockingQueue<Job> sync = new SynchronousQueue<>();        // capacity 0; hand-off

queue.put(job);          // blocks if full (bounded only)
Job j = queue.take();    // blocks if empty
queue.offer(job);        // returns false if full
queue.poll(1, TimeUnit.SECONDS);
```

Producer-consumer pattern → use `BlockingQueue`. The framework on which `ThreadPoolExecutor` is built.

### 4.4 `ConcurrentSkipListMap` / `ConcurrentSkipListSet`

Thread-safe sorted map/set, replacing `TreeMap`/`TreeSet` for concurrent use.

---

## 5. `ThreadPoolExecutor` — the underlying class

The factory `Executors.newFixedThreadPool(n)` creates a `ThreadPoolExecutor`:

```java
new ThreadPoolExecutor(
    coreSize,                   // threads kept alive even when idle
    maxSize,                    // upper bound under load
    keepAliveTime, TimeUnit,    // idle threads above core die after this
    workQueue,                  // BlockingQueue of pending tasks
    threadFactory,              // how to create threads (naming, daemon, etc.)
    rejectionHandler            // what to do when queue full + max threads
);
```

You'll see this directly when you want:
- Custom thread naming (use `ThreadFactory` from Guava or `Executors.defaultThreadFactory` wrapped):
  ```java
  ThreadFactory f = r -> {
      Thread t = new Thread(r, "my-pool-" + counter.incrementAndGet());
      t.setDaemon(true);
      return t;
  };
  ```
- Bounded queues + a sensible rejection policy (`CallerRunsPolicy` is a common back-pressure choice).
- Different keep-alive behavior.

For most apps, `Executors.newFixedThreadPool(n)` is fine.

---

## 6. Synchronizers — `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`

Coordination utilities.

### 6.1 `CountDownLatch`

Wait for N events to complete. One-shot.

```java
var latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    pool.submit(() -> { doWork(); latch.countDown(); });
}
latch.await();        // blocks until count hits 0
System.out.println("all done");
```

### 6.2 `CyclicBarrier`

N threads wait at a rendezvous, then all proceed. Reusable.

```java
var barrier = new CyclicBarrier(4, () -> System.out.println("phase done"));
// each of 4 threads: ... ; barrier.await(); ...
```

### 6.3 `Semaphore`

Limit concurrent access to a resource.

```java
Semaphore permits = new Semaphore(10);     // max 10 concurrent
permits.acquire();
try { useResource(); }
finally { permits.release(); }
```

### 6.4 `Phaser`

Generalization of `CountDownLatch` and `CyclicBarrier`. Supports dynamic registration. Used in fork/join-style algorithms.

---

## 7. `ForkJoinPool` and parallel streams

`ForkJoinPool` is a work-stealing pool. Each worker has its own deque; idle workers steal from busy ones. Good for divide-and-conquer.

```java
new ForkJoinPool().submit(() -> task);
ForkJoinPool.commonPool();   // shared default
```

`Stream.parallelStream()` runs on `commonPool` by default. Module 16 covers parallel-stream caveats. Be aware: a CPU-hogging parallel stream affects every other parallel stream user in the JVM.

---

## 8. Idioms

### 8.1 Producer-consumer with bounded queue

```java
BlockingQueue<Job> queue = new LinkedBlockingQueue<>(1000);
ExecutorService consumers = Executors.newFixedThreadPool(8);
for (int i = 0; i < 8; i++) {
    consumers.submit(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Job j = queue.take();
                process(j);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    });
}
producer.run(() -> queue.put(makeJob()));
```

### 8.2 Atomic check-and-update on a map

```java
ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
V v = cache.computeIfAbsent(key, k -> expensiveLoad(k));
```

The function runs at most once per key (per JVM) under contention.

### 8.3 Background scheduled tasks

```java
ScheduledExecutorService sch = Executors.newSingleThreadScheduledExecutor();
sch.scheduleAtFixedRate(metrics::flush, 0, 30, TimeUnit.SECONDS);
```

Always shut down on application stop.

### 8.4 Bounded parallelism for I/O

```java
Semaphore limit = new Semaphore(50);
List<CompletableFuture<R>> tasks = inputs.stream()
    .map(input -> CompletableFuture.supplyAsync(() -> {
        limit.acquireUninterruptibly();
        try { return ioCall(input); }
        finally { limit.release(); }
    }, pool))
    .toList();
```

(Module 20 makes this cleaner with virtual threads.)

---

## 9. The full hierarchy at a glance

```
Locking primitives:
  synchronized            (intrinsic monitor)
  ReentrantLock
  ReentrantReadWriteLock
  StampedLock

Atomics:
  AtomicInteger / AtomicLong / AtomicBoolean / AtomicReference
  AtomicIntegerArray / AtomicLongArray / AtomicReferenceArray
  LongAdder / DoubleAdder

Thread pools / executors:
  ExecutorService (interface)
  ThreadPoolExecutor
  ScheduledThreadPoolExecutor
  ForkJoinPool

Concurrent collections:
  ConcurrentHashMap
  ConcurrentLinkedQueue / Deque
  CopyOnWriteArrayList / Set
  BlockingQueue: LinkedBlockingQueue, ArrayBlockingQueue, SynchronousQueue, PriorityBlockingQueue
  ConcurrentSkipListMap / Set

Synchronizers:
  CountDownLatch
  CyclicBarrier
  Semaphore
  Phaser
  Exchanger
```

---

## 10. Try this

1. Replace this:
   ```java
   private final Object lock = new Object();
   private int n;
   public void inc() { synchronized (lock) { n++; } }
   public int get() { synchronized (lock) { return n; } }
   ```
   with `AtomicInteger`. Then with `LongAdder`. Benchmark all three under 8-thread contention.

2. Build a bounded producer-consumer with `LinkedBlockingQueue<Integer>(10)`. One producer, three consumers; each consumer increments a shared `LongAdder`. After 1M items, print the count.

3. Implement a memoized `Function<K, V>` with `ConcurrentHashMap.computeIfAbsent`. Why is `computeIfAbsent` correct and `if (!m.containsKey(k)) m.put(k, compute(k));` not?

4. Use `CountDownLatch` to coordinate: spawn 5 worker threads that each take ~random time; main thread waits for all, then prints "done". Now do the same with `ExecutorService.invokeAll`. Which is more idiomatic?

---

**Next:** [Module 20 — `Future`, `CompletableFuture`, virtual threads](./20-completablefuture.md)
