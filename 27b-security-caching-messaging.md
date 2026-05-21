# Module 27b — Security, Caching, Messaging, Resilience

> Goal: cover the four "we'll need this in any real service" topics that Module 27 only previewed. Not exhaustive — practical depth on Spring Security, caching strategies, Kafka/messaging, and resilience patterns (retry, circuit breaker, rate limiting).

---

## 1. Spring Security — the production view

`spring-boot-starter-security` adds defaults: every endpoint requires authentication, CSRF on, session in cookies, password printed at startup. You configure beyond that with a `SecurityFilterChain`:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                       // stateless API
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/health/**", "/actuator/health/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

### 1.1 The two main models

**Form login (server-rendered apps).** Sessions in cookies. CSRF on. User logs in via a form; the browser sends the session cookie on every request.

**JWT (REST APIs).** Stateless. Client sends `Authorization: Bearer <token>` on each request; the server validates and extracts claims. CSRF can be off (no cookies in play).

For service-to-service auth in microservices, JWT (or mTLS) is the standard.

### 1.2 OAuth2 resource server — JWT validation

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/myrealm
          # or for symmetric:
          # jwk-set-uri: ...
```

Spring downloads the JWKS, validates incoming JWTs (signature, expiry, issuer), populates `Authentication`. Done.

To extract roles from a custom claim:
```java
@Bean
public JwtAuthenticationConverter jwtAuthConverter() {
    var grants = new JwtGrantedAuthoritiesConverter();
    grants.setAuthoritiesClaimName("roles");
    grants.setAuthorityPrefix("ROLE_");
    var c = new JwtAuthenticationConverter();
    c.setJwtGrantedAuthoritiesConverter(grants);
    return c;
}
```

### 1.3 Method-level security

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(long id) { ... }

@PreAuthorize("#user.id == authentication.name")    // SpEL — only allow owner
public void updateUser(User user) { ... }

@PostAuthorize("returnObject.owner == authentication.name")
public Document load(long id) { ... }

@PreFilter("filterObject.owner == authentication.name")
public void deleteAll(List<Item> items) { ... }
```

Enable with `@EnableMethodSecurity` (replaces the older `@EnableGlobalMethodSecurity`).

### 1.4 Reading the current user

```java
@RestController
class FooController {
    @GetMapping("/me")
    public String me(Authentication auth) {
        return "you are " + auth.getName();
    }

    @GetMapping("/details")
    public Map<String, Object> details(@AuthenticationPrincipal Jwt jwt) {
        return jwt.getClaims();
    }
}
```

`Authentication` has `getName()`, `getAuthorities()`, etc. For OAuth2/JWT, `@AuthenticationPrincipal Jwt jwt` gives the parsed token.

### 1.5 CSRF, CORS, and headers

```java
http
    .cors(cors -> cors.configurationSource(corsSource()))   // CORS for browser apps
    .csrf(csrf -> csrf
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))   // cookie-based CSRF
    .headers(h -> h
        .frameOptions(f -> f.deny())
        .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'")));
```

For browser-facing apps you want all of these. For REST APIs called only by other services, CSRF can usually be off.

### 1.6 Common Security pitfalls

- **Permit-all on `/actuator/**`** — exposes metrics, env, heapdumps. Restrict to localhost or admin role.
- **Disabling CSRF on a browser-facing app** — opens you to CSRF attacks.
- **No HTTPS in production** — passwords/cookies leak. Terminate TLS at the load balancer.
- **Trusting incoming JWT without verifying issuer + audience** — attacker forges tokens. Validate strictly.
- **Long-lived session cookies without refresh** — stolen cookie = persistent access. Use short-lived sessions + refresh tokens.

---

## 2. Caching — strategies and pitfalls

`@Cacheable` is the easy interface (Module 26). Production caching strategy:

### 2.1 Cache levels

- **In-process (Caffeine, ConcurrentMap)** — fastest, per-instance. Stale across instances.
- **Distributed (Redis, Hazelcast)** — shared across instances. Network round-trip per access.
- **CDN / edge** — for HTTP responses. Outside your app entirely.

A common pattern: **two-level cache** — Caffeine in front of Redis. Reads hit local first; misses go to Redis; misses go to the source.

### 2.2 Caffeine — modern in-process

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```java
@Bean
public CacheManager cacheManager() {
    var mgr = new CaffeineCacheManager("users", "products");
    mgr.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .recordStats());
    return mgr;
}
```

`maximumSize` limits memory. `expireAfterWrite` / `expireAfterAccess` controls TTL. `recordStats()` exposes hit/miss to Micrometer.

### 2.3 Redis cache

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000
  data:
    redis:
      host: localhost
      port: 6379
```

`@Cacheable("users")` now writes to Redis. Distributed; but every read is a network call.

Use Redis when:
- Multiple instances need to share cache.
- Cache survives restarts.
- Backing data is expensive to recompute.

Don't use Redis when:
- Per-instance is fine and the latency matters.
- Cached data is small and re-computable cheaply.

### 2.4 Cache invalidation strategies

| Strategy | When to use |
|---|---|
| **TTL only** | Stale-tolerant data. Simplest. |
| **Write-through** | `@CachePut` updates cache when DB updates. Stale window = 0 if all writes go through this path. |
| **Cache-aside** | App reads from cache; on miss, loads from DB and stores. The `@Cacheable` default. |
| **Event-based eviction** | Subscribe to DB change events / Kafka events; evict matching keys. Useful when writes happen elsewhere. |
| **Versioned keys** | Include a version in the key; bump version to invalidate everything. |

The hardest bug class: **stale cache after a DB write made elsewhere**. If your cache layer doesn't see a write that another path makes, you serve stale data forever (until TTL). Either route all writes through a `@CachePut` path or use event-based eviction.

### 2.5 Cache stampede

When a popular cache entry expires, hundreds of requests hit the source simultaneously. Mitigations:
- **Probabilistic early refresh** — refresh slightly before expiry with low probability per request.
- **Lock around the load** — only one caller computes; others wait.
- **Stale-while-revalidate** — serve stale; refresh in background.

Caffeine and Redis both have patterns for this; check your library.

---

## 3. Messaging — Kafka in detail

### 3.1 Producer

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate(ProducerFactory<String, OrderEvent> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

Key config:
- **`acks=all`** — wait for all in-sync replicas. Highest durability.
- **`enable.idempotence=true`** — exactly-once-per-partition producing.
- **`retries`** — automatic retry on transient failures.

### 3.2 Sending

```java
@Service
public class OrderEvents {
    private final KafkaTemplate<String, OrderEvent> kafka;
    public OrderEvents(KafkaTemplate<String, OrderEvent> kafka) { this.kafka = kafka; }

    public CompletableFuture<SendResult<String, OrderEvent>> publish(OrderEvent e) {
        return kafka.send("orders", e.orderId().toString(), e);   // partition by orderId
    }
}
```

Partitioning by a key (orderId) ensures all events for one order land on the same partition → ordered consumption.

For transactional sending across multiple topics:
```java
kafka.executeInTransaction(t -> {
    t.send("orders", e.orderId(), e);
    t.send("audit", e.orderId(), audit);
    return null;
});
```

### 3.3 Consumer

```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig { ... }   // similar shape, with deserializers

@Component
public class OrderConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "billing-service",
        containerFactory = "kafkaListenerContainerFactory")
    public void onOrder(
        @Payload OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset
    ) {
        // process
    }
}
```

Concurrency: each `@KafkaListener` runs on a single thread by default. Set `concurrency = 4` to consume from multiple partitions in parallel:
```java
@KafkaListener(topics = "orders", concurrency = "4")
```

### 3.4 Error handling

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafka) {
    var dlq = new DeadLetterPublishingRecoverer(kafka);
    return new DefaultErrorHandler(dlq, new FixedBackOff(1000L, 3));   // retry 3x with 1s delay
}
```

After retries, the message goes to a dead-letter topic (`orders.DLT` by convention). You then have alerting on DLT depth, manual replay tools, etc.

### 3.5 Exactly-once semantics

Only achievable with the **transactional producer + read-process-write** pattern in Kafka itself. For "consume from topic A, write to topic B" pipelines, use Kafka Streams or `@Transactional` over the consumer + producer.

For "consume + write to DB", the **outbox pattern** is the standard:
1. Service writes to DB *and* an `outbox` table in one transaction.
2. A separate job (Debezium, custom poller) reads the outbox and publishes to Kafka.

This gives you exactly-once delivery despite the dual-write problem.

### 3.6 Schema management

For evolving event schemas, use a schema registry (Confluent Schema Registry) with Avro / Protobuf serialization. JSON is easier but offers no compatibility guarantees — fields can disappear, types can change.

---

## 4. Resilience — retry, circuit breaker, rate limit

`spring-cloud-starter-circuitbreaker-resilience4j` brings Resilience4j integration. Production services need these patterns when calling unreliable downstreams.

### 4.1 Retry

```java
@Retry(name = "stripeApi", fallbackMethod = "fallback")
public Receipt charge(long cents) {
    return stripeClient.charge(cents);
}

public Receipt fallback(long cents, Throwable t) {
    return Receipt.failed(t.getMessage());
}
```

Configuration in `application.yml`:
```yaml
resilience4j:
  retry:
    instances:
      stripeApi:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException
```

Exponential backoff:
```yaml
exponential-backoff-multiplier: 2
```

### 4.2 Circuit breaker

After many failures, "open" the circuit — fast-fail without calling the downstream — then test periodically to see if it's recovered.

```java
@CircuitBreaker(name = "stripeApi", fallbackMethod = "fallback")
public Receipt charge(long cents) { ... }
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      stripeApi:
        sliding-window-size: 100
        failure-rate-threshold: 50         # 50% failures opens the breaker
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
```

States: closed (normal) → open (fast-fail) → half-open (try a few) → closed (recovered).

### 4.3 Rate limiter

```java
@RateLimiter(name = "stripeApi")
public Receipt charge(long cents) { ... }
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      stripeApi:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 0
```

Requests over the limit get `RequestNotPermitted`. Useful for staying under a vendor's rate limit.

### 4.4 Bulkhead

Limit concurrent calls so one slow downstream doesn't tie up your whole thread pool:

```java
@Bulkhead(name = "stripeApi", type = Bulkhead.Type.SEMAPHORE)
public Receipt charge(long cents) { ... }
```

```yaml
resilience4j:
  bulkhead:
    instances:
      stripeApi:
        max-concurrent-calls: 20
        max-wait-duration: 500ms
```

Combine these as needed: retry inside circuit-breaker inside bulkhead. Resilience4j composes them correctly.

---

## 5. Observability — distributed tracing in detail

### 5.1 Setup

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

```yaml
management:
  tracing:
    sampling:
      probability: 1.0           # 100% in dev; 0.01-0.1 in prod
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

Spring Boot auto-instruments:
- Incoming HTTP requests.
- Outgoing `RestTemplate` / `WebClient` calls.
- JDBC queries (with `p6spy` or `datasource-proxy`).
- Kafka producer/consumer.
- Caching (Caffeine, Redis).

### 5.2 Custom spans

```java
@Service
public class CheckoutService {

    private final Tracer tracer;

    public CheckoutService(Tracer tracer) { this.tracer = tracer; }

    public void place(Order o) {
        var span = tracer.nextSpan().name("checkout").start();
        try (var ws = tracer.withSpan(span)) {
            span.tag("order.id", o.id().toString());
            // do work
        } finally {
            span.end();
        }
    }
}
```

`@Observed` (Micrometer) is the higher-level shortcut:
```java
@Observed(name = "checkout", contextualName = "place-order")
public void place(Order o) { ... }
```

### 5.3 Trace correlation in logs

Spring Boot's default Logback pattern includes `%X{traceId}` and `%X{spanId}` from MDC. With JSON logs, every line has these fields → log aggregator can show all logs for one trace.

---

## 6. Practical operational tips

- **Liveness vs readiness** — liveness only fails on truly stuck (deadlock, unrecoverable). Readiness fails when not yet ready to serve (still initializing, downstream unhealthy). K8s restarts on liveness, removes from load balancer on readiness.
- **`/actuator/health/readiness`** can include downstream checks (DB, Kafka). Be careful — cascading failures.
- **Bulk retries during incidents amplify load.** Pair retry with bulkheads and circuit breakers.
- **Always log sampled** — full request bodies for failed requests, headers (with secrets stripped) for everything.
- **Don't log PII** unless your logging stack treats it as sensitive (PII redaction layer, restricted access).
- **Set `Authorization`-stripping in your log filters** — easy to leak tokens.

---

## 7. Try this

1. Add Spring Security with JWT validation (use a simple Keycloak instance or any OAuth2 provider). Restrict `/admin/**` to a role.
2. Add `@Cacheable` with Caffeine. Tune `maximumSize` and `expireAfterWrite`. Hit `/actuator/metrics/cache.gets?tag=cache:users` and verify hits.
3. Send a message to Kafka via `KafkaTemplate`. Add a `@KafkaListener` consumer in another service. Add a dead-letter topic for poisoned messages.
4. Wrap an external HTTP call with `@Retry` + `@CircuitBreaker`. Force failures and observe the breaker open / close.
5. Add OpenTelemetry tracing. View traces in Jaeger or Tempo. Verify `traceId` appears in your logs.

---

**Back to:** [Module 27 — Enterprise patterns](./27-enterprise-patterns.md)
