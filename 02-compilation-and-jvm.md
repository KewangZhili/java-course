# Module 02 — How Java Compiles and Runs

> Goal: walk the entire pipeline from `.java` source to a running program. Understand what `javac` produces, what the classloader does, what bytecode looks like, why there is *both* an interpreter and a JIT, and where memory comes from. This is the mental model every other module relies on.

You can write Java without knowing this — but the moment something goes wrong (a `NoClassDefFoundError`, a slow startup, a memory leak, a "why is the same code 10× faster the second time it runs?") you'll be lost without it.

---

## 1. The full pipeline at a glance

```
   Hello.java
      │
      │  javac  (compile-time)
      ▼
   Hello.class       ──┐
   List.class        ──┤   .jar files,
   String.class      ──┤   directories,
   …                  │   modules
                      │
                      │  java   (start the JVM)
                      ▼
   ┌──────────────────────────────────────────────┐
   │             JVM (running process)            │
   │                                              │
   │   ┌─────────────┐    ┌──────────────────┐    │
   │   │ Classloader │ →  │ Bytecode verifier│    │
   │   └─────────────┘    └──────────────────┘    │
   │                              │               │
   │                              ▼               │
   │   ┌──────────┐      ┌────────────────┐       │
   │   │Interpreter│────▶│ Profiler / JIT │       │
   │   └──────────┘      └────────────────┘       │
   │         │                    │               │
   │         ▼                    ▼               │
   │   ┌────────────────────────────────┐         │
   │   │ Native code, executed on CPU   │         │
   │   └────────────────────────────────┘         │
   │                                              │
   │   ┌──────────────────────────────────────┐   │
   │   │ Memory: heap (objects) + stacks      │   │
   │   │         + metaspace (class metadata) │   │
   │   └──────────────────────────────────────┘   │
   │                                              │
   │   ┌──────────────────────────────────────┐   │
   │   │ Garbage Collector                    │   │
   │   └──────────────────────────────────────┘   │
   └──────────────────────────────────────────────┘
```

We walk each box.

---

## 2. `javac`: source to bytecode

`javac Hello.java` does the following:

1. **Lex** — turns text into a stream of tokens (`public`, `class`, `Hello`, `{`, ...).
2. **Parse** — builds an abstract syntax tree (AST) from the tokens.
3. **Resolve symbols** — for every name (`String`, `System.out`, `args`), find what it refers to. This is when imports, the classpath, and module path matter (Module 07).
4. **Type-check** — verify every operation is well-typed. This is where most "doesn't compile" errors come from.
5. **Rewrite** — desugar high-level constructs into simpler ones. For example:
   - `for (T x : iterable)` becomes a regular `Iterator` loop.
   - String `+` concatenation in many cases becomes `StringBuilder.append(...)` calls (or, since Java 9, `invokedynamic` to a `StringConcatFactory`).
   - Generics are *erased* — `List<String>` and `List<Integer>` both become plain `List` plus casts (Module 12).
   - Inner classes become separate `.class` files.
   - Lambdas become `invokedynamic` instructions backed by a hidden synthetic class (Module 15).
6. **Emit bytecode** — produce one `.class` file per top-level *and* nested type.

The output is a **`.class` file** containing:
- A **constant pool** — all string literals, class names, method signatures used by this class, indexed.
- **Field info** — name, type, modifiers for every field.
- **Method info** — for each method, its signature, modifiers, and a `Code` attribute holding bytecode + a `LineNumberTable` (which bytecode index maps to which source line — that's how stack traces show line numbers).
- **Attributes** — annotations, source file name, etc.

A `.class` file is *not* an executable in the OS sense. It is data that a JVM knows how to consume.

You can inspect any class with `javap`:

```
$ javac Hello.java
$ javap -c Hello
Compiled from "Hello.java"
public class Hello {
  public Hello();
    Code:
       0: aload_0
       1: invokespecial #1   // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #7   // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #13  // String Hello, world
       5: invokevirtual #15  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

That's bytecode. A few things worth noticing even now:

- There's an *implicit* constructor `Hello()` you didn't write. The compiler synthesized it.
- It calls `Object.<init>` — every Java class extends `Object` (Module 11).
- `getstatic` reads a static field; `ldc` loads a constant; `invokevirtual` makes a virtual method call.
- `Ljava/lang/String;` is the **internal type descriptor** — `L<classname>;` for class types, `I` for `int`, `V` for void, `[I` for `int[]`. You'll occasionally see these in error messages.

You don't need to read bytecode for daily work. You should know that `javap -c` exists — it's the ultimate "what is the compiler actually doing?" tool.

---

## 3. Bytecode: what the JVM consumes

Java bytecode is a **stack-based instruction set** for an idealized machine. Each frame has:
- a **local variable array** (parameters and locals),
- an **operand stack** (where instructions push/pop their arguments and results),
- a reference to the **constant pool** of its class.

Key instruction families:
- `iload`, `istore`, `aload`, `astore` — load/store ints / object references between locals and stack.
- `iadd`, `isub`, `imul`, `idiv` — int arithmetic on the stack.
- `getfield` / `putfield` — read/write instance fields.
- `getstatic` / `putstatic` — read/write static fields.
- `invokevirtual` — virtual call (resolved by the *runtime* type of the receiver — i.e. dynamic dispatch).
- `invokestatic` — call a static method.
- `invokespecial` — call a constructor, a private method, or a `super.` call. Resolved at compile time.
- `invokeinterface` — call through an interface reference.
- `invokedynamic` — call resolved by user-supplied logic the *first* time it's hit. Used for lambdas, string concat, and `switch` on patterns.
- `if_icmpeq`, `goto`, `tableswitch`, `lookupswitch` — control flow.
- `new`, `newarray`, `anewarray` — allocate objects/arrays.
- `athrow` — throw an exception.
- `return`, `ireturn`, `areturn` — return.

There are about 200 opcodes, each one byte (hence "bytecode"). Bytecode is *verifiable* — the JVM checks every method satisfies stack/type rules before it runs (no wild jumps, no popping a non-existent stack entry, types match). This verification is what gives Java its memory-safety guarantee even for code from untrusted sources.

---

## 4. Starting the JVM

When you run `java Hello`:

1. The OS launches the `java` binary (a thin C program around HotSpot).
2. It parses arguments (`-Xmx`, `-cp`, etc.).
3. It initializes the JVM:
   - Reserves memory regions for the heap, metaspace, code cache, etc. (Module 21.)
   - Starts threads: the main thread, GC threads, JIT compiler threads, finalizer thread, signal handler thread, and so on. A "Hello, world" program already has a dozen JVM-internal threads.
4. It locates the `Hello` class (via classpath / module path), **loads** it, **links** it, **initializes** it.
5. It invokes `Hello.main(String[] args)`.
6. When `main` returns, the JVM waits for any non-daemon threads to finish, then exits.

### 4.1 Classloading: load → link → initialize

These are three distinct phases for every class:

**Load.** Find the bytes of `Hello.class` (search the classpath / module path), read them, and create an in-memory `Class<Hello>` object representing the type.

**Link.** Three sub-steps:
- *Verify*: bytecode is well-formed (the verifier I mentioned above).
- *Prepare*: allocate memory for `static` fields, set them to default values (`0`, `null`, `false`).
- *Resolve*: turn symbolic references (e.g. "the class named `java.lang.String`") into direct ones. This may trigger loading other classes — typically lazy, on first use.

**Initialize.** Run the `<clinit>` method — the static field initializers and `static {}` blocks, in source order. This happens **lazily**, the first time a class is *actively* used (instantiated, a static method called, a non-final static field accessed).

```java
class Logger {
    static final long START = System.nanoTime();
    static {
        System.out.println("Logger initialized");
    }
}
```

If nothing ever references `Logger`, that print statement never runs. The class is loaded and linked only on demand.

This lazy initialization is occasionally a source of bugs: code in `static` blocks runs at unpredictable times relative to other code. Don't rely on side effects in `static` blocks.

### 4.2 Where the bytes come from: the classloader hierarchy

The JVM has a tree of classloaders, each delegating up before searching itself:

- **Bootstrap classloader** — loads core classes (`java.lang.*`, `java.util.*`) from the JDK itself. Implemented in native code.
- **Platform classloader** (Java 9+; called extension loader pre-9) — loads JDK modules outside core (`java.sql`, `java.xml`, etc.).
- **System / application classloader** — loads classes from your `-cp` classpath. This is the one your code typically runs in.
- **Custom classloaders** — frameworks (Spring Boot, OSGi, application servers) load plugins, hot-redeploys, or isolated module subsets via custom loaders.

Standard rule: a child classloader asks its parent first. Only if the parent can't find the class does the child search its own paths. This *delegation model* prevents user code from shadowing `java.lang.String`. You'll occasionally see `ClassLoader` API used directly — usually only by frameworks. Most application code never touches it.

---

## 5. Interpretation, profiling, JIT

Once a method is loaded, the JVM doesn't immediately compile it to native code. Instead:

1. **Interpret** — the JVM walks the bytecode and executes each instruction. Slow per instruction, but starts immediately.
2. **Profile** — while interpreting, the JVM tracks how often each method is called, which branches are taken, what types pass through `invokevirtual` calls.
3. **Tier up** — once a method is "hot" (e.g. called many times, or has a hot loop — counted by *invocation* and *backedge* counters), the **JIT** (Just-In-Time compiler) compiles it to native code.
4. **Replace on the fly** — subsequent calls go to the native code. If the method is *currently* running in interpreted mode (a long loop), an **on-stack replacement (OSR)** can swap the running method to native code mid-execution at a safepoint (Module 21).
5. **Deoptimize** — if a JIT optimization assumed something that turns out to be wrong (e.g. that a class is never subclassed, then a new subclass loads), the JVM throws away the native code and falls back to the interpreter, possibly to recompile later.

HotSpot has two JIT compilers used in tiers:

- **C1** (client) — fast compilation, modest optimization. Used early.
- **C2** (server) — slow compilation, aggressive optimization (inlining, loop unrolling, escape analysis, vectorization). Used for hottest methods.

This **tiered compilation** is why Java programs:
- start cool but warm up,
- often run as fast as C/C++ in steady state,
- benchmark badly if you only measure the first 100ms.

You can see what got compiled with `-XX:+PrintCompilation`. You can disable JIT entirely with `-Xint` (then watch your code crawl).

GraalVM offers an alternative high-tier JIT (Graal) and **ahead-of-time** (AOT) compilation to native binaries — useful for fast startup (CLI tools, serverless), but we won't use it here.

---

## 6. Memory at runtime

Inside one JVM process, memory is divided into regions. A simplified picture:

```
┌─────────────────────────────────────────────────┐
│ Heap                                            │
│   - all Java objects live here                  │
│   - garbage-collected                           │
│   - subdivided (young / old generations etc.)   │
├─────────────────────────────────────────────────┤
│ Per-thread stacks                               │
│   - one stack per live thread                   │
│   - holds method frames: locals, operand stack  │
│   - primitives and references stored here       │
│   - StackOverflowError if too deep              │
├─────────────────────────────────────────────────┤
│ Metaspace                                       │
│   - class metadata (Class objects, method info) │
│   - in native memory (off-heap), since Java 8   │
├─────────────────────────────────────────────────┤
│ Code cache                                      │
│   - JIT-compiled native code for hot methods    │
├─────────────────────────────────────────────────┤
│ Direct buffers / native memory                  │
│   - allocations made via NIO ByteBuffer.allocateDirect, JNI │
└─────────────────────────────────────────────────┘
```

Two facts you must internalize:

- **All Java objects live on the heap.** When you write `new Foo()`, that allocates on the heap and gives you back a reference (effectively a pointer the JVM owns).
- **Local variables holding references live on a stack.** When the method returns, its frame pops; the locals disappear; but the heap objects they pointed to may still be alive (referenced from elsewhere) or become garbage.

The heap is partitioned by the GC into generations (young / old) and regions, depending on which collector you use. Module 21 covers G1, ZGC, and the others in detail.

---

## 7. Garbage collection (one-paragraph version, pre-Module 21)

The GC's job is to identify objects no longer reachable from any *root* (active stack frames, static fields, JNI handles, ...) and reclaim their memory. Java has had several collectors over the years; the modern defaults are:
- **G1** (default since Java 9) — pause-time-targeted, region-based, generational.
- **ZGC** and **Shenandoah** — concurrent, sub-millisecond pauses, used for large heaps.
- **Parallel GC** — throughput-oriented, used in batch jobs.

You don't choose at code-write time. You choose at runtime via flags. Module 21.

---

## 8. Putting it together — what happens when `Hello` runs

```
$ java Hello
```

1. `java` launcher starts. Reads `JAVA_TOOL_OPTIONS` env, command-line flags.
2. JVM initializes: reserves heap, starts GC threads, starts JIT threads, etc.
3. Bootstrap classloader loads `java.lang.Object`, `java.lang.String`, `java.lang.System`, ...
4. System classloader loads `Hello.class` from current directory (default classpath = `.`).
5. JVM verifies bytecode of `Hello`.
6. JVM runs `<clinit>` for `Hello` if it has any static initializers (none here).
7. JVM calls `Hello.main(new String[]{})`.
8. `main`'s frame is pushed: locals = `args` (one slot, holds reference to the empty `String[]`), operand stack starts empty.
9. Bytecode runs in the interpreter:
   - `getstatic System.out` — pushes the `PrintStream` reference onto the stack.
   - `ldc "Hello, world"` — pushes the interned `String` reference.
   - `invokevirtual PrintStream.println(String)` — pops both, dispatches.
   - `println` writes to a `BufferedWriter` wrapping the OS stdout fd.
10. `main` returns. Its frame pops.
11. JVM checks for non-daemon threads. None remaining. Process exits.

Steps 4–11 are the entire lifetime of "Hello, world".

---

## 9. The flags you'll see

You don't need to memorize JVM flags, but recognize the categories:

- **Heap sizing**: `-Xms256m` (initial heap), `-Xmx2g` (max heap).
- **Stack size**: `-Xss512k` (per-thread).
- **GC choice**: `-XX:+UseG1GC`, `-XX:+UseZGC`, `-XX:+UseParallelGC`.
- **Diagnostics**: `-XX:+PrintCompilation`, `-Xlog:gc*` (modern GC logging), `-XX:+HeapDumpOnOutOfMemoryError`.
- **System properties**: `-Dfoo=bar` — readable from code as `System.getProperty("foo")`.
- **Module path**: `--module-path`, `--add-modules`.
- **Classpath**: `-cp dir:lib1.jar:lib2.jar` (or `;` on Windows).

You'll always set at least `-Xmx` in production. Everything else has reasonable defaults.

---

## 10. Try this

1. Run `javap -c Hello`. Identify (a) the implicit no-arg constructor; (b) the `getstatic` for `System.out`; (c) the `ldc` for the string.
2. Add an `int x = 1 + 2;` to `main` and disassemble again. Notice the compiler folds it into a single `iconst_3` — *constant folding* at compile time.
3. Run `java -XX:+PrintCompilation -version`. You'll see hundreds of methods being compiled — the JIT warming up the JDK itself before your code even runs.
4. Look at `Hello.class` size. Look at `Hello.java` size. Why is the class so much bigger? (Constant pool, attributes, descriptors, the synthesized constructor.)

---

**Next:** [Module 03 — Variables, primitives, references](./03-variables-primitives-references.md)
