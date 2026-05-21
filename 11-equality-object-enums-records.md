# Module 11 — Equality, `Object`, Enums, Records

> Goal: implement `equals` and `hashCode` correctly, understand the `Object` methods every class inherits, and know when to reach for `enum` vs `record` instead of writing a class from scratch. After this module you can model values precisely.

---

## 1. The `Object` class — what every class inherits

Every Java class inherits from `java.lang.Object`. The methods are:

| Method | Purpose |
|---|---|
| `boolean equals(Object o)` | logical equality |
| `int hashCode()` | hash for `HashMap` / `HashSet` |
| `String toString()` | debug / log representation |
| `Class<?> getClass()` | runtime class object |
| `Object clone()` | deep-ish copy (broken design — avoid) |
| `void finalize()` | runs before GC. **Don't use** — deprecated |
| `void wait()`, `notify()`, `notifyAll()` | low-level monitor synchronization (M18) |

Three of these — `equals`, `hashCode`, `toString` — you'll override for almost every value class you write.

---

## 2. `==` vs `.equals()` (the rule again, with mechanics)

- `==` on primitives compares values.
- `==` on references compares **identity** — do they point to the same object?
- `.equals()` is a method on `Object`, meant for **logical equality**.

The default `Object.equals` is:
```java
public boolean equals(Object o) { return this == o; }
```
which is just identity. Most classes override it.

```java
String a = "hello";
String b = "hello";
String c = new String("hello");

a == b;          // true (both interned literal)
a == c;          // false
a.equals(c);     // true
```

**Rule of thumb**: never use `==` on objects, except for `null` checks and enum constants. For everything else: `equals`.

`Objects.equals(a, b)` is null-safe — returns true if both are null, false if one is null and the other isn't.

---

## 3. The `equals` / `hashCode` contract

If you override `equals`, you **must** also override `hashCode`. The contract:

1. **Reflexive**: `x.equals(x)` is true.
2. **Symmetric**: `x.equals(y)` ⇔ `y.equals(x)`.
3. **Transitive**: `x.equals(y) ∧ y.equals(z)` ⇒ `x.equals(z)`.
4. **Consistent**: repeated calls return the same answer (assuming no mutation).
5. `x.equals(null)` is false.
6. **Equal objects must have equal hashes**: `x.equals(y)` ⇒ `x.hashCode() == y.hashCode()`.

The reverse of #6 is *not* required: two unequal objects may share a hash. (Hash collisions are normal.)

Why #6 matters: `HashMap` and `HashSet` look up keys by hash first. If you have an object that *equals* a stored key but *hashes* differently, the map can't find it.

### 3.1 Hand-rolling `equals` and `hashCode`

```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                      // identity short-circuit
        if (!(o instanceof Point p)) return false;       // null + class check (pattern matches)
        return x == p.x && y == p.y;                     // field-by-field
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                       // Objects.hash combines fields safely
    }

    @Override
    public String toString() {
        return "Point{" + x + "," + y + "}";
    }
}
```

Key idioms:
- `if (this == o) return true;` — fast path.
- `if (!(o instanceof Point p)) return false;` — handles both null and "wrong class" in one. The pattern form binds `p`.
- `Objects.hash(...)` — implementation built into the JDK.
- For primitive arrays: `Arrays.hashCode` / `Arrays.equals`.
- For deep arrays: `Arrays.deepHashCode` / `Arrays.deepEquals`.

A common bug: people write `equals(Object o)` but the parameter type is wrong (e.g. `Point p` instead of `Object o`). Then it's an *overload*, not an *override* — `HashMap` won't use it. **Always write `@Override`.**

### 3.2 Inheritance and `equals` — symmetry trap

If `Point` and `ColoredPoint extends Point` both define `equals`, you can break symmetry:
- `point.equals(coloredPoint)` returns true if `Point.equals` only checks x, y.
- `coloredPoint.equals(point)` returns false because `ColoredPoint.equals` also checks color.

There's no clean fix that satisfies all three contract properties for inheritance + `equals`. *Effective Java* recommends:
- Either use `getClass() == o.getClass()` (rejects subclass equality entirely — may surprise users), or
- Use composition instead of inheritance for value types,
- Or use **records** — which solve this for you.

---

## 4. `toString`

The default `Object.toString` returns `"ClassName@hexHash"` — useless for debugging. Always override it for any class whose instances appear in logs or test failures.

A reasonable form:
```java
@Override public String toString() {
    return "Point{x=" + x + ", y=" + y + "}";
}
```

Or with `String.format`:
```java
return "Point{x=%d, y=%d}".formatted(x, y);
```

Records auto-generate this. JDK collections call `toString` on elements when you print them, so `[Point{x=0,y=0}, Point{x=1,y=2}]` just works once each element has a sensible `toString`.

---

## 5. `enum` types

An `enum` declares a fixed, finite set of named constants — each constant is its own *singleton instance* of the enum class.

```java
public enum Status {
    NEW, IN_PROGRESS, DONE;
}

Status s = Status.NEW;
if (s == Status.NEW) { ... }       // == is fine for enums (singletons)
```

Use enums for:
- Workflow states (`NEW`, `APPROVED`, `REJECTED`).
- Modes / strategies (`READ_ONLY`, `READ_WRITE`).
- Days, months, directions.

### 5.1 Enums can have fields, constructors, methods

Enums are full classes — they can carry data and behavior:

```java
public enum HttpStatus {
    OK(200, "ok"),
    NOT_FOUND(404, "not found"),
    SERVER_ERROR(500, "boom") {
        @Override public boolean isFatal() { return true; }   // per-constant override
    };

    private final int code;
    private final String message;

    HttpStatus(int code, String message) {                    // private by default; can't be public
        this.code = code;
        this.message = message;
    }

    public int code()         { return code; }
    public String message()   { return message; }
    public boolean isFatal()  { return false; }
}

HttpStatus.OK.code();                  // 200
HttpStatus.SERVER_ERROR.isFatal();     // true
```

Per-constant overrides (`SERVER_ERROR { ... }`) effectively create an anonymous subclass for that constant.

### 5.2 Built-in enum capabilities

```java
HttpStatus.values();             // HttpStatus[] of all constants in declaration order
HttpStatus.valueOf("OK");        // parse by name; throws IllegalArgumentException if not found
HttpStatus.OK.name();            // "OK"
HttpStatus.OK.ordinal();         // 0   (declaration position — DO NOT persist)
```

`ordinal()` is dangerous to persist — adding a new enum value in the middle changes ordinals. Persist `name()` instead.

### 5.3 Enums are great `switch` targets

```java
String label = switch (status) {
    case NEW         -> "just created";
    case IN_PROGRESS -> "working";
    case DONE        -> "finished";
};
```

Modern `switch` will warn (or error) if you forget a case for an enum — especially in expression form, which must be exhaustive.

### 5.4 Enums in collections

`EnumSet` and `EnumMap` are specialized for enum keys — much faster and more memory-efficient than `HashSet<Status>` / `HashMap<Status, V>`.

```java
EnumSet<Status> open = EnumSet.of(Status.NEW, Status.IN_PROGRESS);
EnumMap<Status, Integer> counts = new EnumMap<>(Status.class);
```

### 5.5 Enums are inherently singletons

There's exactly one `Status.NEW` for the lifetime of the JVM. Hence `==` works. Hence enums are the recommended way to implement the singleton pattern:

```java
public enum Logger {
    INSTANCE;
    public void log(String msg) { ... }
}

Logger.INSTANCE.log("hi");
```

Thread-safe, serialization-safe, lazy on first reference.

---

## 6. Records (Java 16+)

A **record** is the modern way to write a small, immutable, value-type class. One line replaces dozens.

```java
public record Point(int x, int y) {}
```

That single line gives you:
- two `private final` fields `x`, `y`,
- a public **canonical constructor** `Point(int x, int y)`,
- public **accessor methods** `x()` and `y()` (note: no `get` prefix),
- `equals`, `hashCode`, `toString` — all based on all components,
- the record implicitly extends `java.lang.Record` and is `final`.

Usage:
```java
var p = new Point(3, 4);
p.x();              // 3
p.toString();       // "Point[x=3, y=4]"
new Point(3, 4).equals(new Point(3, 4));  // true
```

### 6.1 Compact constructor

Validate or normalize inputs:

```java
public record Range(int lo, int hi) {
    public Range {                        // compact constructor — no parameter list, no body assignment
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
    }
}
```

The compact constructor runs *before* the implicit field assignments. Use it for validation and parameter normalization.

### 6.2 Adding methods

Records can have any methods:

```java
public record Range(int lo, int hi) {
    public int span() { return hi - lo; }
    public boolean contains(int x) { return x >= lo && x < hi; }
}
```

### 6.3 Records can implement interfaces

```java
public sealed interface Shape permits Circle, Rectangle {}
public record Circle(double r) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}
```

### 6.4 Records cannot extend a class

They implicitly extend `java.lang.Record` and cannot extend anything else. They *can* implement multiple interfaces.

### 6.5 Records are not just data containers — they're values

The record contract is: the components are the entire identity. Two records with the same components are equal. This is rigid by design — if you need anything else (mutable fields, identity equality, inheritance), use a regular class.

### 6.6 When to use records

- POJOs / DTOs: ✓
- Immutable value types (Money, Coordinate, DateRange, Result): ✓
- Tuples in stream pipelines: ✓
- API request/response bodies: ✓
- Things with mutable state, lifecycle, identity: ✗ — use a class.
- Things that need to subclass another concrete type: ✗.

Code that used to be 80 lines of boilerplate becomes one line. Records are one of the biggest readability wins in modern Java — use them aggressively.

---

## 7. From class to record — a side-by-side

Old way:
```java
public final class Money {
    private final long cents;
    private final String currency;

    public Money(long cents, String currency) {
        this.cents = cents;
        this.currency = currency;
    }

    public long cents()      { return cents; }
    public String currency() { return currency; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;
        return cents == m.cents && Objects.equals(currency, m.currency);
    }

    @Override public int hashCode() { return Objects.hash(cents, currency); }

    @Override public String toString() {
        return "Money{cents=" + cents + ", currency=" + currency + "}";
    }
}
```

Modern way:
```java
public record Money(long cents, String currency) {
    public Money {
        Objects.requireNonNull(currency, "currency");
    }
}
```

Same semantics. Twelve lines down to four.

---

## 8. `clone()` and `Cloneable` — what to know and what to avoid

`Object.clone()` is the legacy "make a copy" mechanism. It's `protected`, requires `implements Cloneable` (a marker interface), and the default behavior is a *shallow* field-by-field copy. The whole design is broken in deeply documented ways (Effective Java item: *avoid clone*).

**Don't use `clone`. Instead:**
- For immutable types: just share the reference.
- For copies: write a copy constructor or static factory: `new ArrayList<>(other)`.
- For records: they're immutable, so *copy by reuse* — return a new record with modified components: `new Point(p.x() + 1, p.y())`.

---

## 9. `finalize()` and resource management

`Object.finalize()` was supposed to run before GC reclaims an object — a destructor analog. It is **deprecated for removal** (since Java 9). Don't use it. It's:
- non-deterministic (you don't know when, or even if, it runs),
- expensive (slows GC),
- a footgun (can resurrect objects, mask bugs).

For resources (files, sockets, connections), use `try-with-resources` and the `AutoCloseable` interface (Module 14). For more complex cleanup, the `Cleaner` API (since Java 9). For 99% of code: `try-with-resources` is the answer.

---

## 10. A complete example

```java
public sealed interface Result<T> permits Ok, Err {
    static <T> Result<T> ok(T value)         { return new Ok<>(value); }
    static <T> Result<T> err(String message) { return new Err<>(message); }
}

public record Ok<T>(T value)         implements Result<T> {}
public record Err<T>(String message) implements Result<T> {}

public class Demo {
    public static Result<Integer> parse(String s) {
        try { return Result.ok(Integer.parseInt(s)); }
        catch (NumberFormatException e) { return Result.err("not an int: " + s); }
    }

    public static void main(String[] args) {
        Result<Integer> r = parse("42");
        switch (r) {                                 // pattern-matching switch (M22)
            case Ok<Integer> o   -> System.out.println("got " + o.value());
            case Err<Integer> e  -> System.out.println("error: " + e.message());
        }
    }
}
```

Records, sealed interfaces, pattern-matching switch — together they give Java a rough equivalent to algebraic data types from functional languages. We'll lean on this for the rest of the course.

---

## 11. Try this

1. Convert the `BankAccount` from Module 06 into a class with proper `equals` (by `owner`) and `hashCode`. Then ask: should two accounts with the same owner *really* be equal? Probably not — `BankAccount` has identity, not value. So: **don't** override `equals`. (This is the lesson: records are for *values*, not for things with identity.)

2. Write `record TimeRange(LocalTime start, LocalTime end)` with a compact constructor that rejects `start > end`. Add `boolean contains(LocalTime t)`.

3. Make an enum `Direction { N, E, S, W }` with a method `Direction turnRight()` that returns the next direction clockwise. Hint: `values()` returns them in declaration order; modulo arithmetic on `ordinal()` is fine here because the set is fixed.

4. Why does this break the `equals`/`hashCode` contract? Find the bug:
   ```java
   class P {
       int x, y;
       P(int x, int y) { this.x = x; this.y = y; }
       @Override public boolean equals(Object o) {
           if (!(o instanceof P p)) return false;
           return x == p.x && y == p.y;
       }
       // no hashCode override
   }
   var p1 = new P(1, 2); var p2 = new P(1, 2);
   var s = new HashSet<P>(); s.add(p1); s.contains(p2);   // ?
   ```

---

**Next:** [Module 12 — Generics](./12-generics.md)
