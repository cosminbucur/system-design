# Java Asynchronous Programming & CompletableFuture Guide

This document summarizes the core concepts, implementation details, and practical examples covering the **Future and Promise pattern**, combining multiple futures, and robust exception handling in Java.

---

## 1. The Future and Promise Pattern in Java

The **Future and Promise pattern** separates the placeholder for an asynchronous result (the **Future**) from the logic that produces and delivers that result (the **Promise**).

In Java’s `java.util.concurrent` package:
- **`Future` / `CompletionStage`**: The read-only / consumer side.
- **`CompletableFuture`**: Serves as both the **Future** and the **Promise** (via explicit completion methods like `.complete()` and `.completeExceptionally()`).

### Real-Life Analogy: Coffee Shop Order App
1. **Client (Consumer):** Places an order and receives an order ticket (the **Future**).
2. **Barista (Producer/Promise Owner):** Takes the order sheet (the **Promise**). When done brewing, they complete the order (`complete(coffee)`). If an error occurs, they report a failure (`completeExceptionally(err)`).

### Basic Code Example

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class CoffeeShopDemo {

    // Acts as the Promise producer
    public static CompletableFuture<String> brewCoffeeAsync(String order) {
        CompletableFuture<String> promise = new CompletableFuture<>();
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        
        scheduler.schedule(() -> {
            if ("Espresso".equalsIgnoreCase(order)) {
                promise.complete("Hot Fresh Espresso ☕");
            } else {
                promise.completeExceptionally(
                    new IllegalArgumentException("Out of stock: " + order)
                );
            }
            scheduler.shutdown();
        }, 2, TimeUnit.SECONDS);

        return promise; // Returns the Future view to the caller
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("[Client] Ordering Espresso...");
        
        CompletableFuture<String> coffeeFuture = brewCoffeeAsync("Espresso");

        coffeeFuture
            .thenApply(coffee -> coffee + " with Biscotti 🍪")
            .thenAccept(finalOrder -> System.out.println("[Client] Received: " + finalOrder))
            .exceptionally(ex -> {
                System.err.println("[Client] Order failed: " + ex.getMessage());
                return null;
            });

        System.out.println("[Client] Doing other work while coffee brews...");
        Thread.sleep(3000);
    }
}
```

---

## 2. Combining Multiple CompletableFutures

When performing parallel tasks (e.g., microservice API calls), Java provides two primary methods for combining results:

1. **`thenCombine`**: Combines **exactly two** independent futures.
2. **`CompletableFuture.allOf`**: Waits for **three or more** (or an arbitrary array of) independent futures to finish.

### Example: E-Commerce Checkout Pipeline

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class OrderAggregationDemo {

    static CompletableFuture<String> fetchUser() {
        return CompletableFuture.supplyAsync(() -> "User: Alex");
    }

    static CompletableFuture<List<String>> fetchCart() {
        return CompletableFuture.supplyAsync(() -> List.of("Laptop", "Mouse"));
    }

    static CompletableFuture<String> fetchPayment() {
        return CompletableFuture.supplyAsync(() -> "Payment: Visa ending in 4242");
    }

    public static void main(String[] args) {
        // --- 1. thenCombine (Two Futures) ---
        CompletableFuture<String> summary = fetchUser().thenCombine(
            fetchCart(), 
            (user, cart) -> user + " | Cart: " + String.join(", ", cart)
        );
        System.out.println(summary.join());

        // --- 2. CompletableFuture.allOf (Multiple Futures) ---
        CompletableFuture<String> userFut = fetchUser();
        CompletableFuture<List<String>> cartFut = fetchCart();
        CompletableFuture<String> payFut = fetchPayment();

        CompletableFuture<Void> allOf = CompletableFuture.allOf(userFut, cartFut, payFut);

        CompletableFuture<String> fullSummary = allOf.thenApply(v -> String.format(
            "Summary:\n - %s\n - Items: %s\n - %s",
            userFut.join(), String.join(", ", cartFut.join()), payFut.join()
        ));

        System.out.println(fullSummary.join());
    }
}
```

### Pattern Comparison

| Feature | `thenCombine` | `CompletableFuture.allOf` |
| :--- | :--- | :--- |
| **Input Count** | Exactly 2 futures | N futures (varargs / array) |
| **Return Type** | `CompletableFuture<V>` (Combined result) | `CompletableFuture<Void>` |
| **Data Extraction** | Passed directly into lambda parameters | Requires `.join()` on completed futures |
| **Primary Use** | Joining pairs of dependent tasks | Batch/parallel synchronization barrier |

---

## 3. Exception Handling

Handling exceptions in `CompletableFuture` allows you to either recover with fallback values or propagate errors down the chain.

### Key Operations

* **`.exceptionally(ex -> fallback)`**: Catches errors and supplies a replacement value.
* **`.handle((result, ex) -> ...)`**: Executes on completion regardless of success or failure.
* **`.exceptionallyCompose(ex -> asyncFallback)`**: Asynchronously handles an error by returning another stage.

### Handling Errors with `thenCombine`

```java
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> "User: Alex");
CompletableFuture<String> cartFuture = CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Cart Service Unavailable");
});

// Individual recovery:
CompletableFuture<String> safeCart = cartFuture.exceptionally(ex -> "Cart: [Empty]");
userFuture.thenCombine(safeCart, (user, cart) -> user + " | " + cart)
          .thenAccept(System.out::println);

// Pipeline recovery:
userFuture.thenCombine(cartFuture, (user, cart) -> user + " | " + cart)
          .exceptionally(ex -> "Order failed: " + ex.getCause().getMessage())
          .thenAccept(System.out::println);
```

### Handling Errors with `CompletableFuture.allOf`

To prevent a single task failure from discarding all other results when using `allOf`:

```java
CompletableFuture<String> t1 = CompletableFuture.supplyAsync(() -> "Data 1");
CompletableFuture<String> t2 = CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("DB Timeout");
});
CompletableFuture<String> t3 = CompletableFuture.supplyAsync(() -> "Data 3");

// Attach individual fallbacks (e.g., return null on error)
CompletableFuture<String> safe1 = t1.exceptionally(ex -> null);
CompletableFuture<String> safe2 = t2.exceptionally(ex -> null);
CompletableFuture<String> safe3 = t3.exceptionally(ex -> null);

List<CompletableFuture<String>> futures = List.of(safe1, safe2, safe3);

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream()
                           .map(CompletableFuture::join)
                           .filter(Objects::nonNull) // Retain only successful results
                           .toList())
    .thenAccept(results -> System.out.println("Successful results: " + results));
```