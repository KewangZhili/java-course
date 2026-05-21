# Module 08 — Access Modifiers, `static`, `final`

> Goal: master the modifiers you sprinkle in front of every Java declaration. Why they exist, what each one means, when to use which. After this module the keywords stop being noise.

These three are responsible for how Java code is structured at scale. They control:
- *who can see* a member (access modifiers),
- *who owns* a member: an instance or the class itself (`static`),
- *whether* a member can be reassigned / overridden / extended (`final`).

---

## 1. The four access levels

Java has four levels of visibility for class members. They form a strict ordering — each level is a superset of the previous:

```
private  ⊂  package-private (default)  ⊂  protected  ⊂  public
```

Pictorially (✓ = accessible from there):

| Visible from... | `private` | *package* | `protected` | `public` |
|---|:-:|:-:|:-:|:-:|
| Same class | ✓ | ✓ | ✓ | ✓ |
| Same package, different class | | ✓ | ✓ | ✓ |
| Subclass in different package | | | ✓ | ✓ |
| Unrelated class in different package | | | | ✓ |

### 1.1 `private`

Visible **only inside the class that declared it**. Other classes — even in the same package, even subclasses — cannot see it.

```java
public class Account {
    private long balanceCents;        // only Account methods touch this
    public long balance() { return balanceCents; }
}
```

`private` is the **default for fields**. Almost every field in well-designed code is `private`. Methods that are internal helpers are `private`. The set of `public` members is your class's *public API*.

Subtlety: **`private` is per top-level class**, not per file or per nested class. A nested class can access its outer class's `private` members and vice versa — they're considered "the same class" for access purposes.

```java
public class Outer {
    private int secret = 42;

    static class Nested {
        int peek(Outer o) { return o.secret; }   // OK — Nested is inside Outer
    }
}
```

### 1.2 *No modifier* — package-private (sometimes called "default")

If you write **no** access modifier, the member is **package-private** — visible only within the same package.

```java
package com.example.shop;
class Order {                  // package-private class — visible only within com.example.shop
    int id;                    // package-private field
    void log() { ... }         // package-private method
}
```

This is Java's underused middle ground. Use it when several classes in the same package collaborate, but you don't want to expose internals to the world.

There's **no keyword** for this level. Some style guides use a comment `/* package */ void foo()` to make it explicit; most don't.

### 1.3 `protected`

Visible to **subclasses (anywhere)** and to **classes in the same package**.

```java
public class Vehicle {
    protected int speed;             // subclasses can read/write
    protected void accelerate() { ... }
}

public class Car extends Vehicle {
    void floorIt() { speed = 200; }   // OK — inherited protected
}
```

Subtle rule: a subclass in a *different package* may access `protected` members **only on instances of its own class (or further subclasses)**, not arbitrary parent instances:

```java
package com.x;
public class Parent { protected int v; }

package com.y;
import com.x.Parent;
public class Child extends Parent {
    void f(Parent other) {
        // this.v = 1;       // OK
        // other.v = 1;      // ERROR — outside same package, not via subclass receiver
    }
}
```

In practice you'll mostly use `protected` to expose internals to subclasses while keeping them hidden from the world. It's narrower than people often think.

### 1.4 `public`

Visible everywhere — across all packages, all modules. This is your **publicly committed API**. Once a class or method is `public`, downstream code can depend on it; changing or removing it is a breaking change.

A common discipline:
- `public` only what users *need*.
- `private` everything else.
- `package-private` and `protected` for collaborators within an implementation.

### 1.5 Class-level access

For a **top-level class**, only two modifiers are allowed: `public` or none. There's no top-level `private` or `protected`. (Inner / nested classes can be `private` etc.)

A `public` top-level class must live in a file whose name matches: `Foo.java` for `public class Foo`. A package-private class can live in any file (and you may have multiple package-private classes per file).

### 1.6 What about modules?

Java 9's module system adds another layer above packages: a package is only visible *outside* the module if the module `exports` it. A `public` class in a non-exported package isn't actually visible to other modules. You'll see this in the JDK itself (`sun.*`, `jdk.internal.*`). For application code on the classpath, packages and the four-level model are what you encounter daily.

### 1.7 Decision rule

Default to the **most restrictive level that works**.
- Field: start `private`. Open it only if needed.
- Method called only from within the class: `private`.
- Method called from elsewhere in the same package only: package-private.
- Method called from subclasses outside the package: `protected`.
- Method that's part of your stable API: `public`.

In SFDC-Bazel-style codebases you'll see almost every field is `private` and almost every "internal" method is package-private; the `public` surface is small and intentional.

---

## 2. `static` — class-level vs instance-level

`static` says: this member belongs to **the class itself**, not to any instance. There's exactly one of it, no matter how many instances exist.

### 2.1 `static` fields

```java
public class Counter {
    private static int totalCounters = 0;   // one slot, shared
    private int n = 0;                       // one slot per instance

    public Counter() {
        totalCounters++;
    }

    public static int totalCounters() { return totalCounters; }
}

new Counter();
new Counter();
System.out.println(Counter.totalCounters());   // 2
```

`static` fields are stored in the `Class` object itself (in metaspace, Module 02). They live for the entire run of the JVM (well, until the class is unloaded — typically never).

### 2.2 `static` methods

A `static` method:
- has no implicit `this`,
- cannot directly access instance fields or call instance methods,
- can access other `static` members.

Common uses: factory methods, utilities (`Math.sqrt`, `Collections.sort`, `Integer.parseInt`), and the `main` method.

```java
public class StringUtils {
    public static boolean isBlank(String s) {
        return s == null || s.isBlank();
    }
}

StringUtils.isBlank(" ");   // call via class name
```

You *can* call a static method through an instance reference (`obj.staticMethod()`) but it's misleading and IDEs warn about it. Always call as `ClassName.staticMethod()`.

### 2.3 Static methods are not virtual

A non-static instance method is dispatched at runtime based on the object's actual class — that's polymorphism (Module 09). A `static` method is bound at compile time to the static type of the receiver.

```java
class A { static String name() { return "A"; } }
class B extends A { static String name() { return "B"; } }

A a = new B();
A.name();         // "A"
B.name();         // "B"
a.name();         // "A" — resolved by static type A, NOT by runtime type B
```

That last call is the kind of thing that gets caught in code review. Always call static methods via the class name, not an instance.

### 2.4 `static` blocks

Code that runs once, when the class is *initialized*:

```java
public class Lookup {
    private static final Map<String, Integer> TABLE;
    static {
        TABLE = new HashMap<>();
        TABLE.put("one", 1);
        TABLE.put("two", 2);
    }
}
```

Static initializers run lazily — the first time the class is "actively used" (an instance is created, a static method is called, a non-final static field is read or written). Multiple static blocks run in source order. They run before any instance method or constructor of the class.

### 2.5 `static` nested classes

A class declared inside another class with `static`:

```java
public class Outer {
    public static class Inner { ... }
}

new Outer.Inner();
```

This is a regular class that just happens to be namespace-d under `Outer`. It does **not** hold a reference to an `Outer` instance. Most "inner" classes you'll write should be `static` nested. (Module 10 covers all four nesting flavors.)

### 2.6 The `main` method, finally fully explained

```java
public static void main(String[] args) { ... }
```

- `public` — JVM (which is "outside" your class) calls it.
- `static` — JVM doesn't have a `Hello` instance; it calls `Hello.main` directly without one.
- `void` — returns nothing. Use `System.exit(n)` for exit codes.
- `args` — argv after the class name.

That's all the modifiers in the entry-point signature, finally with reasons.

---

## 3. `final`

`final` is overloaded across three contexts. Each is its own rule.

### 3.1 `final` on a variable

The variable can be assigned **exactly once**. After that, it's locked.

```java
final int MAX = 100;
MAX = 101;   // ERROR

final List<Integer> xs = new ArrayList<>();
xs.add(1);   // OK — mutating the LIST
xs = new ArrayList<>();   // ERROR — reassigning the VARIABLE
```

`final` doesn't mean "deeply immutable". It's purely about whether the *binding* can change.

For local variables, `final` is a stylistic / safety choice. Lambdas and anonymous inner classes can capture only effectively-final local variables (Module 15) — `final` makes that explicit.

For fields, `final` is stronger: the JVM enforces "assigned exactly once during construction". A `final` field set after construction (via reflection, etc.) is undefined behavior territory.

### 3.2 `final` on a method

The method **cannot be overridden** by subclasses.

```java
public class Service {
    public final void boot() { ... }
}

public class FastService extends Service {
    @Override public void boot() { ... }   // ERROR — cannot override final
}
```

Use this when a method's behavior is essential to the class's invariants and overriding it would break the contract. Common in `protected` template-method designs.

### 3.3 `final` on a class

The class **cannot be extended**.

```java
public final class String { ... }     // (yes, java.lang.String is final)
public final class LocalDate { ... }
```

`final` classes are common in the standard library — they guarantee the implementation can't be substituted by a misbehaving subclass.

In your own code: **mark a class `final` unless you've actively designed it for inheritance.** This is Effective Java's advice — design for inheritance or prohibit it. The default should be "prohibited".

### 3.4 `static final` — constants

The combination is the standard way to declare constants:

```java
public class Limits {
    public static final int    MAX_RETRIES   = 3;
    public static final String DEFAULT_HOST  = "localhost";
    public static final long   TIMEOUT_MS    = TimeUnit.SECONDS.toMillis(30);
}
```

Conventions:
- Names in `UPPER_SNAKE_CASE`.
- Almost always `public` if they belong to a constants holder; `private` if they're internal.

A subtle compiler optimization: if the value is a *compile-time constant expression* (a literal or simple constant arithmetic) and the field is `static final`, the **compiler inlines the value at every use site** in dependent code. So if you change the constant later and don't recompile dependents, the dependents still use the old value. In practice you almost always rebuild together; just be aware.

This inlining doesn't apply when the initializer isn't a compile-time constant (e.g. `static final long T = System.nanoTime();` — the field is final but the value is computed at class init).

---

## 4. Putting modifiers in order

The conventional order, when several apply, is the one in the JLS table:

```
public protected private    abstract static final transient volatile synchronized native strictfp
```

Most often you write a subset:
- `public static final`
- `private final`
- `public abstract`
- `protected static`

Linters and IntelliJ enforce this order automatically.

---

## 5. Other modifiers (one-line tour)

You'll see these in real code; full coverage in their dedicated modules.

- `abstract` — the class or method has no implementation; subclasses must provide one. (M10)
- `synchronized` — method holds an intrinsic lock for the duration. (M18)
- `volatile` — field reads/writes are not cached per-thread; visible across threads. (M18)
- `transient` — field is skipped during default serialization.
- `native` — method is implemented in native (C/JNI) code, not Java.
- `strictfp` — strict floating-point semantics. Mostly historical; default since Java 17.
- `default` — used on interface methods to provide an implementation. (M10)
- `sealed`, `non-sealed`, `permits` — for sealed type hierarchies. (M10/M22)
- `record` — declares a record class. (M11)

---

## 6. A worked-out example showing every modifier in action

```java
package com.example.shop;

import java.util.Objects;

public final class Money {

    public static final Money ZERO_USD = new Money(0, "USD");

    private final long cents;
    private final String currency;

    public Money(long cents, String currency) {
        if (currency == null || currency.length() != 3) {
            throw new IllegalArgumentException("currency must be 3 letters");
        }
        this.cents = cents;
        this.currency = currency;
    }

    public long cents()        { return cents; }
    public String currency()   { return currency; }

    public Money plus(Money other) {
        ensureSameCurrency(other);
        return new Money(this.cents + other.cents, currency);
    }

    public Money minus(Money other) {
        ensureSameCurrency(other);
        return new Money(this.cents - other.cents, currency);
    }

    private void ensureSameCurrency(Money other) {
        if (!Objects.equals(currency, other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
    }

    public static Money usd(long dollars, long cents) {
        return new Money(dollars * 100 + cents, "USD");
    }

    @Override
    public String toString() {
        return cents + " " + currency;
    }
}
```

Annotations:
- `public final class Money` — class is part of the public API; cannot be subclassed.
- `public static final Money ZERO_USD` — a class-level constant. One per JVM.
- `private final long cents` — instance field, immutable, hidden.
- `public Money(...)` — public constructor.
- `public Money plus(...)` — public instance method, virtual (no `final`, no `static`).
- `private void ensureSameCurrency(...)` — internal helper.
- `public static Money usd(...)` — static factory method (alternative to a public constructor).

This is the typical shape of a small immutable value class. Module 11 will turn this into a record.

---

## 7. Try this

1. Take the `BankAccount` from Module 06. Mark every field `private final` that should be (which? hint: only `owner` is set once). Mark methods that subclasses absolutely should not override `final`. Mark the class `final`.
2. Add a `static int activeAccounts` that increments in the constructor and decrements in `close`. Add a `static int activeAccounts()` accessor. Why does the field need to be `private` and the accessor `public`?
3. Why is this dangerous? Fix it.
   ```java
   public class Config {
       public static List<String> ALLOWED_HOSTS = new ArrayList<>(List.of("a","b"));
   }
   ```
4. In what scenarios is `protected` actually justified versus `package-private`? (Hint: when you intend the class to be subclassed *outside its package*.)

---

**Next:** [Module 09 — The four OOP pillars](./09-oop-pillars.md)
