# Spring IoC Container & Dependency Injection

> The core backbone of the Spring Framework — understanding this deeply separates a Spring *user* from a Spring *engineer*.

---

## Table of Contents

1. [Inversion of Control (IoC)](#1-inversion-of-control-ioc)
2. [BeanFactory](#2-beanfactory)
3. [ApplicationContext](#3-applicationcontext)
4. [BeanFactory vs ApplicationContext — Deep Comparison](#4-beanfactory-vs-applicationcontext--deep-comparison)
5. [Spring Beans & Lifecycle](#5-spring-beans--lifecycle)
4. [Bean Scopes](#4-bean-scopes)
6. [Bean Scopes](#6-bean-scopes)
7. [Dependency Injection Types](#7-dependency-injection-types)
8. [@Autowired Mechanics](#8-autowired-mechanics)
9. [Bean Configuration Approaches](#9-bean-configuration-approaches)
10. [@ComponentScan](#10-componentscan)
11. [Circular Dependency](#11-circular-dependency)
12. [Lazy Initialization](#12-lazy-initialization)
13. [BeanFactoryPostProcessor & BeanPostProcessor](#13-beanfactorypostprocessor--beanpostprocessor)
14. [Aware Interfaces](#14-aware-interfaces)
15. [FactoryBean vs @Bean](#15-factorybean-vs-bean)
16. [@Conditional — Conditional Bean Registration](#16-conditional--conditional-bean-registration)
17. [Profiles](#17-profiles)
18. [Environment Abstraction](#18-environment-abstraction)
19. [Common Interview Questions](#19-common-interview-questions)

---

## 1. Inversion of Control (IoC)

### Concept

**Inversion of Control** is a design principle where the control of object creation and dependency wiring is *inverted* — instead of a class creating its own dependencies (`new SomeDependency()`), an external container creates and injects them.

Without IoC:
```java
public class OrderService {
    // OrderService controls creation — tightly coupled
    private PaymentService paymentService = new PaymentService();
}
```

With IoC:
```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Container injects the dependency
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### Motivation

| Without IoC | With IoC |
|---|---|
| Tight coupling between classes | Loose coupling via interfaces |
| Hard to unit-test (can't mock deps) | Easy to mock and inject fakes in tests |
| Manual wiring across the whole app | Container manages wiring automatically |
| Object graph changes require code changes | Configuration changes suffice |

### Real-World Analogy

Think of a **restaurant kitchen**:
- **Without IoC**: Each chef goes to the market, buys their own ingredients, and cooks.
- **With IoC**: A central procurement team (the container) sources all ingredients and delivers them to each chef's station. Chefs just cook — they don't worry about sourcing.

The "procurement team" is the **Spring IoC container**. It knows what each bean needs and delivers it.

> **Interview Tip:** IoC is a *principle*; Dependency Injection is the *mechanism* that implements it. Don't conflate the two — interviewers often probe this distinction.

---

## 2. BeanFactory

`BeanFactory` is the **root interface** of the Spring IoC container hierarchy. It provides the fundamental contract for accessing beans from a Spring container — nothing more, nothing less.

### Interface Definition (simplified)

```java
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    boolean containsBean(String name);
    boolean isSingleton(String name);
    boolean isPrototype(String name);
    Class<?> getType(String name);
    String[] getAliases(String name);
}
```

### What BeanFactory Does

| Responsibility | Description |
|---|---|
| Bean definition registry | Reads and stores bean metadata (class, scope, dependencies) |
| Bean instantiation | Creates bean instances when requested |
| Dependency wiring | Injects constructor args and property values |
| Scope management | Handles singleton vs prototype lifecycles |
| Lazy loading | Instantiates singleton beans **only when first requested** via `getBean()` |

### Key Implementations

| Class | Description |
|---|---|
| `DefaultListableBeanFactory` | Core workhorse — used internally by all `ApplicationContext` impls |
| `XmlBeanFactory` | Legacy XML-backed factory (deprecated since Spring 3.1) |
| `StaticListableBeanFactory` | For pre-registered singletons; no bean creation logic |

### Direct Usage (rare)

```java
// Manually create and configure a BeanFactory
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("beans.xml"));

// BeanPostProcessors are NOT auto-registered — you must do it manually
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());

MyService svc = factory.getBean(MyService.class);
```

> **Key Limitation:** `BeanFactory` does **not** automatically register `BeanPostProcessor` or `BeanFactoryPostProcessor` beans. You must register them programmatically. This makes it error-prone for anything beyond trivial use cases.

### What BeanFactory Does NOT Do

- No i18n / `MessageSource` support
- No application event publishing
- No `@Autowired`, `@Value` processing out of the box (requires manual `BeanPostProcessor` registration)
- No AOP auto-proxy creation
- No `@Configuration` class processing
- No `@ComponentScan`

> **Interview Tip:** BeanFactory is the *low-level* plumbing. It is the contract; `DefaultListableBeanFactory` is the implementation. You will almost never use it directly in application code — but Spring itself uses it internally everywhere.

---

## 3. ApplicationContext

`ApplicationContext` **extends** `BeanFactory` and is the container you actually work with. It adds all the enterprise-grade features needed for real applications.

### Interface Hierarchy

```
BeanFactory
    └── ListableBeanFactory          (enumerate all bean names/types)
    └── HierarchicalBeanFactory      (parent-child context support)
            └── ApplicationContext
                    ├── MessageSource           (i18n)
                    ├── ApplicationEventPublisher (events)
                    ├── ResourcePatternResolver  (classpath scanning)
                    └── EnvironmentCapable       (profiles, properties)
```

### What ApplicationContext Adds Over BeanFactory

| Feature | Detail |
|---|---|
| **Eager singleton initialization** | All singleton beans are created at startup — fail-fast on misconfiguration |
| **Auto BeanPostProcessor registration** | Detects `BeanPostProcessor` beans in context and registers them automatically |
| **Auto BeanFactoryPostProcessor** | Same for `BeanFactoryPostProcessor` — e.g., `PropertySourcesPlaceholderConfigurer` |
| **MessageSource** | Resolves i18n messages via `ctx.getMessage(...)` |
| **ApplicationEventPublisher** | Publish/subscribe model via `ctx.publishEvent(...)` |
| **ResourceLoader** | Load classpath/file/URL resources |
| **Environment** | Unified access to properties, profiles, system env |
| **AOP auto-proxy** | `@Transactional`, `@Async`, `@Cacheable` etc. work automatically |
| **@Configuration processing** | Processes `@Bean`, `@Import`, `@ComponentScan` |

### Concrete Implementations

| Class | When to Use |
|---|---|
| `AnnotationConfigApplicationContext` | Standalone (non-web) apps with Java/annotation config |
| `AnnotationConfigWebApplicationContext` | Traditional servlet web apps (Spring MVC + `web.xml`) |
| `ClassPathXmlApplicationContext` | Legacy apps using XML bean config from classpath |
| `FileSystemXmlApplicationContext` | Legacy apps using XML bean config from file system |
| `GenericWebApplicationContext` | Programmatic web setup; rarely used directly |
| `AnnotationConfigServletWebServerApplicationContext` | Spring Boot servlet web app (auto-selected) |
| `AnnotationConfigReactiveWebServerApplicationContext` | Spring Boot reactive/WebFlux app (auto-selected) |

### Creating an ApplicationContext

```java
// 1. Annotation-based (standard Spring, no Boot)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// 2. Component scan shortcut
ApplicationContext ctx = new AnnotationConfigApplicationContext("com.example");

// 3. XML-based (legacy)
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");

// 4. Spring Boot — context type chosen automatically based on classpath
ConfigurableApplicationContext ctx = SpringApplication.run(MyApp.class, args);
```

### ApplicationContext Lifecycle Events

The context itself publishes built-in events you can listen to:

| Event | When Published |
|---|---|
| `ContextRefreshedEvent` | After context is fully initialized (all singletons ready) |
| `ContextStartedEvent` | After `ctx.start()` is called |
| `ContextStoppedEvent` | After `ctx.stop()` is called |
| `ContextClosedEvent` | After `ctx.close()` — all beans destroyed |
| `ApplicationReadyEvent` *(Boot)* | After Spring Boot is fully started and ready to serve |
| `ApplicationStartedEvent` *(Boot)* | After context refresh but before runners execute |

```java
@Component
public class StartupListener implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        System.out.println("Context refreshed — all beans are ready");
    }
}

// Modern alternative using @EventListener
@Component
public class StartupListener {
    @EventListener(ContextRefreshedEvent.class)
    public void onRefresh() {
        System.out.println("Context refreshed");
    }
}
```

### Parent–Child ApplicationContext Hierarchy

Spring supports nested contexts. A child context can see beans from its parent, but not vice versa.

```
Root ApplicationContext        ← shared beans: services, repos, datasource
        │
        └── DispatcherServlet Context   ← web-layer beans: controllers, view resolvers
```

```java
// Programmatic parent-child setup
AnnotationConfigApplicationContext parent = new AnnotationConfigApplicationContext(RootConfig.class);
AnnotationConfigApplicationContext child  = new AnnotationConfigApplicationContext();
child.setParent(parent);
child.register(WebConfig.class);
child.refresh();
```

> **Interview Tip:** In a classic Spring MVC app (pre-Boot), the `ContextLoaderListener` creates the **root** context and the `DispatcherServlet` creates a **child** context. This is why `@Transactional` must be in the root context to apply to service beans — the child context doesn't manage them.

### MessageSource — i18n

```java
@Configuration
public class AppConfig {
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
        ms.setBasename("messages");   // reads messages.properties, messages_fr.properties, etc.
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }
}

// Usage
@Autowired
private ApplicationContext ctx;

String msg = ctx.getMessage("welcome.message", null, Locale.FRENCH);
```

### Event Publishing

```java
// Custom event
public class OrderPlacedEvent extends ApplicationEvent {
    private final String orderId;
    public OrderPlacedEvent(Object source, String orderId) {
        super(source);
        this.orderId = orderId;
    }
    public String getOrderId() { return orderId; }
}

// Publisher
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void placeOrder(String orderId) {
        // business logic...
        publisher.publishEvent(new OrderPlacedEvent(this, orderId));
    }
}

// Listener
@Component
public class NotificationService {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Sending notification for order: " + event.getOrderId());
    }
}
```

> **Interview Tip:** `ApplicationEventPublisher` is synchronous by default. Use `@Async` on the listener method + `@EnableAsync` on a config class to make it asynchronous.

---

## 4. BeanFactory vs ApplicationContext — Deep Comparison

### Feature Matrix

| Feature | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| **Core interface** | Root IoC interface | Extends `BeanFactory` |
| **Bean instantiation timing** | Lazy — on first `getBean()` | Eager — all singletons at startup |
| **BeanPostProcessor auto-registration** | Manual | Automatic (detected from context) |
| **BeanFactoryPostProcessor auto-registration** | Manual | Automatic |
| **@Autowired / @Value processing** | Requires manual setup | Built-in via `AutowiredAnnotationBeanPostProcessor` |
| **@Configuration / @Bean processing** | Not supported | Built-in via `ConfigurationClassPostProcessor` |
| **Component scanning** | Not supported | Supported |
| **i18n (MessageSource)** | Not supported | Supported |
| **Application events** | Not supported | Supported |
| **AOP auto-proxy** | Not supported | Supported |
| **Environment / Profiles** | Not supported | Supported |
| **Resource loading** | Basic | Rich (`classpath:`, `file:`, `http:`) |
| **Startup failure detection** | Deferred (first `getBean()`) | Fail-fast at startup |
| **Typical use** | Internal Spring plumbing | Your application code — always |

### Startup Behavior Difference

```
BeanFactory:
  startup → reads bean definitions
  first getBean(A) → creates A, injects deps
  first getBean(B) → creates B, injects deps
  (misconfiguration discovered at runtime, lazily)

ApplicationContext:
  startup → reads bean definitions
           → creates ALL singletons eagerly
           → wires all dependencies
           → runs BeanPostProcessors
           → fail immediately on any error
  getBean(A) → returns already-created instance
```

This makes `ApplicationContext` **fail-fast** — a missing bean or misconfigured dependency blows up at startup, not silently at runtime under load.

### BeanPostProcessor Registration Difference

```java
// With BeanFactory — manual, error-prone
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// ...load definitions...
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new CommonAnnotationBeanPostProcessor());  // @PostConstruct, @PreDestroy

// With ApplicationContext — nothing to do; Spring detects and registers automatically
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// @Autowired, @PostConstruct, @PreDestroy, AOP proxies — all work automatically
```

### When Would You Ever Use BeanFactory Directly?

| Scenario | Reason |
|---|---|
| Framework/library authors | Building a lightweight embedded container with no enterprise features |
| Unit/integration tests with extremely minimal startup | Avoid full context overhead |
| Custom Spring extension code | Accessing the underlying `DefaultListableBeanFactory` via `ctx.getBeanFactory()` |

```java
// Accessing the underlying BeanFactory from an ApplicationContext
ConfigurableApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
ConfigurableListableBeanFactory beanFactory = ctx.getBeanFactory();

// Register a singleton manually (advanced use)
beanFactory.registerSingleton("myBean", new MyBean());
```

> **Interview Tip (the most common question):** *"What is the difference between BeanFactory and ApplicationContext?"*
> — **BeanFactory** is the low-level root interface: it reads bean definitions and creates/wires beans on demand (lazy). **ApplicationContext** extends BeanFactory and adds eager singleton initialization, auto-registration of post-processors, i18n, events, AOP, profiles, and component scanning. In all real applications you use `ApplicationContext`. The only time you deal with `BeanFactory` directly is when writing Spring framework internals or accessing the underlying factory via `ctx.getBeanFactory()`.

---

## 5. Spring Beans & Lifecycle

A **Spring bean** is simply an object whose lifecycle is managed by the Spring IoC container. Not every object in your app is a bean — only those registered with the container.

### Full Bean Lifecycle

```
1.  Bean Definition loaded (XML / @Bean / @Component scan)
2.  BeanFactoryPostProcessor runs (modifies bean definitions)
         ↓
3.  Bean Instantiation (constructor called)
         ↓
4.  Populate Properties (setter injection, field injection)
         ↓
5.  BeanNameAware.setBeanName()
         ↓
6.  BeanFactoryAware.setBeanFactory()
         ↓
7.  ApplicationContextAware.setApplicationContext()
         ↓
8.  BeanPostProcessor.postProcessBeforeInitialization()   ← AOP proxies often start here
         ↓
9.  @PostConstruct method
         ↓
10. InitializingBean.afterPropertiesSet()
         ↓
11. Custom init-method (from @Bean(initMethod="..."))
         ↓
12. BeanPostProcessor.postProcessAfterInitialization()    ← AOP proxy wrapping completes here
         ↓
13. *** BEAN IS READY — lives in context ***
         ↓
14. @PreDestroy method
         ↓
15. DisposableBean.destroy()
         ↓
16. Custom destroy-method (from @Bean(destroyMethod="..."))
```

### Code Example — All Hooks

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class OrderService implements
        BeanNameAware,
        BeanFactoryAware,
        ApplicationContextAware,
        InitializingBean,
        DisposableBean {

    public OrderService() {
        System.out.println("1. Constructor called");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("2. BeanNameAware → bean name: " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("3. BeanFactoryAware → factory injected");
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        System.out.println("4. ApplicationContextAware → context injected");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("5. @PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("6. InitializingBean.afterPropertiesSet()");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("7. @PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("8. DisposableBean.destroy()");
    }
}
```

### Init & Destroy via @Bean

```java
@Configuration
public class AppConfig {

    @Bean(initMethod = "init", destroyMethod = "cleanup")
    public PaymentService paymentService() {
        return new PaymentService();
    }
}

public class PaymentService {
    public void init() {
        System.out.println("Custom init-method");
    }
    public void cleanup() {
        System.out.println("Custom destroy-method");
    }
}
```

> **Interview Tip:** The order of init hooks is: `@PostConstruct` → `afterPropertiesSet()` → `init-method`. The order of destroy hooks is: `@PreDestroy` → `destroy()` → `destroyMethod`. Prefer `@PostConstruct`/`@PreDestroy` — they are JSR-250 standard, not Spring-specific.

> **Gotcha:** For **prototype** beans, Spring calls init hooks but does **NOT** call destroy hooks. The container doesn't track prototype instances after handing them out. You are responsible for cleanup.

---

## 6. Bean Scopes

| Scope | Instances | Lifecycle owner | Available in |
|---|---|---|---|
| `singleton` | One per container | Container | All contexts |
| `prototype` | New instance per `getBean()` | Caller | All contexts |
| `request` | One per HTTP request | Container | Web contexts |
| `session` | One per HTTP session | Container | Web contexts |
| `application` | One per `ServletContext` | Container | Web contexts |
| `websocket` | One per WebSocket session | Container | WebSocket contexts |

```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.web.context.WebApplicationContext;

@Component
@Scope("singleton")         // default — can be omitted
public class ConfigService { }

@Component
@Scope("prototype")
public class CartItem { }

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestTracker { }

@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession { }
```

### The Scope Mismatch Problem

#### What Is the Problem?

Every Spring bean has a **lifecycle**. A singleton bean lives for the entire lifetime of the application context. A request-scoped bean lives only for a single HTTP request. A session-scoped bean lives for a single user's HTTP session.

The scope mismatch problem occurs when you **inject a shorter-lived bean into a longer-lived bean**. Spring resolves constructor/field/setter dependencies **once at startup** when the singleton is created. At that moment, there may be no active HTTP request at all — so Spring either fails, or worse, injects and freezes a single instance of the shorter-lived bean inside the singleton forever.

```
Singleton lifecycle:  |====== entire app lifetime ============|
Request scope:        |--req1--|  |--req2--|  |--req3--|

Singleton is created at startup → Spring injects RequestTracker
→ which RequestTracker instance? There's no request yet!
→ Even if there was, Request #1's tracker gets locked in forever
→ Every subsequent request uses Request #1's stale tracker
```

#### Concrete Bug Example

```java
// A request-scoped bean — should hold data for one HTTP request only
@Component
@Scope(WebApplicationContext.SCOPE_REQUEST)   // NO proxyMode — this is the broken version
public class RequestContext {
    private String correlationId;
    private String userId;
    // getters/setters
}

// A singleton service — lives forever
@Service
public class AuditService {

    @Autowired
    private RequestContext requestContext;  // INJECTED ONCE AT STARTUP — WRONG!

    public void audit(String action) {
        // BUG: requestContext is the SAME stale object for every request
        // correlationId and userId from request #1 will appear in request #100's audit logs
        log.info("User {} performed {} [correlationId={}]",
            requestContext.getUserId(),       // always request #1's userId
            action,
            requestContext.getCorrelationId() // always request #1's correlationId
        );
    }
}
```

**What actually happens at startup:**

```
ApplicationContext starts
  → Creates AuditService (singleton)
  → Tries to inject RequestContext
  → No HTTP request is active
  → Spring throws: java.lang.IllegalStateException:
      No thread-bound request found: Are you referring to request attributes
      outside of an actual web request?
```

Even if startup doesn't crash (e.g., in some lazy scenarios), the bug manifests as **data bleed between requests** — one user's session data leaking into another user's audit trail. In a banking or healthcare app, this is a critical security defect.

#### Why It Happens — The Root Cause

Spring dependency injection is performed **once per singleton bean, at context initialization time**. The container resolves all injection points at that moment and stores the resolved reference. There is no mechanism in basic DI to say "re-resolve this reference on every method call".

```
Startup (time T=0):
  AuditService created
  → @Autowired RequestContext resolved → RequestContext instance #A stored in field

Request 1 (time T=10):
  AuditService.audit() called
  → uses RequestContext instance #A  ← correct for request 1, but only by coincidence

Request 2 (time T=20):
  AuditService.audit() called
  → still uses RequestContext instance #A  ← WRONG, should be a new instance for request 2
```

#### The Fix: `proxyMode = ScopedProxyMode.TARGET_CLASS`

The solution is to tell Spring: *"don't inject the real bean — inject a proxy. Every time a method is called on the proxy, look up the real bean for the current thread/request/session and delegate to it."*

```java
@Component
@Scope(
    value = WebApplicationContext.SCOPE_REQUEST,
    proxyMode = ScopedProxyMode.TARGET_CLASS   // ← this is the fix
)
public class RequestContext {
    private String correlationId = UUID.randomUUID().toString();
    private String userId;
    // getters/setters
}
```

Now Spring injects a **CGLIB proxy** into `AuditService` instead of a real `RequestContext` instance. The proxy holds no data — it is a thin delegation shell. Every time any method on the proxy is called, it:

1. Looks up the current thread's `RequestAttributes` from `RequestContextHolder`
2. Finds (or creates) the real `RequestContext` for the current HTTP request
3. Delegates the method call to that real instance
4. Returns the result

```
Startup (T=0):
  AuditService created
  → @Autowired resolved → CGLIB proxy injected into field (proxy has no real data)

Request 1 (T=10, Thread A):
  AuditService.audit() called
  → proxy.getUserId() called
  → proxy looks up RequestContextHolder for Thread A
  → finds (or creates) RequestContext#A for this request
  → returns RequestContext#A.getUserId() → "alice"

Request 2 (T=20, Thread B):
  AuditService.audit() called
  → proxy.getUserId() called
  → proxy looks up RequestContextHolder for Thread B
  → finds (or creates) RequestContext#B for this request
  → returns RequestContext#B.getUserId() → "bob"
```

Each request gets its own `RequestContext` instance. No data bleed. The singleton `AuditService` holds only one proxy reference, yet effectively gets a fresh `RequestContext` per call.

#### How the Proxy Works Internally

```
AuditService.auditService field
        │
        ▼
  RequestContext$$SpringCGLIB$$0   ← CGLIB-generated proxy class (subclass of RequestContext)
        │
        │  on every method call:
        │    1. RequestContextHolder.currentRequestAttributes()
        │    2. getAttribute("scopedTarget.requestContext", REQUEST_SCOPE)
        │    3. if null → create new RequestContext, store it
        │    4. delegate to real RequestContext instance
        ▼
  Real RequestContext instance
  (stored in request attributes, lives and dies with the HTTP request)
```

The proxy is created by `ScopedProxyFactoryBean`, which wraps the target bean definition and generates a CGLIB subclass at startup. The real bean is stored under the key `scopedTarget.<beanName>` in the appropriate scope store (request attributes, session attributes, etc.).

#### proxyMode Options

| `ScopedProxyMode` | When to Use |
|---|---|
| `NO` (default) | Bean is not proxied — **causes the mismatch bug** if injected into a longer-lived bean |
| `TARGET_CLASS` | CGLIB proxy — works for any class (with or without interfaces). **Most common fix.** |
| `INTERFACES` | JDK dynamic proxy — only works if the bean implements at least one interface |

```java
// Use TARGET_CLASS (works always)
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)

// Use INTERFACES only if the bean implements an interface AND you want a JDK proxy
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.INTERFACES)
public class UserSessionBean implements UserSession { }
```

#### All Mismatch Combinations

| Injecting (shorter-lived) | Into (longer-lived) | Problem? | Fix |
|---|---|---|---|
| `prototype` | `singleton` | Yes — prototype injected once, reused forever | `@Lookup`, `ObjectProvider`, or `ApplicationContext.getBean()` |
| `request` | `singleton` | Yes — no request at startup | `proxyMode = TARGET_CLASS` |
| `request` | `session` | Yes — request dies before session | `proxyMode = TARGET_CLASS` |
| `session` | `singleton` | Yes — session dies, singleton keeps stale ref | `proxyMode = TARGET_CLASS` |
| `singleton` | `request` | No — singleton lives longer; always available | N/A |
| `singleton` | `session` | No | N/A |
| `prototype` | `request` | No — request is shorter-lived, prototype re-injected per request bean creation | N/A |

#### The Prototype-in-Singleton Special Case

Prototype is not a web scope, but it suffers the same mismatch problem. `proxyMode` doesn't help for prototype because a proxy would return the same prototype instance (the one created when the proxy was first used). Instead:

**Option 1: `@Lookup` method injection**

```java
@Component
public abstract class OrderProcessor {

    // Spring CGLIB-overrides this method to return a NEW CartSession each call
    @Lookup
    public abstract CartSession getCartSession();

    public void processOrder(String orderId) {
        CartSession session = getCartSession();  // fresh prototype every time
        session.setOrderId(orderId);
        // ...
    }
}
```

**Option 2: `ObjectProvider<T>` (recommended — works on concrete classes)**

```java
@Service
public class OrderProcessor {

    private final ObjectProvider<CartSession> cartSessionProvider;

    public OrderProcessor(ObjectProvider<CartSession> cartSessionProvider) {
        this.cartSessionProvider = cartSessionProvider;
    }

    public void processOrder(String orderId) {
        CartSession session = cartSessionProvider.getObject();  // new prototype each call
        session.setOrderId(orderId);
    }
}
```

**Option 3: `ApplicationContext.getBean()`** (least elegant — couples to Spring API)

```java
@Service
public class OrderProcessor implements ApplicationContextAware {

    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void processOrder(String orderId) {
        CartSession session = ctx.getBean(CartSession.class);  // new prototype each call
    }
}
```

#### Complete Working Example

```java
// Request-scoped bean with proxy — correctly holds per-request data
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {

    private final String correlationId = UUID.randomUUID().toString();
    private String authenticatedUserId;

    public String getCorrelationId() { return correlationId; }
    public String getAuthenticatedUserId() { return authenticatedUserId; }
    public void setAuthenticatedUserId(String id) { this.authenticatedUserId = id; }
}

// Singleton service — safely uses request-scoped data via proxy
@Service
public class AuditService {

    private final RequestContext requestContext;  // holds a CGLIB proxy

    public AuditService(RequestContext requestContext) {
        this.requestContext = requestContext;  // proxy injected once
    }

    public void audit(String action) {
        // Each call transparently resolves to the CURRENT request's RequestContext
        log.info("[{}] User '{}' performed '{}'",
            requestContext.getCorrelationId(),       // current request's correlationId
            requestContext.getAuthenticatedUserId(), // current request's userId
            action
        );
    }
}

// Servlet filter sets user on the request-scoped bean
@Component
public class AuthFilter extends OncePerRequestFilter {

    private final RequestContext requestContext;  // also gets the proxy

    public AuthFilter(RequestContext requestContext) {
        this.requestContext = requestContext;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String userId = extractUserFromToken(req.getHeader("Authorization"));
        requestContext.setAuthenticatedUserId(userId);  // sets on THIS request's instance
        chain.doFilter(req, res);
    }
}

// Controller — also safely uses the request-scoped bean
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final AuditService auditService;

    public OrderController(AuditService auditService) {
        this.auditService = auditService;
    }

    @PostMapping
    public ResponseEntity<Order> placeOrder(@RequestBody OrderRequest req) {
        auditService.audit("PLACE_ORDER");  // logs current request's user + correlationId
        // ...
        return ResponseEntity.ok(order);
    }
}
```

> **Interview Tip:** There are two distinct mismatch problems — **web-scoped beans** (fix: `proxyMode`) and **prototype beans** (fix: `@Lookup` or `ObjectProvider`). Don't mix up the solutions. `proxyMode` on a prototype bean does NOT give you a new instance per call — it gives you the same proxy-wrapped instance created on first use.

> **Gotcha:** If you call `requestContext.getClass()` on the injected proxy, you'll get something like `com.example.RequestContext$$SpringCGLIB$$0` — not `RequestContext`. This can break `instanceof` checks and serialization. Always check the actual bean class via `AopUtils.getTargetClass(proxy)` if needed.

---

## 7. Dependency Injection Types

### Constructor Injection (Preferred)

```java
@Service
public class OrderService {

    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    // @Autowired is optional if there's exactly one constructor (Spring 4.3+)
    public OrderService(PaymentService paymentService, InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
}
```

### Setter Injection

```java
@Service
public class OrderService {

    private NotificationService notificationService;

    @Autowired
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

### Field Injection (Avoid in Production Code)

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService; // No constructor, no setter

    @Autowired
    private InventoryService inventoryService;
}
```

### Comparison Table

| Criteria | Constructor | Setter | Field |
|---|---|---|---|
| Immutability | ✅ `final` fields | ❌ Mutable | ❌ Mutable |
| Testability | ✅ Plain `new` in tests | ✅ Call setter in tests | ❌ Needs reflection or Spring context |
| Mandatory deps | ✅ Enforced at compile time | ❌ Optional — can forget to inject | ❌ Fails only at runtime |
| Circular dependency | ❌ Fails fast (good!) | ✅ Can tolerate (Spring setter-injects) | ✅ Can tolerate (Spring field-injects) |
| Readability | ✅ All deps visible in one place | ⚠️ Scattered | ❌ Hidden inside class |
| Spring recommendation | ✅ **Preferred** | Use for optional deps | Avoid in prod code |

> **Interview Tip:** Spring's own documentation (since Spring 4) recommends constructor injection. The main arguments: immutability, testability without Spring, and "fail-fast" on missing dependencies. Field injection is fine in tests (`@SpringBootTest`) but avoid in production beans.

---

## 8. @Autowired Mechanics

### How Spring Resolves Beans

Spring uses a **3-step resolution** process:

1. **By type** — find all beans matching the injection point type.
2. **By `@Primary`** — if multiple candidates, pick the one marked `@Primary`.
3. **By name / `@Qualifier`** — if still ambiguous, match by bean name or explicit qualifier.

```
Injection point: PaymentService
  → Step 1: Find all beans of type PaymentService
       → [CreditCardPaymentService, PayPalPaymentService]
  → Step 2: Is one @Primary?
       → Yes → inject that one
  → Step 3: Does @Qualifier match?
       → Use @Qualifier("creditCard") → inject CreditCardPaymentService
  → Else: NoUniqueBeanDefinitionException
```

### @Primary

```java
@Service
@Primary  // default choice when there are multiple PaymentService beans
public class CreditCardPaymentService implements PaymentService { }

@Service
public class PayPalPaymentService implements PaymentService { }

@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // Gets CreditCardPaymentService
}
```

### @Qualifier

```java
@Service
@Qualifier("paypal")
public class PayPalPaymentService implements PaymentService { }

@Service
public class OrderService {
    @Autowired
    @Qualifier("paypal")
    private PaymentService paymentService; // Explicitly selects PayPal
}
```

### @Autowired vs @Resource vs @Inject

| Annotation | Package | Resolves by | Qualifier support | Spring-specific? |
|---|---|---|---|---|
| `@Autowired` | `org.springframework` | Type first, then name | `@Qualifier` | Yes |
| `@Resource` | `jakarta.annotation` (JSR-250) | Name first, then type | `name` attribute | No — JSR standard |
| `@Inject` | `jakarta.inject` (JSR-330) | Type first, then name | `@Named` | No — JSR standard |

```java
// @Autowired — Spring native
@Autowired
@Qualifier("paypal")
private PaymentService paymentService;

// @Resource — resolves by name first
@Resource(name = "payPalPaymentService")
private PaymentService paymentService;

// @Inject — JSR-330, interchangeable with @Autowired
@Inject
@Named("payPalPaymentService")
private PaymentService paymentService;
```

> **Interview Tip:** `@Resource` does **name-first** lookup, making it convenient when the field name matches the bean name. `@Autowired` does **type-first**. If you want portable code (not Spring-specific), use `@Inject` + `@Named`.

### @Autowired on Collections

```java
@Service
public class PaymentRouter {

    @Autowired
    private List<PaymentService> allPaymentServices; // Spring injects ALL beans of this type

    @Autowired
    private Map<String, PaymentService> paymentServiceMap; // key = bean name
}
```

---

## 9. Bean Configuration Approaches

### 1. XML Configuration (Legacy)

```xml
<!-- applicationContext.xml -->
<beans>
    <bean id="paymentService" class="com.example.CreditCardPaymentService"/>

    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="paymentService"/>
    </bean>
</beans>
```

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
```

### 2. Java @Configuration + @Bean (Explicit Wiring)

```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new CreditCardPaymentService();
    }

    @Bean
    public OrderService orderService(PaymentService paymentService) {
        return new OrderService(paymentService);  // constructor injection
    }
}
```

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
```

> **Gotcha:** `@Configuration` classes use CGLIB subclassing to make `@Bean` methods return the *same* singleton instance. If you call `paymentService()` directly inside the config class, you still get the singleton — not a new object. This does NOT apply to `@Component` classes with `@Bean` methods (`lite mode` — no proxy, direct call creates new instance).

### 3. Component Scanning (Implicit Wiring)

| Annotation | Layer | Extends |
|---|---|---|
| `@Component` | Generic Spring bean | — |
| `@Service` | Business/service layer | `@Component` |
| `@Repository` | Persistence/DAO layer | `@Component` + exception translation |
| `@Controller` | MVC controller (returns views) | `@Component` |
| `@RestController` | REST controller | `@Controller` + `@ResponseBody` |
| `@Configuration` | Config class | `@Component` |

```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

@Repository
public class OrderRepository {
    // Spring Data JPA or JdbcTemplate usage here
}
```

> **Interview Tip:** `@Repository` is special — Spring's `PersistenceExceptionTranslationPostProcessor` translates JPA/Hibernate-specific exceptions into Spring's `DataAccessException` hierarchy. `@Service` and `@Component` get no such treatment.

---

## 10. @ComponentScan

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",                    // scan this package and sub-packages
    basePackageClasses = OrderService.class,         // type-safe alternative to string packages
    includeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = MyCustomAnnotation.class)
    },
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX, pattern = ".*Legacy.*"),
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = LegacyService.class)
    }
)
public class AppConfig { }
```

### Filter Types

| FilterType | Description |
|---|---|
| `ANNOTATION` | Beans with a specific annotation |
| `ASSIGNABLE_TYPE` | Beans of a specific type (class/interface) |
| `ASPECTJ` | AspectJ expression |
| `REGEX` | Regex matching class name |
| `CUSTOM` | Implement `TypeFilter` interface |

> **Interview Tip:** `@SpringBootApplication` is a composed annotation that includes `@ComponentScan`. By default it scans the package of the main class and all sub-packages. This is why placing the main class at the root package is a Spring Boot convention.

---

## 11. Circular Dependency

### What It Is

A circular dependency occurs when Bean A requires Bean B, and Bean B (directly or transitively) requires Bean A.

```java
@Service
public class ServiceA {
    public ServiceA(ServiceB serviceB) { } // A needs B
}

@Service
public class ServiceB {
    public ServiceB(ServiceA serviceA) { } // B needs A → CYCLE
}
```

### How Spring Handles It

**For Singleton beans with Constructor Injection:**
Spring detects the cycle during startup and throws `BeanCurrentlyInCreationException`. This is **intentional fail-fast behavior** — constructor injection cannot break cycles because the full object must be created before injection.

**For Singleton beans with Setter/Field Injection:**
Spring *can* handle the cycle using a **3-level cache**:
1. `singletonObjects` — fully initialized beans
2. `earlySingletonObjects` — early-exposed beans (post-processed but not fully initialized)
3. `singletonFactories` — factory lambdas to create early references

Spring first creates A (partially), exposes it via `singletonFactories`, creates B (which gets the partial A), finishes B, then finishes A. This works — but the beans are in an inconsistent state during initialization, which is why Spring 6+ warns about this pattern and constructor injection is preferred.

### How to Break Circular Dependency

**Option 1: Restructure (Best)** — Extract a shared dependency into a new class.

**Option 2: Use `@Lazy` on one injection point**

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;

    public ServiceA(@Lazy ServiceB serviceB) {
        this.serviceB = serviceB;  // serviceB is a proxy; real bean created on first use
    }
}
```

**Option 3: Use `@PostConstruct` with setter injection**

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB; // field injection to break constructor cycle
}
```

**Option 4: Use `ApplicationContext` to look up the bean at runtime** (rarely ideal)

> **Interview Tip:** In Spring Boot 2.6+, circular dependencies via field/setter injection are *disabled by default* (`spring.main.allow-circular-references=false`). This is a strong hint that circular deps are a design smell. The correct fix is always to redesign — extract the overlapping concern.

---

## 12. Lazy Initialization

By default, singleton beans are **eagerly initialized** at startup. Lazy initialization defers bean creation until the first `getBean()` call or the first injection point is accessed.

### Per-bean with @Lazy

```java
@Service
@Lazy
public class HeavyReportingService {
    public HeavyReportingService() {
        System.out.println("HeavyReportingService created");  // Only when first used
    }
}
```

```java
@Service
public class DashboardService {
    private final HeavyReportingService reportingService;

    // If injecting a @Lazy bean, you ALSO need @Lazy here for the proxy to work
    public DashboardService(@Lazy HeavyReportingService reportingService) {
        this.reportingService = reportingService;
    }
}
```

### Global Lazy Initialization (Spring Boot)

```yaml
# application.yml
spring:
  main:
    lazy-initialization: true
```

> **Interview Tip:** Global lazy init reduces startup time but shifts errors. Misconfigured beans that would fail at startup now fail at first use in production. Use per-bean `@Lazy` for genuinely expensive optional beans. Don't enable it globally just to speed up startup without understanding the trade-off.

---

## 13. BeanFactoryPostProcessor & BeanPostProcessor

These are two of the most powerful extension points in Spring, and they run at very different stages.

### Stage Comparison

```
Bean Definitions loaded
        ↓
BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()  ← add/modify definitions
        ↓
BeanFactoryPostProcessor.postProcessBeanFactory()                        ← modify bean definitions (e.g., change property values)
        ↓
Bean Instantiation begins
        ↓
BeanPostProcessor.postProcessBeforeInitialization()                      ← runs on each bean after instantiation
        ↓
@PostConstruct / afterPropertiesSet / init-method
        ↓
BeanPostProcessor.postProcessAfterInitialization()                       ← AOP proxy wrapping happens here
```

### BeanFactoryPostProcessor

Modifies **bean definitions** (metadata) before any beans are instantiated. Classic use: property placeholder resolution.

```java
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        BeanDefinition bd = beanFactory.getBeanDefinition("orderService");
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);  // change scope programmatically
        System.out.println("Modified orderService bean definition scope to prototype");
    }
}
```

**Built-in example:** `PropertySourcesPlaceholderConfigurer` is a `BeanFactoryPostProcessor` that resolves `${...}` placeholders in `@Value`.

### BeanDefinitionRegistryPostProcessor

Extends `BeanFactoryPostProcessor` — allows **registering new bean definitions** dynamically.

```java
@Component
public class DynamicBeanRegistrar implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        GenericBeanDefinition bd = new GenericBeanDefinition();
        bd.setBeanClass(AuditService.class);
        registry.registerBeanDefinition("dynamicAuditService", bd);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // optional additional processing
    }
}
```

**Spring Boot uses this:** `ConfigurationClassPostProcessor` (a `BeanDefinitionRegistryPostProcessor`) is responsible for processing `@Configuration`, `@ComponentScan`, `@Import`, and `@Bean` — it is the engine that builds the entire bean definition graph.

### BeanPostProcessor

Operates on **bean instances** after instantiation. Runs on every single bean. This is how Spring implements AOP — `AbstractAutoProxyCreator` is a `BeanPostProcessor` that wraps beans in CGLIB/JDK proxies.

```java
@Component
public class TimingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean; // MUST return the bean (or a wrapper/proxy)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Can return a CGLIB proxy here — this is what Spring AOP does
        System.out.println("After init: " + beanName);
        return bean;
    }
}
```

> **Interview Tip:** A very common question: "Where does Spring AOP create the proxy?" — Answer: in `BeanPostProcessor.postProcessAfterInitialization()`. Specifically `AbstractAutoProxyCreator` checks if a bean should be proxied based on pointcut matching and, if yes, returns a proxy instead of the original bean.

> **Gotcha:** `BeanPostProcessor` beans themselves are instantiated **before** regular beans and before other `BeanPostProcessor`s. Do not make a `BeanPostProcessor` depend on a regular bean — it creates complex ordering issues. Spring will warn you with "Bean X is not eligible for auto-proxying" if this happens.

---

## 14. Aware Interfaces

Aware interfaces let beans access Spring infrastructure. Use sparingly — tight coupling to Spring infrastructure should be avoided in business logic.

| Interface | Method | What you get |
|---|---|---|
| `BeanNameAware` | `setBeanName(String)` | The bean's name in the container |
| `BeanFactoryAware` | `setBeanFactory(BeanFactory)` | The owning `BeanFactory` |
| `ApplicationContextAware` | `setApplicationContext(ApplicationContext)` | The full context |
| `EnvironmentAware` | `setEnvironment(Environment)` | Active profiles, properties |
| `ResourceLoaderAware` | `setResourceLoader(ResourceLoader)` | Load classpath/filesystem resources |
| `MessageSourceAware` | `setMessageSource(MessageSource)` | i18n message resolution |

```java
@Component
public class ContextHolder implements ApplicationContextAware {

    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        ContextHolder.context = applicationContext;
    }

    // Utility: programmatic bean lookup — useful in non-Spring-managed classes
    public static <T> T getBean(Class<T> type) {
        return context.getBean(type);
    }
}
```

```java
@Component
public class FeatureToggleService implements EnvironmentAware {

    private Environment environment;

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public boolean isFeatureEnabled(String feature) {
        return environment.getProperty("feature." + feature, Boolean.class, false);
    }
}
```

> **Interview Tip:** The `ContextHolder` pattern above (static `ApplicationContext` holder) is a common solution for accessing Spring beans from non-Spring-managed code (e.g., JPA entity listeners, static utility classes). It's a pragmatic pattern but overusing it defeats the purpose of IoC.

---

## 15. FactoryBean vs @Bean

### @Bean (Standard)

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        return ds;
    }
}
```

### FactoryBean (Advanced — for complex/framework objects)

`FactoryBean<T>` is a Spring interface that acts as a factory for creating complex objects. When Spring encounters a `FactoryBean`, it calls `getObject()` instead of using the bean directly.

```java
@Component("connectionPool")
public class ConnectionPoolFactoryBean implements FactoryBean<DataSource> {

    @Override
    public DataSource getObject() throws Exception {
        // complex initialization logic
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        ds.setMaximumPoolSize(20);
        return ds;
    }

    @Override
    public Class<?> getObjectType() {
        return DataSource.class;
    }

    @Override
    public boolean isSingleton() {
        return true; // DataSource should be singleton
    }
}
```

```java
// Getting the product (DataSource)
DataSource ds = ctx.getBean("connectionPool", DataSource.class);

// Getting the FactoryBean itself — prefix with &
ConnectionPoolFactoryBean factory = ctx.getBean("&connectionPool", ConnectionPoolFactoryBean.class);
```

### Comparison

| | `@Bean` | `FactoryBean` |
|---|---|---|
| Verbosity | Simple, inline | More boilerplate |
| Reusability | Per config class | Reusable component |
| Accessing the factory itself | N/A | Prefix bean name with `&` |
| Used by Spring internally | No | Yes — `JndiObjectFactoryBean`, `ProxyFactoryBean`, `SqlSessionFactoryBean` (MyBatis) |

> **Interview Tip:** `FactoryBean` is mostly a legacy pattern from pre-annotation Spring. Today you almost always use `@Bean`. But knowing the `&` prefix trick for getting the `FactoryBean` itself is a classic interview stumper.

---

## 16. @Conditional — Conditional Bean Registration

`@Conditional` lets you register beans only when certain conditions are met at startup.

### Built-in Conditions (Spring Boot)

| Annotation | Registers bean when... |
|---|---|
| `@ConditionalOnProperty` | A property has a specific value |
| `@ConditionalOnClass` | A class is on the classpath |
| `@ConditionalOnMissingClass` | A class is NOT on the classpath |
| `@ConditionalOnBean` | Another bean already exists |
| `@ConditionalOnMissingBean` | No bean of that type exists yet |
| `@ConditionalOnWebApplication` | App is a web application |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |

```java
@Configuration
public class CacheConfig {

    // Only activate Redis cache if spring.cache.type=redis
    @Bean
    @ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")
    public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.create(factory);
    }

    // Fallback in-memory cache if no CacheManager exists yet
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public ConcurrentMapCacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager("orders", "products");
    }
}
```

### Custom Condition

```java
public class OnProductionCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String profile = context.getEnvironment().getProperty("spring.profiles.active", "");
        return profile.contains("prod");
    }
}

@Bean
@Conditional(OnProductionCondition.class)
public MonitoringService monitoringService() {
    return new DatadogMonitoringService();
}
```

> **Interview Tip:** `@ConditionalOnMissingBean` is the cornerstone of Spring Boot auto-configuration's "back off" pattern — it registers a default bean only if the user hasn't defined their own. This is how you override auto-configured beans.

---

## 17. Profiles

Profiles let you register different beans for different environments (dev, test, prod).

### @Profile on Beans

```java
@Service
@Profile("dev")
public class MockPaymentService implements PaymentService {
    public String charge(double amount) { return "MOCK_OK"; }
}

@Service
@Profile("prod")
public class StripePaymentService implements PaymentService {
    public String charge(double amount) { /* real Stripe call */ return "REAL_OK"; }
}

@Service
@Profile("!prod")   // any profile except prod
public class LoggingPaymentDecorator implements PaymentService { }
```

### @Profile on @Configuration Classes

```java
@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // Production connection pool
        return new HikariDataSource(productionConfig());
    }
}

@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

### Activating Profiles

```yaml
# application.yml
spring:
  profiles:
    active: dev
```

```bash
# JVM system property
java -Dspring.profiles.active=prod -jar app.jar

# Environment variable
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

### Default Profile

```java
@Service
@Profile("default")  // Active only when NO profile is explicitly set
public class DefaultPaymentService implements PaymentService { }
```

### Profile-specific Property Files

Spring Boot auto-loads `application-{profile}.yml`:
```
application.yml          ← base config (always loaded)
application-dev.yml      ← loaded when dev profile is active
application-prod.yml     ← loaded when prod profile is active
```

Profile-specific files **override** base `application.yml` values.

> **Interview Tip:** `@Profile("!prod")` — the `!` negation operator is handy and often overlooked. Also: multiple active profiles are supported (`spring.profiles.active=dev,feature-x`). A bean with `@Profile({"dev","test"})` is active if *either* profile is active.

---

## 18. Environment Abstraction

The `Environment` interface unifies access to profiles and properties from any source.

### @Value — Simple Property Injection

```java
@Service
public class PaymentService {

    @Value("${payment.gateway.url}")
    private String gatewayUrl;

    @Value("${payment.timeout:5000}")   // default value: 5000
    private int timeoutMs;

    @Value("${payment.retry.enabled:true}")
    private boolean retryEnabled;

    @Value("#{systemProperties['java.home']}")  // SpEL expression
    private String javaHome;
}
```

### @ConfigurationProperties — Structured Binding (Preferred for Groups)

```java
@Configuration
@ConfigurationProperties(prefix = "payment")
@Validated  // enables JSR-380 validation on properties
public class PaymentProperties {

    @NotBlank
    private String gatewayUrl;

    @Min(1000) @Max(30000)
    private int timeoutMs = 5000;

    private Retry retry = new Retry();

    // getters and setters required (or use @ConstructorBinding for records)
    // ...

    public static class Retry {
        private boolean enabled = true;
        private int maxAttempts = 3;
        // getters/setters
    }
}
```

```yaml
# application.yml
payment:
  gateway-url: https://api.stripe.com
  timeout-ms: 10000
  retry:
    enabled: true
    max-attempts: 5
```

```java
@Service
public class PaymentService {
    private final PaymentProperties props;  // injected as a regular bean

    public PaymentService(PaymentProperties props) {
        this.props = props;
    }
}
```

### PropertySource Hierarchy (Highest to Lowest Priority)

```
1. Command-line arguments (--server.port=9090)
2. JNDI attributes (java:comp/env)
3. Java system properties (-Dserver.port=9090)
4. OS environment variables (SERVER_PORT=9090)
5. Profile-specific application files (application-prod.yml)
6. application.yml / application.properties
7. @PropertySource annotations
8. Default properties (SpringApplication.setDefaultProperties)
```

### Programmatic Environment Access

```java
@Component
public class FeatureFlags implements EnvironmentAware {

    private Environment env;

    @Override
    public void setEnvironment(Environment env) { this.env = env; }

    public boolean isNewCheckoutEnabled() {
        return env.getProperty("feature.new-checkout", Boolean.class, false);
    }

    public boolean isProductionProfile() {
        return env.acceptsProfiles(Profiles.of("prod"));
    }
}
```

> **Interview Tip:** `@Value` is for simple scalar properties. `@ConfigurationProperties` is for a group of related properties, supports type-safe binding, nested objects, validation, and relaxed binding (camelCase = kebab-case = UPPER_SNAKE_CASE). Always prefer `@ConfigurationProperties` for feature flags or service configs with multiple fields.

---

## 19. Common Interview Questions

---

### Q1: What is the difference between BeanFactory and ApplicationContext?

**Answer:** `BeanFactory` is the root interface providing basic IoC — bean instantiation and dependency injection with lazy initialization. `ApplicationContext` extends it with: eager singleton initialization, i18n via `MessageSource`, event publication via `ApplicationEventPublisher`, AOP integration, automatic `BeanPostProcessor`/`BeanFactoryPostProcessor` registration, and `ResourceLoader` support. In practice, always use `ApplicationContext` (`BeanFactory` is for memory-constrained/embedded scenarios only).

---

### Q2: What happens when two beans of the same type are defined and you @Autowire without a qualifier?

**Answer:** Spring throws `NoUniqueBeanDefinitionException` at startup. Resolution options:
1. Mark one bean `@Primary`
2. Use `@Qualifier("beanName")` at the injection point
3. Change field name to match the desired bean name (Spring falls back to name matching)
4. Inject `List<PaymentService>` to get all of them

---

### Q3: What is the difference between @Component, @Service, @Repository, and @Controller?

**Answer:** All four are `@Component` specializations and trigger component scanning. The differences are **semantic + one functional difference**:
- `@Repository`: Spring registers a `PersistenceExceptionTranslationPostProcessor` that translates JPA/Hibernate-specific exceptions into Spring's `DataAccessException`.
- `@Controller` + `@RestController`: recognized by Spring MVC's `DispatcherServlet` for request mapping.
- `@Service`: purely semantic — no extra behavior. Signals business logic layer.

---

### Q4: Explain the singleton bean scope — is a Spring singleton the same as the Singleton design pattern?

**Answer:** No. The Singleton *pattern* guarantees one instance per **ClassLoader**. A Spring singleton is one instance per **ApplicationContext** (Spring container). If you create two `ApplicationContext` instances in the same JVM, you get two instances of the singleton bean. Also, Spring singletons are not thread-safe by default — the singleton scope means one instance, but concurrent access to mutable state within that instance is still your responsibility.

---

### Q5: What is the bean lifecycle order for @PostConstruct vs afterPropertiesSet() vs init-method?

**Answer:** `@PostConstruct` runs first, then `InitializingBean.afterPropertiesSet()`, then the custom `init-method` declared in `@Bean(initMethod="...")`. For destroy: `@PreDestroy` → `DisposableBean.destroy()` → custom `destroyMethod`. Note: for **prototype** beans, Spring does NOT call destroy hooks.

---

### Q6: What is the circular dependency issue with constructor injection, and how does Spring handle it?

**Answer:** With constructor injection, Spring cannot create Bean A without Bean B, and cannot create Bean B without Bean A — a true deadlock. Spring detects this and throws `BeanCurrentlyInCreationException` at startup (fail-fast). With setter/field injection, Spring uses its 3-level singleton cache to expose a partially-constructed bean and break the cycle. However, in Spring Boot 2.6+, even setter/field circular deps are disabled by default. The correct fix is always to redesign — extract the shared responsibility into a third class.

---

### Q7: When does Spring create AOP proxies, and what are the two proxy mechanisms?

**Answer:** AOP proxies are created in `BeanPostProcessor.postProcessAfterInitialization()`, specifically by `AbstractAutoProxyCreator`. Two mechanisms:
- **JDK Dynamic Proxy**: used when the bean implements at least one interface. Proxies the interface, not the class.
- **CGLIB Proxy**: used when the bean has no interface (or `proxyTargetClass=true`). Subclasses the target class at runtime.

Implication: `@Transactional`, `@Cacheable`, `@Async`, etc. only work on Spring-managed beans fetched from the context — calling a proxied method from within the same class bypasses the proxy (self-invocation problem).

---

### Q8: What is the difference between BeanFactoryPostProcessor and BeanPostProcessor?

**Answer:**
- `BeanFactoryPostProcessor`: operates on **bean definitions** (metadata) *before* any beans are instantiated. Use to change scope, add properties, or register new definitions. Example: `PropertySourcesPlaceholderConfigurer`.
- `BeanPostProcessor`: operates on **bean instances** *after* instantiation and dependency injection. Runs before and after init methods. Can replace the bean with a proxy. Example: `AbstractAutoProxyCreator` (AOP), `@Autowired` processing (`AutowiredAnnotationBeanPostProcessor`).

They run at completely different stages and serve fundamentally different purposes.

---

### Q9: How does @Transactional work under the hood? (Connects IoC and AOP)

**Answer:** `@Transactional` is implemented via a `BeanPostProcessor` that creates a CGLIB/JDK proxy around your service bean. When you call a `@Transactional` method:
1. The proxy intercepts the call.
2. `TransactionInterceptor` checks the transaction propagation rules.
3. If no transaction exists and propagation is `REQUIRED`, a new transaction is started (connection obtained from pool, set autoCommit=false).
4. The real method runs.
5. On success: commit. On `RuntimeException`: rollback (or as configured by `rollbackFor`).
6. Control returns to the caller.

Self-invocation breaks this because calling `this.someMethod()` bypasses the proxy — the AOP interceptor never runs.

---

### Q10: What is the difference between @Value and @ConfigurationProperties?

**Answer:**

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Binding granularity | Single property | Group of related properties |
| Type safety | Basic (`String`, `int`, etc.) | Full type-safe binding, nested objects |
| Validation | Manual | `@Validated` + JSR-380 annotations |
| IDE support | Limited | Full IntelliJ/metadata support |
| SpEL support | Yes | No |
| Relaxed binding | No | Yes (camelCase = kebab-case = UPPER_SNAKE) |
| Best for | Simple scalar values | Service configs, feature flags |

---

### Q11: If a singleton bean depends on a prototype bean, what happens?

**Answer:** The prototype dependency is injected **once** when the singleton is created — after that, the singleton always uses the same prototype instance, defeating the purpose of prototype scope. Solutions:
1. Inject `ApplicationContext` and call `ctx.getBean(PrototypeBean.class)` each time.
2. Use `@Lookup` method injection — Spring overrides the method via CGLIB to return a new prototype instance on each call.
3. Use `ObjectFactory<PrototypeBean>` or `Provider<PrototypeBean>` (JSR-330).

```java
@Component
public abstract class SingletonService {

    @Lookup  // Spring CGLIB-overrides this method to return a new prototype each time
    public abstract PrototypeBean getPrototypeBean();

    public void doWork() {
        PrototypeBean bean = getPrototypeBean(); // fresh instance every call
        bean.execute();
    }
}
```

---

### Q12: Explain @ConditionalOnMissingBean and how Spring Boot uses it for auto-configuration.

**Answer:** `@ConditionalOnMissingBean` registers a bean only if no bean of that type already exists in the context. Spring Boot's auto-configuration classes are loaded last (after user `@Configuration` classes). Each auto-configured bean is annotated with `@ConditionalOnMissingBean`. So if you define your own `DataSource` bean, Spring Boot's `DataSourceAutoConfiguration` backs off and doesn't register its default `DataSource`. This is the "convention over configuration with escape hatch" principle — you get sensible defaults, but you can always override by defining your own bean.

---

## Summary

| Concept | Key Takeaway |
|---|---|
| IoC | Object creation/wiring delegated to the container — decouples construction from use |
| BeanFactory vs ApplicationContext | Always use `ApplicationContext`; it adds AOP, events, i18n, auto-registration |
| Bean lifecycle | Know the exact order of hooks: constructor → Aware → BPP before → `@PostConstruct` → `afterPropertiesSet` → init-method → BPP after → destroy in reverse |
| Scopes | Singleton (default) = one per context; Prototype = new per `getBean()`; Web scopes need `proxyMode` for singleton injection |
| DI types | Prefer constructor injection — immutable, testable, fail-fast; use setter for optional deps |
| @Autowired resolution | Type → `@Primary` → `@Qualifier`/name; `@Resource` = name-first |
| Circular deps | Constructor injection fails fast (good!); setter/field uses 3-level cache; always fix by redesign |
| BeanFactoryPostProcessor | Modifies bean *definitions* before instantiation |
| BeanPostProcessor | Operates on bean *instances*; how AOP proxies are created |
| @Conditional | Enables "back off" pattern; core of Spring Boot auto-configuration |
| Profiles | Environment-specific beans; `@Profile("!prod")` for negation |
| @ConfigurationProperties | Preferred over `@Value` for groups of properties — type-safe, validated, IDE-friendly |
