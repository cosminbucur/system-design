# Using the `Price` Value Object in Domain-Driven Design (DDD) with Java

In Domain-Driven Design (DDD), a **Price** is a classic example of a **Value Object**. Because money requires strict precision and context, combining `BigDecimal` with Java's built-in `java.util.Currency` is the idiomatic way to implement it.

---

## 1. The Implementation Example

Using a Java `record` (available since Java 14/16) is ideal for Value Objects because records are immutable by default and automatically generate `equals`, `hashCode`, and `toString` based on their components.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

public record Price(BigDecimal amount, Currency currency) {

    // Compact constructor for validation and normalization
    public Price {
        Objects.requireNonNull(amount, "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        
        if (amount.signum() < 0) {
            throw new IllegalArgumentException("Price amount cannot be negative");
        }

        // Normalize scale based on the currency's standard fractional digits (e.g., 2 for USD/EUR)
        int scale = currency.getDefaultFractionDigits();
        if (scale >= 0) {
            amount = amount.setScale(scale, RoundingMode.HALF_UP);
        }
    }

    // Domain behavior: Addition
    public Price add(Price other) {
        validateSameCurrency(other);
        return new Price(this.amount.add(other.amount), this.currency);
    }

    // Domain behavior: Subtraction
    public Price subtract(Price other) {
        validateSameCurrency(other);
        BigDecimal newAmount = this.amount.subtract(other.amount);
        return new Price(newAmount, this.currency);
    }

    // Factory method for convenience
    public static Price of(double amount, String currencyCode) {
        return new Price(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    private void validateSameCurrency(Price other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                String.format("Currency mismatch: cannot combine %s and %s", 
                    this.currency.getCurrencyCode(), 
                    other.currency.getCurrencyCode())
            );
        }
    }
}
```

---

## 2. Instantiation

Use factory methods or direct constructors. Because of the compact constructor validation, invalid inputs (like negative amounts or missing currencies) fail fast right at instantiation.

```java
// Using the convenience factory method
Price priceUsd = Price.of(29.99, "USD");

// Using standard constructor with BigDecimal
Price priceEur = new Price(new BigDecimal("45.00"), Currency.getInstance("EUR"));

// This will throw an IllegalArgumentException immediately
// Price invalid = Price.of(-10.00, "USD");
```

---

## 3. Performing Domain Operations

Because value objects are immutable, operations like addition or subtraction return a **new** instance rather than modifying the existing one.

```java
Price itemPrice = Price.of(100.00, "USD");
Price shippingFee = Price.of(15.50, "USD");

// Returns a new Price instance: 115.50 USD
Price totalPrice = itemPrice.add(shippingFee);

// Attempting to mix currencies throws a domain exception
Price eurPrice = Price.of(50.00, "EUR");
// itemPrice.add(eurPrice); -> Throws IllegalArgumentException (Currency mismatch)
```

---

## 4. Integrating into an Entity or Aggregate Root

In DDD, entities and aggregates hold value objects to encapsulate state and behavior. Here is how `Price` looks inside an `OrderItem` aggregate component:

```java
import java.math.BigDecimal;

public class OrderItem {
    private final OrderItemId id;
    private final ProductId productId;
    private final int quantity;
    private final Price unitPrice;
    private final Price subtotal;

    public OrderItem(OrderItemId id, ProductId productId, int quantity, Price unitPrice) {
        this.id = id;
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
        this.subtotal = calculateSubtotal(unitPrice, quantity);
    }

    private Price calculateSubtotal(Price unitPrice, int qty) {
        BigDecimal totalAmount = unitPrice.amount().multiply(BigDecimal.valueOf(qty));
        // Reuses the currency from the unit price automatically
        return new Price(totalAmount, unitPrice.currency());
    }

    public Price getSubtotal() {
        return subtotal;
    }
}
```

---

## 5. Equality and Comparisons

Because `Price` is a Java `record`, structural equality (`equals` and `hashCode`) works out of the box. Two different instances with the same amount and currency are automatically considered equal.

```java
Price priceA = Price.of(19.99, "USD");
Price priceB = Price.of(19.99, "USD");

boolean areEqual = priceA.equals(priceB); // true
```

---

## Key Design Considerations

* **Use `BigDecimal`, never primitives:** Floating-point types (`double` or `float`) introduce rounding errors due to binary representation. `BigDecimal` ensures exact decimal math.
* **Leverage `java.util.Currency`:** Instead of representing currencies as raw `String` primitives, `java.util.Currency` guarantees ISO 4217 validation out of the box (`Currency.getInstance("USD")`).
* **Enforce Currency Safety in Operations:** Arithmetic operations must always check that the currencies match.
* **Normalize Scale:** Using `currency.getDefaultFractionDigits()` ensures your amounts match the natural precision of the currency.