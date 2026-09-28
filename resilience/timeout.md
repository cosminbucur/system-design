# Timeout Pattern in Microservices Architecture

The **Timeout Pattern** is a core stability and resilience pattern used in microservice architectures to prevent system-wide failures when a downstream service invocation takes longer than expected. 

Rather than waiting indefinitely for a slow, unresponsive, or hung network call, the caller configures a maximum waiting threshold. If the downstream dependency fails to respond within that timeframe, the caller aborts the request, frees up its allocated system resources, and executes a fallback strategy.

---

## 1. The Problem: Cascading Failures and Thread Starvation

In a distributed microservice system, services make network calls to one another (e.g., via HTTP/REST, gRPC, or GraphQL). When Service A invokes Service B, it typically consumes a worker thread or connection from its resource pool to handle the wait.

If Service B experiences high latency, unindexed database queries, or a network deadlock *without* dropping the connection:

1. Service A's worker thread remains blocked waiting for a response.
2. New incoming requests continue to arrive at Service A, consuming additional threads.
3. Service A exhausts its available thread pool or memory capacity (**Thread Starvation**).
4. Service A stops accepting new incoming traffic and fails, causing any upstream services depending on Service A to fail as well (**Cascading Failure**).

```
[ Client ] ---> [ Service A ] ---> [ Service B ] ---> [ Unresponsive DB / External API ]
                    |
              (Threads blocked
              waiting for B...)
                    |
            [ Thread Pool Exhausted ] 🔴 Service A Crashes
```

---

## 2. How the Timeout Pattern Works

When a service call is wrapped in a timeout mechanism, a timer runs in parallel with the request context:

```
+-----------------------------------------------------------------------+
|                              Service A                                |
|                                                                       |
|  [Initiate Request] ---> ( Start Timer: e.g., 500ms )                 |
|                                 |                                     |
|              +------------------+------------------+                  |
|              |                                     |                  |
|      (Response < 500ms)                    (Elapsed Time >= 500ms)   |
|              |                                     |                  |
|              v                                     v                  |
|       Cancel Timer                         Abort Network Call         |
|      Return Success                        Throw TimeoutException     |
|                                            Execute Fallback           |
+-----------------------------------------------------------------------+
```

1. **Successful Path:** Service B responds within the threshold (e.g., 250 ms). The timer is cancelled, and Service A processes the payload normally.
2. **Timeout Path:** Service B does not respond within the threshold (e.g., 500 ms). Service A severs the caller-side connection, throws a `TimeoutException`, releases its thread back to the pool, and routes execution to a fallback mechanism.

---

## 3. Key Design Considerations

### Dynamic vs. Static Thresholds
Setting timeouts too high (e.g., standard HTTP defaults of 30–60 seconds) does not prevent thread starvation under heavy load. Setting them too low risks premature failures during minor network spikes.
* **Metric-Driven Thresholds:** Base the duration on operational telemetry (e.g., the $P_{99}$ latency of the downstream service plus a small safety margin).
* **SLA/SLO Alignment:** Choose a timeout duration that respects the caller's overall Service Level Agreement.

### Non-Blocking/Asynchronous I/O
When using asynchronous or non-blocking frameworks (e.g., Netty, Node.js, Project Reactor), a timeout releases event-loop registrations rather than OS-level threads, enabling high concurrency even under degraded network conditions.

### Idempotency Issues
A timeout only cancels the request **on the client/caller side**. Service B may still receive and process the request asynchronously.
* If the request is a mutating operation (e.g., `POST /payments`), state inconsistencies can occur.
* Ensure downstream operations are **idempotent** or use unique **idempotency keys** to prevent duplicate execution when retries occur after a timeout.

---

## 4. Multi-Pattern Integration

The Timeout pattern is rarely used in isolation; it works best alongside other resilience patterns:

* **Circuit Breaker:** If consecutive requests fail due to timeouts, open the circuit breaker to fail fast immediately without making subsequent network calls.
* **Retry Pattern:** Retrying immediately after a timeout can worsen a downstream service's load. Always pair retries after timeouts with **exponential backoff** and **jitter**.
* **Bulkhead Pattern:** Isolate thread pools for different downstream dependencies so a timing-out service cannot consume all threads allocated to other healthy services.

---

## 5. Implementation Examples

### Java (Resilience4j + Spring WebClient)
```java
TimeLimiterConfig config = TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofMillis(500))
    .cancelRunningFuture(true)
    .build();

TimeLimiter timeLimiter = TimeLimiter.of("userServiceLimiter", config);

Supplier<CompletionStage<User>> restrictedCall = TimeLimiter
    .decorateCompletionStage(timeLimiter, () -> fetchUserAsync(userId));

CompletableFuture<User> result = restrictedCall.get()
    .toCompletableFuture()
    .exceptionally(throwable -> getFallbackUser(userId));
```

### Go (`context.WithTimeout`)
```go
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

req, err := http.NewRequestWithContext(ctx, "GET", "http://service-b/api/resource", nil)
if err != nil {
    return err
}

resp, err := client.Do(req)
if err != nil {
    if errors.Is(ctx.Err(), context.DeadlineExceeded) {
        // Handle timeout & fallback logic
    }
    return err
}
```

### Infrastructure Level (Service Mesh - Istio)
Timeouts can also be managed declaratively outside application code using sidecar proxies:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: service-b-route
spec:
  hosts:
  - service-b
  http:
  - route:
    - destination:
        host: service-b
    timeout: 0.5s
```