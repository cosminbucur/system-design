# API Performance Troubleshooting Guide

When a customer reports that an API is running slow, follow this structured incident investigation runbook to systematically narrow down the root cause—from the network edge to downstream database queries.

---

## Workflow Overview

```
[Customer Complaint] 
       │
       ▼
1. Scope & Triage ──────────────► Isolate: User, Endpoint, Region, Latency Tier
       │
       ▼
2. Ingress & Perimeter ─────────► Check: DNS, CDN Hits, Gateway Rate Limits, TTFB
       │
       ▼
3. Application & Microservices ─► Check: OpenTelemetry Spans, CPU/RAM, Thread Starvation
       │
       ▼
4. Data & Dependencies ─────────► Check: Slow Queries, Lock Contention, Pool Limits, External APIs
       │
       ▼
5. Resolution & Reporting ──────► Verify $p_{95}$/$p_{99}$ Recovery, Send Customer Post-Mortem
```

---

## Phase 1: Scope & Triage

Before diving into application code, isolate whether the latency issue is systemic or localized to prevent chasing false positives.

* **Isolate Scope:**
  * Check if slowness affects all endpoints or a specific route (e.g., `POST /orders` vs `GET /health`).
  * Determine if the issue impacts all clients or a specific user, region, device type, or authentication tier.
* **Review High-Level Metrics:**
  * Inspect API Gateway and Load Balancer metrics.
  * Check latency distributions across $p_{50}$, $p_{95}$, and $p_{99}$ percentiles.
  * Look for correlated spikes in error rates (e.g., HTTP `504 Gateway Timeout` or `503 Service Unavailable`) and overall traffic volume.

---

## Phase 2: Ingress & Perimeter Layer

Investigate potential network or edge bottlenecks before checking internal application logic.

* **DNS & CDN / Edge Cache:**
  * Verify DNS resolution times and check CDN cache hit vs. miss ratios to ensure static or cacheable responses are not unexpectedly bypassing the cache layer.
* **Rate Limiting & Throttling:**
  * Confirm if API gateway rate limits, concurrent connection caps, or DDoS mitigations are queuing requests at the perimeter.
* **Network & TLS Latency:**
  * Compare Time to First Byte (TTFB) against total request duration to distinguish between network transport delays and backend execution delays.

---

## Phase 3: Application & Microservices Layer

Trace the request execution lifecycle across internal microservice boundaries.

* **Distributed Tracing:**
  * Filter trace telemetry (e.g., OpenTelemetry, Jaeger, Datadog) using the affected `customer_id`, `trace_id`, or route path.
  * Identify specific service spans contributing the largest proportion of elapsed duration.
* **Resource Saturation:**
  * Inspect host and container metrics for CPU throttling, memory leaks, thread pool starvation, and Garbage Collection (GC) pauses.
* **Queue & Worker Backlogs:**
  * For asynchronous processing workloads, audit job queue depths and consumer processing lag to spot background execution bottlenecks.

---

## Phase 4: Data & External Dependency Layer

Investigate downstream datastores and third-party integrations that could block execution.

* **Database Query Performance:**
  * Review slow query logs for unindexed lookups, expensive `JOIN` operations, or full table scans triggered by specific query payloads.
* **Locking & Connection Pools:**
  * Audit database connection pool utilization for starvation or row/table-level lock contention blocking concurrent operations.
* **Cache Layer Health:**
  * Check Redis/Memcached hit rates, memory eviction events, and key lookup latencies.
* **Third-Party Service Calls:**
  * Audit outbound HTTP/gRPC calls to external vendor services to verify that unhandled timeouts or slow external responses are not blocking main execution threads.

---

## Phase 5: Verification & Remediation

Validate the resolution and document findings for internal teams and the customer.

1. **Apply Remediation:** Deploy query indexing, resource scaling, connection pool adjustments, or circuit breakers as appropriate.
2. **Verify Performance:** Run synthetic load tests or monitor canary deployments to confirm $p_{95}$ and $p_{99}$ latency metrics return to baseline levels.
3. **Customer Communication:** Provide a transparent status report detailing the identified root cause, short-term fix, and long-term preventive actions.