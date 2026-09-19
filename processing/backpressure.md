Backpressure is what happens when a producer generates work faster than a consumer can process it, and the mechanism (or lack of one) for handling that gap. Without an explicit strategy, the default outcome is an ever-growing in-memory buffer — which eventually becomes an `OutOfMemoryError` rather than a clean, deliberate decision. The underlying question — "what do we do when the consumer can't keep up?" — is the same question at every layer of a system, from a single in-process queue up to a whole downstream service.

## 1. The Options When a Consumer Can't Keep Up

There are really only a handful of fundamental strategies — every backpressure mechanism at every layer is some combination of these.

| Strategy           | Behavior                                                                                   | Cost                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Buffer             | Queue the excess work, hoping the consumer catches up later                                | Unbounded buffering risks OOM; bounded buffering just delays which other strategy kicks in once full         |
| Block the producer | Make the producer wait until the consumer has capacity                                     | Producer throughput drops to match the consumer's — correct, but propagates slowness upstream                |
| Drop               | Discard some work (newest or oldest) rather than queue or block                            | Lossy — only acceptable when losing data is genuinely tolerable                                              |
| Reject / fail fast | Refuse new work outright once at capacity, signaling the caller to back off or retry later | Requires the caller to handle the rejection — but keeps the system's own state bounded and predictable       |
| Sample / throttle  | Only process a fraction of incoming work, deliberately                                     | Systematic data loss by design, not incidental — appropriate for high-volume telemetry, not for transactions |

The right choice is never universal — it depends entirely on whether the specific data flowing through can tolerate being blocked, delayed, or lost, the same judgment call already made for cache staleness and for tracing sample rates in observability.

## 2. In-Process Backpressure: Bounded Queues

The simplest, most direct form of backpressure in Java is a bounded `BlockingQueue` — its `put()` method blocks the calling (producer) thread once the queue is full, until the consumer drains space.

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000); // bounded — this capacity IS the backpressure mechanism

// Producer
queue.put(task); // blocks here once the queue holds 1000 items, until a consumer takes one

// Consumer
Task task = queue.take();
process(task);
```

An **unbounded** queue (`new LinkedBlockingQueue<>()` with no capacity argument) has no backpressure at all — a producer outpacing its consumer just grows the queue indefinitely, and the eventual failure mode is memory exhaustion, not a controlled slowdown. Choosing a bounded queue is the single highest-leverage backpressure decision in ordinary Java code, and it's opt-in — the unbounded default is exactly why this is easy to get wrong by simply not thinking about it.

## 3. Thread Pool Queues and Rejection Policies

An `ExecutorService`'s internal work queue faces the exact same question, and Java's `ThreadPoolExecutor` makes the choice explicit via a `RejectedExecutionHandler` once both the pool and its queue are full.

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 4, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(100), // bounded queue — the capacity decision, applied here
    new ThreadPoolExecutor.CallerRunsPolicy() // what happens once BOTH the pool and queue are full
);
```

| Rejection Policy        | Behavior                                                                                                                                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `AbortPolicy` (default) | Throws `RejectedExecutionException` — the caller must handle the rejection explicitly                                                                                                                  |
| `CallerRunsPolicy`      | The _submitting_ thread runs the task itself, synchronously — a clever, self-throttling form of backpressure: the producer is directly slowed down by doing the work itself instead of a worker thread |
| `DiscardPolicy`         | Silently drops the task — rarely correct, since a caller has no idea their submission vanished                                                                                                         |
| `DiscardOldestPolicy`   | Drops the oldest queued task to make room for the new one — a deliberate "prefer recent work" tradeoff                                                                                                 |

`CallerRunsPolicy` is worth calling out specifically: it converts pool saturation directly into producer slowdown without any explicit queue-full handling code — the producer thread, forced to execute the task itself, simply can't submit new work as fast while it's busy running the old work. This is backpressure achieved as a natural side effect of the mechanism, rather than an if-check the caller has to remember to write.

## 4. Reactive Streams: Explicit, Pull-Based Backpressure

The short version relevant here: a `Subscriber` calls `request(n)` to pull exactly `n` items, so the `Publisher` is structurally prevented from pushing faster than the subscriber asked for. This is the only mechanism in this note that's backpressure by design from the ground up, rather than backpressure bolted onto something that wasn't built with it in mind.

```java
Flux<Order> orderStream = orderRepository.findAll();

orderStream
    .onBackpressureBuffer(1000)                          // buffer strategy — bounded, though
    // .onBackpressureDrop(order -> log.warn("dropped {}", order))  // drop strategy
    // .onBackpressureLatest()                                       // keep only the most recent item, drop older ones
    .subscribe(order -> processSlowly(order));
```

Reactor's `onBackpressureBuffer`/`onBackpressureDrop`/`onBackpressureLatest` operators are directly the same menu of strategies, just exposed as explicit, composable choices at the reactive-pipeline level — which strategy to pick depends on the same "can this data tolerate loss or delay" question as everywhere else in this note.

## 5. Message Queues: Pull vs. Push Changes Everything

The backpressure-specific distinction between them is worth calling out on its own.

- **Kafka is pull-based by design**: a consumer calls `poll()` and only receives as many records as it explicitly asks for, at its own pace — this means Kafka has no "the broker is force-feeding the consumer too fast" problem at all; the consumer inherently controls its own rate. The actual risk with Kafka isn't the consumer being overwhelmed per-poll, it's **consumer lag** accumulating in the topic if the consumer's sustained processing rate is slower than the production rate — a form of the buffer strategy, with the topic's retention window as the buffer's effective size limit.
- **RabbitMQ is push-based by default**, so it needs an explicit backpressure mechanism: the **prefetch count** (QoS setting) limits how many unacknowledged messages the broker will push to a consumer at once, preventing the broker from flooding a slow consumer faster than it can `ack`.

```java
// RabbitMQ: cap in-flight unacknowledged messages per consumer — this IS the backpressure control
channel.basicQos(10); // never push more than 10 unacked messages to this consumer at a time
```

## 6. HTTP and the API Layer

At the API boundary, backpressure takes the form of an explicit rejection signal back to the caller rather than an internal buffering mechanism — `429 Too Many Requests` with a `Retry-After` header is the reject strategy, applied at the HTTP layer, telling a well-behaved client exactly how long to back off rather than retrying immediately and making the overload worse. Underneath this, TCP itself has its own low-level flow control (a receive window limiting how much unacknowledged data a sender can have in flight) — invisible to application code, but the same fundamental "don't let the sender outpace the receiver" idea, one layer down the stack.

## 7. Structural Backpressure: Scaling the Consumer

Sometimes the right response to sustained backpressure isn't a queue/reject/drop decision at all — it's adding more consumer capacity. Kubernetes' Horizontal Pod Autoscaler reacting to a growing queue depth or request rate is a structural way of resolving backpressure by scaling out the slow side, rather than only ever managing the symptom at the point of contention. This doesn't replace the mechanisms above — a bounded queue and a rejection policy are still needed for the time between load increasing and new capacity actually coming online — but it addresses the underlying capacity mismatch rather than only ever coping with it.

## 8. Choosing a Strategy Per Data Class

The same system typically needs different backpressure strategies for different kinds of data flowing through it — there's no single right answer at the whole-system level.

| Data                                     | Tolerable strategy                    | Why                                                                                         |
| ---------------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------- |
| Financial transactions, orders, payments | Block or reject — never silently drop | Losing a transaction is a correctness/compliance failure, not an acceptable degradation     |
| Application metrics/traces               | Sample or drop under load             | A gap in telemetry during a spike is tolerable                                              |
| Real-time analytics/dashboards           | Drop-oldest or keep-latest            | A slightly stale or incomplete view is acceptable; the next update will supersede it anyway |
| Audit events                             | Block or reject, never drop           | Same reasoning as transactions; a missing audit record defeats the entire point of auditing |

Getting this wrong in the more forgiving direction (blocking/rejecting telemetry as strictly as transactions) just wastes capacity on data that didn't need the guarantee; getting it wrong in the other direction (silently dropping transaction data under load) is a correctness bug wearing the disguise of a performance optimization.

## 9. Best Practices

| Practice                                                                          | Recommendation                                                                                                                                  |
| --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Never use an unbounded queue for anything under real load                         | An unbounded queue's failure mode is `OutOfMemoryError`, not a controlled slowdown — always size a queue deliberately.                          |
| Match the backpressure strategy to what the data can actually tolerate            | Block/reject for anything that must not be lost (transactions, audit events); drop/sample for telemetry that can tolerate gaps.                 |
| Prefer `CallerRunsPolicy` over silent drops for thread pool saturation            | Self-throttles the producer directly, rather than either crashing on rejection or silently discarding submitted work.                           |
| Use Reactor's backpressure operators explicitly, not the default                  | `onBackpressureBuffer`/`Drop`/`Latest` make the strategy a deliberate choice rather than an implicit, unexamined default.                       |
| Monitor queue depth / consumer lag as a leading indicator, not just latency       | A growing backlog (Kafka consumer lag, a thread pool's queue size) predicts trouble before it becomes an outage.                                |
| Communicate rejection to the caller with enough information to back off correctly | `429` + `Retry-After` at the API layer — lets a well-behaved caller self-throttle instead of retrying immediately and compounding the overload. |
| Treat autoscaling as a complement to queueing/rejection, not a replacement        | New capacity takes time to come online — a bounded queue and a rejection policy still need to cover the gap in between.                         |
