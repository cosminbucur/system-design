JPA (Jakarta Persistence API) is a specification for mapping Java objects to relational database tables (ORM). Hibernate is the most common implementation, and Spring Data JPA wraps it with repository conveniences. The core idea: you work with entities and let the persistence provider generate the SQL.

1. Entities: Mapping Objects to Tables
   An entity is a plain Java class annotated with `@Entity`, mapped to a table row.

```java
@Entity
@Table(name = "accounts")
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "owner_name", nullable = false, length = 100)
    private String ownerName;

    @Column(precision = 19, scale = 2)
    private BigDecimal balance;

    @Enumerated(EnumType.STRING)
    private AccountStatus status;

    @Version
    private Long version; // optimistic locking, see section 6

    protected Account() {} // JPA requires a no-arg constructor

    public Account(String ownerName, BigDecimal balance) {
        this.ownerName = ownerName;
        this.balance = balance;
        this.status = AccountStatus.ACTIVE;
    }
    // getters/setters omitted
}
```

Key points:

- `@Id` marks the primary key; `GenerationType.IDENTITY` delegates ID generation to the DB (auto-increment). `SEQUENCE` is usually better for batch inserts.
- `@Column` is optional if the field name already matches the column name — use it to override name, nullability, length, precision.
- Entities need a no-arg constructor (JPA/Hibernate instantiates them via reflection).

Never rely on Hibernate's `spring.jpa.hibernate.ddl-auto=update` to manage a real schema — it's convenient for local prototyping but gives you no reviewable history, no safe rollback, and no control over rollout ordering during a deployment. Use db migrations (Liquibase/Flyway) to own the schema explicitly instead, and set `ddl-auto=validate` (or `none`) so Hibernate only checks the entities match the migrated schema rather than silently altering it.

2. The Persistence Context and `EntityManager`
   The `EntityManager` is the core API for CRUD + queries. It tracks a "persistence context" — a first-level cache of managed entities per transaction.

```java
@PersistenceContext
private EntityManager em;

public Account findById(Long id) {
    return em.find(Account.class, id); // managed entity, tracked for changes
}

public void updateBalance(Long id, BigDecimal newBalance) {
    Account account = em.find(Account.class, id);
    account.setBalance(newBalance); // no explicit save() needed
    // change is flushed to DB automatically at transaction commit ("dirty checking")
}
```

Entity states: **transient** (new, not persisted) → **managed** (attached to persistence context, tracked) → **detached** (was managed, context closed) → **removed** (marked for deletion).

3. Spring Data JPA Repositories
   In practice, you rarely touch `EntityManager` directly — Spring Data JPA generates the implementation from an interface.

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    Optional<Account> findByOwnerName(String ownerName);

    List<Account> findByStatusAndBalanceGreaterThan(AccountStatus status, BigDecimal min);

    @Query("SELECT a FROM Account a WHERE a.balance < :threshold")
    List<Account> findLowBalanceAccounts(@Param("threshold") BigDecimal threshold);

    @Modifying
    @Query("UPDATE Account a SET a.status = :status WHERE a.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") AccountStatus status);
}
```

`findBy...` method names are parsed into queries automatically (derived queries). Use `@Query` (JPQL) for anything more complex, and `@Modifying` for UPDATE/DELETE queries.

4. Relationships

   | Relationship | Annotation                                       | Example                                |
   | ------------ | ------------------------------------------------ | -------------------------------------- |
   | One-to-many  | `@OneToMany` (owning side is usually the "many") | One `Customer` has many `Order`s       |
   | Many-to-one  | `@ManyToOne`                                     | Many `Order`s belong to one `Customer` |
   | Many-to-many | `@ManyToMany` with `@JoinTable`                  | `Student` ↔ `Course`                   |
   | One-to-one   | `@OneToOne`                                      | `User` ↔ `UserProfile`                 |

```java
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}

@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

`mappedBy` marks the non-owning side (the owning side has the actual foreign key column via `@JoinColumn`). Always set both sides of a bidirectional relationship in your helper methods to keep them consistent in memory.

5. Fetch Types: Lazy vs Eager — the #1 Source of Bugs
   FetchType | Behavior | Default for
   `LAZY` | Related data loaded only when accessed | `@OneToMany`, `@ManyToMany`
   `EAGER` | Related data loaded immediately with the parent | `@ManyToOne`, `@OneToOne`

The classic trap: `LazyInitializationException` — accessing a lazy association after the persistence context/session is closed (e.g., in a controller after the `@Transactional` service method returned).

```java
// BAD: throws LazyInitializationException if orders wasn't already loaded
Customer customer = customerRepository.findById(id).get();
return customer.getOrders().size(); // session already closed outside @Transactional
```

Fixes: fetch what you need inside the transaction, use a JPQL `JOIN FETCH`, or use a projection/DTO:

```java
@Query("SELECT c FROM Customer c JOIN FETCH c.orders WHERE c.id = :id")
Optional<Customer> findByIdWithOrders(@Param("id") Long id);
```

Rule of thumb: default everything to `LAZY` and fetch eagerly per-query when you actually need the association — never flip `@ManyToOne` to `EAGER` globally "just in case."

6. Transactions and `@Transactional`
   Persistence operations must run inside a transaction. Spring's `@Transactional` demarcates the boundary and handles commit/rollback.

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));
        // both updates commit together, or both roll back on exception
    }
}
```

By default, `@Transactional` rolls back on unchecked exceptions (`RuntimeException`) but NOT on checked exceptions — override with `rollbackFor = Exception.class` if needed.

7. Optimistic vs Pessimistic Locking
   Concurrent updates to the same row need a locking strategy:

```java
// Optimistic: @Version field (see Account entity above) — Hibernate checks the
// version column on UPDATE; if it changed since read, throws OptimisticLockException.
// Good default: no DB locks held, works well for low-contention data.

// Pessimistic: actually locks the row in the DB until the transaction ends.
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdForUpdate(@Param("id") Long id);
```

Prefer optimistic locking by default; reach for pessimistic locking only for high-contention critical sections (e.g., balance transfers under load).

8. The N+1 Query Problem
   Fetching a list of parents, then lazily fetching each parent's children individually, generates 1 query for the parents + N queries for the children.

```java
List<Customer> customers = customerRepository.findAll(); // 1 query
for (Customer c : customers) {
    c.getOrders().size(); // N additional queries — one per customer!
}
```

Fixes: `JOIN FETCH` in JPQL (section 5), `@EntityGraph` on the repository method, or batch fetching (`@BatchSize` / `hibernate.default_batch_fetch_size`). Always check the generated SQL log (`spring.jpa.show-sql=true`) when working with associations.

9. Pagination
   Spring Data's `Pageable`/`Page` is offset-based (`LIMIT`/`OFFSET` under the hood) — fine for small, mostly-static lists, but degrades on large or fast-changing tables..

10. Best Practices

| Practice                                    | Recommendation                                                                                                      |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Default to LAZY fetching                    | Avoid unintended eager loading of entire object graphs.                                                             |
| Use DTOs/projections for reads              | Don't return full entities from REST endpoints — map to DTOs to control payload shape and avoid lazy-loading traps. |
| Keep transactions short                     | Long-running `@Transactional` methods hold DB connections and locks longer than necessary.                          |
| Avoid `CascadeType.ALL` carelessly          | Cascading REMOVE can delete more than intended; scope cascades to what the parent truly owns.                       |
| Watch for N+1 queries                       | Enable SQL logging in dev and check query counts, especially with lists of entities.                                |
| Use `Optional<T>` for single-result lookups | `findById` returns `Optional<Account>` — handle absence explicitly instead of null checks.                          |
