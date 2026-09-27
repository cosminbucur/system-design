# ThreadLocal vs ScopedValue in Java: Summary

This document summarizes Alex Klimenko's article **"ThreadLocal vs ScopedValue"**, comparing how Java manages thread state across traditional and modern concurrency paradigms.

---

## Overview

* **`ThreadLocal`**: Provides thread-isolated, mutable copies of variables. Useful for holding state tied to individual threads (e.g., user sessions in web servers).
* **`ScopedValue`**: Introduced in **Java 20** (via Project Loom) to share **immutable data within a bounded execution scope**, designed specifically to work seamlessly with virtual threads and structured concurrency.

---

## 1. Code Examples

### `ThreadLocal` Example
`ThreadLocal` allows each thread in a thread pool to manage its own isolated, mutable copy of a variable:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalExample {
    private static ThreadLocal<Integer> threadLocalValue = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        
        Runnable task = () -> {
            int value = threadLocalValue.get();
            value += (int) (Math.random() * 100);
            threadLocalValue.set(value);
            
            System.out.println(Thread.currentThread().getName() + " initial value: " + value);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " final value: " + threadLocalValue.get());
        };

        executor.submit(task);
        executor.submit(task);
        executor.shutdown();
    }
}
```

#### Simplified Usage:
```java
public class ThreadLocalExample {
    private static ThreadLocal<Integer> threadLocalValue = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        threadLocalValue.set(123);
        Integer value = threadLocalValue.get(); // Retrieves 123 for the current thread
    }
}
```

---

### `ScopedValue` Example
`ScopedValue` defines an immutable value bound strictly to the execution lifecycle of a specific block/scope:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import jdk.incubator.concurrent.ScopedValue;

public class ScopedValueExample {
    private static final ScopedValue<String> scopedValue = ScopedValue.newInstance();

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Runnable task = () -> {
            ScopedValue.where(scopedValue, "Value in " + Thread.currentThread().getName()).run(() -> {
                System.out.println(Thread.currentThread().getName() + " initial value: " + scopedValue.get());
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " final value: " + scopedValue.get());
            });
        };

        executor.submit(task);
        executor.submit(task);
        executor.shutdown();
    }
}
```

#### Simplified Usage:
```java
import jdk.incubator.concurrent.ScopedValue;

public class ScopedValueExample {
    private static final ScopedValue<String> scopedValue = ScopedValue.newInstance();

    public static void main(String[] args) {
        ScopedValue.where(scopedValue, "Scoped Value Example").run(() -> {
            String value = scopedValue.get(); // Retrieves "Scoped Value Example" within this scope
            System.out.println(value);
        });
    }
}
```

---

## 2. Key Differences in How They Work

| Feature | `ThreadLocal` | `ScopedValue` |
| :--- | :--- | :--- |
| **Storage Mechanism** | Managed via a per-thread hash map (`ThreadLocalMap`). | Managed within a bounded execution context/scope. |
| **Mutability** | **Mutable**: Threads can set, change, or remove values anytime. | **Immutable**: Value cannot be modified once set for a scope. |
| **Scope & Lifecycle** | Tied to thread lifetime; requires manual cleanup (`.remove()`) to avoid memory leaks (especially in thread pools). | Tied strictly to execution scope; automatically cleaned up when the scope exits. |
| **Child Thread Propagation** | Does not naturally propagate to child threads without extra mechanisms (e.g., `InheritableThreadLocal`). | Built for structured concurrency and Project Loom virtual threads to easily pass contexts. |

---

## 3. When to Use Which?

### Use `ThreadLocal` when:
* Building on legacy systems or standard thread-per-request architectures.
* Threads require independent, **mutable** local state.

### Use `ScopedValue` when:
* Building modern Java applications leveraging **virtual threads** and **structured concurrency**.
* You need safe, **immutable** context propagation (e.g., security context, request tracing IDs).
* Automatic context cleanup and prevention of memory leaks are top priorities.