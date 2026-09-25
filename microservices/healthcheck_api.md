# Healthcheck API Pattern in Microservices Architecture

The **Healthcheck API Pattern** is a core design pattern in microservices architecture. It requires each service instance to expose one or more dedicated endpoints (e.g., `/health`, `/healthz`, `/live`, `/ready`) that report the operational status of the service and its critical dependencies.

Container orchestrators (like Kubernetes), API gateways, load balancers, and monitoring systems regularly query these endpoints to determine how to manage traffic and instance lifecycles.

---

## 1. Primary Objectives

- **Automated Recovery:** Enables orchestrators to detect deadlocked or crashed processes and automatically restart them.
- **Traffic Routing:** Prevents load balancers from sending user requests to instances that are starting up, shutting down, or overwhelmed.
- **Observability & Alerting:** Provides monitoring tools (e.g., Prometheus, Datadog) with real-time status data to trigger alerts when service degrade.

---

## 2. Core Categories of Health Checks

Modern orchestration environments (such as Kubernetes) categorize health checks into three primary types:

| Check Type | Typical Endpoint | Purpose | Action Taken on Failure |
| :--- | :--- | :--- | :--- |
| **Liveness Probe** | `/health/live` | Checks if the application process is alive and responsive (not in a deadlock or infinite loop). | The orchestrator **kills and restarts** the container instance. |
| **Readiness Probe** | `/health/ready` | Checks if the application is fully initialized and capable of accepting network traffic. | The load balancer **removes the instance** from the active routing pool until it passes. |
| **Startup Probe** | `/health/startup` | Checks if a slow-starting application has finished booting. | Disables liveness/readiness checks until it succeeds, avoiding premature container restarts. |

---

## 3. Shallow vs. Deep Health Checks

### Shallow Health Checks
- **Scope:** Checks only the local process status (e.g., memory usage, event loop status, simple HTTP response).
- **Usage:** Ideal for **Liveness Probes**.
- **Pros:** Extremely fast and lightweight.
- **Cons:** Does not guarantee the service can complete business operations.

### Deep Health Checks
- **Scope:** Verifies access to essential external dependencies, such as SQL databases, Redis caches, message queues, and critical downstream microservices.
- **Usage:** Ideal for **Readiness Probes**.
- **Pros:** Reflects the actual operational capability of the service.
- **Cons:** Higher resource consumption; potential to cause cascading failures if misconfigured.

---

## 4. Sample JSON Payload Standard

While HTTP status codes (`200 OK` for healthy, `503 Service Unavailable` for unhealthy) convey the primary signal, the response body often provides diagnostic metadata.

```json
{
  "status": "UP",
  "timestamp": "2026-09-24T21:00:00Z",
  "version": "2.4.1",
  "checks": [
    {
      "name": "Database Connection (PostgreSQL)",
      "status": "UP",
      "responseTimeMs": 14
    },
    {
      "name": "Redis Cache",
      "status": "UP",
      "responseTimeMs": 3
    },
    {
      "name": "Payment Gateway API",
      "status": "DOWN",
      "error": "Connection timeout after 2000ms"
    }
  ]
}
```

---

## 5. Architectural Best Practices

1. **Keep Liveness and Readiness Separate**
   - *Crucial Rule:* Never check external dependencies (like databases) in a `/health/live` check. If a database experiences temporary downtime, failing the liveness check will cause the orchestrator to restart all application containers simultaneously, creating a **Thundering Herd** problem upon database recovery.

2. **Implement Caching for Deep Checks**
   - High-frequency health probes from load balancers can overwhelm backend databases with status queries. Cache deep check results for 5 to 10 seconds to reduce unnecessary overhead.

3. **Set Low Timeouts**
   - Health check endpoints should fail fast. Configure response timeouts (e.g., 2–5 seconds) so that a hanging check does not block monitoring threads or obscure actual latency.

4. **Security and Data Privacy**
   - Avoid exposing sensitive diagnostic details (e.g., database connection strings, stack traces, host IP addresses) publicly. Restrict detailed diagnostic outputs to internal networks or require administrative authentication.

5. **Graceful Shutdown Integration**
   - When an instance receives a termination signal (`SIGTERM`), its readiness endpoint should immediately start returning `HTTP 503` (Unhealthy). This signals the load balancer to drain active connections before the container process is terminated.