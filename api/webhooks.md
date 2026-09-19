A webhook inverts the normal API call direction: instead of a client polling "has anything happened yet," the server proactively sends an HTTP request to a URL the client registered in advance, the moment something happens. It's the standard integration mechanism when you need to notify a party you don't operate infrastructure for — a customer's system, a third-party partner — which is exactly the situation message queues doesn't fit (a queue assumes you control both the producer and the consumer's connection to your broker).

## 1. Webhooks vs. Polling vs. Message Queues — Different Integration Boundaries

All three notify a consumer that something happened; the right choice depends on who controls each side.

| Mechanism     | Fits when                                                                                         | Doesn't fit when                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Polling       | Consumer can tolerate delay; simplest to implement on both sides                                  | Wastes resources on both sides if events are rare; adds latency between event and consumer awareness              |
| Message queue | You control both producer and consumer, and can share broker infrastructure                       | The consumer is a separate organization/system that can't (or won't) connect to your internal broker              |
| Webhook       | The consumer is external, has its own public HTTP endpoint, and wants near-real-time notification | The consumer has no publicly reachable endpoint, or genuinely needs guaranteed ordering/replay from a durable log |

This is exactly the async-notification alternative flagged in API design for long-running operations — a webhook lets you tell the client "done" instead of making them poll a status endpoint.

## 2. Webhook Payload Shape

A webhook payload should look a lot like the domain events or the audit events — because it's answering the same fundamental question ("what happened, to what, when") for an audience outside your own system.

```json
{
  "id": "evt_8f3a1c2b",
  "type": "payment.completed",
  "createdAt": "2026-09-15T10:30:00Z",
  "apiVersion": "2026-06-01",
  "data": {
    "paymentId": "pay_123",
    "amount": "49.99",
    "currency": "USD",
    "status": "completed"
  }
}
```

```java
public record WebhookEvent(
    String id,           // unique event ID — the basis for idempotent processing
    String type,          // "payment.completed", "account.updated" — lets one endpoint handle many event types
    Instant createdAt,
    String apiVersion,    // payload shape versioning
    Map<String, Object> data
) {}
```

Include a stable, unique `id` per event and a `type` field even if you only send one kind of event today — both cost nothing up front and are exactly what's missing (and painful to retrofit) the first time a consumer needs deduplication or wants to filter by event type.

## 3. Delivery Is At-Least-Once — Consumers Must Be Idempotent

A webhook sender can't know for certain whether a delivery succeeded if the response times out or the connection drops after the receiver actually processed it — the only safe assumption is **at-least-once delivery**.

```java
@PostMapping("/webhooks/payments")
public ResponseEntity<Void> handlePaymentWebhook(@RequestBody WebhookEvent event) {
    if (processedEventRepository.existsById(event.id())) {
        return ResponseEntity.ok().build(); // already handled this exact event — safe no-op, not an error
    }

    paymentService.handlePaymentEvent(event);
    processedEventRepository.save(new ProcessedEvent(event.id()));
    return ResponseEntity.ok().build();
}
```

Deduplicate by the event's own `id`, never by inferring "this looks like a duplicate" from the payload contents alone — two genuinely distinct events can have identical-looking data.

## 4. Verifying Authenticity: HMAC Signatures

A webhook receiver endpoint is, by necessity, publicly reachable — anyone who finds the URL could send a fake request pretending to be the real sender. The standard defense is an HMAC signature: the sender computes a signature over the payload using a shared secret, sends it as a header, and the receiver recomputes and compares it before trusting the payload at all.

```java
// Sender side: sign the payload before sending
public String signPayload(String payload, String secret) {
    Mac hmac = Mac.getInstance("HmacSHA256");
    hmac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
    byte[] signatureBytes = hmac.doFinal(payload.getBytes(StandardCharsets.UTF_8));
    return Hex.encodeHexString(signatureBytes);
}
```

```java
// Receiver side: verify BEFORE deserializing/trusting the payload
@PostMapping("/webhooks/payments")
public ResponseEntity<Void> handlePaymentWebhook(
        @RequestHeader("X-Webhook-Signature") String signature,
        @RequestBody String rawPayload) {

    String expectedSignature = signPayload(rawPayload, webhookSecret);
    if (!MessageDigest.isEqual(expectedSignature.getBytes(), signature.getBytes())) {
        // constant-time comparison — a naive .equals() leaks timing information
        // that could help an attacker guess the correct signature byte by byte
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }

    WebhookEvent event = objectMapper.readValue(rawPayload, WebhookEvent.class);
    // ... proceed only now that authenticity is confirmed
}
```

`MessageDigest.isEqual` (or an equivalent constant-time comparison) matters specifically because it doesn't return early on the first mismatched byte — a naive string comparison's early exit is exactly the kind of timing side-channel.

## 5. Preventing Replay Attacks

A valid, correctly-signed payload captured once (e.g., from network logs, or a compromised intermediary) could be replayed later to trigger the same action twice — sign a timestamp along with the payload, and reject anything too old.

```java
long timestamp = Long.parseLong(request.getHeader("X-Webhook-Timestamp"));
if (Math.abs(Instant.now().getEpochSecond() - timestamp) > 300) { // reject anything older than 5 minutes
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
}
// sign timestamp + payload together, not just the payload — otherwise the timestamp itself could be tampered with
String expectedSignature = signPayload(timestamp + "." + rawPayload, webhookSecret);
```

## 6. Sender-Side Retries With Exponential Backoff

The receiver's endpoint can be temporarily down, slow, or briefly returning errors — the sender should retry, but not immediately and not forever.

```java
@Retryable(
    retryFor = { WebhookDeliveryException.class },
    maxAttempts = 6,
    backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 60000) // 1s, 2s, 4s, 8s, 16s, 32s (capped)
)
public void deliver(WebhookEvent event, String targetUrl) {
    ResponseEntity<Void> response = restClient.post()
        .uri(targetUrl)
        .body(event)
        .retrieve()
        .toBodilessEntity();

    if (!response.getStatusCode().is2xxSuccessful()) {
        throw new WebhookDeliveryException(event.id(), targetUrl, response.getStatusCode());
    }
}

@Recover
public void onDeliveryExhausted(WebhookDeliveryException ex, WebhookEvent event, String targetUrl) {
    deadLetterWebhookRepository.save(new FailedWebhook(event, targetUrl, ex.getMessage()));
    // same dead-letter idea as message queues — a webhook that never succeeds needs
    // a landing place for investigation, not silent, permanent loss
}
```

Adding random jitter to the backoff delay (rather than a fixed schedule) avoids many failed webhooks all retrying at the exact same moment and creating a synchronized load spike against a recovering receiver — the same thundering-herd concern as cache stampede mitigation.

## 7. Fast Acknowledgment, Async Processing

A receiver should respond `2xx` quickly and do the actual work afterward — a slow response risks the sender's own timeout firing and triggering a retry for an event that's actually still being processed, producing an unnecessary duplicate delivery.

```java
@PostMapping("/webhooks/payments")
public ResponseEntity<Void> handlePaymentWebhook(@RequestBody WebhookEvent event) {
    verifySignature(event); // fast — must happen synchronously, before acknowledging

    webhookProcessingQueue.publish(event); // hand off to internal async processing
    return ResponseEntity.ok().build(); // acknowledge immediately; the sender's job is done
}
```

This is the same "accepted, processed later" shape as the async API pattern — the webhook receiver is itself effectively producing an internal message for further, decoupled processing, rather than doing the real work inline within the HTTP request/response cycle.

## 8. Versioning Webhook Payloads

Once a partner has built an integration against a payload shape, changing that shape breaks them silently, often without you finding out until they complain. Include an explicit `apiVersion` in the payload (or as a header), and only add fields — never remove or repurpose an existing one — without bumping that version and giving consumers a migration path.

## 9. Operational Visibility: Delivery Logs

Because a webhook crosses an organizational boundary, the receiver's own logs won't show you (the sender) anything about a failed delivery — you need to keep your own delivery history so a partner asking "why didn't I receive event X" can actually be answered.

```java
public record WebhookDeliveryAttempt(
    String eventId, String targetUrl, Instant attemptedAt,
    int httpStatus, String responseBody, boolean succeeded
) {}
```

Exposing this delivery history (even just internally, ideally to the receiving customer as a dashboard — the pattern Stripe and GitHub both use) turns "our webhook isn't working and we don't know why" from a support escalation into a self-service debugging step.

## 10. Testing Webhooks Locally

A webhook receiver needs a publicly reachable URL, which a local development machine doesn't have — tunneling tools (ngrok, smee.io, Cloudflare Tunnel) expose a local port to a temporary public URL for development/testing, so you can iterate on receiver logic against real sender traffic without deploying first.

## 11. Best Practices

| Practice                                                                | Recommendation                                                                                                                  |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Sign every payload with HMAC and verify with a constant-time comparison | A naive string comparison leaks timing information that helps an attacker forge a valid signature.                              |
| Include and check a timestamp to prevent replay                         | A captured, valid signed payload should not be replayable indefinitely.                                                         |
| Deduplicate by the event's own unique ID                                | Delivery is at-least-once — never assume a webhook arrives exactly once, and never infer duplicates from payload content alone. |
| Acknowledge fast, process asynchronously                                | A slow synchronous handler risks the sender's timeout firing and re-delivering an event that's still mid-processing.            |
| Retry with exponential backoff and jitter, then dead-letter             | A permanently failing delivery needs a landing place for investigation, not infinite retries or silent loss.                    |
| Version the payload shape explicitly, and only add fields               | Breaking an external partner's integration silently is far costlier to discover and fix than an internal API change.            |
| Keep your own delivery history, not just relying on the receiver's logs | You're the only party positioned to answer "why didn't this webhook arrive".                                                    |
