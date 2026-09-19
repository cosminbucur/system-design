Data streaming processes an unbounded, continuously arriving sequence of events as they happen, rather than waiting for a complete, bounded dataset to run a job against. Where message queues focus on reliably delivering individual messages between producers and consumers, stream processing focuses on computing something over the data in motion — running aggregations, joins, and transformations continuously as events flow through, producing results that update incrementally rather than once at the end of a batch run.

## 1. Streaming vs. Batch vs. Messaging — Three Related but Different Things

| | Message queue | Stream processing | Batch processing |
| --- | --- | --- | --- |
| What it's for | Reliably delivering individual messages between services | Computing over a continuous flow of events (aggregate, join, transform) | Computing over a large, bounded, already-collected dataset |
| Data shape | Discrete messages, consumed once each | An unbounded, ordered sequence of events | A finite, complete dataset |
| Latency | Near-immediate delivery | Near-immediate, incremental results | Minutes to hours, until the job finishes |
| Typical tool | Kafka, RabbitMQ (as a transport) | Kafka Streams, Flink, Spark Structured Streaming | Spring Batch, Spark (batch mode) |

A message broker like Kafka is often the transport underneath a streaming system — the stream processor reads from and writes back to Kafka topics — but the broker itself just moves messages; the stream processing layer is what computes running aggregates, windowed counts, or joins across multiple streams as the data flows.

## 2. Stateless vs. Stateful Stream Processing

A stateless transformation processes each event independently, with no memory of previous events — filtering, mapping, or reformatting one record at a time.

```java
// Stateless: each event transformed independently, no memory needed between events
KStream<String, Order> orders = builder.stream("orders");
KStream<String, OrderSummary> summaries = orders.mapValues(order -> toSummary(order));
```

A stateful transformation needs to remember something across events — a running count, a sum, the last N events for a join — which introduces real complexity: where does that state live, how is it recovered after a crash, and how large can it grow.

```java
// Stateful: counting orders per customer requires remembering a running total across events
KTable<String, Long> orderCountsPerCustomer = orders
    .groupBy((key, order) -> order.getCustomerId())
    .count(); // backed by a local state store, checkpointed to Kafka for fault tolerance
```

Frameworks like Kafka Streams back this local state with an embedded store (RocksDB) that's continuously backed up to a Kafka topic (a "changelog") — if the processing instance crashes, its state is rebuilt from the changelog on a replacement instance rather than being lost.

## 3. Event Time vs. Processing Time

A critical distinction once you're aggregating over time windows: **event time** is when something actually happened in the real world (embedded in the event itself); **processing time** is when your system happens to observe and process that event. These are not the same, and the gap between them (due to network delay, retries, or an upstream batch delay) is where a lot of streaming complexity comes from.

```
Event time:      10:00:00  ← when the order was actually placed
Processing time: 10:00:07  ← when this service's stream processor actually saw the event (7s of lag)
```

Aggregating "orders per minute" by processing time gives you a number that reflects your pipeline's own lag characteristics, not reality — a downstream outage that delays delivery by 10 minutes would make it look like there were no orders during that window, when in fact there were plenty, they just arrived late. Aggregating by event time gives the correct, real-world answer, but requires explicitly handling out-of-order and late-arriving events.

## 4. Windowing: Aggregating Over Time

Since a stream never "ends," aggregations need a window — a bounded slice of time to aggregate over — rather than trying to aggregate over the entire infinite stream.

| Window type | Behavior | Example |
| --- | --- | --- |
| Tumbling | Fixed-size, non-overlapping, back-to-back | "Orders per 1-minute window," each event belongs to exactly one window |
| Hopping (sliding) | Fixed-size, but windows overlap by a configured step | "Orders in the last 5 minutes, recalculated every 1 minute" |
| Session | Dynamically sized, closed after a gap of inactivity | "Group a user's clicks into one session, closed after 30 minutes idle" |

```java
KTable<Windowed<String>, Long> ordersPerMinute = orders
    .groupBy((key, order) -> order.getCustomerId())
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count();
```

## 5. Watermarks: Deciding When a Window Is "Done"

Because events can arrive late (event time lags behind processing time), a stream processor needs a rule for when to stop waiting for more events in a window and emit a result — a watermark is that rule: an estimate of "we don't expect to see event times earlier than this anymore."

```java
.windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(1), Duration.ofSeconds(30)))
// accept events up to 30 seconds "late" (by event time) before finalizing a 1-minute window
```

Setting the grace period too short means legitimately late events get dropped or handled as a separate "late data" case; setting it too long means results are held back longer before being emitted, and more in-flight state has to be kept around. This is a genuine tradeoff between result completeness and result latency, not something with one universally correct value.

## 6. Delivery Semantics: At-Most-Once, At-Least-Once, Exactly-Once

The same delivery semantics that apply to message queues apply to stream processing, but now across a whole pipeline of read-process-write steps rather than a single hop.

| Semantic | Guarantee | Risk |
| --- | --- | --- |
| At-most-once | Each event processed zero or one times | Data loss on failure — an event mid-processing when a crash happens is simply gone |
| At-least-once | Each event processed one or more times | Duplicate processing on failure/retry — downstream aggregates or side effects need to tolerate this |
| Exactly-once | Each event's effect is applied exactly once, even across failures | Requires coordinated, transactional reads/processing/writes — real overhead, but eliminates the duplicate-handling burden |

Kafka Streams' exactly-once semantics (`processing.guarantee=exactly_once_v2`) achieve this by making the read-process-write cycle transactional — the consumer offset commit and the output writes succeed or fail together, so a crash mid-processing can't leave the input "consumed" while the output was never actually produced (or the reverse). This removes the need for the idempotent-consumer pattern that at-least-once systems otherwise require, at the cost of some throughput and coordination overhead.

## 7. Joining Multiple Streams

A common real requirement is combining two live streams — matching an order event with a payment event by a shared key, within some time proximity of each other.

```java
KStream<String, Order> orders = builder.stream("orders");
KStream<String, Payment> payments = builder.stream("payments");

KStream<String, OrderWithPayment> joined = orders.join(
    payments,
    (order, payment) -> new OrderWithPayment(order, payment),
    JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10)) // only join events within 10 minutes of each other
);
```

The join window matters for the same reason a time window matters for aggregation: two events with the same key but arriving hours apart probably shouldn't be joined together as if they were related in real time — the window bounds how far apart two events can be and still be considered a match.

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Aggregate by event time, not processing time, for anything correctness-sensitive | Processing-time aggregation reflects your pipeline's lag, not what actually happened in the real world. |
| Choose a window type deliberately | Tumbling for fixed, non-overlapping periods; hopping for overlapping rolling views; session for activity-based grouping. |
| Set the watermark/grace period based on real observed lateness, not a guess | Too short drops legitimate late data; too long delays every result and holds more state in memory. |
| Use exactly-once processing guarantees for financially or otherwise correctness-sensitive pipelines | Removes the burden of building idempotent consumers everywhere downstream, at some throughput cost. |
| Externalize stream processing state with fault-tolerant backing (a changelog topic) | Local-only state is lost on a crash — a changelog-backed store can be rebuilt on a replacement instance. |
| Bound join windows deliberately | An unbounded or too-wide join window can match events that were never actually related in real time. |
| Monitor consumer/processing lag as a leading indicator | Growing lag means the pipeline is falling behind the real-time rate of incoming events, the streaming equivalent of a growing queue backlog. |
