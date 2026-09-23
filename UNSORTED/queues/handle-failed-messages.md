- process
- retry
- exponential backoff (2,4,8s)
- dead letter queue

**Best practice**

Store error stacks, attempt counts, timestamps, and original payloads

## Ensure idempotency for consumers

- use message IDs or deduplication keys
- maintain state to track processed messages

- table `processed_messages` with id (consumer_name, message_id)
- insert before processing, rollback on failure

- ℹ️ poison pill - is a specific type of corrupted, malformed, or un-parsable message that enters a queue and causes a consumer to repeatedly crash or fail.
