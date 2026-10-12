> Explain core concepts, best practices and practical examples for

# architecture

- hexagonal architecture - adapters and ports to separate core from infrastructure (db, api)
- when domain is complex (banking, insurance, pricing)

- event sourcing

# message queues

- rabbitmq
- kafka

# hibernate

- flush()

  // Assume these 2 operations are not in a transaction.

  entityDao.save(entity);
  dependentEntityDao.getByJoinQuery(dependentEntity, entity);

  // The second query could fail as it required data from first query to be persisted.

# microservices

# security

- sticky sessions
- passkeys (passwordless authentication)
  - Storage: Website stores the public key in its database.
  - Unlocking: User device holds the private key, unlocked locally via biometrics.

# fundamentals

- switch
- Object.requireNonNull() - lighter than Optional
- records - immutable data classes, dtos, return multiple values from methods
- default methods (interfaces) - backward compatibility, not forcing implementation of a new method

# logs

# system design

https://www.systemdesignhandbook.com/guides/design-a-stock-exchange-system/#non-functional-requirements-that-drive-the-design

https://algomaster.io/learn/system-design-interviews/design-notification-service

# spring

springa rest https://www.educative.io/courses/rest-api-with-spring-jpa-and-springfox

spring data https://www.educative.io/courses/spring-data-bridging-multiple-databases

# concepts

    1 Database
    2 SQL vs NoSQL
    3 Availability
    4 Reliability
    5 Latency
    6 API Design
    7 REST
    8 Scalability
    9 Load Balancing
    10 Caching
    11 Fault Tolerance
    12 High Availability
    13 CAP Theorem
    14 GraphQL
    15 gRPC
    16 Sharding
    17 Indexing
    18 Leader Election
    19 Consensus
    20 Consistency Models
    21 Event-Driven
    22 Message Queue
    23 Pub/Sub
    24 Denormalization
    25 Secrets Management
    26 Bulkhead
    27 Retry Logic
    28 Timeout
    29 Feature Flags
    30 ACID
    31 Blue-Green Deployment
    32 Canary Release
    33 Erasure Coding
    34 Service Discovery
    35 Sync vs Async
    36 Correlation ID
    37 BFF
    38 Connection Pooling
    39 Monitoring
    40 Alerting
    41 Query Optimization
    42 WebSockets
    43 Cache Stampede
    44 MapReduce
    45 WebRTC
    46 Deserialization
    47 Strangler Pattern
    48 LSM Trees
    49 Batch Processing
    50 Stream Processing
