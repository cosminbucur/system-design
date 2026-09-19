BASE (Basically Available, Soft state, Eventual consistency) is the consistency model most NoSQL and distributed systems actually operate under, offered as the explicit counterpart to ACID. Where ACID guarantees that a transaction leaves the database in a single, immediately consistent state, BASE accepts that a distributed system will spend some of its time in a state that hasn't fully settled yet — in exchange for staying available and responsive even when parts of the system can't currently talk to each other.

## 1. BASE vs. ACID — Two Different Bets

| | ACID | BASE |
| --- | --- | --- |
| Priority | Correctness and consistency, immediately | Availability and responsiveness, consistency eventually |
| Behavior under a network partition | May refuse or delay a request rather than risk an inconsistent result | Keeps responding, even if the answer might be briefly stale |
| Typical system | Traditional RDBMS (Postgres, MySQL) in a synchronous setup | Distributed NoSQL stores (Cassandra, DynamoDB in default config) |
| What a client can assume | A read immediately after a write always reflects that write | A read immediately after a write might not yet reflect it, but will eventually |

Neither is universally "better" — they represent a genuine tradeoff, and this is the same underlying choice CAP theorem frames as Consistency versus Availability during a network partition: ACID leans toward consistency, BASE leans toward availability.

## 2. Basically Available

The system guarantees a response to every request, even during a failure or partition — but that response might come from a replica that hasn't yet received the very latest write, rather than the system refusing to answer at all.

```java
// A DynamoDB "eventually consistent" read: always returns something, fast,
// even if a very recent write from another region hasn't fully propagated yet
GetItemRequest request = GetItemRequest.builder()
    .tableName("orders")
    .key(Map.of("orderId", AttributeValue.fromS("ord-123")))
    .consistentRead(false) // basically available: prioritize a fast response over guaranteed freshness
    .build();
```

This is a deliberate design choice, not a bug: a system that would rather answer with slightly stale data than refuse to answer at all is optimizing for uptime and responsiveness over guaranteed real-time accuracy.

## 3. Soft State

The system's state can change over time even without new input, purely as a consequence of replication converging in the background. Unlike a strongly consistent system where "the data" is a single, settled fact the moment a transaction commits, a BASE system's state at any given instant might be mid-transition between replicas.

```
Node A (just received the write):  balance = 150
Node B (hasn't received it yet):   balance = 100   ← will update to 150 shortly, with no new write required
```

"Soft" here means the value you read from one particular node isn't guaranteed to be the final, converged value — it's whatever that node currently has, which may still be catching up.

## 4. Eventual Consistency

Given enough time with no new writes, all replicas will converge to the same value — but there's no guarantee about exactly when "eventually" happens, only that it will, assuming the system keeps propagating updates.

```java
// Write to one node
dynamoDbClient.putItem(request); // acknowledged immediately by one replica

// A read against a DIFFERENT replica, moments later, might still see the old value
GetItemResponse response = dynamoDbClient.getItem(readRequest); // could be stale for a short window
```

This is the same staleness-tolerance question that comes up with caching and read replicas: eventual consistency is acceptable exactly when the business logic reading that data can tolerate a short window of staleness, and unacceptable when it can't (an account balance about to be used for an overdraft decision needs a stronger guarantee than "it'll be correct eventually").

## 5. Read-Your-Own-Writes: A Common Middle Ground

Pure eventual consistency can create a confusing experience: a user updates their profile, immediately reloads the page, and sees the old data because the read happened to hit a replica that hadn't caught up yet. Many systems offer a stronger, narrower guarantee — read-your-own-writes consistency — specifically for this case, without paying for strong consistency on every read.

| Technique | How it works |
| --- | --- |
| Sticky sessions/routing | Route a user's reads to the same replica/region their write went to, for some window after the write |
| Session tokens | The client passes back a token from its last write; reads with that token are guaranteed to reflect it or later |
| Strongly consistent read on demand | Explicitly request a stronger, slower read (`consistentRead(true)` in DynamoDB) only where it's actually needed |

This is a pragmatic middle ground: most reads stay fast and eventually consistent, while the specific case that would otherwise feel broken to a user (not seeing your own recent change) gets a targeted, stronger guarantee.

## 6. Where BASE Shows Up in Practice

| System/pattern | BASE property in play |
| --- | --- |
| Cassandra, DynamoDB (default) | Basically available + eventually consistent reads across replicas |
| A CQRS read-side projection | Soft state — the read model catches up to the write model asynchronously |
| A cache-aside cache | The cache's value is soft state that converges toward the source of truth on the next write/invalidation |
| A read replica of a primary database | Eventually consistent — replication lag is exactly the "not yet converged" window |
| An event-driven microservices architecture | The whole system's cross-service state is eventually consistent by design, not just one datastore |

BASE isn't only a NoSQL database property — it's the same underlying tradeoff showing up anywhere a system chooses to propagate a change asynchronously rather than block on full consistency across every reader.

## 7. Choosing Between ACID and BASE for a Specific Piece of Data

The right framing isn't "is my system ACID or BASE" as a single global answer — most real systems are a mix, choosing per use case.

| Signal | Leans toward |
| --- | --- |
| A brief staleness window would cause real harm (double-spending, overselling, a compliance violation) | ACID / strong consistency |
| Availability matters more than perfect freshness (a product catalog, a social feed, a dashboard) | BASE / eventual consistency |
| The operation spans multiple distributed services with no shared database | BASE, since a distributed ACID transaction isn't available — a saga with compensations is the practical alternative |
| A single service, single database, correctness-critical operation | ACID — there's no reason to give up strong guarantees you can actually have here |

## 8. Best Practices

| Practice | Recommendation |
| --- | --- |
| Choose BASE deliberately, per use case, not as a system-wide default | Availability and eventual consistency are a real tradeoff — apply them where staleness is tolerable, not universally. |
| Use read-your-own-writes techniques where user experience depends on it | Prevents the confusing "I just saved this and it's gone" experience without paying for strong consistency everywhere. |
| Never assume "eventually" means "immediately" | Design UI/business logic around the fact that convergence has no fixed upper bound unless the system explicitly guarantees one. |
| Reserve strong consistency for genuinely correctness-critical operations | Financial balances, inventory used for overselling checks, and compliance-relevant records need ACID-level guarantees, not BASE. |
| Recognize BASE-style tradeoffs outside the database layer too | Caches, read replicas, and CQRS read models all carry the same soft-state, eventually-consistent behavior. |
| Communicate the consistency guarantee explicitly in an API/service contract | Callers need to know whether a response might be stale, rather than discovering it the hard way. |
