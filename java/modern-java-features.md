Modern Java (17–21+) added a cluster of features aimed squarely at reducing boilerplate and making illegal states harder to represent — records, sealed classes, and pattern matching work together as one coherent story: model your domain precisely, then let the compiler check that you've handled every case. This is also what makes several patterns far less verbose than their pre-Java-16 equivalents.

## 1. Records — Data Carriers Without the Boilerplate

A `record` declares an immutable data class in one line — the compiler generates the constructor, accessors, `equals()`, `hashCode()`, and `toString()` for you.

```java
public record Money(BigDecimal amount, Currency currency) {}

Money price = new Money(new BigDecimal("19.99"), Currency.getInstance("USD"));
price.amount();   // accessor — note: no "get" prefix
price.currency();
```

Compare to the pre-record equivalent: a class with a private final field per component, a constructor, getters, and manually written/generated `equals`/`hashCode`/`toString` — records eliminate that entirely for the common case of "an immutable bundle of values."

### Compact Constructors — Validation Without Restating Fields

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.signum() < 0) {
            throw new IllegalArgumentException("Money cannot be negative: " + amount);
        }
        // no need to write `this.amount = amount;` — the compact constructor does that implicitly
    }
}
```

### Records as Value Objects

Records are a near-perfect fit for the value objects described in DDD — immutable, structurally equal, self-validating. Use them for money, coordinates, date ranges, and similar concepts instead of raw primitives.

```java
public record OrderId(UUID value) {}   // a typed wrapper instead of a raw UUID/String
public record Email(String address) {
    public Email {
        if (!address.contains("@")) throw new IllegalArgumentException("Invalid email: " + address);
    }
}
```

A record is the wrong tool when identity matters more than values, or when the object needs mutable internal state — entities generally still want a regular class.

## 2. Sealed Classes and Interfaces — Closed Type Hierarchies

`sealed` restricts which classes/interfaces may extend or implement a type — the compiler knows the _complete_ set of subtypes, which is what unlocks exhaustive pattern matching.

```java
public sealed interface PaymentMethod
    permits CreditCard, BankTransfer, DigitalWallet {}

public record CreditCard(String last4, YearMonth expiry) implements PaymentMethod {}
public record BankTransfer(String iban) implements PaymentMethod {}
public record DigitalWallet(String provider, String accountId) implements PaymentMethod {}
```

Without `sealed`, `PaymentMethod` could be implemented by any class anywhere, so the compiler could never guarantee you've handled every case in a `switch`. With `sealed`, adding a new permitted subtype (e.g., `CryptoPayment`) causes every exhaustive `switch` over `PaymentMethod` elsewhere in the codebase to fail to compile until you handle the new case — turning a "did I remember to update every place that switches on this?" runtime risk into a compile-time guarantee.

This is effectively a modern alternative to the classic Visitor pattern: both exist to let you add operations over a fixed set of types safely, but sealed types + pattern matching achieve it with far less ceremony than a `visit()` method per subtype.

## 3. Switch Expressions — No More Fall-Through Bugs

The traditional `switch` statement's fall-through was a classic bug source (forgetting a `break`). Switch _expressions_ (`->` syntax) produce a value directly, don't fall through, and can be exhaustiveness-checked by the compiler.

```java
// Old switch statement — verbose, fall-through risk, needs a separate variable
String description;
switch (status) {
    case ACTIVE:
        description = "Active";
        break;
    case SUSPENDED:
        description = "Suspended";
        break;
    default:
        description = "Unknown";
}

// Switch expression — concise, no fall-through, evaluates directly to a value
String description = switch (status) {
    case ACTIVE -> "Active";
    case SUSPENDED -> "Suspended";
    case CLOSED -> "Closed";
};
```

If `status` is an `enum` and every constant is covered, the compiler doesn't even require a `default` branch — and if a new enum constant is added later without updating this switch, compilation fails immediately rather than silently falling through to a wrong default at runtime. Use `yield` when a branch needs a multi-statement block rather than a single expression:

```java
int discountPercent = switch (customerTier) {
    case GOLD -> 20;
    case SILVER -> 10;
    default -> {
        log.warn("Unknown tier: {}", customerTier);
        yield 0;
    }
};
```

## 4. Pattern Matching for `instanceof`

Eliminates the redundant cast that used to always follow an `instanceof` check.

```java
// Old: check, then cast — repeats the type name and risks a ClassCastException if they drift apart
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Pattern matching: the cast is implicit, and the variable is scoped to where the check is true
if (obj instanceof String s) {
    System.out.println(s.length());
}
```

The pattern variable's scope extends naturally with flow analysis — including through a negated check:

```java
if (!(obj instanceof String s)) {
    return; // s is not in scope here
}
System.out.println(s.length()); // s IS in scope here — the compiler knows we returned otherwise
```

## 5. Pattern Matching in `switch` — The Real Payoff

Combines sealed types and switch expressions into exhaustive, type-safe branching over a hierarchy — this is where the modern-Java story fully comes together.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
        // no default needed — Shape is sealed and every permitted subtype is covered
    };
}
```

Guarded patterns (`when` clauses) add conditions without leaving pattern matching:

```java
String classify(Shape shape) {
    return switch (shape) {
        case Circle c when c.radius() > 100 -> "Huge circle";
        case Circle c -> "Circle";
        case Rectangle r when r.width() == r.height() -> "Square";
        case Rectangle r -> "Rectangle";
        case Triangle t -> "Triangle";
    };
}
```

Order matters with `when` guards: `Circle c when c.radius() > 100` must come before the plain `Circle c` case, or the more general pattern would match first and the guarded one would be unreachable.

## 6. Record Patterns — Deconstruction

Record patterns let you destructure a record directly in an `instanceof`/`switch`, pulling out its components without calling accessors manually — and they nest, so you can deconstruct nested records in one pattern.

```java
public record Point(int x, int y) {}
public record Line(Point start, Point end) {}

// Deconstructing a nested record in one pattern
if (shape instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
    double length = Math.hypot(x2 - x1, y2 - y1);
}
```

```java
// Combined with switch and guards — reads like a direct description of the domain rule
String describe(Shape shape) {
    return switch (shape) {
        case Circle(double r) when r == 0 -> "Degenerate circle (a point)";
        case Circle(double r) -> "Circle with radius " + r;
        case Rectangle(double w, double h) when w == h -> "Square with side " + w;
        case Rectangle(double w, double h) -> "Rectangle " + w + "x" + h;
        case Triangle t -> "Triangle";
    };
}
```

## 7. Text Blocks — Multi-Line Strings Without Escaping

Removes the need for manual `\n` and string concatenation for embedded JSON, SQL, or HTML.

```java
// Old: escaping and concatenation obscure the actual content
String json = "{\n" +
    "  \"name\": \"Cosmin\",\n" +
    "  \"role\": \"developer\"\n" +
    "}";

// Text block: the content is visually what it represents
String json = """
    {
      "name": "Cosmin",
      "role": "developer"
    }
    """;
```

Particularly useful for the JPQL/SQL query strings seen in jpa-persistence and jdbi — no more `"SELECT * FROM orders " + "WHERE status = :status"` concatenation.

## 8. `var` — Local Type Inference

`var` infers the type from the right-hand side at the declaration site — reduces noise for obvious types, but the type still exists and is checked at compile time (this is not dynamic typing).

```java
var orders = new ArrayList<Order>();      // clearly a List<Order> — fine
var result = orderService.process(order); // what does this return? not obvious from the line alone
```

Use `var` when the type is already obvious from the right-hand side (a constructor call, a clearly-named factory method) — avoid it when the inferred type isn't obvious to a reader, since that reintroduces the exact ambiguity naming/typing was meant to remove.

## 9. Best Practices

| Practice                                                            | Recommendation                                                                                        |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Default to records for immutable data carriers                      | Especially for DTOs, value objects, and API request/response bodies.                                  |
| Seal a type hierarchy when the set of subtypes is meant to be fixed | Unlocks exhaustive `switch` checking — the compiler catches missed cases when a new subtype is added. |
| Prefer switch expressions over switch statements                    | No fall-through risk, and exhaustiveness checking on sealed types/enums.                              |
| Use pattern matching to eliminate redundant casts                   | `instanceof String s` over `instanceof String` + manual cast.                                         |
| Reach for record patterns when destructuring nested data            | More direct than chains of accessor calls, especially inside a `switch`.                              |
| Don't overuse `var` where the type isn't obvious                    | If a reader can't tell the type from the line itself, spell it out.                                   |
| Use text blocks for embedded SQL/JSON/HTML                          | Removes escaping noise that obscures the actual content.                                              |
