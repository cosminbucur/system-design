# Spring Cloud Config: Architecture, Use Cases, and Code Samples

**Spring Cloud Config** provides server-side and client-side support for externalized configuration in a distributed system. In a microservices architecture, it acts as a single, centralized source of truth for all application configuration properties across multiple environments (Development, Staging, Production).

---

## How It Works

The architecture relies on two main components:

1. **The Config Server:** 
   - A dedicated Spring Boot application annotated with `@EnableConfigServer`.
   - It typically connects to a backend version-control system (like a Git repository) or HashiCorp Vault where raw configuration files (`.properties` or `.yml`) are stored.
   - It exposes a clean HTTP resource-based API that maps client requests using three parameters: `{application}` (maps to `spring.application.name`), `{profile}` (maps to active Spring profiles), and `{label}` (maps to a Git branch or commit hash).

2. **The Config Client:**
   - Any standard Spring Boot microservice.
   - During startup, it connects to the Config Server using `spring.config.import`, fetches its configuration profile, and binds the remote properties directly into the Spring `Environment` before the application context fully initializes.

---

## Real-Life Use Cases

* **Zero-Downtime Configuration Changes:** Changing database connection pools, feature toggles, or rate-limiting thresholds globally across dozens of service instances without forcing code redeployments. Combined with Spring Cloud Actuator's `/refresh` endpoint or Spring Cloud Bus, beans annotated with `@RefreshScope` update their values dynamically at runtime.
* **Environment Isolation & Gitops Workflow:** Keeping environment-specific properties (e.g., prod database URLs vs. local H2 URLs) isolated in a secure Git repository, allowing teams to audit changes via pull requests.
* **Secret Management & Encryption:** Storing sensitive values (like API keys or passwords) encrypted in the configuration repository. The Config Server can automatically decrypt these values before serving them to authorized clients, or integrate directly with HashiCorp Vault.

---

## Code Implementation Samples

### 1. Setting Up the Config Server

**`pom.xml` dependencies:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

**Main Application Class:**
```java
package com.example.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**`application.yml` (Server):**
```yaml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
```

*(Assuming your Git repo contains a file named `order-service-prod.yml` with properties like `app.discount-rate=15`)*

---

### 2. Setting Up the Config Client

**`pom.xml` dependencies:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

**`application.yml` (Client):**
Using modern Spring Boot configuration import syntax (`spring.config.import`), the client knows where to fetch its properties before building the context:
```yaml
spring:
  application:
    name: order-service
  profiles:
    active: prod
  config:
    import: optional:configserver:http://localhost:8888
```

**Consuming and Dynamically Refreshing Properties:**
To allow runtime reloading of configuration properties without a full app restart, use `@RefreshScope` on your component:

```java
package com.example.orderservice;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope
public class ConfigController {

    @Value("${app.discount-rate:0}")
    private int discountRate;

    @GetMapping("/discount")
    public String getDiscount() {
        return "Current discount rate: " + discountRate + "%";
    }
}
```

If you update `app.discount-rate` in your Git repository, commit, and send a POST request to `http://localhost:<client-port>/actuator/refresh`, the client will fetch the latest property values and update beans annotated with `@RefreshScope` instantly.