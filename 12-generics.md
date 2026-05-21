# Module 12 — Generics and the Type System

> Goal: read and write any generic signature you'll meet in real code, including bounded wildcards, generic methods, and the rules that fall out of *type erasure*. After this module, signatures like `<T extends Comparable<? super T>>` stop being mysterious.

---

## 1. Why generics

Without them:
```java
List items = new ArrayList();
items.add("hello");
items.add(42);                       // no error
String s = (String) items.get(0);    // explicit cast
String t = (String) items.get(1);    // ClassCastException at runtime
```

With them:
```java
List<String> items = new ArrayList<>();
items.add("hello");
items.add(42);             // compile error
String s = items.get(0);   // no cast
```

Generics give you **compile-time type safety** for parametric polymorphism — same code, many element types — and remove a class of casting bugs.

---

## 2. Generic classes

A class can declare type parameters in angle brackets right after its name:

```java
public class Box<T> {
    private T value;
    public Box(T value) { this.value = value; }
    public T get()          { return value; }
    public void set(T value){ this.value = value; }
}
```

`T` is a placeholder for a type, supplied by the caller:
```java
Box<Integer> a = new Box<>(42);
Box<String>  b = new Box<>("hi");
int n = a.get();
```

The `<>` on the right (the "diamond") means "infer the type arguments from the left side" (Java 7+).

Multiple type parameters: `class Pair<A, B> { ... }`. Conventional names:
- `T` — Type (general)
- `E` — Element (in collections)
- `K`, `V` — Key, Value (in maps)
- `R` — Result
- `N` — Number

These are conventions, not rules.

---

## 3. Generic methods

A method can declare its own type parameters, independent of any enclosing class:

```java
public class Util {
    public static <T> List<T> repeat(T value, int times) {
        var out = new ArrayList<T>();
        for (int i = 0; i < times; i++) out.add(value);
        return out;
    }
}

List<String> hellos = Util.repeat("hi", 3);   // T inferred as String
```

Type parameters appear *before* the return type. The compiler infers them from the arguments at the call site.

You can supply them explicitly if inference fails:
```java
Util.<String>repeat("hi", 3);
```

Rare in practice — IDEs almost always infer.

---

## 4. Bounded type parameters

Constrain `T` to be a subtype of something:

```java
public static <T extends Comparable<T>> T max(List<T> xs) {
    T best = xs.get(0);
    for (T x : xs) if (x.compareTo(best) > 0) best = x;
    return best;
}
```

`T extends Comparable<T>` reads "T must be (or be a subtype of) something that implements `Comparable<T>`". This unlocks `compareTo` on values of `T`.

Multiple bounds:
```java
<T extends Number & Comparable<T>>     // both
```
First bound may be a class; additional bounds must be interfaces.

---

## 5. Wildcards: `?`, `? extends T`, `? super T`

Type parameters and wildcards are different things:
- A type **parameter** (`<T>`) is a name you'll refer to within scope.
- A **wildcard** (`?`) is "some type — I don't need to name it".

You'll see wildcards in *type uses* — usually in parameters of methods that don't introduce their own `<T>`.

```java
List<?>                  // a list of "something"; safe to read as Object, can't add anything (except null)
List<? extends Number>   // a list of Number-or-subtype; safe to read as Number, can't add
List<? super Integer>    // a list of Integer-or-supertype; safe to add Integer, only readable as Object
```

Why this matters: in Java, generics are **invariant** by default — `List<Integer>` is **NOT** a subtype of `List<Number>`. Wildcards re-introduce variance per use site:

```java
List<Integer> ints = List.of(1, 2, 3);
List<Number>  nums = ints;                  // ERROR — invariant
List<? extends Number> any = ints;          // OK
```

### 5.1 PECS — Producer Extends, Consumer Super

The classic mnemonic from *Effective Java*:

- If the parameter **produces** Ts (you read from it), use `<? extends T>`.
- If the parameter **consumes** Ts (you write to it), use `<? super T>`.
- If both, use plain `<T>`.

A method that copies from one list into another:
```java
public static <T> void copy(List<? extends T> src, List<? super T> dst) {
    for (T t : src) dst.add(t);
}

List<Integer> ints = List.of(1, 2, 3);
List<Number>  nums = new ArrayList<>();
copy(ints, nums);       // OK
```

Reading from `List<? extends T>`: you can *pull a T out* (every element is at least a T).

Writing to `List<? extends T>`: you **can't** — `List<? extends Number>` could be a `List<Integer>` or a `List<Double>`; the compiler can't prove that `add(someNumber)` is safe.

Writing to `List<? super T>`: you can — every element slot is willing to accept a T.

Reading from `List<? super T>`: you only get `Object` back.

---

## 6. Type erasure — the secret of Java generics

**At runtime, generic type arguments are erased.** A `List<String>` and a `List<Integer>` are both just `List` to the JVM. The compiler:
1. Type-checks your generic code at compile time.
2. **Erases** type parameters — `T` becomes its leftmost bound (`Object` if unbounded).
3. Inserts casts at use sites.

So `List<String> xs; xs.get(0)` actually compiles to "fetch element 0; cast to String". The cast can fail at runtime if the list was contaminated (e.g. via raw types or reflection) — `ClassCastException`.

### 6.1 Things that don't compile (or don't do what you'd expect) because of erasure

```java
class Foo<T> {
    T t = new T();              // ERROR — can't `new` a type parameter
    T[] arr = new T[10];        // ERROR — can't make a `new T[]`
}
```

```java
void f(Object o) {
    if (o instanceof List<String>) { }   // ERROR — List<String> isn't a runtime type
    if (o instanceof List<?>) { }        // OK — wildcard form is the legitimate runtime check
}
```

```java
static void g(List<String> xs)  {}
static void g(List<Integer> xs) {}    // ERROR — both erase to g(List)
```

You also can't:
- Have a generic exception: `class MyException<T> extends Exception` — illegal.
- Catch a parameterized type: `catch (MyException<String> e)` — illegal.
- Use a type parameter in a `static` field/method of the enclosing class without re-declaring it on the method.

### 6.2 Why erasure?

Backward compatibility. Generics arrived in Java 5 (2004). The team chose not to break the existing classfile format and not to bifurcate the JDK. The cost is the limitations above. Project Valhalla (in progress) aims to add **specialized generics** for primitives and value types. Until that lands, `List<int>` doesn't exist; you use `List<Integer>` or a primitive array.

### 6.3 How to *get* a `T[]` despite erasure

Pass a `Class<T>` token, then use reflection:

```java
import java.lang.reflect.Array;

public static <T> T[] newArray(Class<T> clazz, int length) {
    @SuppressWarnings("unchecked")
    T[] a = (T[]) Array.newInstance(clazz, length);
    return a;
}

String[] arr = newArray(String.class, 10);
```

This is one of the few legitimate uses of reflection. Frameworks (Jackson, Spring, JPA) use `Class<T>` tokens routinely for this reason.

---

## 7. Raw types — don't

A *raw type* is a generic class used without type arguments:
```java
List items = new ArrayList();      // raw — DON'T
items.add("hi");
items.add(42);
```

The compiler issues an "unchecked" warning. You'll see raw types in pre-Java-5 code or framework internals. Any new code should never use them.

If you genuinely don't care about element type, use the unbounded wildcard `List<?>` — you keep type safety (reading gives you `Object`, writing is forbidden except `null`).

---

## 8. `instanceof`, generics, and the `Class<T>` workaround

`instanceof T<...>` against a parameterized type isn't usually allowed:
```java
if (o instanceof List<?> list) { ... }    // wildcard form is allowed
```

For class tokens:
```java
if (clazz.isInstance(o)) {
    T t = clazz.cast(o);
}
```

---

## 9. Variance, summarized

| Form | Variance | Read | Write |
|---|---|---|---|
| `List<T>`             | invariant     | `T`     | `T`         |
| `List<? extends T>`   | covariant     | `T`     | nothing (except `null`) |
| `List<? super T>`     | contravariant | `Object`| `T`         |

Java is **use-site variance**: you write `? extends`/`? super` per use. Languages like Kotlin/Scala have **declaration-site variance** (`<out T>` / `<in T>`) which is sometimes cleaner but always present even when you don't need it.

---

## 10. Generic interfaces — Comparable and Comparator

Two of the most-used generic interfaces in the JDK:

```java
public interface Comparable<T> {
    int compareTo(T o);                     // how does this compare to o?
}

public interface Comparator<T> {
    int compare(T a, T b);                  // external ordering for any two Ts
}
```

`compareTo` (and `compare`) returns negative / zero / positive when the receiver (or `a`) is less / equal / greater than the argument.

A class is *naturally ordered* if it implements `Comparable<Self>`:
```java
public record Money(long cents) implements Comparable<Money> {
    @Override public int compareTo(Money other) {
        return Long.compare(cents, other.cents);
    }
}
```

A `Comparator<T>` is an *external* ordering — you can have many for the same type:
```java
Comparator<Person> byAge  = Comparator.comparingInt(Person::age);
Comparator<Person> byName = Comparator.comparing(Person::name);

people.sort(byAge.thenComparing(byName));
```

`Comparator.comparing`, `comparingInt`, `thenComparing`, `reversed` — these factory methods compose comparators and are very nice.

---

## 11. Reading complex generic signatures

Real JDK signatures are dense. A practiced eye picks them apart left-to-right.

`Collections.max`:
```java
public static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll)
```

Read it:
- `<T extends Object & Comparable<? super T>>` — T must be Comparable to itself or any supertype. (The redundant `& Object` exists for backward-compatible erasure quirks.)
- `Collection<? extends T>` — produces Ts (PECS).
- Returns a T.

Stream.collect:
```java
<R, A> R collect(Collector<? super T, A, R> collector)
```

- T is the stream's element type (declared on Stream).
- A is the collector's *accumulator* type (an internal scratch type).
- R is the result type.
- `Collector<? super T, A, R>` — accepts any collector that consumes T-or-supertype, has accumulator A, produces R.

The first time you read these, walk through every bracketed bit. After 50 of them your eyes glaze less.

---

## 12. A small worked example

A type-safe heterogeneous container — different keys yield different value types:

```java
public class TypedMap {
    private final Map<Class<?>, Object> map = new HashMap<>();

    public <T> void put(Class<T> key, T value) {
        map.put(Objects.requireNonNull(key), value);
    }

    public <T> T get(Class<T> key) {
        return key.cast(map.get(key));
    }
}

var m = new TypedMap();
m.put(String.class, "hello");
m.put(Integer.class, 42);

String s = m.get(String.class);   // typed
int n = m.get(Integer.class);     // unboxed
```

This pattern — keyed by `Class<T>` — is how Spring's `@Qualifier`, JPA's `EntityManager.find`, and many configuration frameworks deliver typed lookups despite erasure.

---

## 13. Try this

1. Write `<T> List<T> filter(List<? extends T> src, Predicate<? super T> pred)`. Why those wildcards? (Hint: PECS — `src` produces, `pred` consumes.)
2. Why does `List<Object> objs = new ArrayList<String>();` not compile? Why does `List<? extends Object>` not need `? extends`? (Trick — `List<?>` is the same as `List<? extends Object>`.)
3. Define `record Pair<A, B>(A first, B second) {}`. Write `<A, B> Pair<B, A> swap(Pair<A, B> p)`.
4. Why does this print `true`?
   ```java
   List<String> a = new ArrayList<>();
   List<Integer> b = new ArrayList<>();
   System.out.println(a.getClass() == b.getClass());
   ```
5. Predict: will this compile?
   ```java
   <T extends Number> double sum(List<T> xs) {
       double s = 0; for (T x : xs) s += x.doubleValue(); return s;
   }
   List<Integer> ints = List.of(1, 2, 3);
   sum(ints);
   ```
   What if you change `List<T>` to `List<? extends Number>` and drop the type parameter? Try both.

---

**Next:** [Module 13 — Collections framework](./13-collections.md)
