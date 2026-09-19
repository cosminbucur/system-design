Profiling answers a different question than observability: metrics/traces tell you _that_ something is slow and roughly _where_ (which service, which endpoint); profiling tells you _why_, down to which method, which line, or which object allocation is actually consuming CPU, memory, or blocking threads. Reach for a profiler when observability has already narrowed the problem to a specific service/process — profiling a whole distributed system at once isn't the right tool.

## 1. When to Profile vs When to Just Look at Metrics

| Situation                                                 | Tool                        |
| --------------------------------------------------------- | --------------------------- |
| "Which service is slow?"                                  | Distributed tracing         |
| "This one service is slow — which method is burning CPU?" | CPU profiler                |
| "Memory usage keeps growing — what's leaking?"            | Heap dump + memory profiler |
| "Threads seem stuck / throughput collapsed under load"    | Thread dump                 |
| "Is this GC pause causing my latency spikes?"             | GC logs / JFR               |
| "Is implementation A actually faster than B?"             | JMH microbenchmark          |

Profiling in production has real overhead — sampling profilers (JFR, async-profiler) are safe for continuous production use; instrumenting/tracing profilers that intercept every method call are not — reserve those for local/staging investigation.

## 2. Java Flight Recorder (JFR) — The Built-In Default

JFR ships with the JDK, has very low overhead (typically 1-2%), and is safe to run continuously in production — it should usually be your first tool, before reaching for a third-party agent.

```bash
# Attach to a running process and record for 60 seconds
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Or enable at JVM startup
java -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar
```

Open the resulting `.jfr` file in JDK Mission Control (JMC) — it shows CPU hotspots, allocation profiles, lock contention, GC pauses, and thread activity, all from one recording. Because it's low-overhead, it's reasonable to leave a continuous JFR recording running in production for exactly this kind of retrospective investigation ("what was happening at 3am when latency spiked").

## 3. CPU Profiling — Finding Hot Methods

A CPU profiler samples thread stacks periodically and builds a picture of where time is actually spent — surfacing the specific method/line consuming CPU, not just "the service is slow."

```bash
# async-profiler — low-overhead sampling profiler, widely used alongside/instead of JFR
./profiler.sh -d 30 -f profile.html <pid>   # 30-second CPU profile, flame graph output
```

Reading a flame graph:

- Each box is a stack frame; width represents time spent (including callees), not call order.
- Wide boxes near the top = hot leaf methods actually consuming CPU.
- Look for unexpectedly wide boxes in code you didn't expect to be expensive (e.g., a `toString()` or logging call showing up wide is a common surprise culprit).

Common CPU hotspot culprits: excessive string concatenation/regex in hot paths, unnecessary serialization/deserialization, inefficient collection operations (e.g., `O(n²)` nested loops), or reflection-heavy code called far more often than intended.

## 4. Memory Profiling and Heap Dumps

When memory usage grows over time (a suspected leak) or `OutOfMemoryError` occurs, a heap dump captures every object on the heap at that moment for analysis.

```bash
# Trigger a heap dump from a running process
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Automatically dump on OOM — always enable this in production
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/dumps/ -jar app.jar
```

Open the `.hprof` file in Eclipse Memory Analyzer (MAT) or JDK Mission Control:

- **Dominator tree**: shows which objects, if removed, would free the most memory — the fastest way to find what's actually hogging the heap.
- **Leak suspects report**: MAT's built-in heuristic that flags collections/objects growing unboundedly.
- **GC roots path**: for a suspected leaked object, trace _why_ it's still reachable (what's holding a reference preventing collection) — this is usually the actual root cause, not just "this object is big."

Classic Java leak patterns: static collections that only ever grow (`static Map` used as a cache with no eviction), listeners/callbacks registered but never unregistered, `ThreadLocal` values not cleared (especially dangerous in thread pools), and unbounded queues feeding a slower consumer.

## 5. Thread Dumps — Diagnosing Stuck/Contended Threads

A thread dump is a snapshot of every thread's stack trace at one instant — essential for diagnosing deadlocks, thread pool exhaustion, or unexpected blocking.

```bash
jcmd <pid> Thread.print > threads.txt
# or, classic alternative:
jstack <pid> > threads.txt
```

What to look for:

- **`BLOCKED` threads**: waiting to acquire a lock another thread holds — if many threads are `BLOCKED` on the same lock, that lock is your bottleneck (see `synchronized`/`ReentrantLock`).
- **Deadlock detection**: `jstack`/`Thread.print` explicitly reports "Found one Java-level deadlock" with the exact thread/lock cycle — no need to trace it manually.
- **`WAITING`/`TIMED_WAITING` threads pooled up on the same call**: often means a downstream dependency is slow and threads are piling up waiting on it (exactly the situation a Bulkhead/Circuit Breaker is meant to prevent).
- **Take 3-4 dumps a few seconds apart** rather than one — comparing them shows whether threads are truly stuck (same stack every time) or just momentarily busy (different stacks each time).

## 6. GC Analysis — Is Garbage Collection the Bottleneck?

Long or frequent GC pauses show up as latency spikes that don't correlate with any specific "slow method" in a CPU profile — because the JVM itself paused, not your code.

```bash
# Enable unified GC logging (JDK 9+)
java -Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=100m -jar app.jar
```

Key signals in GC logs / JFR's GC view:

- **Pause frequency and duration**: frequent long pauses (100ms+) directly show up as latency tail spikes (the p99 problem from observability).
- **Heap usage after GC not going down over time**: suggests a genuine memory leak (objects that should be collectible but aren't) rather than just heap being "full but healthy."
- **High allocation rate**: if the app is allocating heavily (visible in JFR's allocation profiling), GC runs more often even without a leak — sometimes the fix is reducing allocations (e.g., reusing buffers, avoiding unnecessary boxing) rather than tuning GC itself.

Don't reach for GC tuning flags before confirming GC is actually the problem — check the pause data first; tuning collector parameters blindly is a common source of wasted effort and regressions.

## 7. JMH — Microbenchmarking Correctly

Never benchmark JVM code with a naive `System.currentTimeMillis()` loop — JIT warmup, dead code elimination, and constant folding will silently produce meaningless numbers. JMH (Java Microbenchmark Harness) handles all of this correctly.

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5)
@Measurement(iterations = 5)
public class StringConcatBenchmark {

    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100; i++) sb.append(i);
        return sb.toString();
    }

    @Benchmark
    public String plusOperator() {
        String result = "";
        for (int i = 0; i < 100; i++) result += i; // creates a new String each iteration
        return result;
    }
}
```

Why this matters: without JMH's warmup phases, the JIT compiler hasn't kicked in yet and you're measuring interpreted bytecode, not the steady-state performance your app actually runs at in production. JMH also guards against the JIT eliminating your "unused" benchmark result entirely (dead code elimination), which would make a benchmark report near-zero time for work that never actually executed.

## 8. Application Startup Profiling

Startup time is a distinct concern from steady-state performance, especially relevant for serverless/container cold starts and CI feedback loops.

```bash
# Class loading and JIT compilation timing
java -Xlog:class+load:file=classload.log -jar app.jar

# Or use JFR's startup event category directly
java -XX:StartFlightRecording:settings=profile,filename=startup.jfr -jar app.jar
```

Common startup bottlenecks in Spring apps: classpath scanning over an overly broad base package, eager bean initialization that could be lazy, and unnecessary auto-configuration being pulled in — Spring Boot's own `--debug` flag prints an auto-configuration report showing exactly what got loaded and why.

## 9. Best Practices

| Practice                                                 | Recommendation                                                                                                                             |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Start with JFR, not a heavier third-party agent          | Low overhead enough to run continuously in production; gives CPU, allocation, lock contention, and GC data from one recording.             |
| Profile in an environment resembling production          | JIT behavior, GC pause characteristics, and contention patterns differ meaningfully between a laptop and a production JVM under real load. |
| Take multiple thread dumps, not one                      | A single dump can't distinguish "genuinely stuck" from "momentarily busy" — compare several taken seconds apart.                           |
| Always enable `HeapDumpOnOutOfMemoryError` in production | An OOM without a heap dump is a lost debugging opportunity — the process is already dying anyway, so the dump costs nothing extra.         |
| Use JMH for any "is A faster than B" question            | Hand-rolled timing loops are routinely wrong due to JIT warmup and dead code elimination.                                                  |
| Confirm GC is the bottleneck before tuning it            | Check pause frequency/duration and heap trend first — don't guess at collector flags.                                                      |
| Profile before optimizing                                | Intuition about "the slow part" is wrong often enough that skipping measurement leads to optimizing code that was never the bottleneck.    |
