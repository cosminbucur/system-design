> Explain core concepts, best practices and practical examples for

# Fundamentals

- Object.requireNonNull() - lighter than Optional
- default methods (interfaces) - backward compatibility, not forcing implementation of a new method

# Algorithms

- [techinterviewhandbook - algorithms](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/)

- [Ashish - Algorithm patterns](https://www.youtube.com/watch?v=DjYZk8nrXVY)

# System design

- [Ashish - System Design](https://www.youtube.com/watch?v=Yc8QS10payQ)

https://www.systemdesignhandbook.com/guides/design-a-stock-exchange-system/#non-functional-requirements-that-drive-the-design

https://algomaster.io/learn/system-design-interviews/design-notification-service

### Design Patterns

- [Ashish - Design Patterns](https://www.youtube.com/watch?v=rpt8PpIPhJQ)

### Low Level Design

- 🔥 [Ashis - Low Level Design - code](https://github.com/ashishps1/awesome-low-level-design)

- [Ashish - Low Level Design](https://www.youtube.com/watch?v=OhCp6ppX6bg)

# Architecture

- hexagonal architecture - adapters and ports to separate core from infrastructure (db, api)
  - when domain is complex (banking, insurance, pricing)
  - https://www.thoughtworks.com/insights/blog/architecture/hexagonal-architecture-explained-practical-example
  - https://github.com/Kinle/pizzaco-hexagonal-ddd/

- event sourcing

# Microservices

### message queues

- rabbitmq
- kafka

# WEB

### spring

springa rest https://www.educative.io/courses/rest-api-with-spring-jpa-and-springfox

spring data https://www.educative.io/courses/spring-data-bridging-multiple-databases

# Persistence

### #hibernate

- flush()

  // Assume these 2 operations are not in a transaction.

  entityDao.save(entity);
  dependentEntityDao.getByJoinQuery(dependentEntity, entity);

  // The second query could fail as it required data from first query to be persisted.

### caching

https://aws.plainenglish.io/caching-in-microservices-self-managed-vs-aws-managed-10789a8889d5

# Security

- sticky sessions

  When a user logs into an online shopping website, the load balancer continues sending all requests from that user to the same server. This allows the user's login session and shopping cart to remain available without requiring re-authentication.

- SSL/TLS termination

  the process in which a load balancer decrypts incoming HTTPS traffic before forwarding the request to backend servers. This reduces the processing overhead on application servers and simplifies SSL certificate management.

  Example: A banking website receives encrypted HTTPS requests from users. The load balancer decrypts the requests, forwards them to the web servers over the internal network, and then encrypts the responses before sending them back to users.

- passkeys (passwordless authentication)
  - Storage: Website stores the public key in its database.
  - Unlocking: User device holds the private key, unlocked locally via biometrics.

# observability

https://aws.plainenglish.io/implementing-observability-in-a-kubernetes-environment-open-source-vs-aws-native-ab016670d9b0

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
