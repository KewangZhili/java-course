# Module 10 — Inheritance, Abstract Classes, Interfaces

> Goal: master Java's full inheritance machinery — `extends`, `super`, abstract classes, interfaces (with default and static methods), `sealed` hierarchies, and nested classes. After this module you can read any class hierarchy in any codebase.

---

## 1. `extends` and `super`

A class extends one parent. Members the parent declared `public`, `protected`, or package-private (if in the same package) are accessible to the subclass.

```java
public class Animal {
    protected final String name;
    public Animal(String name) { this.name = name; }
    public String speak() { return "..."; }
    public String describe() { return name + " says " + speak(); }
}

public class Dog extends Animal {
    public Dog(String name) { super(name); }
    @Override public String speak() { return "woof"; }
}

new Dog("Rex").describe();   // "Rex says woof"
```

`super` lets you reach the parent's version of an overridden method:

```java
public class LoudDog extends Dog {
    public LoudDog(String name) { super(name); }
    @Override public String speak() { return super.speak().toUpperCase(); }
    // returns "WOOF"
}
```

You can only reach **one level up** — there's no `super.super.speak()`.

---

## 2. The Object hierarchy is a tree

Every class transitively extends `Object`. Every reference type (including arrays) is a subtype of `Object`. So:

```java
Object o = new Dog("Rex");           // Dog → Animal → Object
Object[] arr = new String[3];        // arrays inherit from Object[]
```

`Object` provides `equals`, `hashCode`, `toString`, `getClass`, `wait`, `notify`, `notifyAll`. Module 11.

---

## 3. Method overriding rules

To override an inherited instance method:
1. Same name.
2. Same parameter list (signature).
3. Return type same or a *covariant* (narrower) type.
4. Visibility same or wider (e.g., parent `protected` → subclass `public` is OK; the reverse is not).
5. Cannot throw broader checked exceptions than the parent declared.

The compiler enforces all of these. `@Override` makes them visible.

You **cannot**:
- Override a `static` method (you can declare a same-named static, but it shadows, doesn't override).
- Override a `final` or `private` method.
- Override a constructor (constructors are not inherited).

---

## 4. `abstract` classes

Mark a class `abstract` when it represents an incomplete implementation — useful for sharing state and partial behavior, while leaving some pieces for subclasses.

```java
public abstract class Shape {
    public abstract double area();                  // unimplemented — subclass must override

    public boolean isLargerThan(Shape other) {       // shared behavior
        return area() > other.area();
    }
}

public class Circle extends Shape {
    private final double r;
    public Circle(double r) { this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}
```

Rules:
- An abstract class **cannot be instantiated** directly: `new Shape()` is illegal.
- An abstract class can have any mix of abstract and concrete methods.
- An abstract class can have constructors — used by subclasses via `super(...)`.
- An abstract class can have fields like any other class.
- A subclass that doesn't implement all abstract methods must itself be declared `abstract`.

Use abstract classes when you have:
- shared **state** (fields) across implementations,
- some default behavior that uses the abstract methods,
- the relationship is genuinely *is-a*.

If you have only behavior, no state, and no implementation logic — prefer an interface.

---

## 5. Interfaces

An **interface** declares a contract: a set of method signatures (and constants). A class promises to fulfill it with `implements`.

```java
public interface Comparable<T> {
    int compareTo(T other);
}

public class Money implements Comparable<Money> {
    private final long cents;
    public Money(long cents) { this.cents = cents; }
    @Override public int compareTo(Money other) {
        return Long.compare(this.cents, other.cents);
    }
}
```

Important properties:
- A class can `implements` **many** interfaces: `class Foo implements A, B, C { ... }`. This is Java's answer to multiple inheritance.
- An interface can `extends` **many** interfaces: `interface Foo extends A, B { ... }`.
- An interface itself cannot be instantiated.
- All methods in an interface are **implicitly `public`** (you don't write the keyword) and **abstract** unless they're `default`, `static`, or `private`.
- All fields in an interface are **implicitly `public static final`**. They're constants.

### 5.1 `default` methods (Java 8+)

Default methods give an interface a method body. Implementing classes inherit it unless they override.

```java
public interface Greeting {
    String name();
    default String hello() {                  // default implementation
        return "Hello, " + name();
    }
}

public class Friendly implements Greeting {
    @Override public String name() { return "Alice"; }
    // gets hello() for free
}
```

Why default methods exist: they let library authors **add new methods** to an interface without breaking every existing implementation. The standard library uses them heavily — `List.sort`, `Map.forEach`, `Collection.stream`, etc. were all added as defaults in Java 8.

### 5.2 `static` methods on interfaces (Java 8+)

You can declare static helpers on the interface itself:

```java
public interface Comparator<T> {
    int compare(T a, T b);

    static <T extends Comparable<T>> Comparator<T> naturalOrder() {
        return Comparable::compareTo;
    }
}

Comparator<Integer> c = Comparator.naturalOrder();
```

These are **not inherited** — call them via the interface name, not an instance.

### 5.3 `private` methods on interfaces (Java 9+)

Helper methods that default methods share, hidden from implementors:

```java
public interface Logger {
    default void info(String msg)  { log("INFO", msg); }
    default void error(String msg) { log("ERROR", msg); }
    private void log(String level, String msg) {
        System.out.println("[" + level + "] " + msg);
    }
}
```

### 5.4 The diamond — and how interfaces dodge it

If two interfaces provide a default method with the same signature, the implementing class must override and disambiguate:

```java
interface A { default String hi() { return "A"; } }
interface B { default String hi() { return "B"; } }

class C implements A, B {
    @Override public String hi() { return A.super.hi(); }   // pick one
}
```

This is *the* reason Java permits multiple-interface "inheritance" but not multiple-class inheritance: interfaces have only behavior (no state), so the diamond reduces to "which method body do I get?", which can be resolved by explicit choice. Class multiple inheritance would also have to merge field layouts, which has no clean answer.

### 5.5 Marker interfaces

An interface with no methods, used purely to *tag* a class:

```java
public interface Serializable {}            // signals: "OK to serialize me"
public class User implements Serializable {}
```

`Serializable`, `Cloneable`, `RandomAccess` are JDK examples. Modern Java prefers **annotations** for this purpose, but marker interfaces are still common.

### 5.6 Functional interfaces (a preview)

An interface with **exactly one abstract method** is a *functional interface*. Java compiles a lambda or method reference into an instance of one. Module 15 covers this fully.

```java
@FunctionalInterface
public interface Mapper<T, R> {
    R apply(T input);
}

Mapper<String, Integer> length = s -> s.length();   // lambda
length.apply("hello");                              // 5
```

---

## 6. Abstract class vs interface — when to use which

| Question | Abstract class | Interface |
|---|:-:|:-:|
| Need to hold state (fields) | ✓ | ✗ (only constants) |
| Need a constructor | ✓ | ✗ |
| Multiple inheritance needed | ✗ (single only) | ✓ |
| Want to provide default methods | ✓ | ✓ (since Java 8) |
| Will use it as a function type (lambda) | ✗ | ✓ (single-method) |
| Tightly coupled "is-a" with shared logic | ✓ | sometimes |

Modern Java code tends toward **interfaces with default methods** for most contracts, with abstract classes used only when shared state or constructor logic genuinely matters.

---

## 7. `sealed` types (Java 17+)

A *sealed* type explicitly lists the classes / interfaces allowed to extend it.

```java
public sealed interface Shape permits Circle, Square, Triangle {}

public final class Circle   implements Shape { ... }
public final class Square   implements Shape { ... }
public final class Triangle implements Shape { ... }
```

A subclass of a sealed type must be:
- `final` (no further inheritance), or
- `sealed` (further constrained), or
- `non-sealed` (re-opens the hierarchy below this point).

Why this matters:
- The compiler knows the **exhaustive** set of subtypes — pattern-matching `switch` (M22) can prove all cases are covered without a `default`.
- You explicitly model **closed unions** — "a Shape is either a Circle, Square, or Triangle, and nothing else".

Sealed types are excellent for modeling domain types like commands, events, or AST nodes:

```java
public sealed interface Result<T> permits Result.Ok, Result.Err {
    record Ok<T>(T value)              implements Result<T> {}
    record Err<T>(String message)      implements Result<T> {}
}
```

Note: `permits` types can be omitted if all permitted subclasses are in the same source file.

---

## 8. Nested types

Java has four flavors of types-inside-types:

```java
public class Outer {

    static class StaticNested { ... }            // 1. static nested
    class Inner { ... }                          // 2. inner (non-static)

    void method() {
        class Local { ... }                      // 3. local
        Runnable r = new Runnable() {            // 4. anonymous
            @Override public void run() { ... }
        };
    }
}
```

### 8.1 Static nested (most common)

A regular class scoped under another. **No reference to the outer instance.** Used for namespacing and grouping helpers tied to the outer class.

```java
public class Map {
    public static class Entry<K, V> { ... }      // Map.Entry
}
```

Inner records, enums, and interfaces are implicitly `static`.

### 8.2 Inner (non-static)

Each instance carries an *implicit* reference to an enclosing outer instance. You can only construct it given an outer instance:

```java
Outer o = new Outer();
Outer.Inner i = o.new Inner();      // unusual syntax — rarely written
```

Inner classes hold the outer alive (preventing GC) for as long as the inner exists. Use sparingly. Most "inner" classes you write should be `static`.

### 8.3 Local classes

A class defined inside a method. Visible only there. Mostly superseded by lambdas; rare in modern code.

### 8.4 Anonymous classes

A class declared and instantiated in one expression — typically to satisfy a single-method interface or extend an abstract class once. Pre-Java 8 they were everywhere; lambdas (Module 15) replace most uses:

```java
// pre-Java-8 idiom
button.addActionListener(new ActionListener() {
    @Override public void actionPerformed(ActionEvent e) { ... }
});

// Java 8+: lambda
button.addActionListener(e -> ...);
```

Anonymous classes still make sense for:
- abstract classes you want to extend ad-hoc,
- multi-method interfaces lambdas can't express.

---

## 9. The `instanceof` operator and pattern matching

`instanceof` checks whether an object is an instance of a type:
```java
if (animal instanceof Dog) {
    Dog d = (Dog) animal;
    d.bark();
}
```

Java 16+ pattern form binds the variable in the matching scope:
```java
if (animal instanceof Dog d) {
    d.bark();
}
```

It composes with `&&`:
```java
if (animal instanceof Dog d && d.age() > 5) { ... }
```

Use pattern matching to replace the verbose `instanceof + cast`. Module 22 covers pattern matching in `switch` over sealed hierarchies.

---

## 10. Realistic example — a small DI-style design

```java
// abstraction
public interface UserRepository {
    Optional<User> findById(long id);
    void save(User u);
}

// production implementation
public class JdbcUserRepository implements UserRepository {
    private final DataSource ds;
    public JdbcUserRepository(DataSource ds) { this.ds = ds; }
    @Override public Optional<User> findById(long id) { ... }
    @Override public void save(User u) { ... }
}

// test implementation
public class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    @Override public Optional<User> findById(long id) {
        return Optional.ofNullable(store.get(id));
    }
    @Override public void save(User u) { store.put(u.id(), u); }
}

// consumer — depends on the abstraction
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }

    public String greet(long id) {
        return repo.findById(id)
                   .map(u -> "Hello, " + u.name())
                   .orElse("unknown user");
    }
}
```

`UserService` is fully testable with `InMemoryUserRepository`. The shape is everywhere in Spring (M25) — the framework wires the right `UserRepository` for you.

---

## 11. Try this

1. Build a `Shape` hierarchy as a sealed interface with three records: `Circle(radius)`, `Rectangle(w, h)`, `Triangle(a, b, c)`. Implement `area()` for each. Use a `switch` expression to compute total area over a `List<Shape>`.

2. Write an interface `Cache<K, V>` with `Optional<V> get(K key)` and `void put(K key, V value)`. Provide two implementations: `NoOpCache` (does nothing) and `MapCache` (backed by a `HashMap`). Use `NoOpCache` to disable caching without changing callers — what OOP pillar is this exercising?

3. Add a `default` method `int size()` to `Cache` returning `-1` ("unknown"). `MapCache` overrides it to return the actual size; `NoOpCache` inherits the default. Why is this a useful pattern for evolving APIs?

4. Why is this a compile error? Fix it.
   ```java
   interface A { default String hi() { return "A"; } }
   interface B { default String hi() { return "B"; } }
   class C implements A, B { }
   ```

---

**Next:** [Module 11 — Equality, `Object`, enums, records](./11-equality-object-enums-records.md)
