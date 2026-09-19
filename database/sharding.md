Sharding splits a dataset across multiple independent database instances, each holding a subset of the overall data — a way to scale storage, write throughput, and I/O beyond what a single machine can handle. It's a fundamentally different axis of splitting data than partitioning (which stays within one database instance) or "database per service" (which splits data by bounded context/domain, not by a shard key within what is otherwise one logical dataset). Sharding is also the database-layer analogue of horizontal scaling covered for application servers — the same "add more machines rather than a bigger one" idea, applied to data storage instead of compute.

## 1. Why Shard — And Why It's a Last Resort, Not a First Reach

Sharding solves a real problem (one instance's storage/IOPS/CPU ceiling), but it introduces genuine, ongoing complexity: cross-shard queries, distributed transactions, and rebalancing all become real engineering problems that don't exist on a single instance. The practical order of operations most systems should actually follow, roughly cheapest-and-least-risky first:

```
1. Vertical scaling — a bigger instance
2. Read replicas — solves READ scaling without splitting data (every replica has the full dataset)
3. Caching — see caching, reduces load on the database for hot reads
4. Partitioning — see partitioning, solves query/maintenance pain within one instance
5. Sharding — only once the above are genuinely exhausted and the write/storage volume
   still exceeds what one instance (even a large one) can handle
```

Sharding is usually adopted too early far more often than too late — the operational cost is easy to underestimate until a team is actually living with it.

## 2. Read Replicas Are Not Sharding — A Common Confusion

Worth stating explicitly because the two are often conflated: a **read replica** holds a full copy of the _entire_ dataset and scales read throughput; **sharding** holds a _subset_ of the dataset per instance and scales both read and write throughput (and storage) by splitting the data itself. Read replicas solve "too many reads for one instance to serve"; sharding solves "too much data/writes for one instance to hold at all." They're frequently combined — each shard can itself have its own read replicas.

## 3. Choosing a Shard Key — The Single Most Important Decision

The shard key determines which shard a given row lives on, and it's extremely expensive to change later (it requires physically moving data). The same hot-key risk already covered for a Kafka partition key and a DynamoDB partition key applies here directly: a poorly chosen key sends disproportionate traffic to one shard (a "hot shard"), defeating the entire point of splitting the data in the first place.

```java
// Example: sharding a banking platform's transactions by account ID
public class ShardRouter {
    private static final int SHARD_COUNT = 8;

    public int shardFor(String accountId) {
        return Math.floorMod(accountId.hashCode(), SHARD_COUNT);
        // every query/write for this account consistently routes to the same shard —
        // critical, since an account's transactions living on ONE shard means queries
        // for that account never need to fan out across multiple shards
    }
}
```

Two competing concerns when picking a key: it needs to **distribute load evenly** (avoid a hot shard), and it needs to **keep together the data that's queried together** (an account's own transactions on one shard, so "get this account's transaction history" is a single-shard query, not a scatter-gather across all of them). Getting the second part wrong is the more common real-world mistake — a key chosen purely for even distribution, ignoring query patterns, turns every common query into an expensive cross-shard fan-out.

## 4. Sharding Strategies

| Strategy        | Mechanism                                                                    | Tradeoff                                                                                                                                                                                      |
| --------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hash-based      | `hash(key) % shardCount`                                                     | Even distribution by default; range queries ("all transactions in March") become a fan-out across every shard, since consecutive keys land on unrelated shards                                |
| Range-based     | Contiguous key ranges assigned to shards (e.g., account IDs 1-1M on shard 0) | Range queries stay on one shard; risk of a hot shard if activity skews toward one range (e.g., all new signups landing on the newest, single shard)                                           |
| Directory-based | A lookup service/table maps each key (or key range) explicitly to a shard    | Most flexible for rebalancing (just update the mapping), but the directory itself becomes a critical dependency and potential bottleneck/single point of failure unless made highly available |

There's no universally "correct" strategy — it's a direct tradeoff against your dominant query pattern, the same way choosing a Kafka partition key or a cursor pagination seek key always trades off against how the data will actually be accessed.

## 5. The Hard Part: Cross-Shard Queries and Transactions

This is the actual operational cost of sharding, and the main reason to exhaust every alternative.

```java
// A query that needs data from MULTIPLE shards requires explicit scatter-gather in application code —
// the database has no way to do this for you across independent instances
public List<Transaction> findLargeTransactionsAcrossAllShards(BigDecimal threshold) {
    List<Transaction> results = new ArrayList<>();
    for (int shard = 0; shard < SHARD_COUNT; shard++) {
        results.addAll(shardDataSources.get(shard).findLargeTransactions(threshold)); // fan out
    }
    return results.stream()
        .sorted(Comparator.comparing(Transaction::getAmount).reversed()) // merge results after the fact
        .toList();
}
```

A query confined to one shard (because the shard key was chosen well) is just a normal query. A query that must span shards requires the application to query every relevant shard and merge results itself — there's no cross-shard `JOIN`, and no cross-shard transaction in the traditional ACID sense. For an operation that must update data on two different shards atomically, the practical answer is the same Saga pattern — a sequence of local, per-shard transactions with compensating actions on failure — rather than attempting a distributed two-phase commit across shards, which is operationally painful and rarely worth it at this scale.

## 6. Resharding — Adding or Removing Shards

Naive `hash(key) % shardCount` sharding has a serious problem: changing `shardCount` (going from 8 shards to 12) changes the shard assignment for nearly _every_ key, requiring almost the entire dataset to be physically moved. **Consistent hashing** is the standard fix: keys and shards are placed on a hash ring, and adding/removing a shard only remaps the keys immediately adjacent to that shard on the ring — a small, bounded fraction of the data moves, not nearly all of it.

```
Naive modulo hashing:      changing shard count reshuffles ~(N-1)/N of all keys
Consistent hashing:        changing shard count remaps only ~1/N of all keys
```

This is a genuinely hard, high-stakes operation regardless of the hashing scheme — resharding a live production system means migrating data while it's still being read and written, which is why directory-based sharding is often preferred specifically for its rebalancing flexibility, even though it adds an extra lookup hop on every query.

## 7. Sharding Middleware and NoSQL's Built-In Sharding

Rather than hand-rolling the routing/scatter-gather logic above, several tools handle it for a relational database:

| Tool           | Approach                                                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Vitess         | Shards MySQL, originally built at YouTube; presents a single logical database to the application                               |
| Citus          | Shards Postgres, distributes tables across worker nodes, understands distributed queries/joins on the shard key                |
| ShardingSphere | A Java-ecosystem sharding middleware/proxy — can shard an existing JDBC-based application with less application-level rewiring |

Many NoSQL databases (DynamoDB, Cassandra) build sharding (by partition key) directly into the product from the start, rather than it being something layered on top of a relational database after the fact — this is a real part of why teams reach for them specifically when they know from the outset that horizontal write/storage scale is a hard requirement, not an afterthought.

## 8. Best Practices

| Practice                                                                                           | Recommendation                                                                                                  |
| -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Exhaust vertical scaling, read replicas, caching, and partitioning first                           | Sharding's operational cost (cross-shard queries, resharding) is real; most systems never actually need it.     |
| Choose the shard key around your dominant query pattern, not just even distribution                | A key that scatters commonly-queried-together data across shards turns routine queries into expensive fan-outs. |
| Avoid naive modulo-based sharding for anything expected to reshard later                           | Consistent hashing bounds the fraction of data that must move when shard count changes.                         |
| Design for eventual cross-shard consistency via Sagas, not distributed transactions                | A cross-shard 2PC is operationally painful and rarely the right tradeoff.                                       |
| Consider a sharding middleware (Vitess/Citus/ShardingSphere) before hand-rolling routing           | Scatter-gather, rebalancing, and shard-aware query planning are hard to get right from scratch.                 |
| Consider a NoSQL store with built-in sharding when horizontal scale is a known day-one requirement | DynamoDB/Cassandra bake shard-key-based partitioning into the data model itself.                                |
| Never treat sharding and read replicas as solving the same problem                                 | Replicas scale reads over the full dataset; sharding scales writes/storage by splitting the dataset.            |
