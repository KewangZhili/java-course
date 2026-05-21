# Module 18 — Concurrency Basics

> Goal: understand threads, what `synchronized` and `volatile` actually do, and the **Java Memory Model** — the rule that decides what one thread can observe of another. After this module the next two (executors, futures) make sense; without it they're magic.

This is the densest module in the course. Concurrency in Java is genuinely hard, not because the syntax is bad but because **you cannot debug data races by inspection** — they manifest only on certain hardware, certain JIT inlinings, certain timings. The only defense is understanding the model.

---

## 1. Threads

A **thread** is an independent flow of execution. The JVM has one main thread (running `main`) at startup, plus internal threads (GC, JIT, finalizer, signal handler). You can start more.

### 1.1 Start a thread

```java
Thread t = new Thread(() -> {
    System.out.println("hi from " + Thread.currentThread().getName());
});
t.start();         // launches; runs concurrently
t.join();          // wait for it to finish
```

Three ways to define what a thread runs:

```java
// (1) lambda → Runnable
new Thread(() -> work()).start();

// (2) Runnable subclass / instance
class Worker implements Runnable {
    @Override public void run() { ... }
}
new Thread(new Worker()).start();

// (3) Thread subclass (rare)
class MyThread extends Thread {
    @Override public void run() { ... }
}
new MyThread().start();
```

Always prefer `Runnable` over subclassing `Thread`. Composition over inheritance.

**Don't call `t.run()`** when you mean `t.start()`. `run()` just executes the method on the *current* thread.

### 1.2 Daemon threads

A thread can be marked daemon: `t.setDaemon(true);` before `start()`. The JVM exits when only daemon threads remain. Use for background helpers (heartbeats, log flushers) that shouldn't keep the JVM alive.

### 1.3 Threads are expensive

A platform thread is mapped 1:1 to an OS thread. Each carries ~1 MB of stack by default. Spawning a thread per request scales poorly past a few thousand.

Solutions:
- **Thread pools** (Module 19) — reuse a fixed set of platform threads.
- **Virtual threads** (Java 21+, Module 20) — lightweight, millions per JVM, mapped many-to-one onto platform carrier threads. Modern recommended path for I/O-bound work.

### 1.4 Interrupting

`t.interrupt()` sets a flag on `t`. Cooperatively, the thread should check `Thread.interrupted()` and exit cleanly. Blocking calls (`Thread.sleep`, `wait`, `Object.wait`, `Future.get`) throw `InterruptedException` when interrupted.

```java
while (!Thread.currentThread().isInterrupted()) {
    try { doStep(); }
    catch (InterruptedException e) {
        Thread.currentThread().interrupt();   // restore the flag
        break;
    }
}
```

**The interrupt rule: when you catch `InterruptedException`, either rethrow it or restore the interrupt flag.** Swallowing it without setting the flag breaks shutdown.

---

## 2. The race condition

Two threads incrementing the same `int`:

```java
class Counter {
    int n = 0;
    void inc() { n = n + 1; }    // NOT atomic
}

var c = new Counter();
Runnable r = () -> { for (int i = 0; i < 1_000_000; i++) c.inc(); };
var t1 = new Thread(r); var t2 = new Thread(r);
t1.start(); t2.start();
t1.join();  t2.join();
System.out.println(c.n);   // expected 2_000_000 — almost never gets it
```

Why the bug:
- `n = n + 1` is three operations: read `n`, add 1, write `n`.
- The two threads can interleave their reads and writes — both read 5, both write 6, an increment is lost.

Two distinct problems are entangled here:
- **Atomicity** — `n = n + 1` is not one indivisible operation.
- **Visibility** — even if it were atomic, one thread might cache `n` in a CPU register and not see another's update.

Java's solution: `synchronized`, `volatile`, and the JMM.

---

## 3. `synchronized` — mutual exclusion

`synchronized` acquires a lock (monitor) for the duration of a block.

```java
class Counter {
    int n = 0;
    synchronized void inc() { n = n + 1; }     // method-level
    synchronized int get()   { return n; }
}
```

Equivalent block form:
```java
void inc() {
    synchronized (this) {
        n = n + 1;
    }
}
```

You can synchronize on any object — the **monitor**:
```java
private final Object lock = new Object();
void inc() {
    synchronized (lock) { n++; }
}
```

Best practice: lock on a *private final object* you own, not on `this` (so external code can't lock you).

Properties:
- The lock is **reentrant**: a thread that already holds the lock can re-acquire it without deadlock.
- The lock is released automatically on block exit, including via exceptions.
- A `static synchronized` method locks on `ClassName.class`, the class object.
- Synchronization establishes **happens-before** — see §6 below.

### 3.1 What `synchronized` does NOT do

- It doesn't prevent concurrent readers if all reads are also synchronized — only one thread inside the block at a time, even readers.
- It doesn't apply across unrelated locks. Two `synchronized(this)` blocks on *different* instances don't exclude each other.
- It doesn't prevent reordering of operations *outside* the block — and the compiler/CPU can reorder operations inside in ways that are still legal.

### 3.2 Deadlock

Two threads, each holding one lock and trying to acquire the other:
```java
synchronized (a) {
    synchronized (b) { ... }
}
// other thread:
synchronized (b) {
    synchronized (a) { ... }
}
```

Both block forever. Avoid by always acquiring locks in the same global order.

### 3.3 `wait`/`notify` (legacy)

Every object has a built-in monitor with `wait()` and `notify()` — primitives for "wait until something is ready". You'll see them in legacy code:

```java
synchronized (queue) {
    while (queue.isEmpty()) queue.wait();
    item = queue.removeFirst();
}
```

Modern code uses `BlockingQueue`, `Condition`, or `CompletableFuture` instead. The `wait`/`notify` machinery is hard to use correctly (must be inside `synchronized`, must use `while` not `if` to recheck the condition, easy to lose notifications). Prefer the higher-level alternatives.

---

## 4. `volatile` — visibility, atomicity (mostly)

`volatile` on a field tells the JVM:
- Every read fetches from main memory (no caching in registers).
- Every write flushes to main memory (no buffering).
- Reads and writes are atomic, even for `long` and `double` (which are otherwise allowed to be non-atomic on 32-bit JVMs).
- Establishes ordering: a write to a volatile variable *happens-before* any subsequent read of the same variable from any thread.

```java
class Service {
    private volatile boolean running = true;

    void shutdown() { running = false; }

    void loop() {
        while (running) doWork();        // sees the flip
    }
}
```

Without `volatile`, the JIT could optimize `while (running)` into `if (running) while (true)` because, from a single-threaded perspective, `running` doesn't change inside the loop.

### 4.1 What `volatile` is NOT

- Not a lock. `volatile int n; n++;` is **still a race** — the read-modify-write isn't atomic. Use `AtomicInteger` (Module 19) or `synchronized`.
- Not enough for compound state. If you have `volatile A a; volatile B b;` and need them updated together, `volatile` doesn't help.

Use `volatile` for **flags**, **once-published references** (after construction), and **double-checked locking** (the canonical example).

### 4.2 Double-checked locking with `volatile`

```java
class Lazy<T> {
    private volatile T value;
    public T get(Supplier<T> init) {
        T v = value;
        if (v == null) {
            synchronized (this) {
                v = value;
                if (v == null) {
                    v = init.get();
                    value = v;
                }
            }
        }
        return v;
    }
}
```

Without `volatile` on `value`, this is a textbook bug — the assignment `value = v` may be reordered relative to `init.get()`'s internal writes. Other threads could see a non-null `value` whose object is partially constructed.

In practice, prefer `Holder.value` static-init idiom or `AtomicReference` — they're harder to misuse.

---

## 5. The Java Memory Model (JMM)

The JMM is the formal rule that says, "given an interleaving of memory operations across threads, what reads are allowed to see what writes?". It is concise to state and ruthless in its consequences.

The model is built on the **happens-before** relation. If one action *happens-before* another, the second sees the effects of the first. Without happens-before, all bets are off — the JVM may reorder operations freely.

Established happens-before edges:
1. **Program order** within a single thread: `a; b;` — `a` happens-before `b`.
2. **Monitor lock**: a `synchronized` block's exit happens-before any subsequent acquisition of the *same* lock.
3. **Volatile**: a write to a volatile variable happens-before every subsequent read of *that* variable.
4. **Thread start/join**: `Thread.start()` happens-before the started thread's first action; the last action of a thread happens-before any thread's `join()` returning successfully.
5. **`final` field publication**: setting a `final` field in a constructor happens-before any thread sees the constructed object — assuming the object reference doesn't escape the constructor.

`Thread.interrupt()`, `CompletableFuture.complete()`, executor `submit`/`get`, `CountDownLatch.countDown()` etc. each establish happens-before too — Module 19.

Without one of these edges between two threads' operations, there is no guarantee one sees the other's writes. *Even if it works on your laptop today.*

### 5.1 An everyday example

```java
class Holder {
    int x;
    boolean ready;
}

// Thread A
holder.x = 42;
holder.ready = true;

// Thread B
if (holder.ready) System.out.println(holder.x);   // could print 0!
```

Without `volatile` or `synchronized`, the compiler / CPU may reorder Thread A's writes — `ready = true` could become visible *before* `x = 42`. Thread B sees `ready=true` and reads `x=0`.

Fix:
```java
volatile boolean ready;        // write to ready happens-before subsequent reads
holder.x = 42;
holder.ready = true;
```

Now `x = 42` (in program order, before the volatile write) is guaranteed visible to any thread that reads `ready` and sees `true`.

### 5.2 The intuition

- Use `synchronized` (or `Lock`) when you need atomicity over multiple operations.
- Use `volatile` for single-variable flags / once-published references.
- Use immutable objects (`final` fields, no setters) where you can — they're naturally thread-safe.
- Use the `java.util.concurrent` classes (Module 19) — they handle the JMM correctly so you don't have to.

---

## 6. Immutability is the easy mode

An immutable object is automatically thread-safe. No locks needed. Sharing across threads is free.

A class is "effectively immutable" if:
- All fields are `final`.
- The class is `final` (so subclasses can't add mutability).
- Constructor doesn't leak `this` to other threads before completing.
- No mutable state is reachable through any field (defensively copy mutable inputs).

Records are immutable by construction. `String`, `LocalDate`, `Money`, `BigInteger`, `URI` are all immutable. Lean on this — pass immutables across threads without ceremony.

---

## 7. Common patterns

### 7.1 Confinement — the safest concurrency

If only one thread ever touches a piece of data, you don't need any synchronization. Per-thread state is the cheapest form of safety. `ThreadLocal<T>` (rare in modern code; useful for things like per-request context).

### 7.2 Stack confinement — local variables

A local variable is by definition confined to one thread. Pass-by-value at method boundaries means you can hand off ownership cleanly.

### 7.3 Defensive copies at boundaries

If your class accepts a mutable list:
```java
public class Cart {
    private final List<Item> items;
    public Cart(List<Item> items) {
        this.items = List.copyOf(items);   // defensive copy
    }
}
```

Now external mutation of the caller's list doesn't affect `Cart`.

---

## 8. Things you should not do

- **Don't synchronize on a `String` literal or a boxed `Integer`.** They're potentially shared with unrelated code via the constant pool / wrapper cache. You'll get unintended cross-locking.
- **Don't synchronize on `this` for public methods.** Callers can grab your lock and deadlock you. Use a private final lock object.
- **Don't depend on `Thread.sleep` for synchronization.** It's a temptation; it's a bug.
- **Don't catch `InterruptedException` and ignore it.** Restore the flag.
- **Don't read/write a `long` or `double` from multiple threads without `volatile` or other synchronization.** They can tear on 32-bit JVMs.

---

## 9. A worked example — a thread-safe counter, three ways

```java
// (1) synchronized
class C1 {
    private int n;
    public synchronized void inc()     { n++; }
    public synchronized int  get()     { return n; }
}

// (2) volatile + atomic op? — no, this is BROKEN:
class C2 {
    private volatile int n;
    public void inc() { n++; }     // race — read-modify-write
}

// (3) AtomicInteger (Module 19)
import java.util.concurrent.atomic.AtomicInteger;
class C3 {
    private final AtomicInteger n = new AtomicInteger();
    public void inc()     { n.incrementAndGet(); }
    public int get()      { return n.get(); }
}
```

`C1` is correct. `C2` is wrong — `volatile` doesn't make `++` atomic. `C3` is the modern preferred form: lock-free, fast, correct.

---

## 10. Try this

1. Reproduce the lost-update race: two threads each running `for (int i = 0; i < 1_000_000; i++) counter.n++;`. Print the final count. Run it 5 times — note variance.
2. Fix it three ways: `synchronized` method, `synchronized` block on a private lock, `AtomicInteger`. Time each.
3. Why does this break?
   ```java
   class Lazy {
       private Object value;
       public Object get() {
           if (value == null) value = compute();
           return value;
       }
   }
   ```
   (Race + visibility.) Fix it with double-checked locking + `volatile`. Then replace with the simpler `Holder` static-init idiom.
4. Write a producer-consumer with a single shared `int` and `wait/notify`. Now rewrite using `BlockingQueue<Integer>` (peek ahead at Module 19). Notice the difference in how much can go wrong.

---

**Next:** [Module 19 — Executors, locks, atomics, concurrent collections](./19-executors-locks-atomics.md)
