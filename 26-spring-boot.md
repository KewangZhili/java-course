# Module 26 — Spring Boot

> Goal: build, run, and deploy a Spring Boot application. Understand auto-configuration, starters, REST controllers, JPA, and Actuator. Boot is *not* a different framework — it's Spring with conventions, sensible defaults, and embedded servers, packaged so a single command starts a production-grade service.

---

## 1. What Boot adds to Spring Core

Spring Core (Module 25) gives you the IoC container. Spring Boot stacks on:

1. **Starters** — curated dependency bundles. `spring-boot-starter-web` brings in MVC + Jackson + Tomcat. `spring-boot-starter-data-jpa` brings JPA + Hibernate.
2. **Auto-configuration** — when certain classes are on the classpath, Boot configures them with sensible defaults (a Tomcat server, a JSON message converter, a `DataSource` from properties).
3. **Embedded server** — a Boot app is a runnable JAR with Tomcat/Jetty/Undertow inside. No `war` file, no separate servlet container.
4. **`application.properties` / `application.yml`** — single, well-documented place for configuration.
5. **Production utilities** — Actuator (health/metrics endpoints), graceful shutdown, structured logging.

Result: a "Hello, world" REST service is ~30 lines.

---

## 2. The minimal Boot app

`pom.xml`:
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.5</version>
    </parent>
    <groupId>com.example</groupId>
    <artifactId>shop</artifactId>
    <version>1.0.0</version>

    <properties><java.version>21</java.version></properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

`Application.java`:
```java
package com.example.shop;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@RestController
class HelloController {
    @GetMapping("/hello")
    public String hello(@RequestParam(defaultValue = "world") String name) {
        return "Hello, " + name;
    }
}
```

Run:
```
mvn spring-boot:run
```

Hit it:
```
$ curl localhost:8080/hello?name=Alice
Hello, Alice
```

Two annotations, two methods. Tomcat is up on port 8080, JSON serialization is wired, exception handling is in place.

---

## 3. Anatomy of `@SpringBootApplication`

It's a meta-annotation combining three:
- `@Configuration` — this class is a Spring config (Module 25).
- `@ComponentScan` — discover `@Component`/`@Service`/`@Repository`/`@Controller` in this package and below.
- `@EnableAutoConfiguration` — turn on Boot's auto-config magic.

The component scan starts from the package of `@SpringBootApplication`. Convention: put the class at the root of your package tree.

---

## 4. REST controllers

```java
@RestController
@RequestMapping("/users")
class UserController {

    private final UserService svc;
    public UserController(UserService svc) { this.svc = svc; }

    @GetMapping("/{id}")
    public ResponseEntity<User> getOne(@PathVariable long id) {
        return svc.findById(id)
                  .map(ResponseEntity::ok)
                  .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping
    public List<User> list(@RequestParam(required = false) String city) {
        return city == null ? svc.all() : svc.byCity(city);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@RequestBody @Valid CreateUserRequest req) {
        return svc.create(req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable long id) {
        svc.delete(id);
    }
}
```

What's happening:
- `@RestController` = `@Controller + @ResponseBody` — return values become JSON via Jackson.
- `@RequestMapping("/users")` — base path for the class.
- `@GetMapping`, `@PostMapping`, etc. — HTTP method shortcuts.
- `@PathVariable` — bind a URL segment (`/users/42`).
- `@RequestParam` — bind a query parameter (`?city=Paris`).
- `@RequestBody` — deserialize JSON request body into the parameter.
- `@Valid` — trigger Bean Validation on the parameter (Module 27).
- `ResponseEntity<T>` — fine control over status code and headers.

### 4.1 Request / response DTOs

Use records for request and response bodies — immutable, auto-equals, perfect for JSON:

```java
public record CreateUserRequest(String name, String email) {}
public record UserView(long id, String name, String email) {}
```

Jackson handles records natively (since Boot 2.6+).

### 4.2 Exception handling

```java
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(EntityNotFoundException e) {
        return Map.of("error", e.getMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, String>> handleBadRequest(IllegalArgumentException e) {
        return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
    }
}
```

Centralized exception → HTTP mapping. Avoids `try`/`catch` in every controller.

---

## 5. Configuration

`application.yml` (or `.properties`):
```yaml
server:
  port: 8080
  shutdown: graceful

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: shop
    password: ${DB_PASSWORD}            # from env var, no plaintext
  jpa:
    hibernate.ddl-auto: validate

logging:
  level:
    root: INFO
    com.example: DEBUG

api:
  timeout-ms: 5000
  base-url: https://api.example.com
```

Inject:
```java
@ConfigurationProperties(prefix = "api")
public record ApiProps(int timeoutMs, String baseUrl) {}

@Service
public class ApiClient {
    public ApiClient(ApiProps props) { ... }
}
```

Enable property binding:
```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { ... }
```

`@ConfigurationProperties` auto-validates with `@Validated`:
```java
@ConfigurationProperties(prefix = "api")
@Validated
public record ApiProps(@NotBlank String baseUrl, @Positive int timeoutMs) {}
```

Misconfigured app fails fast at startup, not at first request.

### 5.1 Profiles

`application-dev.yml`, `application-prod.yml` overlay the base file when their profile is active.

```
SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run
```

Or property: `spring.profiles.active=dev`.

---

## 6. JPA and `@Transactional`

`spring-boot-starter-data-jpa` brings in JPA (the spec) + Hibernate (the implementation).

### 6.1 Entity

```java
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    protected UserEntity() {}      // JPA needs a no-arg ctor (can be protected)
    public UserEntity(String name, String email) { this.name = name; this.email = email; }

    // getters / setters / accessors
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}
```

### 6.2 Repository

```java
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    Optional<UserEntity> findByEmail(String email);
    List<UserEntity> findByCity(String city);
}
```

`JpaRepository` provides `save`, `findById`, `findAll`, `delete`, etc. for free. Method names like `findByEmail` are auto-implemented from the name (Spring Data parses the method).

### 6.3 Service with `@Transactional`

```java
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }

    @Transactional
    public UserEntity create(String name, String email) {
        if (repo.findByEmail(email).isPresent()) {
            throw new IllegalArgumentException("email already taken");
        }
        return repo.save(new UserEntity(name, email));
    }

    @Transactional(readOnly = true)
    public Optional<UserEntity> findById(long id) {
        return repo.findById(id);
    }
}
```

`@Transactional` opens a DB transaction before the method, commits at the end, rolls back on (uncaught) `RuntimeException`. `readOnly = true` is a Hibernate hint — fewer writes, slight performance gain.

### 6.4 The N+1 problem

JPA's lazy loading (`@OneToMany`, `@ManyToOne`) defaults to fetching associations on access. Loop over 100 entities, each with a lazy collection → 1 + 100 queries. Fix:
- `@EntityGraph` or `JOIN FETCH` in JPQL,
- DTO projection — return a record directly from the query.

```java
@Query("SELECT new com.example.UserView(u.id, u.name, u.email) FROM UserEntity u")
List<UserView> listAllAsView();
```

---

## 7. Actuator — production endpoints

Add `spring-boot-starter-actuator`. Exposes (depending on config):
- `/actuator/health` — liveness/readiness for Kubernetes.
- `/actuator/info` — build info, git commit.
- `/actuator/metrics`, `/actuator/prometheus` — runtime metrics (JVM, GC, request counts, latencies).
- `/actuator/loggers` — view and toggle log levels at runtime.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

In production, integrate with Prometheus + Grafana. The `prometheus` endpoint emits metrics in Prometheus format out of the box.

---

## 8. Testing a Boot app

Three layers, picked to suit the test:

### 8.1 Plain unit (no Spring)
```java
@Test
void greetsKnownUser() {
    var repo = mock(UserRepository.class);
    when(repo.findByEmail("a@b")).thenReturn(Optional.of(...));
    var svc = new UserService(repo);
    assertThat(svc.greet("a@b")).isEqualTo("Hello, ...");
}
```

### 8.2 Web slice
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService svc;

    @Test void getsUser() throws Exception {
        when(svc.findById(1)).thenReturn(Optional.of(new UserView(1, "Alice", "a@b")));
        mvc.perform(get("/users/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

`@WebMvcTest` bootstraps just the MVC layer (controllers, JSON converters, exception handlers) — no DB, no full app context. Fast.

### 8.3 Full integration
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class IntegrationTest {
    @Autowired MockMvc mvc;
    // entire context is loaded
}
```

For database tests, use `@DataJpaTest` (JPA layer only, with H2 by default) or **Testcontainers** for a real Postgres in Docker.

---

## 9. Packaging and running

```
mvn package                            # builds target/shop-1.0.0.jar — a fat / "executable" JAR
java -jar target/shop-1.0.0.jar
```

The JAR contains your code + every dependency. Boot's launcher script unpacks classloaders correctly. To pass JVM options:
```
java -Xmx2g -jar target/shop-1.0.0.jar --server.port=9090
```

Spring Boot args after the JAR: properties prefixed with `--` override `application.yml`. Env vars: `SERVER_PORT=9090 java -jar ...`.

For Docker, use a layered image:
```
FROM eclipse-temurin:21-jre
COPY target/shop-1.0.0.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

---

## 10. The whole picture

A typical Spring Boot service:

```
              HTTP/JSON
                 │
        ┌────────▼────────┐
        │  Controllers     │   @RestController
        └────────┬────────┘
                 │ DTOs (records)
        ┌────────▼────────┐
        │  Services        │   @Service, @Transactional
        └────────┬────────┘
                 │ entities or domain types
        ┌────────▼────────┐
        │  Repositories    │   JpaRepository
        └────────┬────────┘
                 │ SQL via Hibernate
        ┌────────▼────────┐
        │  Database        │
        └─────────────────┘

   plus Actuator for ops, Logging via SLF4J/Logback,
   Validation via Jakarta Validation, Security via spring-security
```

You'll add features (security, async, messaging, caching) by adding starters. Each is a few annotations.

---

## 11. Things to know about Boot

- **Convention over configuration**, but configuration is always available. When auto-config is wrong, override it.
- **`@SpringBootTest` is slow.** A single test loads the whole context (~5–30s). Prefer `@WebMvcTest` / `@DataJpaTest` slices.
- **`spring.profiles.active`** is the lever for environment-specific behavior. Use it.
- **Logging**: Boot uses SLF4J + Logback by default. Loggers are named after the class:
  ```java
  private static final Logger log = LoggerFactory.getLogger(UserService.class);
  log.info("User {} created", user.id());
  ```
  Don't `System.out.println`. Don't concatenate strings (`log.info("user " + id)`) — the placeholder form lets the framework skip the concat when the level is disabled.
- **Graceful shutdown**: `server.shutdown=graceful` lets in-flight requests complete on SIGTERM.

---

## 12. Try this

1. Build a "library" REST service: `Book(id, title, author)`. CRUD with JPA + H2 in-memory database. Test with `@WebMvcTest` and `@DataJpaTest`.
2. Add an `Actuator` endpoint and verify `GET /actuator/health` returns `{"status":"UP"}`.
3. Add a `@ConfigurationProperties` record for an "api" prefix with a base URL and a timeout. Wire it into a service. Override with an env var.
4. Move from `findAll()` returning entities to `findAll()` returning a list of `BookView` records via a JPQL projection.

---

**Next:** [Module 27 — Enterprise patterns, AOP, transactions](./27-enterprise-patterns.md)
