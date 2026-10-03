## Best Practices for Variable Naming in Java Entity Classes

When naming variables in a Java entity class (often mapped to database tables via JPA/Hibernate), following consistent conventions improves readability, prevents mapping errors, and ensures compatibility with frameworks.

Here are the best practices organized by type and context:

### 1. General Naming Conventions

- **CamelCase:** Use camelCase for all variable names (e.g., `firstName`, `accountBalance`). Never use snake_case (`first_name`) or kebab-case inside Java code, as it violates standard Java naming conventions.
- **Descriptive Names:** Avoid cryptic abbreviations. Choose `customerAddress` instead of `custAddr`.
- **Nouns over Verbs:** Variables represent data or states, so use nouns or noun phrases (e.g., `isActive`, `createdDate`).

### 2. Database Mapping & Annotations

- **Naming Strategy:** Java typically uses camelCase, while relational databases often use snake_case. Let your ORM handle the translation rather than forcing snake_case into Java:
  - **Java:** `private String postalCode;`
  - **Database:** Maps automatically to `postal_code` using standard naming strategies (like Spring Boot's `SpringPhysicalNamingStrategy`), or explicitly via `@Column(name = "postal_code")`.
- **Boolean Variables:**
  - Avoid prefixes like `is` in the actual variable name if it causes mapping issues with frameworks (e.g., use `active` instead of `isActive`, which generates `isActive()` and `setActive()` getters/setters correctly).
  - If using primitive `boolean`, a field named `active` creates `isActive()` as a getter, which aligns well with standard conventions.

### 3. Handling Relationships (Associations)

- **To-One Relationships (`@ManyToOne`, `@OneToOne`):** Name the field after the target entity in singular form, omitting foreign key suffixes like `_id`:
  - `private Department department;` (instead of `departmentId` or `dept`).
- **To-Many Relationships (`@OneToMany`, `@ManyToMany`):** Name the field as a plural noun representing the collection:
  - `private List<Order> orders;` (instead of `orderList`).

### 4. Special Fields & Identifiers

- **Primary Keys:** Name the primary key identifier simply `id`:
  - `private Long id;`
- **Audit Fields:** Keep standard tracking fields consistent across all entities:
  - `private LocalDateTime createdAt;`
  - `private LocalDateTime updatedAt;`

> **Note:** Avoid using reserved SQL keywords (like `order`, `select`, `table`) as variable names to prevent syntax errors and complications when writing JPQL or native queries. If unavoidable, explicitly escape them using `@Column(name = "\"order\"")`.
