# Module 20 — `Future`, `CompletableFuture`, Virtual Threads

> Goal: write asynchronous code that composes — chains of "do this, then that" without blocking, error handling that survives several stages, timeouts, and proper thread-pool management. Then meet **virtual threads**, Java 21's answer to `async/await` from other languages: write blocking-style code, run a million in parallel.

---

## 1. `Future<T>` — the basic promise

A `Future<T>` is "a value that will be available later". It came in Java 5 with `ExecutorService`.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> compute());
int result = f.get();      // blocks
```

Limitations:
- `get()` blocks the calling thread.
- No way to chain "when this finishes, run that".
- No way to combine multiple futures elegantly.
- Awkward error handling.

`CompletableFuture<T>` (Java 8+) replaces it for almost every use case.

---

## 2. `CompletableFuture<T>` — composition

A `CompletableFuture<T>` is a `Future<T>` that you can *also* complete manually, and on which you can chain continuations.

### 2.1 Starting one

```java
CompletableFuture<Integer> f = CompletableFuture.supplyAsync(() -> compute());

// already complete
CompletableFuture<Integer> done = CompletableFuture.completedFuture(42);

// uncompleted, manually complete it later
var promise = new CompletableFuture<Integer>();
pool.submit(() -> {
    try { promise.complete(compute()); }
    catch (Throwable t) { promise.completeExceptionally(t); }
});
```

`supplyAsync(supplier)` runs on the **common ForkJoinPool** by default. Pass an explicit executor:
```java
CompletableFuture.supplyAsync(() -> compute(), myPool);
```

### 2.2 Chaining

| Method | Purpose | Returns |
|---|---|---|
| `thenApply(fn)` | T → R, sync | `CompletableFuture<R>` |
| `thenApplyAsync(fn)` | T → R, async | `CompletableFuture<R>` |
| `thenAccept(consumer)` | T → void | `CompletableFuture<Void>` |
| `thenRun(runnable)` | independent of T, runs after | `CompletableFuture<Void>` |
| `thenCompose(fn)` | T → CompletableFuture<R> (flatMap) | `CompletableFuture<R>` |
| `thenCombine(other, biFn)` | (T, U) → R after both | `CompletableFuture<R>` |
| `applyToEither(other, fn)` | first to finish | `CompletableFuture<R>` |

```java
CompletableFuture<String> userFuture = lookup(userId);

CompletableFuture<String> greeting = userFuture
    .thenApply(user -> "Hello, " + user)              // sync transform
    .thenCompose(g -> log(g))                          // chain another async call
    .thenApply(String::toUpperCase);
```

`thenApply` vs `thenCompose`: same as `map` vs `flatMap` on `Optional`/`Stream`. If your function returns a `CompletableFuture`, use `thenCompose` or you'll end up with `CompletableFuture<CompletableFuture<R>>`.

### 2.3 Combining

```java
var a = lookup(userId);
var b = loadOrders(userId);

var combined = a.thenCombine(b, (user, orders) ->
        new Profile(user, orders));

// All of N
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
// allOf returns Void; to collect results:
CompletableFuture<List<R>> joined = all.thenApply(v ->
    Stream.of(f1, f2, f3).map(CompletableFuture::join).toList());

// Any of N — first to finish
CompletableFuture<Object> any = CompletableFuture.anyOf(f1, f2, f3);
```

### 2.4 Error handling

```java
f.exceptionally(ex -> defaultValue)                   // catch-and-recover
 .handle((value, ex) -> ex == null ? value : fallback) // both branches
 .whenComplete((value, ex) -> log(value, ex));         // observe; doesn't change value
```

Errors propagate down the chain — once a stage fails, downstream `thenApply` etc. skip and pass the exception along until something handles it.

```java
CompletableFuture<Integer> f = CompletableFuture
    .supplyAsync(() -> 1 / 0)
    .thenApply(n -> n + 1)            // skipped — exception flowing through
    .exceptionally(e -> -1);          // handles, returns -1
```

The exception you receive is wrapped in `CompletionException`. Unwrap with `getCause()`.

### 2.5 Blocking

```java
f.get();                  // throws checked InterruptedException, ExecutionException
f.join();                 // unchecked; same effect
f.getNow(defaultIfMissing);  // non-blocking — returns default if not done
f.complete(value);        // forcibly complete (only if not already done)
```

Use `join()` in lambdas inside `thenApply` etc. to avoid the checked `InterruptedException`.

### 2.6 Timeouts (Java 9+)

```java
f.orTimeout(5, TimeUnit.SECONDS);                 // throws TimeoutException
f.completeOnTimeout(defaultValue, 5, SECONDS);    // returns default
```

Before Java 9, you had to schedule a timeout yourself.

---

## 3. Choosing executors thoughtfully

The "Async" suffix is important. The non-Async variants run on whatever thread completes the previous stage:

```java
fetchUser(id)                // runs on pool A
    .thenApply(this::format) // runs on the thread that completed fetchUser
    .thenAcceptAsync(this::log, pool);   // explicitly on `pool`
```

If you don't pass an executor to `*Async`, it goes to the **common ForkJoinPool**, which is shared by parallel streams and other CFs. **For blocking I/O, always pass your own pool** — otherwise you can starve the common pool and freeze unrelated parts of the JVM.

```java
ExecutorService myIo = Executors.newFixedThreadPool(50);
CompletableFuture.supplyAsync(this::blockingHttpCall, myIo);
```

Or — better — use virtual threads (§5).

---

## 4. A realistic example

Fan-out, fan-in: load three things in parallel, combine.

```java
import java.util.concurrent.*;
import java.util.concurrent.CompletableFuture;

public class Profile {
    private final ExecutorService io = Executors.newFixedThreadPool(50);

    public CompletableFuture<UserProfile> load(long userId) {
        var u = CompletableFuture.supplyAsync(() -> userRepo.find(userId), io);
        var o = CompletableFuture.supplyAsync(() -> orderRepo.findByUser(userId), io);
        var p = CompletableFuture.supplyAsync(() -> prefRepo.findByUser(userId), io);

        return u.thenCombine(o, (user, orders) -> new Object[]{user, orders})
                .thenCombine(p, (uo, prefs) ->
                        new UserProfile((User)((Object[])uo)[0],
                                        (List<Order>)((Object[])uo)[1],
                                        (Prefs) prefs))
                .orTimeout(2, TimeUnit.SECONDS)
                .exceptionally(ex -> UserProfile.empty(userId));
    }
}
```

The triple-combine is awkward — for >2 futures, the `allOf` + `join` pattern is cleaner:

```java
return CompletableFuture.allOf(u, o, p).thenApply(v ->
    new UserProfile(u.join(), o.join(), p.join()));
```

---

## 5. Virtual threads (Java 21+)

Project Loom delivered virtual threads in Java 21. They are the most important Java feature in a decade — they make async/composition mostly *unnecessary* for I/O-bound work.

### 5.1 What they are

A **virtual thread** is a `Thread` that the JVM schedules onto a small pool of *carrier* OS threads. They:
- Are cheap — millions per JVM (vs. a few thousand platform threads).
- Block normally — `Thread.sleep`, `socket.read`, `synchronized`, `BlockingQueue.take` — but blocking only parks the virtual thread, not the carrier.
- Use the same `Thread` API. Existing libraries work.

```java
Thread.startVirtualThread(() -> handle(socket));
```

Or, with an executor:
```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Socket s : connections) {
        pool.submit(() -> handle(s));
    }
}
```

The `try-with-resources` works because `ExecutorService` extends `AutoCloseable` since Java 19 — `close()` calls `shutdown` and waits.

### 5.2 What they replace

Pre-Java 21 idiomatic async server:
```java
serverSocket.accept().thenComposeAsync(socket ->
    readRequest(socket).thenComposeAsync(this::process)
                       .thenComposeAsync(this::write))
```

Java 21:
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    while (true) {
        Socket s = serverSocket.accept();
        executor.submit(() -> {
            Request r = readRequest(s);   // blocking, but cheap
            Response x = process(r);
            write(s, x);
        });
    }
}
```

Same throughput as the futures version, hugely more readable. **For I/O-bound code, virtual threads are now the default recommendation.**

### 5.3 What they don't help with

- CPU-bound work — a virtual thread mid-computation still ties up its carrier. Use a small pool of platform threads (or `parallelStream`).
- `synchronized` blocks **pin** the virtual thread to its carrier — a long `synchronized` body across blocking I/O can starve carriers. Prefer `ReentrantLock` for that pattern.
- Native code (JNI) similarly pins.

### 5.4 Structured concurrency (preview / incubator)

Going forward, `StructuredTaskScope` lets you treat a group of concurrent subtasks as a unit:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var u = scope.fork(() -> userRepo.find(id));
    var o = scope.fork(() -> orderRepo.findByUser(id));
    scope.join();          // wait for all
    scope.throwIfFailed(); // propagate first failure
    return new Profile(u.get(), o.get());
}
```

Not yet final API as of Java 21, but the direction Java is going. Errors and cancellation propagate cleanly.

---

## 6. Idioms

### 6.1 Map a list of inputs to a list of futures, then to a future of list

```java
List<CompletableFuture<Result>> futures = inputs.stream()
    .map(in -> CompletableFuture.supplyAsync(() -> work(in), pool))
    .toList();

CompletableFuture<List<Result>> all = CompletableFuture
    .allOf(futures.toArray(CompletableFuture[]::new))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList());
```

### 6.2 Retry with backoff

```java
public <T> CompletableFuture<T> retry(Supplier<CompletableFuture<T>> task, int attempts) {
    var f = task.get();
    if (attempts <= 1) return f;
    return f.thenApply(CompletableFuture::completedFuture)
            .exceptionally(ex -> retry(task, attempts - 1))
            .thenCompose(Function.identity());
}
```

### 6.3 Cache asynchronous results

```java
ConcurrentHashMap<K, CompletableFuture<V>> cache = new ConcurrentHashMap<>();

public CompletableFuture<V> get(K key) {
    return cache.computeIfAbsent(key, k -> CompletableFuture.supplyAsync(() -> load(k), pool));
}
```

`computeIfAbsent` ensures only one in-flight load per key. If the load fails, you may want to remove from cache: `whenComplete((v, ex) -> { if (ex != null) cache.remove(key); })`.

---

## 7. Mistakes to avoid

- **Using `get()` inside `thenApply`** to wait for another future. Use `thenCompose` instead.
- **Forgetting to handle exceptions.** A CF whose chain ends without `exceptionally` or `handle` silently swallows errors when its result is never observed.
- **Long-running work on the common pool.** Pass your own executor.
- **Mutable shared state across stages without synchronization.** Stages run on different threads — Module 18 rules apply.
- **`Thread.sleep` in a `synchronized` block on a virtual thread.** Pinning. Replace with `ReentrantLock`.

---

## 8. Try this

1. Convert this:
   ```java
   ExecutorService io = Executors.newFixedThreadPool(50);
   List<Future<String>> fs = urls.stream().map(u -> io.submit(() -> fetch(u))).toList();
   List<String> results = fs.stream().map(f -> {
       try { return f.get(); } catch (Exception e) { throw new RuntimeException(e); }
   }).toList();
   ```
   to a `CompletableFuture` pipeline. Then to a virtual-thread executor — note how much cleaner the second version is.

2. Implement a "fastest-of-N" call: send the same request to three replicas; return the first response and cancel the others. Use `anyOf`.

3. Why does this print "1 done" but not "2 done"?
   ```java
   CompletableFuture.supplyAsync(() -> 1)
                    .thenApply(n -> { throw new RuntimeException("boom"); })
                    .thenAccept(n -> System.out.println(n + " done"));
   ```
   Add an `.exceptionally(...)` to print "2 done" with `-1` instead.

4. Use `Executors.newVirtualThreadPerTaskExecutor` to run 100 000 simulated I/O calls (each `Thread.sleep(100)`). Measure wall-clock time. Compare to a fixed pool of 100 platform threads.

---

**Next:** [Module 21 — JVM internals: memory, GC, safepoints, JIT](./21-jvm-internals.md)
