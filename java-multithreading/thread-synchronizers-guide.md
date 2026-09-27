# High-Level Thread Synchronizers in Java

Java's `java.util.concurrent` package provides high-level thread synchronizers that manage complex inter-thread coordination patterns without requiring low-level `synchronized` blocks or manual `wait()` / `notify()` mechanics.

---

## 1. CountDownLatch

A `CountDownLatch` blocks a set of threads until a specified count is decremented to zero. It acts as a **one-time gate**: once open, the latch cannot be reset or reused.

### Key Concepts & Use Cases
* **Key Concept:** Threads wait for $N$ events to complete. Decrementing threads do not need to pause—they decrement and proceed.
* **Practical Applications:** Application startup checks (waiting for DB connections, cache warming, and external services before starting the server), batch processing completion, concurrent execution benchmarks.

### Code Example

```java
import java.util.concurrent.CountDownLatch;

public class LatchExample {
    public static void main(String[] args) throws InterruptedException {
        int workerCount = 3;
        CountDownLatch latch = new CountDownLatch(workerCount);

        for (int i = 1; i <= workerCount; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    System.out.println("Service " + id + " initializing...");
                    Thread.sleep(1000 * id); // Simulate startup work
                    System.out.println("Service " + id + " READY.");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown(); // Decrement count
                }
            }).start();
        }

        System.out.println("Main thread waiting for services to initialize...");
        latch.await(); // Block until count reaches 0
        System.out.println("All services initialized. Application STARTED.");
    }
}
```

---

## 2. CyclicBarrier

A `CyclicBarrier` enables a fixed set of threads to rendezvous at a common execution point. Unlike `CountDownLatch`, a `CyclicBarrier` is **reusable** (cyclic) and resets automatically after all threads arrive.

### Key Concepts & Use Cases
* **Key Concept:** Threads wait for *each other*. All participating threads block at `await()` until the last thread arrives. It can also execute an optional barrier action when the barrier trips.
* **Practical Applications:** Multi-stage parallel computations (e.g., iterative simulation loops, parallel matrix operations, or multi-threaded game engine frame processing).

### Code Example

```java
import java.util.concurrent.CyclicBarrier;

public class BarrierExample {
    public static void main(String[] args) {
        int partyCount = 3;
        
        // Barrier action runs automatically when all threads arrive
        CyclicBarrier barrier = new CyclicBarrier(partyCount, () -> 
            System.out.println("--- All threads reached phase barrier. Aggregating results... ---\n")
        );

        for (int i = 1; i <= partyCount; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    // Phase 1 Work
                    System.out.println("Worker " + id + " completed Phase 1.");
                    barrier.await(); // Block until all 3 workers reach here

                    // Phase 2 Work
                    System.out.println("Worker " + id + " completed Phase 2.");
                    barrier.await(); // Reused! Block until all reach Phase 2 finish
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

---

## 3. Semaphore

A `Semaphore` maintains a fixed number of **permits** to restrict concurrent access to a shared resource. Threads must acquire a permit before proceeding and release it when finished.

### Key Concepts & Use Cases
* **Key Concept:** Rate-limiting or resource-bounding concurrency rather than pure thread coordination.
* **Practical Applications:** Connection pooling, API rate limiting, limiting concurrent file/disk writes, or bounding heavy memory resource usage.

### Code Example

```java
import java.util.concurrent.Semaphore;

public class SemaphoreExample {
    public static void main(String[] args) {
        // Only 2 permits available (max 2 threads at once)
        Semaphore pool = new Semaphore(2);

        for (int i = 1; i <= 5; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    System.out.println("Worker " + id + " waiting for connection permit...");
                    pool.acquire(); // Blocks if 0 permits are left
                    
                    System.out.println("Worker " + id + " acquired connection. Processing...");
                    Thread.sleep(1500); 
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    System.out.println("Worker " + id + " releasing connection.");
                    pool.release(); // Return permit to the pool
                }
            }).start();
        }
    }
}
```

---

## 4. Phaser

A `Phaser` is a flexible, dynamic synchronization barrier (introduced in Java 7). It supports multi-phase synchronization similar to `CyclicBarrier`, but allows the **number of registered threads to change dynamically** over time.

### Key Concepts & Use Cases
* **Key Concept:** Threads can register, arrive, await, or deregister (`arriveAndDeregister()`) on the fly as execution progresses across discrete phase numbers.
* **Practical Applications:** Dynamic pipeline processing, parallel task decomposition with dynamic sub-tasks (e.g., parallel web crawling where discovered links spawn new tasks), phased workflow orchestration.

### Code Example

```java
import java.util.concurrent.Phaser;

public class PhaserExample {
    public static void main(String[] args) {
        // Register main thread initially
        Phaser phaser = new Phaser(1);

        for (int i = 1; i <= 3; i++) {
            phaser.register(); // Dynamically register new worker thread
            final int id = i;
            new Thread(() -> {
                System.out.println("Worker " + id + " starting Phase 0.");
                phaser.arriveAndAwaitAdvance(); // Signal completion and wait for phase 0 to finish

                System.out.println("Worker " + id + " starting Phase 1.");
                phaser.arriveAndDeregister(); // Signal completion and unregister from future phases
            }).start();
        }

        // Main thread coordinates phase progression
        System.out.println("Main thread advancing Phase 0...");
        phaser.arriveAndAwaitAdvance(); // Advance Phase 0

        System.out.println("Main thread advancing Phase 1...");
        phaser.arriveAndAwaitAdvance(); // Advance Phase 1

        phaser.deregister(); // Deregister main thread
        System.out.println("All phases finished.");
    }
}
```

---

## Comparison Summary

| Synchronizer | Reusable? | Dynamic Threads? | Primary Purpose | Key Method(s) |
| :--- | :--- | :--- | :--- | :--- |
| **CountDownLatch** | No | No | Wait for $N$ completion events | `countDown()`, `await()` |
| **CyclicBarrier** | Yes | No | Wait for fixed $N$ threads to meet at a barrier | `await()` |
| **Semaphore** | N/A | N/A | Control access to a limited resource pool | `acquire()`, `release()` |
| **Phaser** | Yes | Yes | Coordinate multi-phase work with variable threads | `register()`, `arriveAndAwaitAdvance()` |