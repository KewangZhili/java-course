# Module 01 — What Java Is, and Your First Program

> Goal: by the end of this module you understand *what* Java is (the language, the platform, the runtime), how it differs from other languages you've heard of, what JDK / JRE / JVM mean, and you have written and run your first program.

---

## 1. What "Java" actually refers to

The word "Java" overloads three things. Distinguishing them is the first hurdle.

1. **The Java language.** A specification — a document — that defines syntax (`if`, `class`, `for`...), semantics ("an integer divided by zero throws `ArithmeticException`"), and rules ("a `final` field can be assigned only once"). Versioned: Java 8, 11, 17, 21, etc.

2. **The Java platform / Java SE (Standard Edition).** A set of standard libraries (`java.util.List`, `java.io.File`, `java.net.URL`...) bundled with the language. When someone says "Java has a `HashMap`", they mean the platform.

3. **The JVM (Java Virtual Machine).** A program — a runtime — that executes Java code. The JVM is also a specification, with multiple implementations (HotSpot from Oracle/OpenJDK is the dominant one; others exist: GraalVM, Eclipse OpenJ9, Azul Zing).

You write source code in the **language**, against the **platform**'s libraries, and at runtime your program executes inside a **JVM**.

---

## 2. JDK vs JRE vs JVM

You'll hear these constantly. They are nested:

```
JDK  ⊃  JRE  ⊃  JVM
```

- **JVM** — the runtime engine that actually executes Java bytecode. Just enough to *run* compiled Java.
- **JRE** (Java Runtime Environment) — the JVM **plus** the standard library (`java.util.*`, `java.io.*`, etc.) packaged together. Enough to run pre-compiled Java applications, not enough to compile.
- **JDK** (Java Development Kit) — the JRE **plus** development tools: the compiler `javac`, the disassembler `javap`, debugger `jdb`, packager `jar`, etc. This is what you install as a developer.

In modern distributions (Java 11+) the line between JRE and JDK has blurred — most distributions are just "JDKs" and the JRE-only download is rare. You install a JDK; that's enough.

**Distributions of OpenJDK:**

The reference implementation is **OpenJDK**. Vendors package and support OpenJDK builds:

- **Eclipse Temurin** (formerly AdoptOpenJDK) — community, free, widely used. Recommended.
- **Oracle JDK** — Oracle's commercial build. Free for development, paid in production.
- **Amazon Corretto**, **Azul Zulu**, **Microsoft Build of OpenJDK** — all OpenJDK builds with different vendors backing them.
- **GraalVM** — an alternative JVM with ahead-of-time native compilation.

For this course: install Temurin 21.

---

## 3. What makes Java distinctive

A short list of decisions Java's designers made, all of which matter later:

- **Compiled to bytecode, not native code.** Source `.java` becomes `.class` files of *bytecode* — instructions for the JVM, not your CPU. Bytecode is portable; the same `.class` runs on Mac, Linux, Windows. (Module 02.)
- **Runs on a managed runtime.** The JVM tracks every object, runs **garbage collection** (GC) to reclaim unused memory, and JIT-compiles hot paths to native code at runtime. You don't `malloc`/`free`. (Module 21.)
- **Statically typed.** Every variable has a type known at compile time. The compiler refuses to compile if types don't match.
- **Single inheritance for classes, multiple inheritance of interfaces.** A class extends one parent class, can implement many interfaces. (Module 10.)
- **Object-oriented by default.** All code lives inside a class. No free functions. There's no `printf` standing alone — it's `System.out.println(...)`.
- **Memory safe.** No raw pointers; you cannot read memory you haven't been given a reference to. Array bounds are checked. Casting is checked. The JVM throws an exception rather than corrupting memory.
- **Backward compatible.** Code written in 1998 still runs. The standard library has grown but rarely breaks.

---

## 4. Installing Java and verifying

```
$ java --version
openjdk 21.0.5 2024-10-15
OpenJDK Runtime Environment Temurin-21.0.5+11 (build 21.0.5+11-LTS)
OpenJDK 64-Bit Server VM Temurin-21.0.5+11 (build 21.0.5+11-LTS, mixed mode, sharing)

$ javac --version
javac 21.0.5
```

If you have only `java` and not `javac`, you have a JRE, not a JDK. Install the JDK.

If you have multiple JDKs, **SDKMAN!** is the cleanest way to switch:

```
curl -s "https://get.sdkman.io" | bash
sdk list java
sdk install java 21.0.5-tem
sdk use java 21.0.5-tem
```

---

## 5. Your first program

Create a file `Hello.java`:

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, world");
    }
}
```

Run it (since JDK 11, you don't need a separate compile step for single files):

```
$ java Hello.java
Hello, world
```

Or, the traditional two-step:

```
$ javac Hello.java        # produces Hello.class
$ java Hello              # note: the class name, not the filename
Hello, world
```

Let's dissect every word.

### 5.1 `public class Hello`

- `class` declares a new type. **Every Java program is built from classes.** There are no free functions.
- `Hello` is the class's name. Convention: **PascalCase** for class names.
- `public` is an *access modifier* — for now read it as "visible to everything". Module 08 explains the alternatives.

**File-name rule:** if a class is `public`, it must live in a file with the matching name (`Hello.java` for `class Hello`). Each `.java` file may have at most one `public` top-level class.

### 5.2 `public static void main(String[] args)`

This exact signature is the entry point — where the JVM starts your program.

Word by word:
- `public` — the JVM (which is "outside" your class) needs to call this, so it must be visible.
- `static` — belongs to the class itself, not to an instance. The JVM doesn't have a `Hello` object; it calls `Hello.main(...)` directly. Module 08.
- `void` — returns nothing.
- `main` — the *only* method name the JVM looks for as the entry point.
- `String[] args` — an array of command-line arguments. If you run `java Hello a b c`, then `args = ["a", "b", "c"]`.

Every Java program has exactly one `main` method *that the JVM uses to start*. Your program may have many classes, each can have its own `main`, and you choose which one to launch by passing the class name to `java`.

> JDK 21+: a simpler "instance main" is available — `void main() { ... }` without `public static` and without `args`. Most code you'll read uses the classic form. Use the classic form until you've fully internalized it.

### 5.3 `System.out.println("Hello, world");`

- `System` is a class in `java.lang`.
- `System.out` is a `static` field of type `PrintStream` — the standard output stream.
- `.println(...)` is a method on `PrintStream` that prints its argument followed by a newline.
- `;` ends every statement. Always.

### 5.4 The braces

Java is a curly-brace language. `{ ... }` defines a block:
- The body of a class is a block.
- The body of a method is a block.
- An `if`, `for`, `while`, `try` body is a block.

Whitespace doesn't matter to the compiler. Indentation is for humans.

---

## 6. Comments

```java
// Single-line comment.

/* Multi-line
   comment. */

/**
 * Javadoc comment — used by the `javadoc` tool to generate API docs.
 * Goes before classes, methods, fields. Recognized by IDEs.
 *
 * @param x explanation
 * @return explanation
 */
```

Javadoc is the de-facto standard for API documentation — every public method in a library you'll use has one.

---

## 7. The mental model: source → running program

To run "Hello, world":

```
[ Hello.java ] -- javac --> [ Hello.class ] -- java (JVM) --> printed output
   (text)                     (bytecode)
```

We'll walk through this pipeline in detail in Module 02 — including what bytecode looks like, what the classloader does, when the JIT kicks in, and where memory comes from.

For now: source code is text; the compiler produces bytecode; the JVM runs the bytecode.

---

## 8. A small extension

Read input. Pass an arg. Use a method.

```java
public class Greet {

    public static void main(String[] args) {
        String name = args.length > 0 ? args[0] : "world";
        System.out.println(greeting(name));
    }

    private static String greeting(String name) {
        return "Hello, " + name + "!";
    }
}
```

Run:
```
$ java Greet.java
Hello, world!
$ java Greet.java Alice
Hello, Alice!
```

Things this introduces:
- `String` — a built-in class for text. Module 05 goes deep.
- `args.length` — arrays expose their size as a `length` field (not a method).
- `args[0]` — zero-indexed array access.
- The ternary operator `cond ? a : b`, same as in C / JS / many languages.
- A second `static` method `greeting`, called as `greeting(name)` from another `static` method in the same class. No prefix needed.
- `private` — the access modifier for "only this class can call it" (Module 08).
- String concatenation with `+`.

---

## 9. Try this

1. Write `Sum.java` that prints the sum of all integer arguments. `java Sum.java 1 2 3` should print `6`. Hint: parse with `Integer.parseInt(args[i])`, loop with `for (int i = 0; i < args.length; i++)`.
2. What happens if you save `public class Hello` in a file called `Goodbye.java`? Try it. Read the compiler error.
3. Remove the `public` from `class Hello`. Does it still compile? Does it still run? (Yes and yes — `public` is required only if a class is referenced from outside its package; for a single-file program it's optional. Module 08.)

---

**Next:** [Module 02 — How Java compiles and runs](./02-compilation-and-jvm.md)
