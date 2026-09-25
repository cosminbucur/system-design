Logs are the first of observability's three pillars — discrete, timestamped records of specific events, answering "what exactly happened, in detail?" They're the most granular signal: a metric tells you something changed, a trace tells you where, but a log line is usually what actually explains why.

## 1. Structured Logging

Plain text logs (`log.info("User " + userId + " logged in")`) are hard to query at scale. Structured logging emits key-value fields (usually as JSON) so log aggregators (ELK, Loki, Splunk) can filter and aggregate on them directly.

```java
// Unstructured — hard to query: "find all failed logins for this user in the last hour"
log.info("Login failed for user " + userId + " from IP " + ipAddress);

// Structured (SLF4J + a JSON encoder like logstash-logback-encoder)
log.atInfo()
    .addKeyValue("event", "login_failed")
    .addKeyValue("userId", userId)
    .addKeyValue("ipAddress", ipAddress)
    .log("Login failed");
```

```json
{
  "timestamp": "2026-09-15T10:30:00Z",
  "level": "INFO",
  "event": "login_failed",
  "userId": "u-123",
  "ipAddress": "10.0.0.5",
  "message": "Login failed"
}
```

Never log PII or secrets in plaintext (passwords, full card numbers, tokens) — mask or omit them; log aggregators retain data far longer and more widely accessible than you expect.

## 2. Log Levels — Use Them Deliberately

| Level | When to use                                                                                                                                                      |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ERROR | Something failed and needs attention — an unhandled exception, a failed payment, a broken integration. Should be rare enough that every one is worth looking at. |
| WARN  | Recoverable or degraded situations — a retry succeeded, a fallback was used, a deprecated API was called.                                                        |
| INFO  | Significant business/application events — order placed, user registered, service started.                                                                        |
| DEBUG | Detailed internal flow, useful when actively diagnosing — too noisy for production by default.                                                                   |

A common anti-pattern: logging every `4xx` as `ERROR`. A validation failure or "not found" is a normal business outcome, not a system fault — reserve `ERROR` for unexpected/unhandled situations.

## 3. Correlation IDs and MDC

In a single request within one service, you still need to tie every log line back to the specific request that produced it — especially under concurrent traffic where logs from different requests interleave.

`MDC (Mapped Diagnostic Context)` is the mechanism that makes this possible: it's a thread-local map, provided by the logging framework (SLF4J/Logback), that holds key-value pairs for the duration of one request. Every log statement made on that thread automatically picks up whatever is currently in the map, without having to pass the correlation ID explicitly into every method and log call. Put a value in once, at the request boundary, and every log line downstream carries it for free.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId); // thread-local, included in every log line for this request
        try {
            response.setHeader("X-Correlation-Id", correlationId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // always clear — thread pools reuse threads across requests
        }
    }
}
```

With a logging pattern like `%X{correlationId}`, every log line during that request automatically includes the ID — grep by it to reconstruct the full sequence of events for one request. `MDC.clear()` in `finally` is essential: thread pools reuse threads, and a forgotten context leaks into the next unrelated request's logs.

A correlation ID works within one service; once a request crosses into another service, the same ID needs to propagate as part of a distributed trace — covered in more depth in traces.

## 4. Best Practices

| Practice                                                             | Recommendation                                                                                                                                         |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Log structured, not free text                                        | Emit key-value fields (JSON) so aggregators can filter/query without regex-parsing messages.                                                           |
| Use log levels deliberately                                          | Reserve ERROR for genuinely unexpected failures; routine 4xx/business outcomes are INFO/WARN at most.                                                  |
| Never log secrets or PII                                             | Mask card numbers, tokens, passwords before they ever reach a log line — logs are retained and searched far more broadly than most people assume.      |
| Propagate correlation IDs through every log line in a request        | Every log line for a request should carry the same ID, making it possible to grep and reconstruct the full sequence of events.                         |
| Always clear MDC in a `finally` block                                | Thread pools reuse threads — a forgotten context leaks into the next unrelated request's logs.                                                         |
| Instrument logs to complement metrics and traces, not duplicate them | A log explains _why_ something happened; lean on metrics for _how often_ and traces for _where_ — don't try to answer all three from log volume alone. |
