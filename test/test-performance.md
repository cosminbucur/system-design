Performance tests verify behavior under load — throughput, latency percentiles, and breaking points — questions unit and integration tests can't answer at all, since they exercise the system with a single request at a time. This is where testing connects to observability (p95/p99 latency is the metric you're actually validating here) and profiling (once a load test finds a bottleneck, a profiler tells you exactly where in the code it lives).

## 1. Why Not Just Unit/Integration Tests

A passing integration test proves one request works correctly in isolation. It says nothing about what happens when 500 requests hit the same endpoint concurrently — connection pool exhaustion, lock contention, N+1 queries that were invisible with one row of test data, or a downstream dependency that degrades under concurrent load. Performance tests exist specifically to surface these.

## 2. Types of Performance Tests

| Type | Goal |
| --- | --- |
| Load test | Confirm the system meets latency/throughput targets at expected production traffic |
| Stress test | Push well past expected traffic to find the actual breaking point and failure mode |
| Soak test | Run sustained moderate load for an extended period — surfaces slow leaks that only show up over time, not in a short burst |
| Spike test | Sudden traffic surge (e.g., a flash sale) — validates autoscaling and backpressure handling, not just steady-state capacity |

## 3. Gatling — Scenario-Based, JVM-Native

Gatling scripts are Scala/Java DSL, run on the JVM, and produce detailed HTML reports (response time distributions, percentiles, requests/sec over time) — a natural fit when your team is already Java-centric and wants load tests versioned and run alongside the rest of the build.

```java
public class OrderApiSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://api.example.com")
        .acceptHeader("application/json");

    ScenarioBuilder placeOrder = scenario("Place Order")
        .exec(http("Create order")
            .post("/api/orders")
            .body(StringBody("""{"sku":"SKU-1","quantity":2}"""))
            .check(status().is(201)));

    {
        setUp(
            placeOrder.injectOpen(
                rampUsersPerSec(10).to(100).during(Duration.ofMinutes(2)), // ramp up
                constantUsersPerSec(100).during(Duration.ofMinutes(5))     // sustain
            )
        ).protocols(httpProtocol)
         .assertions(
             global().responseTime().percentile(99).lt(500), // p99 under 500ms
             global().successfulRequests().percent().gt(99.0)  // <1% error rate
         );
    }
}
```

Gatling's built-in assertions (`.assertions(...)`) let a load test pass/fail against explicit SLO-like thresholds — the same p99 discipline, but enforced automatically in CI rather than eyeballed on a dashboard.

## 4. k6 — Scriptable in JavaScript, CI/Cloud-Native

k6 (Grafana Labs) scripts are plain JavaScript, lightweight, and designed to integrate cleanly into CI pipelines and Grafana dashboards — a common choice when the load-testing scripts are owned by a platform/SRE team rather than embedded in the Java codebase itself.

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 50 }, // ramp up to 50 virtual users
    { duration: "2m", target: 50 }, // sustain
    { duration: "30s", target: 0 }, // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(99)<500"], // p99 under 500ms
    http_req_failed: ["rate<0.01"], // <1% error rate
  },
};

export default function () {
  const res = http.post(
    "https://api.example.com/api/orders",
    JSON.stringify({ sku: "SKU-1", quantity: 2 }),
    {
      headers: { "Content-Type": "application/json" },
    },
  );
  check(res, { "status is 201": (r) => r.status === 201 });
  sleep(1);
}
```

k6's `thresholds` serve the same purpose as Gatling's `.assertions(...)` — the test run exits non-zero if a threshold is breached, so it can gate a CI/CD pipeline the same way a failing unit test would.

## 5. Gatling vs. k6

| Aspect | Gatling | k6 |
| --- | --- | --- |
| Scripting language | Scala/Java DSL | JavaScript |
| Best fit | Java-centric teams wanting load tests alongside the codebase, detailed HTML reports | Cross-team/platform ownership, tight Grafana/CI integration, lighter-weight scripts |
| Reporting | Rich built-in HTML report per run | Console output + Grafana Cloud/dashboard integration |
| Resource footprint | Runs on the JVM — heavier per virtual user | Written in Go — very low overhead per virtual user, can simulate more load from one machine |

## 6. Where to Run Load Tests

Never load-test production without warning anyone, and never load-test a shared staging environment other teams depend on without coordinating first — a load test is, by design, indistinguishable from a mini denial-of-service. Run against a dedicated performance environment sized like production, or a staging slot explicitly reserved for the test window.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Assert on percentiles, not averages | A p99 threshold catches tail latency problems an average would hide. |
| Include a soak test, not just short load bursts | Short load tests miss slow leaks and gradual degradation that only appear after sustained traffic. |
| Run load tests in CI against a dedicated environment | Never point a load test at shared staging or production without explicit coordination. |
| Set explicit pass/fail thresholds, not just eyeballed dashboards | Gatling's `.assertions(...)` or k6's `thresholds` let a load test gate a pipeline the same way a failing unit test would. |
| Choose the tool by team ownership, not just features | Gatling fits a Java-centric team wanting tests alongside the codebase; k6 fits a platform/SRE team owning scripts separately. |
| Pair a load test finding with a profiler, not just a retest | Once a load test finds a bottleneck, a profiler pinpoints exactly where in the code it lives. |
