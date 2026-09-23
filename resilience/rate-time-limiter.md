Rate limiting, time limiting, and bulkheading solve related failure modes that all show up as "a dependency is being asked to do more than it should" — a rate limiter caps *how many* calls are allowed in a given window, a time limiter caps *how long* any single call is allowed to take, and a bulkhead caps *how much concurrent capacity* any one dependency can consume. All three exist to protect a service (or its caller) from being overwhelmed, but they intervene on different axes: volume, duration, and concurrency.

## 1. Rate Limiter — Capping Call Volume

A rate limiter restricts the number of calls permitted within a time window, rejecting or delaying anything beyond that cap. It can be applied on either side of a call: a service protecting itself from too many incoming requests, or a caller deliberately throttling itself against a downstream dependency's known capacity.

```java
@Service
public class InventoryClientService {

    @RateLimiter(name = "inventoryService", fallbackMethod = "rateLimitFallback")
    public StockLevel getStock(String sku) {
        return inventoryClient.getStock(sku);
    }

    private StockLevel rateLimitFallback(String sku, Throwable ex) {
        throw new TooManyRequestsException("Inventory lookups throttled, try again shortly");
    }
}
```

```yaml
resilience4j.ratelimiter:
  instances:
    inventoryService:
      limit-for-period: 50       # allow at most 50 calls...
      limit-refresh-period: 1s   # ...per 1-second window
      timeout-duration: 0        # don't wait for a permit — reject immediately if none available
```

At the API boundary, the same idea is expressed as an HTTP-level contract rather than an internal library setting:

```java
@GetMapping("/api/orders")
public ResponseEntity<List<Order>> getOrders() {
    if (!rateLimiter.tryAcquire(clientId)) {
        return ResponseEntity.status(429) // Too Many Requests
            .header("Retry-After", "5")
            .build();
    }
    return ResponseEntity.ok(orderService.findAll());
}
```

`429 Too Many Requests` with a `Retry-After` header tells a well-behaved caller exactly how long to back off, rather than retrying immediately and making the overload worse.

## 2. Rate Limiting Algorithms

| Algorithm | Behavior | Tradeoff |
| --- | --- | --- |
| Fixed window | Count requests in a fixed time bucket (e.g., per calendar second); reset the count each new window | Simple, but allows a burst of 2x the limit right across a window boundary |
| Sliding window | Count requests over a rolling window ending "now," not a fixed calendar boundary | Smooths out the boundary-burst problem, slightly more bookkeeping |
| Token bucket | Tokens refill at a steady rate into a bucket of fixed capacity; each request consumes one token | Naturally allows short bursts up to the bucket size while still capping sustained rate |
| Leaky bucket | Requests queue and are processed at a strictly constant output rate | Smooths bursts into a steady rate, at the cost of added latency for queued requests |

Token bucket is the most common default in practice — it tolerates brief, legitimate bursts (a user double-clicking, a batch of related calls) without abandoning an overall sustained-rate cap.

## 3. Time Limiter — Capping Call Duration

A time limiter bounds how long a single call is allowed to run before it's treated as failed, regardless of whether it would have eventually succeeded. This matters most for asynchronous calls where there's no natural thread-blocking timeout already in place.

```java
@Service
public class InventoryClientService {

    @TimeLimiter(name = "inventoryService", fallbackMethod = "timeoutFallback")
    public CompletableFuture<StockLevel> getStockAsync(String sku) {
        return CompletableFuture.supplyAsync(() -> inventoryClient.getStock(sku));
    }

    private CompletableFuture<StockLevel> timeoutFallback(String sku, Throwable ex) {
        return CompletableFuture.completedFuture(StockLevel.unknown(sku));
    }
}
```

```yaml
resilience4j.timelimiter:
  instances:
    inventoryService:
      timeout-duration: 2s   # cancel and fail if the call hasn't completed within 2 seconds
      cancel-running-future: true
```

Without an explicit time limiter, an async call has no upper bound at all — it completes whenever the underlying dependency gets around to responding, which means a slow dependency can leave callers waiting indefinitely even though nothing ever technically "failed."

## 4. Why Both Matter Together

A rate limiter without a time limiter still lets individual calls hang indefinitely — capping *how many* calls happen doesn't help if each one that does get through can take forever. A time limiter without a rate limiter still lets a dependency be overwhelmed by sheer volume, even if no single call runs long — each one just fails fast individually while the flood continues. Used together, they cap both axes: no more than N calls per window, and no call allowed to run past a fixed bound.

## 5. Where Each One Belongs in the Chain

```
Client → API Gateway (rate limit per client/API key, reject with 429)
            → Service A (time limit on its own outbound calls to Service B)
                → Service B (rate limit on its own incoming request volume, protecting itself)
```

Rate limiting commonly appears at the edge (protecting the whole system from a noisy client) and again at each individual service (protecting that specific service from any caller, internal or external). Time limiting is applied per outbound call, wherever a service depends synchronously or asynchronously on something that could be slow.

## 6. Bulkhead — Capping Concurrent Capacity Per Dependency

A bulkhead isolates the resources (threads, connection pool slots) used to call one dependency from the resources used to call every other dependency, so that one slow or failing dependency can't exhaust capacity that unrelated calls also need. It's named after a ship's watertight compartments — a hull breach floods one compartment, not the whole ship.

```java
@Service
public class InventoryClientService {

    @Bulkhead(name = "inventoryService", fallbackMethod = "fallbackStock")
    public StockLevel getStock(String sku) {
        return inventoryClient.getStock(sku);
    }

    private StockLevel fallbackStock(String sku, Throwable ex) {
        return StockLevel.unknown(sku);
    }
}
```

```yaml
resilience4j.bulkhead:
  instances:
    inventoryService:
      max-concurrent-calls: 10
    paymentService:
      max-concurrent-calls: 20
```

Without this isolation, a shared thread pool serving calls to both `InventoryService` and `PaymentService` lets a slow `InventoryService` consume every available thread — starving calls to the completely healthy `PaymentService` simply because they happened to share a pool. Once each dependency has its own bounded slice of capacity, `InventoryService` can only ever exhaust its own 10 slots, leaving `PaymentService`'s 20 fully available regardless of what's happening to inventory calls.

### Two Bulkhead Implementations

| Type | How it isolates | Cost |
| --- | --- | --- |
| Semaphore bulkhead | A counting semaphore limits concurrent calls on the *calling* thread itself | Cheap, no extra threads, but a blocked call still ties up a caller thread until it returns |
| Thread pool bulkhead | Calls run on a dedicated, separate thread pool per dependency | Fully isolates the caller's own threads from a slow dependency, at the cost of context-switching and managing more thread pools |

A thread pool bulkhead gives the strongest isolation — the calling thread itself is freed immediately and never blocks on the dependency at all — but it's heavier to run than a semaphore bulkhead, which is often sufficient when the caller can tolerate briefly waiting on its own thread.

## 7. How All Three Fit Together

Rate limiter, time limiter, and bulkhead each cap a different axis of the same underlying risk, and normally stack rather than substitute for one another:

```
Rate limiter  → caps HOW MANY calls are attempted at all
Time limiter  → caps HOW LONG any single call is allowed to run
Bulkhead      → caps HOW MUCH CONCURRENT capacity any one dependency can consume
```

A rate limiter without a bulkhead still lets the calls it does permit pile up on a shared pool and starve unrelated dependencies. A bulkhead without a rate limiter still lets a dependency's dedicated pool fill up entirely under enough concurrent load, even if no single caller is misbehaving. Layered together with a circuit breaker (which stops calling something clearly broken) and retry/fallback (which decide what to do about individual failures), these form the standard resilience toolkit for any synchronous call to another service — a healthy system under real load typically needs all of them, each addressing a distinct way a dependency call can go wrong.

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Set rate limits based on the dependency's actual known capacity | A limit that's just a guess either throttles legitimate traffic or fails to protect anything. |
| Prefer token bucket over fixed window for bursty, legitimate traffic | Fixed windows allow a boundary-crossing burst of up to 2x the intended limit. |
| Return `429` + `Retry-After` at the API boundary, not a silent failure | Gives a well-behaved caller the information needed to back off correctly instead of retrying immediately. |
| Always pair a time limiter with async calls | Without one, a slow dependency can leave a caller waiting indefinitely with no natural bound. |
| Give each downstream dependency its own bulkhead | A shared pool lets one slow dependency starve calls to every other dependency sharing it. |
| Use a thread pool bulkhead when the caller can't tolerate blocking at all | A semaphore bulkhead is cheaper but still ties up the caller's own thread until the call returns. |
| Apply rate limiting at both the edge and per-service | Protects the whole system from a noisy client at the gateway, and each service from any caller individually. |
| Don't rely on any single mechanism alone | Rate limiter, time limiter, bulkhead, circuit breaker, and retry/fallback each address a distinct failure mode — combine them. |
| Make limiter and bulkhead thresholds configurable, not hardcoded | Real-world capacity changes over time — a limit baked into code requires a redeploy to adjust under a real incident. |
