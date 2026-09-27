# Idempotency and Deduplication in Distributed Systems

Distributed systems are designed for scalability, but they also introduce uncertainty. If you can't tell whether a unit of work was already processed, you're one network glitch away from double-charging your user.

---

## What Are We Talking About?

In distributed systems, especially those using message queues (like Kafka, Redis Streams, and AWS SQS), a unit of work—often called a job, event, or task—is handed off from one service to another.

Because queues typically offer **at-least-once delivery**, your consumer (a worker process) might receive the same job multiple times. This is not a bug—it's a fundamental property of distributed systems.

A message might be:

- Delivered twice.
- Partially processed before a crash.
- Acknowledged after the worker dies, causing a re-delivery.
- Lost and retried due to a transient network partition.

---

## Real-World Failures from Duplicate Processing

When duplicate jobs aren't handled properly, serious real-world issues occur:

- **Double-charging users** during a payment retry.
- **Sending duplicate emails** or push notifications.
- **Deducting inventory twice** for a single order.
- **Generating multiple invoices** for one transaction.
- **Triggering external APIs** more than once.

---

## The Core Scenario: A Worker Crash

Let's say you have a worker processing jobs from a Redis Stream or message queue:

1. Worker pulls a job from the queue.
2. Worker updates the database or calls an external payment gateway.
3. Worker crashes or experiences a network failure _before_ marking the job as completed.

On restart, the message queue sees that the job was never acknowledged. The queue re-delivers the exact same job to a new worker. Unless your logic was idempotent or deduplicated, **the job just ran twice.**

---

## But… Isn't Retry the Whole Point?

Yes, retries are expected and necessary in distributed architecture. But you should retry **only when it's safe**.

- ✅ **Retrying a job that safely fails** = Good engineering.
- ❌ **Retrying a job that has already succeeded or produced side effects** = Dangerous.

This leads us to a core truth: **You must make your job logic idempotent to safely support retries.**

---

## Key Definitions: Idempotency vs. Deduplication

### 1. Idempotency

A function or operation is **idempotent** if calling it multiple times produces the exact same side effects as calling it once.
$$f(f(x)) = f(x)$$

In backend engineering:

- Re-processing a job doesn't create duplicate database records.
- External APIs aren't triggered multiple times.
- Side effects are tracked and safely guarded.

### 2. Deduplication

**Deduplication** is the mechanism of detecting whether a specific job or event key has already been seen or processed, short-circuiting execution before any side effects happen.

Idempotency and deduplication work hand-in-hand:

- **Idempotency** makes retries safe.
- **Deduplication** prevents unnecessary retries.

---

## Implementing Deduplication with Redis

A common pattern for deduplication involves using an **atomic check-and-set** operation in Redis using `SET` with `NX` (Not Exists) and `EX` (Expiration Time):

```text
1. Acquire key: SET idempotency_key {job_id} NX EX 3600
2. If key exists -> Skip job (already processed or currently processing).
3. If key set successfully -> Process job side-effects.
4. Mark job as COMPLETED in DB / state store.
```

### Why this pattern works:

- **`NX` (Not Exists):** Ensures atomic execution so only one worker process can acquire the key.
- **`EX` (TTL):** Prevents deadlocks if a worker crashes midway through execution.
- **Explicit Lifecycle Management:** If the job succeeds, mark it done. If it fails transiently, clear the lock key to allow safe retries.

---

## Why Not Just Use Job Status in the Database?

A common question is: _"Why can't I just check `SELECT status FROM jobs WHERE id = 123`?"_

Relying solely on job status in a database introduces two major issues:

1. **Race Conditions:** Two workers pulling duplicate messages concurrently might both read `status = PENDING` before either updates it.
2. **Side-Effect Reliance:** If a job calls an external service (e.g., Stripe API) before updating the database status, a crash between those two steps leaves the database out of sync with the real-world side effect.

---

## Summary Architecture Checklist

To build resilient queue consumers, your architecture should include:

- ✅ **Message Broker:** Redis Streams, Kafka, or AWS SQS with at-least-once delivery.
- ✅ **Deduplication Key:** A unique, deterministic key generated per distinct unit of work.
- ✅ **Atomic Guard:** Redis or database unique constraint to prevent race conditions.
- ✅ **Idempotent Handlers:** Business logic structured so retrying side-effects is safe.
- ✅ **Dead Letter Queue (DLQ):** To isolate permanently failing messages after max retries.

---

## Final Thoughts

Distributed systems _will_ retry operations. Workers _will_ crash. Networks _will_ partition.

If you aren't actively defending against duplicate delivery at the consumer level, your system will eventually execute duplicate actions.

- **Idempotency** makes retrying safe.
- **Deduplication** prevents unnecessary processing.

Together, they form the cornerstone of reliable distributed system design.
