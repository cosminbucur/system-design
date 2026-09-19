Unit and integration tests verify that individual pieces of behavior are correct. Architecture tests verify something different: that the codebase's structure stays correct as it grows — that layering rules, naming conventions, and dependency direction don't quietly erode over hundreds of commits and many contributors. ArchUnit is the standard Java tool for this, and it runs as a normal JUnit test, so a violation fails the build exactly like any other test failure.

## 1. Why Architecture Needs Its Own Tests

A code review can miss a new class quietly reaching across a layer boundary — a controller calling a repository directly, bypassing the service layer, three months after the original architecture diagram was drawn. Nobody re-checks a wiki diagram on every PR, but a failing CI build can't be missed. ArchUnit encodes architectural decisions as executable rules instead of documentation that silently drifts out of sync with the actual codebase.

## 2. Enforcing Layering

```java
@AnalyzeClasses(packages = "com.example.orders")
class ArchitectureTest {

    @ArchTest
    static final ArchRule controllersShouldNotAccessRepositoriesDirectly =
        noClasses().that().resideInAPackage("..controller..")
            .should().accessClassesThat().resideInAPackage("..repository..");
            // enforces layering: controller -> service -> repository, never controller -> repository

    @ArchTest
    static final ArchRule servicesShouldNotDependOnControllers =
        noClasses().that().resideInAPackage("..service..")
            .should().dependOnClassesThat().resideInAPackage("..controller..");
            // catches an accidental dependency cycle before it's discovered at 2am

    @ArchTest
    static final ArchRule layeredArchitectureIsRespected =
        layeredArchitecture().consideringOnlyDependenciesInLayers()
            .layer("Controller").definedBy("..controller..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")
            .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service");
}
```

The `layeredArchitecture()` rule is worth calling out on its own: it checks the *entire* dependency graph between named layers in one declaration, rather than writing one `noClasses()` rule per forbidden direction — usually the cleaner way to express a layering policy once there are more than two layers involved.

## 3. Beyond Layering: What Else to Encode

ArchUnit rules aren't limited to layer boundaries — anything about a codebase's structure that a team has agreed on and wants automatically enforced is fair game:

| Rule category | Example |
| --- | --- |
| Dependency direction | Domain classes must never import a Spring annotation — keeps the domain model framework-agnostic |
| Bounded-context boundaries | Package `com.example.orders` must never depend on `com.example.billing` directly, only via a published interface |
| Injection style | Forbid field injection (`@Autowired` on fields) in favor of constructor injection |
| Naming/location conventions | Every class implementing `*Repository` must reside in a `..repository..` package |
| Persistence leakage | Classes annotated `@Entity` must never be returned from a `@RestController` method |

```java
@ArchTest
static final ArchRule domainShouldNotDependOnSpring =
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage("org.springframework..");
        // keeps the domain model framework-agnostic and independently testable

@ArchTest
static final ArchRule noFieldInjection =
    noFields().that().areAnnotatedWith(Autowired.class)
        .should().bePresent();
        // forces constructor injection, which makes dependencies visible and testable
```

## 4. Keeping the Rule Set Useful, Not Noisy

Keep the rule set focused on decisions the team has actually agreed on and cares about enforcing automatically — an overly strict, rapidly-changing rule set becomes noise that gets disabled rather than fixed, the same trap as alert fatigue in observability. A small number of rules the team genuinely holds to is worth far more than an exhaustive rule set that gets `@Disabled` the first time it's inconvenient.

## 5. Best Practices

| Practice | Recommendation |
| --- | --- |
| Encode architectural decisions as ArchUnit rules, not just documentation | A layering violation fails CI the same way a broken test does — a wiki diagram can't stop a bad import. |
| Start with the rules the team has actually agreed on | A rule nobody actually cares about enforcing gets disabled the first time it's inconvenient, undermining the whole rule set's credibility. |
| Use `layeredArchitecture()` for multi-layer policies | Cleaner than one `noClasses()` rule per forbidden direction once more than two layers are involved. |
| Keep the domain model framework-agnostic via a rule, not just convention | A rule forbidding framework imports in the domain package catches drift automatically, where a code review might miss it. |
| Treat a failing architecture test like any other build failure | It should block a merge exactly the same way a broken unit test would. |
| Don't let the rule set grow faster than the team's actual consensus | An overly strict, rapidly changing rule set becomes noise that erodes trust in the whole suite. |
