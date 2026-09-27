# Java List Implementations: ArrayList vs. LinkedList vs. Vector

| Feature | `ArrayList` | `LinkedList` | `Vector` |
| :--- | :--- | :--- | :--- |
| **Data Structure** | **Dynamic Array** (resizes automatically as elements are added). | **Doubly-Linked List** (nodes containing references to next and previous nodes). | **Dynamic Array** (resizes automatically, legacy implementation). |
| **Random Access (`get(i)`)** | **$O(1)$** (fast direct index lookup). | **$O(n)$** (must traverse nodes sequentially to reach index). | **$O(1)$** (fast direct index lookup). |
| **Insertion / Deletion** | **$O(n)$** worst/average case due to element shifting; **$O(1)$ amortized** at the end. | **$O(1)$** if node reference is known or at ends; **$O(n)$** if searching for index first. | **$O(n)$** worst/average case due to element shifting and synchronization overhead. |
| **Memory Overhead** | **Low** (contiguously stored in contiguous memory blocks; slight unused capacity padding). | **High** (stores extra pointers to `next` and `prev` for every single element). | **Low** (similar to `ArrayList`). |
| **Thread Safety** | **Not Thread-Safe** (fast, unsynchronized). | **Not Thread-Safe** (fast, unsynchronized). | **Thread-Safe** (all methods are `synchronized`, causing performance overhead). |

---

## Detailed Breakdown

### 1. ArrayList

* **How it works:** Backed by an internal array that dynamically expands when full (typically growing by 50% capacity).
* **Strengths:** Excellent read performance with $O(1)$ constant time access by index. Cache-friendly because elements are stored contiguously in memory.
* **Weaknesses:** Deletions or insertions in the middle of the list require shifting all subsequent elements in memory ($O(n)$).
* **When to use:** Your default choice for a list. Ideal when reading and appending elements (`add()`) are the primary operations.

---

### 2. LinkedList

* **How it works:** Implements both `List` and `Deque` interfaces using a doubly-linked chain of node objects.
* **Strengths:** Efficient $O(1)$ insertions and removals at the beginning or end of the list without shifting elements.
* **Weaknesses:** Poor cache locality and high memory consumption per node. Index-based lookups require iterating through elements sequentially from either end ($O(n)$).
* **When to use:** When you frequently add or remove elements from the head/tail (e.g., queues, stacks, or deques). *Note: `ArrayDeque` is usually preferred over `LinkedList` even for stack/queue use cases due to better cache locality.*

---

### 3. Vector

* **How it works:** A legacy wrapper around a dynamic array (doubles capacity when full) where almost every method uses the `synchronized` keyword.
* **Strengths:** Built-in thread safety out of the box.
* **Weaknesses:** Synchronization imposes significant lock acquisition overhead even in single-threaded contexts. Considered largely obsolete.
* **When to use:** Rarely in modern Java. If thread safety is required, preferred alternatives include `Collections.synchronizedList()` or concurrent structures like `CopyOnWriteArrayList`.

---

## Summary Recommendation

* **Use `ArrayList`** in almost all standard scenarios.
* **Use `LinkedList`** (or `ArrayDeque`) if implementing FIFO queues or double-ended queues requiring fast end operations.
* **Avoid `Vector`** in modern applications; use `CopyOnWriteArrayList` for high-read concurrent scenarios instead.