# Java Garbage Collectors: Comparison & Tuning Guide

Choosing the right Java Garbage Collector (GC) comes down to balancing three key trade-offs: **Throughput** (how fast your application runs overall), **Latency** (how long application execution stops during GC), and **Footprint** (how much RAM and CPU the GC itself consumes).

---

## 1. Core Comparison Table

| Garbage Collector | Target Goal | Typical Pause Time | Heap Size Range | Default In | Best For | Flag to Enable |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Serial GC** | Minimal footprint & simplicity | High ($100\text{ ms} - 1\text{ s}+$) | Very small ($< 100\text{ MB} - 512\text{ MB}$) | Single-CPU systems | Micro-containers, CLI tools, embedded apps | `-XX:+UseSerialGC` |
| **Parallel GC** | Maximum throughput | Moderate/High ($100\text{ ms} - 2\text{ s}$) | Small to Large ($512\text{ MB} - 4\text{ GB}+$) | Java 8 and earlier | Batch jobs, offline data processing, ETL | `-XX:+UseParallelGC` |
| **G1 GC** | Balanced throughput & latency | Predictable ($\approx 10\text{ ms} - 200\text{ ms}$) | Medium to Large ($2\text{ GB} - 64\text{ GB}+$) | Java 9 through present | Web applications, REST APIs, microservices | `-XX:+UseG1GC` |
| **ZGC** | Ultra-low latency | Sub-millisecond to $< 10\text{ ms}$ | Large to Massive ($8\text{ MB} - 16\text{ TB}$) | Optional (Generational in modern JDKs) | Financial trading, real-time analytics, gaming | `-XX:+UseZGC` |
| **Shenandoah** | Ultra-low latency | Sub-millisecond to $< 10\text{ ms}$ | Large ($2\text{ GB} - 100\text{ GB}+$) | Optional (Red Hat / OpenJDK builds) | Latency-critical apps, large heaps on OpenJDK | `-XX:+UseShenandoahGC` |

---

## 2. Detailed Collector Overviews

### G1 GC (Garbage-First) — *The Modern Standard*
* **Mechanism:** Divides the heap into thousands of equal-sized regions rather than continuous generational boundaries. It estimates which regions contain the most garbage and reclaims those first.
* **Strengths:** High operational stability and predictable response times. Soft pause target limit set via `-XX:MaxGCPauseMillis=200`.
* **Trade-offs:** Higher CPU and memory metadata overhead compared to Parallel GC.
* **Best Use Cases:** Microservices, web services, Spring Boot applications, and general production workloads.

### ZGC (Z Garbage Collector) — *The Low-Latency Choice*
* **Mechanism:** Performs almost all compaction and reclamation concurrently alongside active application threads using colored pointers and load barriers.
* **Strengths:** Guarantees sub-millisecond to low-single-digit millisecond pause times regardless of heap size (scaling seamlessly up to 16 TB).
* **Trade-offs:** Requires additional CPU headroom; raw throughput may drop slightly compared to Parallel or G1 under extreme allocation spikes.
* **Best Use Cases:** High-frequency trading, interactive low-latency APIs, and massive multi-gigabyte memory footprints.

### Parallel GC — *The Throughput Engine*
* **Mechanism:** Uses a full Stop-The-World (STW) architecture, utilizing all available CPU threads in parallel to clean memory as fast as possible.
* **Strengths:** Maximizes raw computational throughput and minimizes GC metadata memory overhead.
* **Trade-offs:** Unpredictable, lengthy pause times during Full GC events.
* **Best Use Cases:** Batch processing, ETL data transformations, and offline analytics where execution speed outweighs pause considerations.

### Serial GC — *The Lightweight Utility*
* **Mechanism:** Single-threaded execution model that halts application threads while performing garbage collection.
* **Strengths:** Extremely minimal memory footprint and zero thread synchronization overhead.
* **Trade-offs:** Scalability bottleneck on multi-core architectures.
* **Best Use Cases:** CLI utilities, serverless functions (e.g., AWS Lambda), or constrained environments ($< 512\text{ MB}$ RAM).

---

## 3. Tuning Guide: G1 GC & ZGC

Tuning Java Garbage Collectors is about setting clear boundaries and eliminating configuration anti-patterns. Modern collectors use dynamic heuristic engines to auto-tune execution.

### Step 1: Establish GC Baseline Logging

Always establish a telemetry baseline before modifying parameters:

```bash
-Xlog:gc*,gc+phases=debug:file=/var/log/jvm/gc.log:time,uptime,pid:filecount=5,filesize=100M
```

---

### Step 2: Tuning G1 GC

#### Primary Configuration Flags
* **`-XX:MaxGCPauseMillis=N`** *(Default: `200`)*
  Defines the soft maximum pause-time target in milliseconds.
  * *Guidance:* Avoid setting unrealistic targets ($< 20\text{ ms}$). An overly strict constraint forces G1 to collect smaller regions more frequently, increasing CPU overhead and lowering throughput.
* **`-XX:InitiatingHeapOccupancyPercent=N`** *(IHOP, Default: `45`)*
  Threshold percentage of total heap occupancy that triggers the concurrent marking cycle.
  * *Guidance:* If logs reveal frequent Full GC occurrences, decrease this value to `30`–`35` to trigger reclamation earlier.

#### Advanced Tuning Parameters
* **`-XX:G1ReservePercent=N`** *(Default: `10`)*
  Sets the percentage of reserved memory headroom to prevent allocation failures.
  * *Guidance:* Increase to `15`–`20` if log outputs show `To-space exhausted` or evacuation failures.
* **`-XX:ConcGCThreads=N`**
  Defines the number of threads allocated to parallel collection phases.
  * *Guidance:* Increase if allocation rate spikes outpace concurrent marking capacity.

---

### Step 3: Tuning ZGC

#### Primary Configuration Flags
* **`-XX:+UseZGC`**
  Activates ZGC.
  * *Note:* Modern JDK releases default to **Generational ZGC**, which handles high allocation rates by isolating young and old generation spaces.
* **`-Xms` and `-Xmx`**
  Set initial and maximum heap allocations to identical values (e.g., `-Xms16g -Xmx16g`).
  * *Guidance:* ZGC relies on adequate heap space to execute concurrent compaction without blocking application threads.

#### Advanced Tuning Parameters
* **`-XX:ConcGCThreads=N`**
  Specifies the dedicated concurrent worker thread count for marking and relocation tasks.
  * *Guidance:* If log streams record *Allocation Stalls*, increase thread count to give the collector more CPU resources.

---

## 4. Best Practices Checklist

1. **Avoid Hardcoding Young Generation Sizes (`-Xmn`):**
   Do not set explicit sizes like `-Xmn` or `-XX:NewSize`, as this disables G1 and ZGC's dynamic runtime sizing heuristics.
2. **Disable Explicit GC Invocations:**
   Always include `-XX:+DisableExplicitGC` in production configurations to prevent third-party code from triggering uncoordinated `System.gc()` calls.
3. **Account for Off-Heap Memory:**
   Ensure native host or container memory limits leave a **20–25% memory buffer** above `-Xmx` to accommodate thread stacks, Metaspace, GC metadata, and JVM native allocations.