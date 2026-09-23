# Understanding HikariCP & Investigating Thread Starvation in PostgreSQL Microservices

## 1. What is HikariCP?

**HikariCP** is a high-performance JDBC connection pool library for Java applications. Instead of incurring the massive overhead of establishing a new TCP connection, performing TLS handshakes, and authenticating with PostgreSQL on every query, HikariCP maintains a reusable pool of active, warm database connections.

```
+-----------------------------------------------------------------------+
|                         Spring Boot Application                       |
|                                                                       |
|  [ Worker Thread 1 ] \                                                |
|  [ Worker Thread 2 ] -- (Get Connection) --> [ HikariCP Connection ]  |
|  [ Worker Thread 3 ] /                            |  Pool            |
+---------------------------------------------------|-------------------+
                                                    | (Reused TCP Sockets)
                                                    v
                                         +---------------------+
                                         | PostgreSQL Database |
                                         +---------------------+
```

---

## 2. How HikariCP Works Under the Hood

When an application thread executes a database operation, it requests a connection using `dataSource.getConnection()`. HikariCP processes this request through a multi-tiered lookup:

1. **`FastList` (Thread-Local Optimization):** HikariCP tracks connections recently used by the calling thread. If a thread repeatedly executes queries, it retrieves its previously assigned connection with zero lock contention.
2. **`ConcurrentBag` (Lock-Free Handoff):** If thread-local lookup fails, HikariCP checks a lock-free data structure. If another thread finishes a query at that moment, HikariCP hands off the connection directly without thread queuing overhead.
3. **Blocking Wait:** If all connections in the pool are checked out, the requesting thread blocks and waits up to `connectionTimeout` (default: `30000` ms / 30 seconds).

---

## 3. What Causes Thread Starvation?

Thread starvation occurs when all incoming web request threads (e.g., Tomcat HTTP worker threads) become blocked waiting for resources—specifically HikariCP connections—causing the application to stop responding to new traffic.

```
Incoming Requests ---> [ HTTP Worker Pool ] (e.g., 200 Threads)
                             |
                             | (All threads block waiting for DB connection)
                             v
                       [ HikariCP Pool ] (e.g., Max 10 Connections)
                             |
         +-------------------+-------------------+
         |                                       |
         v                                       v
[ Slow Query / Lock ]                [ External Network Call ]
(Missing Index / Row Lock)           (I/O Block Inside Transaction)
```

### Key Triggers in Microservices with PostgreSQL

#### A. Connection Pool Sizing Misconceptions
Configuring `maximumPoolSize` to match the HTTP thread pool (e.g., setting HikariCP pool size to 200) is counterproductive. 

PostgreSQL processes each connection using a dedicated process. Oversubscribing connections causes aggressive CPU context switching on the database server.

> **Recommended Sizing Formula (PostgreSQL/HikariCP):**
> $$\text{Connections} = (\text{CPU Cores} \times 2) + \text{Effective Spindle Count}$$
> For a typical 4-vCPU database server, **10 to 15 connections total across all instances** often yields optimal throughput.

#### B. The "Database-over-HTTP" Anti-Pattern
Wrapping external REST/gRPC calls inside a `@Transactional` block holds database connections idle:

1. Thread acquires a HikariCP connection.
2. Thread begins a PostgreSQL transaction (`BEGIN`).
3. Thread initiates a synchronous HTTP request to another service (taking 500ms–3000ms).
4. The database connection remains checked out and completely idle.

Under high load, all pooled connections become occupied by threads waiting for external network responses.

#### C. PostgreSQL Lock Contention and Slow Queries
When queries slow down (due to unindexed table scans or `SELECT ... FOR UPDATE` row locks), connection checkout times increase. If average query latency increases from **2ms to 2000ms**, a pool of 10 connections drops from processing **5,000 queries/sec** to **5 queries/sec**.

---

## 4. Diagnostic & Investigation Workflow

### Step 1: Monitor HikariCP Metrics
Track these key Prometheus / Micrometer metrics during performance degradation:

* `hikaricp.connections.pending`: Number of threads waiting for a connection (**Alert if > 0**).
* `hikaricp.connections.active`: Currently checked-out connections.
* `hikaricp.connections.acquire`: Time spent waiting to acquire a connection.
* `hikaricp.connections.usage`: Total duration connections are held by application threads.

### Step 2: Capture Application Thread Dumps
Generate thread dumps during an active degradation window:

```bash
jcmd <PID> Thread.print > thread_dump.txt
```

Inspect thread dumps for state patterns:
* Search for threads in `WAITING` or `TIMED_WAITING`.
* Look for stack traces blocked at `com.zaxxer.hikari.pool.HikariPool.getConnection()`.
* Identify what code paths hold checked-out connections without releasing them promptly.

### Step 3: Inspect Active PostgreSQL Sessions
Execute the following query directly against PostgreSQL to analyze active backend sessions:

```sql
SELECT 
    pid, 
    now() - query_start AS duration, 
    state, 
    wait_event_type, 
    wait_event, 
    query 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY duration DESC;
```

#### Diagnostic Indicators:
* **`state = 'idle in transaction'`**: Application checked out a connection and started a transaction, but is stalled waiting for non-database work (e.g., network I/O or heavy computation).
* **`wait_event_type = 'Lock'`**: Queries are blocked waiting on table or row locks held by other transactions.
* **High duration with active queries**: Missing indexes or unoptimized query execution plans.

---

## 5. Prevention Checklist

1. **Keep Connection Pools Lean:** Keep `maximumPoolSize` around 10 per application instance. Scale horizontally by adding application replicas rather than increasing pool size.
2. **Fail Fast:** Reduce `connectionTimeout` from 30,000 ms to `2000ms - 3000ms`. Returning a 503 error quickly prevents worker thread pool exhaustion.
3. **Isolate Transactions:** Remove network calls, file reads, and CPU-heavy operations from `@Transactional` blocks.
4. **Enforce Socket Timeouts:** Configure network timeouts to prevent threads from hanging indefinitely on stalled connections:
   ```properties
   spring.datasource.hikari.data-source-properties.socketTimeout=5000
   ```