# Java Date and Time Guide (java.time API)

This document summarizes best practices for working with dates and times in modern Java (Java 8+ `java.time` package, built on JSR-310).

---

## 1. Choosing the Right Class

Never use the legacy `java.util.Date` or `java.util.Calendar`. They are mutable, thread-unsafe, and poorly designed. Instead, use the modern `java.time` classes based on your specific use case:

| Use Case | Recommended Class | Example | Description |
| :--- | :--- | :--- | :--- |
| **Date only** (e.g., birthdays, holidays) | `LocalDate` | `LocalDate.of(2026, 10, 5)` | No time, no time zone. |
| **Time only** (e.g., store opening hours) | `LocalTime` | `LocalTime.of(14, 30)` | No date, no time zone. |
| **Date and Time** (local context) | `LocalDateTime` | `LocalDateTime.now()` | Combines date and time, but still no time zone. |
| **Global timestamp** (exact point in time) | `Instant` | `Instant.now()` | Machine-readable timestamp (UTC). Best for logs and database storage. |
| **Date/Time with Time Zone** | `ZonedDateTime` or `OffsetDateTime` | `ZonedDateTime.now(ZoneId.of("America/New_York"))` | Full context including zone rules or UTC offset. |

---

## 2. How to Compare Dates and Times

Just like `BigDecimal`, you should avoid legacy comparison methods. The modern `java.time` classes implement `Comparable` and provide clean, readable methods.

### Comparison Methods

* **`isBefore()`**: Checks if a date/time is earlier than another.
* **`isAfter()`**: Checks if a date/time is later than another.
* **`isEqual()`**: Checks for equality (handles time zones correctly where applicable).
* **`compareTo()`**: Returns negative, zero, or positive integers like standard Java comparators.

```java
LocalDate today = LocalDate.now();
LocalDate deadline = LocalDate.of(2026, 12, 31);

if (today.isBefore(deadline)) {
    // You still have time!
}

if (today.isEqual(deadline)) {
    // Due today!
}
```

---

## 3. Formatting and Parsing

To convert strings to dates and dates to strings, use **`DateTimeFormatter`**. It is thread-safe and immutable (unlike the old `SimpleDateFormat`).

### Parsing a String into a Date
```java
String dateStr = "2026-10-05";
LocalDate date = LocalDate.parse(dateStr, DateTimeFormatter.ISO_LOCAL_DATE);

// Or with a custom pattern:
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate customDate = LocalDate.parse("05/10/2026", formatter);
```

### Formatting a Date into a String
```java
LocalDate now = LocalDate.now();
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("MMMM d, yyyy");
String formattedDate = now.format(formatter); // e.g., "October 5, 2026"
```

---

## 4. Summary Best Practices

1. **Never use `java.util.Date` or `java.util.Calendar`** in new code.
2. **Use `Instant` for database storage and logging** when you need a universal machine timestamp.
3. **Use `LocalDate` or `LocalDateTime`** when dealing with user-facing local schedules where time zones don't matter or shouldn't shift.
4. **Use `DateTimeFormatter`** for all string parsing and formatting (and note that it is thread-safe, unlike `SimpleDateFormat`).