Load shedding is a defensive resiliency pattern where a service intentionally rejects or drops incoming requests once it reaches its capacity limits. Rather than attempting to process every request and risking resource exhaustion, cascading timeouts, or a full crash, the system "fails fast" to protect its core availability — deliberately sacrificing some traffic to keep serving the rest well.

## 1. Why It's Needed: The Overload Death Spiral

When incoming traffic exceeds a system's processing capacity, queues build up, and that has a compounding effect rather than a linear one:

| Effect | What happens |
| --- | --- |
| High latency | Requests wait in queue longer than the client's own timeout |
| Wasted work | The server finishes processing a request, but the client already gave up and abandoned it — the work happened for nothing |
| Cascading failure | Upstream services time out and retry, which multiplies the incoming traffic and knocks down other dependent systems too |

Shedding load breaks this spiral by dropping excess requests immediately with an error (e.g., `503 Service Unavailable`) instead of queueing them — this keeps throughput and latency healthy for the traffic the system does accept, at the cost of visibly rejecting the rest.

## 2. Decision Flow

```
[ Incoming Traffic Surge ]
           │
           ▼
┌───────────────────────────┐
│  Load Shedding Evaluator  │
└─────────────┬─────────────┘
              │
     Is system overloaded?
     ┌────────┴────────┐
   No│                Yes│
     ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│ Process Request  │  │ Fail Fast (503) │
└─────────────────┘  └─────────────────┘
```

## 3. Common Shedding Strategies

| Strategy | How it decides what to drop |
| --- | --- |
| Priority-based shedding | Categorize traffic by importance — critical endpoints (`/checkout`, `/login`) always get processed, non-essential ones (`/recommendations`, `/product-reviews`) are dropped first |
| Concurrency / queue depth limiting | Reject new requests once active thread pools or memory queues hit a strict maximum |
| CPU / memory usage thresholds | Automatically drop incoming requests once node CPU (or memory) crosses a safe limit, e.g. >85% |
| CoDel / queue-time-based shedding | Measure how long a request has already waited in queue; if that exceeds a threshold (e.g. 200ms), drop it instantly without processing — a request that's already waited too long is unlikely to still be useful to its caller by the time it'd finish |

CoDel-style shedding is the sharpest of these: it doesn't need to know CPU or memory at all, just how stale a queued request already is, which makes it a good proxy for "is this request even still worth doing."

## 4. Real-World Example: Black Friday Flash Sale

An e-commerce platform receives 10x its normal traffic during a flash sale.

- **Without load shedding**: CPU spikes to 100%, memory fills up, database connections lock up, and the whole site crashes — 0% of users can buy anything.
- **With load shedding**: the service drops product recommendation widgets and analytics trackers (`503`) while continuing to prioritize the checkout API. 80% of users complete their purchase and the platform stays online.

The core trade being made is deliberate: sacrifice the least valuable requests early so the most valuable ones (revenue-generating) keep working at all.

## 5. Load Shedding vs. Rate Limiting

| | Rate Limiting | Load Shedding |
| --- | --- | --- |
| Focus | Client-focused | System-focused |
| Trigger | A fixed quota per client/API key, agreed in advance | The server's own real-time resource strain (CPU, memory, queue depth/time) |
| Rejects based on | Who is asking and how much they've already used | Whether the system itself is currently overloaded, regardless of who's asking |
| Still rejects when the system is healthy | Yes, if a client exceeds its quota | No — only activates once the system is actually struggling |

The two are complementary, not competing: rate limiting caps predictable per-client volume before it ever becomes a problem; load shedding is the reactive last line of defense for whatever aggregate overload gets through anyway — from a traffic spike, a retry storm, or reduced capacity during a partial outage.
