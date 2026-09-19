Functional programming treats computation as evaluating functions rather than executing a sequence of state-mutating steps — functions are values you can pass around, combine, and return, and the preferred building block is a pure function whose output depends only on its input. Java isn't a purely functional language and was never redesigned to be one, but since Java 8 it supports this style well enough that it's now a normal, idiomatic way to write everyday business logic, especially the "transform this data" and "decide what to do" categories of code. This is distinct from the Stream API mechanics (covered separately) — this is about the underlying paradigm those APIs are built on.

## 1. Pure Functions and Referential Transparency

A pure function's output depends only on its input, and calling it produces no observable side effects — no mutating shared state, no I/O, nothing beyond computing and returning a value.

```java
// Pure — same input always produces the same output, nothing else happens
BigDecimal applyTax(BigDecimal price, BigDecimal taxRate) {
    return price.add(price.multiply(taxRate));
}

// Impure — depends on and mutates state outside its own scope
BigDecimal total = BigDecimal.ZERO; // shared mutable state
void addToTotal(BigDecimal amount) {
    total = total.add(amount); // side effect: something outside this method changed
    auditLog.record(amount);   // side effect: I/O
}
```

Pure functions are trivially unit-testable (no setup/mocking needed beyond the input itself), safely reusable in any order or in parallel (nothing to race on), and easy to reason about in isolation — you can understand exactly what a pure function does just by reading its signature and body, without tracing through the rest of the program's state.

## 2. Immutability: The Structural Enabler of Purity

Pure functions and shared mutable state don't mix well — if a function's input could be changed out from under it by something else, "same input always produces the same output" stops holding. Immutable data (already the default posture for a `record`, covered in modern Java features) removes this risk structurally rather than by convention.

```java
// Mutable — every caller of applyDiscount must trust it doesn't secretly mutate the passed-in order
public void applyDiscount(Order order, BigDecimal discount) {
    order.setTotal(order.getTotal().subtract(discount)); // mutates the caller's object
}

// Immutable — the function can only return a new value, never corrupt the caller's original
public Order applyDiscount(Order order, BigDecimal discount) {
    return new Order(order.id(), order.total().subtract(discount)); // original 'order' is untouched
}
```

## 3. Functions as Values: Java's Functional Interfaces

Java represents "a function you can pass around" as an instance of a single-method interface (a `@FunctionalInterface`) — a lambda or method reference is just syntax for creating one of these on the fly. `java.util.function` provides the common shapes so you rarely need to declare your own.

| Interface | Signature | Use for |
| --- | --- | --- |
| `Function<T,R>` | `T -> R` | Transforming a value into another value |
| `Predicate<T>` | `T -> boolean` | A yes/no test on a value |
| `Consumer<T>` | `T -> void` | Doing something with a value, producing no result |
| `Supplier<T>` | `() -> T` | Producing a value with no input (e.g., lazy computation, a default) |
| `BiFunction<T,U,R>` | `(T,U) -> R` | Combining two inputs into one result |
| `UnaryOperator<T>` | `T -> T` | A `Function` where input and output are the same type |

```java
Function<BigDecimal, BigDecimal> applyVat = price -> price.multiply(new BigDecimal("1.20"));
Predicate<Order> isHighValue = order -> order.total().compareTo(new BigDecimal("1000")) > 0;
Supplier<Order> defaultOrder = () -> new Order("UNKNOWN", BigDecimal.ZERO);

BigDecimal withVat = applyVat.apply(basePrice);
boolean qualifies = isHighValue.test(order);
```

## 4. Higher-Order Functions: Functions That Take or Return Functions

A higher-order function either accepts a function as a parameter or returns one — this is what lets behavior itself become a configurable input, rather than something hardcoded inside a method.

```java
// Accepts a function as a parameter — the caller decides *how* to validate
List<Order> filterOrders(List<Order> orders, Predicate<Order> condition) {
    return orders.stream().filter(condition).toList();
}

List<Order> highValueOrders = filterOrders(orders, o -> o.total().compareTo(threshold) > 0);

// Returns a function — builds a customized function based on its own arguments
Function<BigDecimal, BigDecimal> discountOf(BigDecimal percentage) {
    return price -> price.multiply(BigDecimal.ONE.subtract(percentage));
}

Function<BigDecimal, BigDecimal> tenPercentOff = discountOf(new BigDecimal("0.10"));
BigDecimal discounted = tenPercentOff.apply(originalPrice);
```

This second example — a function that returns a function pre-configured with some of its arguments — is the practical form partial application/currying takes in Java: instead of a dedicated language construct, it's just a method returning a lambda that closed over the earlier argument.

## 5. Function Composition

Rather than writing one large function that does several things, functional style favors composing several small, single-purpose functions together — `Function` provides `andThen` and `compose` specifically for this.

```java
Function<String, String> trim = String::trim;
Function<String, String> toLowerCase = String::toLowerCase;
Function<String, String> normalize = trim.andThen(toLowerCase); // runs trim, then feeds result into toLowerCase

String result = normalize.apply("  HELLO World  "); // "hello world"

// compose runs in the opposite order: argument.compose(other) means "run other first, then argument"
Function<String, String> normalizeAlt = toLowerCase.compose(trim); // identical behavior to the line above
```

`Predicate` offers the equivalent for boolean logic — `and`, `or`, and `negate` compose conditions without writing a single new lambda body:

```java
Predicate<Order> isHighValue = o -> o.total().compareTo(threshold) > 0;
Predicate<Order> isFromVip = o -> o.customer().isVip();
Predicate<Order> needsReview = isHighValue.and(isFromVip.negate()); // high value, but NOT already a VIP
```

## 6. Closures: Capturing State From the Enclosing Scope

A lambda can reference variables from its enclosing scope — this is a closure, and Java requires anything captured this way to be effectively final (assigned exactly once), which is what keeps the captured value's meaning stable and safe to use later, including from another thread.

```java
BigDecimal exchangeRate = fetchCurrentRate(); // effectively final — never reassigned after this line
Function<BigDecimal, BigDecimal> convertToUsd = amount -> amount.multiply(exchangeRate);
// exchangeRate = fetchCurrentRate(); // would NOT compile once the lambda above captures it
```

This restriction exists precisely to prevent the confusing scenario where a captured variable changes after the lambda was created but before it actually runs — Java sidesteps the ambiguity entirely by disallowing it at compile time, rather than defining some specific (and likely surprising) runtime behavior for it.

## 7. Where Java's Functional Style Has Real Limits

Java remains a hybrid, object-oriented-first language, and being upfront about the boundaries avoids fighting the language:

- **Checked exceptions don't compose cleanly** with functional interfaces — `Function<T,R>`'s `apply` method doesn't declare `throws`, so a lambda body that calls a checked-exception-throwing method needs an explicit try/catch or a wrapping helper, which quickly erodes the clean, declarative style functional code is going for.
- **No true tail-call optimization** — a deeply recursive functional-style solution can still blow the call stack in Java, unlike languages built around recursion as the primary looping construct.
- **Mutation is still everywhere in the ecosystem** — most Java libraries, ORMs, and frameworks are built around mutable JavaBeans, so a purely functional style is an internal discipline choice within your own code, not something the whole platform enforces.

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Default to pure functions for business/calculation logic | They're trivially testable and safe to reuse or run in parallel — reserve side effects for the edges of the system (I/O, persistence). |
| Prefer immutable data (`record`s, final fields) for anything passed through functional-style code | Removes an entire class of bugs where a function's output changes because something else mutated its input. |
| Reach for `java.util.function`'s existing interfaces before declaring a custom one | `Function`, `Predicate`, `Consumer`, `Supplier`, `BiFunction` already cover the overwhelming majority of cases. |
| Compose small functions instead of writing one large one | `andThen`/`compose`/`and`/`or` make intent explicit and each small piece independently testable. |
| Remember captured lambda variables must be effectively final | Design around this constraint (e.g., return a new value instead of mutating a captured variable) rather than fighting it. |
| Don't force purity where Java's ecosystem expects mutable objects | JPA entities, many framework lifecycle callbacks, and most existing libraries assume mutable objects — apply functional discipline to your own logic layer, not by rewriting how the whole stack works. |
