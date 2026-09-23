Reactive programming (Project Reactor, used by Spring WebFlux) applies a declarative, pipeline-based style to asynchronous, non-blocking I/O. But it solves a different problem than the Stream API: streams transform an already-available, finite, in-memory collection, synchronously, once; reactive streams orchestrate a _sequence of events over time_, often from I/O, asynchronously, with backpressure. The operator names overlap (`map`/`filter`/`flatMap`) precisely because the pipeline mental model is shared.

## 1. Why Reactive Programming — A Different Problem Than Streams

Reactive programming handles a sequence of events over time — an HTTP response streaming in, database rows arriving one at a time, WebSocket messages — asynchronously and **non-blockingly**: the thread that starts the operation isn't held waiting for it to finish.

The problem it solves: in a traditional blocking model (one thread per request, blocked on I/O), you eventually run out of threads under high concurrent load — even though those blocked threads are doing nothing but waiting. Non-blocking reactive code frees the thread during the wait, so a small, fixed pool of threads can handle a very large number of concurrent in-flight I/O operations. This is the same underlying problem virtual threads solve in async programming — reactive programming was the pre-virtual-threads answer to "don't block a thread on I/O," and the two approaches now coexist as different tools for the same goal.

## 2. The Reactive Streams Contract

Reactive Streams is a specification (not a library) that Project Reactor, RxJava, and `java.util.concurrent.Flow` all implement — it defines four interfaces and, critically, **backpressure**: a way for a slow consumer to tell a fast producer to slow down, so the producer can't overwhelm the consumer's memory buffering unbounded items.

```
Publisher<T>   — emits a sequence of items to a Subscriber
Subscriber<T>  — receives items, and signals how many more it can handle
Subscription   — the link between them; the Subscriber calls request(n) to pull n items
Processor<T,R> — both a Subscriber and a Publisher — a transformation stage in the pipeline
```

The key idea: the _subscriber_ controls the flow rate (`subscription.request(n)`), not the publisher blindly pushing everything as fast as it can — this is what prevents a fast database returning a million rows from overwhelming a slow consumer processing them one at a time. It's also the one thing a plain streams pipeline has no equivalent of, since a `Stream` always assumes its source is already fully available and pulls at its own pace with no producer to slow down. See backpressure for how this same "slow consumer, fast producer" problem is handled at every other layer of a system — bounded queues, thread pool rejection policies, Kafka/RabbitMQ, and the API layer.

## 3. Project Reactor: `Mono` and `Flux`

Reactor is Spring's reactive library — the foundation of Spring WebFlux. It offers two core types, both implementing the Reactive Streams `Publisher` interface:

| Type      | Represents                                                                                                            |
| --------- | --------------------------------------------------------------------------------------------------------------------- |
| `Mono<T>` | Zero or one asynchronous result — the reactive analogue of `Optional<T>`/`CompletableFuture<T>`                       |
| `Flux<T>` | Zero to many asynchronous results — the reactive analogue of a `Stream<T>`, but asynchronous and potentially infinite |

```java
Mono<Account> accountMono = accountRepository.findById(id); // returns immediately, nothing has executed yet

Flux<Transaction> transactionFlux = transactionRepository.findByAccountId(id);

Mono<AccountSummary> summary = accountMono
    .flatMap(account -> transactionRepository.findByAccountId(account.getId())
        .collectList()
        .map(transactions -> new AccountSummary(account, transactions)));

// Nothing above has actually run yet — like streams, Reactor pipelines are lazy/declarative.
// Execution only starts on subscribe():
summary.subscribe(
    result -> log.info("Summary: {}", result),
    error -> log.error("Failed", error)
);
```

Operator vocabulary deliberately mirrors the Stream API: `map`, `filter`, `flatMap`, `reduce` all exist on `Mono`/`Flux` with the same conceptual meaning, but operating asynchronously over time instead of synchronously over an in-memory collection.

```java
Flux<Order> highValueOrders = orderRepository.findAll()
    .filter(order -> order.getTotal().compareTo(BigDecimal.valueOf(1000)) > 0)
    .map(this::applyLoyaltyDiscount)
    .onErrorResume(ex -> {
        log.error("Failed to process order", ex);
        return Flux.empty(); // gracefully degrade instead of terminating the whole stream
    });
```

## 4. Spring WebFlux — Reactive Web Layer

```java
@RestController
public class AccountController {

    @GetMapping("/api/accounts/{id}")
    public Mono<AccountResponse> getAccount(@PathVariable Long id) {
        return accountRepository.findById(id)
            .map(AccountResponse::from)
            .switchIfEmpty(Mono.error(new AccountNotFoundException(id)));
        // the controller method returns immediately; the actual HTTP response
        // is written only once this Mono completes — the request-handling thread
        // is freed to serve other requests in the meantime
    }

    @GetMapping(value = "/api/accounts/{id}/transactions", produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<TransactionResponse> streamTransactions(@PathVariable Long id) {
        return transactionRepository.findByAccountId(id).map(TransactionResponse::from);
        // streams results to the client as they become available, rather than
        // buffering the entire result set in memory before responding
    }
}
```

WebFlux runs on Netty by default (an event-loop, non-blocking server) instead of Tomcat's traditional thread-per-request model — a small, fixed number of event-loop threads handle a large number of concurrent connections, as long as _nothing in the request-handling chain blocks_. This is the sharpest edge of reactive programming in practice: a single accidental blocking call (a JDBC call, a synchronous HTTP client, even a poorly-written `Thread.sleep()`) inside a WebFlux handler can stall an entire event-loop thread, degrading every other concurrent request sharing it — far worse than a blocked thread in a traditional thread-per-request model, where only that one request's thread is affected. Reactive persistence requires a genuinely non-blocking driver (R2DBC, not standard JDBC, which are both JDBC-based and therefore blocking) all the way down the stack.

## 5. Error Handling and Testing Reactive Pipelines

`onErrorResume`/`onErrorReturn` replace try/catch for reactive pipelines — an exception thrown inside an operator terminates that pipeline with an error signal rather than propagating like a normal Java exception up a call stack, so it must be handled as part of the pipeline itself, not with a surrounding try/catch.

```java
@Test
void shouldReturnFallbackOnDownstreamFailure() {
    Mono<AccountSummary> result = accountService.getSummary(accountId);

    StepVerifier.create(result)
        .expectNextMatches(summary -> summary.accountId().equals(accountId))
        .verifyComplete();
}
```

`StepVerifier` (Reactor's test utility) subscribes to a `Mono`/`Flux` and asserts on the emitted signals directly — prefer it over calling `.block()` in a test and asserting on the blocked result, which defeats the point of testing reactive code and can hide timing-dependent bugs `StepVerifier` would surface.

## 6. When to Choose Reactive vs. Blocking (Including Virtual Threads)

This is the practical decision most teams actually face today, and it's more nuanced than "reactive is faster."

| Situation                                                                                                       | Recommendation                                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New service, I/O-bound, team unfamiliar with reactive operators                                                 | Prefer virtual threads with ordinary blocking code — same scalability benefit for I/O-bound workloads, without the operator-chaining learning curve or the "one blocking call stalls the event loop" failure mode |
| Existing reactive stack already fully non-blocking end-to-end (R2DBC, reactive clients)                         | Stay reactive — rewriting a working, fully-reactive stack to virtual threads for its own sake isn't worth the churn                                                                                               |
| Need genuine streaming semantics (server-sent events, WebSocket, backpressure-aware consumption of a live feed) | Reactive (`Flux`) models this naturally; blocking code needs more manual plumbing to achieve the same thing                                                                                                       |
| Team already deeply fluent in reactive operators, existing codebase is idiomatic Reactor                        | Reactive is a reasonable continued choice — the "hard part" (the learning curve) is already paid for                                                                                                              |
| CPU-bound work, not I/O-bound                                                                                   | Neither reactive nor virtual threads help; virtual threads and reactive's non-blocking model both target I/O waiting, not CPU-bound computation                                                                   |
| Just transforming an already-in-memory collection, no I/O involved                                              | Plain streams — reactive types add no value when there's no asynchrony or backpressure need                                                                                                                       |

The practical shift since virtual threads (Java 21+): reactive programming's main historical justification — avoiding thread-per-request exhaustion under high I/O concurrency — is no longer the _only_ way to get that property. For a brand-new codebase, virtual threads + ordinary blocking code (JDBC, `RestTemplate`/`RestClient`) usually deliver similar scalability with far less cognitive overhead and far fewer accidental-blocking footguns than a fully reactive stack. Reactive remains the right choice for genuinely stream-shaped problems (live event feeds, backpressure-sensitive integrations) regardless of virtual threads' existence.

## 7. Best Practices

| Practice                                                                                           | Recommendation                                                                                                                                                                            |
| -------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Never block inside a WebFlux handler or Reactor operator                                           | A single blocking call (plain JDBC, `Thread.sleep`) can stall an entire event-loop thread, affecting unrelated concurrent requests.                                                       |
| Choose virtual threads over reactive for new I/O-bound services by default                         | Similar scalability with ordinary blocking code and far less operator-chaining complexity — reserve reactive for genuine streaming/backpressure needs.                                    |
| Handle errors as part of the pipeline (`onErrorResume`/`onErrorReturn`), not surrounding try/catch | A reactive pipeline's exceptions are error signals, not stack-unwinding exceptions a try/catch around the subscription can intercept.                                                     |
| Test with `StepVerifier`, not by blocking and asserting                                            | `.block()` in a test defeats the point of testing reactive code and can hide timing-dependent bugs `StepVerifier` surfaces directly.                                                      |
| Ensure the whole stack is non-blocking before going reactive                                       | A blocking JDBC call anywhere in a WebFlux chain reintroduces the exact thread-exhaustion problem reactive programming exists to avoid — use R2DBC or a reactive client all the way down. |
| Don't reach for `Mono`/`Flux` just to process an in-memory collection                              | If there's no I/O and no backpressure need, streams is the simpler, correct tool.                                                                                                         |
