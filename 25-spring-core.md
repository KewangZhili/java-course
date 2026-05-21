# Module 25 — Spring Core: IoC and Dependency Injection

> Goal: understand the Spring Framework's central idea — *Inversion of Control* via *Dependency Injection* — independent of Spring Boot. After this module you can read Spring code, follow how objects get wired, and build a small Spring app without the Boot magic.

Module 26 stacks Spring Boot on top. The two are often spoken of together, but Spring Core is the foundation; Boot is "Spring with conventions and auto-configuration".

---

## 1. The problem Spring solves

Without a framework, your `main` method has to construct every object and wire them together:

```java
public static void main(String[] args) {
    var dataSource = createDataSource();
    var userRepo   = new JdbcUserRepository(dataSource);
    var emailSvc   = new SmtpEmailService("smtp.example.com");
    var userSvc    = new UserService(userRepo, emailSvc);
    var controller = new UserController(userSvc);
    var server     = new HttpServer(8080, controller);
    server.start();
}
```

In a small program, fine. In an enterprise application with hundreds of services, this wiring becomes:
- enormous,
- duplicated across `main` and tests,
- brittle (every new dependency means edits in many places),
- hard to swap implementations (production vs test).

**Inversion of Control** flips it: instead of you building objects, a *container* builds them. You declare what each component needs; the container figures out the construction order and supplies the dependencies.

**Dependency Injection** is the mechanism: a component receives its dependencies (typically through its constructor) rather than creating them itself.

---

## 2. The container

Spring's `ApplicationContext` is the IoC container. It:
- discovers your *beans* (the Java objects it should manage),
- builds them in the right order,
- injects dependencies,
- gives you a registry to look them up.

```java
@Configuration
public class AppConfig {
    @Bean DataSource dataSource() { return createDataSource(); }
    @Bean UserRepository userRepository(DataSource ds) { return new JdbcUserRepository(ds); }
    @Bean EmailService emailService() { return new SmtpEmailService("smtp.example.com"); }
    @Bean UserService userService(UserRepository repo, EmailService email) {
        return new UserService(repo, email);
    }
}

public class Main {
    public static void main(String[] args) {
        var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService svc = ctx.getBean(UserService.class);
        svc.doStuff();
    }
}
```

The container reads `AppConfig`, sees four `@Bean`-annotated methods, and figures out:
- `dataSource` has no deps; build it first.
- `userRepository` needs a `DataSource`; pass `dataSource()`.
- `emailService` has no deps.
- `userService` needs both; pass them.

You never write `new UserService(...)` in `main` again.

---

## 3. Stereotype annotations and component scanning

Writing `@Bean` methods for every class is tedious. Stereotypes let the container *find* classes automatically:

```java
@Component  public class JdbcUserRepository implements UserRepository { ... }
@Service    public class UserService { ... }
@Repository public class JdbcOrderRepository { ... }
@Controller public class UserController { ... }       // Spring MVC
@RestController public class UserApi { ... }          // returns JSON
```

All of these are specializations of `@Component`. They're functionally equivalent for the container — the more specific names communicate intent and are picked up by additional Spring features (e.g., `@Repository` for exception translation in JPA).

Component scanning:
```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {}
```

Now any `@Component`-annotated class under `com.example` is discovered, instantiated, and wired automatically. (`@SpringBootApplication` does this implicitly — Module 26.)

---

## 4. Dependency injection — three ways

### 4.1 Constructor injection — preferred

```java
@Service
public class UserService {
    private final UserRepository repo;
    private final EmailService email;

    public UserService(UserRepository repo, EmailService email) {
        this.repo = repo;
        this.email = email;
    }
}
```

Since Spring 4.3, if a class has *exactly one* constructor, `@Autowired` is implicit — you don't need to write it. Constructor injection is preferred because:
- Required dependencies are visible in the constructor signature.
- Fields can be `final`.
- The class is testable without the framework: `new UserService(mockRepo, mockEmail)`.
- You can't accidentally create a half-constructed instance.

### 4.2 Setter injection

```java
@Service
public class UserService {
    private UserRepository repo;
    @Autowired public void setRepo(UserRepository repo) { this.repo = repo; }
}
```

Used for **optional** dependencies. Rare in modern code.

### 4.3 Field injection (avoid)

```java
@Service
public class UserService {
    @Autowired private UserRepository repo;
}
```

Concise but bad: fields can't be `final`, the class is hard to test without Spring, and circular dependencies hide. Old codebases are full of this; new code shouldn't add more.

---

## 5. Wiring multiple implementations

If two beans implement the same interface, Spring needs disambiguation:

```java
public interface PaymentGateway { ... }

@Service
public class StripeGateway implements PaymentGateway { ... }

@Service
public class FakeGateway implements PaymentGateway { ... }
```

Now `@Autowired PaymentGateway gateway` is ambiguous. Three resolutions:

### 5.1 `@Primary`

```java
@Service @Primary
public class StripeGateway implements PaymentGateway { ... }
```

`@Primary` wins by default; others are still injectable when explicitly named.

### 5.2 `@Qualifier`

```java
@Service @Qualifier("fake")
public class FakeGateway implements PaymentGateway { ... }

@Service
public class CheckoutService {
    public CheckoutService(@Qualifier("fake") PaymentGateway gateway) { ... }
}
```

### 5.3 Inject all

```java
public CheckoutService(List<PaymentGateway> gateways) { ... }
public CheckoutService(Map<String, PaymentGateway> byName) { ... }
```

---

## 6. Bean scopes

By default, Spring creates exactly **one** instance per bean type — a *singleton*. Other scopes:

```java
@Service
@Scope("prototype")           // a new instance every injection / lookup
public class JobRunner { ... }

@Service
@Scope("request")             // one per HTTP request (web app)
public class RequestContext { ... }

@Service
@Scope("session")             // one per HTTP session
public class UserCart { ... }
```

The vast majority of beans are singletons. Make sure singleton beans are *thread-safe* — multiple HTTP requests will hit them simultaneously.

---

## 7. Lifecycle hooks

```java
@Service
public class CacheLoader {
    @PostConstruct           // after dependencies are injected
    public void warmUp() { ... }

    @PreDestroy              // before the container shuts the bean down
    public void persist() { ... }
}
```

Use sparingly. Most apps don't need lifecycle hooks; constructor and try-with-resources usually suffice.

---

## 8. Configuration and properties

Externalize config so the same code runs in dev/test/prod with different values. Properties live in `application.properties` (Boot convention):

```
db.url=jdbc:postgresql://localhost/shop
db.username=shop_user
api.timeout-ms=5000
```

Inject:
```java
@Service
public class ApiClient {
    public ApiClient(@Value("${api.timeout-ms}") int timeoutMs) { ... }
}
```

Or strongly-typed with `@ConfigurationProperties` (Spring Boot, Module 26):

```java
@ConfigurationProperties(prefix = "api")
public record ApiProps(int timeoutMs, String baseUrl) {}
```

---

## 9. Profiles

Different beans for different environments:

```java
@Service @Profile("prod")
public class StripeGateway implements PaymentGateway { ... }

@Service @Profile({"dev", "test"})
public class FakeGateway implements PaymentGateway { ... }
```

Activate via property `spring.profiles.active=dev` or env var `SPRING_PROFILES_ACTIVE=prod`.

Profile-specific properties: `application-dev.properties`, `application-prod.properties`.

---

## 10. The bean lifecycle in detail

For each bean, Spring:
1. **Constructs** — calls the constructor with resolved dependencies.
2. **Sets** — calls setters / field injection (rare).
3. **Aware-callbacks** — if the bean implements `BeanNameAware`, `ApplicationContextAware`, etc., calls those.
4. **`@PostConstruct`** — initialization callbacks.
5. **Custom init method** if specified.
6. *(Bean is now ready to use.)*
7. **`@PreDestroy`** — when the context shuts down.
8. Custom destroy method.

Most beans only see steps 1, 4 (sometimes), and 7 (rarely).

---

## 11. AOP — a sneak peek

Spring uses *aspect-oriented programming* to add cross-cutting behavior (transactions, logging, security) without cluttering business code. When you annotate a method `@Transactional`, Spring wraps the bean in a *proxy* that:
- starts a DB transaction before the method call,
- commits on success or rolls back on exception.

```java
@Service
public class OrderService {
    @Transactional
    public void place(Order o) { ... }    // wrapped in transaction by the proxy
}
```

A consequence to understand: **a self-call within the same bean bypasses the proxy.**
```java
@Service
public class OrderService {
    public void outer() { inner(); }       // this calls inner directly — proxy NOT invoked
    @Transactional public void inner() { ... }
}
```
The `outer → inner` call doesn't go through the proxy, so `@Transactional` on `inner` is ignored. Module 27 returns to this with the right pattern.

---

## 12. A complete tiny example

```java
// User.java
public record User(long id, String name) {}

// UserRepository.java
public interface UserRepository {
    Optional<User> findById(long id);
}

// InMemoryUserRepository.java
import org.springframework.stereotype.Repository;
import java.util.*;

@Repository
public class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> store = Map.of(
        1L, new User(1, "Alice"),
        2L, new User(2, "Bob"));
    @Override public Optional<User> findById(long id) {
        return Optional.ofNullable(store.get(id));
    }
}

// UserService.java
import org.springframework.stereotype.Service;

@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
    public String greet(long id) {
        return repo.findById(id)
                   .map(u -> "Hello, " + u.name())
                   .orElse("unknown user");
    }
}

// AppConfig.java
import org.springframework.context.annotation.*;

@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {}

// Main.java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {
    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            UserService svc = ctx.getBean(UserService.class);
            System.out.println(svc.greet(1));   // Hello, Alice
            System.out.println(svc.greet(99));  // unknown user
        }
    }
}
```

`pom.xml` snippet for the dependency:
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.1.14</version>
</dependency>
```

That's a complete, runnable Spring application. No Boot, no web server — just the core IoC container.

---

## 13. Why this style scales

- **Testability**: `new UserService(mockRepo)` works without Spring. Module 24's tests on this design are trivial.
- **Loose coupling**: `UserService` depends on the `UserRepository` *interface*, not the JDBC implementation. Swap to JPA later by changing one annotation.
- **Configuration external to code**: production tweaks happen in `application.properties` or env vars, not source.
- **Cross-cutting via AOP**: transactions, security, logging applied uniformly, declaratively.

This is the design Salesforce's enterprise Java code uses heavily. You'll see hundreds of `@Service`, `@Repository`, and constructor-injected fields once you start reading.

---

## 14. Try this

1. Build the example above. Run it. Add a `LoggingUserRepository` decorator that wraps any `UserRepository` and logs each call. Wire it via `@Primary` and `@Qualifier`.
2. Add a `Greeter` interface with two implementations: `EnglishGreeter` and `AssameseGreeter`. Use `@Profile("en")` and `@Profile("as")`. Toggle which one runs without changing code.
3. Why is constructor injection preferred over field injection? Try writing a unit test for a service that uses `@Autowired` private fields — what's awkward?
4. Explain what goes wrong here:
   ```java
   @Service
   public class A {
       @Autowired private B b;
   }
   @Service
   public class B {
       @Autowired private A a;
   }
   ```
   And how Spring would react with constructor injection instead. (Hint: with field injection, Spring lazily resolves; constructors expose the cycle and refuse to start.)

---

**Next:** [Module 26 — Spring Boot](./26-spring-boot.md)
