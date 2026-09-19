An API operation is idempotent when making the same call twice has the same effect as making it once — the second call either does nothing new or returns the same result, rather than repeating the side effect. This matters enormously in distributed systems because network failures are ambiguous: if a client sends a request and the connection drops before the response arrives, the client genuinely cannot tell whether the server processed it or not. Idempotency is what makes "just retry" a safe default answer to that ambiguity instead of a risk of duplicating the operation.

## 1. The Ambiguous Timeout Problem

```
Client sends: POST /api/payments { amount: 100, accountId: "acc-1" }
Client waits... connection times out
Client has NO WAY to know: did the server never receive it? receive but crash before processing?
                            process it fully and the response was lost on the way back?
```

All three of those are indistinguishable from the client's point of view, yet they call for different retry behavior. Blindly retrying a `POST` that creates a new payment risks charging the customer twice if the original request actually succeeded — idempotency is the mechanism that makes retrying safe regardless of which of the three actually happened.

## 2. Naturally Idempotent HTTP Methods

Some HTTP methods are idempotent by definition in the HTTP spec, and a correctly implemented API should honor that — retrying them is always safe.

| Method | Idempotent? | Why |
| --- | --- | --- |
| `GET` | Yes | Reading data has no side effect to repeat |
| `PUT` | Yes | Replacing a resource with the same representation twice leaves it in the same final state |
| `DELETE` | Yes | Deleting an already-deleted resource still ends with it deleted (typically a `404` on the second call, but the end state is identical) |
| `POST` | No | Creating a new resource each time is the default behavior — two identical `POST`s normally create two resources |

`POST` is the one that needs explicit help, precisely because "create a new order" has no natural way to recognize "wait, I already did this one" without additional information supplied by the client.

## 3. Idempotency Keys: Making POST Safe to Retry

The client generates a unique key (a UUID) once per logical operation and sends it with the request. The server remembers which keys it has already processed and, on a repeat, returns the original result instead of repeating the side effect.

```java
@PostMapping("/api/payments")
public ResponseEntity<Payment> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    Optional<Payment> existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get()); // already processed — return the same result, don't charge again
    }

    Payment payment = paymentService.charge(request, idempotencyKey);
    return ResponseEntity.status(201).body(payment);
}
```

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(255) NOT NULL UNIQUE, -- the DB constraint is what actually prevents a race
    account_id VARCHAR(255) NOT NULL,
    amount DECIMAL NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
```

The `UNIQUE` constraint on `idempotency_key` matters more than the application-level check above it — two concurrent retries of the same request (a real possibility, not just a theoretical race) could both pass the `findByIdempotencyKey` check before either commits, and only a database-level uniqueness constraint reliably prevents both from creating a duplicate payment.

## 4. Handling the Race Explicitly

Because the check-then-insert has a real race window, the insert itself needs to handle the constraint violation as an expected, not exceptional, outcome.

```java
public Payment charge(PaymentRequest request, String idempotencyKey) {
    try {
        Payment payment = new Payment(request, idempotencyKey);
        return paymentRepository.save(payment); // may throw on the UNIQUE constraint
    } catch (DataIntegrityViolationException ex) {
        // another concurrent request with the same key won the race — fetch and return its result instead
        return paymentRepository.findByIdempotencyKey(idempotencyKey)
            .orElseThrow(() -> ex); // truly unexpected if it's still missing
    }
}
```

## 5. What the Idempotency Key Actually Identifies

A key represents one specific *attempt* at an operation from the client's perspective, not the underlying business entity. The client is responsible for generating a new key for each genuinely new logical operation, and reusing the same key deliberately for retries of that same attempt.

```java
String idempotencyKey = UUID.randomUUID().toString();

// First attempt
httpClient.post("/api/payments", request, Map.of("Idempotency-Key", idempotencyKey));

// If it times out, retry with the SAME key — this is what makes the retry safe
httpClient.post("/api/payments", request, Map.of("Idempotency-Key", idempotencyKey));

// A genuinely new payment later gets a NEW key
String newKey = UUID.randomUUID().toString();
```

A common mistake is generating a fresh key on every retry attempt — that defeats the entire mechanism, since the server has no way to recognize a "new" key as a repeat of anything.

## 6. Key Expiry and Storage Scope

Idempotency keys shouldn't be remembered forever — most APIs (Stripe is the commonly cited reference implementation) expire them after a bounded window (commonly 24 hours), since a client is expected to retry within a reasonable time, not resend a request from a week-old key.

```java
@Scheduled(cron = "0 0 * * * *") // hourly cleanup
public void expireOldIdempotencyKeys() {
    idempotencyKeyRepository.deleteByCreatedAtBefore(Instant.now().minus(Duration.ofHours(24)));
}
```

Also worth being explicit about: if a retried request arrives with the same idempotency key but a *different* request body than the original, that's a client bug, not a legitimate retry — a correct implementation should detect the mismatch and reject it (typically `422`) rather than silently returning the original, unrelated result.

## 7. Idempotency vs. the Transactional Outbox Pattern

Idempotency keys make an individual client-to-service call safe to retry; the transactional outbox pattern makes a service's own downstream event publishing reliable despite the same kind of dual-write ambiguity. They solve the same underlying class of problem (an at-least-once delivery guarantee needs a way to detect duplicates) at two different points in a request's life — a client retrying a `POST`, and a consumer receiving a redelivered event — and in both cases the fix is the same shape: track what's already been processed, keyed by something the sender controls.

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Require an idempotency key for any non-idempotent write (`POST`) that has a real side effect | Especially payments, order creation, and anything with a cost to duplicating. |
| Enforce uniqueness at the database level, not just in application code | A check-then-insert without a `UNIQUE` constraint has a real race window under concurrent retries. |
| Have the client generate one key per logical operation and reuse it on retry | Generating a new key per retry attempt defeats the entire mechanism. |
| Detect a mismatched body on a reused key and reject it | Silently returning an unrelated cached result for a genuinely different request hides a real client bug. |
| Expire idempotency keys after a bounded window | A day is a common default — don't keep them (or their storage cost) indefinitely. |
| Return the original response, not just a generic "already processed" message | The client's retry logic usually expects the same shape of response it would have gotten the first time. |
| Treat `GET`/`PUT`/`DELETE` as idempotent by design, and keep them that way | Don't accidentally introduce side effects into these methods that would break the safety callers already assume. |
