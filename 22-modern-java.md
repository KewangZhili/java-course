# Module 22 — Modern Java (17 → 21)

> Goal: round up the language features added since Java 11 that a modern codebase actively uses. Most are referenced individually elsewhere in the course; this module catalogs them with practical examples in one place. After this you can read any modern Java without surprise.

The headline features:
1. Local-variable type inference (`var`)
2. Switch expressions
3. Text blocks
4. Records
5. Sealed types
6. Pattern matching (`instanceof`, `switch`)
7. Virtual threads
8. The Java module system (a quick once-over)
9. Helpful `NullPointerException` messages
10. Smaller niceties

---

## 1. `var` — local type inference (Java 10+)

Already covered in M03. Worth restating:
```java
var name = "Alice";                       // String
var users = new ArrayList<User>();        // ArrayList<User>
for (var u : users) { ... }
```
- Local variables only — not fields, not return types, not parameters.
- Type is fixed at compile time.
- Use it when the type is obvious from the right side; spell out the type when it documents intent.

---

## 2. Switch expressions (Java 14+)

`switch` can produce a value:
```java
String label = switch (status) {
    case NEW         -> "fresh";
    case IN_PROGRESS -> "working";
    case DONE        -> "complete";
};
```

Properties:
- The arrow form has **no fall-through** — no `break` needed.
- Multiple labels in one branch: `case A, B, C ->`.
- `yield` returns a value from a block-bodied case:
  ```java
  String label = switch (n) {
      case 1 -> "one";
      case 2 -> "two";
      default -> {
          var s = compute(n);
          yield s;
      }
  };
  ```
- Switch expressions must be **exhaustive** — for enums and sealed types, the compiler verifies you covered every case (or have a `default`).

Use switch expressions whenever you'd write `if/else if` chains assigning to a variable.

---

## 3. Text blocks (Java 15+)

Multi-line string literals:
```java
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
```
Indentation is normalized: the closing `"""` sets the baseline, and that much leading whitespace is stripped from every line.

Newlines, quotes, and most escapes work the same as regular strings. Use `\` at end of line to suppress the newline; `\s` for trailing space (text blocks strip trailing whitespace by default).

---

## 4. Records (Java 16+)

Already covered in M11. Reminder:
```java
public record Point(int x, int y) {}
public record Range(int lo, int hi) {
    public Range { if (lo > hi) throw new IllegalArgumentException(); }   // compact ctor
}
```

Use records for immutable data classes. Auto-generates fields, accessors, `equals`, `hashCode`, `toString`.

---

## 5. Sealed types (Java 17+)

Closed inheritance hierarchies:
```java
public sealed interface Shape permits Circle, Square {}
public final  class Circle  implements Shape { ... }
public final  class Square  implements Shape { ... }
```

A subtype must be `final`, `sealed`, or `non-sealed`. The compiler knows the *exhaustive* set, enabling exhaustive `switch`:
```java
double area = switch (shape) {                      // no default needed
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Square s -> s.side() * s.side();
};
```

---

## 6. Pattern matching for `instanceof` (Java 16+)

```java
if (obj instanceof String s) {
    System.out.println(s.length());
}
if (obj instanceof String s && s.startsWith("X")) { ... }
```

`s` is bound only in the scope where the test holds. Replaces the verbose `instanceof + cast`.

---

## 7. Pattern matching in `switch` (Java 21)

The biggest language change in years. `switch` can now match against **types** and **patterns**:

```java
Object o = ...;

String label = switch (o) {
    case Integer i when i > 0 -> "positive int " + i;
    case Integer i             -> "non-positive int";
    case String s              -> "string " + s;
    case List<?> l             -> "list of size " + l.size();
    case null                   -> "null";
    default                    -> "other";
};
```

Notes:
- `case T t` matches if `o instanceof T`, then binds `t`.
- `when <bool>` is a guard.
- `case null` is now allowed — pre-21, switch always threw NPE on null.
- Combined with sealed hierarchies, switches can be exhaustive without a `default`.

### 7.1 Record patterns (Java 21)

Match and destructure records:
```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r)              implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area = switch (shape) {
    case Circle(double r)            -> Math.PI * r * r;
    case Rectangle(double w, double h) -> w * h;
};
```

Nested patterns:
```java
record Pair<A, B>(A a, B b) {}
record Box(Pair<String, Integer> p) {}

if (obj instanceof Box(Pair(String s, Integer i))) {
    // s, i are bound
}
```

Modern Java now has **algebraic data types** with exhaustive matching. Use sealed + records + switch patterns to model domain unions cleanly.

---

## 8. Virtual threads (Java 21)

Already covered in M20. Capsule:
```java
try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Socket s : sockets) pool.submit(() -> handle(s));
}
```
Cheap (millions per JVM), block-friendly, blocking I/O in virtual threads doesn't pin a carrier thread. Modern recommended path for I/O-bound workloads.

---

## 9. The Java module system (Java 9+, brief)

Modules add a stricter visibility layer above packages. A module declares:
```java
// module-info.java
module com.example.shop {
    requires java.sql;
    requires com.example.shared;
    exports com.example.shop.api;          // public to other modules
    // com.example.shop.internal is NOT exported — invisible to other modules even if public
    opens com.example.shop.entity to spring.core;   // reflection access
}
```

Practical implications:
- The JDK itself is modular. Familiarity with `java.base`, `java.sql`, `java.xml`, `java.net.http` etc. helps when reading docs.
- `--add-opens` and `--add-exports` flags are how you grant runtime access to modules that aren't otherwise open. You'll see these in build configs that touch reflection-heavy libraries (older Spring, Mockito).
- Most application code uses the **classpath**, not the **module path**. SFDC's Bazel codebase is on the classpath.

You can read modular Java without writing it.

---

## 10. Helpful NullPointerExceptions (Java 14+)

Pre-14:
```
Exception: NullPointerException
  at Foo.bar(Foo.java:42)
```

Java 14+:
```
Exception: NullPointerException: Cannot invoke "User.name()"
  because "user" is null
  at Foo.bar(Foo.java:42)
```

The JVM points to the exact expression that was null. Always on by default since Java 15. A small thing that saves real debugging time.

---

## 11. Other small upgrades since Java 11

- `String.repeat(n)` — Java 11.
- `String.strip()` — Unicode-aware trim, Java 11.
- `String.isBlank()` — Java 11.
- `Files.readString` / `writeString` — Java 11.
- `List.of`, `Map.of`, `Set.of` — immutable factories, Java 9.
- `Stream.toList()` — Java 16, returns immutable list (replacing `Collectors.toList()` which is mutable).
- `Optional.isEmpty()` — Java 11.
- `var` in lambda parameters — Java 11.
- `Map.copyOf` / `List.copyOf` — Java 10.
- `LocalDate.datesUntil`, `Period.between`, etc. — `java.time` continues to grow.
- HTTP client (`java.net.http.HttpClient`) — Java 11. Replaces the old `HttpURLConnection`.
- Records / sealed / patterns — covered above.
- Foreign Function & Memory API (Java 22+) — replacing JNI for native interop. Useful only if you go deep.

---

## 12. The HTTP client

Worth one explicit example since you'll use it often:

```java
import java.net.http.*;
import java.net.URI;

HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users/1"))
    .header("Accept", "application/json")
    .GET()
    .build();

// synchronous
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.statusCode());
System.out.println(resp.body());

// asynchronous
CompletableFuture<HttpResponse<String>> async = client.sendAsync(req, HttpResponse.BodyHandlers.ofString());
async.thenAccept(r -> System.out.println(r.body()));
```

---

## 13. A combined modern example

```java
public sealed interface Result<T> {
    record Ok<T>(T value)        implements Result<T> {}
    record Err<T>(String error)  implements Result<T> {}

    static <T> Result<T> ok(T v)   { return new Ok<>(v); }
    static <T> Result<T> err(String m) { return new Err<>(m); }
}

public class Demo {
    public static Result<Integer> parse(String s) {
        try { return Result.ok(Integer.parseInt(s)); }
        catch (NumberFormatException e) { return Result.err("not an int: " + s); }
    }

    public static void main(String[] args) {
        var input = """
                1
                two
                3
                """;
        input.lines()
             .map(Demo::parse)
             .forEach(r -> {
                 String msg = switch (r) {
                     case Result.Ok<Integer>(Integer v)    -> "ok " + v;
                     case Result.Err<Integer>(String error) -> "err " + error;
                 };
                 System.out.println(msg);
             });
    }
}
```

Records, sealed type, text block, pattern-matching switch, streams — all in 30 lines of clear code.

---

## 14. Try this

1. Convert this old `if/else if` chain to a switch expression with patterns:
   ```java
   String label;
   if (o instanceof String s)            label = "str:" + s;
   else if (o instanceof Integer i && i < 0) label = "neg int";
   else if (o instanceof Integer i)      label = "int:" + i;
   else if (o == null)                   label = "null";
   else                                   label = "other";
   ```

2. Define `sealed interface Tree<T> permits Leaf, Node` with `record Leaf<T>(T value)` and `record Node<T>(Tree<T> left, Tree<T> right)`. Write `<T> int size(Tree<T> t)` using a switch with record patterns.

3. Take a small piece of code that uses anonymous classes and convert to lambdas + records + a sealed interface. Compare line counts.

4. Spawn 100,000 virtual threads, each sleeping 1s. Compare to a fixed thread pool of 100. Note actual wall-clock time and memory.

---

**Next:** [Module 23 — Build tools](./23-build-tools.md)
