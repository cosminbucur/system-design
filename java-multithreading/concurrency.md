Concurrency and parallelism are the two foundational _concepts_ behind running more than one thing "at once" — this note covers what each actually means, how they differ, and the fundamental problem (shared mutable state) that concurrency introduces regardless of how you implement it. The concrete Java implementation techniques for achieving concurrency and/or parallelism — multithreading (`synchronized`, `ExecutorService`, virtual threads, structured concurrency) and asynchronous programming (`CompletableFuture`, Spring `@Async`) — are covered in async-programming, so this note stays at the conceptual level rather than duplicating that API-level detail.

## 1. Concurrency vs. Parallelism — Two Distinct Concepts

These two are used almost interchangeably in casual conversation, but they describe different things, and the distinction matters for reasoning about what a piece of code actually needs.

| Concept     | Means                                                                                                                                                                   | Requires multiple cores?                                           |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Concurrency | Multiple tasks make progress in _overlapping_ time periods — structurally capable of being interleaved or run out of order, not necessarily at the same literal instant | No — a single core can be concurrent by time-slicing between tasks |
| Parallelism | Multiple tasks execute at the _literal same instant_, on separate cores                                                                                                 | Yes — parallelism is concurrency happening simultaneously          |

A single CPU core running a time-sliced scheduler across many tasks is concurrent (each task makes progress over time, interleaved) but not parallel (only one instruction from any task executes at any given instant). A multi-core machine actually running four threads at the same moment is both concurrent and parallel. The relationship only goes one way: **parallelism implies concurrency, but concurrency doesn't require parallelism** — you can structure a program concurrently (as many independent, interleavable tasks) and still run it on a single core, which is exactly what a single-threaded event loop does.

## 2. The Core Problem: Shared Mutable State

Regardless of which implementation technique eventually executes them, concurrent tasks that read and write the same mutable data without coordination produce race conditions — this is the central problem concurrency introduces, and it exists at the conceptual level before any specific API enters the picture.

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++; // NOT atomic: read, add, write = 3 steps. Two interleaved tasks can lose an update.
    }
}
```

If two concurrent tasks both execute `increment()` at overlapping times, both can read the same starting value before either writes back the incremented result — one update is silently lost, with no exception, no warning, just a wrong final count. Fixing this requires either eliminating the shared mutability (immutability) or coordinating access to it (mutual exclusion, atomic operations).

The _theory_ underneath why any particular fix actually works — happens-before, visibility vs. atomicity, safe publication; this note only establishes that the problem exists and why, not how each fix achieves correctness.

## 3. CPU-Bound vs. I/O-Bound Work — Which Concept Actually Applies

Whether a workload benefits from parallelism, concurrency without parallelism, or neither depends entirely on what the task is actually spending its time doing.

| Workload type | Bottleneck                                                      | What helps                                                                                                                                                                            |
| ------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CPU-bound     | Computation itself — the CPU is continuously busy               | Parallelism — more cores genuinely doing more work at once; concurrency alone (time-slicing one core) doesn't add throughput                                                          |
| I/O-bound     | Waiting — for a network response, a disk read, a database query | Concurrency — overlap the _waiting_ of many tasks, even on very few (or virtual) threads; adding more cores doesn't speed up something that's mostly idle, waiting on a remote system |

A common mistake is applying parallelism (spinning up many OS threads or cores) to a fundamentally I/O-bound workload — the bottleneck is the wait, not the CPU, so extra cores sit idle just like the original thread did. The right response to an I/O-bound workload is concurrency at low resource cost (see virtual threads and reactive programming's non-blocking model in async-programming and reactive-programming), not more parallel compute capacity.

## 4. Amdahl's Law: The Limit of Parallelism

Not every part of a task can be parallelized — some portion is inherently sequential (setup, coordination, merging results), and that sequential portion caps how much speedup adding more cores can ever provide, no matter how many you add.

```
Speedup(N cores) = 1 / (S + (1 - S) / N)

where S = the fraction of the task that MUST run sequentially
```

If 10% of a task is inherently sequential (`S = 0.10`), the maximum possible speedup — even with infinite cores — is `1 / 0.10 = 10x`, never more, regardless of how many cores you throw at the other 90%. This is precisely why doubling core count rarely doubles real-world throughput: the sequential fraction (I/O waits, lock contention, coordination overhead) becomes the dominant cost long before parallelism runs out of cores to use. It's also a direct, quantitative reason to profile _before_ reaching for more parallelism — if the actual bottleneck is a small sequential section, adding parallel workers elsewhere won't move the needle much.

## 5. Choosing Concurrency, Parallelism, or Both for a Workload

- **Purely CPU-bound, parallelizable work** (image processing, batch computation over independent data) → reach for parallelism — see `.parallelStream()` in streams, sized to actual core count, not an arbitrarily large thread count.
- **Purely I/O-bound work** (calling downstream services, querying a database) → reach for concurrency without needing parallelism — virtual threads or reactive programming let you overlap thousands of waits cheaply, without needing thousands of CPU cores to do it.
- **A mix of both** (a request that does some computation and some I/O) → the I/O-bound portions benefit from concurrency; only the genuinely CPU-heavy portions benefit from also being parallelized — conflating the two and parallelizing everything indiscriminately wastes resources on the I/O-waiting parts that were never CPU-bound to begin with.

## 6. Best Practices

| Practice                                                                         | Recommendation                                                                                                                               |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Identify whether a workload is CPU-bound or I/O-bound before choosing a strategy | Parallelism helps the former; plain concurrency (cheap, high-count waiting) helps the latter.                                                |
| Don't add parallelism to fix an I/O-bound bottleneck                             | More cores don't speed up waiting on a remote system — the fix is concurrency at low resource cost, not more compute.                        |
| Treat shared mutable state as the default source of concurrency bugs             | Prefer immutability or clear ownership over ad-hoc coordination.                                                                             |
| Profile before assuming more parallelism will help                               | Amdahl's Law means a sequential bottleneck caps speedup regardless of core count.                                                            |
| Treat "concurrent" and "parallel" as distinct claims about a system              | A correct claim about one doesn't imply the other; conflating them leads to reaching for the wrong implementation tool in async-programming. |
