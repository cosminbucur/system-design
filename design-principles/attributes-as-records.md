# Domain-Driven Design (DDD): Modeling Attributes as Records

## 1. Why Model `BookId`, `ISBN`, and `Price` as Records?

In Domain-Driven Design (DDD), modeling attributes like `BookId`, `ISBN`, and `Price` as records (acting as **Value Objects**) rather than primitive types (`string`, `decimal`) solves several core architectural and domain modeling problems:

1. **Eliminating Primitive Obsession:** Using raw primitives like a `string` for an `ISBN` or a `decimal` for a `Price` leads to a domain model devoid of business meaning. Elevating these concepts into distinct types aligns the code closely with the ubiquitous language.
2. **Encapsulating Business Invariants & Validation:** Value objects validate their own state upon creation. Invalid states become unrepresentable:
   - **`ISBN`**: Enforces structural and checksum rules.
   - **`Price`**: Ensures non-negative amounts and bundles currency to prevent silent currency mismatches.
   - **`BookId`**: Enforces ID structure or UUID versioning.
3. **Value-Based Equality:** Value Objects are defined entirely by their attributes. Records provide structural equality out of the box, making comparisons, testing, and caching seamless.
4. **Immutability:** Value objects are conceptually immutable. Records enforce this natively, preventing side effects and making domain logic thread-safe.
5. **Type Safety and Compile-Time Protection:** Distinct types prevent parameter confusion (e.g., passing a price where a book ID is expected).

---

## 2. Java Record Implementations

### `BookId` Record
Encapsulates the unique identifier of the book entity, ensuring it is never null and providing a convenient factory method for generation.

```java
import java.util.Objects;
import java.util.UUID;

public record BookId(UUID value) {
    
    public BookId {
        Objects.requireNonNull(value, "BookId value cannot be null");
    }

    public static BookId generate() {
        return new BookId(UUID.randomUUID());
    }

    public static BookId from(String uuidString) {
        return new BookId(UUID.fromString(uuidString));
    }
}
```

### `ISBN` Record
Encapsulates string formatting rules and ensures structural validity (supporting ISBN-10 or ISBN-13 patterns) before creation.

```java
import java.util.Objects;

public record ISBN(String value) {

    private static final String ISBN_REGEX = "^(?:\\d{9}X|\\d{10}|\\d{13})$";

    public ISBN {
        Objects.requireNonNull(value, "ISBN value cannot be null");
        
        String normalized = value.trim().replaceAll("[- ]", "").toUpperCase();
        
        if (!normalized.matches(ISBN_REGEX)) {
            throw new IllegalArgumentException("Invalid ISBN format: " + value);
        }
    }
}
```

### `Price` Record
Combines a numeric amount with a currency to prevent currency mismatch bugs and guarantees non-negative amounts.

```java
import java.math.BigDecimal;
import java.util.Currency;
import java.util.Objects;

public record Price(BigDecimal amount, Currency currency) {

    public Price {
        Objects.requireNonNull(amount, "Price amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Price amount cannot be negative: " + amount);
        }
    }

    public static Price of(double amount, String currencyCode) {
        return new Price(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    public Price add(Price other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add prices with different currencies.");
        }
        return new Price(this.amount.add(other.amount), this.currency);
    }
}
```

---

## 3. Integration in a `Book` Aggregate

```java
public class Book {
    private final BookId id;
    private final ISBN isbn;
    private String title;
    private Price price;

    public Book(BookId id, ISBN isbn, String title, Price price) {
        this.id = Objects.requireNonNull(id);
        this.isbn = Objects.requireNonNull(isbn);
        this.title = Objects.requireNonNull(title);
        this.price = Objects.requireNonNull(price);
    }

    public void updatePrice(Price newPrice) {
        this.price = Objects.requireNonNull(newPrice);
    }
}