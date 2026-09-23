# Apache Tomcat Thread Pool Architecture & Thread Starvation Guide

This guide details how Apache Tomcat manages worker threads, how thread starvation occurs within Tomcat, and strategies for investigating and mitigating starvation issues in microservices.

---

## 1. Apache Tomcat Thread Pool Architecture

Tomcat operates as a JVM-based HTTP application server. It relies on an `Executor` (an implementation of Java's `ThreadPoolExecutor`) to handle concurrent web requests without incurring the overhead of creating and destroying threads for every incoming connection.

```
Incoming HTTP Requests
         │
         ▼
 ┌───────────────┐
 │ Acceptor Thread│
 └───────┬───────┘
         │
         ▼
  ┌─────────────┐       No Available Workers      ┌──────────────────┐
  │ Task Queue  │────────────────────────────────►│ OS Socket Queue  │
  └──────┬──────┘                                 │ (acceptCount)    │
         │ Picks up task                          └──────────────────┘
         ▼
┌────────────────────────────────────────────────────────────────────┐
│ Tomcat Thread Pool (maxThreads)                                    │
│                                                                    │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      │
│  │ Worker Thread│      │ Worker Thread│      │ Worker Thread│      │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘      │
└─────────┼─────────────────────┼─────────────────────┼──────────────┘
          │                     │                     │
          ▼                     ▼                     ▼
┌────────────────────────────────────────────────────────────────────┐
│ Servlet Container / Application Logic                              │
│ (Spring MVC / Controllers / Downstream HTTP Calls / I/O)            │
└────────────────────────────────────────────────────────────────────┘
```

### Request Flow
1. **Acceptor Thread:** A dedicated background thread listens on the server socket for incoming TCP connection requests.
2. **Task Queue:** Once accepted, the connection task is offered to Tomcat's internal work queue.
3. **Worker Threads:** A pooled worker thread (e.g., named `http-nio-8080-exec-*`) picks up the task from the queue, parses the HTTP headers, routes the request to the appropriate Servlet/Controller logic, and streams the response back to the client.
4. **Recycling:** Once the HTTP response completes, the worker thread is returned to the idle pool to process subsequent requests.

### Key Configuration Parameters
* **`maxThreads` (Default: 200):** The maximum number of concurrent worker threads Tomcat will create to process requests.
* **`minSpareThreads` (Default: 10–25):** The minimum number of idle worker threads Tomcat keeps alive at all times to handle sudden traffic bursts.
* **`acceptCount` (Default: 100):** The maximum queue length for incoming connection requests when all possible request processing threads (`maxThreads`) are busy. Once this queue fills, Tomcat rejects incoming TCP connections (`Connection Refused`).
* **`maxConnections`:** The maximum number of connections that the server accepts and processes at any given time (for NIO, typically `8192` or `10000`).

---

## 2. What Is Tomcat Thread Starvation?

**Tomcat thread starvation** occurs when all worker threads defined by `maxThreads` are simultaneously occupied, usually waiting on blocked I/O operations or lock contention. As a result, Tomcat has zero threads available to pick up new incoming tasks from the queue.

### Symptoms
* **High Latency & Gateway Timeouts:** Upstream proxies (like NGINX, API Gateways, or load balancers) start returning `504 Gateway Timeout` or `503 Service Unavailable`.
* **Low CPU Utilization:** Despite the service being unresponsive, host CPU usage may remain low (e.g., < 10%) because all worker threads are suspended in waiting or blocked states.
* **Connection Rejections:** Once the OS socket queue (`acceptCount`) fills up, new connection attempts fail immediately.

---

## 3. Causes of Tomcat Thread Starvation

In microservice architectures, Tomcat thread starvation is rarely caused by heavy CPU-bound computation. Instead, it is almost always caused by **blocking I/O operations** holding worker threads hostage:

```
Upstream Client
      │
      ▼
┌─────────────────────────────────────────┐
│ Tomcat Worker Thread (Occupied)         │
│                                         │
│   Calls Downstream REST Service B       │───► [ Slow / Unresponsive Service B ]
│   (Thread BLOCKS waiting for response)  │     (Holds thread indefinitely)
└─────────────────────────────────────────┘
```

1. **Unbounded Downstream HTTP / RPC Calls:** Making synchronous outbound HTTP calls without strict socket read timeouts. If a downstream microservice hangs, the Tomcat thread remains blocked waiting for data.
2. **Database Connection Pool Bottlenecks:** If all database pool connections are in use or slow to respond, Tomcat worker threads block while calling `getConnection()`, quickly depleting `maxThreads`.
3. **Unbounded Thread Wait / Sync Locks:** Application code waiting indefinitely on synchronized blocks, latches (`CountDownLatch`), or shared resource locks.

---

## 4. Investigating Tomcat Thread Starvation

### Step 1: Capture JVM Thread Dumps
Take multiple thread dumps spaced 5 to 10 seconds apart during the outage to identify what worker threads are waiting on.

```bash
# Obtain JVM Process ID
jps -l

# Take thread dump using jcmd
jcmd <pid> Thread.print > tomcat_dump_1.txt
```

### Step 2: Analyze Tomcat Worker Thread States
Search the thread dump for threads named `http-nio-<port>-exec-*`. 

* **State: `RUNNABLE` (Normal / Active Execution):**
  Thread is actively executing Java byte code or executing an OS socket operation.

* **State: `TIMED_WAITING` or `WAITING` (Potential Starvation Source):**
  The thread is blocked waiting for an external event, lock, or socket read.

#### Example: Starvation Due to Slow Outbound HTTP Call
```text
"http-nio-8080-exec-42" #112 daemon prio=5 os_prio=0 tid=0x00007f8a3c108000 nid=0x4d32 runnable [0x00007f8a143fe000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.read(SocketInputStream.java:150)
        ...
        at org.apache.http.impl.io.SessionInputBufferImpl.fillBuffer(SessionInputBufferImpl.java:153)
        at org.apache.http.impl.conn.DefaultManagedHttpClientConnection.receiveResponseHeader(DefaultManagedHttpClientConnection.java:163)
        ...
        at com.example.service.ExternalApiClient.getUserData(ExternalApiClient.java:45)
```
*Interpretation:* The thread is stuck in `socketRead0`, blocking Tomcat while waiting for a response from an external service.

#### Example: Starvation Due to DB Connection Pool Exhaustion
```text
"http-nio-8080-exec-15" #85 daemon prio=5 os_prio=0 tid=0x00007f8a3c004000 nid=0x4c21 waiting on condition [0x00007f8a152ff000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000076c123456> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:197)
        at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:162)
```
*Interpretation:* Tomcat threads are waiting for an available connection from the database pool.

### Step 3: Monitor Tomcat Thread Metrics
Expose Micrometer or JMX metrics to monitor thread pool saturation in real time:

* **`tomcat.threads.config.max`:** Total configured `maxThreads`.
* **`tomcat.threads.busy`:** Number of worker threads currently processing a task.
* **`tomcat.threads.current`:** Total threads currently spawned in the pool.

*Starvation Condition:* `tomcat.threads.busy` equals `tomcat.threads.config.max` continuously while request queue lengths spike.

---

## 5. Prevention Strategies

1. **Enforce Aggressive Timeouts:** Configure strict connection and read timeouts on all outbound HTTP clients, database drivers, and messaging tools (e.g., read timeout = 2000ms).
2. **Implement Circuit Breakers:** Use fault-tolerance libraries (e.g., Resilience4j) to automatically open circuit breakers and fail fast when downstream services slow down, protecting Tomcat worker threads from piling up.
3. **Use Asynchronous Processing / Reactive Models:** For non-blocking I/O operations, use asynchronous servlets (`DeferredResult`, `CompletableFuture`) or reactive frameworks (Spring WebFlux) to decouple request connections from OS worker threads.