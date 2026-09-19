Creational patterns are reusable solutions to the problem of object creation itself — how to construct something without tightly coupling the calling code to a specific class, a specific set of constructor parameters, or a specific number of instances. They're vocabulary for communicating structure ("just use a Builder here") more than code to copy-paste, and modern Java (lambdas, records, sealed classes) makes several of them far less boilerplate-heavy than the original 1994 Gang of Four book.

## 1. Builder

Solves telescoping constructors — too many overloaded constructors needed just to cover every combination of optional parameters.

```java
public class Account {
    private final String ownerName;
    private final BigDecimal balance;
    private final AccountStatus status;

    private Account(Builder b) {
        this.ownerName = b.ownerName;
        this.balance = b.balance;
        this.status = b.status;
    }

    public static class Builder {
        private String ownerName;
        private BigDecimal balance = BigDecimal.ZERO;
        private AccountStatus status = AccountStatus.ACTIVE;

        public Builder ownerName(String v) { this.ownerName = v; return this; }
        public Builder balance(BigDecimal v) { this.balance = v; return this; }
        public Builder status(AccountStatus v) { this.status = v; return this; }
        public Account build() { return new Account(this); }
    }
}

Account account = new Account.Builder()
    .ownerName("Cosmin")
    .balance(new BigDecimal("100.00"))
    .build();
```

In modern Java, a `record` with a compact canonical constructor often replaces simple builders entirely — only reach for Builder when you have several optional fields or need step-by-step validation.

## 2. Factory Method

Delegates object creation to a subclass or dedicated method instead of calling `new` directly, so callers depend on an interface, not a concrete class.

```java
public interface PaymentProcessor {
    void process(BigDecimal amount);
}

public class PaymentProcessorFactory {
    public static PaymentProcessor create(PaymentType type) {
        return switch (type) {
            case CREDIT_CARD -> new CreditCardProcessor();
            case BANK_TRANSFER -> new BankTransferProcessor();
            case CRYPTO -> new CryptoProcessor();
        };
    }
}
```

## 3. Abstract Factory

A "factory of factories" — a family of related objects that must be created together, consistently, with the concrete family swapped as a unit. Where Factory Method produces one product, Abstract Factory produces an entire matching set (a UI toolkit's buttons *and* checkboxes for the same theme, for example).

```java
public interface UIComponentFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class DarkThemeFactory implements UIComponentFactory {
    public Button createButton() { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}

public class LightThemeFactory implements UIComponentFactory {
    public Button createButton() { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}

// Caller only depends on the abstraction — swapping the whole family is one line
UIComponentFactory factory = darkModeEnabled ? new DarkThemeFactory() : new LightThemeFactory();
Button button = factory.createButton();
Checkbox checkbox = factory.createCheckbox(); // guaranteed to match the same theme as the button
```

The guarantee Abstract Factory buys you that plain Factory Method doesn't: it's structurally impossible to accidentally mix a `DarkButton` with a `LightCheckbox`, because both come from the same concrete factory instance.

## 4. Prototype

Creates new objects by cloning an existing, fully-configured instance rather than building one from scratch — useful when constructing an object from raw inputs is expensive, or when you need many variations of a complex base configuration.

```java
public class ReportTemplate implements Cloneable {
    private String header;
    private List<String> sections;
    // ... expensive-to-build formatting/layout state

    @Override
    public ReportTemplate clone() {
        try {
            ReportTemplate copy = (ReportTemplate) super.clone();
            copy.sections = new ArrayList<>(this.sections); // deep-copy mutable fields explicitly
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e); // Cloneable was implemented — this can't actually happen
        }
    }
}

ReportTemplate base = buildExpensiveBaseTemplate();
ReportTemplate variantA = base.clone();
variantA.setHeader("Q1 Report"); // starts from the expensive base, only the small delta is new work
```

Java's built-in `Cloneable`/`clone()` is notoriously error-prone (shallow-copy pitfalls, checked exception plumbing) — in modern code, a copy constructor or a `record`'s natural immutability more often serves the same purpose with far fewer footguns. Reach for actual `Cloneable`-based Prototype only when working with a legacy codebase that already uses this convention.

## 5. Singleton

Ensures exactly one instance exists. In Spring, you almost never write this by hand — a `@Bean`/`@Component` is a singleton by default within the application context. If you must write one manually, use an `enum` (thread-safe, serialization-safe by construction):

```java
public enum ConfigRegistry {
    INSTANCE;
    private final Map<String, String> config = new ConcurrentHashMap<>();
    public String get(String key) { return config.get(key); }
}
```

## 6. Pattern Selection Cheat Sheet

| Problem | Reach for |
| --- | --- |
| Object needs many optional constructor params | Builder |
| Caller shouldn't know the concrete class being instantiated | Factory Method |
| A whole family of related objects must be created consistently together | Abstract Factory |
| Building an object from scratch is expensive; need variations of a base config | Prototype (or a copy constructor in modern Java) |
| Need exactly one shared instance | Singleton (or just a Spring bean) |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Reach for a `record` before a full Builder for simple, small objects | A compact canonical constructor covers most cases without the boilerplate of a nested Builder class. |
| Use Abstract Factory only when products genuinely must be created as a matching set | If products are created independently with no cross-consistency requirement, plain Factory Method is enough. |
| Prefer a copy constructor over `Cloneable` in new Java code | `clone()`'s shallow-copy pitfalls and checked-exception plumbing make it a common source of subtle bugs. |
| Let Spring manage singletons instead of hand-rolling them | An `@Component`/`@Bean` is already a singleton within the application context — a manual enum-based Singleton is only needed outside a DI framework. |
| Don't force a creational pattern where a plain constructor is clear enough | Patterns solve specific recurring problems — adding one where the problem doesn't exist is indirection with no payoff. |
