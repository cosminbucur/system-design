# Java Set Implementations: HashSet vs. LinkedHashSet vs. TreeSet

| Feature | `HashSet` | `LinkedHashSet` | `TreeSet` |
| :--- | :--- | :--- | :--- |
| **Ordering** | **No order guaranteed** (unpredictable; can change on resizing). | **Insertion order** (iterates in the exact order elements were added). | **Sorted order** (natural ordering or defined by a custom `Comparator`). |
| **Time Complexity** | $O(1)$ average for `add`, `remove`, `contains`. | $O(1)$ average for `add`, `remove`, `contains` (slightly higher constant factor). | $O(\log n)$ for `add`, `remove`, `contains`. |
| **Data Structure** | **Hash table** (backed internally by a `HashMap` instance). | **Hash table + Doubly-linked list** (backed internally by a `LinkedHashMap`). | **Red-Black Tree** (backed internally by a `NavigableMap` / `TreeMap`). |
| **Null Elements** | Allows **1 `null` element**. | Allows **1 `null` element**. | **No `null` elements allowed** (throws `NullPointerException` during comparison). |

## Detailed Breakdown

### 1. HashSet

* **How it works:** Backed by an internal `HashMap` where set elements act as keys and a dummy object serves as the map value. Uses `hashCode()` and `equals()` to place and locate elements across array buckets.

* **Strengths:** Maximum performance with $O(1)$ constant-time lookup, insertion, and deletion on average.

* **Weaknesses:** Provides no iteration order guarantees. Order can alter dynamically when the internal array resizes.

* **When to use:** Your default choice when uniqueness matters, order does not, and speed is paramount.

### 2. LinkedHashSet

* **How it works:** Extends `HashSet` by maintaining a doubly-linked list through all inserted elements alongside the underlying hash table.

* **Strengths:** Combines near $O(1)$ performance with predictable, insertion-ordered iteration.

* **Weaknesses:** Slightly higher memory footprint due to maintaining extra pointers for the linked list.

* **When to use:** When you need fast performance while preserving the sequence in which elements were added (e.g., deduplicating a list while keeping original element order).

### 3. TreeSet

* **How it works:** Implements the `NavigableSet` interface using a self-balancing Red-Black tree under the hood.

* **Strengths:** Keeps elements continuously sorted. Provides efficient range operations (such as `subSet`, `headSet`, `tailSet`) and neighbor lookups (`higher`, `lower`, `ceiling`, `floor`).

* **Weaknesses:** $O(\log n)$ operational cost due to rebalancing comparisons. Rejects `null` elements because sorting relies on object comparisons (`compareTo` or `compare`).

* **When to use:** When elements must remain ordered naturally or via custom logic, or when requiring range queries.

## Key Summary

* **Use `HashSet`** for general, high-speed unique collections where order does not matter.
* **Use `LinkedHashSet`** when you need unique elements while preserving original insertion order.
* **Use `TreeSet`** when you need elements continuously sorted or need navigational range queries.