# Comprehensive Guide to Spring Transaction Propagation

Spring transaction propagation defines how transactions behave when one transactional method calls another. It determines whether a method runs in an existing transaction, starts a new one, or runs without a transaction entirely.

---

## Overview Matrix

| Propagation Type | Existing Transaction Present | No Transaction Present | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`REQUIRED`** *(Default)* | Joins existing transaction | Creates new transaction | Standard business workflows |
| **`REQUIRES_NEW`** | Pauses existing, starts new | Creates new transaction | Independent audit logging / notifications |
| **`NESTED`** | Creates savepoint within current | Creates new transaction | Optional/resilient sub-operations |
| **`SUPPORTS`** | Joins existing transaction | Executes non-transactionally | Read-only lookups |
| **`NOT_SUPPORTED`**| Pauses existing, runs non-transactionally | Executes non-transactionally | Slow external I/O / API calls |
| **`MANDATORY`** | Joins existing transaction | Throws `IllegalTransactionStateException` | Low-level database helper operations |
| **`NEVER`** | Throws `IllegalTransactionStateException` | Executes non-transactionally | Bulk batch processing / migrations |

---

## Practical Examples & Detailed Use Cases

### 1. `REQUIRED` (Default)
> **Rule:** *Join if present, create if missing.*

* **Scenario:** Creating an Online Order
* **Use Case:** You have an `OrderService.createOrder()` method that saves the order, updates stock levels, and generates an invoice. All three steps must succeed or fail as a single unit.
* **Why:** If stock updating fails, you want the entire transaction (including order creation) rolled back.

```java
@Transactional(propagation = Propagation.REQUIRED)
public void createOrder(Order order) {
    saveOrder(order);
    inventoryService.updateStock(order); // Joins outer transaction
    invoiceService.generateInvoice(order); // Joins outer transaction
}
```

---

### 2. `REQUIRES_NEW`
> **Rule:** *Always pause the current transaction and start a brand-new, independent one.*

* **Scenario:** Audit Logging or Notification History
* **Use Case:** A user tries to transfer money, but the transfer fails due to insufficient funds. You still want to record the failed attempt in an `AuditLog` table, regardless of the transfer rolling back.
* **Why:** `REQUIRED` would roll back the audit log entry along with the payment failure. `REQUIRES_NEW` ensures the log commits independently.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logTransactionAttempt(String userId, String status) {
    auditRepository.save(new AuditLog(userId, status)); // Commits even if main flow rolls back
}
```

---

### 3. `NESTED`
> **Rule:** *Execute within a nested transaction using JDBC Savepoints.*

* **Scenario:** Optional Sub-Tasks (e.g., Applying Loyalty Points)
* **Use Case:** A checkout process completes, and as a bonus, the system attempts to process loyalty rewards points. If the loyalty service fails, you roll back the loyalty points but still let the main order go through.
* **Why:** Unlike `REQUIRES_NEW`, if the outer transaction fails later, the nested transaction **will** also roll back. But if *only* the nested part fails, it rolls back to its savepoint without ruining the outer transaction.

```java
@Transactional(propagation = Propagation.REQUIRED)
public void checkout(Order order) {
    saveOrder(order);
    try {
        rewardService.applyLoyaltyPoints(order); // NESTED: savepoint created
    } catch (RewardException e) {
        // Loyalty points failed and rolled back to savepoint, but order still completes!
    }
}
```

---

### 4. `SUPPORTS`
> **Rule:** *Use a transaction if available, but don't bother creating one if not.*

* **Scenario:** Read-Only Lookup or Search Queries
* **Use Case:** Fetching user details or product descriptions.
* **Why:** If called during a larger checkout flow (transactional), it reads within that same transaction context for dirty-read prevention. If called directly from an open API endpoint, it executes standard non-transactional SQL, saving overhead.

```java
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public Product getProductDetails(Long productId) {
    return productRepository.findById(productId);
}
```

---

### 5. `NOT_SUPPORTED`
> **Rule:** *Pause any active transaction and execute non-transactionally.*

* **Scenario:** Slow External API Calls or Large File Downloads
* **Use Case:** In the middle of an order processing flow, you need to call a third-party shipping API or render a complex PDF receipt.
* **Why:** Holding an open database transaction while waiting on external network I/O locks up connection pools and DB rows. Pausing the transaction frees up DB resources during the long wait.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public SyncResponse callSlowExternalVendorApi(VendorRequest request) {
    // DB transaction is suspended here so connection isn't held hostage by network latency
    return vendorClient.sendRequest(request); 
}
```

---

### 6. `MANDATORY`
> **Rule:** *Must be called inside an active transaction; throw an exception otherwise.*

* **Scenario:** Low-Level Data Processing Helpers
* **Use Case:** Updating a ledger balance table directly without performing business validation.
* **Why:** The method itself doesn't manage business boundaries and is dangerous if called stand-alone. Forcing an active transaction guarantees caller code has properly initiated safety checks and boundaries.

```java
@Transactional(propagation = Propagation.MANDATORY)
public void updateRawLedgerBalance(AccountId accountId, BigDecimal amount) {
    // Throws IllegalTransactionStateException if invoked without a parent transaction
    ledgerRepository.updateBalance(accountId, amount);
}
```

---

### 7. `NEVER`
> **Rule:** *Must run without a transaction; throw an exception if a transaction exists.*

* **Scenario:** Heavy Batch Operations / Administrative Tools
* **Use Case:** Running a bulk CSV data import or a database migration script that manages its own internal chunking/flushing.
* **Why:** Running a massive bulk insert inside an existing Spring-managed transaction could exhaust memory or hit lock timeouts. `NEVER` acts as a guardrail to prevent developers from mistakenly wrapping it in an outer transaction.

```java
@Transactional(propagation = Propagation.NEVER)
public void importBulkCsvFile(File file) {
    // Throws exception if invoked within a transaction to prevent high memory/lock retention
    csvImporter.processInBatches(file);
}
```

---

## Summary Key Takeaways

1. **Default Safety:** Use `REQUIRED` for standard transactional operations that should atomically succeed or fail together.
2. **Isolation:** Use `REQUIRES_NEW` when you need an action (like audit logging) to commit regardless of whether the main process succeeds or fails.
3. **Performance:** Use `NOT_SUPPORTED` to suspend transactions during slow external I/O or HTTP calls to prevent locking database connections.