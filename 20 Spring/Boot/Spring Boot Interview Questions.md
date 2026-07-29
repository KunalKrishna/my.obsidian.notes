

```
╔══════════════════════════════════════════════════════════════════╗
║     SPRING BOOT INTERVIEW CHECKLIST — MID-LEVEL (5 YOE)          ║
╚══════════════════════════════════════════════════════════════════╝

WEEK 1 — CORE FOUNDATIONS
□ Explain IoC, DI, and bean scopes confidently
□ Know constructor vs. field injection and why constructor is preferred
□ Explain @SpringBootApplication internals
□ Understand auto-configuration and @Conditional annotations
□ Know externalized config priority order
□ Explain @ConfigurationProperties vs @Value
□ Walk through full HTTP request lifecycle (DispatcherServlet)
□ Handle exceptions with @ControllerAdvice
□ Apply Bean Validation (@Valid, custom validators)

WEEK 2 — DATA, SECURITY & TESTING
□ Explain JPA vs Hibernate vs Spring Data JPA
□ Know all @Transactional propagation types + rollback rules
□ Solve the N+1 problem (EntityGraph / JOIN FETCH)
□ Know JpaRepository vs CrudRepository
□ Set up Flyway or Liquibase migrations
□ Implement JWT-based authentication end-to-end
□ Configure SecurityFilterChain (Spring Security 6)
□ Know @PreAuthorize / method-level security
□ Write unit tests with JUnit 5 + Mockito
□ Write controller tests with @WebMvcTest + MockMvc
□ Write JPA tests with @DataJpaTest
□ Know difference between @Mock and @MockBean
□ Know how to use TestContainers

WEEK 3 — MICROSERVICES & OBSERVABILITY
□ Explain service discovery (Eureka/Consul)
□ Configure Spring Cloud Gateway with filters
□ Implement Circuit Breaker with Resilience4j
□ Use @FeignClient for inter-service calls
□ Set up Kafka producer/consumer (@KafkaListener)
□ Know @Async and thread pool configuration
□ Know @Scheduled (cron, fixed rate, fixed delay)
□ Set up @Cacheable with Redis
□ Configure Actuator endpoints + secure them
□ Understand distributed tracing (trace IDs, Zipkin)
□ Know structured logging with MDC

WEEK 4 — ADVANCED & PRACTICAL
□ Know Spring Boot 3.x / Jakarta EE changes
□ Explain why @Transactional self-invocation fails
□ Write a custom AOP aspect (@Around)
□ Know application lifecycle events
□ Write a Dockerfile for a Spring Boot app
□ Know K8s liveness/readiness probe configuration
□ Know 12-factor app principles
□ Prepare 5+ STAR behavioral stories
□ Prepare answers for scenario-based questions
□ Complete 2 full mock interviews
□ Prepare 5 questions to ask the interviewer

SCENARIO QUESTIONS — CAN YOU ANSWER THESE?
□ App is slow — how do you investigate?
□ LazyInitializationException in prod — what do you do?
□ Distributed transaction across 2 services — how?
□ How do you design a rate-limited API?
□ How do you migrate monolith to microservices?
□ How do you secure sensitive config values?

CONFIDENCE CHECK — RATE YOURSELF (1–5)
□ Spring Core & Auto-config        [ /5 ]
□ REST API Design                  [ /5 ]
□ Spring Data JPA                  [ /5 ]
□ Spring Security / JWT            [ /5 ]
□ Testing (unit + integration)     [ /5 ]
□ Microservices Patterns           [ /5 ]
□ Messaging (Kafka/RabbitMQ)       [ /5 ]
□ Caching & Performance            [ /5 ]
□ Observability (Actuator/Tracing) [ /5 ]
□ Docker / K8s basics              [ /5 ]
□ Behavioral / Scenario questions  [ /5 ]
```
# Spring Boot Interview Questions & Scratchpad Notes
### ~10 Questions Per Section with Answer Guides

---
## 📦 SECTION 1: Spring Core & Auto-Configuration

---

**Q1. What is Inversion of Control (IoC) and how does Spring implement it?**

```
SCRATCHPAD NOTES:
- IoC = you don't create objects, the framework does it for you
- Traditional code: MyService s = new MyService(); ← you control creation
- With IoC: Spring creates, wires, and manages object lifecycle
- Spring's IoC container = ApplicationContext (BeanFactory is lower-level)
- Two forms of DI: Constructor injection, Setter injection (field injection is bad)
- Key benefit: loose coupling, easier to test, swap implementations

MENTION: ApplicationContext > BeanFactory (ApplicationContext adds AOP, events, i18n)
GOTCHA: Don't say "IoC and DI are the same thing" — IoC is the principle, DI is how Spring achieves it.
```

---

**Q2. What is the difference between constructor injection, setter injection, and field injection? Which is preferred and why?**

```
SCRATCHPAD NOTES:
- Field injection (@Autowired on field): simplest to write, but:
    * Can't make fields final (not immutable)
    * Harder to test — can't inject without Spring context
    * Hides dependencies (not visible in constructor : code smell)
- Setter injection: optional dependencies; allows changing after construction (bad for most cases)
- Constructor injection (PREFERRED):
    * Dependencies are explicit and required
    * Fields can be final → truly immutable
    * Easy to unit test — just call new MyService(mockRepo)
    * Spring 4.3+: @Autowired not even needed if single constructor
- Spring team officially recommends constructor injection

CODE EXAMPLE TO MENTION:
  private final UserRepository repo;
  public UserService(UserRepository repo) { this.repo = repo; }

GOTCHA: Circular dependency with constructor injection → Spring throws error at startup (good, fail-fast)
        With field injection, circular deps can go undetected longer
```

---

**Q3. Explain the Spring Bean lifecycle from instantiation to destruction.**

```
SCRATCHPAD NOTES:
Order of events:
1. Bean instantiation (constructor called)
2. Dependency injection (fields/setters populated)
3. BeanNameAware, BeanFactoryAware callbacks (if implemented)
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct method runs
6. InitializingBean.afterPropertiesSet() (if implemented)
7. Custom init-method (if configured)
8. BeanPostProcessor.postProcessAfterInitialization()
9. Bean is ready for use
--- on shutdown ---
10. @PreDestroy method runs
11. DisposableBean.destroy() (if implemented)
12. Custom destroy-method

PRACTICAL ANSWER: For most code, just know @PostConstruct and @PreDestroy
USE CASE: @PostConstruct for initializing a cache, opening a connection pool
         @PreDestroy for cleanup, closing resources

GOTCHA: Prototype beans — Spring doesn't call @PreDestroy on them!
```

---

**Q4. What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?**

```
SCRATCHPAD NOTES:
- All 4 are stereotypes — all ultimately annotated with @Component
- All are picked up by component scanning
- Functional differences:
    * @Repository: enables PersistenceExceptionTranslation (converts DB-specific 
      exceptions to Spring DataAccessException hierarchy)
    * @Service: no extra behavior, but semantic marker for business logic
    * @Controller/@RestController: enables request mapping, MVC processing
    * @Component: generic, catch-all

INTERVIEWER FOLLOW-UP: "So @Service and @Component are identical?"
ANSWER: Functionally yes, but semantically no — @Service tells the reader/team 
        this is a service layer class. Spring itself may add behavior in future versions.
        Also useful for AOP pointcut targeting by annotation type.

KEY POINT: @Repository's exception translation is the only real technical difference
```

---

**Q5. How does Spring Boot auto-configuration work?**

```
SCRATCHPAD NOTES:
- @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
- @EnableAutoConfiguration triggers the magic
- Spring Boot looks at:
    * Spring Boot 2.x: META-INF/spring.factories (key: EnableAutoConfiguration)
    * Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
- Each auto-config class is annotated with @Conditional variants:
    * @ConditionalOnClass — only configure if certain class is on classpath
    * @ConditionalOnMissingBean — only configure if user hasn't defined their own
    * @ConditionalOnProperty — only configure if property is set
- Example: You add spring-boot-starter-data-jpa → JPA classes on classpath →
  JpaAutoConfiguration triggers → EntityManagerFactory, DataSource auto-created

REAL EXAMPLE TO WALK THROUGH:
  "Add H2 dependency → DataSourceAutoConfiguration sees it → creates in-memory 
   DataSource automatically. You provide your own DataSource bean → 
   @ConditionalOnMissingBean ensures auto-config backs off."

GOTCHA: You can debug by running with --debug flag → prints auto-configuration report
        Positive matches / Negative matches / Exclusions
```

---

**Q6. What is the difference between `@Configuration` and `@Component`? What is the `proxyBeanMethods` attribute?**

```
SCRATCHPAD NOTES:
- @Component: simple class, scanned and registered as bean
- @Configuration: specialized @Component, indicates it's a config class
- KEY DIFFERENCE: @Configuration classes are CGLIB-proxied by default
  - This means @Bean methods calling other @Bean methods go through the proxy
  - Ensures the same singleton bean is returned (not new instance each call)

EXAMPLE:
  @Configuration
  public class AppConfig {
      @Bean
      public ServiceA serviceA() { return new ServiceA(dataSource()); }
      @Bean
      public ServiceB serviceB() { return new ServiceB(dataSource()); }
      @Bean
      public DataSource dataSource() { return new DataSource(); }
  }
  // dataSource() is called twice but only ONE DataSource is created (proxy intercepts)

- proxyBeanMethods = false (Lite mode):
  * No CGLIB proxy created
  * Faster startup, less memory
  * BUT: @Bean methods calling other @Bean methods create new instances each time
  * Use when @Bean methods don't call each other

GOTCHA: @SpringBootApplication uses proxyBeanMethods=false internally for speed
```

---

**Q7. How does `@ConfigurationProperties` differ from `@Value`? When would you choose each?**

```
SCRATCHPAD NOTES:
@Value:
  - Injects single property: @Value("${server.port}")
  - SpEL expressions supported: @Value("#{systemProperties['user.home']}")
  - Tightly coupled — property key is hardcoded in code
  - No type safety — just String, needs conversion
  - Good for: one-off property injection, SpEL use cases

@ConfigurationProperties:
  - Binds a whole prefix of properties to a POJO
  - Type-safe (int, boolean, List, Map all work)
  - IDE autocompletion with spring-configuration-metadata
  - Can validate with @Validated + JSR-380
  - Better for: groups of related config (database settings, feature flags, etc.)

EXAMPLE:
  @ConfigurationProperties(prefix = "app.mail")
  public class MailProperties {
      private String host;
      private int port;
      private boolean ssl;
      // getters/setters
  }
  // application.yml: app.mail.host=smtp.example.com

BEST PRACTICE: Prefer @ConfigurationProperties for anything beyond a single value
GOTCHA: Need @EnableConfigurationProperties or @ConfigurationPropertiesScan 
        (or @Component on the class)
```

---

**Q8. What are the different bean scopes in Spring and when would you use each?**

```
SCRATCHPAD NOTES:
Scopes:
- Singleton (default): ONE instance per ApplicationContext
    * Shared across all requests/threads
    * DANGER: mutable state in singleton = thread safety problem
- Prototype: NEW instance every time it's requested from the container
    * Use for stateful beans
    * Spring does NOT manage destruction (no @PreDestroy called)
- Request: new instance per HTTP request (web apps only)
- Session: new instance per HTTP session (web apps only)
- Application: one instance per ServletContext
- WebSocket: one instance per WebSocket session

PRACTICAL EXAMPLE: 
  "Shopping cart should be Session scope — each user has their own cart"
  "Service classes should be Singleton — they're stateless"

GOTCHA: Injecting prototype bean into singleton bean — you get same prototype instance!
        Solution: Use ApplicationContext.getBean(), @Lookup, ObjectProvider<T>, or scoped proxy

INTERVIEW TRAP: "Is Spring thread-safe by default?"
ANSWER: Singleton beans are shared, so if they have mutable instance variables, 
        they are NOT thread-safe. Design singleton beans to be stateless.
```

---

**Q9. Explain how `@Conditional` annotations work and give examples of creating conditional beans.**

```
SCRATCHPAD NOTES:
- @Conditional is the base annotation — takes a Condition class with matches() method
- Spring Boot provides many built-in variants:
    * @ConditionalOnClass(X.class) — if X is on classpath
    * @ConditionalOnMissingClass — opposite
    * @ConditionalOnBean — if a bean of that type exists
    * @ConditionalOnMissingBean — if no bean of that type exists
    * @ConditionalOnProperty(name="feature.enabled", havingValue="true")
    * @ConditionalOnExpression — SpEL expression
    * @ConditionalOnWebApplication
    * @ConditionalOnResource — if resource file exists

USE CASES:
  - Custom auto-configuration
  - Feature flags: @ConditionalOnProperty("feature.new-checkout.enabled")
  - Environment-specific beans: @ConditionalOnProperty("app.storage", havingValue="s3")

CUSTOM CONDITION EXAMPLE:
  class OnLinuxCondition implements Condition {
      public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata m) {
          return System.getProperty("os.name").contains("Linux");
      }
  }
  @Bean @Conditional(OnLinuxCondition.class)
  public FileWatcher linuxFileWatcher() { ... }
```

---

**Q10. What happens internally when a Spring Boot application starts up?**

```
SCRATCHPAD NOTES:
Walk through these steps:
1. main() calls SpringApplication.run(MyApp.class, args)
2. SpringApplication is created:
   - Detects application type (SERVLET, REACTIVE, NONE)
   - Loads SpringApplicationRunListeners from spring.factories
3. prepareEnvironment():
   - Loads application.properties / application.yml
   - Applies env variables, system properties
   - Fires EnvironmentPreparedEvent
4. Creates ApplicationContext (AnnotationConfigServletWebServerApplicationContext for web)
5. prepareContext():
   - Registers @SpringBootApplication class as bean definition
6. refreshContext():
   - Component scanning (finds @Component, @Service, etc.)
   - Processes @Configuration classes and @Bean methods
   - Auto-configuration kicks in (reads spring.factories)
   - Initializes all singleton beans (unless lazy)
   - Starts embedded Tomcat server
7. Fires ApplicationStartedEvent, then ApplicationReadyEvent
8. CommandLineRunner / ApplicationRunner beans execute

GOTCHA: If anything fails during refresh(), context is closed and app exits
KEY EVENTS: ApplicationStartingEvent → EnvironmentPreparedEvent → 
            ContextPreparedEvent → ContextLoadedEvent → 
            ApplicationStartedEvent → ApplicationReadyEvent
```

---
## 🌐 SECTION 2: REST API Design (Spring MVC)

---

**Q1. How does `DispatcherServlet` process an incoming HTTP request?**

```
SCRATCHPAD NOTES:
Step-by-step:
1. Request arrives at embedded Tomcat → passed to DispatcherServlet
2. DispatcherServlet consults HandlerMapping to find the right controller method
   (RequestMappingHandlerMapping is most common)
3. HandlerAdapter invokes the controller method
   (RequestMappingHandlerAdapter handles @RequestMapping methods)
4. Controller method runs, returns a value
5. If @RestController (or @ResponseBody): 
   - HttpMessageConverter converts return object to JSON/XML
6. If @Controller:
   - ViewResolver resolves view name to actual view (Thymeleaf, JSP, etc.)
7. Response written back to client

KEY COMPONENTS:
- HandlerInterceptor: pre/post processing around controller
- HttpMessageConverter: Jackson for JSON, JAXB for XML
- HandlerExceptionResolver: handles exceptions from controllers

GOTCHA: DispatcherServlet itself is a Servlet registered in web container.
        Spring Boot auto-registers it via DispatcherServletAutoConfiguration.
```

---

**Q2. What is the difference between `@RestController` and `@Controller`?**

```
SCRATCHPAD NOTES:
- @Controller: marks class as MVC controller, returns view names by default
- @RestController: @Controller + @ResponseBody on EVERY method
- @ResponseBody: tells Spring to serialize return value directly to HTTP response body
  (via HttpMessageConverter, usually Jackson → JSON)

WHEN TO USE:
  - @RestController: REST APIs (always returning data, never a view)
  - @Controller: traditional MVC apps returning HTML views (Thymeleaf, JSP)
  - Mix: @Controller with @ResponseBody on specific methods that return data

FOLLOW-UP: "How does Spring know to use Jackson?"
  - jackson-databind on classpath → MappingJackson2HttpMessageConverter auto-registered
  - Content-Type: application/json in request / Accept: application/json triggers it
```

---

**Q3. How do you handle exceptions globally in Spring Boot? What is `@ControllerAdvice`?**

```
SCRATCHPAD NOTES:
Options (in order of preference):
1. @ControllerAdvice + @ExceptionHandler (BEST for REST APIs)
2. @ResponseStatus on custom exception classes
3. HandlerExceptionResolver (lower level)

@ControllerAdvice:
  - Applied globally to all controllers (or scoped with basePackages, annotations)
  - @ExceptionHandler(ResourceNotFoundException.class) maps exception to handler method
  - Return ResponseEntity with proper HTTP status and error body

EXAMPLE STRUCTURE:
  @ControllerAdvice
  public class GlobalExceptionHandler {
      @ExceptionHandler(ResourceNotFoundException.class)
      public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
          return ResponseEntity.status(404).body(new ErrorResponse(ex.getMessage()));
      }
      @ExceptionHandler(MethodArgumentNotValidException.class)
      public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
          // extract field errors
      }
  }

Spring 6 / Spring Boot 3:
  - ProblemDetail (RFC 7807) built-in: type, title, status, detail, instance fields

GOTCHA: @ExceptionHandler only catches exceptions from controller layer by default.
        Exceptions in filters don't reach @ControllerAdvice — handle those separately.
```

---

**Q4. How does request validation work in Spring Boot? How do you create custom validators?**

```
SCRATCHPAD NOTES:
Setup:
  - spring-boot-starter-validation pulls in Hibernate Validator (JSR-380 impl)
  - Annotate DTO fields: @NotNull, @NotBlank, @Size, @Min, @Max, @Email, @Pattern
  - Add @Valid or @Validated to @RequestBody parameter in controller

@Valid vs @Validated:
  - @Valid: standard JSR-380, used in controllers/service methods
  - @Validated: Spring's version, adds group validation support

Error handling:
  - If validation fails → MethodArgumentNotValidException thrown
  - Catch in @ControllerAdvice → extract BindingResult errors → return 400

Custom Validator:
  1. Create annotation: @interface ValidAge { ... }
  2. Create validator class: implements ConstraintValidator<ValidAge, Integer>
  3. Override isValid() with custom logic

EXAMPLE:
  public class AgeValidator implements ConstraintValidator<ValidAge, Integer> {
      public boolean isValid(Integer age, ConstraintValidatorContext ctx) {
          return age != null && age >= 18 && age <= 120;
      }
  }

GOTCHA: @Validated needed on class (not method) for validating method parameters 
        in @Service classes (not just controllers)
```

---

**Q5. How does content negotiation work in Spring MVC?**

```
SCRATCHPAD NOTES:
Content negotiation = deciding what format to return (JSON, XML, etc.)

Spring determines format by (in order):
1. URL path extension (deprecated)
2. Request parameter (?format=json) — if configured
3. Accept header in request: Accept: application/json or Accept: application/xml

On the controller side:
  - produces = "application/json" → only respond with JSON
  - consumes = "application/xml" → only accept XML request bodies

If Accept: application/xml → Spring needs JAXB or Jackson-dataformat-xml on classpath
                           → @XmlRootElement on the DTO
                           → if converter not found → 406 Not Acceptable

PRACTICAL USE:
  @GetMapping(value="/user/{id}", produces = {APPLICATION_JSON_VALUE, APPLICATION_XML_VALUE})
  public User getUser(@PathVariable Long id) { ... }

FOLLOW-UP: "What's the difference between produces and consumes?"
  - produces: what this endpoint returns (relates to Accept header)
  - consumes: what this endpoint accepts as input (relates to Content-Type header)
```

---

**Q6. What is `ResponseEntity` and when should you use it?**

```
SCRATCHPAD NOTES:
- ResponseEntity<T> = full HTTP response: status code + headers + body
- Gives fine-grained control over the response

WHEN TO USE:
  - Need to return specific status codes (201 Created with Location header)
  - Need to return custom headers
  - Conditional responses (return 404 if not found, 200 if found)
  - Return 204 No Content (no body)

EXAMPLE:
  @PostMapping("/users")
  public ResponseEntity<User> createUser(@RequestBody @Valid UserDto dto) {
      User user = userService.create(dto);
      URI location = URI.create("/users/" + user.getId());
      return ResponseEntity.created(location).body(user);  // 201 + Location header
  }

  @GetMapping("/users/{id}")
  public ResponseEntity<User> getUser(@PathVariable Long id) {
      return userService.findById(id)
              .map(ResponseEntity::ok)
              .orElse(ResponseEntity.notFound().build());  // 404
  }

WHEN NOT TO USE:
  - If you always return 200 with body → just return T directly, simpler
  - If exception handling is in @ControllerAdvice → no need for ResponseEntity in every method
```

---

**Q7. What are `HandlerInterceptors` and `Filters`? What is the difference?**

```
SCRATCHPAD NOTES:
Filter (javax/jakarta.servlet.Filter):
  - Part of Servlet API, NOT Spring-specific
  - Runs BEFORE DispatcherServlet sees the request
  - Can intercept ALL requests (including static resources)
  - Used for: CORS, auth token extraction, request logging, compression

HandlerInterceptor (Spring MVC):
  - Spring-specific, works WITHIN DispatcherServlet
  - preHandle() — before controller method
  - postHandle() — after controller, before view rendering (has access to ModelAndView)
  - afterCompletion() — after full request completed (good for cleanup/logging)

KEY DIFFERENCE:
  - Filters operate at Servlet container level (no access to Spring context)
  - Interceptors have full access to Spring beans, handler method info
  - Filters run BEFORE interceptors

USE CASES:
  - Security (JWT extraction) → Filter (runs before everything)
  - Request logging with method name → Interceptor (knows the handler)
  - MDC context setup → Filter

Register Filter: @Bean FilterRegistrationBean or @Component + @Order
Register Interceptor: WebMvcConfigurer.addInterceptors()

GOTCHA: @ControllerAdvice does NOT catch exceptions from Filters
```

---

**Q8. How do you version a REST API? What are the approaches and trade-offs?**

```
SCRATCHPAD NOTES:
4 common approaches:

1. URI Path versioning: /api/v1/users, /api/v2/users
   ✅ Simple, visible, cacheable, easy to route
   ❌ "Versioning should be in metadata not the URL" (REST purists)

2. Request Parameter: /api/users?version=1
   ✅ Simple
   ❌ Pollutes query string, not RESTful

3. Header versioning: X-API-Version: 1
   ✅ Clean URLs
   ❌ Harder to test (need tools), not cache-friendly

4. Media type (Accept header): Accept: application/vnd.myapp.v1+json
   ✅ Most RESTful
   ❌ Complex, verbose, harder to use for consumers

SPRING BOOT IMPLEMENTATION:
  URI versioning is easiest → just different @RequestMapping paths
  Header versioning → @GetMapping(headers = "X-API-Version=1")

REAL ANSWER: "In practice, URI versioning is most widely used. 
             I'd choose it for its simplicity and tooling support."

GOTCHA: Avoid versioning every endpoint — use versioning when breaking changes occur
```

---
**Q9. How does CORS work in Spring Boot? How do you configure it?**

```
SCRATCHPAD NOTES:
CORS = Cross-Origin Resource Sharing
Problem: Browser blocks JS from calling API on different origin (domain/port/protocol)
Solution: Server tells browser it's OK via response headers

Headers involved:
  Access-Control-Allow-Origin: https://myfrontend.com
  Access-Control-Allow-Methods: GET, POST
  Access-Control-Allow-Headers: Content-Type, Authorization

Spring Boot options:

1. @CrossOrigin on controller/method (fine-grained):
   @CrossOrigin(origins = "http://localhost:3000")

2. Global config (WebMvcConfigurer):
   @Override
   public void addCorsMappings(CorsRegistry registry) {
       registry.addMapping("/api/**")
               .allowedOrigins("https://myfrontend.com")
               .allowedMethods("GET", "POST", "PUT", "DELETE")
               .allowedHeaders("*")
               .allowCredentials(true);
   }

3. CorsFilter bean (for use with Spring Security):
   Security filter chain runs BEFORE MVC CORS, so you need to configure 
   CORS in Spring Security too: http.cors(withDefaults())

GOTCHA: If using Spring Security, you MUST configure CORS there too.
        MVC CORS config alone won't work with Security.
        Preflight OPTIONS requests must be permitted without authentication.
```

---
**Q10. How do you implement pagination and sorting in a REST API?**

```
SCRATCHPAD NOTES:
Spring Data makes this easy:

Controller:
  @GetMapping("/users")
  public Page<UserDto> getUsers(Pageable pageable) {
      return userService.findAll(pageable);
  }

Client calls: GET /users?page=0&size=20&sort=lastName,asc

Pageable is auto-resolved by PageableHandlerMethodArgumentResolver

Page<T> response contains:
  - content: actual data
  - totalElements, totalPages, number, size, first, last, numberOfElements

Customization:
  @PageableDefault(size = 10, sort = "createdAt", direction = DESC)
  public Page<UserDto> getUsers(Pageable pageable)

DTO projection (don't expose entities):
  return users.map(userMapper::toDto);  // map Page<User> to Page<UserDto>

Sorting security concern:
  Spring Data allows sorting by ANY field by default
  Use PageableHandlerMethodArgumentResolver.setMaxPageSize() to limit page size
  Consider whitelisting allowed sort fields

GOTCHA: Returning Page<T> directly in response exposes Spring internals 
        Consider wrapping in a custom PageResponse<T> DTO
```

---
## 🗄️ SECTION 3: Spring Data JPA

---

**Q1. What is the difference between JPA, Hibernate, and Spring Data JPA?**

```
SCRATCHPAD NOTES:
- JPA (Jakarta Persistence API): specification/interface only — defines annotations, 
  EntityManager API. It's a standard, not an implementation.
- Hibernate: the most popular JPA implementation (also has its own native API)
- Spring Data JPA: abstraction ON TOP of JPA that reduces boilerplate
  * Provides repository interfaces (JpaRepository)
  * Generates query implementations at runtime
  * Still uses Hibernate (or another JPA provider) under the hood

LAYERED VIEW:
  Your Code → Spring Data JPA → JPA (EntityManager) → Hibernate → JDBC → Database

Spring Data JPA generates the actual SQL (via Hibernate) from method names like:
  findByLastNameAndFirstName(String lastName, String firstName)

KEY INSIGHT: "Spring Data JPA doesn't replace JPA or Hibernate. 
             It's a convenience layer. The heavy lifting is still done by Hibernate."

GOTCHA: Sometimes you need to go below Spring Data (use EntityManager directly)
        for complex queries. Spring Data doesn't hide that option.
```

---

**Q2. Explain `@Transactional` — how does it work internally, what are propagation types?**

```
SCRATCHPAD NOTES:
HOW IT WORKS:
- @Transactional uses AOP (Spring proxy)
- When you call a @Transactional method from OUTSIDE the bean, 
  the call goes through a proxy
- Proxy opens transaction → calls real method → commits or rolls back

PROPAGATION TYPES (most important ones):
- REQUIRED (default): join existing tx or create new one
- REQUIRES_NEW: always creates NEW tx, suspends existing one
  (use for: audit logging — must save even if main tx rolls back)
- SUPPORTS: join if exists, no tx if not
- NOT_SUPPORTED: run without tx, suspend if exists
- MANDATORY: must have existing tx, throw if none
- NEVER: must NOT have tx, throw if one exists
- NESTED: runs in nested tx within existing one (savepoints)

ISOLATION LEVELS:
- READ_UNCOMMITTED: dirty reads possible
- READ_COMMITTED: no dirty reads
- REPEATABLE_READ: no non-repeatable reads
- SERIALIZABLE: full isolation, slowest

ROLLBACK RULES:
- Default: rolls back on RuntimeException and Error
- Does NOT roll back on checked exceptions by default!
- Override: @Transactional(rollbackFor = Exception.class)

CRITICAL GOTCHA (self-invocation):
  public class UserService {
      public void outerMethod() {
          this.innerMethod(); // PROXY BYPASSED! No transaction!
      }
      @Transactional
      public void innerMethod() { ... }
  }
  // Fix: inject self, use AspectJ weaving, or restructure code
```

---
**Q3. What is the N+1 problem and how do you fix it?**

```
SCRATCHPAD NOTES:
PROBLEM:
  Fetch 1 list of Orders (1 query)
  For each Order, fetch its Items (N queries)
  = N+1 queries total → terrible performance

EXAMPLE:
  List<Order> orders = orderRepository.findAll(); // 1 query
  orders.forEach(o -> o.getItems().size());       // N queries (LAZY loading)

SOLUTIONS:

1. FETCH JOIN in JPQL (best for most cases):
   @Query("SELECT o FROM Order o JOIN FETCH o.items")

2. @EntityGraph (declarative, works with derived queries):
   @EntityGraph(attributePaths = {"items", "customer"})
   List<Order> findAll();

3. FetchType.EAGER (be careful!):
   @OneToMany(fetch = FetchType.EAGER)
   — always loads even when not needed → can cause more data than needed
   — anti-pattern for collections

4. Batch fetching (Hibernate):
   @BatchSize(size = 10) → fetches in batches instead of one-by-one

5. DTO projection with JPQL:
   @Query("SELECT new com.app.dto.OrderDto(o.id, o.total) FROM Order o")
   — fetch only needed columns, no lazy loading issues

DETECTION: Hibernate SQL logging, p6spy, Datasource-Proxy show individual queries

GOTCHA: FetchType.EAGER on @ManyToMany → Cartesian product problem
```

---

**Q4. What is the difference between `CrudRepository`, `JpaRepository`, and `PagingAndSortingRepository`?**

```
SCRATCHPAD NOTES:
Hierarchy:
  Repository (marker)
    └── CrudRepository<T, ID>         — basic CRUD: save, findById, findAll, delete
          └── PagingAndSortingRepository<T, ID>  — adds findAll(Pageable), findAll(Sort)
                └── JpaRepository<T, ID>          — adds flush(), saveAndFlush(),
                                                     deleteInBatch(), getOne()/getReferenceById()

JpaRepository adds:
  - saveAll() with flush
  - deleteAllInBatch() — single DELETE query (more efficient)
  - getReferenceById() — returns proxy without hitting DB (use when you have ID, need reference)
  - Supports JPA-specific features

WHICH TO USE:
  - JpaRepository in most cases (most features, JPA-specific)
  - CrudRepository if writing a generic library (no JPA dependency)
  - PagingAndSortingRepository rarely used directly

GOTCHA: findById() returns Optional<T> → always handle empty case
        getOne()/getById() returns proxy → throws EntityNotFoundException on access 
        if not found, not on call
```

---
**Q5. How do you write custom queries in Spring Data JPA?**

```
SCRATCHPAD NOTES:
Options (in order of complexity):

1. Derived query methods (automatic from method name):
   findByEmailAndStatus(String email, Status status)
   findByAgeBetween(int min, int max)
   findByNameContainingIgnoreCase(String name)
   countByStatus(Status status)
   existsByEmail(String email)
   
2. @Query with JPQL:
   @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
   Optional<User> findActiveByEmail(@Param("email") String email);

3. @Query with Native SQL:
   @Query(value = "SELECT * FROM users WHERE ...", nativeQuery = true)

4. @NamedQuery (on entity class):
   @NamedQuery(name = "User.findByEmail", query = "SELECT u FROM User u WHERE ...")

5. Specifications (JPA Criteria API wrapper):
   Predicate-based dynamic queries
   userRepository.findAll(Specification.where(hasEmail(email)).and(isActive()))

6. QueryDSL (type-safe queries):
   QUser user = QUser.user;
   repository.findAll(user.age.gt(18).and(user.active.isTrue()));

7. Custom repository implementation:
   Create UserRepositoryCustom interface + UserRepositoryImpl class
   UserRepository extends JpaRepository + UserRepositoryCustom

WHEN TO USE WHAT:
  Simple → derived methods
  Medium complexity → @Query JPQL
  Dynamic filters → Specifications or QueryDSL
  Complex/optimized → native query
```

---
**Q6. What are the different fetch types and strategies in JPA? What is LAZY vs EAGER loading?**

```
SCRATCHPAD NOTES:
FetchType.LAZY:
  - Data loaded only when accessed
  - Default for @OneToMany, @ManyToMany
  - Problem: LazyInitializationException if accessed outside transaction (session closed)

FetchType.EAGER:
  - Data loaded immediately with parent
  - Default for @ManyToOne, @OneToOne
  - Problem: Loads data even when not needed → performance issues

LazyInitializationException:
  - Common cause: entity fetched in service layer, session closed, 
    then view/controller tries to access lazy collection
  - Solutions:
    * Fetch what you need inside transaction (JOIN FETCH)
    * Open Session in View pattern (bad practice — avoid in production)
    * Use DTO projections (load only what you need)
    * @Transactional on service method that accesses the lazy property

DEFAULT FETCH TYPES:
  @OneToOne    → EAGER
  @ManyToOne   → EAGER
  @OneToMany   → LAZY
  @ManyToMany  → LAZY

BEST PRACTICE: 
  - Use LAZY for all associations
  - Fetch eagerly only when needed using JOIN FETCH or @EntityGraph
  - Never use EAGER on collections
```

---
**Q7. How do you implement database migrations with Flyway or Liquibase?**

```
SCRATCHPAD NOTES:
WHY MIGRATIONS:
  - JPA ddl-auto=update is NOT suitable for production (dangerous, can lose data)
  - Need version-controlled, repeatable schema changes

FLYWAY:
  - SQL-based migration files
  - Naming: V1__create_users.sql, V2__add_email_column.sql
  - Stored in: src/main/resources/db/migration
  - Spring Boot auto-configures if flyway on classpath
  - Tracks applied migrations in flyway_schema_history table
  - Only runs new scripts (checks checksum)

LIQUIBASE:
  - More powerful: YAML/XML/JSON/SQL changesets
  - Supports rollbacks
  - Database-agnostic (same changelog for MySQL, Postgres, etc.)
  - More complex configuration

FLYWAY vs LIQUIBASE:
  - Flyway: simpler, SQL feels natural to DBAs, less overhead
  - Liquibase: rollback support, multi-DB support, more features
  - Both are valid for production

FLYWAY SETUP:
  spring.flyway.enabled=true
  spring.flyway.locations=classpath:db/migration
  spring.jpa.hibernate.ddl-auto=validate (not update/create!)

GOTCHA: Never modify an existing migration file (Flyway checks checksums and will fail)
        Only add new migration files
```

---
**Q8. What are DTO projections and why are they important?**

```
SCRATCHPAD NOTES:
PROBLEM: Returning @Entity objects directly from repository/controller:
  - Exposes DB schema in API
  - Lazy loading exceptions
  - Bidirectional relationships → JSON serialization infinite loops
  - Over-fetching (loading 20 columns when you need 3)
  - Security risk (exposing sensitive fields like password)

PROJECTION TYPES:

1. Interface-based (Closed Projection):
   interface UserSummary {
       String getFirstName();
       String getEmail();
   }
   List<UserSummary> findByStatus(Status status);
   → Spring Data creates proxy that reads only those fields

2. Interface-based (Open Projection with SpEL):
   @Value("#{target.firstName + ' ' + target.lastName}")
   String getFullName();
   → computed value, but loads full entity

3. Class-based (DTO Projection):
   record UserSummary(String firstName, String email) {}
   @Query("SELECT new com.app.UserSummary(u.firstName, u.email) FROM User u")
   → most efficient, actual SQL only selects needed columns

4. Dynamic Projections:
   <T> List<T> findByStatus(Status status, Class<T> type);
   
BEST PRACTICE: 
  - Never return @Entity from @RestController
  - Use DTO/record classes at API boundary
  - Use interface projections for read-only queries in repository
  - MapStruct or ModelMapper for entity-to-DTO conversion
```

---
**Q9. How does HikariCP work and how would you tune a connection pool?**

```
SCRATCHPAD NOTES:
HikariCP:
  - Default connection pool in Spring Boot 2+
  - Known for: fastest, lightweight, reliable
  - Manages a pool of reusable DB connections

KEY CONFIG PROPERTIES:
  spring.datasource.hikari.maximum-pool-size=10   (max connections)
  spring.datasource.hikari.minimum-idle=5          (min idle connections)
  spring.datasource.hikari.connection-timeout=30000 (ms to wait for conn)
  spring.datasource.hikari.idle-timeout=600000     (ms before idle conn removed)
  spring.datasource.hikari.max-lifetime=1800000    (ms max conn lifetime)
  spring.datasource.hikari.pool-name=MyPool

TUNING GUIDELINES:
  - Max pool size formula: (Core Count * 2) + Number of Disk Spindles
  - Typical starting point: 10 connections
  - Too many connections → DB server overwhelmed, context switching
  - Too few → requests queue up, timeouts
  - Monitor: Actuator /metrics → hikaricp.connections.* metrics

COMMON ISSUES:
  - Connection leaks: transaction never closed → pool exhausted
    (Detect with: spring.datasource.hikari.leak-detection-threshold)
  - Long-running transactions hold connections
  - DB behind firewall kills idle connections 
    → fix with: keepaliveTime, connectionTestQuery

GOTCHA: Don't set pool size too high thinking it'll always be faster
        DB itself has a max connection limit (PostgreSQL default ~100)
```

---
**Q10. What is the difference between `save()`, `saveAndFlush()`, and `saveAll()` in JpaRepository?**

```
SCRATCHPAD NOTES:
- save(entity): 
  * If entity is new (no ID) → persist (INSERT)
  * If entity has ID → merge (UPDATE)
  * Changes staged in persistence context (1st level cache)
  * Actual SQL executed at end of transaction (or when flush occurs)

- saveAndFlush(entity):
  * Same as save() + immediately flushes to database
  * Useful when: you need the DB side effects immediately (triggers, sequences)
  * Use case: save entity, read back computed columns, test with @DataJpaTest

- saveAll(entities):
  * Saves a collection, calls save() for each
  * NOT a single bulk INSERT by default (still N inserts)
  * For true bulk insert: use @Modifying @Query or JDBC batch insert

- flush():
  * Writes all pending changes to DB within current transaction
  * Does NOT commit the transaction

WHAT "NEW" MEANS:
  Spring Data detects new entities by:
  1. Entity has @Version field that is null
  2. ID is null (for non-primitive IDs)
  3. Implement Persistable<ID> and override isNew()

GOTCHA: save() on a managed entity (already in persistence context) is often 
        unnecessary — dirty checking handles it. But it's harmless.
```

---
## 🔐 SECTION 4: Spring Security / JWT

---

**Q1. How does the Spring Security filter chain work?**

```
SCRATCHPAD NOTES:
- Spring Security is a chain of servlet filters
- Every request passes through this chain before reaching your controller

KEY FILTERS (in order):
1. SecurityContextPersistenceFilter — loads/saves SecurityContext
2. UsernamePasswordAuthenticationFilter — handles form login
3. BasicAuthenticationFilter — handles HTTP Basic auth
4. BearerTokenAuthenticationFilter — handles JWT (Resource Server)
5. ExceptionTranslationFilter — handles AuthenticationException, AccessDeniedException
6. AuthorizationFilter — final access decision

Spring Security 6 (Spring Boot 3):
  - Configures via SecurityFilterChain @Bean (not WebSecurityConfigurerAdapter — removed)
  
  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
      return http
          .csrf(AbstractHttpConfigurer::disable)  // for stateless APIs
          .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/api/auth/**").permitAll()
              .anyRequest().authenticated())
          .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
          .build();
  }

GOTCHA: Multiple SecurityFilterChain beans possible (for different URL patterns)
        Use @Order to control which chain matches first
```

---

**Q2. Walk me through implementing JWT authentication in Spring Boot.**

```
SCRATCHPAD NOTES:
FLOW:
1. Client sends POST /api/auth/login with username/password
2. Server validates credentials (UserDetailsService.loadUserByUsername)
3. Server generates JWT token (signed with secret key)
4. Client stores token (localStorage/cookie)
5. Client sends subsequent requests with: Authorization: Bearer <token>
6. Server validates token on every request via a filter

COMPONENTS NEEDED:
1. JwtUtil class:
   - generateToken(UserDetails user)
   - validateToken(String token, UserDetails user)
   - extractUsername(String token)
   - Uses: io.jsonwebtoken (jjwt) library

2. JwtAuthenticationFilter extends OncePerRequestFilter:
   - Extract token from Authorization header
   - Validate token
   - Load UserDetails
   - Set Authentication in SecurityContextHolder

3. AuthController:
   - POST /api/auth/login → return JwtResponse(token)
   
4. UserDetailsService implementation:
   - loadUserByUsername → query DB for user

5. SecurityFilterChain:
   - sessionManagement = STATELESS
   - addFilterBefore(jwtFilter, ...)

JWT STRUCTURE: header.payload.signature (Base64 encoded)
  Payload contains: sub (subject), iat (issued at), exp (expiration), roles

SECURITY CONSIDERATIONS:
  - Use strong secret (256-bit minimum for HS256)
  - Short expiration for access tokens (15 min)
  - Refresh tokens for long-lived sessions
  - Store refresh tokens in DB to enable revocation
  - Use HTTPS always

GOTCHA: JWTs can't be invalidated (stateless) without a token blacklist or short expiry
```

---

**Q3. What is the difference between authentication and authorization in Spring Security?**

```
SCRATCHPAD NOTES:
Authentication = "Who are you?" 
  - Verifying identity (username/password, token, certificate)
  - Result: Authentication object in SecurityContext
  - Handled by: AuthenticationManager → AuthenticationProvider → UserDetailsService

Authorization = "What are you allowed to do?"
  - Verifying permissions/roles after identity confirmed
  - Handled by: AccessDecisionManager (old) / AuthorizationManager (new)
  - Based on: roles (ROLE_ADMIN), permissions (user:write)

SPRING SECURITY HIERARCHY:
  AuthenticationManager
    └── ProviderManager
          └── DaoAuthenticationProvider (most common)
                └── UserDetailsService.loadUserByUsername()
                └── PasswordEncoder.matches()

POST-AUTH:
  SecurityContextHolder holds Authentication object
  Authentication.getPrincipal() → UserDetails
  Authentication.getAuthorities() → roles/permissions

SPRING SECURITY EXCEPTIONS:
  AuthenticationException → 401 Unauthorized (not logged in)
  AccessDeniedException → 403 Forbidden (logged in but no permission)

GOTCHA: Spring Security requires ROLE_ prefix for hasRole() but NOT for hasAuthority()
  hasRole("ADMIN") checks for "ROLE_ADMIN" in authorities
  hasAuthority("ROLE_ADMIN") checks for exact match
```

---

**Q4. How do you implement method-level security?**

```
SCRATCHPAD NOTES:
Enable with: @EnableMethodSecurity (Spring Security 6) 
             was: @EnableGlobalMethodSecurity (deprecated in Spring Security 5.6+)

ANNOTATIONS:
@PreAuthorize — checked BEFORE method executes:
  @PreAuthorize("hasRole('ADMIN')")
  @PreAuthorize("hasAuthority('user:write')")
  @PreAuthorize("#userId == authentication.principal.id")  // user can only access own data
  @PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")

@PostAuthorize — checked AFTER method executes (can access return value):
  @PostAuthorize("returnObject.owner == authentication.name")

@PreFilter / @PostFilter — filter collections:
  @PostFilter("filterObject.owner == authentication.name")
  List<Document> getUserDocuments();

@Secured (simpler, less SpEL support):
  @Secured("ROLE_ADMIN")

@RolesAllowed (JSR-250 standard):
  @RolesAllowed("ADMIN")

WHEN TO USE:
  - Fine-grained control within service layer
  - Row-level security (user can only see their own data)
  - Don't want to rely only on URL-level security

GOTCHA: Method security uses AOP proxy — same self-invocation problem as @Transactional
        Must call from outside the bean for security to be enforced
```

---

**Q5. How do you configure CORS in Spring Security?**

```
SCRATCHPAD NOTES:
Problem: Spring Security CORS must be configured SEPARATELY from MVC CORS
         Because Security filters run BEFORE DispatcherServlet

SOLUTION — Configure CORS in Security config:

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
    // rest of config...
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myfrontend.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

ALSO CRITICAL: Preflight OPTIONS requests must be permitted:
  .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()

GOTCHA: http.cors(withDefaults()) uses CorsConfigurationSource bean if defined
        Don't set allowedOrigins("*") with allowCredentials(true) — browsers reject this
```

---

**Q6. What is CSRF and when should you disable it?**

```
SCRATCHPAD NOTES:
CSRF (Cross-Site Request Forgery):
  - Attacker tricks authenticated user's browser into making unwanted request
  - Works because browser automatically sends cookies with requests
  - Example: Evil website sends POST request to your bank while user is logged in

Spring Security CSRF Protection:
  - Generates unique CSRF token per session
  - Requires token be included in state-changing requests (POST, PUT, DELETE)
  - Token is NOT automatically sent by browser → CSRF attack fails

WHEN TO DISABLE CSRF:
  - Stateless REST APIs using JWT (tokens are NOT stored in cookies)
    → Browser can't automatically send JWT → CSRF attack not possible
  - Non-browser clients (mobile apps, server-to-server)
  
  http.csrf(AbstractHttpConfigurer::disable)  // for stateless JWT APIs

WHEN TO KEEP CSRF ENABLED:
  - Session-based authentication (cookies)
  - Server-rendered forms (Thymeleaf)
  - Any app where browsers store auth in cookies

REAL-WORLD ANSWER: 
  "In my current project we use JWT in Authorization header — no cookies — 
   so CSRF is disabled. If we used cookie-based sessions, we'd keep CSRF protection."

GOTCHA: Even with CSRF disabled, use SameSite=Strict/Lax cookies if using cookies
```

---

**Q7. How do you handle OAuth2 in Spring Boot?**

```
SCRATCHPAD NOTES>
OAuth2 ROLES:
  - Resource Owner: the user
  - Client: your application
  - Authorization Server: Google, Okta, Keycloak (issues tokens)
  - Resource Server: your API (validates tokens)

SPRING BOOT AS OAUTH2 CLIENT (login with Google):
  Dependency: spring-boot-starter-oauth2-client
  Config:
    spring.security.oauth2.client.registration.google.client-id=...
    spring.security.oauth2.client.registration.google.client-secret=...
  http.oauth2Login(withDefaults())  // adds OAuth2 login flow

SPRING BOOT AS RESOURCE SERVER (validate JWT from auth server):
  Dependency: spring-boot-starter-oauth2-resource-server
  Config:
    spring.security.oauth2.resourceserver.jwt.issuer-uri=https://auth.example.com
  http.oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()))
  
  Spring auto-fetches JWKS from issuer to validate tokens

SPRING AUTHORIZATION SERVER (your own auth server):
  Separate project: spring-authorization-server
  Issues access tokens, refresh tokens, handles flows

FLOWS:
  - Authorization Code: web apps (most common)
  - Client Credentials: server-to-server (no user involved)
  - Implicit: deprecated
  - PKCE: for SPAs/mobile (no client secret)

GOTCHA: Understand which role your service plays — client, resource server, or both
```

---

**Q8. How does password encoding work in Spring Security?**

```
SCRATCHPAD NOTES:
PasswordEncoder interface:
  - encode(rawPassword): hash the password for storage
  - matches(rawPassword, encodedPassword): verify at login

DEFAULT CHOICE: BCryptPasswordEncoder
  - Adaptive hash function (work factor can increase over time)
  - Built-in salt (random, stored in hash)
  - Slow by design (makes brute force harder)
  - Standard strength: 10 rounds

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

STORING PASSWORDS:
  // At registration:
  user.setPassword(passwordEncoder.encode(rawPassword));
  userRepository.save(user);
  
  // At login (Spring does this automatically if using DaoAuthenticationProvider)
  passwordEncoder.matches(rawInput, storedHash);

DelegatingPasswordEncoder (modern approach):
  - Stores passwords with prefix: {bcrypt}$2a$10$...
  - Allows multiple encoding strategies in same app
  - Can migrate from old algorithm to new
  - PasswordEncoderFactories.createDelegatingPasswordEncoder()

WHAT NOT TO DO:
  ❌ MD5, SHA-1, SHA-256 (fast hashes — crackable)
  ❌ Storing plain text
  ❌ Same salt for all users
  ❌ Writing your own crypto

GOTCHA: Never log passwords, never return them in API responses
```

---

**Q9. What is the difference between `@PreAuthorize` and URL-based security? Which is better?**

```
SCRATCHPAD NOTES:
URL-BASED SECURITY (in SecurityFilterChain):
  .authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/admin/**").hasRole("ADMIN")
      .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN"))
  
  Pros: Centralized, visible, applies before controller even runs
  Cons: URL matching can be tricky, may miss endpoints added later,
        can't express data-level access (only URL patterns)

@PreAuthorize (method-level):
  @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
  
  Pros: Close to the code it protects, can use SpEL with parameters,
        supports row-level security, works on service methods too
  Cons: Security logic scattered across codebase, easy to forget on new methods

BEST PRACTICE — Use BOTH:
  - URL security for broad rules (authenticated vs. not, admin vs. user)
  - @PreAuthorize for fine-grained, data-level access control

REAL EXAMPLE:
  URL security: /api/admin/** requires ADMIN
  @PreAuthorize: @GetMapping("/api/users/{id}") has @PreAuthorize("#id == principal.id")
                 so users can only see their own profile, even if URL is "permitted"

GOTCHA: Test both layers — easy to configure URL security and forget @PreAuthorize,
        or vice versa. Spring Security Test (@WithMockUser) helps here.
```

---

**Q10. How do you secure sensitive configuration values (API keys, passwords) in Spring Boot?**

```
SCRATCHPAD NOTES:
LAYERS OF PROTECTION:

1. Never commit to source control:
   - .gitignore your application-prod.yml
   - Use placeholder in code: ${DB_PASSWORD}

2. Environment Variables (for containers/cloud):
   spring.datasource.password=${DB_PASSWORD}
   → Set at runtime, not in code

3. Spring Cloud Config Server:
   - Centralized config, encrypted values
   - spring.cloud.config.server.encrypt.key=...
   - {cipher}encrypted_value in config files

4. Vault (HashiCorp):
   spring-cloud-starter-vault-config
   → Secrets stored securely, dynamic credentials, auto-rotation

5. AWS Secrets Manager / Azure Key Vault / GCP Secret Manager:
   - Cloud-native secret management
   - Spring Cloud AWS integration

6. Kubernetes Secrets:
   - Store as K8s Secret, inject as env var or mounted file
   - Note: K8s Secrets are base64 encoded, not encrypted by default
           Use sealed secrets or Vault for true encryption

7. Jasypt (for encrypted properties files):
   ENC(encryptedValue) in properties

NEVER:
  ❌ Hardcode secrets in code
  ❌ Commit secrets to Git (even private repos)
  ❌ Log secrets or return in API responses
  ❌ Put secrets in Docker image layers

REAL ANSWER: "In production, we inject sensitive values via environment variables 
             set by our CI/CD system or via AWS Secrets Manager integration."
```

---

## 🧪 SECTION 5: Testing

---

**Q1. Explain the testing pyramid and how it applies to Spring Boot.**

```
SCRATCHPAD NOTES:
PYRAMID (bottom to top):
  Unit Tests (many, fast, cheap)
    ↑
  Integration Tests (medium amount, slower)
    ↑
  E2E / UI Tests (few, slow, expensive)

In Spring Boot context:

UNIT TESTS:
  - Test single class in isolation
  - Mock all dependencies (Mockito)
  - No Spring context loaded
  - Run in milliseconds
  - Example: UserService with mocked UserRepository

INTEGRATION TESTS:
  - Test multiple components together
  - @SpringBootTest or slice tests
  - Real or in-memory database
  - Test: Controller → Service → Repository → DB
  - Slower (seconds)
  
SLICE TESTS (Spring Boot's middle ground):
  - @WebMvcTest: only web layer (Controller + MVC config)
  - @DataJpaTest: only JPA layer (repos, JPA, H2)
  - @JsonTest: only JSON serialization
  - Faster than full @SpringBootTest, more focused

E2E:
  - Running app + real DB (TestContainers)
  - REST Assured or WebTestClient for HTTP calls
  - Test full flow: HTTP request → response

STRATEGY:
  "Write many unit tests for business logic.
   Slice tests for controller and JPA layers.
   A few integration tests for critical happy paths.
   E2E for smoke testing."
```

---

**Q2. What is the difference between `@Mock` and `@MockBean`?**

```
SCRATCHPAD NOTES:
@Mock (Mockito):
  - Creates a pure Mockito mock
  - No Spring context involved
  - Used in unit tests
  - Works with @ExtendWith(MockitoExtension.class)
  - Fast — no Spring overhead

@MockBean (Spring Boot Test):
  - Creates a Mockito mock AND registers it as a Spring bean
  - Replaces the real bean in the Spring ApplicationContext
  - Used when you need a Spring context but want to mock certain beans
  - Works with @SpringBootTest, @WebMvcTest
  - Context is reloaded if @MockBean setup changes between tests (can be slow)

EXAMPLE:
  // Unit test — no Spring context
  @ExtendWith(MockitoExtension.class)
  class UserServiceTest {
      @Mock UserRepository userRepository;
      @InjectMocks UserService userService;
  }

  // Spring slice test — Spring context needed
  @WebMvcTest(UserController.class)
  class UserControllerTest {
      @Autowired MockMvc mockMvc;
      @MockBean UserService userService;  // must use @MockBean here
  }

GOTCHA: Using @MockBean in @SpringBootTest causes context reload → use sparingly
        For service layer tests, prefer @Mock (no Spring context needed)
        @SpyBean wraps real bean and allows partial mocking
```

---

**Q3. How do you test a REST controller with `@WebMvcTest` and `MockMvc`?**

```
SCRATCHPAD NOTES:
@WebMvcTest:
  - Loads ONLY the web layer: Controllers, ControllerAdvice, Filters, WebMvcConfigurer
  - Does NOT load: Services, Repositories, full application context
  - Services/Repos used by controller must be @MockBean

BASIC STRUCTURE:
  @WebMvcTest(UserController.class)
  class UserControllerTest {
      @Autowired MockMvc mockMvc;
      @MockBean UserService userService;
      @Autowired ObjectMapper objectMapper;
      
      @Test
      void createUser_shouldReturn201() throws Exception {
          CreateUserDto dto = new CreateUserDto("John", "john@example.com");
          UserDto response = new UserDto(1L, "John", "john@example.com");
          given(userService.create(any())).willReturn(response);
          
          mockMvc.perform(post("/api/users")
              .contentType(APPLICATION_JSON)
              .content(objectMapper.writeValueAsString(dto)))
              .andExpect(status().isCreated())
              .andExpect(jsonPath("$.id").value(1L))
              .andExpect(jsonPath("$.email").value("john@example.com"));
      }
  }

USEFUL ASSERTIONS:
  .andExpect(status().isOk())
  .andExpect(content().contentType(APPLICATION_JSON))
  .andExpect(jsonPath("$.name").value("John"))
  .andExpect(jsonPath("$.items").isArray())
  .andExpect(jsonPath("$.items", hasSize(3)))

SECURITY IN TESTS:
  @WithMockUser(roles = "ADMIN")  — simulate authenticated user
  @WithAnonymousUser — simulate unauthenticated

GOTCHA: @WebMvcTest loads Security config by default. 
        Exclude with: @WebMvcTest(excludeAutoConfiguration = SecurityAutoConfiguration.class)
        or configure mock security in the test
```

---

**Q4. How do you write JPA tests with `@DataJpaTest`?**

```
SCRATCHPAD NOTES:
@DataJpaTest:
  - Loads only JPA layer: entities, repositories, JPA config
  - Does NOT load: controllers, services, full context
  - Configures in-memory H2 database by default
  - @Transactional by default (each test rolled back after)

BASIC STRUCTURE:
  @DataJpaTest
  class UserRepositoryTest {
      @Autowired UserRepository userRepository;
      @Autowired TestEntityManager entityManager;
      
      @Test
      void findByEmail_shouldReturnUser() {
          User user = new User("John", "john@example.com");
          entityManager.persistAndFlush(user);
          
          Optional<User> found = userRepository.findByEmail("john@example.com");
          assertThat(found).isPresent();
          assertThat(found.get().getName()).isEqualTo("John");
      }
  }

TestEntityManager:
  - Wrapper around EntityManager
  - persistAndFlush() — saves and immediately flushes
  - find() — find by ID
  - clear() — clear persistence context (to force DB hit on next find)

USING REAL DATABASE (TestContainers):
  @DataJpaTest
  @AutoConfigureTestDatabase(replace = NONE)  // don't replace with H2
  @Testcontainers
  class UserRepositoryTest {
      @Container
      static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
      
      @DynamicPropertySource
      static void configureProperties(DynamicPropertyRegistry registry) {
          registry.add("spring.datasource.url", postgres::getJdbcUrl);
      }
  }

GOTCHA: If using Flyway/Liquibase, H2 might not support your SQL dialect
        → Use @AutoConfigureTestDatabase(replace = NONE) + TestContainers
```

---

**Q5. What is TestContainers and when would you use it?**

```
SCRATCHPAD NOTES:
TestContainers:
  - Java library that starts real Docker containers for tests
  - Provides real PostgreSQL, MySQL, Redis, Kafka, etc. for integration tests
  - Containers start/stop automatically per test class

WHY USE IT:
  - H2 in-memory DB doesn't support all SQL features (Postgres-specific functions, etc.)
  - Tests should run against the same DB as production
  - Kafka, Redis, MongoDB tests — no good in-memory alternative

SETUP:
  @SpringBootTest
  @Testcontainers
  class OrderServiceIntegrationTest {
      @Container
      static PostgreSQLContainer<?> postgres = 
          new PostgreSQLContainer<>("postgres:15-alpine")
              .withDatabaseName("testdb");
      
      @DynamicPropertySource
      static void overrideProps(DynamicPropertyRegistry registry) {
          registry.add("spring.datasource.url", postgres::getJdbcUrl);
          registry.add("spring.datasource.username", postgres::getUsername);
          registry.add("spring.datasource.password", postgres::getPassword);
      }
  }

CONTAINER OPTIONS:
  new PostgreSQLContainer<>("postgres:15")
  new MySQLContainer<>("mysql:8")
  new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.0.0"))
  new GenericContainer<>("redis:7").withExposedPorts(6379)

PERFORMANCE TIP:
  Use static containers — reuse across tests in same class
  Use Singleton pattern for shared containers across multiple test classes
  Enable Ryuk (auto-cleanup) vs manually stopping

GOTCHA: Requires Docker running locally. Can slow down CI — use parallel test execution
```

---

**Q6. How do you test a service that calls an external HTTP API?**

```
SCRATCHPAD NOTES:
OPTIONS:

1. MockRestServiceServer (for RestTemplate):
   @Autowired RestTemplate restTemplate;
   MockRestServiceServer mockServer = MockRestServiceServer.createServer(restTemplate);
   
   mockServer.expect(requestTo("https://api.example.com/users/1"))
             .andExpect(method(HttpMethod.GET))
             .andRespond(withSuccess(jsonBody, APPLICATION_JSON));

2. WireMock (most powerful, works with any HTTP client):
   @SpringBootTest(webEnvironment = RANDOM_PORT)
   @AutoConfigureWireMock(port = 0)  // random port
   class PaymentServiceTest {
       @Value("${payment.api.url}")
       String paymentApiUrl;
       
       @Test
       void charge_shouldReturnSuccess() {
           stubFor(post(urlEqualTo("/charge"))
               .willReturn(okJson("{\"status\": \"success\"}")));
           // test...
       }
   }

3. Mockito (mock the HTTP client bean):
   @Mock PaymentApiClient paymentClient;  // mock your wrapper/FeignClient
   when(paymentClient.charge(any())).thenReturn(new ChargeResponse("success"));

4. WebClientTest (Spring WebFlux WebClient):
   @ExtendWith(SpringExtension.class)
   // Use MockWebServer from OkHttp

BEST PRACTICES:
  - Don't test against real external APIs in unit/integration tests
  - WireMock for HTTP-level testing of your client code
  - Mockito mock for testing business logic that uses the client
  - Contract testing (Pact) for producer-consumer API contracts
```

---

**Q7. What is the difference between `@SpringBootTest` slice tests and full integration tests?**

```
SCRATCHPAD NOTES:
@SpringBootTest (Full Context):
  - Loads ENTIRE application context
  - All beans created (unless @MockBean used)
  - Very slow (5-30+ seconds depending on app size)
  - Use for: true end-to-end integration tests, when you need full wiring

  webEnvironment options:
    MOCK (default): mock servlet environment, use MockMvc
    RANDOM_PORT: starts real server on random port, use TestRestTemplate/WebTestClient
    DEFINED_PORT: uses server.port property
    NONE: no servlet environment

SLICE TESTS (partial context):
  @WebMvcTest: web layer only (Controller + MVC infra)
  @DataJpaTest: JPA layer only (repos + entities)
  @DataRedisTest: Redis layer
  @JsonTest: JSON serialization/deserialization
  @RestClientTest: REST client (RestTemplate/WebClient)
  @WebFluxTest: reactive web layer
  
  Faster because only relevant beans loaded

STRATEGY:
  Use slice tests for most integration tests (faster, focused)
  Use @SpringBootTest(RANDOM_PORT) for smoke tests / critical paths

CONTEXT CACHING:
  Spring caches ApplicationContext across tests with same configuration
  Different @MockBean, different properties → new context created
  
GOTCHA: Mixing @MockBean differently across tests destroys cache benefit
        Centralize @MockBean into a base test class for reuse
```

---

**Q8. How do you test `@Async` methods and scheduled tasks?**

```
SCRATCHPAD NOTES:
TESTING @Async:

Problem: @Async runs in different thread — test may complete before async task finishes

Solution 1: Override async config in test to run synchronously:
  @TestConfiguration
  public class SyncAsyncConfig implements AsyncConfigurer {
      @Override
      public Executor getAsyncExecutor() {
          return new SyncTaskExecutor();  // runs in same thread
      }
  }

Solution 2: Await with Awaitility:
  @Test
  void shouldProcessAsync() {
      service.doAsyncWork();
      await().atMost(5, SECONDS)
             .until(() -> resultRepository.count() == 1);
  }

Solution 3: Use CompletableFuture.get():
  @Test
  void shouldComplete() throws Exception {
      CompletableFuture<String> result = service.asyncMethod();
      String value = result.get(5, TimeUnit.SECONDS);
      assertThat(value).isEqualTo("done");
  }

TESTING @Scheduled:

Option 1: Call the method directly (it's just a regular method):
  @Test
  void scheduledMethodShouldRun() {
      scheduledTask.runTask();
      verify(repo).deleteOldRecords();
  }

Option 2: Use @Scheduled with very short interval in test and Awaitility

Option 3: Replace with manual trigger in tests (disable scheduling):
  @SpringBootTest(properties = "scheduling.enabled=false")

GOTCHA: Don't test that @Scheduled fires at right time — test the method logic directly
```

---

**Q9. What is mutation testing and how does it help evaluate test quality?**

```
SCRATCHPAD NOTES:
CODE COVERAGE PROBLEM:
  - 100% line coverage doesn't mean tests are good
  - A test can cover a line without asserting anything meaningful
  
MUTATION TESTING:
  - Tool (PITest for Java) modifies (mutates) your code:
    * Changes > to >= 
    * Removes conditionals
    * Changes return values
    * Negates conditions
  - Runs your tests against mutated code
  - "Killed mutant" = test caught the change (good!)
  - "Survived mutant" = your tests didn't catch the change (bad!)
  
MUTATION SCORE:
  Killed mutants / Total mutants = mutation score
  Higher score = better test quality

SETUP WITH SPRING BOOT:
  Add PITest Maven/Gradle plugin
  Run: mvn test-compile org.pitest:pitest-maven:mutationCoverage
  
COMMON SURVIVED MUTATIONS INDICATE:
  - Missing boundary condition tests
  - Assertions that are too loose
  - Missing negative test cases
  - Logic not actually being tested

GOTCHA: Mutation testing is SLOW for large codebases
        Run on critical modules only (core business logic)
        Don't target @SpringBootTest — target unit tests
```

---

**Q10. How do you write good integration tests with `@Transactional` in test classes?**

```
SCRATCHPAD NOTES:
@Transactional on test class/method:
  - Wraps each test in a transaction
  - Rolls back after each test → DB is clean for next test
  - Great for isolation

PROBLEM WITH @Transactional IN TESTS:
  - If your service is also @Transactional, test and service share same transaction
  - Changes are not yet committed → can't verify with separate DB query
  
EXAMPLE GOTCHA:
  @Test @Transactional
  void testCreateUser() {
      userService.create(dto);
      // If service uses REQUIRES_NEW, it committed to DB
      // But test transaction hasn't committed → findAll() might show different state
  }

SOLUTION OPTIONS:
1. Don't use @Transactional in tests when testing commit behavior
   → Use @BeforeEach/@AfterEach with manual cleanup

2. Use TestTransaction utility:
   TestTransaction.flagForCommit();
   TestTransaction.end();
   TestTransaction.start();

3. Use @Sql to set up / tear down data:
   @Sql("classpath:test-data.sql")
   @Sql(scripts = "cleanup.sql", executionPhase = AFTER_TEST_METHOD)

4. Use TestContainers + let each test handle its own data

BEST PRACTICE:
  - @Transactional on tests = fast isolation for read-heavy tests
  - For write/commit tests = manage transactions manually or use cleanup SQL
```

---

## ☁️ SECTION 6: Microservices & Spring Cloud

---

**Q1. What are the key challenges of microservices and how does Spring Cloud address them?**

```
SCRATCHPAD NOTES:
CHALLENGES → SPRING CLOUD SOLUTIONS:

1. Service Discovery
   Challenge: Services need to find each other (IPs change)
   Solution: Spring Cloud Netflix Eureka / Consul
   
2. Load Balancing
   Challenge: Multiple instances, need to distribute traffic
   Solution: Spring Cloud LoadBalancer (replaced Ribbon)

3. Configuration Management
   Challenge: Config scattered across many services
   Solution: Spring Cloud Config Server

4. Resilience / Fault Tolerance
   Challenge: Cascading failures when downstream service fails
   Solution: Resilience4j (Circuit Breaker, Retry, Rate Limiter)

5. API Gateway / Routing
   Challenge: Single entry point, cross-cutting concerns (auth, rate limiting)
   Solution: Spring Cloud Gateway

6. Distributed Tracing
   Challenge: Can't trace request across multiple services
   Solution: Micrometer Tracing + Zipkin/Jaeger

7. Distributed Transactions
   Challenge: ACID transactions across services
   Solution: Saga Pattern (not a Spring Cloud feature, but a design pattern)

8. Inter-service Communication
   Challenge: Boilerplate HTTP client code
   Solution: OpenFeign (declarative REST client)

THE HONEST ANSWER:
  "Microservices solve organizational scale problems but introduce distributed systems 
   complexity. Spring Cloud helps but doesn't eliminate the challenges."
```

---

**Q2. What is a Circuit Breaker pattern? How do you implement it with Resilience4j?**

```
SCRATCHPAD NOTES:
PROBLEM WITHOUT CIRCUIT BREAKER:
  Service A calls Service B repeatedly
  Service B is down/slow
  Service A threads pile up waiting → Service A goes down too (cascading failure)

CIRCUIT BREAKER STATES:
  CLOSED: everything working, requests pass through
  OPEN: too many failures detected, requests IMMEDIATELY fail with fallback
  HALF-OPEN: after timeout, allow limited requests to test if B has recovered
             → success → CLOSED; failure → OPEN again

RESILIENCE4J SETUP:
  Dependency: resilience4j-spring-boot3 (or 2)

  @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
  public PaymentResponse processPayment(PaymentRequest request) {
      return paymentClient.charge(request);
  }
  
  public PaymentResponse paymentFallback(PaymentRequest req, Exception e) {
      log.warn("Payment service unavailable: {}", e.getMessage());
      return PaymentResponse.queued(); // queue for later, or return cached
  }

CONFIG (application.yml):
  resilience4j:
    circuitbreaker:
      instances:
        paymentService:
          sliding-window-size: 10
          failure-rate-threshold: 50       # open if 50% fail
          wait-duration-in-open-state: 60s
          permitted-calls-in-half-open: 5

OTHER RESILIENCE4J FEATURES:
  @Retry(name = "paymentService", fallbackMethod = "...")
  @RateLimiter(name = "paymentService")
  @Bulkhead(name = "paymentService")  — limit concurrent calls
  @TimeLimiter — timeout for async calls

GOTCHA: @CircuitBreaker fallback method signature must match main method + Throwable
```

---

**Q3. What is the Saga pattern? How do you handle distributed transactions in microservices?**

```
SCRATCHPAD NOTES:
PROBLEM:
  Order Service deducts stock
  Payment Service charges card
  If payment fails AFTER stock deducted → inconsistent state
  Can't use ACID transactions across services (different DBs)

SAGA PATTERN — sequence of local transactions, each publishes event/message

TWO TYPES:

1. Choreography (Event-driven):
   - Each service listens for events, does its work, publishes next event
   - Order Placed → Inventory Reserved → Payment Charged → Order Confirmed
   - If payment fails → Payment Failed event → Inventory Released → Order Cancelled
   ✅ Decoupled, simple to start
   ❌ Hard to track overall state, complex failure paths

2. Orchestration:
   - Central Saga Orchestrator coordinates everything
   - Orchestrator calls each service and handles outcomes
   - Easier to see business flow
   ✅ Central visibility, easier to debug
   ❌ Coupling to orchestrator, potential single point of failure

COMPENSATING TRANSACTIONS:
  - Each action must have a corresponding undo/compensate action
  - Reserve Inventory → compensate = Release Inventory
  - Charge Payment → compensate = Refund Payment

TOOLS:
  - Axon Framework (Spring-based, supports Saga)
  - Eventuate Tram
  - Manual with Kafka/RabbitMQ + state machine

REAL ANSWER: "We used choreography with Kafka — each service emitted domain events.
             Worked fine but debugging distributed failures was challenging."
```

---

**Q4. How does Spring Cloud Gateway work? What can you do with it?**

```
SCRATCHPAD NOTES:
Spring Cloud Gateway (NOT Zuul which is deprecated):
  - Reactive (built on Spring WebFlux + Netty)
  - Single entry point for all client requests
  - Routes requests to appropriate microservices

CORE CONCEPTS:
  Route: id + destination URI + predicates + filters
  Predicate: condition to match (path, header, method, query param)
  Filter: modify request/response (add header, auth check, rate limit)

BASIC CONFIG:
  spring:
    cloud:
      gateway:
        routes:
          - id: user-service
            uri: lb://user-service  # lb = load balanced via discovery
            predicates:
              - Path=/api/users/**
            filters:
              - StripPrefix=1
              - AddRequestHeader=X-Request-From, gateway

COMMON USE CASES:
  - Authentication/Authorization filter (verify JWT at gateway)
  - Rate limiting: RequestRateLimiter filter (with Redis)
  - Request logging
  - Circuit breaker at gateway level
  - URL rewriting / stripping prefixes
  - SSL termination

GLOBAL FILTERS (applies to all routes):
  @Component
  public class AuthFilter implements GlobalFilter {
      Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
          // check token, reject if invalid
      }
  }

GOTCHA: Spring Cloud Gateway is REACTIVE — don't mix with blocking Spring MVC code
        Can't use standard Spring MVC annotations here
```

---

**Q5. How does service discovery work with Eureka?**

```
SCRATCHPAD NOTES:
PROBLEM: In microservices, service instances come and go (scaling, failures).
         Hard-coded IPs don't work in cloud/containers.

EUREKA COMPONENTS:
  - Eureka Server: registry of all services
  - Eureka Client: each service registers itself and queries registry

FLOW:
1. Service starts → registers with Eureka Server (sends heartbeat every 30s)
2. Service fails to send heartbeats → removed after 90s (configurable)
3. Client asks Eureka: "Where is order-service?" → gets list of instances
4. Client-side load balancing picks one instance

SETUP:

Eureka Server:
  @SpringBootApplication @EnableEurekaServer
  eureka.client.register-with-eureka=false
  eureka.client.fetch-registry=false

Eureka Client:
  @SpringBootApplication @EnableDiscoveryClient (or auto-detected)
  spring.application.name=user-service  ← registration name
  eureka.client.service-url.defaultZone=http://eureka:8761/eureka/

USING DISCOVERY:
  // RestTemplate with load balancing:
  @Bean @LoadBalanced
  public RestTemplate restTemplate() { return new RestTemplate(); }
  
  // Then use service name:
  restTemplate.getForObject("http://user-service/api/users/{id}", User.class, id);
  
  // Feign Client:
  @FeignClient(name = "user-service")

GOTCHA: Eureka has eventual consistency — brief period where registry is stale
        "Split brain" during network partitions (Eureka prefers availability over consistency)
        In Kubernetes, often replaced by K8s built-in service discovery (K8s Services)
```

---

**Q6. What is Feign Client and how do you use it?**

```
SCRATCHPAD NOTES:
Feign = declarative HTTP client — write interface, Spring generates implementation

SETUP:
  Dependency: spring-cloud-starter-openfeign
  @EnableFeignClients on main class

BASIC USAGE:
  @FeignClient(name = "user-service", url = "${user.service.url}")
  public interface UserServiceClient {
      @GetMapping("/api/users/{id}")
      UserDto getUserById(@PathVariable("id") Long id);
      
      @PostMapping("/api/users")
      UserDto createUser(@RequestBody CreateUserRequest request);
  }
  
  // Inject and use:
  @Autowired UserServiceClient userClient;
  UserDto user = userClient.getUserById(1L);

WITH EUREKA (no hardcoded URL):
  @FeignClient(name = "user-service")  // just use app name
  // Spring Cloud LoadBalancer handles discovery

ERROR HANDLING:
  Implement FeignErrorDecoder:
  public class UserServiceErrorDecoder implements ErrorDecoder {
      public Exception decode(String method, Response response) {
          if (response.status() == 404) return new UserNotFoundException(...);
          return new Default().decode(method, response);
      }
  }

WITH CIRCUIT BREAKER:
  @FeignClient(name = "user-service", fallback = UserServiceFallback.class)
  // or fallbackFactory for access to the exception

INTERCEPTORS (add auth header to all requests):
  @Bean
  public RequestInterceptor jwtInterceptor() {
      return template -> template.header("Authorization", "Bearer " + getToken());
  }

GOTCHA: @FeignClient and @RequestMapping don't always play well together
        Use @GetMapping/@PostMapping on Feign interfaces instead
```

---

**Q7. What is Spring Cloud Config and how does it work?**

```
SCRATCHPAD NOTES:
PURPOSE: Centralized configuration management for microservices
         Instead of each service having its own application.yml, 
         config lives in a Git repo (or Vault, S3, etc.)

COMPONENTS:
  Config Server: serves configuration files
  Config Client: each service fetches its config at startup

CONFIG SERVER SETUP:
  @SpringBootApplication @EnableConfigServer
  
  application.yml:
  spring.cloud.config.server.git.uri=https://github.com/org/config-repo

CONFIG REPO STRUCTURE:
  config-repo/
    application.yml           ← shared by all services
    user-service.yml          ← specific to user-service
    user-service-prod.yml     ← user-service in prod profile

CONFIG CLIENT:
  spring.application.name=user-service
  spring.config.import=configserver:http://config-server:8888

DYNAMIC REFRESH:
  @RefreshScope on beans that should reload on config change
  POST /actuator/refresh to trigger reload (no restart needed)
  
  With Spring Cloud Bus + Kafka/RabbitMQ:
    POST /actuator/busrefresh → all service instances refresh

ENCRYPTION:
  Config Server supports encrypted values:
  {cipher}encrypted_value_here

GOTCHA: If Config Server is down at startup → service fails to start
        Use spring.cloud.config.fail-fast=false for optional config server
        Add local fallback: spring.cloud.config.fail-fast=false with local defaults
        In K8s: often replaced by ConfigMaps + Secrets
```

---

**Q8. How do you implement rate limiting in Spring Cloud Gateway?**

```
SCRATCHPAD NOTES:
Spring Cloud Gateway has built-in rate limiter using Redis (token bucket algorithm)

SETUP:
  Dependencies: spring-cloud-starter-gateway + spring-boot-starter-data-redis-reactive

CONFIG:
  spring:
    cloud:
      gateway:
        routes:
          - id: api-route
            uri: lb://api-service
            predicates:
              - Path=/api/**
            filters:
              - name: RequestRateLimiter
                args:
                  redis-rate-limiter.replenishRate: 10   # tokens/second
                  redis-rate-limiter.burstCapacity: 20   # max burst
                  redis-rate-limiter.requestedTokens: 1  # cost per request
                  key-resolver: "#{@userKeyResolver}"

KEY RESOLVER (what to rate-limit by):
  // By IP:
  @Bean
  public KeyResolver ipKeyResolver() {
      return exchange -> Mono.just(
          exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
      );
  }
  
  // By authenticated user:
  @Bean
  public KeyResolver userKeyResolver() {
      return exchange -> ReactiveSecurityContextHolder.getContext()
          .map(ctx -> ctx.getAuthentication().getName());
  }

ON LIMIT EXCEEDED: returns 429 Too Many Requests

ALTERNATIVE without Redis (application-level):
  Resilience4j @RateLimiter for per-service rate limiting
  Bucket4j for more advanced rate limiting

GOTCHA: Gateway rate limiting is instance-level by default unless using Redis (distributed)
        For true distributed rate limiting, Redis is required
```

---

**Q9. What is the difference between synchronous and asynchronous inter-service communication?**

```
SCRATCHPAD NOTES:
SYNCHRONOUS (HTTP/REST, gRPC):
  Service A calls Service B, waits for response
  
  Pros:
    ✅ Simple to understand, debug, implement
    ✅ Immediate response — know success/failure right away
    ✅ Easy to return data to caller
    
  Cons:
    ❌ Tight temporal coupling — B must be available when A needs it
    ❌ Cascading failures — B down = A fails
    ❌ Blocking threads (unless reactive)
    ❌ Hard to scale independently

ASYNCHRONOUS (Kafka, RabbitMQ, SQS):
  Service A publishes event/message, doesn't wait
  
  Pros:
    ✅ Loose coupling — B can be down, messages queue up
    ✅ Better resilience
    ✅ Natural load leveling
    ✅ Services scale independently
    
  Cons:
    ❌ Complex to implement and debug
    ❌ Eventual consistency (no immediate confirmation)
    ❌ Message ordering, deduplication challenges
    ❌ Harder to get response back to original caller

WHEN TO USE WHICH:
  Sync: Query for data you need NOW, user-facing request/response (read APIs)
  Async: Events that happened (order placed, payment processed), long-running jobs,
         notifications, updating downstream systems

REAL ANSWER: 
  "Use sync for queries where you need data back immediately.
   Use async for commands/events where eventual consistency is OK.
   Saga pattern uses async messaging for distributed transactions."
```

---

**Q10. What is the Strangler Fig pattern and how would you migrate a monolith to microservices?**

```
SCRATCHPAD NOTES:
STRANGLER FIG PATTERN:
  Named after a plant that grows around a tree and slowly replaces it
  Don't rewrite everything at once — gradually extract services

MIGRATION APPROACH:

1. Start with a Facade/API Gateway:
   All traffic goes through gateway
   Initially routes everything to monolith

2. Identify seams to extract (Domain-Driven Design):
   Look for: bounded contexts, separate teams, different scaling needs
   Good first candidates: user auth, email/notifications, reporting

3. Extract one service at a time:
   - Create new microservice with its own DB
   - Route specific paths to new service via gateway
   - Monolith still handles everything else
   - Gradually migrate data and routes

4. Handle data:
   - Dual writes during transition (monolith + new service)
   - Eventual consistency (sync data via events)
   - Extract DB tables to separate schema/DB

5. Remove monolith code as services take over

ANTI-PATTERNS TO AVOID:
  ❌ Big bang rewrite — risky, expensive, often fails
  ❌ Distributed monolith — microservices that are tightly coupled (worst of both worlds)
  ❌ Too many services too fast — start small (2-3 services)
  ❌ Shared database across services — defeats the purpose

REAL ANSWER KEY POINT:
  "I'd start by identifying the bounded context with the clearest domain boundary 
   and most independent scaling need. Extract it incrementally behind an API gateway."
```

---

## 📨 SECTION 7: Messaging (Kafka / RabbitMQ)

---

**Q1. What is Apache Kafka and how does it differ from traditional message queues like RabbitMQ?**

```
SCRATCHPAD NOTES:
KAFKA:
  - Distributed log / event streaming platform
  - Messages (events) stored durably in partitioned, ordered logs
  - Consumers track their own offset (position in log)
  - Messages are NOT deleted after consumption — retained for configurable period
  - Multiple consumer groups can independently read the same events
  - Excellent for: high throughput, event sourcing, audit logs, stream processing

RABBITMQ:
  - Traditional message broker
  - Producer → Exchange → Queue → Consumer
  - Message removed after consumed (acknowledged)
  - Broker routes messages based on routing keys / exchange types
  - Better for: task queues, RPC patterns, complex routing, when you need acknowledgment

KEY DIFFERENCES:
  | Feature          | Kafka                    | RabbitMQ                   |
  |------------------|--------------------------|----------------------------|
  | Message deletion | Retained (configurable)  | Deleted after consumed     |
  | Multiple readers | Yes (consumer groups)    | Only one consumer per msg  |
  | Ordering         | Per-partition            | Per-queue                  |
  | Throughput       | Very high                | Moderate                   |
  | Use case         | Event streaming, logs    | Task queues, RPC           |
  | Routing          | Topic-based              | Exchange-based (flexible)  |

WHEN TO USE KAFKA:
  - Event-driven architecture
  - Event sourcing
  - High throughput (millions of events/second)
  - Multiple consumers need same event
  - Audit trail / replay capability

WHEN TO USE RABBITMQ:
  - Task distribution
  - Complex routing logic
  - Request/reply patterns
  - When simplicity matters more than scale
```

---

**Q2. How do you set up a Kafka producer and consumer in Spring Boot?**

```
SCRATCHPAD NOTES:
DEPENDENCY: spring-kafka

PRODUCER:
  // Config (or auto-configured with spring.kafka.bootstrap-servers)
  spring.kafka.bootstrap-servers=localhost:9092
  spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
  spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

  // Send message:
  @Autowired KafkaTemplate<String, OrderEvent> kafkaTemplate;
  
  public void publishOrderEvent(OrderEvent event) {
      kafkaTemplate.send("orders-topic", event.getOrderId().toString(), event);
  }
  
  // With callback:
  kafkaTemplate.send("orders-topic", event)
      .whenComplete((result, ex) -> {
          if (ex != null) log.error("Failed to send", ex);
          else log.info("Sent to partition {}", result.getRecordMetadata().partition());
      });

CONSUMER:
  spring.kafka.consumer.group-id=order-processor
  spring.kafka.consumer.key-deserializer=StringDeserializer
  spring.kafka.consumer.value-deserializer=JsonDeserializer
  spring.kafka.consumer.properties.spring.json.trusted.packages=com.myapp.events

  @KafkaListener(topics = "orders-topic", groupId = "order-processor")
  public void handleOrder(OrderEvent event, @Header(KafkaHeaders.RECEIVED_PARTITION) int partition) {
      log.info("Processing order {} from partition {}", event.getOrderId(), partition);
      orderService.process(event);
  }

GOTCHA: Consumer deserializer must trust the package of the event class
        Use spring.kafka.consumer.properties.spring.json.trusted.packages=*
        (or specific package in production)
```

---

**Q3. What are Kafka consumer groups and partitions? Why do they matter?**

```
SCRATCHPAD NOTES:
PARTITION:
  - A topic is divided into N partitions
  - Each partition is an ordered, immutable log
  - Messages within a partition are strictly ordered
  - Partitions allow parallel consumption

CONSUMER GROUP:
  - Group of consumer instances sharing work
  - Kafka assigns each partition to ONE consumer in the group
  - Multiple consumer groups can read the same topic independently
  - If consumers > partitions → some consumers are idle

SCALING RULES:
  - Want more parallelism? Increase partitions + consumer instances
  - Max parallelism = number of partitions
  - 3 partitions → max 3 consumers can work in parallel in same group

CONSUMER GROUP EXAMPLE:
  Topic: orders (3 partitions)
  
  Group "email-service" (2 consumers):
    consumer-1: reads partitions 0, 1
    consumer-2: reads partition 2
    
  Group "analytics-service" (1 consumer):
    consumer-1: reads ALL 3 partitions independently

OFFSET MANAGEMENT:
  - Each consumer tracks offset per partition
  - Committed offset = "I've processed up to here"
  - On restart → resume from committed offset
  - spring.kafka.consumer.auto-offset-reset=earliest/latest
  - spring.kafka.consumer.enable-auto-commit=false (manual for exactly-once)

REBALANCING:
  - Consumer joins/leaves group → partitions reassigned
  - During rebalance, consumption stops briefly (rebalance pause)

GOTCHA: Ordering is only guaranteed within a partition, not across partitions
        Use the same message key for related events → same key → same partition
```

---

**Q4. How do you handle message failures and dead-letter queues in Kafka/RabbitMQ?**

```
SCRATCHPAD NOTES:
KAFKA FAILURE HANDLING:

1. Retry on error:
  @KafkaListener(topics = "orders")
  @RetryableTopic(attempts = "3", 
                  backoff = @Backoff(delay = 1000, multiplier = 2))
  public void handleOrder(OrderEvent event) { ... }
  
  → Creates retry topics: orders-retry-0, orders-retry-1
  → Dead Letter Topic (DLT): orders-dlt (after all retries exhausted)

2. Manual retry topics:
  spring.kafka.listener.ack-mode=manual
  → Call acknowledgment.acknowledge() on success
  → On failure: don't ack → redelivered OR seek back to offset

3. Error handler:
  @Bean
  public DefaultErrorHandler errorHandler(KafkaTemplate<?,?> template) {
      DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
      return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));
  }

RABBITMQ FAILURE HANDLING:

1. Manual acknowledgment:
  @RabbitListener(queues = "orders")
  public void handleOrder(OrderEvent event, Channel channel, 
                          @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
      try {
          process(event);
          channel.basicAck(tag, false);
      } catch (Exception e) {
          channel.basicNack(tag, false, false); // false = don't requeue → goes to DLQ
      }
  }

2. Dead Letter Queue setup:
  @Bean
  Queue ordersQueue() {
      return QueueBuilder.durable("orders")
          .withArgument("x-dead-letter-exchange", "dlx")
          .withArgument("x-dead-letter-routing-key", "orders.dead")
          .build();
  }

IDEMPOTENCY:
  Critical for retries — processing same message twice must be safe
  Use unique message ID + idempotency key in DB to deduplicate
```

---

**Q5. What is the Outbox Pattern and why is it important in event-driven microservices?**

```
SCRATCHPAD NOTES:
PROBLEM (Dual Write):
  public void createOrder(Order order) {
      orderRepository.save(order);           // DB write
      kafkaTemplate.send("orders", event);   // Kafka write
  }
  
  What if DB succeeds but Kafka fails? 
  Or DB fails after Kafka succeeds?
  → Inconsistent state — order saved but no event, or event sent for non-existent order

OUTBOX PATTERN SOLUTION:
  1. Write to DB AND outbox table in SAME local transaction:
     INSERT INTO orders (...)
     INSERT INTO outbox (event_type, payload, status='PENDING')
     → Atomic! Either both succeed or both fail
     
  2. Separate process polls outbox table, publishes to Kafka, marks as PUBLISHED:
     SELECT * FROM outbox WHERE status = 'PENDING'
     → publish to Kafka
     → UPDATE outbox SET status = 'PUBLISHED'

IMPLEMENTATION OPTIONS:
  a) Polling-based: Scheduled task polls outbox table (simple but DB load)
  
  b) Debezium (Change Data Capture):
     Reads DB transaction log (WAL for Postgres)
     Streams changes directly to Kafka
     More efficient — no polling needed
     
  c) Spring Application Events + @TransactionalEventListener:
     @TransactionalEventListener(phase = AFTER_COMMIT)
     → Fires ONLY after transaction commits
     → Still at-least-once risk if app crashes between commit and event publishing

GUARANTEE:
  At-least-once delivery (deduplication needed on consumer side)
  Exactly-once very hard in distributed systems

GOTCHA: Outbox table can grow large — add cleanup job for old PUBLISHED records
```

---

**Q6. How do you configure `@Async` in Spring Boot and what are the pitfalls?**

```
SCRATCHPAD NOTES:
SETUP:
  @EnableAsync on @Configuration class
  
  @Service
  public class EmailService {
      @Async
      public void sendEmail(String to, String subject) {
          // runs in separate thread
      }
      
      @Async
      public CompletableFuture<String> processAsync() {
          // return result from async method
          return CompletableFuture.completedFuture("done");
      }
  }

DEFAULT EXECUTOR:
  SimpleAsyncTaskExecutor — creates a NEW thread for EVERY call (bad for prod!)
  
CUSTOM THREAD POOL (recommended):
  @Bean
  public Executor taskExecutor() {
      ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
      executor.setCorePoolSize(5);
      executor.setMaxPoolSize(10);
      executor.setQueueCapacity(100);
      executor.setThreadNamePrefix("AsyncTask-");
      executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
      executor.initialize();
      return executor;
  }

PITFALLS:
  1. Self-invocation: same proxy issue as @Transactional
     calling this.asyncMethod() from same class → NOT async
     
  2. Exception handling: exceptions in @Async are silently swallowed
     → Implement AsyncUncaughtExceptionHandler
     
  3. SecurityContext: not propagated to async thread by default
     → Use DelegatingSecurityContextExecutor
     
  4. @Transactional + @Async: each runs in its own transaction context
  
  5. Return type must be void or Future<T>/CompletableFuture<T>

GOTCHA: Thread pool exhaustion under load → CallerRunsPolicy or proper monitoring
```

---

**Q7. What is `@TransactionalEventListener` and when would you use it over `@EventListener`?**

```
SCRATCHPAD NOTES:
@EventListener:
  Fires when event is published (regardless of transaction state)
  If called inside a transaction → fires BEFORE transaction commits
  Problem: event fires but transaction rolls back → downstream processed non-existent data

@TransactionalEventListener:
  Tied to transaction lifecycle
  PHASES:
    AFTER_COMMIT (default): fires only after transaction COMMITS
    AFTER_ROLLBACK: fires if transaction rolls back
    AFTER_COMPLETION: fires after commit OR rollback
    BEFORE_COMMIT: fires just before commit

EXAMPLE USE CASE:
  // Service creates order and publishes event
  @Transactional
  public Order createOrder(CreateOrderDto dto) {
      Order order = orderRepository.save(new Order(dto));
      eventPublisher.publishEvent(new OrderCreatedEvent(order));
      return order;
  }
  
  // This only fires if the transaction above commits successfully
  @TransactionalEventListener(phase = AFTER_COMMIT)
  public void handleOrderCreated(OrderCreatedEvent event) {
      emailService.sendConfirmation(event.getOrder());
      kafkaTemplate.send("orders", event);
  }

WHEN TO USE:
  @EventListener: non-transactional contexts, or when you WANT to fire before commit
  @TransactionalEventListener: side effects that should only happen on successful save
                                (send email, send Kafka event, update read model)

GOTCHA: @TransactionalEventListener AFTER_COMMIT runs OUTSIDE the original transaction
        → No active transaction in the listener by default
        → If listener needs its own transaction: add @Transactional(propagation = REQUIRES_NEW)
```

---

**Q8. What are RabbitMQ exchanges and what are the different exchange types?**

```
SCRATCHPAD NOTES:
COMPONENTS:
  Producer → Exchange → Binding → Queue → Consumer

EXCHANGE TYPES:

1. Direct Exchange:
   - Routes messages to queues where binding key = routing key
   - One-to-one routing
   - Example: order.created → order-processing queue

2. Topic Exchange:
   - Pattern matching on routing key
   - * matches one word, # matches zero or more words
   - Example: order.* → all order events, *.failed → all failure events

3. Fanout Exchange:
   - Broadcasts to ALL bound queues (ignores routing key)
   - Publish/subscribe pattern
   - Example: Push notifications to all subscribers

4. Headers Exchange:
   - Routes based on message headers instead of routing key
   - Rarely used in practice

SPRING SETUP:
  @Bean
  DirectExchange orderExchange() {
      return new DirectExchange("order-exchange");
  }
  
  @Bean
  Queue orderQueue() {
      return new Queue("order-processing", true);
  }
  
  @Bean
  Binding orderBinding(Queue orderQueue, DirectExchange orderExchange) {
      return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.created");
  }
  
  // Consumer:
  @RabbitListener(queues = "order-processing")
  public void handleOrder(OrderEvent event) { ... }

GOTCHA: Durable queues/exchanges survive RabbitMQ restart
        Non-durable = lost on restart
        Persistent messages = survive restart if queue is durable
```

---

**Q9. How do you ensure message ordering and exactly-once delivery in Kafka?**

```
SCRATCHPAD NOTES:
MESSAGE ORDERING:
  - Guaranteed WITHIN a partition, NOT across partitions
  - Use same key for related messages → same key → same partition
    kafkaTemplate.send("orders", order.getCustomerId().toString(), event);
  - If strict global ordering needed → single partition (kills parallelism)
  - Usually domain-level ordering (all events for customer X are ordered)

EXACTLY-ONCE DELIVERY:

At-most-once: fire and forget (may lose messages, no duplicate)
At-least-once: acknowledge only on success (may get duplicates)
Exactly-once: hardest, Kafka supports it with transactions

KAFKA IDEMPOTENT PRODUCER:
  spring.kafka.producer.properties.enable.idempotence=true
  → Producer assigns sequence numbers
  → Broker deduplicates within producer session
  → Exactly-once PRODUCER semantics

KAFKA TRANSACTIONS (produce + consume atomically):
  spring.kafka.producer.transaction-id-prefix=tx-
  @Transactional  (Spring Kafka @Transactional)
  
  // Consume from topic A, process, produce to topic B — atomically:
  @KafkaListener(topics = "input")
  @Transactional
  public void process(InputEvent event) {
      OutputEvent output = transform(event);
      kafkaTemplate.send("output", output);
  }

CONSUMER IDEMPOTENCY (most practical approach):
  - Store processed message IDs in DB
  - Check before processing: if already processed → skip
  
  SELECT count(*) FROM processed_events WHERE event_id = ?
  → If 0: process and INSERT event_id
  → This is idempotency key pattern

GOTCHA: Kafka exactly-once works within Kafka ecosystem
        If your consumer writes to DB + Kafka → still a dual-write problem
        → Use Outbox pattern
```

---

**Q10. What is `@Scheduled` and how do you configure it properly for production?**

```
SCRATCHPAD NOTES:
SETUP: @EnableScheduling on @Configuration class

TYPES:
  @Scheduled(fixedRate = 5000)           // every 5 sec regardless of execution time
  @Scheduled(fixedDelay = 5000)          // 5 sec AFTER previous execution finishes
  @Scheduled(fixedDelay = 5000, initialDelay = 10000)  // wait 10s before first run
  @Scheduled(cron = "0 0 2 * * MON-FRI")  // 2am weekdays

CRON FORMAT: second minute hour dayOfMonth month dayOfWeek
  "0 */15 * * * *" — every 15 minutes
  "0 0 12 * * *"   — every day at noon
  "0 0 0 1 * *"    — first day of every month

THREAD POOL (CRITICAL for production):
  By default: single-threaded TaskScheduler
  → One long-running task BLOCKS all other scheduled tasks!
  
  @Bean
  public TaskScheduler taskScheduler() {
      ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
      scheduler.setPoolSize(5);
      scheduler.setThreadNamePrefix("scheduled-");
      return scheduler;
  }

DISTRIBUTED SCHEDULING (multiple instances):
  Problem: In cluster, same job runs on EVERY instance simultaneously
  
  Solutions:
  1. ShedLock: annotation-based distributed lock
     @SchedulerLock(name = "cleanupJob", lockAtMostFor = "PT10M")
  2. Quartz Scheduler with JDBC job store (cluster-aware)
  3. Spring Batch for complex jobs
  4. Kubernetes CronJob (only one pod runs the job)

ENABLING/DISABLING:
  @ConditionalOnProperty(name = "scheduling.enabled", havingValue = "true", matchIfMissing = true)
  → Disable in tests: spring.task.scheduling.pool.size=0 or @TestPropertySource

GOTCHA: @Async + @Scheduled on same method → may not work as expected
        Separate the scheduling trigger from the async work
```

---

## ⚡ SECTION 8: Caching & Performance

---

**Q1. How does Spring's cache abstraction work? What annotations are available?**

```
SCRATCHPAD NOTES:
SETUP: @EnableCaching on @Configuration

ANNOTATIONS:
  @Cacheable — cache method result, skip execution if cache hit:
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) { ... }
    
    // Custom key (SpEL):
    @Cacheable(value = "users", key = "#dto.email + '-' + #dto.orgId")
    
    // Conditional caching:
    @Cacheable(value = "products", condition = "#id > 0", unless = "#result == null")

  @CachePut — ALWAYS executes method, updates cache with result:
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) { ... }

  @CacheEvict — removes from cache:
    @CacheEvict(value = "users", key = "#id")           // remove specific
    @CacheEvict(value = "users", allEntries = true)     // remove all
    @CacheEvict(value = "users", beforeInvocation = true) // evict before method runs

  @Caching — multiple cache ops on one method:
    @Caching(evict = {
        @CacheEvict("users"),
        @CacheEvict(value = "usersByEmail", key = "#user.email")
    })

CACHE PROVIDERS:
  Default: ConcurrentHashMap (in-memory, no TTL, not distributed)
  Redis: distributed, TTL support, shared across instances
  Caffeine: high-performance in-memory, TTL/size eviction

GOTCHA: Same self-invocation proxy issue — @Cacheable won't work if called within same class
        Only works when called from OUTSIDE the bean through proxy
```

---

**Q2. How do you configure Redis as a cache in Spring Boot?**

```
SCRATCHPAD NOTES:
DEPENDENCY: spring-boot-starter-data-redis

CONFIG:
  spring.data.redis.host=localhost
  spring.data.redis.port=6379
  spring.cache.type=redis
  spring.cache.redis.time-to-live=600000  # 10 minutes in ms

PROGRAMMATIC CONFIGURATION (for custom TTL per cache):
  @Bean
  public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
      RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
          .entryTtl(Duration.ofMinutes(10))
          .disableCachingNullValues()
          .serializeValuesWith(
              RedisSerializationContext.SerializationPair.fromSerializer(
                  new GenericJackson2JsonRedisSerializer()));
      
      Map<String, RedisCacheConfiguration> configs = Map.of(
          "users", defaultConfig.entryTtl(Duration.ofHours(1)),
          "products", defaultConfig.entryTtl(Duration.ofMinutes(5))
      );
      
      return RedisCacheManager.builder(factory)
          .cacheDefaults(defaultConfig)
          .withInitialCacheConfigurations(configs)
          .build();
  }

DIRECT REDIS ACCESS (beyond caching):
  @Autowired RedisTemplate<String, Object> redisTemplate;
  
  redisTemplate.opsForValue().set("key", value, 10, MINUTES);
  Object value = redisTemplate.opsForValue().get("key");
  redisTemplate.opsForHash().put("user:1", "name", "John");
  redisTemplate.opsForSet().add("online-users", userId);

CACHE PATTERNS:
  Cache-Aside (default with @Cacheable): app checks cache, if miss → load from DB → cache it
  Write-Through: update cache and DB together (@CachePut)
  Write-Behind: update cache, async write to DB (complex, not built into Spring)

GOTCHA: Objects stored in Redis must be Serializable (or use JSON serialization)
        Default Java serialization is problematic across versions → use Jackson JSON
```

---

**Q3. How would you diagnose and fix a slow Spring Boot API endpoint?**

```
SCRATCHPAD NOTES:
STEP 1 — MEASURE FIRST, don't guess:
  - Check Actuator /metrics (response times, DB connection pool metrics)
  - Enable Micrometer + Prometheus + Grafana for visibility
  - Add distributed tracing (Zipkin) to see where time is spent
  - Enable slow query logging in DB (slow_query_log in MySQL, log_min_duration in Postgres)

STEP 2 — IDENTIFY BOTTLENECK CATEGORY:
  
  A) Database issues:
    - N+1 queries (enable SQL logging: spring.jpa.show-sql=true)
    - Missing indexes (EXPLAIN ANALYZE the slow query)
    - Large result sets (add pagination)
    - Connection pool exhaustion (check HikariCP metrics)
    - Long-running transactions holding locks
    
  B) Application code:
    - Synchronous calls to external services (add timeouts, consider async)
    - Unnecessary computation in request path
    - Missing caching for expensive operations
    - Memory allocation / GC pauses (check GC logs, heap dumps)
    
  C) Thread starvation:
    - All threads busy waiting for DB/external services
    - Reactive/async could help
    
  D) Network:
    - External API latency (circuit breaker, caching, async)

STEP 3 — FIXES:
  - Add @Cacheable for expensive queries
  - Add JOIN FETCH or @EntityGraph to fix N+1
  - Add DB indexes on frequently queried columns
  - Add pagination to large queries
  - Add timeouts to external service calls
  - Enable async for non-blocking operations
  - Increase thread pool / connection pool if CPU/IO bound

STEP 4 — VERIFY: measure again after fix

GOTCHA: "I'd never optimize without first profiling. 
         Premature optimization often makes code worse without measurable benefit."
```

---

**Q4. What is Caffeine cache and when would you use it over Redis?**

```
SCRATCHPAD NOTES>
Caffeine:
  - High-performance Java in-memory cache
  - Window TinyLFU eviction algorithm (better hit rate than LRU)
  - Built-in support in Spring Boot (spring.cache.type=caffeine)
  - Supports TTL, size-based eviction, refresh-after-write

WHEN TO USE CAFFEINE:
  ✅ Single-instance app
  ✅ Latency-sensitive (nanosecond access vs Redis network hop)
  ✅ Data that fits in JVM heap
  ✅ Cache doesn't need to be shared across instances
  Examples: lookup tables, user permissions that don't change often, 
            country/timezone data

WHEN TO USE REDIS:
  ✅ Multiple instances (need shared cache)
  ✅ Cache survives app restart
  ✅ Session storage
  ✅ Large cached objects (offload from JVM heap)
  ✅ Pub/Sub, distributed locks, rate limiting

CAFFEINE SETUP:
  spring.cache.type=caffeine
  spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=10m

  // Or programmatic:
  @Bean
  public CacheManager cacheManager() {
      CaffeineCacheManager mgr = new CaffeineCacheManager();
      mgr.setCaffeine(Caffeine.newBuilder()
          .expireAfterWrite(10, MINUTES)
          .maximumSize(1000)
          .recordStats());  // for metrics
      return mgr;
  }

TWO-LEVEL CACHE (Local + Redis):
  Caffeine (L1 cache) → Redis (L2 cache) → Database
  Use cache2k or custom implementation
  Benefit: local hit = no network; Redis = shared; DB = fallback

GOTCHA: Caffeine in cluster → each node has its own cache → stale data possible
        Need cache invalidation broadcast via Spring Cloud Bus or pub/sub if data changes
```

---

**Q5. What are common JVM performance tuning considerations for Spring Boot apps?**

```
SCRATCHPAD NOTES:
HEAP SIZING:
  -Xms512m -Xmx2g  (min/max heap)
  Start with: max heap = 75% of available container memory
  Too small → frequent GC; too large → long GC pauses
  
GC SELECTION:
  G1GC (default Java 9+): good balance, low pause times for most apps
    -XX:+UseG1GC -XX:MaxGCPauseMillis=200
  ZGC (Java 15+): sub-millisecond pauses, great for latency-sensitive apps
    -XX:+UseZGC
  Shenandoah: similar to ZGC, used in OpenJDK
  ParallelGC: high throughput, OK for batch processing (longer pauses)

COMMON SPRING BOOT SPECIFIC:
  - Lazy initialization: spring.main.lazy-initialization=true 
    (faster startup, first request slower)
  - Tomcat thread pool: server.tomcat.threads.max=200 (default 200)
  - Enable HTTP/2: server.http2.enabled=true
  - Compression: server.compression.enabled=true

MEMORY ISSUES:
  - OutOfMemoryError: analyze heap dump (jmap, VisualVM, YourKit)
  - Metaspace OOM: -XX:MaxMetaspaceSize=256m
  - Memory leak: look for growing cache, unclosed resources, event listeners

STARTUP OPTIMIZATION:
  - Spring Boot 3 + GraalVM native image (much faster startup, less memory)
  - Lazy bean initialization
  - Reduce classpath scanning scope
  - Remove unused auto-configurations

JAVA 21 VIRTUAL THREADS:
  spring.threads.virtual.enabled=true  (Spring Boot 3.2+)
  Enables virtual threads for Tomcat → much better I/O throughput
  Changes the performance model completely for I/O-bound apps

MONITORING TOOLS:
  Actuator → Micrometer → Prometheus → Grafana
  Java Flight Recorder: -XX:+FlightRecorder
  jstack: thread dump analysis
```

---

**Q6. How do you implement optimistic vs pessimistic locking in JPA?**

```
SCRATCHPAD NOTES>
PROBLEM: Two users update the same entity concurrently → lost update

OPTIMISTIC LOCKING (for low contention, preferred):
  - Assume conflicts are rare
  - Check at UPDATE time whether entity has changed since read
  - Uses @Version field
  
  @Entity
  public class Product {
      @Id Long id;
      int quantity;
      @Version int version;  // automatically incremented on each update
  }
  
  Flow:
    Thread 1 reads Product (version=1)
    Thread 2 reads Product (version=1)
    Thread 1 updates → version becomes 2 → SUCCESS
    Thread 2 tries to update → version=1 but DB has version=2 → 
    OptimisticLockException thrown → retry or tell user

PESSIMISTIC LOCKING (for high contention):
  - Lock the row when reading, prevent others from reading/writing
  - LockModeType.PESSIMISTIC_WRITE → SELECT FOR UPDATE
  
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  Optional<Product> findById(Long id);
  
  Or with entity manager:
  Product p = em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE);

PESSIMISTIC READ vs WRITE:
  PESSIMISTIC_READ: shared lock (others can read, not write)
  PESSIMISTIC_WRITE: exclusive lock (nobody else can read or write)

WHEN TO USE:
  Optimistic: most cases, low contention, better performance, no DB locks held
  Pessimistic: high contention scenarios (inventory management, ticket booking)
               when you can't afford retry logic
               when stale data could cause critical business problems

GOTCHA: Optimistic locking requires retry handling → @Retryable or manual retry loop
        Pessimistic locking can cause deadlocks if lock order isn't consistent
```

---

**Q7. How do you profile a Spring Boot application to find memory leaks?**

```
SCRATCHPAD NOTES:
SYMPTOMS:
  - Heap growing over time, GC not reclaiming memory
  - OutOfMemoryError: Java heap space
  - App slows down over time and eventually crashes

TOOLS:

1. Actuator Heap Metrics:
   GET /actuator/metrics/jvm.memory.used
   Monitor over time with Prometheus + Grafana

2. Heap Dump Analysis:
   Generate: jmap -dump:format=b,file=heap.hprof <pid>
   Or: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
   Analyze with: Eclipse MAT (Memory Analyzer Tool), VisualVM

3. Java Flight Recorder (JFR):
   -XX:StartFlightRecording=filename=recording.jfr,duration=60s,settings=profile
   Analyze in JDK Mission Control

4. VisualVM / JConsole:
   Attach to running JVM
   See heap usage over time, trigger GC, take thread dumps

COMMON SPRING BOOT MEMORY LEAKS:
  - Static fields holding references (static List, Map that grows unbounded)
  - Unclosed resources (streams, connections)
  - Event listeners not unregistered
  - Unbounded cache (no eviction policy)
  - ThreadLocal variables not cleared
  - Hibernate 1st level cache (session) holding too many objects

HEAP DUMP ANALYSIS APPROACH:
  1. Look at retained heap — what's using most memory?
  2. Find GC root path to suspected objects
  3. Common finding: ArrayList, HashMap growing without bound
     → Trace back to what's adding to it

SPRING-SPECIFIC:
  - Check ApplicationContext for unexpected bean retention
  - ApplicationEventPublisher — listeners holding references
  - Caches without eviction policy
```

---

**Q8. What is lazy initialization in Spring Boot and when should you use it?**

```
SCRATCHPAD NOTES:
EAGER initialization (default):
  All singleton beans created at startup
  Startup is slow if many beans, but first request is fast
  Errors caught at startup time

LAZY initialization:
  Beans created only when first requested
  Faster startup time
  First request that triggers bean creation is slower

ENABLE GLOBALLY:
  spring.main.lazy-initialization=true
  
ENABLE PER BEAN:
  @Bean @Lazy
  public HeavyService heavyService() { ... }
  
INJECT LAZILY:
  @Autowired @Lazy HeavyService heavyService;  // proxy injected, bean created on first use

WHEN TO USE:
  ✅ CLI tools / batch jobs that only use subset of beans
  ✅ Development — faster restart during development
  ✅ Serverless / functions where cold start time matters
  ✅ Beans rarely used (admin features, reporting)

WHEN NOT TO USE:
  ❌ Production web apps (you want startup-time failure detection)
  ❌ When you need early validation of configuration
  ❌ When startup time isn't an issue

SPRING NATIVE (GraalVM):
  Better option than lazy init for startup time in production
  AOT compilation → much faster startup than even lazy init

GOTCHA: Lazy init means misconfiguration errors surface at runtime 
        (when bean is first accessed) not at startup
        In production this could mean errors occur under user load
```

---

**Q9. What are virtual threads (Project Loom) and how do they affect Spring Boot?**

```
SCRATCHPAD NOTES:
TRADITIONAL PLATFORM THREADS:
  - 1:1 mapping with OS threads
  - Each thread uses ~1MB stack memory
  - Creating thousands of threads → memory exhaustion + context switching overhead
  - Blocking I/O (DB call, HTTP call) → thread sits idle but still consuming resources

VIRTUAL THREADS (Java 21, GA):
  - Lightweight threads managed by JVM, not OS
  - Can have MILLIONS of virtual threads
  - When virtual thread blocks → JVM parks it (not the OS thread)
  - OS thread (carrier thread) is free to run other virtual threads

SPRING BOOT 3.2+ INTEGRATION:
  spring.threads.virtual.enabled=true
  → Enables virtual threads for Tomcat request handling
  → Each HTTP request gets its own virtual thread

IMPACT:
  - Traditional reactive programming (WebFlux) was needed to avoid thread starvation
  - With virtual threads → blocking code is OK → use regular imperative Spring MVC
  - Much simpler code (no Mono/Flux complexity)
  - Similar throughput to WebFlux for I/O bound apps

WHEN VIRTUAL THREADS HELP:
  ✅ High concurrency with I/O-bound work (DB, HTTP calls)
  ✅ Simplifying reactive code back to imperative

WHEN VIRTUAL THREADS DON'T HELP:
  ❌ CPU-bound work (computation) — still limited by CPU cores
  ❌ Synchronized blocks can cause "pinning" → carrier thread blocked
  ❌ ThreadLocal misuse at massive scale

GOTCHA: Some libraries use synchronized (Hibernate, some JDBC drivers) which causes
        virtual thread pinning — check compatibility
        JVM flag: -XX:+VirtualThreadPinching to detect pinning
```

---

**Q10. How do you measure and improve API response times?**

```
SCRATCHPAD NOTES:
MEASUREMENT:
  1. Micrometer + Spring Boot Actuator:
     http.server.requests metric → percentiles (p50, p95, p99)
     
  2. Custom @Timed annotation:
     @Timed(value = "order.processing.time", percentiles = {0.5, 0.95, 0.99})
     
  3. Distributed tracing: Zipkin shows time spent in each service/method
  
  4. Database: log slow queries, use EXPLAIN ANALYZE

IMPROVEMENT STRATEGIES:

Response Time Buckets:
  < 100ms: target for simple read APIs
  100-500ms: acceptable for moderate DB work
  500ms-2s: noticeable to user, needs attention
  > 2s: unacceptable for interactive UI

TECHNIQUES:
  1. Async processing: return 202 Accepted for long tasks
  2. Caching: avoid redundant DB/external calls
  3. Database optimization: indexes, query optimization, connection pooling
  4. Pagination: don't load 10,000 records for a list view
  5. HTTP compression: server.compression.enabled=true
  6. Projection queries: select only needed columns
  7. Avoid serializing unnecessary fields (Jackson @JsonIgnore)
  8. Connection pool tuning: right-size HikariCP pool
  9. Lazy loading properly configured: don't over-fetch
  10. CDN for static assets, cache-control headers

LOAD TESTING:
  k6, JMeter, Gatling
  Test under realistic load, find bottlenecks before production

GOTCHA: Optimize the p99 not just the average
        Average can hide terrible outliers that affect real users
```

---

## 📊 SECTION 9: Observability (Actuator, Metrics, Tracing)

---

**Q1. What is Spring Boot Actuator and what endpoints does it provide?**

```
SCRATCHPAD NOTES:
Actuator: production-ready features for monitoring and management

KEY ENDPOINTS:
  /actuator/health      — app health status (UP/DOWN), DB, disk, custom
  /actuator/metrics     — JVM, HTTP, DB, cache metrics (via Micrometer)
  /actuator/info        — app metadata (version, git info, custom)
  /actuator/env         — environment properties (careful — can expose secrets!)
  /actuator/loggers     — view/change log levels at runtime (very useful!)
  /actuator/threaddump  — current thread dump
  /actuator/heapdump    — download heap dump (POST)
  /actuator/beans       — all beans in context
  /actuator/mappings    — all @RequestMapping endpoints
  /actuator/conditions  — auto-configuration conditions
  /actuator/scheduledtasks — scheduled task info
  /actuator/caches      — cache info + eviction
  /actuator/shutdown    — gracefully shut down app (disabled by default!)

SECURITY:
  Endpoints are sensitive — secure them!
  
  management.endpoints.web.exposure.include=health,info,metrics
  management.endpoints.web.base-path=/management
  
  In SecurityFilterChain:
  .requestMatchers("/management/**").hasRole("ADMIN")
  Or on separate port:
  management.server.port=8081  (not exposed externally)

CUSTOM HEALTH INDICATOR:
  @Component
  public class ExternalServiceHealthIndicator implements HealthIndicator {
      public Health health() {
          boolean up = checkExternalService();
          return up ? Health.up().withDetail("url", url).build()
                    : Health.down().withDetail("reason", "timeout").build();
      }
  }

CUSTOM INFO:
  info.app.version=@project.version@  (from Maven/Gradle properties)
  info.app.name=My Service

GOTCHA: /actuator/env and /actuator/beans can expose sensitive data
        Expose ONLY what you need, especially in production
```

---

**Q2. How does Micrometer work? How do you integrate it with Prometheus?**

```
SCRATCHPAD NOTES:
Micrometer:
  - Vendor-neutral metrics facade for JVM apps
  - Like SLF4J but for metrics
  - Spring Boot auto-configures it via Actuator
  - Supports many backends: Prometheus, Datadog, New Relic, CloudWatch, etc.

SETUP:
  Dependencies:
    spring-boot-starter-actuator
    micrometer-registry-prometheus
    
  application.yml:
    management.endpoints.web.exposure.include=prometheus
    management.metrics.tags.application=${spring.application.name}

  Prometheus scrapes: GET /actuator/prometheus

BUILT-IN METRICS (auto-collected):
  - JVM: jvm.memory.used, jvm.gc.pause, jvm.threads.live
  - HTTP: http.server.requests (with URI, method, status tags)
  - DB: hikaricp.connections.*, jdbc.connections.*
  - Cache: cache.gets, cache.puts, cache.evictions
  - Kafka: kafka.consumer.records-consumed-rate, etc.

CUSTOM METRICS:
  @Autowired MeterRegistry registry;
  
  // Counter (increases):
  Counter.builder("orders.placed").tag("region", "US").register(registry).increment();
  
  // Gauge (current value):
  Gauge.builder("queue.size", queue, Queue::size).register(registry);
  
  // Timer (latency):
  Timer.builder("order.processing").register(registry)
       .record(() -> processOrder(order));
  
  // Distribution Summary:
  DistributionSummary.builder("order.amount").register(registry).record(amount);

  // @Timed annotation shortcut:
  @Timed("checkout.duration")
  public Order checkout(Cart cart) { ... }

PROMETHEUS + GRAFANA STACK:
  Prometheus: pulls metrics from /actuator/prometheus every 15s
  Grafana: visualizes, creates dashboards and alerts
  Spring Boot Grafana dashboard: Import #4701 from Grafana.com
```

---

**Q3. What is distributed tracing and how do you implement it in Spring Boot?**

```
SCRATCHPAD NOTES>
PROBLEM:
  Request enters API Gateway → User Service → Order Service → Inventory Service
  Something is slow. Which service? Where in the chain?
  Logs from each service are separate — hard to correlate

DISTRIBUTED TRACING CONCEPTS:
  Trace: entire journey of one request across all services (one unique trace ID)
  Span: one unit of work within a trace (one service call, one DB query)
  Parent-child relationships between spans
  
  Request → [TraceID: abc123, SpanID: 1] UserService
                         → [TraceID: abc123, SpanID: 2] OrderService
                                        → [TraceID: abc123, SpanID: 3] DB query

SPRING BOOT 3 (Micrometer Tracing):
  Dependencies:
    micrometer-tracing-bridge-brave (or otel)
    zipkin-reporter-brave
    
  Config:
    management.tracing.sampling.probability=1.0  # 100% sampling (use 0.1 in prod)
    management.zipkin.tracing.endpoint=http://zipkin:9411/api/v2/spans

AUTOMATIC INSTRUMENTATION:
  - All HTTP requests get trace/span IDs automatically
  - RestTemplate, WebClient, Feign propagate trace context in headers
  - JDBC, Kafka also instrumented
  - TraceID added to MDC → appears in logs automatically

ACCESSING TRACE INFO:
  @Autowired Tracer tracer;
  Span span = tracer.currentSpan();
  span.tag("user.id", userId.toString());
  span.event("order.validated");

LOG CORRELATION:
  With Micrometer Tracing → traceId and spanId in log MDC
  %X{traceId} in logback pattern
  → Every log line has trace ID → search by trace ID in Kibana/Loki

BACKENDS:
  Zipkin (open source, easy setup)
  Jaeger (CNCF project, more features)
  AWS X-Ray, Datadog APM (commercial)

GOTCHA: Spring Boot 2 used Spring Cloud Sleuth (auto-configured tracing)
        Spring Boot 3 → Sleuth deprecated → use Micrometer Tracing directly
```

---

**Q4. How do you configure structured logging in Spring Boot?**

```
SCRATCHPAD NOTES:
WHY STRUCTURED LOGGING:
  Plain text logs: "User 123 logged in at 2024-01-15 10:23:45"
  Structured (JSON): {"timestamp":"2024-01-15T10:23:45Z","userId":123,"event":"login","level":"INFO"}
  
  Structured logs are searchable/filterable in ELK, Splunk, Datadog, Loki

LOGBACK SETUP (default in Spring Boot):
  Dependencies: logstash-logback-encoder
  
  logback-spring.xml:
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
          <includeMdcKeyName>traceId</includeMdcKeyName>
          <includeMdcKeyName>spanId</includeMdcKeyName>
      </encoder>
  </appender>

MDC (Mapped Diagnostic Context) — add contextual data to all logs:
  MDC.put("userId", userId.toString());
  MDC.put("requestId", UUID.randomUUID().toString());
  log.info("Processing order");  // log line includes userId, requestId automatically
  MDC.clear();  // IMPORTANT: clear at end of request (thread may be reused)
  
  Use Filter to set MDC at request entry:
  MDC.put("requestId", request.getHeader("X-Request-ID"));

SPRING BOOT 3.4+ STRUCTURED LOGGING:
  logging.structured.format.console=ecs  (Elastic Common Schema)
  logging.structured.format.file=logstash  (Logstash JSON)
  Built-in without logstash-logback-encoder!

LOG LEVELS PER COMPONENT:
  logging.level.org.hibernate.SQL=DEBUG
  logging.level.com.myapp=DEBUG
  logging.level.org.springframework.security=TRACE
  
  Change at runtime via Actuator:
  POST /actuator/loggers/com.myapp
  Body: {"configuredLevel": "DEBUG"}

GOTCHA: Don't log sensitive data (passwords, tokens, PII) — it persists in log systems
        Use @ToString(exclude = "password") in Lombok
```

---

**Q5. How do you create a custom Actuator endpoint?**

```
SCRATCHPAD NOTES:
Use case: expose custom operational data not covered by built-in endpoints
Examples: cache contents, feature flag status, circuit breaker state, custom health

IMPLEMENTATION:
  @Component
  @Endpoint(id = "features")          // accessible at /actuator/features
  public class FeatureFlagsEndpoint {
  
      @ReadOperation                  // HTTP GET
      public Map<String, Boolean> getFeatures() {
          return featureFlagService.getAllFlags();
      }
      
      @ReadOperation                  // GET with path variable: /actuator/features/payment
      public Boolean getFeature(@Selector String featureName) {
          return featureFlagService.isEnabled(featureName);
      }
      
      @WriteOperation                 // HTTP POST
      public void setFeature(@Selector String feature, boolean enabled) {
          featureFlagService.setFlag(feature, enabled);
      }
      
      @DeleteOperation                // HTTP DELETE
      public void clearFeature(@Selector String feature) {
          featureFlagService.removeFlag(feature);
      }
  }

EXPOSE THE ENDPOINT:
  management.endpoints.web.exposure.include=health,info,features

SECURING IT:
  .requestMatchers("/actuator/features/**").hasRole("ADMIN")

WEB-ONLY ENDPOINT:
  @WebEndpoint(id = "features")       // only HTTP, not JMX

JMX-ONLY ENDPOINT:
  @JmxEndpoint(id = "features")

HEALTH CONTRIBUTOR (simpler than full endpoint for health checks):
  implements HealthContributor + HealthIndicator
  Automatically included in /actuator/health response

GOTCHA: @Endpoint is framework-agnostic (works for HTTP + JMX)
        @WebEndpoint is web-only but supports Web-specific features (request headers etc.)
```

---

**Q6. What is the difference between liveness and readiness probes? How do you configure them with Actuator?**

```
SCRATCHPAD NOTES:
KUBERNETES PROBES:

LIVENESS PROBE: "Is the app alive? Should Kubernetes restart it?"
  - Checks if app is running and not deadlocked
  - If fails → Kubernetes kills and restarts the pod
  - Should ONLY fail for truly unrecoverable states
  - DON'T fail liveness for: temporary DB outage, cache miss
    (would cause restart loops — pods constantly restarting)

READINESS PROBE: "Is the app ready to receive traffic?"
  - Checks if app is ready to serve requests
  - If fails → Kubernetes removes pod from load balancer (stops sending traffic)
  - Recovers automatically when probe passes again
  - CAN fail for: DB unavailable, external service down, startup in progress

SPRING BOOT ACTUATOR INTEGRATION:
  application.yml:
  management.endpoint.health.probes.enabled=true
  management.health.livenessState.enabled=true
  management.health.readinessState.enabled=true
  
  Endpoints created:
  /actuator/health/liveness   → { "status": "UP" }
  /actuator/health/readiness  → { "status": "UP" }

KUBERNETES CONFIG:
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 60   # wait for startup
    periodSeconds: 10
    failureThreshold: 3
    
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 5

PROGRAMMATIC STATE CHANGE:
  @Autowired ApplicationContext context;
  
  // Mark as not ready (during maintenance):
  AvailabilityChangeEvent.publish(context, ReadinessState.REFUSING_TRAFFIC);
  
  // Restore:
  AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);

STARTUP PROBE (K8s):
  Separate from liveness, allows long startup before liveness kicks in
  Map to /actuator/health/liveness with longer initialDelay
```

---

**Q7. How do you implement health checks for downstream dependencies?**

```
SCRATCHPAD NOTES:
Spring Boot Auto-Configures Health Indicators For:
  - DataSource (DB connection check)
  - Redis (ping)
  - Kafka (connection check)
  - RabbitMQ
  - Disk space
  - Elasticsearch
  - MongoDB

These all contribute to /actuator/health

HEALTH RESPONSE (verbose with details):
  management.endpoint.health.show-details=always
  (or: when-authorized for production)

  {
    "status": "DOWN",
    "components": {
      "db": { "status": "UP", "details": { "database": "PostgreSQL" } },
      "redis": { "status": "DOWN", "details": { "error": "Connection refused" } }
    }
  }

CUSTOM HEALTH INDICATOR:
  @Component
  public class PaymentGatewayHealthIndicator implements HealthIndicator {
      @Override
      public Health health() {
          try {
              boolean up = paymentClient.ping();
              return Health.up()
                  .withDetail("url", paymentGatewayUrl)
                  .withDetail("latency", latencyMs + "ms")
                  .build();
          } catch (Exception e) {
              return Health.down()
                  .withDetail("error", e.getMessage())
                  .build();
          }
      }
  }

COMPOSITE HEALTH INDICATOR:
  Group multiple indicators:
  management.endpoint.health.group.external.include=paymentGateway,emailService
  → /actuator/health/external

HEALTH CONTRIBUTION TO READINESS:
  If you want DB health to affect readiness probe:
  management.health.readinessState.enabled=true
  → DB down → readiness REFUSING_TRAFFIC → K8s removes from load balancer

GOTCHA: Health checks shouldn't be expensive queries
        Use ping/simple connection test, not SELECT COUNT(*) FROM huge_table
```

---

**Q8. How do you implement correlation IDs for request tracing across services?**

```
SCRATCHPAD NOTES:
PROBLEM:
  Request enters system → passes through 5 services
  User reports error — you have timestamp, but can't find which logs belong to their request

CORRELATION ID PATTERN:
  Single UUID assigned at entry point, passed through all services in headers

IMPLEMENTATION:

1. Filter at API Gateway / entry service:
  @Component
  public class CorrelationIdFilter extends OncePerRequestFilter {
      @Override
      protected void doFilterInternal(HttpServletRequest req, 
                                       HttpServletResponse res, 
                                       FilterChain chain) throws IOException, ServletException {
          String correlationId = Optional
              .ofNullable(req.getHeader("X-Correlation-ID"))
              .orElse(UUID.randomUUID().toString());
          
          MDC.put("correlationId", correlationId);
          res.setHeader("X-Correlation-ID", correlationId);  // pass back to client
          
          try {
              chain.doFilter(req, res);
          } finally {
              MDC.remove("correlationId");  // clean up thread
          }
      }
  }

2. Propagate to outgoing requests:
  RestTemplate interceptor:
  template.getInterceptors().add((req, body, exec) -> {
      req.getHeaders().add("X-Correlation-ID", MDC.get("correlationId"));
      return exec.execute(req, body);
  });
  
  Feign interceptor:
  @Bean RequestInterceptor correlationInterceptor() {
      return template -> template.header("X-Correlation-ID", MDC.get("correlationId"));
  }

3. Add to log pattern:
  %X{correlationId} in Logback pattern

RELATIONSHIP WITH DISTRIBUTED TRACING:
  If using Micrometer Tracing → traceId serves the same purpose
  Correlation ID = human-friendly, client-facing
  Trace ID = internal, for tracing systems (Zipkin)
  They're complementary — often use both

GOTCHA: Must clear MDC after request, especially with thread pools
        Virtual threads make this cleaner but MDC cleanup is still important
```

---

**Q9. What logging best practices should you follow in a Spring Boot production application?**

```
SCRATCHPAD NOTES:
LOG LEVELS — USE CORRECTLY:
  ERROR: System errors requiring immediate attention (exceptions, data loss risk)
  WARN: Unexpected situations, app still works (retry happened, fallback used)
  INFO: Normal business events (order placed, user logged in)
  DEBUG: Diagnostic info for debugging (only in dev/staging)
  TRACE: Very detailed (SQL parameters, request/response bodies)

CONFIGURATION BY ENVIRONMENT:
  application.yml:   logging.level.com.myapp=INFO
  application-dev.yml: logging.level.com.myapp=DEBUG
  application-prod.yml: logging.level.com.myapp=WARN

WHAT TO LOG:
  ✅ Service entry/exit for important operations
  ✅ Errors with stack traces
  ✅ External service calls (success/failure)
  ✅ Business events (audit-worthy actions)
  ✅ Performance warnings (slow queries, timeouts)

WHAT NOT TO LOG:
  ❌ Passwords, API keys, tokens
  ❌ PII (names, SSNs, credit cards) — GDPR/CCPA compliance
  ❌ Request/response bodies by default (too verbose, may contain secrets)
  ❌ Full stack traces for expected errors (e.g., ResourceNotFoundException → just WARN)

PERFORMANCE:
  Use parameterized logging: log.debug("User {} logged in", userId)  NOT string concat
  Reason: String not built if DEBUG level disabled
  Use SLF4J (never log4j directly or java.util.logging)

LOG AGGREGATION TOOLS:
  ELK Stack (Elasticsearch + Logstash + Kibana)
  Grafana Loki (more lightweight)
  Splunk (enterprise)
  Datadog Logs

GOTCHA: Log rotation on disk: logback.xml → RollingFileAppender
        Don't let logs fill up the disk (seen this kill production!)
        Use async appender in production for performance
```

---

**Q10. How do you set up alerting based on application metrics?**

```
SCRATCHPAD NOTES>
ALERTING STACK (most common):
  Spring Boot Actuator → Micrometer → Prometheus → Alertmanager → PagerDuty/Slack

PROMETHEUS ALERTING RULES:
  # alert.rules.yml
  groups:
    - name: springboot
      rules:
        - alert: HighErrorRate
          expr: rate(http_server_requests_seconds_count{status="5xx"}[5m]) > 0.1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on {{ $labels.application }}"
        
        - alert: SlowResponses
          expr: histogram_quantile(0.99, http_server_requests_seconds_bucket) > 2
          for: 2m
          labels:
            severity: warning
        
        - alert: DatabaseConnectionPoolExhausted
          expr: hikaricp_connections_pending > 5
          for: 1m

GRAFANA ALERTING:
  Create alert rules directly in Grafana dashboards
  Notify via: Slack, PagerDuty, email, OpsGenie

KEY METRICS TO ALERT ON:
  Error rate: http_server_requests_seconds_count{status=~"5.."}
  Latency p99 > threshold
  JVM heap usage > 80%
  DB connection pool: pending connections > 0
  Circuit breaker: opened circuit breakers
  Kafka: consumer lag growing (messages not being processed)

SPRING BOOT APP-LEVEL ALERTS:
  Custom counter: orders.failed.total → alert if rate > threshold
  Custom gauge: checkout.queue.size → alert if > max

ALERTING BEST PRACTICES:
  - Alert on symptoms, not causes (slow response = symptom; high CPU = cause)
  - Set appropriate thresholds — avoid alert fatigue
  - Include runbook links in alerts
  - Have different severity levels (warning, critical, page)
  - Test your alerts regularly

GOTCHA: Without Alertmanager deduplication, same alert fires every evaluation
        Set 'for: Xm' to require condition persists before alerting
```

---

## 🐳 SECTION 10: Docker / Kubernetes

---

**Q1. How do you write a production-ready Dockerfile for a Spring Boot application?**

```
SCRATCHPAD NOTES:
NAIVE APPROACH (don't do this):
  FROM openjdk:17
  COPY target/app.jar app.jar
  ENTRYPOINT ["java", "-jar", "app.jar"]
  
  Problems: large image, entire JAR rebuilt if any change, run as root

PRODUCTION DOCKERFILE (multi-stage, layered):
  # Stage 1: Extract layers
  FROM eclipse-temurin:21-jre AS builder
  WORKDIR /app
  COPY target/app.jar app.jar
  RUN java -Djarmode=layertools -jar app.jar extract

  # Stage 2: Run
  FROM eclipse-temurin:21-jre
  WORKDIR /app
  
  # Don't run as root!
  RUN useradd -r -u 1001 appuser
  USER appuser
  
  # Copy layers separately (Docker cache optimization)
  COPY --from=builder /app/dependencies/ ./
  COPY --from=builder /app/spring-boot-loader/ ./
  COPY --from=builder /app/snapshot-dependencies/ ./
  COPY --from=builder /app/application/ ./
  
  # Health check
  HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1
  
  EXPOSE 8080
  ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]

WHY LAYERED JAR:
  Spring Boot JARs have layers: dependencies (rarely change) vs application code (often changes)
  Separate layers → Docker caches dependency layer → only app layer rebuilt on code change
  Much faster builds in CI/CD

JVM CONTAINER AWARENESS:
  Modern JVM (11+) is container-aware → respects cgroup memory/CPU limits
  Set: JAVA_OPTS=-Xmx512m or let JVM auto-detect: -XX:MaxRAMPercentage=75.0
  
  ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.JarLauncher"]

SMALLER IMAGES:
  Use eclipse-temurin:21-jre (not jdk — runtime only, smaller)
  Or use GraalVM native image for smallest size (but complex build)
  
GOTCHA: CMD vs ENTRYPOINT difference:
  ENTRYPOINT: always runs (can't override easily)
  CMD: default args, overridable at docker run
```

---

**Q2. How do you configure a Spring Boot app for Kubernetes deployment?**

```
SCRATCHPAD NOTES:
KEY K8S CONCEPTS FOR SPRING BOOT:

1. ConfigMap → application properties:
   kubectl create configmap app-config \
     --from-literal=SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb
   
   In deployment:
   envFrom:
     - configMapRef:
         name: app-config

2. Secret → sensitive data:
   kubectl create secret generic app-secrets \
     --from-literal=SPRING_DATASOURCE_PASSWORD=mypassword
   
   In deployment:
   env:
     - name: SPRING_DATASOURCE_PASSWORD
       valueFrom:
         secretKeyRef:
           name: app-secrets
           key: SPRING_DATASOURCE_PASSWORD

3. Liveness/Readiness probes (as discussed):
   livenessProbe:
     httpGet:
       path: /actuator/health/liveness
       port: 8080
   readinessProbe:
     httpGet:
       path: /actuator/health/readiness
       port: 8080

4. Resource limits:
   resources:
     requests:
       memory: "256Mi"
       cpu: "250m"
     limits:
       memory: "512Mi"
       cpu: "500m"
   
   JVM must respect these: -XX:MaxRAMPercentage=75.0

5. Graceful shutdown:
   server.shutdown=graceful
   spring.lifecycle.timeout-per-shutdown-phase=30s
   
   K8s deployment:
   lifecycle:
     preStop:
       exec:
         command: ["sleep", "5"]  # give load balancer time to stop sending traffic

6. Horizontal Pod Autoscaler:
   kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=70

SPRING BOOT + KUBERNETES INTEGRATION:
  spring-cloud-starter-kubernetes-client:
    - Auto-reads ConfigMaps/Secrets as Spring properties
    - Service discovery via K8s Services (no Eureka needed in K8s)
```

---

**Q3. What is Docker Compose and how do you use it for local microservices development?**

```
SCRATCHPAD NOTES:
Docker Compose: tool for running multi-container apps locally

USE CASE: Run your Spring Boot app with PostgreSQL, Redis, Kafka locally
          without installing these on your machine

BASIC COMPOSE FILE (docker-compose.yml):
  version: '3.8'
  services:
    app:
      build: .
      ports:
        - "8080:8080"
      environment:
        SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
        SPRING_DATA_REDIS_HOST: redis
        SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      depends_on:
        postgres:
          condition: service_healthy
        
    postgres:
      image: postgres:15-alpine
      environment:
        POSTGRES_DB: mydb
        POSTGRES_PASSWORD: password
      volumes:
        - postgres_data:/var/lib/postgresql/data
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U postgres"]
        interval: 5s
        timeout: 5s
        retries: 5
    
    redis:
      image: redis:7-alpine
      ports:
        - "6379:6379"
    
    kafka:
      image: confluentinc/cp-kafka:7.5.0
      environment:
        KAFKA_BROKER_ID: 1
        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      # ... (Kafka setup is verbose, reference Confluent docs)

  volumes:
    postgres_data:

COMMANDS:
  docker-compose up -d           # start all in background
  docker-compose down            # stop and remove containers
  docker-compose logs -f app     # follow app logs
  docker-compose ps              # status of all services

SPRING BOOT 3.1+ DOCKER COMPOSE SUPPORT:
  spring-boot-docker-compose dependency
  Auto-starts docker-compose.yml when app starts (dev only!)
  Auto-configures datasource/redis/etc. from running containers

GOTCHA: depends_on only waits for container START, not for service READY
        Use healthcheck + condition: service_healthy for proper ordering
```

---

**Q4. What are Kubernetes ConfigMaps and Secrets? How do you use them in Spring Boot?**

```
SCRATCHPAD NOTES>
ConfigMap:
  - Stores non-sensitive configuration as key-value pairs
  - Decouples config from container image
  - Can be mounted as env vars or files

Secret:
  - Same as ConfigMap but for sensitive data
  - Values are base64 encoded (NOT encrypted by default!)
  - For real encryption: use Sealed Secrets, Vault, or K8s encryption at rest

CREATING:
  # ConfigMap:
  kubectl create configmap app-config \
    --from-literal=APP_LOG_LEVEL=INFO \
    --from-file=application.yml=./k8s/application.yml

  # Secret:
  kubectl create secret generic app-secrets \
    --from-literal=DB_PASSWORD=secret123

USING IN DEPLOYMENT:
  # As environment variables:
  env:
    - name: APP_LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD

  # Mount entire ConfigMap as env vars:
  envFrom:
    - configMapRef:
        name: app-config

  # Mount as file (good for application.yml):
  volumes:
    - name: config-volume
      configMap:
        name: app-config
  volumeMounts:
    - mountPath: /config
      name: config-volume
  
  # Spring Boot picks it up:
  spring.config.location=file:/config/application.yml

SPRING CLOUD KUBERNETES:
  Auto-reads ConfigMaps/Secrets as Spring properties
  spring.cloud.kubernetes.config.name=app-config
  
  Dynamic refresh: ConfigMapWatcher watches for ConfigMap changes → refresh beans

GOTCHA: Secrets in K8s are only base64 encoded, not encrypted
        RBAC controls who can read Secrets
        In production: use external secret management (Vault, AWS Secrets Manager)
        with External Secrets Operator
```

---

**Q5. How do you implement graceful shutdown in a Spring Boot Kubernetes app?**

```
SCRATCHPAD NOTES:
PROBLEM WITHOUT GRACEFUL SHUTDOWN:
  K8s sends SIGTERM to pod → pod killed immediately
  In-flight requests fail with 502/503
  Active DB transactions rolled back abruptly
  Message processing interrupted

SPRING BOOT GRACEFUL SHUTDOWN:
  server.shutdown=graceful
  spring.lifecycle.timeout-per-shutdown-phase=30s  # wait up to 30s for requests to finish
  
  Behavior:
  1. SIGTERM received
  2. Spring marks readiness as REFUSING_TRAFFIC (K8s stops routing new requests)
  3. Waits for existing requests to complete (up to timeout)
  4. Closes connections, runs @PreDestroy methods
  5. JVM exits

KUBERNETES SIDE:
  terminationGracePeriodSeconds: 60  # K8s SIGKILL after 60s
  
  lifecycle:
    preStop:
      exec:
        command: ["sleep", "15"]  
        # Pause before shutdown signal — gives time for:
        # - Load balancer to remove pod from rotation
        # - In-flight requests already in the network to arrive

COMPLETE SHUTDOWN SEQUENCE:
  1. K8s marks pod as terminating → readiness probe fails
  2. Load balancer stops sending new traffic (takes ~5-10s to propagate)
  3. preStop hook runs (sleep 15s to ensure LB updated)
  4. SIGTERM sent to process
  5. Spring graceful shutdown waits for current requests
  6. Spring context closes (@PreDestroy, @EventListener ApplicationStoppingEvent)
  7. JVM exits
  8. K8s sends SIGKILL if not dead within terminationGracePeriodSeconds

IMPORTANT:
  terminationGracePeriodSeconds > preStop sleep + max request processing time

KAFKA CONSUMERS:
  @KafkaListener on graceful shutdown: finishes current message processing
  Set: spring.kafka.listener.stop-container-when-fenced=true

GOTCHA: If you don't have preStop sleep, the pod may still receive requests
        AFTER SIGTERM (LB hasn't updated yet) → requests fail
```

---

**Q6. What is a multi-stage Docker build and why is it important?**

```
SCRATCHPAD NOTES:
PROBLEM WITHOUT MULTI-STAGE:
  Single stage that builds AND runs:
  FROM maven:3.9-eclipse-temurin-21
  COPY . .
  RUN mvn package
  ENTRYPOINT ["java", "-jar", "target/app.jar"]
  
  Result: ~700MB+ image that includes Maven, source code, build tools

MULTI-STAGE BUILD:
  # Stage 1: Build (large, temporary)
  FROM maven:3.9-eclipse-temurin-21 AS build
  WORKDIR /app
  COPY pom.xml .
  RUN mvn dependency:resolve  # cache dependencies separately
  COPY src ./src
  RUN mvn package -DskipTests

  # Stage 2: Run (small, production image)
  FROM eclipse-temurin:21-jre
  COPY --from=build /app/target/app.jar app.jar
  ENTRYPOINT ["java", "-jar", "app.jar"]

BENEFITS:
  - Final image only contains JRE + JAR (no Maven, no source code)
  - Smaller image → faster pulls, smaller attack surface
  - Build tools can't be exploited in production
  - Source code not in production image

LAYER CACHING OPTIMIZATION:
  Copy pom.xml → resolve dependencies → THEN copy source
  Dependency layer cached until pom.xml changes → faster builds

LAYER SIZES (rough):
  maven:3.9-eclipse-temurin-21 → ~500MB
  eclipse-temurin:21-jre → ~200MB
  eclipse-temurin:21-jre-alpine → ~100MB
  GraalVM native image → ~60MB (no JVM at all)

GOOGLE JIB (alternative to Dockerfile):
  Maven/Gradle plugin that builds Docker images without Docker
  Optimized layer structure automatically
  No Dockerfile needed
  <plugin><groupId>com.google.cloud.tools</groupId><artifactId>jib-maven-plugin</artifactId>
```

---

**Q7. How do you handle secrets securely in a containerized Spring Boot application?**

```
SCRATCHPAD NOTES:
ANTI-PATTERNS (what not to do):
  ❌ Hardcode in application.properties → committed to git
  ❌ Bake into Docker image (ENV in Dockerfile)
  ❌ Pass as Docker run --env (visible in process list, docker inspect)
  ❌ K8s ConfigMap for sensitive values (not encrypted)

SECURE APPROACHES:

1. K8s Secrets (basic, acceptable for small teams):
   - Values encoded in etcd (encrypt etcd at rest for better security)
   - Access via RBAC
   - Mount as env vars or files

2. HashiCorp Vault (gold standard for enterprise):
   - Centralized secret management
   - Dynamic secrets (generates DB credentials per-use, auto-rotated)
   - Spring Cloud Vault:
     spring.cloud.vault.token=...
     spring.cloud.vault.kv.enabled=true
   - Secrets injected as Spring properties at startup

3. AWS Secrets Manager:
   - spring-cloud-starter-aws-secrets-manager
   - /secret/my-app/prod/db-password
   - Auto-rotation built in

4. External Secrets Operator (K8s):
   - Syncs external secret stores (Vault, AWS SM, GCP SM) to K8s Secrets
   - Best of both worlds

5. Sealed Secrets (K8s):
   - Encrypted K8s Secrets that can be committed to git
   - Only cluster can decrypt

RUNTIME INJECTION FLOW:
  CI/CD pulls secret from Vault at deploy time
  → Injects as K8s Secret
  → Pod reads as env var
  → Spring reads env var as property

GOTCHA: Secret rotation means your app needs to reload properties
        @RefreshScope + Spring Cloud Config/Vault supports this
        Or: rolling deployment with new secret value
```

---

**Q8. What is horizontal pod autoscaling (HPA) and how do you prepare a Spring Boot app for it?**

```
SCRATCHPAD NOTES:
HPA: Automatically scales pod count based on metrics (CPU, memory, custom)

HOW IT WORKS:
  HPA checks metrics every 15s
  If avg CPU > target → scale up (add pods)
  If avg CPU < target → scale down (remove pods)

SETUP:
  kubectl autoscale deployment myapp \
    --min=2 --max=10 --cpu-percent=70
    
  Or YAML:
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  spec:
    scaleTargetRef:
      name: myapp
    minReplicas: 2
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
      - type: Pods
        pods:
          metric:
            name: http_requests_per_second  # custom metric via Prometheus Adapter

SPRING BOOT REQUIREMENTS FOR HPA:

1. STATELESS: no local state, any pod can handle any request
   - Don't store sessions in memory → use Redis for session storage
   - Don't store uploaded files locally → use S3/blob storage

2. RESOURCE LIMITS SET: HPA needs limits to calculate utilization
   resources:
     requests:
       cpu: "250m"
     limits:
       cpu: "500m"

3. READINESS PROBE: new pods only get traffic when ready

4. GRACEFUL SHUTDOWN: scale-down kills pods gracefully

5. HORIZONTALLY SCALABLE CACHES: use Redis (shared) not Caffeine (local)

6. DISTRIBUTED SCHEDULING: only one instance runs scheduled jobs (ShedLock)

GOTCHA: HPA won't work without metrics-server installed in K8s cluster
        Scaling down can be slow (default cooldown: 5 minutes)
```

---

**Q9. How do you manage application configuration across multiple environments in K8s?**

```
SCRATCHPAD NOTES>
ENVIRONMENTS: dev, staging, prod

APPROACHES:

1. K8s Namespaces + environment-specific resources:
   kubectl apply -f k8s/dev/ --namespace=dev
   kubectl apply -f k8s/prod/ --namespace=prod

2. Helm Charts (templating):
   values.yaml (defaults)
   values-dev.yaml (dev overrides)
   values-prod.yaml (prod overrides)
   
   helm install myapp ./chart -f values-prod.yaml
   
   Handles: image tags, replica counts, resource limits, env-specific secrets

3. Kustomize (K8s built-in):
   base/ → common K8s resources
   overlays/dev/ → dev-specific patches
   overlays/prod/ → prod-specific patches
   
   kubectl apply -k overlays/prod/

SPRING BOOT PROFILE + K8S:
  env:
    - name: SPRING_PROFILES_ACTIVE
      value: prod  # or from ConfigMap

CONFIGURATION PRIORITY IN PRACTICE:
  Defaults in application.yml
  → Overrides in application-{profile}.yml
  → K8s ConfigMap values (higher priority as env vars)
  → K8s Secrets (highest priority env vars)
  
  Spring property priority: env vars > application.yml

WHAT VARIES BY ENVIRONMENT:
  Resource limits: dev=small, prod=large
  Replica counts: dev=1, staging=2, prod=3+
  Log levels: dev=DEBUG, prod=INFO/WARN
  DB connections: dev=small pool, prod=larger pool
  Feature flags: some features only in staging

GITOPS PATTERN:
  Config stored in Git
  ArgoCD/Flux watches Git → deploys to K8s automatically
  Secrets in Vault/External Secrets Operator → not in Git

GOTCHA: Don't use different application code per environment
        Same image, different config (12-factor principle #3)
```

---

**Q10. What is the 12-factor app methodology and how does it apply to Spring Boot?**

```
SCRATCHPAD NOTES:
12-FACTOR PRINCIPLES → SPRING BOOT APPLICATION:

1. Codebase: One codebase, multiple deploys
   → One Git repo per service, deploy same code to dev/staging/prod

2. Dependencies: Explicitly declare and isolate dependencies
   → Maven/Gradle pom.xml/build.gradle

3. Config: Store config in environment
   → No hardcoded config, use ${ENV_VAR} or application.yml with env var overrides
   → spring.datasource.url=${DATABASE_URL}

4. Backing services: Treat as attached resources
   → DB, Redis, Kafka are interchangeable resources, not special
   → Point to different DB via config only

5. Build, release, run: Strictly separate stages
   → CI builds JAR, CD creates release (JAR + config), K8s runs it

6. Processes: Execute as stateless, share-nothing processes
   → No local session storage → use Redis
   → No local file storage → use S3

7. Port binding: Export services via port binding
   → Spring Boot embedded Tomcat binds to port → no external app server needed

8. Concurrency: Scale out via process model
   → Scale by running more pods (horizontal), not bigger pods (vertical)
   → HPA

9. Disposability: Fast startup, graceful shutdown
   → server.shutdown=graceful
   → Lazy initialization for faster startup
   → GraalVM native for ultra-fast startup

10. Dev/prod parity: Keep dev, staging, prod as similar as possible
    → Docker Compose locally mirrors K8s
    → TestContainers uses same DB version as prod

11. Logs: Treat as event streams
    → Log to stdout (not files) → container runtime captures it
    → LOG_LEVEL env var to control level

12. Admin processes: Run admin/mgmt tasks as one-off processes
    → Flyway migration as startup task
    → K8s Job for one-time data migrations

GOTCHA: Factor #6 (stateless) is commonly violated with local caching or in-memory sessions
```

---

## 🎭 SECTION 11: Behavioral & Scenario-Based Questions

---

**Q1. Tell me about a time you debugged a difficult production issue in a Spring Boot app.**

```
SCRATCHPAD NOTES (STAR Framework):

STRUCTURE YOUR STORY:
  S — Situation: what was the system, what was the impact?
  T — Task: what was your role?
  A — Action: what did YOU specifically do? (most important part)
  R — Result: measurable outcome

SAMPLE STORY FRAMEWORK:
  Situation: "We had an e-commerce API that started returning 500 errors intermittently 
              under heavy load. ~5% of checkout requests failing, costing revenue."
  
  Task: "I was the on-call engineer and lead the debugging effort."
  
  Action (most important — be specific):
    1. "Checked Grafana dashboards → saw DB connection pool exhausted 
        (hikaricp.connections.pending > 0)"
    2. "Enabled SQL logging for affected users → found queries running 10x slower than normal"
    3. "Used EXPLAIN ANALYZE → discovered a missing index on orders.customer_id"
       (recently added search feature had triggered full table scans)
    4. "Immediate fix: added index in Flyway migration, deployed"
    5. "Long-term: added @Timed on checkout flow, created Prometheus alert for 
        connection pool exhaustion"
  
  Result: "Error rate dropped to 0.01%, checkout latency improved by 70%, 
           added monitoring to prevent recurrence"

KEY THINGS TO MENTION:
  - How you identified the problem systematically (not by guessing)
  - Collaboration with team
  - The fix AND the follow-up to prevent recurrence
  - What you learned

GOTCHA: Be specific — vague stories fail
        "We had a bug and fixed it" → won't get you far
        Show your debugging methodology
```

---

**Q2. Tell me about a performance optimization you implemented.**

```
SCRATCHPAD NOTES:

STORY FRAMEWORK:
  - Context: what was the baseline performance problem?
  - How did you MEASURE it first?
  - What did you find?
  - What did you change?
  - What was the measured improvement?

COMMON SCENARIOS TO DRAW FROM:

Scenario A — N+1 Fix:
  "Product listing API was taking 8 seconds for page of 20 products.
   Enabled SQL logging → saw 41 queries (1 + 20 product queries + 20 category queries)
   Fixed with JOIN FETCH + @EntityGraph → 1 query → down to 200ms"

Scenario B — Cache Addition:
  "User permissions check on every API call hitting DB each time.
   Permissions rarely change. Added @Cacheable with Redis, 10-minute TTL.
   Removed 50,000 DB queries/hour, DB CPU dropped 30%."

Scenario C — Async Processing:
  "Order confirmation email sent synchronously in checkout flow.
   Email service sometimes slow (2-3s). Added @Async for email sending.
   Checkout response time dropped from 3.5s to 400ms."

KEY METRICS TO MENTION (makes it credible):
  - Before/after response times
  - Query count before/after
  - Error rate before/after
  - Database CPU/load change

WHAT TO EMPHASIZE:
  - You measured before optimizing (not guessing)
  - You understood the root cause (not random fixes)
  - You verified the improvement (measured after too)
  - No negative side effects (tested thoroughly)
```

---

**Q3. How would you design a system to handle 10 million users?**

```
SCRATCHPAD NOTES:

CLARIFY FIRST (always ask these):
  - 10M concurrent or 10M total registered?
  - Read-heavy or write-heavy?
  - Geographic distribution?
  - Latency requirements?
  - What features?

GENERAL SCALABILITY LAYERS TO DISCUSS:

1. Load Balancing:
   - Multiple app instances behind load balancer (AWS ALB, Nginx)
   - Horizontal scaling (more pods in K8s)

2. Database:
   - Read replicas for read-heavy workloads
   - Connection pooling (HikariCP, PgBouncer)
   - Database sharding or partitioning for massive data
   - Consider NoSQL for certain access patterns

3. Caching:
   - Redis cluster for distributed caching
   - CDN for static assets + API response caching
   - Application-level caching (@Cacheable)

4. Async Processing:
   - Message queues (Kafka) for heavy operations
   - Don't make users wait for non-essential processing

5. Microservices:
   - Split by domain → scale hot services independently

6. API Design:
   - Pagination everywhere
   - GraphQL or sparse fieldsets to reduce data transfer
   - Efficient serialization

7. Infrastructure:
   - K8s with HPA
   - Multi-region deployment for global users
   - CDN at edge

SPRING BOOT SPECIFIC ANSWERS:
  "Server-side: virtual threads for high concurrency"
  "Caching hot data with Redis"
  "Event-driven with Kafka for decoupling"
  "Circuit breakers for resilience"

GOTCHA: Don't over-engineer. Start with: "I'd first understand the actual bottleneck.
         10M users doesn't mean 10M concurrent — most apps have much lower concurrency."
```

---

**Q4. How would you introduce testing into a legacy Spring Boot codebase with no tests?**

```
SCRATCHPAD NOTES:

DON'T SAY: "I'd write tests for everything immediately" (unrealistic)

REALISTIC APPROACH — prioritize strategically:

PHASE 1 — Establish Foundation:
  - Add testing dependencies (JUnit 5, Mockito, AssertJ)
  - Set up code coverage baseline (even if 0% — measure it first)
  - Agree with team on coverage targets and test standards

PHASE 2 — Test New Code Immediately:
  - Every new feature/bugfix gets tests (prevent regression)
  - "Test gate": no PR merged without tests for changed code
  - Boy Scout Rule: leave code better than you found it

PHASE 3 — Strategic Retroactive Testing:
  - Start with most critical business logic (not UI, not boilerplate)
  - Test areas with most bugs/incidents first
  - High-risk paths: payment processing, auth, data calculations
  - Unit test services (pure business logic — easiest to test)

PHASE 4 — Integration Tests for Critical Paths:
  - Happy path for top 5 user journeys
  - @SpringBootTest for smoke tests

TACTICS FOR LEGACY CODE:
  - Extract methods to make logic testable (tiny refactors before testing)
  - Dependency injection instead of direct instantiation (makes mocking possible)
  - Seams: identify where to inject mocks
  - Don't refactor AND test at same time — risky

TOOLING:
  - JaCoCo for coverage reports
  - Mutation testing to find weak tests
  - Sonar for code quality + coverage enforcement

REAL ANSWER: "I've done this. The key is: don't try to test everything at once.
              Write tests for ALL new work, and incrementally add tests to 
              high-risk legacy code areas when touching them."
```

---

**Q5. Tell me about a significant architectural decision you made. What were the trade-offs?**

```
SCRATCHPAD NOTES:

GOOD ARCHITECTURAL DECISIONS TO TALK ABOUT:
  - Choosing Kafka over REST for inter-service communication
  - Adding an API Gateway
  - Moving from monolith to microservices (or deciding NOT to)
  - Choosing reactive (WebFlux) vs imperative (MVC)
  - Adding a caching layer
  - Database choice (SQL vs NoSQL)

STORY STRUCTURE:
  Context → Problem/Requirement → Options Considered → Decision → Trade-offs → Outcome

EXAMPLE STORY:
  "We needed to add real-time notifications to our e-commerce platform.
   Three options: polling from frontend, WebSockets, or Server-Sent Events.
   
   Evaluated:
   - Polling: simple but inefficient (constant requests)
   - WebSockets: bidirectional, but we only needed server→client
   - SSE: simpler, HTTP-based, unidirectional (fit our use case)
   
   Chose SSE because: simpler infrastructure, works through proxies,
   fits our one-way notification pattern.
   
   Trade-off: Limited browser connection limit per domain (6 concurrent).
   Fixed with: HTTP/2 (multiplexing) and connection consolidation.
   
   Result: Notifications working with <100ms latency, no polling overhead."

WHAT INTERVIEWERS WANT TO SEE:
  - You considered multiple options (not just picked the first idea)
  - You understood and articulated trade-offs
  - You made a DECISION (not analysis paralysis)
  - You learned from the outcome (if it didn't work perfectly)

GOTCHA: Don't present it as perfect. Real decisions have trade-offs.
        "In hindsight, X was the right choice, but Y would have been better if..."
        shows growth and honesty.
```

---

**Q6. Our Spring Boot application is showing memory leaks in production. Walk me through how you'd diagnose it.**

```
SCRATCHPAD NOTES:

STEP 1 — CONFIRM IT'S A LEAK (not just high usage):
  - Heap usage grows steadily over time (doesn't plateau)
  - GC can't reclaim memory (Old Gen keeps growing after full GC)
  - Check: Actuator /metrics/jvm.memory.used over time
  - Or: Prometheus + Grafana graph over 24-48 hours

STEP 2 — GATHER DATA (don't guess yet):
  Option A: Heap dump (if app still running):
    jmap -dump:format=b,file=heap.hprof <pid>
    Or auto-dump: -XX:+HeapDumpOnOutOfMemoryError
  
  Option B: Java Flight Recorder (less disruptive):
    jcmd <pid> JFR.start duration=300s filename=recording.jfr settings=profile
  
  Option C: Enable GC logging:
    -Xlog:gc*:file=/tmp/gc.log  (Java 11+)

STEP 3 — ANALYZE HEAP DUMP:
  Tool: Eclipse MAT (Memory Analyzer Tool)
  Look for:
  - Dominator tree: what objects retain most heap?
  - Leak Suspects Report (automatic in MAT)
  - Retained heap of top objects

COMMON SPRING BOOT LEAK CAUSES:
  1. Unbounded cache: @Cacheable without eviction policy, Map growing forever
     FIX: Add Caffeine/Redis cache with TTL and max size
  
  2. Static collection growing: static List/Map that's never cleared
     FIX: Review static fields, add bounded queue
  
  3. Event listener not unregistered:
     context.addApplicationListener(listener) → listener holds references
     FIX: Implement DisposableBean, unregister in destroy()
  
  4. ThreadLocal not cleaned:
     MDC.put() without MDC.clear() → thread pool thread holds reference forever
     FIX: Always clean in finally block
  
  5. Hibernate session/entity accumulation:
     Long transaction with many entities loaded → all in 1st level cache
     FIX: session.clear() periodically, or batch processing

STEP 4 — FIX AND VERIFY:
  Deploy fix → monitor heap over same time period → confirm plateau

GOTCHA: "I'd add monitoring BEFORE the next incident — 
         Prometheus + Grafana heap dashboard with alert at 80% heap usage"
```

---

**Q7. How would you handle a situation where two microservices need to maintain data consistency?**

```
SCRATCHPAD NOTES:

FIRST — establish what kind of consistency is needed:
  Strong consistency: all reads see latest write immediately
  Eventual consistency: reads may see stale data temporarily but eventually converge
  
  "In most microservices, eventual consistency is acceptable and much simpler"

APPROACHES:

A) Saga Pattern (recommended for most cases):
  Choreography: each service reacts to events from others
  Orchestration: central orchestrator coordinates
  → Handles failure with compensating transactions
  → Gives up strong consistency for availability

B) Two-Phase Commit (2PC) — usually avoid:
  Coordinator asks all services to prepare, then commit
  Problems: blocking protocol, single point of failure
  "In practice, 2PC across microservices is an anti-pattern"

C) Outbox Pattern + Event Sourcing:
  Write to local DB + outbox table atomically
  Outbox processor publishes events
  Downstream services process events → eventually consistent

D) Shared Database (last resort, anti-pattern):
  Both services use the same DB
  Breaks service independence, tight coupling

REAL EXAMPLE ANSWER:
  "In our order service: when an order was created, we needed to:
   1. Save order in orders DB
   2. Reserve inventory in inventory service
   3. Create payment in payment service
   
   We used a choreography saga:
   - Order created → OrderPlaced event → Kafka
   - Inventory service consumes → reserves stock → InventoryReserved event
   - Payment service consumes → charges → PaymentProcessed event
   - If payment fails → PaymentFailed event → inventory service releases stock
   
   Compensating transactions handled the rollback scenario."

GOTCHA: "There's no perfect solution — you're trading consistency for availability 
         and simplicity. The right choice depends on business requirements."
```

---

**Q8. Tell me about a time you had to convince your team to adopt a technical practice.**

```
SCRATCHPAD NOTES:

EXAMPLES OF GOOD SCENARIOS:
  - Convincing team to write unit tests
  - Introducing code review practices
  - Migrating from one framework/tool to another
  - Adopting CI/CD
  - Introducing contract testing for APIs
  - Introducing monitoring/observability

STORY STRUCTURE (STAR):
  Situation: what was the problem / what was missing?
  Task: what was your role? (not just "I thought it was a good idea")
  Action: how did you convince? (this is the key part)
  Result: what changed?

KEY ELEMENTS FOR ACTION:
  1. Empathy first: "I understood why the team hadn't done it yet" 
     (time pressure, unclear ROI, habit)
  2. Show ROI: concrete numbers, not abstract "it's a best practice"
  3. Start small: spike, proof of concept, one service
  4. Lead by example: do it yourself first
  5. Make it easy: provide tools, templates, documentation

EXAMPLE:
  "The team was writing manual SQL migrations in ad-hoc scripts.
   Proposed Flyway. Some pushback: 'adds complexity'.
   
   I created a proof of concept in one sprint:
   - Showed how it prevented 'which script did you run?' questions
   - Showed CI/CD failing fast on migration issues
   - Referenced production incident 6 months prior caused by migration order bug
   
   Team agreed to pilot it in one service. After 2 sprints — no complaints.
   Adopted team-wide 3 months later."

WHAT INTERVIEWERS LOOK FOR:
  - Collaboration, not forcing your ideas
  - Communication skills
  - Ability to influence without authority
  - Long-term thinking (not just "I was right")
```

---

**Q9. How do you stay up to date with Spring Boot and Java ecosystem changes?**

```
SCRATCHPAD NOTES:

HONEST + STRUCTURED ANSWER:

PRIMARY SOURCES:
  - spring.io/blog — official Spring releases, what's new
  - Java release notes (JEP list for new Java features)
  - GitHub spring-projects/spring-boot — release notes, issues

COMMUNITY + AGGREGATORS:
  - InfoQ (good for architecture articles)
  - Baeldung (practical Spring tutorials)
  - DZone / Java Weekly (Tomasz Nurkiewicz)
  - JVM Weekly newsletter

PRACTICAL LEARNING:
  - Upgrade to new Spring Boot version in personal projects
  - Try new features in side projects before production
  - Read release notes when upgrading (especially for major versions)

CONFERENCES + TALKS:
  - Spring I/O (YouTube)
  - Devoxx (YouTube)
  - JFokus

SOCIAL:
  - Follow Spring team on Twitter/X (@springframework)
  - LinkedIn connections in Java/Spring space

FOR THIS INTERVIEW — SHOW YOU'RE CURRENT:
  Mention something specific recently:
  "I've been looking into Spring Boot 3.2's virtual thread support with Java 21 —
   the simplification over reactive programming is interesting for our use case"
  
  OR "Following the Spring Authorization Server project as we're considering
      moving off our custom auth to a standard OAuth2 server"

GOTCHA: Don't say "I read everything" (not believable)
        Pick 2-3 sources you actually use and mention a recent specific thing you learned
```

---

**Q10. Where do you see areas for improvement in your current/last Spring Boot project?**

```
SCRATCHPAD NOTES:

WHY THEY ASK THIS:
  - Self-awareness and growth mindset
  - Understanding of what "good" looks like
  - Honesty (everyone has areas for improvement)

GOOD AREAS TO MENTION (pick real ones):

Technical Improvements:
  - "Test coverage is around 40% — would benefit from more service layer unit tests"
  - "We don't have distributed tracing — debugging cross-service issues is hard"
  - "Still using RestTemplate in some places, would modernize to WebClient"
  - "No circuit breakers on external API calls — a risk if they go down"
  - "DB queries aren't optimized — some N+1 issues we haven't addressed"

Architectural Improvements:
  - "Tight coupling between some services — shared DTO library creates versioning challenges"
  - "No API versioning strategy — breaking changes are risky"
  - "Config management is inconsistent — some hardcoded, some externalized"

Process Improvements:
  - "Could use better observability — we find out about issues from users, not monitoring"
  - "No automated performance testing in CI pipeline"

HOW TO FRAME IT:
  1. Identify the gap honestly
  2. Explain WHY it's a gap (trade-offs, time pressure, history)
  3. State what BETTER would look like
  4. Optional: mention steps you're taking or have proposed

GOTCHA: Don't say "nothing — everything is perfect" → unrealistic, raises red flags
        Don't throw teammates under the bus → "the team made bad decisions"
        Frame as SYSTEM/PROCESS issues, not personal blame
        
BONUS: Turn it into a strength:
  "I've already proposed adding TestContainers to our integration tests
   and got buy-in from my tech lead — planning to start next sprint"
```

---

> **💡 How to Use These Notes:** These are scratchpad guides, not scripts. Read each section the night before you focus on that topic. Practice answering out loud — fluency comes from speaking, not reading. Add your own real examples to the behavioral questions — authenticity is what separates good answers from great ones.