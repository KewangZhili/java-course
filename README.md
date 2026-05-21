# Java: From First Principles to Enterprise

A structured, exhaustive, notes-style course on Java. Start at Module 01 — every concept is built from scratch, in order. No skipping links. No motivational fluff. Just notes, examples, and the *why* behind each thing.

The endgame: you can write competitive-programming code, read and modify large enterprise codebases (SFDC-Bazel monorepo), build Spring Boot services, and reason about JVM internals (GC, safepoints, JIT) when production stalls.

---

## How to use this course

1. Read modules **in order**. Each builds on the previous.
2. **Type the examples.** Don't paste. Muscle memory matters.
3. JDK 21 (the current LTS at time of writing) — install Temurin or use SDKMAN: `sdk install java 21.0.5-tem`.
4. Editor: IntelliJ IDEA Community Edition, or VS Code with the Java extension pack.
5. Most examples can be run with `java FileName.java` (single-file source mode, no project setup).
6. Don't skip Module 02 (compilation pipeline) and Module 21 (JVM internals). They are what make a Java *developer* rather than a Java *user*.

---

## Course map

### Part I — Foundations: writing your first Java code

| # | Module |
|---|---|
| 01 | [What Java is, JDK/JRE/JVM, your first program](./01-what-is-java.md) |
| 02 | [How Java compiles and runs — the full pipeline](./02-compilation-and-jvm.md) |
| 03 | [Variables, primitives, references](./03-variables-primitives-references.md) |
| 04 | [Operators, control flow, methods](./04-operators-control-flow-methods.md) |
| 05 | [Strings and arrays](./05-strings-and-arrays.md) |

### Part II — Object-oriented Java

| # | Module |
|---|---|
| 06 | [Classes, objects, constructors, `this`](./06-classes-and-objects.md) |
| 07 | [Packages, imports, classpath](./07-packages-imports-classpath.md) |
| 08 | [Access modifiers, `static`, `final`](./08-access-static-final.md) |
| 09 | [The four OOP pillars in Java](./09-oop-pillars.md) |
| 10 | [Inheritance, abstract classes, interfaces](./10-inheritance-abstract-interfaces.md) |
| 11 | [Equality, `Object`, enums, records](./11-equality-object-enums-records.md) |
| 12 | [Generics and the type system](./12-generics.md) |

### Part III — Standard library and idioms

| # | Module |
|---|---|
| 13 | [Collections framework](./13-collections.md) |
| 14 | [Exceptions](./14-exceptions.md) |
| 15 | [Lambdas and functional interfaces](./15-lambdas.md) |
| 16 | [Streams API and `Optional`](./16-streams-optional.md) |
| 17 | [File I/O and NIO.2](./17-io-nio.md) |

### Part IV — Concurrency

| # | Module |
|---|---|
| 18 | [Concurrency basics: threads, synchronized, JMM](./18-concurrency-basics.md) |
| 19 | [Executors, locks, atomics, concurrent collections](./19-executors-locks-atomics.md) |
| 20 | [`Future`, `CompletableFuture`, virtual threads](./20-completablefuture.md) |

### Part V — Under the hood

| # | Module |
|---|---|
| 21 | [JVM internals: memory, GC, safepoints, JIT, classloading](./21-jvm-internals.md) |
| 22 | [Modern Java (records, sealed, pattern matching, modules)](./22-modern-java.md) |

### Part VI — Building real systems

| # | Module |
|---|---|
| 23 | [Build tools: Maven, Gradle, Bazel](./23-build-tools.md) |
| 24 | [Testing: JUnit, Mockito, AssertJ](./24-testing.md) |
| 25 | [Spring core: IoC and dependency injection](./25-spring-core.md) |
| 26 | [Spring Boot](./26-spring-boot.md) |
| 27 | [Enterprise patterns, AOP, transactions](./27-enterprise-patterns.md) |

### Part VII — Reading other people's Java

| # | Module |
|---|---|
| 28 | [Reading SFDC-Bazel-style codebases](./28-reading-sfdc-bazel.md) |
| 29 | [Competitive programming in Java](./29-competitive-programming.md) |

---

## Conventions in this course

- Code blocks are **runnable** unless explicitly marked with `// fragment` or `// pseudo`.
- "JEP" = JDK Enhancement Proposal — the spec doc for a Java feature. Mentioned only when useful.
- Anything Java 17+ or 21+ is flagged at the section head.
- "Try this" exercises are at the end of each module. Do them before moving on.

Start at [Module 01 — What Java is](./01-what-is-java.md).
