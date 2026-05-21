# Module 04 — Operators, Control Flow, Methods

> Goal: by the end you can write any imperative logic in Java — arithmetic, conditionals, loops, methods (including overloads and varargs) — and you understand exactly how Java passes arguments. The "every Java method is pass-by-value" rule will become muscle memory.

---

## 1. Operators

Mostly conventional. The categories:

### 1.1 Arithmetic

`+ - * / %` — work on numeric primitives.

- `/` between two integers is *integer division* (truncates toward zero). `7 / 2` → `3`. `-7 / 2` → `-3`.
- `%` returns the remainder with the **dividend's sign**. `-7 % 2` → `-1`.
- Integer division by zero **throws `ArithmeticException`**. Floating division by zero gives `Infinity`/`NaN`.
- Integer overflow **silently wraps**. `Integer.MAX_VALUE + 1` is `Integer.MIN_VALUE`. There's no overflow check unless you use `Math.addExact(a, b)` (which throws on overflow).

### 1.2 Increment / decrement

`++` / `--` exist as prefix and postfix. Same semantics as C-family languages.

```java
int i = 5;
int a = i++;     // a = 5, i = 6 (post: read first, then increment)
int b = ++i;     // b = 7, i = 7 (pre: increment first, then read)
```

In modern code, prefer the explicit `i = i + 1` for clarity unless the increment is the obvious idiom (`for` loops, etc.).

### 1.3 Comparison

`< <= > >= == !=`. They produce `boolean`.

For *references*, `==` and `!=` compare *identity* (do they point to the same object?). For value equality on objects, use `.equals()` — see Module 11. The only references where `==` is idiomatic are enum constants and `null` checks.

### 1.4 Logical

- `&&`, `||` — short-circuit. The right side isn't evaluated if the left determines the result. **Use these.**
- `&`, `|` — non-short-circuit; both sides always evaluated. (Also bitwise on integers.)
- `!` — logical not.

```java
if (s != null && s.length() > 0) { ... }   // short-circuits the null check
```

If you used `&` instead of `&&` here, `s.length()` would always be evaluated and crash on `null`.

### 1.5 Bitwise

On integer types: `& | ^ ~`, plus shifts:
- `<<`  — left shift, fills with 0
- `>>`  — arithmetic right shift, fills with sign bit
- `>>>` — logical right shift, fills with 0 (Java has this; C doesn't)

```java
int x = -1;
System.out.println(Integer.toBinaryString(x >> 1));   // 11111111111111111111111111111111
System.out.println(Integer.toBinaryString(x >>> 1));  // 01111111111111111111111111111111
```

Shift amount is taken `mod 32` for `int`, `mod 64` for `long`. So `1 << 32` is `1`, not `0`.

### 1.6 Assignment

`= += -= *= /= %= &= |= ^= <<= >>= >>>=`.

`x += y` is equivalent to `x = (T)(x + y)` where `T` is the type of `x` — that *implicit cast* is occasionally surprising:

```java
byte b = 100;
b = b + 1;     // ERROR: int can't fit in byte
b += 1;        // OK — implicit cast back to byte
```

### 1.7 String `+`

If either operand of `+` is a `String`, the other is converted (via `String.valueOf`), and the result is a new `String`:

```java
"x = " + 5;             // "x = 5"
"a" + 1 + 2;            // "a12"   (left to right: "a" + 1 = "a1", then + 2)
1 + 2 + "a";            // "3a"    (1 + 2 = 3 first, then 3 + "a")
```

### 1.8 Ternary

```java
String label = n >= 0 ? "non-negative" : "negative";
```

Both branches must produce compatible types. The result type is the broader one.

### 1.9 `instanceof`

```java
if (o instanceof String) { ... }                   // classical
if (o instanceof String s) { System.out.println(s.length()); }   // pattern, Java 16+
```

The pattern form binds `s` to the value, properly typed, in scopes where the test holds (the `then` branch, `&&`-continuations).

---

## 2. Control flow

### 2.1 `if` / `else`

```java
if (cond) {
    ...
} else if (other) {
    ...
} else {
    ...
}
```

Braces are optional for single statements but always use them. `if (x = 5)` doesn't compile in Java (assignment isn't a `boolean`) — so you can't accidentally write `==` as `=` on a boolean.

### 2.2 `for` loop

```java
for (int i = 0; i < n; i++) {
    ...
}

// multiple init / update
for (int i = 0, j = n - 1; i < j; i++, j--) { ... }

// any of init/cond/update can be omitted
for (;;) { /* infinite */ }
```

### 2.3 Enhanced `for` ("for-each")

```java
for (T x : iterable) {
    ...
}
```

Works on any array and on anything that implements `Iterable<T>`. Read-only access — you can read each element but you can't reassign through `x` or `remove` from the underlying collection (use `Iterator.remove()` for that — Module 13).

```java
int[] xs = {1, 2, 3};
for (int x : xs) System.out.println(x);

List<String> names = List.of("a", "b", "c");
for (var n : names) System.out.println(n);
```

### 2.4 `while` and `do`/`while`

```java
while (cond) { ... }

do { ... } while (cond);
```

### 2.5 `break` and `continue`

`break;` exits the innermost loop or `switch`. `continue;` skips to the next iteration.

Java has **labeled break/continue**:

```java
outer:
for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        if (matrix[i][j] < 0) break outer;   // exits both loops
        if (matrix[i][j] == 0) continue outer; // continues the outer loop
    }
}
```

Useful for nested loops; rare in well-factored code.

### 2.6 `switch`

Two forms — both alive and well in modern Java.

**Classic** (statement, with fall-through):

```java
switch (day) {
    case MONDAY:
    case FRIDAY:
        System.out.println("ok");
        break;
    case TUESDAY:
        System.out.println("meh");
        break;
    default:
        System.out.println("other");
}
```

The `:` form has fall-through. **Forgetting `break` is a classic bug.**

**Arrow form** (Java 14+, no fall-through):

```java
switch (day) {
    case MONDAY, FRIDAY -> System.out.println("ok");
    case TUESDAY        -> System.out.println("meh");
    default             -> System.out.println("other");
}
```

`switch` can also be an **expression** that produces a value:

```java
String label = switch (day) {
    case MONDAY, FRIDAY -> "ok";
    case TUESDAY        -> "meh";
    default             -> "other";
};
```

Switch supports: `int` and smaller, `char`, enums, `String`, and (Java 21) any reference type with **patterns**:

```java
Object o = ...;
String desc = switch (o) {
    case Integer i when i > 0 -> "positive int " + i;
    case Integer i             -> "non-positive int";
    case String s              -> "string " + s;
    case null                  -> "null";
    default                    -> "other";
};
```

(Pattern-matching switch is in Module 22.)

### 2.7 `try` / `catch` / `finally` / `try-with-resources`

Full treatment in Module 14. Briefly:

```java
try {
    risky();
} catch (IOException e) {
    log(e);
} finally {
    cleanup();   // always runs
}

try (var in = Files.newBufferedReader(path)) {
    // in.close() called automatically at end of block, even on exception
}
```

### 2.8 No `goto`

`goto` is a reserved word but unused. Labeled break/continue cover the legitimate uses.

---

## 3. Methods

### 3.1 The shape

```java
[modifiers] returnType name(parameters) [throws ...] {
    body
}
```

Examples:
```java
public static int add(int a, int b) {
    return a + b;
}

public String repeat(String s, int n) {
    var sb = new StringBuilder();
    for (int i = 0; i < n; i++) sb.append(s);
    return sb.toString();
}

public void log(String message) {
    System.out.println(message);
}
```

`void` means "returns nothing". A `void` method may use `return;` (no value) to exit early.

### 3.2 Calling

Inside the same class:
```java
add(1, 2);             // implicit `this.` for instance methods, or class for static
```

On another instance / class:
```java
instance.method(args);
ClassName.staticMethod(args);
```

You cannot call an instance method without an instance. You **cannot** call a static method through an instance reference cleanly — well, you can, but it's misleading; always prefer `ClassName.staticMethod()`.

### 3.3 Pass-by-value, always — the rule that matters

Java is **always pass-by-value**. There is no pass-by-reference.

This sounds wrong if you've heard "objects are passed by reference in Java" — that phrase is shorthand and misleading. The rule is precise:

- For a primitive parameter, the value is copied. Reassigning the parameter inside the method has no effect on the caller.
- For a reference parameter, the *reference* is copied — both caller and callee now hold pointers to the same object. The callee can mutate the shared object (visible to the caller). The callee cannot rebind the caller's variable.

```java
class Box { int n; }

static void grow(Box b)        { b.n += 1; }      // mutates the shared object
static void rebind(Box b)      { b = new Box(); } // rebinds the LOCAL parameter only
static void incrementInt(int x){ x = x + 1; }     // operates on a copy

public static void main(String[] args) {
    var box = new Box();
    grow(box);     System.out.println(box.n);     // 1 (mutation visible)
    rebind(box);   System.out.println(box.n);     // 1 (rebind invisible)
    int n = 0;
    incrementInt(n); System.out.println(n);       // 0 (primitive copy)
}
```

Internalize this. Half of the "why didn't my change stick?" questions on Stack Overflow trace back to it.

### 3.4 Overloading

Multiple methods with the same name but different *parameter lists* (number, types, or order) — resolved at compile time by the compiler picking the best match.

```java
public static int    add(int a, int b)       { return a + b; }
public static long   add(long a, long b)     { return a + b; }
public static double add(double a, double b) { return a + b; }
public static int    add(int a, int b, int c){ return a + b + c; }
```

Differing only in **return type** is **not** overloading — it doesn't compile. The signature is name + parameter types.

### 3.5 No default parameters

Java has no `void f(int n = 0)` syntax. Two idioms:

(1) Overloading:
```java
public void log(String msg)             { log(msg, "INFO"); }
public void log(String msg, String lvl) { ... }
```

(2) Builders or option objects (used heavily by libraries with many parameters; Module 27).

### 3.6 Varargs

The last parameter can be `T...` to accept any number of `T` arguments. The method receives them as a `T[]`.

```java
public static int sum(int... ns) {
    int s = 0;
    for (int n : ns) s += n;
    return s;
}

sum();              // 0
sum(1, 2, 3);       // 6
sum(new int[]{1,2,3});  // 6 — you may pass an array directly
```

Only one varargs parameter, and only at the end. `String.format(String fmt, Object... args)` is a famous example.

### 3.7 Recursion

Plain recursion works; the JVM does **not** optimize tail calls. Deep recursion hits the per-thread stack limit and throws `StackOverflowError`. Default stack size is around 512 KB — depths of 10 000–50 000 frames depending on local-variable footprint.

For deep recursion, rewrite iteratively or use an explicit `Deque` as a stack.

### 3.8 `throws` clause

A method declares the *checked* exceptions it may throw:

```java
public String read(Path p) throws IOException { ... }
```

Module 14 explains in detail. For now: if you call a method that `throws X`, you must either catch `X` or declare `throws X` yourself.

---

## 4. The `main` method, more carefully

```java
public static void main(String[] args) { ... }
```

- `public` — JVM (outside this class) calls it.
- `static` — JVM doesn't have a `Hello` instance, so the method must belong to the class.
- `void` — returns nothing. The exit code is set via `System.exit(n)` (default 0).
- `args` — command-line args; never `null`, an empty array if no args.

You may declare `main` `throws Throwable` if you want to propagate everything; the JVM will catch and print a stack trace.

```java
public static void main(String[] args) throws Exception { ... }
```

You'll see this in small programs and tests where you don't want to clutter `main` with try/catch.

---

## 5. Building habits

A few stylistic rules that will save you grief:

- **Always use braces** around `if`/`for`/`while` bodies, even single statements.
- **Use `&&`/`||` for booleans** unless you have a clear reason for `&`/`|`.
- **Don't reuse loop counters as flags.** `for (int i = 0; i < n; i++)` and use a different name for any "found" sentinel.
- **Prefer for-each** for traversal, indexed `for` only when you need the index.
- **Use `var` for obvious local types**, full type names where it documents intent (parameters, fields, return types — `var` isn't allowed there anyway).

---

## 6. A worked example

A small command-line program: print FizzBuzz for `n = Integer.parseInt(args[0])`, with a method per concern.

```java
public class FizzBuzz {

    public static void main(String[] args) {
        int n = parseN(args);
        for (int i = 1; i <= n; i++) {
            System.out.println(label(i));
        }
    }

    private static int parseN(String[] args) {
        if (args.length == 0) return 15;
        return Integer.parseInt(args[0]);
    }

    private static String label(int i) {
        boolean three = i % 3 == 0;
        boolean five  = i % 5 == 0;
        if (three && five) return "FizzBuzz";
        if (three)         return "Fizz";
        if (five)          return "Buzz";
        return Integer.toString(i);
    }
}
```

`java FizzBuzz.java` → 1..15 with Fizz/Buzz in the right places.

This program already shows: `static` methods, parameter passing (primitive `int`, reference `String[]`), method overloading (we used the implicit overloads of `Integer.parseInt`), early `return`, and `boolean` short-circuit evaluation.

---

## 7. Try this

1. Implement `boolean isPrime(int n)` iteratively with the only-check-up-to-`sqrt(n)` optimization. Be careful with `n < 2`.
2. Implement `int[] primesUpTo(int n)` returning all primes ≤ n. Reuse `isPrime` first; then rewrite using the Sieve of Eratosthenes.
3. Predict and explain:
   ```java
   void mutate(int[] arr) { arr[0] = 99; arr = new int[]{1,2,3}; }
   ...
   int[] xs = {0};
   mutate(xs);
   System.out.println(xs[0]);          // ?
   System.out.println(xs.length);      // ?
   ```
4. Convert this `if`-chain to an arrow `switch` expression:
   ```java
   String mood;
   if (h < 6)        mood = "asleep";
   else if (h < 12)  mood = "morning";
   else if (h < 18)  mood = "afternoon";
   else              mood = "evening";
   ```

---

**Next:** [Module 05 — Strings and arrays](./05-strings-and-arrays.md)
