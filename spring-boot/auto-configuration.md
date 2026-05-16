# Spring Boot Auto-Configuration

## 1. Overview

**What is it?**  
Spring Boot Auto-configuration is a mechanism that automatically configures your Spring application based on the JARs present on the classpath, beans already defined, and properties set. It removes the need to write boilerplate `@Bean` definitions and XML configuration for common concerns like DataSource, JPA, Security, Web MVC, etc.

**Why does it exist?**  
Before Spring Boot, you had to manually declare every bean — `DataSource`, `EntityManagerFactory`, `TransactionManager`, etc. — even for a standard JPA app. Auto-configuration applies the **"convention over configuration"** principle: if you have `spring-data-jpa` on the classpath, Spring Boot can set up a complete JPA stack for you, while still letting you override any piece of it.

---

## 2. How It Works Internally

### Step-by-step internals

```
1. @SpringBootApplication
        ↓
2. @EnableAutoConfiguration
        ↓
3. AutoConfigurationImportSelector.selectImports()
        ↓
4. Load candidate configs from:
   - META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports  (Boot 2.7+)
   - META-INF/spring.factories (Boot < 2.7, still supported)
        ↓
5. Filter using @Conditional* annotations
        ↓
6. Import surviving @Configuration classes into ApplicationContext
        ↓
7. Beans are registered — but only if conditions pass
```

### Key mechanism in detail

#### `@SpringBootApplication`
A meta-annotation composed of three annotations:
```java
@SpringBootConfiguration       // = @Configuration
@EnableAutoConfiguration       // triggers auto-config
@ComponentScan                 // scans the current package
```

#### `@EnableAutoConfiguration`
Imports `AutoConfigurationImportSelector` as an `ImportSelector`. This class implements `DeferredImportSelector`, meaning it runs **after** all user-defined `@Configuration` classes. This is critical — it lets your beans take precedence over auto-configured beans.

#### `AutoConfigurationImportSelector`
1. Reads `AutoConfiguration.imports` (or `spring.factories` key `EnableAutoConfiguration`).
2. Applies **exclusions** (`@SpringBootApplication(exclude = ...)` or `spring.autoconfigure.exclude` property).
3. Applies **`@Conditional` filters** — only configs whose conditions are satisfied are imported.
4. Applies **ordering** (`@AutoConfigureBefore`, `@AutoConfigureAfter`, `@AutoConfigureOrder`).

#### `AutoConfiguration.imports` (Boot 2.7+)
Located at:
```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
Each line is a fully-qualified auto-configuration class name:
```
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```
There are **~150+** such classes in `spring-boot-autoconfigure`.

#### `@Conditional*` Annotations — The Real Gate-keepers

| Annotation | Condition |
|---|---|
| `@ConditionalOnClass` | Class is present on classpath |
| `@ConditionalOnMissingClass` | Class is absent from classpath |
| `@ConditionalOnBean` | A bean of that type/name already exists |
| `@ConditionalOnMissingBean` | No bean of that type/name exists |
| `@ConditionalOnProperty` | A property has a specific value |
| `@ConditionalOnResource` | A resource file exists |
| `@ConditionalOnWebApplication` | Running as a web application |
| `@ConditionalOnExpression` | A SpEL expression evaluates to true |
| `@ConditionalOnSingleCandidate` | Exactly one bean of that type exists |

**The most important:** `@ConditionalOnMissingBean` — this is how user beans override auto-configured beans.

---

## 3. Key Annotations / Classes / Interfaces

| Name | Role |
|---|---|
| `@EnableAutoConfiguration` | Triggers the auto-config mechanism |
| `@SpringBootApplication` | Shortcut: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `AutoConfigurationImportSelector` | Reads and filters the list of auto-config candidates |
| `@AutoConfiguration` | Marks a class as an auto-configuration (Boot 2.7+, replaces `@Configuration` for this purpose) |
| `@ConditionalOnClass` | Condition based on classpath |
| `@ConditionalOnMissingBean` | Condition: no user-defined bean → back off |
| `@AutoConfigureBefore` / `@AutoConfigureAfter` | Controls ordering between auto-configs |
| `@AutoConfigureOrder` | Numeric ordering hint |
| `spring.autoconfigure.exclude` | Property to disable specific auto-configs |
| `ConditionEvaluationReport` | Debug report of which conditions passed/failed |

---

## 4. Real-World Example

### Scenario: You add `spring-boot-starter-data-jpa` to your pom

Spring Boot detects `HikariCP`, `Hibernate`, `JPA` on the classpath.

**`DataSourceAutoConfiguration`** fires:
```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceCheckpointRestoreConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class, ... })
    protected static class PooledDataSourceConfiguration { }
}
```

**The flow for `application.properties`:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/shop
spring.datasource.username=admin
spring.datasource.password=secret
spring.datasource.driver-class-name=org.postgresql.Driver
```

Spring Boot binds these to `DataSourceProperties` and creates a `HikariDataSource` bean automatically.

**Overriding it** — if you declare your own:
```java
@Bean
public DataSource dataSource() {
    // your custom DataSource
}
```
`@ConditionalOnMissingBean(DataSource.class)` causes the auto-configured one to **back off** entirely.

---

## 5. Scenario Walkthroughs

### Scenario 1: E-commerce app — What happens when the app starts?

You have `spring-boot-starter-web` and `spring-boot-starter-data-jpa`.

1. `SpringApplication.run()` creates an `AnnotationConfigServletWebServerApplicationContext`.
2. `@ComponentScan` picks up your `@Service`, `@Repository`, `@RestController` classes.
3. `AutoConfigurationImportSelector` loads ~150 candidate configs from `AutoConfiguration.imports`.
4. **`WebMvcAutoConfiguration`** — `@ConditionalOnClass(DispatcherServlet.class)` → passes → registers `DispatcherServlet`, `HandlerMapping`, `HandlerAdapter`, etc.
5. **`DataSourceAutoConfiguration`** — `@ConditionalOnClass(DataSource.class, HikariDataSource.class)` → passes → creates `HikariDataSource`.
6. **`HibernateJpaAutoConfiguration`** — creates `EntityManagerFactory`, `JpaTransactionManager`.
7. **`JpaRepositoriesAutoConfiguration`** — `@ConditionalOnBean(EntityManagerFactory.class)` → passes → enables `@EnableJpaRepositories`, scanning your repository interfaces.
8. **`EmbeddedWebServerFactoryCustomizerAutoConfiguration`** — creates embedded Tomcat on port 8080.
9. Application is ready.

Total time: ~2–3 seconds for a standard app. Each auto-config is evaluated but only surviving ones contribute beans.

---

### Scenario 2: Debugging — Why is my auto-config not being applied?

You add `spring-boot-starter-security` but your custom `SecurityFilterChain` isn't working as expected. 

Enable the condition evaluation report:
```properties
# application.properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```
Or run with `--debug` flag.

Output:
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
SecurityAutoConfiguration matched:
   - @ConditionalOnClass found required class 'org.springframework.security.web.SecurityFilterChain'

Negative matches:
-----------------
UserDetailsServiceAutoConfiguration:
   - @ConditionalOnMissingBean (types: UserDetailsService) found bean 'userDetailsService'
```

This tells you exactly why each auto-config fired or didn't.

---

## 6. Common Interview Questions

### Q1: What is `@EnableAutoConfiguration` and how does it work?
**A:** It imports `AutoConfigurationImportSelector` which loads all auto-configuration candidates from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each candidate is a `@Configuration` class guarded by `@Conditional*` annotations. Only those whose conditions are satisfied get imported into the `ApplicationContext`. The selector is a `DeferredImportSelector`, so it runs after user-defined beans — ensuring user beans take priority.

---

### Q2: How does Spring Boot know not to override your manually defined beans?
**A:** Almost every auto-configured `@Bean` method is annotated with `@ConditionalOnMissingBean`. If you define a bean of the same type, Spring registers yours first (it's scanned before auto-configs run because `DeferredImportSelector` delays auto-config imports). When the auto-config runs, `@ConditionalOnMissingBean` detects your bean and the auto-configured one backs off.

---

### Q3: What's the difference between `spring.factories` and `AutoConfiguration.imports`?
**A:** 
- **`spring.factories`** (pre Boot 2.7): a general-purpose file for many extension points. Auto-config classes were listed under `org.springframework.boot.autoconfigure.EnableAutoConfiguration` key. It could be misused as a dumping ground.
- **`AutoConfiguration.imports`** (Boot 2.7+): dedicated only to auto-configuration. More explicit, better tooling support. `spring.factories` still works for backward compatibility, but new code should use `AutoConfiguration.imports`.

---

### Q4: How do you exclude a specific auto-configuration?
**A:** Three ways:
```java
// 1. In the annotation
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })

// 2. In properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

// 3. In @EnableAutoConfiguration directly
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
```
Use case: running tests without a database, or when you want full manual control over a subsystem.

---

### Q5: How do you write your own auto-configuration for a library?
**A:**
1. Create a `@AutoConfiguration` class with appropriate `@Conditional*` guards.
2. Define `@Bean` methods with `@ConditionalOnMissingBean` so users can override.
3. Register it in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
4. Use `@AutoConfigureAfter` / `@AutoConfigureBefore` to control ordering relative to other auto-configs.

```java
@AutoConfiguration
@ConditionalOnClass(MyClient.class)
@ConditionalOnMissingBean(MyClient.class)
@EnableConfigurationProperties(MyClientProperties.class)
public class MyClientAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyClient myClient(MyClientProperties props) {
        return new MyClient(props.getUrl(), props.getApiKey());
    }
}
```

---

### Q6: What is the ordering of auto-configurations and why does it matter?
**A:** Auto-configs can depend on beans produced by other auto-configs. For example, `JpaRepositoriesAutoConfiguration` needs the `EntityManagerFactory` bean produced by `HibernateJpaAutoConfiguration`. Ordering annotations:
- `@AutoConfigureAfter(HibernateJpaAutoConfiguration.class)` — run after this class
- `@AutoConfigureBefore(...)` — run before this class
- `@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)` — numeric priority

Without correct ordering, a `@ConditionalOnBean` check could fail simply because the dependency hasn't been processed yet.

---

### Q7: What is a `DeferredImportSelector` and why is it used for auto-configuration?
**A:** `DeferredImportSelector` is a variant of `ImportSelector` that delays its `selectImports()` call until **all** `@Configuration` classes in the application context have been processed. Auto-configuration uses this so that user-defined beans are always registered before the auto-config conditions are evaluated. If a regular `ImportSelector` were used, auto-configs might run before user beans, causing `@ConditionalOnMissingBean` to incorrectly think no user bean exists.

---

### Q8: How does Spring Boot auto-configure an embedded Tomcat?
**A:** `ServletWebServerFactoryAutoConfiguration` is triggered when:
- `@ConditionalOnClass(ServletRequest.class)` — servlet API on classpath
- `@ConditionalOnWebApplication(type = SERVLET)` — running as a servlet app

It imports `EmbeddedTomcat` configuration which creates a `TomcatServletWebServerFactory` bean (guarded by `@ConditionalOnMissingBean`). `SpringApplication` then calls `factory.getWebServer(...)` to start Tomcat programmatically. There is no WAR deployment — the app is self-contained.

---

## 7. Common Pitfalls & Gotchas

**1. Forgetting that `@ConditionalOnMissingBean` is type-based by default**  
If you declare a `@Bean` with a subtype, the parent type might still trigger auto-config. Always check what type the condition is checking.

**2. Excluding auto-configs in the wrong place**  
Using `exclude` on a `@Configuration` class other than the primary `@SpringBootApplication` class sometimes doesn't work. Prefer the property-based exclusion or the main class annotation.

**3. `AutoConfiguration.imports` not on the right classpath**  
If you're writing a library, the file must be in `src/main/resources/META-INF/spring/`. Putting it in the wrong location silently causes the auto-config to never load.

**4. Ordering issues with `@ConditionalOnBean`**  
If you use `@ConditionalOnBean(SomeBean.class)` in an auto-config, and `SomeBean` is produced by another auto-config, you must use `@AutoConfigureAfter`. Otherwise the condition may evaluate before `SomeBean` exists.

**5. Auto-configuration in tests**  
`@SpringBootTest` loads all auto-configs. `@WebMvcTest` and `@DataJpaTest` use **slice** configurations that only load a subset. Using `@SpringBootTest` when you just want to test a controller is a common over-engineering mistake that slows test suites.

**6. `spring.factories` vs `AutoConfiguration.imports` confusion**  
In Boot 2.7+, if you put auto-config class names in `spring.factories` under `EnableAutoConfiguration`, it still works but triggers a deprecation warning. Migrate to `AutoConfiguration.imports`.

**7. Auto-config is not magic — it's conditional `@Configuration`**  
Beginners treat auto-config as mysterious. It's just `@Configuration` classes with `@Conditional` guards. You can read the source code of any auto-config in `spring-boot-autoconfigure-x.x.x.jar` to understand exactly what it does.

---

## 8. Summary

- Auto-configuration is triggered by `@EnableAutoConfiguration`, which loads `@Configuration` candidates from `AutoConfiguration.imports` via `AutoConfigurationImportSelector`.
- Each auto-config class is guarded by `@Conditional*` annotations — it only activates if conditions (classpath, existing beans, properties) are satisfied.
- `@ConditionalOnMissingBean` is the "back-off" mechanism — your beans always win over auto-configured ones.
- `DeferredImportSelector` ensures user beans are processed before auto-config conditions are evaluated.
- Use `--debug` or `logging.level...=DEBUG` to get the **Condition Evaluation Report** and debug which configs fired and why.
- To disable a specific auto-config: use `exclude` on `@SpringBootApplication` or the `spring.autoconfigure.exclude` property.
