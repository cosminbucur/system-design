A Saga coordinates a business transaction that spans multiple services, each owning its own database, where a single ACID transaction across all of them isn't possible. Instead of one atomic commit, a saga is a sequence of local transactions — each service commits its own local change and publishes something that triggers the next step — with explicit compensating transactions defined to undo prior steps if a later one fails. It trades true atomicity for a workflow that's guaranteed to reach a consistent end state, either fully completed or fully compensated.

## 1. The Problem: No Distributed ACID Transaction

Once `OrderService`, `InventoryService`, and `PaymentService` each own their own database, placing an order that touches all three can't be wrapped in one database transaction the way it could in a monolith.

```
OrderService.createOrder()      → its own DB, own transaction
InventoryService.reserveStock() → its own DB, own transaction
PaymentService.chargeCard()     → its own DB, own transaction
```

If the payment step fails after inventory was already reserved, there's no automatic rollback spanning all three — each commit already happened locally, in a different database, with no shared transaction coordinator holding them together. A saga is the explicit, application-level mechanism for handling that: if step 3 fails, deliberately run a compensating action that undoes step 2 (release the reserved stock) and step 1 (cancel the order).

## 2. Local Transactions Plus Compensating Transactions

Each step in a saga is a normal local ACID transaction within one service. What makes it a saga is that every step that changes state also has a corresponding compensating transaction defined for undoing it, since there's no database-level rollback available across service boundaries.

| Step | Forward action | Compensating action |
| --- | --- | --- |
| 1 | `OrderService`: create order (status `PENDING`) | Cancel the order |
| 2 | `InventoryService`: reserve stock | Release the reserved stock |
| 3 | `PaymentService`: charge the card | Refund the charge |

Compensations aren't a true rollback in the ACID sense — they're a new, forward-moving business action that semantically undoes the effect of an earlier step. "Refund the charge" doesn't erase the fact that a charge and a refund both happened; it just leaves the customer in the correct final financial state. This is a real, visible difference from a database transaction rollback, and it needs to make business sense on its own (e.g., a refund might trigger its own notification, unlike a silent rollback).

## 3. Choreography: Services React to Each Other's Events

In choreography, there's no central coordinator — each service listens for events from the previous step and decides what to do next, including whether to trigger a compensation.

```java
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserveStock(event.getOrderId(), event.getItems());
        publisher.publishEvent(new StockReservedEvent(event.getOrderId()));
    } catch (InsufficientStockException ex) {
        publisher.publishEvent(new OrderFailedEvent(event.getOrderId(), "OUT_OF_STOCK"));
        // OrderService listens for OrderFailedEvent and compensates (cancels the order)
    }
}
```

```
OrderCreatedEvent → InventoryService reacts → StockReservedEvent → PaymentService reacts → PaymentCompletedEvent → ...
                                            ↘ (failure) → OrderFailedEvent → OrderService reacts (compensates)
```

Choreography is simple to set up for a short saga (2-3 steps) — each service only needs to know about the events it cares about, with no shared orchestration logic anywhere. The cost shows up as the chain grows: there's no single place to read "what does this whole business transaction actually do end to end," and tracing a failure means following event listeners scattered across several services' codebases.

## 4. Orchestration: A Central Coordinator Drives Each Step

In orchestration, a dedicated saga orchestrator explicitly calls each service in sequence and explicitly handles what to do — including which compensations to run — when a step fails.

```java
@Component
public class OrderSagaOrchestrator {

    public void placeOrder(CreateOrderCommand cmd) {
        Order order = orderService.create(cmd);
        try {
            inventoryService.reserveStock(order.getId(), order.getItems());
            paymentService.chargeCard(order.getId(), order.getTotal());
            orderService.markConfirmed(order.getId());
        } catch (InsufficientStockException ex) {
            orderService.cancel(order.getId()); // compensate step 1, nothing to undo in inventory yet
        } catch (PaymentFailedException ex) {
            inventoryService.releaseStock(order.getId()); // compensate step 2
            orderService.cancel(order.getId());            // compensate step 1
        }
    }
}
```

The entire workflow — happy path and every compensation — is readable in one place, which makes it far easier to reason about, test, and debug as the number of steps grows. The cost is the orchestrator itself: it now knows about every participating service, which is a form of coupling that choreography avoids, and it becomes a piece of infrastructure that needs its own reliability story (what happens if the orchestrator itself crashes mid-saga).

## 5. Choreography vs. Orchestration

| | Choreography | Orchestration |
| --- | --- | --- |
| Coordination | Implicit — each service reacts to events | Explicit — a central component drives every step |
| Coupling | Services only know the events they consume | The orchestrator knows about every participating service |
| Visibility | The overall flow is scattered across services' event listeners | The overall flow is readable in one place |
| Best fit | A short saga (2-3 steps), simple compensation logic | A longer or more complex saga, needing clear failure/compensation logic |
| Failure to add a new step | Ripples through multiple services' event handling | Usually just one more step in the orchestrator |

Neither is universally better — the general guidance is to start with choreography for a genuinely simple, short saga, and move to orchestration once the number of steps or the complexity of failure handling makes the implicit, scattered coordination of choreography harder to reason about than a single orchestrator would be.

## 6. Semantic Lock: Preventing Interference Mid-Saga

Because a saga's intermediate state is visible to the rest of the system (unlike an ACID transaction, which is invisible until commit), other requests can observe and act on data that's only provisionally correct — an order marked "confirmed" before payment has actually gone through, for instance. A common technique is a semantic lock: mark the record with a pending/in-progress status for the duration of the saga, so other parts of the system know not to treat it as final yet.

```java
public enum OrderStatus {
    PENDING,    // saga in progress — not a final state, other services should treat this as "not yet confirmed"
    CONFIRMED,  // saga completed successfully
    CANCELLED   // saga compensated
}
```

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Define a compensating action for every step that changes state | Without one, a failure partway through leaves the system in an inconsistent state with no way to unwind it. |
| Make every step (and every compensation) idempotent | A retried step or a redelivered event must not double-charge, double-reserve, or double-compensate. |
| Start with choreography for short, simple sagas | Avoids building orchestration infrastructure for a workflow that doesn't need it yet. |
| Move to orchestration once the flow gets hard to trace | A central, explicit coordinator is easier to reason about, test, and debug than event chains scattered across many services. |
| Mark in-progress saga state explicitly (semantic lock) | Prevents the rest of the system from treating a not-yet-finalized record as if the saga had already completed. |
| Design compensations as real business actions, not silent rollbacks | A refund, a cancellation notice, or a stock release is a visible action with its own consequences — model it as such. |
| Give the orchestrator (if used) its own durability story | If the orchestrator can crash mid-saga, it needs to persist progress and resume, not lose track of an in-flight transaction. |
