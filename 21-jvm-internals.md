# Module 21 — JVM Internals: Memory, GC, Safepoints, JIT, Classloading

> Goal: this is the module that turns "Java user" into "Java developer". You'll come away knowing the JVM's memory layout, the major garbage collectors and when to use which, what a safepoint is, what the JIT actually does (and why your code is 10× faster the second time it runs), and how classloading shapes large applications. When production stalls you should be able to read a GC log and form a hypothesis.

---

## 1. JVM memory layout

A running JVM divides memory into regions. Simplified picture:

```
JVM process memory
├── Heap                   ← Java objects live here. GC manages it.
│   ├── Young Generation   ← short-lived objects
│   │   ├── Eden
│   │   └── Survivor 0 / Survivor 1
│   └── Old (Tenured) Generation   ← long-lived objects
├── Metaspace               ← class metadata (off-heap, native)
├── Code Cache              ← JIT-compiled native code
├── Per-thread stacks       ← method frames, primitives, references
├── Direct buffers          ← NIO ByteBuffer.allocateDirect
└── Native memory           ← JNI, internal JVM structures
```

### 1.1 Heap

Every `new T()` allocates here. The garbage collector reclaims unreachable objects. The heap is sized by JVM flags:
- `-Xms` initial heap size
- `-Xmx` max heap size
- `-Xms2g -Xmx2g` (recommended in production: equal initial and max — avoids resize pauses)

When you exhaust the heap and GC can't free enough, you get `OutOfMemoryError: Java heap space`.

### 1.2 Generations (in generational GCs)

Most GCs are **generational** — they exploit the empirical "weak generational hypothesis": *most objects die young*.

- **Young (Nursery)** — Eden + two Survivor spaces. New allocations land in Eden. When Eden fills, a *minor GC* happens: live objects copy to a Survivor; objects that survive several rounds *promote* to Old.
- **Old (Tenured)** — long-lived objects. *Major (full) GC* collects here, more expensively.

Why this matters: minor GCs are fast (sweep small region, copy out the few survivors). Full GCs are slower. Allocating short-lived objects is cheap; allocating long-lived ones via promotion is normal.

### 1.3 Metaspace

`Class<?>` objects, method bytecode, JIT structures. Pre-Java 8 this lived in the heap as **PermGen**; since Java 8 it's in **Metaspace** — native memory, sized by `-XX:MaxMetaspaceSize`. Default is "unlimited" (use the OS).

Symptom of a metaspace leak: `OutOfMemoryError: Metaspace`. Almost always caused by classloader leaks (web apps that redeploy without unloading).

### 1.4 Code cache

Native code produced by the JIT lives here. Sized by `-XX:ReservedCodeCacheSize`. Symptom of overflow: `CodeCache is full. Compiler has been disabled.` followed by warm code dropping back to interpreted mode.

### 1.5 Stacks

One per live thread. Holds method frames: locals, operand stack, return address, partial bytecode state. Sized by `-Xss512k`. Default ~512 KB to 1 MB. Deep recursion → `StackOverflowError`.

A platform thread *commits* its stack lazily on demand, so 1 MB doesn't actually mean 1 MB of RSS per thread.

### 1.6 Direct memory

`ByteBuffer.allocateDirect(n)` allocates outside the heap. Used by NIO for zero-copy I/O. Sized by `-XX:MaxDirectMemorySize`. Leaks here look like RSS growth without heap growth.

---

## 2. Garbage collectors

Modern OpenJDK ships several GCs. Pick one with a flag:

| GC | Flag | Use case |
|---|---|---|
| **Serial** | `-XX:+UseSerialGC` | tiny heaps, single-core, embedded |
| **Parallel** | `-XX:+UseParallelGC` | batch jobs, throughput priority |
| **G1** (default since Java 9) | `-XX:+UseG1GC` | general-purpose, balanced latency/throughput |
| **ZGC** | `-XX:+UseZGC` | very large heaps (TB), sub-millisecond pauses |
| **Shenandoah** (Red Hat) | `-XX:+UseShenandoahGC` | similar to ZGC; in OpenJDK |

Defaults change by JDK version. Java 21's default is **G1**.

### 2.1 G1 (Garbage-First)

The general-purpose GC for most apps. The heap is divided into ~2,048 fixed-size regions (each ~1–32 MB depending on heap size). Each region is dynamically Eden, Survivor, or Old.

G1 prioritizes regions with the most garbage (hence "garbage-first"). It uses concurrent marking — most of the work happens while application threads run. Stop-the-world pauses target a budget you set (`-XX:MaxGCPauseMillis=200`). G1 makes a best effort to meet it; it's not a hard guarantee.

What you'll see in a G1 log:
- *Minor (Young) GC* — frequent, short. Copies survivors out of Eden.
- *Concurrent marking cycle* — runs in the background.
- *Mixed GC* — collects young + a subset of old regions.
- *Full GC* — last resort, single-threaded by default. If you see frequent full GCs, your heap is too small or you're allocating too aggressively.

### 2.2 ZGC and Shenandoah

Concurrent collectors with sub-millisecond pauses, regardless of heap size. They use **load barriers** and **forwarding pointers** to evacuate objects while the application reads them.

Use when:
- Large heaps (10+ GB).
- Latency-sensitive workloads (low p99 tail).
- Modern Java (21).

Tradeoff: somewhat lower throughput than G1, slightly higher CPU use.

### 2.3 Tuning vs. not tuning

The default G1 with sane heap sizing is fine for 95% of services. Reach for explicit GC tuning only when:
- You see GC pauses exceeding your latency budget.
- GC overhead is a measurable percentage of CPU.
- You've already eliminated obvious allocation hotspots.

Two flags worth knowing:
- `-Xms2g -Xmx2g` — set initial = max so the heap doesn't resize.
- `-Xlog:gc*:file=gc.log:time,level,tags` — modern GC logging (Java 9+; replaced the old `-XX:+PrintGC*` flags).

---

## 3. Safepoints

A **safepoint** is a point in the running program where the JVM can stop *all* application threads to do certain global work — full GC marking, deoptimization, biased-lock revocation, OS thread dumps.

The JIT-compiled code has safepoint *checks* — typically at:
- method entry/exit,
- back edges of loops,
- before allocation/calls.

When the JVM signals "safepoint", every thread runs until it hits its next safepoint check, then parks. Once *every* thread has parked, the JVM does its work and resumes everyone.

Why this matters:
- A "stop-the-world" pause includes the time to *get to* the safepoint. A thread in a tight loop without back edges (or in JNI / native code) may delay the safepoint.
- Long safepoint stalls reported as longer GC pauses than expected — the GC itself was fast, but threads took 30ms to reach a safepoint.

You can log safepoints: `-Xlog:safepoint*:file=sp.log`.

Counted vs uncounted loops: an `int`-counted `for (int i = 0; i < n; i++)` once was treated as "no safepoint inside" — the JIT could JIT a loop with no back-edge poll. Modern JVMs insert safepoint polls in counted loops too (since Java 10ish), but the principle remains: tight loops can starve safepoints. Keep this in mind for low-latency code.

---

## 4. The JIT

We covered the basics in Module 02. To deepen:

### 4.1 Tiered compilation

HotSpot uses **C1** and **C2**:
- **C1** (client) — fast compile, modest optimization. Tier 1 / 2 / 3.
- **C2** (server) — slow compile, aggressive optimization. Tier 4.

Methods start interpreted (Tier 0). Once hot enough they tier up: 0 → 3 (C1 with profiling) → 4 (C2). Cold methods may be evicted from the code cache.

Watch with:
```
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:+PrintCompilation
```

### 4.2 Inlining

C2's most important optimization. Small hot methods are inlined into their callers, removing the call overhead and enabling further optimizations (constant folding, dead-code elimination across the inlined code).

Why this means your code is faster than it looks:
- A `getter` method gets inlined to a direct field access.
- A `synchronized` method gets inlined and may have its lock elided if escape analysis proves it's thread-local.
- A `forEach` lambda call gets inlined into the loop, becoming equivalent to a hand-written for.

### 4.3 Escape analysis

If the JIT proves an object never escapes its allocating method (no reference is stored, returned, passed to unknown code), it may:
- **Scalar-replace** — split fields into individual locals. The "object" never exists in memory.
- **Stack-allocate** (rare in HotSpot) — allocate on the stack.
- **Eliminate locks** (`synchronized` on a thread-local object is a no-op).

Result: lots of small temporary objects (records, `Optional`, lambdas) cost almost nothing if the JIT can prove they don't escape. Don't pre-optimize to avoid them.

### 4.4 Speculative optimization & deoptimization

C2 optimizes based on profile data — "this `invokevirtual` is monomorphic; inline the one target". If a new subclass loads later that breaks the assumption, the JIT **deoptimizes** — discards the native code and falls back to interpreted execution at the next safepoint. The method may recompile with a different strategy.

This is why benchmarks are tricky: the first iteration runs interpreted; the next thousand at full speed. Use **JMH** (Java Microbenchmark Harness) for serious measurement.

### 4.5 GraalVM and AOT

GraalVM provides:
- **Graal JIT** — a high-tier JIT in Java, replacing C2.
- **Native Image** — ahead-of-time compilation to a native binary. Fast startup (milliseconds), low memory, but limited reflection / dynamic class loading. Quarkus / Spring Native rely on this for serverless workloads.

Day-to-day, you're on HotSpot.

---

## 5. Class loading and unloading

We covered loading in Module 02 (Load → Link → Initialize). What about unloading?

A `Class<?>` object can be unloaded only when its **defining classloader** becomes garbage. The bootstrap and system classloaders never go away — their classes are permanent. Custom classloaders (used by application servers, OSGi, Spring Boot's nested-jar loader) can be GC'd, taking their classes with them.

This is what causes the classic "metaspace leak" in long-running web apps: a redeployed app drops references to its old classloader, but a single static reference *somewhere* (say, a `ThreadLocal` in a thread that survives redeploys) keeps the classloader reachable. Every redeploy doubles classloaded metadata. After a few cycles, metaspace OOMs.

Diagnosing this: heap dump → look for the supposedly-unreachable old classloader; trace its GC roots.

---

## 6. Memory leaks in Java (yes, they exist)

GC reclaims *unreachable* objects. Reachable but no-longer-needed objects are leaks.

Common causes:
1. **Static collections that grow.** `static final Map<K, V> CACHE = new HashMap<>()` with `put` but no `remove` is a slow leak.
2. **Listener registration without deregistration.** `eventBus.register(this)` from a short-lived object that doesn't unregister.
3. **`ThreadLocal`s in pooled threads.** The thread outlives the request; the `ThreadLocal` value never gets cleared.
4. **`InheritableThreadLocal` / class metadata** retained across redeploys.
5. **Holding onto large objects via inner-class references** (non-static inner class references its outer instance).

Diagnose with a heap dump (`jcmd <pid> GC.heap_dump out.hprof`), open in Eclipse MAT or VisualVM, look at *retained heap* and *path to GC roots*.

---

## 7. Inspecting a running JVM

All bundled with the JDK:

| Tool | What it does |
|---|---|
| `jps` | list running JVMs (PIDs and main classes) |
| `jstat -gc <pid> 1s` | live GC stats |
| `jstack <pid>` | full thread dump |
| `jcmd <pid> Thread.print` | newer thread dump |
| `jcmd <pid> GC.heap_dump file.hprof` | heap dump |
| `jcmd <pid> GC.run` | request a full GC (debug only) |
| `jcmd <pid> VM.flags` | effective JVM flags |
| `jcmd <pid> JFR.start ...` | Java Flight Recorder — built-in profiler |
| `jconsole`, `VisualVM` | GUI front-ends |

**JFR** is the production-safe profiler. Low overhead, ships with the JDK, opens in **Mission Control**. The first tool to reach for when something is slow.

---

## 8. Reading a GC log (snippet)

Modern format (`-Xlog:gc*`):
```
[2.345s][info][gc] GC(0) Pause Young (G1 Evacuation Pause) 256M->48M(2048M) 12.345ms
```

- `2.345s` — uptime.
- `Pause Young` — minor GC.
- `256M->48M(2048M)` — heap before → after, of total 2 GB. Most of Eden was reclaimed.
- `12.345ms` — pause duration.

Patterns to recognize:
- Frequent young pauses, healthy ratios → normal allocation churn.
- Repeated `Full GC (Allocation Failure)` with little reclaimed → heap too small or memory leak.
- Long mixed GCs → G1 is struggling to keep up; consider tuning region size or moving to ZGC.

---

## 9. Practical takeaways

1. **Allocate freely for short-lived objects.** Generational GC handles that exceptionally well.
2. **Don't pool small objects.** You're fighting the GC; you'll lose readability and rarely win performance.
3. **Avoid `synchronized` for things that aren't truly contended.** Most modern code uses concurrent collections / atomics.
4. **Set `-Xms == -Xmx`.** No surprise heap resizes.
5. **Use JFR for profiling.** It's free and accurate.
6. **Watch metaspace** in long-running apps with classloader churn.
7. **Don't write a custom finalizer / cleaner unless you genuinely need it.** Use try-with-resources.

---

## 10. Two example diagnostic stories

### 10.1 "p99 latency spiked at noon every day"

GC log shows a long Full GC every 24 hours. Heap dump shows a `static List<Event>` that never expires. Fix: a sliding window or a `Caffeine` cache with size limits.

### 10.2 "metaspace OOM after 5 hot deploys"

Production OSGi container redeploys a bundle. Heap dump shows the old `BundleClassLoader` retained by a thread in `ScheduledExecutorService` belonging to an unrelated bundle that scheduled a task referencing the old class. Fix: bundle's `stop()` cancels its scheduled tasks before letting its classloader unload.

---

## 11. Try this

1. Run a tiny program with `-Xlog:gc*` and watch a few minor GCs roll by. Add a loop allocating 1M objects; trigger a young GC.
2. `jcmd <pid> VM.flags` on a running JVM and identify which GC and heap sizes are in effect.
3. Take a heap dump with `jcmd`, open in VisualVM, find the largest class by retained size.
4. Why does the following take ~constant memory in modern JVMs?
   ```java
   for (int i = 0; i < 1_000_000_000; i++) {
       var p = new int[]{i, i+1};
       sum += p[0] + p[1];
   }
   ```
   (Hint: escape analysis + scalar replacement. The "allocations" never reach the heap.)
5. Start a JFR recording for 60s of a small workload, open it in Mission Control, find the hottest method.

---

**Next:** [Module 22 — Modern Java](./22-modern-java.md)
