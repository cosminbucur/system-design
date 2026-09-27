# System design

- 🔴 🎯 [System Design](system-design.md)
- 📗 [API performance Guide](api-performance-guide.md)
- [Multi Tenant Core Banking](multi-tenant-core-banking.md)

# Java

- 🔴 [Exceptions](java/exceptions.md)
- 🔴 [Streams](java/streams.md)
- 🔴 [Functional Programming](java/functional-programming.md)
- 🔴 [Sorting: Comparable and Comparator](java/sorting.md)
- 🔴 [Modern Java features](java/modern-java-features.md)

# Data Structures

- 🔴 [Collections](data-structures/collections.md)
- 🔴 [Java List Implementations](data-structures/java-list.md)
- 🔴 [Java Map Implementations](data-structures/java-map.md)
- 🔴 [Java Set Implementations](data-structures/java-set.md)

# Algorithms

- [Time - Space Complexity](algorithms/time-space-complexity.md) Big O

- Sorting
  - [Sorting Algorithms](algorithms/algorithms-sort.md)

- Searching
  - [Searching Algorithms](algorithms/algorithms-search.md)

- Linear
  - 🔴 📦 [Arrays](algorithms/arrays.md)
  - 🔴 📦 [Hashing](algorithms/hashing.md)
  - [Two-pointers](algorithms/two-pointers.md)
  - 📦 [Stack](algorithms/stack.md)
  - 🟠 [Binary Search](algorithms/binary-search.md)
  - [Sliding Window](algorithms/sliding-window.md)
  - 🔴 📦 [Linked List](algorithms/linked-list.md)

- Non Linear
  - 📦 [Trees](algorithms/trees.md)
  - 📦 [Tries](algorithms/tries.md)
  - 📦 [Heap](algorithms/heap.md)
  - 📦 [Queue](algorithms/queue.md) - music player, printer
    - [Priority Queue](algorithms/queue-priority.md) - airport
    - [Circular Queue](algorithms/queue-circular.md) - load balancing
    - [Deque](algorithms/queue-deque.md) - browser history
    - [Blocking/Non-Blocking Queue](algorithms/queue-blocking.md)
  - [Intervals](algorithms/intervals.md)

- Paradigms
  - 🔴 [Greedy](algorithms/greedy.md)
  - 🔴 [Backtracking](algorithms/backtracking.md)
  - 📦 [Graphs](algorithms/graphs.md)
  - [Dynamic programming 1D](algorithms/dynamic-programming-1d.md)
  - [Dynamic programming 2D](algorithms/dynamic-programming-2d.md)
  - [Bit-manipulation](algorithms/bit-manipulation.md)
  - [Math-geometry](algorithms/math-geometry.md)

# Java Multithreading

- [I/O vs CPU Bound](java-multithreading/io-cpu-bound.md)
- [Concurrency vs Parallelism](java-multithreading/concurrency.md)

- 🔴 🎯 [Multithreading](java-multithreading/_multithreading.md) ExecutorService
- 🔴 📗 [Thread Safety Guide](java-multithreading/thread-safety-guide.md)
- 🔴 📗 [Immutability Guide](java-multithreading/java-immutability-guide.md)

- 🔴 [ThreadLocal vs ScopedValue](java-multithreading/threadlocal-scopedvalue.md)

- 🔴 Synchronization mechanisms
  - Low-level keywords
    - [synchronized](java-multithreading/synchronized-deadlock.md)
    - 🔴 [Volatile vs Atomic Classes](java-multithreading/volatile-atomic.md)
  - Explicit locks
    - 🔴 [ReentrantLock](java-multithreading/reentrantlock.md)
    - ReentrantReadWriteLock
    - StampedLock
  - [High-level thread synchronizers](java-multithreading/thread-synchronizers-guide.md)
    - CountDownLatch
    - CyclicBarrier
    - Semaphore
    - Phaser

- Thread-safe collections
  - 🔴 [ConcurrentHashMap](data-structures/concurrent-hashmap.md)
  - CopyOnWriteArrayList
  - BlockingQueue

- 🟠 Patterns
  - 📦 [Read/Write Lock](java-multithreading/readwritelock.md) ReadWriteLock - in-memory cache
  - 📦 [Barrier](java-multithreading/barrier.md) CyclicBarrier - multi-part file processing
  - 📦 [Fork/Join](java-multithreading/fork-join.md) - recursive file system processing

# Java Async

- 🟠 🎯 [Async Programming](java-async/async-programming.md)
- 📦 [Future and Promise](java-async/future-and-promise.md) CompletableFutures
- [Virtual Threads](java-async/virtual-threads.md)
- [NIO - Non-Blocking IO](java-async/nio.md)

# Java Reactive

- 🎯 [Reactive Programming](java-reactive/reactive-programming.md) Reactor

# Database

- 🔴 [Databases](database/_databases.md)
- 🔴 [Normalization/Denormalization](database/database-normalization-guide.md)
- [Materialized Views](database/materialized-views.md)

- Transactions
  - 🔴 [Isolation levels](database/isolation-levels.md)
  - 🔴 📗 [Spring Transactions Guide](database/spring-transactions-guide.md)
  - 🔴 [Spring Transactions Propagation](database/spring-transaction-propagation.md)
  - 🔴 📦 [ACID](database/acid.md)
  - 🔴 📦 [CAP Theorem](database/cap-theorem.md)
  - 📦 [BASE - NoSQL](database/base-principles.md)

- 🔴 [Locking: Optimistic vs Pessimistic](database/locking-optimist-pessimist.md)
- 🔴 [JPA Persistence](database/jpa-persistence.md)
- 🔴 [Hibernate cache](database/hibernate-cache.md)
- [JDBI](database/jdbi.md)
- 🔴 [Pagination](database/pagination.md)
- [Cursor Pagination](database/pagination-cursor.md)

- [DB migration](database/db-migration.md)
- [Audit](database/auditing.md)
- 📗 [HikariCP Guide](database/hikaricp-guide.md)
- 📗 [Lock Contention Guide](database/lock-contention-guide.md)

# Database Scaling

- [DB Scaling](database/database-scaling.md)
- [Db Indexing](database/db-indexing.md)
  - 📗 [Composite Index Guide](database/composite-index-guide.md)
- [ID generation](database/id-generation.md)
- [Db Replication](database/db-replication.md) - solves READ scaling without splitting data
- 🔴 [Caching](database/caching.md) - reduces load on the database for hot reads
  - [Cache Eviction](database/cache-eviction.md)
- [Partitioning](database/partitioning.md) - solves query/maintenance pain within one instance
- [Sharding](database/sharding.md) - data split on multiple instances

- 🟠 📗 [Slow Query Guide](database/slow-query-guide.md)

# API

- 🟠 [HTTP protocols](api/http-protocols.md)

- API Styles
  - 🔴 [API](api/_api.md)
  - 🔴 [REST](api/rest.md)
  - [GraphQL](api/graphql.md)
  - [gRPC](api/grpc.md)
  - [Websockets](api/websockets.md)

  - 🟠 [Validation](validation.md)
  - 🟠 [Error Handling](api/error-handling.md)
  - 🔴 [API idempotency](api/api-idempotency.md)
  - 🔴 [Deduplication](api/deduplication.md)
  - [API latency tiers](api/api_latency_tiers.md)

- API integration patterns
  - [Webhooks](api/webhooks.md)

# Testing

- 🔴 [Unit Tests](test/test-unit.md)
  - [JUnit](test/junit.md)
  - [Mockito](test/mockito.md)
- 🔴 [Integration Tests](test/test-integration.md)
  - [Testcontainers](test/testcontainers.md)
- [End-to-End Tests](test/test-e2e.md) Playwright
- 🟠 [Performance Tests](test/test-performance.md) Gatling, K6
- [BDD Tests](test/test-bdd.md) Cucumber
- [Architecture Tests](test/test-architecture.md) Archunit

- 📦 [Pattern: Service Integration Contract Test]
- 📦 [Pattern: Service Component Test]

- Code Quality
  - 🎯 [Static Analysis](test/static-analysis.md)
  - [Code Coverage: JaCoCo](test/jacoco.md)
  - [SonarQube](test/sonarqube.md)

# Design principles

- 🔴 [Clean code](design-principles/clean-code.md)
- 🔴 📗 [Java OOP Guide](design-principles/java-oop-guide.md)
- 🔴 [SOLID Principles](design-principles/solid-principles.md)

- 🔴 [Design Patterns - Behavior](design-principles/design-patterns-behavior.md)
- 🔴 [Design Patterns - Creation](design-principles/design-patterns-creation.md)
- 🔴 [Design Patterns - Structure](design-principles/design-patterns-structure.md)

- [DDD](design-principles/ddd.md)
- [UI patterns](design-principles/ui-patterns.md)

# Architecture

- 🎯 [Architecture Layers](architecture/_architecture-layers.md)
- 🎯 [Architecture patterns](architecture/architecture.md)
- 🎯 [IoT, Edge, Cloud](architecture/iot-edge-cloud.md)

# Edge Layer

- [DNS](layer-edge/dns.md)
- [CDN](layer-edge/cdn.md)
- 📦 [API gateway](layer-edge/api-gateway.md)

# Infrastructure Layer

- 📦 [Pattern: Server/Client-side discovery](layer-infrastructure/service-discovery.md)
- 📦 [Pattern: Internal Secret Management](layer-infrastructure/internal-secret-management.md)
- 📦 [Pattern: Externalized Configuration](layer-infrastructure/externalized-configuration.md)
- [Microservice Chassis]

- 🎯 [Observability](layer-infrastructure/observability.md)
  - [Logs](layer-infrastructure/logs.md) Loki
    - [Log Types](layer-infrastructure/log-types.md)
    - 📗 [API gateway logging guide](layer-infrastructure/api-gateway-logging-guide.md)
    - 📦 [Distributed Logging](layer-infrastructure/distributed-logging.md) correlation ID + log aggregation
    - [Audit Logging]
  - [Metrics](layer-infrastructure/metrics.md)
    - [Prometheus](layer-infrastructure/prometheus.md)
    - [Distributed Tracing](layer-infrastructure/distributed-tracing.md) Jaeger
  - [Alerting](layer-infrastructure/alerting.md) Prometheus + Grafana
  - [Exception tracking]

  - [Profiling](layer-infrastructure/profiling.md) JFR
    - 🔴 [JVM Architecture](layer-infrastructure/jvm-architecture.md)
    - 🔴 [JVM Heap vs Stack](layer-infrastructure/jvm-heap-stack.md)
    - 🔴 [Garbage Collection](layer-infrastructure/garbage-collection.md)
    - 🔴 📗 [GC Guide](layer-infrastructure/java-gc-guide.md)
    - 🔴 📗 [GC Pause Guide](layer-infrastructure/java-gc-pause-guide.md)
    - 📗 [Tomcat Thread Pool Guide](layer-infrastructure/tomcat-thread-pool-guide.md)
    - 📗 [Java Heap Dump Guide](layer-infrastructure/java-heap-dump-guide.md)
    - 📗 [Memory Leaks Guide](layer-infrastructure/java-memory-leaks-guide.md)

- Infrastructure as Code
  - [Terraform](layer-infrastructure/terraform.md)

- Linux
  - [Linux File System](layer-infrastructure/linux-file-system.md)

# Application Layer

- Spring
  - 📗 [Spring Bean Scope guide](layer-application/spring-bean-scope-guide.md)

# Data Layer

- Data
  - 📦 [Database per Service](microservices/database-per-service.md)

- Processing
  - 🎯 [Batch Processing](processing/batch-processing.md) Spring Batch, Spark
  - 🎯 [Data Streaming](processing/data-streaming.md) Kafka Streams
  - [Big Data](processing/big-data.md) Hadoop
  - 📗 [Serialization Guide](processing/serialization-guide.md)
  - 📗 [Serialization Avro Guide](processing/avro-guide.md)

- Messaging
  - 🔴 🎯 [Messaging](processing/_messaging.md)
  - 🔴 📦 [Pub/Sub](processing/pub-sub.md)
  - 🔴 📦 [Message Queues](processing/message-queues.md)
  - [Backpressure](processing/backpressure.md)

- Transactional Messaging
  - 📦 [Transactional Outbox](microservices/transactional-outbox.md)
  - Event Publisher
    - 📦 [Polling Publisher](microservices/polling-publisher.md)
    - 📦 [Change Data Capture](microservices/cdc.md)

- Service Collaboration
  - Command
    - 📦 [Saga](microservices/saga.md)
    - [Command-side replica]
  - Query
    - [API Composition]
    - 📦 [CQRS](microservices/cqrs.md)
  - 📦 [Event Sourcing](microservices/event-sourcing.md)

  - [Remote Procedure Invocation]

# Security

- 🔴 🎯 [Security](security/security.md) OIDC, Oauth2, JWT
  - 🔴 [Authentication](security/authentication.md)
  - [Encryption](security/encryption.md) Symmetric vs Asymmetric
  - [SSL/TLS, HTTPS](security/ssl-tls-https.md)
  - [Access Token](security/access-token.md)
  - [Session Management](security/session-management.md)
  - [Session Management Guide](security/session-context-management-guide.md)

- 🔴 [Spring Security](security/spring-security.md)
- [Password Storage Guide](security/password-storage-guide.md)

# Microservices

- 🎯 [Distributed Systems](microservices/distributed-system.md)
- [Microservices Patterns](microservices/microservices-patterns.md)

- Decomposition
  - 📦 [Decompose by business capability](microservices/decompose-by-capability.md)
  - 📦 [Decompose by subdomain](microservices/decompose-by-subdomain.md)

- 🔴 [Spring Cloud](microservices/spring-cloud.md)

# Resilience

- 📦 [Load Balancing](resilience/load-balancing.md)
- 📦 [Circuit Breaker](resilience/circuit-breaker.md) stops calling something that's clearly broken - Resilience4j

- Downstream
  - 📦 [Timeout](microservices/timeout.md)
  - 📦 [Retry/Fallback](microservices/retry-fallback.md) handle recovery
- Upstream
  - [Load Shedding vs. Rate Limiting](microservices/load-shed-vs-rate-limit.md)
  - 📦 [Load Shedding](microservices/load-shedding.md) reject requests based on the service's own real-time health
  - 📦 [Rate Limiter / Time Limiter / Bulkhead](microservices/rate-time-limiter.md) cap volume / duration / concurrent capacity
  - 📦 [Health check API](microservices/healthcheck-api.md)

- Deployment
  - [Single Service per Host]
  - [Multiple Services per Host]

# UI

- [ReactJs](react/react.md)
  - [Server-side page fragment composition]
  - [Client-side UI composition]

# Cloud

- [Cloud Providers](cloud/cloud-providers.md)
- [AWS Microservices](cloud/aws-microservices.md)

# DevOps and CI/CD

- [CI/CD](devops/ci-cd.md)
- [Continous Deployment](devops/continuous-deployment.md)
- [CI/CD Pipeline Guide](devops/ci-cd-pipeline-guide.md)

- [Containers](devops/docker.md) Docker
- [Orchestration](devops/kubernetes.md) Kubernetes
- [GitOps](devops/gitops.md) ArgoCD

# Multi tenancy

# AI

- [AI agents](ai/ai-agents.md)
- [Cursor Agent](ai/cursor-agent.md)

🔴 Important
🟠 Nice to have
🎯 Topic
📗 Guide
