# Module 13 — Collections Framework

> Goal: choose the right data structure and use it correctly. The Java Collections Framework is enormous; you don't need every class memorized — you need the *hierarchy*, the *complexity guarantees*, and the *idioms*. After this module you can pick the right collection by instinct.

---

## 1. The hierarchy

The framework divides into four major branches:

```
Iterable
   │
   └─ Collection
        ├─ List          (ordered, indexable, allows duplicates)
        │     ArrayList, LinkedList, CopyOnWriteArrayList
        │
        ├─ Set           (no duplicates)
        │     HashSet, LinkedHashSet, TreeSet, EnumSet
        │
        └─ Queue / Deque (insertion/removal at ends)
              ArrayDeque, PriorityQueue, LinkedList,
              ConcurrentLinkedDeque, BlockingQueue impls

Map (separate hierarchy — does NOT extend Collection)
   ├─ HashMap, LinkedHashMap, TreeMap, EnumMap
   ├─ ConcurrentHashMap
   └─ WeakHashMap, IdentityHashMap, etc.
```

Two facts to internalize first:
- `Map` is **not** a `Collection`. It pairs keys with values. You get `map.keySet()`, `map.values()`, `map.entrySet()` to get views as `Set`/`Collection`/`Set<Entry>`.
- `Iterable` is the most generic abstraction — anything you can write `for (T x : ...)` over.

---

## 2. `List` — ordered, indexable

Two main implementations.

### 2.1 `ArrayList` — the default

Backed by a resizable array. Operations:

| Operation | Time |
|---|---|
| `get(i)`, `set(i, v)` | O(1) |
| `add(v)` (at end) | amortized O(1) |
| `add(0, v)` (front) | O(n) |
| `remove(i)` | O(n) |
| `contains(v)` | O(n) |
| `size()` | O(1) |

**Use `ArrayList` for almost every list-shaped need.** If you don't have a strong reason to use something else, `ArrayList` is right.

```java
List<String> names = new ArrayList<>();
names.add("Alice");
names.add("Bob");
names.get(0);           // "Alice"
names.size();           // 2
names.remove(0);        // "Alice"

for (String n : names) { ... }
names.forEach(System.out::println);
```

### 2.2 `LinkedList`

Doubly-linked list. Operations:

| Operation | Time |
|---|---|
| `get(i)` | O(n) — must walk |
| `add(0, v)`, `removeFirst` | O(1) |
| `addLast`, `removeLast` | O(1) |

**Almost never use `LinkedList`.** If you need a queue/deque, prefer `ArrayDeque` — same O(1) at both ends, far better cache behavior. The intuition that linked lists are good for "fast inserts" rarely beats `ArrayList` in practice on modern CPUs.

### 2.3 `List.of(...)` and immutables

```java
List<String> a = List.of("a", "b", "c");      // immutable, fixed-size, no nulls
a.add("d");                                    // UnsupportedOperationException
```

Use `List.of` for constants and short literals. For a mutable copy: `new ArrayList<>(List.of(...))`.

---

## 3. `Set` — no duplicates

### 3.1 `HashSet` — the default

Backed by a `HashMap`. O(1) average for `add`, `remove`, `contains`. Iteration order is unspecified (and may change between JVM runs).

```java
Set<String> seen = new HashSet<>();
seen.add("a"); seen.add("b"); seen.add("a");
seen.size();                // 2
seen.contains("a");         // true
```

The element type **must** have correct `equals` and `hashCode`. (Module 11.) Records and primitives wrappers do; your custom classes might not.

### 3.2 `LinkedHashSet` — insertion-order iteration

Adds a doubly-linked list on top of `HashSet`. Same O(1) operations, iteration follows insertion order. Use it when you need uniqueness *and* predictable iteration order.

### 3.3 `TreeSet` — sorted

Red-black tree (a balanced BST). O(log n) operations. Iteration in *natural order* (or a `Comparator` you supply). Useful for "give me things in order" or "next element ≥ x":

```java
NavigableSet<Integer> ts = new TreeSet<>(List.of(3, 1, 4, 1, 5, 9, 2, 6));
ts.first();          // 1
ts.last();           // 9
ts.higher(4);        // 5  — strictly greater
ts.ceiling(4);       // 4  — ≥
ts.floor(4);         // 4  — ≤
ts.lower(4);         // 3  — strictly less
```

### 3.4 `EnumSet` — for enum keys

A bitset under the hood. Tiny memory footprint, very fast.

```java
EnumSet<Status> open = EnumSet.of(Status.NEW, Status.IN_PROGRESS);
EnumSet<Status> all  = EnumSet.allOf(Status.class);
```

Use whenever the element type is an enum.

---

## 4. `Map` — key/value pairs

The most important data structure. Used everywhere.

### 4.1 `HashMap` — the default

Backed by a hash table. O(1) average for `get`, `put`, `remove`. Keys can be any object with proper `equals`/`hashCode`. Allows one `null` key and any number of `null` values.

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Alice", 30);
ages.put("Bob", 25);

ages.get("Alice");                  // 30
ages.getOrDefault("Carol", -1);     // -1

ages.containsKey("Alice");          // true
ages.containsValue(30);             // true (O(n))

ages.remove("Bob");

for (var entry : ages.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}

ages.forEach((k, v) -> System.out.println(k + "=" + v));
```

### 4.2 Useful `Map` methods (Java 8+)

```java
ages.putIfAbsent("Alice", 99);        // only puts if key absent
ages.computeIfAbsent("Carol", k -> 0); // only computes if key absent — common idiom
ages.computeIfPresent("Alice", (k, v) -> v + 1);
ages.compute("Alice", (k, v) -> (v == null ? 0 : v) + 1);
ages.merge("Alice", 1, Integer::sum); // ages["Alice"] = ages["Alice"] + 1, default 0
```

The `computeIfAbsent` pattern is everywhere when building inverted-index-style structures:
```java
Map<String, List<String>> byCity = new HashMap<>();
for (User u : users) {
    byCity.computeIfAbsent(u.city(), k -> new ArrayList<>()).add(u.name());
}
```

### 4.3 `LinkedHashMap` — insertion-order

Like `HashMap` but iterates in insertion order. Constructor option for *access-order* iteration — basis of a simple LRU cache:
```java
new LinkedHashMap<K, V>(capacity, 0.75f, true /* accessOrder */) {
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
};
```

### 4.4 `TreeMap` — sorted by key

Red-black tree. O(log n). Iterates by key in natural / `Comparator` order. Same navigation methods as `TreeSet` (`firstKey`, `lastKey`, `floorEntry`, etc.).

### 4.5 `EnumMap` — for enum keys

Backed by an array of size `enum.values().length`. Very fast, tiny memory.

### 4.6 Immutable `Map.of`

```java
Map.of("a", 1, "b", 2);                       // up to 10 entries
Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
);
```

---

## 5. `Queue` and `Deque`

A `Queue` adds at one end, removes at the other. A `Deque` ("deck") allows both ends.

### 5.1 `ArrayDeque` — the default

Resizable circular array. O(1) at both ends. Use it as either a queue or a stack:

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);          // adds at the head
stack.push(2);
stack.peek();           // 2
stack.pop();            // 2

Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1);         // adds at the tail
queue.offer(2);
queue.poll();           // 1 (head)
```

**Don't use `java.util.Stack`** — it's a legacy class extending `Vector`, synchronized, slow. `ArrayDeque` is what you want.

**Don't use `LinkedList` as a queue** unless you need to mutate the *middle* with an `Iterator` (rare). `ArrayDeque` outperforms it.

### 5.2 `PriorityQueue`

A binary heap. O(log n) insert, O(log n) remove-min, O(1) peek. Useful for top-K, scheduling, Dijkstra's, etc.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1); pq.offer(2);
pq.poll();      // 1
pq.poll();      // 2
```

By default, smallest-first (natural order). Pass a `Comparator` for max-heap or custom orderings:
```java
var maxHeap = new PriorityQueue<Integer>(Comparator.reverseOrder());
```

### 5.3 Concurrent / blocking queues

`BlockingQueue<E>` — adds blocking semantics for producer/consumer:
- `LinkedBlockingQueue`, `ArrayBlockingQueue` — bounded/unbounded queues with `put`/`take` that block.
- `SynchronousQueue` — zero-capacity, hand-off.

Used heavily with `ExecutorService`. Module 19.

---

## 6. Iteration

Several ways to walk a collection:

```java
List<String> xs = List.of("a", "b", "c");

// Enhanced for — most common
for (String x : xs) System.out.println(x);

// Indexed (for List)
for (int i = 0; i < xs.size(); i++) System.out.println(xs.get(i));

// Iterator — needed if you want to remove during iteration
Iterator<String> it = xs.iterator();
while (it.hasNext()) {
    String x = it.next();
    if (x.equals("b")) it.remove();        // OK — only via Iterator.remove
}

// forEach with lambda
xs.forEach(System.out::println);

// Stream — Module 16
xs.stream().filter(x -> x.length() > 0).forEach(...);
```

**Modifying a collection during a for-each loop throws `ConcurrentModificationException`.** Either use `Iterator.remove`, or build a separate "to remove" list and `removeAll` after the loop, or use a `removeIf` predicate:
```java
xs.removeIf(x -> x.startsWith("a"));
```

---

## 7. Sorting

```java
List<Integer> xs = new ArrayList<>(List.of(3, 1, 4, 1, 5, 9, 2, 6));

Collections.sort(xs);                          // mutates xs, ascending
xs.sort(Comparator.reverseOrder());             // descending

// sort by a derived key
List<Person> people = ...;
people.sort(Comparator.comparing(Person::name));
people.sort(Comparator.comparingInt(Person::age).reversed());
people.sort(Comparator.comparing(Person::lastName).thenComparing(Person::firstName));
```

Sort is **stable** — equal elements preserve their relative order.

For a sorted *copy* (without mutating the original), use a stream:
```java
var sorted = xs.stream().sorted().toList();
```

---

## 8. Choosing the right collection — quick decision tree

- Need a key/value store? → `HashMap`. Need it sorted by key? `TreeMap`. Insertion order? `LinkedHashMap`. Enum keys? `EnumMap`.
- Need a uniqueness check? → `HashSet`. Sorted? `TreeSet`. Insertion order? `LinkedHashSet`. Enum elements? `EnumSet`.
- Need a sequence with random-access? → `ArrayList`.
- Need a queue/stack/deque? → `ArrayDeque`.
- Need a priority queue / top-K? → `PriorityQueue`.
- Concurrent access? → `ConcurrentHashMap`, `CopyOnWriteArrayList`, blocking queues. Module 19.

---

## 9. Conversions

```java
List → Set:        new HashSet<>(list);
Set → List:        new ArrayList<>(set);
Array → List:      Arrays.asList(arr)        (fixed-size, mutating arr/list affects both)
Array → mutable:   new ArrayList<>(Arrays.asList(arr));
List → Array:      list.toArray(new String[0]);    // canonical idiom
Map → List<K>:     new ArrayList<>(map.keySet());
Map → List<V>:     new ArrayList<>(map.values());
List<int[]>→int[]: list.stream().flatMapToInt(Arrays::stream).toArray();
```

`Arrays.asList(1, 2, 3)` returns a fixed-size list backed by the original array. You can `set` but not `add`/`remove`. For a mutable list use `new ArrayList<>(Arrays.asList(...))` or `Stream.of(...).collect(Collectors.toList())`.

---

## 10. Unmodifiable views vs immutable copies

```java
List<Integer> xs = new ArrayList<>();
xs.add(1);

List<Integer> view = Collections.unmodifiableList(xs);
view.add(2);       // UnsupportedOperationException
xs.add(2);         // OK — `view` reflects this change

List<Integer> copy = List.copyOf(xs);   // immutable snapshot
xs.add(3);
// copy still has [1, 2]
```

`Collections.unmodifiableX(...)` wraps a *view* — readers can't mutate, but the underlying collection still can. `List.copyOf` / `Set.copyOf` / `Map.copyOf` make defensive snapshots.

---

## 11. `Map`, `entrySet`, common idioms

Counting:
```java
Map<String, Integer> counts = new HashMap<>();
for (String w : words) counts.merge(w, 1, Integer::sum);
```

Grouping:
```java
Map<String, List<User>> byCity = new HashMap<>();
for (User u : users) byCity.computeIfAbsent(u.city(), k -> new ArrayList<>()).add(u);
```

(Module 16's stream `Collectors.groupingBy` is even cleaner.)

Iterating with both key and value:
```java
for (var e : map.entrySet()) {
    K k = e.getKey();
    V v = e.getValue();
}

map.forEach((k, v) -> { ... });
```

---

## 12. Equality and hashing — your responsibility

A `HashMap`/`HashSet` lookup uses `hashCode` to find a bucket and `equals` to confirm. If your key class has wrong/missing `equals`/`hashCode`, lookups silently fail.

```java
class BadKey {
    int n;
    BadKey(int n) { this.n = n; }
    // forgot equals + hashCode
}

var map = new HashMap<BadKey, String>();
map.put(new BadKey(1), "a");
map.get(new BadKey(1));       // null! — different objects, default equals is identity
```

Fix: make `BadKey` a record, or override `equals`/`hashCode`. (Module 11.)

Records are *especially* nice as map keys for this reason.

---

## 13. A working example

```java
import java.util.*;

public class WordCount {
    public static Map<String, Long> count(List<String> words) {
        Map<String, Long> counts = new HashMap<>();
        for (String w : words) {
            counts.merge(w, 1L, Long::sum);
        }
        return counts;
    }

    public static List<Map.Entry<String, Long>> topK(Map<String, Long> counts, int k) {
        var pq = new PriorityQueue<Map.Entry<String, Long>>(
                Comparator.comparingLong(Map.Entry::getValue));
        for (var e : counts.entrySet()) {
            pq.offer(e);
            if (pq.size() > k) pq.poll();        // keep only top k
        }
        var out = new ArrayList<>(pq);
        out.sort(Comparator.comparingLong(Map.Entry<String, Long>::getValue).reversed());
        return out;
    }

    public static void main(String[] args) {
        var words = List.of("a", "b", "a", "c", "b", "a");
        var counts = count(words);
        System.out.println(counts);                   // {a=3, b=2, c=1}
        System.out.println(topK(counts, 2));           // [a=3, b=2]
    }
}
```

This combines `HashMap`, `PriorityQueue`, `Comparator`, and `merge` — the everyday toolset.

---

## 14. Try this

1. Write `Map<Character, Integer> charFreq(String s)` using `merge`.
2. Implement an LRU cache in 10 lines using `LinkedHashMap` access-order with `removeEldestEntry`.
3. Why does this fail at `get` time?
   ```java
   record Key(int x, int[] arr) {}
   var m = new HashMap<Key, String>();
   m.put(new Key(1, new int[]{1,2}), "a");
   m.get(new Key(1, new int[]{1,2}));    // null — why?
   ```
   Hint: array `equals`/`hashCode` is identity-based. Records use the field's `equals`/`hashCode`. Replace `int[]` with `List<Integer>` (or write custom logic).
4. You have a `List<Order>` and need orders grouped by status. Do it with a plain loop using `computeIfAbsent`. Then re-do it with `Collectors.groupingBy` after Module 16.

---

**Next:** [Module 14 — Exceptions](./14-exceptions.md)
