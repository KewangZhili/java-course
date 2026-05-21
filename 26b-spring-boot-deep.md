# Module 26b — Spring Boot Deep Dive

> Goal: go past the "Hello, REST" surface. After this you understand what auto-configuration *actually* does, the JPA caveats that bite production, the testing slices that matter, deployment, and observability. This is the depth you need to debug a Spring Boot app in production.

This supplements [Module 26](./26-spring-boot.md). Read that first.

---

## 1. Auto-configuration — what's really happening

`@SpringBootApplication` includes `@EnableAutoConfiguration`. That triggers Spring Boot to scan, at startup, hundreds of "auto-configuration" classes — each annotated with **conditional** annotations that decide whether to apply.

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)        // only if DataSource is on the classpath
@ConditionalOnProperty(name = "spring.datasource.url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                  // only if you haven't defined one
    public DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();
    }
}
```

The conditionals you'll see:
- **`@ConditionalOnClass`** — only if a class is on the classpath. Drives "if Tomcat is here, configure a Tomcat server".
- **`@ConditionalOnMissingBean`** — only if the user hasn't already defined one. Lets you override defaults silently.
- **`@ConditionalOnProperty`** — only if a property is set/has a value. Toggle features via `application.yml`.
- **`@ConditionalOnWebApplication`** / **`@ConditionalOnNotWebApplication`** — context-aware.
- **`@AutoConfigureBefore` / `@AutoConfigureAfter`** — ordering hints.

### 1.1 Where they live

The list of auto-configurations comes from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` files (one per starter JAR). Each starter you depend on contributes some.

`spring-boot-starter-web` brings the auto-configs that wire Tomcat, Jackson, Spring MVC, error handling.
`spring-boot-starter-data-jpa` adds Hibernate, the EntityManagerFactory, the transaction manager.

### 1.2 Debugging auto-configuration

When something isn't configured the way you expect, run with `--debug`:

```
java -jar app.jar --debug
```

This prints an **auto-configuration report** showing every conditional — which matched, which didn't, why. Indispensable when "why isn't my bean wired?" comes up.

You can also enable the `/actuator/conditions` endpoint to inspect at runtime.

### 1.3 Overriding defaults

Three ways:

1. **Define your own `@Bean`.** `@ConditionalOnMissingBean` defers to you.
2. **Set a property.** Hundreds of `application.yml` knobs change auto-config behavior.
3. **Exclude an auto-config.** `@SpringBootApplication(exclude = SomeAutoConfig.class)` or `spring.autoconfigure.exclude=...` in properties.

---

## 2. Properties and `@ConfigurationProperties` — production-grade

`@Value("${...}")` works for one-off values. For structured config, `@ConfigurationProperties` is far better:

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @NotBlank String name,
    @NotNull Database database,
    @Min(1) int maxConnections
) {
    public record Database(
        @NotBlank String url,
        @NotBlank String username,
        String password
    ) {}
}
```

```yaml
app:
  name: shop
  database:
    url: jdbc:postgresql://...
    username: shop
    password: ${DB_PASSWORD}
  max-connections: 50
```

Enable scanning:
```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { ... }
```

Now any class annotated `@ConfigurationProperties` is registered as a bean. Inject it like any other:
```java
@Service
public class Foo {
    public Foo(AppProperties props) { ... }
}
```

Two big wins:
- **Validation at startup.** Misconfigured app fails fast with a useful error, not at first request.
- **Strongly typed.** No `Integer.parseInt("...".substring())` strewn through your code.

### 2.1 Property sources, in priority order

(Highest priority wins)

1. Devtools global settings (when running locally).
2. Test properties from `@TestPropertySource`.
3. Command-line args (`--server.port=8081`).
4. Servlet config / context init params.
5. JNDI attributes from `java:comp/env`.
6. Java System properties (`-Dserver.port=8081`).
7. OS environment variables (`SERVER_PORT=8081` — note: dashes/dots are mapped to underscores).
8. Random values (`${random.uuid}`).
9. Profile-specific properties from `application-{profile}.yml`.
10. `application.yml` / `application.properties` from `application/` config directories.
11. `application.yml` / `application.properties` packaged in your JAR.
12. `@PropertySource` annotations.
13. Default properties from `SpringApplication.setDefaultProperties`.

In practice you'll lean on:
- packaged `application.yml` for defaults,
- profile-specific `application-prod.yml` for environment overrides,
- env vars (Kubernetes secrets, Docker env) for secrets and host-specific values.

### 2.2 Profiles — beyond the basics

```yaml
# application.yml
spring:
  profiles:
    active: dev          # default; override with SPRING_PROFILES_ACTIVE
    group:
      production: prod, monitoring, security
```

A *group* lets you activate multiple profiles with one name. Useful when "production" really means "prod config + monitoring + security stack".

`@Profile("!dev")` for "everything except dev". `@Profile({"dev","test"})` for "any of these".

Beans without `@Profile` are registered always.

### 2.3 `@Conditional` — your own conditional bean

```java
@Bean
@ConditionalOnProperty(name = "feature.experimental", havingValue = "true")
public ExperimentalFeature experimentalFeature() { ... }
```

Toggle a bean entirely by setting `feature.experimental: true` in config. Useful for feature flags.

---

## 3. The JPA / Hibernate gotchas

Module 26 introduced JPA. Production usage hits these caveats:

### 3.1 The N+1 query problem — detection and fixes

Pattern that explodes:
```java
@Entity public class Order {
    @ManyToOne @JoinColumn(name = "customer_id")
    private Customer customer;
}

orders.forEach(o -> System.out.println(o.getCustomer().getName()));
// 1 query for orders + N queries (one per order) for customers
```

Fixes (in increasing power):

**Option A — `JOIN FETCH`:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
```

**Option B — `@EntityGraph`:**
```java
@EntityGraph(attributePaths = {"customer", "items"})
@Query("SELECT o FROM Order o")
List<Order> findAllWithGraph();
```

**Option C — DTO projection (often cleanest):**
```java
public record OrderView(long id, String status, String customerName) {}

@Query("SELECT new com.example.OrderView(o.id, o.status, o.customer.name) FROM Order o")
List<OrderView> findAllAsView();
```

DTO projection avoids loading entities entirely — fastest, most explicit, no lazy-loading surprises.

To **detect** N+1 problems, enable Hibernate SQL logging in dev:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

Or use the `Hibernate Statistics` MBean / Actuator metrics.

### 3.2 LazyInitializationException

Default `@OneToMany` and `@ManyToMany` are lazy. If you access them outside the transaction, you get this exception:
```java
Order o = orderRepo.findById(1L).get();
// transaction closes here
o.getItems();   // LazyInitializationException
```

Fixes:
- Access lazy fields **inside** the transaction (e.g. inside the service method).
- Use `JOIN FETCH` to load eagerly when you need them.
- Use a DTO projection to avoid the entity entirely.
- (Last resort) `@Transactional(propagation = REQUIRES_NEW)` extending the boundary — usually a smell.

`OpenEntityManagerInViewFilter` keeps the EM open across the entire HTTP request — Spring Boot enables it by default (`spring.jpa.open-in-view: true`). It hides N+1 problems and lazy-init bugs in development; **disable it in production** (`spring.jpa.open-in-view: false`) so they surface.

### 3.3 `@Transactional` reminders

- Default rolls back on `RuntimeException`, commits on checked. Override with `rollbackFor = ...`.
- Self-call from one bean method to another is **not proxied** — internal calls bypass `@Transactional`. (See Module 27.)
- Only public methods are proxied (CGLIB proxies require it; JDK proxies require interfaces).
- Long-running transactions hold DB connections. Keep them short.
- Don't make external calls (HTTP, S3) inside `@Transactional` — extends transaction time, holds connection.

### 3.4 Connection pool tuning

Spring Boot uses **HikariCP** by default. Tune via properties:
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # default 10
      minimum-idle: 5
      connection-timeout: 5000   # ms
      max-lifetime: 1800000       # 30 minutes
      idle-timeout: 600000        # 10 minutes
```

Sizing rule of thumb: `maximum-pool-size = (cores * 2) + 1` for CPU-bound; higher for I/O-bound. Watch DB-side `max_connections` — many app instances × pool size shouldn't exceed it.

### 3.5 Two-level cache (rarely worth it)

JPA's L1 cache is per-`EntityManager` (per transaction). The L2 cache (per-application, shared) integrates with EhCache, Hazelcast, Caffeine. Adds complexity and bugs (stale data, invalidation); use only when profiling shows the gain.

---

## 4. Spring Security — the 5-minute version

Spring Security is huge. The shape:

`pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Out of the box: every endpoint requires HTTP Basic auth with a password printed at startup. Customize:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/health", "/actuator/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .csrf(csrf -> csrf.disable())     // enable for browser apps; disable for stateless APIs
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
}
```

Common patterns:
- **JWT-validating REST API** — `oauth2ResourceServer().jwt()`. Validates a token, extracts claims, builds an `Authentication`.
- **Form login (server-rendered)** — `formLogin()`, sessions in cookies.
- **OAuth2 login** — `oauth2Login()`, redirect to Google/GitHub/your IdP.

Method-level annotations:
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(long id) { ... }

@PostAuthorize("returnObject.owner == authentication.name")
public Document loadDoc(long id) { ... }
```

Get the current user:
```java
@RestController
class Foo {
    @GetMapping("/me")
    public String me(Authentication auth) {     // Spring injects this
        return auth.getName();
    }
}
```

Production security is its own course. Recognize the surface; learn deeper when you need it.

---

## 5. Caching

`@Cacheable` adds caching declaratively:
```java
@Service
public class UserService {
    @Cacheable("users")
    public User findById(long id) { ... }      // result cached by id

    @CacheEvict(value = "users", key = "#id")
    public void delete(long id) { ... }

    @CachePut(value = "users", key = "#user.id")
    public User update(User user) { ... }      // updates the cache
}
```

Enable globally:
```java
@SpringBootApplication
@EnableCaching
public class Application { ... }
```

The default cache is in-memory `ConcurrentMapCacheManager`. Production usually uses **Redis** or **Caffeine**:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: redis
  redis:
    host: localhost
```

Caveats:
- The cache is keyed by method args' `hashCode`. Records work great. Custom objects need `equals`/`hashCode`.
- Self-call (within the same bean) bypasses the proxy — same as `@Transactional`.
- Don't cache mutable objects without thinking — multiple callers will see the same instance.

---

## 6. Messaging — Kafka

The pattern with `spring-kafka`:

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: shop-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example
```

```java
@Component
public class OrderEvents {

    private final KafkaTemplate<String, OrderCreated> kafka;
    public OrderEvents(KafkaTemplate<String, OrderCreated> kafka) { this.kafka = kafka; }

    public void publish(OrderCreated event) {
        kafka.send("orders", event.orderId().toString(), event);
    }
}

@Component
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "billing-service")
    public void onOrder(OrderCreated event) {
        // process
    }
}
```

Consumer concurrency, retry, dead-letter topics, and exactly-once semantics are tuned via configuration. Spring Kafka handles the threading; you focus on the per-event logic.

For RabbitMQ, the shape is similar with `spring-amqp` and `@RabbitListener`.

---

## 7. Scheduling

```java
@Component
public class Cleanup {

    @Scheduled(fixedDelay = 60000)            // every 60s after previous completes
    public void cleanup() { ... }

    @Scheduled(cron = "0 0 2 * * *", zone = "UTC")  // 02:00 daily
    public void nightly() { ... }
}
```

Enable:
```java
@SpringBootApplication
@EnableScheduling
public class Application { ... }
```

For multi-instance deployments, only one instance should run a scheduled job. Use `@SchedulerLock` (from `shedlock`) backed by a database or Redis:
```java
@Scheduled(...)
@SchedulerLock(name = "cleanupTask", lockAtMostFor = "5m")
public void cleanup() { ... }
```

For complex async workflows, look at Spring's `@Async` (drop-in async method via thread pool) and Spring Batch (heavyweight batch jobs).

---

## 8. Async methods

```java
@Service
public class NotificationService {

    @Async
    public CompletableFuture<Void> sendEmail(String to, String body) {
        // runs on a separate thread pool, not the caller's
        emailClient.send(to, body);
        return CompletableFuture.completedFuture(null);
    }
}
```

Enable globally with `@EnableAsync`. Configure the pool:
```java
@Bean(name = "applicationTaskExecutor")
public Executor applicationTaskExecutor() {
    ThreadPoolTaskExecutor pool = new ThreadPoolTaskExecutor();
    pool.setCorePoolSize(8);
    pool.setMaxPoolSize(32);
    pool.setQueueCapacity(1000);
    pool.setThreadNamePrefix("async-");
    pool.initialize();
    return pool;
}
```

Same self-call gotcha as `@Transactional` — calling an `@Async` method from the same bean runs it synchronously on the caller's thread.

---

## 9. Actuator — production endpoints in detail

The starter:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Default endpoints (under `/actuator/*`):

| Endpoint | Use |
|---|---|
| `/health` | Liveness/readiness — Kubernetes-friendly |
| `/info` | Build info, git commit |
| `/metrics` | Runtime metrics (JVM, GC, request, custom) |
| `/prometheus` | Prometheus scrape format |
| `/loggers` | View / change log levels at runtime |
| `/conditions` | Auto-config report |
| `/configprops` | All `@ConfigurationProperties` beans |
| `/env` | Property sources — careful, prints values |
| `/heapdump` | On-demand heap dump |
| `/threaddump` | Thread dump |
| `/scheduledtasks` | List of `@Scheduled` tasks |
| `/mappings` | All Spring MVC routes |

Expose selectively:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true             # adds /actuator/health/liveness and /readiness for K8s
```

### 9.1 Custom health indicators

```java
@Component
public class DatabaseHealth implements HealthIndicator {
    private final JdbcTemplate jdbc;
    public DatabaseHealth(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Override
    public Health health() {
        try {
            jdbc.queryForObject("SELECT 1", Integer.class);
            return Health.up().build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

`/actuator/health` reports `UP` only if all health indicators pass.

### 9.2 Custom metrics with Micrometer

Spring Boot Actuator uses **Micrometer** as its metrics façade — same API, multiple backends (Prometheus, Datadog, CloudWatch, etc.).

```java
@Service
public class CheckoutService {
    private final Counter ordersPlaced;
    private final Timer checkoutTime;

    public CheckoutService(MeterRegistry registry) {
        this.ordersPlaced = registry.counter("orders.placed");
        this.checkoutTime = registry.timer("checkout.duration");
    }

    public void place(Order o) {
        checkoutTime.record(() -> {
            // do work
            ordersPlaced.increment();
        });
    }
}
```

Out of the box, Spring Boot exposes JVM, GC, HTTP request metrics. Add your business metrics for user-facing behavior.

---

## 10. Observability — tracing, logs

### 10.1 Structured logging

```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"
  level:
    root: INFO
    com.example: DEBUG
```

For JSON logs (Splunk, ELK, Loki):
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>...</version>
</dependency>
```

Configure logback to use the JSON encoder. Each log line becomes JSON with fields the aggregator can index.

### 10.2 Distributed tracing — Micrometer Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

Auto-instruments incoming requests, HTTP client calls, JDBC queries. Spans are exported via OTLP to your trace backend (Jaeger, Tempo, Datadog).

Trace context is propagated via HTTP headers; downstream services inherit the `traceId`.

### 10.3 MDC + log correlation

The trace ID is added to MDC (Mapped Diagnostic Context) so log lines include it:
```
2024-01-15 10:00:00 INFO  [main,abc123,def456] CheckoutService - placed order
                                ^traceId  ^spanId
```

Log aggregator can then filter all logs for one trace.

---

## 11. Testing layers

Module 24 introduced unit / web slice / full integration. The full set:

| Annotation | Loads |
|---|---|
| `@WebMvcTest` | MVC + filters + advice + JSON; mock the rest |
| `@DataJpaTest` | JPA + an embedded DB (H2 by default); rolls back per test |
| `@JsonTest` | Jackson configuration + your DTOs |
| `@RestClientTest` | RestTemplate / WebClient + mock server |
| `@SpringBootTest` | Full context — slow but realistic |

Pick the smallest slice that covers your test. `@SpringBootTest` is the slowest (5–30s context boot). Slices are ~1s.

### 11.1 Testcontainers for real databases

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@Testcontainers
@SpringBootTest
class IntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void config(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Test void something() { ... }
}
```

Spins up a real Postgres in Docker per test class. Slower than H2 but catches real SQL bugs. The standard for integration tests.

For Kafka, MongoDB, Redis, etc. — corresponding containers exist.

---

## 12. Deployment

### 12.1 Building

```
mvn clean package
```

Produces `target/myapp-1.0.0.jar` — a fat / "executable" JAR with all dependencies bundled.

### 12.2 Running

```
java -Xms512m -Xmx2g -jar myapp-1.0.0.jar --server.port=8080
```

Common JVM flags for production:
```
-Xms2g -Xmx2g                    # fixed heap; no resize
-XX:+UseG1GC                     # default in modern JVMs
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump
-Xlog:gc*:file=/var/log/gc.log:time,level,tags:filecount=5,filesize=10M
```

### 12.3 Docker — layered builds

A Dockerfile that respects Boot's layer optimization:
```dockerfile
FROM eclipse-temurin:21-jre AS extract
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Layer order (rare → frequent change) gives Docker layer cache the best chance.

For Cloud Native Buildpacks:
```
mvn spring-boot:build-image
```

Produces a containerized image without writing a Dockerfile.

### 12.4 Graceful shutdown

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

On SIGTERM, the server stops accepting new connections, lets in-flight requests finish (up to the timeout), then exits. Pair with Kubernetes `terminationGracePeriodSeconds: 60`.

### 12.5 Native images (GraalVM)

```
mvn -Pnative native:compile
```

Produces a native binary. ~50ms startup vs 5s for the JVM, ~50MB RAM vs ~300MB. Trades:
- Slower compile (minutes).
- Some libraries don't work without configuration.
- Reflection / dynamic class loading needs hints.

Use for serverless / scale-to-zero workloads. For long-running services, the JVM is usually fine.

---

## 13. The full production checklist

Before shipping a Spring Boot service, verify:

- [ ] `application.yml` has profile-specific overrides; secrets via env vars.
- [ ] `spring.jpa.open-in-view: false` (production).
- [ ] HikariCP `maximum-pool-size` matches your load.
- [ ] `@Transactional(readOnly = true)` on read-only services.
- [ ] N+1 queries identified and fixed (DTO projections / `@EntityGraph`).
- [ ] Actuator `health/liveness` + `health/readiness` for K8s.
- [ ] Prometheus `/actuator/prometheus` enabled.
- [ ] Structured (JSON) logging in production.
- [ ] Distributed tracing wired up.
- [ ] `server.shutdown: graceful`.
- [ ] JVM flags: `-Xmx` set, GC logging on, heap dump on OOM.
- [ ] Unit + slice + integration (Testcontainers) tests in CI.
- [ ] Security: CSRF setting matches your auth model; sessions stateless for APIs.

---

## 14. Try this

1. Take an existing Spring Boot app. Run with `--debug` and inspect the auto-config report. Find one auto-config that didn't apply and explain why.
2. Find an N+1 problem in your code (turn on `org.hibernate.SQL: DEBUG`). Fix with a DTO projection.
3. Add Micrometer counters for one service method. Verify in `/actuator/prometheus`.
4. Wrap an existing test in Testcontainers + real Postgres. Note the startup cost vs H2.
5. Add `server.shutdown: graceful`. Send SIGTERM during a slow request and verify it finishes.

---

**Back to:** [Module 26 — Spring Boot](./26-spring-boot.md) | **Next:** [Module 27 — Enterprise patterns](./27-enterprise-patterns.md)
