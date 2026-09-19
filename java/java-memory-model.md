The Java Memory Model (JMM) defines what guarantees the JVM actually makes about when one thread's writes become visible to another thread, and in what order operations appear to happen — it's the specification underneath why `synchronized`, `volatile`, and the `java.util.concurrent` utilities work the way they do. Without understanding it, concurrent bugs look like nonsensical, unreproducible magic; with it, they're predictable consequences of specific, nameable rules.

## 1. Why a Memory Model Is Needed At All

Modern CPUs and compilers reorder instructions and cache values in per-core caches for performance — entirely invisible and harmless in single-threaded code, but a direct source of bugs once multiple threads read and write shared state, because a write on one CPU core isn't guaranteed to be immediately visible to a different core reading the same variable.

```java
class Flag {
    private boolean ready = false;
    private int value = 0;

    void writer() {
        value = 42;    // (1)
        ready = true;  // (2)
    }

    void reader() {
        if (ready) {           // may see (2) without having seen (1) — no ordering guarantee without synchronization
            System.out.println(value); // could print 0, not 42!
        }
    }
}
```

Without a memory model rule connecting `writer()` and `reader()`, the compiler/CPU is free to reorder (1) and (2), and the reading thread might never observe `writer()`'s effects at all, or observe them out of order — this isn't a hypothetical, it's exactly the kind of bug that appears intermittently, only under real concurrent load, and never in a debugger (which serializes execution and hides the reordering). This is the _same_ underlying problem the atomicity discussion in concurrency is about — atomicity and visibility are two distinct concerns, both needed for correct concurrent code.

## 2. Atomicity vs Visibility — Two Different Problems

It's easy to conflate these, but fixing one doesn't fix the other:

| Problem    | What it means                                                                                                                   | Fixed by                                                                   |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Atomicity  | An operation (like `count++`) actually completes as one indivisible step, with no other thread able to interleave in the middle | `synchronized`, `AtomicInteger`, locks                                     |
| Visibility | A write made by one thread is guaranteed to be seen by another thread, in a timely and correctly-ordered way                    | `volatile`, `synchronized`, or anything establishing a happens-before edge |

`volatile` fixes visibility but **not** atomicity — `volatile int count; count++;` is still a read-modify-write with a race condition, because `count++` is three separate operations (read, add, write) and `volatile` only guarantees each individual read/write is visible, not that the whole sequence is atomic. Conversely, `synchronized`/locks fix both at once, which is why they're the more commonly reached-for tool despite being heavier-weight.

## 3. Happens-Before — The Core Rule

The JMM formalizes visibility guarantees through a **happens-before** relationship: if action A happens-before action B, then A's effects (including all writes to shared memory before A) are guaranteed visible to B. Without an explicit happens-before edge between two threads' operations, the JMM makes _no_ visibility or ordering guarantee at all — not "probably fine," genuinely unspecified.

Key happens-before rules:

| Rule              | Effect                                                                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Program order     | Within a single thread, each action happens-before the next action in that thread's code                                                      |
| Monitor lock      | Releasing a lock (`synchronized` block exit) happens-before any subsequent acquire of that same lock                                          |
| Volatile variable | A write to a `volatile` field happens-before every subsequent read of that same field                                                         |
| Thread start      | `Thread.start()` happens-before any action in the started thread                                                                              |
| Thread join       | Every action in a thread happens-before another thread successfully returns from `Thread.join()` on it                                        |
| Final field       | A properly constructed object's `final` fields are visible to any thread that gets a reference to the object, without further synchronization |

```java
class Flag {
    private volatile boolean ready = false; // volatile fixes the bug
    private int value = 0;

    void writer() {
        value = 42;    // (1)
        ready = true;  // (2) — a volatile write
    }

    void reader() {
        if (ready) {    // a volatile read of the same field — happens-before guarantees (1) is visible here too
            System.out.println(value); // guaranteed to print 42, never 0
        }
    }
}
```

The volatile write to `ready` happens-before the volatile read of `ready` — and because program order also holds within each thread, everything the writer thread did _before_ writing `ready` (including setting `value`) becomes visible to anything that happens _after_ the reader thread's read of `ready`. This is the exact mechanism that makes the fix work, not just "volatile makes things visible" as a vague rule.

## 4. `synchronized` — Both Atomicity and Visibility

A `synchronized` block gives you the monitor-lock happens-before edge _and_ mutual exclusion (only one thread executes the block at a time) — this is why it's the more commonly reached-for default over `volatile` alone.

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++; // atomic (mutual exclusion) AND visible (monitor happens-before) to the next synchronized caller
    }

    public synchronized int get() {
        return count; // must also be synchronized — otherwise this read has no happens-before edge with increment()
    }
}
```

A common mistake: synchronizing the writer but not the reader (e.g., a plain `public int get() { return count; }` without `synchronized`). Without the reader also acquiring the same lock, there's no happens-before edge for it — the read might never observe the write, even though the write itself is properly synchronized. Both sides of the same shared state need to participate in the same synchronization mechanism for the happens-before guarantee to apply.

## 5. `final` Fields and Safe Publication

A correctly-constructed object's `final` fields are guaranteed visible to any thread that obtains a reference to the object — even _without_ explicit synchronization — as long as the reference doesn't escape the constructor before construction finishes (a subtle but critical caveat).

```java
public final class ImmutablePoint {
    private final int x;
    private final int y;

    public ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
        // as long as `this` isn't leaked here (e.g., registering a listener with `this`
        // before the constructor returns), any thread that later gets a reference to
        // this ImmutablePoint is guaranteed to see the fully-initialized x and y
    }
}
```

This is exactly why immutable objects — value objects, and records specifically — are inherently thread-safe with no extra synchronization: their `final` fields get this safe-publication guarantee for free, for any thread, as long as the reference is published after construction completes.

## 6. The Classic Broken Pattern: Double-Checked Locking

A famous historical bug, worth knowing specifically because it demonstrates how subtle visibility issues can be — even experienced developers got this wrong for years before the JMM was clarified in Java 5.

```java
// BROKEN without `volatile` on `instance` (pre-Java-5 understanding, or forgetting volatile today)
public class Singleton {
    private static Singleton instance; // missing volatile — the bug

    public static Singleton getInstance() {
        if (instance == null) {              // first check, no lock — fast path
            synchronized (Singleton.class) {
                if (instance == null) {        // second check, inside the lock
                    instance = new Singleton(); // (*) can appear to complete before the constructor actually finishes!
                }
            }
        }
        return instance;
    }
}
```

The problem at `(*)`: without `volatile`, the compiler/CPU can reorder the write to `instance` so it becomes visible to another thread _before_ the `Singleton` constructor has actually finished running — a second thread's first, unlocked check (`if (instance == null)`) can see a non-null but only partially-constructed object. The fix is exactly the `volatile` fix:

```java
private static volatile Singleton instance; // volatile restores the needed happens-before edge
```

In modern Java, prefer avoiding this pattern altogether — an `enum` singleton or a static holder class-based lazy singleton sidesteps the whole problem without needing to reason about `volatile` and happens-before at all.

## 7. Where `java.util.concurrent` Fits

Every higher-level tool from is built on top of these same JMM guarantees, so you rarely need to reason about happens-before directly in application code — but it's useful to know where the guarantee actually comes from:

| Tool                              | JMM guarantee it relies on / provides                                                                                                          |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `AtomicInteger`/`AtomicReference` | Internally uses CPU-level compare-and-swap plus volatile-equivalent semantics — atomic AND visible                                             |
| `ReentrantLock`                   | `lock()`/`unlock()` establish the same happens-before edge as `synchronized`'s monitor entry/exit                                              |
| `ConcurrentHashMap`               | Reads/writes on the same bucket establish happens-before relationships internally — you get correct visibility without managing locks yourself |
| `ExecutorService.submit()`        | Submitting a task happens-before the task starts running; the task's completion happens-before `Future.get()` returns                          |
| `CompletableFuture`               | Each stage's completion happens-before the next stage begins                                                                                   |

This is precisely why async programming recommends reaching for these utilities instead of hand-rolled `wait`/`notify`/raw `volatile` logic — they encode the correct happens-before relationships internally, so you don't have to re-derive them (and risk getting them subtly wrong) for every piece of concurrent code you write.

## 8. Best Practices

| Practice                                                                                               | Recommendation                                                                                                                           |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Never assume a write is visible to another thread without an explicit happens-before edge              | No synchronization, no guarantee — this is unspecified behavior, not "probably fine in practice."                                        |
| Remember `volatile` fixes visibility, not atomicity                                                    | `volatile int count; count++;` is still a race condition — reach for `AtomicInteger` or `synchronized` for compound operations.          |
| Synchronize both the writer and the reader of shared state                                             | A `synchronized` write with an unsynchronized read has no happens-before edge on the read side — the visibility guarantee doesn't apply. |
| Prefer immutability for cross-thread sharing                                                           | `final` fields on a safely-published object are visible without any explicit synchronization.                                            |
| Avoid hand-rolled double-checked locking                                                               | Use an enum singleton or a static holder class instead of reasoning about `volatile` placement yourself.                                 |
| Reach for `java.util.concurrent` utilities over raw `synchronized`/`volatile` for anything non-trivial | They encode correct happens-before relationships internally.                                                                             |
