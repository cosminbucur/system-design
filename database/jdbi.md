JDBI is a lightweight SQL-convenience library for Java — a middle ground between raw JDBC (verbose, manual) and full ORMs like JPA/Hibernate (a lot of magic, harder to see the actual SQL). You write real SQL, JDBI handles the boilerplate: connection/statement lifecycle, parameter binding, and mapping result sets to objects.

## 1. Why JDBI Instead of JPA/Hibernate

| Aspect          | JPA/Hibernate                                  | JDBI                                                                           |
| --------------- | ---------------------------------------------- | ------------------------------------------------------------------------------ |
| SQL control     | Generated for you (JPQL/Criteria)              | You write the actual SQL                                                       |
| Learning curve  | Steep (entity lifecycle, fetch types, caching) | Shallow — thin wrapper over JDBC                                               |
| Best fit        | Complex object graphs, CRUD-heavy apps         | Reporting, complex joins, performance-sensitive queries, teams that prefer SQL |
| Hidden behavior | Lazy loading, dirty checking, N+1 risk         | None — what you write is what runs                                             |

Reach for JDBI when you want SQL to stay SQL, or when an ORM's abstraction is fighting you more than helping.

## 2. Core Setup: `Jdbi` and Handles

```java
Jdbi jdbi = Jdbi.create("jdbc:postgresql://localhost/mydb", "user", "pass");
jdbi.installPlugin(new SqlObjectPlugin());   // enables the declarative API (section 4)
jdbi.installPlugin(new PostgresPlugin());    // vendor-specific type support
```

A `Handle` represents one JDBC connection/session. Two main usage styles:

```java
// Manual handle — you control open/close
try (Handle handle = jdbi.open()) {
    List<Account> accounts = handle.createQuery("SELECT * FROM accounts")
        .mapToBean(Account.class)
        .list();
}

// withHandle / inTransaction — JDBI manages the handle lifecycle for you (preferred)
List<Account> accounts = jdbi.withHandle(handle ->
    handle.createQuery("SELECT * FROM accounts")
        .mapToBean(Account.class)
        .list()
);
```

Prefer `withHandle`/`withExtension`/`inTransaction` over manually opening handles — they guarantee cleanup even on exceptions.

## 3. Fluent API: Queries and Updates

```java
// SELECT with parameter binding
Optional<Account> account = jdbi.withHandle(handle ->
    handle.createQuery("SELECT * FROM accounts WHERE id = :id")
        .bind("id", accountId)
        .mapToBean(Account.class)
        .findOne()
);

// INSERT/UPDATE/DELETE
jdbi.useHandle(handle ->
    handle.createUpdate("UPDATE accounts SET balance = :balance WHERE id = :id")
        .bind("balance", newBalance)
        .bind("id", accountId)
        .execute()
);

// Getting a generated key back
long id = jdbi.withHandle(handle ->
    handle.createUpdate("INSERT INTO accounts (owner_name, balance) VALUES (:owner, :balance)")
        .bind("owner", ownerName)
        .bind("balance", initialBalance)
        .executeAndReturnGeneratedKeys("id")
        .mapTo(Long.class)
        .one()
);
```

Named parameters (`:id`) are always preferred over positional (`?`) — safer to maintain as queries grow, and self-documenting.

## 4. Declarative API: `SqlObject`

Instead of writing fluent code, declare an interface and let JDBI generate the implementation — similar in spirit to Spring Data JPA repositories.

```java
public interface AccountDao {

    @SqlQuery("SELECT * FROM accounts WHERE id = :id")
    Optional<Account> findById(@Bind("id") long id);

    @SqlQuery("SELECT * FROM accounts WHERE balance < :threshold")
    List<Account> findLowBalance(@Bind("threshold") BigDecimal threshold);

    @SqlUpdate("INSERT INTO accounts (owner_name, balance) VALUES (:ownerName, :balance)")
    @GetGeneratedKeys
    long insert(@BindBean Account account);

    @SqlUpdate("UPDATE accounts SET balance = :balance WHERE id = :id")
    int updateBalance(@Bind("id") long id, @Bind("balance") BigDecimal balance);
}
```

```java
AccountDao dao = jdbi.onDemand(AccountDao.class); // one instance, opens a handle per call
Optional<Account> account = dao.findById(42L);
```

`@BindBean` binds an object's getters to named parameters matching field names — handy for inserts/updates that map cleanly to a single object.

## 5. Mapping Results to Objects

Three main strategies:

```java
// 1. mapToBean — matches column names to setters (needs a no-arg constructor + setters)
handle.createQuery("SELECT * FROM accounts").mapToBean(Account.class).list();

// 2. Constructor mapping — immutable objects, no setters needed
@JdbiConstructor
public Account(@ColumnName("id") long id,
                @ColumnName("owner_name") String ownerName,
                @ColumnName("balance") BigDecimal balance) { ... }

// 3. Custom RowMapper — full control
RowMapper<Account> mapper = (rs, ctx) -> new Account(
    rs.getLong("id"),
    rs.getString("owner_name"),
    rs.getBigDecimal("balance")
);
handle.createQuery("SELECT * FROM accounts").map(mapper).list();
```

Register mappers once and reuse them across queries:

```java
jdbi.registerRowMapper(Account.class, mapper);
// now .mapTo(Account.class) works anywhere without re-specifying the mapper
```

Constructor mapping is generally the best default — it forces immutability and fails fast (at construction) if a column is missing, rather than silently leaving a field null.

## 6. Transactions

```java
jdbi.useTransaction(handle -> {
    handle.createUpdate("UPDATE accounts SET balance = balance - :amount WHERE id = :from")
        .bind("amount", amount).bind("from", fromId).execute();

    handle.createUpdate("UPDATE accounts SET balance = balance + :amount WHERE id = :to")
        .bind("amount", amount).bind("to", toId).execute();
    // both statements commit together; any exception triggers rollback
});
```

With `SqlObject`, annotate a DAO method directly:

```java
@SqlUpdate("...")
@Transaction
void transferFunds(...);
```

## 7. Batch Operations

For bulk inserts/updates, use `PreparedBatch` instead of looping individual statements — drastically fewer round-trips:

```java
jdbi.useHandle(handle -> {
    PreparedBatch batch = handle.prepareBatch(
        "INSERT INTO accounts (owner_name, balance) VALUES (:owner, :balance)");

    for (Account a : newAccounts) {
        batch.bind("owner", a.getOwnerName())
             .bind("balance", a.getBalance())
             .add();
    }
    batch.execute();
});
```

## 8. Pagination

For paging through query results at scale, prefer cursor/keyset queries (`WHERE (created_at, id) < (:createdAt, :id)`) over `OFFSET` — JDBI maps directly onto the raw seek SQL.

## 9. Best Practices

| Practice                                                                  | Recommendation                                                                                                                |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Use named parameters                                                      | `:name` over `?` — safer against argument-order mistakes as queries evolve.                                                   |
| Prefer constructor mapping                                                | Immutable result objects, fail fast on missing columns, no reflection-based setter calls.                                     |
| Always parameterize, never concatenate                                    | String-concatenated SQL reopens SQL injection — bind values, never interpolate user input into the query text.                |
| Use `onDemand` for simple DAOs, `inTransaction` for multi-statement units | `onDemand` opens/closes a handle per call — fine for single statements, wrong for multi-step transactions.                    |
| Batch bulk writes                                                         | Use `PreparedBatch` for loops of inserts/updates instead of one round-trip per row.                                           |
| Keep SQL in one place                                                     | Co-locate SQL with its DAO interface/method (via `@SqlQuery`/`@SqlUpdate`) so it's easy to find and reason about performance. |
