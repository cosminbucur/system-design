# Database Concurrency Control: PostgreSQL & Hibernate

This guide covers the principles, differences, real-life use cases, PostgreSQL implementation details, best practices, and Hibernate/JPA code for **Optimistic Locking** and **Pessimistic Locking**.

---

## Overview & Core Concepts

| Feature | Optimistic Locking | Pessimistic Locking |
| :--- | :--- | :--- |
| **Core Assumption** | Conflicts are rare. | Conflicts are frequent. |
| **Locking Mechanism** | Application-managed (`version` column). | Database-managed (`FOR UPDATE` row locks). |
| **Concurrency / Scaling** | High read/write throughput. | Lower throughput under high lock contention. |
| **User Impact on Conflict** | Application gets 0 rows updated (must retry or notify user). | Request hangs/waits for lock or hits lock timeout. |
| **Deadlock Risk** | None (at DB lock level). | High if transactions lock rows in different orders. |

---

## 1. Pessimistic Locking

Pessimistic locking assumes conflict **will happen**. It explicitly locks the database rows before reading or modifying them, forcing other transactions to wait until the lock is released.

### PostgreSQL Implementation
PostgreSQL achieves pessimistic locking using `SELECT ... FOR UPDATE` (or `FOR NO KEY UPDATE`, `FOR SHARE`). When a transaction acquires a row lock, other transactions attempting to lock or update that same row are blocked.

```sql
-- Transaction 1: Lock the row explicitly
BEGIN;
SELECT * FROM accounts WHERE id = 42 FOR UPDATE;

-- Perform calculation and update
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT; -- Lock released here
```

If **Transaction 2** attempts to run `SELECT ... FOR UPDATE` or `UPDATE` on row 42 while Transaction 1 is open, it halts until Transaction 1 runs `COMMIT` or `ROLLBACK`.

### Real-Life Example: High-Demand Event Ticket Sales
Imagine booking seats for a massively popular concert.
* **Scenario:** Two users try to select the exact same seat (`Seat 12B`) at the same second.
* **Why Pessimistic Locking:** You cannot afford double-booking, and ticket holds are short-lived. Using `SELECT * FROM seats WHERE id = '12B' FOR UPDATE` guarantees that as soon as User A opens the checkout screen, User B is blocked or immediately told the seat is locked.

### Pros & Cons
* **Pros:** Guaranteed data consistency; prevents unnecessary application-level work that would end up failing.
* **Cons:** High chance of blocking/deadlocks; degrades database throughput under heavy concurrency.

---

## 2. Optimistic Locking

Optimistic locking assumes conflict **rarely happens**. It lets any transaction read and prepare changes without locking the row. Before saving, it checks whether another transaction modified the row in the meantime. If the row changed, the update fails or retries.

### PostgreSQL Implementation
PostgreSQL does not have a native "optimistic lock" keyword; instead, you implement it using a `version` integer column or an `updated_at` timestamp.

```sql
-- Step 1: Read the current state and version
SELECT balance, version FROM accounts WHERE id = 42;
-- Result: balance = 500, version = 3

-- Step 2: Attempt the update, asserting version hasn't changed
UPDATE accounts 
SET balance = 400, version = version + 1 
WHERE id = 42 AND version = 3;
```

* **If 1 row is updated:** Success.
* **If 0 rows are updated:** Another transaction updated the row first (version is now 4). The application detects that zero rows were affected, rolls back, and chooses whether to abort or retry.

### Real-Life Example: Collaborative Document / User Profile Editing
Imagine two admins updating a user's profile information or an article in a CMS.
* **Scenario:** Admin A opens the profile form at 10:00 AM. Admin B opens the same profile form at 10:01 AM. Admin A saves changes at 10:05 AM (updating version $1 \rightarrow 2$). Admin B hits save at 10:06 AM.
* **Why Optimistic Locking:** Holding database locks while human users take minutes filling out form fields would lock up the entire system. Instead, Admin B's attempt fails gracefully with a message: *"This profile was updated by someone else while you were editing. Please refresh and try again."*

### Pros & Cons
* **Pros:** No database locks held; maximum throughput and scalability; ideal for web/stateless applications.
* **Cons:** Retries can waste application CPU resources if conflict rates are high.

---

## PostgreSQL Best Practices

1. **Default to Optimistic Locking for Web Apps**
   * HTTP is stateless. Holding database locks across multi-step user workflows or web requests is an anti-pattern. Use version columns (`version integer DEFAULT 1`) for standard CRUD operations.

2. **Use Pessimistic Locking for Short, High-Contention Transactions**
   * Use `SELECT ... FOR UPDATE` when conflict is almost guaranteed and immediate consistency is mandatory (e.g., inventory deduction during a flash sale, wallet balance withdrawals). Keep the transaction as short as possible.

3. **Prevent Deadlocks on Pessimistic Locks**
   * Always lock rows in a deterministic order (e.g., sort IDs before locking multiple rows: `SELECT * FROM items WHERE id IN (1, 2, 3) ORDER BY id FOR UPDATE`).
   * Set a lock timeout to prevent threads from hanging indefinitely:
     ```sql
     SET lock_timeout = '2s';
     ```

4. **Use Non-Blocking Lock Options in Postgres**
   * **`NOWAIT`:** Immediately throws an error if the row is already locked instead of waiting.
     ```sql
     SELECT * FROM jobs WHERE status = 'pending' FOR UPDATE NOWAIT;
     ```
   * **`SKIP LOCKED`:** Ignores locked rows entirely—perfect for building high-throughput background job queues.
     ```sql
     SELECT * FROM jobs WHERE status = 'pending' FOR UPDATE SKIP LOCKED LIMIT 1;
     ```

---

## Implementation in Hibernate / JPA

### 1. Optimistic Locking in Hibernate

Hibernate natively supports optimistic locking using the `@Version` annotation. Hibernate automatically handles checking and incrementing the version field on updates.

#### Entity Definition

```java
import jakarta.persistence.*;

@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    
    private String email;

    // Hibernate uses this field automatically for optimistic locking
    @Version
    private Long version;

    // Standard Getters and Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public Long getVersion() { return version; }
}
```

#### Application Code (Service Layer)

When two users fetch the entity and try to modify it, the second save attempt triggers an `OptimisticLockException` (or `ObjectOptimisticLockingFailureException` in Spring Data).

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.OptimisticLockException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserProfileService {

    private final EntityManager entityManager;

    public UserProfileService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional
    public void updateUserEmail(Long profileId, String newEmail) {
        try {
            // 1. Fetch profile (includes current version, e.g., version = 1)
            UserProfile profile = entityManager.find(UserProfile.class, profileId);
            
            profile.setEmail(newEmail);

            // 2. Commit/Flush automatically generates SQL:
            // UPDATE user_profiles SET email = ?, version = 2 WHERE id = ? AND version = 1;
            entityManager.flush();
            
        } catch (OptimisticLockException e) {
            // 3. Triggered if another transaction updated the record first
            throw new RuntimeException("Profile was modified by another user. Please refresh and try again.", e);
        }
    }
}
```

---

### 2. Pessimistic Locking in Hibernate

Pessimistic locking explicitly instructs Hibernate to issue a `SELECT ... FOR UPDATE` query at the database level when fetching the record.

#### Entity Definition

```java
import jakarta.persistence.*;

@Entity
@Table(name = "seats")
public class Seat {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String seatNumber;

    private boolean reserved;

    // Getters and Setters
    public Long getId() { return id; }
    public String getSeatNumber() { return seatNumber; }
    public boolean isReserved() { return reserved; }
    public void setReserved(boolean reserved) { this.reserved = reserved; }
}
```

#### Application Code using `EntityManager`

Pass `LockModeType.PESSIMISTIC_WRITE` (generates `FOR UPDATE`) when performing a `find` or query execution.

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.LockModeType;
import jakarta.persistence.PessimisticLockException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;

@Service
public class TicketBookingService {

    private final EntityManager entityManager;

    public TicketBookingService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    @Transactional
    public void reserveSeat(Long seatId) {
        try {
            // Generates: SELECT ... FROM seats WHERE id = ? FOR UPDATE
            // Optional property sets a lock timeout (e.g., 2000 ms = 2 seconds)
            Seat seat = entityManager.find(
                Seat.class, 
                seatId, 
                LockModeType.PESSIMISTIC_WRITE,
                Map.of("jakarta.persistence.lock.timeout", 2000)
            );

            if (seat.isReserved()) {
                throw new IllegalStateException("Seat is already reserved!");
            }

            seat.setReserved(true);
            // Lock released upon transaction completion (@Transactional end)

        } catch (PessimisticLockException e) {
            throw new RuntimeException("Could not acquire lock on seat within timeout period.", e);
        }
    }
}
```

---

### 3. Spring Data JPA Repository Example

With Spring Data JPA, declaratively apply locking modes using the `@Lock` annotation on repository interface methods.

```java
import jakarta.persistence.LockModeType;
import jakarta.persistence.QueryHint;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.QueryHints;
import org.springframework.data.repository.query.Param;

import java.util.Optional;

public interface SeatRepository extends JpaRepository<Seat, Long> {

    // Issues: SELECT ... FROM seats WHERE id = ? FOR UPDATE
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        // Sets PostgreSQL lock timeout to 2 seconds
        @QueryHint(name = "jakarta.persistence.lock.timeout", value = "2000")
    })
    @Query("SELECT s FROM Seat s WHERE s.id = :id")
    Optional<Seat> findByIdForUpdate(@Param("id") Long id);
}
```

---

### Mapping Lock Modes to PostgreSQL SQL

| Hibernate `LockModeType` | Generated PostgreSQL SQL |
| :--- | :--- |
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT ... FOR UPDATE` (and increments `@Version` field) |