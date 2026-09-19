Metrics are observability's second pillar — aggregated numbers over time, answering "how is the system behaving in aggregate?" Where a log tells you about one specific event, a metric tells you about a trend: is the error rate climbing, is latency degrading, is a queue backing up.

## 1. Counters, Gauges, Histograms

| Type | Behavior | Example |
| --- | --- | --- |
| Counter | Monotonically increasing, only goes up (until reset) | Total HTTP requests served |
| Gauge | A value that can go up or down | Current active DB connections, current queue depth |
| Histogram/Timer | Distribution of observed values, usually with percentiles | Request latency (p50, p95, p99) |

```java
@Service
public class OrderService {
    private final MeterRegistry meterRegistry;

    public void placeOrder(Order order) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            // ... business logic
            meterRegistry.counter("orders.placed", "status", "success").increment();
        } catch (Exception ex) {
            meterRegistry.counter("orders.placed", "status", "failure").increment();
            throw ex;
        } finally {
            sample.stop(meterRegistry.timer("orders.placement.duration"));
        }
    }
}
```

Spring Boot Actuator + Micrometer expose these automatically for common things (HTTP request timing, JVM memory/GC, thread pool usage, DataSource connection pool stats) with almost no manual instrumentation — add custom counters/timers only for business-meaningful events (orders placed, payments failed), not to duplicate what's already auto-instrumented.

## 2. Why p99 Matters More Than Average

Averages hide the worst experiences. If 99 requests take 10ms and 1 takes 5 seconds, the average (~60ms) looks fine while 1% of users are having a terrible time — and at scale, "1% of requests" can be thousands of real users per hour.

```
p50 (median): typical request experience
p95: the slower 5% — often where real problems start showing
p99: the tail — what your least lucky users experience; frequently where timeouts/cascading failures originate
```

Always look at percentile latency, not just averages, when setting alerts or SLOs.

## 3. Exposing Metrics: Spring Boot Actuator

Actuator exposes operational endpoints out of the box — health checks, metrics, and environment info — with almost no setup.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus, info
  endpoint:
    health:
      show-details: when-authorized # never "always" in production — avoid leaking internal details publicly
```

```java
@Component
public class DownstreamHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return isDownstreamReachable()
            ? Health.up().build()
            : Health.down().withDetail("reason", "downstream unreachable").build();
    }
}
```

`/actuator/health` feeds Kubernetes liveness/readiness probes; `/actuator/prometheus` exposes metrics in a format Prometheus scrapes directly — the pull-based model already covered in more depth in alerting with Prometheus.

## 4. Alerting and SLOs

Metrics are only useful operationally if someone (or something) acts on them. An SLO (Service Level Objective) states a measurable target — e.g., "99.9% of requests complete under 500ms over a rolling 30 days" — and alerts should fire on SLO burn rate, not on every minor blip.

Practical guidance:

- Alert on symptoms users would notice (error rate, latency, availability) — not on every internal metric fluctuation.
- Avoid alert fatigue: an alert that fires constantly and gets ignored is worse than no alert — it trains people to ignore pages.
- Tie alerts to an actionable runbook — if firing the alert doesn't tell the on-call engineer what to check first, the alert needs more context, not just a threshold.

## 5. Best Practices

| Practice | Recommendation |
| --- | --- |
| Add custom metrics only for business-meaningful events | Actuator/Micrometer already auto-instrument HTTP timing, JVM, and connection pool stats — don't duplicate what's already free. |
| Watch p95/p99, not just average latency | Averages hide the tail experience where real user pain and cascading failures live. |
| Never expose full health details publicly | `show-details: when-authorized`, not `always` — avoid leaking internal dependency status to unauthenticated callers. |
| Alert on user-facing symptoms with an actionable runbook | Avoid alert fatigue from noisy, low-value alerts nobody acts on. |
| Alert on SLO burn rate, not every minor threshold breach | A single blip crossing a static threshold is a weaker signal than a sustained rate of budget consumption. |
| Choose counters, gauges, and histograms deliberately | A counter can't represent something that goes down (like queue depth); a gauge can't cheaply give you a percentile — pick the type that matches what's actually being measured. |
