# Java Map Implementations: HashMap vs. LinkedHashMap vs. TreeMap

| Feature | `HashMap` | `LinkedHashMap` | `TreeMap` |
| :--- | :--- | :--- | :--- |
| **Ordering** | **No order guaranteed** (unpredictable and can shift as capacity resizes). | **Insertion order** by default (or **access order** if configured for LRU). | **Sorted order** by **keys** (natural ordering or via custom `Comparator`). |
| **Time Complexity** | $O(1)$ average for `get`/`put`/`remove`. | $O(1)$ average for `get`/`put`/`remove` (slightly higher memory/constant overhead). | $O(\log n)$ for `get`/`put`/`remove`. |
| **Data Structure** | **Hash table** (Array of buckets / Linked lists, converting to balanced trees for heavy collisions). | **Hash table + Doubly-linked list** running through all entries. | **Red-Black Tree** (Self-balancing binary search tree). |
| **Null Keys** | Allows **1 `null` key**. | Allows **1 `null` key**. | **No `null` keys allowed** (throws `NullPointerException` due to key comparisons). |

---

## Detailed Breakdown

### 1. HashMap
* **How it works:** It uses `hashCode()` to route keys into specific array buckets. When collisions occur, entries within the same bucket form a linked list (or a red-black tree if collisions exceed a certain threshold).
* **When to use:** Your default choice when order does not matter and you need maximum performance ($O(1)$ lookup, insertion, and deletion time).

### 2. LinkedHashMap
* **How it works:** Extends `HashMap` by maintaining a **doubly-linked list** running through all of its entries. As entries are added, pointers link them sequentially to maintain the order.
* **When to use:** When you need fast $O(1)$ performance but require predictable iteration matching insertion sequence (or access order for LRU Caches).

### 3. TreeMap
* **How it works:** Implements a **Red-Black tree** (a self-balancing binary search tree). Every insertion compares keys (via `Comparable` or `Comparator`) to place the node in the tree while maintaining a balanced height.
* **When to use:** When your keys must stay continuously sorted, or when you need navigation operations like range queries (`subMap`), finding the first key (`firstKey`), or finding nearest keys (`ceilingKey`).

---

## Key Takeaway: Sorting Scope

> **Note:** Sorting in a `TreeMap` strictly applies to **keys**, not values. The values are stored alongside their respective keys and do not affect the tree structure or iteration order.