# Java BigDecimal Comparison and Initialization Guide

This document summarizes the best practices for comparing and initializing `BigDecimal` objects in Java, based on our session.

---

## 1. How to Compare BigDecimals

Never use `==` or `.equals()` for standard numerical comparisons. Instead, use the **`compareTo()`** method.

### The Comparison Operators Cheat Sheet

| Condition        | `BigDecimal` Equivalent                                              |
| :--------------- | :------------------------------------------------------------------- |
| **`price <= 0`** | `price.compareTo(BigDecimal.ZERO) <= 0`                              |
| **`price < 0`**  | `price.compareTo(BigDecimal.ZERO) < 0`                               |
| **`price == 0`** | `price.compareTo(BigDecimal.ZERO) == 0` _(or `price.signum() == 0`)_ |
| **`price > 0`**  | `price.compareTo(BigDecimal.ZERO) > 0`                               |
| **`price >= 0`** | `price.compareTo(BigDecimal.ZERO) >= 0`                              |

### Why `.equals()` and `==` fail:

- **`==`**: Compares object memory references, not values. Two different instances of `5` will return `false`.
- **`.equals()`**: Compares both value **and scale**. `2.00` and `2.0` are considered **not equal** because their scales (`2` vs `1`) differ.

ℹ️ Pro-Tip: `signum()`
If you just want to check if a number is negative, zero, or positive, BigDecimal has a built-in method called signum() that returns -1, 0, or 1

---

## 2. Initialization: `new BigDecimal()` vs `BigDecimal.valueOf()`

### Key Differences

- **`new BigDecimal(double)`**: Prone to precision errors because binary floating-point numbers cannot precisely represent certain decimals (e.g., `new BigDecimal(0.1)` becomes `0.10000000000000000555...`).
- **`BigDecimal.valueOf(double)`**: Safely converts the `double` to a string behind the scenes (`Double.toString(val)`) and caches frequently used values (like `0` through `10`) for performance.

### Best Practices

1. **Use `BigDecimal.valueOf(...)`** when initializing from primitive numbers (`double` or `long`).
2. **Use `new BigDecimal("...")` (with a String)** when parsing from text, user input, or JSON to ensure absolute precision.
3. **Avoid `new BigDecimal(double)`** directly.
