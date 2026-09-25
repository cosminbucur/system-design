# Java Garbage Collection (GC): Architecture and mechanics

Garbage Collection (GC) is the automatic memory management process in the Java Virtual Machine (JVM). Its primary objective is to reclaim memory occupied by unreferenced objects on the **Java Heap**, preventing memory exhaustion while freeing developers from manual memory allocation and deallocation (`malloc`/`free`).

---

## 1. Core Mechanics: How GC Works

Java GC operates on two fundamental principles: **Reachability Analysis** and the **Generational Hypothesis**.

### Reachability Analysis & GC Roots

An object is eligible for garbage collection if it cannot be reached by traversing a chain of references starting from a **GC Root**. If an object is disconnected from all GC Roots, it is deemed unreachable and marked for collection.

**Common GC Roots include:**

- **Thread Stack Variables:** Local variables and parameters in active thread execution stacks.
- **Static Fields:** Class-level variables stored in loaded classes.
- **JNI References:** Native code references created during Java Native Interface calls.
- **System Classes:** Core classes loaded by the bootstrap classloader.

```
[ GC Root ] ---> [ Active Object A ] ---> [ Active Object B ]

[ Unreferenced Object C ] ---> [ Object D ]  (Eligible for GC)
```

---

### The Generational Hypothesis

Empirical data shows that in most applications, **most objects die young** (e.g., short-lived method variables). To optimize memory management, the Java Heap is divided into distinct generations.

```
+-------------------------------------------------------------------+
|                            JAVA HEAP                              |
+------------------------------------+------------------------------+
|          Young Generation          |        Old Generation        |
|  +------------------+----+----+    |          (Tenured)           |
|  |       Eden       | S0 | S1 |    |                              |
|  +------------------+----+----+    |                              |
+------------------------------------+------------------------------+
```

1. **Young Generation**
   - **Eden Space:** The landing area where newly created objects are initially allocated.
   - **Survivor Spaces (S0 & S1 / From & To):** Two equal-sized spaces used during Minor GC. Only one survivor space holds objects at any given moment while the other remains empty.
2. **Old (Tenured) Generation**
   - Holds long-lived objects that have survived a specified threshold of Minor GC cycles (configured via `-XX:MaxTenuringThreshold`, default up to 15).
3. **Metaspace (Off-Heap, Native Memory)**
   - Introduced in Java 8 to replace the **Permanent Generation (PermGen)**.
   - Stores class metadata, bytecode, and method definitions in OS native memory rather than on the Java Heap, reducing heap-related `OutOfMemoryError` issues.

---

### Major Types of Collections

- **Minor GC:** Triggered when the Eden space becomes full. Cleans the Young Generation using a copying algorithm (fast, high yield).
- **Major GC / Full GC:** Cleans the Old Generation (and often the Young Generation as well). Requires scanning a much larger memory footprint, leading to longer **Stop-The-World (STW)** pauses.

---

## 2. Core GC Algorithms & Collectors

Modern JVMs offer several GC implementations optimized for different tradeoffs between **throughput** and **latency**.

| Garbage Collector             | Strategy & Behavior                                                                                                                                                                  | Best Use Case                                                                            |
| :---------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **G1 (Garbage-First)**        | Default since Java 9. Divides heap into thousands of equal-sized regions. Predicts region garbage density and collects areas with the most garbage first within a target pause time. | Large heap applications (4GB+) needing balanced throughput and low pause times.          |
| **ZGC (Z Garbage Collector)** | Low-latency collector using colored pointers and load barriers. Performs concurrent marking and compacting with sub-millisecond pause times.                                         | Large-scale, latency-sensitive services with heaps ranging from gigabytes to terabytes.  |
| **Shenandoah GC**             | Ultra-low-latency collector that performs concurrent compaction alongside application threads to keep pause times independent of heap size.                                          | Latency-critical applications needing consistent response times regardless of heap size. |
| **Parallel GC**               | Multi-threaded throughput collector. Halts application threads during collection to maximize total CPU application execution time.                                                   | Batch processing, analytics, and non-interactive background jobs.                        |

---

## 3. Key Topics

### 1. Stop-The-World (STW) Events

During certain phases of GC, all application execution threads are paused so the garbage collector can safely inspect or relocate objects without race conditions. Modern collectors (G1, ZGC) focus on minimizing or eliminating STW duration.

### 2. Common Collection Algorithms

- **Mark-and-Sweep:** Identifies live objects, then sweeps unreferenced objects away. Can cause memory fragmentation over time.
- **Mark-Sweep-Compact:** Adds a compaction step to consolidate live objects and defragment memory.
- **Copying:** Moves live objects from one memory region (e.g., Eden) to another (e.g., Survivor), leaving contiguous free space behind.

### 3. Memory Leaks in Java

Clarify that automatic GC **does not eliminate memory leaks**. A memory leak in Java occurs when an object is no longer needed by application logic, but remains reachable via a GC Root.

- **Common Causes:**
  - Static maps or collections that grow indefinitely without removal.
  - Unclosed resources, streams, or connections.
  - Unregistered event listeners or callbacks.
  - Stale references in custom caches or `ThreadLocal` variables.

### 4. Java Reference Types (`java.lang.ref`)

- **Strong Reference (`Object obj = new Object();`):** Default reference type. The object is never collected as long as it is reachable.
- **Soft Reference (`SoftReference`):** Collected **only** when memory is low and an `OutOfMemoryError` is imminent. Often used for memory-sensitive caches.
- **Weak Reference (`WeakReference`):** Collected during the **next GC cycle**, regardless of available memory. Used in metadata maps like `WeakHashMap`.
- **Phantom Reference (`PhantomReference`):** Used to enqueue post-mortem cleanup tasks before object memory is reclaimed, providing a cleaner alternative to deprecated finalizers.

### 5. Key JVM Tuning Flag Summary

```bash
# Heap Sizing
-Xms4g                         # Set initial heap size to 4GB
-Xmx8g                         # Set maximum heap size to 8GB

# Collector Selection
-XX:+UseG1GC                   # Use G1 Garbage Collector
-XX:+UseZGC                    # Use Z Garbage Collector

# Performance Tuning
-XX:MaxGCPauseMillis=200       # Target maximum pause time (G1)
-XX:NewRatio=2                 # Ratio of Old to Young Generation size

# Logging & Diagnostic
-Xlog:gc*                      # Enable unified GC logging (Java 9+)
-XX:+HeapDumpOnOutOfMemoryError # Dump heap snapshot on OOM
```
