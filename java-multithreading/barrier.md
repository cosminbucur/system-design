# Java Multithreading: The Barrier Pattern (`CyclicBarrier`)

This document provides a comprehensive summary of the **Barrier Pattern** in Java multithreading, covering core concepts, real-world examples, failure handling with `BrokenBarrierException`, and comparison with `CountDownLatch`.

---

## 1. Overview of the Barrier Pattern

The **Barrier Pattern** is a concurrency design pattern used when a fixed set of threads must all wait for each other to reach a specific execution point (the "barrier") before any of them can proceed.

In Java, this is implemented by `java.util.concurrent.CyclicBarrier`. It is called *cyclic* because it can be reused after the waiting threads are released.

### Core Concepts

* **Parties:** The fixed number of threads that must reach the barrier.
* **`await()`:** Called by each thread when it finishes its work for the phase. The thread blocks until the required number of threads have also called `await()`.
* **Barrier Action (Optional):** A `Runnable` task that automatically runs *once* when all threads reach the barrier, right before they are unblocked.

```
Thread 1 ---> [ Work Phase 1 ] ---\
Thread 2 ---> [ Work Phase 1 ] ----+---> [ BARRIER ACTION ] ---> [ Work Phase 2 ]
Thread 3 ---> [ Work Phase 1 ] ---/
```

---

## 2. Real-World Analogy & Basic Example

### Scenario: Multi-Part File Processing & Aggregation
Imagine a financial system that processes daily transaction reports split into three parallel chunks (e.g., Credit, Debit, and Wire Transfers).

1. 3 worker threads process their individual data chunks concurrently.
2. All 3 worker threads wait at a barrier until everyone finishes.
3. **Barrier Action:** A single aggregator task runs to combine the results into a final daily balance sheet.
4. The worker threads are released to begin the next phase (e.g., sending notification emails).

### Basic Java Code Implementation

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class BarrierPatternDemo {

    private static final int NUM_WORKERS = 3;

    public static void main(String[] args) {
        // 1. Create a CyclicBarrier for 3 threads with an optional barrier action
        CyclicBarrier barrier = new CyclicBarrier(NUM_WORKERS, () -> {
            System.out.println("\n--- BARRIER ACTION: All threads ready. Aggregating results... ---\n");
        });

        // 2. Start worker threads
        for (int i = 1; i <= NUM_WORKERS; i++) {
            new Thread(new ReportWorker("Worker-" + i, barrier)).start();
        }
    }

    static class ReportWorker implements Runnable {
        private final String name;
        private final CyclicBarrier barrier;

        public ReportWorker(String name, CyclicBarrier barrier) {
            this.name = name;
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                // Phase 1: Independent Work
                System.out.println(name + " starts processing data...");
                Thread.sleep((long) (Math.random() * 2000 + 1000));
                System.out.println(name + " finished processing. Waiting at barrier...");

                // Block until all threads reach this line
                barrier.await();

                // Phase 2: Post-barrier execution
                System.out.println(name + " proceeds to the post-processing stage.");

            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

---

## 3. `CyclicBarrier` vs. `CountDownLatch`

| Feature | `CyclicBarrier` | `CountDownLatch` |
| :--- | :--- | :--- |
| **Reusability** | **Reusable** via `.reset()` or automatically after tripping. | **One-time use**; count cannot be reset once it reaches zero. |
| **Waiting Target** | **Threads wait for each other**. | One or more threads wait for **tasks/events** to complete. |
| **Action Execution** | Supports a built-in `Runnable` barrier action. | No built-in barrier action callback. |

---

## 4. Understanding `BrokenBarrierException`

`BrokenBarrierException` signals that a `CyclicBarrier` has entered a **broken state** and can no longer guarantee that all participating threads will reach the barrier together.

Once a barrier is broken:
* Any thread currently waiting at `barrier.await()` immediately throws `BrokenBarrierException`.
* Any new thread attempting to call `await()` will immediately throw `BrokenBarrierException`.

### What Causes a Barrier to Break?

1. **Thread Interruption:** A waiting thread receives an `interrupt()` signal while inside `await()`.
2. **Timeout Expiration:** A thread calling `await(timeout, unit)` times out before all parties arrive.
3. **Barrier Action Failure:** The `Runnable` barrier action throws an unhandled runtime exception.
4. **Explicit Reset:** A thread calls `barrier.reset()` while other threads are blocked in `await()`.

### Why Does Java Break the Barrier?

The pattern relies on **all-or-nothing completion**. If one thread fails or times out, downstream operations dependent on that thread's results would be invalid. To prevent other threads from waiting indefinitely, Java breaks the barrier so all threads wake up, fail fast, and execute cleanup routines.

---

## 5. Resilient Error Handling (Timeouts & Interrupts)

To build robust concurrent systems with `CyclicBarrier`:

1. **Use timed `await()`:** Use `await(timeout, unit)` instead of plain `await()` to avoid indefinite deadlocks.
2. **Catch specific exceptions:** Handle `TimeoutException`, `BrokenBarrierException`, and `InterruptedException` independently.
3. **Reset and Clean up:** Roll back partial state, release locks, and call `barrier.reset()` if you plan to reuse the barrier.

### Resilient Code Implementation

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class ResilientBarrierDemo {

    private static final int NUM_WORKERS = 3;
    private static final CyclicBarrier barrier = new CyclicBarrier(NUM_WORKERS, () -> {
        System.out.println("\nAll tasks completed successfully! Aggregating results...\n");
    });

    public static void main(String[] args) {
        new Thread(new Worker("Worker-1", 1000)).start();
        new Thread(new Worker("Worker-2", 1000)).start();
        // Worker-3 takes 5 seconds, triggering a timeout for the barrier
        new Thread(new Worker("Worker-3", 5000)).start();
    }

    static class Worker implements Runnable {
        private final String name;
        private final int workDurationMs;

        public Worker(String name, int workDurationMs) {
            this.name = name;
            this.workDurationMs = workDurationMs;
        }

        @Override
        public void run() {
            try {
                System.out.println(name + " starting work...");
                Thread.sleep(workDurationMs);
                
                System.out.println(name + " reached barrier, waiting (max 2s)...");
                barrier.await(2, TimeUnit.SECONDS);
                
                System.out.println(name + " finished post-barrier work.");

            } catch (TimeoutException e) {
                System.err.println("[" + name + " ERROR] Timed out waiting at barrier! Aborting operation.");
                barrier.reset(); // Reset barrier for reuse

            } catch (BrokenBarrierException e) {
                System.err.println("[" + name + " ERROR] Another thread failed/timed out. Barrier broken!");
                // Perform local rollback or cleanup

            } catch (InterruptedException e) {
                System.err.println("[" + name + " ERROR] Interrupted while waiting.");
                Thread.currentThread().interrupt(); // Restore interrupt status
            }
        }
    }
}
```

---

## 6. CyclicBarrier Inspection Methods

* `barrier.isBroken()` — Returns `true` if the barrier is in a broken state.
* `barrier.getNumberWaiting()` — Returns the number of threads currently waiting at `await()`.
* `barrier.getParties()` — Returns the total number of parties required to trip the barrier.