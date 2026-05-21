# Module 15 — Lambdas and Functional Interfaces

> Goal: read and write Java's lambda syntax fluently, understand what a lambda compiles into, and know the standard functional interfaces (`Function`, `Predicate`, `Consumer`, `Supplier`) that show up everywhere in modern code.

---

## 1. The problem lambdas solve

Pre-Java 8, passing behavior around required an anonymous class:
```java
button.addActionListener(new ActionListener() {
    @Override public void actionPerformed(ActionEvent e) {
        System.out.println("clicked");
    }
});
```

Five lines of boilerplate to express "do this on click". Lambdas (Java 8+) condense that to:
```java
button.addActionListener(e -> System.out.println("clicked"));
```

Same semantics, much less ceremony.

---

## 2. Lambda syntax

```java
(parameters) -> expression
(parameters) -> { statements; }
```

Every example:

```java
() -> 42                                  // no params, returns 42
x -> x * 2                                // one param, no parens needed
(x) -> x * 2                              // parens optional but allowed
(int x) -> x * 2                          // explicit type — needed only sometimes
(x, y) -> x + y                           // multiple params, parens required
(x, y) -> {
    int s = x + y;
    return s;
}                                         // block body — return required for non-void
() -> { System.out.println("hi"); }       // void body
```

Rules:
- A single inferred-type parameter may omit parentheses: `x -> x * 2`.
- Multiple parameters or any explicit type require parentheses.
- A block `{ ... }` must use `return` for non-void results.
- An expression body (no braces) implicitly returns its value.
- Type annotations and `final` work on parameters when in parentheses: `(final @NonNull String s) -> ...`.

---

## 3. Functional interfaces

A **functional interface** is an interface with exactly **one abstract method** (SAM — single abstract method). A lambda creates an instance of one.

```java
@FunctionalInterface
public interface Mapper<T, R> {
    R map(T t);
}

Mapper<String, Integer> length = s -> s.length();
length.map("hello");          // 5
```

The `@FunctionalInterface` annotation is optional but recommended — the compiler then errors if you accidentally add a second abstract method.

The interface can have any number of `default` and `static` methods — only the *abstract* count matters. `equals`, `hashCode`, `toString` (inherited from `Object`) don't count either.

### 3.1 Built-in functional interfaces

`java.util.function` defines the canonical set:

| Interface | Method | Use |
|---|---|---|
| `Function<T, R>` | `R apply(T t)` | T → R transform |
| `BiFunction<T, U, R>` | `R apply(T, U)` | (T, U) → R |
| `Predicate<T>` | `boolean test(T t)` | filter / boolean test |
| `BiPredicate<T, U>` | `boolean test(T, U)` | two-arg test |
| `Consumer<T>` | `void accept(T t)` | side-effect |
| `BiConsumer<T, U>` | `void accept(T, U)` | two-arg side-effect |
| `Supplier<T>` | `T get()` | factory / lazy value |
| `UnaryOperator<T>` | `T apply(T t)` | T → T |
| `BinaryOperator<T>` | `T apply(T, T)` | (T, T) → T (reduce) |

Plus primitive-specialized variants to avoid boxing — `IntFunction<R>`, `ToIntFunction<T>`, `IntPredicate`, `IntConsumer`, `IntSupplier`, etc. Used in streams (Module 16) for performance.

```java
Function<String, Integer> length = String::length;
Predicate<String> nonEmpty = s -> !s.isEmpty();
Consumer<String> print = System.out::println;
Supplier<List<Integer>> emptyList = ArrayList::new;
UnaryOperator<Integer> inc = n -> n + 1;
BinaryOperator<Integer> sum = Integer::sum;
```

---

## 4. Method references

Shorthand for lambdas that just delegate to an existing method.

```java
// Lambda                         // Method reference
s -> s.length()                   String::length              // unbound instance method
() -> new ArrayList<>()           ArrayList::new              // constructor
() -> System.out.println("hi")    // — (no equivalent — needs the arg)
x -> System.out.println(x)        System.out::println         // bound to System.out
(a, b) -> a.compareTo(b)          String::compareTo           // unbound
```

Four kinds:
- **`ClassName::staticMethod`** — `Integer::parseInt`. Treats the static method as a function.
- **`Object::instanceMethod`** — bound: the captured object is the receiver. `printer::print`.
- **`ClassName::instanceMethod`** — unbound: the first lambda parameter is the receiver. `String::length` is `s -> s.length()`.
- **`ClassName::new`** — constructor reference. `ArrayList::new` is `() -> new ArrayList<>()`.

Method references read better than lambdas when they're available. Both compile to the same thing.

---

## 5. Capturing — what a lambda can see

A lambda sees:
- its own parameters and local variables,
- the **effectively final** local variables of the enclosing scope,
- the enclosing instance's fields and methods (via implicit `this`).

"Effectively final" means the local was never reassigned after its first assignment — even without the `final` keyword. Once the compiler can prove that, capture is allowed.

```java
String prefix = "x:";          // not declared final, but never reassigned
List.of("a","b").forEach(s -> System.out.println(prefix + s));   // OK

int counter = 0;
List.of("a","b").forEach(s -> counter++);    // ERROR — counter would need to be assigned
```

Why effectively final? Lambdas may run on a different thread or after the enclosing scope has returned. Mutable locals would have ambiguous semantics. If you need to share mutable state, use an `AtomicInteger` or a one-element array as a workaround:

```java
var counter = new int[1];
list.forEach(s -> counter[0]++);
```

(But really — usually you should be using `Stream.count()` or `reduce`. Module 16.)

`this` inside a lambda refers to the enclosing instance — different from anonymous classes, where `this` was the anonymous class itself.

---

## 6. What does a lambda compile to?

The compiler does **not** create an anonymous class file at compile time. Instead, the bytecode contains an `invokedynamic` instruction (Module 02). The first time it's hit at runtime, a *bootstrap method* (`LambdaMetafactory.metafactory`) builds a synthetic class implementing the target interface, with one method whose body is the lambda. Subsequent calls reuse the cached implementation.

Two practical consequences:
- **Lambdas with no captured state are reused** — the JVM caches a single instance.
- **Capturing lambdas allocate** — a new closure object each time the enclosing code runs and captures fresh values. In tight loops, this matters. Prefer non-capturing lambdas when possible (`String::length` vs `s -> some + s.length()`).

---

## 7. Lambdas vs anonymous classes

| | Lambda | Anonymous class |
|---|---|---|
| Syntax | terse | verbose |
| Allowed for any interface | only single-abstract-method | any |
| `this` refers to | enclosing instance | the anonymous instance |
| Multiple methods | no | yes |
| Allocate per use | depends on capture | yes |

Use lambdas for SAM cases. Anonymous classes for the rare multi-method or stateful cases.

---

## 8. Composition

Most functional interfaces have `default` methods to compose:

```java
Predicate<String> nonEmpty = s -> !s.isEmpty();
Predicate<String> short_   = s -> s.length() < 5;
Predicate<String> ok       = nonEmpty.and(short_);
Predicate<String> bad      = nonEmpty.negate();

Function<String, Integer> len = String::length;
Function<String, String> half  = s -> s.substring(0, s.length() / 2);
Function<String, Integer> halfLen = half.andThen(len);     // f then g
Function<String, Integer> compose = len.compose(half);     // g of f — same as andThen here

Consumer<String> log = System.out::println;
Consumer<String> double_ = log.andThen(s -> System.out.println("done: " + s));

Comparator<Person> byName = Comparator.comparing(Person::name);
Comparator<Person> byAgeThenName = Comparator.comparingInt(Person::age).thenComparing(byName);
```

Composition is what makes the standard library feel almost like a small DSL.

---

## 9. Common patterns

### 9.1 Replacing `if/else` with maps of lambdas

```java
Map<String, BiFunction<Integer, Integer, Integer>> ops = Map.of(
    "+", Integer::sum,
    "-", (a, b) -> a - b,
    "*", (a, b) -> a * b,
    "/", (a, b) -> a / b
);
int r = ops.get("+").apply(3, 4);    // 7
```

Strategy pattern, in three lines.

### 9.2 Lazy evaluation

```java
public void log(Level level, Supplier<String> message) {
    if (level.ordinal() >= currentLevel.ordinal()) {
        out.println(message.get());     // only build the string if we'll print
    }
}

log(DEBUG, () -> expensiveDescription(obj));
```

The expensive `expensiveDescription` doesn't run unless logging is at DEBUG. Same trick the JDK's `Logger.fine(Supplier<String>)` uses.

### 9.3 Callbacks

```java
public class Cache<K, V> {
    public V getOrCompute(K key, Function<K, V> compute) { ... }
}

cache.getOrCompute(key, k -> repo.load(k));
```

### 9.4 Converting throwing lambdas

`Function<T, R>` can't throw checked exceptions. Wrap or define a throwing variant:

```java
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;
}

public static <T, R> Function<T, R> uncheck(ThrowingFunction<T, R, ? extends Exception> f) {
    return t -> {
        try { return f.apply(t); }
        catch (Exception e) {
            if (e instanceof RuntimeException re) throw re;
            throw new RuntimeException(e);
        }
    };
}

paths.stream().map(uncheck(Files::readString)).toList();
```

Libraries like Vavr, jOOλ, and many in-house utility classes provide this.

---

## 10. A worked example — building a query DSL

```java
import java.util.List;
import java.util.function.Predicate;

public class Query {
    public static <T> List<T> from(List<T> source, Predicate<T> where) {
        var out = new java.util.ArrayList<T>();
        for (T x : source) if (where.test(x)) out.add(x);
        return out;
    }

    public record Person(String name, int age) {}

    public static void main(String[] args) {
        List<Person> people = List.of(
            new Person("Alice", 30),
            new Person("Bob",   17),
            new Person("Carol", 42)
        );

        Predicate<Person> adult     = p -> p.age() >= 18;
        Predicate<Person> startsA   = p -> p.name().startsWith("A");

        List<Person> adultsAndA = from(people, adult.and(startsA));
        adultsAndA.forEach(System.out::println);
    }
}
```

A handful of lambdas + composition gives you a tiny query builder.

---

## 11. Try this

1. Write `<T> Predicate<T> negate(Predicate<T> p)`. Why is `Predicate.negate()` a `default` method on the interface itself, not a static helper?
2. Build a `Map<String, Function<Integer, Integer>>` of named transformations (`"square"`, `"double"`, `"absolute"`). Apply them to a list of integers.
3. Why does this not compile?
   ```java
   int total = 0;
   List.of(1,2,3).forEach(x -> total += x);
   ```
   Replace with `mapToInt + sum` from streams (M16) — but try the workaround with `int[] total = {0}` first.
4. Convert these anonymous classes to lambdas:
   ```java
   Comparator<String> byLen = new Comparator<>() {
       @Override public int compare(String a, String b) { return a.length() - b.length(); }
   };
   Runnable r = new Runnable() {
       @Override public void run() { System.out.println("hi"); }
   };
   ```

---

**Next:** [Module 16 — Streams API and `Optional`](./16-streams-optional.md)
