CQRS (Command Query Responsibility Segregation) separates the model used to change data (commands) from the model used to read it (queries), instead of using one unified model for both. A traditional CRUD service uses the same entity/table for writes and reads; CQRS deliberately splits that into two models that can be shaped, stored, and even scaled completely independently — because "the shape that makes writes correct" and "the shape that makes reads fast" are frequently not the same shape at all.

## 1. The Problem: One Model Serving Two Very Different Needs

A single, normalized domain model is usually right for writes — it enforces invariants, prevents duplication, and keeps business rules in one place. But that same normalized shape is often wrong for reads: a UI screen wants pre-joined, denormalized data shaped exactly like what it displays, and a reporting query wants an aggregate across many records — both of which force expensive joins and transformations against a write-optimized schema.

```java
// One model trying to serve both purposes — normalized for correctness, awkward for reads
@Entity
public class Order {
    private String id;
    private String customerId; // reads need the customer's NAME, not just the id — requires a join every time
    private OrderStatus status;
    @OneToMany private List<OrderLine> lines; // reads want a total, not a raw line collection
}
```

## 2. The Split: Separate Write and Read Models

```java
// COMMAND side — enforces business rules, normalized, this is the source of truth
@PostMapping("/orders")
public void createOrder(@RequestBody CreateOrderCommand cmd) {
    orderService.create(cmd); // validates, persists to the write DB, publishes OrderCreatedEvent
}

// QUERY side — denormalized, shaped exactly like what the UI needs, no joins at read time
@GetMapping("/orders/{id}/summary")
public OrderSummaryView getSummary(@PathVariable String id) {
    return orderSummaryReadRepository.findById(id); // pre-joined, kept in sync asynchronously
}
```

```sql
-- Write-side table: normalized, optimized for correctness and constraint enforcement
CREATE TABLE orders (id UUID PRIMARY KEY, customer_id UUID NOT NULL, status VARCHAR(20));
CREATE TABLE order_lines (order_id UUID, sku VARCHAR(50), quantity INT);

-- Read-side table: denormalized, optimized for exactly the query the UI runs
CREATE TABLE order_summary_view (
    order_id UUID PRIMARY KEY,
    customer_name VARCHAR(255),   -- already joined in, no lookup needed at read time
    status VARCHAR(20),
    total_amount DECIMAL,
    item_count INT
);
```

The read model can even live in an entirely different kind of datastore — a search index for full-text queries, a wide denormalized table for dashboards, an in-memory cache for the hottest lookups — chosen for the query pattern, not constrained by whatever the write side needs.

## 3. Keeping the Read Model in Sync

Since the two models are now physically separate, something has to propagate a write into an updated read-side projection. The write side publishes an event when something changes, and a projector consumes that event to update the read model.

```java
@Service
public class OrderService {
    @Transactional
    public void create(CreateOrderCommand cmd) {
        Order order = new Order(cmd);
        orderRepository.save(order); // write model, source of truth
        eventPublisher.publish(new OrderCreatedEvent(order)); // triggers the read-side update, asynchronously
    }
}

@Component
public class OrderSummaryProjector {
    @EventListener
    public void on(OrderCreatedEvent event) {
        OrderSummaryView view = new OrderSummaryView(
            event.orderId(), customerService.getName(event.customerId()),
            event.status(), event.totalAmount(), event.itemCount());
        orderSummaryReadRepository.save(view); // update the denormalized read model
    }
}
```

This means the read model is **eventually consistent** with the write model — there's a small window after a write where a query against the read model can return stale data. This is the central tradeoff of CQRS: you gain read performance and flexibility, but give up the immediate read-your-own-write consistency a single unified model gives you for free.

## 4. CQRS vs. a Simple Read Replica

It's worth distinguishing CQRS from just pointing reads at a read replica of the same database — a much simpler technique that solves a narrower problem.

| | Read replica | CQRS |
| --- | --- | --- |
| Schema | Same schema as the write side, just a copy | A genuinely different, purpose-built schema for reads |
| Sync mechanism | Database-level replication | Application-level events/projections |
| What it solves | Read *scaling* (spreading read load across more instances) | Read *shape* (a fundamentally different, denormalized model for the query pattern) |
| Complexity | Low — mostly infrastructure | Higher — requires building and maintaining projectors and a separate read schema |

A read replica is the right first step if the problem is purely "too much read load on one database instance" and the query shape itself is fine. CQRS is justified when the *shape* of what reads need genuinely diverges from what writes need — replicating the same schema faster doesn't help if every read still has to do the same expensive joins/aggregations against it.

## 5. CQRS and Event Sourcing Often Pair Together (But Aren't the Same Thing)

CQRS is about splitting models; event sourcing is about how the write side stores its state (as a sequence of events rather than current-state rows). They combine naturally — the event stream that event sourcing already produces is exactly what a CQRS projector needs to build read-side views — but CQRS doesn't require event sourcing, and event sourcing doesn't require CQRS. A CQRS write side can just as easily be an ordinary normalized table that happens to also publish change events for the read side to consume.

## 6. When CQRS Is Worth the Complexity

| Situation | CQRS justified? |
| --- | --- |
| Read and write patterns look basically the same (simple CRUD) | No — this is pure overhead: two models, a sync mechanism, and eventual consistency to reason about, for no real benefit |
| Complex reporting/dashboard views that would otherwise require expensive joins on every read | Yes — a purpose-built, pre-aggregated read model turns an expensive query into a simple lookup |
| Read and write load scale very differently (e.g., 1000x more reads than writes) | Yes — the read side can be scaled, cached, and optimized entirely independently of write-side concerns |
| Multiple very different consumers need different shapes of the same underlying data | Yes — each can get its own tailored projection instead of one compromise model serving everyone |

The pattern earns its complexity specifically when the read and write concerns have genuinely diverged — introducing it preemptively on a simple service just adds two models and an eventual-consistency window with nothing to show for it.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Only adopt CQRS when read/write patterns genuinely diverge | For simple CRUD, one unified model is simpler and has no consistency lag to manage. |
| Make projectors idempotent | The same event may be redelivered (at-least-once messaging) — reapplying it must not corrupt or duplicate the read model. |
| Be explicit about the eventual-consistency window with consumers | A UI or API contract relying on the read side needs to know a write may not be immediately visible there. |
| Choose the read-side datastore for the query pattern, not out of convenience | A search index, a wide denormalized table, or a cache each fit different read shapes — pick deliberately. |
| Keep the write side as the single source of truth | The read model is a derived, disposable projection — it should always be rebuildable from the write side's events. |
| Don't conflate CQRS with just using a read replica | A read replica solves read *scaling* with the same schema; CQRS solves read *shape* with a genuinely different one. |
| Version events the read-side projectors consume | Read models evolve independently of the write side — a breaking event schema change can silently stop projections from updating correctly. |
