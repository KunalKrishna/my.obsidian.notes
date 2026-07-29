---
tags:

- spring-boot
- java
- spring/beans
- spring/ioc
- spring/annotations created: 2025-07-18 status: permanent
---

# How to Register Beans with the Spring Container

## The Sharpest Way to Say It

> [!tip] Two mechanisms. One goal. `@ComponentScan` → _"Go FIND all beans wearing a badge."_ (implicit) `@Configuration + @Bean` → _"Here's a bean — I'll build it for you."_ (explicit)

Both paths end at the same place: the **Spring IoC container**, which holds and injects them.

---
## The Two Distinct Groups

```
Two mechanisms to register beans
│
├── Mechanism 1 — Implicit  (ComponentScan discovers them)
│   @Component, @Service, @Repository, @Controller, @RestController
│
└── Mechanism 2 — Explicit  (@Bean methods declare them)
    @Configuration + @Bean
```
![[Pasted image 20260719065332.png]]
## Mechanism 1 — Stereotype Annotations (Implicit)

Spring walks your **entire package tree** at startup looking for classes wearing one of these annotations. Any class it finds gets instantiated and registered as a bean automatically.
### The 5 Stereotype Annotations

| Annotation        | Layer          | Extra Behaviour                                                         |
| ----------------- | -------------- | ----------------------------------------------------------------------- |
| `@Component`      | Generic / any  | Base annotation — all others extend this                                |
| `@Service`        | Business logic | Semantic marker only — no extra behaviour                               |
| `@Repository`     | Data access    | Translates DB exceptions to Spring's `DataAccessException`              |
| `@Controller`     | Web / MVC      | Marks class as a request handler for `DispatcherServlet`                |
| `@RestController` | REST APIs      | `@Controller` + `@ResponseBody` — auto-serialises return values to JSON |

> [!info] Use when Registering **your own classes** — services, repositories, controllers, components. You can annotate them yourself, so just put the badge on and let `@ComponentScan` find them.

```java
@Service
public class OrderService {
    // No @Bean method needed anywhere.
    // @ComponentScan finds this at startup and registers it.
}

@Repository
public class UserRepository {
    // Same — Spring discovers and registers this automatically.
}

@RestController
public class PaymentController {
    // Also discovered. All three end up in the IoC container.
}
```
### How `@ComponentScan` triggers this

`@ComponentScan` is included inside `@SpringBootApplication`. It scans the **current package + all sub-packages** by default.

```java
@SpringBootApplication   // includes @ComponentScan under the hood
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

> [!warning] Common mistake If your `@Service` class lives in a package **outside** the main class's package tree, `@ComponentScan` won't find it. Either move the class, or extend the scan: `@ComponentScan(basePackages = "com.example.other")`.

---
## Mechanism 2 — `@Bean` Methods (Explicit)

You write a method inside a `@Configuration` class. Spring calls that method at startup and registers the **return value** as a bean. You own the construction logic entirely.

> [!info] Use when Registering **third-party / library classes** you don't own and can't annotate. e.g. `BCryptPasswordEncoder`, `RestTemplate`, `ObjectMapper`, `DataSource`. You can't put `@Service` on `BCryptPasswordEncoder` — it's not your code. So you write a `@Bean` method yourself.

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        // YOU control how this object is constructed.
        return new BCryptPasswordEncoder();
    }

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate rt = new RestTemplate();
        rt.setErrorHandler(new CustomErrorHandler());
        return rt;  // Spring takes this object and registers it as a bean.
    }
}
```

```java
@Configuration
public class CacheConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

> [!note] `@Configuration` is itself a stereotype `@Configuration` is a specialisation of `@Component`. So a `@Configuration` class is **also** picked up by `@ComponentScan`, and its `@Bean` methods are processed to register the additional beans they declare.

---

The difference is what happens **after** `@ComponentScan` finds it:

```
@ComponentScan finds a class...

├── It's @Service / @Repository / @Controller / @RestController / @Component
│     └── Action: register the class itself as a bean. Done.
│
└── It's @Configuration
      └── Action: register the class as a bean (like above)
          AND ALSO process every @Bean method inside it
               → each method's return value = another bean
```

```
@ComponentScan
  └── finds @Component + ALL its specializations
        ├── @Service            → registers class as bean
        ├── @Repository         → registers class as bean
        ├── @Controller         → registers class as bean
        ├── @RestController     → registers class as bean
        └── @Configuration      → registers class as bean
                                  + processes @Bean methods
                                    → each return value = extra bean
```

|Found by @ComponentScan|What gets registered|
|---|---|
|`@Service` / `@Repository` / `@Controller` / `@RestController`|The class itself|
|`@Configuration`|The class itself **+** every `@Bean` method's return value|


---
## The One-Line Test — Which Mechanism to Use?

> [!important] Decision rule **"Can I put `@Service` / `@Component` on this class?"**
> 
> - **Yes** (it's my class) → annotate it → `@ComponentScan` finds it.
> - **No** (it's a library class) → write a `@Bean` method in a `@Configuration` class.

---

## `@Configuration` vs `@SpringBootConfiguration`

This is a common point of confusion.

||`@Configuration`|`@SpringBootConfiguration`|
|---|---|---|
|What it is|Standard Spring annotation|Spring Boot's specialisation of `@Configuration`|
|Where you use it|**Every config class you write**|Only the main class (inside `@SpringBootApplication`)|
|Who writes it|You, repeatedly|Already written for you — you never write it by hand|
|Enables `@Bean`?|Yes|Yes (it extends `@Configuration`)|


```java
// ✅ You write this — on ALL your config classes
@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder encoder() { return new BCryptPasswordEncoder(); }
}

// ❌ You never write this yourself — it's inside @SpringBootApplication
@SpringBootConfiguration   // ← internal detail, hands off
@EnableAutoConfiguration
@ComponentScan
public class MyApp {
    public static void main(String[] args) { SpringApplication.run(MyApp.class, args); }
}
```

> [!tip] Memory hook You'll write `@Configuration` hundreds of times. You'll write `@SpringBootConfiguration` zero times.

---

## The Annotation Hierarchy

```
@Component  ← root of all stereotypes
│
├── @Service            (business logic — semantic only)
├── @Repository         (data layer — adds exception translation)
├── @Controller         (web layer — MVC request handler)
│   └── @RestController (@Controller + @ResponseBody)
└── @Configuration      (config class — hosts @Bean methods)
    └── @SpringBootConfiguration  (main class only — inside @SpringBootApplication)
```

---

## Everything Together — Quick Reference

```java
// ─── Mechanism 1: Spring FINDS these (implicit) ───────────────

@Service
public class OrderService { ... }           // business logic

@Repository
public class OrderRepository { ... }        // data access

@RestController
public class OrderController { ... }        // REST endpoint


// ─── Mechanism 2: You DECLARE these (explicit) ────────────────

@Configuration
public class AppConfig {

    @Bean
    public PasswordEncoder encoder() {       // third-party class
        return new BCryptPasswordEncoder();
    }

    @Bean
    public RestTemplate restTemplate() {     // third-party class
        return new RestTemplate();
    }
}


// ─── All of the above end up here ─────────────────────────────

// Spring IoC Container
// ├── OrderService        ← via @ComponentScan
// ├── OrderRepository     ← via @ComponentScan
// ├── OrderController     ← via @ComponentScan
// ├── PasswordEncoder     ← via @Bean method
// └── RestTemplate        ← via @Bean method
```

---

## Related Notes

- [[Spring Boot — How @SpringBootApplication Works Internally]]
- [[Spring Boot — @ComponentScan vs @SpringBootConfiguration]]
- [[Spring Boot — Constructor vs Field Injection]]
- [[Spring Boot — Bean Lifecycle]]
- [[Spring Boot — N+1 Problem in JPA]]