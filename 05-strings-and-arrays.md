# Module 05 — Strings and Arrays

> Goal: handle text and fixed-length sequences correctly. Strings are everywhere in Java code; understanding immutability, the string pool, `StringBuilder`, and the right APIs will make your code shorter and faster. Arrays are the foundation that every collection sits on.

---

## 1. Strings — what they are

`String` is a final class in `java.lang`. A `String` is **immutable**: once created, its characters never change. Methods that look like they modify (`toUpperCase`, `substring`, `replace`) actually return *new* `String` objects.

```java
String s = "hello";
s.toUpperCase();       // returns "HELLO" but throws away the result
System.out.println(s); // still "hello"
s = s.toUpperCase();   // now reassign — s now references "HELLO"
```

A `String` internally holds a byte array (`byte[]`) plus an encoding flag. Pre-Java 9 it was a `char[]` (UTF-16). Since Java 9, *compact strings* — if the contents are pure Latin-1, it's stored as 1 byte per char; otherwise 2 bytes per char. This roughly halves memory for typical English text.

### 1.1 Why immutability

- **Hashable.** A string's hash code can be computed once and cached.
- **Safe to share.** Pass a `String` to a thread or method without copying.
- **Substring sharing was a feature.** (Pre-Java 7 substrings shared the underlying `char[]`. Today they copy, which is safer but means substring is O(n) — see §1.5.)
- **Safe in keys.** A `HashMap<String, ...>`'s keys can't drift after insertion.

The price: any "modification" allocates a new object.

### 1.2 String literals and the string pool

A *literal* (a string written in source: `"hello"`) is stored once, in a JVM-managed pool of interned strings. Any other identical literal in the same program refers to that same object.

```java
String a = "hello";
String b = "hello";
System.out.println(a == b);     // true — both reference the same pool entry

String c = new String("hello"); // explicit new — fresh heap object, NOT in pool
System.out.println(a == c);     // false
System.out.println(a.equals(c));// true
```

`String.intern()` returns the pool's canonical instance:
```java
String d = c.intern();
System.out.println(a == d);     // true
```
You should rarely call `intern()` — the pool is unbounded growth territory if you intern user input. Just don't compare strings with `==`.

### 1.3 Equality on strings

```java
"abc".equals("abc")              // true
"abc".equals("ABC")              // false
"abc".equalsIgnoreCase("ABC")    // true
```

For null safety:
```java
Objects.equals(s1, s2)           // null-safe: two nulls are "equal", null/anything-else is not
```

### 1.4 Common operations

```java
String s = "Hello, World";

s.length();                      // 12 (UTF-16 code units, not Unicode code points!)
s.charAt(0);                     // 'H'
s.codePointAt(0);                // 72

s.indexOf('W');                  // 7
s.indexOf("orl");                // 8
s.indexOf('z');                  // -1 if not found
s.lastIndexOf('o');              // 8
s.contains("World");             // true
s.startsWith("Hello");           // true
s.endsWith("World");             // true

s.substring(7);                  // "World"
s.substring(7, 10);              // "Wor"   (start inclusive, end exclusive)
s.toUpperCase();                 // "HELLO, WORLD"
s.toLowerCase();                 // "hello, world"
s.strip();                       // trims Unicode whitespace (prefer over trim())
s.replace('o', '0');             // "Hell0, W0rld"
s.replace("World", "Java");      // "Hello, Java"

"a,b,,c".split(",");             // {"a","b","","c"}  (regex! see §1.6)
String.join(",", "a", "b", "c"); // "a,b,c"
String.join("-", List.of("a","b","c"));  // "a-b-c"

s.toCharArray();                 // char[]
s.getBytes(StandardCharsets.UTF_8);  // byte[] — always specify the charset
String.valueOf(42);              // "42"   (works for any object — calls toString or "null")
```

`String.format` for sprintf-style:
```java
String.format("x=%d y=%.2f s=%s", 1, 3.14, "hi");   // "x=1 y=3.14 s=hi"
```

Java 15+ — the alternative `formatted` method is more readable:
```java
"x=%d y=%.2f".formatted(1, 3.14);
```

### 1.5 Surrogate pairs (briefly)

`charAt` returns a 16-bit `char`, which is a UTF-16 *code unit*, not a Unicode *code point*. Characters above U+FFFF (most emoji, some CJK) are stored as a *surrogate pair* of two `char`s. Length-counting and indexing operate on code units, not code points.

```java
String s = "a😀b";
s.length();               // 4   (😀 is a surrogate pair: 2 chars)
s.codePointCount(0, s.length()); // 3
```

Use `codePointAt(i)`, `s.codePoints()`, `Character.toChars(cp)` if you need to be Unicode-correct.

### 1.6 `split` and `replaceAll` are regex-based

```java
"a.b.c".split(".");      // {} — '.' is "any char" in regex!
"a.b.c".split("\\.");    // {"a","b","c"}
```

For a literal split with no regex semantics:
```java
"a.b.c".split(java.util.regex.Pattern.quote("."));
```

`replaceAll(regex, repl)` is also regex. `replace(literal, repl)` is plain text. Subtle but bites everyone once.

### 1.7 Text blocks (Java 15+)

For multi-line strings:

```java
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
```

The closing `"""` indentation determines how much leading whitespace is stripped from each line.

---

## 2. `StringBuilder`: the right way to build strings

Concatenating strings in a loop is **O(n²)** because each `+` creates a new String:

```java
String s = "";
for (int i = 0; i < 100_000; i++) {
    s = s + i;       // each iteration: copy entire previous string + append digits
}
```

`StringBuilder` is a mutable buffer. Use it for any non-trivial building:

```java
var sb = new StringBuilder();
for (int i = 0; i < 100_000; i++) {
    sb.append(i);
}
String s = sb.toString();
```

Operations:
```java
sb.append(...);              // any type — number, char, String, char[]
sb.insert(index, ...);
sb.delete(start, end);
sb.deleteCharAt(i);
sb.reverse();
sb.charAt(i);
sb.length();
sb.setLength(n);             // truncate or pad with \0
```

For thread-safe building, `StringBuffer` (synchronized). You almost never need it — building strings is rarely shared across threads. Use `StringBuilder`.

The compiler often *automatically* uses `StringBuilder` for plain `+` chains (`"a" + x + "b" + y`) — that single expression is compiled into a few `append` calls. The `O(n²)` bug appears specifically when you reassign `s = s + ...` in a loop.

---

## 3. Arrays

An array is a fixed-length sequence of values, with O(1) indexed access. It's a **reference type** (so an array variable is a reference to an array on the heap), but its element type can be a primitive or a reference.

### 3.1 Declaration and creation

```java
int[]    nums   = new int[10];           // 10 ints, all 0
int[]    primes = {2, 3, 5, 7, 11};      // array literal — only allowed in declaration
String[] names  = new String[3];         // 3 nulls

// 2D
int[][] grid    = new int[3][4];         // 3 rows of 4 ints, all 0
int[][] jagged  = { {1,2}, {3}, {4,5,6} };  // rows can have different lengths
```

Anywhere an array is *expected*, you can use either form. As a parameter or in non-declaration contexts, use `new int[]{1,2,3}`.

### 3.2 Access and length

```java
nums[0] = 1;
int x = nums[0];
int n = nums.length;     // a *field*, not a method

for (int v : nums) { ... }
for (int i = 0; i < nums.length; i++) { ... }
```

`length` is a field. `String.length()` is a method. They differ. (Yes, this is mildly annoying.)

Out-of-range access throws `ArrayIndexOutOfBoundsException`. Negative index too.

### 3.3 Arrays are *covariant* (and that's a footgun)

```java
Object[] objs = new String[3];   // legal — String[] is a subtype of Object[]
objs[0] = 42;                     // ArrayStoreException at runtime
```

The JVM checks every store into an `Object[]` against the actual element type. Compare to generics (`List<String>` is *not* a subtype of `List<Object>`) — Module 12 will return to this.

### 3.4 The `Arrays` utility class

```java
import java.util.Arrays;

int[] xs = {3, 1, 4, 1, 5, 9, 2, 6};
Arrays.sort(xs);                            // in-place ascending
Arrays.sort(xs, 2, 5);                      // sort range [2,5)
int idx = Arrays.binarySearch(xs, 4);       // requires sorted array

int[] copy   = Arrays.copyOf(xs, xs.length);
int[] slice  = Arrays.copyOfRange(xs, 2, 5);
int[] filled = new int[10]; Arrays.fill(filled, 7);

String s = Arrays.toString(xs);             // "[1, 1, 2, 3, 4, 5, 6, 9]"
String s2 = Arrays.deepToString(grid);      // works for nested arrays

boolean equal = Arrays.equals(a, b);
int hash      = Arrays.hashCode(a);

List<Integer> list = Arrays.stream(xs).boxed().toList();   // int[] -> List<Integer>
```

For an array of objects you can sort with a `Comparator`:
```java
String[] names = {"Bob", "alice", "Carol"};
Arrays.sort(names);                                    // case-sensitive: ["Bob","Carol","alice"]
Arrays.sort(names, String.CASE_INSENSITIVE_ORDER);      // ["alice","Bob","Carol"]
Arrays.sort(names, Comparator.comparingInt(String::length));
```

### 3.5 Arrays vs `List`

Most application code uses `List<T>` (Module 13), not raw arrays. Arrays show up at boundaries — `String[] args`, varargs, byte buffers for I/O, performance-critical inner loops. Internalize:
- arrays: fixed-length, primitive-friendly, covariant, no generics-aware equality
- `List<T>`: dynamic-length, only object elements, invariant, has `equals`/`hashCode` based on contents

Convert easily:
```java
String[] arr = {"a","b","c"};
List<String> list = Arrays.asList(arr);          // fixed-size, backed by arr (mutating arr or list affects both, no add/remove)
List<String> ml   = new ArrayList<>(Arrays.asList(arr));  // mutable copy
List<String> il   = List.of(arr);                // immutable copy (Java 9+)

String[] back     = list.toArray(new String[0]); // canonical idiom
```

### 3.6 Multi-dimensional reality

A `int[][]` is an array of `int[]`. Each row is a separate heap object. There is no true 2D contiguous array in Java; if you need cache-friendly contiguous storage, use a 1D `int[rows*cols]` with manual `i*cols + j` indexing.

---

## 4. Interop: text I/O

Reading a whole file as a string:
```java
import java.nio.file.Files;
import java.nio.file.Path;

String text = Files.readString(Path.of("data.txt"));            // UTF-8 by default since Java 11
List<String> lines = Files.readAllLines(Path.of("data.txt"));   // UTF-8
```

Writing:
```java
Files.writeString(Path.of("out.txt"), "hello\n");
Files.write(Path.of("out.txt"), List.of("a", "b", "c"));
```

Module 17 covers I/O properly.

### 4.1 Default charset gotcha

Pre-Java 18, `new String(bytes)` and `Files.readAllLines(...)` without an explicit charset used the platform default (often UTF-8 on macOS/Linux, but Windows-1252 on some Windows JVMs). **Always pass `StandardCharsets.UTF_8`** unless you have a reason. Since Java 18, the JDK default is UTF-8 on all platforms.

---

## 5. The `String` cookbook (just a few)

Trim and lowercase:
```java
String key = input.strip().toLowerCase(Locale.ROOT);
```
Note: `toLowerCase()` without a `Locale` uses the default locale, which famously breaks Turkish (`"I".toLowerCase()` becomes `"ı"` not `"i"`). For machine identifiers, always use `Locale.ROOT`.

Check empty / blank:
```java
s == null || s.isEmpty()      // null or zero-length
s == null || s.isBlank()      // null or only whitespace (Java 11+)
```

Reverse:
```java
new StringBuilder(s).reverse().toString();
```

Repeat:
```java
"ab".repeat(3);    // "ababab"  — Java 11+
```

Pad:
```java
String.format("%5d", 42);     // "   42"
String.format("%-5s|", "hi"); // "hi   |"
```

Format dollars (don't — use `BigDecimal` and `NumberFormat` for real money):
```java
NumberFormat.getCurrencyInstance(Locale.US).format(1234.5);   // "$1,234.50"
```

Build CSV row:
```java
String row = String.join(",", "a", "b", "c");
```

---

## 6. A short example

Read a number from stdin and print its digits in reverse, separated by spaces:

```java
import java.util.Scanner;

public class Reverse {
    public static void main(String[] args) {
        var sc = new Scanner(System.in);
        long n = Math.abs(sc.nextLong());

        var sb = new StringBuilder();
        if (n == 0) {
            sb.append('0');
        } else {
            while (n > 0) {
                sb.append(n % 10);
                if (n >= 10) sb.append(' ');
                n /= 10;
            }
        }
        System.out.println(sb);
    }
}
```

Things this uses: `Scanner` for input, `Math.abs`, `StringBuilder` for accumulation, primitive `long` arithmetic, `%` for digit extraction.

---

## 7. Try this

1. Without using `String.contains`, write `boolean substring(String haystack, String needle)`. Naive O(n·m) is fine.
2. Implement `String[] reverseWords(String s)` — `"hello world from java"` → `"java from world hello"`. Use `split` and a `StringBuilder` (or `String.join`).
3. Why does this take *minutes* on `n = 1_000_000`?
   ```java
   String s = "";
   for (int i = 0; i < n; i++) s = s + "x";
   ```
   Replace with `StringBuilder` and time both with `System.nanoTime()`.
4. Implement `int[] rotate(int[] xs, int k)` returning a new array rotated right by `k`. Two ways: with `Arrays.copyOfRange` + concatenation, and with a single allocation + index arithmetic. Which is more idiomatic?

---

**Next:** [Module 06 — Classes, objects, constructors](./06-classes-and-objects.md)
