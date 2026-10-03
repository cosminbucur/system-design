# Spring Data Auditing: Fields and Annotations Reference

For auditing purposes in Spring Data (whether using JPA, MongoDB, R2DBC, or Cassandra), auditing metadata is handled using a set of core annotations, supporting interfaces, and configuration classes.

---

## 1. Core Auditing Annotations

Spring Data provides four primary field-level annotations to automatically capture tracking information:

| Annotation | Description | Supported Field Types |
| :--- | :--- | :--- |
| **`@CreatedDate`** | Automatically populates the field with the timestamp of when the entity was first persisted. | `Instant`, `LocalDateTime`, `LocalDate`, `Long`, `long`, `Date`, `Calendar` |
| **`@LastModifiedDate`** | Automatically populates/updates the field with the timestamp of every update operation. | `Instant`, `LocalDateTime`, `LocalDate`, `Long`, `long`, `Date`, `Calendar` |
| **`@CreatedBy`** | Automatically captures the principal/user responsible for creating the entity. | `String`, `Long`, custom User types (must match your `AuditorAware` return type) |
| **`@LastModifiedBy`** | Automatically captures the principal/user who last modified the entity. | `String`, `Long`, custom User types |

---

## 2. Configuration & Lifecycle Annotations

To activate and manage the auditing infrastructure, specific class-level annotations are required:

* **`@EnableJpaAuditing`** (Configuration-level)
  * Placed on a Spring `@Configuration` class to enable the JPA auditing mechanism globally.
  * **Key Attributes:** 
    * `auditorAwareRef`: Specifies the bean name of the custom `AuditorAware` implementation.
    * `dateTimeProviderRef`: Specifies a custom `DateTimeProvider` bean if default system clocks need overriding.
* **`@EntityListeners(AuditingEntityListener.class)`** (Entity-level)
  * Placed on a JPA entity class (or a shared mapped superclass) to ensure Spring's `AuditingEntityListener` intercepts lifecycle events (`PrePersist`, `PreUpdate`).

---

## 3. Interface-Based Auditing (Alternative Approach)

If you prefer inheritance or strict type contracts over field annotations, Spring Data offers built-in base classes and interfaces:

* **`AbstractAuditable<U, ID>`**: A convenient base class implementing `Auditable<U, ID>` that provides standard fields and getters/setters for:
  * `createdBy`
  * `createdDate`
  * `lastModifiedBy`
  * `lastModifiedDate`

---

## 4. Supporting Components (`AuditorAware`)

To make `@CreatedBy` and `@LastModifiedBy` function, you must provide a bean implementing the **`AuditorAware<T>`** interface so Spring knows *who* is currently performing the action:

```java
import org.springframework.data.domain.AuditorAware;
import org.springframework.stereotype.Component;
import java.util.Optional;

@Component
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        // Retrieve current username or principal ID from your security context
        return Optional.of("system-user"); 
    }
}
```

---

## 5. Deep Revision Tracking: Hibernate Envers

When basic timestamps and user fields are insufficient and you require **full historical revision auditing** (storing a complete snapshot of table rows across every modification or deletion), Spring Data integrates smoothly with **Hibernate Envers**:

* **`@Audited`**: Placed on an entity class or specific properties to track history in dedicated audit tables (e.g., `entity_name_AUD`).
* **`@NotAudited`**: Placed on specific fields within an audited entity to exclude them from historical tracking.