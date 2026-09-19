Database per Service means each microservice owns a private database (or at minimum a private schema), and no other service is ever allowed to read or write those tables directly. All cross-service data access happens through the owning service's API or through published events — never through shared SQL. This is what actually makes services independently deployable: if two services still share direct database access, a schema change in one can silently break the other, and you have a distributed monolith wearing a microservices costume.

## 1. What It Looks Like

```
OrderService     → orders_db      (owns Order, OrderLine)
InventoryService → inventory_db   (owns Stock, Reservation)
BillingService   → billing_db     (owns Invoice, Payment)
```

Each database can even use a different technology suited to that service's access pattern — `OrderService` on Postgres, a `SearchService` on Elasticsearch, a `SessionService` on Redis. The isolation is what matters, not the specific engine.

```java
// WRONG: BillingService reaching directly into orders_db's tables
@Query(value = "SELECT * FROM orders_db.orders WHERE customer_id = :id", nativeQuery = true)
List<Order> findOrdersForBilling(Long id);

// RIGHT: BillingService asks OrderService's API for what it needs
List<OrderSummary> orders = orderServiceClient.getOrdersForCustomer(customerId);
```

## 2. The Consequence: No Cross-Service Joins

Once data lives in separate databases, a SQL join across service boundaries is simply not possible. Getting data that spans two services requires one of:

- **Synchronous API call** — call the owning service's endpoint at request time. Simple, always fresh, but adds a network hop and a runtime dependency on that service being up.
- **Local read-only replica/projection** — subscribe to the owning service's events and keep a denormalized local copy tailored to your own read needs. Fast, no runtime dependency, but eventually consistent.
- **API composition** — a caller (or gateway) fires off several service calls in parallel and stitches the results together in memory.

There is no option that gives you both a live cross-service join and full service independence — that tradeoff is the whole point of the pattern, not an oversight to work around.

## 3. Why This Discipline Exists

A shared database was always the easiest way to get data consistency in a monolith — one transaction, one set of foreign keys, ACID across everything. Splitting into services throws that away on purpose, because the alternative (services sharing tables) means:

- A column rename or index change in one service's table can break another service's queries with no compile-time warning.
- Two services can't be deployed, scaled, or given different uptime/latency guarantees independently, because they're really one database-coupled unit.
- Ownership of data correctness becomes ambiguous — if three services all write to the same table, no single service can enforce its own invariants.

## 4. Pros

| Benefit | Why it matters |
| --- | --- |
| True independent deployability | A schema migration in one service can never break another service that never touches its tables. |
| Technology fit per service | Each service picks the datastore that fits its access pattern (relational, document, key-value, search) instead of one compromise for everyone. |
| Independent scaling | A high-write service's database can be scaled/tuned without affecting a read-heavy neighbor sharing infrastructure. |
| Clear ownership and enforced invariants | Only one service can write to a given table, so only that service needs to guarantee its data stays correct. |
| Failure isolation | A slow query or lock contention in one service's database doesn't degrade an unrelated service's database. |

## 5. Cons

| Cost | Why it hurts |
| --- | --- |
| No cross-service transactions | Updating data in two services atomically requires a Saga or similar pattern instead of a single database transaction — real added complexity. |
| No cross-service joins | Any report or screen needing data from multiple services needs an API call, a local projection, or a dedicated read-side aggregation. |
| Eventual consistency | A local copy of another service's data (kept via events) is, by definition, sometimes momentarily stale. |
| Operational overhead | More databases to provision, back up, monitor, and patch than one shared instance. |
| Data duplication | Denormalized local projections mean the same underlying fact can exist in more than one place, formatted differently for each consumer's needs. |

## 6. When Direct DB Sharing Still Shows Up (And Why It's a Trap)

The most common way this pattern gets violated isn't a deliberate decision — it's a "just this once" shortcut: a new service is added next to an existing one, and it's tempting to give it read access to the existing service's tables directly because building a proper API takes longer. This works right up until the existing service refactors its schema and breaks the new one without ever knowing it did. If two services genuinely need the same data at the same consistency level, that's usually a sign either they're not actually separate services, or the data they need should be exposed through an explicit contract (API or event), not a shared table.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Never grant cross-service direct database access | Not even read-only — a schema change should only ever be able to break the service that owns that schema. |
| Expose data through an API or event, not a table | Treat the owning service's public contract (REST/gRPC endpoint or published event) as the only supported way in. |
| Use local projections for read-heavy cross-service needs | Subscribe to events and keep a denormalized local copy rather than calling another service synchronously on every read. |
| Handle cross-service writes with a Saga, not a distributed transaction | Two-phase commit across independently owned databases doesn't scale operationally — compensating actions do. |
| Pick the datastore per service's actual access pattern | Don't default every service to the same relational database out of habit once ownership is already separated. |
| Version any contract another service depends on | An owning service's API/event schema is now effectively public — changes need the same care as any external API. |
