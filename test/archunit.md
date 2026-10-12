# ArchUnit: Overview, Core Concepts & Production Examples

**ArchUnit** is a Java testing library that allows developers to test their architecture automatically using standard unit testing frameworks like JUnit. Instead of relying on manual documentation or code reviews, ArchUnit treats Java bytecode as data, letting you write explicit test assertions for package dependencies, layer boundaries, naming conventions, and coding patterns.

---

## Why Use ArchUnit?

- **Automated Guardrails:** Prevents architectural decay (e.g., domain models depending on web frameworks or persistence layers).
- **CI/CD Integration:** Runs as part of your standard test suite during builds, failing fast if architectural rules are violated.
- **Fluent API:** Uses a readable, English-like syntax that makes rules self-documenting.

ArchUnit's architecture revolves around a few foundational abstractions designed to treat compiled Java bytecode as a queryable object model.

---

### 1. ClassFileImporter and JavaClasses

ArchUnit operates entirely on compiled bytecode rather than raw source code (.java files).

- **`ClassFileImporter`**: The entry point for loading classes. It scans specified packages or directories from your classpath.
- **`JavaClasses`**: The container returned by the importer. It holds a collection of all parsed classes and acts as the target against which your architectural rules are evaluated.

### 2. The Domain Model

When ArchUnit imports your classes, it constructs a rich object model that mirrors Java's structure:

- **`JavaClass`**: Represents any type (class, interface, enum, annotation), tracking its package, modifiers, superclasses, interfaces, annotations, fields, and methods.
- **`JavaMethod` / `JavaField**`: Represent individual methods and fields, allowing rules to inspect things like method return types, parameter types, or field annotations.
- **`JavaAccess` / `JavaCall**`: Tracks dependencies between components—such as which class calls a specific method or accesses a particular field.

### 3. ArchRule and Fluent Syntax

An **`ArchRule`** is an encapsulation of a specific architectural constraint. ArchUnit provides a readable, English-like fluent API built around static factory methods:

- **Selectors (`classes()`, `noClasses()`, `methods()`, `fields()`)**: Define the subset of code the rule targets (e.g., classes ending with `Repository`).
- **Predicates (`that()`, `arePublic()`, `resideInAPackage()`)**: Refine the scope of the rule.
- **Conditions (`should()`, `mayOnlyAccessClassesThat()`, `beAnnotatedWith()`)**: Define what is required of the selected classes.

### 4. Rule Execution and Verification

Once a rule is defined, it is executed against a `JavaClasses` instance using the `.check(...)` method:

```java
JavaClasses classes = new ClassFileImporter().importPackages("com.example.app");

ArchRule rule = classes()
    .that().resideInAPackage("..service..")
    .should().onlyBeAccessedByAnyLayer("..controller..", "..service..");

// Throws an AssertionError if any class violates the rule
rule.check(classes);

```

If a violation occurs, ArchUnit provides detailed failure messages pointing to the exact offending classes, methods, or lines of code, making debugging straightforward within standard unit test reports.

---

To make your ArchUnit test compatible with **JUnit 5** and modern Java, you need to update your imports (`org.junit.jupiter.api.Test`) and ensure you are using a modern version of ArchUnit (v1.3.0 or higher) which fully supports recent Java bytecode features up to Java 25.

Here are two ways to write this: **Option 1** keeps your procedural style with explicit class importing, and **Option 2** uses ArchUnit's idiomatic JUnit 5 integration (`@AnalyzeClasses` and `@ArchTest`).

---

### Option 1: Standard JUnit 5 `@Test` Method (Closest to your original)

```java
import com.tngtech.archunit.core.importer.ClassFileImporter;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

class LayeredArchitectureTest {

    @Test
    void layeredArchitectureShouldBeRespected() {
        var importedClasses = new ClassFileImporter().importPackages("com.yourcompany.project");

        layeredArchitecture()
                .layer("Controller").definedBy("..controller..")
                .layer("Service").definedBy("..service..")
                .layer("Repository").definedBy("..repository..")

                .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
                .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
                .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
                .check(importedClasses);
    }
}

```

---

### Option 2: Idiomatic ArchUnit JUnit 5 Extension (Recommended)

This approach lets ArchUnit manage class importing automatically via annotations, making the test cleaner and faster (as it caches imported classes across test executions).

```java
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import com.tngtech.archunit.library.Architectures;

@AnalyzeClasses(packages = "com.yourcompany.project")
class LayeredArchitectureTest {

    @ArchTest
    static final ArchRule layeredArchitectureShouldBeRespected = Architectures.layeredArchitecture()
            .layer("Controller").definedBy("..controller..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")

            .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");
}

```

---

### Build Tool Dependency Check

Ensure your build file uses the JUnit 5 artifact rather than the legacy JUnit 4 one:

- **Maven (`pom.xml`):**

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version> <!-- or latest version -->
    <scope>test</scope>
</dependency>

```

- **Gradle (`build.gradle`):**

```groovy
testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'

```
