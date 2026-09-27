# Java Immutability Guide

An **immutable object** is an object whose state cannot be changed after it is constructed. Immutability makes code thread-safe, simplifies concurrent programming, reduces side effects, and makes objects safer to use as keys in collections like `HashMap`.

---

## 1. Core Rules for Writing an Immutable Class

To make a custom class fully immutable in Java, you must follow five fundamental rules:

1. **Declare the class as `final`**  
   Prevents subclassing, which stops child classes from overriding methods and adding mutable fields.
2. **Make all fields `private` and `final`**  
   Ensures fields cannot be accessed directly or reassigned after construction.
3. **Do not provide setter methods**  
   Avoid any methods that alter internal fields or states.
4. **Initialize all fields via constructor (with defensive copies)**  
   When receiving mutable objects (e.g., `Date`, arrays, lists) in the constructor, create deep or defensive copies before assigning them to fields.
5. **Return defensive copies in getter methods**  
   Never return references to internal mutable objects. Return defensive copies or unmodifiable wrappers instead.

---

## 2. Code Examples

### Traditional Class Implementation (Java 8+)

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.List;

public final class UserProfile {
    private final String username;
    private final Date joiningDate;
    private final List<String> permissions;

    public UserProfile(String username, Date joiningDate, List<String> permissions) {
        this.username = username;
        // Defensive copy for mutable Date object
        this.joiningDate = new Date(joiningDate.getTime());
        // Defensive copy for mutable List
        this.permissions = new ArrayList<>(permissions);
    }

    public String getUsername() {
        return username; // String is inherently immutable
    }

    public Date getJoiningDate() {
        // Return a copy to prevent external mutation
        return new Date(joiningDate.getTime());
    }

    public List<String> getPermissions() {
        // Return an unmodifiable view
        return Collections.unmodifiableList(permissions);
    }
}
```

---

### Java Records Implementation (Java 16+)

Java 16 introduced **Records**, which dramatically reduce boilerplate when creating immutable data carriers. Records automatically make fields `private final`, generate getters, `equals()`, `hashCode()`, `toString()`, and mark the class as `final`.

To handle mutable references inside a record, use a **compact constructor**:

```java
import java.util.List;

public record UserProfile(String username, List<String> permissions) {
    // Compact constructor to enforce defensive copying
    public UserProfile {
        // List.copyOf creates an unmodifiable copy of the list
        permissions = List.copyOf(permissions);
    }
}
```

---

## 3. Standard Library Immutable Types

Java provides many built-in immutable classes across its core API:

| Category | Immutable Types |
| :--- | :--- |
| **Primitives Wrappers & Text** | `String`, `Integer`, `Double`, `Boolean`, `BigDecimal`, `BigInteger` |
| **Date and Time (Java 8+)** | `LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `Instant`, `Duration` |
| **Unmodifiable Collections (Java 9+)** | `List.of()`, `Set.of()`, `Map.of()`, `List.copyOf()` |

---

## 4. Common Pitfalls & How to Avoid Them

### Pitfall 1: Leaking Mutable References
Assigning or returning mutable field references directly breaks immutability.

```java
// BAD: Modifying 'roles' externally will change internal state
public class BadImmutable {
    private final List<String> roles;
    public BadImmutable(List<String> roles) {
        this.roles = roles; // Leaks reference
    }
}

// GOOD: Copy input and output
public class GoodImmutable {
    private final List<String> roles;
    public GoodImmutable(List<String> roles) {
        this.roles = List.copyOf(roles);
    }
    public List<String> getRoles() {
        return roles; // Already unmodifiable via List.copyOf
    }
}
```

### Pitfall 2: Modifying Unmodifiable Collections
Calling mutating methods like `.add()` or `.remove()` on unmodifiable collections throws `UnsupportedOperationException`. To change state, construct a **new** instance with the updated data.

```java
List<String> currentList = List.of("READ", "WRITE");

// BAD: Throws UnsupportedOperationException
// currentList.add("EXECUTE");

// GOOD: Derive a new collection instance
List<String> updatedList = Stream.concat(currentList.stream(), Stream.of("EXECUTE"))
                                 .toList();
```

---

## 5. Benefits of Immutability

1. **Thread Safety**: Immutable objects can be shared freely across threads without synchronization (`synchronized` or locks).
2. **Simplifies Reasoning**: Eliminates unexpected side effects when passing objects to external methods or libraries.
3. **Safe Map Keys & Set Elements**: The hash code of an immutable object never changes, making it safe for `HashMap` and `HashSet`.
4. **Failure Atomicity**: Immutable objects are either in a valid state or fail during construction; they cannot enter corrupted states mid-execution.