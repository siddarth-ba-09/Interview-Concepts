# Spring Boot Starters — Complete Reference

---

## 1. Overview

A **Spring Boot Starter** is a curated, ready-to-use dependency descriptor (a Maven/Gradle POM) that bundles all the libraries you need for a specific feature — along with their compatible, tested versions — into a single dependency.

**The problem they solve**: Before starters, adding a feature like JPA to a Spring project meant manually hunting down 8–10 individual JARs (`hibernate-core`, `spring-orm`, `spring-jdbc`, `HikariCP`, `spring-tx`, `javax.persistence-api`, ...), figuring out which versions were compatible with each other, and dealing with classpath conflicts. One typo in a version number could break the build.

Starters encode Spring Boot's **curated BOM (Bill of Materials)** knowledge: "these libraries, at these versions, work well together for this feature."

**Key properties:**
- Starters are **pure POM artifacts** — they contain no source code.
- They pull in **auto-configuration classes** that conditionally wire beans.
- All official starters follow the naming convention: `spring-boot-starter-*`.
- Third-party starters should follow: `*-spring-boot-starter`.

---

## 2. Core Concepts & Sub-Topics

### 2.1 What a Starter Contains

A starter POM has three things:

```
spring-boot-starter-web
       │
       ├── Dependency declarations
       │       spring-boot-starter          (core: spring-context, spring-beans, logging)
       │       spring-boot-starter-json     (jackson-databind, jackson-datatype-jdk8, ...)
       │       spring-boot-starter-tomcat   (embedded Tomcat)
       │       spring-web                   (Spring MVC core)
       │       spring-webmvc                (DispatcherServlet, controllers)
       │
       ├── Auto-configuration trigger
       │       META-INF/spring/...AutoConfiguration.imports
       │       (lists DispatcherServletAutoConfiguration,
       │              WebMvcAutoConfiguration,
       │              HttpMessageConvertersAutoConfiguration, ...)
       │
       └── Version management (via spring-boot-dependencies BOM)
               All versions are aligned — no manual version pinning needed
```

### 2.2 Starter vs Auto-Configuration vs BOM

| Concept | What It Is | Role |
|---|---|---|
| **Starter** | POM-only artifact | Pulls in the right JARs |
| **Auto-Configuration** | `@Configuration` classes | Wires beans conditionally based on classpath |
| **BOM (spring-boot-dependencies)** | Dependency management POM | Declares compatible versions for 300+ libraries |

These three work together: the starter brings the JARs; the BOM ensures versions are compatible; auto-configuration creates the beans automatically.

### 2.3 Starter Naming Conventions

| Pattern | Example | Meaning |
|---|---|---|
| `spring-boot-starter-{feature}` | `spring-boot-starter-web` | Official Spring Boot starter |
| `{feature}-spring-boot-starter` | `mybatis-spring-boot-starter` | Third-party starter |
| `spring-boot-starter` (no suffix) | `spring-boot-starter` | Core starter — included by every other starter |

### 2.4 The Parent POM vs BOM Approach

**Option A — Spring Boot Parent POM** (most common)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>
```
Gives you:
- Version management for all Spring Boot dependencies
- Default compiler settings (Java 17 source/target)
- Resource filtering for `application.properties` / `application.yml`
- Plugin management (spring-boot-maven-plugin, surefire, failsafe, etc.)

**Option B — Import BOM only** (when you have your own parent POM)
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.5</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 3. How Starters Work Internally

```
Step 1 — Maven/Gradle resolves the starter POM
         spring-boot-starter-web → transitive dependency tree resolved

Step 2 — JARs land on the classpath
         spring-webmvc.jar, jackson-databind.jar, tomcat-embed-core.jar, ...

Step 3 — SpringApplication.run() fires
         AutoConfigurationImportSelector reads:
         META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

Step 4 — Conditional evaluation
         Each auto-config class is checked:
         @ConditionalOnClass(DispatcherServlet.class)  → true (spring-webmvc is on classpath)
         @ConditionalOnMissingBean(WebMvcConfigurationSupport.class) → true (user hasn't defined one)

Step 5 — Beans are registered
         DispatcherServlet, TomcatServletWebServerFactory, HttpMessageConverters, ...
         are created and registered in ApplicationContext

Step 6 — User's beans override defaults
         If user defines @Bean DataSource, the auto-configured DataSource is skipped
         (@ConditionalOnMissingBean ensures this)
```

### Conditional Annotations That Guard Auto-Configurations

| Annotation | Meaning |
|---|---|
| `@ConditionalOnClass` | Applies only if the given class is on the classpath |
| `@ConditionalOnMissingClass` | Applies only if the class is NOT on the classpath |
| `@ConditionalOnBean` | Applies only if the given bean already exists in context |
| `@ConditionalOnMissingBean` | Applies only if the given bean does NOT exist — the most common one |
| `@ConditionalOnProperty` | Applies only if the property is set to a specific value |
| `@ConditionalOnWebApplication` | Applies only in a web (`SERVLET` or `REACTIVE`) application |
| `@ConditionalOnNotWebApplication` | Applies only in a non-web application |
| `@ConditionalOnResource` | Applies if a resource exists on the classpath |
| `@ConditionalOnExpression` | SpEL expression must evaluate to `true` |

---

## 4. Official Starter Catalogue

### 4.1 Core Starters

| Starter | What It Includes | Auto-Configurations Activated |
|---|---|---|
| `spring-boot-starter` | Core Spring (context, beans, SpEL, AOP), Logback, Log4j2 bridge, SLF4J | `LoggingAutoConfiguration`, `PropertyPlaceholderAutoConfiguration` |
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson JSON | `WebMvcAutoConfiguration`, `DispatcherServletAutoConfiguration`, `TomcatServletWebServerFactoryAutoConfiguration`, `HttpMessageConvertersAutoConfiguration` |
| `spring-boot-starter-webflux` | Spring WebFlux, Reactor Netty | `ReactiveWebServerFactoryAutoConfiguration`, `WebFluxAutoConfiguration` |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Hamcrest, MockMvc, Testcontainers support | Test-only — does not activate production auto-configs |

### 4.2 Data Starters

| Starter | What It Includes | Key Auto-Configurations |
|---|---|---|
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, JDBC, HikariCP | `JpaRepositoriesAutoConfiguration`, `HibernateJpaAutoConfiguration`, `DataSourceAutoConfiguration` |
| `spring-boot-starter-data-jdbc` | Spring Data JDBC (no Hibernate) | `JdbcRepositoriesAutoConfiguration`, `DataSourceAutoConfiguration` |
| `spring-boot-starter-jdbc` | Spring JDBC (`JdbcTemplate`), HikariCP | `DataSourceAutoConfiguration`, `JdbcTemplateAutoConfiguration` |
| `spring-boot-starter-data-mongodb` | Spring Data MongoDB, MongoDB Java driver | `MongoAutoConfiguration`, `MongoDataAutoConfiguration` |
| `spring-boot-starter-data-redis` | Spring Data Redis, Lettuce | `RedisAutoConfiguration`, `RedisRepositoriesAutoConfiguration` |
| `spring-boot-starter-data-elasticsearch` | Spring Data Elasticsearch, Elasticsearch REST client | `ElasticsearchRestClientAutoConfiguration` |
| `spring-boot-starter-data-cassandra` | Spring Data Cassandra, DataStax driver | `CassandraAutoConfiguration` |
| `spring-boot-starter-data-r2dbc` | Spring Data R2DBC (reactive SQL) | `R2dbcAutoConfiguration`, `R2dbcRepositoriesAutoConfiguration` |

### 4.3 Security Starters

| Starter | What It Includes | Key Auto-Configurations |
|---|---|---|
| `spring-boot-starter-security` | Spring Security (web security filter chain) | `SpringBootWebSecurityConfiguration`, `SecurityAutoConfiguration`, `SecurityFilterAutoConfiguration` |
| `spring-boot-starter-oauth2-resource-server` | OAuth2 JWT/opaque token validation | `OAuth2ResourceServerAutoConfiguration` |
| `spring-boot-starter-oauth2-client` | OAuth2 login, client credentials flow | `OAuth2ClientAutoConfiguration` |

### 4.4 Messaging Starters

| Starter | What It Includes | Key Auto-Configurations |
|---|---|---|
| `spring-boot-starter-amqp` | Spring AMQP, RabbitMQ client | `RabbitAutoConfiguration` |
| `spring-boot-starter-activemq` | Spring JMS, ActiveMQ | `ActiveMQAutoConfiguration`, `JmsAutoConfiguration` |
| `spring-boot-starter-kafka` | Spring Kafka, Kafka clients | `KafkaAutoConfiguration` |
| `spring-boot-starter-mail` | Spring Mail, JavaMail | `MailSenderAutoConfiguration` |
| `spring-boot-starter-websocket` | Spring WebSocket, STOMP | `WebSocketAutoConfiguration` |

### 4.5 Observability & Operations Starters

| Starter | What It Includes | Key Auto-Configurations |
|---|---|---|
| `spring-boot-starter-actuator` | Spring Boot Actuator, Micrometer | `EndpointAutoConfiguration`, `HealthEndpointAutoConfiguration`, `MetricsAutoConfiguration` |
| `spring-boot-starter-aop` | Spring AOP, AspectJ weaver | `AopAutoConfiguration` |
| `spring-boot-starter-cache` | Spring Cache abstraction | `CacheAutoConfiguration` |
| `spring-boot-starter-validation` | Hibernate Validator, Jakarta Bean Validation | `ValidationAutoConfiguration` |

### 4.6 Template Engine Starters

| Starter | Engine | Key Auto-Configurations |
|---|---|---|
| `spring-boot-starter-thymeleaf` | Thymeleaf | `ThymeleafAutoConfiguration` |
| `spring-boot-starter-freemarker` | FreeMarker | `FreeMarkerAutoConfiguration` |
| `spring-boot-starter-mustache` | Mustache | `MustacheAutoConfiguration` |

### 4.7 Embedded Server Alternatives

`spring-boot-starter-web` uses Tomcat by default. To switch:

```xml
<!-- Switch to Jetty -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

| Starter | Server | Use Case |
|---|---|---|
| `spring-boot-starter-tomcat` (default) | Apache Tomcat | General purpose, widest ecosystem support |
| `spring-boot-starter-jetty` | Eclipse Jetty | Lighter footprint, WebSocket-heavy workloads |
| `spring-boot-starter-undertow` | JBoss Undertow | High throughput, non-blocking HTTP/2 |
| `spring-boot-starter-reactor-netty` | Reactor Netty | Reactive (WebFlux) applications |

---

## 5. Configuration & Setup

### 5.1 Minimal Web Application

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- No version tag needed — managed by parent BOM -->

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 5.2 Full Microservice Stack

```xml
<dependencies>
    <!-- Web layer -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Persistence -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Caching -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Observability -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Messaging -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-kafka</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 5.3 Overriding a Managed Version

```xml
<!-- Override a specific library version managed by the BOM -->
<properties>
    <jackson.version>2.16.1</jackson.version>
    <hibernate.version>6.4.4.Final</hibernate.version>
</properties>
```

---

## 6. Real-World Production Code Example

### Building a Custom Starter for a Shared Audit Library

A senior engineer at a large company creates a reusable audit-logging starter that all microservices consume.

**Project structure:**
```
audit-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/com/company/audit/
    │   ├── AuditProperties.java          @ConfigurationProperties
    │   ├── AuditService.java             the core bean
    │   ├── AuditAspect.java              AOP advice
    │   └── AuditAutoConfiguration.java  @AutoConfiguration
    └── resources/META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

**Step 1 — Properties**

```java
@ConfigurationProperties(prefix = "audit")
public class AuditProperties {
    private boolean enabled = true;
    private String logLevel = "INFO";
    private String topic = "audit-events";
    // getters + setters
}
```

**Step 2 — The Core Bean**

```java
public class AuditService {

    private final AuditProperties properties;
    private final KafkaTemplate<String, AuditEvent> kafkaTemplate;

    public AuditService(AuditProperties properties,
                        KafkaTemplate<String, AuditEvent> kafkaTemplate) {
        this.properties = properties;
        this.kafkaTemplate = kafkaTemplate;
    }

    public void record(String action, String entityId, String userId) {
        if (!properties.isEnabled()) return;
        AuditEvent event = new AuditEvent(action, entityId, userId, Instant.now());
        kafkaTemplate.send(properties.getTopic(), event);
    }
}
```

**Step 3 — AOP Aspect**

```java
@Aspect
public class AuditAspect {

    private final AuditService auditService;

    public AuditAspect(AuditService auditService) {
        this.auditService = auditService;
    }

    // Intercept any method annotated with @Audited
    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        Object result = pjp.proceed();
        auditService.record(audited.action(), audited.entityId(), getCurrentUserId());
        return result;
    }
}
```

**Step 4 — Auto-Configuration Class**

```java
@AutoConfiguration
@ConditionalOnClass(KafkaTemplate.class)            // only if Kafka is on classpath
@ConditionalOnProperty(prefix = "audit", name = "enabled", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                        // user can override by defining their own
    public AuditService auditService(AuditProperties properties,
                                     KafkaTemplate<String, AuditEvent> kafkaTemplate) {
        return new AuditService(properties, kafkaTemplate);
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean(AuditService.class)
    public AuditAspect auditAspect(AuditService auditService) {
        return new AuditAspect(auditService);
    }
}
```

**Step 5 — Register the Auto-Configuration**

```
# src/main/resources/META-INF/spring/
# org.springframework.boot.autoconfigure.AutoConfiguration.imports

com.company.audit.AuditAutoConfiguration
```

**Step 6 — Starter POM (pure POM, no code)**

```xml
<!-- audit-spring-boot-starter/pom.xml -->
<artifactId>audit-spring-boot-starter</artifactId>
<packaging>pom</packaging>

<dependencies>
    <!-- The autoconfigure module -->
    <dependency>
        <groupId>com.company</groupId>
        <artifactId>audit-spring-boot-autoconfigure</artifactId>
    </dependency>
    <!-- Optional: Kafka only needed if user wants Kafka-backed audit -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

**Consumer microservice — zero config needed:**

```java
// Any service just adds the starter dependency and uses @Audited
@Service
public class OrderService {

    @Audited(action = "ORDER_CREATED", entityId = "#request.orderId")
    @Transactional
    public OrderDTO createOrder(CreateOrderRequest request) {
        // audit is recorded automatically via AOP
        ...
    }
}
```

```yaml
# application.yml in consumer — optional overrides
audit:
  enabled: true
  topic: order-audit-events
  log-level: DEBUG
```

---

## 7. Production Use-Cases & Best Practices

1. **Never specify versions for starter-managed dependencies** — the BOM guarantees version compatibility. Adding manual `<version>` tags breaks the BOM contract and introduces conflict risks.

2. **Exclude the default Tomcat starter when deploying to an external container** — WAR deployments to standalone Tomcat should exclude `spring-boot-starter-tomcat` and mark it `provided`:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-tomcat</artifactId>
       <scope>provided</scope>
   </dependency>
   ```

3. **Use `spring-boot-starter-test` scope `test`** — it pulls in Mockito, AssertJ, JUnit 5. Never use `compile` scope for test starters.

4. **Separate autoconfigure module from starter module** — the official Spring Boot pattern. The autoconfigure module has the code; the starter POM just declares dependencies. This allows consumers to use `autoconfigure` directly without the full starter if needed.

5. **Always use `@ConditionalOnMissingBean` in custom auto-configurations** — this is the contract that lets consumers override your defaults. Without it, your bean and the user's bean would conflict.

6. **Disable unwanted auto-configurations via `exclude`**:
   ```java
   @SpringBootApplication(exclude = {
       DataSourceAutoConfiguration.class,
       SecurityAutoConfiguration.class
   })
   ```
   Or in `application.properties`:
   ```properties
   spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
   ```

7. **Use `spring-boot-configuration-processor`** in starters that expose `@ConfigurationProperties` — generates `META-INF/spring-configuration-metadata.json` for IDE autocomplete.
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-configuration-processor</artifactId>
       <optional>true</optional>
   </dependency>
   ```

8. **Audit your effective dependency tree** regularly with `mvn dependency:tree` — starters pull in many transitive dependencies; version conflicts surface here.

---

## 8. Scenario Walkthroughs

### Scenario 1: Two starters pull in conflicting Jackson versions

**Setup**: `spring-boot-starter-web` (Jackson 2.15) and a third-party `analytics-spring-boot-starter` that explicitly declares `jackson-databind:2.13.0`.

**What happens**:
1. Maven's nearest-wins strategy resolves `jackson-databind` to `2.13.0` (explicitly declared in analytics starter, wins over transitive 2.15).
2. Spring Boot's `JacksonAutoConfiguration` configures beans assuming 2.15 API — some new methods are absent in 2.13, causing `NoSuchMethodError` at runtime.

**How to diagnose**: `mvn dependency:tree | grep jackson`

**Fix**:
```xml
<!-- Force the BOM-managed version -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson-bom.version}</version>  <!-- from BOM -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

### Scenario 2: Auto-configuration fires even though you don't want it

**Setup**: You add `spring-boot-starter-security` to the project for JWT validation. Spring Security's default auto-configuration immediately secures all endpoints with HTTP Basic, breaking your local development.

**What happened**:
- `SpringBootWebSecurityConfiguration` is triggered by `@ConditionalOnDefaultWebSecurity` — fires when no custom `SecurityFilterChain` bean exists.
- The default rule: all requests require authentication via HTTP Basic.

**Fix options**:
```java
// Option 1: Define your own SecurityFilterChain (most common)
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}

// Option 2: Exclude the default auto-configuration
@SpringBootApplication(exclude = SecurityAutoConfiguration.class)
```

---

### Scenario 3: Custom starter not being picked up in consumer

**Setup**: Team creates `company-audit-spring-boot-starter`. Consumer adds the dependency but `AuditService` is not found in the context.

**Root cause checklist**:
1. `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file is **missing** or has a typo in the class name.
2. The file is in the wrong module — it must be in the **autoconfigure module** JAR, not the starter POM.
3. `@ConditionalOnClass` references a class not on the consumer's classpath.
4. The auto-configuration class itself has a compilation error, silently skipped.

**Verification**:
```java
// Add this to your SpringBootTest to see which auto-configurations are applied:
@SpringBootTest
class AutoConfigTest {
    @Autowired
    ApplicationContext context;

    @Test
    void auditServiceIsPresent() {
        assertThat(context.getBean(AuditService.class)).isNotNull();
    }
}
```

```bash
# Or check the auto-configuration report
java -jar app.jar --debug 2>&1 | grep -i audit
```

---

## 9. Common Pitfalls & Gotchas

| Pitfall | Why It Happens | Fix |
|---|---|---|
| `NoSuchBeanDefinitionException` for auto-configured bean | Missing starter dependency — the class condition fails silently | Add the correct starter |
| Duplicate bean conflict at startup | Two starters auto-configure the same bean type without `@ConditionalOnMissingBean` | Check which starter is creating it; exclude the unwanted auto-config |
| `@ConditionalOnMissingBean` not working | Your `@Bean` definition is in a `@Configuration` class processed AFTER the auto-config | Ensure auto-configs are in the right order using `@AutoConfigureBefore` / `@AutoConfigureAfter` |
| Custom starter works locally but not in tests | The `AutoConfiguration.imports` file is not in `src/main/resources` — it's in `src/test/resources` | Move to correct resource folder |
| Starter added but no auto-configuration fires | `spring.factories` used in Spring Boot 2.x; Spring Boot 3.x uses `AutoConfiguration.imports` | Update the registration file to `AutoConfiguration.imports` |
| Transitive dependency brings in an unwanted starter | e.g. A library internally uses `spring-boot-starter-security` | Use `<exclusions>` to remove the unwanted transitive starter |
| Overriding a BOM-managed version via `<properties>` doesn't work | The BOM uses `<dependencyManagement>` which can be overridden, but the property key must match exactly | Check the exact property name in `spring-boot-dependencies` POM |
| `spring-boot-starter-test` on compile scope | Leaks test libraries (Mockito, JUnit) into production artifacts | Always use `<scope>test</scope>` |
| WAR deployment with embedded Tomcat | `spring-boot-starter-tomcat` not excluded leads to conflict with container's Tomcat | Mark it `<scope>provided</scope>` |
| Gradual migration from `spring.factories` to `AutoConfiguration.imports` | Both are supported in Spring Boot 3.x but `spring.factories` is deprecated | Migrate to `AutoConfiguration.imports` immediately |

---

## 10. Miscellaneous & Advanced Details

### 10.1 How `spring-boot-starter-parent` Differs from `spring-boot-dependencies`

```
spring-boot-starter-parent
    └── parent: spring-boot-dependencies  (version management)
    └── adds: default plugin configs, resource filtering,
              java.version property (default 17),
              UTF-8 encoding, surefire/failsafe config
              @{...} resource filtering delimiters

spring-boot-dependencies
    └── only: <dependencyManagement> — 300+ library version declarations
    └── use this if you have your own corporate parent POM
```

### 10.2 Viewing All Active Auto-Configurations

```bash
# Run your app with --debug flag
java -jar app.jar --debug

# Output includes CONDITIONS EVALUATION REPORT:
# Positive matches (activated):
#   DataSourceAutoConfiguration (matched)
#     - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)
# Negative matches (skipped):
#   MongoAutoConfiguration (not matched)
#     - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

Or in code:
```java
@Autowired
private ConditionEvaluationReport report;

// Or via Actuator:
// GET /actuator/conditions
```

### 10.3 Spring Boot 2.x vs 3.x Starter Registration

| Aspect | Spring Boot 2.x | Spring Boot 3.x |
|---|---|---|
| Registration file | `META-INF/spring.factories` key `EnableAutoConfiguration` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| `@AutoConfiguration` annotation | Not available — use `@Configuration` | Use `@AutoConfiguration` (preferred) |
| Jakarta EE | `javax.*` | `jakarta.*` |
| AOT / GraalVM | Limited support | First-class native image support via AOT |

### 10.4 `@AutoConfigureBefore` and `@AutoConfigureAfter`

Controls the ordering of auto-configuration classes:

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
public class MyDatabaseAutoConfiguration {
    // Runs after DataSource is available
}

@AutoConfiguration(before = WebMvcAutoConfiguration.class)
public class MyWebCustomizationAutoConfiguration {
    // Runs before WebMvc sets up
}
```

### 10.5 Lazy Starter Initialization

Spring Boot 2.2+ supports lazy initialization globally:
```properties
spring.main.lazy-initialization=true
```
All beans (including auto-configured ones) are created on first use instead of at startup — reduces startup time but may push failures to first request. Useful for development; **not recommended for production** without thorough testing.

### 10.6 Actuator `/actuator/conditions` Endpoint

```json
// GET /actuator/conditions
{
  "positiveMatches": {
    "WebMvcAutoConfiguration": [
      { "condition": "OnClassCondition", "message": "@ConditionalOnClass found required class 'DispatcherServlet'" },
      { "condition": "OnWebApplicationCondition", "message": "found 'session' scope" }
    ]
  },
  "negativeMatches": {
    "MongoAutoConfiguration": {
      "notMatched": [
        { "condition": "OnClassCondition", "message": "did not find required class 'com.mongodb.MongoClient'" }
      ]
    }
  }
}
```

### 10.7 `spring-boot-starter-test` Contents

```
spring-boot-starter-test
    ├── JUnit 5 (junit-jupiter)
    ├── Mockito (mockito-core, mockito-junit-jupiter)
    ├── AssertJ
    ├── Hamcrest
    ├── JSONassert
    ├── JsonPath
    ├── spring-test (MockMvc, TestContext framework)
    ├── spring-boot-test (SpringBootTest, WebMvcTest, etc.)
    └── Awaitility (async test utilities)
```

---

## 11. Interview Questions — Standard

**Q1. What is a Spring Boot Starter and what problem does it solve?**

A starter is a POM-only dependency descriptor that bundles all required JARs for a feature into a single `<dependency>` entry. Before starters, developers manually listed 8–12 individual JARs for a feature like JPA, hunted for compatible versions, and dealt with classpath conflicts. Starters encode Spring Boot's curated BOM knowledge — all included libraries are guaranteed to be compatible with each other and with the Spring Boot version. Adding `spring-boot-starter-data-jpa` pulls in Hibernate, Spring Data JPA, Spring ORM, Spring JDBC, and HikariCP in one line with no version management required.

---

**Q2. What is the difference between `spring-boot-starter-parent` and `spring-boot-dependencies`?**

Both provide dependency version management via a BOM. `spring-boot-starter-parent` is a full parent POM that also provides default build configuration: Java compiler version (17), UTF-8 encoding, resource filtering for `application.properties`, and plugin management for `spring-boot-maven-plugin`, `surefire`, `failsafe`, etc. `spring-boot-dependencies` is just the BOM — it only contains `<dependencyManagement>` entries. Use `spring-boot-dependencies` (imported into `<dependencyManagement>`) when your project already has a corporate parent POM and you cannot use `spring-boot-starter-parent` as the parent.

---

**Q3. How does Spring Boot decide which auto-configuration to apply when a starter is added?**

When a starter is added, it brings JARs onto the classpath. At startup, `AutoConfigurationImportSelector` reads all class names from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. For each auto-configuration class, Spring evaluates the `@Conditional*` annotations — primarily `@ConditionalOnClass` (is the required class on the classpath?), `@ConditionalOnMissingBean` (has the user not defined this bean already?), and `@ConditionalOnProperty` (is the feature enabled?). Only configurations that pass all conditions are applied. This "classpath-driven, condition-guarded" approach is how adding a single starter results in all the right beans being created.

---

**Q4. What is `@ConditionalOnMissingBean` and why is it critical in auto-configurations?**

`@ConditionalOnMissingBean` registers a bean only if no bean of the same type already exists in the `ApplicationContext`. In auto-configurations, it is the mechanism that ensures user-defined beans always override auto-configured defaults. For example, `DataSourceAutoConfiguration` creates a `DataSource` only if the user hasn't defined one. Without this annotation, if a user provides their own `DataSource`, Spring would have two `DataSource` beans and throw `NoUniqueBeanDefinitionException`. It is the core contract of "convention over configuration, but configuration wins."

---

**Q5. How do you switch from Tomcat to Undertow in a Spring Boot application?**

Exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web` and add `spring-boot-starter-undertow`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

Spring Boot detects `UndertowServletWebServerFactory` on the classpath and auto-configures it instead of Tomcat. No other code change is needed because all embedded servers implement the same `WebServer` abstraction.

---

**Q6. How do you create a custom Spring Boot Starter?**

There are two modules: an `autoconfigure` module containing the `@AutoConfiguration` class and the actual beans, and a `starter` POM that lists the autoconfigure module plus required dependencies. The auto-configuration class is annotated with `@AutoConfiguration` and uses `@ConditionalOnClass` / `@ConditionalOnMissingBean` guards. Its fully qualified class name is registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. The starter POM is a `<packaging>pom</packaging>` artifact with no code. Consumer services add only the starter dependency and get all beans automatically.

---

**Q7. What does `mvn dependency:tree` tell you about starters?**

It shows the full transitive dependency tree. Since starters are POM-only, they appear as a node in the tree, with all their transitive JARs listed beneath them. This is essential for: (1) diagnosing version conflicts where two starters bring incompatible versions of the same library, (2) verifying that an exclusion worked, (3) finding unexpected JARs that bloat the artifact. Running `mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:*` filters results to show only Jackson-related dependencies.

---

**Q8. What is the difference between `spring-boot-starter-web` and `spring-boot-starter-webflux`?**

`spring-boot-starter-web` is for **servlet-based** (blocking, thread-per-request) REST APIs backed by an embedded Tomcat/Jetty/Undertow. `spring-boot-starter-webflux` is for **reactive** (non-blocking, event-loop) REST APIs backed by Reactor Netty. They cannot both be the primary web stack in the same application — adding both means `spring-boot-starter-web` wins by default (it has higher auto-configuration priority). WebFlux supports backpressure and reactive types (`Mono`, `Flux`); the servlet stack uses traditional `List` / POJO return types.

---

**Q9. How can you disable a specific auto-configuration that a starter activates?**

Three approaches: (1) `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` — compile-time exclusion, type-safe. (2) `spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration` in `application.properties` — runtime, useful when the class to exclude is not on the compile classpath. (3) For conditional configurations, set the controlling property: `spring.datasource.enabled=false` if the auto-config is guarded by `@ConditionalOnProperty`.

---

**Q10. What happens if you add `spring-boot-starter-security` to an existing application?**

As soon as the starter is on the classpath, `SecurityAutoConfiguration` and `SpringBootWebSecurityConfiguration` activate. A default `SecurityFilterChain` is created that requires HTTP Basic authentication for ALL endpoints. A random password is generated and printed to the console at startup. The default username is `user`. All existing endpoints are immediately secured. This is the correct behaviour for the "secure by default" philosophy, but requires you to define your own `SecurityFilterChain` bean to customise access rules.

---

## 12. Interview Questions — Tricky / Scenario-Based

**Q1. You add a starter but the expected bean is not in the ApplicationContext. How do you diagnose it?**

Run the app with `--debug` flag to get the Conditions Evaluation Report. Under "Negative matches", look for your expected auto-configuration class and read why it was skipped. Typical causes: (1) `@ConditionalOnClass` — a required class is missing from the classpath, meaning a JAR is absent; (2) `@ConditionalOnMissingBean` inverted — you already have a bean of that type so the auto-configured one was skipped; (3) `@ConditionalOnProperty` — a required property is not set or is set to `false`; (4) The `AutoConfiguration.imports` file is missing or misspelled, so the class is never evaluated. Use `/actuator/conditions` for a running application.

---

**Q2. A colleague adds `spring-boot-starter-data-jpa` but your application doesn't have a database — startup fails. What happens and how do you fix it?**

Adding `spring-boot-starter-data-jpa` puts `DataSourceAutoConfiguration` on the active configurations list. It tries to configure a `DataSource` but finds no `spring.datasource.url` — it throws `DataSourceBeanCreationException` at startup. Three fixes: (1) Add a database URL + driver dependency; (2) Add H2 in-memory DB for testing: `spring-boot-starter-h2` with `scope=runtime`; (3) Exclude the auto-configuration: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` — appropriate if you're adding JPA entities for schema generation only, without runtime DB connectivity.

---

**Q3. Two custom starters both auto-configure a `CacheManager` bean. What happens?**

Spring throws `NoUniqueBeanDefinitionException` — two beans of the same type, `CacheManager`, are found and Spring cannot determine which to inject. Root cause: one or both auto-configurations are missing `@ConditionalOnMissingBean(CacheManager.class)`. Fix: if you own the starters, add the conditional. If you don't, exclude one auto-configuration: `@SpringBootApplication(exclude = OffendingCacheAutoConfiguration.class)` or define your own `CacheManager` bean — since `@ConditionalOnMissingBean` will suppress both auto-configured ones once you provide your own.

---

**Q4. Your custom starter works with Spring Boot 2.7 but breaks after upgrading to Spring Boot 3.0. Why?**

Spring Boot 3.0 changed auto-configuration registration from `META-INF/spring.factories` (under key `org.springframework.boot.autoconfigure.EnableAutoConfiguration`) to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. If your starter still uses `spring.factories`, it is ignored in Spring Boot 3.0+. Additionally: (1) All `javax.*` imports must be changed to `jakarta.*`; (2) `@Configuration` on auto-config classes should be replaced with `@AutoConfiguration`; (3) The `spring-boot-autoconfigure-processor` annotation processor now generates different metadata. Fix: create the new `AutoConfiguration.imports` file and update package imports.

---

**Q5. You have a multi-module Maven project. Should each module have its own `spring-boot-starter-parent`?**

No. In a multi-module Maven project, only the **root parent POM** should declare `spring-boot-starter-parent` as its parent. Individual modules inherit from the root parent and get version management transitively. If each module independently declares `spring-boot-starter-parent`, you lose the ability to centrally manage versions across modules and risk different modules using different Spring Boot versions. The pattern is: root POM inherits `spring-boot-starter-parent` → child modules inherit root POM. Each child module can then use starters without version tags.

---

**Q6. What is the risk of using `spring.main.lazy-initialization=true` in production with starters?**

Lazy initialization defers bean creation until first use. The risk is that startup-time failures are pushed to request-time failures — a broken `DataSource` or misconfigured `KafkaProducer` that would have crashed startup now crashes on the first real request. Health checks that run at startup (e.g. `DataSourceHealthIndicator`) also fail silently. For production, it is generally safer to fail fast at startup so the issue is caught immediately by deployment monitoring rather than hitting live users. Lazy init is best suited for development to reduce startup time.

---

**Q7. You need `spring-boot-starter-security` for one module but it should NOT affect another module in the same application. Is this possible?**

Not directly — Spring Security's `SecurityFilterChain` applies globally to the embedded web server, not per-module. The correct approach is to use `SecurityFilterChain` beans with `securityMatcher()` (in Spring Security 6+) to limit each chain's scope:

```java
@Bean
@Order(1)
public SecurityFilterChain adminChain(HttpSecurity http) throws Exception {
    return http.securityMatcher("/admin/**")
               .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("ADMIN"))
               .build();
}

@Bean
@Order(2)
public SecurityFilterChain publicChain(HttpSecurity http) throws Exception {
    return http.securityMatcher("/public/**")
               .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
               .build();
}
```

---

## 13. Quick Revision Summary

- A **starter is a POM artifact** — no source code, only dependency declarations that pull in the right JARs.
- **Starter + Auto-configuration + BOM** are three distinct things that work together: starter brings JARs, BOM ensures version compatibility, auto-config wires beans.
- All official starters: `spring-boot-starter-*`; third-party starters: `*-spring-boot-starter`.
- **`@ConditionalOnMissingBean`** is the core contract — user-defined beans always override auto-configured defaults.
- Adding `spring-boot-starter-security` **immediately secures all endpoints** with HTTP Basic — must define a custom `SecurityFilterChain` to customise.
- `spring-boot-starter-parent` = BOM + build defaults; `spring-boot-dependencies` = BOM only (for projects with existing parent POMs).
- Switch embedded server by **excluding** `spring-boot-starter-tomcat` and **adding** `spring-boot-starter-jetty` or `spring-boot-starter-undertow`.
- **Spring Boot 3.x**: auto-configuration registration moved from `spring.factories` to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- Custom starters: **two modules** — `*-autoconfigure` (has `@AutoConfiguration` class) + `*-starter` (POM only, depends on autoconfigure).
- Debug auto-configuration decisions with `--debug` flag or `GET /actuator/conditions`.
- `mvn dependency:tree` is your first tool for diagnosing classpath conflicts introduced by starters.
- `spring.main.lazy-initialization=true` reduces startup time but pushes failures to request-time — avoid in production.
