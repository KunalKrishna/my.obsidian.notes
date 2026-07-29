# Spring Boot Internals — Deep Dive Notes

> **How to use these notes:** Each section maps to a branch of the trunk question. Read top to bottom once, then use as reference. Every `> [!interview]` block is a ready answer. Every `> [!danger]` block is a trap interviewers set.

---
## Tags
`#spring-boot` `#internals` `#senior-level` `#interview-prep` `#java`

---
## 📑 Table of Contents

- [[#The Trunk Question]]
- [[#Part 1 — Bootstrapping & Startup Sequence]]
- [[#Part 2 — Auto-Configuration Internals]]
- [[#Part 3 — @ComponentScan Internals]]
- [[#Part 4 — @Configuration and @Bean Internals]]
- [[#Part 5 — Bean Lifecycle Internals]]
- [[#Part 6 — AOP and Proxy Internals]]
- [[#Part 7 — Environment and Properties]]
- [[#Mental Model — The Complete Picture]]

---
## The Trunk Question

> *"Walk me through what happens internally when a Spring Boot application starts."*

Every other question on this list is a **branch** off this trunk. The strategy is:

1. Answer the trunk with a structured, layered response
2. Signal you know the branches exist: *"...and if you want I can go deeper on auto-configuration, or the AOP proxy creation, or the property loading order..."*
3. Let the interviewer pull the branch they care about
4. Go 2–3 levels deep on that branch

> [!tip] Senior vs Mid-Level Framing
> A **mid-level** answer describes *what* happens.
> A **senior-level** answer describes *why* it was designed that way, *what breaks* if you misunderstand it, and *where you've seen it matter in production*.

---
## Part 1 — Bootstrapping & Startup Sequence

### 1.1 The Entry Point

```java
@SpringBootApplication
public class ShopWaveApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopWaveApplication.class, args);
    }
}
```

This three-line `main()` launches a sequence of ~57 distinct steps internally. The static `run()` is a convenience wrapper:

```java
// SpringApplication.java — what the static method actually does
public static ConfigurableApplicationContext run(
        Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}
```

Two things happen:
1. `new SpringApplication(primarySource)` — **construction phase** (setup)
2. `.run(args)` — **execution phase** (actual startup)

---
### 1.2 `new SpringApplication()` — The Construction Phase

Most developers never think about this, but it's where critical detection happens.

```java
// Simplified from SpringApplication.java source
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

    // STEP 1: Detect what kind of application this is
    this.webApplicationType = WebApplicationType.deduceFromClasspath();

    // STEP 2: Load bootstrap registrars
    this.bootstrapRegistrars = getSpringFactoriesInstances(
        BootstrapRegistryInitializer.class);

    // STEP 3: Load context initializers
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));

    // STEP 4: Load application listeners
    setListeners((Collection) getSpringFactoriesInstances(
        ApplicationListener.class));

    // STEP 5: Find the main class (for logging/display)
    this.mainApplicationClass = deduceMainApplicationClass();
}
```
#### 1.2.1 `WebApplicationType.deduceFromClasspath()`
This is how Spring Boot decides which `ApplicationContext` to create.

```java
// WebApplicationType.java — actual source logic
static WebApplicationType deduceFromClasspath() {
    // Check for REACTIVE: DispatcherHandler present, DispatcherServlet absent
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    // Check for NON-WEB: none of the web-related classes present
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    // Default: SERVLET
    return WebApplicationType.SERVLET;
}

// The constants being checked:
private static final String WEBMVC_INDICATOR_CLASS =
    "org.springframework.web.servlet.DispatcherServlet";
private static final String WEBFLUX_INDICATOR_CLASS =
    "org.springframework.web.reactive.DispatcherHandler";
```

> [!important] What this determines downstream
> | `WebApplicationType` | `ApplicationContext` Created | Embedded Server |
> |---|---|---|
> | `SERVLET` | `AnnotationConfigServletWebServerApplicationContext` | Tomcat / Jetty / Undertow |
> | `REACTIVE` | `AnnotationConfigReactiveWebServerApplicationContext` | Netty (default) |
> | `NONE` | `AnnotationConfigApplicationContext` | None |

> [!interview] Interview Answer
> *"Spring Boot uses classpath detection at `SpringApplication` construction time. It calls `WebApplicationType.deduceFromClasspath()` which checks for the presence of specific classes — `DispatcherServlet` for servlet, `DispatcherHandler` for reactive. This determines which `ApplicationContext` implementation to instantiate later. If both are present, servlet wins. You can override it explicitly with `app.setWebApplicationType(WebApplicationType.NONE)` which is useful in batch or CLI applications."*

```
                    ┌─────────────────────────────┐
                    │ START: deduceFromClasspath()│
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼──────────────┐
               ┌────┤  ① DispatcherHandler       ├────┐
               │ NO │     on classpath?          │YES │
               │    └────────────────────────────┘    │
               │                                      ▼
               │              ┌───────────────────────────┐
               │         ┌────┤  ② DispatcherServlet      ├──┐
               │         │YES │     on classpath?         │NO│
               │         │    └───────────────────────────┘  │
               │         │                                   ▼
               │         │        ┌───────────────────────────────┐
               │         │   ┌────┤  ③ Jersey ServletContainer    ├──┐
               │         │   │YES │     on classpath?             │NO│
               │         │   │    └───────────────────────────────┘  │
               │         │   │                                       │
               ▼         ▼   ▼                                       ▼
        ┌───────────────────────────┐                         ┌──────────────┐
        │ SERVLET_INDICATOR_CLASSES │                         │   REACTIVE   │
        │ loop (checks ④ and ⑤)     │                         └──────────────┘
        └──────────────┬────────────┘
                       │
        ┌──────────────▼──────────────┐
   ┌────┤  ④ jakarta.servlet.Servlet  ├────┐
   │ NO │     on classpath?           │YES │
   ▼    └─────────────────────────────┘    │
 NONE                                      │
                       ┌───────────────────▼──────────────────┐
                  ┌────┤  ⑤ ConfigurableWebApplicationContext ├────┐
                  │ NO │     on classpath?                    │YES │
                  ▼    └──────────────────────────────────────┘    ▼
                 NONE                                           SERVLET
```
#### 1.2.2 `SpringFactoriesLoader` — The Discovery Engine

`getSpringFactoriesInstances()` is called three times in the constructor. Understanding it is critical.
```java
// SpringFactoriesLoader.java — the core mechanism
public static <T> List<T> loadFactories(Class<T> factoryType, 
                                         ClassLoader classLoader) {
    // 1. Find all META-INF/spring.factories files on the classpath
    //    (one per jar — could be 50+ files from 50+ jars)
    Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
    // FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"

    // 2. Parse each file as a Properties file
    // 3. For the given factoryType, collect all implementation class names
    // 4. Instantiate them via reflection
    // 5. Return sorted list
}
```

The `spring.factories` file inside `spring-boot-autoconfigure.jar` looks like:

```properties
# META-INF/spring.factories (inside spring-boot-autoconfigure.jar)
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

org.springframework.boot.ApplicationContextFactory=\
org.springframework.boot.web.reactive.context.ReactiveWebServerApplicationContextFactory,\
org.springframework.boot.web.servlet.context.ServletWebServerApplicationContextFactory

# Note: Auto-configurations moved OUT of spring.factories in Boot 3.x
# They now live in:
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

> [!note] Spring Boot 2.x vs 3.x
> **Boot 2.x:** Auto-configs listed under `EnableAutoConfiguration` key in `spring.factories`
> **Boot 3.x:** Auto-configs moved to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
> The `spring.factories` file still exists in Boot 3.x but is no longer used for auto-configurations. This change was made for performance — the new format is simpler to parse.

---
### 1.3 `run(args)` — The Execution Phase

```java
// SpringApplication.run() — heavily simplified but structurally accurate
public ConfigurableApplicationContext run(String... args) {
    // A. Start timing
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // B. Create bootstrap context (for early config loading like Vault)
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();

    ConfigurableApplicationContext context = null;

    // C. Get the run listeners (bridges startup events to ApplicationEvents)
    SpringApplicationRunListeners listeners = getRunListeners(args);

    // D. FIRE: ApplicationStartingEvent
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // E. Prepare the environment (load application.yml, env vars, etc.)
        ConfigurableEnvironment environment = prepareEnvironment(listeners, 
                                                                  bootstrapContext, 
                                                                  applicationArguments);

        // F. Print the banner ("Spring" ASCII art)
        Banner printedBanner = printBanner(environment);

        // G. Create the ApplicationContext (type determined in constructor)
        context = createApplicationContext();

        // H. prepareContext() — wire everything together before refresh
        prepareContext(bootstrapContext, context, environment, 
                       listeners, applicationArguments, printedBanner);

        // I. THE BIG ONE — refresh() — most work happens here
        refreshContext(context);

        // J. Post-refresh hook (empty in base class)
        afterRefresh(context, applicationArguments);

        // K. Stop timing, log startup time
        stopWatch.stop();
        logStartupInfo(stopWatch);

        // L. FIRE: ApplicationStartedEvent
        listeners.started(context, stopWatch.getTotalTimeMillis());

        // M. Run ApplicationRunner and CommandLineRunner beans
        callRunners(context, applicationArguments);

    } catch (Throwable ex) {
        // N. If anything fails, fire ApplicationFailedEvent and rethrow
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    // O. FIRE: ApplicationReadyEvent
    listeners.ready(context, stopWatch.getTotalTimeMillis());

    return context;
}
```

---
### 1.4 `prepareEnvironment()` — Loading Your Config

```java
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        DefaultBootstrapContext bootstrapContext,
        ApplicationArguments applicationArguments) {

    // 1. Create the right Environment type based on WebApplicationType
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //   SERVLET  → StandardServletEnvironment
    //   REACTIVE → StandardReactiveWebEnvironment
    //   NONE     → StandardEnvironment

    // 2. Configure property sources (command line args, system properties)
    configureEnvironment(environment, applicationArguments.getSourceArgs());

    // 3. Attach ConfigurationPropertySources (enables relaxed binding)
    ConfigurationPropertySources.attach(environment);

    // 4. FIRE: EnvironmentPreparedEvent
    //    This triggers EnvironmentPostProcessorApplicationListener
    //    Which triggers ConfigDataEnvironmentPostProcessor
    //    Which READS application.yml / application.properties !
    listeners.environmentPrepared(bootstrapContext, environment);

    // 5. Move 'spring.main.*' properties to SpringApplication itself
    bindToSpringApplication(environment);

    return environment;
}
```

The property source loading happens inside `ConfigDataEnvironmentPostProcessor` which is triggered by the `EnvironmentPreparedEvent`. This is important — **your `application.yml` is loaded as a side effect of an event, not directly**.

Property source priority (highest wins):

```
Priority Chain (Highest → Lowest):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1.  Command-line args               --server.port=9090
2.  @TestPropertySource             (test only)
3.  @SpringBootTest properties      (test only)
4.  SPRING_APPLICATION_JSON         env var with JSON
5.  ServletConfig init params       (web only)
6.  ServletContext init params      (web only)
7.  JNDI (java:comp/env)
8.  System.getProperties()          JVM system properties
9.  System.getenv()                 OS environment variables
10. application-{profile}.yml       profile-specific config
11. application.yml                 default config
12. @PropertySource files           explicitly imported
13. SpringApplication.setDefaultProperties()
```

---
### 1.5 `prepareContext()` — Wiring Before Refresh

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext,
                             ConfigurableApplicationContext context,
                             ConfigurableEnvironment environment,
                             SpringApplicationRunListeners listeners,
                             ApplicationArguments applicationArguments,
                             Banner printedBanner) {

    // 1. Attach the prepared environment to the context
    context.setEnvironment(environment);

    // 2. Post-process the context (set ClassLoader, ConversionService)
    postProcessApplicationContext(context);

    // 3. Run ApplicationContextInitializers
    //    These were loaded from spring.factories in the constructor
    //    Example: SharedMetadataReaderFactoryContextInitializer
    //    Example: ConditionEvaluationReportLoggingListener
    applyInitializers(context);

    // 4. FIRE: ContextPreparedEvent
    listeners.contextPrepared(context);

    // 5. Close the BootstrapContext (no longer needed)
    bootstrapContext.close(context);

    // 6. Register special beans:
    //    - springApplicationArguments (the CLI args as a bean)
    //    - springBootBanner (the printed banner as a bean)
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }

    // 7. Load the primary source (your @SpringBootApplication class)
    //    as a bean definition — THIS IS THE SEED
    Set<Object> sources = getAllSources();
    load(context, sources.toArray(new Object[0]));

    // 8. FIRE: ContextLoadedEvent
    listeners.contextLoaded(context);
}
```

> [!important] The Chicken-and-Egg Answer
> Step 7 above is the answer to: *"How is the main class registered if `@ComponentScan` hasn't run yet?"*
>
> The main class (`ShopWaveApplication.class`) is **explicitly registered** as a bean definition by `prepareContext()` — specifically by `BeanDefinitionLoader.load()`. This happens BEFORE `refresh()`. It's the **seed** that makes everything else work. `@ComponentScan` is processed DURING `refresh()` starting from this pre-registered seed class.

---
### 1.6 `refreshContext()` — The Core

This delegates to `AbstractApplicationContext.refresh()` which is one of the most important methods in the entire Spring Framework.

```java
// AbstractApplicationContext.refresh() — the heart of Spring
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        // 1. prepareRefresh() — housekeeping, validate required properties
        prepareRefresh();

        // 2. obtainFreshBeanFactory() — get the DefaultListableBeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3. prepareBeanFactory() — configure the factory with built-ins
        prepareBeanFactory(beanFactory);

        try {
            // 4. postProcessBeanFactory() — subclass extension point
            //    Web context: registers web-specific scopes (request, session)
            postProcessBeanFactory(beanFactory);

            // 5. invokeBeanFactoryPostProcessors() — THE DEFINITION PHASE
            //    This is where @ComponentScan, @Configuration, auto-config runs
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. registerBeanPostProcessors() — register BPPs for bean lifecycle
            registerBeanPostProcessors(beanFactory);

            // 7. initMessageSource() — i18n
            initMessageSource();

            // 8. initApplicationEventMulticaster() — event system
            initApplicationEventMulticaster();

            // 9. onRefresh() — START EMBEDDED TOMCAT HERE
            onRefresh();

            // 10. registerListeners() — wire ApplicationListeners to multicaster
            registerListeners();

            // 11. finishBeanFactoryInitialization() — THE INSTANTIATION PHASE
            //     All singleton beans created here
            finishBeanFactoryInitialization(beanFactory);

            // 12. finishRefresh() — lifecycle callbacks, port opens
            finishRefresh();

        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
    }
}
```
#### 1.6.1 `invokeBeanFactoryPostProcessors()` — The Definition Phase

This is where all bean **definitions** (blueprints) are discovered and registered.
```
Key BeanFactoryPostProcessors invoked (in order):

1. ConfigurationClassPostProcessor (most important)
   ├── Processes @SpringBootApplication class (the seed)
   ├── @ComponentScan → scans packages → finds @Component, @Service, etc.
   ├── @Configuration classes → processes @Bean methods
   ├── @Import → processes imported classes
   └── @EnableAutoConfiguration → triggers AutoConfigurationImportSelector
       └── Reads AutoConfiguration.imports
       └── Evaluates @Conditional on each auto-config
       └── Registers matching auto-config bean definitions

2. PropertySourcesPlaceholderConfigurer
   └── Resolves ${...} placeholders in @Value and XML configs

3. EventListenerMethodProcessor (registered here, runs later)
   └── Finds @EventListener methods

After this step:
  - BeanFactory has ALL bean definitions (blueprints)
  - Zero beans have been instantiated (no memory used yet)
  - This is why startup can fail at this point with NoSuchBeanDefinitionException
```

#### 1.6.2 `registerBeanPostProcessors()` — Preparing the Lifecycle Hooks

```java
// These BeanPostProcessors are instantiated BEFORE all other beans
// because they need to intercept the creation of other beans

// 1. AutowiredAnnotationBeanPostProcessor
//    → Handles @Autowired, @Value, @Inject
//    → Reads field/constructor/setter injection points

// 2. CommonAnnotationBeanPostProcessor  
//    → Handles @PostConstruct, @PreDestroy, @Resource
//    → Calls @PostConstruct during postProcessBeforeInitialization()

// 3. PersistenceAnnotationBeanPostProcessor (if JPA present)
//    → Handles @PersistenceContext, @PersistenceUnit

// 4. AnnotationAwareAspectJAutoProxyCreator (if AOP present)
//    → The AOP proxy creator — wraps beans in CGLIB/JDK proxies
//    → This is what makes @Transactional, @Cacheable, @Async work

// Order matters — BPPs are sorted by PriorityOrdered > Ordered > rest
```

#### 1.6.3 `onRefresh()` — Tomcat Starts Here

```java
// ServletWebServerApplicationContext.onRefresh()
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    } catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();

    if (webServer == null && servletContext == null) {
        // Get the factory bean (TomcatServletWebServerFactory by default)
        ServletWebServerFactory factory = getWebServerFactory();

        // Create and start Tomcat — but NOT yet accepting connections
        this.webServer = factory.getWebServer(getSelfInitializer());
        
        // Register shutdown hook
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
    }
    // ...
}
```

> [!important] When Does Tomcat Start Accepting Requests?
> Tomcat is **created** in `onRefresh()` (step 9 of refresh).
> But Tomcat only **starts accepting requests** in `finishRefresh()` (step 12), specifically when `LifecycleProcessor` calls `start()` on `WebServerStartStopLifecycle`.
>
> This is intentional — you don't want traffic arriving while beans are still being created.

#### 1.6.4 `finishBeanFactoryInitialization()` — The Instantiation Phase

```java
// This is where EVERY singleton bean comes to life

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Set ConversionService if present
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) && ...) {
        beanFactory.setConversionService(...);
    }

    // Freeze the bean definition registry — no more definitions can be added
    beanFactory.freezeConfiguration();

    // *** INSTANTIATE ALL NON-LAZY SINGLETON BEANS ***
    beanFactory.preInstantiateSingletons();
}
```

The `preInstantiateSingletons()` loop creates beans in **dependency order** using a three-level cache to handle circular dependencies:

```java
// DefaultSingletonBeanRegistry — the three-level cache
// Level 1: singletonObjects — fully created, ready-to-use beans
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// Level 2: earlySingletonObjects — created but not yet fully initialized
//          (no BeanPostProcessors run yet)
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// Level 3: singletonFactories — ObjectFactory to create the early reference
//          Used to break circular dependency in setter/field injection
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

**Single bean creation walkthrough:**

```
For each bean definition in topological order:

1. Constructor called (dependencies resolved recursively first)
   ↓
2. Bean placed in singletonFactories (L3 cache) ← circular dep resolution happens here
   ↓
3. populate properties (AutowiredAnnotationBeanPostProcessor)
   @Autowired fields injected
   @Value fields resolved from Environment
   ↓
4. Aware interfaces called (if implemented):
   BeanNameAware.setBeanName()
   BeanFactoryAware.setBeanFactory()
   ApplicationContextAware.setApplicationContext()
   ↓
5. BeanPostProcessor.postProcessBeforeInitialization()
   ← CommonAnnotationBeanPostProcessor calls @PostConstruct here
   ↓
6. InitializingBean.afterPropertiesSet() (if implemented)
   ↓
7. Custom init-method (if configured: @Bean(initMethod="init"))
   ↓
8. BeanPostProcessor.postProcessAfterInitialization()
   ← AnnotationAwareAspectJAutoProxyCreator wraps in AOP proxy HERE
   ↓
9. Bean moved to singletonObjects (L1 cache) — READY
```

---

### 1.7 `ApplicationRunner` and `CommandLineRunner`

These run **after** `refreshContext()` completes but **before** `ApplicationReadyEvent`.

```java
// Example: warming up a cache at startup
@Component
@Order(1)  // controls execution order among multiple runners
public class CacheWarmupRunner implements ApplicationRunner {

    private final ProductService productService;
    private final CacheManager cacheManager;

    public CacheWarmupRunner(ProductService productService, CacheManager cacheManager) {
        this.productService = productService;
        this.cacheManager = cacheManager;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Typed access to arguments
        if (args.containsOption("skip-warmup")) {
            log.info("Skipping cache warmup");
            return;
        }
        log.info("Warming up product cache...");
        productService.findAll().forEach(p -> 
            cacheManager.getCache("products").put(p.getId(), p));
        log.info("Cache warmup complete");
    }
}

// CommandLineRunner is simpler — raw String args
@Component
@Order(2)
public class DatabaseSeedRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // args is the raw command-line string array
    }
}
```

> [!note] `ApplicationRunner` vs `CommandLineRunner`
> Prefer `ApplicationRunner` — it gives you `ApplicationArguments` which properly parses `--key=value` flags vs positional arguments.
>
> `CommandLineRunner` gives you the raw `String[]` exactly as passed to `main()`.

> [!interview] When do Runners run?
> *"Runners run after `refreshContext()` but before `ApplicationReadyEvent`. Specifically: `ApplicationStartedEvent` fires → runners execute → `ApplicationReadyEvent` fires. This means the full application context is available, all beans are created, but the application hasn't signaled readiness to external health checks yet. In Kubernetes, this means the pod won't receive traffic until runners complete — which is a useful property if you're using runners for cache warmup or DB validation."*

---

### 1.8 The Complete Event Timeline

```
SpringApplication.run() begins
         │
         ▼
ApplicationStartingEvent ──────────────────────────── Logging system initializes
         │
         ▼
ApplicationEnvironmentPreparedEvent ────────────────── application.yml loaded
                                                        Profiles activated
         │
         ▼
ApplicationContextInitializedEvent ─────────────────── Context created
                                                        Initializers run
         │
         ▼
ApplicationPreparedEvent ───────────────────────────── Main class registered as seed
                                                        ContextLoadedEvent
         │
         ▼
    [ refresh() begins ]
         │
         ▼
ContextRefreshedEvent ──────────────────────────────── All beans created
                                                        Tomcat starts accepting
                                                        Kafka/RabbitMQ listeners start
         │
         ▼
ApplicationStartedEvent ────────────────────────────── Context is up
AvailabilityChangeEvent → LivenessState.CORRECT         /health/liveness = UP
         │
         ▼
    [ ApplicationRunners execute ]
    [ CommandLineRunners execute ]
         │
         ▼
ApplicationReadyEvent ──────────────────────────────── Ready for traffic
AvailabilityChangeEvent → ReadinessState.ACCEPTING      /health/readiness = UP
         │
         ▼
    SpringApplication.run() returns
```

> [!tip] Practical use of events
> ```java
> @Component
> public class StartupListener {
>
>     // Fires when context is refreshed, beans are ready
>     // Good for: validation, warming up connections
>     @EventListener(ContextRefreshedEvent.class)
>     public void onRefresh(ContextRefreshedEvent event) {
>         // Fires on EVERY refresh (including test context reloads!)
>         // Watch out for double-execution in tests
>     }
>
>     // Fires when app is fully ready
>     // Good for: sending "application started" notification
>     @EventListener(ApplicationReadyEvent.class)
>     public void onReady(ApplicationReadyEvent event) {
>         // Fires ONCE — use this over ContextRefreshedEvent for startup tasks
>     }
>
>     // Fires if startup fails
>     @EventListener(ApplicationFailedEvent.class)
>     public void onFailure(ApplicationFailedEvent event) {
>         log.error("App failed to start", event.getException());
>         // Alert your monitoring system
>     }
> }
> ```

---

## Part 2 — Auto-Configuration Internals

### 2.1 The Chain That Makes It Work

```java
@SpringBootApplication
    └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector.class)
            └── AutoConfigurationImportSelector.selectImports()
                └── AutoConfigurationMetadataLoader.loadMetadata()
                    └── Reads: META-INF/spring-autoconfigure-metadata.properties
                            (quick filter — avoid loading class files)
                └── SpringFactoriesLoader / AutoConfiguration.imports
                    └── Gets list of ~150 auto-config class names
                └── For each: evaluate @Conditional annotations
                └── Register passing ones as @Configuration imports
```

### 2.2 `@EnableAutoConfiguration` Internals

```java
// EnableAutoConfiguration.java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage      // ← registers your base package for JPA entity scanning
@Import(AutoConfigurationImportSelector.class)  // ← the selector that does the work
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

```java
// AutoConfigurationImportSelector.java — the key method
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationEntry autoConfigurationEntry = 
        getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

protected AutoConfigurationEntry getAutoConfigurationEntry(
        AnnotationMetadata annotationMetadata) {

    // 1. Get ALL candidate auto-configurations (from AutoConfiguration.imports)
    List<String> configurations = getCandidateConfigurations(
        annotationMetadata, attributes);

    // 2. Remove duplicates (multiple jars might list the same config)
    configurations = removeDuplicates(configurations);

    // 3. Apply exclusions (@EnableAutoConfiguration(exclude=...))
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    configurations.removeAll(exclusions);

    // 4. Filter using spring-autoconfigure-metadata.properties
    //    (fast pre-filtering without loading the actual class)
    configurations = getConfigurationClassFilter().filter(configurations);

    // 5. Fire AutoConfigurationImportEvent (for test support)
    fireAutoConfigurationImportEvents(configurations, exclusions);

    return new AutoConfigurationEntry(configurations, exclusions);
}
```

### 2.3 How Spring Avoids Loading All 140+ Configs

> [!important] The Two-Stage Filter — This is Senior-Level Knowledge

**Stage 1: Metadata-based pre-filter (no class loading)**

Spring generates `spring-autoconfigure-metadata.properties` at **build time** (via annotation processor). This file contains conditions without requiring the classes to be loaded:

```properties
# spring-autoconfigure-metadata.properties (generated, inside spring-boot-autoconfigure.jar)
# Format: ConfigClassName.ConditionType=ConditionValue

org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration.ConditionalOnClass=\
  javax.persistence.EntityManager,\
  org.springframework.data.jpa.repository.JpaRepository

org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration.ConditionalOnClass=\
  org.apache.kafka.clients.producer.Producer

org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration.ConditionalOnClass=\
  org.springframework.data.redis.core.RedisOperations
```

Spring checks this metadata file FIRST. If the required class isn't on the classpath, the auto-config is eliminated **without ever loading its class file**. This is how 140+ configs are filtered down to maybe 20-30 relevant ones in milliseconds.

**Stage 2: Full `@Conditional` evaluation (class loaded, conditions evaluated)**

```java
// Only for configs that passed Stage 1:
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration { }

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class,
              DataSourceConfiguration.Tomcat.class, ... })
    protected static class PooledDataSourceConfiguration { }
}
```

### 2.4 How `@ConditionalOnClass` Works Without Instantiating the Class

```java
// OnClassCondition.java — the actual implementation
@Order(Ordered.HIGHEST_PRECEDENCE)
class OnClassCondition extends FilteringSpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, 
                                             AnnotatedTypeMetadata metadata) {

        ClassLoader classLoader = context.getClassLoader();

        // Gets class names from @ConditionalOnClass annotation
        // WITHOUT loading the classes themselves
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);

        ConditionMessage.Builder message = ConditionMessage.forCondition(ConditionalOnClass.class);

        if (onClasses != null) {
            // Uses ClassLoader.loadClass() but catches NoClassDefFoundError
            // So if the class doesn't exist, it returns false — doesn't throw
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                return ConditionOutcome.noMatch(
                    message.didNotFind("required class").items(Style.QUOTE, missing));
            }
        }
        return ConditionOutcome.match(message.found("required class").items(...));
    }
}

// ClassNameFilter.MISSING checks using this approach:
boolean isMissing(String className, ClassLoader classLoader) {
    try {
        // This checks existence without initializing the class
        Class.forName(className, false, classLoader);
        return false; // found
    } catch (ClassNotFoundException ex) {
        return true;  // not found
    }
}
```

> [!note] `Class.forName(name, false, classLoader)` — The Key Detail
> The second argument `false` means **do not initialize** the class. This means static initializers don't run, no side effects. It purely checks if the class bytecode exists on the classpath. This is why `@ConditionalOnClass` is fast and safe.

### 2.5 Bean Registration Order and Conflict Resolution

```
Bean Definition Registration Order:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Explicitly registered beans (via prepareContext's BeanDefinitionLoader)
   └── Your @SpringBootApplication class (the seed)

2. User-defined beans (via @ComponentScan during ConfigurationClassPostProcessor)
   └── Your @Service, @Repository, @Controller, @Configuration classes
   └── Your @Bean methods inside @Configuration classes

3. Auto-configured beans (via AutoConfigurationImportSelector)
   └── Only registered if @ConditionalOnMissingBean passes
   └── Auto-configs process AFTER user beans are registered
       (because @AutoConfigureBefore/After ordering puts them later)

WINNER ON CONFLICT: User-defined beans ALWAYS win.
```

```java
// Example: How @ConditionalOnMissingBean enforces "user wins"
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean   // ← "only create me if user hasn't defined one"
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .build();
    }
}

// User defines their own:
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        // Custom HikariCP config
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }
}
// Result: DataSourceAutoConfiguration.dataSource() NEVER RUNS
//         @ConditionalOnMissingBean detects the user's DataSource bean
//         and skips the auto-configured one entirely.
```

### 2.6 Writing Your Own Auto-Configuration

```java
// Step 1: Write the auto-configuration class
@AutoConfiguration  // Spring Boot 3 — was @Configuration in Boot 2
@ConditionalOnClass(EmailClient.class)        // only if library is present
@ConditionalOnProperty(                        // only if property set
    prefix = "app.email",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true)
@EnableConfigurationProperties(EmailProperties.class)  // bind properties
public class EmailAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // user can override
    public EmailClient emailClient(EmailProperties properties) {
        return new EmailClient(properties.getHost(), properties.getPort());
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean(EmailClient.class)  // only if EmailClient exists
    public EmailService emailService(EmailClient client) {
        return new EmailService(client);
    }
}

// Step 2: Define the properties class
@ConfigurationProperties(prefix = "app.email")
public class EmailProperties {
    private String host = "localhost";  // sensible defaults
    private int port = 587;
    private boolean enabled = true;
    // getters and setters
}

// Step 3: Register in the right file
// Spring Boot 3: src/main/resources/META-INF/spring/
//   org.springframework.boot.autoconfigure.AutoConfiguration.imports
// Contents:
com.mycompany.email.EmailAutoConfiguration

// Spring Boot 2: src/main/resources/META-INF/spring.factories
// org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
//   com.mycompany.email.EmailAutoConfiguration
```

> [!tip] Debugging Auto-Configuration
> ```bash
> # Run with --debug to see the conditions report
> java -jar app.jar --debug
>
> # Or in application.yml:
> debug: true
>
> # Output shows:
> # Positive matches (auto-configs that APPLIED):
> #   DataSourceAutoConfiguration matched:
> #     - @ConditionalOnClass found 'javax.sql.DataSource' (OnClassCondition)
> #
> # Negative matches (auto-configs that DID NOT apply):
> #   KafkaAutoConfiguration:
> #     Did not match:
> #       - @ConditionalOnClass did not find required class 
> #         'org.apache.kafka.clients.producer.Producer'
> ```

---

## Part 3 — @ComponentScan Internals

### 3.1 What `@ComponentScan` Actually Does

```java
// @SpringBootApplication includes @ComponentScan with no explicit packages:
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
})

// No 'basePackages' means: scan the package of the annotated class
// and ALL sub-packages recursively.
```

The scanning is performed by `ClassPathBeanDefinitionScanner`:

```java
// ClassPathScanningCandidateComponentProvider — the actual scanner

public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    // Converts package name to path: com.shopwave → com/shopwave
    String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
        resolveBasePackage(basePackage) + '/' + this.resourcePattern;
    // resourcePattern = "**/*.class"

    // 1. Get all .class files in the package
    Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);

    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    for (Resource resource : resources) {
        // 2. Read class metadata WITHOUT loading the class
        //    Uses ASM bytecode reader — no ClassLoader.loadClass() call!
        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);

        // 3. Check if it's a candidate (has @Component or meta-annotation with @Component)
        if (isCandidateComponent(metadataReader)) {
            ScannedGenericBeanDefinition sbd = 
                new ScannedGenericBeanDefinition(metadataReader);
            if (isCandidateComponent(sbd)) {
                candidates.add(sbd);
            }
        }
    }
    return candidates;
}
```

> [!important] ASM Bytecode Reader — No Class Loading During Scan
> Spring reads `.class` files using **ASM** (a bytecode manipulation library) to extract annotation metadata. This means `@Component` detection happens WITHOUT loading classes. Loading happens only when the bean is actually instantiated.
>
> This is a critical performance optimization. Your app might scan 10,000 classes but only instantiate 200 beans.

### 3.2 What Gets Scanned vs What Gets Skipped

```java
// isCandidateComponent() checks these include/exclude filters:

// DEFAULT INCLUDE FILTER: @Component (or annotations meta-annotated with @Component)
@Component        // ✅ scanned
@Service          // ✅ scanned (@Service is @Component)
@Repository       // ✅ scanned (@Repository is @Component)
@Controller       // ✅ scanned (@Controller is @Component)
@RestController   // ✅ scanned (@RestController is @Controller is @Component)
@Configuration    // ✅ scanned (@Configuration is @Component)
@Aspect           // ❌ NOT scanned (unless also @Component)

// EXPLICITLY EXCLUDED by @SpringBootApplication:
@AutoConfiguration  // ❌ excluded via AutoConfigurationExcludeFilter
                    // (prevents scanning auto-configs you've created locally)

// DEFAULT EXCLUSION RULES:
// - Interfaces are skipped
// - Abstract classes are skipped
// - Anonymous classes are skipped
// - Classes outside basePackage are skipped
```

### 3.3 The Chicken-and-Egg: How Is the Main Class Registered?

```
Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

BEFORE refresh():
  prepareContext() calls BeanDefinitionLoader.load(ShopWaveApplication.class)
  → ShopWaveApplication registered as bean definition MANUALLY
  → This is the SEED

DURING refresh() → invokeBeanFactoryPostProcessors():
  ConfigurationClassPostProcessor processes ShopWaveApplication
  → Sees @ComponentScan on it
  → ClassPathBeanDefinitionScanner scans com.shopwave.**
  → Finds UserService, ProductService, OrderRepository, etc.
  → Registers them as bean definitions

The main class is registered BEFORE @ComponentScan runs.
@ComponentScan runs BECAUSE the main class was registered.
Chicken: @ComponentScan needs main class
Egg: main class needs @ComponentScan to be processed

Resolution: the main class is injected manually as the seed.
            @ComponentScan processing happens as a consequence.
```

### 3.4 Moving a Class Outside the Base Package

```java
// Package structure:
com.shopwave.ShopWaveApplication.java    ← base package: com.shopwave
com.shopwave.service.UserService.java    ← ✅ found (sub-package)
com.shopwave.repo.UserRepository.java    ← ✅ found (sub-package)
com.external.HelperService.java          ← ❌ NOT found (different root)

// The problem:
@Service
public class HelperService {  // lives in com.external
    // Spring never finds this
}

// Fix 1: Move it to the base package
// Fix 2: Add basePackages to @ComponentScan
@SpringBootApplication
@ComponentScan(basePackages = {"com.shopwave", "com.external"})
public class ShopWaveApplication { }

// Fix 3: Use basePackageClasses (type-safe, refactor-safe)
@SpringBootApplication
@ComponentScan(basePackageClasses = {
    ShopWaveApplication.class,    // com.shopwave
    HelperServiceMarker.class     // com.external (marker interface/class)
})
public class ShopWaveApplication { }

// Fix 4: @Import (most explicit)
@SpringBootApplication
@Import(HelperService.class)
public class ShopWaveApplication { }
```

> [!danger] Common Interview Trap
> *"If you have a `@Configuration` class with `@Bean` methods outside the base package, will the beans be created?"*
>
> **Answer: NO.** The `@Configuration` class must be component-scanned (or explicitly imported) for its `@Bean` methods to be processed. `@Bean` methods are only processed when their `@Configuration` class is registered as a bean definition first.

### 3.5 Custom Include/Exclude Filters

```java
@ComponentScan(
    basePackages = "com.shopwave",
    includeFilters = {
        // Include all classes annotated with @UseCase
        @ComponentScan.Filter(type = FilterType.ANNOTATION, 
                               classes = UseCase.class),
        // Include classes that match a regex
        @ComponentScan.Filter(type = FilterType.REGEX, 
                               pattern = ".*Service"),
        // Include using custom filter logic
        @ComponentScan.Filter(type = FilterType.CUSTOM, 
                               classes = MyCustomFilter.class)
    },
    excludeFilters = {
        // Exclude test doubles from prod scan
        @ComponentScan.Filter(type = FilterType.ANNOTATION, 
                               classes = TestDouble.class)
    }
)
```

---

## Part 4 — `@Configuration` and `@Bean` Internals

### 4.1 Why `@Configuration` Gets a CGLIB Proxy

```java
// WITHOUT @Configuration (using @Component):
@Component
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(dataSource()); // calls dataSource()
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(dataSource()); // calls dataSource() AGAIN
    }

    @Bean
    public DataSource dataSource() {
        System.out.println("Creating DataSource...");
        return new HikariDataSource(); // CALLED TWICE → 2 DataSource instances!
    }
}

// WITH @Configuration:
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(dataSource()); // goes through proxy
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(dataSource()); // goes through proxy
    }

    @Bean
    public DataSource dataSource() {
        System.out.println("Creating DataSource...");
        return new HikariDataSource(); // called ONCE → 1 DataSource instance ✅
    }
}
```

**What the CGLIB proxy does:**

```java
// What CGLIB generates (conceptually):
public class AppConfig$$SpringCGLIB$$0 extends AppConfig {

    @Override
    public DataSource dataSource() {
        // Instead of calling super.dataSource() directly, it checks the BeanFactory:
        String beanName = "dataSource";
        if (beanFactory.containsSingleton(beanName)) {
            // Already exists → return the EXISTING instance from cache
            return (DataSource) beanFactory.getSingleton(beanName);
        }
        // Doesn't exist yet → call the real method → store → return
        return super.dataSource();
    }

    @Override
    public ServiceA serviceA() {
        // Same pattern for every @Bean method
        if (beanFactory.containsSingleton("serviceA")) {
            return (ServiceA) beanFactory.getSingleton("serviceA");
        }
        return super.serviceA();
    }
}
```

### 4.2 `proxyBeanMethods = false` — Lite Mode

```java
// Full @Configuration (proxyBeanMethods = true, the default):
// → CGLIB proxy created
// → @Bean methods calling other @Bean methods → singleton guarantee
// → Slower to instantiate, more memory
// → Use when: @Bean methods call other @Bean methods in same class

@Configuration  // proxyBeanMethods = true by default
public class FullConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(dataSource()); // SAFE - goes through proxy
    }
    @Bean
    public DataSource dataSource() { ... }
}

// Lite @Configuration (proxyBeanMethods = false):
// → No CGLIB proxy
// → @Bean methods are plain Java methods
// → Faster startup, less memory
// → Use when: @Bean methods DON'T call each other
// → Spring Boot's own auto-configs use this for performance

@Configuration(proxyBeanMethods = false)  // explicit lite mode
public class LiteConfig {
    @Bean
    public EmailService emailService(EmailProperties props) {
        // Takes EmailProperties as a parameter (injected by Spring)
        // instead of calling emailProperties() directly
        // → No inter-method calls → lite mode is safe here
        return new EmailService(props.getHost());
    }

    @Bean
    public EmailProperties emailProperties() { ... }
}
```

> [!note] How Spring Boot Uses This Internally
> Almost all Spring Boot auto-configuration classes use `@Configuration(proxyBeanMethods = false)`. This is a deliberate performance optimization — auto-configs are processed for every application startup, so the cost savings multiply. Your application code usually has fewer concerns about startup time, so `proxyBeanMethods = true` (the default) is appropriate unless you've profiled and identified a problem.

### 4.3 `@Configuration` vs `@Component` for Hosting `@Bean` Methods

```java
// Option A: @Component hosting @Bean (LITE mode implicitly)
@Component
public class UtilityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
    // ✅ Works fine IF these beans don't need to call each other
    // ❌ If passwordEncoder() called objectMapper() → TWO instances of objectMapper
}

// Option B: @Configuration (FULL mode — proxy created)
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authManager() {
        // Calls passwordEncoder() — safe due to proxy
        return new DaoAuthenticationProvider(passwordEncoder(), userDetailsService());
    }
    // ✅ Always correct — proxy ensures singleton semantics
}
```

> [!interview] Configuration vs Component Answer
> *"Both can host `@Bean` methods, but there's a critical difference. `@Configuration` classes are CGLIB-proxied, so `@Bean` methods that call other `@Bean` methods always return the same singleton instance. `@Component` hosted beans are in 'lite mode' — no proxy — so calling one `@Bean` method from another creates a new instance each time, breaking singleton semantics. I use `@Configuration` when my `@Bean` definitions have inter-dependencies in the same class. I use `@Component` or `@Configuration(proxyBeanMethods=false)` for independent bean definitions where I'm wiring via method parameters, not direct method calls."*

### 4.4 `@Bean` Method Parameter Injection

```java
@Configuration
public class OrderServiceConfig {

    // Spring resolves parameters from the context — type-based lookup
    @Bean
    public OrderService orderService(
            OrderRepository orderRepository,      // found by type
            ProductService productService,        // found by type
            @Qualifier("sagaKafkaTemplate")       // qualifier when multiple exist
            KafkaTemplate<String, Object> kafkaTemplate,
            Environment environment) {            // Environment is pre-registered

        boolean asyncEnabled = environment.getProperty(
            "order.async.enabled", Boolean.class, true);

        return new OrderService(orderRepository, productService, 
                                kafkaTemplate, asyncEnabled);
    }
}
```

---

## Part 5 — Bean Lifecycle Internals

### 5.1 `BeanFactoryPostProcessor` vs `BeanPostProcessor`

This is one of the most commonly confused distinctions.

```
Timeline positioning:

PHASE 1: DEFINITION (no beans instantiated)
    ├── BeanFactoryPostProcessor runs HERE
    ├── Receives: ConfigurableListableBeanFactory
    ├── Can: add/remove/modify BEAN DEFINITIONS
    ├── Can: read bean definition metadata
    └── Cannot: trigger bean instantiation (dangerous circular risk)

PHASE 2: INSTANTIATION (beans being created)
    ├── BeanPostProcessor runs HERE (before AND after each bean's init)
    ├── Receives: the BEAN INSTANCE (already created)
    ├── Can: wrap bean in proxy, add behavior, validate
    ├── Can: return a DIFFERENT object than what was created
    └── postProcessAfterInitialization() returning different object = proxy replacement
```

```java
// BeanFactoryPostProcessor example:
// Use case: programmatically set property values before beans are created
@Component
public class DatabaseUrlOverrider implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Runs BEFORE any bean is instantiated
        // Modify bean definitions here

        BeanDefinition dataSourceDef = beanFactory.getBeanDefinition("dataSource");
        MutablePropertyValues props = dataSourceDef.getPropertyValues();

        // Override URL based on environment
        String url = System.getenv("DYNAMIC_DB_URL");
        if (url != null) {
            props.add("url", url);
        }
    }
}

// BeanPostProcessor example:
// Use case: log all bean creation times (performance monitoring)
@Component
public class BeanCreationTimer implements BeanPostProcessor {

    private final Map<String, Long> startTimes = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Called AFTER injection, BEFORE @PostConstruct
        startTimes.put(beanName, System.currentTimeMillis());
        return bean; // return same bean (or a wrapper)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Called AFTER @PostConstruct, AFTER init-method
        Long start = startTimes.remove(beanName);
        if (start != null) {
            long duration = System.currentTimeMillis() - start;
            if (duration > 500) {
                log.warn("Slow bean initialization: {} took {}ms", beanName, duration);
            }
        }
        return bean; // can return a proxy instead of the original bean!
    }
}
```

### 5.2 The Complete Bean Lifecycle — With Code

```java
// Entity that demonstrates EVERY lifecycle hook:
@Service
public class OrderService implements
        BeanNameAware,             // knows its own bean name
        BeanFactoryAware,          // knows the BeanFactory
        ApplicationContextAware,   // knows the ApplicationContext
        InitializingBean,          // afterPropertiesSet() callback
        DisposableBean {           // destroy() callback

    private final OrderRepository repository;
    private String beanName;
    private ApplicationContext context;

    // STEP 1: Constructor called — dependencies injected via constructor
    public OrderService(OrderRepository repository) {
        System.out.println("1. Constructor called");
        this.repository = repository;
    }

    // (STEP 2: @Autowired field injection would happen here for field injection)

    // STEP 3: Aware interfaces
    @Override
    public void setBeanName(String name) {
        System.out.println("3a. BeanNameAware.setBeanName: " + name);
        this.beanName = name;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        System.out.println("3b. BeanFactoryAware.setBeanFactory called");
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("3c. ApplicationContextAware.setApplicationContext called");
        this.context = ctx;
    }

    // STEP 4: @PostConstruct (called by CommonAnnotationBeanPostProcessor)
    @PostConstruct
    public void init() {
        System.out.println("4. @PostConstruct — validate connections, warm cache");
        // Safe to use all injected dependencies here
        // Context is available via ApplicationContextAware
    }

    // STEP 5: InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() {
        System.out.println("5. InitializingBean.afterPropertiesSet");
        // Same timing as @PostConstruct but more coupled to Spring
        // Prefer @PostConstruct in application code
    }

    // (STEP 6: Custom init-method if @Bean(initMethod="customInit") configured)

    // --- BEAN IS NOW READY — serving requests ---

    // STEP 7: @PreDestroy (on shutdown)
    @PreDestroy
    public void cleanup() {
        System.out.println("7. @PreDestroy — flush buffers, close connections");
    }

    // STEP 8: DisposableBean.destroy()
    @Override
    public void destroy() {
        System.out.println("8. DisposableBean.destroy");
    }
}
```

> [!important] @PostConstruct vs afterPropertiesSet() vs init-method
> All three happen at the same lifecycle point (after injection). Execution order: `@PostConstruct` → `afterPropertiesSet()` → `init-method`.
>
> **Recommendation:** Use `@PostConstruct` for application beans (no Spring interface coupling). Use `afterPropertiesSet()` only in Spring infrastructure code. Avoid `init-method` unless you can't annotate the class (third-party code).

### 5.3 When Dependency Injection Happens

```java
@Service
public class UserService {

    // CONSTRUCTOR INJECTION: happens in Step 1 (constructor call)
    // Dependencies MUST exist and are REQUIRED
    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserRepository repository, EmailService emailService) {
        // At this point: repository and emailService are FULLY CREATED
        // (they were created first, recursively)
        this.repository = repository;
        this.emailService = emailService;
    }

    // FIELD INJECTION: happens AFTER constructor (Step 2)
    // Processed by AutowiredAnnotationBeanPostProcessor
    // AFTER constructor runs, but BEFORE @PostConstruct
    @Autowired  // ← set AFTER constructor completes
    private AuditService auditService;

    // @Value: also resolved during Step 2
    @Value("${user.default.role:USER}")
    private String defaultRole;

    @PostConstruct
    public void validate() {
        // At this point: ALL injections (constructor + field + @Value) are done
        // Safe to use repository, emailService, auditService, defaultRole
        Objects.requireNonNull(repository, "UserRepository must not be null");
        log.info("UserService initialized with default role: {}", defaultRole);
    }
}
```

### 5.4 Circular Dependencies — Detection and Resolution

```java
// CASE 1: Constructor injection circular dependency
// Spring FAILS AT STARTUP with BeanCurrentlyInCreationException
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { } // A needs B
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { } // B needs A
    // STARTUP FAILURE: Spring can't create A without B, can't create B without A
}

// CASE 2: Field injection circular dependency
// Spring CAN resolve this using the three-level cache
@Service
public class ServiceA {
    @Autowired ServiceB b; // A needs B (field, not constructor)
}

@Service
public class ServiceB {
    @Autowired ServiceA a; // B needs A (field, not constructor)
}

// How the three-level cache resolves this:
//
// 1. Start creating ServiceA
// 2. Constructor runs (no deps in constructor → empty object created)
// 3. ServiceA's early reference added to singletonFactories (L3 cache)
// 4. Try to inject field ServiceB → start creating ServiceB
// 5. Constructor runs for ServiceB
// 6. ServiceB's early reference added to L3 cache
// 7. Try to inject field ServiceA → 
//    → Check L1 (singletonObjects): not there
//    → Check L2 (earlySingletonObjects): not there
//    → Check L3 (singletonFactories): FOUND! Return early reference
// 8. ServiceB gets the EARLY (partially constructed) ServiceA
// 9. ServiceB completes initialization
// 10. Back in ServiceA: inject ServiceB (now fully constructed)
// 11. ServiceA completes initialization
```

> [!danger] The Circular Dependency Trap
> Field injection circular dependencies WORK but are a **code smell**. The fact that they work silently is actually a problem — it hides bad design. Constructor injection FAILING FAST on circular deps is a feature, not a bug.
>
> Spring Boot 2.6+ changed the default: circular dependencies are now **disallowed by default** and will cause startup failure even for field injection. You need `spring.main.allow-circular-references=true` to re-enable them. This was a deliberate breaking change to encourage better design.

---

## Part 6 — AOP and Proxy Internals

### 6.1 How `@Transactional` Works — The Full Story

```java
// What you write:
@Service
public class OrderService {

    @Transactional
    public Order createOrder(CreateOrderDto dto) {
        Order order = new Order(dto);
        orderRepository.save(order);
        inventoryService.reserve(order);  // if this throws, everything rolls back
        return order;
    }
}

// What Spring actually creates (conceptually):
// CGLIB generates a subclass at startup:
public class OrderService$$SpringCGLIB$$0 extends OrderService {

    private TransactionInterceptor transactionInterceptor;

    @Override  // overrides the @Transactional method
    public Order createOrder(CreateOrderDto dto) {
        // 1. Interceptor begins transaction management
        TransactionInfo txInfo = transactionInterceptor.createTransactionIfNecessary(
            txAttr, "OrderService.createOrder");

        Order retVal;
        try {
            // 2. Calls the REAL method on the actual OrderService object
            retVal = super.createOrder(dto);
        } catch (Throwable ex) {
            // 3. On exception: should we roll back?
            transactionInterceptor.completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // 4. Clean up transaction info from thread local
            transactionInterceptor.cleanupTransactionInfo(txInfo);
        }
        // 5. Commit
        transactionInterceptor.commitTransactionAfterReturning(txInfo);
        return retVal;
    }
}
```

### 6.2 The Transaction Infrastructure — `TransactionSynchronizationManager`

The transaction context is stored in **thread-local storage**:

```java
// TransactionSynchronizationManager — the thread-local store
public abstract class TransactionSynchronizationManager {

    // One Connection per thread per DataSource
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    // Current transaction name
    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

    // Is current transaction read-only?
    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
        new NamedThreadLocal<>("Current transaction read-only status");

    // Is there an active transaction?
    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
}

// This is why:
// 1. @Transactional is per-thread (each HTTP request has its own transaction)
// 2. @Async methods run in different threads = no transaction propagation
// 3. Passing an entity between threads = LazyInitializationException risk
```

### 6.3 Why `@Transactional` Doesn't Work on Private Methods

```java
@Service
public class OrderService {

    @Transactional  // ← THIS HAS NO EFFECT
    private Order createOrderInternal(CreateOrderDto dto) {
        // No transaction here!
        return orderRepository.save(new Order(dto));
    }
}
```

**Two reasons:**

```
Reason 1 — CGLIB cannot override private methods:
  CGLIB creates a SUBCLASS to intercept methods.
  Java rule: subclasses CANNOT override private methods.
  Therefore: CGLIB cannot intercept private @Transactional methods.
  Therefore: the transaction interceptor never runs.

Reason 2 — Even with JDK proxy, interface methods must be public:
  JDK proxy requires the method to be declared in the interface.
  Private methods can't be in interfaces.
  Therefore: same result — no transaction.

The Spring warning:
  Spring 5.3+ logs a warning when it detects @Transactional on a private method:
  "Transactional annotation on private method: OrderService.createOrderInternal"
```

### 6.4 Why Self-Invocation Bypasses the Transaction

```java
@Service
public class OrderService {

    @Transactional(propagation = REQUIRED)
    public void createOrder(CreateOrderDto dto) {
        // This call goes through THIS object, not through the proxy
        this.notifyAndSave(dto);
        // ↑ TRANSACTION LOST — notifyAndSave's @Transactional annotation ignored
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void notifyAndSave(CreateOrderDto dto) {
        // SHOULD be in a new transaction, but ISN'T because of self-invocation
    }
}

// What's in the Spring context:
//   "orderService" bean → OrderService$$SpringCGLIB$$0 (the proxy)

// What happens at runtime:
//   Controller calls orderService.createOrder()
//   → GOES TO PROXY ✅ → Transaction 1 started
//   → Proxy calls real.createOrder()
//   → Inside createOrder(): this.notifyAndSave()
//   → "this" = the real OrderService, NOT the proxy
//   → BYPASSES PROXY ❌ → No new transaction
```

**Three ways to fix self-invocation:**

```java
// FIX 1: Inject self (Spring Boot resolves circular injection for this case)
@Service
public class OrderService {

    @Autowired
    @Lazy  // prevent circular dependency issue
    private OrderService self;  // injects the PROXY

    @Transactional
    public void createOrder(CreateOrderDto dto) {
        self.notifyAndSave(dto);  // goes through proxy ✅
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void notifyAndSave(CreateOrderDto dto) { ... }
}

// FIX 2: Extract to a separate bean (cleanest — also better design)
@Service
public class OrderService {
    private final OrderNotificationService notificationService;

    @Transactional
    public void createOrder(CreateOrderDto dto) {
        notificationService.notifyAndSave(dto);  // different bean → goes through proxy ✅
    }
}

@Service
public class OrderNotificationService {
    @Transactional(propagation = REQUIRES_NEW)
    public void notifyAndSave(CreateOrderDto dto) { ... }
}

// FIX 3: Use AopContext.currentProxy() (pragmatic but AOP-aware code)
@Service
@EnableAspectJAutoProxy(exposeProxy = true)  // required config
public class OrderService {

    @Transactional
    public void createOrder(CreateOrderDto dto) {
        ((OrderService) AopContext.currentProxy()).notifyAndSave(dto);  // proxy ✅
        // Works but makes your code aware of the AOP mechanism — not ideal
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void notifyAndSave(CreateOrderDto dto) { ... }
}
```

### 6.5 JDK Dynamic Proxy vs CGLIB

```
JDK Dynamic Proxy:
━━━━━━━━━━━━━━━━━━
  - Created by: java.lang.reflect.Proxy
  - Requires: the target class must implement at least one interface
  - Implements: the same interfaces as the target
  - Callers must use the INTERFACE type, not the class type
  - Slightly faster creation (no bytecode generation)
  - Cannot proxy final methods or classes

  Example:
  interface UserService { User findById(Long id); }
  class UserServiceImpl implements UserService { ... }

  // Spring creates: Proxy.newProxyInstance(classLoader,
  //                     new Class[]{UserService.class},
  //                     new AopProxyInvocationHandler(target))
  // Type: UserService (the interface), not UserServiceImpl

CGLIB Proxy:
━━━━━━━━━━━━
  - Created by: CGLIB library (now embedded in Spring)
  - Does NOT require interfaces
  - Creates a SUBCLASS of the target class
  - Overrides all non-final public/protected methods
  - Cannot proxy final classes or final methods
  - Slightly slower creation (bytecode generation)
  - Default for Spring Boot since 5.x / Spring Boot 2.x

  Example:
  class ProductService { ... }  // no interface

  // CGLIB creates: ProductService$$SpringCGLIB$$0 extends ProductService
  //   and overrides all non-final public methods
```

**When Spring uses each:**

```java
// Spring Boot 2.x default: CGLIB for everything (@Transactional, @Cacheable, etc.)
// (changed from Spring 4 which defaulted to JDK proxy when interface available)

// Force JDK proxy:
@EnableTransactionManagement(proxyTargetClass = false)  // use JDK proxy if interface exists
@EnableAspectJAutoProxy(proxyTargetClass = false)

// CGLIB required when:
//   1. No interface available (CGLIB only option)
//   2. @Configuration classes (always CGLIB)
//   3. proxyTargetClass = true (explicit setting)

// JDK proxy required when:
//   1. Target is already a JDK proxy (can't CGLIB a proxy)
//   2. Class is final (CGLIB can't subclass it)

// Common mistake:
@Service
public final class OrderService {  // ← FINAL CLASS
    @Transactional
    public void createOrder(...) { ... }
}
// FAILS at startup: Cannot subclass final class OrderService
// Spring Boot 3 gives: BeanCreationException: Circular reference / final class
```

### 6.6 `AnnotationAwareAspectJAutoProxyCreator` — The Proxy Factory

```java
// This BeanPostProcessor is the one that creates all AOP proxies
// It runs in postProcessAfterInitialization() for every bean:

public class AnnotationAwareAspectJAutoProxyCreator 
        extends AbstractAutoProxyCreator {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean != null) {
            // Check: does this bean need to be proxied?
            Object cacheKey = getCacheKey(bean.getClass(), beanName);

            // Check for applicable advisors (advice that applies to this bean)
            List<Advisor> advisors = findEligibleAdvisors(bean.getClass(), beanName);

            if (!advisors.isEmpty()) {
                // Create the proxy!
                return createProxy(bean.getClass(), beanName, advisors, 
                                    new SingletonTargetSource(bean));
                // The PROXY is returned — Spring stores the proxy, not the original bean
            }
        }
        return bean; // no proxy needed — return as-is
    }
}

// findEligibleAdvisors checks for:
// - @Transactional on the class or its methods
// - @Cacheable, @CachePut, @CacheEvict on methods
// - @Async on methods
// - @PreAuthorize, @PostAuthorize on methods
// - Custom @Aspect advice (pointcut expressions evaluated)
```

---

## Part 7 — Environment and Properties

### 7.1 The `Environment` Abstraction

```java
// Environment is a unified facade over all property sources
public interface Environment extends PropertyResolver {
    // Property access:
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);

    // Profile access:
    String[] getActiveProfiles();
    String[] getDefaultProfiles();
    boolean acceptsProfiles(Profiles profiles);
}

// Behind Environment is a chain of PropertySources:
public class MutablePropertySources implements PropertySources {
    // Ordered list of PropertySource objects
    // First match wins
    private final List<PropertySource<?>> propertySourceList;
}

// PropertySource implementations:
//   SystemEnvironmentPropertySource    → OS env vars
//   PropertiesPropertySource           → .properties files
//   YamlPropertySourceLoader           → .yml files
//   CommandLinePropertySource          → --key=value args
//   MapPropertySource                  → in-memory map (tests)
```

### 7.2 How `application.yml` Gets Loaded

The loading chain:

```
EnvironmentPreparedEvent fires
    ↓
EnvironmentPostProcessorApplicationListener receives it
    ↓
Calls all EnvironmentPostProcessor implementations
    ↓
ConfigDataEnvironmentPostProcessor runs
    ↓
ConfigDataEnvironment.processAndApply()
    ↓
ConfigDataLocationResolvers resolve "classpath:/"
    ↓
StandardConfigDataLocationResolver looks for:
  application.properties
  application.xml
  application.yml
  application.yaml
  (in that order — last one doesn't win, FIRST match per key wins
   because PropertySources are ordered highest priority first)
    ↓
ConfigDataLoaders load each file
    ↓
YamlPropertySourceLoader parses .yml files using SnakeYAML
    ↓
PropertySources added to Environment:
  "Config resource 'class path resource [application.yml]'"
    ↓
Profile-specific files also loaded:
  application-{activeProfile}.yml
  (added with HIGHER priority than base application.yml)
```

```java
// How profile-specific config overrides base config:
// application.yml:
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost/shopwave_dev

// application-prod.yml:
spring:
  datasource:
    url: jdbc:postgresql://prod-db.internal/shopwave  # overrides base

// The PropertySources list (highest priority first):
// 1. CommandLineArgs
// 2. SystemEnvironment
// 3. "Config resource 'application-prod.yml'" ← profile-specific
// 4. "Config resource 'application.yml'"       ← base config
// 
// When you ask: environment.getProperty("spring.datasource.url")
// → Source 3 has it → prod URL returned ✅
// When you ask: environment.getProperty("server.port")
// → Source 3 doesn't have it → Source 4 has it → 8080 returned ✅
```

### 7.3 How `@Value` Gets Resolved

```java
@Service
public class PaymentService {

    @Value("${payment.gateway.url}")
    private String gatewayUrl;

    @Value("${payment.timeout.ms:5000}")  // default 5000 if not set
    private int timeoutMs;

    @Value("#{systemProperties['user.home']}")  // SpEL expression
    private String userHome;

    @Value("#{@paymentProperties.getApiKey()}")  // SpEL calling a bean method
    private String apiKey;
}
```

**Resolution chain:**

```
AutowiredAnnotationBeanPostProcessor detects @Value field
    ↓
Calls DefaultListableBeanFactory.resolveEmbeddedValue()
    ↓
StringValueResolver chain:
  1. PropertySourcesPlaceholderConfigurer's resolver
     → Looks up "${payment.gateway.url}" in Environment
     → Environment walks PropertySource chain (highest priority first)
     → Returns "https://payment-gw.example.com"
  2. SpEL resolver (for #{...} expressions)
     → StandardBeanExpressionResolver
     → Evaluates SpEL in the context of the BeanFactory
     → Can access beans via @beanName notation
    ↓
Result injected into the field via reflection
```

```java
// ConfigurationProperties vs @Value — the internal difference:
@ConfigurationProperties(prefix = "payment.gateway")
public class PaymentGatewayProperties {
    private String url;              // bound from payment.gateway.url
    private int timeoutMs = 5000;   // bound from payment.gateway.timeout-ms
    private Map<String, String> headers = new HashMap<>();  // complex types work
    private List<String> allowedIps; // collections work
    private Duration connectionTimeout;  // Duration type works with "30s" syntax
    // getters + setters required (or records in Boot 3)
}

// @Value: uses StringValueResolver + type conversion
// → Simple types only unless you write SpEL
// → Property key is hardcoded in code (refactoring risk)
// → No IDE autocompletion

// @ConfigurationProperties: uses Binder API
// → Full type safety
// → Collections, Maps, nested objects all work
// → IDE autocompletion via spring-configuration-metadata
// → @Validated works for Bean Validation on properties
// → Relaxed binding: payment.timeout-ms, PAYMENT_TIMEOUT_MS, paymentTimeoutMs all bind
```

### 7.4 Relaxed Binding — How `@ConfigurationProperties` Is More Flexible

```java
// All of these bind to 'timeoutMs' in @ConfigurationProperties:
payment.gateway.timeout-ms=5000      // kebab-case (preferred in .yml)
payment.gateway.timeoutMs=5000       // camelCase
payment.gateway.timeout_ms=5000      // underscore
PAYMENT_GATEWAY_TIMEOUT_MS=5000      // uppercase with underscore (env vars)

// @Value does NOT have relaxed binding:
@Value("${payment.gateway.timeout-ms}")  // ONLY matches exactly this key
```

### 7.5 `@ConfigurationProperties` Deep Dive

```java
// Full example with validation:
@ConfigurationProperties(prefix = "app.security")
@Validated  // enables Bean Validation
public class SecurityProperties {

    @NotBlank
    private String jwtSecret;

    @Min(60)
    @Max(86400)
    private int jwtExpirationSeconds = 3600;

    @NotNull
    private TokenRefreshProperties refresh = new TokenRefreshProperties();

    @Valid  // cascade validation to nested object
    private CorsProperties cors = new CorsProperties();

    // Nested class for grouped properties:
    public static class TokenRefreshProperties {
        private boolean enabled = true;
        @Min(3600)
        private int expirationSeconds = 604800;  // 7 days
        // getters + setters
    }

    public static class CorsProperties {
        @NotEmpty
        private List<String> allowedOrigins = List.of("http://localhost:3000");
        private List<String> allowedMethods = List.of("GET", "POST", "PUT", "DELETE");
        // getters + setters
    }
    // getters + setters for all fields
}

// Usage in application.yml:
// app:
//   security:
//     jwt-secret: "very-secret-key-256-bits-minimum"
//     jwt-expiration-seconds: 900
//     refresh:
//       enabled: true
//       expiration-seconds: 604800
//     cors:
//       allowed-origins:
//         - "https://shopwave.com"
//         - "https://admin.shopwave.com"
//       allowed-methods:
//         - GET
//         - POST

// Register it (choose one approach):
@SpringBootApplication
@ConfigurationPropertiesScan  // scans for @ConfigurationProperties in the same package tree
public class ShopWaveApplication { }

// Or explicitly:
@EnableConfigurationProperties(SecurityProperties.class)

// Or add @Component to the properties class itself (simplest but mixes concerns):
@Component
@ConfigurationProperties(prefix = "app.security")
public class SecurityProperties { ... }
```

---

## Part 8 — Putting It All Together

### 8.1 The Mental Model — Objects in Space

```
When Spring Boot finishes starting, the state of the JVM is:

DefaultListableBeanFactory (the engine):
  singletonObjects (ConcurrentHashMap, ~200-500 entries):
    "userService"           → UserService$$SpringCGLIB$$0     (CGLIB proxy)
    "orderService"          → OrderService$$SpringCGLIB$$0    (CGLIB proxy)
    "userRepository"        → $Proxy45 (JDK proxy)            (Spring Data proxy)
    "dataSource"            → HikariDataSource                (no proxy needed)
    "entityManagerFactory"  → LocalContainerEntityManagerFactoryBean
    "transactionManager"    → JpaTransactionManager
    "environment"           → StandardServletEnvironment
    ... ~200 more entries

Each CGLIB proxy holds a reference to:
  - The real object (the actual UserService instance)
  - An Advised object (list of interceptors: TransactionInterceptor, etc.)
  - A TargetSource (wraps the real object)

Each JDK proxy holds a reference to:
  - InvocationHandler (usually JdkDynamicAopProxy)
  - The real object via TargetSource

The proxy IS what you get when you @Autowire.
The real object lives INSIDE the proxy.
```

### 8.2 The Senior-Level Answer Framework

> [!tip] How to Organize a Senior-Level Answer
> Structure your answer in **concentric circles**:
>
> **Inner circle (always say):** The high-level phases — SpringApplication construction, environment preparation, context refresh (definition phase then instantiation phase), post-refresh runners.
>
> **Middle circle (say if relevant):** The mechanism that matters for the question — CGLIB proxy for @Transactional questions, auto-config conditional evaluation for "how does starter X work" questions, property source chain for configuration questions.
>
> **Outer circle (say to show depth):** The "why it matters in production" angle — how it explains a bug you've seen, why a particular design decision makes sense, what breaks if you misunderstand it.

### 8.3 Quick Reference — "Why Doesn't My X Work?"

| Symptom | Root Cause | Internal Explanation |
|---|---|---|
| `@Transactional` has no effect | Self-invocation or private method | CGLIB proxy bypassed — call goes to real object |
| `@Cacheable` has no effect | Same class call | Same proxy bypass issue |
| `@Value` is null in constructor | Injection timing | `@Value` resolved AFTER constructor in `AutowiredAnnotationBeanPostProcessor` |
| Bean not found / not created | Outside base package | `@ComponentScan` only scans base package + sub-packages |
| Auto-config not applying | Class not on classpath | `@ConditionalOnClass` check failed in pre-filter stage |
| Custom `DataSource` ignored | Wrong bean name or wrong type | `@ConditionalOnMissingBean` checks by type — your bean must be `DataSource` type |
| `@PostConstruct` runs twice | `ContextRefreshedEvent` triggered twice | Happens in parent+child contexts or test context reloads |
| `LazyInitializationException` | Outside-transaction lazy load | Hibernate session closed — proxy accessed outside session/transaction |
| Circular dependency error (Boot 2.6+) | Constructor injection circular ref OR boot default changed | `spring.main.allow-circular-references=true` or redesign |
| `final` class can't be proxied | CGLIB can't subclass final | Remove `final`, or use interface + JDK proxy |

---

## Part 9 — Code You Should Be Able to Write in an Interview

### 9.1 Custom `BeanPostProcessor`

```java
// Shows understanding of bean lifecycle and proxy creation
@Component
public class PerformanceMonitorBeanPostProcessor implements BeanPostProcessor {

    private static final Logger log = LoggerFactory.getLogger(
        PerformanceMonitorBeanPostProcessor.class);

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Only wrap @Service beans that have @Monitored annotation
        if (!bean.getClass().isAnnotationPresent(Monitored.class)) {
            return bean;  // return unchanged
        }

        // Create a JDK dynamic proxy that times every method call
        return Proxy.newProxyInstance(
            bean.getClass().getClassLoader(),
            bean.getClass().getInterfaces(),
            (proxy, method, args) -> {
                long start = System.nanoTime();
                try {
                    return method.invoke(bean, args);
                } finally {
                    long durationMs = (System.nanoTime() - start) / 1_000_000;
                    log.info("[PERF] {}.{}() took {}ms",
                        beanName, method.getName(), durationMs);
                }
            });
    }
}

// Usage:
@Monitored
@Service
public class ProductService implements ProductServicePort {
    // Every method call will be timed
}
```

### 9.2 Custom `BeanFactoryPostProcessor`

```java
// Shows understanding of bean definition phase
@Component
public class ConditionalPropertyOverride implements BeanFactoryPostProcessor {

    private final Environment environment;

    public ConditionalPropertyOverride(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Runs BEFORE any bean instantiation
        // Use case: dynamically configure beans based on environment

        String region = environment.getProperty("cloud.region", "us-east-1");

        // Modify the DataSource bean definition based on region
        if (beanFactory.containsBeanDefinition("dataSource")) {
            BeanDefinition bd = beanFactory.getBeanDefinition("dataSource");
            // For JDBC URL override based on region:
            String url = getRegionalDbUrl(region);
            bd.getPropertyValues().add("jdbcUrl", url);

            log.info("Configured DataSource for region: {} → {}", region, url);
        }
    }

    private String getRegionalDbUrl(String region) {
        return switch (region) {
            case "us-east-1" -> "jdbc:postgresql://us-east-1-db.internal/shopwave";
            case "eu-west-1" -> "jdbc:postgresql://eu-west-1-db.internal/shopwave";
            default          -> "jdbc:postgresql://localhost/shopwave";
        };
    }
}
```

### 9.3 Custom Auto-Configuration (Full Working Example)

```java
// Step 1: Properties class
@ConfigurationProperties(prefix = "app.rate-limiter")
public class RateLimiterProperties {
    private boolean enabled = true;
    private int requestsPerSecond = 100;
    private Duration windowSize = Duration.ofSeconds(1);
    // getters + setters
}

// Step 2: Auto-configuration class
@AutoConfiguration
@ConditionalOnClass(name = "com.bucket4j.Bucket")  // only if bucket4j on classpath
@ConditionalOnProperty(
    prefix = "app.rate-limiter",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = true)  // enabled by default
@EnableConfigurationProperties(RateLimiterProperties.class)
public class RateLimiterAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // user can provide their own
    public RateLimiter rateLimiter(RateLimiterProperties props) {
        Bandwidth limit = Bandwidth.classic(
            props.getRequestsPerSecond(),
            Refill.greedy(props.getRequestsPerSecond(), props.getWindowSize()));
        return Bucket.builder().addLimit(limit).build();
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnBean(RateLimiter.class)
    public RateLimiterFilter rateLimiterFilter(RateLimiter rateLimiter) {
        return new RateLimiterFilter(rateLimiter);
    }
}

// Step 3: Register it
// File: src/main/resources/META-INF/spring/
//   org.springframework.boot.autoconfigure.AutoConfiguration.imports
// Content:
// com.mycompany.ratelimit.RateLimiterAutoConfiguration
```

### 9.4 Demonstrating Proxy Understanding in Code

```java
// This test demonstrates the proxy concept — useful to write in whiteboard interviews
@SpringBootTest
class ProxyUnderstandingTest {

    @Autowired
    private OrderService orderService;

    @Test
    void autowiredBeanIsActuallyAProxy() {
        // The injected bean is a CGLIB proxy, not the real OrderService
        assertThat(orderService.getClass().getName())
            .contains("SpringCGLIB"); // or "$$EnhancerBySpringCGLIB"

        // But it IS an instance of OrderService (CGLIB extends it)
        assertThat(orderService).isInstanceOf(OrderService.class);
    }

    @Test
    void selfInvocationBreaksTransactional() {
        // If OrderService has a self-invoking @Transactional method,
        // we can verify the transaction doesn't propagate
        // by checking TransactionSynchronizationManager:

        orderService.methodThatSelfInvokes();
        // The inner @Transactional(REQUIRES_NEW) call will NOT create a new transaction
        // because it goes through 'this', not the proxy
    }

    @Test
    void getTheRealObjectFromProxy() {
        // Sometimes needed for testing:
        if (AopUtils.isAopProxy(orderService)) {
            Object realObject = AopTestUtils.getTargetObject(orderService);
            assertThat(realObject.getClass()).isEqualTo(OrderService.class);
            assertThat(realObject.getClass().getName()).doesNotContain("CGLIB");
        }
    }
}
```

---

## Quick Reference Card

```
STARTUP SEQUENCE (memorize this order):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1.  SpringApplication constructed
    └── WebApplicationType detected
    └── Initializers + Listeners loaded from spring.factories

2.  ApplicationStartingEvent 🔔

3.  Environment prepared
    └── application.yml loaded (via ConfigDataEnvironmentPostProcessor)
    └── Profiles activated

4.  EnvironmentPreparedEvent 🔔

5.  ApplicationContext created (type = detected WebApplicationType)

6.  prepareContext():
    └── Initializers run
    └── Main class registered as SEED bean definition

7.  ContextPreparedEvent 🔔, ContextLoadedEvent 🔔

8.  refresh() begins:
    a) BeanFactoryPostProcessors run (DEFINITION PHASE)
       └── @ComponentScan executes
       └── @Configuration + @Bean processed
       └── Auto-configuration evaluated + registered
    b) BeanPostProcessors registered (but not yet running on beans)
    c) Embedded Tomcat created (NOT yet accepting connections)
    d) Singleton beans INSTANTIATED (INSTANTIATION PHASE)
       └── Constructor → injection → Aware → @PostConstruct → AOP proxy
    e) ContextRefreshedEvent 🔔
    f) Tomcat starts ACCEPTING CONNECTIONS

9.  ApplicationStartedEvent 🔔
    LivenessState.CORRECT set

10. ApplicationRunners run
11. CommandLineRunners run

12. ApplicationReadyEvent 🔔
    ReadinessState.ACCEPTING_TRAFFIC set

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
KEY FACTS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• @ComponentScan uses ASM (reads .class bytecode, no ClassLoader.load())
• Auto-configs filtered by metadata BEFORE class loading (performance)
• @Configuration gets CGLIB proxy → @Bean call idempotency
• @Bean method calls go through BeanFactory if proxied
• BeanPostProcessor runs after construction, during postProcessAfterInit
  → This is where CGLIB/JDK proxies are CREATED
• @Transactional context stored in ThreadLocal
• Self-invocation bypasses proxy because 'this' = real object
• Private methods: CGLIB can't override → @Transactional has no effect
• Three-level cache resolves circular dependencies (field injection only)
• Constructor injection circular deps → fail at startup (good!)
• Spring Boot 2.6+: all circular deps disallowed by default
```

---

> [!note] How to Use These Notes for Interview Prep
> 1. **Day 1-2:** Read Parts 1 and 2 thoroughly. Draw the startup sequence from memory.
> 2. **Day 3:** Read Parts 3 and 4. Write the CGLIB proxy example from scratch.
> 3. **Day 4:** Read Parts 5 and 6. Write all three fixes for self-invocation without looking.
> 4. **Day 5:** Read Part 7. Explain the property priority order to yourself out loud.
> 5. **Day 6-7:** Write code from Part 9 from scratch. Run it. Break it intentionally. Fix it.
> 6. **Ongoing:** For every interview question, locate which section it maps to in the startup sequence, then answer with: *what happens* → *why it's designed that way* → *what breaks if you misunderstand it*.

# ANALOGY

# What Happens When Spring Boot Starts — The Analogy Edition

> **No code. Just understanding.**
> Read this once and you'll never forget the startup sequence again.

---
## The One Analogy to Rule Them All

**Spring Boot starting up is exactly like a restaurant opening for the day.**

Not a restaurant that's been open for years. A **brand new restaurant**, opening its doors for the very first time, on day one. Every single morning it opens, it goes through the same ritual — hiring staff, setting up the kitchen, stocking the pantry — from scratch.

That restaurant is your Spring Boot application.

Let's walk through the entire morning.

---
## Before the Story — One Thing You Must Accept

> **Spring Boot is not pre-built. It builds itself fresh every time you run it.**

When you click Run in IntelliJ, Spring Boot doesn't load a saved state. It constructs the entire application — every connection, every service, every configuration — from zero. Every. Single. Time.

This is why startup takes a few seconds. It's not loading. It's **building**.

---
## Chapter 1 — The Owner Shows Up

```
You type:  java -jar shopwave.jar
                     │
                     ▼
           main() method is called
```

Think of this as the **restaurant owner arriving at the building** in the morning, key in hand. Nothing is set up yet. The building is empty.

The owner's first job is not to start cooking. It's to answer one fundamental question:

> **"What kind of establishment are we running today?"**

---
## Chapter 2 — What Kind of Restaurant Is This?

The owner looks around the building and checks what equipment is installed.

```
Owner looks around and asks:

"Do we have a wood-fired pizza oven?"     → YES → We're a pizzeria (REACTIVE)
"Do we have a standard commercial range?" → YES → We're a normal restaurant (SERVLET)
"Do we have neither?"                     → We're a catering kitchen, no dine-in (NONE)
```

In Spring Boot terms, it looks at the classpath — the collection of all libraries you added — and asks *"what kind of app is this supposed to be?"*

- **Web app with MVC?** → Normal restaurant that serves customers at tables
- **Reactive app?** → A fast-food counter — non-stop, never waiting, always moving
- **No web at all?** → A catering kitchen — does all its work behind the scenes, no customers walk in

**This decision happens before anything else.** Because the entire setup depends on it. You can't set up a sushi bar if you're running a steakhouse.

---
## Chapter 3 — Reading the Recipe Book (The Environment)

The owner goes to the back office and picks up **The Recipe Book** — your `application.yml` or `application.properties`.

```
The Recipe Book contains things like:

  "Our database is at this address..."
  "We have 10 tables (connection pool size)..."
  "Today we're in PRODUCTION mode, not training mode..."
  "The kitchen opens at port 8080..."
```

But here's something important about how this recipe book works:
### The Recipe Book Has Layers

Imagine the recipe book is actually a stack of sticky notes on top of a printed page:

```
┌─────────────────────────────────────┐
│  🟡 STICKY NOTE (Command line args) │  ← "Today only: use port 9090"
│     --server.port=9090              │
├─────────────────────────────────────┤
│  🟠 STICKY NOTE (Environment vars)  │  ← "DB password from the safe"
│     DB_PASSWORD=secret              │
├─────────────────────────────────────┤
│  🟢 STICKY NOTE (Profile config)    │  ← "We're in PROD today"
│     application-prod.yml            │
├─────────────────────────────────────┤
│  📄 PRINTED PAGE (Base config)      │  ← The permanent recipe book
│     application.yml                 │
└─────────────────────────────────────┘

Rule: The TOP sticky note always wins.
      It covers whatever the page below says.
```

If the sticky note says *"use port 9090"* but the printed page says *"use port 8080"* — the sticky note wins. The owner reads from the top down and the first answer they find is the one they use.

This is why environment variables override your yml file. They're a higher sticky note.

---
## Chapter 4 — The Franchise Playbook Arrives

Here's where Spring Boot does something remarkable.

Imagine your restaurant is part of a **franchise**. When you join the franchise, a thick binder arrives at your door. It's called **The Franchise Playbook**.

```
THE FRANCHISE PLAYBOOK
(spring-boot-autoconfigure.jar)

Table of Contents:
  Chapter 1:  How to set up a database connection
  Chapter 2:  How to set up the web server
  Chapter 3:  How to set up security
  Chapter 4:  How to set up Kafka messaging
  Chapter 5:  How to set up Redis cache
  Chapter 6:  How to set up email sending
  Chapter 7:  How to set up Elasticsearch
  ...
  Chapter 150: ...
```

This playbook was written **once** by the Spring Boot team. It covers every possible scenario. It knows how to set up 150 different things.

But here's the key: **you don't read all 150 chapters every morning.** You only read the chapters that are relevant to YOUR restaurant.
### The Smart Playbook — It Knows What to Skip

Each chapter in the playbook starts with a checklist:

```
Chapter 1: Database Setup
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE YOU READ THIS CHAPTER, CHECK:

☐ Do you have a database driver installed?
    → NO? Skip this entire chapter.
    → YES? Continue...

☐ Did the owner already set up their own database connection?
    → YES? Skip this chapter — they've handled it.
    → NO?  Continue and set it up for them.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[rest of chapter only read if both checks pass]
```

```
Chapter 4: Kafka Messaging Setup
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BEFORE YOU READ THIS CHAPTER, CHECK:

☐ Do you have Kafka libraries installed?
    → NO? Skip this entire chapter.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[skipped — you never added Kafka to your project]
```

Out of 150 chapters, maybe 20 pass their checklists for your specific app. The other 130 are silently skipped before anyone reads them.

**This is the entire secret of Auto-Configuration.**

> It's not magic. It's a 150-chapter playbook where each chapter has a checklist that says "only read me if you need me."

---
## Chapter 5 — The Two-Phase Morning Prep

Now the real work begins. The morning prep happens in **two completely separate phases** that must not be confused.

Think of it this way:

```
PHASE 1 — PLANNING          PHASE 2 — DOING
(no one is hired yet)        (everyone gets hired)

"We need a head chef"    →   Head chef is hired
"We need a cashier"      →   Cashier is hired
"We need a dishwasher"   →   Dishwasher is hired
"We need 2 waiters"      →   2 waiters are hired

All decisions made first.    Then all hiring happens.
No actual people yet.        Now actual people exist.
```

Spring Boot does the same thing:

```
PHASE 1 — BEAN DEFINITION      PHASE 2 — BEAN INSTANTIATION
(no objects created)            (objects actually created)

"We'll need a DataSource"   →   DataSource object created
"We'll need a UserService"  →   UserService object created
"We'll need a Tomcat"       →   Tomcat object created

All decisions made first.       Then all creation happens.
Zero memory used for beans.     Now beans live in memory.
```
### Why Two Phases?

Imagine if a restaurant tried to hire people while simultaneously deciding what roles exist. The head chef shows up but there's no kitchen yet. The cashier arrives but there's no till. Chaos.

Spring separates these phases because:

1. During planning, it needs to know everything before it builds anything
2. Some things depend on other things — the waiter can't be trained until the head chef exists
3. If your configuration has mistakes, you want to find out during planning, not halfway through hiring

---
## Chapter 5a — Phase 1: The Planning Board

The owner stands in front of a giant whiteboard. This is a planning session. Nobody is hired. No work starts. Just planning.

```
THE PLANNING WHITEBOARD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

YOUR HANDWRITING (ComponentScan):
  The owner walks through every room of the building
  and writes down every role they find a note for:
  
  "Found a note saying: need a ProductService"
  "Found a note saying: need an OrderRepository"
  "Found a note saying: need a UserController"
  → All written on the board. Still just words.

FRANCHISE PLAYBOOK ADDITIONS (Auto-Configuration):
  The franchise manager reads the relevant chapters
  and adds the standard roles:
  
  "Standard: need a DataSource (database connection)"
  "Standard: need an EntityManagerFactory (JPA)"
  "Standard: need a TransactionManager"
  "Standard: need an embedded Tomcat"
  → Added to the board. Still just words.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AT THE END OF PHASE 1:

The whiteboard is full of role names.
Zero actual people have been hired.
Zero actual objects exist in memory.
It's a complete plan with zero execution.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### The "Franchise Doesn't Override the Owner" Rule

There's one critical rule written at the top of the whiteboard:

> **"If the owner has already written a role, the franchise cannot write it again."**

So if you've already written *"I have my own custom DataSource"* — the franchise playbook skips its DataSource chapter entirely. Your custom setup wins.

This is `@ConditionalOnMissingBean` in plain English.

---
## Chapter 5b — Phase 2: The Hiring Fair

Now the whiteboard is full. Every role is listed. Phase 2 begins.

Spring Boot looks at the whiteboard and starts hiring — **but in a very specific order**. You can't hire a waiter before you have a head chef, because the waiter needs to know the menu, which the chef defines.

```
HIRING ORDER (dependency-based):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. First: hire the DataSource
   (everyone else needs to talk to the database)

2. Then: hire the EntityManagerFactory
   (needs the DataSource to already exist)

3. Then: hire the TransactionManager
   (needs the EntityManagerFactory to already exist)

4. Then: hire your Repositories
   (need the EntityManagerFactory)

5. Then: hire your Services
   (need the Repositories)

6. Then: hire your Controllers
   (need the Services)

7. Last: hire and start Tomcat
   (needs everything else to be ready)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Each hire follows a ritual:

```
HIRING RITUAL FOR EACH PERSON:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. They walk in the door (constructor called)

2. They get given their tools
   (dependencies injected — their DataSource, their Repo, etc.)

3. They complete a brief orientation (@PostConstruct)
   "Here's how we do things here.
    Here's your workstation.
    Here's your checklist."

4. IF they need a manager to supervise them
   (they have @Transactional, @Cacheable, etc.)
   → A MANAGER is assigned to shadow them
   → More on this in a moment

5. They are now ready for work
   → Placed in the Staff Directory (ApplicationContext)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---
## Chapter 6 — The Manager Analogy (Proxies)

This is one of the most important concepts to understand, and it's almost never explained well.

Some staff members work with **a manager standing right behind them.** Every time a customer comes to that staff member, they have to go through the manager first.

```
WITHOUT A MANAGER:

Customer ──────────────────────────► Chef
                                     (chef does the work directly)


WITH A MANAGER (proxy):

Customer ──► Manager ──► Chef
             │           (chef does the actual cooking)
             │
             Manager does extra things:
             - "Start a transaction before the chef touches anything"
             - "Cache the result so we don't ask the chef twice"
             - "Check if this customer is authorized first"
             - "If the chef fails, roll everything back"
             Manager then passes result back to customer
```

This manager is what Spring calls a **proxy**.

When you put `@Transactional` on a service, Spring doesn't modify your service class. Instead, it **hires a manager** and puts that manager between the outside world and your service.

### The Critical Rule About Managers

Here's what makes this trip people up:

> **The manager only intercepts calls that come FROM OUTSIDE.**

```
CASE 1: Call comes from outside (works correctly)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Controller ──► Manager ──► OrderService.createOrder()
              ✅ Manager starts transaction
              ✅ Chef (service) does work
              ✅ Manager commits transaction


CASE 2: The chef calls their own colleague (BYPASSES manager)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
OrderService.createOrder()
    │
    └──► this.sendNotification()   ← "this" means the chef calls directly
                                      The manager is standing outside.
                                      This call NEVER passes through the manager.
                                      ❌ No transaction for sendNotification!
```

This is why calling your own `@Transactional` method from within the same class doesn't work. The call never passes through the manager. It goes directly to the real person, bypassing all the manager's duties.

---
## Chapter 7 — The Kitchen Opens (Tomcat Starts)

At some point during the hiring fair, the kitchen is set up — but the **front door stays locked.**

```
DURING HIRING FAIR:
  Kitchen equipment installed ✅
  Staff being hired ✅
  Front door: LOCKED 🔒
  No customers allowed in yet

AFTER EVERY SINGLE HIRE IS COMPLETE:
  All staff hired ✅
  All staff at their stations ✅
  Front door: OPEN 🔓
  Customers (HTTP requests) can now enter
```

This is deliberate. You do not want customers walking in while you're still hiring staff. Imagine someone ordering while the chef is still filling out their paperwork. Disaster.

Tomcat (the web server) is set up **during** the process, but it only begins **accepting requests** at the very end, once every single bean is created and ready.

---
## Chapter 8 — The Opening Checklist (Runners)

The restaurant is set up. All staff are at their stations. But before the owner unlocks the front door, they run through a final opening checklist:

```
OPENING CHECKLIST (ApplicationRunner / CommandLineRunner):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

☐ Is the soup of the day cache warmed up?
☐ Is the register initialized?
☐ Did we verify the database has the right tables?
☐ Should we seed any test data?

These are YOUR tasks that run once, at startup,
before the first customer walks in.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Only after this checklist completes does the owner flip the sign to **OPEN**.

---
## Chapter 9 — The Full Morning Timeline

Now let's put it all together as one story:

```
THE RESTAURANT OPENING — FULL TIMELINE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🕐 Owner arrives at building
   "What kind of restaurant is this?" (WebApplicationType detection)


📣 First announcement: "We're starting!"
   (ApplicationStartingEvent)


📖 Owner reads the Recipe Book
   application.yml loaded — database URL, port, profiles
   Profile-specific config loaded on top (prod, dev, etc.)
   (Environment prepared)

📣 Announcement: "Recipe book is ready!"
   (EnvironmentPreparedEvent)


🏗️ Empty restaurant framework set up
   The building structure exists but nothing inside yet
   (ApplicationContext created)


📋 PHASE 1 — THE PLANNING SESSION
   Owner walks every room → writes roles on whiteboard
   Franchise playbook consulted → standard roles added
   Each franchise chapter asks "do I apply here?"
   → Database chapter: ✅ applies (driver found)
   → Kafka chapter:    ❌ skipped (no Kafka found)
   → Security chapter: ❌ skipped (not added)
   Whiteboard now has complete list of ALL needed roles
   ZERO actual staff hired yet

📣 Announcement: "Planning complete, let's hire!"
   (ContextLoadedEvent)


👷 PHASE 2 — THE HIRING FAIR
   Each role on whiteboard → person hired, in dependency order
   → DataSource hired first (everyone needs the DB)
   → EntityManagerFactory hired (needs DataSource)
   → Repositories hired (need EntityManagerFactory)
   → Services hired (need Repositories)
   → Controllers hired (need Services)
   → Managers assigned to supervise @Transactional staff
   → Kitchen set up (Tomcat created, but door still locked)

📣 Announcement: "Kitchen set up, last few hires happening..."
   (ContextRefreshedEvent)

🔓 FRONT DOOR UNLOCKED — Tomcat starts accepting connections


📣 Announcement: "Restaurant is open internally!"
   (ApplicationStartedEvent)
   Liveness probe: ✅ "We are alive"


✅ OPENING CHECKLIST runs
   Your ApplicationRunners / CommandLineRunners execute
   Cache warmup, DB validation, startup tasks


📣 Final announcement: "Ready for customers!"
   (ApplicationReadyEvent)
   Readiness probe: ✅ "We are ready for traffic"
   (Kubernetes now sends real traffic to this pod)


🎉 RESTAURANT IS FULLY OPEN
   Every HTTP request is now being served

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Chapter 10 — The Key Insights Summarized

These are the things that should *click* after reading this:

---
### Insight 1 — Spring Boot builds itself from scratch every run

It's not loading a saved state. It's constructing everything fresh. That's why adding a dependency changes behavior — you're changing what the franchise playbook finds during its checks.

---
### Insight 2 — Auto-configuration is not magic, it's a conditional playbook

150 pre-written setup recipes. Each one has a checklist. Most get skipped because you don't have the relevant library. The ones that run set things up sensibly so you don't have to.

---
### Insight 3 — Your config always wins over the playbook

The franchise rule: *"If the owner already set it up, the playbook doesn't touch it."* You define a DataSource bean → the auto-configured one never runs. You're always in control.

---
### Insight 4 — Planning happens completely before hiring

Spring Boot decides 100% of what needs to be created **before** creating anything. This means if your configuration is contradictory or missing something, you find out at planning time — not halfway through operation.

---
### Insight 5 — The manager (proxy) pattern explains half of Spring's "magic"

`@Transactional`, `@Cacheable`, `@Async` — none of these modify your class. Spring hires a manager who stands between the outside world and your code. That manager adds the extra behavior. This is why these annotations only work when called from outside your class.

---
### Insight 6 — The door stays locked until everyone is hired

No request gets in until every single bean is created and ready. This is why apps take a few seconds to start — they're completing the entire hiring fair before accepting their first customer.

---
### Insight 7 — The two readiness levels matter in production

```
ApplicationStartedEvent → "We are alive"
                          The app didn't crash. The hiring fair is done.
                          Kubernetes: "This pod is alive, don't kill it."

ApplicationReadyEvent   → "We are ready for customers"
                          Opening checklist complete. Cache warmed. DB checked.
                          Kubernetes: "Send real traffic to this pod now."
```

In Kubernetes, liveness and readiness probes map to these two moments. Your pod can be alive but not yet ready. Both states matter.

---
## The 30-Second Version

If someone asks you this in an interview and you only have 30 seconds:

> *"Spring Boot starts by detecting what kind of app it is — web, reactive, or none — based on what's on the classpath. Then it loads all your configuration files. Then it goes through a planning phase where it decides what beans are needed — including running the auto-configuration playbook which adds standard beans for whatever libraries you've included. Then it instantiates all those beans in dependency order, wrapping some in proxy managers for transaction handling and caching. Finally, once everything is created, the web server opens its port to accept traffic. The whole thing usually takes a few seconds because it's building the application from scratch every time, not loading a cached state."*

---

> [!tip] The One-Line Summary
> Spring Boot starts up by **surveying your ingredients, consulting the franchise playbook, drawing up a complete staffing plan, hiring everyone in the right order, assigning managers where needed, and only then opening the doors to customers.**