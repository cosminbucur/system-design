Every row needs a primary key, and how you generate it is a decision with real consequences: index fragmentation, write throughput, cross-service uniqueness, URL-safety, and whether an attacker can guess adjacent IDs. There's no single right answer — the right ID strategy depends on whether you need global uniqueness across distributed nodes, sortability by creation time, or just the simplest thing that works for a single database.

## 1. Auto-Increment / Sequence — Simple, but Single-Node

The database itself hands out the next integer, either via a native auto-increment column or an explicit sequence object.

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY, -- Postgres: backed by an implicit sequence
    total NUMERIC(10,2) NOT NULL
);
```

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "orders_seq")
private Long id;
```

| Pros | Cons |
| --- | --- |
| Compact (8 bytes), fast to index, human-readable | Only unique within one database — breaks the moment you shard or run multiple writers |
| Sequential inserts are cache-friendly for B-tree indexes | Predictable/guessable — `id=1043` tells an attacker `id=1044` probably exists too |
| Simplest option, zero application code needed | Requires a round-trip to the DB before the ID is known — can't generate an ID client-side before an insert |

`GenerationType.IDENTITY` delegates directly to the DB's auto-increment and defeats JDBC batch inserts (each insert must complete before the next ID is known); `GenerationType.SEQUENCE` lets Hibernate pre-fetch a batch of IDs, which is why it's usually the better choice for high-throughput inserts.

## 2. UUID — Distributed-Safe, but Index-Unfriendly

A UUID (128 bits, typically v4/random) is generated anywhere — client, server, any node — with a collision probability low enough to treat as zero, with no coordination required between nodes.

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id; // e.g. 3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

The problem is random UUIDs (v4) are, by design, uniformly distributed across the entire ID space — the exact opposite of what a B-tree index wants. Every insert lands at a random point in the index rather than appending to the end, which means:

- **Page splits**: the B-tree constantly has to rebalance to fit a new value in the middle of an already-full page, instead of just appending to the last page.
- **Poor cache locality**: recently-inserted rows (the ones most often queried together) end up scattered across many different disk pages instead of sitting near each other.
- **Larger index size**: 16 bytes per UUID vs. 8 bytes for a `BIGINT`, compounding across every index that includes the key.

This single tradeoff — global uniqueness without coordination, at the cost of index locality — is exactly what every "sortable ID" scheme below exists to fix.

## 3. UUIDv7 — Time-Ordered UUIDs

UUIDv7 (standardized in 2024) keeps the UUID format (128 bits, same column type, same collision guarantees) but replaces the random prefix with a millisecond timestamp, making generated IDs monotonically increasing over time.

```
UUIDv4: f47ac10b-58cc-4372-a567-0e02b2c3d479  (fully random — no ordering)
UUIDv7: 018f4a3e-7c21-7d2e-8f3a-2b1c9e8d7f6a  (first 48 bits = timestamp — sorts by creation time)
```

Because new IDs are numerically greater than previous ones, inserts append to the end of the B-tree instead of scattering — solving the index-fragmentation problem of v4 while keeping the "generate anywhere, no coordination" property that made UUIDs attractive for distributed systems in the first place. This is why UUIDv7 (and the schemes below) are increasingly the default recommendation over plain random UUIDs for new systems.

## 4. Snowflake IDs — Structured, Sortable, Compact

Twitter's Snowflake scheme packs a timestamp, a machine/worker ID, and a per-millisecond sequence number into a single 64-bit integer.

```
| 1 bit unused | 41 bits timestamp (ms since epoch) | 10 bits machine ID | 12 bits sequence |
```

- The timestamp bits make IDs sortable by creation time and roughly increasing — good for index locality, same as UUIDv7.
- The machine ID bits let up to 1024 nodes generate IDs independently with zero coordination and zero collision risk between them.
- The sequence bits allow up to 4096 IDs per millisecond per machine before needing to wait for the next tick.

The tradeoff versus UUIDv7: a Snowflake ID fits in a standard 64-bit `BIGINT` (half the storage of a UUID) and is still globally unique and time-sortable, but requires each generating node to have a distinct, correctly-assigned machine ID — a small piece of coordination UUIDv7 doesn't need at all (it embeds no machine identity, relying purely on the timestamp plus enough random bits to make collisions negligible).

## 5. TSID — A Practical Java Implementation of the Snowflake Idea

TSID (Time-Sorted Identifier) is a widely-used Java library implementation of the Snowflake concept, designed specifically to be a drop-in, sortable alternative to UUID in JVM applications.

```xml
<dependency>
    <groupId>com.github.f4b6a3</groupId>
    <artifactId>tsid-creator</artifactId>
    <version>5.2.6</version>
</dependency>
```

```java
// Generates a 64-bit long, encoded as a 13-character Crockford Base32 string when serialized
long id = TsidCreator.getTsid().toLong();      // e.g. 38352658567418872
String str = TsidCreator.getTsid().toString(); // e.g. 0S4C58QZBBCXF

// Node/machine ID assigned explicitly (analogous to Snowflake's machine ID bits) —
// required when running multiple instances, to guarantee no collisions between them
TsidFactory factory = TsidFactory.builder()
    .withNode(1) // this instance's unique node number, e.g. from pod ordinal or config
    .build();
long id2 = factory.create().toLong();
```

```java
@Id
private Long id; // populate with TsidCreator.getTsid().toLong() in a pre-persist hook, or via a custom Hibernate IdentifierGenerator
```

TSID's practical appeal for a Java shop specifically:

| Property | Why it matters |
| --- | --- |
| Fits in a `long` (64 bits) | Same storage and index cost as a `BIGINT` — none of UUID's 16-byte overhead |
| Time-ordered | New IDs are always numerically greater — appends to the end of a B-tree, no page-split fragmentation |
| No central coordinator needed at runtime | Only a one-time node ID assignment per instance (e.g., from a Kubernetes pod ordinal); after that, generation is fully local |
| String encoding is compact and URL-safe | Crockford Base32, 13 characters, no special characters to escape — nicer in a URL path than a UUID's hyphens |

The one thing to get right operationally is the **node ID**: two instances generating IDs with the same node ID at overlapping timestamps can collide, so the node ID must be reliably unique per running instance — typically derived from a Kubernetes StatefulSet pod ordinal, an assigned config value per deployment, or a coordination service, never hardcoded to the same value across replicas.

## 6. Choosing an ID Strategy

| Situation | Recommended approach |
| --- | --- |
| Single database, no sharding, no client-side ID generation needed | Auto-increment / sequence — simplest option, no downsides that matter at this scale |
| Multiple writers or services need to generate IDs independently, no B-tree index performance concerns | Plain UUID (v4) — simplest globally-unique option when index locality doesn't matter (e.g., a key-value store, not a B-tree-indexed relational table) |
| Distributed writers, but the primary key is a relational B-tree index that needs good insert locality | UUIDv7, Snowflake, or TSID — same distributed-safe uniqueness as UUID v4, without the index fragmentation |
| Java application wanting a compact, `BIGINT`-sized, sortable, distributed-safe ID with a mature library | TSID — purpose-built for exactly this case |
| IDs are exposed externally (URLs, APIs) and must not reveal creation order or approximate row count | Avoid plain auto-increment (sequential and guessable); consider UUIDv4, or a TSID/Snowflake ID with an additional opaque encoding layer if hiding creation time specifically also matters |

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Don't default to random UUIDs for a B-tree-indexed primary key | Random UUIDs fragment the index on every insert — prefer a time-ordered scheme (UUIDv7, Snowflake, TSID) instead. |
| Use `SEQUENCE` over `IDENTITY` for high-throughput inserts | `SEQUENCE` lets Hibernate batch-fetch IDs; `IDENTITY` forces one round-trip per insert and blocks JDBC batching. |
| Never expose a sequential auto-increment ID as the only public identifier | It leaks approximate row counts and lets an attacker enumerate adjacent records. |
| Assign TSID/Snowflake node IDs deterministically, never by accident | A duplicated node ID across replicas is a real collision risk — derive it from a pod ordinal or explicit config, not a random guess. |
| Pick the ID scheme based on where uniqueness must hold | Single DB → sequence is enough; distributed writers → need a globally-unique, ideally time-ordered scheme. |
| Match ID size to actual need | A `BIGINT`-sized ID (Snowflake/TSID) is half the storage of a UUID in every index that includes it — meaningful at scale. |
