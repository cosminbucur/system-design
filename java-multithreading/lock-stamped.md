# Understanding the `StampedLock` Pattern in Java

The `StampedLock` is a synchronization mechanism introduced in Java 8 (under the `java.util.concurrent.locks` package) designed to provide optimal performance for read-heavy concurrency scenarios. Unlike traditional locks like `ReentrantLock` or `ReadWriteLock`, `StampedLock` introduces the concept of **optimistic reading**.

---

## How `StampedLock` Works

Traditional read-write locks block writers when readers are active, and vice versa. This can cause writer starvation if read traffic is heavy. `StampedLock` solves this with three modes:

1. **Writing (`writeLock()`)**: Exclusive access. Blocks all other readers and writers. Returns a `long` stamp used to unlock.
2. **Reading (`readLock()`)**: Pessimistic non-exclusive read. Blocks writers, but allows other concurrent readers. Returns a stamp.
3. **Optimistic Reading (`tryOptimisticRead()`)**: Non-blocking read that doesn't acquire any actual lock. It returns a stamp and lets you read shared state freely. Afterward, you **validate** whether a write occurred in the meantime. If validation fails, you fall back to a standard pessimistic read lock.

---

## Real-Life Application: In-Memory Configuration Cache

Imagine a high-frequency trading platform or web application that reads application configuration parameters thousands of times per second, but updates them only a few times a day (e.g., feature flags, timeout limits, rate limits). 

Using a standard `ReentrantLock` would create a bottleneck for reader threads. `StampedLock` with optimistic reading allows readers to fetch configuration data instantly with zero locking overhead, only validating or falling back to a read lock if a configuration update happens concurrently.

---

## Java Code Example

Here is a complete, runnable example demonstrating how to use `StampedLock` with write locks, pessimistic read locks, and the high-performance optimistic read pattern.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.StampedLock;

public class ConfigurationManager {
    private final StampedLock lock = new StampedLock();
    private final Map<String, String> config = new HashMap<>();

    // --- WRITE METHOD ---
    public void setConfig(String key, String value) {
        long stamp = lock.writeLock(); // Acquire exclusive write lock
        try {
            config.put(key, value);
            System.out.println("Updated config: " + key + " = " + value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    // --- OPTIMISTIC READ PATTERN ---
    public String getConfigOptimistic(String key) {
        // 1. Try an optimistic read (does not block writers)
        long stamp = lock.tryOptimisticRead();
        
        // 2. Read the state into local variables
        String value = config.get(key);

        // 3. Validate if a write occurred since we got the stamp
        if (!lock.validate(stamp)) {
            // Fallback to a standard pessimistic read lock if validation fails
            stamp = lock.readLock();
            try {
                value = config.get(key);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return value;
    }

    // --- STANDARD PESSIMISTIC READ ---
    public String getConfigPessimistic(String key) {
        long stamp = lock.readLock(); // Acquires shared read lock
        try {
            return config.get(key);
        } finally {
            lock.unlockRead(stamp);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ConfigurationManager manager = new ConfigurationManager();
        
        // Initial setup
        manager.setConfig("timeout", "5000");

        // Simulating a fast optimistic read
        System.out.println("Read timeout (optimistic): " + manager.getConfigOptimistic("timeout"));
    }
}
```

---

## Important Best Practices & Caveats

* **Not Reentrant**: Unlike `ReentrantLock`, `StampedLock` is **not reentrant**. If a thread holding a write lock attempts to acquire another write lock, it will deadlock.
* **No Condition Support**: `StampedLock` does not support `Condition` objects directly out of the box. If you need advanced thread coordination (like `await`/`signal`), stick to `ReentrantLock`.
* **Ideal Workloads**: Use optimistic reading only when reads vastly outnumber writes, and the critical section (the code reading state) is short.