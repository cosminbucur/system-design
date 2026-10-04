# Understanding How Java's `ReentrantLock` Works

https://www.youtube.com/watch?v=ahBC69_iyk4

`ReentrantLock` (from `java.util.concurrent.locks`) is an explicit lock implementation in Java that provides advanced thread synchronization capabilities beyond traditional `synchronized` blocks.

---

## 1. Underlying Architecture: AbstractQueuedSynchronizer (AQS)

At its core, `ReentrantLock` uses **`AbstractQueuedSynchronizer` (AQS)** to manage thread synchronization.

- **Synchronization State (`state`):** An atomic integer maintained by AQS that tracks lock ownership and reentrancy depth.
- **Wait Queue (FIFO):** A doubly-linked list of thread nodes managed by AQS. Threads that fail to acquire the lock are queued here and blocked using `LockSupport.park()`.

---

## 2. Reentrancy & Hold Count Mechanics

Reentrancy allows a thread that currently holds the lock to acquire it again without deadlocking.

1. **First Acquisition:**
   - When a thread acquires an unheld lock (`state == 0`), `state` is updated to `1`, and the thread is recorded as the exclusive owner.
2. **Reentrant Acquisition:**
   - If the owner thread requests the lock again, the acquisition succeeds immediately, and `state` increments ($1 \to 2 \to 3$).
3. **Releasing the Lock:**
   - Calling `unlock()` decrements the `state` count.
   - The lock is only fully released—and waiting threads unparked—when `state` reaches `0`.

```java
ReentrantLock lock = new ReentrantLock();

lock.lock(); // state = 1
try {
    lock.lock(); // state = 2 (reentered)
    try {
        // Critical section
    } finally {
        lock.unlock(); // state = 1
    }
} finally {
    lock.unlock(); // state = 0 (lock completely released)
}
```

**tryLock: Non-blocking Acquisition**

One of the most useful features of ReentrantLock is tryLock(), which attempts to acquire the lock without blocking. Here is a basic example.

```java
ReentrantLock lock = new ReentrantLock();

if (lock.tryLock()) {
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} else {
    // Lock not available, do something else
}
```

No equivalent exists for synchronized. You either block waiting for the lock, or you do not try at all. This makes tryLock() invaluable for implementing patterns like "try the primary resource, fall back to secondary if busy."

**Timed Lock Acquisition**

Building on tryLock(), you can specify a maximum wait time. This is essential for systems with SLAs or timeout requirements.

```java
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} else {
    // Timed out - take alternative action
    log.warn("Could not acquire lock within 100ms, returning cached value");
    return cachedValue;
}
```

Timed locking is invaluable for preventing deadlocks (if a lock cycle exists, one thread will eventually timeout) and implementing responsive systems that fail fast rather than hang indefinitely.

**Interruptible Lock Acquisition**

What if you want to cancel a thread that is waiting for a lock? With synchronized, you cannot. The thread will wait until it acquires the lock or the JVM terminates. ReentrantLock solves this with lockInterruptibly().

```java
try {
    lock.lockInterruptibly();  // Can be interrupted while waiting
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    // Handle interruption - clean up, return, etc.
    Thread.currentThread().interrupt();  // Restore interrupt status
    return;
}
```

This is essential for implementing graceful shutdown. When shutting down a server, you interrupt all worker threads. With synchronized, any thread blocked on a monitor wait continues blocking. With lockInterruptibly(), those threads receive an InterruptedException and can exit cleanly.

---

## 3. Core Working Features & Capabilities

### Fairness Policies

- **Non-Fair Mode (Default):**
  - When a thread calls `lock()`, it attempts to acquire ownership using CAS (Compare-And-Swap) immediately, even if other threads are waiting in the AQS queue.
  - **Tradeoff:** Higher performance and throughput, but risks thread starvation.
- **Fair Mode (`new ReentrantLock(true)`):**
  - Forces strict FIFO order. A thread will check if any other threads precede it in the queue before attempting to grab the lock.
  - **Tradeoff:** Prevents starvation, but incurs extra queue-checking overhead.

### Non-Blocking & Timed Lock Attempts (`tryLock`)

- `tryLock()` attempts immediate lock acquisition and returns `boolean` (`true` if successful, `false` otherwise) without blocking.
- `tryLock(time, timeUnit)` blocks for up to the specified timeout before giving up.

### Interruptible Locks (`lockInterruptibly`)

- Unlike `synchronized` blocks (where a blocked thread cannot be interrupted), a thread waiting inside `lockInterruptibly()` will respond to `Thread.interrupt()` and throw an `InterruptedException`, enabling clean deadlock recovery.

### Condition Variables (`Condition`)

- `ReentrantLock.newCondition()` produces `Condition` objects.
- Supports multiple wait-sets per lock (unlike `synchronized`, which only has one wait set via `Object.wait()` and `Object.notify()`), providing finer control over thread notification signaling.
