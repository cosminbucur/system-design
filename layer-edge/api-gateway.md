An API Gateway is a single entry point that sits in front of a set of backend services, routing each incoming request to the right one and centralizing concerns that would otherwise have to be duplicated inside every service — authentication, rate limiting, request logging, response shaping. Clients (mobile apps, web frontends, third-party integrators) talk to one address instead of needing to know the internal topology of dozens of services.

## 1. What It Solves

Without a gateway, every client needs to know every service's address, every service has to implement its own auth/logging/rate-limiting, and any cross-cutting change (rotating an auth mechanism, adding a new header) means touching every service individually. A gateway moves that shared logic to one place, and gives clients one stable contract regardless of how the backend is decomposed internally.

```yaml
# Spring Cloud Gateway route config
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
```

## 2. Core Responsibilities

| Responsibility | Example |
| --- | --- |
| Routing | `/api/orders/**` → `order-service`, `/api/inventory/**` → `inventory-service` |
| Authentication/authorization | Validate a JWT once at the edge, so backend services trust a header instead of each re-validating |
| Rate limiting / throttling | Cap requests per API key or IP before they ever reach a backend service |
| Request/response transformation | Strip internal headers, reshape a response for a specific client type |
| TLS termination | Handle HTTPS at the edge so internal traffic can stay plain HTTP within a trusted network |
| Centralized logging/metrics | One place to see every request that entered the system, before it fans out internally |

```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !jwtValidator.isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange); // valid — forward to the backend service
    }
}
```

## 3. Backend for Frontend (BFF) — One Gateway per Client Type

A single generic gateway can end up trying to satisfy a mobile app, a web app, and a third-party partner API all at once, each wanting a different shape of response. The Backend for Frontend variant runs a separate, purpose-built gateway per client type, each composing/shaping backend calls specifically for that consumer.

```
Mobile app       → Mobile BFF   → (aggregates/shapes calls to) OrderService, InventoryService
Web app          → Web BFF      → (aggregates/shapes calls to) OrderService, InventoryService, RecommendationService
Partner API      → Partner BFF  → (exposes a stable, versioned subset) OrderService
```

This avoids the single gateway becoming a tangle of client-specific conditional logic, at the cost of running and maintaining more than one gateway.

## 4. Request Aggregation — Reducing Round Trips

A gateway can also compose several backend calls into one response, saving a client (especially a mobile client on a slow network) from making multiple round trips itself.

```java
@GetMapping("/api/order-summary/{orderId}")
public Mono<OrderSummary> getOrderSummary(@PathVariable String orderId) {
    Mono<Order> order = orderClient.getOrder(orderId);
    Mono<Customer> customer = order.flatMap(o -> customerClient.getCustomer(o.getCustomerId()));
    Mono<List<InventoryStatus>> inventory = order.flatMap(o -> inventoryClient.getStatus(o.getItems()));

    return Mono.zip(order, customer, inventory)
        .map(tuple -> new OrderSummary(tuple.getT1(), tuple.getT2(), tuple.getT3()));
}
```

This trades an extra hop for the client (one call instead of three) for extra responsibility inside the gateway — it now has to know how to combine data from multiple services, which starts to blur into business logic if taken too far.

## 5. The Trap: Turning the Gateway into a Second Monolith

The most common way an API Gateway goes wrong is accumulating actual business logic over time — validation rules, orchestration workflows, domain-specific decisions — until it becomes a de facto second monolith that every team depends on and is afraid to change. The gateway should route and enforce cross-cutting policy; business logic belongs in the owning service. A useful test: if removing the gateway and calling a service directly would change the *business outcome* (not just remove auth/logging), too much has leaked into the gateway.

## 6. Gateway as a Single Point of Failure

Since every request passes through it, the gateway's own availability becomes a hard dependency for the entire system — if it's down, every service behind it is effectively unreachable even if all of them are individually healthy. This means the gateway itself needs to be treated with the same operational seriousness as the most critical service behind it: run multiple replicas, keep it stateless so any replica can handle any request, and give it its own dedicated monitoring separate from the services it fronts.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Keep the gateway free of business logic | Route, authenticate, rate-limit, and log — leave domain decisions to the owning service. |
| Run the gateway highly available and stateless | Every request depends on it — treat its uptime as seriously as the most critical service behind it. |
| Use a Backend for Frontend when client needs diverge significantly | Avoids one gateway accumulating tangled, client-specific conditional logic. |
| Validate auth once at the edge, not redundantly in every service | Backend services can then trust a propagated identity/claims header instead of re-validating a token each time. |
| Use request aggregation sparingly | Composing a few calls to save client round trips is fine; orchestrating complex workflows there is a sign logic has leaked in. |
| Version the gateway's public routes independently from internal service versions | Lets backend services evolve their internal APIs without breaking the external contract clients depend on. |
| Monitor the gateway separately from backend services | A slow or failing gateway can look identical to a slow backend from the client's perspective — separate metrics are needed to tell them apart. |
