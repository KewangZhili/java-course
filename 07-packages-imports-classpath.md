# Module 07 — Packages, Imports, Classpath

> Goal: understand how Java code is organized into namespaces (packages), how the compiler and JVM find your code (classpath / module path), and how `import` works. This is the structure underneath every multi-file project.

---

## 1. Why packages exist

Names collide. If two libraries both define a `User` class, you need a way to distinguish them. Packages are Java's namespace mechanism.

A **package** is a name like `java.util` or `com.salesforce.shared.json`. By convention:
- All-lowercase, dot-separated.
- **Reverse-DNS** of the owning organization: a company `salesforce.com` uses `com.salesforce.*`.
- Sub-packages reflect logical grouping: `com.example.shop.payment.stripe`.

Packages don't have to follow reverse-DNS — but you'll see this convention in every real codebase, including SFDC's monorepo.

---

## 2. Declaring the package

The first non-comment line of a `.java` file is its package declaration:

```java
// File: src/main/java/com/example/shop/Order.java
package com.example.shop;

public class Order {
    ...
}
```

Rules:
- A `.java` file may declare *at most one* package.
- A file with no `package` line is in the **default (unnamed) package**. Don't use the default package outside throwaway scripts. Production code is always in a named package.
- The directory layout must match the package: `com.example.shop.Order` lives at `<sourceRoot>/com/example/shop/Order.java`. The compiler enforces this.

---

## 3. Fully qualified names (FQN)

A class's full name is `<package>.<simpleName>`:
- `java.lang.String`
- `java.util.HashMap`
- `com.example.shop.Order`

Anywhere you can write a class name, you can write the fully qualified name. Two classes named `User` from different packages are unambiguous when fully qualified:

```java
java.time.Duration d1 = ...;
org.joda.time.Duration d2 = ...;
```

---

## 4. Imports

Typing `java.util.Map<java.util.List<java.lang.String>, java.lang.Integer>` everywhere would be unbearable. **Imports** let you refer to other classes by their simple name within a file.

```java
package com.example.shop;

import java.util.List;
import java.util.Map;

public class Catalog {
    Map<String, List<Order>> ordersByCustomer;
}
```

Important properties:
- **Imports are compile-time only.** No runtime cost. They don't *include* code (unlike C `#include`). They just resolve names.
- Import statements appear after the package declaration, before the class.
- Order doesn't matter to the compiler. IDEs sort them.

### 4.1 Wildcard imports

```java
import java.util.*;        // import every public top-level type from java.util
```

Pulls in `List`, `Map`, `ArrayList`, `Set`, ... but *not* sub-packages. `java.util.*` does **not** import `java.util.concurrent.*`.

Wildcards are fine; some code-style guides forbid them in favor of explicit imports because they make it slightly harder to see at a glance where a class came from. IDEs handle either way painlessly.

### 4.2 Static imports

```java
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;
import static java.util.stream.Collectors.toList;
import static java.util.Map.entry;            // factory method
```

Now `PI` and `sqrt(x)` are usable as bare names within this file. Useful for `Math.*`, `Collectors.*`, JUnit's `assertEquals`/`assertThat`, and similar utility methods.

Wildcard variant: `import static java.lang.Math.*;`.

### 4.3 What you don't need to import

Classes in the **`java.lang`** package — `String`, `Object`, `Integer`, `System`, `Thread`, `Math`, `Throwable`, `Exception`, etc. — are auto-imported. So is your own package; classes within the same package see each other without imports.

```java
package com.example.shop;
public class Catalog {
    Order o;       // no import — Order is also in com.example.shop
    String s;      // no import — java.lang.String
}
```

### 4.4 Conflict resolution

If two imports collide, only one can be on the import line; the other you reference by FQN:
```java
import java.util.Date;       // chosen
// import java.sql.Date;     // skipped

Date now = new Date();
java.sql.Date dbDate = new java.sql.Date(0L);
```

---

## 5. The classpath

When you run `java SomeClass`, the JVM has to find the bytes of `SomeClass.class` and every class it transitively references. Where does it look?

### 5.1 The classpath: a list of places to search

The **classpath** is an ordered list of:
- directories containing `.class` files (in the package layout: `<dir>/com/example/shop/Order.class`),
- `.jar` files (a JAR is a ZIP file containing the same kind of layout).

You set it via:
- `-cp` (or `-classpath`) flag: `java -cp out:lib/foo.jar com.example.Main`. On Windows, separator is `;` not `:`.
- `CLASSPATH` environment variable (rarely used today — flags are more reliable).
- The default is `.` (the current directory) if neither is set.

Order matters. The JVM stops at the first match.

### 5.2 What's inside a JAR

A `.jar` is a ZIP archive of `.class` files plus a `META-INF/` directory:
```
META-INF/MANIFEST.MF      <- text file with metadata (Main-Class, Class-Path, etc.)
com/example/shop/Order.class
com/example/shop/Catalog.class
...
```

If `META-INF/MANIFEST.MF` declares `Main-Class: com.example.Main`, you can run the JAR directly:
```
java -jar app.jar
```
The JVM reads the manifest, sets the classpath to the JAR, and calls the named main.

### 5.3 The bootstrap and platform classes

You don't put `java.lang.String` on your classpath. The JVM ships those (the **JDK modules**) and loads them via the bootstrap and platform classloaders (Module 02). Your classpath is purely *user* code.

---

## 6. Modules (very brief; Module 22 is deeper)

Java 9 introduced the **module system** (project Jigsaw) — a stricter level above packages. A *module* is a named bundle of packages with explicit declarations of:
- which packages it `exports` (visible to other modules),
- which modules it `requires` (depends on).

A module declaration lives in a file `module-info.java` at the root of the module:

```java
module com.example.shop {
    requires java.sql;
    exports com.example.shop.api;
}
```

Most application code doesn't bother with modules; they're more important for the JDK itself and for libraries that want strong API boundaries. SFDC's Bazel-built code is generally on the **classpath**, not the module path. You can read modular Java without writing it for years.

---

## 7. Visibility within and across packages

Module 08 covers access modifiers in depth, but here's the connection to packages:

- A `public` class is visible everywhere. A class with no modifier ("package-private") is visible **only inside its own package**.
- A `public` method on a public class is callable everywhere. A method with no modifier is callable only by code in the same package.
- `protected` adds *subclasses in any package* on top of package-private.

So packages aren't just naming — they form a *visibility boundary*. Classes in `com.example.shop.internal` can be designed as implementation details that callers in `com.example.shop.api` use, while `com.example.shop.internal` types stay hidden behind public façades.

---

## 8. A small two-package example

```
src/
└── main/java/
    ├── com/example/shop/
    │   ├── Catalog.java
    │   └── Order.java
    └── com/example/shop/billing/
        └── Invoice.java
```

`Order.java`:
```java
package com.example.shop;

public class Order {
    public final long id;
    public Order(long id) { this.id = id; }
}
```

`Catalog.java`:
```java
package com.example.shop;

import java.util.ArrayList;
import java.util.List;

public class Catalog {
    private final List<Order> orders = new ArrayList<>();
    public void add(Order o) { orders.add(o); }
    public List<Order> all()  { return orders; }
}
```

`Invoice.java`:
```java
package com.example.shop.billing;

import com.example.shop.Order;

public class Invoice {
    public static String describe(Order o) {
        return "Invoice for order #" + o.id;
    }
}
```

Compile and run:

```
$ javac -d out  src/main/java/com/example/shop/Order.java \
                src/main/java/com/example/shop/Catalog.java \
                src/main/java/com/example/shop/billing/Invoice.java

$ java -cp out com.example.shop.billing.Invoice         # if it had a main
```

The `-d out` flag tells `javac` to put `.class` files under `out/` mirroring the package structure: `out/com/example/shop/Order.class`, `out/com/example/shop/billing/Invoice.class`.

---

## 9. Why this matters for reading enterprise code

Real codebases have hundreds to tens of thousands of classes spread across hundreds of packages. To navigate:

- **Read the package name** to guess responsibility. `com.salesforce.foo.dao` is data access; `com.salesforce.foo.api` is external API; `com.salesforce.foo.internal` is private.
- **Track imports** to see dependencies — what does *this* file actually use?
- IntelliJ "Optimize Imports" (Ctrl-Alt-O) cleans up unused imports.
- "Go to Class" (Ctrl-N) jumps by simple name.
- "Go to Symbol" (Ctrl-Alt-Shift-N) jumps by method/field name across the project.

Module 28 returns to navigation in SFDC-Bazel codebases.

---

## 10. Try this

1. Create `com.example.geom.Point` and `com.example.geom.Circle`. `Circle` holds a `Point` center and a radius. Put both in their own files. Compile from the source root with `javac -d out src/main/java/com/example/geom/*.java`.
2. Add a `com.example.geom.test.Demo` class with a `main` that creates a `Circle`. Run it with `java -cp out com.example.geom.test.Demo`.
3. What error does the compiler give if you put `package com.example.geom;` at the top of `Point.java` but save it under `src/main/java/com/example/wrong/Point.java`? Try it.
4. Replace your individual `import` lines with a single `import com.example.geom.*;`. Does it compile? Does anything change at runtime?

---

**Next:** [Module 08 — Access modifiers, `static`, `final`](./08-access-static-final.md)
