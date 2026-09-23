# Java ReadWriteLock Mechanics & Real-World Application

A `ReentrantReadWriteLock` (part of `java.util.concurrent.locks`) enables concurrent shared access for multiple reader threads while ensuring exclusive access for writer threads.

---

## 1. Core Mechanics & State Representation

The lock manages state using a single 32-bit `int` inside **AbstractQueuedSynchronizer (AQS)**:

* **High 16 bits:** Shared read lock count.
* **Low 16 bits:** Exclusive write lock hold count.

```
 32-bit AQS state integer:
 ┌───────────────────────────────┬───────────────────────────────┐
 │       Read Lock Count         │       Write Lock Count        │
 │          (16 bits)            │          (16 bits)            │
 └───────────────────────────────┴───────────────────────────────┘
  31                           16 15                            0
```

### State Bitwise Operations
* **Read Count:** `state >>> 16`
* **Write Count:** `state & 0x0000FFFF`

---

## 2. Lock Acquisition Workflows

### Write Lock Acquisition (`writeLock().lock()`)

```
                  ┌─────────────────────────────────┐
                  │      writeLock().lock()         │
                  └────────────────┘────────────────┘
                                   │
                                   ▼
                   ┌───────────────────────────────┐
                   │ Is overall lock state != 0?   │
                   └───────────────┬───────────────┘
                                   │
                         ┌─────────┴─────────┐
                      NO │                   │ YES
                         ▼                   ▼
      ┌────────────────────────────┐   ┌───────────────────────────────┐
      │ Acquire write lock via CAS │   │  Is read count > 0 OR write   │
      └────────────────────────────┘   │ hold owned by another thread? │
                                       └───────────────┬───────────────┘
                                                       │
                                             ┌─────────┴─────────┐
                                          YES│                   │ NO
                                             ▼                   ▼
                              ┌────────────────────┐   ┌───────────────────┐
                              │ Block in AQS queue │   │ Increment reentrant│
                              └────────────────────┘   │ write count       │
                                                       └───────────────────┘
```

### Read Lock Acquisition (`readLock().lock()`)

```
                  ┌─────────────────────────────────┐
                  │       readLock().lock()         │
                  └────────────────┘────────────────┘
                                   │
                                   ▼
                   ┌───────────────────────────────┐
                   │     Is write count > 0?       │
                   └───────────────┬───────────────┘
                                   │
                         ┌─────────┴─────────┐
                      NO │                   │ YES
                         ▼                   ▼
      ┌────────────────────────────┐   ┌───────────────────────────────┐
      │ Increment read count (CAS) │   │ Is current thread the writer? │
      └────────────────────────────┘   └───────────────┬───────────────┘
                                                       │
                                             ┌─────────┴─────────┐
                                          YES│                   │ NO
                                             ▼                   ▼
                              ┌────────────────────┐   ┌───────────────────┐
                              │ Lock Downgrading:  │   │ Block in AQS      │
                              │ Acquire read lock  │   │ queue             │
                              └────────────────────┘   └───────────────────┘
```

---

## 3. Key Rules & Behavioral Properties

* **Lock Downgrading:** Supported. A thread holding a write lock can acquire a read lock, and then release its write lock, safely preserving read access.
* **Lock Upgrading:** **Not supported.** A thread holding a read lock cannot directly acquire a write lock. Attempting this across multiple threads causes instant deadlocks.
* **Thread Tracking:** While total read counts are stored in AQS, Java utilizes `ThreadLocalHoldCounter` to keep track of per-thread reentrant read counts.

---

## 4. Real-World Application: In-Memory Feature Toggle Service

A primary real-world use case is an **In-Memory Cache with High Read-to-Write Ratios** (e.g., feature flag lookups, configuration data, or routing tables).

### Practical Example

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class FeatureToggleService {

    private final Map<String, Boolean> featureFlags = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    /**
     * Called on EVERY incoming API request (High Concurrency / Non-blocking).
     */
    public boolean isFeatureEnabled(String featureKey) {
        readLock.lock();
        try {
            // Concurrent threads can read simultaneously without blocking each other
            return featureFlags.getOrDefault(featureKey, false);
        } finally {
            readLock.unlock();
        }
    }

    /**
     * Called when an admin updates flags (Infrequent / Exclusive Access).
     */
    public void updateFlags(Map<String, Boolean> updatedFlags) {
        writeLock.lock();
        try {
            // Blocks all incoming readers until the batch update completes
            featureFlags.putAll(updatedFlags);
        } finally {
            writeLock.unlock();
        }
    }
}
```

---

## 5. When to Use vs. Alternatives

| Mechanism | Ideal Use Case | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **`ReentrantReadWriteLock`** | Read-heavy workloads ($>90\%$ reads) with longer critical sections. | Shared concurrent reads; reentrant; simple model. | Overhead of state management and ThreadLocals. |
| **`ConcurrentHashMap`** | Key-value read/write access. | Extremely fast bucket-level concurrency. | Cannot lock across complex bulk structures atomically without extra logic. |
| **`StampedLock`** | Read-dominated workloads requiring high performance (Java 8+). | Provides optimistic reading without CAS overhead. | Non-reentrant; usage complexity. |