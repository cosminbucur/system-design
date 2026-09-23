A class or piece of code is thread-safe when it behaves correctly no matter how many threads call it concurrently, and no matter how the JVM/OS interleaves their execution — no corrupted state, no lost updates, no inconsistent reads, regardless of timing. This note is about the design-level strategies for achieving that property; the low-level mechanics each strategy relies on (locks, atomics, deadlock) are covered in multithreading, and the precise memory-visibility guarantees behind them are covered in the java memory model.

## 1. The Strategies, at a Glance

There's more than one way to make code thread-safe, and they're not equally good — from strongest/simplest to weakest/most complex to get right.

| Strategy | Core idea | Tradeoff |
| --- | --- | --- |
| Immutability | No mutable state exists, so there's nothing to corrupt | Requires designing data as unchangeable values, not always a natural fit |
| Thread confinement | Mutable state exists, but only one thread ever touches it | Requires discipline to never let the confined object escape to another thread |
| Synchronization | Mutable, shared state exists, access is serialized via locks | Correct but costs throughput, and is easy to get subtly wrong (deadlock, forgotten lock) |
| Lock-free / atomic operations | Mutable, shared state exists, coordinated via CPU-level compare-and-swap instead of locks | Faster under contention, but limited to simple operations (single variables) |
| Thread-safe collections/utilities | Someone else already solved this for a specific data structure | Only covers what the library provides — doesn't generalize to arbitrary custom logic |

The right first question for any shared piece of state isn't "which lock do I need" — it's "can this just not be mutable, or not be shared, in the first place?" Synchronization is the fallback for when the first two options genuinely don't apply, not the default starting point.

## 2. Immutability: Nothing to Corrupt

An object whose state can never change after construction is automatically thread-safe — there's no "current state" for one thread to be reading while another is mid-write, so none of the usual race conditions have anywhere to occur.

```java
// Immutable — every field is final, set once in the constructor, never modified afterward
public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        // returns a NEW Money instead of mutating this one
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

Any number of threads can hold a reference to the same `Money` instance and call `add` concurrently without any coordination at all — each call just produces a new object, and the original is never touched. This is the same discipline already covered as a functional programming principle, and it's worth recognizing that immutability's thread-safety benefit is really the same underlying property (no shared mutable state) viewed from a concurrency angle instead of a functional-style one.

## 3. Thread Confinement: One Thread, No Sharing

If mutable state genuinely needs to exist, the second-best option is ensuring only one thread ever accesses it — not through discipline alone, but by structurally preventing it from being shared.

```java
// Stack confinement — a local variable never escapes the method, so it's inherently thread-safe
void processOrder(Order order) {
    List<LineItem> workingCopy = new ArrayList<>(order.getLineItems()); // mutable, but confined to this call
    workingCopy.sort(Comparator.comparing(LineItem::getSku));
    // no other thread can ever see 'workingCopy' — it never leaves this stack frame
}
```

```java
// ThreadLocal — each thread gets its own independent copy, no coordination needed between them
private static final ThreadLocal<SimpleDateFormat> FORMATTER =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

String format(Date date) {
    return FORMATTER.get().format(date); // this thread's own instance — never shared with another thread
}
```

`ThreadLocal` is the deliberate version of confinement: instead of one shared instance that would need synchronization, every thread gets its own private copy, so there's genuinely nothing to coordinate. This is also the standard fix for wrapping a class that's mutable and *not* thread-safe on its own (like `SimpleDateFormat`) — rather than synchronizing every call to a single shared instance, give each thread its own.

## 4. Synchronization: Coordinating Access to What's Actually Shared

When state must be both mutable and genuinely shared across threads, synchronization (already covered in depth in multithreading) is the fallback — a lock ensures only one thread modifies the state at a time.

```java
public class SharedCounter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}
```

The discipline that matters here isn't just "add `synchronized` somewhere" — it's making sure *every* path that reads or writes the shared state goes through the same lock, consistently. A class that synchronizes some methods touching a field but not others isn't thread-safe at all; it just looks like it might be.

## 5. Thread-Safe by Design vs. Thread-Safe by Accident

A subtle trap: a class can *happen* to work correctly in single-threaded testing while being fundamentally unsafe under concurrent access, because the specific race window never got hit during testing. `SimpleDateFormat` is the canonical example — it's mutable internal state (a shared `Calendar` field) makes concurrent `format()`/`parse()` calls corrupt each other's results, but this often goes unnoticed until production traffic actually creates the race.

```java
// BAD: shared, mutable, and not internally synchronized — silently produces wrong results under concurrency
private static final SimpleDateFormat FORMAT = new SimpleDateFormat("yyyy-MM-dd");

// GOOD: java.time classes (Java 8+) are immutable and inherently thread-safe
private static final DateTimeFormatter FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

This is why "it passed all my tests" is a weak signal for thread safety specifically — a race condition's symptoms are timing-dependent and often don't show up until real concurrent load, sometimes only under production traffic patterns that never occurred in a dev environment. Preferring the modern `java.time` API (immutable) over legacy mutable date classes sidesteps this entire class of bug rather than requiring careful, ongoing vigilance.

## 6. A Common Trap in Frameworks: Stateful Singleton Beans

Spring beans are singletons by default — one shared instance serving every request, across every thread. Adding a mutable instance field to a `@Service`/`@Component` silently turns it into shared mutable state across every concurrent request the application serves.

```java
// BAD: a mutable instance field on a singleton bean — every request shares and corrupts the same state
@Service
public class OrderProcessor {
    private Order currentOrder; // shared across every concurrent request — a real bug waiting to happen

    public void process(Order order) {
        this.currentOrder = order; // one request's order can be overwritten by another's, mid-processing
        // ...
    }
}

// GOOD: state lives in a local variable / method parameter — inherently confined per call
@Service
public class OrderProcessor {
    public void process(Order order) {
        // 'order' is a parameter, not shared instance state — each call has its own
    }
}
```

The fix is almost always the same: keep request-specific data as a local variable or method parameter (naturally confined per call, per thread), not as an instance field on a shared singleton bean.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Prefer immutability first, before reaching for synchronization | An object that can't be mutated has nothing to corrupt — no lock is needed for something that never changes. |
| Use confinement (local variables, `ThreadLocal`) for state that doesn't need to be shared at all | Structurally preventing sharing avoids needing to reason about coordination in the first place. |
| Never add a mutable instance field to a singleton-scoped bean without deliberate synchronization | Every concurrent request shares that same instance — an unguarded mutable field is shared mutable state across the whole application. |
| Prefer `java.time` over legacy mutable date/calendar classes | Sidesteps `SimpleDateFormat`-style concurrency bugs entirely, rather than requiring careful synchronization discipline every time it's used. |
| Don't trust single-threaded test passes as evidence of thread safety | Race conditions are timing-dependent — a class can appear correct in testing and still fail under real concurrent load. |
| Ensure every access path to shared mutable state goes through the same synchronization mechanism | A class that synchronizes some methods touching a field but not others provides no real guarantee at all. |
