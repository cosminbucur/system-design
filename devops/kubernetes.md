Kubernetes runs and orchestrates many containers reliably at scale — scheduling them onto nodes, restarting failed ones, load-balancing traffic across replicas, and scaling capacity up or down. For a Java service, it's the layer that turns a set of container images into a resilient, horizontally-scalable running system, and it connects directly to concerns like service discovery, health checks, and autoscaling that would otherwise have to be built by hand.

## 1. Core Objects

| Object | Purpose |
| --- | --- |
| Pod | The smallest deployable unit — one or more tightly-coupled containers sharing network/storage |
| Deployment | Declares desired state (image, replica count) and manages rolling updates/rollbacks of Pods |
| Service | A stable network endpoint in front of a set of Pods — Kubernetes' built-in service discovery |
| ConfigMap / Secret | Externalized configuration / sensitive values, injected into Pods without baking them into the image |
| Ingress | Routes external HTTP traffic into Services — often where an API Gateway-like layer lives |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.2
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

`requests` is what Kubernetes guarantees/reserves for scheduling; `limits` is the hard ceiling. Setting memory `requests` == `limits` is common for JVM workloads — the JVM's `-XX:MaxRAMPercentage` behaves predictably against a fixed ceiling, rather than fluctuating with whatever's momentarily available on the node.

## 2. Health Probes — Wiring in Spring Boot Actuator

Kubernetes needs to know whether a Pod is alive and whether it's ready for traffic — this maps directly onto the Actuator health endpoints.

```yaml
containers:
  - name: order-service
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      periodSeconds: 5
```

- **Liveness probe**: "is this process alive, or should Kubernetes kill and restart the Pod?" A failing liveness probe triggers a restart — use it for genuinely unrecoverable states (deadlock, corrupted internal state), not transient issues.
- **Readiness probe**: "should this Pod currently receive traffic?" A failing readiness probe just removes the Pod from the Service's load-balancing pool — used for temporary conditions (still warming up a cache, a downstream dependency is briefly unavailable). This is the Kubernetes-level version of the Bulkhead/Circuit Breaker idea — don't route traffic to something that can't handle it right now.

Getting liveness wrong is a classic outage cause: if a liveness probe checks a downstream dependency (e.g., the database) and that dependency is briefly down, Kubernetes will restart every Pod in a crash loop even though the app itself is fine — liveness should only check the process's own health, not its dependencies.

## 3. Horizontal vs Vertical Scaling

Two fundamentally different answers to "this service needs more capacity":

| | Horizontal scaling (scale out) | Vertical scaling (scale up) |
| --- | --- | --- |
| What changes | More Pod replicas, same size each | Same number of Pods, more CPU/memory per Pod |
| Ceiling | Limited mainly by cluster capacity and the app's ability to run many instances | Hard-capped by the largest node in the cluster — you can't vertically scale past your biggest machine |
| Failure impact | Losing one of N replicas is a small capacity dip | Losing the one (bigger) Pod is a much larger capacity hit |
| Fit for stateless Java services | Natural fit — a Deployment with `replicas: N` behind a Service already load-balances across them | Occasionally needed for a single JVM that must hold a large heap (e.g., a big in-memory cache) that can't be sharded |
| Kubernetes mechanism | Horizontal Pod Autoscaler (HPA) | Vertical Pod Autoscaler (VPA), or manually adjusting `resources.requests/limits` |

For a typical stateless Java web service, **prefer horizontal scaling by default**: more replicas behind a Service both add capacity and add redundancy — a Pod crashing or being evicted only removes a fraction of total capacity. Vertical scaling doesn't buy you redundancy at all — a single, bigger Pod is still a single point of failure, just one that can absorb more load before falling over. Reach for vertical scaling when a workload is genuinely hard to horizontally distribute — e.g., a single JVM process holding a large unshardable in-memory dataset, or a workload bottlenecked on per-request CPU rather than concurrent request volume (more, smaller Pods won't help a single slow computation finish faster).

A subtlety specific to the JVM: vertical scaling isn't free just because you gave the container more memory. A bigger container limit does translate into a bigger heap automatically, but a much larger heap also means longer GC pauses unless you've validated that with the collector in use — "just give it more RAM" can trade one bottleneck (capacity) for another (GC pause latency) if you don't check.

## 4. Horizontal Pod Autoscaler (HPA)

Scales replica count based on observed metrics (CPU by default, or custom metrics like request rate/queue depth via Prometheus adapters).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Vertical Pod Autoscaler (VPA) is the counterpart — instead of adding replicas, it adjusts a Pod's `resources.requests`/`limits` (typically by recommending, or in "Auto" mode applying, new values that require the Pod to restart). HPA and VPA can conflict if both target CPU on the same Deployment — a common setup is HPA on custom/request-based metrics with VPA left in recommendation-only mode to inform right-sizing, rather than running both in fully automatic mode against the same signal.

## 5. ConfigMaps and Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://order-db:5432/orders
```

```yaml
containers:
  - name: order-service
    envFrom:
      - secretRef:
          name: order-service-secrets # e.g. DB password, API keys
    volumeMounts:
      - name: config
        mountPath: /app/config
volumes:
  - name: config
    configMap:
      name: order-service-config
```

Secrets in plain Kubernetes are only base64-encoded, not encrypted, at rest by default — treat raw `kind: Secret` manifests as sensitive, and prefer an external secrets manager (Vault, cloud KMS-backed secret stores) integrated via an operator for anything genuinely sensitive in production.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Set explicit resource requests/limits | Prevents one runaway Pod from starving others on the same node, and lets the JVM size itself correctly via `MaxRAMPercentage`. |
| Keep liveness probes dependency-free | Liveness should only check "is this process itself healthy," never a downstream dependency — otherwise a database blip causes a cluster-wide crash loop. |
| Prefer horizontal scaling by default for stateless services | Adds both capacity and redundancy; vertical scaling adds capacity but not redundancy, and hits a hard node-size ceiling. |
| Validate GC behavior before scaling vertically | A bigger heap from a larger container limit can trade a capacity bottleneck for a GC pause-latency bottleneck if left unchecked. |
| Choose HPA/VPA deliberately, not both fully automatic on the same signal | They can conflict if both target CPU on the same Deployment — pair HPA on real metrics with VPA in recommendation-only mode. |
| Never commit real secrets into a plain `kind: Secret` manifest | Base64 is encoding, not encryption — use a secrets manager or sealed-secrets approach for anything genuinely sensitive. |
