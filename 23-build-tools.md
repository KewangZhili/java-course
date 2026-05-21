# Module 23 — Build Tools: Maven, Gradle, Bazel

> Goal: pick up any Java project and understand how it builds. Read its build file, add a dependency, run its tests. Three systems dominate: **Maven** (still most common in industry), **Gradle** (popular in modern projects, Android), and **Bazel** (large monorepos including SFDC).

You don't need to be an expert in all three. You need to recognize what each one is doing.

---

## 1. Why a build tool

`javac` compiles one file at a time. Real projects:
- have hundreds of files,
- depend on dozens of external libraries (downloaded as JARs),
- need to run tests, package outputs, publish artifacts,
- combine all of the above repeatably across machines.

A build tool automates this: it knows your project layout, downloads dependencies, compiles in the right order, runs tests, packages JARs, optionally publishes them.

The tool's two core jobs:
1. **Dependency management** — declare "I need Guava 33.x"; the tool fetches it from a repository (Maven Central, internal Nexus/Artifactory) and adds it to the classpath.
2. **Lifecycle / task orchestration** — run compile → test → package in order, with proper inputs/outputs.

---

## 2. Maven

Maven's been the default for nearly two decades. Big in enterprise; nearly universal in older Java code.

### 2.1 Project layout (the conventional one)

```
project/
├── pom.xml                          ← build file (XML)
├── src/
│   ├── main/
│   │   ├── java/com/example/...     ← production code
│   │   └── resources/                ← non-Java files (config, templates)
│   └── test/
│       ├── java/com/example/...      ← test code
│       └── resources/
└── target/                           ← build output (JARs, classes)
```

Maven enforces this layout. Convention over configuration is the philosophy.

### 2.2 The POM

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>shop-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>33.3.1-jre</version>
        </dependency>

        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.11.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Coordinates: a dependency is identified by `groupId:artifactId:version` ("GAV"). That tuple uniquely identifies a JAR on Maven Central.

`<scope>`:
- `compile` (default) — present at compile and runtime.
- `provided` — at compile, not packaged (e.g. servlet API in a WAR).
- `runtime` — runtime only (e.g. JDBC drivers).
- `test` — test compile/run only.

### 2.3 Lifecycle phases

Maven's *default* lifecycle is a fixed sequence:
```
validate → compile → test → package → verify → install → deploy
```

You run a phase: `mvn package` runs validate, compile, test, package (in that order). Other lifecycles: `clean` (deletes `target/`) and `site` (generates docs).

Common commands:
```
mvn clean                       # delete target/
mvn compile                     # compile main code
mvn test                        # compile + run unit tests
mvn package                     # compile + test + package JAR
mvn install                     # package + install to local repo (~/.m2)
mvn dependency:tree             # show dependency graph
mvn -DskipTests package         # skip tests
```

### 2.4 Multi-module projects

For larger codebases, a parent POM aggregates child modules:
```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>shop-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    <modules>
        <module>shop-api</module>
        <module>shop-impl</module>
        <module>shop-web</module>
    </modules>
</project>
```

`mvn install` from the parent builds all children in dependency order.

### 2.5 What people complain about

XML is verbose. The XML schema and plugin configurations get baroque. Customizing the lifecycle requires plugins. But for day-to-day "compile, test, package" Maven works extremely well and you can usually understand a `pom.xml` at a glance.

---

## 3. Gradle

Newer than Maven, popular in Spring projects, default for Android. Build script is in **Groovy** (`build.gradle`) or **Kotlin** (`build.gradle.kts`). Same conventions as Maven for layout.

### 3.1 A typical `build.gradle.kts`

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.5"
}

group = "com.example"
version = "1.0.0"

java {
    toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("com.google.guava:guava:33.3.1-jre")
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.3")
}

tasks.test {
    useJUnitPlatform()
}
```

Coordinates and conventions are the same. Notable differences from Maven:
- DSL — easier to extend with custom logic. You can write small programs in your build file.
- `implementation` / `api` distinction — `implementation` deps don't leak to consumers' compile classpaths (faster builds, cleaner APIs).
- `compileOnly` / `runtimeOnly` / `testImplementation` — finer control.
- Incremental builds — Gradle aggressively caches and skips unchanged tasks. Faster builds on large projects.

### 3.2 Common commands

```
./gradlew build                  # compile, test, package
./gradlew test
./gradlew bootRun                # run a Spring Boot app
./gradlew dependencies           # full dep graph
./gradlew tasks                  # list tasks
```

The `./gradlew` *wrapper* script ensures everyone uses the right Gradle version (downloaded into `.gradle/`).

### 3.3 What people complain about

Build scripts are turing-complete — easy to write unmaintainable build logic. But for standard Java projects, `build.gradle.kts` is concise and readable.

---

## 4. Bazel — the monorepo answer (and SFDC's choice)

Bazel is Google's build system, open-sourced. Designed for **monorepos** with millions of files where:
- correctness is paramount (no "works on my machine"),
- speed across an entire repo is critical,
- caching and remote execution are first-class features.

Salesforce's Java codebase uses Bazel.

### 4.1 BUILD files instead of pom.xml

In Bazel, every directory you want to build something in has a `BUILD` (or `BUILD.bazel`) file:

```python
java_library(
    name = "shop_lib",
    srcs = glob(["src/main/java/**/*.java"]),
    deps = [
        "//common/util:util_lib",
        "@maven//:com_google_guava_guava",
    ],
    visibility = ["//visibility:public"],
)

java_test(
    name = "shop_test",
    srcs = glob(["src/test/java/**/*.java"]),
    deps = [
        ":shop_lib",
        "@maven//:org_junit_jupiter_junit_jupiter",
    ],
)

java_binary(
    name = "shop_main",
    main_class = "com.example.shop.Main",
    runtime_deps = [":shop_lib"],
)
```

Key concepts:
- A **target** is a build artifact — `:shop_lib`, `:shop_test`, `:shop_main`. Referenced as `<package>:<name>` like `//services/shop:shop_lib`.
- `deps` is an explicit list. **No transitive auto-include.** If you use `Guava` in your code, you must list `@maven//:com_google_guava_guava` in `deps`. Sounds painful; in practice it makes dependency graphs cleaner and incremental builds dramatically faster.
- `visibility` — which other packages can depend on this. Defaults to private to the package.

### 4.2 Why Bazel for monorepos

- **Hermetic builds.** Inputs are fully declared; the same inputs always produce the same outputs.
- **Aggressive caching.** Local + remote. A change in `services/shop/` doesn't rebuild `services/billing/`.
- **Parallelism.** Bazel can fan out builds across many cores or machines.
- **Cross-language.** One tool for Java, Go, Python, TypeScript, native code.

### 4.3 Common commands

```
bazel build //services/shop:shop_lib
bazel test //services/shop:shop_test
bazel run //services/shop:shop_main
bazel query "deps(//services/shop:shop_lib)"
```

### 4.4 The cost

- Steeper learning curve.
- Adding a new external library is a multi-step process (typically updates a `maven_install.json` lock file).
- Tooling around Bazel (IDE integration, debuggers, profilers) is *less mature* than Maven/Gradle's, though improving.

For SFDC's Java codebase you'll spend more time **reading** `BUILD` files than authoring complex ones — usually you copy from a sibling target and tweak `srcs`/`deps`. Module 28 returns to navigation.

---

## 5. Dependency resolution and version conflicts

All three tools fetch JARs from **Maven repositories** (Maven Central is the public default; private ones for internal libraries). When two of your dependencies pull in different versions of the same library, you have a conflict.

- **Maven** picks the *nearest in the dependency tree* (closest = wins). Predictable but sometimes wrong.
- **Gradle** picks the *highest* version by default. Generally what you want.
- **Bazel** with `rules_jvm_external` typically pins a single version per artifact via `maven_install`. No mid-build resolution.

`mvn dependency:tree`, `./gradlew dependencies`, and `bazel query "deps(...)"` show the graph. You'll do this when diagnosing a `NoSuchMethodError` or `ClassNotFoundException` at runtime — usually a sign of version skew.

---

## 6. Reading any project's build file

Pattern for opening a fresh repo:
1. Look for `pom.xml`, `build.gradle{,.kts}`, or `BUILD{.bazel}`. That tells you the build system.
2. Find the dependencies block. Skim what's in there — it tells you what frameworks the project uses (Spring? JPA? Jackson?).
3. Find `mainClass` / `mainClassName` / `java_binary main_class`. That's the entry point.
4. Find the test config to understand how tests run.
5. README usually has the build commands. If not, the standard ones (above) almost always work.

---

## 7. Local repository conventions

- **Maven** — `~/.m2/repository/`. Cached downloaded JARs.
- **Gradle** — `~/.gradle/caches/`.
- **Bazel** — per-workspace `bazel-out/` plus a remote/local cache. `~/.cache/bazel/`.

Sometimes things go wrong (corrupt cache, partial download). Last-resort fixes:
- `mvn -U` (force update snapshots) or wipe `~/.m2/repository/<groupId>/`.
- `./gradlew --refresh-dependencies` or wipe `~/.gradle/caches/`.
- `bazel clean` (or `bazel clean --expunge` for full wipe).

---

## 8. A complete tiny example — Maven

A minimal compilable project:

```
demo/
├── pom.xml
└── src/main/java/com/example/Hello.java
```

`pom.xml`:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>1.0.0</version>
    <properties>
        <maven.compiler.release>21</maven.compiler.release>
    </properties>
</project>
```

`Hello.java`:
```java
package com.example;
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Maven");
    }
}
```

Build and run:
```
mvn package
java -cp target/demo-1.0.0.jar com.example.Hello
```

Or use the `exec-maven-plugin` for `mvn exec:java -Dexec.mainClass=com.example.Hello`.

---

## 9. Try this

1. Find an open-source Java project on GitHub. Identify its build system. Build it. Run its tests.
2. Add a dependency on `com.google.guava:guava:33.3.1-jre` to a tiny Maven project. Use it from a test.
3. Generate the dependency tree (`mvn dependency:tree` or `./gradlew dependencies`). Find a transitive dependency that surprises you.
4. (If you have access to SFDC's monorepo) Find a `BUILD.bazel`, identify the `java_library` and `java_test` targets, and run `bazel test` on a single target. Note the cache hit on a second run.

---

**Next:** [Module 24 — Testing](./24-testing.md)
