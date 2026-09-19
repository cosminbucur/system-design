Publish/Subscribe (pub/sub) is a messaging pattern where publishers send messages to a named channel without knowing who (if anyone) is listening, and subscribers receive messages from a channel without knowing who published them. The defining property is this mutual unawareness — publishers and subscribers are decoupled in time, space, and identity, communicating only through the channel itself, already touched on briefly in message queues but worth understanding as its own pattern independent of any specific broker.

## 1. The Core Idea: Decoupling Through a Channel

A publisher's only job is to emit a message to a topic/channel. It never calls a subscriber directly, never knows how many subscribers exist, and never waits for them to process anything.

```java
// Publisher — has no reference to any subscriber, doesn't know or care who's listening
eventPublisher.publish("order.created", new OrderCreatedEvent(orderId, customerId));

// Subscriber A — reacts independently, added without ever modifying the publisher
@Subscribe(topic = "order.created")
public void sendConfirmationEmail(OrderCreatedEvent event) { /* ... */ }

// Subscriber B — a second, completely independent subscriber to the same event
@Subscribe(topic = "order.created")
public void recordAnalytics(OrderCreatedEvent event) { /* ... */ }
```

Adding Subscriber B required zero changes to the publisher or to Subscriber A — this is the practical payoff of the decoupling: new reactions to an existing event are purely additive, which is the same benefit the Observer design pattern describes at the level of in-process objects, just extended across process/service boundaries.

## 2. Pub/Sub vs. Point-to-Point: One Message, How Many Consumers

The distinction that matters most: in point-to-point messaging, a message is delivered to exactly one consumer, even if several are listening (they compete for it). In pub/sub, a message is delivered to *every* subscriber, independently.

| Pattern | Delivery | Typical use |
| --- | --- | --- |
| Point-to-point (queue) | Exactly one consumer gets each message — competing consumers share the workload | Task processing, load-balanced job execution |
| Publish/Subscribe (topic) | Every subscriber gets its own copy of each message | Broadcasting a fact/event to multiple independent, unrelated interested parties |

This is a decision about the *shape* of the problem, not a technology choice — "should exactly one thing happen per message, split across workers" is point-to-point; "should every interested party independently react" is pub/sub. The same broker (Kafka, RabbitMQ) can implement either, chosen via configuration (consumer groups, exchange type) rather than requiring different software.

## 3. Fan-Out: The Defining Behavior

Fan-out means a single published message results in multiple independent deliveries — one per subscriber, each unaware of the others.

```
Publisher publishes ONE message to topic "order.created"
         │
         ├──> Subscriber A (email service) — gets its own copy
         ├──> Subscriber B (analytics service) — gets its own copy
         └──> Subscriber C (inventory service) — gets its own copy

Each subscriber processes independently — one failing or being slow doesn't block the others.
```

This is exactly what a message broker's fanout exchange (RabbitMQ) or having multiple independent consumer groups read the same topic (Kafka) provides — both are already covered in more mechanical depth in message queues; the point here is the pattern itself, not either specific implementation.

## 4. Topics, Wildcards, and Filtering

Most pub/sub systems let subscribers narrow what they receive, either by subscribing to a specific named topic or by pattern-matching against a family of related topics.

```java
// Subscribe to one exact topic
subscribe("order.created", handler);

// Subscribe to a wildcard pattern — receives order.created, order.cancelled, order.shipped, etc.
subscribe("order.*", handler);
```

This filtering is what keeps pub/sub practical at scale — without it, every subscriber would need to receive every message ever published and filter client-side, wasting bandwidth and processing on messages it was never going to act on anyway.

## 5. The Slow Subscriber Problem

Because publishers don't wait for subscribers, a slow or backed-up subscriber doesn't block the publisher or other subscribers — but it does raise the question of what happens to messages that subscriber hasn't caught up on yet.

| Approach | Behavior |
| --- | --- |
| Broker retains messages (Kafka-style log) | A slow subscriber simply reads from where it left off later — no message loss, but the subscriber falls behind in near-real-time terms |
| Broker doesn't retain (fire-and-forget, e.g. plain Redis Pub/Sub) | A subscriber that's offline or too slow simply misses messages published during that window — no replay possible |
| Bounded buffer per subscriber, with backpressure | The broker slows the publisher down (or drops messages) once a subscriber's buffer fills, trading throughput for guaranteed delivery to that subscriber |

Choosing between these is really choosing how much "at least once, eventually" matters for a given subscriber — a real-time price ticker UI can tolerate dropped updates (a newer one supersedes it anyway), while a billing event absolutely cannot be silently missed.

## 6. Common Pub/Sub Implementations, Briefly

| System | Notes |
| --- | --- |
| Kafka (topics + independent consumer groups) | Retains messages for a configurable period — a new subscriber can replay history, not just receive what's published after it joins |
| RabbitMQ (fanout/topic exchange) | Classic broker-based pub/sub — messages not retained once delivered/acknowledged by default |
| Redis Pub/Sub | Extremely lightweight, no persistence — a subscriber that's offline misses everything published in the meantime |
| Cloud pub/sub services (SNS, Google Pub/Sub, Azure Service Bus topics) | Managed, broker-as-a-service versions of the same pattern, often paired with per-subscriber queues for durable delivery |
| WebSocket-based pub/sub (e.g., a browser subscribing to live updates) | Applies the same pattern at the client-facing edge — a server publishes an event, every connected client subscribed to that channel receives it |

## 7. Recognizing When Pub/Sub Applies

| Signal in the problem | Why pub/sub fits |
| --- | --- |
| Multiple, independent, unrelated services need to react to the same event | Fan-out delivers one message to every interested party without the publisher knowing who they are |
| New consumers of an event should be addable without touching existing code | Decoupling means adding Subscriber C never requires modifying the publisher or Subscriber A/B |
| Real-time updates need to reach many simultaneous listeners (live dashboards, chat, notifications) | Broadcast-by-design is the natural fit, especially over WebSockets |
| The workload should be split across workers, not duplicated to each | This is actually point-to-point, not pub/sub — a common source of confusing the two |

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Keep subscribers truly independent | A subscriber's failure or slowness should never block the publisher or any other subscriber — verify this holds for your chosen broker's configuration. |
| Choose retention deliberately, not by default | A log-style broker (Kafka) lets late/restarted subscribers catch up; a fire-and-forget one (plain Redis Pub/Sub) silently drops anything missed while offline. |
| Use topic naming/wildcards to avoid over-broad subscriptions | Filtering at the broker is cheaper than every subscriber receiving everything and discarding most of it client-side. |
| Don't use pub/sub when exactly-one-consumer processing is actually what you need | That's point-to-point queuing — using pub/sub there means every subscriber redundantly processes the same task. |
| Make each subscriber's handler idempotent | Most pub/sub delivery guarantees are at-least-once in practice — a handler that isn't safe to run twice will eventually cause a real bug. |
| Design new subscribers to be purely additive | If adding a new subscriber ever requires changing the publisher, the decoupling that makes pub/sub valuable has already been lost. |
