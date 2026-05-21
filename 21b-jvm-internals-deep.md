# Module 21b — JVM Internals Deep Dive

> Goal: go past "G1 is the default GC" and "the JIT inlines stuff". Understand G1's phases, ZGC's load barriers, JIT tiers and deoptimization triggers, escape analysis with concrete examples, and how to read JFR output. Production-level depth.

This supplements [Module 21](./21-jvm-internals.md). Read that first.

---

## 1. G1 — phases and behavior

G1 ("Garbage-First") is region-based. The heap is divided into ~2,048 fixed-size regions (~1–32 MB each, depending on heap size). Each region is dynamically Eden, Survivor, Old, or Humongous (large objects > half a region).

### 1.1 The G1 cycle

```
Young GC (frequent, short)
   ├── Eden + Survivor regions evacuated
   ├── Surviving objects copied to a Survivor region
   ├── Long-survivors promoted to Old
   └── Stop-the-world

[when Old occupancy exceeds InitiatingHeapOccupancyPercent (IHOP, default 45%):]
Concurrent Marking Cycle:
   ├── Initial Mark        ← STW; piggybacks on a Young GC
   ├── Concurrent Mark     ← runs alongside app threads
   ├── Remark              ← STW; finishes marking
   └── Cleanup             ← STW; reclaims completely-empty regions

Mixed GC (one or more)
   ├── Young + a subset of Old regions chosen for highest reclaim
   ├── Stop-the-world
   └── Repeated until Old occupancy is back below threshold

Full GC (last resort, slow, single-threaded by default)
   └── Triggered when concurrent cycle can't keep up
```

**Mixed GCs** are the key to G1's pause budget — instead of collecting all of Old at once (a Full GC), G1 spreads Old reclamation across multiple shorter pauses targeting `MaxGCPauseMillis`.

### 1.2 What you watch in production

- **Frequency of Young GCs** — too frequent (>1/sec) suggests Eden is too small or allocation is too high. Either grow the heap or chase allocation hotspots.
- **Mixed GC count** — should run periodically. If Old fills before mixed GCs catch up, you'll see Full GCs.
- **Full GCs** — in a healthy G1 service, full GCs are rare or absent. Frequent Full GCs = heap too small or memory leak.
- **Time-to-safepoint** — log with `-Xlog:safepoint*`. Long delays getting to safepoint usually mean tight loops with no back-edge polls.

### 1.3 Tunables that actually help

- **`-Xms == -Xmx`** — fixed heap. Avoids resize pauses. Always set in production.
- **`-XX:MaxGCPauseMillis=200`** — target. G1 sizes regions and pacing toward this.
- **`-XX:G1HeapRegionSize`** — usually let G1 pick. Override for very large heaps.
- **`-XX:InitiatingHeapOccupancyPercent`** — when to start concurrent marking. Lower = more frequent cycles, less pressure on Old. Default 45%.
- **`-XX:G1MixedGCCountTarget`** — number of mixed GCs after a marking cycle. Default 8.

Don't tune blindly. Measure first.

---

## 2. ZGC — concurrent and almost pause-free

ZGC's promise: sub-millisecond pauses, regardless of heap size. How:

### 2.1 Colored pointers + load barriers

Each pointer in ZGC carries metadata bits ("colors") encoding mark/relocation state. On every load of a reference:
- A *load barrier* checks the color bits.
- If the pointer is "good", proceed.
- If not, fix it up (remap to new location after relocation, mark, etc.) — concurrently with app code.

This trades a small per-load-of-pointer cost for the ability to evacuate concurrently. Allocations and collections happen mostly in parallel with application threads. STW pauses are tiny — only the global state transitions (mark start, relocate start) need them.

### 2.2 When to use ZGC

- Heaps >10–20 GB.
- Latency-sensitive (low p99 tail).
- Modern Java (21+).

Trade-offs vs G1:
- Slightly lower throughput (~5–15% on average; varies).
- Higher CPU use (load barriers, concurrent threads).
- More memory overhead (metadata).

For most services with <8GB heaps and 200ms p99 budgets, G1 is fine. ZGC shines with big heaps or microsecond-level latency requirements.

### 2.3 Generational ZGC (Java 21)

ZGC was originally not generational. Java 21 introduced **Generational ZGC** (`-XX:+UseZGC -XX:+ZGenerational`) — adds a young generation, exploiting the "most objects die young" hypothesis without giving up sub-ms pauses. Better throughput than non-generational ZGC; the recommended ZGC mode going forward.

---

## 3. Safepoints — what makes pauses long

A **safepoint** is a point where the JVM can stop *all* threads to do global work. Inserted at:
- Method entry/exit.
- Loop back-edges (long-counted loops have polls every iteration; short ones may have polls less often after JIT).
- Memory allocations.
- Calls.

When the JVM signals "STW now":
1. Each thread runs until it hits its next safepoint check.
2. Once *every* thread parks at a safepoint, the global work runs.
3. The slowest thread to reach a safepoint determines the pause length.

### 3.1 "GC pause was 50ms but the GC took 5ms"

Where did the other 45ms go? **Time-to-safepoint.** A thread in a tight, JIT-optimized loop without polls can take a long time to safepoint.

To measure:
```
-Xlog:safepoint*:file=safepoint.log:time
```

Output includes:
```
Total time for which application threads were stopped: 0.05s
Stopping threads took: 0.045s
```

If "stopping threads" is significant, the actual STW is dominated by waiting for threads to reach safepoints, not the GC work itself.

### 3.2 Common causes of long time-to-safepoint

- **Counted loops with no polls** — JIT optimizes some `for (int i = 0; i < n; i++)` to remove polls. Modern JVMs add polls anyway, but it can still happen.
- **JNI / native code** — threads in JNI don't safepoint until they return.
- **Long monitor waits** — usually safepoint properly, but check if you're using `Object.wait` carefully.

The fix is usually JVM-level (newer JDK), not application code.

---

## 4. JIT — tiered compilation and deoptimization

### 4.1 The tiers

HotSpot uses two JIT compilers in five tiers:

| Tier | Compiler | Notes |
|---|---|---|
| 0 | interpreter | initial; profiles invocation |
| 1 | C1 | quick compile, no profiling |
| 2 | C1 + invocation counters | partial profile |
| 3 | C1 + full profile | profiling for C2's benefit |
| 4 | C2 | aggressive optimization |

Methods escalate through tiers as they get hot. A method may run interpreted for ~10K invocations, then C1 for ~10K more, then C2 for the rest of the program.

### 4.2 What C2 does

- **Inlining** — small hot methods inlined into callers.
- **Escape analysis** — proves objects don't escape; may scalar-replace or stack-allocate.
- **Loop unrolling** — replicates loop body to reduce overhead per iteration.
- **Vectorization (SuperWord)** — combines scalar ops into SIMD instructions.
- **Range check elimination** — for array accesses where the JIT proves the index is safe.
- **Lock elision / coarsening** — drops or merges `synchronized` when single-threaded usage is proven.
- **Dead code elimination** — across inlined methods.
- **Branch prediction profiling** — biases generated code toward observed paths.

### 4.3 Deoptimization

C2 makes optimistic assumptions based on profiles. If they're wrong, the JIT **deoptimizes**:
- Throw away the native code.
- Fall back to interpreted execution at the next safepoint.
- Possibly recompile with a different strategy.

Common deopt triggers:
- **Class hierarchy changes.** C2 inlined a virtual call assuming there's only one impl. A new subclass loads → invalidates the assumption.
- **Uncommon trap.** Code path that "rarely" runs (per profile) suddenly runs. C2 had compiled assuming it wouldn't.
- **`null` where C2 thought it'd never see null.**

You see deopts with:
```
-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining
```

Mostly: don't worry about deopts unless profiling shows them dominating CPU. The JIT recompiles within seconds and steady-state recovers.

### 4.4 Inlining in detail

C2's most powerful optimization. Inlining decisions are based on:
- Method size (`-XX:MaxInlineSize`, default 35 bytecodes for small).
- Hot frequency (`-XX:FreqInlineSize`, default 325).
- Recursive depth limit.
- Whether the call is monomorphic (one target), bimorphic (two), or megamorphic.

**Monomorphic** calls inline cleanly. **Bimorphic** can inline both with a type check. **Megamorphic** typically can't inline — virtual dispatch via vtable.

This is why "default to `final` classes / methods" is fast: monomorphic. It's also why interfaces with many implementations are sometimes slower than abstract classes — but profile-guided inlining usually handles this fine.

### 4.5 Escape analysis — concrete example

```java
void process() {
    Point p = new Point(1, 2);     // does this allocate?
    int sum = p.x + p.y;
    System.out.println(sum);
}
```

C2 analyzes `p`:
- Is `p` returned? No.
- Is `p` stored in a field, array, or static? No.
- Is `p` passed to a method whose body might store it? `Point.x` access doesn't.

C2 concludes `p` does not escape. It then **scalar-replaces** `p`:
- `p.x` → local int variable.
- `p.y` → local int variable.
- `new Point(...)` → no allocation.

Result: zero heap allocation. The "object" never existed at runtime.

This is why short-lived `Optional`, lambdas, records, and `Stream` pipelines are usually free — C2 proves they don't escape and removes them. **Don't pre-optimize to avoid these abstractions.**

What breaks escape analysis:
- Storing the object in a field.
- Passing to a virtual method whose impl is unknown.
- Throwing an exception that captures the object.
- Synchronizing on the object (forces it onto the heap).

---

## 5. Class loading and unloading — the practical view

### 5.1 The classloader hierarchy

```
Bootstrap classloader
   └── Platform classloader
          └── System (application) classloader
                 └── Custom classloaders (Spring Boot, app servers, plugins)
```

Each child consults its parent before searching itself (delegation). Classes loaded by different classloaders are **different runtime types** even if their source is identical.

### 5.2 Class unloading

A `Class<?>` becomes garbage only when its **defining classloader** is garbage. The bootstrap and system classloaders never go away — their classes are permanent. Custom classloaders can be GC'd, taking their classes.

### 5.3 The classic memory leak — the redeploy

A long-running web app redeploys a bundle. The old `WebappClassLoader` should die with the old app. But:
- A **`ThreadLocal`** in a worker thread that survives redeploys holds a reference to a class from the old loader.
- A **scheduled task** scheduled by the old app references a class.
- A **JNDI binding**, **MBean**, or **logger** holds a reference.

Result: every redeploy doubles classloaded metadata in metaspace. After a few cycles, `OutOfMemoryError: Metaspace`.

Diagnosing: heap dump → look for the old `WebappClassLoader`. Trace its GC roots. The first root in the chain is your culprit.

### 5.4 Spring Boot's nested-jar classloader

Spring Boot's "fat jar" packs dependencies as nested jars and loads them with a custom `LaunchedURLClassLoader`. This classloader survives normal application lifecycle. When you deploy to Kubernetes (one app per container), this is fine. When you deploy multiple apps in one JVM (uncommon today), be aware.

---

## 6. Reading a heap dump

Take a dump:
```
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

Open in Eclipse MAT or VisualVM. Things to look for:

### 6.1 "Leak Suspects" report (MAT)

MAT analyzes the dump and suggests what's hogging memory. Often points right at your problem.

### 6.2 Dominator tree

For each object, what *retains* it? "Retained heap" = the memory that would be freed if this object went away. Sort by retained heap; the top entries are usually:
- A static `Map` or `List` that's grown unboundedly.
- A cache without eviction.
- `ThreadLocal`s.
- A large `byte[]` or `char[]` (string cache, parser buffer, etc.).

### 6.3 Path to GC roots

Pick a suspicious object. Right-click → "Path to GC Roots". Shows the chain of references keeping it alive. The tail of the chain (the GC root) is usually a static field, a thread, or a `ClassLoader`.

### 6.4 Patterns to look for

- **Many `String`s** that look like log lines or HTTP responses, all referenced from one place. Logging or HTTP-buffer leak.
- **`HashMap$Entry`s** with the same value pattern, referenced from one map. Cache without eviction.
- **A few large `byte[]`** referenced from `DirectByteBuffer`. Native memory leak.

---

## 7. JFR — Java Flight Recorder

The built-in profiler. Low overhead (~1%), production-safe, ships with the JDK.

### 7.1 Starting / stopping

```
jcmd <pid> JFR.start name=mine duration=60s filename=/tmp/recording.jfr
jcmd <pid> JFR.dump name=mine filename=/tmp/dump.jfr     # while running
jcmd <pid> JFR.stop name=mine
```

Or at JVM start: `-XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr`.

Always-on continuous recording (rotates):
```
-XX:StartFlightRecording=settings=continuous,maxage=30m,disk=true,dumponexit=true,filename=/tmp/jfr-on-exit.jfr
```

### 7.2 What JFR captures

- **Method profiling** — sampling, low overhead.
- **Allocation profiling** — what's being allocated, where.
- **GC events** — every GC, with details.
- **Lock contention** — who's blocked on what.
- **JIT compilation** — what's being compiled, deopts.
- **I/O events** — file, socket reads/writes.
- **Custom events** — your own via `jdk.jfr.Event`.

### 7.3 Reading the output

Open in **JDK Mission Control (JMC)**:
- **CPU** — flame graphs, hotspot methods.
- **Allocation** — who's allocating what, where.
- **Garbage Collections** — pause times, reclaimed amounts.
- **Lock Instances** — contended locks.
- **TLABs** — thread-local allocation buffers; useful for understanding allocation churn.

For the first hour with JFR: take a 60s recording during peak load. Look at CPU flame graph and allocation profile. The hotspots and allocation drivers usually surface in minutes.

### 7.4 Custom events

```java
import jdk.jfr.*;

@Label("Order Processing")
@Category("My App")
@StackTrace(false)
public class OrderProcessingEvent extends Event {
    @Label("Order Id") long orderId;
    @Label("Customer Id") long customerId;
}

var e = new OrderProcessingEvent();
e.orderId = order.id();
e.customerId = order.customer();
e.begin();
processOrder(order);
e.commit();
```

Now JFR records the timing and IDs for every call. Correlate slow events with system metrics (GC, locks).

---

## 8. Production troubleshooting playbook

### 8.1 "p99 latency just spiked"

Suspect causes:
1. **GC pause** — check GC logs for full GC.
2. **Lock contention** — JFR "Lock Instances" or `jstack` repeatedly to spot threads piled on a lock.
3. **Downstream slowness** — saturating thread pool while waiting on a slow DB / HTTP call.
4. **Allocation pressure** — sudden Eden saturation; frequent young GCs.

### 8.2 "Memory is climbing"

1. Heap dump (`jcmd ... GC.heap_dump`). Open in MAT. Find the dominators.
2. Check metaspace (`jcmd ... VM.native_memory summary` if NMT is on, or look at `/actuator/metrics/jvm.memory.used?tag=area:nonheap`).
3. Check direct memory (NIO buffers).
4. If heap is fine but RSS keeps growing — native memory leak (JNI, malloc-backed allocator, off-heap caches).

### 8.3 "Threads are stuck"

`jstack <pid>` for a thread dump. Take 3 dumps 5 seconds apart. Threads in the same state → blocked. Look for:
- Blocked on monitor (`waiting to lock <0x...>`) — lock contention. Find the holder (`locked <0x...>`).
- Blocked on socket read — slow downstream.
- All in `getConnection` — DB pool exhausted.

For lock contention specifically, JFR captures call stacks of contended locks far better than `jstack`.

### 8.4 "JVM is using more CPU than expected"

JFR CPU profile usually nails it. Hot methods + call stacks. Often:
- Tight loop somewhere.
- JIT compiling repeatedly (look at "Hot Compilation" events — if methods are being compiled often, deopt is happening).
- Logging volume too high (formatting + I/O dominates).

---

## 9. JVM flags worth knowing

```
# Memory
-Xms4g -Xmx4g                              # fixed heap
-XX:MaxDirectMemorySize=1g                  # direct (NIO) buffers
-XX:MaxMetaspaceSize=512m                   # metaspace cap (default unbounded)

# GC
-XX:+UseG1GC                                 # default in modern Java
-XX:MaxGCPauseMillis=200
-XX:+UseZGC                                  # for big heaps / low latency
-XX:+ZGenerational                           # generational ZGC (Java 21)
-Xlog:gc*:file=gc.log:time,level,tags:filecount=10,filesize=100M

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof
-XX:ErrorFile=/var/log/jvm-crash.log

# JIT
-XX:+PrintCompilation                        # what got compiled
-XX:+PrintInlining                            # what got inlined (verbose)
-XX:ReservedCodeCacheSize=512m                # JIT code cache

# Safepoints
-Xlog:safepoint*:file=sp.log

# JFR
-XX:StartFlightRecording=...

# Strings / classes
-XX:+UseStringDeduplication                   # G1 only; dedup String backing arrays
-XX:+ClassUnloadingWithConcurrentMark         # G1; unload unused classes during marking
```

You'll only set a small subset in production. The rest are diagnostic.

---

## 10. Try this

1. Run your service with `-Xlog:gc*:stdout` and watch a few young GCs. Note the times and ratios.
2. Trigger a Full GC (deliberately fill the heap). Note the much longer pause and the `[Full GC]` log line.
3. Take a heap dump while a service is hot. Open in VisualVM or MAT. Find the largest class by retained size.
4. Run `jcmd <pid> JFR.start duration=30s filename=/tmp/r.jfr` during load. Open in JMC. Find the hottest method and the highest-allocation site.
5. Add a JFR custom event around your slowest service method. Verify it shows up in the recording.

---

**Back to:** [Module 21 — JVM internals](./21-jvm-internals.md)
