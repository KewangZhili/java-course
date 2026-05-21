# Module 16 — Streams API and `Optional`

> Goal: process collections declaratively. Read any `.stream().filter(...).map(...).collect(...)` chain. Use `Optional` correctly. Know when a parallel stream helps and when it hurts.

---

## 1. What a stream is

A **stream** is a sequence of elements that supports composable operations. It is **not** a data structure. Key properties:
- A stream pulls from a source (a collection, an array, a generator, a file).
- Operations are **lazy**: nothing happens until a *terminal* operation runs.
- Streams are **single-use**: once consumed, you can't reuse them.
- Operations come in two flavors:
  - **Intermediate** — transform a stream into another stream. Returns `Stream<...>`. Lazy.
  - **Terminal** — produce a result (a value, a collection, or a side effect). Triggers execution.

Quick example:
```java
List<String> names = List.of("Alice", "Bob", "Carol", "Dave");

long count = names.stream()
                  .filter(n -> n.length() <= 4)   // intermediate
                  .map(String::toUpperCase)       // intermediate
                  .count();                       // terminal
// count = 2 (BOB, DAVE)
```

Reads like a description of *what* you want, not *how* to compute it. The JVM is free to fuse operations and skip unnecessary work.

---

## 2. Creating a stream

```java
list.stream();                     // from a Collection
Arrays.stream(arr);                // from an array
Stream.of("a", "b", "c");          // from values
Stream.empty();
Stream.generate(() -> 0)           // infinite — must limit()
      .limit(10);
Stream.iterate(0, n -> n + 1)      // infinite, like a fold
      .limit(10);
Files.lines(path);                 // lines of a file (closes when terminal runs!)
"a,b,c".chars();                    // IntStream of code points
IntStream.range(0, 10);            // 0..9
IntStream.rangeClosed(1, 10);      // 1..10
```

For primitive streams (no boxing): `IntStream`, `LongStream`, `DoubleStream`. Convert with `.boxed()` if you need a `Stream<Integer>`.

---

## 3. Intermediate operations

Each returns a new stream.

```java
.filter(Predicate<T>)              // keep elements that pass
.map(Function<T, R>)                // 1-to-1 transform
.flatMap(Function<T, Stream<R>>)    // 1-to-many: flatten nested streams
.distinct()                         // dedup
.sorted()                           // natural order
.sorted(Comparator<T>)              // custom order
.peek(Consumer<T>)                  // side-effect; useful for debugging
.limit(n)                           // first n
.skip(n)                            // skip first n
.takeWhile(Predicate<T>)            // (Java 9+) keep until predicate fails
.dropWhile(Predicate<T>)            // (Java 9+) skip while predicate holds
```

Examples:
```java
// extract all unique non-empty trimmed lines
lines.stream()
     .map(String::strip)
     .filter(s -> !s.isEmpty())
     .distinct()
     .toList();

// flatMap — flatten lists of lists
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4));
List<Integer> flat = nested.stream().flatMap(List::stream).toList();   // [1,2,3,4]
```

`flatMap` is the most-misunderstood operation. It maps each element to a *stream*, then concatenates. Use it whenever you have nested structures: parsing each line into multiple tokens, expanding each parent into its children.

---

## 4. Terminal operations

These trigger the pipeline and produce a result.

### 4.1 Reductions

```java
.count()                                          // long
.findFirst()                                      // Optional<T>
.findAny()                                        // Optional<T> — order-irrelevant; matters for parallel
.anyMatch(p)  .allMatch(p)  .noneMatch(p)         // boolean
.min(Comparator<T>)  .max(Comparator<T>)          // Optional<T>
.reduce(identity, accumulator)                    // T
.reduce(BinaryOperator)                           // Optional<T>
```

Examples:
```java
boolean hasNeg = ints.stream().anyMatch(n -> n < 0);

int sum = ints.stream().reduce(0, Integer::sum);
int product = ints.stream().reduce(1, (a, b) -> a * b);

Optional<Integer> max = ints.stream().max(Comparator.naturalOrder());
```

### 4.2 To collections / arrays

```java
.toList()                          // Java 16+, returns immutable list
.toArray()                         // Object[]
.toArray(String[]::new)            // typed
.collect(Collectors.toList())      // mutable list (the older idiom)
.collect(Collectors.toSet())
.collect(Collectors.toMap(keyFn, valueFn))
```

### 4.3 `forEach` / `forEachOrdered`

```java
list.stream().forEach(System.out::println);      // unordered (parallel may interleave)
list.stream().forEachOrdered(System.out::println); // preserves encounter order
```

Use `forEach` only when you need a side effect that doesn't fit in a collector or reduction — most of the time, `toList` and a regular for-each loop is clearer.

---

## 5. Collectors — the workhorse

`java.util.stream.Collectors` has the most useful collector factories.

### 5.1 To collections

```java
.collect(toList())               // mutable list
.collect(toUnmodifiableList())
.collect(toSet())
.collect(toCollection(TreeSet::new))
```

(Static-import `Collectors.*` for readability.)

### 5.2 Grouping

```java
import static java.util.stream.Collectors.*;

Map<String, List<User>> byCity =
    users.stream().collect(groupingBy(User::city));

Map<String, Long> countByCity =
    users.stream().collect(groupingBy(User::city, counting()));

Map<String, Double> avgAgeByCity =
    users.stream().collect(groupingBy(User::city, averagingInt(User::age)));
```

`groupingBy(keyFn, downstream)` is the pattern: classify by key, then collect into the downstream collector.

### 5.3 Partitioning

```java
Map<Boolean, List<User>> partitioned =
    users.stream().collect(partitioningBy(u -> u.age() >= 18));
// .get(true) → adults; .get(false) → minors
```

### 5.4 Joining strings

```java
String joined = users.stream()
                     .map(User::name)
                     .collect(joining(", ", "[", "]"));
// "[Alice, Bob, Carol]"
```

### 5.5 toMap

```java
Map<Long, User> byId = users.stream().collect(toMap(User::id, u -> u));
// or: toMap(User::id, identity())
```

If keys collide, `toMap` throws `IllegalStateException`. Provide a merge function for collisions:
```java
toMap(User::city, u -> 1, Integer::sum);
```

---

## 6. Primitive streams: `IntStream`, `LongStream`, `DoubleStream`

For numeric work without boxing:

```java
int sum = IntStream.range(1, 101).sum();              // 1..100 sum = 5050
double avg = IntStream.of(1, 2, 3, 4).average().orElse(0);
int max = IntStream.of(1, 2, 3, 4).max().getAsInt();

IntStream.range(0, 10).forEach(System.out::println);
```

Convert:
```java
list.stream().mapToInt(String::length)             // Stream<String> -> IntStream
ints.boxed()                                       // IntStream -> Stream<Integer>
ints.asLongStream() / asDoubleStream();
```

Primitive streams have specialized `sum`, `average`, `min`, `max`, `summaryStatistics`.

---

## 7. Parallel streams — when (rarely) to use

```java
list.parallelStream()
    .filter(...)
    .map(...)
    .reduce(...);
```

Splits work across the **common ForkJoinPool**. Useful for *expensive per-element work on large data sets*, where:
- the operation is **stateless** and **side-effect-free**,
- the elements are independent,
- the data structure splits well (`ArrayList` / arrays good; `LinkedList` / `Stream.iterate` bad).

Pitfalls:
- Common pool is shared across the JVM. A misbehaving parallel stream can starve other parts of your app.
- Order-sensitive operations (`forEachOrdered`, `findFirst`) limit parallelism.
- `forEach` with shared mutable state is a race.
- For most application code, sequential streams are faster *and* simpler. Profile before reaching for parallel.

A reasonable rule: if your sequential stream is fast enough, leave it sequential.

---

## 8. Streams are single-use

```java
var s = list.stream();
s.count();
s.count();   // IllegalStateException — already consumed
```

If you need to rerun a pipeline, regenerate the source: `list.stream()...`.

---

## 9. `Optional<T>`

A container for *either* a value or *no value*. Designed as a return type for methods where "no result" is valid.

```java
public Optional<User> findById(long id) {
    return Optional.ofNullable(map.get(id));
}
```

Construction:
```java
Optional.of(value)            // value must be non-null, else NPE
Optional.ofNullable(maybeNull) // empty if null
Optional.empty()
```

### 9.1 Use

```java
Optional<User> u = repo.findById(42);

if (u.isPresent())   { ... }              // works but discouraged
u.ifPresent(x -> println(x.name()));      // preferred

String name = u.map(User::name).orElse("anonymous");

User user = u.orElseThrow(() -> new NotFoundException(42));

u.map(User::name)
 .filter(n -> n.startsWith("A"))
 .ifPresent(System.out::println);
```

### 9.2 What `Optional` is *for*

- ✓ A method's **return type** when "absent" is a valid outcome (`findById`, `findFirst`).

### 9.3 What `Optional` is **NOT** for

- ✗ Fields. Don't write `private Optional<String> middleName;`. Use a nullable field with a getter that returns `Optional`.
- ✗ Method parameters. `void f(Optional<X> x)` is awkward. Overload, or use `null`.
- ✗ Collections. Use empty collections, not `Optional<List<T>>`.

A common smell:
```java
if (opt.isPresent()) return opt.get();   // anti-pattern
return defaultValue;
```
Use `.orElse(defaultValue)` instead.

`OptionalInt`, `OptionalLong`, `OptionalDouble` exist for primitives.

---

## 10. A realistic example

Group orders by customer, sum their values, return the top 3 customers.

```java
public record Order(String customer, long amountCents) {}

public static List<Map.Entry<String, Long>> topSpenders(List<Order> orders, int k) {
    Map<String, Long> totals = orders.stream()
        .collect(Collectors.groupingBy(
            Order::customer,
            Collectors.summingLong(Order::amountCents)));

    return totals.entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
        .limit(k)
        .toList();
}
```

Compare to the imperative loop equivalent — much shorter, still readable once you've internalized the vocabulary.

---

## 11. Stream gotchas

**Side effects in stream operations.** Don't mutate external collections from `map` / `filter`. Use a collector or `forEach` (last).

**Closing file streams.** `Files.lines(path)` returns a `Stream<String>` that holds an open file. Use it inside `try-with-resources`:
```java
try (var lines = Files.lines(path)) {
    lines.filter(...).forEach(...);
}
```

**Don't call `Stream.collect(Collectors.toList())` and then `.add(x)` to a list you assume is mutable.** It is, but `.toList()` (Java 16) returns immutable. Pick the right one.

**Beware of `peek`.** It's for debugging. Many JVM optimizations skip `peek` if the result isn't observed. Don't rely on it for side effects.

---

## 12. Try this

1. Convert this loop to a stream pipeline:
   ```java
   List<String> result = new ArrayList<>();
   for (String s : input) {
       if (s != null) {
           String t = s.trim().toLowerCase();
           if (!t.isEmpty()) result.add(t);
       }
   }
   ```

2. Given a `List<String>` of words, build a `Map<Integer, List<String>>` grouping words by length, with each list sorted alphabetically. (Hint: `groupingBy(String::length, mapping(identity(), toList()))` then sort, or `groupingBy(String::length, collectingAndThen(toList(), l -> { l.sort(...); return l; }))`.)

3. Find the second-highest distinct number in `int[] xs` using a stream.

4. Why does this print only "1" and not "1\n2\n3"?
   ```java
   var s = Stream.of(1, 2, 3);
   s.findFirst().ifPresent(System.out::println);
   s.forEach(System.out::println);
   ```

---

**Next:** [Module 17 — File I/O and NIO.2](./17-io-nio.md)
