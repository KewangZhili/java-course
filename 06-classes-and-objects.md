# Module 06 — Classes, Objects, Constructors, `this`

> Goal: define your own types — classes — properly. Understand the lifecycle of an object: how it gets created, what `this` means, and what runs in what order. After this module you can model any small domain (an `Account`, a `Point`, a `User`).

---

## 1. What a class is

A **class** is a blueprint for objects. It declares:
- **Fields** — the data each object carries.
- **Methods** — the operations the object supports.
- **Constructors** — special methods that initialize a new object.
- (Plus nested classes, static members, etc. — covered later.)

An **object** (or **instance**) is one runtime value built from that blueprint. You create instances with `new ClassName(...)`.

A minimal example:

```java
public class Point {
    int x;
    int y;

    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    double distanceTo(Point other) {
        int dx = x - other.x;
        int dy = y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
}
```

Using it:

```java
Point a = new Point(0, 0);
Point b = new Point(3, 4);
System.out.println(a.distanceTo(b));   // 5.0
```

Anatomy:
- `class Point { ... }` — the class declaration.
- `int x, int y` — instance fields. Each `Point` has its own `x` and `y`.
- `Point(int x, int y) { ... }` — the constructor. Same name as the class; no return type.
- `double distanceTo(Point other) { ... }` — an instance method. `x` inside it refers to *this* point's `x`; `other.x` refers to the argument's.

---

## 2. `new` — what happens at object creation

When you write `new Point(3, 4)`:

1. The JVM allocates a new `Point` object on the heap. All fields are zeroed (`x = 0, y = 0`, references = `null`).
2. The constructor `Point(int, int)` runs:
   - First, it implicitly calls the parent class's constructor — `super()` — unless you wrote an explicit `super(...)` or `this(...)` as the first statement.
   - Then field initializers and `{ ... }` instance-initializer blocks run, in source order.
   - Then the constructor body runs.
3. The `new` expression evaluates to a **reference** to the newly initialized object.

You almost always store that reference: `Point p = new Point(3, 4);`.

Memory picture:
```
Stack frame of caller            Heap
┌─────────────────┐              ┌───────────────────┐
│  p ─────────────┼─────────────▶│ Point             │
└─────────────────┘              │   x = 3           │
                                 │   y = 4           │
                                 └───────────────────┘
```

---

## 3. Fields

Two kinds:

- **Instance fields** — declared without `static`. Each object has its own copy.
- **Static fields** (Module 08) — belong to the class. One copy total, shared across all instances.

```java
class Counter {
    static int totalCounters = 0;   // one slot in the Counter class itself
    int n = 0;                       // one slot in each Counter instance

    Counter() { totalCounters++; n = 0; }
}
```

### 3.1 Field initialization

You may give fields an initial value at the declaration site:
```java
class Order {
    String status = "NEW";          // runs at construction time, before the body
    List<String> items = new ArrayList<>();
}
```

These initializers run as part of construction (between the implicit `super(...)` and the constructor body).

You may also use **instance initializer blocks** — a `{ ... }` block at class scope:
```java
class Demo {
    int x;
    { x = 5; System.out.println("instance init"); }
}
```
Rare. They run for *every* constructor (after `super(...)`, before the body). Useful if multiple constructors share initialization logic that doesn't fit a field initializer. Most code prefers a private helper or constructor delegation (§5.4).

### 3.2 Default values, again

If you don't explicitly initialize a field, it gets its type's default: `0`, `0.0`, `false`, ` `, or `null`.

```java
class Demo {
    int n;            // 0
    String s;         // null
    boolean b;        // false
}
```

This matters: a `String` field that's never assigned is `null`, not `""`.

---

## 4. Methods on a class

You've seen these. The rule that matters now: an instance method always operates on a particular instance, available as `this`.

```java
class Account {
    long balanceCents;

    void deposit(long cents) {
        this.balanceCents += cents;   // explicit this
    }

    long balance() {
        return balanceCents;          // implicit this — same as this.balanceCents
    }
}
```

`this` is a **read-only reference** to the receiver of the method call. You can:
- read fields (`this.x`),
- call methods (`this.foo()` — usually written without `this.`),
- pass `this` to other methods (`other.compareTo(this)`),
- use `this` as the value of a field assignment that delegates to a constructor (§5.4).

You cannot reassign `this`. There's no equivalent in Java.

### 4.1 When you need to write `this.` explicitly

Two cases:
- **Disambiguation**: a parameter or local has the same name as a field.
  ```java
  Account(long balanceCents) {
      this.balanceCents = balanceCents;   // field = parameter
  }
  ```
- **Style**: in some teams, *always* writing `this.field` makes it visually clear which names are fields vs locals. Up to your team.

Otherwise, omit it.

---

## 5. Constructors

A constructor is a special method with these rules:
- **Same name as the class.**
- **No return type** — not even `void`.
- Called only via `new ClassName(...)` (or via `this(...)` / `super(...)` from another constructor).

If you write no constructor at all, the compiler generates an implicit no-arg constructor:
```java
class Empty {}            // gets a synthetic public Empty() {}
new Empty();              // works
```

Once you write *any* constructor, the implicit no-arg one disappears:
```java
class WithCtor {
    WithCtor(int n) { ... }
}
new WithCtor();   // ERROR — there's no no-arg constructor
```

If you need both, declare both.

### 5.1 Multiple constructors (overloading)

Like methods, constructors can be overloaded:
```java
class Order {
    long id;
    String status;

    Order(long id, String status) {
        this.id = id;
        this.status = status;
    }

    Order(long id) {
        this(id, "NEW");      // delegate to the other constructor
    }
}
```

### 5.2 `this(...)` — constructor delegation

A constructor's first statement may be `this(...)` to delegate to another constructor of the same class. The delegated-to constructor runs first; control returns to the outer one.

You can have at most one such delegation, and it **must** be the first statement (Java 22 relaxed this restriction in some cases via "flexible constructor bodies", but in 21 and earlier code it's the rule).

### 5.3 `super(...)` — calling the parent's constructor

If your class extends another (Module 10), the parent's constructor must run before yours. The first statement of your constructor may be `super(...)`. If you don't write one, the compiler inserts an implicit `super()` (parameterless). If the parent has no parameterless constructor, this is a compile error and you must write `super(...)` explicitly.

```java
class Animal {
    String name;
    Animal(String name) { this.name = name; }
}

class Dog extends Animal {
    String breed;
    Dog(String name, String breed) {
        super(name);            // must come first
        this.breed = breed;
    }
}
```

Either `this(...)` or `super(...)` may be first — never both.

### 5.4 Order of construction

When you `new SubClass(args)`:

1. Allocate memory for the SubClass object. All fields zero.
2. **Invoke the SubClass constructor.** Its first statement is `this(...)` or `super(...)`:
   a. If `this(...)`, run that other SubClass constructor (recursing through the rules).
   b. If `super(...)`, recursively initialize the parent (back up to `Object`).
3. Run **SubClass's instance field initializers and instance initializer blocks**, in source order.
4. Run the rest of SubClass's constructor body.

Two consequences worth remembering:

- A subclass's fields are **not** initialized when the parent constructor runs. If the parent constructor calls an overridable method that the subclass overrides — the override sees subclass fields still at their default values (often `null`). This is a sharp edge:
  ```java
  class Parent {
      Parent() { print(); }
      protected void print() { System.out.println("parent"); }
  }
  class Child extends Parent {
      private final String name;
      Child(String name) { this.name = name; }
      @Override protected void print() {
          System.out.println(name.toUpperCase());   // NPE — name is still null
      }
  }
  ```
  Rule: **don't call overridable methods from a constructor.**

- `static` initializers (next module) run before *any* instance is built. They run once per class, when the class is first initialized.

---

## 6. Constructors should leave the object in a valid state

A typical robust constructor:

```java
public class Account {
    private final String owner;
    private long balanceCents;

    public Account(String owner, long openingCents) {
        if (owner == null || owner.isBlank()) {
            throw new IllegalArgumentException("owner required");
        }
        if (openingCents < 0) {
            throw new IllegalArgumentException("opening balance must be non-negative");
        }
        this.owner = owner;
        this.balanceCents = openingCents;
    }
    ...
}
```

Three habits:
- **Validate** invariants at construction. Reject bad input fast (Module 14 covers exceptions).
- **Mark fields `final`** if they shouldn't change after construction — the compiler enforces "assigned exactly once".
- **Make objects immutable when you can.** Easier to reason about, safe to share across threads.

`Objects.requireNonNull` is the idiomatic null check:
```java
this.owner = Objects.requireNonNull(owner, "owner");
```

---

## 7. Instance methods, mutators, accessors

Conventional naming:
- **Accessor** (getter): `String getOwner()` or `String owner()`. Returns a field.
- **Mutator** (setter): `void setOwner(String owner)`. Updates a field.
- **Verb** for actions: `deposit`, `transferTo`, `cancel`, `submit`.

Many JDK and Spring/JPA code uses `getXxx`/`setXxx` (the "JavaBeans" convention) because reflection-based libraries look for those exact names. Records (Module 11) use the bare-name form (`x()` instead of `getX()`) — that's a deliberate departure.

### 7.1 Method signatures

The signature of a method is its **name + parameter types** (in order). Return type is *not* part of the signature.

You can have:
```java
int  doIt(String s) { ... }
int  doIt(int n)    { ... }     // OK — different parameter types
long doIt(String s) { ... }     // ERROR — same signature as the first one
```

---

## 8. A complete worked class

```java
import java.util.Objects;

public class BankAccount {
    private final String owner;
    private long balanceCents;
    private boolean closed;

    public BankAccount(String owner, long openingCents) {
        this.owner = Objects.requireNonNull(owner, "owner");
        if (openingCents < 0) {
            throw new IllegalArgumentException("opening balance must be non-negative");
        }
        this.balanceCents = openingCents;
        this.closed = false;
    }

    public String owner()          { return owner; }
    public long   balanceCents()   { return balanceCents; }
    public boolean isClosed()      { return closed; }

    public void deposit(long cents) {
        ensureOpen();
        if (cents <= 0) throw new IllegalArgumentException("cents must be positive");
        balanceCents += cents;
    }

    public boolean withdraw(long cents) {
        ensureOpen();
        if (cents <= 0) throw new IllegalArgumentException("cents must be positive");
        if (cents > balanceCents) return false;
        balanceCents -= cents;
        return true;
    }

    public void close() {
        ensureOpen();
        if (balanceCents != 0) {
            throw new IllegalStateException("balance must be zero before close");
        }
        closed = true;
    }

    private void ensureOpen() {
        if (closed) throw new IllegalStateException("account closed");
    }

    @Override
    public String toString() {
        return "BankAccount{" + owner + ", " + balanceCents + " cents, closed=" + closed + "}";
    }

    public static void main(String[] args) {
        var a = new BankAccount("Alice", 10_000);
        a.deposit(2_500);
        System.out.println(a);                       // 12500 cents
        System.out.println(a.withdraw(20_000));      // false — insufficient
        System.out.println(a.balanceCents());        // 12500
        a.withdraw(12_500);
        a.close();
        System.out.println(a);                       // closed=true
    }
}
```

This is the shape of a careful, idiomatic mutable Java class. We'll modernize it (records, immutability) in Module 11.

---

## 9. Inner shape: `equals`, `hashCode`, `toString`

Every class implicitly extends `java.lang.Object`, which has these methods. The defaults are usually wrong:
- `toString()` returns `BankAccount@1a2b3c` — class name and identity hash.
- `equals(Object)` returns `this == that` (identity).
- `hashCode()` returns the identity hash.

Whenever your class represents a *value* (not a unique identity), override `equals` and `hashCode` (and consider `toString`). Module 11 walks through the contract and patterns. For now, just know they're inherited.

---

## 10. Try this

1. Write a `Range` class with two `int` fields `lo`, `hi`. Constructor validates `lo <= hi`. Add `boolean contains(int x)` and `Range intersect(Range other)` (returning the overlap, or null if none).
2. Predict and explain what this prints:
   ```java
   class A {
       A() { System.out.print("A "); init(); }
       void init() { System.out.print("A.init "); }
   }
   class B extends A {
       int x = 7;
       @Override void init() { System.out.print("B.init x=" + x + " "); }
   }
   new B();
   ```
3. Modify `BankAccount` so the balance can never go negative *and* an attempted overdraw throws (rather than returning `false`). What's the trade-off? Which is more "Java idiomatic" — return value or exception? (Hint: M14 — exceptions are for *exceptional* conditions; insufficient funds is a normal outcome.)

---

**Next:** [Module 07 — Packages, imports, classpath](./07-packages-imports-classpath.md)
