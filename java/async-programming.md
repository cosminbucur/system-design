Asynchronous programming structures work as non-blocking callbacks/futures that may or may not need an extra thread while waiting — a different way of organizing code than multithreading itself, even though the two are often combined. Where multithreading is about actual threads and how they're scheduled and coordinated, asynchronous programming is about describing "what happens when this eventually finishes" without a thread necessarily blocking to find out.

## 1. Asynchronous Programming vs. Multithreading — a Different Axis

| Aspect | Multithreading | Asynchronous programming |
| --- | --- | --- |
| Mechanism | Multiple threads (platform or virtual) executing independently, scheduled by the JVM/OS | Non-blocking calls that register a continuation instead of blocking a thread while waiting |
| Where the "waiting" happens | A thread is often blocked while waiting (unless it's a virtual thread) | No thread is necessarily blocked at all — a callback fires later, possibly on a different thread or none in particular until it does |
| Composition style | Shared state coordinated via locks/atomics | Chained callbacks/futures (`thenApply`, `thenCompose`) |

In modern Java, these aren't mutually exclusive — a `CompletableFuture` is often *implemented* using a thread pool underneath (multithreading), while presenting an asynchronous, non-blocking-to-the-caller interface on top. Virtual threads blur the line further: they let you write code that *looks* like ordinary blocking multithreaded code while behaving, from a resource-cost perspective, much more like the cheap-to-hold-many-in-flight nature of asynchronous I/O.

## 2. `Future` vs. `CompletableFuture`

Plain `Future<T>` (from `ExecutorService.submit()`) is a passive handle: you can `get()` (blocking) or `isDone()` (polling), but you can't chain a follow-up action, combine it with another future, or react to its completion without blocking a thread to wait. `CompletableFuture<T>` adds composition on top of the same basic idea.

```java
// Future: no composition — you can only block and wait
Future<Account> future = executor.submit(() -> accountService.findById(id));
Account account = future.get(); // blocks the calling thread

// CompletableFuture: composable — describe what happens next WITHOUT blocking to find out
CompletableFuture<Account> future = CompletableFuture.supplyAsync(() -> accountService.findById(id));
future.thenApply(Account::getBalance)
      .thenAccept(balance -> log.info("Balance: {}", balance));
// the calling thread is never blocked waiting for any of this to complete
```

## 3. Building and Chaining a Pipeline

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))         // start an async computation
    .thenApply(User::getName)                      // transform the result (like Stream's map)
    .thenCompose(name -> callExternalService(name)) // chain to ANOTHER async operation (like flatMap — avoids CompletableFuture<CompletableFuture<T>>)
    .exceptionally(ex -> "fallback-value");         // recover from a failure anywhere upstream in the chain

String result = pipeline.join(); // blocks ONLY here, at the one point you actually need the final value
```

`thenApply` vs. `thenCompose` is exactly the `map` vs. `flatMap` distinction from streams: use `thenApply` when the next step is a plain, synchronous transformation; use `thenCompose` when the next step itself returns another `CompletableFuture` — using `thenApply` there by mistake produces a `CompletableFuture<CompletableFuture<T>>`, a nested future nobody actually wanted.

## 4. Combining Independent Async Operations

For operations that don't depend on each other's results, run them concurrently and combine once both finish, rather than chaining them sequentially and paying both latencies back-to-back.

```java
CompletableFuture<Account> accountFuture = CompletableFuture.supplyAsync(() -> fetchAccount(id));
CompletableFuture<List<Transaction>> txFuture = CompletableFuture.supplyAsync(() -> fetchTransactions(id));

CompletableFuture<AccountSummary> summary = accountFuture.thenCombine(txFuture,
    (account, transactions) -> new AccountSummary(account, transactions));
// both fetches run concurrently; summary completes once BOTH are done, not sequentially
```

```java
// Fan-out to many, then wait for all of them
List<CompletableFuture<Order>> orderFutures = orderIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchOrder(id)))
    .toList();

CompletableFuture<Void> allDone = CompletableFuture.allOf(orderFutures.toArray(new CompletableFuture[0]));
List<Order> orders = allDone.thenApply(v -> orderFutures.stream().map(CompletableFuture::join).toList()).join();
```

`allOf` completes once every future finishes (but doesn't itself return their combined results — you still collect them via `.join()` on each, as above, since by that point every future is already done and `join()` returns immediately); `anyOf` completes as soon as the *first* one does, useful for a "whichever responds first" race between redundant calls.

## 5. Which Thread Actually Runs the Continuation

This is the single most common source of confusion: `thenApply`/`thenAccept`/`thenCompose` (no `Async` suffix) run their continuation on *whichever thread completed the previous stage* — which might be the original calling thread, or might be a pool thread, depending on timing. The `...Async` variants make this explicit and controllable.

```java
CompletableFuture.supplyAsync(() -> slowLookup())
    .thenApplyAsync(result -> transform(result), dedicatedExecutor); // explicit: runs on YOUR executor, not whatever ran the previous stage
```

Without an explicit `Executor` argument, `supplyAsync`/the `...Async` methods default to the JVM's shared common `ForkJoinPool` — the exact same pool `.parallelStream()` uses. Running blocking I/O inside a `CompletableFuture` stage without specifying a dedicated executor risks starving that shared pool for every other unrelated parallel stream or `CompletableFuture` chain running elsewhere in the same JVM — always pass an explicit, appropriately-sized `Executor` for anything beyond trivial, fast, CPU-only continuations.

## 6. Exception Handling — and the Silent-Swallow Trap

An exception thrown inside any stage propagates through the chain as a failed future, not as a normal Java exception up a call stack — it must be handled as part of the chain itself.

```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> transform(result))
    .exceptionally(ex -> {
        log.error("Operation failed", ex);
        return fallbackValue; // recovers — downstream stages see this value, not a failure
    });

// handle() sees BOTH the success value and the exception, letting you branch on either
future.handle((result, ex) -> ex != null ? fallbackValue : result);

// whenComplete() observes the outcome (for logging/metrics) WITHOUT altering it
future.whenComplete((result, ex) -> {
    if (ex != null) meterRegistry.counter("operation.failed").increment();
});
```

The trap: a `CompletableFuture` chain with no `exceptionally`/`handle` and that's never `.join()`ed or `.get()`ed anywhere simply swallows its exception silently — nothing ever surfaces it, because nothing ever asked the future for its result. Always either terminate a chain with explicit exception handling, or ensure something eventually calls `join()`/`get()` where a thrown exception (wrapped in `CompletionException`) can actually be observed.

## 7. Timeouts and Cancellation

An async operation with no timeout can hang indefinitely if the underlying work never completes — Java 9+ adds this directly to `CompletableFuture`.

```java
CompletableFuture<Account> future = CompletableFuture.supplyAsync(() -> slowLookup())
    .orTimeout(3, TimeUnit.SECONDS); // completes exceptionally with TimeoutException if not done in time

CompletableFuture<Account> withFallback = CompletableFuture.supplyAsync(() -> slowLookup())
    .completeOnTimeout(Account.defaultAccount(), 3, TimeUnit.SECONDS); // completes with a fallback value instead of failing
```

A common misconception: calling `.cancel(true)` on a `CompletableFuture` does **not** actually interrupt the underlying running task — it only marks the future itself as cancelled, so anything still awaiting *this particular future* sees a `CancellationException`, but the task keeps running to completion in the background regardless. True cancellation of the underlying work requires the task itself to periodically check `Thread.interrupted()` and exit voluntarily — a future's cancellation flag alone doesn't reach into and stop the work.

## 8. Spring's `@Async`

For application code, `@Async` turns a plain method call into an asynchronous one without manually wrapping it in `CompletableFuture.supplyAsync` at every call site.

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100); // bounded — see backpressure on why this matters
        executor.setThreadNamePrefix("notification-");
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {

    @Async("notificationExecutor") // runs on the named executor, NOT the caller's thread
    public CompletableFuture<Void> sendConfirmationEmail(String orderId) {
        emailClient.send(orderId);
        return CompletableFuture.completedFuture(null);
    }
}
```

Never rely on `@Async`'s default executor (`SimpleAsyncTaskExecutor`) in anything beyond a quick prototype — it creates a **new, unbounded thread per invocation** rather than pooling them, which is exactly the unbounded-resource-growth problem warns against, just manifesting as threads instead of a queue. Always configure and name a dedicated, bounded `ThreadPoolTaskExecutor` per logical group of async work, the same sizing discipline as any other executor.

Also worth knowing: `@Async` doesn't work when called from within the same class (Spring's proxy-based AOP can't intercept a self-invocation, the same limitation covered generally for `@Transactional` and other Spring annotations) — the annotated method must be called from a different Spring-managed bean.

## 9. `CompletableFuture` vs. Reactive vs. Virtual Threads — Where Each Fits

Reactive programming already covers the reactive-vs-virtual-threads decision in depth; `CompletableFuture` is the third point of comparison, sitting between the two.

| Aspect | `CompletableFuture` | Reactive (`Mono`/`Flux`) | Virtual threads + blocking code |
| --- | --- | --- | --- |
| Programming model | Explicit callback-chaining (`thenApply`/`thenCompose`) | Declarative pipeline with backpressure | Ordinary sequential, blocking-looking code |
| Backpressure | None built in | Built in (`request(n)`) | Not applicable — no stream abstraction |
| Learning curve | Moderate — a specific API to learn, but still "normal" Java control flow | Steepest — a different mental model end-to-end | Lowest — write code as if it were synchronous |
| Best fit today | Combining a small, known set of independent async calls; existing codebases already using it | Genuine streaming/backpressure-sensitive needs | New I/O-bound services |

For a brand-new service, the practical guidance from reactive-programming still applies: default to virtual threads with plain blocking code (covered in multithreading), and reserve `CompletableFuture` specifically for the case it's genuinely good at — combining a handful of independent, already-async operations without needing full reactive backpressure semantics.

## 10. Best Practices

| Practice | Recommendation |
| --- | --- |
| Use `thenCompose`, not `thenApply`, when the next step returns another `CompletableFuture` | Using `thenApply` there produces an unwanted nested `CompletableFuture<CompletableFuture<T>>`. |
| Always terminate a chain with exception handling, or ensure something calls `join()`/`get()` | An unobserved failed future silently swallows its exception. |
| Pass an explicit, dedicated `Executor` to `...Async` methods doing blocking work | Avoids starving the shared common `ForkJoinPool` that `.parallelStream()` also depends on. |
| Never rely on `@Async`'s default executor in production | `SimpleAsyncTaskExecutor` creates an unbounded thread per call — always configure a named, bounded `ThreadPoolTaskExecutor`. |
| Add a timeout to any async operation that could hang | `orTimeout`/`completeOnTimeout` — an operation with no timeout can block indefinitely on a stuck dependency. |
| Don't assume `.cancel()` stops the underlying work | True cancellation requires the task itself to check for interruption. |
| Prefer virtual threads with blocking code for new I/O-bound services, reserving `CompletableFuture` for combining known-async calls | `CompletableFuture` isn't the default answer just because it's built into the JDK. |
