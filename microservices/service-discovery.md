Service discovery answers a question that only exists once a service has multiple, dynamically changing instances: given a logical service name, how does a caller find a currently healthy instance to actually send a request to? Instances come and go constantly in a microservices deployment — scaling events, deployments, crashes — so a hardcoded list of IP addresses goes stale almost immediately. Discovery is the mechanism that keeps that mapping current. There are two fundamentally different places to put that lookup logic: server-side and client-side.

## 1. The Service Registry — Shared by Both Approaches

Whichever side does the discovery, something has to track which instances currently exist and are healthy — that's the service registry. Instances register themselves (or are registered automatically by the platform) and are removed from rotation the moment they fail a health check.

| Registry | Typical pairing |
| --- | --- |
| Kubernetes API (etcd-backed) | Kubernetes Service (server-side) |
| Eureka | Spring Cloud LoadBalancer (client-side) |
| Consul | Either — HAProxy/Envoy for server-side, Consul client library for client-side |
| Cloud provider's own registry (AWS Cloud Map, etc.) | ALB/NLB target groups (server-side) |

## 2. Server-Side Discovery

The caller sends a request to a single well-known address, unaware that multiple instances or a registry even exist. A router in front of the actual instances queries the registry and forwards the request.

```
Client → Router/Load Balancer → [queries the Service Registry for current instances] → Instance A / B / C
```

```yaml
# Kubernetes Service is the most common real-world example of server-side discovery
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service # the registry lookup: "which pods currently match this label?"
  ports:
    - port: 80
      targetPort: 8080
```

A pod calling `http://order-service` never queries the Kubernetes API itself — `kube-proxy` (or the CNI's equivalent) intercepts the connection and routes it to one of the matching, ready pods. The client makes a plain HTTP call; the discovery and load-balancing decision both happen invisibly, server-side.

## 3. Client-Side Discovery

The caller itself queries the registry (directly, or via a client library) and picks an instance to connect to directly — no router in the middle.

```
Client → [queries the Service Registry directly] → picks an instance → Instance A / B / C
```

```java
// Spring Cloud LoadBalancer resolves "order-service" to a specific instance per call
@LoadBalanced
@Bean
public RestClient.Builder restClientBuilder() {
    return RestClient.builder();
}

// Calling code just uses the logical service name — the instance lookup happens underneath, in-process
restClient.get().uri("http://order-service/api/orders/{id}", orderId).retrieve();
```

The registry lookup and the load-balancing algorithm both run inside the calling service's own process — there's no extra network hop, but every caller needs a discovery-aware client library, not just a plain HTTP client.

## 4. Server-Side vs. Client-Side, Side by Side

| | Server-side discovery | Client-side discovery |
| --- | --- | --- |
| Who looks up the registry | The router/load balancer | The calling service itself |
| What the client needs to know | Just a fixed hostname | The registry's location and a discovery-aware client library |
| Extra network hop | Yes — every call passes through the router | No — client connects directly to the chosen instance |
| Client complexity | None — plain HTTP client, any language | Needs discovery logic built into every client (e.g., Spring Cloud LoadBalancer) |
| Where load-balancing logic lives | Centralized, in the router | Duplicated in every client/language calling the service |
| Failure mode if the discovery layer is down | The router itself becomes a single point of failure | Only the individual caller relying on a stale/failed registry lookup is affected |

Server-side discovery trades a small amount of latency and a shared piece of infrastructure (the router) for dramatically simpler clients — any language, any framework, even a `curl` command, can call the service correctly. Client-side discovery removes that hop and that shared dependency, at the cost of every calling service needing the same discovery-aware logic built in, in whatever language it happens to be written in.

## 5. Common Real-World Examples

- **Server-side**: Kubernetes Service (the default in almost any Kubernetes deployment, whether or not a team thinks of it as "a pattern"), AWS ALB/NLB with target groups, an API Gateway in front of a service mesh.
- **Client-side**: Spring Cloud LoadBalancer + Eureka (the classic Netflix-OSS-era Spring Cloud stack), a service mesh sidecar making the routing decision on the caller's own host before the request leaves the node.

Note that a service mesh sidecar (Envoy running alongside each instance) blurs the line somewhat: from the application code's point of view it looks like server-side discovery (a plain call to `localhost`), but the actual registry lookup happens in a process colocated with the caller rather than in a shared central router — it gets the client simplicity of server-side discovery without the shared-router single point of failure.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Default to server-side discovery unless you have a specific reason not to | Simpler clients and centralized routing logic outweigh the extra hop for most systems — this is why Kubernetes bakes it in by default. |
| Reserve client-side discovery for cases needing the extra hop removed | Ultra-low-latency internal calls, or when a shared router would itself become an unacceptable single point of failure. |
| Make a server-side router highly available | Since every call passes through it, treat its uptime as a hard dependency for the entire system, not an afterthought. |
| Keep health checks accurate and fast on either approach | Both depend entirely on the registry's view of instance health being current — a slow or wrong health check routes traffic to a bad instance either way. |
| Don't hand-roll client-side discovery per language | Use an established client library (Spring Cloud LoadBalancer, a service mesh sidecar) rather than reimplementing registry polling and load-balancing logic per service. |
| Consider a service mesh sidecar for the best of both | Gets client-side-like simplicity (no shared router) with server-side-like simplicity for the application code (a plain local call). |
