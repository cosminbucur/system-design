Alerting turns a metric that's been collected into a notification that someone (or some automated process) needs to act on right now. Prometheus is the standard toolchain for this in a Kubernetes-native Java stack: it scrapes and stores metrics as time series, evaluates alerting rules written in PromQL against that data continuously, and hands anything that fires to Alertmanager, which decides who gets notified, how, and how often.

## 1. How the Pieces Fit Together

```
Spring Boot app (/actuator/prometheus) → Prometheus scrapes metrics every N seconds
  → evaluates alerting rules (PromQL) continuously
      → a rule's condition becomes true → fires an alert
          → Alertmanager receives it → routes, groups, silences, notifies
```

Prometheus itself only decides _whether_ an alert condition is true — it has no concept of on-call schedules, notification channels, or "don't page anyone at 3am for a warning-level issue." That's Alertmanager's job, kept as a deliberately separate component so the alerting _logic_ (what counts as a problem) stays decoupled from the alerting _routing_ (who hears about it and how).

## 2. Writing an Alerting Rule in PromQL

```yaml
groups:
  - name: order-service-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{application="order-service", status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{application="order-service"}[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Order service error rate above 5%"
          description: "{{ $value | humanizePercentage }} of requests are failing (5xx) over the last 5 minutes."
          runbook_url: "https://wiki.example.com/runbooks/order-service-high-error-rate"
```

Key pieces:

- **`expr`**: the PromQL expression evaluated on every scrape interval — here, the ratio of 5xx responses to total requests, computed as a rate over a 5-minute window.
- **`for`**: how long the condition must stay true before the alert actually fires — without this, a single noisy scrape interval could trigger a page for a blip that resolves itself a few seconds later.
- **`labels`**: metadata Alertmanager uses for routing (`severity: critical` might page on-call immediately, `severity: warning` might just post to a Slack channel).
- **`annotations`**: human-readable context shown in the notification — `runbook_url` specifically is what turns "something's wrong" into "here's what to check first."

## 3. Rate, Not Raw Counters

Prometheus counters (like `http_server_requests_seconds_count`) only ever go up — alerting directly on a raw counter value is meaningless, since it grows forever regardless of whether things are actually fine. `rate()` converts a counter into a per-second average rate over a time window, which is what you actually want to threshold against.

```promql
# WRONG: this number only ever increases, comparing it to a fixed threshold makes no sense
http_server_requests_seconds_count{status="500"} > 100

# RIGHT: the rate of 5xx responses per second over the last 5 minutes
rate(http_server_requests_seconds_count{status="500"}[5m]) > 0.5
```

## 4. Alerting on Latency: `histogram_quantile`

Latency is usually recorded as a Prometheus histogram (bucketed counts of how many requests fell under each latency threshold), and PromQL's `histogram_quantile` function computes an approximate percentile from those buckets.

```promql
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{application="order-service"}[5m])) by (le)
) > 0.5
```

This alerts when p99 latency exceeds 500ms over a 5-minute window — the same percentile-over-average discipline that applies everywhere else in observability, now expressed as an alerting condition instead of just a dashboard panel.

## 5. Alertmanager: Routing, Grouping, and Silencing

Once Prometheus fires an alert, Alertmanager decides what actually happens with it.

```yaml
route:
  receiver: default-slack
  group_by: ["alertname", "application"]
  group_wait: 30s # wait briefly to batch related alerts firing together
  group_interval: 5m # how often to send updates about an existing firing group
  repeat_interval: 4h # don't re-notify about the same ongoing alert more often than this
  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
      continue: true # also fall through to the default route below
    - match:
        severity: warning
      receiver: slack-warnings

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: "<pagerduty-integration-key>"
  - name: slack-warnings
    slack_configs:
      - channel: "#order-service-alerts"
```

- **`group_by`**: bundles related alerts (e.g., the same alert firing across many pods) into a single notification instead of paging once per pod.
- **`group_wait`/`group_interval`/`repeat_interval`**: control notification pacing — without these, a flapping condition could fire a new notification every scrape interval.
- **Routing by label** (`severity: critical` → PagerDuty, `severity: warning` → Slack) is what implements "page a human only for things that actually need immediate action."

## 6. Silences and Inhibition

Two mechanisms exist specifically to prevent noisy or redundant notifications:

| Mechanism  | Purpose                                                                                                                                                                                                      |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Silence    | Manually suppress notifications for a specific alert/label match during a known window (a planned deploy, a maintenance window)                                                                              |
| Inhibition | Automatically suppress a lower-priority alert when a related, higher-priority alert is already firing (e.g., don't alert on every individual service being unreachable if the whole cluster is already down) |

```yaml
inhibit_rules:
  - source_matchers:
      - severity="critical"
      - alertname="ClusterDown"
    target_matchers:
      - severity=~"critical|warning"
    equal: ["cluster"]
```

Without inhibition, a single root-cause outage (the cluster itself being down) can trigger dozens of individual "service X unreachable" alerts simultaneously — inhibition rules recognize that relationship and suppress the noise, leaving just the one alert that actually explains what's happening.

## 7. Multi-Window, Multi-Burn-Rate Alerting for SLOs

A naive SLO alert ("error budget consumed too fast") has a real tension: a short window reacts fast but is noisy (false alarms on brief blips); a long window is stable but slow to notice a real, ongoing problem. The standard fix, popularized by Google's SRE practice, is alerting on multiple burn-rate windows simultaneously.

```promql
# Fast burn: consuming the error budget 14.4x too fast over 1 hour — page immediately, this is severe
(
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h]))
  /
  sum(rate(http_server_requests_seconds_count[1h]))
) > (14.4 * 0.001)  # 0.001 = the SLO's allowed error rate (99.9% target)

# Slow burn: consuming the error budget 3x too fast over 6 hours — ticket, not a page, less urgent
(
  sum(rate(http_server_requests_seconds_count{status=~"5.."}[6h]))
  /
  sum(rate(http_server_requests_seconds_count[6h]))
) > (3 * 0.001)
```

The fast-burn rule catches a severe, sudden problem quickly (and is allowed to be a bit noisy, because a page for a real severe issue is worth it); the slow-burn rule catches a sustained, lower-grade degradation that a short window would miss entirely. Running both together, at different severities, is what actually resolves the false-alarm-vs-slow-to-notice tension rather than picking one window and accepting its specific weakness.

## 8. Best Practices

| Practice                                                           | Recommendation                                                                                                                        |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| Alert on rate, never on a raw counter value                        | A monotonically increasing counter has no meaningful fixed threshold — always wrap it in `rate()` first.                              |
| Use `for` to avoid paging on a single noisy scrape                 | A condition should hold for a real duration before it's worth waking someone up.                                                      |
| Route by severity, not by a single flat notification channel       | Critical issues should page; warnings should land somewhere lower-urgency (Slack, a ticket queue).                                    |
| Use multi-window burn-rate alerting for SLOs                       | A single window forces a tradeoff between noise and slow detection — running fast- and slow-burn rules together avoids that tradeoff. |
| Set up inhibition for known root-cause relationships               | Prevents a single outage from paging on-call with dozens of redundant, individually-true alerts.                                      |
| Attach a runbook link to every alert                               | An alert that doesn't tell the on-call engineer what to check first needs more context, not just a threshold.                         |
| Alert on user-facing symptoms, not internal implementation details | Error rate, latency, and availability are what users actually experience — not every internal metric fluctuation deserves a page.     |
