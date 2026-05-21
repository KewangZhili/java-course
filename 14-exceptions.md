# Module 14 — Exceptions

> Goal: handle errors the Java way. Understand the difference between checked and unchecked exceptions, the `try`/`catch`/`finally`/`try-with-resources` machinery, and the conventions for designing your own exception types.

---

## 1. The exception hierarchy

```
Throwable
├── Error                      (JVM-level, don't catch)
│     OutOfMemoryError, StackOverflowError, ...
└── Exception
      ├── RuntimeException     (unchecked)
      │     NullPointerException, IllegalArgumentException,
      │     IllegalStateException, IndexOutOfBoundsException, ...
      └── (everything else)    (checked)
            IOException, SQLException, InterruptedException, ...
```

Three categories, distinguished by whether the compiler forces you to handle them:

- **`Error`** — serious JVM failures (out of memory, stack overflow). You don't catch these. They terminate your application.
- **Checked exceptions** (subclasses of `Exception` but **not** `RuntimeException`) — the compiler enforces "either catch it or declare you throw it". `IOException`, `SQLException`, `InterruptedException`.
- **Unchecked exceptions** (subclasses of `RuntimeException`, plus `Error`) — no compiler enforcement. Throwable from anywhere. `NullPointerException`, `IllegalArgumentException`.

A method's signature can declare:
```java
public void readFile(Path p) throws IOException { ... }
```
which contributes to the compile-time checked-exception model: any caller must catch `IOException` or declare it themselves.

---

## 2. Throwing

```java
throw new IllegalArgumentException("balance must be non-negative");
throw new IOException("file not found", causeException);    // chained
```

`throw` requires a `Throwable`. `new SomeException(message, [cause])` is the canonical form. The cause sets the *chained* exception — visible in stack traces.

You can throw any subclass of `Throwable`:
```java
throw new MyDomainException(...);
```

If the exception is **checked**, you must either:
- catch it before the method returns, or
- declare it in the method's `throws` clause:
  ```java
  void f() throws IOException { ... }
  ```

If it's **unchecked** (RuntimeException family), no declaration is required. The compiler still allows you to declare it for documentation, and many style guides suggest doing so.

---

## 3. `try` / `catch` / `finally`

```java
try {
    risky();
} catch (IOException e) {
    log.error("IO failed", e);
    return null;
} catch (RuntimeException e) {
    log.error("unexpected", e);
    throw e;
} finally {
    cleanup();      // always runs
}
```

Order matters: catch blocks are tried top-to-bottom. A more specific exception must come *before* a more general one (otherwise the general clause swallows everything and the specific clause is unreachable).

### 3.1 Multi-catch

A single block can catch multiple unrelated exception types:

```java
try {
    ...
} catch (IOException | SQLException e) {
    log.error("operation failed", e);
}
```

Inside such a block, `e` is typed as the **common supertype** of the listed types. You can't reassign `e` (it's effectively final).

### 3.2 `finally`

Always runs — after a normal return, after a thrown exception (caught or not), even after `return` inside `try` or `catch`.

```java
try {
    return 1;
} finally {
    return 2;       // legal, but never do this — silently overrides
}
```

The `finally` block returning or throwing overrides the `try`'s outcome. Don't return or throw from `finally`.

### 3.3 What gets to your stack trace

Java records the full stack trace at the throw point. Pretty-printed by `Throwable.printStackTrace()` and standard logging. Includes class names, method names, file names, line numbers — that's what `LineNumberTable` in the bytecode is for (Module 02).

---

## 4. `try-with-resources` — Java's RAII

Any object whose class implements `AutoCloseable` (or `Closeable`) can be declared in `try (...)` parentheses. Its `close()` method runs at the end of the block, automatically, even if an exception is thrown:

```java
try (var in = Files.newBufferedReader(path);
     var out = Files.newBufferedWriter(other)) {
    String line;
    while ((line = in.readLine()) != null) {
        out.write(line); out.newLine();
    }
} // in.close() and out.close() called here in reverse order
```

Multiple resources separated by `;`. They're closed in reverse declaration order (LIFO).

This replaces the classic verbose pattern:
```java
BufferedReader in = null;
try {
    in = Files.newBufferedReader(path);
    ...
} finally {
    if (in != null) try { in.close(); } catch (IOException ignore) {}
}
```

**Always use try-with-resources for files, streams, sockets, DB connections.** Forgetting `.close()` leaks file descriptors / connections — a common cause of "too many open files" in production.

### 4.1 Suppressed exceptions

If `try` throws and `close()` also throws, the original is the *primary* and the second is *suppressed* (attached as `Throwable.getSuppressed()`). Both appear in stack traces. This is exactly what you want — you see the original cause without losing the cleanup failure.

### 4.2 Implementing `AutoCloseable`

Your own resource:
```java
public class TempFile implements AutoCloseable {
    private final Path path;
    public TempFile() throws IOException {
        path = Files.createTempFile("x", ".tmp");
    }
    public Path path() { return path; }

    @Override public void close() throws IOException {
        Files.deleteIfExists(path);
    }
}

try (var t = new TempFile()) {
    Files.writeString(t.path(), "hello");
}   // file deleted automatically
```

---

## 5. Checked vs unchecked: when to use which

This is opinionated. The mainstream view:

- Use **unchecked exceptions** for *programming errors* — bugs that the caller could have prevented. NPE, IllegalArgument, IllegalState. The caller shouldn't catch them; they should fix the bug.
- Use **checked exceptions** for *recoverable conditions* — situations the caller is expected to handle. File not found, network timeout. Forcing the caller to acknowledge "this can fail" is the language helping you.

In practice, modern Java (and frameworks like Spring) leans toward unchecked exceptions. Reasons:
- Checked exceptions don't compose well with lambdas (a `Function<T, R>` can't throw a checked exception).
- Long `throws IOException` chains pollute every method signature.
- Most "recoverable" conditions are handled at boundaries (top-level handler), not at every level.

Result: most application-defined exceptions extend `RuntimeException`. The standard library still throws checked exceptions where they were originally introduced (`IOException`, `SQLException`).

A common pattern is to **wrap** checked exceptions:
```java
try {
    return Files.readString(path);
} catch (IOException e) {
    throw new UncheckedIOException(e);          // standard JDK wrapper
}
```

`UncheckedIOException`, `RuntimeException`, custom domain exceptions — the goal is to bubble through stream/lambda code without `throws` chains.

---

## 6. The standard exception types

Familiarity with these saves time. Most are in `java.lang`:

| Exception | Use for |
|---|---|
| `IllegalArgumentException` | bad parameter (negative count, blank string) |
| `IllegalStateException` | object is in wrong state for the call (closed, uninitialized) |
| `NullPointerException` | dereferenced null. Usually a bug in caller |
| `IndexOutOfBoundsException` | array / list / string index out of range |
| `ArithmeticException` | integer divide-by-zero |
| `ClassCastException` | bad cast |
| `UnsupportedOperationException` | this method is not implemented (e.g. `unmodifiableList.add`) |
| `NumberFormatException` | `Integer.parseInt("abc")` |
| `ConcurrentModificationException` | mutated a collection during iteration |
| `IOException` (checked) | file/network failure |
| `InterruptedException` (checked) | thread was interrupted while blocked (M18) |
| `TimeoutException` (checked) | a `Future.get(timeout)` timed out |

Don't invent custom exceptions for these cases — reuse the standard ones.

---

## 7. Designing your own exceptions

For domain-specific failures, define an exception class:

```java
public class InsufficientFundsException extends RuntimeException {
    private final long requested;
    private final long available;

    public InsufficientFundsException(long requested, long available) {
        super("Requested %d, only %d available".formatted(requested, available));
        this.requested = requested;
        this.available = available;
    }

    public long requested()  { return requested; }
    public long available()  { return available; }
}
```

Conventions:
- Name ends in `Exception`.
- Extend `RuntimeException` unless you have a strong reason for checked.
- Provide a constructor with `String message`, ideally another with `(String, Throwable cause)` so callers can chain.
- Carry **structured data** (the requested/available amounts here). Logs and tests can use it.

For a family of related exceptions:
```java
public abstract class PaymentException extends RuntimeException {
    protected PaymentException(String msg) { super(msg); }
    protected PaymentException(String msg, Throwable cause) { super(msg, cause); }
}
public class CardDeclinedException     extends PaymentException { ... }
public class CardExpiredException      extends PaymentException { ... }
public class GatewayTimeoutException   extends PaymentException { ... }
```

---

## 8. The chain: catching and rethrowing with context

Don't lose the original cause:
```java
try {
    return repository.findById(id);
} catch (SQLException e) {
    throw new DataAccessException("failed to load user " + id, e);   // pass cause
}
```

`Throwable.getCause()` will return the original. The stack trace shows both. Bad pattern:
```java
catch (SQLException e) {
    throw new DataAccessException("failed");          // lost cause and original stack
}
```

---

## 9. `try` semantics: control flow gotchas

```java
int demo() {
    try {
        return 1;
    } finally {
        System.out.println("cleanup");
    }
}
```
Returns `1`, prints `cleanup` *first*. The `try` block's value is computed and stashed; `finally` runs; the stashed value is returned.

```java
int demo() {
    try {
        return 1;
    } finally {
        return 2;       // overrides
    }
}
```
Returns `2`. Don't do this.

A `finally` block that throws also overrides whatever the `try`/`catch` was doing:
```java
try { return 1; } finally { throw new RuntimeException(); }   // throws, doesn't return
```

---

## 10. Exception handling and lambdas

A lambda matching `Function<T, R>` cannot throw a checked exception:
```java
Function<Path, String> f = p -> Files.readString(p);   // ERROR — IOException is checked
```

Two fixes:

(1) Wrap inside the lambda:
```java
Function<Path, String> f = p -> {
    try { return Files.readString(p); }
    catch (IOException e) { throw new UncheckedIOException(e); }
};
```

(2) Define your own throwing functional interface:
```java
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;
}
```
And write helpers that adapt. Module 15 covers this style.

---

## 11. Idioms

### 11.1 Validate at construction
```java
public Account(String owner, long openingCents) {
    this.owner = Objects.requireNonNull(owner, "owner");
    if (openingCents < 0) throw new IllegalArgumentException("openingCents must be >= 0");
    ...
}
```

### 11.2 Fail-fast on impossible states
```java
public void close() {
    if (closed) throw new IllegalStateException("already closed");
    closed = true;
    ...
}
```

### 11.3 Don't catch and ignore

```java
catch (IOException e) {}   // do not do this
```

If you genuinely don't care, comment why. Consider logging at debug. Silent swallowing eats production diagnostics.

### 11.4 Catch `Exception` only at top-level boundaries

A web server's request handler might catch `Exception` to convert it to an HTTP 500 — that's valid. Inside business logic, catch *specific* exceptions.

### 11.5 Don't catch `Throwable`

`Throwable` includes `Error` (OOM, StackOverflow). Catching those is almost always wrong — you can't sensibly recover.

---

## 12. A worked example

```java
import java.io.*;
import java.nio.file.*;
import java.util.List;

public class FileOps {

    public static String readOrDefault(Path p, String fallback) {
        try {
            return Files.readString(p);
        } catch (NoSuchFileException e) {
            return fallback;
        } catch (IOException e) {
            throw new UncheckedIOException("failed to read " + p, e);
        }
    }

    public static void copy(Path src, Path dst) {
        try (var in  = Files.newBufferedReader(src);
             var out = Files.newBufferedWriter(dst)) {
            String line;
            while ((line = in.readLine()) != null) {
                out.write(line);
                out.newLine();
            }
        } catch (IOException e) {
            throw new UncheckedIOException("copy failed", e);
        }
    }

    public static void main(String[] args) {
        System.out.println(readOrDefault(Path.of("missing.txt"), "no file"));
    }
}
```

This shows: catching specific subtypes (`NoSuchFileException` before `IOException`), wrapping checked exceptions, try-with-resources for two streams, and using `Path` from NIO.2 (Module 17).

---

## 13. Try this

1. Write `int parsePositive(String s)` that returns `Integer.parseInt(s)` if it's >= 0, else throws `IllegalArgumentException` with a helpful message.
2. Write a method `String urlOrNull(String s)` that returns a parsed URL's host if `s` parses, else `null`. Use `try`/`catch` on `MalformedURLException` (or `URISyntaxException` if you use `URI`).
3. Why does the following compile? What's wrong with it operationally?
   ```java
   try { ... } catch (Exception e) { /* ignored */ }
   ```
4. Implement `try-with-resources` style for a custom `Connection` class. What does the JDK get when `close()` itself throws inside a `try-with-resources`?

---

**Next:** [Module 15 — Lambdas and functional interfaces](./15-lambdas.md)
