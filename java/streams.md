The Stream API (Java 8) brought a functional, declarative style to processing collections — describe _what_ transformation you want, not the loop mechanics of _how_. Streams are for transforming in-memory data, synchronously, once — a different problem than the asynchronous, over-time event sequences deals with, even though the two share a lot of operator vocabulary (`map`/`filter`/`flatMap`) and the resemblance is exactly why they're easy to conflate.

## 1. Streams — Declarative Collection Processing

```java
// Imperative: describes HOW — a loop, a mutable accumulator, manual filtering
List<String> activeNames = new ArrayList<>();
for (Account account : accounts) {
    if (account.getStatus() == AccountStatus.ACTIVE) {
        activeNames.add(account.getOwnerName().toUpperCase());
    }
}

// Declarative: describes WHAT — filter, then transform, then collect
List<String> activeNames = accounts.stream()
    .filter(account -> account.getStatus() == AccountStatus.ACTIVE)
    .map(account -> account.getOwnerName().toUpperCase())
    .collect(Collectors.toList());
```

A stream isn't a data structure — it doesn't store elements. It's a pipeline description over a source (a collection, an array, a generator) that only actually runs when a terminal operation is invoked.

## 2. Intermediate vs Terminal Operations — Laziness Matters

Intermediate operations (`filter`, `map`, `sorted`, `distinct`) are lazy — they just build up the pipeline description and don't touch any data until a terminal operation (`collect`, `forEach`, `reduce`, `count`, `findFirst`) triggers execution.

```java
Stream<Account> pipeline = accounts.stream()
    .filter(a -> { System.out.println("filtering " + a.getId()); return a.isActive(); })
    .map(a -> { System.out.println("mapping " + a.getId()); return a.getOwnerName(); });
// Nothing has printed yet — no terminal operation has been called

List<String> names = pipeline.collect(Collectors.toList());
// NOW the whole pipeline runs, element by element
```

A key consequence of laziness: elements are processed one at a time through the _entire_ pipeline (filter → map → collect for element 1, then filter → map → collect for element 2...), not filter-all-then-map-all-then-collect-all. This is why short-circuiting operations (`findFirst`, `anyMatch`, `limit`) can stop processing early without touching the whole source — something a naive multi-pass imperative loop wouldn't get for free.

**A stream can only be consumed once** — calling a terminal operation on an already-consumed stream throws `IllegalStateException`. If you need to run the same source through multiple pipelines, re-create the stream from the source each time (`accounts.stream()` again).

## 3. Common Operations

```java
// map: transform each element
List<String> names = accounts.stream().map(Account::getOwnerName).toList();

// filter: keep elements matching a predicate
List<Account> active = accounts.stream().filter(Account::isActive).toList();

// sorted: order elements (natural order, or with a Comparator)
List<Account> byBalance = accounts.stream()
    .sorted(Comparator.comparing(Account::getBalance).reversed())
    .toList();

// distinct: remove duplicates (uses equals()/hashCode())
List<String> statuses = accounts.stream().map(a -> a.getStatus().name()).distinct().toList();

// reduce: combine elements into a single result
BigDecimal totalBalance = accounts.stream()
    .map(Account::getBalance)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// flatMap: flatten a stream of collections into a single stream
List<Transaction> allTransactions = accounts.stream()
    .flatMap(account -> account.getTransactions().stream())
    .toList();

// limit/skip: pagination-like slicing of a stream (not a substitute for DB pagination)
List<Account> firstTen = accounts.stream().limit(10).toList();
```

`flatMap` is the one that trips people up initially: use it whenever your `map` step would otherwise produce a `Stream<Stream<T>>` — `flatMap` collapses that extra nesting into a single flat `Stream<T>`.

## 4. Collectors — Turning a Stream Back Into Something Useful

```java
// Grouping — like a SQL GROUP BY
Map<AccountStatus, List<Account>> byStatus = accounts.stream()
    .collect(Collectors.groupingBy(Account::getStatus));

// Grouping with a downstream aggregation
Map<AccountStatus, BigDecimal> totalBalanceByStatus = accounts.stream()
    .collect(Collectors.groupingBy(Account::getStatus,
        Collectors.mapping(Account::getBalance, Collectors.reducing(BigDecimal.ZERO, BigDecimal::add))));

// Partitioning — a special case of grouping into exactly two buckets (true/false)
Map<Boolean, List<Account>> partitioned = accounts.stream()
    .collect(Collectors.partitioningBy(a -> a.getBalance().compareTo(BigDecimal.valueOf(1000)) > 0));

// toMap — build a lookup map directly from a stream
Map<Long, Account> accountsById = accounts.stream()
    .collect(Collectors.toMap(Account::getId, Function.identity()));

// joining — build a single delimited String
String csv = accounts.stream().map(Account::getOwnerName).collect(Collectors.joining(", "));

// summary statistics — count/min/max/sum/average in one pass
DoubleSummaryStatistics stats = accounts.stream()
    .mapToDouble(a -> a.getBalance().doubleValue())
    .summaryStatistics();
```

`Collectors.toMap` throws `IllegalStateException` on a duplicate key by default — pass a merge function (`(existing, replacement) -> replacement`) explicitly if duplicates are expected and you want to control which one wins, rather than letting the collector crash on the first duplicate it hits.

## 5. `Optional` — Streams' Companion for "Maybe One Value"

`Optional<T>` represents a value that might be absent — the same idea as a single-element stream, and it composes with the same style of chained operations rather than manual null checks.

```java
Optional<Account> account = accountRepository.findById(id);

// Chain transformations without a null check at every step
String displayName = account
    .map(Account::getOwnerName)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");

// Terminal-style operations
account.ifPresent(a -> log.info("Found: {}", a.getOwnerName()));
account.ifPresentOrElse(
    a -> log.info("Found: {}", a.getOwnerName()),
    () -> log.warn("Account {} not found", id)
);

Account resolved = account.orElseThrow(() -> new AccountNotFoundException(id));
```

Never call `.get()` on an `Optional` without first checking `.isPresent()` — that reintroduces exactly the "forgot to check" risk `Optional` exists to prevent. And never use `Optional` as a field type or method parameter — it's designed as a return type communicating "this call might not produce a value," not a general-purpose null-replacement for every field.

## 6. Parallel Streams — A Narrow Tool, Not a Free Speedup

`.parallelStream()` splits the pipeline across the common `ForkJoinPool`, using multiple CPU cores — but it's only a win under specific conditions, and a common source of surprising slowdowns when applied casually.

```java
// Only worth it for CPU-bound work over a genuinely large dataset
double total = largeDataset.parallelStream()
    .mapToDouble(this::expensiveComputation)
    .sum();
```

When parallel streams backfire:

- **Small collections**: the overhead of splitting work and coordinating threads exceeds any benefit — parallelism only pays off with real per-element work and enough elements to amortize the coordination cost.
- **I/O-bound operations** (a DB call, an HTTP request per element): parallel streams use a fixed-size common `ForkJoinPool` shared JVM-wide — blocking I/O inside it can starve _other, unrelated_ parallel streams running elsewhere in the same JVM, since they all draw from the same pool. Use a dedicated `ExecutorService` for I/O-bound concurrent work instead — or Reactive-programming's non-blocking model, which was built for exactly this.
- **Non-associative or order-dependent operations**: `reduce` with a non-associative combiner, or any operation assuming encounter order, can produce wrong or non-deterministic results under parallel execution.
- **Shared mutable state** in the lambda: the same data race risks from concurrency apply — a parallel stream doesn't protect you from mutating shared state unsynchronized.

Rule of thumb: default to sequential streams; only reach for parallel after profiling shows a genuine CPU-bound bottleneck over a large enough dataset that the numbers justify it.

## 7. Method References — Syntactic Sugar for a Lambda That Just Calls a Method

```java
accounts.stream().map(account -> account.getOwnerName())  // lambda
accounts.stream().map(Account::getOwnerName)                // equivalent method reference

accounts.forEach(account -> log.info("{}", account))       // lambda
accounts.forEach(log::info)                                  // only if signatures line up exactly
```

Four forms: `ClassName::staticMethod`, `object::instanceMethod`, `ClassName::instanceMethod` (the first argument becomes the receiver), and `ClassName::new` (constructor reference, e.g., `Account::new` used as a `Supplier<Account>`). Use a method reference whenever a lambda's entire body is "just call this one method" — it's shorter and, once familiar, more readable than the equivalent lambda.

## 8. Streams vs Reactive — Don't Conflate Them

Because `Mono`/`Flux` in Reactive-programming deliberately mirror Stream operator names (`map`, `filter`, `flatMap`), it's easy to assume they're "just async streams." They're not quite that simple: a `Stream` is synchronous, pulls from an already-available in-memory source, and runs once, on the calling thread. A `Flux`/`Mono` is asynchronous, can represent data arriving over time from I/O (not yet available when the pipeline is built), and includes backpressure — a mechanism a plain `Stream` has no concept of at all, because it never needed one. If you find yourself reaching for reactive types just to "stream-process" an already-in-memory `List`, a plain `Stream` is almost certainly the right (and simpler) tool.

## 9. Best Practices

| Practice                                                              | Recommendation                                                                                                  |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Prefer method references over lambdas that just call one method       | `Account::getOwnerName` over `a -> a.getOwnerName()` — shorter, and idiomatic once familiar.                    |
| Don't parallelize streams by default                                  | Only after profiling confirms a CPU-bound bottleneck over a large enough dataset.                               |
| Use `Optional` as a return type only, never a field or parameter type | It communicates "this call might not produce a value" at the API boundary, not a general null-replacement.      |
| Keep stream pipelines read-only, no shared mutable state in lambdas   | Same data-race caution as concurrency — especially relevant the moment you reach for `.parallelStream()`.       |
| Re-create a stream from its source rather than reusing a consumed one | A stream is single-use; a second terminal operation on the same stream instance throws `IllegalStateException`. |
| Reach for `flatMap` when `map` would produce a nested stream          | Collapses a `Stream<Stream<T>>` into a flat `Stream<T>` instead of leaving callers to unwrap it manually.       |
| Don't use reactive types just to process an in-memory collection      | If the data is already fully available and synchronous, a plain `Stream` is simpler.                            |
