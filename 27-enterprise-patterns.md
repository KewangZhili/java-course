# Module 27 — Enterprise Patterns: Layering, AOP, Transactions, Validation

> Goal: recognize the patterns that show up in every non-trivial Java service. The Controller / Service / Repository split. `@Transactional` and what proxies actually do. AOP for cross-cutting concerns. Bean validation. After this module you can read any "enterprise" Java service and predict where each concern lives.

---

## 1. The standard layered architecture

A Spring (or any IoC) service typically has three layers:

```
[ HTTP, CLI, message broker, scheduler ]   ← inbound
              │
       ┌──────▼──────┐
       │ Controller  │   maps requests → service calls
       └──────┬──────┘
              │
       ┌──────▼──────┐
       │  Service    │   business logic; orchestrates repositories and other services
       └──────┬──────┘
              │
       ┌──────▼──────┐
       │ Repository  │   persistence: load and save entities
       └──────┬──────┘
              │
       [ Database, external API, cache ]
```

Responsibilities:

- **Controller** (or "handler", or message listener) — translates the protocol-specific input (HTTP request, Kafka message, scheduled tick) into a method call on a service. It is *thin* — input validation, deserialization, calling the service, formatting the response.
- **Service** — the brain. Encodes business rules. Composes repositories and other services. Wraps multi-step work in transactions. Throws domain exceptions.
- **Repository** (or "DAO") — single-purpose persistence. Loads and saves a particular entity. Knows about SQL/JPA, hides it from the rest of the world.

What's *not* there:
- Controllers don't query the database directly.
- Repositories don't call other services.
- Services don't know about HTTP.

This separation is what makes the codebase testable. You can unit-test a service with mocked repositories. You can `@WebMvcTest` a controller with a mocked service. Each layer's tests are fast and focused.

A common addition: a **DTO mapping layer** between layers. Controllers receive `CreateOrderRequest`; services work with domain types; repositories work with `OrderEntity`. Mappers convert between representations. Sometimes the layers collapse to one type when the model is simple.

---

## 2. `@Transactional` — and the proxy that runs it

Spring's transactions are not a deep thread-local hack. They're a **proxy pattern** built on AOP.

When the container constructs a `@Service` bean and any of its methods are `@Transactional`, Spring substitutes the bean with a *proxy*:

```
   client                 (Spring proxy)            (real bean)
   ┌──────┐    call        ┌──────────┐    delegate    ┌──────────┐
   │  …   │───────────────▶│  Proxy   │───────────────▶│  Bean    │
   └──────┘                └──────────┘                └──────────┘
                            │ before: open Tx
                            │ after:  commit/rollback
```

The proxy implements the same interface (or extends the same class via CGLIB) and forwards calls to the real bean, with transaction management around the body.

### 2.1 Self-call gotcha

The proxy intercepts only **external** calls. A method inside the bean calling another method of *the same bean* goes directly:

```java
@Service
public class OrderService {

    public void outer(Order o) {
        validate(o);
        inner(o);     // direct call — proxy not invoked, @Transactional ignored!
    }

    @Transactional
    public void inner(Order o) {
        repo.save(o);
    }
}
```

`outer` is not transactional and the call to `inner` does **not** start a transaction. This is one of the most-bitten Spring traps.

Three fixes (in increasing preference):
1. Move `@Transactional` to `outer`.
2. Inject the same service into itself (`self.inner(o)` via Spring) — works but smells.
3. Extract `inner` into a separate bean. Now the call crosses bean boundaries → through a proxy.

### 2.2 Rollback rules

By default, Spring **rolls back on `RuntimeException` and `Error`**, **commits on checked exceptions**. To roll back on a checked exception:

```java
@Transactional(rollbackFor = IOException.class)
```

The "rolls back on RuntimeException" rule is why most service exceptions extend `RuntimeException` (Module 14 §5).

### 2.3 Propagation

What if a transactional method calls another transactional method?

```java
@Transactional
void a() { b(); }

@Transactional(propagation = Propagation.REQUIRES_NEW)
void b() { ... }
```

- `REQUIRED` (default) — join the existing transaction or start a new one. Most uses.
- `REQUIRES_NEW` — suspend the outer, start an independent one. The inner commits/rolls back independently.
- `MANDATORY` — must already be in a transaction.
- `SUPPORTS` — use one if present, fine without.
- `NEVER` — must not be in a transaction.

Use the default unless you have a reason. `REQUIRES_NEW` is occasionally useful for audit logs that should commit even if the surrounding business transaction rolls back.

### 2.4 readOnly hint

```java
@Transactional(readOnly = true)
public List<User> findAll() { ... }
```

A hint to Hibernate: skip dirty-checking (faster), and (with some drivers) route to a read replica. Always set this on read-only service methods.

---

## 3. Aspect-Oriented Programming (AOP) in one sweep

AOP lets you declare *cross-cutting* behavior in one place and weave it across many. Spring uses it for transactions, security, caching, async, and lets you write your own.

The vocabulary:
- **Aspect** — a class that holds cross-cutting code (a logging aspect, a metrics aspect).
- **Pointcut** — a predicate selecting *where* the aspect applies (which methods).
- **Advice** — *what* the aspect does at those join points (`@Before`, `@After`, `@Around`).
- **Join point** — a point in execution where advice can run (a method call, in Spring AOP).

A small example — log every public method in a package:

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger log = LoggerFactory.getLogger(LoggingAspect.class);

    @Around("execution(public * com.example.shop.service..*(..))")
    public Object timed(ProceedingJoinPoint pjp) throws Throwable {
        long t0 = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long us = (System.nanoTime() - t0) / 1_000;
            log.info("{} took {} us", pjp.getSignature().toShortString(), us);
        }
    }
}
```

`@Around` advice wraps the method; calling `pjp.proceed()` runs the original. Add `spring-boot-starter-aop` and Spring weaves this into every matching bean automatically.

### 3.1 What AOP is good for

- Logging / metrics / tracing.
- Security (`@PreAuthorize`).
- Caching (`@Cacheable`).
- Retry (`@Retryable` from spring-retry).
- Audit fields (set `createdAt`/`updatedAt` automatically).

### 3.2 What it's not

- A way to hide complex logic. AOP is best for *simple, uniform* behavior. Complex logic in an aspect is hard to debug — stack traces have synthetic frames, breakpoints don't behave intuitively.
- A drop-in for missing design. If "every service method needs logging", that's fine. If different services need different logging — codify it explicitly.

---

## 4. Bean Validation

Annotate fields/parameters with constraints; let the framework check them.

`spring-boot-starter-validation` (or just `jakarta.validation-api` outside Boot) brings:

```java
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank String name,
    @Email   String email,
    @Min(0)  int age
) {}
```

Trigger validation:
```java
@PostMapping
public User create(@RequestBody @Valid CreateUserRequest req) { ... }
```

Spring will reject invalid requests with a 400 and a body describing what failed.

Validate inside services:
```java
@Service
@Validated
public class UserService {
    public User create(@NotBlank String email) { ... }
}
```

Or via the API:
```java
Validator v = ...;
Set<ConstraintViolation<CreateUserRequest>> errors = v.validate(req);
```

Common constraints: `@NotNull`, `@NotBlank`, `@NotEmpty`, `@Size(min, max)`, `@Min`, `@Max`, `@Pattern(regex)`, `@Email`, `@Past`, `@Future`. Records make great validated DTOs.

---

## 5. Logging

Use **SLF4J** as the API; Boot configures **Logback** as the implementation.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    public void create(User u) {
        log.info("Creating user {}", u.id());        // placeholder form — SLF4J handles formatting
        try { ... }
        catch (Exception e) {
            log.error("Failed to create user {}", u.id(), e);  // exception as last arg
            throw e;
        }
    }
}
```

Rules:
- Use `{}` placeholders, not `+` concatenation. Cheaper when the log level is disabled.
- Pass exceptions as the *last* argument (no placeholder needed).
- Log levels: `error` (something failed) > `warn` (anomaly, recovered) > `info` (significant event) > `debug` (developer detail) > `trace` (very fine-grained).
- Don't log secrets (passwords, tokens, PII). Configure logback to mask if needed.

For structured logging (JSON), add `logstash-logback-encoder`. Most production apps emit JSON for downstream parsing (Splunk, ELK, etc.).

---

## 6. Security (one-paragraph version)

Spring Security is its own multi-day topic. The 30-second view:
- Add `spring-boot-starter-security`. Out of the box, every endpoint requires HTTP Basic auth with a generated password (printed at startup).
- A `SecurityFilterChain` bean configures URL patterns: which paths are public, which require which roles.
- Authentication providers (form login, JWT, OAuth2, SAML) plug in.
- Method-level security: `@PreAuthorize("hasRole('ADMIN')")` on a service method.

Most production services use either a session-based form login (internal apps) or JWTs (APIs). Module is enormous — pick a tutorial when you need it.

---

## 7. Configuration profiles for environments

Already shown in Module 25; emphasizing here because it's pervasive:

- `application.yml` — base config.
- `application-dev.yml` — overrides for `dev` profile.
- `application-prod.yml` — overrides for `prod` profile.
- Beans annotated `@Profile("dev")` only register under that profile.

Activate via `SPRING_PROFILES_ACTIVE=prod` or `--spring.profiles.active=prod`.

Pattern: in dev, use an in-memory H2 DB and a stub email service. In prod, real Postgres and a real SMTP gateway. The application code is identical.

---

## 8. Putting it together — a vertical slice

`UserController.java`:
```java
@RestController
@RequestMapping("/users")
class UserController {
    private final UserService svc;
    public UserController(UserService svc) { this.svc = svc; }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserView create(@RequestBody @Valid CreateUserRequest req) {
        return svc.create(req).asView();
    }
}
```

`UserService.java`:
```java
@Service
public class UserService {
    private final UserRepository repo;
    private final EmailService email;
    private final Clock clock;

    public UserService(UserRepository repo, EmailService email, Clock clock) {
        this.repo = repo; this.email = email; this.clock = clock;
    }

    @Transactional
    public UserEntity create(CreateUserRequest req) {
        if (repo.findByEmail(req.email()).isPresent()) {
            throw new EmailTakenException(req.email());
        }
        var user = repo.save(new UserEntity(req.name(), req.email(), Instant.now(clock)));
        email.sendWelcome(user.getEmail());
        return user;
    }
}
```

`UserRepository.java`:
```java
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByEmail(String email);
}
```

Cross-cutting:
```java
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(EmailTakenException.class)
    public ResponseEntity<Map<String, String>> handle(EmailTakenException e) {
        return ResponseEntity.status(409).body(Map.of("error", e.getMessage()));
    }
}
```

You'd also have an `@Aspect` for metrics, an actuator endpoint, profile-specific properties, validation on the request — all the patterns from this module.

---

## 9. Anti-patterns to recognize and fix

- **Anemic services that just delegate to repositories.** If `UserService.findById` is one line calling `repo.findById`, the service layer adds no value. Either it'll grow (good) or skip it (a controller can talk to a repository directly for read-only flows; don't be religious).
- **Fat controllers.** Controllers handling business logic, validation, permission checks. Push it into a service.
- **Repositories that return DTOs.** Repositories should speak in entities (or projections); the service layer maps to DTOs.
- **`@Transactional` everywhere.** Costs DB connections; can cause subtle deadlocks. Use it on service methods that genuinely span multiple repository calls or modify data.
- **Catch-and-rethrow with a generic exception, losing the cause.** Always pass `cause` (Module 14).
- **Self-call to a `@Transactional` method inside the same bean.** The transaction does not start.
- **Synchronous calls to slow downstreams from a request handler.** Hold-and-block degrades throughput. Use `CompletableFuture` (M20) or virtual threads.

---

## 10. Try this

1. Refactor a fat controller (one with database calls and business logic inline) into Controller → Service → Repository.
2. Add a `@Transactional` method that does two writes; force a `RuntimeException` after the first; verify the first rolls back.
3. Demonstrate the self-call gotcha: build two methods in the same `@Service`, one annotated `@Transactional`. Show with logs that no transaction starts when the inner is called from the outer.
4. Build a tiny `@Aspect` that logs every `@Service` method call's duration. Apply to a small service; verify logs.
5. Use `@Valid` + `@NotBlank` + `@Email` on a `CreateUserRequest`. Send a malformed JSON; verify the 400 response.

---

**Next:** [Module 28 — Reading SFDC-Bazel-style codebases](./28-reading-sfdc-bazel.md)
