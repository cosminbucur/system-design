# System design

- 🎯 [System Design](system-design.md)
- 📗 [API performance Guide](api-performance-guide.md)

# Java

- [Streams](java/streams.md)
- [Functional Programming](java/functional-programming.md)
- [Modern Java features](java/modern-java-features.md)

# Data Structures

- [Collections](java/collections.md)

# Data structures & Algorithms

- [Time - Space Complexity](algorithms/time-space-complexity.md) Big O

- Linear
  - 📦 [Arrays](algorithms/arrays.md)
  - 📦 [Hashing](algorithms/hashing.md)
  - [Two-pointers](algorithms/two-pointers.md)
  - 📦 [Stack](algorithms/stack.md)
  - [Binary-search](algorithms/binary-search.md)
  - [Sliding-window](algorithms/sliding-window.md)
  - 📦 [Linked-list](algorithms/linked-list.md)

- Non Linear
  - 📦 [Trees](algorithms/trees.md)
  - 📦 [Tries](algorithms/tries.md)
  - 📦 [Heap](algorithms/heap.md)
  - 📦 [Queue](algorithms/queue.md) - music player, printer
    - [Priority Queue](algorithms/queue-priority.md) - airport
    - [Circular Queue](algorithms/queue-circular.md) - load balancing
    - [Deque](algorithms/queue-deque.md) - browser history
    - [Blocking/Non-Blocking Queue](algorithms/queue-blocking.md) - thread safe
  - [Intervals](algorithms/intervals.md)

- Paradigms
  - [Greedy](algorithms/greedy.md)
  - [Backtracking](algorithms/backtracking.md)
  - 📦 [Graphs](algorithms/graphs.md)
  - [Dynamic programming 1D](algorithms/dynamic-programming-1d.md)
  - [Dynamic programming 2D](algorithms/dynamic-programming-2d.md)
  - [Bit-manipulation](algorithms/bit-manipulation.md)
  - [Math-geometry](algorithms/math-geometry.md)

# Java Multithreading

- [Concurrency vs Parallelism](java-multithreading/concurrency.md)

- 🎯 [Multithreading](java-multithreading/_multithreading.md) ExecutorService
- 📗 [Thread Safety Guide](java-multithreading/thread-safety-guide.md)

- Patterns
  - [Pattern: Read/Write Lock](java-multithreading/readwritelock.md) ReadWriteLock - in-memory cache
  - [Pattern: Barrier](java-multithreading/barrier.md) CyclicBarrier - multi-part file processing
  - [Pattern: Fork/Join](java-multithreading/fork-join.md) - recursive file system processing
  - 📗 [Tomcat Thread Pool Guide](java-multithreading/tomcat-thread-pool-guide.md)

- [JVM memory model](java-multithreading/java-memory-model.md)

# Java Async

- 🎯 [Async Programming](java-async/async-programming.md)
- [Pattern: Future and Promise](java-async/future-and-promise.md) CompletableFutures
- [Virtual Threads](java-multithreading/virtual-threads.md)

# Java Reactive

- 🎯 [Reactive Programming](java-reactive/reactive-programming.md) Reactor

# Database

- [Databases](database/_databases.md)
- 📦 [ACID](database/acid.md)
- 📦 [CAP Theorem](database/cap-theorem.md)
- 📦 [BASE - NoSQL](database/base-principles.md)

- [JPA Persistence](database/jpa-persistence.md)
- [JDBI](database/jdbi.md)
- ℹ️ [Pagination](database/pagination.md)
- ℹ️ [Cursor Pagination](database/pagination-cursor.md)

- [Db migration](database/db-migration.md)
- [Audit](database/auditing.md)
- 📗 [HikariCP Guide](database/hikaricp-guide.md)
- 📗 [Lock Contention Guide](database/lock-contention-guide.md)

# Database Scaling

- [DB Scaling](database/database-scaling.md)
- [Db Indexing](database/db-indexing.md)
  - 📗 [Composite Index Guide](database/composite-index-guide.md)
- [ID generation](database/id-generation.md)
- [Db Replication](database/db-replication.md) - solves READ scaling without splitting data
- [Caching](database/caching.md) - reduces load on the database for hot reads
  - [Cache Eviction](database/cache-eviction.md)
- [Partitioning](database/partitioning.md) - solves query/maintenance pain within one instance
- [Sharding](database/sharding.md) - splits a dataset across multiple independent database instances, each holding a subset of the overall data

- 📗 [Slow Query Guide](database/slow-query-guide.md)

# API

- API Styles
  - [REST](api/rest.md)
  - [GraphQL](api/graphql.md)
  - [gRPC](api/grpc.md)
  - [Websockets](api/websockets.md)

  - ℹ️ [Validation](validation.md)
  - ℹ️ [Error Handling](api/error-handling.md)
  - ℹ️ [API idempotency](api/api-idempotency.md)
  - ℹ️ [API latency tiers](api/api_latency_tiers.md)

- API integration patterns
  - [Webhooks](api/webhooks.md)

# Testing

- [Unit Tests](test/test-unit.md) Mockito
- [Integration Tests](test/test-integration.md) Testcontainers
- [End-to-End Tests](test/test-e2e.md) Playwright
- [Performance Tests](test/test-performance.md) Gatling, K6
- [BDD Tests](test/test-bdd.md) Cucumber
- [Architecture Tests](test/test-architecture.md) Archunit

- [Profiling](test/profiling.md) JFR
- 📗 [Memory Leaks Guide](test/java-memory-leaks-guide.md)
- 📗 [GC Pause Guide](test/java-gc-pause-guide.md)

# Design principles

- [Clean code](design-principles/clean-code.md)
- [SOLID Principles](design-principles/solid-principles.md)

- [Design Patterns - Behavior](design-principles/design-patterns-behavior.md)
- [Design Patterns - Creation](design-principles/design-patterns-creation.md)
- [Design Patterns - Structure](design-principles/design-patterns-structure.md)

- [DDD](design-principles/ddd.md)
- [UI patterns](design-principles/ui-patterns.md)

# Architecture

- [Architecture patterns](architecture/architecture.md)

# Processing

- 🎯 [Batch Processing](processing/batch-processing.md) Spring Batch, Spark
- 🎯 [Data Streaming](processing/data-streaming.md) Kafka Streams
- [Big Data](processing/big-data.md) Hadoop
- 📗 [Serialization Guide](processing/serialization_guide.md)
- 📗 [Serialization Avro Guide](processing/avro_guide.md)

# Security

- 🎯 [Security](security/security.md) OIDC, Oauth2, JWT
  - ℹ️ [Authentication](security/authentication.md)
  - ℹ️ [Encryption](security/encryption.md) Symmetric vs Asymmetric
  - ℹ️ [TCP, SSL/TLS, HTTPS](security/ssl-tls-https.md)
  - ℹ️ [Access Token](microservices/access-token.md)
  - ℹ️ [Session Management](microservices/session-management.md)
  - 📦 [API gateway](microservices/api-gateway.md)
  - [DNS](security/dns.md)

# Edge Layer

- 🎯 [Edge Computing](layer-edge/edge-computing.md)
- [CDN](layer-edge/cdn.md)

# Microservices

- 🎯 [Distributed Systems](microservices/distributed-system.md)
- [Microservices Patterns](microservices/microservices-patterns.md)

- Decomposition
  - 📦 [Decompose by business capability](microservices/decompose-by-capability.md)
  - 📦 [Decompose by subdomain](microservices/decompose-by-subdomain.md)

- Data
  - 📦 [Database per Service](microservices/database-per-service.md)

- Service Discovery
  - 📦 [Server/Client-side discovery](microservices/service-discovery.md)

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
    - 📦 [CQRS](microservices/cqrs.md) - pattern
  - 📦 [Event Sourcing](microservices/event-sourcing.md) - architecture

- Messaging
  - 🎯 [Messaging](processing/_messaging.md)
  - 📦 [Pub/Sub](processing/pub-sub.md)
  - 📦 [Message Queues](processing/message-queues.md)
  - ℹ️ [Backpressure](processing/backpressure.md)

  - [Remote Procedure Invocation]

# Resilience

- Downstream
  - Timeout
  - 📦 [Retry/Fallback](resilience/retry-fallback.md) handle recovery
- Upstream
  - ℹ️ [Load Shedding vs. Rate Limiting](resilience/load-shed-vs-rate-limit.md)
  - 📦 [Load Shedding](resilience/load-shedding.md) reject requests based on the service's own real-time health
  - 📦 [Rate Limiter / Time Limiter / Bulkhead](resilience/rate-time-limiter.md) cap volume / duration / concurrent capacity
  - [Health check API]
- 📦 [Load Balancing](resilience/load-balancing.md)
- 📦 [Circuit Breaker](resilience/circuit-breaker.md) stops calling something that's clearly broken - Resilience4j

- Deployment
  - [Single Service per Host]
  - [Multiple Services per Host]

- Testing
  - [Service Integration Contract Test]
  - [Service Component Test]

- UI
  - [Server-side page fragment composition]
  - [Client-side UI composition]

- Cross-cutting concerns
  - [Microservice Chassis]
  - [Externalized Configuration]

# Cloud

- [Cloud Providers](cloud/cloud-providers.md)
- [CDN](cloud/cdn.md)

# DevOps and CI/CD

- [CI/CD](devops/ci-cd.md)
- [Continous Deployment](devops/continuous-deployment.md)
- [CI/CD Pipeline Guide](devops/ci-cd-pipeline-guide.md)

- [Containers](devops/docker.md) Docker
- [Orchestration](devops/kubernetes.md) Kubernetes
- [GitOps](devops/gitops.md) ArgoCD

- 🎯 [Observability](devops/observability.md)
  - [Logs - concepts]()
  - [Logs](devops/logs.md)
  - [Metrics](devops/metrics.md)
  - [Distributed Tracing](devops/distributed-tracing.md)
  - [Alerting](devops/alerting.md) Prometheus + Grafana
  - 📦 [Distributed Logging](devops/distributed-logging.md) correlation ID + log aggregation
  - [Distributed Tracing]
  - [Audit Logging]
  - [Application metrics]
  - [Exception tracking]

# Multi tenancy

- [Multi Tenant Core Banking](multi-tenant-core-banking.md)

# AI

- [AI agents](ai/ai-agents.md)
