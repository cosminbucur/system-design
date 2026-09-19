The Transactional Outbox pattern solves a specific consistency problem that shows up the moment a service needs to both update its own database and publish an event about that update: a database write and a message publish are two separate systems, and there's no way to wrap them in a single atomic transaction. Without a fix, you're stuck choosing which one to risk getting wrong — and both choices are bad.

## 1. The Problem: Dual Writes Aren't Atomic

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);              // write #1: the database
    messagePublisher.publish(new OrderPlacedEvent(order)); // write #2: the message broker
}
```

This looks fine, but the two writes aren't covered by the same transaction — the `@Transactional` boundary only protects the database write. Consider the failure windows:

- The database commit succeeds, then the process crashes (or the broker is briefly unreachable) before the publish happens → the order exists, but no event was ever sent, and nothing downstream ever finds out.
- The publish succeeds, but the database transaction then rolls back → an event went out for an order that doesn't actually exist.

Either way, the database and the rest of the system silently disagree about what happened — a correctness bug that's often invisible until someone notices a downstream service missing data it should have received.

## 2. The Fix: Write the Event to the Same Database, Same Transaction

Instead of publishing to the broker directly inside the business transaction, write the event as a row in an `outbox` table in the *same* database, as part of the *same* transaction as the business data change. A database transaction is atomic by nature — either both rows are committed, or neither is.

```java
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);                     // business data, same transaction
    outboxRepository.save(OutboxMessage.builder()
        .aggregateType("Order")
        .aggregateId(order.getId())
        .eventType("OrderPlaced")
        .payload(toJson(new OrderPlacedEvent(order)))
        .build());                                    // outbox row, same transaction
}
```

```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    processed BOOLEAN NOT NULL DEFAULT false
);
```

Now there's no dual-write problem at the point of the business transaction — the order and the fact that "an OrderPlaced event needs to go out" are committed together, atomically, or not at all. The actual publish to the message broker becomes a separate step, decoupled from the original request.

## 3. Getting Events Out of the Outbox: Two Approaches

Something still has to read the outbox table and actually publish to the broker. Two common ways to do that:

| Approach | How it works | Tradeoff |
| --- | --- | --- |
| Polling publisher | A background job periodically queries `WHERE processed = false`, publishes each row, then marks it processed | Simple to build, but introduces publish latency equal to the poll interval, and polling load on the database |
| Change Data Capture (CDC) | A tool (Debezium) tails the database's transaction log directly and streams new outbox rows to the broker automatically | Near real-time, no polling load, but adds a CDC connector as new operational infrastructure |

```java
@Scheduled(fixedDelay = 500)
public void publishPendingEvents() {
    List<OutboxMessage> pending = outboxRepository.findByProcessedFalseOrderByCreatedAtAsc();
    for (OutboxMessage message : pending) {
        kafkaTemplate.send("order-events", message.getPayload());
        message.setProcessed(true);
        outboxRepository.save(message);
    }
}
```

CDC-based publishing (Debezium reading Postgres's write-ahead log or MySQL's binlog) is generally preferred at scale — it eliminates polling entirely and publishes almost immediately after commit, but it does mean the database's replication log becomes part of the architecture, not just an internal implementation detail.

## 4. At-Least-Once Delivery, Not Exactly-Once

The outbox pattern guarantees the event will eventually be published if the business transaction committed — but it doesn't guarantee it's published exactly once. If the publisher crashes after sending to the broker but before marking the row processed, the same event gets sent again on restart.

```java
// Consumers of outbox-published events must be idempotent —
// they WILL occasionally see the same event more than once
@KafkaListener(topics = "order-events")
public void handle(OrderPlacedEvent event) {
    if (processedEventRepository.existsById(event.getEventId())) {
        return; // already handled this exact event, skip it
    }
    // ... actually process it
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

This is an unavoidable consequence of the pattern, not a bug to eliminate — every consumer of outbox-published events needs to be written to tolerate redundant delivery, typically by tracking already-processed event IDs the same way an idempotency key works for API calls.

## 5. Cleaning Up Processed Rows

An outbox table that only ever grows will eventually become a performance problem for both the publisher's polling query and the table itself. Processed rows need a retention/cleanup strategy — a scheduled job deleting rows older than some window, or partitioning the table by date and dropping old partitions outright, rather than letting it accumulate indefinitely.

## 6. When This Pattern Is Worth the Complexity

| Situation | Outbox needed? |
| --- | --- |
| A service updates its own data and must reliably notify other services | Yes — this is exactly the problem it solves |
| The event only needs to be "probably" delivered, occasional loss is tolerable | Probably not — the added table, publisher, and idempotent-consumer requirement may not be worth it |
| The service already uses event sourcing, where the event stream IS the source of truth | Not needed the same way — there's no separate business-data write to keep in sync with, since the event log already is the data |

The pattern earns its complexity specifically when losing an event silently would be a real correctness problem (an order placed with no downstream fulfillment ever triggered) rather than a minor, tolerable gap.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Write the outbox row in the same transaction as the business data | This atomicity is the entire point — the pattern doesn't work if the two writes aren't truly transactional together. |
| Make every consumer of outbox events idempotent | The pattern guarantees at-least-once delivery, never exactly-once — duplicate delivery is expected, not exceptional. |
| Prefer CDC (Debezium) over polling at any real scale | Removes polling latency and database load; polling is fine for lower-volume services where the added infrastructure isn't justified yet. |
| Clean up processed outbox rows on a schedule | An unbounded outbox table degrades both the publisher's query and general table performance over time. |
| Include enough context in the outbox payload to avoid a callback | The event payload should generally be self-contained, not require the consumer to call back into the publishing service for details. |
| Don't reach for this pattern when occasional event loss is truly tolerable | The added table, publisher process, and idempotency requirements are real costs — reserve it for genuinely correctness-sensitive events. |
