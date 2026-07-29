---
type: concept
area: java, spring boot
topic: spring boot
status: solid
confidence: 3
last-reviewed: 2026-07-23
tags:
  - high-yield
---
- what "auto" in `AutoConfiguration` means ? 
- how "auto configuration works? (no magic - somebody did the hard work of writing default config of ~150 classes)
- auto-detection & auto-backoff (Fallback Configuration).
- how to write Custom Auto Configuration.
- list of the standard conditional annotations provided by Spring Boot.
## What is "Auto" in Auto-Configuration?
The word **"Auto"** means three specific things:
```css
┌─────────────────────────────────────────────────────────────────┐ 
│                  What "AUTO" actually means                     │ ├─────────────────────────────────────────────────────────────────┤
│                                                                 │ 
│  1. AUTOMATIC DETECTION                                         │ 
│     Spring Boot detects what libraries you added                │ 
│     by checking what classes are on the classpath.              │ 
│     You don't register anything — it finds it.                  │ 
│                                                                 │ 
│  2. AUTOMATIC CONFIGURATION                                     │ 
│     Based on what it detects, Spring Boot creates               │ 
│     the right beans with sensible default settings.             │ 
│     You don't write the @Bean methods — it writes them.         │ 
│                                                                 │ 
│  3. AUTOMATIC BACK-OFF                                          │ 
│     If you define your own bean, Spring Boot steps back.        │ 
│     Your configuration always wins over auto-configuration.     │ 
│     It never overrides your decisions.                          │ 
│                                                                 │ └─────────────────────────────────────────────────────────────────┘
```

> [!tip] The Mental Model 
> Think of Auto-Configuration like a **smart assistant** who says: _"I see you brought PostgreSQL. I know the standard way to set that up. I'll do it for you — but if you want to do it differently, just tell me and I'll step aside."_

## Write Custom Auto Configuration 

```less
[META-INF/...AutoConfiguration.imports]
                   │
                   ▼ (Spring Boot finds the supervisor)
     [MyFeatureAutoConfiguration]
                   │
                   ▼ (Supervisor checks conditions and builds the worker)
         [MyFeatureService Bean]

```

Building a custom auto-configuration allows you to package reusable components into a library that automatically registers beans when added as a dependency to another project. 

Here is a summary of the steps followed by a concrete code example.

Summary of Steps
1. **Define the Core Component**: Create the service or component class you want to offer.
2. **Create Configuration with Conditions**: Write a `@Configuration` class that initializes your component using conditions (e.g., `@ConditionalOnMissingBean`, `@ConditionalOnClass`) so it stays flexible. e.g. `com.example.MyFeatureAutoConfiguration`
3. **Register via the `.imports` File**: Tell Spring Boot **about your configuration** by listing its **fully qualified name** inside `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
	1. paste `com.example.MyFeatureAutoConfiguration`

---

Complete Code Example

### Step 1: Create the Component
This is the service class that other projects will want to use automatically.

```java
package com.example.log.service;

public class CustomLoggerService {
    private final String prefix;

    public CustomLoggerService(String prefix) {
        this.prefix = prefix;
    }

    public void logMessage(String message) {
        System.out.println("[" + prefix + "] " + message);
    }
}
```
### Step 2: Create the Auto-Configuration Class
We use conditional annotations here to make sure this bean is _only_ created if the consuming project hasn't already defined its own version.

```java
package com.example.log.config;

import com.example.log.service.CustomLoggerService;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnClass(CustomLoggerService.class) // Activates only if this service class exists on classpath
// can be conditioned on anything not neccesarily on the service class 
public class CustomLoggerAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean // Activates only if the consuming app hasn't declared its own CustomLoggerService bean
    public CustomLoggerService customLoggerService() {
        return new CustomLoggerService("AUTO-CONFIG");
    }
}
```
### Step 3: Register via the `.imports` File
For modern Spring Boot versions, create a file named exactly **`org.springframework.boot.autoconfigure.AutoConfiguration.imports`** in this directory structure: `src/main/resources/META-INF/spring/`.

Inside that file, write the fully qualified name of your configuration class: 

```text
com.example.log.config.CustomLoggerAutoConfiguration
```
### How a Consumer Uses It
When someone includes your packaged JAR file in their project dependencies, Spring Boot scans that `.imports` file at startup. A developer can immediately inject your service into their controllers or services with zero configuration lines: 

```java
@RestController
public class MyController {
    
    private final CustomLoggerService logger;

    // Automatically injected out-of-the-box by your AutoConfiguration library!
    public MyController(CustomLoggerService logger) {
        this.logger = logger;
    }
}
```

### Conditional Annotations for Custom Auto Configuration

Here is the exhaustive categorized list of the standard conditional annotations provided by Spring Boot. You can mix and match these on your custom auto-configuration classes or directly on individual `@Bean` methods to fine-tune exactly when your components should activate. 
#### 1. Class-Based Conditions
These annotations check the project's classpath dependencies. They are highly used in libraries to activate code only when external drivers or libraries are present.  
- **`@ConditionalOnClass`**: Activates only if the specified class or classes are present on the classpath.
- **`@ConditionalOnMissingClass`**: Activates only if the specified class or classes are **not** present on the classpath. 
#### 2. Bean-Based Conditions
These annotations check the current state of the Spring Application Context. They prevent duplicate bean registration conflicts. 
- **`@ConditionalOnBean`**: Activates only if a bean of the specified class or name is already registered in the application context.
- **`@ConditionalOnMissingBean`**: Activates only if no bean of the specified class or name has been registered yet. This is the ultimate tool for allowing developers to override your default starter beans. 
#### 3. Property-Based Conditions
These annotations read configuration values out of environment variables, system properties, or `application.properties`/`application.yml` files.
- **`@ConditionalOnProperty`**: Activates based on the presence, absence, or value of a specific configuration property key.
    - _Useful parameters:_ `name`, `havingValue`, and `matchIfMissing=true` (lets you define default fallback activation states).
#### 4. Web Application Conditions
These annotations check what kind of web architecture environment the Spring application is running within. 
- **`@ConditionalOnWebApplication`**: Activates only if the current application is a web application. You can narrow this down using its `type` attribute to target specific stacks:
    - `Type.SERVLET` (Standard Spring MVC)
    - `Type.REACTIVE` (Spring WebFlux)
- **`@ConditionalOnNotWebApplication`**: Activates only if the application is **not** a web application (e.g., a batch processing CLI app or basic worker application). 
#### 5. Resource and File Conditions
These annotations look into the internal or external file storage layer at boot time.
- **`@ConditionalOnResource`**: Activates only if a specific file resource is available on the classpath (e.g., `resources: @ConditionalOnResource(resources = "classpath:schema.sql")`). 
#### 6. Cloud and Platform Conditions
These annotations look outward at the system hosting your application runtime container.
- **`@ConditionalOnCloudPlatform`**: Activates only if the application is running on a specified cloud platform provider.
    - _Supported platforms:_ `CloudPlatform.KUBERNETES`, `CloudPlatform.HEROKU`, `CloudPlatform.CLOUD_FOUNDRY`, `CloudPlatform.SAP`. 
- **`@ConditionalOnExpression`**: The "nuclear option." Activates based on a complex custom **SpEL (Spring Expression Language)** statement string. (e.g., `@ConditionalOnExpression("'${my.custom.property}' == 'dev' and ${my.other.toggle:true}")`). Use this sparingly, as it bypasses compile-time checks.
