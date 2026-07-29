# 🛒 "ShopWave" — Distributed E-Commerce Backend Platform

> A single project that naturally touches every Spring Boot concept, tool, and pattern from the roadmap. E-commerce is chosen deliberately — the domain is universally understood, and every feature maps cleanly to a real engineering challenge.

---

## 🎯 Project Overview

**ShopWave** is a production-grade e-commerce backend that handles:
- User registration, authentication, and profiles
- Product catalog with search and filtering
- Shopping cart and order placement
- Payment processing with failure handling
- Real-time order status notifications
- Admin dashboard APIs

> **Why this project?** Every concept has a *natural reason to exist here*. JWT isn't forced in — users need to log in. Kafka isn't forced in — order events genuinely need async processing. Circuit breakers aren't forced in — payment services genuinely fail. This makes learning stick.

---

## 🏗️ Full System Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           CLIENT APPLICATIONS            │
                        │   (Mobile App / Web Frontend / Postman)  │
                        └──────────────────┬──────────────────────┘
                                           │ HTTPS
                                           ▼
                        ┌─────────────────────────────────────────┐
                        │           API GATEWAY                    │
                        │      (Spring Cloud Gateway)              │
                        │  • JWT Validation Filter                 │
                        │  • Rate Limiting (Redis)                 │
                        │  • Request Routing                       │
                        │  • CORS Configuration                    │
                        │  • Circuit Breaker (Resilience4j)        │
                        └──────┬──────┬──────┬──────┬─────────────┘
                               │      │      │      │
              ┌────────────────┘      │      │      └─────────────────┐
              │                       │      │                         │
              ▼                       ▼      ▼                         ▼
   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │   USER SERVICE   │  │ PRODUCT SERVICE  │  │  ORDER SERVICE   │  │ PAYMENT SERVICE  │
   │                  │  │                  │  │                  │  │                  │
   │ • Registration   │  │ • Catalog CRUD   │  │ • Place Order    │  │ • Process Payment│
   │ • JWT Auth       │  │ • Search/Filter  │  │ • Order History  │  │ • Refunds        │
   │ • Profiles       │  │ • Stock Mgmt     │  │ • Saga Orchestr. │  │ • Fraud Check    │
   │ • Roles (RBAC)   │  │ • Redis Cache    │  │ • Status Track   │  │ • Circuit Breaker│
   │                  │  │                  │  │                  │  │                  │
   │ PostgreSQL (own) │  │ PostgreSQL (own) │  │ PostgreSQL (own) │  │ PostgreSQL (own) │
   └──────────────────┘  └──────────────────┘  └────────┬─────────┘  └──────────────────┘
                                                          │
                                      ┌───────────────────┘
                                      │  Publishes Domain Events
                                      ▼
                        ┌─────────────────────────────────────────┐
                        │           APACHE KAFKA                   │
                        │                                          │
                        │  Topics:                                 │
                        │  • order.placed                          │
                        │  • order.confirmed                       │
                        │  • order.cancelled                       │
                        │  • payment.processed                     │
                        │  • payment.failed                        │
                        │  • inventory.reserved                    │
                        └──────────────────┬──────────────────────┘
                                           │
                              ┌────────────┘
                              │  Consumes Events
                              ▼
                   ┌──────────────────────┐
                   │ NOTIFICATION SERVICE │
                   │                      │
                   │ • Email (JavaMail)   │
                   │ • Push Notifications │
                   │ • Order Updates      │
                   │ • @KafkaListener     │
                   └──────────────────────┘

   ┌─────────────────────────────────────────────────────────────────────┐
   │                     INFRASTRUCTURE SERVICES                          │
   │                                                                       │
   │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
   │  │  CONFIG SERVER   │  │DISCOVERY SERVER  │  │     REDIS        │  │
   │  │ (Spring Cloud    │  │    (Eureka)       │  │ • Cache Layer    │  │
   │  │  Config + Git)   │  │ Service Registry  │  │ • Rate Limiting  │  │
   │  └──────────────────┘  └──────────────────┘  │ • Session Store  │  │
   │                                               └──────────────────┘  │
   │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
   │  │     ZIPKIN       │  │   PROMETHEUS     │  │    GRAFANA       │  │
   │  │ Distributed      │  │ Metrics Scraping │  │  Dashboards &    │  │
   │  │ Tracing          │  │                  │  │  Alerting        │  │
   │  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Service Breakdown

### Service 1: User Service
```
Concepts Covered:
├── Spring Security 6 (SecurityFilterChain)
├── JWT token generation + validation
├── BCryptPasswordEncoder
├── @PreAuthorize / method-level security
├── Spring Data JPA (User, Role entities)
├── @Transactional (registration flow)
├── Bean Validation (@Valid, custom validators)
├── @ControllerAdvice (exception handling)
├── @WebMvcTest + @DataJpaTest (testing)
└── Flyway migrations (user tables)

Key APIs:
POST   /api/auth/register
POST   /api/auth/login        → returns JWT
POST   /api/auth/refresh
GET    /api/users/me
PUT    /api/users/me
GET    /api/admin/users       → ROLE_ADMIN only
```

---

### Service 2: Product Service
```
Concepts Covered:
├── Spring Data JPA (Product, Category, Inventory)
├── @Cacheable / @CacheEvict with Redis
├── Custom @Query (JPQL + native)
├── DTO Projections (ProductSummaryDto)
├── Pagination + Sorting (Pageable)
├── Spring Data Specifications (dynamic filtering)
├── @Transactional (stock updates - optimistic locking)
├── @Version (optimistic locking on Inventory)
├── @DataJpaTest with TestContainers
└── Flyway migrations (product tables)

Key APIs:
GET    /api/products?category=X&minPrice=Y&sort=price,asc
GET    /api/products/{id}
POST   /api/admin/products    → ROLE_ADMIN
PUT    /api/admin/products/{id}
DELETE /api/admin/products/{id}
PUT    /api/admin/products/{id}/stock
```

---

### Service 3: Order Service
```
Concepts Covered:
├── @Transactional (complex propagation — REQUIRED, REQUIRES_NEW)
├── Saga Pattern (Choreography via Kafka events)
├── Outbox Pattern (dual-write safety)
├── KafkaTemplate (publish order events)
├── @KafkaListener (consume payment/inventory events)
├── ApplicationEventPublisher + @TransactionalEventListener
├── @SpringBootTest integration tests
├── TestContainers (Postgres + Kafka)
├── @Scheduled (cleanup old PENDING orders)
└── Flyway migrations (order tables + outbox table)

Key APIs:
POST   /api/orders            → place order
GET    /api/orders            → user's orders
GET    /api/orders/{id}
PUT    /api/orders/{id}/cancel

Order State Machine:
PENDING → CONFIRMED → PROCESSING → SHIPPED → DELIVERED
       ↘ CANCELLED ↙
```

---

### Service 4: Payment Service
```
Concepts Covered:
├── Circuit Breaker (Resilience4j @CircuitBreaker)
├── @Retry with exponential backoff
├── Fallback methods
├── Feign Client (to external mock payment gateway)
├── Idempotency key pattern (prevent double charging)
├── @Transactional (payment recording)
├── WireMock (test external payment gateway)
└── Dead Letter Topic handling (failed payments)

Key APIs:
POST   /api/payments/charge
POST   /api/payments/refund
GET    /api/payments/{orderId}
```

---

### Service 5: Notification Service
```
Concepts Covered:
├── @KafkaListener (consumer groups)
├── Dead Letter Queue handling
├── @Async for email sending
├── @Scheduled (retry failed notifications)
├── Spring Mail (JavaMailSender)
├── @TransactionalEventListener
└── Custom retry with exponential backoff

No HTTP endpoints — purely event-driven consumer
```

---

### Infrastructure Services
```
Config Server:
├── Spring Cloud Config
├── Git-backed config repository
├── Encrypted properties ({cipher})
└── /actuator/busrefresh for live reload

Discovery Server:
├── Spring Cloud Netflix Eureka
├── Health dashboard
└── Service registration/deregistration

API Gateway:
├── Spring Cloud Gateway (reactive)
├── JWT validation GlobalFilter
├── RequestRateLimiter (Redis-backed)
├── CircuitBreaker filter per route
└── Retry filter
```

---

## 📂 Repository Structure

```
shopwave/
│
├── infrastructure/
│   ├── config-server/          # Spring Cloud Config
│   ├── discovery-server/       # Eureka Server
│   └── config-repo/            # Git repo for configs
│       ├── application.yml
│       ├── user-service.yml
│       ├── product-service.yml
│       ├── order-service.yml
│       ├── payment-service.yml
│       └── notification-service.yml
│
├── services/
│   ├── api-gateway/
│   ├── user-service/
│   ├── product-service/
│   ├── order-service/
│   ├── payment-service/
│   └── notification-service/
│
├── docker-compose.yml          # Full local stack
├── docker-compose-infra.yml    # Just infra (Kafka, Redis, DBs)
│
└── k8s/
    ├── base/
    │   ├── deployments/
    │   ├── services/
    │   └── configmaps/
    └── overlays/
        ├── dev/
        └── prod/
```

---

## 🗓️ Phased Implementation Plan

---

### 🟢 PHASE 1 — The Monolith Foundation
**Duration: Week 1–2 | Goal: Build a working monolith first**

> *"Build it as a monolith first. You'll understand why microservices are needed once the codebase grows."*

**What You Build:**
A single Spring Boot application with all features in one codebase.

```
shopwave-monolith/
├── controller/
│   ├── AuthController.java
│   ├── ProductController.java
│   └── OrderController.java
├── service/
│   ├── UserService.java
│   ├── ProductService.java
│   └── OrderService.java
├── repository/
│   ├── UserRepository.java
│   ├── ProductRepository.java
│   └── OrderRepository.java
├── entity/
│   ├── User.java, Role.java
│   ├── Product.java, Category.java, Inventory.java
│   └── Order.java, OrderItem.java
├── dto/
├── security/
│   ├── JwtUtil.java
│   ├── JwtAuthFilter.java
│   └── SecurityConfig.java
├── exception/
│   └── GlobalExceptionHandler.java
└── resources/
    ├── application.yml
    └── db/migration/
        ├── V1__create_users.sql
        ├── V2__create_products.sql
        └── V3__create_orders.sql
```

**Concepts Practiced:**
```
✅ Spring Boot auto-configuration
✅ @SpringBootApplication internals
✅ Constructor injection everywhere
✅ @ConfigurationProperties (AppConfig, JwtConfig)
✅ Spring Data JPA + Hibernate
✅ Flyway migrations
✅ @Transactional (order placement flow)
✅ Spring Security 6 + JWT
✅ @ControllerAdvice + custom exceptions
✅ Bean Validation (@Valid, custom validators)
✅ Pagination + Sorting
✅ DTO projections (never expose entities)
✅ @WebMvcTest for controller tests
✅ @DataJpaTest for repository tests
✅ Mockito for service unit tests
```

**Specific Tasks:**
```
Day 1-2: Project setup
  - Spring Initializr: Web, JPA, Security, Validation, Flyway, PostgreSQL
  - Docker Compose for local PostgreSQL
  - application.yml with @ConfigurationProperties

Day 3-4: User & Auth
  - User, Role entities with Flyway migration
  - UserRepository with custom queries
  - JWT implementation (JwtUtil, JwtAuthFilter)
  - SecurityFilterChain configuration
  - Registration + Login endpoints with tests

Day 5-6: Product Catalog
  - Product, Category, Inventory entities
  - Product search with Specifications (dynamic filtering)
  - @Version on Inventory (optimistic locking)
  - @Cacheable with Caffeine (in-memory first)
  - Full @WebMvcTest + @DataJpaTest

Day 7-8: Order Flow
  - Order, OrderItem entities
  - Order placement: validate stock → deduct → create order
  - @Transactional with proper propagation
  - N+1 fix with @EntityGraph
  - Integration test with @SpringBootTest
```

**Deliverable:** Working REST API, fully tested, runs locally with Docker Compose PostgreSQL.

---

### 🔵 PHASE 2 — Security Hardening & Data Layer Excellence
**Duration: Week 3 | Goal: Make the monolith production-quality**

> *"Make what you have really good before breaking it apart."*

**What You Add:**

```
Security Additions:
  ├── Refresh token flow (store in DB, revoke on logout)
  ├── @PreAuthorize on all sensitive endpoints
  ├── CORS global configuration
  ├── Rate limiting (manual with a Counter + @Scheduled cleanup)
  └── Password reset flow (email token)

Data Layer Improvements:
  ├── TestContainers (replace H2 with real PostgreSQL in tests)
  ├── Custom Flyway migration (add indexes)
  ├── Proper DTO mapping (add MapStruct)
  ├── Audit fields (@CreatedDate, @LastModifiedDate via @EntityListeners)
  ├── Soft delete pattern (@Where annotation)
  └── Batch operations (deleteAllInBatch for cart cleanup)

Performance:
  ├── Replace Caffeine with Redis (@Cacheable with RedisCacheManager)
  ├── HikariCP tuning (log pool metrics via Actuator)
  ├── Async email sending (@Async + custom ThreadPoolTaskExecutor)
  └── Add @Scheduled job (cancel PENDING orders older than 30 min)
```

**New Tests to Write:**
```
✅ TestContainers integration tests (real PostgreSQL)
✅ @WithMockUser security tests
✅ Test refresh token invalidation
✅ Test @Scheduled job behavior
✅ Test @Async email with Awaitility
✅ Test optimistic locking (concurrent stock update)
```

**Deliverable:** A robust, tested monolith. If you stopped here, this is already better than most production codebases.

---

### 🟡 PHASE 3 — Breaking into Microservices
**Duration: Week 4–5 | Goal: Extract services, add Spring Cloud infrastructure**

> *"Now you'll feel the pain that microservices solve — and the new pain they introduce."*

**Step-by-step Extraction (strangler fig approach):**

```
Step 1: Add API Gateway (keep monolith running)
  ├── Spring Cloud Gateway project
  ├── Route all traffic through gateway to monolith
  ├── Add JWT validation GlobalFilter at gateway
  ├── Add rate limiting with Redis
  └── Monolith still works unchanged

Step 2: Config Server + Discovery Server
  ├── Spring Cloud Config Server pointing to local config-repo/
  ├── Eureka Discovery Server
  ├── Register monolith with Eureka (add @EnableDiscoveryClient)
  └── Move application.yml to config-repo/

Step 3: Extract User Service
  ├── New Spring Boot project: user-service
  ├── Own PostgreSQL schema, own Flyway migrations
  ├── Gateway routes /api/auth/** and /api/users/** to user-service
  ├── Remove auth code from monolith
  └── Monolith now calls user-service for user validation
      (using @FeignClient or RestTemplate with @LoadBalanced)

Step 4: Extract Product Service
  ├── New Spring Boot project: product-service
  ├── Own PostgreSQL schema
  ├── Redis cache (shared Redis instance for now)
  ├── Gateway routes /api/products/** to product-service
  └── Order code still in monolith, calls product-service via Feign

Step 5: Extract Order + Payment Services
  ├── order-service (most complex — has the Saga logic)
  ├── payment-service (with mock external payment gateway)
  ├── Now you need inter-service communication
  └── This is where Kafka becomes essential (Phase 4)
```

**New Concepts Practiced:**
```
✅ Spring Cloud Config (centralized config, encrypted values)
✅ Eureka service registration and discovery
✅ Spring Cloud Gateway routing + filters
✅ @FeignClient (user-service → calls product-service)
✅ @LoadBalanced RestTemplate
✅ @RefreshScope (live config reload)
✅ Multiple SecurityFilterChain beans
✅ Service-to-service JWT propagation
✅ Feign error decoder (handle 404 from product-service)
```

**You'll Hit These Real Problems (intentionally):**
```
Problem 1: "How does order-service know if a product exists?"
  → Feign Client to product-service

Problem 2: "Order-service and product-service both need user info"
  → Gateway extracts user from JWT, passes as X-User-Id header

Problem 3: "How do I test order-service if product-service isn't running?"
  → WireMock to stub product-service responses

Problem 4: "Config is scattered across 5 services again"
  → Config Server with profile-specific files
```

**Deliverable:** 5 separate Spring Boot services, all communicating, all registered with Eureka, all getting config from Config Server. Local Docker Compose runs everything.

---

### 🟠 PHASE 4 — Messaging, Resilience & the Saga Pattern
**Duration: Week 6 | Goal: Make the system resilient and event-driven**

> *"This is where it gets real. Order placement touches 4 services. What happens when payment fails after inventory is reserved?"*

**The Order Placement Saga:**

```
Order Placement Flow (Choreography Saga):

1. order-service receives POST /api/orders
2. Creates order in DB with status PENDING
3. Writes to outbox table (same transaction!)
4. Outbox processor publishes OrderPlaced event to Kafka

5. product-service consumes OrderPlaced
   → Validates and reserves inventory
   → Publishes InventoryReserved OR InventoryInsufficient

6. payment-service consumes InventoryReserved
   → Charges payment (calls mock payment gateway)
   → Publishes PaymentProcessed OR PaymentFailed

7a. order-service consumes PaymentProcessed
    → Updates order status to CONFIRMED
    → Publishes OrderConfirmed

7b. order-service consumes PaymentFailed
    → Updates order status to CANCELLED
    → Publishes OrderCancelled (compensating event)

8. product-service consumes OrderCancelled
   → COMPENSATING TRANSACTION: releases inventory reservation

9. notification-service consumes OrderConfirmed / OrderCancelled
   → Sends email to customer
```

**What You Build:**

```
Kafka Infrastructure (docker-compose):
  - Kafka broker + Zookeeper
  - Topics: order.placed, inventory.reserved, inventory.insufficient,
            payment.processed, payment.failed, order.confirmed,
            order.cancelled, order.*.DLT (dead letter topics)

Outbox Pattern in order-service:
  - outbox table (event_id, event_type, payload, status, created_at)
  - @TransactionalEventListener (AFTER_COMMIT) → publish to Kafka
  - @Scheduled poller for failed/missed events

Resilience4j in payment-service:
  - @CircuitBreaker wrapping external gateway call
  - @Retry with exponential backoff (3 attempts)
  - Fallback: queue payment for manual processing
  - @TimeLimiter (timeout if gateway too slow)

Dead Letter Topic Handling:
  - @RetryableTopic on all @KafkaListener methods
  - DLT listener logs and alerts
  - @Scheduled job retries DLT messages after delay

notification-service:
  - Consumes all *final state* events
  - @Async email sending
  - Failed notification tracking + @Scheduled retry
```

**New Concepts Practiced:**
```
✅ KafkaTemplate (producer)
✅ @KafkaListener with consumer groups
✅ @RetryableTopic + Dead Letter Topics
✅ DefaultErrorHandler with DeadLetterPublishingRecoverer
✅ Outbox Pattern (transactional event publishing)
✅ @TransactionalEventListener (AFTER_COMMIT)
✅ Choreography Saga with compensating transactions
✅ Resilience4j @CircuitBreaker, @Retry, @TimeLimiter
✅ @RateLimiter in payment-service
✅ Idempotency key pattern
✅ TestContainers with KafkaContainer
✅ WireMock for external payment gateway
```

**Integration Test to Write (the big one):**
```java
@SpringBootTest
@Testcontainers
class OrderSagaIntegrationTest {
    // Full saga test:
    // 1. Place order
    // 2. Verify InventoryReserved event published
    // 3. Stub payment gateway (success)
    // 4. Verify order status = CONFIRMED
    // 5. Verify notification sent
    
    // Failure scenario:
    // 1. Place order with insufficient stock
    // 2. Verify order status = CANCELLED
    // 3. Verify notification sent (order failed)
    
    // Payment failure scenario:
    // 1. Place order
    // 2. Inventory reserved
    // 3. Stub payment gateway (failure)
    // 4. Verify inventory released (compensating tx)
    // 5. Verify order status = CANCELLED
}
```

**Deliverable:** Fully event-driven system. Place an order → watch the saga execute across services → check each service's logs → see Kafka messages flowing.

---

### 🔴 PHASE 5 — Observability, Performance & Hardening
**Duration: Week 7 | Goal: Make the system observable and performant**

> *"If it's not measured, it's not managed."*

**Observability Stack:**

```
Setup (add to docker-compose.yml):
  - Prometheus (metrics scraping)
  - Grafana (dashboards + alerting)
  - Zipkin (distributed tracing)
  - Add to all services: micrometer-registry-prometheus
                         micrometer-tracing-bridge-brave
                         zipkin-reporter-brave
```

**What You Add to Each Service:**

```
Actuator Configuration (all services):
  management.endpoints.web.exposure.include=health,info,metrics,prometheus
  management.endpoint.health.show-details=when-authorized
  management.endpoint.health.probes.enabled=true  # K8s probes
  management.metrics.tags.application=${spring.application.name}
  management.tracing.sampling.probability=1.0

Custom Metrics:
  order-service:
    - Counter: orders.placed.total (tagged by status)
    - Timer: order.saga.duration (time from PENDING to CONFIRMED)
    - Gauge: orders.pending.count (current pending orders)
  
  product-service:
    - Counter: cache.hits vs cache.misses
    - Timer: product.search.duration
  
  payment-service:
    - Counter: payment.attempts.total (tagged: success/failed/circuit_open)

Structured Logging:
  - logstash-logback-encoder (JSON output)
  - MDC correlation ID in all log lines
  - CorrelationIdFilter (extract/generate X-Correlation-ID)
  - traceId + spanId auto-added by Micrometer Tracing

Distributed Tracing Flow to Observe:
  POST /api/orders (gateway) 
    → order-service (span 1)
      → product-service via Feign (span 2)  ← see the call
        → DB query (span 3)                 ← see the query time
      → Kafka publish (span 4)
  
  In Zipkin: visualize complete trace, see where time is spent

Grafana Dashboards to Build:
  - Service Health Overview (all /actuator/health endpoints)
  - HTTP Request Rate + Error Rate per service
  - p50/p95/p99 latency per endpoint
  - JVM memory + GC pause times
  - HikariCP connection pool usage
  - Kafka consumer lag per topic
  - Order placement funnel (placed → confirmed rate)

Alerts to Configure:
  - Error rate > 1% for 5 minutes → CRITICAL
  - p99 latency > 2s → WARNING
  - DB connection pool pending > 5 → CRITICAL  
  - Kafka consumer lag > 1000 → WARNING
  - JVM heap > 80% → WARNING
  - Circuit breaker OPEN → CRITICAL
```

**Performance Exercises:**
```
Exercise 1 — Find and fix N+1:
  Enable: spring.jpa.show-sql=true
  Load test: GET /api/orders?userId=1 with 20 orders each with 5 items
  Observe: 1 + 20 + 100 queries
  Fix: @EntityGraph or JOIN FETCH
  Observe: 1 query
  Before/after: log the improvement

Exercise 2 — Add meaningful caching:
  Identify hot path: GET /api/products/{id} (called on every order line item render)
  Add @Cacheable(value="products", key="#id")
  Add cache metrics to Grafana
  Test: cache hit rate under load

Exercise 3 — Tune HikariCP:
  Set: spring.datasource.hikari.maximum-pool-size=5
  Load test order-service
  Watch: hikaricp.connections.pending spike
  Fix: tune pool size
  Observe in Grafana: pending connections drop to 0
```

**Deliverable:** Full observability. You can place an order and trace its entire journey through all services in Zipkin. Grafana shows real-time system health.

---

### 🟣 PHASE 6 — Docker, Kubernetes & Production Readiness
**Duration: Week 8 | Goal: Deploy to Kubernetes**

> *"The last mile. Everything works locally — now make it work in a cluster."*

**Step 1: Production Dockerfiles (all services)**
```dockerfile
# Each service gets a proper multi-stage, layered Dockerfile
FROM eclipse-temurin:21-jre AS builder
WORKDIR /app
COPY target/service.jar service.jar
RUN java -Djarmode=layertools -jar service.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd -r -u 1001 appuser && chown appuser:appuser /app
USER appuser

COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/actuator/health/liveness || exit 1

EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.JarLauncher"]
```

**Step 2: Kubernetes Manifests (per service)**
```yaml
# k8s/base/deployments/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: order-service
          image: shopwave/order-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db-password
          envFrom:
            - configMapRef:
                name: order-service-config
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "15"]
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**Step 3: Kustomize Overlays**
```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployments/
│   ├── services/
│   └── configmaps/
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── replicas-dev.yaml  # replicas: 1 for dev
    └── prod/
        ├── kustomization.yaml
        └── patches/
            └── replicas-prod.yaml  # replicas: 3 for prod
```

**Step 4: Horizontal Pod Autoscaler**
```yaml
# HPA for order-service (most traffic-sensitive)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Step 5: Graceful Shutdown (already configured in Phase 5)**
```yaml
# application.yml (all services)
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**Step 6: Virtual Threads (Java 21 + Spring Boot 3.2)**
```yaml
# Enable for order-service and product-service
spring:
  threads:
    virtual:
      enabled: true
```

**Deliverable:** Full Kubernetes deployment. `kubectl apply -k k8s/overlays/dev` brings up the entire platform. HPA scales order-service under load.

---

## 📊 Concept Coverage Map

```
┌─────────────────────────────────────────┬──────────────────────────────────────────┐
│              CONCEPT                    │           WHERE IN PROJECT               │
├─────────────────────────────────────────┼──────────────────────────────────────────┤
│ IoC, DI, Bean Lifecycle                 │ All services (constructor injection)     │
│ Auto-configuration                      │ Phase 1 (understand what's configured)   │
│ @ConfigurationProperties                │ Phase 1 (JwtConfig, MailConfig)          │
│ Spring MVC / DispatcherServlet          │ Phase 1 (all REST controllers)           │
│ @ControllerAdvice                       │ Phase 1 (GlobalExceptionHandler)         │
│ Bean Validation                         │ Phase 1 (all request DTOs)               │
│ Spring Data JPA + Hibernate             │ Phase 1 (all entities)                   │
│ @Transactional deep dive                │ Phase 1 & 4 (order + saga)               │
│ N+1 Problem                             │ Phase 1 & 5 (find + fix)                 │
│ Flyway Migrations                       │ Phase 1 (all services)                   │
│ TestContainers                          │ Phase 2 (replace H2)                     │
│ Optimistic Locking                      │ Phase 1 (Inventory @Version)             │
│ Spring Security 6                       │ Phase 1 (JWT auth)                       │
│ JWT Authentication                      │ Phase 1 (User Service)                   │
│ @PreAuthorize                           │ Phase 1 (admin endpoints)                │
│ CORS Configuration                      │ Phase 1 (global CORS)                    │
│ @WebMvcTest / MockMvc                   │ Phase 1 (controller tests)               │
│ @DataJpaTest                            │ Phase 1 (repository tests)               │
│ @Mock vs @MockBean                      │ Phase 1 (unit vs slice tests)            │
│ WireMock                                │ Phase 4 (payment gateway stub)           │
│ Saga Integration Test                   │ Phase 4 (full saga test)                 │
│ Service Discovery (Eureka)              │ Phase 3                                  │
│ API Gateway (Spring Cloud Gateway)      │ Phase 3                                  │
│ Config Server (Spring Cloud Config)     │ Phase 3                                  │
│ @FeignClient                            │ Phase 3 (inter-service calls)            │
│ Circuit Breaker (Resilience4j)          │ Phase 4 (payment-service)                │
│ @Retry, @RateLimiter                    │ Phase 4 (payment-service)                │
│ Kafka Producer / Consumer               │ Phase 4 (all services)                   │
│ Dead Letter Topics                      │ Phase 4 (retry + DLT)                    │
│ Outbox Pattern                          │ Phase 4 (order-service)                  │
│ Saga Pattern (choreography)             │ Phase 4 (order → inventory → payment)    │
│ @TransactionalEventListener             │ Phase 4 (outbox publishing)              │
│ @Async + Thread Pool                    │ Phase 2 (email), Phase 4 (notifications) │
│ @Scheduled                              │ Phase 2 (order timeout cleanup)          │
│ @Cacheable / Redis                      │ Phase 2 (product cache)                  │
│ RedisCacheManager                       │ Phase 2 (custom TTL per cache)           │
│ HikariCP Tuning                         │ Phase 5 (pool size tuning exercise)      │
│ Spring Boot Actuator                    │ Phase 5 (all services)                   │
│ Micrometer + Prometheus                 │ Phase 5                                  │
│ Custom Metrics (@Timed, Counter)        │ Phase 5 (order + payment metrics)        │
│ Distributed Tracing (Zipkin)            │ Phase 5                                  │
│ Structured Logging + MDC                │ Phase 5 (correlation IDs)                │
│ Grafana Dashboards + Alerting           │ Phase 5                                  │
│ Liveness / Readiness Probes             │ Phase 5 (Actuator probes)                │
│ Dockerfile (multi-stage, layered)       │ Phase 6                                  │
│ Docker Compose                          │ Phases 1-5 (local dev)                   │
│ Kubernetes Deployments                  │ Phase 6                                  │
│ K8s ConfigMaps + Secrets                │ Phase 6                                  │
│ Horizontal Pod Autoscaler               │ Phase 6                                  │
│ Graceful Shutdown                       │ Phase 5 + 6                              │
│ Virtual Threads (Java 21)               │ Phase 6 (enable + test)                  │
│ 12-Factor App Principles                │ All phases (applied throughout)          │
└─────────────────────────────────────────┴──────────────────────────────────────────┘
```

---

## 🔁 How Phases Map to Interview Prep Weeks

```
Interview Prep Week 1 ──→ Project Phase 1 (Core Foundations)
Interview Prep Week 2 ──→ Project Phase 2 (Data + Security + Testing)
Interview Prep Week 3 ──→ Project Phase 3 + 4 (Microservices + Messaging)
Interview Prep Week 4 ──→ Project Phase 5 + 6 (Observability + K8s)
```

---

## 💡 Key Learning Moments by Phase

```
Phase 1: "Why does @Transactional self-invocation not work?"
  → You'll hit this when trying to call a @Transactional method from the same class
  → You'll understand proxies by fixing it

Phase 2: "Why does @Cacheable not work from same class?"
  → Same proxy issue — reinforces the concept with a different annotation

Phase 3: "How does user-service know the JWT was valid if product-service also needs it?"
  → You'll design the token propagation pattern yourself

Phase 4: "What if Kafka is down when we try to publish the order event?"
  → You'll understand why Outbox Pattern exists when you hit this problem

Phase 5: "Our checkout is slow but we don't know why"
  → Zipkin trace shows product-service Feign call taking 400ms
  → Investigate → missing Redis cache → add it → trace shows 5ms

Phase 6: "Rolling update caused 30 seconds of 502 errors"
  → You'll understand why preStop sleep + readiness probes matter
  → You'll fix it and understand graceful shutdown deeply
```

---

## 🚀 Getting Started (Day 1 Commands)

```bash
# 1. Create project structure
mkdir shopwave && cd shopwave

# 2. Start infrastructure
cat > docker-compose-infra.yml << 'EOF'
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: shopwave
    ports:
      - "5432:5432"
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
EOF

docker-compose -f docker-compose-infra.yml up -d

# 3. Generate monolith project
# spring.io → Add: Web, JPA, Security, Validation, 
#             Flyway, PostgreSQL, Actuator, Cache

# 4. Start building Phase 1
# First commit: "Initial project setup with Docker Compose"
# Second commit: "Add User entity, repository, and Flyway migration V1"
# Third commit: "Implement JWT authentication (JwtUtil + JwtAuthFilter)"
# ...
```

---

> **🎯 Final Advice:** Don't aim to finish every phase before your interview. Phase 1 + 2 done *very well* is more impressive than all 6 phases done superficially. When you talk about this project in an interview, you'll have **real answers** to questions like *"tell me about a time you handled a distributed transaction"* or *"how did you debug a slow endpoint?"* because you actually did those things — not just read about them.