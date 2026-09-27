# Integration Testing with Spring Boot & Testcontainers

Integration testing verifies that different layers of your application—such as your Spring context, database, cache, and messaging queues—work together correctly.

Using **Spring Boot** alongside **Testcontainers** provides real infrastructure dependencies running inside short-lived Docker containers, replacing slow, unmaintained, or inaccurate in-memory alternatives (like H2 or Embedded Kafka).

---

## 1. Core Concepts

### Spring Context Management
* **`@SpringBootTest`**: Loads the full application context. It tests the end-to-end integration of controllers, services, repositories, and configurations.
* **Test Slices (e.g., `@DataJpaTest`, `@WebMvcTest`)**: Load only a specific subset of the Spring context (e.g., repository layer only), making them faster than loading the entire application context.
* **Context Caching**: Spring caches and reuses the `ApplicationContext` across test classes that share identical configurations to keep test suites fast.

### Testcontainers Core Mechanisms
* **Disposable Infrastructure**: Spins up real Docker containers (PostgreSQL, Redis, Kafka) during test execution and tears them down when the test suite finishes.
* **Dynamic Property Injection**: Because Testcontainers binds containers to random available host ports to avoid port collisions, connection properties (like host and port) must be injected into Spring's environment dynamically at startup.
* **`@ServiceConnection` (Spring Boot 3.1+)**: Replaces verbose configuration boilerplate. It automatically detects the container type (e.g., `PostgreSQLContainer`) and wires connection properties (e.g., `spring.datasource.url`) straight into the Spring Environment.

---

## 2. Best Practices

1. **Avoid In-Memory Databases for Production Databases**  
   Use real database engines in containers rather than H2. H2 lacks database-specific functions, indexing behavior, JSON types, and strict locking behavior.
2. **Share Container Instances (Singleton Pattern)**  
   Starting a new container for every test class adds 2–5 seconds per class. Prefer sharing container definitions across test classes using a base abstract class.
3. **Control Database State Explicitly**  
   Ensure test isolation without restarting containers. Use `@Transactional` on test methods to auto-rollback state changes, or run schema cleanup tools (e.g., Flyway/Liquibase or database truncating) before each test.
4. **Use Fixed, Lightweight Image Tags**  
   Always pin container versions (e.g., `postgres:16-alpine`) instead of using `latest` to ensure reproducible builds in CI environments and faster pull times.
5. **Decouple Container Lifecycle for Dev Environments (`@TestConfiguration`)**  
   Leverage `spring-boot-testcontainers` with `TestApplication` for local development (`bootTestRun`) to share live containers with your development session without starting app-embedded Docker setups manually.

---

## 3. Practical Example: Spring Boot 3 + PostgreSQL Testcontainer

### Dependencies (`build.gradle` snippet)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // Test dependencies
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers' // Spring Boot 3.1+ integration
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:postgresql'
}
```

---

### Step 1: Base Abstract Integration Test (Singleton Pattern)

Define containers inside an abstract class so that JUnit 5 and Spring initialize the container **once** across all test suites inheriting from it.

```java
package com.example.testing;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class BaseIntegrationTest {

    // ServiceConnection automatically hooks JDBC properties into Spring Context
    @ServiceConnection
    static final PostgreSQLContainer<?> postgresContainer = 
            new PostgreSQLContainer<>("postgres:16-alpine");

    static {
        // Start container manually in static block to preserve JVM-wide singleton lifecycle
        postgresContainer.start();
    }
}
```

---

### Step 2: The Domain & Repository

```java
package com.example.testing.domain;

import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String name;

    private Double price;

    public Product() {}

    public Product(String name, Double price) {
        this.name = name;
        this.price = price;
    }

    public UUID getId() { return id; }
    public String getName() { return name; }
    public Double getPrice() { return price; }
}
```

```java
package com.example.testing.domain;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.UUID;

public interface ProductRepository extends JpaRepository<Product, UUID> {
}
```

---

### Step 3: Integration Test Class

```java
package com.example.testing;

import com.example.testing.domain.Product;
import com.example.testing.domain.ProductRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;

@Transactional // Auto-rolls back DB changes after each test method execution
class ProductRepositoryIntegrationTest extends BaseIntegrationTest {

    @Autowired
    private ProductRepository productRepository;

    @Test
    @DisplayName("Should persist and retrieve product from real PostgreSQL container")
    void shouldSaveAndFindProduct() {
        // Given
        Product newProduct = new Product("Mechanical Keyboard", 129.99);

        // When
        Product savedProduct = productRepository.save(newProduct);

        // Then
        assertThat(savedProduct.getId()).isNotNull();
        
        Product fetchedProduct = productRepository.findById(savedProduct.getId()).orElse(null);
        assertThat(fetchedProduct).isNotNull();
        assertThat(fetchedProduct.getName()).isEqualTo("Mechanical Keyboard");
        assertThat(fetchedProduct.getPrice()).isEqualTo(129.99);
    }
}
```