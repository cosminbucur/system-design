Traces are observability's third pillar — the path a single request took across services, answering "what path did this one request take, and where did it spend time?" Where a log explains one event and a metric shows a trend, a trace reconstructs the actual causal chain: this request called that service, which called this other one, and here's exactly how long each hop took.

## 1. From Correlation ID to Distributed Trace

A correlation ID (covered in logs) ties log lines together within one service; a trace extends the same idea across service boundaries, using a `traceId` that's automatically propagated in outbound HTTP/messaging headers. This is what actually lets you see the full request path.

```java
// Spring Boot + Micrometer Tracing (OpenTelemetry bridge) — mostly automatic instrumentation
// Each service's logs automatically include the same traceId via MDC, and each hop becomes a "span"
```

Key vocabulary:

- **Trace**: the entire journey of one request across all services it touched.
- **Span**: one unit of work within that trace (e.g., "call InventoryService," "run this SQL query") — has a start time, duration, and parent span.
- **Span context propagation**: the trace/span IDs are passed along in request headers (`traceparent` in the W3C Trace Context standard) so each service's spans link back into the same trace.

```java
@Bean
public ObservationHandler<Observation.Context> tracingHandler() {
    // Micrometer Observation API wraps a unit of work with both a metric AND a trace span from one instrumentation point
    return new DefaultTracingObservationHandler(tracer);
}
```

Tools: OpenTelemetry (vendor-neutral instrumentation/collection standard — the safe default to instrument against), Jaeger/Zipkin/Tempo (trace storage + visualization UI).

## 2. Core Concepts

| Concept | What it is |
| --- | --- |
| Trace | The entire journey of one request across every service it touched — identified by a single `traceId` shared by every span in it. |
| Span | One unit of work within that trace (e.g., "call InventoryService," "run this SQL query"). Has a name, start time, duration, its own `spanId`, and a reference to its parent span. |
| Span tree | Spans nest — a parent span (e.g., the inbound HTTP request) has child spans (e.g., each downstream call it makes), forming a tree whose shape mirrors the actual call graph. |
| Span context | The minimal set of IDs needed to link a new span back into the right trace: `traceId` + the current `spanId` (as its parent) + trace flags (e.g., "sampled or not"). |
| Context propagation | How the span context travels between services — carried in outbound request headers so the receiving service can create its own child span under the same trace instead of starting a brand new, disconnected one. |
| Span kind | Marks what role a span plays: `CLIENT` (making an outbound call), `SERVER` (handling an inbound one), `PRODUCER`/`CONSUMER` (async messaging), or `INTERNAL` (in-process work with no network hop). This is what lets tooling distinguish "waiting on the network" from "doing local work." |
| Span attributes/events | Key-value metadata attached to a span (e.g., `http.status_code=500`, `sku=SKU-42`) plus timestamped events within it (e.g., "cache miss at t+15ms") — this is what turns a generic waterfall bar into something you can actually debug from. |
| Baggage | Arbitrary key-value context propagated alongside the trace itself (not just IDs) — e.g., a `tenantId` that every service along the path can read, even ones that don't care about tracing specifically. Unlike span attributes, baggage travels *forward* with the request, not just attached to one span. |

Context propagation is the one piece that makes all the others possible: without the trace/span IDs actually crossing the service boundary in the request itself, each service would just record its own isolated, disconnected spans — same request, but no way to stitch them back into one trace.

## 3. Reading a Trace

A trace's real value is visual: seeing the full waterfall of spans immediately shows which hop is the bottleneck, without guessing from logs scattered across several services.

```
Request → OrderService (12ms)
              └─> InventoryService (340ms)   ← the bottleneck, immediately obvious
              └─> PaymentService (8ms)
```

This is the typical observability workflow in practice: a metric/alert tells you *something* is wrong (error rate or latency spiked); a trace tells you *where* in the call chain it's slow or failing; logs (filtered by that trace's ID) tell you *why* — the actual exception, the specific input that triggered it.

## 4. Sampling — You Can't Trace Everything

At high request volume, recording a full trace for every single request is prohibitively expensive to store and process. Sampling records only a subset, and *when* that decision gets made matters as much as the rate:

| Strategy | When the decision happens | Tradeoff |
| --- | --- | --- |
| Head-based sampling | Upfront, at the very first span — before anyone knows how the request will turn out | Cheap and simple, but decides to keep or drop a trace before knowing it was actually the slow/failed one worth keeping |
| Tail-based sampling | After the full trace completes, once its outcome (error, latency) is known | Needs to buffer all spans until the trace finishes before deciding, which costs more to run, but keeps exactly the traces worth debugging |

```yaml
management:
  tracing:
    sampling:
      probability: 0.1 # head-based: trace ~10% of requests, decided upfront
```

Better than a flat percentage: **always keep errors and slow requests** via tail-based sampling, since those are exactly the traces you need when debugging — sampling uniformly at random means your most useful traces are the ones most likely to get discarded.

## 5. Best Practices

| Practice | Recommendation |
| --- | --- |
| Propagate trace context on every outbound call | A single hop that drops the `traceparent` header breaks the chain — the rest of the request's journey becomes invisible. |
| Instrument with OpenTelemetry, not a vendor-specific SDK | Keeps you portable across trace backends (Jaeger, Tempo, vendor APM tools) later. |
| Use tail-based sampling — always keep errors and slow requests | Uniform random sampling discards exactly the traces most useful for debugging, since failures are rare relative to normal traffic. |
| Treat a trace as the map, logs as the detail | Use the trace to find *which* span is the bottleneck or failure point, then pull that span's `traceId`-filtered logs for the actual root cause. |
| Name spans meaningfully, not just by class/method | A span named `call InventoryService: reserve stock` is far more useful in a waterfall view than a generic method name. |
| Don't rely on tracing alone for aggregate trends | Even with 100% sampling, a metric's cheap aggregation answers "is this getting worse over time" far more efficiently than scanning individual traces. |
