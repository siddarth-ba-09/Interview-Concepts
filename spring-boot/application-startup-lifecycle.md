# Spring Boot Application Startup — Complete Deep Dive

---

## 1. Overview

When you call `SpringApplication.run(MyApp.class, args)`, a carefully orchestrated sequence of ~40+ steps executes before your first request is served. Understanding this sequence is critical for:
- Debugging startup failures
- Knowing when beans are available
- Writing correct `ApplicationContextInitializer`, `ApplicationListener`, `CommandLineRunner`
- Tuning startup performance

This document walks through every phase, from the JVM entry point to "Started MyApp in 2.3 seconds".

---

## 2. The Big Picture

```
main()
  └── SpringApplication.run(MyApp.class, args)
            │
            ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 1: SpringApplication Construction                        │
    │  - Detect application type (SERVLET / REACTIVE / NONE)          │
    │  - Load ApplicationContextInitializers from spring.factories    │
    │  - Load ApplicationListeners from spring.factories              │
    │  - Identify main application class                              │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 2: run() — Bootstrap                                     │
    │  - Start StopWatch (timing)                                     │
    │  - Load SpringApplicationRunListeners (spring.factories)        │
    │  - Fire ApplicationStartingEvent                                │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 3: Prepare Environment                                   │
    │  - Create ConfigurableEnvironment                               │
    │  - Attach PropertySources (system props, env vars, args)        │
    │  - Fire ApplicationEnvironmentPreparedEvent                     │
    │  - Bind spring.main.* properties to SpringApplication           │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 4: Create ApplicationContext                             │
    │  - Instantiate the right context type                           │
    │    (AnnotationConfigServletWebServerApplicationContext, etc.)   │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 5: Prepare ApplicationContext                            │
    │  - Set environment on context                                   │
    │  - Apply ApplicationContextInitializers                         │
    │  - Fire ApplicationContextInitializedEvent                      │
    │  - Load bean definitions from sources (@SpringBootApplication)  │
    │  - Fire ApplicationPreparedEvent                                │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 6: Refresh ApplicationContext  ← THE HEAVY PHASE        │
    │  (AbstractApplicationContext.refresh() — 12 steps)              │
    │  - Register BeanFactoryPostProcessors                           │
    │  - ConfigurationClassPostProcessor runs                         │
    │    └── @ComponentScan, @Import, @Bean, Auto-configuration       │
    │  - Register BeanPostProcessors                                  │
    │  - Initialize MessageSource, ApplicationEventMulticaster        │
    │  - onRefresh() → Start Embedded Web Server (Tomcat)             │
    │  - Register ApplicationListeners                                │
    │  - finishBeanFactoryInitialization() → Instantiate all singletons│
    │  - finishRefresh() → Fire ContextRefreshedEvent                 │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────────────┐
    │  PHASE 7: After Refresh                                         │
    │  - Fire ApplicationStartedEvent                                 │
    │  - Run ApplicationRunner / CommandLineRunner beans              │
    │  - Fire ApplicationReadyEvent                                   │
    └─────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
                   App is READY to serve requests
```

---

## 3. Phase 1 — SpringApplication Construction

```java
// Your entry point
@SpringBootApplication
public class OrderServiceApp {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApp.class, args);
    }
}
```

`SpringApplication.run(...)` is a static convenience method. It internally does:

```java
public static ConfigurableApplicationContext run(Class<?> source, String... args) {
    return new SpringApplication(source).run(args);
    //     ─────────────────────────────
    //     ① Constructor          ② run()
}
```

### What the Constructor Does

```
new SpringApplication(OrderServiceApp.class)
         │
         ├── 1. Store primarySources = [OrderServiceApp.class]
         │
         ├── 2. Detect WebApplicationType
         │        │
         │        ├── REACTIVE  → if reactor.netty is on classpath
         │        ├── SERVLET   → if DispatcherServlet + Servlet are on classpath
         │        └── NONE      → otherwise (batch jobs, CLI tools)
         │
         ├── 3. Load BootstrapRegistryInitializers
         │        └── from spring.factories: BootstrapRegistryInitializer
         │
         ├── 4. Load ApplicationContextInitializers
         │        └── from spring.factories: ApplicationContextInitializer
         │            Examples:
         │            - DelegatingApplicationContextInitializer
         │            - SharedMetadataReaderFactoryContextInitializer
         │            - ConfigurationWarningsApplicationContextInitializer
         │
         ├── 5. Load ApplicationListeners
         │        └── from spring.factories: ApplicationListener
         │            Examples:
         │            - EnvironmentPostProcessorApplicationListener
         │            - AnsiOutputApplicationListener
         │            - LoggingApplicationListener
         │            - BackgroundPreinitializer
         │
         └── 6. Detect main application class
                  └── walks the call stack, finds class with main() method
```

---

## 4. Phase 2 — Bootstrap (start of `run()`)

```
run(args)
  │
  ├── 1. Create DefaultBootstrapContext
  │        └── Notifies BootstrapRegistryInitializers
  │
  ├── 2. Load SpringApplicationRunListeners
  │        └── from spring.factories: SpringApplicationRunListener
  │            → EventPublishingRunListener (the only built-in one)
  │              bridges run lifecycle → ApplicationEvent publishing
  │
  ├── 3. listeners.starting(bootstrapContext, mainClass)
  │        └── Fires: ApplicationStartingEvent
  │            Caught by: LoggingApplicationListener
  │                       → Initializes logging system (Logback/Log4j2)
  │
  └── 4. ApplicationArguments wrapper created around args[]
```

---

## 5. Phase 3 — Environment Preparation

```
prepareEnvironment(listeners, bootstrapContext, applicationArguments)
         │
         ├── 1. Create ConfigurableEnvironment based on app type
         │        ├── SERVLET  → ApplicationServletEnvironment
         │        ├── REACTIVE → ApplicationReactiveWebEnvironment
         │        └── NONE     → ApplicationEnvironment
         │
         ├── 2. Configure PropertySources (in priority order, highest first)
         │
         │   ┌──────────────────────────────────────────────────────┐
         │   │           PropertySource Priority (High → Low)        │
         │   ├──────────────────────────────────────────────────────┤
         │   │  1. CLI arguments (--server.port=9090)                │
         │   │  2. SPRING_APPLICATION_JSON env var                   │
         │   │  3. Servlet init params / context params              │
         │   │  4. JNDI (java:comp/env)                              │
         │   │  5. Java System Properties (System.getProperties())   │
         │   │  6. OS Environment Variables (System.getenv())        │
         │   │  7. application-{profile}.properties                  │
         │   │  8. application.properties / application.yml          │
         │   │  9. @PropertySource annotations                       │
         │   │  10. Default properties (SpringApplication.setDefault)│
         │   └──────────────────────────────────────────────────────┘
         │
         ├── 3. Configure active profiles
         │
         ├── 4. listeners.environmentPrepared(bootstrapContext, env)
         │        └── Fires: ApplicationEnvironmentPreparedEvent
         │            Caught by: EnvironmentPostProcessorApplicationListener
         │                       → Invokes all EnvironmentPostProcessor impls
         │                         including ConfigDataEnvironmentPostProcessor
         │                         → This is where application.yml is loaded!
         │
         └── 5. Bind spring.main.* to SpringApplication itself
                  (e.g. spring.main.lazy-initialization=true takes effect here)
```

### How `application.yml` Actually Gets Loaded

```
ConfigDataEnvironmentPostProcessor
    └── ConfigDataEnvironment.processAndApply()
            └── ConfigDataLocationResolvers resolve locations:
                    - classpath:/
                    - classpath:/config/
                    - file:./
                    - file:./config/
                Each location tried with each active profile
                → application.yml, application-dev.yml, etc.
                → Loaded via YamlPropertySourceLoader / PropertiesPropertySourceLoader
                → Added to Environment's PropertySources
```

---

## 6. Phase 4 — Create ApplicationContext

```
createApplicationContext()
        │
        ├── SERVLET  → AnnotationConfigServletWebServerApplicationContext
        ├── REACTIVE → AnnotationConfigReactiveWebServerApplicationContext
        └── NONE     → AnnotationConfigApplicationContext

Hierarchy:
AnnotationConfigServletWebServerApplicationContext
    └── extends ServletWebServerApplicationContext
            └── extends GenericWebApplicationContext
                    └── extends GenericApplicationContext
                            └── extends AbstractApplicationContext
                                    └── implements ConfigurableApplicationContext
```

At this point the context is just an empty shell — no beans, no scanning yet.

---

## 7. Phase 5 — Prepare ApplicationContext

```
prepareContext(bootstrapContext, context, environment, listeners, 
               applicationArguments, printedBanner)
         │
         ├── 1. context.setEnvironment(environment)
         │
         ├── 2. postProcessApplicationContext(context)
         │        └── Register shared ConversionService
         │            Register ResourceLoader / ClassLoader
         │
         ├── 3. Apply ApplicationContextInitializers
         │        For each initializer (loaded in Phase 1):
         │        initializer.initialize(context)
         │        │
         │        Examples:
         │        ├── DelegatingApplicationContextInitializer
         │        │       → Delegates to context.initializer.classes property
         │        ├── SharedMetadataReaderFactoryContextInitializer
         │        │       → Shares CachingMetadataReaderFactory across processors
         │        └── ConfigurationWarningsApplicationContextInitializer
         │                → Registers a BeanFactoryPostProcessor to warn about
         │                  common config mistakes (e.g., scanning org.springframework)
         │
         ├── 4. listeners.contextPrepared(context)
         │        └── Fires: ApplicationContextInitializedEvent
         │            bootstrapContext closes here
         │
         ├── 5. Register special singleton beans:
         │        - springApplicationArguments (ApplicationArguments)
         │        - springBootBanner (Banner)
         │
         ├── 6. context.setLazyInitialization(lazyInitialization)
         │        → if spring.main.lazy-initialization=true, all beans lazy
         │
         ├── 7. Load sources into context
         │        context.load(OrderServiceApp.class)
         │        → Registers OrderServiceApp as a BeanDefinition
         │           (not instantiated yet — just the definition)
         │
         └── 8. listeners.contextLoaded(context)
                  └── Fires: ApplicationPreparedEvent
                      Listeners registered in spring.factories are now
                      added to the ApplicationContext's event multicaster
```

---

## 8. Phase 6 — Refresh ApplicationContext (The Heart of It All)

This is where 90% of the work happens. It calls `AbstractApplicationContext.refresh()`.

```
context.refresh()
    │
    ├── Step 1:  prepareRefresh()
    │               - Set startupDate, mark context as active
    │               - Initialize property sources (for web: ServletContext props)
    │               - Validate required properties
    │
    ├── Step 2:  obtainFreshBeanFactory()
    │               - Returns the underlying DefaultListableBeanFactory
    │               - In GenericApplicationContext, the factory already exists
    │
    ├── Step 3:  prepareBeanFactory(beanFactory)
    │               - Register ClassLoader
    │               - Register standard BeanPostProcessors:
    │                   • ApplicationContextAwareProcessor
    │                     (injects ApplicationContext, Environment, etc. via Aware)
    │                   • ApplicationListenerDetector
    │                     (detects beans that implement ApplicationListener)
    │               - Register resolvable dependencies:
    │                   • BeanFactory → the factory itself
    │                   • ApplicationContext → the context itself
    │                   • Environment, ResourceLoader, etc.
    │               - Register environment beans as singletons
    │
    ├── Step 4:  postProcessBeanFactory(beanFactory)
    │               - Web contexts: register web-specific scopes
    │                   (request, session, application)
    │               - Register ServletContext, ServletConfig as beans
    │
    ├── Step 5:  invokeBeanFactoryPostProcessors(beanFactory)  ← BIG STEP
    │               │
    │               │  Runs ALL BeanFactoryPostProcessors in order:
    │               │
    │               ├── ① ConfigurationClassPostProcessor  (most important)
    │               │       │
    │               │       ├── Reads @SpringBootApplication on OrderServiceApp
    │               │       ├── Processes @ComponentScan
    │               │       │       → Scans packages, finds @Component/@Service/etc.
    │               │       │       → Registers their BeanDefinitions
    │               │       ├── Processes @Import
    │               │       │       → @EnableAutoConfiguration imports
    │               │       │         AutoConfigurationImportSelector
    │               │       │       → Reads AutoConfiguration.imports
    │               │       │       → Filters by @Conditional* conditions
    │               │       │       → Registers surviving auto-config BeanDefinitions
    │               │       ├── Processes @Bean methods
    │               │       └── Processes @ImportResource
    │               │
    │               ├── ② PropertySourcesPlaceholderConfigurer
    │               │       → Resolves ${...} placeholders in @Value annotations
    │               │
    │               └── ③ Any custom BeanFactoryPostProcessors you defined
    │
    │        At the END of Step 5, all BeanDefinitions are registered.
    │        No beans instantiated yet.
    │
    ├── Step 6:  registerBeanPostProcessors(beanFactory)
    │               - Finds all BeanPostProcessor beans in the factory
    │               - Instantiates them (they must exist before other beans)
    │               - Registers in order (PriorityOrdered → Ordered → rest)
    │               Key BeanPostProcessors:
    │               ├── AutowiredAnnotationBeanPostProcessor
    │               │       → Handles @Autowired, @Value, @Inject
    │               ├── CommonAnnotationBeanPostProcessor
    │               │       → Handles @PostConstruct, @PreDestroy, @Resource
    │               ├── PersistenceAnnotationBeanPostProcessor
    │               │       → Handles @PersistenceContext, @PersistenceUnit
    │               └── AsyncAnnotationBeanPostProcessor
    │                       → Handles @Async
    │
    ├── Step 7:  initMessageSource()
    │               - Sets up MessageSource for i18n
    │               - If no bean defined, uses DelegatingMessageSource
    │
    ├── Step 8:  initApplicationEventMulticaster()
    │               - Creates SimpleApplicationEventMulticaster
    │               - This is the pub/sub hub for all ApplicationEvents
    │
    ├── Step 9:  onRefresh()   ← Embedded Server Starts Here
    │               │
    │               │  In ServletWebServerApplicationContext:
    │               └── createWebServer()
    │                       │
    │                       ├── Get ServletWebServerFactory bean
    │                       │     (TomcatServletWebServerFactory — auto-configured)
    │                       ├── factory.getWebServer(getSelfInitializer())
    │                       │     → Creates embedded Tomcat instance
    │                       │     → Tomcat is CREATED but NOT started yet
    │                       └── Store webServer reference
    │
    ├── Step 10: registerListeners()
    │               - Find ApplicationListener beans
    │               - Register with the ApplicationEventMulticaster
    │               - Publish any early events that were queued
    │
    ├── Step 11: finishBeanFactoryInitialization(beanFactory)  ← BIGGEST STEP
    │               │
    │               │  Instantiates ALL non-lazy singleton beans
    │               │
    │               ├── Freeze configuration (no more BeanDefinitions accepted)
    │               └── beanFactory.preInstantiateSingletons()
    │                       │
    │                       For each BeanDefinition (in dependency order):
    │                       │
    │                       └── getBean(beanName)
    │                               │
    │                               └── doGetBean()
    │                                       │
    │                                       ├── Check singleton cache (3 levels)
    │                                       │     singletonObjects (fully ready)
    │                                       │     earlySingletonObjects (exposed early)
    │                                       │     singletonFactories (for circular deps)
    │                                       │
    │                                       ├── Resolve dependencies (other beans)
    │                                       │
    │                                       ├── createBean()
    │                                       │     ├── instantiateBean()
    │                                       │     │     Constructor injection happens here
    │                                       │     ├── populateBean()
    │                                       │     │     @Autowired / setter injection here
    │                                       │     └── initializeBean()
    │                                       │           ├── Aware callbacks
    │                                       │           │   (setBeanName, setApplicationContext)
    │                                       │           ├── BeanPostProcessor.postProcessBeforeInit
    │                                       │           ├── @PostConstruct method
    │                                       │           ├── InitializingBean.afterPropertiesSet()
    │                                       │           ├── init-method
    │                                       │           └── BeanPostProcessor.postProcessAfterInit
    │                                       │               ← AOP proxies created here
    │                                       │
    │                                       └── Add to singletonObjects cache
    │
    └── Step 12: finishRefresh()
                    ├── clearResourceCaches()
                    ├── initLifecycleProcessor()
                    ├── getLifecycleProcessor().onRefresh()
                    │       └── Starts all Lifecycle beans
                    │           → Embedded Tomcat ACTUALLY STARTS here
                    │             (binds to port 8080)
                    ├── publishEvent(new ContextRefreshedEvent(this))
                    └── LiveBeansView MBean registration (if enabled)
```

---

## 9. Bean Instantiation Lifecycle (Zoom In on Step 11)

```
For each singleton bean during preInstantiateSingletons():

  ┌──────────────────────────────────────────────────────────────┐
  │                    BEAN LIFECYCLE                            │
  └──────────────────────────────────────────────────────────────┘

  1. Instantiation
     ┌─────────────────────────────────────────┐
     │  new OrderService(paymentRepo)           │
     │  Constructor injection resolved here     │
     └─────────────────────────────────────────┘
              ↓
  2. Populate Properties
     ┌─────────────────────────────────────────┐
     │  @Autowired field injection              │
     │  @Value injection                        │
     │  Setter injection                        │
     └─────────────────────────────────────────┘
              ↓
  3. Aware Interfaces
     ┌─────────────────────────────────────────┐
     │  BeanNameAware.setBeanName()             │
     │  BeanFactoryAware.setBeanFactory()       │
     │  ApplicationContextAware                 │
     │    .setApplicationContext()              │
     └─────────────────────────────────────────┘
              ↓
  4. BeanPostProcessor — Before Init
     ┌─────────────────────────────────────────┐
     │  postProcessBeforeInitialization()       │
     │  → @PostConstruct method runs here       │
     │    (via CommonAnnotationBeanPostProcessor│
     └─────────────────────────────────────────┘
              ↓
  5. Initialization
     ┌─────────────────────────────────────────┐
     │  InitializingBean.afterPropertiesSet()   │
     │  Custom init-method                      │
     └─────────────────────────────────────────┘
              ↓
  6. BeanPostProcessor — After Init
     ┌─────────────────────────────────────────┐
     │  postProcessAfterInitialization()        │
     │  → AOP proxy wrapping happens here       │
     │    (AnnotationAwareAspectJAutoProxyCreator│
     │    checks if bean needs proxy)           │
     │  → The proxy replaces the raw bean       │
     │    in the ApplicationContext             │
     └─────────────────────────────────────────┘
              ↓
  7. Bean is READY — added to singletonObjects cache
              ↓
  ... app runs ...
              ↓
  8. Context Close / Shutdown Hook
     ┌─────────────────────────────────────────┐
     │  @PreDestroy method                      │
     │  DisposableBean.destroy()                │
     │  Custom destroy-method                   │
     └─────────────────────────────────────────┘
```

---

## 10. Phase 7 — After Refresh

```
afterRefresh(context, applicationArguments)
    │ (currently empty — hook for subclasses)
    │
    ├── stopWatch.stop()
    │
    ├── listeners.started(context, timeTaken)
    │       └── Fires: ApplicationStartedEvent
    │
    ├── callRunners(context, applicationArguments)
    │       │
    │       ├── Collect all ApplicationRunner beans (ordered)
    │       │       → applicationRunner.run(ApplicationArguments)
    │       │
    │       └── Collect all CommandLineRunner beans (ordered)
    │               → commandLineRunner.run(String... args)
    │
    │   ↑ If any runner throws, ApplicationFailedEvent is fired
    │
    └── listeners.ready(context, timeTaken)
            └── Fires: ApplicationReadyEvent
                ← Application is now fully ready
```

---

## 11. Spring Boot Events Timeline

```
SpringApplication.run() lifecycle events (in order):

  ┌─────────────────────────────────────────────────────────────────────┐
  │  Event                          │ When                              │
  ├─────────────────────────────────┼───────────────────────────────────┤
  │  ApplicationStartingEvent       │ Before anything (logging init)    │
  │  ApplicationEnvironmentPrepared │ Environment ready, context not yet│
  │  ApplicationContextInitialized  │ Context created + initializers run│
  │  ApplicationPreparedEvent       │ Beans loaded, context not refresh │
  │  ContextRefreshedEvent          │ Context fully refreshed           │
  │  ApplicationStartedEvent        │ After refresh, before runners     │
  │  AvailabilityChangeEvent        │ CORRECT liveness state            │
  │  ApplicationReadyEvent          │ After runners, ready for traffic  │
  │  AvailabilityChangeEvent        │ ACCEPTING_TRAFFIC readiness state │
  └─────────────────────────────────┴───────────────────────────────────┘

  On failure:
  └── ApplicationFailedEvent        │ If startup throws any exception   │
```

> **Key gotcha:** `ApplicationEnvironmentPreparedEvent` fires before the `ApplicationContext` even exists. You cannot use `@Autowired` in listeners for this event — they must be registered in `spring.factories`.

---

## 12. Embedded Tomcat Startup (Zoom In)

```
onRefresh() [Step 9 of refresh]
    └── createWebServer()
            └── TomcatServletWebServerFactory.getWebServer(initializer)
                    │
                    ├── new Tomcat()
                    ├── Configure connector (port 8080, protocol HTTP/1.1)
                    ├── Configure Engine, Host, Context
                    ├── TomcatEmbeddedWebappClassLoader set up
                    ├── Self-initializer lambda stored (not called yet)
                    └── return new TomcatWebServer(tomcat, ...)
                            └── tomcat.start() called in TomcatWebServer constructor?
                                NO — start happens in finishRefresh()

finishRefresh() [Step 12 of refresh]
    └── lifecycleProcessor.onRefresh()
            └── starts Lifecycle beans including WebServer
                    └── webServer.start()
                            └── Tomcat.start()
                                    ├── Binds to port 8080
                                    ├── Calls the self-initializer
                                    │     → Registers DispatcherServlet
                                    └── Tomcat is now accepting connections
```

---

## 13. Auto-Configuration During Startup (Zoom In on Step 5)

```
ConfigurationClassPostProcessor.processConfigBeanDefinitions()
    │
    └── Parses OrderServiceApp (the @SpringBootApplication class)
            │
            └── Finds @EnableAutoConfiguration
                    │
                    └── AutoConfigurationImportSelector.selectImports()
                            │
                            ├── Load ~150 class names from:
                            │     META-INF/spring/
                            │     org.springframework.boot.autoconfigure
                            │     .AutoConfiguration.imports
                            │
                            ├── Apply exclusions
                            │     (spring.autoconfigure.exclude / @SpringBootApplication(exclude=))
                            │
                            ├── Evaluate @Conditional* on each candidate
                            │     ├── @ConditionalOnClass  → check classpath
                            │     ├── @ConditionalOnMissingBean → check existing beans
                            │     ├── @ConditionalOnProperty → check environment
                            │     └── ... etc
                            │
                            ├── Apply @AutoConfigureAfter / @AutoConfigureBefore ordering
                            │
                            └── Register surviving configs as BeanDefinitions
                                  e.g., DataSourceAutoConfiguration
                                        WebMvcAutoConfiguration
                                        HibernateJpaAutoConfiguration
                                        JpaRepositoriesAutoConfiguration
```

---

## 14. Complete Timeline with Wall-Clock Perspective

```
t=0ms    main() called
t=1ms    SpringApplication constructed, listeners/initializers loaded
t=2ms    ApplicationStartingEvent → Logging system initialized
t=5ms    Environment created, application.yml loaded
t=6ms    ApplicationEnvironmentPreparedEvent
t=8ms    ApplicationContext created (empty shell)
t=10ms   ApplicationContextInitializers applied
t=11ms   ApplicationContextInitializedEvent
t=12ms   OrderServiceApp BeanDefinition registered
t=13ms   ApplicationPreparedEvent
         ─── context.refresh() begins ───
t=15ms   BeanFactory prepared (standard BPPs registered)
t=20ms   ConfigurationClassPostProcessor runs
           → @ComponentScan: 47 beans found
           → Auto-configuration: 18 configs survive filtering
           → Total BeanDefinitions registered: ~200
t=25ms   BeanPostProcessors instantiated
t=26ms   MessageSource, EventMulticaster initialized
t=27ms   onRefresh(): Embedded Tomcat instance created
t=28ms   ApplicationListeners registered
t=30ms   ────── preInstantiateSingletons() begins ──────
           Bean instantiation + dependency injection + @PostConstruct
           for all ~200 singleton beans
           (DataSource pool initialized, Hibernate SessionFactory built,
            JPA repositories proxied, DispatcherServlet created, etc.)
t=1800ms preInstantiateSingletons() completes
t=1802ms finishRefresh():
           Tomcat STARTS on port 8080
           ContextRefreshedEvent fired
         ─── context.refresh() ends ───
t=1810ms ApplicationStartedEvent
t=1811ms CommandLineRunners / ApplicationRunners execute
t=1815ms ApplicationReadyEvent
t=1816ms "Started OrderServiceApp in 1.816 seconds"
```

---

## 15. Common Interview Questions on This Topic

### Q1: What is the difference between `ApplicationContextInitializer` and `ApplicationListener`?
**A:**
- `ApplicationContextInitializer` runs during **context preparation** (Phase 5), before `refresh()`. It's used to programmatically configure the `ConfigurableApplicationContext` — e.g., adding custom property sources, setting active profiles, registering additional bean definitions. Registered via `spring.factories` or `spring.application.context-initializer.classes`.
- `ApplicationListener` responds to **ApplicationEvents** published during the lifecycle. Some events (like `ApplicationEnvironmentPreparedEvent`) fire before the context exists, so those listeners must be registered in `spring.factories`, not as `@Component` beans.

---

### Q2: When exactly are `@PostConstruct` methods called?
**A:** During `Step 11` of `refresh()` (`finishBeanFactoryInitialization`), for each bean being instantiated. Specifically inside `initializeBean()` → `applyBeanPostProcessorsBeforeInitialization()`. `CommonAnnotationBeanPostProcessor` detects the `@PostConstruct` annotation and invokes the method. This happens **after** all dependencies are injected, so it's safe to use `@Autowired` fields. But it happens **before** AOP proxies are created (that's in `postProcessAfterInitialization()`), so you can't rely on transactional behavior inside `@PostConstruct`.

---

### Q3: At what point in startup can you NOT use `@Autowired`?
**A:** In `ApplicationListener` implementations that listen to events fired **before** `finishBeanFactoryInitialization()`. Events like `ApplicationEnvironmentPreparedEvent` and `ApplicationContextInitializedEvent` fire before beans are instantiated. Listeners registered via `spring.factories` are plain objects, not Spring beans — they have no injection. Use `SpringApplication.addListeners()` or `spring.factories` for these, and do constructor-based wiring manually.

---

### Q4: How does Spring Boot decide which `ApplicationContext` type to create?
**A:** `WebApplicationType.deduceFromClasspath()` checks the classpath:
- If `DispatcherHandler` is present AND `DispatcherServlet` is absent → `REACTIVE`
- If `Servlet` and `ConfigurableWebApplicationContext` are present → `SERVLET`
- Otherwise → `NONE` (standard, non-web)

The context class for each type:
- `SERVLET` → `AnnotationConfigServletWebServerApplicationContext`
- `REACTIVE` → `AnnotationConfigReactiveWebServerApplicationContext`
- `NONE` → `AnnotationConfigApplicationContext`

---

### Q5: When does the embedded Tomcat actually bind to port 8080?
**A:** In `Step 12` of `refresh()` — `finishRefresh()`. The `Tomcat` instance is **created** in `Step 9` (`onRefresh()`), but it only **starts** (binds to the port and begins accepting connections) when `lifecycleProcessor.onRefresh()` is called in `finishRefresh()`. This means all beans are fully initialized before any HTTP request can come in.

---

### Q6: What is the difference between `CommandLineRunner` and `ApplicationRunner`?
**A:** Both run after `ApplicationStartedEvent` and before `ApplicationReadyEvent`. The difference is the argument type:
- `CommandLineRunner.run(String... args)` — raw string arguments
- `ApplicationRunner.run(ApplicationArguments args)` — parsed arguments with access to option args (`--key=value`) and non-option args separately

Use `ApplicationRunner` when you need to parse arguments programmatically. Both support `@Order` for sequencing.

---

### Q7: What is a `BeanFactoryPostProcessor` vs a `BeanPostProcessor`?
**A:**
| | `BeanFactoryPostProcessor` | `BeanPostProcessor` |
|---|---|---|
| Runs | After all `BeanDefinition`s registered, before instantiation | Around each bean's instantiation |
| Operates on | `BeanDefinition` metadata | Bean instances |
| Use case | Modify property values, register new definitions | Add proxies, inject values, lifecycle callbacks |
| Example | `PropertySourcesPlaceholderConfigurer` | `AutowiredAnnotationBeanPostProcessor` |

`ConfigurationClassPostProcessor` is the most important `BeanFactoryPostProcessor` — it drives the entire `@ComponentScan` / `@Import` / auto-config processing.

---

### Q8: What happens if a bean throws an exception in `@PostConstruct`?
**A:** The exception propagates up through `initializeBean()` → `doCreateBean()` → `createBean()` → `getBean()` → `preInstantiateSingletons()` → `finishBeanFactoryInitialization()` → `refresh()`. Spring does not catch it — `refresh()` throws a `BeanCreationException`. `SpringApplication.run()` catches this, fires `ApplicationFailedEvent`, calls `context.close()` (triggering `@PreDestroy` on any already-initialized beans), and then rethrows. The JVM exits with a non-zero code.

---

## 16. Common Pitfalls & Gotchas

**1. Assuming `@PostConstruct` runs in a transactional proxy**  
AOP proxies are created in `postProcessAfterInitialization()`, which runs **after** `@PostConstruct`. The proxy wraps the raw bean, but `@PostConstruct` already ran on the raw object. If you call a `@Transactional` method from `@PostConstruct`, the transaction proxy is not active — you're calling the raw method. Use `ApplicationRunner` instead, which runs after full context setup.

**2. Listening to `ApplicationEnvironmentPreparedEvent` as a `@Component`**  
Won't work. This event fires before the context is even created. Must register the listener in `spring.factories`.

**3. Slow startup due to `preInstantiateSingletons()`**  
If startup is slow, the culprit is almost always bean initialization (Step 11) — typically: JPA `EntityManagerFactory` build (Hibernate scanning entities), connection pool initialization, or expensive `@PostConstruct` logic. Use `spring.main.lazy-initialization=true` in dev to skip eager initialization.

**4. Port binding failure**  
`Port 8080 already in use` exception comes from `finishRefresh()` (Step 12), not Step 9. By this point, all your beans are initialized. The error message says "APPLICATION FAILED TO START" and all `@PostConstruct` methods have already run.

**5. `CommandLineRunner` vs startup context**  
Runners execute after `ApplicationStartedEvent` but before `ApplicationReadyEvent`. If Actuator's `/health` endpoint uses `ReadinessState`, the app is not yet `ACCEPTING_TRAFFIC` during runner execution. Slow runners delay readiness.

**6. Order of multiple auto-configurations**  
`@ConditionalOnBean` in auto-config A depends on a bean from auto-config B. Without `@AutoConfigureAfter(B.class)`, condition evaluation order is non-deterministic and can fail intermittently across Spring Boot versions.

---

## 17. Summary

- `SpringApplication.run()` has **7 major phases**: construct → bootstrap → environment → create context → prepare context → refresh → after refresh.
- The **critical phase** is `AbstractApplicationContext.refresh()` with its 12 steps — this is where bean definitions are registered (`ConfigurationClassPostProcessor`), beans are instantiated (`preInstantiateSingletons`), and the embedded server starts (`finishRefresh`).
- Auto-configuration happens inside `ConfigurationClassPostProcessor` (Step 5), which evaluates `@Conditional*` annotations to decide which auto-config classes contribute beans.
- AOP proxies are created during `BeanPostProcessor.postProcessAfterInitialization()`, which means `@PostConstruct` always runs on the **raw** (unproxied) bean.
- The embedded Tomcat **binds to the port in `finishRefresh()`**, after all beans are initialized — ensuring no requests arrive before your app is ready.
- `ApplicationReadyEvent` is the signal that the application is fully started and ready to serve traffic.
