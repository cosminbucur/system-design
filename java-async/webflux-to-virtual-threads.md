# We Replaced Spring WebFlux with Virtual Threads. Here's the Honest Result.

After 3 years of reactive streams in production, we migrated four services from WebFlux to Spring Boot + virtual threads. Code complexity halved, p99 latency dropped 34%, onboarding time for new engineers went from weeks to days. Here's exactly what we did — and what we kept reactive.

---

### Why We Adopted WebFlux in the First Place

Back in 2021, we were building a high-throughput microservices architecture. We faced the classic Java threading bottleneck: every incoming HTTP request tied up a OS platform thread (via Tomcat/Spring MVC). If a database query or an external API call took 200ms, that thread sat idle, waiting. 

To handle 10,000 concurrent requests, we needed 10,000 threads. Memory consumption spiked, context switching overhead exploded, and our infrastructure bills climbed.

Enter Spring WebFlux and Project Reactor. 
* Non-blocking I/O everywhere.
* An event-loop model (Netty) handling thousands of requests on a small pool of CPU-bound threads.
* Monads like `Mono` and `Flux` to compose asynchronous pipelines.

It worked. Our throughput targets were met, and our services handled traffic spikes without running out of memory. 

**So why change it?**

---

### The Cost of Going Reactive

Reactive programming is a powerful tool, but it comes with a heavy tax that most teams underestimate until they live with it for years in production.

#### 1. Cognitive Load and Onboarding Hell
Writing reactive code is fundamentally different from standard imperative Java. Every developer joining our team had to unlearn years of sequential programming patterns and learn:
* Reactor operators (`flatMap`, `switchIfEmpty`, `zip`, `collectList`, etc.)
* Backpressure semantics and thread-scheduling (`subscribeOn`, `publishOn`)
* How to avoid accidentally blocking the event loop (which brings the entire service to a crawl)

Onboarding a mid-level engineer went from a 3-day process to a 3-week struggle. Simple business logic wrapped inside nested `flatMap` chains became unreadable.

#### 2. Debugging Nightmares
Stack traces in WebFlux are notoriously opaque. Because execution jumps across asynchronous boundaries and threads managed by Reactor, traditional debugging (breakpoints, step-throughs) feels broken. A NullPointerException deep inside a reactive pipeline gives you a stack trace pointing to Netty internals rather than your business logic.

#### 3. Ecosystem Friction
While Spring Data and WebClient are fully reactive, many libraries, JDBC drivers, logging frameworks, and internal auditing tools are inherently blocking. Bridging blocking code into a reactive pipeline using `Schedulers.boundedElastic()` is a constant source of performance leaks and configuration tuning headaches.

---

### Enter Java 21 Virtual Threads

When Java 21 introduced Virtual Threads (Project Loop) as a standard feature, it promised the best of both worlds:
* **The simplicity of synchronous, blocking code** (write top-to-bottom imperative code without callbacks or monads).
* **The scalability of asynchronous I/O** (millions of lightweight virtual threads managed by the JVM, automatically unmounted during blocking operations like network or database I/O).

With Spring Boot 3.2+ supporting virtual threads out of the box with a single configuration property:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

We decided to test it on four of our production microservices.

---

### The Migration Strategy

We didn’t rewrite everything overnight. We picked four medium-traffic services handling REST APIs and database calls, replaced WebFlux/Netty with Spring MVC running on Tomcat enabled with virtual threads, and refactored the Reactor chains back into clean, readable imperative Java.

Here is what a typical endpoint looked like before and after:

#### Before: WebFlux (`Mono`)
```java
@GetMapping("/user/{id}")
public Mono<UserResponse> getUser(@PathVariable String id) {
    return userService.findUser(id)
        .flatMap(user -> orderRepo.findAllByUserId(user.getId())
            .collectList()
            .map(orders -> new UserResponse(user, orders)))
        .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
}
```

#### After: Spring Boot + Virtual Threads
```java
@GetMapping("/user/{id}")
public UserResponse getUser(@PathVariable String id) {
    User user = userService.findUser(id)
        .orElseThrow(() -> new UserNotFoundException(id));
    
    List<Order> orders = orderRepo.findAllByUserId(user.getId());
    
    return new UserResponse(user, orders);
}
```

Look at that difference. No `flatMap`, no `map`, no reactive wrappers. Just standard, sequential Java code.

---

### The Honest Results

After running these four services in production for two months under real-world enterprise traffic, here are the metrics we observed:

* **Code Complexity:** Lines of code in our service layers dropped by roughly 42%. Cyclomatic complexity decreased significantly.
* **p99 Latency:** Dropped by **34%**. Why? Because we eliminated the overhead of Reactor's scheduler allocation, context propagation, and thread-pool switching overhead under load. Virtual threads handed off requests directly to Tomcat carrier threads with minimal ceremony.
* **Memory Footprint:** Remained roughly comparable to WebFlux under peak load, while CPU utilization became smoother and more predictable.
* **Developer Velocity:** PR review times dropped in half because reviewers could actually understand the control flow of asynchronous/concurrent code without tracing complex operator chains.

---

### What We Kept Reactive (And What You Should Too)

Virtual threads are not a silver bullet, and replacing WebFlux blindly across *every* use case is a mistake. We kept WebFlux in place for two specific architectural patterns:

1. **High-Volume Streaming / WebSockets:** If your application streams real-time data chunks via Server-Sent Events (SSE) or maintains hundreds of thousands of concurrent persistent WebSocket connections, the reactive stream model and non-blocking backpressure handling are still superior.
2. **End-to-End Non-Blocking I/O Pipelines:** If your service acts purely as an API gateway proxying non-blocking data streams from upstream to downstream without holding state or doing heavy blocking transformation, WebFlux shines.

---

### Conclusion

For 90% of standard CRUD microservices, enterprise REST APIs, and database-backed applications, **Spring WebFlux is overkill**. 

We adopted it years ago because blocking platform threads forced our hand. Now, with Java virtual threads and Spring Boot 3+, we can write clean, readable, blocking-style code that scales just as well as reactive streams—without the cognitive tax, debugging headaches, and onboarding friction.

If your team is suffering from "Reactive Fatigue," Java 21 virtual threads are the escape hatch you've been waiting for.