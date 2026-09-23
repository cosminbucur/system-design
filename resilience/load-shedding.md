Load shedding is a service deliberately rejecting some incoming requests when it detects it's overloaded, so it can keep serving the rest well instead of degrading for everyone. It's a self-protection mechanism triggered by the service's own health signals (queue depth, latency, CPU, thread pool saturation) — not a fixed external quota. The alternative to shedding load on purpose is the service slowing down for every caller until it falls over entirely; shedding trades a controlled number of fast rejections for keeping the rest of the traffic healthy.

## 1. Load Shedding vs. Rate Limiting — Different Trigger, Different Job

Both reject requests, which makes them easy to confuse, but they decide *when* to reject based on completely different signals. See [rate/time limiter](./rate-time-limiter.md) for the rate limiter itself.

| | Rate Limiter | Load Shedder |
| --- | --- | --- |
| Trigger | A fixed quota per client/key/window, known in advance | The service's own real-time health (queue depth, latency, CPU, saturation) |
| Answers | "Has this caller used up its allowance?" | "Am I currently overloaded, regardless of who's asking?" |
| Applies per | Client/API key | The service as a whole |
| Still rejects under | Normal load, if a client exceeds its quota | Only when the service itself is actually struggling |

A rate limiter can reject a well-behaved client's request even when the service is perfectly healthy, just because that client hit its cap. A load shedder does the opposite: it ignores who's asking and only rejects once the service itself is in trouble. In practice they stack — rate limiting caps predictable per-client volume, load shedding is the last line of defense against unpredictable aggregate overload that gets through anyway.

## 2. What Triggers Shedding

```
Signal            Example threshold
------            -----------------
Queue depth       Reject new requests once the pending queue exceeds 500
Request latency    Shed if p99 latency crosses 2s (a sign the service can't keep up)
CPU / thread pool  Shed once thread pool utilization exceeds 90%
```

The signal should reflect actual strain, not a guess — a static threshold picked without load testing either sheds too early (wasting healthy capacity) or too late (the service is already falling over by the time it kicks in).

## 3. Prioritized Shedding

Naive shedding rejects requests indiscriminately once a threshold is crossed. A better approach sheds the least important traffic first, so critical functionality survives longer under overload:

```java
@RestControllerAdvice
public class LoadSheddingFilter implements Filter {

    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        if (systemOverloaded() && isLowPriority(req)) {
            ((HttpServletResponse) res).setStatus(503);
            return; // shed before doing any real work
        }
        chain.doFilter(req, res);
    }

    private boolean isLowPriority(ServletRequest req) {
        // e.g. analytics/reporting endpoints shed before checkout/payment endpoints
        return LOW_PRIORITY_PATHS.contains(((HttpServletRequest) req).getRequestURI());
    }
}
```

A checkout endpoint and a "recently viewed items" endpoint are not equally important — under overload, shedding the second to protect the first is a deliberate product decision, not an accident of whichever request happened to arrive last.

## 4. Where It Belongs

```
Client → API Gateway (rate limit per client) → Service (load shed based on its own health)
```

Rate limiting at the edge handles the predictable case (a known client sending too much). Load shedding inside the service handles the unpredictable case: traffic that's individually within quota but collectively still more than the service can currently handle — a traffic spike, a slow dependency backing up the queue, or a partial outage reducing effective capacity.

## 5. Best Practices

| Practice | Recommendation |
| --- | --- |
| Shed before doing expensive work, not after | Reject at the front door (a filter/gateway) — shedding after the request already touched the DB wastes the exact capacity you're trying to protect |
| Prioritize by business importance, not arrival order | Not all endpoints matter equally under overload — decide in advance what's expendable |
| Base the trigger on real health signals, load-tested | A threshold picked without testing either wastes capacity or reacts too late |
| Return a clear, fast rejection (e.g. `503`) | The caller should fail fast and know to back off, not hang waiting on a response that was never coming |
| Combine with rate limiting and bulkheads, don't replace them | Each protects against a different shape of overload — predictable per-client volume vs. unpredictable aggregate strain |
