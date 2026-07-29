# Spring Boot Interview Roadmap — Mid-Level Role (5 YOE) | 1 Month Plan

---
## 📋 Overview & Expectations for Mid-Level Role

At 5 years of experience, interviewers expect you to go **beyond syntax**. You'll be tested on:
- **Why** Spring Boot does what it does (not just how)
- Architecture decisions and trade-offs
- Real production scenarios (failures, performance, security)
- Microservices design and integration patterns
- Code quality, testing, and observability

---
## 📅 Week 1 — Core Spring Boot Foundations (Solidify the Base)

> Goal: Refresh core concepts with depth. You likely know these — focus on *explaining them clearly* and knowing the internals.

### Day 1–2: Spring Core & IoC / DI
| Topic                     | What to Know                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------- |
| IoC Container             | How `ApplicationContext` manages beans lifecycle                                                        |
| Dependency Injection      | `@Autowired`, constructor vs. setter vs. field injection and **why constructor injection is preferred** |
| Bean Scopes               | Singleton, Prototype, Request, Session — implications of each                                           |
| Bean Lifecycle            | `@PostConstruct`, `@PreDestroy`, `InitializingBean`, `DisposableBean`                                   |
| `@Component` vs `@Bean`   | When to use each; `@Configuration` internals                                                            |
| `@Qualifier` & `@Primary` | Resolving ambiguity                                                                                     |
**Practice question:** *"Explain what happens when Spring Boot starts up — from `main()` to a running application."*

---
### Day 3–4: Auto-Configuration & Spring Boot Internals
| Topic                                              | What to Know                                                                                   |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `@SpringBootApplication`                           | It's `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`                          |
| Auto-configuration mechanism                       | `spring.factories` / `AutoConfiguration.imports` (Spring Boot 3.x), `@Conditional` annotations |
| `@ConditionalOnClass`, `@ConditionalOnMissingBean` | How Spring decides what to configure                                                           |
| `application.properties` vs `application.yml`      | Profile-specific configs (`application-dev.yml`)                                               |
| `@ConfigurationProperties` vs `@Value`             | Type-safe binding, when to use each                                                            |
| Externalized Configuration                         | Priority order (env vars > properties file > defaults)                                         |
| Embedded Server                                    | Tomcat/Jetty/Undertow; how to switch; tuning thread pool                                       |
**Practice question:** *"How would you create a custom Spring Boot starter?"*

---
### Day 5–6: Spring MVC & REST APIs
| Topic | What to Know |
|---|---|
| `DispatcherServlet` | How it routes requests end-to-end |
| `@RestController` vs `@Controller` | Difference; when `@ResponseBody` is needed |
| `@RequestMapping` variants | `@GetMapping`, `@PostMapping`, etc. |
| Request handling | `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` |
| `ResponseEntity<T>` | Returning status codes, headers, body |
| Exception Handling | `@ControllerAdvice`, `@ExceptionHandler`, `ProblemDetail` (Spring 6) |
| Validation | `@Valid`, `@Validated`, Bean Validation (JSR-380), custom validators |
| Content Negotiation | JSON/XML, `produces`/`consumes` |

---
### Day 7: Review + Practice
- Write a REST API from scratch (no IDE assistance) with proper exception handling and validation
- Explain the full request lifecycle out loud

---

## 📅 Week 2 — Data Layer, Security & Testing

> Goal: These are high-frequency interview areas. Depth matters here.

### Day 8–9: Spring Data JPA & Database
| Topic | What to Know |
|---|---|
| JPA vs Hibernate vs Spring Data JPA | Layered relationship |
| `@Entity`, `@Table`, `@Id`, `@GeneratedValue` | Mapping strategies |
| Repository hierarchy | `CrudRepository` → `JpaRepository` → `PagingAndSortingRepository` |
| Query methods | Derived query names, `@Query` (JPQL & native), `@NamedQuery` |
| `@Transactional` | Propagation types (REQUIRED, REQUIRES_NEW, NESTED), isolation levels, rollback rules |
| N+1 Problem | What it is, `@EntityGraph`, `JOIN FETCH`, `FetchType.LAZY` vs `EAGER` |
| Pagination & Sorting | `Pageable`, `Page<T>`, `Slice<T>` |
| Projections | Interface-based, class-based (DTO projections) |
| Database migrations | Flyway vs Liquibase — when and why |
| Connection Pooling | HikariCP configuration and tuning |

**Practice question:** *"How do you avoid the N+1 problem in a Spring Data JPA application?"*

---

### Day 10–11: Spring Security
| Topic | What to Know |
|---|---|
| Security Filter Chain | How filters work, `SecurityFilterChain` bean (Spring Security 6) |
| Authentication vs Authorization | Core difference; `AuthenticationManager`, `UserDetailsService` |
| JWT Authentication | Stateless auth flow — token generation, validation filter, refresh tokens |
| OAuth2 / OIDC | Resource server, authorization server, `@EnableResourceServer` (deprecated) vs modern config |
| Method-level security | `@PreAuthorize`, `@PostAuthorize`, `@Secured` |
| CSRF | When to disable it (stateless APIs) and why |
| Password encoding | `BCryptPasswordEncoder`, `PasswordEncoder` contract |
| CORS | `@CrossOrigin` vs global CORS config |

**Practice question:** *"Walk me through implementing JWT-based authentication in a Spring Boot app."*

---

### Day 12–13: Testing in Spring Boot
| Topic | What to Know |
|---|---|
| Testing pyramid | Unit → Integration → E2E |
| JUnit 5 | `@Test`, `@BeforeEach`, `@AfterEach`, `@ParameterizedTest` |
| Mockito | `@Mock`, `@InjectMocks`, `@Spy`, `when().thenReturn()`, `verify()` |
| `@SpringBootTest` | Full context load; `webEnvironment` options |
| `@WebMvcTest` | Slice test for controllers; `MockMvc` |
| `@DataJpaTest` | Slice test for JPA; uses in-memory H2 by default |
| `@MockBean` vs `@Mock` | Key difference — `@MockBean` replaces bean in Spring context |
| TestContainers | Real database testing in integration tests |
| `@TestConfiguration` | Providing test-specific beans |

**Practice question:** *"How would you test a service that calls an external API?"*

---

### Day 14: Review + Coding Practice
- Write unit tests + integration tests for a sample CRUD application
- Mock an external REST call using `MockRestServiceServer` or WireMock

---

## 📅 Week 3 — Microservices, Messaging & Observability

> Goal: Mid-level engineers are expected to understand distributed system patterns.

### Day 15–16: Microservices Patterns & Spring Cloud
| Topic | What to Know |
|---|---|
| Microservices vs Monolith | Trade-offs, when to choose each |
| Service Discovery | Eureka (`@EnableDiscoveryClient`), Consul |
| API Gateway | Spring Cloud Gateway (not Zuul) — routing, filters, rate limiting |
| Load Balancing | Client-side (Spring Cloud LoadBalancer), `@LoadBalanced` RestTemplate |
| Config Server | Spring Cloud Config — centralized config, refresh with `@RefreshScope` |
| Circuit Breaker | Resilience4j — `@CircuitBreaker`, `@Retry`, `@RateLimiter`, fallback methods |
| Feign Client | Declarative HTTP client, `@FeignClient`, error decoder |
| Inter-service communication | Sync (REST, gRPC) vs Async (messaging) — trade-offs |

---

### Day 17–18: Messaging & Async Processing
| Topic | What to Know |
|---|---|
| Spring Kafka | Producer/Consumer setup, `@KafkaListener`, `KafkaTemplate`, consumer groups |
| Spring RabbitMQ | `@RabbitListener`, exchanges, queues, routing keys, dead-letter queues |
| `@Async` | Thread pool configuration, return `CompletableFuture<T>` |
| `@Scheduled` | Fixed rate vs fixed delay, cron expressions |
| Event-Driven Architecture | `ApplicationEventPublisher`, `@EventListener`, `@TransactionalEventListener` |
| Outbox Pattern | Solving dual-write problem in microservices |

---

### Day 19–20: Caching, Performance & Observability
| Topic | What to Know |
|---|---|
| Spring Cache Abstraction | `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` |
| Cache providers | Redis (most common), Caffeine (in-memory) |
| Redis | `RedisTemplate`, `StringRedisTemplate`, TTL, cache-aside pattern |
| Actuator | `/health`, `/metrics`, `/info`, `/env`, custom endpoints, security |
| Micrometer | Metrics collection, integration with Prometheus |
| Distributed Tracing | Spring Cloud Sleuth (Boot 2) / Micrometer Tracing (Boot 3), Zipkin, trace IDs in logs |
| Logging | Structured logging (Logback, Log4j2), MDC for trace context, log levels per environment |
| Performance tuning | JVM tuning basics, HikariCP pool size, async processing, lazy loading |

---

### Day 21: Review + System Design
- Design a simple microservices system (e.g., an e-commerce order service) on paper
- Include: API Gateway, Auth, Service Discovery, Async messaging, Circuit Breaker

---

## 📅 Week 4 — Advanced Topics, Real Scenarios & Mock Interviews

> Goal: Bridge knowledge to real-world problem-solving. This is what differentiates mid-level candidates.

### Day 22–23: Advanced Spring Boot Topics
| Topic | What to Know |
|---|---|
| Spring Boot 3.x changes | Jakarta EE namespace, native images (GraalVM), `ProblemDetail`, virtual threads (Java 21) |
| `@Transactional` deep dive | Proxy-based; why self-invocation doesn't work; `TransactionSynchronizationManager` |
| Custom Auto-configuration | `@AutoConfiguration`, `AutoConfiguration.imports`, custom starters |
| AOP | `@Aspect`, `@Before`, `@After`, `@Around`, `@AfterReturning`, pointcut expressions |
| Application Events | Startup events (`ApplicationReadyEvent`, `ContextRefreshedEvent`) |
| Profiles | `@Profile`, activating profiles, profile-specific beans |
| Reactive Programming (basic) | Spring WebFlux, `Mono<T>`, `Flux<T>`, when to use reactive vs imperative |

---

### Day 24–25: Containerization, CI/CD & Cloud Basics
| Topic | What to Know |
|---|---|
| Docker | Writing a `Dockerfile` for Spring Boot, multi-stage builds, layered JARs |
| Docker Compose | Local microservices setup with databases and message brokers |
| Kubernetes basics | Pods, Services, Deployments, ConfigMaps, Secrets, health probes (liveness/readiness) |
| Cloud deployment | Deploying to AWS (ECS/EKS), Azure, or GCP — high-level awareness |
| Health probes | Mapping Actuator endpoints to K8s liveness/readiness probes |
| 12-factor app | Configuration, statelessness, logging — principles relevant to Spring Boot |

---

### Day 26–27: Behavioral + Scenario-Based Preparation
Prepare detailed STAR stories for these scenarios:
- A production bug you debugged in a Spring Boot app
- A performance issue you identified and resolved
- A design decision you made and why (e.g., sync vs async, monolith vs microservices)
- How you handled a security vulnerability
- A time you introduced testing into a legacy codebase
- How you worked with a team to agree on an architecture decision

**Common scenario-based technical questions to prepare:**
- "Our Spring Boot app is slow — how do you investigate and fix it?"
- "How would you migrate a monolith to microservices?"
- "Your service keeps throwing `LazyInitializationException` in production — what do you do?"
- "How do you ensure consistency between two microservices in a transaction?"

---

### Day 28–29: Mock Interviews + Gap Filling
- Do 2 full mock interviews (use a friend, mentor, or platforms like Pramp/Interviewing.io)
- Revisit your weakest areas from Weeks 1–3
- Practice explaining complex topics in 2 minutes (elevator pitch style)
- Review common Spring Boot GitHub issues and Stack Overflow questions

---

### Day 30: Light Review + Rest
- Skim your notes/checklist
- Prepare your own questions to ask the interviewer
- Rest — mental clarity matters

---

## 🔑 High-Frequency Interview Questions (Quick Reference)

| Category | Must-Know Questions |
|---|---|
| Core | How does Spring Boot auto-configuration work? |
| Core | What is the difference between `@Component`, `@Service`, `@Repository`? |
| Data | How does `@Transactional` work internally (proxy)? |
| Data | How do you handle the N+1 problem? |
| Security | How do you implement JWT in Spring Boot? |
| Microservices | How do you handle distributed transactions? (Saga pattern) |
| Testing | What is the difference between `@Mock` and `@MockBean`? |
| Performance | How would you profile and optimize a slow API? |
| Patterns | Explain Circuit Breaker and when you'd use it |
| Design | How would you design a rate-limited public API? |

---

---

# ✅ Printable Interview Preparation Checklist

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

---

## 📚 Recommended Resources

| Resource | Use For |
|---|---|
| **Official Spring Docs** (docs.spring.io) | Authoritative reference |
| **Baeldung** (baeldung.com) | Practical how-to articles |
| **Spring Boot in Action** (Craig Walls) | Conceptual depth |
| **Java Brains / Amigoscode** (YouTube) | Visual learning |
| **GitHub: spring-projects/spring-boot** | Reading actual source code |
| **LeetCode / HackerRank** | If coding rounds are expected |

---

> **💡 Key Mindset Tip:** For a mid-level role, interviewers are evaluating *judgment*, not just knowledge. Always frame answers with trade-offs: *"I chose X over Y because in our use case, Z mattered most."* That signals engineering maturity.