# Module 25b — Spring Core Deep Dive

> Goal: go past constructor injection. Understand the bean lifecycle in detail, how proxies work, the order in which the container does things, and the patterns that show up in mature Spring codebases.

This supplements [Module 25](./25-spring-core.md). Read that first.

---

## 1. `BeanFactory` vs `ApplicationContext`

Two layers in Spring's container:

- **`BeanFactory`** — the bare core. Knows how to register bean definitions and create instances. Minimal.
- **`ApplicationContext`** — extends `BeanFactory` with: event publication, internationalization, resource loading, environment + profile management, bean post-processors, lifecycle callbacks.

You'll never use `BeanFactory` directly in application code. `ApplicationContext` is what `@SpringBootApplication` builds. Mention it because:
- Reading internals (`ListableBeanFactory`, `ConfigurableApplicationContext`) shows up in advanced configuration.
- The distinction explains why some things "just work" — events, resources, etc., come from `ApplicationContext`'s extra layers.

The implementation you'll see most:
- **`AnnotationConfigApplicationContext`** — annotation-driven, the Spring Boot default.
- **`ClassPathXmlApplicationContext`** — older XML-config-based. Legacy; recognize it.
- **`AnnotationConfigServletWebServerApplicationContext`** — Spring Boot's web context.

---

## 2. The bean lifecycle in full detail

For each bean, Spring walks these phases:

```
1. Resolve dependencies from definitions
2. Instantiate (constructor)
3. Populate properties (setters / @Autowired fields)
4. Aware-callbacks (BeanNameAware, ApplicationContextAware, ...)
5. BeanPostProcessor.postProcessBeforeInitialization
6. @PostConstruct
7. InitializingBean.afterPropertiesSet
8. Custom init method (initMethod = "...")
9. BeanPostProcessor.postProcessAfterInitialization   ← this is where AOP proxies are built
10. (Bean ready — consumers get the proxy if any)
11. ... (use) ...
12. @PreDestroy
13. DisposableBean.destroy
14. Custom destroy method
```

You'll only ever directly use a few of these (constructor, `@PostConstruct`, `@PreDestroy`). The rest are how the container works internally.

### 2.1 BeanPostProcessors — where the magic happens

A `BeanPostProcessor` runs *for every bean* between instantiation and "ready". Spring Boot registers many:

- **`AutowiredAnnotationBeanPostProcessor`** — processes `@Autowired`, `@Value`.
- **`CommonAnnotationBeanPostProcessor`** — processes `@PostConstruct`, `@PreDestroy`, `@Resource`.
- **`AnnotationAwareAspectJAutoProxyCreator`** — wraps beans in AOP proxies.
- **`AsyncAnnotationBeanPostProcessor`** — wraps `@Async` methods.
- **`ConfigurationPropertiesBindingPostProcessor`** — binds `@ConfigurationProperties`.

This is where "annotations turn into behavior". You can register your own to add custom post-processing.

---

## 3. AOP proxies — the actual mechanics

`@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize` all work through proxies. The mechanism:

When the container builds a bean that has any of these annotations (directly or via interface), Spring creates a *proxy* and registers the proxy in place of the real bean. Two proxy types:

### 3.1 JDK dynamic proxies

Used when the bean implements at least one interface. The proxy implements the same interface(s):
```
client → JDK proxy (implements UserService) → real UserServiceImpl
```

Limited to interface methods. If the consumer holds a reference typed as the concrete class instead of the interface, the proxy doesn't apply.

### 3.2 CGLIB proxies

Used when the bean has no interfaces (or when forced via `proxyTargetClass = true`). CGLIB generates a *subclass* of the bean's class at runtime; the proxy is an instance of that subclass.

```
client → CGLIB subclass extends UserService → real UserService methods called via super
```

Limitations:
- The class can't be `final`.
- `final` methods can't be advised.
- `private` methods can't be advised.
- Static methods can't be advised.

Spring Boot's default is CGLIB (configurable via `spring.aop.proxy-target-class: false` for JDK proxies when interfaces exist).

### 3.3 The self-call problem (revisited)

```java
@Service
public class UserService {

    public void outer() {
        innerWithTx();      // direct call — bypasses proxy
    }

    @Transactional
    public void innerWithTx() { ... }
}
```

`outer()` calls `innerWithTx()` directly through `this`, which is the **real bean**, not the proxy. The `@Transactional` advice never runs. Common bug.

Fixes:
1. Move the annotation to `outer`.
2. Inject the service into itself (`AopContext.currentProxy()` or `@Autowired private UserService self;`) — works but smells.
3. Split into two beans so the call crosses bean boundaries.

Same applies to `@Async`, `@Cacheable`, `@Retryable`.

---

## 4. Bean scopes in depth

| Scope | Lifecycle | Use case |
|---|---|---|
| `singleton` (default) | one per container | most beans |
| `prototype` | new instance every injection / lookup | stateful per-call helpers |
| `request` | one per HTTP request | request-scoped state |
| `session` | one per HTTP session | per-user state |
| `application` | one per ServletContext | rarely useful |
| `websocket` | one per WS session | WS apps |

```java
@Service
@Scope("prototype")
public class JobRunner { ... }
```

The tricky case: a **singleton injecting a prototype**. By default, Spring resolves the dependency *once* — the singleton always sees the same prototype instance, defeating the point. Fix: inject `Provider<JobRunner>` or `ObjectFactory<JobRunner>` and call `.get()` each time you want a fresh instance:
```java
@Service
public class Coordinator {
    @Autowired private Provider<JobRunner> runners;

    public void run() {
        var r = runners.get();    // new instance per call
        r.execute();
    }
}
```

For request-scoped or session-scoped beans injected into singletons, the same problem exists; Spring solves it via *scoped proxies* automatically (`proxyMode = ScopedProxyMode.TARGET_CLASS`).

---

## 5. Configuration — `@Bean` methods

`@Bean` methods inside `@Configuration` classes are CGLIB-proxied so calls between `@Bean` methods return the registered bean instance:

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() { return new ServiceA(); }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(serviceA());      // returns the registered serviceA, not a fresh one
    }
}
```

If the class is annotated `@Configuration(proxyBeanMethods = false)` or just `@Component`, the method calls produce **fresh objects**, not the cached bean. Boot leans toward `proxyBeanMethods = false` for performance; if a bean really must be the same, inject it as a parameter:

```java
@Bean
public ServiceB serviceB(ServiceA a) { return new ServiceB(a); }
```

That always gets the registered bean. Cleaner anyway.

---

## 6. Conditional beans

`@Conditional` underpins all the `@ConditionalOn*` annotations. You can write your own:

```java
public class IsLinuxCondition implements Condition {
    @Override public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return System.getProperty("os.name").toLowerCase().contains("linux");
    }
}

@Bean
@Conditional(IsLinuxCondition.class)
public LinuxOnlyService svc() { ... }
```

Useful for environment-specific or feature-flagged beans.

---

## 7. Events

`ApplicationContext` is a publish/subscribe bus.

```java
public record OrderPlaced(long orderId) {}

@Component
public class OrderService {
    private final ApplicationEventPublisher publisher;
    public OrderService(ApplicationEventPublisher publisher) { this.publisher = publisher; }

    public void place(Order o) {
        // ...
        publisher.publishEvent(new OrderPlaced(o.id()));
    }
}

@Component
public class EmailListener {
    @EventListener
    public void onOrderPlaced(OrderPlaced event) {
        // send email
    }
}
```

Synchronous by default. Add `@Async` to an `@EventListener` for async dispatch (requires `@EnableAsync`).

`@TransactionalEventListener` defers handling until the *publishing transaction* commits — common pattern: "publish event but don't deliver until DB is consistent":
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderPlaced(OrderPlaced event) { ... }
```

---

## 8. Bean ordering — `@Order`, `@DependsOn`

Sometimes Spring needs to know which bean comes first:

```java
@Service
@Order(1)
public class FirstFilter implements Filter { ... }

@Service
@Order(2)
public class SecondFilter implements Filter { ... }

@Service
public class App {
    public App(List<Filter> filters) { ... }   // sorted by @Order
}
```

`List<Interface>` injection picks up all beans implementing the interface, sorted by `@Order` (default `Ordered.LOWEST_PRECEDENCE`).

`@DependsOn("otherBean")` forces creation order without making the dependency explicit. Use sparingly — if A truly needs B initialized first, inject B.

---

## 9. `ApplicationContextInitializer` and `ApplicationListener`

Hook into Spring Boot's startup before / during context creation:
```java
public class CustomInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        ctx.getEnvironment().getPropertySources().addFirst(...);
    }
}

@Bean
public ApplicationListener<ApplicationReadyEvent> readyListener() {
    return e -> System.out.println("ready!");
}
```

Useful for low-level config injection, custom property sources, or running code after startup.

---

## 10. `@Import`, `@ImportResource`, `@ComponentScan` filters

Beyond simple component scan:

```java
@Import({BeanA.class, BeanB.class})           // import specific @Configuration classes or beans
@ImportResource("classpath:legacy-config.xml")  // import XML config (rarely needed)

@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = LegacyService.class),
    includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyAnnotation.class)
)
```

---

## 11. Practical patterns from real codebases

### 11.1 Façade / configuration interface

A configuration class that encapsulates wiring for a subsystem:
```java
@Configuration
@EnableConfigurationProperties(CacheProperties.class)
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(CacheProperties props) {
        // build and return the cache manager from props
    }

    @Bean
    @ConditionalOnProperty("cache.enabled")
    public CacheStats cacheStats(CacheManager cm) { ... }
}
```

Single place to find "how is caching wired here".

### 11.2 Composite injection

```java
@Service
public class NotificationService {
    private final List<NotificationChannel> channels;

    public NotificationService(List<NotificationChannel> channels) { this.channels = channels; }

    public void notify(String msg) {
        channels.forEach(c -> c.send(msg));
    }
}
```

Spring injects every bean implementing `NotificationChannel`. Adding a new channel = adding a new `@Component`. No config changes.

For named lookup:
```java
public NotificationService(Map<String, NotificationChannel> channels) { ... }
// keys are bean names
```

### 11.3 Strategy injection by qualifier

```java
@Service @Qualifier("stripe") class StripeGateway implements PaymentGateway { ... }
@Service @Qualifier("paypal") class PayPalGateway implements PaymentGateway { ... }

@Service
public class CheckoutService {
    private final PaymentGateway gateway;
    public CheckoutService(@Qualifier("stripe") PaymentGateway gateway) { ... }
}
```

### 11.4 `@Primary` for "default if ambiguous"

```java
@Service @Primary class ProductionGateway implements PaymentGateway { ... }
@Service class FakeGateway implements PaymentGateway { ... }

@Service
public class CheckoutService {
    public CheckoutService(PaymentGateway gateway) { ... }   // gets ProductionGateway
}
```

---

## 12. Common pitfalls

- **Field injection** — hard to test, can hide circular deps. Constructor injection is the way.
- **Too many `@Configuration` classes** — overlap, ambiguity, hard to debug. Group cohesively.
- **Bean with state shared across threads** — singleton beans need to be thread-safe. Stateful logic per-request → service uses dependencies but doesn't *hold* per-request state in fields.
- **Calling `@Transactional` / `@Async` / `@Cacheable` self-methods** — bypasses proxy.
- **Holding `EntityManager` outside a transaction** — `LazyInitializationException`.
- **Singletons that depend on prototypes** — without `Provider`, you only get one instance.

---

## 13. Try this

1. Create a `BeanPostProcessor` that logs every bean's construction time. Register it; observe.
2. Write a custom `@Conditional` that checks for an environment variable. Use it to gate a bean.
3. Demonstrate the self-call gotcha: build two methods in the same `@Service`; one annotated `@Async`. Show with logs that the inner one runs synchronously when called from the outer.
4. Inject `List<MyInterface>` — observe Spring's automatic ordering. Add `@Order` annotations and verify.
5. Use `@TransactionalEventListener` with `phase = AFTER_COMMIT`. What happens if the publishing transaction rolls back?

---

**Back to:** [Module 25 — Spring Core](./25-spring-core.md)
