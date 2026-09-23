Retry and fallback are two different answers to the same moment: a call to a dependency just failed — now what? Retry says "try again, this might have been transient." Fallback says "give up on this attempt and do something else instead that still lets the caller proceed." They're often used together — retry a bounded number of times first, and only fall back once retries are exhausted — but they represent genuinely different assumptions about the failure, and applying the wrong one to the wrong kind of failure causes real problems.

## 1. Retry: For Transient Failures

A retry re-attempts a failed call, on the assumption that the failure was momentary (a dropped packet, a brief GC pause on the other side, an instance that was mid-restart) rather than a persistent problem that a second attempt will hit again identically.

```java
@Service
public class InventoryClientService {

    @Retry(name = "inventoryService", fallbackMethod = "fallbackStock")
    public StockLevel getStock(String sku) {
        return inventoryClient.getStock(sku);
    }

    private StockLevel fallbackStock(String sku, Throwable ex) {
        return StockLevel.unknown(sku);
    }
}
```

```yaml
resilience4j.retry:
  instances:
    inventoryService:
      max-attempts: 3
      wait-duration: 200ms
      retry-exceptions:
        - java.io.IOException
        - org.springframework.web.client.ResourceAccessException
      ignore-exceptions:
        - com.example.ValidationException # retrying a 400 won't fix a 400
```

Notice `ignore-exceptions`: retry only makes sense for failures that could plausibly succeed on a second attempt. A validation error, a `404`, or an authorization failure will fail identically every time — retrying those just adds latency and load for no benefit.

## 2. Retry Strategies: Fixed, Backoff, and Jitter

| Strategy | Behavior | Risk without it |
| --- | --- | --- |
| Fixed delay | Wait the same interval between every attempt | Fine for low-volume calls; can synchronize retries under load |
| Exponential backoff | Each retry waits longer than the last (e.g., 200ms, 400ms, 800ms) | Prevents hammering an already-struggling dependency with immediate retries |
| Jitter | Add randomness to the backoff delay | Without it, many clients that failed at the same moment retry in lockstep, creating synchronized load spikes ("thundering herd") |

```yaml
resilience4j.retry:
  instances:
    inventoryService:
      max-attempts: 4
      wait-duration: 200ms
      enable-exponential-backoff: true
      exponential-backoff-multiplier: 2
      enable-randomized-wait: true # jitter — spreads retries out instead of all firing in sync
```

Exponential backoff with jitter is the standard combination for anything calling a shared dependency under real load — a fixed short delay retried by many clients simultaneously can turn a brief blip into a self-inflicted spike exactly when the dependency was starting to recover.

## 3. Idempotency: The Precondition for Safe Retries

Retrying is only safe when the operation is idempotent — running it twice has the same effect as running it once. This is the single most important thing to verify before enabling retry on any given call.

```java
// DANGEROUS to retry blindly: a network timeout doesn't tell you whether the charge
// actually succeeded on the far end before the response was lost
paymentClient.chargeCard(request); // if this times out, did the card get charged or not?

// SAFER: make the operation idempotent with a client-generated idempotency key,
// so a retried request with the same key is recognized and not double-processed
paymentClient.chargeCard(request.withIdempotencyKey(requestId));
```

A `GET` is naturally idempotent and always safe to retry. A `POST` that creates a new resource is not, unless the API explicitly supports an idempotency key so the server can recognize and deduplicate a retried request. Retrying a non-idempotent operation without this safeguard risks the exact failure mode retries are supposed to prevent: a duplicate charge, a duplicate order, a duplicate email.

## 4. Fallback: For When Retrying Won't Help

A fallback provides an alternative result when the primary call has failed (whether immediately, or after retries were exhausted) rather than propagating the failure to the caller. It's the same concept introduced alongside circuit breakers, but it applies just as directly here — any failed call needs a decision about what happens next, not just whether to try it again.

```java
@Service
public class RecommendationClientService {

    @Retry(name = "recommendationService")
    @CircuitBreaker(name = "recommendationService", fallbackMethod = "fallbackRecommendations")
    public List<Product> getRecommendations(String customerId) {
        return recommendationClient.getRecommendations(customerId);
    }

    private List<Product> fallbackRecommendations(String customerId, Throwable ex) {
        return productService.getPopularProducts(); // degrade gracefully, don't fail the whole page
    }
}
```

| Fallback strategy | Example | When it's appropriate |
| --- | --- | --- |
| Default/generic response | Show "popular products" instead of personalized recommendations | The feature is non-essential — a degraded experience beats no experience |
| Cached/stale data | Last-known price instead of a live lookup | Slight staleness is acceptable for this specific data |
| Explicit failure | Return `503` with a clear error instead of guessing | The data is essential and no safe approximation exists (e.g., an account balance) |

The same rule from circuit breakers applies here just as strongly: never let a fallback quietly fabricate a false success for something where being wrong is worse than failing visibly — a fallback that pretends a payment succeeded when it didn't turns a visible outage into a much harder-to-find correctness bug.

## 5. Combining Retry and Fallback

The common real pattern is layered: retry a bounded number of times for genuinely transient failures, and only fall back once retries are exhausted — never retry indefinitely, and never skip straight to a fallback for a failure that a second attempt would likely have resolved cleanly.

```
Call fails
  → Retry (up to N times, with backoff + jitter)
      → still failing → Fallback (degrade gracefully, or fail explicitly)
```

Ordering the annotations/config matters in frameworks like Resilience4j — retry should wrap the raw call, and the circuit breaker/fallback should sit around the retrying call, so an already-open circuit breaker skips the retries entirely rather than retrying into a dependency already known to be down.

## 6. Best Practices

| Practice | Recommendation |
| --- | --- |
| Only retry idempotent operations, or ones with an idempotency key | Retrying a non-idempotent call blindly risks duplicate side effects — the exact problem retries exist to prevent. |
| Use exponential backoff with jitter, not a fixed short delay | Prevents synchronized retry storms from many callers hitting a recovering dependency at once. |
| Explicitly exclude non-retryable failures | Validation errors, `404`s, and auth failures will fail identically every time — retrying them only adds latency. |
| Cap the number of retry attempts | Unbounded retry just delays an inevitable failure while adding load to an already-struggling dependency. |
| Design a real fallback, not a placeholder that hides failure | Decide deliberately: degrade gracefully, use cached data, or fail explicitly — based on what the data actually means. |
| Never let a fallback fabricate a false success | Especially for financial/transactional outcomes — a hidden fallback masking a real failure is worse than a visible one. |
| Let a circuit breaker skip retries once it's open | Retrying into a dependency already known to be down just adds load without any real chance of success. |
