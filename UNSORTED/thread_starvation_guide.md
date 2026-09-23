# Understanding and Investigating Thread Starvation in Microservices

## What is Thread Starvation?

**Thread starvation** occurs when a process or service runs out of available threads to process incoming work. In a high-latency microservice architecture, thread starvation rarely happens because the CPU is overloaded; rather, it occurs because execution threads are **blocked waiting on external dependencies** (such as downstream HTTP services, database queries, message brokers, or file I/O).

When worker threads spend long periods idle or blocked awaiting responses, incoming HTTP requests accumulate in execution queues. Once those queues fill up, the service starts dropping requests, timing out, or returning `503 Service Unavailable` errors—causing cascading failures throughout the system.

---

## Root Causes in High-Latency Microservices

1. **Synchronous Blocking I/O:** Using synchronous HTTP clients or JDBC drivers where each request holds onto a OS/JVM thread until a response returns.
2. **Missing or Generous Timeouts:** Lacking socket/connect/read timeouts on external clients, causing threads to hang indefinitely when downstream dependencies degrade.
3. **Connection Pool Bottlenecks:** Database or HTTP connection pool sizes being smaller than the thread pool size, causing threads to block waiting for an available connection.
4. **Shared Thread Pools (Unisolated Workloads):** Running slow background tasks or heavy analytical queries on the same thread pool that handles incoming API traffic.
5. **Deadlocks or Thread Contention:** Synchronized blocks or lock contention where threads wait on shared resources.

---

## Step-by-Step Investigation Workflow

```
[1. APM & Metrics] ---> [2. Identify Pool Saturation] ---> [3. Thread Dumps] ---> [4. Remediate]
```

### Step 1: Detect Queueing & Latency Spikes (APM / Distributed Tracing)

Check APM tools (Datadog, Prometheus/Grafana, Dynatrace) and distributed tracing (Jaeger, Zipkin, OpenTelemetry).

- **Check Request Duration vs. Execution Time:** Look for high $p99$ or $p99.9$ latency. If total request duration is much larger than actual CPU processing time, threads are waiting on I/O or queueing.
- **Inspect Trace Gaps:** In distributed traces, look for empty gaps between span creation and execution start. A delay before the first downstream call span indicates time spent sitting in a web server work queue waiting for an available thread.

---

### Step 2: Analyze Thread Pool & Runtime Metrics

Examine metrics exported by your application runtime or web server (e.g., Tomcat, Netty, Jetty, gRPC, or .NET ThreadPool).

- **Active vs. Max Threads:** Check if `thread_pool_active_count == thread_pool_max_size`.
- **Queue Depth:** Check if `thread_pool_queue_size` is steadily increasing or maxed out.
- **System CPU Usage:** If CPU utilization is low (< 20-30%) while latency and thread usage are at 100%, thread starvation due to blocking I/O is almost guaranteed.

---

### Step 3: Capture and Analyze Thread Dumps

Take 3 to 5 consecutive thread dumps spaced 5 to 10 seconds apart during the incident to see what threads are doing over time.

#### Capturing Thread Dumps

- **Java (JVM):**

  ```bash
  # Using jstack
  jstack -l <pid> > thread_dump_1.txt

  # Using CLI inside Docker/Kubernetes
  kubectl exec -it <pod-name> -- jcmd 1 Thread.print > thread_dump_1.txt
  ```

#### Analyzing the Dumps

Look for recurring stack traces across multiple dumps:

1. **State `BLOCKED` or `WAITING`:** Look for threads stuck on network socket reads or database driver methods.
   - _Example:_ `java.net.SocketInputStream.socketRead0` or `com.mysql.cj.jdbc` waiting on socket read.
2. **Synchronized Lock Contention:** Search for `waiting to lock <0x0000000...>` and locate the thread holding that lock (`locked <0x0000000...>`).
3. **Thread State Distribution:** Calculate the percentage of worker threads in `TIMED_WAITING` or `WAITING` versus `RUNNABLE`.

---

### Step 4: Inspect Downstream Dependencies and Resource Pools

If threads are blocked on external calls:

- **Verify Downstream Health:** Is a downstream microservice experiencing high latency or degraded performance?
- **Check Connection Pools:** Ensure connection pools (e.g., HikariCP for JDBC, Apache HttpClient pool) aren't exhausted. If threads are waiting for connections (`getConnection()`), the connection pool size or timeout needs adjustment.

---

## Remediation & Prevention Strategies

| Strategy                         | Mechanism                                                                                  | Best Used For                                                                              |
| :------------------------------- | :----------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------- |
| **Strict Timeouts & Retries**    | Set tight connection and read timeouts (e.g., 500ms - 2s).                                 | Preventing slow downstream services from holding upstream threads indefinitely.            |
| **Circuit Breakers**             | Fail fast (e.g., Resilience4j, Envoy) when error/latency thresholds are exceeded.          | Isolating failing downstream microservices and freeing threads immediately.                |
| **Bulkheading / Pool Isolation** | Assign separate thread pools to different downstream routes or background tasks.           | Preventing a single slow endpoint from starving the entire application thread pool.        |
| **Non-Blocking / Reactive I/O**  | Move to event-driven/reactive frameworks (Netty, WebClient, Spring WebFlux, Node.js).      | High-concurrency services making frequent network calls.                                   |
| **Virtual Threads (Java 21+)**   | Use lightweight threads (Project Loom) where blocking I/O yields the underlying OS thread. | Legacy synchronous Java codebases needing high concurrency without architectural rewrites. |

---

## Summary Checklist for Incident Response

1. **Confirm issue:** High latency + low CPU + high active thread count.
2. **Collect evidence:** Take 3+ thread dumps spaced 5 seconds apart.
3. **Locate bottleneck:** Search dumps for common call stacks (socket reads, lock acquisition, pool waits).
4. **Immediate mitigation:**
   - Scale out replicas horizontally (temporary buffer).
   - Apply rate limiting at the API gateway level.
   - Enable or tighten client timeouts.
