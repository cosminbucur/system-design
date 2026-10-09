You should choose **only the fields used in the `equals()` method** for the `hashCode()` implementation. This ensures that equal objects always produce the same hash code, which is critical for the correct functioning of hash-based collections like `HashMap` and `HashSet`.

- **Subset Rule**: The fields used for hashing should be a **subset** of the fields used for equality checks. Including extra fields in `hashCode()` that are not used in `equals()` can cause equal objects to have different hash codes, breaking collection contracts.
- **Avoid Mutable Fields**: It is best practice to **exclude mutable fields** from `hashCode()` because changing a field's value after an object is stored in a hash collection can make it unretrievable.
- **Handle Nulls Safely**: For reference fields (like `String`), use a conditional check (e.g., `field == null ? 0 : field.hashCode()`) to avoid `NullPointerException`.
- **Standard Approach**: A common and robust pattern is to start with a non-zero constant (e.g., 31 or 17) and combine the hash codes of the selected fields using a prime number multiplier (typically 31).

```java
@Override
public int hashCode() {
    int result = 31; // Non-zero constant
    result = 31 * result + (id == null ? 0 : id.hashCode());
    result = 31 * result + (name == null ? 0 : name.hashCode());
    return result;
}
```

Alternatively, you can use **`java.util.Objects.hash(...)`** (Java 7+) or libraries like **Apache Commons Lang's `HashCodeBuilder`** to simplify the combination of these selected fields.
