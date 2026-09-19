The polling publisher is the simplest way to get events out of a transactional outbox table: a background job periodically queries for unpublished rows, publishes each one to a message broker, and marks it processed. It's the low-infrastructure counterpart to CDC — no connector, no transaction-log tailing, just a scheduled query — at the cost of publish latency and database load that CDC avoids.

## 1. The Core Idea

A scheduled job runs on an interval, selects rows that haven't been published yet, publishes them, and marks them done — the same read-publish-mark cycle repeated forever.

```java
@Scheduled(fixedDelay = 500) // poll every 500ms
public void publishPendingEvents() {
    List<OutboxEvent> pending = outboxRepository.findTop100ByProcessedFalseOrderByCreatedAtAsc();
    for (OutboxEvent event : pending) {
        kafkaTemplate.send(event.getTopic(), event.getPayload());
        outboxRepository.markProcessed(event.getId());
    }
}
```

This is deliberately simple — no new infrastructure beyond the application itself and a scheduler, which is exactly why it's often the first implementation reached for, and often good enough for a service that isn't yet at a scale where the tradeoffs below start to bite.

## 2. The Two Costs: Latency and Database Load

Every polling design pays for its simplicity in two specific, measurable ways.

| Cost | Cause | Mitigation |
| --- | --- | --- |
| Publish latency | An event can sit unpublished for up to one full poll interval before the next run picks it up | Shorten the interval — but this directly increases the second cost |
| Database load | Every poll interval issues a query against the live outbox table, whether or not there's anything new to publish | Index the query's filter/sort columns; widen the interval — but this directly increases latency |

These two costs are in direct tension — there's no interval that minimizes both simultaneously, only a tradeoff point you choose deliberately based on how much latency the downstream consumers can actually tolerate. This tension is exactly why CDC (tailing the database's transaction log instead of querying it) is generally preferred at real scale: it removes the tradeoff entirely rather than just tuning it, since there's no polling query to slow down or interval to widen.

## 3. Ordering: Getting the `ORDER BY` Right

Publishing order matters whenever downstream consumers rely on event sequence (e.g., an `OrderCreated` event must never be processed after an `OrderShipped` event for the same order) — the polling query's sort order is the entire ordering guarantee this pattern provides.

```sql
-- created_at alone isn't a safe sort key if two rows can share the same timestamp
SELECT * FROM outbox WHERE processed = false ORDER BY created_at ASC, id ASC LIMIT 100;
```

Adding the primary key `id` as a tie-breaker after `created_at` is the standard fix for the same reason it matters in cursor pagination — a timestamp alone isn't guaranteed unique, and without a deterministic tie-breaker, rows with an identical timestamp could be published in an inconsistent order across different polling runs.

## 4. Concurrency: Multiple Instances Polling the Same Table

A service typically runs multiple instances for availability — but a naive polling query run by more than one instance at once will select and attempt to publish the same rows twice, duplicating events downstream.

```sql
-- Postgres: SKIP LOCKED lets concurrent pollers each grab a different, non-overlapping batch
SELECT * FROM outbox
WHERE processed = false
ORDER BY created_at ASC, id ASC
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE` locks the selected rows for the duration of the transaction; `SKIP LOCKED` tells a second concurrent poller to simply skip rows another poller already has locked rather than blocking and waiting for them — the two instances end up dividing the work instead of colliding on it. Without this, running more than one instance of the polling job requires an external coordination mechanism (a distributed lock, or designating only one instance as the active poller) to prevent duplicate publishing.

## 5. At-Least-Once Delivery Is Still the Guarantee

Even with `SKIP LOCKED` solving the concurrency problem, a crash between "publish the event" and "mark it processed" still means the same event gets published again on the next poll — the polling publisher gives at-least-once delivery, never exactly-once, same as CDC-based publishing.

```java
kafkaTemplate.send(event.getTopic(), event.getPayload()); // succeeds
// <-- crash here, before the next line runs
outboxRepository.markProcessed(event.getId()); // never happens — this event gets republished on next poll
```

This is why every consumer of these events still needs to be idempotent (already covered in API idempotency and CDC) — no amount of careful polling-job design eliminates the possibility of a duplicate delivery, it only makes duplicates rare rather than impossible.

## 6. When Polling Is Actually the Right Choice

Despite CDC's advantages, polling remains a reasonable default in several real situations, not just a stepping stone to eventually replace.

| Situation | Why polling still fits |
| --- | --- |
| Low event volume, latency tolerance in the hundreds of milliseconds to seconds | The database load and latency costs are both negligible at low volume |
| No existing CDC infrastructure (Debezium, Kafka Connect) in the organization | Avoids introducing an entirely new operational component for a single service's needs |
| A small team without dedicated platform/infra ownership for CDC connectors | CDC's operational surface (connector health, log retention, schema evolution) needs ongoing ownership polling doesn't require |
| Early-stage service where requirements are still shifting | Simpler to build, understand, and modify while the outbox usage pattern itself is still being figured out |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Always add a tie-breaker to the `ORDER BY` | `created_at` alone can have ties — pairing it with the primary key guarantees deterministic, repeatable ordering. |
| Use `FOR UPDATE SKIP LOCKED` when running multiple instances | Without it, concurrent pollers will select and publish the same rows, producing duplicate downstream events. |
| Treat delivery as at-least-once, and make consumers idempotent accordingly | A crash between publish and mark-processed always risks a duplicate — this is a property of the pattern, not a bug to eliminate. |
| Index the columns the polling query filters and sorts on | An unindexed `WHERE processed = false ORDER BY created_at` degrades to a full table scan as the outbox table grows. |
| Batch the query with a `LIMIT`, don't select every unprocessed row at once | Keeps each poll cycle's work bounded and predictable, especially important after any backlog builds up (e.g., after downtime). |
| Migrate to CDC once polling's latency or database load actually becomes a measured problem | Polling is a legitimate default, not a mistake — switch when the numbers justify the added operational complexity, not preemptively. |
