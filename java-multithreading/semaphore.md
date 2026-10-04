# The Semaphore Pattern in Java

## Understanding the Semaphore Pattern

A **Semaphore** is a concurrency synchronization construct managed by the `java.util.concurrent.Semaphore` class. Unlike a standard lock (`ReentrantLock` or `synchronized`), which allows only *one* thread to access a critical section at a time, a semaphore maintains a set of **permits**. 

* **Permits Control Access:** Threads request permits using `acquire()` and return them using `release()`. If no permits are available, the calling thread blocks until another thread releases one.
* **Fairness:** It can be initialized with a fairness policy (FIFO order for waiting threads) by passing `true` to its constructor.

---

## Real-Life Application

A classic real-life analogy is a **parking garage with a fixed number of stalls**. When the garage is full, incoming cars must wait at the gate until a parked car leaves and frees up a space.

In software engineering, semaphores are commonly used for:
* **Database Connection Pools:** Limiting the maximum number of concurrent database connections to prevent overwhelming the server.
* **Rate Limiters / Throttling:** Restricting the number of concurrent API requests a service or client can execute.
* **Resource Throttling:** Controlling access to heavy hardware operations, file handles, or network sockets.

---

## Java Code Example

Here is an example demonstrating a **Connection Pool** using a `Semaphore` to limit concurrent access to a fixed pool of simulated resources:

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

class DatabaseConnectionPool {
    private final Semaphore semaphore;
    private final int poolSize;

    public DatabaseConnectionPool(int poolSize) {
        this.poolSize = poolSize;
        // Initialize semaphore with 'poolSize' permits, using fair ordering
        this.semaphore = new Semaphore(poolSize, true);
    }

    public void accessDatabase(String threadName) {
        try {
            System.out.println(threadName + " is waiting for a database connection...");
            semaphore.acquire(); // Request a permit
            
            System.out.println("--> " + threadName + " acquired a connection. Active queries running.");
            // Simulate work with the database
            Thread.sleep(1500); 
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            System.out.println("<-- " + threadName + " releasing the connection.");
            semaphore.release(); // Return the permit
        }
    }
}

public class SemaphoreDemo {
    public static void main(String[] args) throws InterruptedException {
        // Allow at most 3 concurrent connections
        DatabaseConnectionPool pool = new DatabaseConnectionPool(3);
        ExecutorService executor = Executors.newFixedThreadPool(6);

        // Simulate 6 clients trying to access the database simultaneously
        for (int i = 1; i <= 6; i++) {
            final String clientName = "Client-" + i;
            executor.submit(() -> pool.accessDatabase(clientName));
        }

        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
    }
}
```

> **Key Takeaway:** Always wrap your critical section and resource release (`semaphore.release()`) inside a `try-finally` block. This guarantees that permits are safely returned even if an exception occurs during execution, preventing deadlocks or permit leaks.