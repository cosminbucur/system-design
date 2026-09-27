# Java Concurrency: `volatile` vs `Atomic`

In Java concurrent programming, managing shared state between threads requires a clear understanding of two fundamental concepts: **Visibility** and **Atomicity**. While both `volatile` and `Atomic` classes (like `AtomicInteger`) address thread safety, they serve distinct purposes and use different underlying mechanisms.

![alt text](volatile-atomic.png)

---

## At a Glance

| Feature                  | `volatile` (Keyword)                                              | `Atomic` (e.g., `AtomicInteger`)                                       |
| :----------------------- | :---------------------------------------------------------------- | :--------------------------------------------------------------------- |
| **Primary Focus**        | **Visibility**                                                    | **Visibility AND Atomicity**                                           |
| **Memory Guarantee**     | Reads and writes bypass CPU cache and go straight to Main Memory. | Inherits `volatile` memory guarantees plus atomic compound operations. |
| **Compound Operations**  | **Not Thread-Safe** (e.g., `count++` causes data loss).           | **Thread-Safe** (e.g., `incrementAndGet()` executes atomically).       |
| **Underlying Mechanism** | Memory Barriers / CPU cache flushing.                             | **Compare-And-Swap (CAS)** hardware instructions.                      |
| **Locking Mechanism**    | Non-blocking, no lock overhead.                                   | Lock-free (optimistic concurrency control).                            |
| **Use Case**             | Single flag updates (e.g., status flags, cancellation signals).   | Shared counters, sequence generators, accumulation registers.          |

---

## 1. The `volatile` Keyword

The `volatile` keyword guarantees **visibility** of changes to variables across threads.

### How It Works

When a variable is declared `volatile`, the Java Virtual Machine (JVM) ensures that:

1. **Direct Memory Access:** Every read of the variable is fetched directly from main memory rather than from a CPU cache.
2. **Immediate Flush:** Every write to the variable is immediately written back to main memory.
3. **Instruction Reordering Prevention:** The compiler and CPU are prevented from reordering instructions around the `volatile` variable (via memory barriers).

### The Limitation of `volatile`

`volatile` does **not** make compound operations atomic.

```java
public class SharedCounter {
    private volatile int count = 0;

    public void increment() {
        // NOT THREAD-SAFE!
        count++;
    }
}
```

The expression `count++` is actually a **three-step non-atomic operation**:

1. **Read** the current value of `count` from main memory.
2. **Modify** the value locally ($count + 1$).
3. **Write** the updated value back to main memory.

#### Race Condition Example

If Thread A and Thread B both call `increment()` concurrently:

1. Thread A reads `count = 5`.
2. Thread B reads `count = 5`.
3. Thread A computes $5 + 1 = 6$ and writes `6`.
4. Thread B computes $5 + 1 = 6$ and writes `6`.

**Result:** `count` ends up as `6` instead of `7`. One increment was lost despite using `volatile`.

---

## 2. The `Atomic` Classes (`java.util.concurrent.atomic`)

`Atomic` classes (such as `AtomicInteger`, `AtomicBoolean`, `AtomicReference`) provide **both visibility and atomicity** for single variables without using heavy synchronization locks.

### How It Works: Compare-And-Swap (CAS)

Instead of using explicit locks (`synchronized` or `ReentrantLock`), atomic classes rely on low-level CPU instructions known as **Compare-And-Swap (CAS)**.

A CAS operation works like this:

1. It takes three parameters:
   - Memory Location ($V$)
   - Expected Old Value ($A$)
   - New Value ($B$)
2. The CPU atomically checks if the value at $V$ equals $A$.
3. If true, the CPU updates $V$ to $B$ and returns `true`.
4. If false (another thread modified $V$ in the meantime), it leaves $V$ unchanged, returns `false`, and typically retries in a loop.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadSafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // THREAD-SAFE! Uses CAS under the hood
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

---

## Summary Guidelines

### When to use `volatile`:

- You have a **single variable** modified by **one thread** and read by **many threads**.
- You need a simple state flag (e.g., `volatile boolean shutdownRequested = false;`).
- Operations are purely single assignment/read (e.g., updating an object reference swap).

### When to use `Atomic` classes:

- Multiple threads **read and write** the variable concurrently.
- You need compound operations like incrementing (`incrementAndGet()`), decrementing, or conditional updates (`compareAndSet()`).
- You want high performance without the overhead and potential deadlocks of `synchronized` blocks.
