# 🌿 Spring Boot Exception Handling — Complete Interview Guide

```
tags: spring-boot, exceptions, rest-api, error-handling, interview, global-handler
type: permanent-note
status: complete
related: [[Java Exceptions — Intuitive Architecture & Deep Mastery]]
```

---
## 📌 Table of Contents

- [[#🧠 Design Philosophy — Why Spring's Approach Is Fundamentally Different]]
- [[#🏗️ Architecture Overview — The Big Picture]]
- [[#🔄 Request Lifecycle & Exception Resolution Flow]]
- [[#📋 Five Layers of Exception Handling]]
- [[#⚙️ HandlerExceptionResolver Chain — The Internals]]
- [[#🏗️ Custom Exception Hierarchy — Production-Grade Pattern]]
- [[#📦 Error Response Model — The Missing Piece]]
- [[#✅ Validation Exception Handling — Complete Guide]]
- [[#🗄️ @Transactional & Exception Rollback Behavior]]
- [[#🔒 Spring Security Exception Handling]]
- [[#⚡ Async Exception Handling — @Async Methods]]
- [[#🚧 Exceptions Before DispatcherServlet — Filter Layer]]
- [[#🔌 External Service Exception Handling]]
- [[#📋 RFC 7807 ProblemDetail — Spring Boot 3+]]
- [[#🗺️ Grand Architecture Diagram]]
- [[#🎯 Key Interview Questions & Answers]]
- [[#⚡ Best Practices & Anti-Patterns]]
- [[#📝 Quick Revision Card]]

---
## 🧠 Design Philosophy — Why Spring's Approach Is Fundamentally Different

> [!IMPORTANT] The Core Philosophical Shift
> In plain Java, exception handling is **co-located** with business logic — the same class throws, catches, formats, and responds. Spring's philosophy is **radical separation of concerns**: your business logic throws exceptions like they're domain signals; a completely separate component decides what HTTP status code, what message format, and what response body to send back.
### The Three Separations Spring Enforces

```
PLAIN JAVA WAY (tightly coupled):
┌──────────────────────────────────────────────────────┐
│  UserController.getUser()                            │
│  ├── Business Logic: fetch user                      │
│  ├── If not found: catch, format JSON, set status    │
│  ├── If DB error: catch, format JSON, set status     │
│  └── If invalid input: validate, format JSON...      │
│  (Controller does EVERYTHING — no SoC)               │
└──────────────────────────────────────────────────────┘

SPRING WAY (cleanly separated):
┌─────────────────────┐   throws    ┌───────────────────────────┐
│  UserController     │─────────────▶ ResourceNotFoundException │
│  (Only business     │             │ (Just a signal — no HTTP  │
│   logic — clean!)   │             │  knowledge at all)        │
└─────────────────────┘             └───────────────────────────┘
                                               │
                                               ▼ caught by
                               ┌────────────────────────────────┐
                               │  GlobalExceptionHandler        │
                               │  @RestControllerAdvice         │
                               │  (ONLY responsible for:        │
                               │   → HTTP status mapping        │
                               │   → Error response formatting  │
                               │   → Logging strategy           │
                               │   → Security — hide internals) │
                               └────────────────────────────────┘
```

| Separation | What It Means | Benefit |
|---|---|---|
| **Business Logic ↔ Error Handling** | Controllers just throw; Advice classes handle | Clean, readable business code |
| **Domain Exception ↔ HTTP Semantics** | Exceptions have no `HttpStatus` knowledge | Domain layer is portable; not tied to HTTP |
| **Error Detection ↔ Error Presentation** | What went wrong vs. how to tell the client | Consistent responses across all endpoints |
### Spring Framework's Own Preference: Unchecked Exceptions

> [!NOTE] Why Spring Itself Uses Unchecked Exceptions
> Spring Framework made a deliberate architectural choice: almost all Spring exceptions (DataAccessException, BeansException, etc.) are **unchecked**. This means:
> - Services don't need `throws` declarations cluttering their signatures
> - Exceptions propagate naturally up to a central handler
> - Lambda/Stream APIs work without wrapper boilerplate
> - The global `@RestControllerAdvice` acts as the **single catch-all net**
>
> This is the direct practical application of the "external vs. developer bug" dimension from our core Java exception note — in Spring Boot, the *external* environment failures (DB, network) should be translated to unchecked domain exceptions that bubble up cleanly.

---
## 🏗️ Architecture Overview — The Big Picture

Spring Boot uses **auto-configuration** to set up a complete error-handling pipeline out of the box. Understanding what's auto-configured tells you exactly what you're overriding when you customize.

```
┌────────────────────────────────────────────────────────────────────┐
│  AUTO-CONFIGURED BY SPRING BOOT (ErrorMvcAutoConfiguration)        │
│                                                                    │
│  ┌──────────────────────────┐   ┌──────────────────────────────┐   │
│  │  BasicErrorController    │   │  DefaultErrorAttributes      │   │
│  │  Handles /error endpoint │   │  Provides: timestamp, status │   │
│  │  Returns: JSON or HTML   │   │  error, message, path        │   │
│  └──────────────────────────┘   └──────────────────────────────┘   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  HandlerExceptionResolverComposite (ordered chain)           │  │
│  │  1. ExceptionHandlerExceptionResolver  (@ExceptionHandler)   │  │
│  │  2. ResponseStatusExceptionResolver    (@ResponseStatus)     │  │
│  │  3. DefaultHandlerExceptionResolver    (Spring MVC internal) │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

YOUR CUSTOMIZATION POINTS:
┌────────────────────────────────────────────────────────────────┐
│  @RestControllerAdvice (GlobalExceptionHandler)                │
│  ← YOUR most important customization — overrides defaults      │
│                                                                │
│  Custom ErrorAttributes implementation                         │
│  ← Override /error endpoint response shape                     │
│                                                                │
│  ErrorController implementation                                │
│  ← Replace BasicErrorController entirely                       │
└────────────────────────────────────────────────────────────────┘
```

---
## 🔄 Request Lifecycle & Exception Resolution Flow

```
HTTP Request
    │
    ▼
╔═════════════════════════════════════════════════════════════════╗
║  FILTER CHAIN (javax.servlet.Filter / jakarta.servlet.Filter)   ║
║  SecurityFilter, LoggingFilter, JwtFilter, CORSFilter...        ║
║  ⚠️  Exceptions here are NOT caught by @ControllerAdvice!       ║
║  ⚠️  Must be handled INSIDE the filter with try-catch           ║
╚═════════════════════════════════════════════════════════════════╝
    │
    ▼
╔═════════════════════════════════════════════════════════════════╗
║  DispatcherServlet (The Front Controller)                       ║
║  Spring MVC's heart ❤️ — routes requests to handlers             ║
╚═════════════════════════════════════════════════════════════════╝
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Interceptors: preHandle() → Controller → postHandle()          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  @RestController / @Controller Method                           │
│  → calls @Service → calls @Repository                           │
└─────────────────────────────────────────────────────────────────┘
    │
    │  💥 Exception Thrown Anywhere in the Call Stack
    ▼
╔═════════════════════════════════════════════════════════════════╗
║  HandlerExceptionResolver CHAIN (ordered, first-match wins)     ║
║                                                                 ║
║  STEP 1: ExceptionHandlerExceptionResolver                      ║
║  ├── Looks for @ExceptionHandler in the CURRENT controller      ║
║  └── Looks for @ExceptionHandler in ALL @ControllerAdvice beans ║
║       → If found: invoke handler, write response, DONE ✅       ║
║       → If not found: try next resolver ↓                       ║
║                                                                 ║
║  STEP 2: ResponseStatusExceptionResolver                        ║
║  ├── Checks if thrown exception has @ResponseStatus annotation  ║
║  └── Handles ResponseStatusException (programmatic)             ║
║       → If found: send HTTP status + reason, DONE ✅            ║
║       → If not found: try next resolver ↓                       ║
║                                                                 ║
║  STEP 3: DefaultHandlerExceptionResolver                        ║
║  ├── Handles ~15 standard Spring MVC exceptions                 ║
║  └── MethodArgumentNotValidException, NoHandlerFoundException.. ║
║       → If found: send standard response, DONE ✅               ║
║       → If not found: ↓                                         ║
╚═════════════════════════════════════════════════════════════════╝
    │
    │  All resolvers returned null (unresolved)
    ▼
╔═════════════════════════════════════════════════════════════════╗
║  /error endpoint → BasicErrorController (LAST RESORT)           ║
║  Request forwarded to /error path                               ║
║  Returns: Whitelabel Error Page (HTML) for browsers             ║
║           JSON error object for REST clients                    ║
║  → This is what you see as "Whitelabel Error Page"              ║
╚═════════════════════════════════════════════════════════════════╝
```

> [!TIP] Key Insight for Interviews
> The **most critical boundary** is the Filter Chain vs. DispatcherServlet boundary. `@ControllerAdvice` / `@ExceptionHandler` live **inside** the DispatcherServlet world. Any exception thrown in a Filter (JWT validation, rate limiting, etc.) must be handled **within the filter itself** — it never reaches your `@RestControllerAdvice`.

---
## 📋 Five Layers of Exception Handling

### Layer 1: `@ExceptionHandler` — Controller-Level (Most Specific)

Handles exceptions **only for the controller it's defined in**. Use when a specific controller has unique error handling needs.

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        // Just throw — no try-catch here. Clean business logic.
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED)
                             .body(userService.create(req));
    }

    // This handler ONLY applies to exceptions thrown from THIS controller
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex,
            HttpServletRequest request) {  // ← Spring injects this for you

        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

> [!WARNING] Scope Limitation
> This `@ExceptionHandler` for `UserNotFoundException` WILL NOT handle the same exception if thrown from `OrderController`. Use `@RestControllerAdvice` for global handling.

---
### Layer 2: `@RestControllerAdvice` — Global Handler (Most Recommended)

The **single most important pattern** in Spring Boot exception handling. One class handles exceptions for **all controllers** in the application.

```java
// @RestControllerAdvice = @ControllerAdvice + @ResponseBody
// Without @ResponseBody, Spring would try to return a VIEW NAME (string)
// @RestControllerAdvice ensures response is serialized to JSON

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // ─── Handle specific custom domain exceptions ──────────────────────────

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        log.warn("Resource not found: {}", ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of(HttpStatus.NOT_FOUND, ex, request));
    }

    @ExceptionHandler(ConflictException.class)
    public ResponseEntity<ErrorResponse> handleConflict(
            ConflictException ex,
            HttpServletRequest request) {

        log.warn("Conflict: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ErrorResponse.of(HttpStatus.CONFLICT, ex, request));
    }

    // ─── Multiple exceptions → same handler ───────────────────────────────

    @ExceptionHandler({IllegalArgumentException.class, IllegalStateException.class})
    public ResponseEntity<ErrorResponse> handleBadRequest(
            RuntimeException ex,
            HttpServletRequest request) {

        log.warn("Bad request: {}", ex.getMessage());
        return ResponseEntity.badRequest()
            .body(ErrorResponse.of(HttpStatus.BAD_REQUEST, ex, request));
    }

    // ─── Catch-all — MUST be last ──────────────────────────────────────────

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(
            Exception ex,
            HttpServletRequest request) {

        // Log with full stack trace — this is unexpected!
        log.error("Unexpected error processing request to [{}]",
                   request.getRequestURI(), ex);

        // ⚠️ NEVER expose internal details (stack trace, DB schema, file paths)
        return ResponseEntity.internalServerError()
            .body(ErrorResponse.of(HttpStatus.INTERNAL_SERVER_ERROR,
                  "An unexpected error occurred. Please try again.", request));
    }
}
```

#### Scoping `@ControllerAdvice` (Advanced)

```java
// Apply only to controllers in a specific package:
@RestControllerAdvice("com.example.api.v2")

// Apply only to specific controller classes:
@RestControllerAdvice(assignableTypes = {UserController.class, OrderController.class})

// Apply only to controllers with a specific annotation:
@RestControllerAdvice(annotations = RestController.class)

// ─── Multiple @ControllerAdvice beans — control order with @Order ────────
@RestControllerAdvice
@Order(1)  // Lower number = higher priority (runs first)
public class ValidationExceptionHandler { ... }

@RestControllerAdvice
@Order(2)
public class GlobalExceptionHandler { ... }
```

---
### Layer 3: `ResponseEntityExceptionHandler` — Handle Spring MVC Exceptions

Spring MVC throws ~15 well-known exceptions internally (method not found, missing argument, type mismatch, etc.). The `ResponseEntityExceptionHandler` base class handles all of them — you just need to **extend and override** to customize the response format.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    //                               ↑ EXTENDS this base class

    // ─── Override to customize Spring MVC validation error response ───────
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,        // HttpStatusCode in Spring 6, HttpStatus in Spring 5
            WebRequest request) {

        // Extract field-level validation errors
        List<FieldViolation> violations = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new FieldViolation(
                fe.getField(),
                fe.getDefaultMessage(),
                fe.getRejectedValue()  // The value that failed validation
            ))
            .toList();

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("One or more fields failed validation")
            .violations(violations)
            .timestamp(Instant.now())
            .path(((ServletWebRequest) request).getRequest().getRequestURI())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    // ─── Malformed JSON in request body ───────────────────────────────────
    @Override
    protected ResponseEntity<Object> handleHttpMessageNotReadable(
            HttpMessageNotReadableException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        log.warn("Malformed JSON request: {}", ex.getMessage());

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Malformed Request")
            .message("Request body is malformed or contains invalid JSON")
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    // ─── No handler found (404) ────────────────────────────────────────────
    @Override
    protected ResponseEntity<Object> handleNoHandlerFoundException(
            NoHandlerFoundException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message("No handler found for " + ex.getHttpMethod() + " " + ex.getRequestURL())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // ─── Custom domain exceptions still handled here too ──────────────────
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Object> handleResourceNotFound(
            ResourceNotFoundException ex,
            WebRequest request) {

        log.warn("Resource not found: {}", ex.getMessage());

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

#### What `ResponseEntityExceptionHandler` Handles Out of the Box

| Spring MVC Exception | Default HTTP Status | Common Cause |
|---|---|---|
| `MethodArgumentNotValidException` | 400 | `@Valid` body validation fails |
| `BindException` | 400 | `@ModelAttribute` validation fails |
| `MethodArgumentTypeMismatchException` | 400 | `@PathVariable` type mismatch |
| `HttpMessageNotReadableException` | 400 | Malformed JSON body |
| `MissingServletRequestParameterException` | 400 | Required `@RequestParam` missing |
| `HttpRequestMethodNotSupportedException` | 405 | POST to GET-only endpoint |
| `HttpMediaTypeNotSupportedException` | 415 | Wrong Content-Type header |
| `HttpMediaTypeNotAcceptableException` | 406 | Client can't accept response type |
| `NoHandlerFoundException` | 404 | No matching controller found |
| `MethodNotAllowedException` | 405 | HTTP method not supported |

> [!TIP] Why Extend `ResponseEntityExceptionHandler`?
> Without extending it, Spring's `DefaultHandlerExceptionResolver` handles these exceptions but sends a **plain empty response** with just the HTTP status code — no JSON body. By extending and overriding, you ensure these exceptions also return your **consistent error response format**.

---
### Layer 4: `@ResponseStatus` & `ResponseStatusException`

Two simple approaches that don't require a global handler — but with trade-offs.

#### `@ResponseStatus` on Exception Class

```java
// The annotation maps this exception type → HTTP status automatically
@ResponseStatus(
    value    = HttpStatus.NOT_FOUND,
    reason   = "User not found"  // ⚠️ Reason becomes "message" in error response
                                  // ⚠️ Exposes internal message — careful in production
)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User with id " + id + " does not exist");
    }
}

// Anywhere this is thrown, Spring automatically sends 404
// ResponseStatusExceptionResolver handles it — no @ExceptionHandler needed
```

**Limitation:** The `reason` field is static. You can't include dynamic data like the actual ID in the HTTP response message (though `getMessage()` still has it internally).

#### `ResponseStatusException` — Programmatic

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResponseStatusException(
            HttpStatus.NOT_FOUND,
            "User not found with id: " + id
            // Optional 3rd param: Throwable cause
        ));
}

// Also useful for conditional HTTP statuses:
@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    boolean deleted = userService.deleteIfExists(id);
    if (!deleted) {
        throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found");
    }
    return ResponseEntity.noContent().build();
}
```

#### Comparison Table

| Approach | When to Use | Pros | Cons |
|---|---|---|---|
| `@ResponseStatus` on exception | Simple apps, few exception types | Zero boilerplate | Static message, limited control |
| `ResponseStatusException` | Simple one-off cases in controller | Inline, no extra class | Pollutes controller, HTTP concern in business layer |
| `@RestControllerAdvice` + `@ExceptionHandler` | **Production recommendation** | Centralized, consistent, powerful | Slightly more setup |

---

### Layer 5: `BasicErrorController` — The Last Resort

If no resolver or advice handles the exception, DispatcherServlet forwards the request to `/error`. `BasicErrorController` handles it.

```java
// You can replace BasicErrorController by implementing ErrorController:
@RestController
@RequestMapping("/error")
public class CustomErrorController implements ErrorController {

    private final ErrorAttributes errorAttributes;

    public CustomErrorController(ErrorAttributes errorAttributes) {
        this.errorAttributes = errorAttributes;
    }

    @RequestMapping
    public ResponseEntity<ErrorResponse> handleError(HttpServletRequest request) {
        WebRequest webRequest = new ServletWebRequest(request);

        // Gets auto-collected attributes: status, error, message, path, timestamp
        Map<String, Object> attrs = errorAttributes.getErrorAttributes(
            webRequest,
            ErrorAttributeOptions.defaults()
        );

        int status = (int) attrs.getOrDefault("status", 500);

        ErrorResponse error = ErrorResponse.builder()
            .status(status)
            .message((String) attrs.get("message"))
            .path((String) attrs.get("path"))
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(status).body(error);
    }
}

// application.properties:
// spring.mvc.throw-exception-if-no-handler-found=true  ← enables NoHandlerFoundException
// spring.web.resources.add-mappings=false              ← prevents ResourceHttpRequestHandler from
//                                                         intercepting and masking 404s
```

---

## ⚙️ HandlerExceptionResolver Chain — The Internals

```
DispatcherServlet.processHandlerException(request, response, handler, ex)
    │
    ▼
Iterates through List<HandlerExceptionResolver> (ordered by @Order / Ordered):
    │
    ├─── [Priority 1] ExceptionHandlerExceptionResolver
    │    ├── Scans @Controller for @ExceptionHandler matching thrown type
    │    │   (most specific exception type match wins — subtype over supertype)
    │    └── Scans all @ControllerAdvice beans (in @Order order)
    │        for @ExceptionHandler matching thrown type
    │        → Match found: invoke method, write response, return ModelAndView
    │        → No match: return null (try next resolver)
    │
    ├─── [Priority 2] ResponseStatusExceptionResolver
    │    ├── Checks: is exception annotated with @ResponseStatus?
    │    └── Checks: is it instanceof ResponseStatusException?
    │        → Yes: send status + reason, return ModelAndView
    │        → No: return null (try next resolver)
    │
    └─── [Priority 3] DefaultHandlerExceptionResolver
         ├── Handles hardcoded list of Spring MVC exceptions
         │   (HttpRequestMethodNotSupportedException, etc.)
         └── Sends appropriate HTTP status code
             → Handled: return ModelAndView
             → Not handled: return null
                 │
                 ▼
         DispatcherServlet forwards to /error endpoint
         → BasicErrorController handles
```

> [!NOTE] @ExceptionHandler Resolution Order — Specificity Wins
> If you have two handlers: `@ExceptionHandler(Exception.class)` and `@ExceptionHandler(ResourceNotFoundException.class)`, and `ResourceNotFoundException` is thrown, Spring picks the **most specific match** — i.e., `ResourceNotFoundException.class` handler. This mimics Java's `catch` block specificity rule.

---

## 🏗️ Custom Exception Hierarchy — Production-Grade Pattern

### The Standard Project Structure

```
src/main/java/com/example/
├── exception/
│   ├── ApplicationException.java          ← Abstract base (extends RuntimeException)
│   ├── ResourceNotFoundException.java     ← 404
│   ├── ConflictException.java             ← 409
│   ├── ValidationException.java           ← 400 (business validation, not field)
│   ├── InfrastructureException.java       ← 503
│   └── UnauthorizedException.java         ← 401
│
├── web/
│   ├── GlobalExceptionHandler.java        ← @RestControllerAdvice
│   └── dto/
│       ├── ErrorResponse.java             ← Standard error response DTO
│       └── FieldViolation.java            ← Field-level validation error
```

### The Complete Implementation

```java
// ─────────────────────────────────────────────────────────────────────────
// 1. Abstract Base Exception
// ─────────────────────────────────────────────────────────────────────────

/**
 * Base for all application domain exceptions.
 * UNCHECKED by design — follows Spring Framework's convention.
 * All subclasses automatically bubble up to GlobalExceptionHandler.
 */
public abstract class ApplicationException extends RuntimeException {

    private final String errorCode;   // Machine-readable code (e.g. "USER_NOT_FOUND")

    protected ApplicationException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected ApplicationException(String errorCode, String message, Throwable cause) {
        super(message, cause);  // ← ALWAYS chain cause — never lose the root
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// ─────────────────────────────────────────────────────────────────────────
// 2. Specific Domain Exceptions
// ─────────────────────────────────────────────────────────────────────────

public class ResourceNotFoundException extends ApplicationException {

    private final String resourceType;
    private final Object resourceId;

    public ResourceNotFoundException(String resourceType, Object resourceId) {
        super(
            "RESOURCE_NOT_FOUND",
            String.format("%s not found with identifier: %s", resourceType, resourceId)
        );
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }

    // Convenience factory methods — clean calling code
    public static ResourceNotFoundException ofUser(Long id) {
        return new ResourceNotFoundException("User", id);
    }

    public static ResourceNotFoundException ofOrder(String orderId) {
        return new ResourceNotFoundException("Order", orderId);
    }

    public String getResourceType() { return resourceType; }
    public Object getResourceId() { return resourceId; }
}


public class ConflictException extends ApplicationException {

    public ConflictException(String message) {
        super("CONFLICT", message);
    }

    // Domain-specific factory
    public static ConflictException userAlreadyExists(String email) {
        return new ConflictException("User with email '" + email + "' already exists");
    }
}


/**
 * For business rule violations — distinct from @Valid field validation.
 * e.g., "Cannot cancel an order that has already shipped"
 */
public class BusinessRuleViolationException extends ApplicationException {

    public BusinessRuleViolationException(String message) {
        super("BUSINESS_RULE_VIOLATION", message);
    }
}


/**
 * Wraps infrastructure failures (DB, message broker, external APIs).
 * Cause MUST always be provided — preserves diagnostic chain.
 */
public class InfrastructureException extends ApplicationException {

    public InfrastructureException(String message, Throwable cause) {
        super("INFRASTRUCTURE_ERROR", message, cause);
        // cause is MANDATORY — without it, we lose the original SQLException/IOException
    }
}
```

### The Error Response DTO

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  // Don't serialize null fields
@Builder
@Getter
public class ErrorResponse {

    private final int            status;
    private final String         error;       // HTTP status text ("Not Found")
    private final String         errorCode;   // Domain error code ("USER_NOT_FOUND")
    private final String         message;     // Human-readable description
    private final String         path;        // Request URI
    private final Instant        timestamp;
    private final List<FieldViolation> violations;  // Only for validation errors; null otherwise

    // Static factory for domain exceptions
    public static ErrorResponse of(HttpStatus status, ApplicationException ex, HttpServletRequest req) {
        return ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .errorCode(ex.getErrorCode())
            .message(ex.getMessage())
            .path(req.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }

    // Static factory for generic messages (catch-all handler)
    public static ErrorResponse of(HttpStatus status, String message, HttpServletRequest req) {
        return ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .message(message)
            .path(req.getRequestURI())
            .timestamp(Instant.now())
            .build();
    }
}

@Value  // Lombok: immutable, all-args constructor, getters
public class FieldViolation {
    String field;           // "email"
    String message;         // "must be a valid email address"
    Object rejectedValue;   // "not-an-email"
}
```

**Sample JSON Response:**

```json
// 404 - ResourceNotFoundException
{
  "status": 404,
  "error": "Not Found",
  "errorCode": "RESOURCE_NOT_FOUND",
  "message": "User not found with identifier: 42",
  "path": "/api/v1/users/42",
  "timestamp": "2024-01-15T10:30:00Z"
}

// 400 - Validation Error
{
  "status": 400,
  "error": "Bad Request",
  "errorCode": "VALIDATION_FAILED",
  "message": "One or more fields failed validation",
  "path": "/api/v1/users",
  "timestamp": "2024-01-15T10:30:00Z",
  "violations": [
    { "field": "email",    "message": "must be a well-formed email address", "rejectedValue": "bad-email" },
    { "field": "age",      "message": "must be greater than or equal to 18",  "rejectedValue": 15         }
  ]
}
```

---
# 📦 Error Response Model — The Missing Piece

## 🧠 The Conceptual Gap — Why This Section Exists

Reading through the exception handling note, you saw `ErrorResponse` used everywhere:

```java
return ResponseEntity.status(HttpStatus.NOT_FOUND)
    .body(ErrorResponse.of(HttpStatus.NOT_FOUND, ex, request));
```

But what **is** `ErrorResponse`? Where does it come from? Is it a Spring class? Is it part of your custom exception? Is it something else entirely?

> [!IMPORTANT] The Core Mental Model
> There are **three completely separate things** working together. Most tutorials collapse them, causing lasting confusion:
>
> ```
> THING 1: Custom Exception       → What went wrong (domain signal)
>          e.g. ResourceNotFoundException
>
> THING 2: ErrorResponse DTO      → What the HTTP client sees (JSON shape)
>          e.g. { "status": 404, "message": "...", "errorCode": "..." }
>
> THING 3: @ExceptionHandler      → The bridge (maps THING 1 → THING 2)
>          in @RestControllerAdvice
> ```
>
> `ErrorResponse` is **your own plain Java class** — a DTO (Data Transfer Object). Spring does not provide it. You write it. It has one job: **define the consistent JSON structure of every error response your API sends.**

---
## 🔍 Why Does `ErrorResponse` Exist? — The REST Contract Problem

### What happens without it

Without a defined error response shape, every handler can return whatever it wants:

```java
// Handler A returns this:
{ "error": "User not found" }

// Handler B returns this:
{ "message": "Not Found", "code": 404 }

// Handler C (Spring default) returns this:
{
  "timestamp": "2024-01-15T10:30:00.000+00:00",
  "status": 404,
  "error": "Not Found",
  "path": "/api/users/42"
}

// Handler D returns a plain String:
"Something went wrong"
```

The client (mobile app, frontend, other microservice) now has to write **different parsing logic for every possible error shape**. This is a broken API contract.

### What it means for REST contracts

A REST API is a **contract**. Just as your success responses have a consistent shape (`UserDto`, `OrderDto`), your error responses must also have a **guaranteed, predictable shape** that every consumer can rely on.

```
SUCCESS CONTRACT:                    ERROR CONTRACT:
──────────────────                   ─────────────────────────────────
HTTP 200                             HTTP 4xx / 5xx
Content-Type: application/json       Content-Type: application/json
{                                    {
  "id": 42,                            "status": 404,
  "name": "Alice",         ←→          "error": "Not Found",
  "email": "alice@..."                 "errorCode": "RESOURCE_NOT_FOUND",
}                                      "message": "User not found: 42",
                                       "path": "/api/users/42",
                                       "timestamp": "2024-01-15T10:30:00Z"
                                     }

Clients can always rely on this shape.
```

`ErrorResponse` is the Java class that **represents and enforces this contract** on the server side. Every `@ExceptionHandler` method returns it, ensuring uniformity.

---
## 🔗 The Relationship Between Custom Exception and `ErrorResponse`

> [!IMPORTANT] They Are Separate Concerns — Do NOT mix them
> A custom exception should know **nothing** about HTTP or JSON. It lives in the domain layer. `ErrorResponse` lives in the web/API layer. The `@ExceptionHandler` bridges them.

```
                    DOMAIN LAYER
        ┌──────────────────────────────────┐
        │  ResourceNotFoundException       │
        │  ─────────────────────────────  │
        │  Fields:                         │
        │    - errorCode: String           │  ← Domain vocabulary
        │    - message: String             │  ← What went wrong
        │    - resourceType: String        │  ← Business context
        │    - resourceId: Object          │  ← Business context
        │                                  │
        │  NO HttpStatus field             │  ← ✅ No HTTP knowledge
        │  NO JSON annotations             │  ← ✅ Not a DTO
        │  NO @JsonProperty                │  ← ✅ Pure domain object
        └──────────────┬───────────────────┘
                       │  thrown
                       ▼  caught by
        ┌──────────────────────────────────┐
        │  @ExceptionHandler               │  ← THE BRIDGE (web layer)
        │  in @RestControllerAdvice        │
        │                                  │
        │  Receives the exception,         │
        │  decides HTTP status,            │
        │  constructs ErrorResponse,       │
        │  returns ResponseEntity          │
        └──────────────┬───────────────────┘
                       │  produces
                       ▼
                   WEB/API LAYER
        ┌──────────────────────────────────┐
        │  ErrorResponse (DTO)             │
        │  ─────────────────────────────  │
        │  Fields:                         │
        │    - status: int                 │  ← HTTP vocabulary
        │    - error: String               │  ← HTTP reason phrase
        │    - errorCode: String           │  ← from domain exception
        │    - message: String             │  ← from domain exception
        │    - path: String                │  ← from HttpServletRequest
        │    - timestamp: Instant          │  ← generated at handler time
        │    - violations: List<...>       │  ← only for validation errors
        │                                  │
        │  @JsonInclude, @JsonFormat       │  ← ✅ JSON serialization
        │  Serialized to JSON by Jackson   │
        └──────────────────────────────────┘
                       │
                       ▼
        HTTP Response Body (what client receives):
        {
          "status": 404,
          "error": "Not Found",
          "errorCode": "RESOURCE_NOT_FOUND",
          "message": "User not found with identifier: 42",
          "path": "/api/v1/users/42",
          "timestamp": "2024-01-15T10:30:00Z"
        }
```

---
## 🔧 Complete `ErrorResponse` Implementation

### Version 1: Using Lombok (recommended in most projects)

```java
package com.example.web.dto;

import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonInclude;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;

import java.time.Instant;
import java.util.List;

/**
 * Standard error response DTO.
 *
 * PURPOSE : Defines the JSON shape of ALL error responses from this API.
 *           This is NOT a Spring class — it is a plain POJO you own.
 *
 * LAYER   : Web / API layer only.
 *           Domain exceptions have NO knowledge of this class.
 *
 * USED BY : Every @ExceptionHandler method in GlobalExceptionHandler.
 *           Never used inside Custom Exception classes themselves.
 */
@JsonInclude(JsonInclude.Include.NON_NULL)  // ← Fields that are null are NOT included in JSON
                                            //   So "violations" won't appear on non-validation errors
@Builder                                    // Lombok: generates builder pattern
@Getter                                     // Lombok: generates getters (Jackson needs them)
@ToString
public class ErrorResponse {

    /**
     * HTTP status code as integer.
     * Redundant with HTTP response status line, but convenient for clients
     * who read the body without inspecting headers.
     * e.g. 404, 400, 500
     */
    private final int status;

    /**
     * HTTP status reason phrase.
     * e.g. "Not Found", "Bad Request", "Internal Server Error"
     * Derived from HttpStatus enum.
     */
    private final String error;

    /**
     * Machine-readable domain error code.
     * Allows clients to handle specific errors programmatically
     * WITHOUT string-matching on the human-readable message.
     *
     * Convention: SCREAMING_SNAKE_CASE
     * e.g. "RESOURCE_NOT_FOUND", "VALIDATION_FAILED", "INSUFFICIENT_BALANCE"
     *
     * Clients can do: if (error.getErrorCode().equals("RESOURCE_NOT_FOUND")) { ... }
     * Instead of:    if (error.getMessage().contains("not found")) { ... }  ← fragile!
     */
    private final String errorCode;

    /**
     * Human-readable description of what went wrong.
     * Safe to display (in some form) to end users.
     * NEVER include: stack traces, SQL error text, file system paths, class names.
     * e.g. "User not found with identifier: 42"
     */
    private final String message;

    /**
     * The request URI that caused the error.
     * Extremely useful for debugging — immediately tells you which endpoint failed.
     * e.g. "/api/v1/users/42"
     */
    private final String path;

    /**
     * When the error occurred.
     * ISO-8601 format via @JsonFormat.
     * Useful for correlating with server logs.
     */
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
    private final Instant timestamp;

    /**
     * Field-level validation violations.
     * ONLY populated for validation errors (HTTP 400 from @Valid / @Validated).
     * NULL (and excluded from JSON via @JsonInclude) for all other error types.
     */
    private final List<FieldViolation> violations;

    /**
     * Distributed tracing ID — correlates this error response with server logs.
     * Populate from MDC: MDC.get("traceId") set by your logging filter.
     * Client support can ask: "What is the traceId?" to find the exact log entry.
     * e.g. "abc123-def456-..."
     */
    private final String traceId;


    // ─── Static Factory Methods — convenience builders ────────────────────────
    //     These centralize the construction logic so @ExceptionHandler methods
    //     stay clean and don't repeat the same builder calls.

    /**
     * For ApplicationException subclasses (domain exceptions with errorCode).
     * Most commonly used factory.
     */
    public static ErrorResponse of(HttpStatus status,
                                    ApplicationException ex,
                                    HttpServletRequest request) {
        return ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .errorCode(ex.getErrorCode())
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .traceId(MDC.get("traceId"))  // from SLF4J MDC (logging context)
            .build();
    }

    /**
     * For generic/unexpected exceptions (catch-all handler).
     * Message is a safe generic string — NOT ex.getMessage() (may expose internals).
     */
    public static ErrorResponse of(HttpStatus status,
                                    String safeMessage,
                                    HttpServletRequest request) {
        return ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .message(safeMessage)
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .traceId(MDC.get("traceId"))
            .build();
    }

    /**
     * For validation errors — includes field violations.
     */
    public static ErrorResponse validationError(List<FieldViolation> violations,
                                                  HttpServletRequest request) {
        return ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error(HttpStatus.BAD_REQUEST.getReasonPhrase())
            .errorCode("VALIDATION_FAILED")
            .message("Request validation failed. See 'violations' for details.")
            .violations(violations)
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .traceId(MDC.get("traceId"))
            .build();
    }
}
```
### Version 2: Plain Java (no Lombok) — for interviews where you write from memory

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {

    private final int                 status;
    private final String              error;
    private final String              errorCode;
    private final String              message;
    private final String              path;
    private final Instant             timestamp;
    private final List<FieldViolation> violations;

    // Private constructor — force use of builder or factory methods
    private ErrorResponse(int status, String error, String errorCode,
                           String message, String path, Instant timestamp,
                           List<FieldViolation> violations) {
        this.status     = status;
        this.error      = error;
        this.errorCode  = errorCode;
        this.message    = message;
        this.path       = path;
        this.timestamp  = timestamp;
        this.violations = violations;
    }

    // Jackson needs getters for serialization
    public int                  getStatus()     { return status;     }
    public String               getError()      { return error;      }
    public String               getErrorCode()  { return errorCode;  }
    public String               getMessage()    { return message;    }
    public String               getPath()       { return path;       }
    public Instant              getTimestamp()  { return timestamp;  }
    public List<FieldViolation> getViolations() { return violations; }

    // Static factory — most common usage
    public static ErrorResponse of(HttpStatus status, String errorCode,
                                    String message, HttpServletRequest req) {
        return new ErrorResponse(
            status.value(),
            status.getReasonPhrase(),
            errorCode,
            message,
            req.getRequestURI(),
            Instant.now(),
            null
        );
    }
}
```
### `FieldViolation` — the nested validation detail

```java
/**
 * Represents a single field-level constraint violation.
 * Only appears inside ErrorResponse.violations for HTTP 400 validation errors.
 *
 * NOT related to any exception class.
 * Populated by the @ExceptionHandler that handles MethodArgumentNotValidException.
 */
@Value           // Lombok: immutable, all-args constructor, getters
@JsonInclude(JsonInclude.Include.NON_NULL)
public class FieldViolation {

    /**
     * The field name that failed validation.
     * Matches the JSON field name in the request body.
     * e.g. "email", "age", "address.zipCode"  ← dot notation for nested fields
     */
    String field;

    /**
     * The violation message from the constraint annotation.
     * Comes from: @NotBlank(message = "Email is required") → "Email is required"
     * Or from ValidationMessages.properties for i18n.
     * e.g. "must be a well-formed email address"
     */
    String message;

    /**
     * The actual value that was rejected.
     * e.g. "not-an-email", -5, null
     *
     * ⚠️ SECURITY: Be cautious with sensitive fields (passwords, card numbers).
     * Consider masking or omitting for sensitive fields.
     */
    Object rejectedValue;
}
```

---
## 📊 `ErrorResponse` Usage Map — When Is It Used?

```
SCENARIO                          USED?    HOW
─────────────────────────────────────────────────────────────────────────
Custom Exception thrown            ✅      @ExceptionHandler catches it,
(ResourceNotFoundException etc.)           CONSTRUCTS ErrorResponse,
                                           returns ResponseEntity<ErrorResponse>

@Valid validation fails            ✅      handleMethodArgumentNotValid()
(MethodArgumentNotValidException)          constructs ErrorResponse
                                           WITH violations list

@Validated param violation         ✅      @ExceptionHandler(ConstraintViolation)
(ConstraintViolationException)             constructs ErrorResponse
                                           WITH violations list

Spring MVC internal exceptions     ✅      Overridden methods in
(HttpRequestMethodNotSupported etc)        ResponseEntityExceptionHandler
                                           construct ErrorResponse

Unexpected Exception               ✅      catch-all @ExceptionHandler(Exception)
(NullPointerException etc.)                constructs ErrorResponse
                                           with SAFE generic message (not ex.getMessage()!)

INSIDE a Custom Exception class    ❌      NEVER. ErrorResponse belongs in
                                           the web layer, not the domain layer.

Spring Security exceptions         ⚠️      NOT via @ExceptionHandler.
(Auth/Access failures)                     Manually write JSON in
                                           AuthenticationEntryPoint /
                                           AccessDeniedHandler using ObjectMapper.
                                           Can use same ErrorResponse class though.

Filter exceptions (JWT etc.)       ⚠️      NOT via @ExceptionHandler.
                                           Manually write JSON using ObjectMapper.
                                           Can use same ErrorResponse class.
```

---
## 🎯 The `errorCode` Field — Why It Deserves Special Attention

This field separates amateur from professional API design and is worth discussing explicitly in interviews.

```
WITHOUT errorCode:
────────────────────────────────────────────────────────
Server sends: { "message": "User not found with id: 42" }

Client code:
if (error.getMessage().contains("not found")) {  ← fragile string matching!
    showNotFoundUI();
}
// Breaks if you change the message text.
// Breaks across languages/locales.
// "Not found" vs "not found" → case sensitivity bug


WITH errorCode:
────────────────────────────────────────────────────────
Server sends: { "errorCode": "RESOURCE_NOT_FOUND", "message": "User not found: 42" }

Client code:
if ("RESOURCE_NOT_FOUND".equals(error.getErrorCode())) {   ← stable contract
    showNotFoundUI();
}
// Never breaks — errorCode is a stable API contract
// message can be reworded, translated, enhanced freely
// Clients can be written in any language


PRACTICAL errorCode EXAMPLES:
────────────────────────────────────────────────────────
RESOURCE_NOT_FOUND         → show "item doesn't exist" UI
VALIDATION_FAILED          → highlight specific form fields (use violations[])
INSUFFICIENT_BALANCE       → show "top up your account" modal
CARD_DECLINED              → show "try another card" flow
DUPLICATE_EMAIL            → show "email already registered, login instead?"
SESSION_EXPIRED            → redirect to login page
RATE_LIMIT_EXCEEDED        → show "try again in X seconds"
MAINTENANCE_MODE           → show maintenance banner
```

---
## 🏗️ Final Architecture: The Three Classes Together

```java
// ─── COMPLETE WORKING EXAMPLE — all three pieces together ─────────────────

// 1. DOMAIN LAYER: Custom Exception (no HTTP/JSON knowledge)
public class ResourceNotFoundException extends ApplicationException {

    private final String resourceType;
    private final Object resourceId;

    public ResourceNotFoundException(String resourceType, Object resourceId) {
        super(
            "RESOURCE_NOT_FOUND",    // errorCode
            resourceType + " not found with identifier: " + resourceId  // message
        );
        this.resourceType = resourceType;
        this.resourceId   = resourceId;
    }
    // Getters...
}


// 2. WEB LAYER: ErrorResponse DTO (HTTP/JSON contract — no exception knowledge)
@JsonInclude(JsonInclude.Include.NON_NULL)
@Builder @Getter
public class ErrorResponse {
    private final int     status;
    private final String  error;
    private final String  errorCode;
    private final String  message;
    private final String  path;
    private final Instant timestamp;
    private final List<FieldViolation> violations;
}


// 3. WEB LAYER: The bridge — @ExceptionHandler converts exception → ErrorResponse
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex,        // ← receives domain exception
            HttpServletRequest request) {

        ErrorResponse body = ErrorResponse.of(  // ← constructs HTTP response DTO
            HttpStatus.NOT_FOUND,
            ex,
            request
        );

        return ResponseEntity                    // ← wraps in HTTP response
                .status(HttpStatus.NOT_FOUND)
                .body(body);
    }
}


// ─── WHAT THE CLIENT RECEIVES ─────────────────────────────────────────────
// HTTP/1.1 404 Not Found
// Content-Type: application/json
//
// {
//   "status": 404,
//   "error": "Not Found",
//   "errorCode": "RESOURCE_NOT_FOUND",
//   "message": "User not found with identifier: 42",
//   "path": "/api/v1/users/42",
//   "timestamp": "2024-01-15T10:30:00Z"
// }
```

---
## ❓ Answering Your Original Questions Directly

| Question | Answer |
|---|---|
| **Why is `ErrorResponse` used?** | To enforce a consistent, predictable JSON shape for all error responses — fulfilling the REST API contract for error cases |
| **When is it used?** | In every `@ExceptionHandler` method inside `@RestControllerAdvice`; also manually in filters and security handlers |
| **Is it used while making Custom Exceptions?** | **No.** Custom Exception classes have zero knowledge of `ErrorResponse`. They live in different layers entirely. The `@ExceptionHandler` bridge method is where they meet |
| **Is it a Spring class?** | **No.** It is a plain Java class you write yourself. You own its shape entirely |
| **Is it related to Spring 6's `ErrorResponse` interface?** | Name collision! Spring 6 has an `ErrorResponse` *interface* (for `ProblemDetail`). Your custom `ErrorResponse` *DTO* is a different, unrelated class. Rename yours to `ApiErrorResponse` or `ErrorApiResponse` if using Spring 6 to avoid confusion |
| **Does it meet REST contracts?** | **Yes — that is its primary purpose.** It is the server-side Java representation of the error response contract your API promises to every consumer |

---
## ⚠️ Spring 6 Name Collision Warning

```java
// Spring Framework 6 / Spring Boot 3 introduced:
org.springframework.web.ErrorResponse  ← Spring's interface (for ProblemDetail)

// Your custom DTO:
com.example.web.dto.ErrorResponse      ← YOUR class

// These have the same simple name → IDE import confusion / compiler errors
// if both are on the classpath and you forget the full package.

// ✅ RECOMMENDATION for Spring Boot 3+ projects:
// Rename your class to avoid collision:
//   ApiErrorResponse        ← clear, unambiguous
//   ErrorApiResponse        ← also clear
//   ApiError                ← compact (used by many open-source projects)
//   HttpErrorResponse       ← explicit about layer

// OR: adopt RFC 7807 ProblemDetail fully and drop your custom DTO:
// spring.mvc.problemdetails.enabled=true
// Use ProblemDetail directly in handlers — no custom DTO needed
```

---
## ✅ Validation Exception Handling — Complete Guide

### The Two Validation Exception Types

```
@Valid on @RequestBody    → MethodArgumentNotValidException   (field-level body validation)
@Validated on @PathVar    → ConstraintViolationException      (method-parameter validation)
@RequestParam             ↑ (requires @Validated on the class)
```
### Complete Validation Setup

```java
// ─── Controller ────────────────────────────────────────────────────────────

@RestController
@RequestMapping("/api/v1/users")
@Validated  // ← Required for @PathVariable / @RequestParam constraint validation
public class UserController {

    // @Valid triggers MethodArgumentNotValidException on failure
    @PostMapping
    public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(userService.create(request));
    }

    // @Min triggers ConstraintViolationException on failure
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getById(@PathVariable @Min(1) Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}

// ─── Request DTO ───────────────────────────────────────────────────────────

@Data
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Must be a valid email address")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120, message = "Age seems unrealistic")
    private Integer age;

    @Valid  // ← Cascades validation to nested objects
    @NotNull
    private AddressRequest address;
}

// ─── Global Handler ────────────────────────────────────────────────────────

@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // Handles @Valid on @RequestBody
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        List<FieldViolation> violations = new ArrayList<>();

        // Field errors: @NotBlank, @Email, @Size failed
        ex.getBindingResult().getFieldErrors().forEach(fe ->
            violations.add(new FieldViolation(
                fe.getField(),
                fe.getDefaultMessage(),
                fe.getRejectedValue()
            ))
        );

        // Global/Object errors: @ScriptAssert, cross-field constraints
        ex.getBindingResult().getGlobalErrors().forEach(ge ->
            violations.add(new FieldViolation(
                ge.getObjectName(),
                ge.getDefaultMessage(),
                null
            ))
        );

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .errorCode("VALIDATION_FAILED")
            .message("Request validation failed. Check 'violations' for details.")
            .violations(violations)
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    // Handles @Validated on @PathVariable / @RequestParam
    // NOTE: NOT covered by ResponseEntityExceptionHandler — add it yourself
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex,
            HttpServletRequest request) {

        List<FieldViolation> violations = ex.getConstraintViolations()
            .stream()
            .map(cv -> new FieldViolation(
                // Extract parameter name from path: "getById.id" → "id"
                cv.getPropertyPath().toString().replaceAll(".*\\.", ""),
                cv.getMessage(),
                cv.getInvalidValue()
            ))
            .toList();

        ErrorResponse error = ErrorResponse.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .errorCode("CONSTRAINT_VIOLATION")
            .message("Request parameter validation failed.")
            .violations(violations)
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }
}
```
### Validation Exception Type Decision Tree

```
@Valid / @Validated used in controller
    │
    ├── @RequestBody @Valid      → MethodArgumentNotValidException
    │   (JSON body field validation)
    │
    ├── @RequestParam @Min(1)    → ConstraintViolationException
    │   @PathVariable @NotNull     (requires @Validated on class)
    │
    ├── @ModelAttribute @Valid   → BindException
    │   (Form binding)
    │
    └── @Valid on service method  → ConstraintViolationException
        (requires @Validated on class + spring-boot-starter-validation)
```

---
## 🗄️ `@Transactional` & Exception Rollback Behavior

> [!IMPORTANT] The Most Critical Gotcha — Directly Tied to Checked vs. Unchecked
> This is asked in **nearly every senior Java/Spring interview**. Spring's `@Transactional` rollback behavior is **directly derived** from the Java exception model:
> - **Unchecked (RuntimeException)** → Auto-rollback ✅ (Spring assumes: something unexpected broke, rollback!)
> - **Checked Exception** → NO rollback by default ❌ (Spring assumes: expected operational condition, commit!)
### Complete Rollback Reference

```java
@Service
@Transactional   // Default: rollback on RuntimeException only
public class PaymentService {

    // ──────────────────────────────────────────────────────────────────
    // SCENARIO 1: Runtime Exception — Auto Rollback ✅
    // ──────────────────────────────────────────────────────────────────
    @Transactional
    public void processPayment(Order order) {
        paymentRepository.save(order);           // DB write #1
        invoiceService.generate(order);          // DB write #2
        throw new InsufficientBalanceException(  // RuntimeException
            "Balance too low"
        );
        // RESULT: BOTH writes ROLLED BACK ✅
        // RuntimeException → Spring automatically rolls back
    }

    // ──────────────────────────────────────────────────────────────────
    // SCENARIO 2: Checked Exception — NO Rollback (Spring default) ⚠️
    // ──────────────────────────────────────────────────────────────────
    @Transactional
    public void processOrder(Order order) throws OrderProcessingException {
        orderRepository.save(order);             // DB write #1 — COMMITTED
        emailService.sendConfirmation(order);    // throws OrderProcessingException (CHECKED)
        // RESULT: orderRepository.save() IS COMMITTED!
        // Checked exception does NOT trigger rollback by default.
        // Order is in DB but email was never sent — DATA INCONSISTENCY!
    }

    // ──────────────────────────────────────────────────────────────────
    // SCENARIO 3: Fix — rollbackFor ✅
    // ──────────────────────────────────────────────────────────────────
    @Transactional(rollbackFor = OrderProcessingException.class)
    public void processOrderFixed(Order order) throws OrderProcessingException {
        orderRepository.save(order);
        emailService.sendConfirmation(order);    // throws OrderProcessingException
        // RESULT: ROLLED BACK ✅ — rollbackFor explicitly includes it
    }

    // ──────────────────────────────────────────────────────────────────
    // SCENARIO 4: rollbackFor = Exception.class — include ALL checked ✅
    // ──────────────────────────────────────────────────────────────────
    @Transactional(rollbackFor = Exception.class)
    public void processAll(Order order) throws Exception {
        // Any exception (checked OR unchecked) → ROLLBACK
    }

    // ──────────────────────────────────────────────────────────────────
    // SCENARIO 5: noRollbackFor — exclude a RuntimeException from rollback
    // ──────────────────────────────────────────────────────────────────
    @Transactional(noRollbackFor = FraudWarningException.class)
    public void processWithFraudCheck(Order order) {
        orderRepository.save(order);         // Should be committed even if fraud warning
        fraudService.check(order);           // throws FraudWarningException (RuntimeException)
        // RESULT: COMMITTED despite RuntimeException ← noRollbackFor overrides default
        // Use case: log the fraud warning but don't roll back the order
    }
}
```
### Rollback Decision Matrix

```
                   RuntimeException?        Checked Exception?
                         │                         │
                         ▼                         ▼
           ┌─────────────────────────┬──────────────────────────┐
           │   @Transactional        │   ROLLBACK ✅            │   NO ROLLBACK ❌       │
           │   (default)             │                          │                         │
           ├─────────────────────────┼──────────────────────────┼─────────────────────────┤
           │   rollbackFor=Ex.class  │   ROLLBACK ✅            │   ROLLBACK ✅          │
           ├─────────────────────────┼──────────────────────────┼─────────────────────────┤
           │   noRollbackFor=MyEx    │   NO ROLLBACK ❌         │   NO ROLLBACK ❌       │
           │   (MyEx is RuntimeEx)   │   (for MyEx only)        │                         │
           └─────────────────────────┴──────────────────────────┴─────────────────────────┘
```
### `@Repository` Exception Translation

A Spring-specific feature that's often missed:

```java
// @Repository annotation does TWO things:
// 1. Marks as Spring component (bean registration)
// 2. Enables PersistenceExceptionTranslationPostProcessor
//    which translates JPA/JDBC checked exceptions to Spring's
//    unchecked DataAccessException hierarchy

@Repository
public class UserRepositoryImpl {

    public User findById(Long id) {
        try {
            return entityManager.find(User.class, id);
        } catch (PersistenceException e) {
            // Spring AUTO-TRANSLATES this to DataAccessException
            // (a RuntimeException) via AOP proxy on @Repository
            // You don't need to do anything!
            throw e;  // This PersistenceException becomes DataAccessException
        }
    }
}

// DataAccessException hierarchy (all UNCHECKED — Spring's preference):
DataAccessException                          ← root
├── DataIntegrityViolationException          ← unique constraint, FK violation
├── EmptyResultDataAccessException           ← expected 1 row, got 0
├── QueryTimeoutException                    ← query took too long
├── CannotAcquireLockException               ← DB deadlock / lock timeout
├── TransientDataAccessException             ← worth retrying
│   └── ConcurrencyFailureException
└── NonTransientDataAccessException          ← bug, not worth retrying
    └── InvalidDataAccessApiUsageException   ← wrong API usage

// In GlobalExceptionHandler:
@ExceptionHandler(DataIntegrityViolationException.class)
public ResponseEntity<ErrorResponse> handleDataIntegrity(
        DataIntegrityViolationException ex,
        HttpServletRequest request) {

    log.error("Data integrity violation", ex);

    // Figure out what constraint was violated
    String message = "Data integrity violation";
    if (ex.getMessage().contains("unique constraint")) {
        message = "A record with this information already exists";
    }

    return ResponseEntity.status(HttpStatus.CONFLICT)
        .body(ErrorResponse.of(HttpStatus.CONFLICT, message, request));
}
```

---
## 🔒 Spring Security Exception Handling

> [!WARNING] Critical Architecture Point
> Spring Security operates in the **Filter Chain** — before DispatcherServlet. Therefore, `@ControllerAdvice` does **NOT** handle Spring Security exceptions. You need dedicated security exception handlers.

```
Authentication Request
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  Spring Security Filter Chain                            │
│                                                          │
│  ExceptionTranslationFilter  ← The key security filter   │
│  ├── Catches: AuthenticationException → 401              │
│  │   → Delegates to: AuthenticationEntryPoint            │
│  └── Catches: AccessDeniedException → 403                │
│      → Delegates to: AccessDeniedHandler                 │
└──────────────────────────────────────────────────────────┘
    │ (Only if auth passes)
    ▼
DispatcherServlet → @ControllerAdvice (too late for Security exceptions)
```

```java
// ─── Custom AuthenticationEntryPoint (401) ──────────────────────────────

@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException) throws IOException {

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);

        ErrorResponse error = ErrorResponse.builder()
            .status(401)
            .error("Unauthorized")
            .message("Authentication required. Please provide a valid token.")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        objectMapper.writeValue(response.getWriter(), error);
    }
}

// ─── Custom AccessDeniedHandler (403) ───────────────────────────────────

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);

        ErrorResponse error = ErrorResponse.builder()
            .status(403)
            .error("Forbidden")
            .message("You don't have permission to access this resource.")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();

        objectMapper.writeValue(response.getWriter(), error);
    }
}

// ─── Register in Security Configuration ─────────────────────────────────

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            CustomAuthenticationEntryPoint authEntryPoint,
            CustomAccessDeniedHandler accessDeniedHandler) throws Exception {

        http
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)   // 401 handler
                .accessDeniedHandler(accessDeniedHandler)   // 403 handler
            )
            // ... other config
        ;
        return http.build();
    }
}
```

---
## ⚡ Async Exception Handling — `@Async` Methods

```
Normal method call:
  Thread A → calls method → method throws → Thread A catches → handled

@Async method call:
  Thread A → calls @Async method → Thread A MOVES ON immediately
                                        ↓
                            Thread Pool Thread B executes the method
                            Thread B throws an exception
                            Thread A is GONE — cannot catch this!
                            Only AsyncUncaughtExceptionHandler can catch it
```

```java
// ─── Async Configuration ─────────────────────────────────────────────────

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }

    // Only called for void return type @Async methods
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}

// ─── Async Exception Handler ─────────────────────────────────────────────

@Component
@Slf4j
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

    private final AlertService alertService;

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error(
            "Uncaught exception in @Async method '{}' with params '{}': {}",
            method.getName(),
            Arrays.toString(params),
            ex.getMessage(),
            ex
        );
        alertService.sendCriticalAlert("Async failure: " + method.getName(), ex);
    }
}

// ─── Service with different @Async patterns ───────────────────────────────

@Service
@Slf4j
public class NotificationService {

    // Pattern 1: void return — AsyncUncaughtExceptionHandler is the only safety net
    @Async
    public void sendEmail(String to, String subject) {
        // If this throws, AsyncUncaughtExceptionHandler handles it
        // Caller has NO way to know if this succeeded
        emailClient.send(to, subject);
    }

    // Pattern 2: CompletableFuture — caller CAN observe success/failure
    @Async
    public CompletableFuture<String> sendEmailTracked(String to, String subject) {
        try {
            String result = emailClient.send(to, subject);
            return CompletableFuture.completedFuture(result);
        } catch (EmailException e) {
            log.error("Email failed to {}: {}", to, e.getMessage(), e);
            return CompletableFuture.failedFuture(e);  // Caller can handle
        }
    }
}

// ─── Calling code handling CompletableFuture ─────────────────────────────

@Service
public class OrderService {

    public void placeOrder(Order order) {
        // Fire-and-forget (void @Async) — we don't wait
        notificationService.sendEmail(order.getCustomerEmail(), "Order Confirmed");

        // Tracked @Async — we can observe the result later
        CompletableFuture<String> emailFuture =
            notificationService.sendEmailTracked(order.getCustomerEmail(), "Invoice");

        // Handle completion asynchronously
        emailFuture.whenComplete((result, ex) -> {
            if (ex != null) {
                log.warn("Email notification failed for order {}: {}", order.getId(), ex.getMessage());
                // Schedule retry, update order status, etc.
            }
        });
    }
}
```

---
## 🚧 Exceptions Before DispatcherServlet — Filter Layer

```java
/**
 * JWT Authentication Filter — exception must be handled INSIDE the filter.
 * If we let exceptions propagate out: Tomcat renders an ugly HTML error page.
 * We need JSON errors consistent with the rest of our API.
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtProvider;
    private final ObjectMapper objectMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        try {
            String token = extractToken(request);

            if (token != null) {
                Authentication auth = jwtProvider.validateAndGetAuth(token);
                SecurityContextHolder.getContext().setAuthentication(auth);
            }

            filterChain.doFilter(request, response);  // Continue filter chain

        } catch (JwtExpiredException ex) {
            // ← @ControllerAdvice CANNOT catch this — must handle here
            writeErrorResponse(response, HttpStatus.UNAUTHORIZED,
                               "TOKEN_EXPIRED", "JWT token has expired");

        } catch (JwtMalformedException ex) {
            writeErrorResponse(response, HttpStatus.UNAUTHORIZED,
                               "TOKEN_INVALID", "JWT token is invalid");

        } catch (Exception ex) {
            log.error("Unexpected error in JWT filter", ex);
            writeErrorResponse(response, HttpStatus.INTERNAL_SERVER_ERROR,
                               "FILTER_ERROR", "Authentication processing failed");
        }
    }

    private void writeErrorResponse(HttpServletResponse response,
                                     HttpStatus status,
                                     String errorCode,
                                     String message) throws IOException {

        response.setStatus(status.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);

        // Manually build JSON — spring context may not be fully available
        ErrorResponse error = ErrorResponse.builder()
            .status(status.value())
            .error(status.getReasonPhrase())
            .errorCode(errorCode)
            .message(message)
            .timestamp(Instant.now())
            .build();

        objectMapper.writeValue(response.getWriter(), error);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

---
## 🔌 External Service Exception Handling

### RestTemplate (Spring Boot 2.x / legacy)

```java
@Component
public class PaymentGatewayClient {

    private final RestTemplate restTemplate;

    public PaymentResponse charge(PaymentRequest request) {
        try {
            return restTemplate.postForObject(
                "/api/charge",
                request,
                PaymentResponse.class
            );

        } catch (HttpClientErrorException ex) {
            // 4xx responses — usually client errors
            if (ex.getStatusCode() == HttpStatus.UNPROCESSABLE_ENTITY) {
                throw new CardDeclinedException("Card was declined: " + ex.getResponseBodyAsString(), ex);
            }
            throw new InfrastructureException("Payment gateway client error", ex);

        } catch (HttpServerErrorException ex) {
            // 5xx responses — gateway internal errors
            throw new InfrastructureException(
                "Payment gateway unavailable (HTTP " + ex.getStatusCode() + ")", ex
            );

        } catch (ResourceAccessException ex) {
            // Network failure (connection refused, timeout)
            throw new InfrastructureException("Cannot reach payment gateway", ex);
        }
    }
}

// ─── Custom ResponseErrorHandler ─────────────────────────────────────────

@Component
public class PaymentGatewayErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().is4xxClientError()
            || response.getStatusCode().is5xxServerError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        if (response.getStatusCode() == HttpStatus.NOT_FOUND) {
            throw new ResourceNotFoundException("Payment", "unknown");
        }
        if (response.getStatusCode().is5xxServerError()) {
            throw new InfrastructureException("Gateway server error: " + response.getStatusCode(), null);
        }
    }
}
```
### WebClient (Spring Boot 3.x / Reactive)

```java
@Component
public class ModernPaymentClient {

    private final WebClient webClient;

    public PaymentResponse charge(PaymentRequest request) {
        return webClient.post()
            .uri("/api/charge")
            .bodyValue(request)
            .retrieve()
            // Map 4xx → custom exception
            .onStatus(HttpStatusCode::is4xxClientError, clientResponse ->
                clientResponse.bodyToMono(String.class)
                    .map(body -> new CardDeclinedException("Declined: " + body))
            )
            // Map 5xx → infrastructure exception
            .onStatus(HttpStatusCode::is5xxServerError, clientResponse ->
                Mono.error(new InfrastructureException("Gateway error", null))
            )
            .bodyToMono(PaymentResponse.class)
            .block();  // sync in non-reactive context
    }
}
```
### Feign Client (Microservices)

```java
// ─── Feign Interface ─────────────────────────────────────────────────────

@FeignClient(
    name = "payment-service",
    url  = "${services.payment.url}",
    configuration = PaymentFeignConfig.class
)
public interface PaymentServiceClient {

    @PostMapping("/api/payments/charge")
    PaymentResponse charge(@RequestBody PaymentRequest request);
}

// ─── Custom Error Decoder ─────────────────────────────────────────────────

@Configuration
public class PaymentFeignConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new PaymentErrorDecoder();
    }
}

public class PaymentErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        String body = extractBody(response);

        return switch (response.status()) {
            case 404 -> new ResourceNotFoundException("Payment", "unknown");
            case 422 -> new CardDeclinedException("Card declined: " + body);
            case 503 -> new InfrastructureException("Payment service unavailable", null);
            default  -> new InfrastructureException(
                            "Unexpected payment service error: HTTP " + response.status(), null
                         );
        };
    }

    private String extractBody(Response response) {
        try (InputStream is = response.body().asInputStream()) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        } catch (IOException e) {
            return "Could not read response body";
        }
    }
}
```

---

## 📋 RFC 7807 ProblemDetail — Spring Boot 3+

Spring Framework 6 (Spring Boot 3+) introduced native support for [RFC 7807 "Problem Details for HTTP APIs"](https://www.rfc-editor.org/rfc/rfc7807).

```properties
# application.properties — Enable auto-handling with ProblemDetail format
spring.mvc.problemdetails.enabled=true
```

```json
// RFC 7807 Standard Response Shape:
{
  "type":     "https://api.example.com/errors/insufficient-balance",
  "title":    "Insufficient Balance",
  "status":   402,
  "detail":   "Current balance 50.00 is less than required amount 150.00",
  "instance": "/api/v1/payments/TXN-001",
  "balance":  50.00,    ← custom extension properties
  "required": 150.00    ← custom extension properties
}
```

```java
// ─── Using ProblemDetail in @RestControllerAdvice ─────────────────────────

@RestControllerAdvice
public class ProblemDetailExceptionHandler extends ResponseEntityExceptionHandler {
    // ResponseEntityExceptionHandler already uses ProblemDetail internally in Spring 6

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex,
                                         HttpServletRequest request) {

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            ex.getMessage()
        );
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setInstance(URI.create(request.getRequestURI()));

        // Extension properties (custom fields)
        problem.setProperty("errorCode",   ex.getErrorCode());
        problem.setProperty("resourceType", ex.getResourceType());
        problem.setProperty("timestamp",   Instant.now());

        return problem;
    }

    @ExceptionHandler(InsufficientBalanceException.class)
    public ProblemDetail handleInsufficientBalance(InsufficientBalanceException ex) {

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.PAYMENT_REQUIRED,
            ex.getMessage()
        );
        problem.setTitle("Insufficient Balance");
        problem.setType(URI.create("https://api.example.com/errors/insufficient-balance"));
        problem.setProperty("currentBalance", ex.getCurrentBalance());
        problem.setProperty("requiredAmount", ex.getRequiredAmount());
        problem.setProperty("shortfall",      ex.getShortfall());

        return problem;
    }
}

// ─── ErrorResponseException (Spring 6) ───────────────────────────────────
// Alternative: your exception implements the ErrorResponse interface

public class ResourceNotFoundException extends RuntimeException
        implements ErrorResponse {  // Spring 6 interface

    private final ProblemDetail body;

    public ResourceNotFoundException(String resourceType, Object id) {
        super(resourceType + " not found with id: " + id);
        this.body = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            this.getMessage()
        );
        this.body.setProperty("resourceType", resourceType);
    }

    @Override
    public HttpStatusCode getStatusCode() { return HttpStatus.NOT_FOUND; }

    @Override
    public ProblemDetail getBody() { return body; }
}
// When this exception is thrown, Spring 6 auto-renders it as ProblemDetail JSON
// No @ExceptionHandler needed!
```

---
## 🗺️ Grand Architecture Diagram

```
╔═════════════════════════════════════════════════════════════════════════════╗
║           SPRING BOOT EXCEPTION HANDLING — COMPLETE ARCHITECTURE            ║
╚═════════════════════════════════════════════════════════════════════════════╝

HTTP REQUEST
     │
     ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  LAYER 0: SERVLET FILTER CHAIN                                             │
│  ─────────────────────────────────────────────────────────────────────     │
│  Filters: JwtFilter, CorsFilter, RateLimitFilter, LoggingFilter            │
│                                                                            │
│  ❌ @ControllerAdvice DOES NOT REACH HERE                                  │
│  ✅ MUST: try-catch inside filter + write JSON to HttpServletResponse      │
│  ✅ Spring Security: ExceptionTranslationFilter handles Auth/Access errors │
│     → AuthenticationException → AuthenticationEntryPoint → 401             │
│     → AccessDeniedException   → AccessDeniedHandler       → 403            │
└──────────────────────────┬─────────────────────────────────────────────────┘
                           │ (auth passes)
                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: DISPATCHERSERVLET                                                │
│  All exception handling below this line is managed by Spring MVC           │
└──────────────────────────┬─────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: CONTROLLER / SERVICE / REPOSITORY                                │
│                                                                            │
│  @RestController          → throws domain exceptions (clean!)              │
│      ↓ calls                                                               │
│  @Service                 → throws domain exceptions                       │
│      ↓ calls                                                               │
│  @Repository              → Spring auto-translates JPA/JDBC exceptions     │
│                              to DataAccessException (unchecked) via AOP    │
│                                                                            │
│  ┌─────────────────────────────────────────────────────┐                   │
│  │  LOCAL @ExceptionHandler (controller-level)         │ ← Most specific   │
│  │  Only handles exceptions from THIS controller       │                   │
│  └─────────────────────────────────────────────────────┘                   │
└──────────────────────────┬─────────────────────────────────────────────────┘
                           │ 💥 Exception propagates
                           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: HandlerExceptionResolver CHAIN                                  │
│  ─────────────────────────────────────────────────────────────────────    │
│                                                                           │
│  ① ExceptionHandlerExceptionResolver   (Highest priority)                 │
│  ├── Scans @ExceptionHandler in current @Controller (local)               │
│  └── Scans @ExceptionHandler in ALL @ControllerAdvice beans (global)      │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  @RestControllerAdvice (GlobalExceptionHandler)                     │  │
│  │  extends ResponseEntityExceptionHandler ← handles Spring MVC excep  │  │
│  │                                                                     │  │
│  │  @ExceptionHandler(ResourceNotFoundException.class) → 404           │  │
│  │  @ExceptionHandler(ConflictException.class)         → 409           │  │
│  │  @ExceptionHandler(BusinessRuleViolationException)  → 422           │  │
│  │  @ExceptionHandler(InfrastructureException.class)   → 503           │  │
│  │  @ExceptionHandler(ConstraintViolationException)    → 400           │  │
│  │  Override handleMethodArgumentNotValid()            → 400 + fields  │  │
│  │  Override handleHttpMessageNotReadable()            → 400           │  │
│  │  @ExceptionHandler(DataIntegrityViolationException) → 409           │  │
│  │  @ExceptionHandler(Exception.class)                 → 500 (catch-all│  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ② ResponseStatusExceptionResolver   (Medium priority)                    │
│  ├── @ResponseStatus on exception class → maps to that HTTP status        │
│  └── ResponseStatusException → uses embedded status                       │
│                                                                           │
│  ③ DefaultHandlerExceptionResolver    (Lowest priority)                   │
│  └── Spring MVC built-ins → sends status code only (no JSON body)         │
└──────────────────────────┬────────────────────────────────────────────────┘
                           │ (if no resolver handled it)
                           ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: BASIC ERROR CONTROLLER (Last Resort)                            │
│  ─────────────────────────────────────────────────────────────────────    │
│  Request forwarded to /error endpoint                                     │
│  BasicErrorController → renders Whitelabel page (HTML) or JSON            │
│                                                                           │
│  ✅ Override: implement ErrorController for custom /error behavior        │
│  ✅ Override: implement ErrorAttributes for custom attribute shape        │
└───────────────────────────────────────────────────────────────────────────┘

═════════════════════════════════════════════════════════════════════════════
SPECIAL DOMAINS — THEIR OWN EXCEPTION HANDLING PATHS:
═════════════════════════════════════════════════════════════════════════════

@Async Methods:
  Void return   → AsyncUncaughtExceptionHandler (separate thread)
  CompletableFuture → caller handles via .exceptionally() / .whenComplete()

@Transactional:
  RuntimeException  → AUTO ROLLBACK (Spring default)
  CheckedException  → NO ROLLBACK (Spring default) ← common gotcha!
  rollbackFor / noRollbackFor → explicit overrides

Spring Security:
  AuthenticationException → AuthenticationEntryPoint → 401
  AccessDeniedException   → AccessDeniedHandler      → 403
  (Both: Filter-level, BEFORE DispatcherServlet)

Feign Client:
  HTTP 4xx/5xx → ErrorDecoder → translate to domain exceptions

@Repository (AOP Proxy):
  JPA PersistenceException → DataAccessException (unchecked)
  (Automatic via @Repository + Spring's PersistenceExceptionTranslationAdvisor)

═══════════════════════════════════════════════════════════════════════════════
TRANSACTION × EXCEPTION DECISION:
  Runtime Unchecked         → ROLLBACK ✅
  Checked / @Transactional  → NO ROLLBACK ❌ (add rollbackFor!)
═══════════════════════════════════════════════════════════════════════════════
```

---

## 🎯 Key Interview Questions & Answers

### Q1: What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?

> `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`.
> `@ControllerAdvice` without `@ResponseBody` would try to resolve handler return values as **view names** (for template-based MVC). Since REST APIs return JSON, `@RestControllerAdvice` ensures handler method return values are **serialized to JSON**. For REST APIs, always use `@RestControllerAdvice`.

---

### Q2: Why doesn't `@ControllerAdvice` catch exceptions thrown in a Spring Security Filter?

> `@ControllerAdvice` handlers are invoked by the `DispatcherServlet`. Spring Security filters execute **before** `DispatcherServlet` in the Servlet filter chain. If a filter throws, the request never reaches `DispatcherServlet`, so the `HandlerExceptionResolver` chain (which backs `@ControllerAdvice`) is never triggered. Solutions: handle in the filter itself using `try-catch`, use `AuthenticationEntryPoint` / `AccessDeniedHandler` for security exceptions.

---

### Q3: By default, does `@Transactional` rollback on checked exceptions?

> **No.** By default, Spring only rolls back on `RuntimeException` (and `Error`). For checked exceptions, you must explicitly add `rollbackFor = YourCheckedException.class`. This is a common production bug: an external API call throws a checked exception mid-transaction, the DB write before it gets committed, leading to data inconsistency.

---

### Q4: How do you handle `MethodArgumentNotValidException` and `ConstraintViolationException` differently?

> - `MethodArgumentNotValidException`: Thrown when `@Valid` on `@RequestBody` fails. Handled by overriding `handleMethodArgumentNotValid()` in `ResponseEntityExceptionHandler`.
> - `ConstraintViolationException`: Thrown when `@Validated` + `@Min`/`@NotBlank` etc. on `@PathVariable`/`@RequestParam` fails. NOT covered by `ResponseEntityExceptionHandler` — must add a separate `@ExceptionHandler(ConstraintViolationException.class)` in your advice class.

---

### Q5: How does `@Repository` exception translation work?

> When a class is annotated with `@Repository`, Spring's AOP infrastructure wraps it in a proxy that intercepts JPA/JDBC exceptions and translates them to Spring's `DataAccessException` hierarchy (unchecked). This is done via `PersistenceExceptionTranslationPostProcessor`. The benefit: service layers don't need to depend on JPA-specific or JDBC-specific exception types, making the codebase portable and the exceptions naturally propagate to `@ControllerAdvice`.

---

### Q6: How do you handle exceptions thrown in `@Async` methods?

> For `void` return `@Async` methods: Implement `AsyncUncaughtExceptionHandler` and register it in `AsyncConfigurer`. The handler receives the exception, method reference, and parameters.  
> For `CompletableFuture` return `@Async` methods: Return `CompletableFuture.failedFuture(e)` on exception — the caller can use `.exceptionally()` or `.whenComplete()` to observe failure. The `AsyncUncaughtExceptionHandler` is NOT called for `Future`-returning methods.

---

### Q7: What is `ResponseEntityExceptionHandler` and why extend it?

> It's a Spring-provided base class for `@ControllerAdvice` that already handles the ~15 standard Spring MVC exceptions (like `MethodArgumentNotValidException`, `NoHandlerFoundException`, `HttpRequestMethodNotSupportedException`). Without extending it, these exceptions are handled by `DefaultHandlerExceptionResolver` which sends an empty body response with just an HTTP status code. By extending and overriding its methods, you ensure these Spring MVC exceptions also return your consistent custom error JSON format.

---

### Q8: How do you ensure exception cause is never lost when translating exceptions across layers?

> Always pass the original exception as the `cause` to the new exception's constructor:
> ```java
> catch (SQLException e) {
>     throw new DataAccessException("DB operation failed", e);  // ← e preserved as cause
> }
> ```
> Without this, `getCause()` returns `null` and `printStackTrace()` shows only your custom exception with no root cause — making debugging in production nearly impossible.

---

### Q9: Checked vs. Unchecked exceptions in Spring — which should custom exceptions extend?

> Spring itself strongly prefers **unchecked exceptions** for the same reasons it built its entire exception hierarchy (DataAccessException, BeansException) as unchecked:
> - Unchecked exceptions **compose cleanly with lambdas and Stream APIs** (functional interfaces can't declare checked exceptions)
> - Services don't need `throws` declarations — cleaner signatures
> - Exceptions naturally propagate to `@RestControllerAdvice` without forced try-catch boilerplate
> - The "catch-ignore" anti-pattern is less tempting
>
> **Rule of thumb:** extend `RuntimeException` for all custom Spring Boot application exceptions unless you have a very specific reason (e.g., you're designing a library API where forcing callers to acknowledge specific failures is an explicit design goal).

---

### Q10: Describe the complete exception handling flow for a JWT validation failure.

> 1. Request enters the Servlet filter chain
> 2. `JwtAuthenticationFilter` (a `OncePerRequestFilter`) processes the `Authorization` header
> 3. `jwtProvider.validateAndGetAuth(token)` throws `JwtExpiredException`
> 4. The `catch (JwtExpiredException ex)` block in the filter executes
> 5. The filter writes a 401 JSON error response directly to `HttpServletResponse` using `ObjectMapper`
> 6. The filter chain is broken — no further filter, no `DispatcherServlet`, no `@ControllerAdvice`
>
> If the filter doesn't catch this, the exception propagates out of the filter chain and Tomcat renders an HTML error page (or Spring Boot's Whitelabel page) — breaking your API's consistent JSON error format.

---

## ⚡ Best Practices & Anti-Patterns

### ✅ Best Practices

```java
// 1. SINGLE global handler — one place, consistent format
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler { }

// 2. ALWAYS chain causes
throw new DomainException("DB write failed", originalException);  // ✅
throw new DomainException("DB write failed");                     // ❌ cause lost

// 3. Specific before general — order matters in @ControllerAdvice
@ExceptionHandler(ResourceNotFoundException.class)  // specific first
@ExceptionHandler(ApplicationException.class)       // general second
@ExceptionHandler(Exception.class)                  // catch-all last

// 4. Never expose internals in production
// ❌ BAD:
.message("NullPointerException at UserService.java:47 in method getUser")
// ✅ GOOD:
.message("An unexpected error occurred. Reference: " + traceId)

// 5. Correct logging level per exception type
log.warn(...)   // for ResourceNotFoundException, ValidationException (expected)
log.error(...)  // for InfrastructureException, unexpected Exception (unexpected)

// 6. Include trace ID in error response (distributed tracing)
String traceId = MDC.get("traceId");  // set by logging filter
ErrorResponse.builder().traceId(traceId).build();

// 7. Rollback for checked exceptions explicitly
@Transactional(rollbackFor = Exception.class)  // when in doubt, rollback for everything

// 8. Validate early in the controller/service
Objects.requireNonNull(request.getEmail(), "Email must not be null");

// 9. Translate infrastructure exceptions at boundaries
// @Repository layer → DataAccessException (Spring auto)
// External HTTP client → your domain exception (you do manually)
```

### ❌ Anti-Patterns

```java
// 1. Silent swallowing ← worst possible
catch (Exception e) { }

// 2. Log AND rethrow at every layer ← log spam, 5 identical stack traces
catch (Exception e) {
    log.error("Error", e);  // logged here
    throw e;                // logged by caller, and by their caller...
}
// ✅ RULE: Log ONCE at the outermost boundary (GlobalExceptionHandler)

// 3. Returning null on exception ← hides bugs
catch (UserNotFoundException e) {
    return null;  // ❌ caller gets null, NullPointerException later, impossible to debug
}

// 4. Generic throws Exception on service methods
public UserDto getUser(Long id) throws Exception { ... }  // ❌ tells caller nothing

// 5. Using Exception for flow control
try {
    Integer.parseInt(input);  // ❌ expensive stack trace for expected condition
} catch (NumberFormatException e) {
    return 0;
}

// 6. Exposing SQL error details to API response ← security risk
catch (SQLException e) {
    throw new RuntimeException(e.getMessage());
    // "ERROR: duplicate key value violates unique constraint users_email_key"
    // ← tells attackers your schema!
}

// 7. @Transactional + checked exception + no rollbackFor ← silent data corruption
@Transactional
public void save(Entity e) throws SomeCheckedException {
    repo.save(e);               // committed!
    externalService.call(e);   // throws SomeCheckedException
    // ← repo.save() was committed. INCONSISTENCY. Bug. ← most dangerous anti-pattern
}

// 8. Catching Throwable in application code
catch (Throwable t) { log.error(t); continue; }  // catches Errors too — dangerous!

// 9. Custom exception extending Throwable directly
class MyException extends Throwable { }  // ❌ Use Exception or RuntimeException
```

### Logging Strategy Reference

```
Exception Category          │  Log Level  │  Include Stack Trace?
────────────────────────────┼─────────────┼──────────────────────
ResourceNotFoundException    │  WARN       │  No (expected condition)
ValidationException          │  WARN       │  No (client error)
ConflictException            │  WARN       │  No (expected)
BusinessRuleViolationException│ WARN       │  No (expected)
InfrastructureException      │  ERROR      │  Yes (unexpected system)
DataAccessException          │  ERROR      │  Yes (DB problem)
RuntimeException (unhandled) │  ERROR      │  Yes (bug)
Exception (catch-all)        │  ERROR      │  Yes (completely unexpected)
```

---

## 📝 Quick Revision Card

```
┌──────────────────────────────────────────────────────────────────────────┐
│  SPRING BOOT EXCEPTION HANDLING — 5-LAYER CHEAT SHEET                   │
├──────────────────────────────────────────────────────────────────────────┤
│  FILTER LAYER          │ Handle inside filter.                           │
│  (Before Servlet)      │ @ControllerAdvice CANNOT reach.                 │
│                        │ Security: EntryPoint (401), DeniedHandler (403) │
├──────────────────────────────────────────────────────────────────────────┤
│  LOCAL HANDLER         │ @ExceptionHandler in @Controller                │
│  (Controller-level)    │ Only for THAT controller. Rarely used.           │
├──────────────────────────────────────────────────────────────────────────┤
│  GLOBAL HANDLER        │ @RestControllerAdvice ← PRIMARY PATTERN         │
│  (Most Important)      │ extends ResponseEntityExceptionHandler          │
│                        │ Handles: domain, validation, infrastructure     │
├──────────────────────────────────────────────────────────────────────────┤
│  PROGRAMMATIC          │ @ResponseStatus on class                        │
│  (Simple cases)        │ ResponseStatusException (inline)                │
├──────────────────────────────────────────────────────────────────────────┤
│  LAST RESORT           │ BasicErrorController → /error                   │
│                        │ Whitelabel Page. Replace with ErrorController.  │
├──────────────────────────────────────────────────────────────────────────┤
│  @TRANSACTIONAL KEY RULE:                                                │
│  RuntimeException    → ROLLBACK ✅ (automatic)                           │
│  CheckedException    → NO ROLLBACK ❌ (add rollbackFor explicitly!)      │
├──────────────────────────────────────────────────────────────────────────┤
│  @ASYNC KEY RULE:                                                        │
│  void return         → AsyncUncaughtExceptionHandler                     │
│  CompletableFuture   → .exceptionally() / .whenComplete() by caller      │
├──────────────────────────────────────────────────────────────────────────┤
│  CUSTOM EXCEPTIONS:                                                      │
│  Extend RuntimeException (Spring prefers unchecked)                      │
│  Always chain cause: super(message, cause)                               │
│  Provide errorCode (machine-readable) + message (human-readable)         │
├──────────────────────────────────────────────────────────────────────────┤
│  VALIDATION:                                                             │
│  @Valid on @RequestBody  → MethodArgumentNotValidException               │
│  @Validated + @Min etc.  → ConstraintViolationException                  │
│  Override handleMethodArgumentNotValid() for custom JSON                 │
│  Add @ExceptionHandler(ConstraintViolationException) manually            │
├──────────────────────────────────────────────────────────────────────────┤
│  @REPOSITORY:                                                            │
│  JPA/JDBC exceptions → DataAccessException (auto, via AOP proxy)        │
├──────────────────────────────────────────────────────────────────────────┤
│  SPRING BOOT 3 / SPRING 6:                                               │
│  ProblemDetail (RFC 7807): spring.mvc.problemdetails.enabled=true        │
│  ErrorResponse interface: exceptions can self-describe their HTTP shape  │
└──────────────────────────────────────────────────────────────────────────┘

GOLDEN RULES:
  ✅  One GlobalExceptionHandler — centralized, consistent
  ✅  Always chain causes: super(message, cause)
  ✅  Log once at global boundary — not at every layer
  ✅  Never expose internals (stack traces, SQL, file paths) in responses
  ✅  rollbackFor = Exception.class when mixing checked + transactions
  ✅  Handle in filter for filter-thrown exceptions (JWT, rate limit)
  ❌  catch (Exception e) { }  — silent swallowing is forbidden
  ❌  cause-less exception construction — breaks production debugging
  ❌  Extend Throwable or Error for custom exceptions — semantic abuse
```

---

*Related Notes: `[[Java Exceptions — Intuitive Architecture & Deep Mastery]]` · `[[Spring Boot Validation Deep Dive]]` · `[[Spring Security Architecture]]` · `[[@Transactional Deep Dive]]` · `[[Microservices Exception Propagation Patterns]]`*

```
tags: #spring-boot #exceptions #rest-api #global-handler #controlleradvice  
      #transactional #validation #async #filter #feign #problemdetail #interview
```