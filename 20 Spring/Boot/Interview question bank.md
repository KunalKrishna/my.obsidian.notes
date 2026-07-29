
```less
STRUCTURED 2-MINUTE ANSWER:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"Spring Boot startup happens in roughly 6 phases:

FIRST — SpringApplication is constructed.
It detects the application type (servlet, reactive, or none),
and loads ApplicationContextInitializers and Listeners
from spring.factories files across all jars on the classpath.

SECOND — The Environment is prepared.
This is where application.yml is loaded, profiles are activated,
and all property sources are assembled in priority order —
command line args override env vars override properties files.

THIRD — The ApplicationContext is created.
For a web app, this is the AnnotationConfigServletWebServerApplicationContext.

FOURTH — refreshContext() runs, which is where most of the work happens.
It has two major sub-phases:

  The bean definition phase: ConfigurationClassPostProcessor runs,
  which does component scanning, processes @Configuration classes and @Bean
  methods, and — crucially — evaluates all auto-configurations.
  That's where Spring reads the AutoConfiguration.imports files from all jars,
  and for each configuration, evaluates @Conditional annotations to decide
  whether to apply it. At the end, all bean definitions are registered
  but no instances exist yet.

  The instantiation phase: all singleton beans are created in dependency order.
  For each bean: constructor called, dependencies injected, @PostConstruct runs,
  and finally if the bean needs AOP — @Transactional, @Cacheable, @Async —
  it's wrapped in a CGLIB proxy. The proxy, not the original object,
  is stored in the context.

  Also during refresh: the embedded Tomcat is created and started,
  though it begins accepting requests only at the END of refresh.

FIFTH — ApplicationStartedEvent fires, then ApplicationRunners
and CommandLineRunners execute.

SIXTH — ApplicationReadyEvent fires, ReadinessState is set to
ACCEPTING_TRAFFIC, and in Kubernetes, this is when the pod gets traffic.

The key insight I'd highlight: the auto-configuration mechanism is what
makes Spring Boot 'opinionated.' You add a dependency to the classpath,
Spring Boot detects it via @ConditionalOnClass, and configures sensible
defaults — but steps back if you've defined your own bean
via @ConditionalOnMissingBean."
```
# Spring Internal 

### The classic opener that leads everywhere

> **"Walk me through what happens internally when a Spring Boot application starts."**

This one question is the trunk. Every question below is a branch off it. At senior level, interviewers expect you to go 2-3 levels deep on any branch they pull.

### The full list, grouped by theme

**Bootstrapping & startup sequence**
- What does `SpringApplication.run()` do internally, step by step?
- How does Spring Boot decide which `ApplicationContext` to create — servlet, reactive, or none?
- When exactly does the embedded Tomcat start during the startup sequence?
- What are `SpringApplicationRunListeners` and what events fire during startup, in order?
- What are `ApplicationRunner` and `CommandLineRunner`? When do they run relative to context refresh?
- What is `prepareContext()` doing before `refreshContext()` is called?

**Auto-configuration internals**
- How does `@EnableAutoConfiguration` work internally?
- What is `spring.factories` / `AutoConfiguration.imports` and how is it read?
- How does Spring Boot avoid loading all 140+ auto-configuration classes every time?
- What is `@ConditionalOnClass` — how does it check without instantiating the class?
- In what order do user-defined beans and auto-configured beans get registered? Who wins on conflict?
- How would you write your own auto-configuration class?

**`@ComponentScan` internals**
- What exactly does `@ComponentScan` scan — and what does it deliberately skip?
- How is the main class registered if `@ComponentScan` doesn't scan it? ← your chicken-and-egg
- If you move a `@Service` class outside the main class package, why does Spring miss it?
- How do you make `@ComponentScan` scan multiple packages?

**`@Configuration` and `@Bean` internals**
- Why does Spring create a CGLIB proxy for `@Configuration` classes?
- What breaks if you call one `@Bean` method from another inside a `@Configuration` class without the proxy?
- What is `proxyBeanMethods = false` on `@Configuration` and when would you use it?
- What is the difference between `@Configuration` and `@Component` when both can host `@Bean` methods?

**Bean lifecycle internals**
- What is the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`?
- At what point in the lifecycle does dependency injection happen?
- When does `@PostConstruct` run relative to injection?
- How does Spring detect and handle circular dependencies? Where does it fail?
- Can constructor injection resolve circular dependencies? Why not?

**AOP and proxy internals (closely tied to bootstrapping)**
- How does `@Transactional` work internally — what creates the proxy?
- Why does `@Transactional` not work on private methods?
- Why does calling a `@Transactional` method from within the same class bypass the transaction?
- What is the difference between JDK dynamic proxy and CGLIB proxy? When does Spring use each?

**Environment and properties**
- What is the property source priority order in Spring Boot?
- How does Spring Boot load `application.yml` vs `application-prod.yml`?
- How does `@Value` get resolved — what reads the property source at injection time?

---
### The ones most likely at a Senior QA / Dev interview

Given your background, the highest-probability questions in a technical interview are:

```
1. Walk me through SpringApplication.run() — the full walkthrough
2. How does auto-configuration work? (@Conditional system)
3. What is the difference between @Configuration and @Component?
4. How does @Transactional work? Why doesn't it work on private methods?
5. How does Spring resolve circular dependencies?
6. What is BeanPostProcessor used for? Give an example.
```

------------------
# Internal Working 

```java
main()
  │
  └─► SpringApplication.run()
            │
            ├─► 1. SpringApplication CREATED
            │         ├── Detect application type
            │         ├── Load ApplicationContextInitializers
            │         ├── Load ApplicationListeners
            │         └── Find main application class
            │
            ├─► 2. BOOTSTRAP PHASE
            │         ├── Create BootstrapContext
            │         └── Fire ApplicationStartingEvent
            │
            ├─► 3. ENVIRONMENT PREPARED
            │         ├── Create Environment object
            │         ├── Load property sources (yml, properties, env vars)
            │         ├── Activate profiles
            │         └── Fire EnvironmentPreparedEvent
            │
            ├─► 4. APPLICATION CONTEXT CREATED
            │         ├── Choose context type (Servlet / Reactive / None)
            │         └── Fire ContextPreparedEvent
            │
            ├─► 5. CONTEXT REFRESHED  ◄── Most work happens here
            │         ├── Component scanning
            │         ├── @Configuration class processing
            │         ├── Auto-configuration
            │         ├── Bean definition loading
            │         ├── Singleton bean instantiation
            │         ├── BeanPostProcessor processing
            │         └── Embedded server started (Tomcat)
            │
            ├─► 6. POST-REFRESH
            │         ├── Fire ApplicationStartedEvent
            │         ├── Run ApplicationRunner / CommandLineRunner beans
            │         └── Fire ApplicationReadyEvent
            │
            └─► ✅ APPLICATION IS RUNNING
```


```java
The main class is registered BEFORE @ComponentScan runs. 
@ComponentScan runs BECAUSE the main class was registered. 

Chicken: @ComponentScan needs main class 
Egg: main class needs @ComponentScan to be processed 

Resolution: the main class is injected manually as the seed. 
			@ComponentScan processing happens as a consequence.
```