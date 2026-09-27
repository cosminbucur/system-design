The Java Collections Framework gives you a handful of interfaces (`List`, `Set`, `Map`, `Queue`) with multiple implementations each, tuned for different access patterns. Picking the right one is mostly about knowing each implementation's internal data structure and what that structure makes fast or slow — the interface name alone (`List`) doesn't tell you the performance characteristics; the implementation (`ArrayList` vs `LinkedList`) does.

## 1. List — Ordered, Duplicates Allowed

### ArrayList — Backed by a Resizable Array

Elements are stored contiguously in an array; when it fills up, a new, larger array is allocated (typically 1.5x growth) and elements are copied over.

```java
List<String> names = new ArrayList<>();
names.add("Cosmin");        // O(1) amortized — occasional resize costs O(n), averaged out
names.get(0);                // O(1) — direct array index
names.add(0, "Alex");        // O(n) — every subsequent element shifts right
names.remove(0);              // O(n) — every subsequent element shifts left
```

Fast random access (`get(index)`), slow insertion/removal in the middle or at the front. This is the right default for almost all list use cases — most access patterns are "iterate" or "get by index," both of which `ArrayList` handles well.

### LinkedList — Backed by a Doubly-Linked Node Chain

Each element is a node holding references to the previous/next node — no contiguous array, no resizing.

```java
List<String> names = new LinkedList<>();
names.get(5);      // O(n) — must walk the chain from the start (or end, whichever is closer)
names.add(0, "x"); // O(1) — just relink a few pointers, no shifting
```

`LinkedList` is rarely the right choice in practice: its O(1) insertion advantage only applies if you already hold a reference to the node (via a `ListIterator`) — a plain `add(0, ...)` or `add(index, ...)` still has to traverse to that position first, which is O(n) anyway. It also has worse cache locality than `ArrayList` (nodes are scattered in memory, not contiguous), making it slower in practice for many workloads despite the same Big-O for some operations. Reach for it specifically when you need a `Deque` (stack/queue operations at both ends) — and even then, `ArrayDeque` (below) usually outperforms it.

### ArrayDeque — The Better Stack/Queue

A resizable circular array optimized for adding/removing at both ends — generally preferred over `LinkedList` even for stack/queue use cases.

```java
Deque<Task> stack = new ArrayDeque<>();
stack.push(task);   // O(1) amortized
stack.pop();          // O(1) amortized

Deque<Task> queue = new ArrayDeque<>();
queue.offer(task);   // O(1) amortized
queue.poll();          // O(1) amortized
```

## 2. Set — Unique Elements, No Duplicates

### HashSet — Backed by a HashMap Internally

`HashSet<E>` is literally implemented as a `HashMap<E, Object>` with a dummy value.

```java
Set<String> skus = new HashSet<>();
skus.add("SKU-1");        // O(1) average
skus.contains("SKU-1");    // O(1) average
```

No guaranteed iteration order — two runs of the same program can iterate a `HashSet` in different orders (order depends on hash values and internal bucket layout, which can shift as the set resizes).

### LinkedHashSet — Insertion-Order Preserving

Adds a doubly-linked list on top of the hash table purely to track insertion order — same O(1) average performance as `HashSet`, plus predictable iteration order.

```java
Set<String> visitedPages = new LinkedHashSet<>(); // iterates in the order pages were first visited
```

### TreeSet — Sorted, Backed by a Red-Black Tree

Maintains elements in sorted order (natural ordering via `Comparable`, or a supplied `Comparator`) at the cost of O(log n) instead of O(1) for basic operations.

```java
Set<BigDecimal> amounts = new TreeSet<>();
amounts.add(new BigDecimal("50.00"));  // O(log n)
amounts.first();                        // O(log n) — smallest element
amounts.higher(new BigDecimal("30.00")); // O(log n) — smallest element strictly greater than 30.00
```

Use `TreeSet` only when you actually need sorted iteration or range queries (`headSet`, `tailSet`, `subSet`) — otherwise it's paying O(log n) for no benefit over a `HashSet`'s O(1).

## 3. Map — Key-Value Pairs

### HashMap Internals — Buckets, Hashing, and Treeification

A `HashMap` stores entries in an array of "buckets." A key's `hashCode()` determines which bucket it lands in; multiple keys landing in the same bucket (a "collision") are historically chained in a linked list within that bucket.

```
hashCode() → spread/mix the hash bits → index = hash & (table.length - 1) → bucket
```

```java
Map<String, Account> accounts = new HashMap<>();
accounts.put("acc-1", account);   // O(1) average — compute hash, jump to bucket
accounts.get("acc-1");             // O(1) average
```

Since Java 8, if a single bucket accumulates enough colliding entries (default threshold: 8), that bucket converts from a linked list to a **red-black tree**, changing worst-case lookup within that bucket from O(n) to O(log n) — a defense against pathological hash collisions (accidental or adversarial) degrading the whole map to linked-list-like behavior. This treeification only helps if the key type properly implements `Comparable`; for keys that don't, the worst case is still a long chain.

**Load factor and resizing**: a `HashMap` resizes (doubles its bucket array and rehashes everything) once it's filled beyond `capacity * loadFactor` (default load factor: 0.75). If you know the approximate final size upfront, pre-sizing the initial capacity avoids repeated resize-and-rehash operations:

```java
Map<String, Account> accounts = new HashMap<>(1024); // avoids several resizes if you expect ~700+ entries
```

### The `equals()`/`hashCode()` Contract — Where Bugs Hide

`HashMap`/`HashSet` correctness depends entirely on keys honoring the contract: **equal objects must have equal hash codes** (the reverse isn't required — unequal objects can share a hash code, that's just a collision).

```java
public class OrderId {
    private final String value;

    @Override
    public boolean equals(Object o) {
        return o instanceof OrderId other && this.value.equals(other.value);
    }

    @Override
    public int hashCode() {
        return value.hashCode(); // MUST be consistent with equals()
    }
}
```

Break this contract (e.g., override `equals()` but forget `hashCode()`, so two "equal" objects hash differently) and a `HashMap`/`HashSet` will silently fail to find entries that are logically present — `get()` returns `null` for a key that `equals()` says should match, because it landed in a different bucket. Modern Java `record`s generate both correctly by construction, which is one more reason to prefer them for value-like keys.

**Never use a mutable object as a `HashMap` key** (or `HashSet` element) if any field involved in `equals()`/`hashCode()` can change after insertion — the object's hash code changes, but its position in the bucket array doesn't move, so it becomes permanently unfindable at its "current" hash even though it's still physically in the map.

### LinkedHashMap and TreeMap

Same relationship as their `Set` counterparts: `LinkedHashMap` adds predictable (insertion or access) iteration order at the same average-case cost as `HashMap`; `TreeMap` keeps keys sorted at O(log n) per operation, backed by a red-black tree, and supports range queries (`firstKey()`, `ceilingKey()`, `subMap()`).

```java
// LinkedHashMap with accessOrder=true is a common building block for a simple LRU cache
Map<String, Product> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Product> eldest) {
        return size() > 100; // evict the least-recently-accessed entry once over capacity
    }
};
```

## 4. Complexity Cheat Sheet

| Operation              | ArrayList      | LinkedList | HashMap/HashSet | TreeMap/TreeSet | LinkedHashMap/Set     |
| ---------------------- | -------------- | ---------- | --------------- | --------------- | --------------------- |
| Get by index/key       | O(1)           | O(n)       | O(1) avg        | O(log n)        | O(1) avg              |
| Insert at end          | O(1) amortized | O(1)       | O(1) avg        | O(log n)        | O(1) avg              |
| Insert at front/middle | O(n)           | O(1)\*     | O(1) avg        | O(log n)        | O(1) avg              |
| Contains/search        | O(n)           | O(n)       | O(1) avg        | O(log n)        | O(1) avg              |
| Maintains order        | Insertion      | Insertion  | None guaranteed | Sorted          | Insertion (or access) |

\*O(1) only if you already hold the node reference (via iterator); otherwise O(n) to reach that position first.

## 5. Choosing the Right Collection

| Need                                                       | Use                                                                        |
| ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| Ordered list, mostly reading/iterating, occasional appends | `ArrayList` (the default choice)                                           |
| Stack or queue behavior                                    | `ArrayDeque`                                                               |
| Fast lookup by key, don't care about order                 | `HashMap`                                                                  |
| Fast lookup by key, need predictable iteration order       | `LinkedHashMap`                                                            |
| Fast lookup by key, need sorted keys or range queries      | `TreeMap`                                                                  |
| Unique elements, don't care about order                    | `HashSet`                                                                  |
| Unique elements, need sorted order or range queries        | `TreeSet`                                                                  |
| Producer/consumer handoff between threads                  | `BlockingQueue`                                                            |
| Thread-safe map with high concurrent read/write            | `ConcurrentHashMap` — never a synchronized `HashMap` wrapper for hot paths |

## 6. Immutable Collections

Since Java 9, factory methods create genuinely immutable collections — attempting to modify one throws `UnsupportedOperationException` at the point of misuse, rather than silently allowing a mutation that violates an assumption elsewhere in the code.

```java
List<String> names = List.of("Cosmin", "Alex");   // immutable
Set<String> skus = Set.of("SKU-1", "SKU-2");
Map<String, Integer> scores = Map.of("Cosmin", 100, "Alex", 90);

names.add("Someone"); // throws UnsupportedOperationException immediately
```

Contrast with `Collections.unmodifiableList(list)`, which wraps a list but doesn't protect against the _original_ mutable list still being changed elsewhere — `List.of(...)` copies/holds its own data and has no such backdoor. Prefer `List.of`/`Set.of`/`Map.of` for genuinely fixed data (configuration, constants, defensive copies returned from a method) — this is the same "make invalid states unrepresentable" instinct behind records and sealed types.

## 7. Comparable vs Comparator

`Comparable<T>` defines a type's single, natural ordering (implemented on the class itself); `Comparator<T>` defines an external, and potentially multiple, orderings without modifying the class.

```java
public class Order implements Comparable<Order> {
    private final Instant createdAt;

    @Override
    public int compareTo(Order other) {
        return this.createdAt.compareTo(other.createdAt); // the ONE natural ordering for Order
    }
}

// Comparator: as many orderings as you need, defined wherever they're needed
Comparator<Order> byTotalDesc = Comparator.comparing(Order::getTotal).reversed();
Comparator<Order> byCustomerThenDate = Comparator.comparing(Order::getCustomerName)
    .thenComparing(Order::getCreatedAt);

orders.sort(byCustomerThenDate);
```

A `TreeMap`/`TreeSet` uses `compareTo`/`compare` — not `equals()` — to determine both ordering AND uniqueness; an inconsistent implementation (where `compareTo` returns 0 for objects that aren't `equals()`) causes one of them to be silently treated as a duplicate and dropped.

## 8. Best Practices

| Practice                                                                                              | Recommendation                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Default to `ArrayList` and `HashMap`                                                                  | They're the right choice for the large majority of list/map use cases — reach for alternatives only when a specific need (sorting, thread-safety, order preservation) justifies it. |
| Avoid `LinkedList` unless you specifically need `Deque` semantics with node-reference-based insertion | `ArrayDeque` usually outperforms it even for stack/queue use cases.                                                                                                                 |
| Never use a mutable object as a map key or set element if its hash-relevant fields can change         | The object becomes unfindable once its hash code changes after insertion.                                                                                                           |
| Always override `equals()` and `hashCode()` together                                                  | Breaking the contract causes `HashMap`/`HashSet` lookups to silently fail for logically-equal keys.                                                                                 |
| Pre-size a `HashMap`/`ArrayList` when the final size is roughly known                                 | Avoids repeated resize-and-copy/rehash operations during population.                                                                                                                |
| Use `TreeMap`/`TreeSet` only when sorted order or range queries are actually needed                   | Otherwise you're paying O(log n) for no benefit over O(1).                                                                                                                          |
| Prefer `List.of`/`Set.of`/`Map.of` for genuinely fixed data                                           | Real immutability (fails fast on mutation attempts), not just a read-only wrapper over a still-mutable backing collection.                                                          |
| Use `ConcurrentHashMap`, not a synchronized wrapper, for concurrent access                            | Far better throughput under contention.                                                                                                                                             |
