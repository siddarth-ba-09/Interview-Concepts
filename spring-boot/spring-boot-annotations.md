# Spring Boot Annotations — Complete Reference

---

## 1. Overview

Spring Boot annotations are meta-annotations or composed annotations that drive the **auto-configuration**, **component scanning**, **dependency injection**, **web layer**, **data layer**, **security**, **testing**, and **AOP** mechanisms of a Spring application.

Understanding the *exact* purpose of every annotation — and crucially **the differences between similar-looking annotations** — is one of the most common interview focus areas for Spring Boot roles.

This document covers every major annotation category, explains how each works internally, and draws explicit comparisons between annotations that are often confused.

---

## 2. Core Concepts & Sub-Topics

### Annotation Categories

| Category | Annotations |
|---|---|
| Bootstrap / Entry Point | `@SpringBootApplication`, `@EnableAutoConfiguration`, `@SpringBootConfiguration` |
| Stereotype (Component Scanning) | `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` |
| Dependency Injection | `@Autowired`, `@Qualifier`, `@Primary`, `@Resource`, `@Inject`, `@Value`, `@ConfigurationProperties` |
| Configuration | `@Configuration`, `@Bean`, `@Import`, `@ImportResource`, `@PropertySource`, `@Profile`, `@Conditional` |
| Web / MVC | `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`, `@ResponseBody`, `@ResponseStatus`, `@RequestHeader`, `@CookieValue`, `@ModelAttribute`, `@SessionAttributes`, `@CrossOrigin` |
| Exception Handling | `@ExceptionHandler`, `@ControllerAdvice`, `@RestControllerAdvice` |
| Data / JPA | `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@Transactional`, `@Query`, `@Repository` |
| Validation | `@Valid`, `@Validated` |
| AOP | `@Aspect`, `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`, `@Pointcut` |
| Caching | `@EnableCaching`, `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` |
| Scheduling & Async | `@EnableScheduling`, `@Scheduled`, `@EnableAsync`, `@Async` |
| Testing | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean`, `@SpyBean` |
| Actuator | `@Endpoint`, `@ReadOperation`, `@WriteOperation`, `@DeleteOperation` |
| Lifecycle | `@PostConstruct`, `@PreDestroy`, `@Lazy` |
| Scope | `@Scope` — Singleton, Prototype, Request, Session, Application |
| Security | `@EnableWebSecurity`, `@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed` |

---

## 3. How It Works Internally

### How Annotations Are Processed

1. **Compile time**: The Java compiler stores annotation metadata in `.class` bytecode.
2. **ClassLoading**: Spring's `ClassPathScanningCandidateComponentProvider` scans the classpath for classes annotated with stereotype annotations.
3. **BeanDefinition registration**: Each discovered class is converted into a `BeanDefinition` and registered in the `BeanDefinitionRegistry`.
4. **BeanPostProcessor chain**: `AutowiredAnnotationBeanPostProcessor` processes `@Autowired`; `CommonAnnotationBeanPostProcessor` processes `@PostConstruct`, `@PreDestroy`, `@Resource`.
5. **AOP proxying**: If any bean has AOP advice or `@Transactional`, Spring wraps it in a CGLIB or JDK dynamic proxy.
6. **Property binding**: `@ConfigurationProperties` binds external properties to POJOs via `ConfigurationPropertiesBindingPostProcessor`.

```
@SpringBootApplication
       |
       +--- @SpringBootConfiguration  → registers this class as a @Configuration
       +--- @EnableAutoConfiguration  → triggers AutoConfigurationImportSelector
       +--- @ComponentScan            → scans basePackage and sub-packages
```

### Auto-Configuration Flow

```
1. JVM loads SpringApplication.run()
2. SpringApplication triggers ApplicationContext creation
3. @EnableAutoConfiguration fires AutoConfigurationImportSelector
4. Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
5. Filters out conditions (@ConditionalOnClass, @ConditionalOnMissingBean, etc.)
6. Registers surviving auto-config classes as @Configuration beans
7. Component scan picks up user-defined @Component / @Service / @Repository / @Controller beans
8. @Autowired wiring is performed
9. ApplicationContext is fully refreshed
```

---

## 4. Key Annotations — Detailed Reference

---

### 4.1 Bootstrap Annotations

#### `@SpringBootApplication`

```java
@Target(ElementType.TYPE)
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { ... }
```

- **Composed of**: `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`
- Places the main class as a configuration class, enables auto-config, and scans all sub-packages.
- `exclude` attribute lets you exclude specific auto-configurations.

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

#### `@SpringBootConfiguration`

- Specialisation of `@Configuration`.
- Signals that this class provides Spring application configuration.
- Only **one** `@SpringBootConfiguration` is allowed per application (enforces single source of truth for tests).

#### `@EnableAutoConfiguration`

- Tells Spring Boot to auto-configure beans based on classpath contents and defined beans.
- Uses `AutoConfigurationImportSelector` which reads `AutoConfiguration.imports`.
- Controlled by `spring.boot.enableautoconfiguration=false` to disable globally.

---

### 4.2 Stereotype Annotations

All four are meta-annotated with `@Component`, so all participate in component scanning and produce Spring beans. The **differences are semantic and functional**, not structural.

| Annotation | Extends | Purpose | Extra Behaviour |
|---|---|---|---|
| `@Component` | — | Generic Spring-managed component | None |
| `@Service` | `@Component` | Business logic layer | None (purely semantic) |
| `@Repository` | `@Component` | Data access layer (DAO) | **Exception translation** — persistence exceptions (e.g. `SQLException`) are translated to Spring's `DataAccessException` hierarchy via `PersistenceExceptionTranslationPostProcessor` |
| `@Controller` | `@Component` | Spring MVC web controller | Works with `DispatcherServlet`; methods can return view names |
| `@RestController` | `@Controller` + `@ResponseBody` | REST API controller | Every method's return value is serialised directly to the HTTP response body |

**Key difference between `@Controller` and `@RestController`:**

```java
// @Controller — returns a VIEW NAME (resolved by ViewResolver)
@Controller
public class PageController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("user", "Alice");
        return "home"; // resolves to templates/home.html (Thymeleaf)
    }
}

// @RestController — returns DATA serialised to JSON/XML
@RestController
public class ApiController {
    @GetMapping("/users/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id); // serialised to JSON automatically
    }
}
```

**Key difference between `@Service` and `@Repository`:**

```java
// @Repository — enables exception translation
@Repository
public class OrderRepository {
    // If a JPA/SQL exception is thrown, Spring wraps it in DataAccessException
    public Order findById(Long id) { ... }
}

// @Service — no exception translation, purely semantic
@Service
public class OrderService {
    public Order processOrder(Long id) { ... }
}
```

---

### 4.3 Dependency Injection Annotations

#### `@Autowired`

- Spring's own annotation. Can be applied to **constructor, setter, or field**.
- **Best practice**: constructor injection (immutable, testable, mandatory dependencies explicit).
- `required = false` makes the dependency optional.

```java
// Constructor injection (preferred)
@Service
public class OrderService {
    private final PaymentService paymentService;

    @Autowired  // optional since Spring 4.3 if only one constructor exists
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

#### `@Qualifier`

- Used alongside `@Autowired` to specify **which bean** to inject when multiple candidates of the same type exist.

```java
@Autowired
@Qualifier("paypalPaymentService")
private PaymentService paymentService;
```

#### `@Primary`

- Marks a bean as the **default candidate** when multiple beans of the same type exist — no `@Qualifier` needed at the injection point.
- `@Qualifier` always wins over `@Primary` if both are present.

```java
@Primary
@Bean
public PaymentService stripePaymentService() { return new StripePaymentService(); }

@Bean
public PaymentService paypalPaymentService() { return new PaypalPaymentService(); }
```

**`@Primary` vs `@Qualifier`:**

| | `@Primary` | `@Qualifier` |
|---|---|---|
| Where defined | On the bean definition | On the injection point |
| When to use | One "default" bean out of many | Precise selection at injection site |
| Overrides | `@Qualifier` overrides `@Primary` | — |

#### `@Resource` (JSR-250)

- Injects by **name first**, then by **type** (opposite of `@Autowired` which injects by type first).
- Part of the JSR-250 standard, not Spring-specific.

```java
@Resource(name = "paypalPaymentService")
private PaymentService paymentService;
```

#### `@Inject` (JSR-330)

- Equivalent to `@Autowired` but from the JSR-330 standard (`javax.inject`).
- Does **not** have a `required` attribute.
- Used when you want framework-agnostic code.

**`@Autowired` vs `@Inject` vs `@Resource`:**

| Feature | `@Autowired` | `@Inject` | `@Resource` |
|---|---|---|---|
| Standard | Spring | JSR-330 | JSR-250 |
| Injection strategy | By type | By type | By name → type |
| `required` attribute | Yes | No | No |
| `@Qualifier` compatible | Yes | Yes | No (use `name` attribute) |

#### `@Value`

- Injects scalar values from `application.properties`, environment variables, or SpEL expressions.

```java
@Value("${app.payment.timeout:30}")  // 30 is the default
private int paymentTimeout;

@Value("#{environment['java.version']}")  // SpEL
private String javaVersion;
```

#### `@ConfigurationProperties`

- Binds a **whole group** of properties to a POJO — type-safe, validated, IDE-friendly.

```java
@ConfigurationProperties(prefix = "app.payment")
@Component
public class PaymentProperties {
    private int timeout;
    private String currency;
    private boolean enabled;
    // getters + setters
}
```

```yaml
app:
  payment:
    timeout: 30
    currency: USD
    enabled: true
```

**`@Value` vs `@ConfigurationProperties`:**

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Granularity | Single property | Group of properties |
| Type safety | Limited | Full (supports nested objects, lists, maps) |
| Validation | None | Supports `@Validated` / JSR-380 |
| Relaxed binding | No | Yes (`camelCase`, `kebab-case`, `SNAKE_CASE` all work) |
| SpEL support | Yes | No |
| Best for | Simple one-off values | Feature/module configuration |

---

### 4.4 Configuration Annotations

#### `@Configuration`

- Marks a class as a source of bean definitions.
- Spring creates a CGLIB subclass of the `@Configuration` class so that `@Bean` methods calling other `@Bean` methods return the **same singleton** instance (not a new one on each call).

#### `@Bean`

- Declares a Spring bean produced by a method inside a `@Configuration` class.
- Bean name defaults to the method name; overridable with `name` attribute.

```java
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean("customDataSource")
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

**`@Component` vs `@Bean`:**

| | `@Component` | `@Bean` |
|---|---|---|
| Placed on | Class | Method inside `@Configuration` |
| Use when | You own the source code | Third-party class or custom instantiation logic needed |
| Scanning | Auto-detected via component scan | Must be inside a `@Configuration` class |
| CGLIB proxy | No | Yes (inside `@Configuration`) |

#### `@Import`

- Imports additional `@Configuration` classes, `ImportSelector`, or `ImportBeanDefinitionRegistrar` implementations.

```java
@Configuration
@Import({ DatabaseConfig.class, CacheConfig.class })
public class AppConfig { }
```

#### `@PropertySource`

- Loads additional `.properties` files into the `Environment`.

```java
@Configuration
@PropertySource("classpath:db-config.properties")
public class DbConfig { }
```

#### `@Profile`

- Registers a bean only when the specified Spring profile is active.

```java
@Bean
@Profile("prod")
public DataSource prodDataSource() { ... }

@Bean
@Profile("dev")
public DataSource devDataSource() { ... }
```

#### `@Conditional` and Friends

| Annotation | Condition |
|---|---|
| `@ConditionalOnClass` | Bean registered only if a class is on the classpath |
| `@ConditionalOnMissingClass` | Only if a class is NOT on the classpath |
| `@ConditionalOnBean` | Only if a specific bean already exists |
| `@ConditionalOnMissingBean` | Only if a specific bean does NOT exist |
| `@ConditionalOnProperty` | Only if a property has a specific value |
| `@ConditionalOnWebApplication` | Only in a web application context |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |

```java
@Bean
@ConditionalOnMissingBean(CacheManager.class)
public CacheManager simpleCacheManager() {
    return new ConcurrentMapCacheManager("orders", "products");
}
```

#### `@Lazy`

- Defers bean initialisation until first use.
- By default, Spring creates all singleton beans eagerly at startup.

```java
@Bean
@Lazy
public ExpensiveService expensiveService() { return new ExpensiveService(); }
```

---

### 4.5 Web / MVC Annotations

#### `@RequestMapping`

- Maps HTTP requests to handler methods. Can be placed on the class (base path) and method (sub-path).
- `method` attribute specifies HTTP verb. Without it, matches all methods.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @RequestMapping(method = RequestMethod.GET)
    public List<Order> getAll() { ... }
}
```

#### Shortcut Mappings

`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping` are composed annotations backed by `@RequestMapping` with a fixed `method`.

```java
@GetMapping("/{id}")           // GET /api/v1/orders/{id}
@PostMapping                   // POST /api/v1/orders
@PutMapping("/{id}")           // PUT /api/v1/orders/{id}
@DeleteMapping("/{id}")        // DELETE /api/v1/orders/{id}
@PatchMapping("/{id}/status")  // PATCH /api/v1/orders/{id}/status
```

#### Request Data Extraction

| Annotation | Extracts From | Notes |
|---|---|---|
| `@PathVariable` | URI template variable `/orders/{id}` | Required by default |
| `@RequestParam` | Query string `?page=1&size=10` | Can have `defaultValue`, `required=false` |
| `@RequestBody` | HTTP request body (JSON/XML) | Deserialised by `HttpMessageConverter` |
| `@RequestHeader` | HTTP header | `@RequestHeader("Authorization")` |
| `@CookieValue` | Cookie | `@CookieValue("JSESSIONID")` |
| `@ModelAttribute` | Form data / request params bound to a POJO | For MVC form submissions |

```java
@GetMapping("/{id}")
public ResponseEntity<OrderDTO> getOrder(
        @PathVariable Long id,
        @RequestParam(defaultValue = "false") boolean includeItems,
        @RequestHeader("X-Correlation-ID") String correlationId) {
    ...
}
```

**`@PathVariable` vs `@RequestParam`:**

```
GET /orders/123          → @PathVariable captures 123
GET /orders?id=123       → @RequestParam captures 123
```

#### `@ResponseBody`

- Tells Spring MVC to serialise the return value directly to the HTTP response body.
- `@RestController` = `@Controller` + `@ResponseBody` on every method.

#### `@ResponseStatus`

- Sets the HTTP status code for a handler method or exception handler.

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)  // returns 201
public OrderDTO createOrder(@RequestBody CreateOrderRequest request) { ... }
```

#### `@CrossOrigin`

- Configures CORS at controller or method level.

```java
@CrossOrigin(origins = "https://frontend.example.com", maxAge = 3600)
@RestController
public class OrderController { }
```

---

### 4.6 Exception Handling Annotations

#### `@ExceptionHandler`

- Handles exceptions thrown within the **same controller class**.

```java
@RestController
public class OrderController {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleOrderNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }
}
```

#### `@ControllerAdvice`

- Global exception handler for all controllers.
- Can also handle `@ModelAttribute` and `@InitBinder` globally.
- Returns a **view** by default (like a regular `@Controller`).

#### `@RestControllerAdvice`

- `@ControllerAdvice` + `@ResponseBody`.
- Returns data (JSON/XML) directly — correct choice for REST APIs.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", "Unexpected error occurred");
    }
}
```

**`@ControllerAdvice` vs `@RestControllerAdvice`:**

| | `@ControllerAdvice` | `@RestControllerAdvice` |
|---|---|---|
| Composed of | `@Component` + advice logic | `@ControllerAdvice` + `@ResponseBody` |
| Return type | View name or `ResponseEntity` | Directly serialised to response body |
| Use for | Traditional MVC apps | REST APIs |

---

### 4.7 Validation Annotations

#### `@Valid` (JSR-380 / Bean Validation)

- Triggers standard Bean Validation on the annotated parameter.
- Validates based on annotations like `@NotNull`, `@Size`, `@Min`, `@Email`.
- Cascades validation into nested objects.

#### `@Validated` (Spring)

- Spring's variant of `@Valid`.
- Supports **validation groups** — run different subsets of constraints.
- Can be applied at the **class level** to enable method-level validation via AOP.

```java
// @Valid — simple validation
@PostMapping
public ResponseEntity<OrderDTO> createOrder(@Valid @RequestBody CreateOrderRequest req) {
    ...
}

// @Validated — group-based validation
public interface OnCreate {}
public interface OnUpdate {}

public class ProductRequest {
    @NotNull(groups = OnCreate.class)
    private String name;

    @NotNull(groups = {OnCreate.class, OnUpdate.class})
    private BigDecimal price;
}

@PutMapping("/{id}")
public ResponseEntity<Void> update(
        @Validated(OnUpdate.class) @RequestBody ProductRequest req) { ... }
```

**`@Valid` vs `@Validated`:**

| | `@Valid` | `@Validated` |
|---|---|---|
| Origin | JSR-380 (javax/jakarta) | Spring (`org.springframework`) |
| Validation groups | Not supported | Supported |
| Class-level use | No | Yes (enables method validation AOP) |
| Cascade | Yes | Yes |

---

### 4.8 Transactional Annotations

#### `@Transactional`

Full details in `spring-boot/spring-transactions.md`, but key attributes:

| Attribute | Default | Description |
|---|---|---|
| `propagation` | `REQUIRED` | Transaction propagation behaviour |
| `isolation` | `DEFAULT` | Isolation level |
| `readOnly` | `false` | Hint to optimise read-only transactions |
| `rollbackFor` | Runtime exceptions | Which exceptions trigger rollback |
| `noRollbackFor` | — | Which exceptions do NOT trigger rollback |
| `timeout` | -1 (none) | Transaction timeout in seconds |

---

### 4.9 Caching Annotations

#### `@EnableCaching`

- Activates Spring's annotation-driven caching on a `@Configuration` class.

#### `@Cacheable`

- Caches the return value of a method. On subsequent calls with the same `key`, the cached value is returned without executing the method.

#### `@CachePut`

- **Always** executes the method AND updates the cache. Used to update the cache after a write.

#### `@CacheEvict`

- Removes entries from the cache. `allEntries = true` clears the whole cache.

#### `@Caching`

- Groups multiple caching annotations on one method.

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepo.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#result.id")
    public Product updateProduct(Product product) {
        return productRepo.save(product);
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepo.deleteById(id);
    }
}
```

**`@Cacheable` vs `@CachePut`:**

| | `@Cacheable` | `@CachePut` |
|---|---|---|
| Executes method? | Only if cache miss | Always |
| Use case | Read operations | Write operations that must refresh cache |

---

### 4.10 Scheduling & Async Annotations

#### `@EnableScheduling`

Placed on a `@Configuration` class to enable `@Scheduled` task detection.

#### `@Scheduled`

| Attribute | Example | Description |
|---|---|---|
| `fixedRate` | `fixedRate = 5000` | Run every N ms, regardless of previous execution time |
| `fixedDelay` | `fixedDelay = 5000` | Wait N ms AFTER previous execution completes |
| `cron` | `cron = "0 0 2 * * ?"` | Cron expression |
| `initialDelay` | `initialDelay = 10000` | Wait before first execution |

```java
@Scheduled(cron = "0 0 1 * * ?")  // 1 AM every day
public void generateDailyReport() { ... }

@Scheduled(fixedDelay = 30_000)   // 30 sec after last run completes
public void syncInventory() { ... }
```

**`fixedRate` vs `fixedDelay`:**

```
Task runs 10s, fixedRate=5000:  executions overlap (next fires 5s into task)
Task runs 10s, fixedDelay=5000: next fires 15s after start (10s run + 5s delay)
```

#### `@EnableAsync`

Enables Spring's async method execution on a `@Configuration` class.

#### `@Async`

- Makes a method execute asynchronously on a separate thread.
- Returns `void`, `Future<T>`, or `CompletableFuture<T>`.
- **Critical**: `@Async` does NOT work when called from within the same class (same self-invocation proxy limitation as `@Transactional`).

```java
@Async
public CompletableFuture<String> sendEmailAsync(String to) {
    // runs on a TaskExecutor thread, not the calling thread
    emailClient.send(to);
    return CompletableFuture.completedFuture("sent");
}
```

---

### 4.11 AOP Annotations

| Annotation | Description |
|---|---|
| `@Aspect` | Marks class as an aspect (contains advice) |
| `@Pointcut` | Defines a reusable pointcut expression |
| `@Before` | Runs before matched method |
| `@After` | Runs after matched method (always, like finally) |
| `@AfterReturning` | Runs after method returns successfully |
| `@AfterThrowing` | Runs after method throws exception |
| `@Around` | Wraps the method — can control invocation, modify args/return |

```java
@Aspect
@Component
public class AuditAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Around("serviceLayer()")
    public Object audit(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} took {}ms", pjp.getSignature(), elapsed);
        return result;
    }
}
```

**`@Before` vs `@Around`:**

| | `@Before` | `@Around` |
|---|---|---|
| Can modify return value | No | Yes |
| Can prevent method execution | No | Yes (don't call `pjp.proceed()`) |
| Can catch exceptions | No | Yes |
| Complexity | Simple | Full control |

---

### 4.12 Lifecycle Annotations

#### `@PostConstruct`

- Called **after** the bean is fully initialised (all dependencies injected).
- Part of JSR-250, not Spring-specific.
- Alternative to `InitializingBean.afterPropertiesSet()`.

#### `@PreDestroy`

- Called just before the bean is destroyed (container shutdown).
- Alternative to `DisposableBean.destroy()`.

```java
@Component
public class ConnectionPoolManager {

    @PostConstruct
    public void init() {
        // runs after all @Autowired fields are set
        pool.connect();
    }

    @PreDestroy
    public void cleanup() {
        // runs before bean is removed from context
        pool.close();
    }
}
```

---

### 4.13 Bean Scope Annotation

#### `@Scope`

```java
@Scope("prototype")           // new instance per injection
@Scope("request")             // one per HTTP request
@Scope("session")             // one per HTTP session
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
```

| Scope | Instances | Typical Use |
|---|---|---|
| `singleton` (default) | 1 per ApplicationContext | Stateless services |
| `prototype` | 1 per injection point | Stateful beans, command objects |
| `request` | 1 per HTTP request | Request-scoped state |
| `session` | 1 per HTTP session | User session state |
| `application` | 1 per ServletContext | Shared application-wide state |

---

### 4.14 Testing Annotations

| Annotation | Loads | Use Case |
|---|---|---|
| `@SpringBootTest` | Full ApplicationContext | Integration tests |
| `@WebMvcTest` | Only MVC layer (no service/repo beans) | Controller unit tests with MockMvc |
| `@DataJpaTest` | Only JPA layer (in-memory DB) | Repository tests |
| `@RestClientTest` | Only REST client components | RestTemplate/WebClient tests |
| `@MockBean` | Adds a Mockito mock to the ApplicationContext | Replace a bean with a mock |
| `@SpyBean` | Adds a Mockito spy (partial mock) | Spy on a real bean |

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;  // replaced with a mock in context

    @Test
    void getOrder_returnsOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(new OrderDTO(1L, "PENDING"));

        mockMvc.perform(get("/api/v1/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("PENDING"));
    }
}
```

**`@MockBean` vs `@Mock`:**

| | `@MockBean` | `@Mock` (Mockito) |
|---|---|---|
| Context | Spring ApplicationContext | Unit test, no Spring context |
| Replaces bean in context | Yes | No |
| Use in | `@SpringBootTest`, `@WebMvcTest` | Pure unit tests |

---

### 4.15 Security Annotations

| Annotation | Description |
|---|---|
| `@EnableWebSecurity` | Activates Spring Security's web security support |
| `@EnableMethodSecurity` | Enables `@PreAuthorize`, `@PostAuthorize`, `@Secured` on methods |
| `@PreAuthorize` | Checks authority expression **before** method executes |
| `@PostAuthorize` | Checks authority expression **after** method returns (can inspect return value) |
| `@Secured` | Restricts access by role names |
| `@RolesAllowed` | JSR-250 equivalent of `@Secured` |

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserDTO getUser(Long userId) { ... }

@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public Order getOrder(Long orderId) { ... }
```

**`@PreAuthorize` vs `@Secured`:**

| | `@PreAuthorize` | `@Secured` |
|---|---|---|
| Expression language | SpEL (full power) | Simple role strings |
| Check parameters | Yes (`#param`) | No |
| Check return value | No (use `@PostAuthorize`) | No |
| Flexibility | High | Low |

---

## 5. Configuration & Setup

```xml
<!-- Spring Boot Starter Web (includes MVC, validation, Tomcat) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Boot Starter Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Boot Starter Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Spring Boot Starter Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Spring Boot Starter Cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  cache:
    type: redis

server:
  port: 8080

logging:
  level:
    org.springframework: INFO
    com.example: DEBUG
```

---

## 6. Real-World Production Code Example

A complete e-commerce Order Service demonstrating most annotations together:

```java
// --- Entity ---
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String customerId;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @CreationTimestamp
    private LocalDateTime createdAt;

    // constructors, getters, setters
}

// --- Repository ---
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o WHERE o.customerId = :customerId AND o.status = :status")
    List<Order> findByCustomerAndStatus(
            @Param("customerId") String customerId,
            @Param("status") OrderStatus status);
}

// --- Configuration Properties ---
@ConfigurationProperties(prefix = "app.order")
@Component
public class OrderProperties {
    private int maxItemsPerOrder = 50;
    private String defaultCurrency = "USD";
    // getters + setters
}

// --- Service ---
@Service
@Slf4j
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final OrderProperties props;

    // Constructor injection — no @Autowired needed (single constructor)
    public OrderService(OrderRepository orderRepository,
                        InventoryClient inventoryClient,
                        OrderProperties props) {
        this.orderRepository = orderRepository;
        this.inventoryClient = inventoryClient;
        this.props = props;
    }

    @Cacheable(value = "orders", key = "#id")
    @Transactional(readOnly = true)
    public OrderDTO getOrder(Long id) {
        return orderRepository.findById(id)
                .map(OrderDTO::from)
                .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @CacheEvict(value = "pcorders", key = "#result.id")
    @Transactional
    public OrderDTO createOrder(CreateOrderRequest request) {
        if (request.getItems().size() > props.getMaxItemsPerOrder()) {
            throw new OrderLimitExceededException("Max items exceeded");
        }
        inventoryClient.reserveStock(request.getItems()); // external call
        Order order = new Order(request.getCustomerId(), OrderStatus.PENDING);
        Order saved = orderRepository.save(order);
        log.info("Order created: {}", saved.getId());
        return OrderDTO.from(saved);
    }

    @Async
    public CompletableFuture<Void> sendConfirmationEmailAsync(String customerId, Long orderId) {
        // runs on async thread pool, does not block calling thread
        emailService.send(customerId, "Order #" + orderId + " confirmed");
        return CompletableFuture.completedFuture(null);
    }
}

// --- Controller ---
@RestController
@RequestMapping("/api/v1/orders")
@Validated
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrder(id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('USER')")
    public OrderDTO createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.createOrder(request);
    }
}

// --- Global Exception Handler ---
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return new ErrorResponse("VALIDATION_FAILED", message);
    }
}

// --- AOP Aspect ---
@Aspect
@Component
@Slf4j
public class PerformanceAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("[PERF] {} took {}ms",
                    pjp.getSignature().toShortString(),
                    System.currentTimeMillis() - start);
        }
    }
}

// --- Scheduled Task ---
@Component
public class OrderCleanupJob {

    private final OrderRepository orderRepository;

    public OrderCleanupJob(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Scheduled(cron = "0 0 3 * * ?") // 3 AM daily
    @Transactional
    public void archiveStaleOrders() {
        orderRepository.archiveOrdersOlderThan(LocalDateTime.now().minusDays(90));
    }
}

// --- Application Entry Point ---
@SpringBootApplication
@EnableCaching
@EnableAsync
@EnableScheduling
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

---

## 7. Production Use-Cases & Best Practices

1. **Use constructor injection everywhere** — avoids null injection in tests, makes dependencies explicit, supports `final` fields.

2. **`@ConfigurationProperties` over `@Value` for groups of settings** — type-safe, validated, supports relaxed binding, IDE autocomplete with `spring-boot-configuration-processor`.

3. **`@RestControllerAdvice` as a single global handler** — centralise error handling rather than `@ExceptionHandler` in every controller; ensures consistent error response format.

4. **`@Transactional(readOnly = true)` on read queries** — tells Hibernate to skip dirty checking and flush; some DB drivers and proxies route these to read replicas.

5. **Never `@Async` call from the same class** — due to proxy limitations, the async advice is not applied. Always inject self or call from a different bean.

6. **Prefer `@PreAuthorize` with SpEL over `@Secured`** — `@PreAuthorize` supports owner checks like `#userId == authentication.principal.id`, which `@Secured` cannot do.

7. **Use `@ConditionalOnMissingBean` in auto-config classes** — lets users override your defaults by just defining their own bean, without touching your code.

8. **`@Cacheable` with explicit key expressions** — never rely on the default key (all parameters); use explicit SpEL keys (`key = "#id"`) to avoid key collisions and aid debugging.

---

## 8. Scenario Walkthroughs

### Scenario 1: `@Transactional` self-invocation failure

**Setup**: `OrderService.placeOrder()` is `@Transactional`. It internally calls `this.validateOrder()`, which is also `@Transactional(propagation = REQUIRES_NEW)`.

**What happens step-by-step**:
1. Spring creates a CGLIB proxy for `OrderService`.
2. External caller calls `proxy.placeOrder()` — Spring intercepts and starts Transaction T1.
3. Inside `placeOrder()`, `this.validateOrder()` is called — `this` is the **raw object**, not the proxy.
4. Spring does NOT intercept the `validateOrder()` call. `REQUIRES_NEW` is ignored.
5. `validateOrder()` executes within T1, not a new transaction.

**Fix**:
```java
// Option 1: Inject self as a proxy
@Service
public class OrderService {
    @Autowired
    private OrderService self; // Spring injects the proxy

    @Transactional
    public void placeOrder(...) {
        self.validateOrder(...); // now goes through the proxy
    }
}

// Option 2: Move validateOrder to a separate @Service
```

---

### Scenario 2: `@Cacheable` with stale data

**Setup**: `getProduct(Long id)` is `@Cacheable`. An admin updates the product price via `updateProduct(Product p)` — NOT annotated with `@CacheEvict` or `@CachePut`.

**What happens**:
1. First call to `getProduct(1L)` hits DB, result stored in cache.
2. Admin calls `updateProduct(...)` — DB is updated but cache is NOT invalidated.
3. Subsequent `getProduct(1L)` returns stale price from cache.

**Fix**:
```java
@CachePut(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepo.save(product); // updates DB AND refreshes cache
}
```

---

### Scenario 3: `@Async` method not executing asynchronously

**Setup**: User expects `sendEmailAsync()` to run in a background thread but observes it running synchronously.

**Root causes to check**:
1. `@EnableAsync` is missing on the `@Configuration` class.
2. `sendEmailAsync()` is called from within the same class (`this.sendEmailAsync()`).
3. The method is `private` — Spring cannot proxy private methods.
4. The class is not a Spring-managed bean (missing `@Service`/`@Component`).

```java
// Wrong — called from same class
@Service
public class NotificationService {
    @Async
    public void sendEmailAsync() { ... }

    public void processOrder() {
        this.sendEmailAsync(); // NOT async — bypasses proxy
    }
}

// Correct — inject the proxy
@Service
public class OrderService {
    @Autowired
    private NotificationService notificationService;

    public void processOrder() {
        notificationService.sendEmailAsync(); // async via proxy
    }
}
```

---

## 9. Common Pitfalls & Gotchas

| Pitfall | Why It Happens | Fix |
|---|---|---|
| `@Transactional` not applied | Self-invocation bypasses proxy | Inject self or extract to another bean |
| `@Async` runs synchronously | Self-invocation, missing `@EnableAsync`, or private method | Same as above; ensure `@EnableAsync` is present |
| `@Cacheable` caches `null` | Spring caches null return values by default | Set `unless = "#result == null"` |
| `@Value` resolves to literal string `${...}` | Missing `@PropertySource` or property not loaded | Ensure property file is on classpath / correct prefix |
| `@Component` vs `@Bean` confusion | Trying to use `@Component` on third-party class | Use `@Bean` inside `@Configuration` for third-party classes |
| `@Qualifier` on field vs `@Primary` precedence confusion | Both applied — which wins? | `@Qualifier` always wins over `@Primary` |
| `@Scheduled` tasks blocking each other | Default scheduler is single-threaded | Configure `TaskScheduler` with a thread pool |
| `@PostConstruct` called before `@Value` injection | If `@Value` uses a `static` field | Never use `@Value` with `static` fields |
| `@RestControllerAdvice` not catching exceptions | Exception thrown outside controller (e.g. in filter) | Use a custom `Filter` or `ErrorController` for filter-level exceptions |
| Bean created for every `@Configuration` method call | Not inside `@Configuration` but inside `@Component` | `@Bean` inside `@Component` does NOT get CGLIB enhancement — methods create new instances on each call |

---

## 10. Miscellaneous & Advanced Details

### `@Configuration` CGLIB Lite Mode (`proxyBeanMethods = false`)

```java
// Full @Configuration (default) — CGLIB proxied, @Bean calls return same singleton
@Configuration
public class AppConfig {
    @Bean
    public A a() { return new A(b()); } // b() returns the SAME singleton

    @Bean
    public B b() { return new B(); }
}

// Lite mode — no CGLIB proxy, slightly faster startup, BUT:
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean
    public A a() { return new A(b()); } // b() creates a NEW B — NOT the singleton!
}
```

Use `proxyBeanMethods = false` only if your `@Bean` methods do not cross-reference each other.

### `@SpringBootTest` vs `@WebMvcTest` vs `@DataJpaTest`

| | Loads | Speed | Use Case |
|---|---|---|---|
| `@SpringBootTest` | Full context | Slow | Full integration tests |
| `@WebMvcTest` | Web layer only | Fast | Controller tests with `MockMvc` |
| `@DataJpaTest` | JPA/DB layer only | Medium | Repository tests |

### Spring Boot 3.x changes (Jakarta EE)

- `javax.*` packages renamed to `jakarta.*` in Spring Boot 3+.
- `@PostConstruct` → `jakarta.annotation.PostConstruct`
- `@Valid` → `jakarta.validation.Valid`
- `@Resource` → `jakarta.annotation.Resource`

### Meta-annotations and Custom Composed Annotations

You can create your own composed annotations:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@RestController
@RequestMapping("/api/v1")
public @interface ApiController { }

// Usage
@ApiController
public class ProductController { ... }
```

---

## 11. Interview Questions — Standard

**Q1. What is the difference between `@Component`, `@Service`, and `@Repository`?**

All three are specialisations of `@Component` and all trigger component scanning to register a Spring bean. `@Service` is purely semantic — it marks the class as belonging to the service layer but adds no extra behaviour. `@Repository` additionally enables **exception translation**: Spring wraps persistence-layer exceptions (like JPA `PersistenceException` or JDBC `SQLException`) into Spring's `DataAccessException` hierarchy via `PersistenceExceptionTranslationPostProcessor`. This means calling code doesn't need to know the underlying data access technology. `@Controller` marks a class as a Spring MVC web controller and integrates with `DispatcherServlet`, while `@RestController` adds `@ResponseBody` to make every method return data instead of a view name.

---

**Q2. What does `@SpringBootApplication` consist of?**

It is a composed annotation of three annotations:
- `@SpringBootConfiguration`: marks the class as a `@Configuration` source.
- `@EnableAutoConfiguration`: triggers Spring Boot's auto-configuration mechanism by loading `AutoConfiguration.imports` and applying conditional beans.
- `@ComponentScan`: scans the package of the annotated class and all sub-packages for `@Component`-annotated classes.

You can replace it with all three explicitly if you need fine-grained control, e.g. to specify different `basePackages` for scanning.

---

**Q3. What is the difference between `@Controller` and `@RestController`?**

`@Controller` is a Spring MVC controller whose methods return **view names** resolved by a `ViewResolver` (e.g. Thymeleaf template names). To return data directly from a `@Controller` method, you must add `@ResponseBody`. `@RestController` is a composed annotation of `@Controller + @ResponseBody`, so every method in it automatically serialises its return value to the HTTP response body as JSON/XML. Use `@Controller` for traditional server-side rendered applications and `@RestController` for REST APIs.

---

**Q4. When would you use `@Qualifier` vs `@Primary`?**

Use `@Primary` on a bean definition to declare it as the default when multiple beans of the same type exist — injection points that don't specify a qualifier will receive this bean. Use `@Qualifier` at the injection point when you need a specific, non-default bean. If both are present, `@Qualifier` takes precedence over `@Primary`. A good pattern is `@Primary` for the production implementation and `@Qualifier` in tests or specific configurations that need an alternative.

---

**Q5. What is the difference between `@Value` and `@ConfigurationProperties`?**

`@Value` is used to inject a **single property** using a property placeholder `${...}` or SpEL `#{...}`. `@ConfigurationProperties` binds an **entire group of properties** (sharing a common prefix) to a typed POJO, which supports nested objects, lists, maps, relaxed binding (`camelCase`, `kebab-case`, `UPPER_CASE`), and JSR-380 validation via `@Validated`. `@ConfigurationProperties` is the preferred approach for feature configuration because it's type-safe and IDE-autocomplete-friendly.

---

**Q6. What is the difference between `@Autowired`, `@Resource`, and `@Inject`?**

`@Autowired` is Spring-specific and resolves by **type first**, then by name if there are multiple candidates (requires `@Qualifier` for disambiguation). `@Resource` (JSR-250) resolves by **name first**, then by type — no `@Qualifier` needed; use the `name` attribute. `@Inject` (JSR-330) is equivalent to `@Autowired` but lacks a `required` attribute. Use `@Autowired` for Spring applications, `@Resource` when you want name-based injection, and `@Inject` for framework-agnostic code.

---

**Q7. What is the role of `@ControllerAdvice`?**

`@ControllerAdvice` is a specialisation of `@Component` that declares global exception handlers (via `@ExceptionHandler`), global model attributes (via `@ModelAttribute`), and global binder initialization (via `@InitBinder`) applicable to all controllers. By default it applies to all `@Controller` classes. You can limit scope using `basePackages`, `assignableTypes`, or `annotations` attributes. For REST APIs, use `@RestControllerAdvice` which adds `@ResponseBody` so handlers return data directly.

---

**Q8. What is the difference between `@Valid` and `@Validated`?**

`@Valid` is the standard JSR-380 annotation that triggers Bean Validation on the annotated parameter and supports cascaded validation of nested objects. `@Validated` is Spring's version that additionally supports **validation groups** — you can run different subsets of constraints in different contexts (e.g. `OnCreate` vs `OnUpdate`). `@Validated` at the class level also activates Spring AOP-based method validation for non-web methods (not just controller request bodies).

---

**Q9. What does `@PostConstruct` do and when is it called?**

`@PostConstruct` marks a method to be called after the bean is fully initialised — meaning after the constructor runs AND after all `@Autowired` dependencies are injected. It is called before the bean is made available to other beans. It is part of JSR-250 and is processed by `CommonAnnotationBeanPostProcessor`. The analogous `@Bean(initMethod = "init")` attribute achieves the same result without requiring modification of the class.

---

**Q10. What happens when you annotate a class method with `@Async` and call it from within the same class?**

The call bypasses Spring's AOP proxy because `this.method()` refers to the raw object, not the proxy. The `@Async` advice is never applied, so the method executes synchronously on the calling thread. The fix is to inject the bean into itself (constructor circular dependency can be avoided with `@Lazy`) or extract the async method to a separate Spring bean.

---

**Q11. What does `@Scope("prototype")` mean and when should you use it?**

It means Spring creates a **new instance of the bean every time it is requested** from the container — either via `getBean()` or injection. Unlike singletons, prototype beans are **not managed by Spring after creation** — Spring does not call `@PreDestroy` on them. Use prototype scope for stateful beans, command/request objects, or beans that hold user-specific state that must not be shared across threads or requests.

---

**Q12. What is the difference between `@Cacheable` and `@CachePut`?**

`@Cacheable` **skips method execution** on a cache hit and returns the cached value. If there is a cache miss, it runs the method and stores the result. It is used for read operations. `@CachePut` **always runs the method** regardless of cache state, and stores/updates the result in the cache after execution. It is used for write operations where the cache must be kept in sync with the updated data.

---

## 12. Interview Questions — Tricky / Scenario-Based

**Q1. Can you have two beans of the same type where neither is `@Primary`? What happens when you inject that type?**

Yes — Spring will throw a `NoUniqueBeanDefinitionException` at startup: *"expected single matching bean but found 2"*. To fix it, you must either mark one with `@Primary`, use `@Qualifier` at the injection point to specify which one, or inject a `List<PaymentService>` / `Map<String, PaymentService>` to receive all beans of that type at once.

```java
@Autowired
private List<PaymentService> paymentServices; // receives both beans
```

---

**Q2. Why does placing `@Transactional` on a `private` method have no effect?**

Spring's transaction support (and most AOP) works through **proxy objects**. CGLIB proxies can only intercept `public` and `protected` methods (and technically package-private in some cases). A `private` method is called directly on the real object — the proxy has no way to intercept it. The `@Transactional` annotation is effectively ignored. Always place `@Transactional` on `public` methods.

---

**Q3. A `@Scheduled` task is supposed to run every 5 seconds but is sometimes skipping executions. What is likely happening?**

Spring's default task scheduler is single-threaded. If the task takes longer than 5 seconds (with `fixedRate`), the scheduler queues up the next execution but cannot run it until the current one finishes — effectively skipping or delaying it. **Fix**:

```java
@Configuration
@EnableScheduling
public class SchedulerConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

---

**Q4. What is the difference between `@Configuration` and `@Component` when both contain `@Bean` methods?**

Inside a `@Configuration` class, Spring wraps it with a CGLIB subclass. When one `@Bean` method calls another, the proxy intercepts the call and returns the **existing singleton** from the context. Inside a `@Component` class, there is no CGLIB wrapping. Each direct method call creates a **new instance** — the singleton contract is broken. This is a subtle but critical distinction in production code that has caused real bugs.

---

**Q5. You annotate a method with `@Cacheable` and notice that calling it with `null` as an argument throws a `NullPointerException` in the cache key generation. Why?**

Spring's default key generation uses `SimpleKeyGenerator`, which wraps parameters in a `SimpleKey`. If a parameter is `null`, and you use it directly in a SpEL key expression like `key = "#id"`, the resulting cache key is the null value itself — some cache backends (e.g. older Redis configurations) cannot store null keys and throw NPE. **Fix**: add a null check in the key expression or add a condition:

```java
@Cacheable(value = "orders", key = "#id", condition = "#id != null")
public Order getOrder(Long id) { ... }
```

---

**Q6. Can `@SpringBootTest` and `@WebMvcTest` be used in the same test class?**

No — they load different application contexts. `@SpringBootTest` loads the full context; `@WebMvcTest` loads only the web layer (controllers, filters, converters). Combining them will result in a conflict or `@WebMvcTest` being ignored. Use `@SpringBootTest(webEnvironment = MOCK)` with `AutoConfigureMockMvc` for full-context MockMvc tests, or `@WebMvcTest` with `@MockBean` for lightweight controller-only tests.

---

**Q7. `@Profile("prod")` is set on a bean but the bean is still created in a dev environment. What could be wrong?**

Several causes:
1. `spring.profiles.active` is not set — the default profile is active and `@Profile` with named profiles is skipped only if that profile is NOT active. But if no profile is active, Spring might still create the bean depending on how `@Profile` is used. Actually, a bean with `@Profile("prod")` is created **only** when "prod" is in the active profiles.
2. The bean is being created via `@Import` or explicit `new`, bypassing the profile check.
3. The profile is set correctly but the test class uses `@SpringBootTest` which loads a different set of properties, ignoring the expected profile.
4. A `@ConfigurationProperties` class annotated with both `@Component` and `@Profile` may be picked up through a different path.

---

**Q8. You use `@MockBean` in a `@SpringBootTest` and notice the test suite gets significantly slower each time you add a new `@MockBean`. Why?**

Each unique combination of `@MockBean` beans causes Spring to create a **new ApplicationContext** — the context caching mechanism (`@DirtiesContext`) is effectively triggered. The more `@MockBean` annotations with different bean types, the more unique contexts are created. **Fix**: centralise mocks using a shared base test class or a shared Spring test configuration, so all tests in the suite share a single context.

---

**Q9. What happens if you annotate a `@Bean` method inside a `@Configuration(proxyBeanMethods = false)` class and that method calls another `@Bean` method in the same class?**

With `proxyBeanMethods = false` (lite mode), the class is NOT enhanced with CGLIB. When `beanA()` calls `beanB()`, it's a plain Java method call — Spring does not intercept it. A **new instance** of `beanB` is created each time, breaking the singleton contract. This is the key danger of lite mode: use it only when `@Bean` methods are fully independent.

---

**Q10. `@Async` is configured but tasks are not running concurrently — they are being queued. Why?**

The default `TaskExecutor` for `@Async` is `SimpleAsyncTaskExecutor`, which creates a new thread for every task with no upper bound by default. However, if a custom `TaskExecutor` is configured with a fixed pool size and `LinkedBlockingQueue` (unbounded queue), tasks pile up in the queue when all threads are busy. Increase pool size or configure a bounded queue with an appropriate rejection policy:

```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(50);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

---

## 13. Quick Revision Summary

- **`@SpringBootApplication`** = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` — the single entry point annotation.
- **`@RestController`** = `@Controller` + `@ResponseBody` — all methods return data, not view names.
- **`@Repository`** is the only stereotype that adds real behaviour: **exception translation** of persistence exceptions to `DataAccessException`.
- **`@Qualifier` beats `@Primary`** — when both are present, `@Qualifier` at the injection site wins.
- **`@Value`** for single properties; **`@ConfigurationProperties`** for groups — type-safe, validated, relaxed binding.
- **`@Valid`** triggers standard Bean Validation; **`@Validated`** additionally supports **validation groups** and class-level method validation.
- **`@ControllerAdvice`** returns views; **`@RestControllerAdvice`** returns data — always use `@RestControllerAdvice` for REST APIs.
- **`@Cacheable`** skips method on cache hit; **`@CachePut`** always runs method and updates cache.
- **`fixedRate`** fires every N ms regardless of task duration; **`fixedDelay`** waits N ms after task completes.
- **Self-invocation breaks `@Transactional`, `@Async`, `@Cacheable`** — all rely on Spring's proxy; `this.method()` bypasses it.
- **`@Configuration` CGLIB enhancement** ensures `@Bean` cross-calls return the same singleton; `@Component` does not — avoid cross-calling `@Bean` methods inside `@Component`.
- **`@MockBean`** replaces a bean in the Spring context (integration tests); `@Mock` is Mockito-only with no Spring context.
