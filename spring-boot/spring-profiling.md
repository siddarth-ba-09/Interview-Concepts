# Spring Profiles

> Control which beans, configurations, and properties are active based on the environment — dev, test, staging, prod — without changing a single line of application code.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts & Sub-Topics](#2-core-concepts--sub-topics)
3. [How It Works Internally](#3-how-it-works-internally)
4. [Key APIs — Annotations / Classes / Interfaces](#4-key-apis--annotations--classes--interfaces)
5. [Configuration & Setup](#5-configuration--setup)
6. [Real-World Production Code Example](#6-real-world-production-code-example)
7. [Production Use-Cases & Best Practices](#7-production-use-cases--best-practices)
8. [Scenario Walkthroughs](#8-scenario-walkthroughs)
9. [Common Pitfalls & Gotchas](#9-common-pitfalls--gotchas)
10. [Miscellaneous & Advanced Details](#10-miscellaneous--advanced-details)
11. [Interview Questions — Standard](#11-interview-questions--standard)
12. [Interview Questions — Tricky / Scenario-Based](#12-interview-questions--tricky--scenario-based)
13. [Quick Revision Summary](#13-quick-revision-summary)

---

## 1. Overview

**Spring Profiles** provide a way to segregate parts of your application configuration and make it active only in certain environments.

**Problem it solves:** Real applications behave differently in different environments:
- `dev` → in-memory H2 database, verbose logging, no auth
- `test` → Testcontainers or embedded DB, fast startup
- `staging` → real DB on a staging server, limited external service calls
- `prod` → production DB, strict security, Datadog/CloudWatch metrics

Without profiles, you'd use `if/else` checks on environment variables throughout your code — ugly, brittle, and error-prone.

**With profiles:** You declaratively annotate beans, properties files, and configuration classes with a profile name. Spring activates only the right set at runtime.

**Where it fits:**
- Part of the **Spring Core** `Environment` abstraction (available since Spring 3.1)
- Deeply integrated with Spring Boot's externalized configuration system
- Works alongside `@Conditional`, `@ConfigurationProperties`, and property sources

---

## 2. Core Concepts & Sub-Topics

### 2.1 The `@Profile` Annotation

`@Profile` on a `@Bean`, `@Component`, or `@Configuration` class means: *"only register this bean when the named profile is active."*

```java
@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // returns a HikariCP pool connected to RDS
    }
}
```

### 2.2 Active Profiles

An **active profile** is a profile that Spring has been told to use for the current run. You set it through any of these mechanisms (listed in ascending priority — later overrides earlier):

| Priority | Mechanism | Example |
|---|---|---|
| 1 (lowest) | `spring.profiles.default` in `application.properties` | `spring.profiles.default=dev` |
| 2 | `SpringApplication.setDefaultProfiles()` | programmatic |
| 3 | `application.properties` / `application.yml` | `spring.profiles.active=prod` |
| 4 | OS environment variable | `SPRING_PROFILES_ACTIVE=prod` |
| 5 | JVM system property | `-Dspring.profiles.active=prod` |
| 6 | `SpringApplication.setAdditionalProfiles()` | programmatic |
| 7 (highest) | Command-line argument | `--spring.profiles.active=prod` |

Multiple active profiles are comma-separated: `spring.profiles.active=prod,metrics,featureflagA`

### 2.3 Default Profile

If no profile is explicitly activated, Spring uses beans and config marked with `@Profile("default")` (and the `application.properties` defaults). You can change the default profile name:

```properties
spring.profiles.default=local
```

### 2.4 Profile-Specific Property Files

Spring Boot automatically loads `application-{profile}.properties` (or `.yml`) when that profile is active.

```
src/main/resources/
├── application.properties          ← always loaded (base)
├── application-dev.properties      ← loaded when 'dev' is active
├── application-prod.properties     ← loaded when 'prod' is active
└── application-test.properties     ← loaded when 'test' is active
```

Profile-specific files **override** the base `application.properties` for any key they define.

### 2.5 Profile Expressions (Spring 5.1+)

`@Profile` accepts logical expressions, not just plain names:

| Expression | Meaning |
|---|---|
| `"prod"` | Active if `prod` is active |
| `"!dev"` | Active if `dev` is NOT active |
| `"prod & eu"` | Active if BOTH `prod` AND `eu` are active |
| `"prod \| staging"` | Active if EITHER `prod` OR `staging` is active |
| `"(prod \| staging) & !test"` | Compound expression |

```java
@Component
@Profile("prod & eu")
public class EuGdprAuditService implements AuditService { }

@Component
@Profile("!prod")
public class NoOpAuditService implements AuditService { }
```

### 2.6 Profile Groups (Spring Boot 2.4+)

Group multiple profiles under a single alias:

```properties
# application.properties
spring.profiles.group.production=prod,actuator-secure,metrics
spring.profiles.group.local=dev,h2-console,swagger
```

Activating `production` automatically activates `prod`, `actuator-secure`, and `metrics`. This prevents forgetting to activate companion profiles.

### 2.7 Multi-Document YAML (Spring Boot 2.4+)

In a single `application.yml`, use `---` to separate documents with profile conditions:

```yaml
# Base config — always active
spring:
  application:
    name: order-service

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:devdb

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-host:5432/orders
```

> **Note:** Prior to Spring Boot 2.4, you used `spring.profiles:` instead of `spring.config.activate.on-profile:`. The old syntax is deprecated.

### 2.8 Programmatic Profile Activation

```java
@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(OrderApplication.class);
        app.setAdditionalProfiles("metrics");   // always add 'metrics' profile
        app.run(args);
    }
}
```

Or in a test:

```java
@SpringBootTest
@ActiveProfiles({"test", "h2"})
class OrderServiceTest { }
```

### 2.9 Profiles and `@ConfigurationProperties`

Profile-specific properties are automatically injected into `@ConfigurationProperties` beans without any extra work — the right values are bound based on the active profile.

```yaml
# application-prod.yml
payment:
  gateway:
    url: https://api.stripe.com/v1
    timeout: 5000

# application-dev.yml
payment:
  gateway:
    url: http://localhost:9090/mock
    timeout: 30000
```

```java
@ConfigurationProperties(prefix = "payment.gateway")
@Component
public class PaymentGatewayProperties {
    private String url;
    private int timeout;
    // getters/setters
}
```

---

## 3. How It Works Internally

### Step-by-Step: How Spring Resolves Profiles

```
1. SpringApplication.run() starts
        ↓
2. Environment is prepared
   → PropertySources are loaded (application.properties, env vars, CLI args)
   → spring.profiles.active is read from all sources (higher priority wins)
   → Active profiles are set on the ConfigurableEnvironment
        ↓
3. ApplicationContext is created
        ↓
4. @Configuration classes are processed by ConfigurationClassPostProcessor
   → For each @Bean / @Component / @Configuration:
       → Check if @Profile is present
       → Evaluate profile condition against Environment.getActiveProfiles()
       → If condition passes → register bean definition
       → If condition fails → SKIP (bean definition never enters the context)
        ↓
5. Profile-specific property files are loaded
   → application-{profile}.properties overrides base application.properties
        ↓
6. Context is refreshed — only profile-matched beans are instantiated
```

### Under the Hood: `@Profile` is `@Conditional`

`@Profile` is implemented as a `@Conditional`:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(ProfileCondition.class)   // ← this is the key
public @interface Profile {
    String[] value();
}
```

`ProfileCondition` implements `Condition` and evaluates `context.getEnvironment().acceptsProfiles(...)`.

```java
class ProfileCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> attrs =
            metadata.getAllAnnotationAttributes(Profile.class.getName());
        if (attrs != null) {
            for (Object value : attrs.get("value")) {
                if (context.getEnvironment().acceptsProfiles(Profiles.of((String[]) value))) {
                    return true;
                }
            }
            return false;
        }
        return true;
    }
}
```

### Profile Precedence in Property Files

```
application.properties          (base)
      ↓ overridden by
application-{profile1}.properties
      ↓ overridden by
application-{profile2}.properties
      ↓ overridden by
CLI args / env vars
```

When multiple profiles are active, the **last one in `spring.profiles.active`** wins for overlapping keys.

---

## 4. Key APIs — Annotations / Classes / Interfaces

| Name | Type | Purpose | Key Attributes |
|---|---|---|---|
| `@Profile` | Annotation | Marks a bean/config as active only for specific profiles | `value` — profile names or expressions |
| `@ActiveProfiles` | Annotation | Sets active profiles in a test class | `value` — profile names |
| `Environment` | Interface | Programmatic access to active profiles and properties | `getActiveProfiles()`, `getDefaultProfiles()`, `acceptsProfiles()` |
| `ConfigurableEnvironment` | Interface | Mutate active profiles programmatically | `setActiveProfiles()`, `addActiveProfile()` |
| `Profiles` | Class (Spring 5.1) | Factory for profile expressions (`Profiles.of("prod & !test")`) | `of(String... expressions)` |
| `ProfileCondition` | Class (internal) | `Condition` that backs `@Profile` | — |
| `SpringApplication` | Class | Entry point; `setAdditionalProfiles()`, `setDefaultProperties()` | `setAdditionalProfiles(String...)` |
| `spring.profiles.active` | Property | Comma-separated list of active profiles | — |
| `spring.profiles.default` | Property | Default profile if none active | — |
| `spring.profiles.group.*` | Property | Groups multiple profiles under an alias | `spring.profiles.group.prod=prod,metrics` |
| `spring.config.activate.on-profile` | YAML property | Activates a YAML document block for a profile | Boot 2.4+ |

---

## 5. Configuration & Setup

### Maven Dependency

No extra dependency needed — profiles are part of `spring-context` (included via any Spring Boot starter).

### application.properties / application.yml

```properties
# application.properties
spring.application.name=order-service

# Activate dev profile by default during local development
# Override via env var or CLI arg in CI/CD and production
spring.profiles.active=dev
```

```yaml
# application.yml — recommended multi-document approach (Boot 2.4+)
spring:
  application:
    name: order-service

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  h2:
    console:
      enabled: true
logging:
  level:
    root: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/orders
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate
logging:
  level:
    root: WARN
```

### Profile Groups (application.properties)

```properties
spring.profiles.group.local=dev,h2-console,swagger-ui
spring.profiles.group.production=prod,actuator-secure,newrelic
spring.profiles.group.ci=test,h2
```

### Activating via Environment Variable (Docker / Kubernetes)

```dockerfile
# Dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
```

```yaml
# Kubernetes Deployment
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"
```

### Activating via CLI

```bash
java -jar order-service.jar --spring.profiles.active=prod
java -Dspring.profiles.active=prod -jar order-service.jar
```

---

## 6. Real-World Production Code Example

**Domain:** E-commerce order service with different infrastructure configs per environment.

### Project Structure

```
src/main/
├── java/com/example/orders/
│   ├── OrderApplication.java
│   ├── config/
│   │   ├── DataSourceConfig.java        ← profile-switched datasource
│   │   ├── SecurityConfig.java          ← profile-switched security
│   │   ├── MessagingConfig.java         ← profile-switched messaging
│   │   └── SwaggerConfig.java           ← dev-only Swagger
│   ├── service/
│   │   ├── NotificationService.java     ← interface
│   │   ├── EmailNotificationService.java ← prod impl
│   │   └── LogNotificationService.java  ← dev/test impl
│   └── payment/
│       ├── PaymentGateway.java          ← interface
│       ├── StripePaymentGateway.java    ← prod
│       └── MockPaymentGateway.java      ← dev/test
└── resources/
    ├── application.yml
    ├── application-dev.yml
    ├── application-prod.yml
    └── application-test.yml
```

### application.yml

```yaml
spring:
  application:
    name: order-service
  profiles:
    group:
      local: dev,swagger
      production: prod,actuator-secure
      ci: test,h2
```

### application-dev.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:orders_dev
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  h2:
    console:
      enabled: true

logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
```

### application-prod.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/orders
    username: ${DB_USER}
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    root: WARN
    com.example: INFO
```

### DataSource Configuration (profile-switched beans)

```java
// config/DataSourceConfig.java
@Configuration
public class DataSourceConfig {

    // Separate @Bean methods with @Profile keep config clean and testable

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .setName("orders_dev")
                .build();
    }

    @Bean
    @Profile("test")
    public DataSource testDataSource() {
        // Testcontainers provides the JDBC URL via TC: prefix
        return DataSourceBuilder.create()
                .url("jdbc:tc:postgresql:15:///orders_test")
                .driverClassName("org.testcontainers.jdbc.ContainerDatabaseDriver")
                .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource(
            @Value("${spring.datasource.url}") String url,
            @Value("${spring.datasource.username}") String username,
            @Value("${spring.datasource.password}") String password) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }
}
```

### Payment Gateway — Profile-Driven Strategy

```java
// Interface
public interface PaymentGateway {
    PaymentResult charge(String orderId, BigDecimal amount);
}

// Production implementation
@Component
@Profile("prod")
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(String orderId, BigDecimal amount) {
        // real Stripe API call
        System.out.println("Charging via Stripe: " + amount);
        return PaymentResult.success("stripe-txn-" + UUID.randomUUID());
    }
}

// Dev/test stub — fast, no external calls
@Component
@Profile("!prod")   // active for dev, test, staging — anything that is not prod
public class MockPaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(String orderId, BigDecimal amount) {
        System.out.println("[MOCK] Charging: " + amount + " for order " + orderId);
        return PaymentResult.success("mock-txn-" + orderId);
    }
}
```

### Notification Service — Profile-Switched Bean

```java
// Production: sends real emails via SES
@Service
@Profile("prod")
public class EmailNotificationService implements NotificationService {
    @Override
    public void notify(String userId, String message) {
        // AWS SES API call
        System.out.println("Sending email to userId=" + userId);
    }
}

// Dev/test: just logs — no external calls, no AWS credentials needed
@Service
@Profile("!prod")
public class LogNotificationService implements NotificationService {
    private static final Logger log = LoggerFactory.getLogger(LogNotificationService.class);
    @Override
    public void notify(String userId, String message) {
        log.info("[NOTIFICATION] userId={} message={}", userId, message);
    }
}
```

### Swagger — Dev Only

```java
@Configuration
@Profile("dev | swagger")   // active for dev or if 'swagger' is explicitly activated
public class SwaggerConfig {
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .info(new Info().title("Order Service API").version("1.0"));
    }
}
```

### Security — Profile-Switched

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    @Profile("dev")
    public SecurityFilterChain devSecurity(HttpSecurity http) throws Exception {
        // Dev: permit all, no CSRF, h2-console access
        http.csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .headers(headers -> headers.frameOptions(fo -> fo.disable())); // h2-console needs this
        return http.build();
    }

    @Bean
    @Profile("prod")
    public SecurityFilterChain prodSecurity(HttpSecurity http) throws Exception {
        // Prod: enforce JWT, strict CSRF, HTTPS only
        http.authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

### OrderService (no profile awareness needed)

```java
@Service
public class OrderService {

    // Spring injects the right implementation based on active profile
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    public OrderService(PaymentGateway paymentGateway,
                        NotificationService notificationService) {
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }

    public Order placeOrder(String userId, BigDecimal amount) {
        Order order = new Order(UUID.randomUUID().toString(), userId, amount);
        PaymentResult result = paymentGateway.charge(order.getId(), amount);
        order.markPaid(result.getTransactionId());
        notificationService.notify(userId, "Order " + order.getId() + " placed");
        return order;
    }
}
```

### Tests

```java
@SpringBootTest
@ActiveProfiles("test")      // activates test datasource + mock payment gateway
class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Test
    void placeOrder_shouldSucceed() {
        Order order = orderService.placeOrder("user-1", new BigDecimal("99.99"));
        assertThat(order.isPaid()).isTrue();
        assertThat(order.getTransactionId()).startsWith("mock-txn-");
    }
}
```

---

## 7. Production Use-Cases & Best Practices

### 1. Environment-Specific Infrastructure (Databases, Message Queues)
Use profile-switched `@Bean` methods for `DataSource`, `ConnectionFactory`, `RedisTemplate`. Never hardcode connection strings — use environment variables injected into profile-specific `application-{profile}.yml`.

### 2. Stubbing External Services in Non-Prod
Use `@Profile("!prod")` for mock/stub implementations of email, SMS, payment, and analytics services. This:
- Eliminates flaky tests due to external dependencies
- Prevents accidental charges in dev/staging
- Speeds up local startup

### 3. Enabling/Disabling Feature Flags via Profiles
```properties
# Activate 'feature-new-checkout' only in staging and prod A/B group
spring.profiles.active=prod,feature-new-checkout
```
Combine with `@Profile("feature-new-checkout")` to gate entire beans or config classes.

### 4. Profile Groups to Avoid Profile Proliferation
Instead of activating 5 profiles manually, define groups in `application.properties` and activate the group alias. Reduces human error in deployments.

### 5. Security Configuration Per Environment
As shown in the code example — dev security is permissive (h2-console, no auth), prod enforces OAuth2/JWT. Keep these in separate `@Bean` methods (each `@Profile`-gated) rather than one giant conditional block.

### 6. Logging Verbosity Per Environment
Profile-specific `application-{profile}.yml` files set log levels:
- `dev` → `DEBUG` for `com.example.*`
- `staging` → `INFO`
- `prod` → `WARN` (avoid log noise and PII leakage)

### 7. Actuator Endpoint Exposure
```yaml
# application-prod.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info   # restricted in prod

# application-dev.yml
management:
  endpoints:
    web:
      exposure:
        include: "*"           # all endpoints in dev
```

### 8. CI/CD Pipeline Best Practice
Set `SPRING_PROFILES_ACTIVE` as an environment variable in your CI/CD pipeline per stage. Never commit `spring.profiles.active=prod` to source control — it should always be injected at runtime.

---

## 8. Scenario Walkthroughs

### Scenario 1: Wrong Bean Injected Because Active Profile Is Unexpected

**Setup:**
- `StripePaymentGateway` annotated with `@Profile("prod")`
- `MockPaymentGateway` annotated with `@Profile("!prod")`
- Developer runs tests locally with `spring.profiles.active=prod` in their local `application.properties`

**What happens:**
- `StripePaymentGateway` is registered
- Test tries to make a real Stripe API call
- Test fails with `401 Unauthorized` (no API key in test env)
- Developer is confused: "It works on CI"

**Root cause:** Local `application.properties` has `spring.profiles.active=prod` committed.

**Fix:**
1. Never commit `spring.profiles.active` to `application.properties` (or set it to `dev` as the safe default)
2. In tests, always annotate with `@ActiveProfiles("test")` — this overrides everything
3. CI sets `SPRING_PROFILES_ACTIVE=test` as environment variable

**Lesson:** `@ActiveProfiles` in test annotations is the most reliable way to control test profiles. Don't rely on `application.properties` for test profile selection.

---

### Scenario 2: Profile-Specific Property Not Overriding Base Property

**Setup:**
```properties
# application.properties
payment.timeout=5000

# application-dev.properties
payment.timeout=30000
```
Developer activates `dev` profile but `payment.timeout` is still `5000`.

**Investigation:**

```java
@Component
public class DiagnosticRunner implements ApplicationRunner {
    @Autowired
    private Environment env;

    @Override
    public void run(ApplicationArguments args) {
        System.out.println("Active profiles: " +
            Arrays.toString(env.getActiveProfiles()));
        System.out.println("payment.timeout: " +
            env.getProperty("payment.timeout"));
    }
}
```

**Root cause options:**
1. Profile isn't actually active — check `env.getActiveProfiles()`
2. `application-dev.yml` exists alongside `application-dev.properties` — Spring Boot 2.4+ uses only one per profile; `.yml` takes precedence
3. A higher-priority source (env var, CLI arg) is overriding it

**Lesson:** Always verify active profiles at startup via logs or Actuator `/actuator/env`. The `Environment` abstraction is your source of truth.

---

### Scenario 3: NoUniqueBeanDefinitionException with Profiles

**Setup:**
```java
@Component
@Profile("prod")
public class StripePaymentGateway implements PaymentGateway { }

@Component
@Profile("staging")
public class BraintreePaymentGateway implements PaymentGateway { }
```

Developer accidentally activates both `prod` and `staging`:
```
spring.profiles.active=prod,staging
```

**What happens:**
```
NoUniqueBeanDefinitionException: expected single matching bean but found 2:
  stripePaymentGateway, braintreePaymentGateway
```

**Fix options:**
1. Use mutually exclusive profiles (`prod & !staging`, `staging & !prod`)
2. Add `@Primary` to the preferred implementation
3. Use `@Qualifier` at the injection point
4. Restructure: use a single `@Profile("prod | staging")` bean + config properties to select vendor

**Lesson:** Profile names should be designed so mutually exclusive environments can't be simultaneously active. Profile groups help enforce this.

---

## 9. Common Pitfalls & Gotchas

### 1. `@Profile` on `@Bean` inside a `@Configuration` class without `@Profile` on the class
**Problem:** The `@Configuration` class is always instantiated even if the `@Bean` method's profile is inactive. This can cause issues if the config class itself has expensive field injection.
**Fix:** Put `@Profile` on both the class and the bean method, or split into separate config classes.

---

### 2. `spring.profiles.active` in `application.properties` Is Committed to Git
**Problem:** Developers commit `spring.profiles.active=dev`, which gets used in CI and accidentally in production builds.
**Fix:** Set `spring.profiles.active` only via environment variables or CLI args in non-local environments. Optionally, `.gitignore` a local `application-local.properties`.

---

### 3. Profile Not Active Because of Typo / Case Sensitivity
**Problem:** Profile names are case-sensitive. `@Profile("Prod")` won't match `spring.profiles.active=prod`.
**Fix:** Use lowercase, kebab-case profile names consistently: `dev`, `prod`, `test`, `staging`.

---

### 4. `@ActiveProfiles` in Test Not Overriding `application.properties`
**Problem:** Developer expects `@ActiveProfiles("test")` to override `spring.profiles.active=dev` in `application.properties`, but it doesn't seem to work.
**Fix:** It does work — `@ActiveProfiles` replaces (not adds to) the active profiles unless you also use `inheritProfiles = false`. The real issue is usually a `@TestPropertySource` or env variable override.

---

### 5. Profile-Specific YAML Multi-Document: Old vs New Syntax
**Problem:** Code using Spring Boot 2.4+ but still using old `spring.profiles:` syntax in YAML multi-documents:
```yaml
# DEPRECATED (Boot 2.3 and earlier)
spring:
  profiles: prod
```
**Fix:**
```yaml
# Boot 2.4+
spring:
  config:
    activate:
      on-profile: prod
```

---

### 6. Prototype-Scoped Bean with `@Profile` Not Switching
**Problem:** Developer has a prototype bean `@Profile("prod")` and a `@Profile("dev")` version. The wrong one is always returned.
**Cause:** They're injecting the bean into a singleton — the singleton caches the first instance. Profile switching is a startup-time decision, not per-request.
**Fix:** Profiles are resolved at context startup. For runtime switching, use strategy pattern + `@Qualifier`, not profiles.

---

### 7. Profile Condition Evaluated Before Context Is Fully Loaded
**Problem:** Writing a custom `@Conditional` that checks a profile-dependent property — the property might not be set yet when the condition is evaluated (during bean definition phase).
**Fix:** Use `ProfileCondition` / `@Profile` which is evaluated against the `Environment` (already loaded from property sources before bean registration).

---

### 8. No Bean Found When Profile Isn't Active (NoSuchBeanDefinitionException)
**Problem:**
```java
@Bean
@Profile("prod")
public AuditService auditService() { ... }
```
In `dev`, `auditService` doesn't exist → any code doing `ctx.getBean(AuditService.class)` throws `NoSuchBeanDefinitionException`.
**Fix:** Always provide a fallback bean annotated `@Profile("!prod")` or use `@Profile("default")` / a no-op implementation.

---

## 10. Miscellaneous & Advanced Details

### Accessing Active Profiles Programmatically

```java
@Autowired
private Environment env;

public void logProfiles() {
    String[] active = env.getActiveProfiles();
    String[] defaults = env.getDefaultProfiles();
    System.out.println("Active: " + Arrays.toString(active));
    System.out.println("Defaults: " + Arrays.toString(defaults));
}

// Check a specific profile
if (env.acceptsProfiles(Profiles.of("prod"))) {
    // prod-specific logic
}
```

### `@Profile` on `@Import`

```java
@Configuration
@Profile("prod")
@Import({ProdDataSourceConfig.class, ProdSecurityConfig.class, MetricsConfig.class})
public class ProdConfig {
    // All imported configurations only active in prod
}
```

### Profile-Aware `EnvironmentPostProcessor`

Add custom property sources based on the active profile before the application context is refreshed:

```java
public class VaultPropertySourcePostProcessor implements EnvironmentPostProcessor {
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment env,
                                       SpringApplication application) {
        if (env.acceptsProfiles(Profiles.of("prod"))) {
            // Load secrets from HashiCorp Vault
            env.getPropertySources().addFirst(new VaultPropertySource());
        }
    }
}
```

Register in `META-INF/spring.factories`:
```properties
org.springframework.boot.env.EnvironmentPostProcessor=\
  com.example.VaultPropertySourcePostProcessor
```

### Spring Boot Actuator — Profile Visibility

```
GET /actuator/env
```
Returns all active profiles and all property sources with their values. Invaluable for debugging "why is this property X and not Y".

```
GET /actuator/info
{
  "activeProfiles": ["prod", "metrics"]
}
```

### Spring Boot 3.x Changes

| Feature | Spring Boot 2.x | Spring Boot 3.x |
|---|---|---|
| `spring.profiles:` YAML key | Deprecated | Removed |
| `spring.config.activate.on-profile:` | Supported (2.4+) | Supported |
| `spring.profiles.group.*` | Supported (2.4+) | Supported |
| `@ActiveProfiles` test annotation | Supported | Supported |
| Multi-document YAML | `spring.profiles:` | `spring.config.activate.on-profile:` |

### Profile Resolution Order in Spring Boot

```
1. Default profile ("default") if no active profile found
2. application.properties spring.profiles.active
3. SPRING_PROFILES_ACTIVE env var
4. -Dspring.profiles.active JVM arg
5. --spring.profiles.active CLI arg
6. @ActiveProfiles (test only — overrides all above)
```

---

## 11. Interview Questions — Standard

### Q1: What is a Spring Profile and why is it used?

A Spring Profile is a named logical grouping for beans and configuration. When a profile is active, only beans and property files associated with that profile are loaded into the application context. It's used to maintain environment-specific configuration (dev, test, staging, prod) without code changes — you switch behavior by activating a profile at deployment time rather than branching in code.

---

### Q2: How do you activate a Spring Profile?

Multiple ways, in ascending priority:
1. `spring.profiles.active=prod` in `application.properties` (lowest priority for external configs)
2. `SPRING_PROFILES_ACTIVE=prod` environment variable
3. `-Dspring.profiles.active=prod` JVM system property
4. `--spring.profiles.active=prod` command-line argument (highest)
5. `@ActiveProfiles("prod")` in test classes
6. Programmatically: `SpringApplication.setAdditionalProfiles("prod")`

The highest-priority source wins. CLI args beat env vars beat `application.properties`.

---

### Q3: What is the difference between `@Profile("dev")` and `@Profile("!dev")`?

`@Profile("dev")` means the bean is registered **only when `dev` is active**. `@Profile("!dev")` means the bean is registered **when `dev` is NOT active** — i.e., for all other profiles. The `!` is a negation operator available since Spring 5.1's expression support. You also have `&` (AND) and `|` (OR) for compound expressions.

---

### Q4: How are profile-specific property files loaded in Spring Boot?

Spring Boot automatically loads `application-{profile}.properties` (or `.yml`) for each active profile. These files **override** values from the base `application.properties`. If multiple profiles are active, their files are loaded in the order they appear in `spring.profiles.active`, with later ones overriding earlier ones. The base file is always loaded first.

---

### Q5: What is the default Spring profile?

The `default` profile is activated when no other profile is explicitly active. You can declare beans/configs with `@Profile("default")` and they'll only be active in this case. You can also rename the default: `spring.profiles.default=local`.

---

### Q6: How is `@Profile` implemented internally?

`@Profile` is a meta-annotation backed by `@Conditional(ProfileCondition.class)`. `ProfileCondition` implements the `Condition` interface and calls `context.getEnvironment().acceptsProfiles(Profiles.of(profileNames))`. If the active profiles match, the condition returns `true` and the bean definition is registered; otherwise it's skipped entirely. This happens during the bean definition phase, before any beans are instantiated.

---

### Q7: What are Profile Groups in Spring Boot?

Introduced in Spring Boot 2.4, profile groups let you define an alias that expands to multiple profiles:
```properties
spring.profiles.group.production=prod,metrics,actuator-secure
```
Activating `production` automatically activates all three. This prevents deployment errors where companion profiles are forgotten.

---

### Q8: Can you use `@Profile` on a method-level `@Bean`?

Yes. You can put `@Profile` on individual `@Bean` methods inside a `@Configuration` class, allowing fine-grained control:
```java
@Configuration
public class AppConfig {
    @Bean
    @Profile("prod")
    public CacheManager redisCacheManager() { ... }

    @Bean
    @Profile("dev")
    public CacheManager simpleCacheManager() { ... }
}
```
The `@Configuration` class is always registered, but only the profile-matched `@Bean` methods are invoked.

---

### Q9: What happens if no bean matches a required type because its profile is inactive?

Spring throws `NoSuchBeanDefinitionException` at context startup (for eager singletons) or at `getBean()` time. To avoid this, always provide a fallback bean for the non-matched case using `@Profile("!prod")` or a `default`-scoped implementation.

---

### Q10: How do profiles interact with `@ConfigurationProperties`?

`@ConfigurationProperties` beans are always registered (no profile gating on the class itself by default). However, the property values they bind come from the active profile's property files. So `PaymentConfig.gatewayUrl` will bind to `payment.gateway.url` from `application-prod.yml` when `prod` is active, and from `application-dev.yml` when `dev` is active — automatically, with no additional code.

---

### Q11: How do you verify which profiles are active at runtime?

1. Spring Boot logs active profiles at startup: `The following 1 profile is active: "prod"`
2. `GET /actuator/env` — shows all property sources and active profiles
3. Inject `Environment` and call `env.getActiveProfiles()`
4. Use `@EventListener(ApplicationReadyEvent.class)` to log profiles after full startup

---

## 12. Interview Questions — Tricky / Scenario-Based

### Q1: You activate both `dev` and `prod` profiles. What happens to `@Profile("prod")` and `@Profile("dev")` beans?

**Both** sets of beans are registered. Spring evaluates each profile independently — if any active profile matches, the bean is included. This means you could end up with two `DataSource` beans, two `PaymentGateway` beans, etc., causing `NoUniqueBeanDefinitionException`. This is a real production risk when profiles are misconfigured. Fix: design profiles as mutually exclusive or use `@Profile("prod & !dev")` expressions.

---

### Q2: `@Profile("test")` is on your `DataSource` bean. You run `./gradlew test` without setting any profile. Does the bean get registered?

**No.** Without explicitly activating `test`, Spring uses the `default` profile. Your `@Profile("test")` bean is skipped, and if nothing else provides a `DataSource`, you get `NoSuchBeanDefinitionException`. Fix: add `@ActiveProfiles("test")` to your test class, or configure Gradle to pass `--spring.profiles.active=test` to the test JVM.

---

### Q3: A `@Transactional` method in a `@Profile("prod")`-gated service is not rolling back in staging. Why?

If `prod` is not active in staging, that `@Profile("prod")` service bean is not registered. If a different service bean exists for staging (e.g., `@Profile("staging")`), it may not have `@Transactional` properly configured, or the transaction manager may be a different bean. Always ensure that the same transactional contract applies across all profile variants of a service, or put `@Transactional` on the interface.

---

### Q4: You have `application.properties` with `spring.profiles.active=dev` and your Kubernetes deployment sets `SPRING_PROFILES_ACTIVE=prod`. Which wins?

The **environment variable wins**. `SPRING_PROFILES_ACTIVE` (which Spring Boot reads as `spring.profiles.active`) sourced from the OS environment has higher priority than `application.properties`. The active profile will be `prod`.

---

### Q5: Can two beans of the same type both be `@Profile("prod")`?

Yes. If both match, both are registered — Spring doesn't deduplicate by type. If anything tries to autowire by type (without `@Primary` or `@Qualifier`), you get `NoUniqueBeanDefinitionException`. This can happen accidentally when you have a `@Component` implementation and a `@Bean` factory method both marked `@Profile("prod")` producing the same type.

---

### Q6: Does `@Profile` work with prototype-scoped beans differently than singleton?

No — `@Profile` is evaluated at **context startup** during the bean definition registration phase, regardless of scope. If the profile matches, the bean definition (prototype or singleton) is registered. If not, it's never registered. The scope determines *when instances are created*; the profile determines *whether the definition exists at all*.

---

### Q7: You use `@Profile("dev")` on a `@Configuration` class and `@Profile("prod")` on a `@Bean` inside it. When `prod` is active (not `dev`), is the `@Bean` registered?

**No.** When `@Profile("dev")` is on the `@Configuration` class, the entire class is skipped when `dev` is not active. Spring never even processes the `@Bean` methods inside it. The `@Profile("prod")` on the method is irrelevant — the class-level `@Profile` gates everything inside it. **Lesson:** Class-level `@Profile` is evaluated first; method-level `@Profile` is only evaluated if the class-level condition passes.

---

### Q8: You need a bean active in `prod` OR `staging` but not `dev`. How do you write this?

```java
@Component
@Profile("prod | staging")      // Spring 5.1+ expression syntax
public class RealAuditService implements AuditService { }

@Component
@Profile("dev")
public class NoOpAuditService implements AuditService { }
```

Or equivalently using negation: `@Profile("!dev")` — but be careful if you have other profiles that should also use the no-op version.

---

## 13. Quick Revision Summary

- **Profile** = a named set of beans/config active only when that profile is activated at runtime.
- **`@Profile("name")`** on a class or `@Bean` method: registers that bean only if `name` is in the active profiles.
- **Profile activation priority (low→high):** `application.properties` → env var `SPRING_PROFILES_ACTIVE` → JVM `-D` arg → CLI `--` arg → `@ActiveProfiles` in tests.
- **Profile-specific files:** `application-{profile}.yml` auto-loaded by Spring Boot; overrides base `application.properties`.
- **`@Profile` is `@Conditional`** internally — backed by `ProfileCondition` which calls `env.acceptsProfiles(...)`.
- **Expressions (Spring 5.1+):** `!prod` (NOT), `prod & eu` (AND), `prod | staging` (OR) — use them to avoid registering conflicting beans.
- **Profile Groups (Boot 2.4+):** `spring.profiles.group.production=prod,metrics` — activating `production` activates all three.
- **Multi-document YAML (Boot 2.4+):** Use `spring.config.activate.on-profile:` (not the old deprecated `spring.profiles:`).
- **Both `dev` and `prod` active simultaneously** → both sets of profile-gated beans are registered → risk of `NoUniqueBeanDefinitionException`.
- **Class-level `@Profile` gates everything inside it** — method-level `@Profile` is only evaluated if the class-level condition already passes.
- **No fallback bean for inactive profile** → `NoSuchBeanDefinitionException` at startup — always provide a `@Profile("!prod")` fallback.
- **Never commit `spring.profiles.active=prod`** to source control — always inject via env var or CLI at deploy time.
