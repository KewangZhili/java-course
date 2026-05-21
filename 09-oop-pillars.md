# Module 09 — The Four OOP Pillars in Java

> Goal: understand encapsulation, inheritance, polymorphism, and abstraction not as buzzwords but as four concrete mechanisms Java gives you, with the syntax and runtime behavior of each. Modules 10 and 11 follow up with the full machinery (interfaces, abstract classes, equals/hashCode, records).

OOP isn't a list of features. It's *one idea*: bundle data with the code that operates on it, and let types substitute for one another through a shared contract. Four mechanisms support it.

---

## 1. Encapsulation — hide *how*, expose *what*

A class controls access to its own state. Outside code only uses the methods. The class can change its representation later without breaking callers.

```java
public class Account {
    private long balanceCents;          // hidden representation

    public void deposit(long cents)  { balanceCents += cents; }
    public long balance()            { return balanceCents; }
}
```

Tomorrow you might switch `balanceCents` to a `BigDecimal`, store it in a database, or compute it from a transaction log. Callers using `deposit` and `balance` don't need to change. That's encapsulation.

Two consequences worth noting:

- **Most fields should be `private`.** Public fields couple every caller to your storage choice. Once a field is public, evolving the class is harder.
- **Validation lives in the methods.** A constructor or setter that rejects bad inputs guarantees the class never enters a bad state. Outside code can't bypass it because it can't write the field directly.

```java
public class Email {
    private final String value;
    public Email(String value) {
        if (!value.contains("@")) throw new IllegalArgumentException();
        this.value = value;
    }
    public String value() { return value; }
}
```

You cannot construct an `Email` without `@`. Encapsulation enforces an invariant.

---

## 2. Inheritance — reuse + subtyping

A subclass `extends` a superclass. It inherits state and behavior, can add to it, can override it.

```java
public class Animal {
    protected final String name;
    public Animal(String name) { this.name = name; }
    public String speak() { return "..."; }
}

public class Dog extends Animal {
    public Dog(String name) { super(name); }
    @Override public String speak() { return "woof"; }
}
```

Two things the subclass gets:
- **State**: every `Dog` has a `name` field (inherited from `Animal`).
- **Subtyping**: every `Dog` *is-a* `Animal`. A `Dog` reference can be assigned to an `Animal` variable.

Java rules:
- A class extends **exactly one** class. There is no multiple class inheritance. (Interfaces are different — Module 10.)
- The implicit parent is `java.lang.Object` if you don't write `extends`.
- A subclass cannot reduce visibility of an inherited member. (Public stays public.)
- Constructors are **not** inherited. A subclass declares its own and explicitly delegates to the parent's via `super(...)`.

### 2.1 Composition over inheritance

A common piece of advice: when in doubt, prefer composition (a `Dog` *has-a* `Engine`/`Bark`) over inheritance (`Dog extends Engine`). Inheritance is a strong claim — the subclass is a subtype of the superclass and inherits all its constraints. If you only need to reuse some behavior, hold a reference to a helper instead.

```java
// inheritance — fragile if Bark behavior changes
class Dog extends Bark { ... }

// composition — looser coupling
class Dog {
    private final Bark bark = new Bark();
    public String speak() { return bark.sound(); }
}
```

You'll see both in real code. Inheritance dominates for "kind of" relationships (a `BankAccount` is-a `Account`); composition for "uses" (a `Service` has-a `Repository`).

---

## 3. Polymorphism — one operation, many forms

Two flavors:

### 3.1 Subtype (runtime) polymorphism

A reference of a parent type can hold any subtype. When you call a method, **the actual class of the object decides which override runs**, not the type of the variable.

```java
Animal a = new Dog("Rex");
System.out.println(a.speak());   // "woof" — Dog's override

Animal b = new Animal("Bob");
System.out.println(b.speak());   // "..." — Animal's version
```

This is **dynamic dispatch**, and it's the default for instance methods in Java. The bytecode instruction `invokevirtual` looks up the method on the runtime class.

### 3.2 Parametric polymorphism (generics)

Same code, different types:
```java
List<String>  names = new ArrayList<>();
List<Integer> ages  = new ArrayList<>();
```
Module 12 covers generics in depth.

### 3.3 What does NOT participate in dynamic dispatch

- `static` methods — bound at compile time to the static type.
- `private` methods — not visible to subclasses, so not overrideable.
- `final` methods — overriding is disallowed.
- Fields — field access is by static type, not dynamic. `parent.x` reads the parent's `x` even if a subclass declares its own `x` (don't shadow fields; it's confusing).

```java
class A {           int x = 1; static String hi() { return "A"; } }
class B extends A { int x = 2; static String hi() { return "B"; } }

A a = new B();
System.out.println(a.x);     // 1 — field access uses static type A
System.out.println(a.hi());  // "A" — static method, static type A
```

That's why instance methods are the polymorphic mechanism, not fields, not static methods.

### 3.4 `@Override`

`@Override` is an annotation that asks the compiler to verify the method actually overrides something. It costs nothing at runtime; it catches typos:

```java
class Animal { public String speak() { ... } }

class Dog extends Animal {
    @Override public String speek() { ... }   // typo — compile error if @Override present
}
```

**Always write `@Override` when overriding.** Without it the typo silently creates a brand-new method that's never called.

### 3.5 Covariant return

An override may narrow the return type:
```java
class Animal { Animal child() { return new Animal("baby"); } }
class Dog extends Animal {
    @Override Dog child() { return new Dog("puppy"); }   // legal — Dog is-a Animal
}
```

---

## 4. Abstraction — code against contracts

Abstraction is *programming to an interface, not an implementation*. You define **what** without committing to **how**, and let multiple implementations plug in.

In Java, two tools express this:
- **`abstract` class** — partial implementation; some methods are unimplemented placeholders that subclasses must fill in.
- **`interface`** — pure contract: a set of method signatures (mostly).

(Module 10 covers both fully.)

```java
public interface PaymentGateway {
    Receipt charge(long cents);
}

public class StripeGateway   implements PaymentGateway { public Receipt charge(long cents) { ... } }
public class FakeGateway     implements PaymentGateway { public Receipt charge(long cents) { ... } }

public class CheckoutService {
    private final PaymentGateway gateway;
    public CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }
    public void buy(long cents) { gateway.charge(cents); }
}
```

`CheckoutService` doesn't know — and doesn't care — which gateway it's using. You can:
- run it against `StripeGateway` in production,
- run it against `FakeGateway` in tests,
- swap to `BraintreeGateway` later without changing `CheckoutService`.

This is *the* design lever in any non-trivial Java codebase. Module 25 (Spring) is essentially the industrialization of this pattern: a framework wires the right implementation into each service for you.

---

## 5. SOLID — vocabulary you'll hear constantly

These are five design heuristics, often credited to Robert Martin / *Effective Java*. You don't memorize them like axioms; you *recognize* them when reviewers cite them.

- **S — Single Responsibility.** A class should have one reason to change. A `UserRepository` saves and loads users; it doesn't also send emails.
- **O — Open/Closed.** Open for extension (subclass / plug-in implementations), closed for modification (don't keep editing the same class for every new feature).
- **L — Liskov Substitution.** A subclass must be safely usable wherever its parent is. If `Square extends Rectangle` but breaks code that resizes rectangles, you've violated this.
- **I — Interface Segregation.** Many small interfaces beat one fat interface. Clients shouldn't depend on methods they don't use.
- **D — Dependency Inversion.** Depend on abstractions, not concretes. `CheckoutService` takes a `PaymentGateway`, not a `StripeGateway`.

These five principles together push you toward **pluggable, testable, evolvable code** — the kind of code Spring (M25) and SFDC's services are built around.

---

## 6. The four pillars, one example

Putting all four into one tiny piece of code:

```java
public abstract class Shape {                  // ← inheritance + abstraction (abstract base)
    public abstract double area();             // ← abstraction (no implementation)
    public final boolean isLargerThan(Shape other) {   // ← encapsulation (uses internal area)
        return area() > other.area();
    }
}

public final class Circle extends Shape {
    private final double radius;               // ← encapsulation (private field)
    public Circle(double radius) { this.radius = radius; }
    @Override public double area() {           // ← polymorphism (override)
        return Math.PI * radius * radius;
    }
}

public final class Rectangle extends Shape {
    private final double w, h;
    public Rectangle(double w, double h) { this.w = w; this.h = h; }
    @Override public double area() { return w * h; }
}

public class Demo {
    public static void main(String[] args) {
        Shape a = new Circle(1.0);
        Shape b = new Rectangle(2.0, 3.0);
        System.out.println(b.isLargerThan(a));    // dynamic dispatch on a.area() and b.area()
    }
}
```

Walk through what each keyword/feature is doing in OOP terms:
- `Shape` is **abstract** — it defines the *what* (`area`) without the *how*.
- `Circle` and `Rectangle` **inherit** from `Shape` and **override** `area` (polymorphism).
- `radius`, `w`, `h` are **private** (encapsulation) — outside code uses `area()` instead.
- `isLargerThan` calls `area()` polymorphically — at runtime it dispatches to the right override.

That's the entire OOP toolkit operating on five lines.

---

## 7. Two Java-specific notes worth pinning

- **All instance methods are virtual by default.** Anything you don't mark `private`, `static`, or `final` participates in dynamic dispatch. (Coming from C++? It's the opposite default: C++ is non-virtual unless you say `virtual`.)
- **`Object` is the implicit base class.** Every class — even `Shape` above — implicitly extends `java.lang.Object`. So every object has `equals`, `hashCode`, `toString`, `getClass`, and the synchronization primitives `wait`/`notify` (Module 18). Module 11 walks through these.

---

## 8. Try this

1. Build a small employee hierarchy:
   - `Employee` (abstract) with `name`, `monthlyPay()`.
   - `SalariedEmployee` — `monthlyPay = annualSalary / 12`.
   - `HourlyEmployee` — `monthlyPay = hourlyRate * hoursLogged`.
   Store them in `List<Employee>`. Sum `monthlyPay()` polymorphically.

2. In the `Shape`/`Circle` example, what happens if you remove `final` from `Circle.area()` (no override) and `Circle` itself? Why might that be undesirable?

3. Take the example from Module 06's `BankAccount`. Suppose you want a `SavingsAccount` that earns interest. Write `SavingsAccount extends BankAccount` with `applyInterest(double rate)`. Where does the interest calculation get its current balance from — directly from the `private` field, or via `balance()` accessor? Why does it matter? (Hint: you can't see the `private` field from a subclass.)

4. Why does this not behave the way you'd expect?
   ```java
   class A { public void greet() { System.out.println("hi from A"); } }
   class B extends A { public static void greet() { System.out.println("hi from B"); } }
   ```
   Try compiling. (Hint: you can't override an instance method with a static one. The compiler will refuse.)

---

**Next:** [Module 10 — Inheritance, abstract classes, interfaces](./10-inheritance-abstract-interfaces.md)
