Cursor pagination (a.k.a. keyset pagination) returns a page of results plus an opaque "cursor" pointing to where the next page starts, instead of an offset/page number. It fixes correctness and performance problems that offset-based pagination has at scale, at the cost of losing "jump to page 7" random access.

## 1. Offset Pagination — The Problem

The naive approach: `LIMIT n OFFSET m`. It has two real problems once data is large or changing.

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```

- **Performance**: the database still has to scan and discard the first 100,000 rows before returning the next 20 — offset cost grows linearly with page depth, even though you only want a handful of rows.
- **Correctness under concurrent writes**: if a row is inserted or deleted between page 1 and page 2, every subsequent offset shifts — a client can see a duplicate row (shown on two different pages) or silently skip one entirely. This isn't a corner case; it happens routinely on any actively-written table.

Offset pagination is still fine for small, mostly-static datasets where you genuinely need "jump to page N" (e.g., an admin table with a few hundred rows) — the tradeoff below only matters once data is large or changes frequently.

## 2. Cursor (Keyset) Pagination — The Fix

Instead of "skip N rows," the query says "give me the next rows after this specific point," using an indexed column (or combination) as the seek key.

```sql
-- First page
SELECT * FROM orders ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page: seek from the last row of the previous page
SELECT * FROM orders
WHERE (created_at, id) < (:lastSeenCreatedAt, :lastSeenId)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Why the tuple `(created_at, id)` instead of just `created_at`: `created_at` alone isn't guaranteed unique — two orders can share a timestamp. Tie-breaking on `id` (or any unique column) as a secondary key guarantees a stable, total order, so no row is ever skipped or duplicated across pages. This combination is the **cursor** — it's derived from the last row returned, not from a row count.

Because the query is a `WHERE` seek on an indexed column rather than a full offset scan, performance stays roughly constant regardless of how deep into the dataset you are — page 5,000 costs about the same as page 1.

## 3. Encoding the Cursor for API Responses

The cursor should be treated as an opaque token by the client — don't expose raw column values directly if you want the freedom to change the underlying implementation later.

```java
public record PageCursor(Instant createdAt, Long id) {

    public String encode() {
        String raw = createdAt.toString() + "|" + id;
        return Base64.getUrlEncoder().withoutPadding().encodeToString(raw.getBytes(StandardCharsets.UTF_8));
    }

    public static PageCursor decode(String token) {
        String raw = new String(Base64.getUrlDecoder().decode(token), StandardCharsets.UTF_8);
        String[] parts = raw.split("\\|", 2);
        return new PageCursor(Instant.parse(parts[0]), Long.parseLong(parts[1]));
    }
}
```

```json
{
  "items": [ { "id": 501, "createdAt": "2026-09-14T10:00:00Z" }, ... ],
  "nextCursor": "MjAyNi0wOS0xNFQxMDowMDowMFp8NTAx"
}
```

The client never parses or constructs the cursor — it just echoes back whatever `nextCursor` it was given. This also means you can change the internal seek keys later without breaking API clients, as long as you keep decoding old cursor formats or version them.

## 4. Implementing with JPA / Spring Data

Spring Data's built-in `Pageable`/`Page` is offset-based under the hood — for real keyset pagination you write the seek query explicitly. `Slice` (rather than `Page`) is often used here since it avoids a separate `COUNT(*)` query.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        SELECT o FROM Order o
        WHERE (o.createdAt < :createdAt) OR (o.createdAt = :createdAt AND o.id < :id)
        ORDER BY o.createdAt DESC, o.id DESC
        """)
    Slice<Order> findNextPage(@Param("createdAt") Instant createdAt,
                               @Param("id") Long id,
                               Pageable pageable); // pageable only supplies the LIMIT here
}
```

```java
public PagedResult<Order> getOrders(String cursorToken, int size) {
    PageCursor cursor = cursorToken != null ? PageCursor.decode(cursorToken) : PageCursor.first();
    Slice<Order> slice = orderRepository.findNextPage(cursor.createdAt(), cursor.id(), PageRequest.of(0, size));

    String nextCursor = slice.hasNext()
        ? new PageCursor(slice.getContent().getLast().getCreatedAt(), slice.getContent().getLast().getId()).encode()
        : null;

    return new PagedResult<>(slice.getContent(), nextCursor);
}
```

Note: `Slice.hasNext()` typically works by fetching `size + 1` rows and checking if the extra row exists, then discarding it — cheaper than a `COUNT(*)` over the whole table.

## 5. Implementing with JDBI

Maps directly onto the raw SQL from section 2.

```java
public interface OrderDao {

    @SqlQuery("""
        SELECT * FROM orders
        WHERE (created_at, id) < (:createdAt, :id)
        ORDER BY created_at DESC, id DESC
        LIMIT :limit
        """)
    List<Order> findNextPage(@Bind("createdAt") Instant createdAt,
                              @Bind("id") long id,
                              @Bind("limit") int limit);
}
```

```java
List<Order> page = orderDao.findNextPage(cursor.createdAt(), cursor.id(), pageSize + 1);
boolean hasNext = page.size() > pageSize;
List<Order> results = hasNext ? page.subList(0, pageSize) : page;
```

Requesting `pageSize + 1` rows and trimming the extra one is the standard trick for cheaply knowing whether a next page exists, without a separate count query.

## 6. Required Index

Cursor pagination's performance guarantee depends entirely on the seek columns being indexed — without an index matching the `ORDER BY`/`WHERE` tuple, the database falls back to a full sort/scan and you lose the whole benefit.

```sql
CREATE INDEX idx_orders_created_at_id ON orders (created_at DESC, id DESC);
```

The index column order must match the query's `ORDER BY`/seek tuple order — an index on `(id, created_at)` won't help a query seeking on `(created_at, id)`.

## 7. Supporting Sorting

Everything above assumed a fixed sort order (`created_at DESC, id`). A real list endpoint often needs to let the client choose the sort field (`sort=balance`, `sort=-ownerName`) — this is where cursor pagination gets genuinely harder than offset pagination, because the seek predicate, the cursor's contents, and the required index all depend on _which_ sort is active.

### The Seek Predicate Generalizes to N Columns

The tuple comparison isn't specific to `(created_at, id)` — it generalizes to whatever sort columns are active, always ending in a unique tie-breaker:

```sql
-- Sorting by balance descending, tie-broken by id
SELECT * FROM accounts
WHERE (balance, id) < (:lastSeenBalance, :lastSeenId)
ORDER BY balance DESC, id DESC
LIMIT 20;

-- Sorting by owner_name ascending, tie-broken by id
SELECT * FROM accounts
WHERE (owner_name, id) > (:lastSeenOwnerName, :lastSeenId)
ORDER BY owner_name ASC, id ASC
LIMIT 20;
```

Postgres and MySQL 8+ support row-value comparison (`(a, b) < (x, y)`) directly, evaluating lexicographically exactly like comparing tuples in code. For databases without that support, the same logic expands into an explicit OR-chain:

```sql
-- Equivalent to (balance, id) < (:lastSeenBalance, :lastSeenId), for engines without row-value comparison
WHERE balance < :lastSeenBalance
   OR (balance = :lastSeenBalance AND id < :lastSeenId)
```

This OR-chain form is also just what section 4/5's JPQL query already does for the fixed `(createdAt, id)` case — generalizing to a dynamic sort field just means substituting whichever column is currently active into the same shape.

### The Cursor Must Encode Both the Values AND Which Sort Was Active

A cursor produced under one sort order is meaningless under a different one — the seek values it carries only make sense relative to the sort that generated them. Encode the active sort spec into the cursor itself, so a cursor can never be silently misapplied to a different `sort=` parameter on the next request.

```java
public record PageCursor(String sortField, String sortDirection, Object sortValue, Long id) {

    public String encode() {
        String raw = sortField + "|" + sortDirection + "|" + sortValue + "|" + id;
        return Base64.getUrlEncoder().withoutPadding().encodeToString(raw.getBytes(StandardCharsets.UTF_8));
    }

    public static PageCursor decode(String token) {
        String raw = new String(Base64.getUrlDecoder().decode(token), StandardCharsets.UTF_8);
        String[] parts = raw.split("\\|", 4);
        return new PageCursor(parts[0], parts[1], parts[2], Long.parseLong(parts[3]));
    }
}
```

```java
public PagedResult<Account> getAccounts(String sortParam, String cursorToken, int size) {
    SortSpec sort = SortSpec.parse(sortParam); // e.g. "-balance" -> field=balance, direction=DESC

    PageCursor cursor = cursorToken != null ? PageCursor.decode(cursorToken) : PageCursor.first(sort);
    if (!cursor.sortField().equals(sort.field()) || !cursor.sortDirection().equals(sort.direction())) {
        throw new InvalidCursorException("Cursor does not match the requested sort order");
        // reject rather than silently seek from the wrong column — a mismatched
        // cursor+sort combination produces confusing, wrong results, not just an error
    }
    // ... build and run the dynamic seek query for sort.field()/sort.direction()
}
```

Rejecting a mismatched cursor explicitly (rather than trying to interpret it under the new sort) matters: silently reusing stale seek values against a different column produces plausible-looking but wrong results — a subtle bug, not an obvious failure.

### Indexing: Restrict Sortable Fields to an Allow-List

Cursor pagination's whole performance case depends on an index matching the exact seek tuple. Allowing the client to sort by _any_ field means you'd need a composite index — column plus the `id` tie-breaker — for every single one of those fields, which usually isn't practical or worth the write-side index maintenance cost. The standard fix: expose a small, explicit allow-list of sortable fields, each backed by its own composite index, and reject (or ignore, falling back to a default) any `sort=` value outside that list.

```sql
CREATE INDEX idx_accounts_balance_id ON accounts (balance DESC, id DESC);
CREATE INDEX idx_accounts_owner_name_id ON accounts (owner_name ASC, id ASC);
-- only sortable fields that actually have a supporting index should be accepted by the API
```

```java
private static final Set<String> ALLOWED_SORT_FIELDS = Set.of("balance", "ownerName", "createdAt");

private SortSpec validateSort(String sortParam) {
    SortSpec sort = SortSpec.parse(sortParam);
    if (!ALLOWED_SORT_FIELDS.contains(sort.field())) {
        throw new InvalidSortFieldException(sort.field());
    }
    return sort;
}
```

One index per sortable direction is usually unnecessary: most databases (Postgres included) can scan a B-tree index backwards efficiently, so an index defined `ASC` typically also serves a `DESC` query on the same column without a second index — confirm this with `EXPLAIN ANALYZE` rather than assuming it, since it depends on the specific query planner and version.

## 8. REST API Conventions

```
GET /api/orders?limit=20                                    → first page, default sort
GET /api/orders?limit=20&sort=-balance                      → first page, sorted by balance descending
GET /api/orders?limit=20&sort=-balance&cursor=MjAyNi0wOS0x... → next page of that same sort
```

```json
{
  "items": [...],
  "nextCursor": "MjAyNi0wOS0xNFQxMDowMDowMFp8NTAx",
  "hasNext": true
}
```

Common conventions: `nextCursor: null` (or omitted) signals the last page; a leading `-` on the `sort` value means descending, no prefix means ascending (`sort=-balance,ownerName` for multiple fields, applied in order); a `sort=` value outside the allow-list should return `400 Bad Request`, not silently fall back or attempt an unindexed query. Some APIs also return a `previousCursor` for bidirectional paging (requires storing/deriving the seek tuple for the first row too). Avoid also exposing a `totalCount` unless you're willing to pay for a `COUNT(*)` — that defeats part of the reason to use cursor pagination in the first place.

## 9. Avoiding `COUNT(*)` When a Total Is Requested

Product/UI often asks for a total ("1,204 results") alongside the page — resist reaching for `COUNT(*)` to get it; it re-adds the exact `O(n)` cost cursor pagination was meant to eliminate. A few alternatives, roughly in order of how often they're actually the right call:

**Don't return a total at all — just `hasNext`.** This is section 4/5's `limit + 1` trick, and it's the right default: most UIs only need "is there more" (to show/hide a "load more" control), not an exact count. If the product requirement can be satisfied with `hasNext`, that's the cheapest and simplest answer — push back on "show the total" before implementing it.

**Maintain a running counter instead of counting on read.** If an exact total is genuinely required, increment/decrement it in the same transaction as the write, so reading it is `O(1)` forever instead of `O(n)` per request.

```sql
UPDATE counters SET order_count = order_count + 1 WHERE key = 'orders';
```

Tradeoff: every write path must remember to maintain it (a batch job or direct SQL update that forgets will silently drift the counter) — same discipline problem as the explicit cache-invalidation-on-write approach.

**Use the database's own statistics for an approximate count.** Postgres tracks a planner estimate that's usually close enough for "about how many," without scanning the table:

```sql
SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders';
```

Fine for a dashboard ("~1,200 results"); wrong for anything that needs to be exact (e.g., billing based on a row count).

**Cache the count with a short TTL.** If a slightly-stale total is acceptable, compute the real `COUNT(*)` occasionally and cache it rather than on every request — amortizes the cost across many reads instead of paying it per page load.

**If an exact filtered count is unavoidable, confirm it's index-only.** Check `EXPLAIN ANALYZE` shows an index-only scan on the filtered columns, not a full table scan — still `O(n)`, but a much smaller constant factor since it never touches the table heap, only the index.

## 10. Best Practices

| Practice                                                                    | Recommendation                                                                                                                                 |
| --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Always include a unique tie-breaker column                                  | Never seek on a non-unique column alone (e.g., `created_at`) — pair it with a unique column like `id` to guarantee no skipped/duplicated rows. |
| Index the exact seek tuple                                                  | The composite index must match the `ORDER BY`/`WHERE` column order, or performance degrades to a full scan.                                    |
| Treat the cursor as opaque on the client                                    | Encode it (e.g., Base64) so clients can't parse or construct it — protects your freedom to change the underlying seek keys later.              |
| Avoid `COUNT(*)`/total pages with cursor pagination                         | Defeats the performance benefit; if you truly need a total, compute it asynchronously or approximate it, don't run it per-request.             |
| Fetch `limit + 1` to detect a next page cheaply                             | Avoids a second round-trip or count query just to know whether more data exists.                                                               |
| Reserve offset pagination for small/static datasets or "jump to page N" UX  | Cursor pagination gives up random page access — use offset only when that tradeoff is acceptable.                                              |
| Restrict sortable fields to an explicit allow-list, each with its own index | Supporting arbitrary sort fields either needs an index per field or produces slow, unindexed sorts.                                            |
| Encode the active sort spec inside the cursor, and reject a mismatched one  | A cursor from one sort order silently reused under a different one produces plausible-looking but wrong results, not an obvious error.         |
