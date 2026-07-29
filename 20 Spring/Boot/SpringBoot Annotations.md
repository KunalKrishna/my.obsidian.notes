# Role Based - meta, composed, composite

**Meta-annotation** — an annotation that sits _on top of another annotation's definition_.
**Composed annotation** — an annotation that _has_ one or more meta-annotations on it.
**Composite annotation** — a composed annotation that bundles _multiple_ meta-annotations into one convenience annotation.

> The key insight: these are **roles**, not types. `@Controller` is a composed annotation (has `@Component` on it) AND a meta-annotation (sits on `@RestController`). Same annotation, two roles, depending on your viewpoint.

![[annotations-meta-composed.png]]

---
### The three terms, locked in by concrete code

#### Meta-annotation — "an annotation ON an annotation's definition"
```java
// Look INSIDE the definition of @Service:
public @interface Service {
    //      ↑ this IS the composed annotation
}

// The @Component sitting ON TOP of @Service's definition
// is the meta-annotation.
@Component        // ← META-ANNOTATION (it annotates @Service)
public @interface Service { }
```

Meta-annotation is a role. `@Component` plays it here because it sits inside another annotation's definition, donating its behaviour down.
#### Composed annotation — "an annotation that HAS a meta-annotation on it"

```java
// @Service is the COMPOSED annotation.
// It is "composed of" @Component.
// When Spring sees @Service on a class, it treats it
// as if @Component were there directly.

@Component          // meta-annotation on @Service
public @interface Service { }

// Usage — Spring sees @Service, finds @Component on it,
// and registers OrderService as a bean.
@Service
public class OrderService { }
```
#### Composite annotation — "a composed annotation that bundles MULTIPLE meta-annotations"
The distinction from a plain composed annotation is just quantity — two or more meta-annotations bundled for convenience.

```java
// @RestController = composite annotation.
// It bundles TWO meta-annotations into one.
@Controller       // ← meta-annotation 1
@ResponseBody     // ← meta-annotation 2
public @interface RestController { }

// Without composite, you'd write both every time:
@Controller
@ResponseBody
public class OrderController { }    // ❌ verbose

// With composite, one annotation does both:
@RestController
public class OrderController { }    // ✅ clean
```

```java
// @SpringBootApplication = the biggest composite in Spring Boot.
// Bundles THREE meta-annotations.
@SpringBootConfiguration   // ← meta-annotation 1
@EnableAutoConfiguration   // ← meta-annotation 2
@ComponentScan             // ← meta-annotation 3
public @interface SpringBootApplication { }
```
## Meta-Annotation (`@Controller`)
An annotation that is applied to _another_ annotation definition is called a **meta-annotation** (an annotation about an annotation). 
- **`@Controller`** acts as a meta-annotation for `@RestController`.
- **`@Component`** acts as a meta-annotation for `@Controller`.
- Because `@Controller` is annotated with `@Component`, any class marked with `@Controller` is automatically registered as a Spring bean. 
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  // <-- meta-annotation 
public @interface Controller {
//      ↑ this IS the composite annotation
}
```
## Composed Annotation (`@RestController`)
When an annotation is built by combining one or more existing annotations, it is called a **composed annotation**. 
- **`@RestController`** is a composed annotation.
- It is built by combining **`@Controller`** and **`@ResponseBody`**.
- Instead of writing both annotations on your class, you use `@RestController` as a shortcut. 
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Controller    // <-- meta-annotation 1
@ResponseBody  // <-- meta-annotation 2
public @interface RestController {
//      ↑ this IS the composite(a composed annotaion with >1 meta-anntn) annotation
	//...
}
```
### The same annotation playing both roles - key to end the confusion
`@Controller` is the clearest example:
```
Looking DOWN at @Controller's own definition:
  @Component is the meta-annotation
  @Controller is the composed annotation

Looking UP at @RestController's definition:
  @Controller is the meta-annotation
  @RestController is the composed/composite annotation
```

These terms describe **the relationship between two annotations**, not a fixed property of one. Any annotation that is composed can simultaneously be a meta-annotation for something above it.

---
### The one-line test for each term

|You're looking at...|Term to use|
|---|---|
|An annotation sitting inside another annotation's `@interface` definition|meta-annotation|
|The `@interface` that has one meta-annotation on its definition|composed annotation|
|The `@interface` that has two or more meta-annotations on its definition|composite annotation|

-------------

# Src code of @Component hierarchy annotations

![[Component hierarchy.png]]

```
A stereotype's PRIMARY job is classification, not behaviour.
Extra behaviour (@Controller's method scanning, @Repository's
exception translation) is a bonus on top of the classification.

@Service has no bonus behaviour — but it still earns its place
because classification IS a first-class purpose, not a
consolation prize.
```

In other words, calling them "stereotype annotations" is Spring explicitly telling you: _these exist to label architectural roles, and the bean registration is a consequence of that labelling, not the other way around._

## @Component
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Indexed  
public @interface Component {  
    //
}
```
## @Service
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Service {  
    //
}
```
## @Repository
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Repository {
    //
}
```
## @Controller
`@Controller` unlocks — method scanning by `RequestMappingHandlerMapping`
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface Controller {  
    //
}
```

```
@Controller is only meaningfully different from @Component
because of what it lets RequestMappingHandlerMapping do.

Remove the thing RequestMappingHandlerMapping would do,
and the difference between @Controller and @Component collapses to zero.
```

`@Controller = @Component + @RequestMapping("/home")`
### @RestController
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Controller  
@ResponseBody  
public @interface RestController {  
    //
}
```
## @Configuration
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {  
    //
}
```
### @SpringBootConfiguration
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {  
    //
}
```

NOTE : Except for @RestController they all look the same. 
# Nuance in @Component hierarchy annotations 

**Q: Can we replace @Service/Controller(child) with @Component(parent) and vice-versa or @Service with @Controller and vice-versa? Why (not) ? Explain w/ examples.** 

A: If we see the source code it seems we can. We can replace those annotations which is purely semantic and which performs task of just registering a bean. But some annotations do unique "extra" work and they are not replaceable. The complete truth table
### Part 1: You are RIGHT about `@Service` vs `@Component`
They are genuinely, functionally identical. Here's their actual Spring source:

```java
// @Service source
@Component          // the only meaningful line
public @interface Service { String value() default ""; }

// @Component source
public @interface Component { String value() default ""; }
```

No Spring framework code looks for `@Service` specifically and treats it differently from `@Component`. Swapping them won't break a single thing at runtime.

```java
// These two are 100% equivalent at runtime. Nothing breaks.
@Service            public class OrderService { }
@Component          public class OrderService { }  // ✅ identical behaviour
```
### Part 2: You are WRONG about `@Controller` vs `@Component`
This is where the code breaks. `@Controller` is not just semantic — the `DispatcherServlet`'s handler detector, `RequestMappingHandlerMapping`, explicitly checks for it:

```java
// Actual Spring source inside RequestMappingHandlerMapping:
@Override
protected boolean isHandler(Class<?> beanType) {
    return (
        AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||  // ← checks THIS
        AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class)  // or THIS
    );
}
```

If `isHandler()` returns `false`, Spring **never scans that class's methods** for `@GetMapping`, `@PostMapping`, etc. They are silently ignored — no error, no warning, just dead endpoints.

```java
// ✅ Works — isHandler() returns true because @Controller is present
@Controller
public class HomeController {
    @GetMapping("/home")
    public String home() { return "home"; }  // endpoint registered ✓
}

// ❌ Silently BROKEN — isHandler() returns false for @Component
@Component
public class HomeController {
    @GetMapping("/home")
    public String home() { return "home"; }  // endpoint NEVER registered ✗
                                              // no error thrown, just 404
}

// ✅ Works — isHandler() second condition catches @RequestMapping at class level
@Component
@RequestMapping("/home")                     // class-level annotation saves it
public class HomeController {
    @GetMapping("/")
    public String home() { return "home"; }  // endpoint registered ✓
}
```

The third case shows the escape hatch — but nobody writes it that way in practice.
### Part 3: `@Repository` also has real extra behaviour
You didn't ask about this, but it's worth knowing — `@Repository` does more than register a bean. `PersistenceExceptionTranslationPostProcessor` specifically looks for it to wrap raw database exceptions into Spring's `DataAccessException` hierarchy:

```java
@Component
public class UserRepository {
    public User find(Long id) {
        // throws raw HibernateException / SQLException on failure
        // no translation — caller must handle vendor-specific exception
    }
}

@Repository
public class UserRepository {
    public User find(Long id) {
        // Spring catches the raw exception and rewraps it as
        // DataAccessException (or subclass) automatically
        // caller handles one consistent exception type ✓
    }
}
```

---
### The complete truth table

|Annotation|Just registers a bean?|Extra Spring framework behaviour|
|---|---|---|
|`@Component`|Yes|None — it is the base|
|`@Service`|Yes|None — purely semantic|
|`@Repository`|No|Exception translation via `PersistenceExceptionTranslationPostProcessor`|
|`@Controller`|No|`RequestMappingHandlerMapping` detects `@GetMapping` / `@PostMapping` methods|
|`@Configuration`|No|CGLIB proxy created; `@Bean` methods processed|

---
### So why does `@Service` exist at all?

Three reasons — none of them about Spring's behaviour, all of them about humans:
1. **Readability.** A fresh developer reading the code immediately knows what layer they're in.
2. **AOP targeting.** You can write aspects that apply only to the service layer:

```java
// This advice applies ONLY to @Service classes, not @Repository or @Controller
@Before("within(@org.springframework.stereotype.Service *)")
public void logServiceCall(JoinPoint jp) { ... }

// If everything were @Component, you'd lose this precision
```
3. **Future-proofing.** Spring could add special behaviour to `@Service` in a future version (just as `@Repository` already has it). Using the right annotation means you'd get it for free.

---
### The verdict on your conclusion

```java
@Component everywhere?

✅ Safe for:  @Service → @Component
❌ Breaks:    @Controller → @Component  (silent 404 on all endpoints)
❌ Loses:     @Repository → @Component  (loses DB exception translation)
❌ Breaks:    @Configuration → @Component  (loses CGLIB proxy, @Bean methods misbehave)

// This would find the bean with @Controller, miss it with @Component 
Map<String, Object> controllers = applicationContext.getBeansWithAnnotation(Controller.class);
```

Your instinct — that redundant-looking abstractions should be questioned — is exactly the right engineering mindset. It just turns out that in this case, only `@Service` is truly just a label. The others all carry real weight.


# @Configuration vs @SpringBootConfiguration

While `@SpringBootConfiguration` is a composed annotation built directly on top of `@Configuration`, they serve different architectural purposes and have strict structural rules regarding where they can be applied. 

---
The Key Difference

The core difference lies in **how they are discovered** by Spring and **how many instances** are allowed in an application: 

- **`@Configuration` (The Component Worker)**: Tells Spring that a class defines one or more custom `@Bean` methods. You can have **dozens** of `@Configuration` classes sprinkled across various packages in your project. They are lazily discovered via package scanning (`@ComponentScan`). 
- **`@SpringBootConfiguration` (The Application Anchor)**: A specialized, Spring Boot-specific version of `@Configuration`. It is meant to mark the **single main entry point configuration** for the entire Spring Boot application. It acts as a beacon, allowing the underlying framework to automatically locate the main application context. 

---

Direct Feature Comparison

|Feature|`@Configuration`|`@SpringBootConfiguration`|
|---|---|---|
|**Origin**|Part of core Spring Framework.|Part of Spring Boot.|
|**Allowed Quantity**|**Unlimited** per application.|**Exactly one** per application.|
|**Primary Use Case**|Declaring typical beans (e.g., database configs, security setups).|Declaring the root bootstrap class of the app.|
|**Test Auto-Discovery**|Cannot be found automatically unless picked up by component scanning.|**Automatically discovered** by slice testing frameworks like `@SpringBootTest`.|

---

Why You Cannot Use Them Interchangeably

1. The "Only One" Rule
The [official Spring Boot Documentation](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/SpringBootConfiguration.html) states that an application should **only ever include one `@SpringBootConfiguration`**. Because `@SpringBootApplication` already includes `@SpringBootConfiguration` under the hood, adding `@SpringBootConfiguration` to another random class will break your application context startup.

2. Test Execution Pitfalls
When you write an integration test using **`@SpringBootTest`**, the testing framework needs to find the root configuration file. It does this by searching upward through your package hierarchy until it hits a class marked with `@SpringBootConfiguration`. 
- If you swap your main class to standard `@Configuration`, your integration tests will throw errors because they won't automatically find the application context configuration. 
- If you label a standard sub-package service class as `@SpringBootConfiguration`, your integration tests may treat that small slice as the entry point of your _entire_ application, skipping your actual main configuration.

How to use them correctly

- Use **`@SpringBootApplication`** (which wraps `@SpringBootConfiguration`) exactly once on your main bootstrap class.
- Use **`@Configuration`** on all other classes where you manually register custom beans (such as `SecurityConfig`, `DatabaseConfig`, or `WebMvcConfig`).

## The history : `@Configuration` came first

**Before Spring Boot 1.2 (pre-2015)** There was no `@SpringBootApplication` at all. Developers wrote all three annotations manually every time:
```java
// Every Spring Boot main class looked like this before 1.2
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

**Spring Boot 1.2 (2015)** `@SpringBootApplication` introduced as a convenience annotation — but it used `@Configuration` inside it:
```java
// @SpringBootApplication in Spring Boot 1.2 — used plain @Configuration
@Configuration              // ← plain Spring Framework annotation
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { }
```

**Spring Boot 1.4 (2016)** `@SpringBootConfiguration` introduced. `@SpringBootApplication` swapped `@Configuration` for it:
```java
// @SpringBootApplication from 1.4 onwards — uses @SpringBootConfiguration
@SpringBootConfiguration    // ← Spring Boot-specific replacement
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication { }
```

---
## Why was `@SpringBootConfiguration` created at all?

One precise reason: **`@SpringBootTest` needed a reliable way to find the main class.**

`@SpringBootTest` auto-detects your primary configuration by walking up the package tree. If it searched for `@Configuration`, it would find every `@Configuration` class in your project:

```java
com.example.MyApp           ← @Configuration ✓ (main class)
com.example.SecurityConfig  ← @Configuration ✓ (security beans)
com.example.DatabaseConfig  ← @Configuration ✓ (datasource beans)
com.example.CacheConfig     ← @Configuration ✓ (cache beans)
// Four hits. Which one is the primary entry point? Impossible to tell.
```

By introducing `@SpringBootConfiguration` and using it **only on the main class**, the search becomes unambiguous:

```java
com.example.MyApp           ← @SpringBootConfiguration ✓ (only one — found it)
com.example.SecurityConfig  ← @Configuration           (ignored by the search)
com.example.DatabaseConfig  ← @Configuration           (ignored by the search)
```

`@SpringBootTest` searches for `@SpringBootConfiguration` specifically, always finds exactly one, and uses it as the application context to boot for tests.
## The actual source code difference

```java
// Spring Framework's @Configuration
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component                  // is a component — picked up by scan
public @interface Configuration {
    boolean proxyBeanMethods() default true;
}

// Spring Boot's @SpringBootConfiguration
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration              // IS a @Configuration — inherits everything
@Indexed                    // ← extra: compile-time classpath index optimisation
public @interface SpringBootConfiguration {
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;
}
```

`@SpringBootConfiguration` adds exactly two things over `@Configuration`: 

**`@Indexed`** — tells the Spring component index (generated at compile time into `META-INF/spring.components`) to register this class, making classpath scanning faster. A minor performance optimisation.  

**Identity as a unique marker** — the real addition. Not a technical annotation, just the fact that `@SpringBootConfiguration` is distinct from `@Configuration` so tooling (`@SpringBootTest`, Spring Boot DevTools, IDE plugins) can locate the main class reliably.  

---
## The complete difference table

|x|`@Configuration`|`@SpringBootConfiguration`|
|---|---|---| 
|Enables `@Bean` processing|✅|✅ (inherits it)|
|CGLIB proxy for bean methods|✅|✅ (inherits it)|
|Picked up by `@ComponentScan`|✅|✅ (inherits `@Component`)|
|Appears on many classes|✅ anywhere|❌ main class only|
|`@SpringBootTest` auto-detection|❌ too many hits|✅ exactly one hit|
|`@Indexed` for scan performance|❌|✅|
|Introduced in|Spring Framework|Spring Boot 1.4|

> **The one-line answer:** `@SpringBootConfiguration` is `@Configuration` with a unique identity. Functionally identical — same bean processing, same CGLIB proxy, same component scan visibility. The only addition is serving as an **unambiguous marker** that Spring Boot tooling (especially `@SpringBootTest`) can search for without false positives.


# @SpringBootConfiguration vs @EnableAutoConfiguration

Both use the **exact same mechanism** — `@Configuration` classes with `@Bean` methods. The difference is purely **who wrote them** and **when they activate**. 

**`@SpringBootConfiguration`** — marks YOUR class as a `@Configuration`. The `@Bean` methods inside it: **you write them.**

**`@EnableAutoConfiguration`** — loads `@Configuration` classes from `spring-boot-autoconfigure.jar`. The `@Bean` methods inside those: **Spring Boot's team wrote them.** 

Same mechanism. Different authors. 

```
@SpringBootConfiguration               @EnableAutoConfiguration
       ↓                                        ↓
  YOUR @Configuration                SPRING'S @Configuration classes
  class (the main class)             (~140 of them, in autoconfigure.jar)
       ↓                                        ↓
  @Bean methods you write            @Bean methods Spring Boot's team wrote
       ↓                                        ↓
  Always runs                        Runs only when @Conditional passes
       ↓                                        ↓
            Both register beans into the same IoC container
                    and YOU always win over Spring's version 
```

>The word "configuration" means exactly the same thing in both annotations — `@Configuration` classes with `@Bean` methods. The only difference is **who holds the pen**.

![[SBConfig nuiances.png]]

>[!tip] The Key Insight to Remember 
>Auto-Configuration is not magic and it's not hidden configuration. It's **regular `@Configuration` classes with `@Conditional` guards** that have been written once by the Spring Boot team and packaged into `spring-boot-autoconfigure.jar`. They run only when the right classes are on the classpath and only when you haven't provided your own configuration. The "auto" means automatic detection and automatic back-off — not that it's doing something mysterious. Every bean it creates, you could have created yourself in a `@Configuration` class — Spring Boot just did it so you don't have to.


obsidian://open?vault=public.notes&file=Java%20Interview%20Prep%2FSpringBoot%2FSpringBoot%20RoadMap%2FAnnotations%2F%40EnableAutoConfiguration

[[@EnableAutoConfiguration]]