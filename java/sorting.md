# Java Sorting: Comparable vs Comparator

In Java, sorting collections or arrays of custom objects requires defining ordering logic. Java provides two interfaces for this purpose: **`Comparable`** and **`Comparator`**.

---

## Quick Comparison

| Feature | `Comparable<T>` | `Comparator<T>` |
| :--- | :--- | :--- |
| **Purpose** | Defines the **natural/default** sorting order for a class. | Defines **custom or alternative** sorting orders. |
| **Package** | `java.lang` | `java.util` |
| **Primary Method** | `compareTo(T o)` | `compare(T o1, T o2)` |
| **Class Modification** | Must modify the original class source code. | Does **not** require modifying the target class. |
| **Flexibility** | Provides a single sort strategy per class. | Allows multiple sorting strategies for the same class. |
| **Usage Example** | `Collections.sort(list)` | `list.sort(comparator)` |

---

## 1. `Comparable`: Natural Ordering

The `Comparable` interface is implemented by a class to give its instances a natural ordering (e.g., `String` alphabetically, `Integer` numerically).

### Implementing `compareTo`
The `compareTo` method compares `this` object with the specified object `o`:
* Returns a **negative integer** if `this` is less than `o`.
* Returns **zero** if `this` is equal to `o`.
* Returns a **positive integer** if `this` is greater than `o`.

### Code Example

```java
package com.example.sorting;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Employee implements Comparable<Employee> {
    private int id;
    private String name;
    private double salary;

    public Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }

    public int getId() { return id; }
    public String getName() { return name; }
    public double getSalary() { return salary; }

    // Natural sort order: Ascending by ID
    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);
    }

    @Override
    public String toString() {
        return String.format("Employee{id=%d, name='%s', salary=%.2f}", id, name, salary);
    }

    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>();
        employees.add(new Employee(103, "Alice", 75000));
        employees.add(new Employee(101, "Bob", 60000));
        employees.add(new Employee(102, "Charlie", 90000));

        // Sorts using Employee's compareTo method
        Collections.sort(employees);

        employees.forEach(System.out::println);
    }
}
```

---

## 2. `Comparator`: Custom Ordering

Use `Comparator` when:
1. You cannot modify the original class source code.
2. You need multiple, distinct sorting strategies for the same object (e.g., sort by name, sort by salary descending).

---

### Option A: Modern Java (Lambdas & Factory Methods)

Java 8+ introduced fluent key-extractor factory methods on `Comparator` such as `Comparator.comparing()`.

```java
package com.example.sorting;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public class ComparatorLambdaExample {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee(103, "Alice", 75000),
            new Employee(101, "Bob", 90000),
            new Employee(102, "Bob", 60000)
        ));

        // 1. Sort by Name (Ascending)
        employees.sort(Comparator.comparing(Employee::getName));

        // 2. Sort by Salary (Descending)
        employees.sort(Comparator.comparingDouble(Employee::getSalary).reversed());

        // 3. Chained Sorting: Sort by Name, then by Salary if names match
        employees.sort(
            Comparator.comparing(Employee::getName)
                      .thenComparingDouble(Employee::getSalary)
        );

        employees.forEach(System.out::println);
    }
}
```

---

### Option B: Standalone Class Implementation

You can also create dedicated comparator classes for reusability across projects.

```java
package com.example.sorting;

import java.util.Comparator;

public class EmployeeSalaryComparator implements Comparator<Employee> {
    @Override
    public int compare(Employee e1, Employee e2) {
        return Double.compare(e1.getSalary(), e2.getSalary());
    }
}
```

**Usage:**
```java
employees.sort(new EmployeeSalaryComparator());
```

---

## Best Practices & Common Pitfalls

1. **Avoid Subtraction Pitfalls:**
   * **Bad:** `return this.id - other.id;` (Can overflow if values are near `Integer.MIN_VALUE` or `Integer.MAX_VALUE`).
   * **Good:** `return Integer.compare(this.id, other.id);`

2. **Ensure Consistency with `equals`:**
   * It is strongly recommended that `(x.compareTo(y) == 0) == (x.equals(y))`.
   * Collections like `TreeSet` and `TreeMap` use `compareTo` or `compare` instead of `equals` to determine equality and duplicate entries.

3. **Handle `null` Values Gracefully:**
   Use `Comparator.nullsFirst` or `Comparator.nullsLast` to prevent `NullPointerException` during sorting:
   ```java
   Comparator<Employee> safeNameSort = Comparator.comparing(
       Employee::getName, 
       Comparator.nullsFirst(Comparator.naturalOrder())
   );
   ```