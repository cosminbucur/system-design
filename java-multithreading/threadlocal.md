# ThreadLocal

- **`ThreadLocal`**: Provides thread-isolated, mutable copies of variables. Useful for holding state tied to individual threads (e.g., user sessions in web servers).

## 1. Code Examples

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

## 2. Key Differences in How They Work

| Feature                      | `ThreadLocal`                                                                                                      | `ScopedValue`                                                                              |
| :--------------------------- | :----------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- |
| **Storage Mechanism**        | Managed via a per-thread hash map (`ThreadLocalMap`).                                                              | Managed within a bounded execution context/scope.                                          |
| **Mutability**               | **Mutable**: Threads can set, change, or remove values anytime.                                                    | **Immutable**: Value cannot be modified once set for a scope.                              |
| **Scope & Lifecycle**        | Tied to thread lifetime; requires manual cleanup (`.remove()`) to avoid memory leaks (especially in thread pools). | Tied strictly to execution scope; automatically cleaned up when the scope exits.           |
| **Child Thread Propagation** | Does not naturally propagate to child threads without extra mechanisms (e.g., `InheritableThreadLocal`).           | Built for structured concurrency and Project Loom virtual threads to easily pass contexts. |

---

## 3. When to Use?

### Use `ThreadLocal` when:

- Building on legacy systems or standard thread-per-request architectures.
- Threads require independent, **mutable** local state.
