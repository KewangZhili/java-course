# Module 24 — Testing: JUnit 5, Mockito, AssertJ

> Goal: write good unit and integration tests in idiomatic Java. After this module you can read any test file in a real codebase and write your own.

The de-facto stack:
- **JUnit 5 (Jupiter)** — test framework: how a test is declared and run.
- **AssertJ** — fluent, readable assertions (better than JUnit's built-in `assertEquals`).
- **Mockito** — mocking dependencies for unit tests.
- **Spring Test / Spring Boot Test** — for integration tests in Spring apps (Module 26).

JUnit 4 is the older generation; you'll see it in legacy code. JUnit 5 (`org.junit.jupiter`) is the current one.

---

## 1. The test as a method

```java
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class CalculatorTest {

    @Test
    void addsTwoNumbers() {
        Calculator c = new Calculator();
        assertThat(c.add(2, 3)).isEqualTo(5);
    }
}
```

Rules:
- Tests live in `src/test/java/`. Same package as the class under test by convention (gives access to package-private members).
- Test classes are named `XxxTest` (suffix). The test runner discovers them.
- Methods annotated `@Test` are tests. They take no arguments, return `void`.
- A failing assertion throws — the test runner reports it.

JUnit doesn't require a `public` modifier on test classes/methods (Jupiter is happy with package-private), and modern style omits it.

---

## 2. Lifecycle annotations

```java
class FooTest {
    @BeforeAll  static void initAll()    { /* runs once before any test */ }
    @BeforeEach void initEach()          { /* runs before each test */ }

    @AfterEach  void cleanupEach()       { /* runs after each test */ }
    @AfterAll   static void cleanupAll() { /* runs once after all tests */ }

    @Test void aTest() { ... }
}
```

`@BeforeAll` / `@AfterAll` must be `static` unless the test class is annotated `@TestInstance(Lifecycle.PER_CLASS)`.

Each `@Test` runs on a **fresh instance** of the test class by default (PER_METHOD lifecycle). State doesn't leak between tests.

---

## 3. Assertions

### 3.1 JUnit's built-in

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(5, c.add(2, 3));
assertTrue(condition);
assertFalse(condition);
assertNull(value);
assertNotNull(value);
assertSame(a, b);          // identity
assertNotSame(a, b);
assertArrayEquals(new int[]{1,2}, actual);
assertIterableEquals(expected, actual);
assertThrows(IllegalArgumentException.class, () -> validate(""));
```

Fine, but verbose for complex assertions.

### 3.2 AssertJ — preferred

```java
import static org.assertj.core.api.Assertions.*;

assertThat(c.add(2, 3)).isEqualTo(5);
assertThat(name).isNotNull().isNotBlank().startsWith("A");
assertThat(list).hasSize(3).contains("a", "b").doesNotContain("z");
assertThat(map).containsEntry("k", 1).hasSize(2);
assertThat(throwable)
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("balance");

assertThatThrownBy(() -> validate(""))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("blank");
```

AssertJ:
- Reads naturally as English.
- Auto-completes friendly methods specific to the type.
- Better failure messages out of the box.

You'll see both styles in real code; AssertJ wins on readability.

### 3.3 Soft assertions

If you want all assertions to run even when one fails (helpful for diagnostic dumps):
```java
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(user.name()).isEqualTo("Alice");
    softly.assertThat(user.age()).isEqualTo(30);
    softly.assertThat(user.email()).contains("@");
});
```

---

## 4. Mockito — mocking dependencies

When the unit under test depends on another component (a repository, an HTTP client, a clock), you don't usually want the real one in a unit test. Mockito creates a stand-in:

```java
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

class UserServiceTest {

    @Test
    void greetsKnownUser() {
        UserRepository repo = mock(UserRepository.class);
        when(repo.findById(42L)).thenReturn(Optional.of(new User(42, "Alice")));

        var svc = new UserService(repo);
        assertThat(svc.greet(42)).isEqualTo("Hello, Alice");

        verify(repo).findById(42L);     // ensure the call happened
    }

    @Test
    void greetsUnknownUser() {
        UserRepository repo = mock(UserRepository.class);
        when(repo.findById(anyLong())).thenReturn(Optional.empty());

        var svc = new UserService(repo);
        assertThat(svc.greet(42)).isEqualTo("unknown user");
    }
}
```

Mockito vocabulary:
- `mock(Class)` — create a mock instance. All methods return defaults (null/0/empty) unless stubbed.
- `when(...).thenReturn(...)` — stub a method.
- `when(...).thenThrow(...)` — stub to throw.
- `verify(mock).method(...)` — assert the method was called.
- `any()`, `anyLong()`, `eq(x)` — argument matchers.
- `ArgumentCaptor<T>` — capture arguments for inspection.

### 4.1 `@Mock` and `@InjectMocks`

Auto-instantiate mocks via JUnit extensions:
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService svc;       // constructor-injects the @Mocks

    @Test void name() {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1, "X")));
        ...
    }
}
```

### 4.2 When NOT to mock

- Don't mock value types (records, `String`, `BigDecimal`). Construct real instances.
- Don't mock the class under test.
- Don't over-mock — every mock is a piece of test code that can lie. If you find yourself mocking a half-dozen layers, your design is probably leaking concerns.

A good rule: **mock at boundaries** — repositories, HTTP clients, message buses. Use real instances of pure logic.

---

## 5. Parameterized tests

Run the same test with multiple inputs:

```java
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;

@ParameterizedTest
@ValueSource(strings = {"a", "abc", "hello world"})
void notBlank(String s) {
    assertThat(s).isNotBlank();
}

@ParameterizedTest
@CsvSource({
    "1, 2, 3",
    "5, 7, 12",
    "0, 0, 0"
})
void addsCorrectly(int a, int b, int expected) {
    assertThat(new Calculator().add(a, b)).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("provideArgs")
void usesMethodSource(String input, int expected) {
    assertThat(input.length()).isEqualTo(expected);
}
static Stream<Arguments> provideArgs() {
    return Stream.of(
        Arguments.of("a", 1),
        Arguments.of("abc", 3)
    );
}
```

Use parameterized tests when you want the same logic exercised against many cases without duplication.

---

## 6. Test display names and grouping

```java
@Test
@DisplayName("rejects negative deposits")
void rejectsNegative() {
    assertThatThrownBy(() -> account.deposit(-1))
        .isInstanceOf(IllegalArgumentException.class);
}

@Nested
class WhenAccountIsClosed {
    @Test void rejectsDeposits() { ... }
    @Test void rejectsWithdrawals() { ... }
}
```

`@Nested` groups related tests under a parent class — produces nicer report trees in IDEs.

---

## 7. Conditional and tagged tests

```java
@Test
@EnabledOnOs(OS.LINUX)
void linuxOnly() { ... }

@Test
@EnabledIfEnvironmentVariable(named = "RUN_INTEGRATION", matches = "true")
void integration() { ... }

@Test
@Tag("slow")
void slowOne() { ... }
```

Run a subset: configure surefire/failsafe (Maven) or `Test` task (Gradle) to include/exclude tags.

---

## 8. Test naming conventions

A good test name describes **what's expected**:
- `accountRejectsNegativeDeposit`
- `parsesEmptyJsonAsEmptyMap`
- `returnsEmptyOptionalWhenUserNotFound`

A common DSL:
- `unit_state_expectedBehavior` — `account_whenClosed_rejectsDeposits`
- "Should" form — `shouldRejectNegativeDeposit`

Pick one and stick to it. Module 26 will show Spring's preferred form.

---

## 9. Test pyramid

A common architectural picture:

```
           /---\
         /   E2E   \           few; full-stack
       /-----------\
     /  integration  \         some; multi-component
   /-------------------\
 /         unit          \     many; pure functions
 -------------------------
```

- **Unit tests** — fast (ms), no I/O, no Spring context, plain `new` + Mockito at boundaries.
- **Integration** — Spring `@SpringBootTest` slices, in-memory DB, real serializers.
- **End-to-end** — full system, browser tests / contract tests against a live deployment.

Most code's tests should be unit. Integration is invaluable for wiring; E2E is expensive — keep it minimal.

---

## 10. A small worked example

Class under test:
```java
public class TaxCalculator {
    private final Clock clock;
    public TaxCalculator(Clock clock) { this.clock = clock; }

    public long taxFor(long amountCents) {
        if (amountCents < 0) throw new IllegalArgumentException("amount must be >= 0");
        var month = LocalDate.now(clock).getMonthValue();
        long rateCents = month == 12 ? 5 : 10;          // discount in December
        return amountCents * rateCents / 100;
    }
}
```

Tests:
```java
import java.time.*;
import org.junit.jupiter.api.*;
import static org.assertj.core.api.Assertions.*;

class TaxCalculatorTest {

    @Test
    void rejectsNegative() {
        var c = new TaxCalculator(Clock.systemUTC());
        assertThatThrownBy(() -> c.taxFor(-1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void appliesDecemberDiscount() {
        var dec = Clock.fixed(Instant.parse("2025-12-15T00:00:00Z"), ZoneOffset.UTC);
        var c = new TaxCalculator(dec);
        assertThat(c.taxFor(10_000)).isEqualTo(500);    // 5%
    }

    @Test
    void usesNormalRateOtherMonths() {
        var jul = Clock.fixed(Instant.parse("2025-07-15T00:00:00Z"), ZoneOffset.UTC);
        var c = new TaxCalculator(jul);
        assertThat(c.taxFor(10_000)).isEqualTo(1_000);  // 10%
    }
}
```

Notice: `Clock` is injected. The tests use `Clock.fixed` to control "now" deterministically. **Pass `Clock` instead of calling `LocalDate.now()` directly** — it's the standard pattern for testable time.

---

## 11. Idioms for clean tests

- **Arrange-Act-Assert** structure: visually separate setup, the call under test, and the assertions.
- One concept per test. Multiple assertions are fine; multiple unrelated concepts make failures ambiguous.
- Don't test getters/setters. They're trivial — testing them adds noise.
- Don't share state between tests. Each test creates the world it needs.
- If a test grows beyond 30 lines, the *production code* probably needs refactoring (too many dependencies, tangled responsibilities).
- Mock ⊕ real boundary: mock interfaces; use real value classes.

---

## 12. Try this

1. Write tests for `BankAccount` from M06: deposit, withdraw, close, can't deposit when closed. Use AssertJ.
2. For `UserService` (depends on `UserRepository`), write a test using Mockito that verifies `service.greet(42)` returns `"unknown user"` when `repo.findById(42)` returns `Optional.empty()`.
3. Convert this loop-y test to a parameterized test:
   ```java
   @Test void emails() {
     for (String s : List.of("a@b.com", "x@y.org")) assertThat(svc.isValid(s)).isTrue();
     for (String s : List.of("nobody", "@nope")) assertThat(svc.isValid(s)).isFalse();
   }
   ```
4. Why is this test fragile?
   ```java
   @Test void usesCurrentTime() {
       assertThat(LocalDate.now()).isEqualTo(LocalDate.of(2025, 11, 1));
   }
   ```
   How do you fix it?

---

**Next:** [Module 25 — Spring core (IoC + DI)](./25-spring-core.md)
