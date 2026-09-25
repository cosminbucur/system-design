Distributed logging is the practice of collecting logs emitted by many independent services/instances and centralizing them into one searchable store, instead of leaving each service's logs sitting only on its own disk. In a monolith, "grep the log file" works because there's one file. Across dozens of service instances that scale up/down and get rescheduled onto different hosts, the logs themselves are ephemeral and scattered — centralizing them is what makes it possible to answer "what happened for this one request?" at all.

## 1. Why a Single Service's Logs Aren't Enough

```
OrderService (3 pods) → InventoryService (5 pods) → PaymentService (2 pods)
```

A single user request can touch several pods across several services, each writing to its own local, rotating log file. Without centralization, answering "why did this request fail?" means knowing which specific pod handled it, at what time, before its logs get rotated away or the pod itself is gone. Centralizing removes that dependency on any one host still existing.

## 2. The Standard Pipeline

```
Service instance (writes structured JSON logs to stdout)
        │
        ▼
Log shipper / agent (Filebeat, Fluent Bit, Promtail, OTel Collector)
        │  tails log files or receives them via stdout, forwards over the network
        ▼
Central aggregator / storage (Elasticsearch, Loki, Splunk, CloudWatch Logs)
        │
        ▼
Query / visualization (Kibana, Grafana)
```

The shipper is the piece that makes this work without changing application code: it runs alongside each service (as a sidecar, or once per node as a DaemonSet in Kubernetes) and forwards whatever the service already writes to stdout/a log file, so the service itself stays unaware that its logs are being centralized at all.

| Stack | Shipper | Storage | Query UI |
| --- | --- | --- | --- |
| ELK / EFK | Filebeat / Fluentd | Elasticsearch | Kibana |
| Grafana Loki | Promtail | Loki (indexes labels, not full text) | Grafana |
| OpenTelemetry-native | OTel Collector | Any OTLP-compatible backend | Vendor-dependent |
| Cloud-managed | Cloud agent | CloudWatch Logs / Cloud Logging | Cloud console |

Loki's design is worth knowing explicitly: unlike Elasticsearch, it doesn't index the full log content, only a small set of labels (e.g. `service`, `pod`) — much cheaper to run at scale, at the cost of slower free-text search across large volumes.

## 3. Correlation ID — What Ties One Request's Logs Together

Centralizing logs only gets you "everything, searchable in one place" — it doesn't by itself let you isolate *one request's* logs out of millions of unrelated lines from other requests hitting other pods concurrently at the same time. The fix is a correlation ID: a single identifier generated once, at the moment a request first enters the system, and then carried along as it moves through every service in the call chain.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId); // included in every log line this thread writes
        try {
            response.setHeader("X-Correlation-Id", correlationId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // thread pools reuse threads — always clear
        }
    }
}
```

Two things have to happen for it to actually work end-to-end:

1. **Every service logs it as a structured field**, not just the originating one — a request that enters at the gateway and fans out to five downstream services needs all five writing the same `correlationId` value into their own log output.
2. **Every service propagates it to the next hop**, typically as an HTTP header (`X-Correlation-Id`) — a service that receives the header must forward it on its own outbound calls, or the chain breaks at that hop and everything downstream of it becomes uncorrelated.

Once logs are centralized, one query reconstructs the full sequence across every service and pod the request touched, without ever knowing in advance which pods those were:

```
query: correlationId="abc-123"
  → OrderService-pod-7:     "received order request"
  → InventoryService-pod-2: "reserving stock"
  → InventoryService-pod-2: "stock reserved"
  → OrderService-pod-7:     "order confirmed"
```

A dropped correlation ID at any single hop is a silent failure — logs before and after that hop still centralize fine, they just can no longer be tied together, and the gap is easy to miss until someone actually needs that request's full trail during an incident.

## 4. Structured Logging Is a Prerequisite

Structured logging (key-value fields, usually JSON, rather than free text) is what makes centralized logs actually queryable. Free-text log lines can still be shipped and stored, but grepping unindexed prose across millions of aggregated lines from many services is exactly the problem structured fields — and the correlation ID field specifically — exist to avoid.

```json
{
  "timestamp": "2026-09-19T10:30:00Z",
  "level": "INFO",
  "service": "InventoryService",
  "correlationId": "abc-123",
  "event": "stock_reserved",
  "sku": "SKU-42"
}
```

## 5. Cost and Volume Tradeoffs

Centralized log storage is not free, and volume grows with every service instance, so a few controls matter in practice:

| Control | Why |
| --- | --- |
| Drop/filter `DEBUG` before shipping, not after storing | Cheaper to filter at the shipper than to pay to index and store logs you'll never query |
| Sample or aggregate very high-volume, low-value logs | Some log lines (e.g. per-request access logs) are useful in aggregate but not worth retaining individually at full volume forever |
| Set retention per log importance | `ERROR`/audit logs often need months; `DEBUG`/access logs may only need days |
| Watch cardinality of indexed labels/fields | A label with unbounded values (e.g. indexing by raw `userId`) can blow up index size and cost, especially in label-indexed stores like Loki |

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Log to stdout, let the platform/shipper handle transport | Don't have application code manage its own network log shipping — that couples business logic to logging infrastructure and adds a failure mode inside the request path |
| Generate the correlation ID once, at the system's entry point | Every downstream service should receive and propagate it, not generate its own — otherwise the chain fragments into disconnected IDs |
| Propagate the correlation ID on every outbound call | A single hop that forgets to forward the header breaks correlation for everything downstream of it |
| Keep logs structured (JSON), not free text | Centralization multiplies the cost of unstructured logs — what's merely inconvenient to grep in one file becomes expensive to query at aggregate scale |
| Filter noise as early in the pipeline as possible | Every log line shipped and stored costs money; filter at the shipper/source, not after ingestion |
| Never centralize secrets/PII | Centralized storage is broadly accessible to many engineers/tools — a leak here is not confined to one host's disk anymore |
| Alert from the aggregated store, not from tailing individual pods | Individual pods are ephemeral and don't represent the system's overall health on their own |
