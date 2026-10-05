# System design

> 🔴 Important 🟠 Nice to have 🎯 Topic 📗 Guide 🆕 Recently added

- ℹ️ Live Page: https://cosminbucur.github.io/system-design/

- 🎯 [Architect Roadmap](1-roadmap/architect-roadmap.md)
- 🎯 [Algorithms Roadmap](1-roadmap/algorithms-roadmap.md)
- 🔴 🎯 [System Design Roadmap](1-roadmap/system-design.md)
- 🔴 📗 [DDD Structure](1-roadmap/ddd-structure.md)
  - [Multi Tenant Core Banking](1-roadmap/multi-tenant-core-banking.md)

- 🎯 [Interview Prep](1-roadmap/interview-prep.md)

# Java

- 🔴 [Exceptions](java/exceptions.md)
- 🔴 [Streams](java/streams.md)
- 🔴 [Functional Programming](java/functional-programming.md)
- 🔴 [Sorting: Comparable and Comparator](java/sorting.md)
- 🔴 [Modern Java features](java/modern-java-features.md)

# Data Structures

- 🔴 [Collections](data-structures/collections.md)
- 🔴 [Java List Implementations](data-structures/java-list.md)
  - ArrayList
  - LinkedList
  - TreeList
- 🔴 [Java Map Implementations](data-structures/java-map.md)
  - HashMap
  - LinkedHashMap
  - TreeMap
- 🔴 [Java Set Implementations](data-structures/java-set.md)
  - HashSet
  - LinkedHashSet
  - TreeSet

- Thread-safe
  - 🔴 [ConcurrentHashMap](data-structures/concurrent-hashmap.md)
  - 🆕 🔴 [CopyOnWriteArrayList](data-structures/copyonwritearraylist.md)
  - 🆕 [Blocking /Non-Blocking Queue](algorithms/queue-blocking.md)
    - Blocking (ArrayBlockingQueue)
    - Non-Blocking (ConcurrentLinkedQueue)

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
  - 🔴 📦 [Trees](algorithms/trees.md)
  - 📦 [Tries](algorithms/tries.md)
  - 📦 [Heap](algorithms/heap.md)
  - 🔴 📦 [Queue](algorithms/queue.md) - music player, printer
    - [PriorityQueue](algorithms/queue-priority.md) - airport
    - [CircularQueue](algorithms/queue-circular.md) - load balancing
    - [Deque - ArrayDequeue](algorithms/queue-deque.md) - browser history
  - [Intervals](algorithms/intervals.md)

- Paradigms
  - 🔴 [Greedy](algorithms/greedy.md)
  - 🔴 [Backtracking](algorithms/backtracking.md)
  - 📦 [Graphs](algorithms/graphs.md)
  - [Dynamic programming 1D](algorithms/dynamic-programming-1d.md)
  - [Dynamic programming 2D](algorithms/dynamic-programming-2d.md)
  - [Bit-manipulation](algorithms/bit-manipulation.md)
  - [Math-geometry](algorithms/math-geometry.md)

# Java Multithreading & Concurrency

- Fundamentals
  - [I/O vs CPU Bound](java-multithreading/io-cpu-bound.md)
  - [Concurrency vs Parallelism](java-multithreading/concurrency.md)
  - 🔴 🎯 [Multithreading](java-multithreading/_multithreading.md) ExecutorService

- Thread Safety
  - 🔴 📗 [Thread Safety Guide](java-multithreading/thread-safety-guide.md)
  - 🔴 📗 [Immutability Guide](java-multithreading/java-immutability-guide.md)
  - 🔴 [ThreadLocal](java-multithreading/threadlocal.md)

- 🔴 Synchronization mechanisms
  - Low-level primitives
    - [synchronized](java-multithreading/synchronized-deadlock.md)
    - 🔴 [Volatile vs Atomic Classes](java-multithreading/volatile-atomic.md)
  - Explicit locks
    - 🔴 [ReentrantLock](java-multithreading/lock-reentrant.md)
    - 📦 [ReadWriteLock](java-multithreading/lock-readwrite.md) - in-memory cache
    - 🆕 [ReentrantReadWriteLock](java-multithreading/lock-reentrantreadwrite.md) - in-memory cache
    - 🆕 [StampedLock](java-multithreading/lock-reentrantreadwrite.md) - in-memory cache (read)
  - High-level thread synchronizers
    - 📗 [Thread Synchronizers Guide](java-multithreading/thread-synchronizers-guide.md)
    - 🆕 [CountDownLatch](java-multithreading/countdownlatch.md) - matchmaking MMO
    - 🆕 📦 [CyclicBarrier](java-multithreading/barrier.md) - multi-part file processing
    - 🆕 📦 [Phaser](java-multithreading/phaser.md) - multi-stage simulation
    - 🆕 📦 [Semaphore](java-multithreading/semaphore.md) - rate limit/throtting, db connection pools

- CPU-Bound Parallelism
  - 📦 [Fork/Join](java-multithreading/fork-join.md) - recursive file system processing

# Java I/O

- 🆕 [NIO.2](java-io/nio2.md) - path and file
- 🟠 [NIO - Non-Blocking IO](java-io/nio.md) - buffer, channels and selectors
- 🆕 📗 [Java Networking Guide](java-iio/java-networking-guide.md)

# Java Asynchronous

- 🟠 🎯 [Async Programming](java-async/async-programming.md)
- 📦 [Future and Promise](java-async/future-and-promise.md) CompletableFutures
- [Virtual Threads](java-async/virtual-threads.md)
- 🆕 [Scoped Value](java-async/scopedvalue.md)
- 🆕 📗[Webflux to Virtual Threads](java-async/webflux-to-virtual-threads.md)

# Java Reactive

- 🎯 [Reactive Programming](java-reactive/reactive-programming.md) Reactor - Webflux

# Persistence

- SQL

- Relational Databases
  - 🔴 [RDBS Databases](database/_databases.md)
  - 🔴 [Normalization/Denormalization](database/database-normalization-guide.md)
  - [Materialized Views](database/materialized-views.md)

  - Transactions
    - 🔴 [Isolation levels](database/isolation-levels.md)
    - 🔴 📗 [Spring Transactions Guide](database/spring-transactions-guide.md)
    - 🔴 [Spring Transactions Propagation](database/spring-transaction-propagation.md)
    - 🔴 📦 [ACID](database/acid.md)
    - 🔴 📦 [CAP Theorem](database/cap-theorem.md)

  - 🔴 [Locking: Optimistic vs Pessimistic](database/locking-optimist-pessimist.md)
  - [Database Locks](database/lock-database.md)
  - 📗 [Lock Contention Guide](database/lock-contention-guide.md)

- [JDBI](database/jdbi.md)

- ORM
  - 🔴 [JPA Persistence](database/jpa-persistence.md)
    - 🆕 🔴 [Entities](database/entities.md)
    - 🆕 [Entities (Lombok)](database/entities-lombok.md)
  - 🆕 [Spring Data](database/spring-data.md)
  - 🆕 [Hibernate](database/hibernate.md)
    - [Hibernate cache](database/hibernate-cache.md)

- 🔴 [Pagination](database/pagination.md)
  - [Cursor Pagination](database/pagination-cursor.md)

- [DB migration](database/db-migration.md)
  - Liquibase
- [Audit](database/auditing.md)
- 🆕 📗[Spring Data Auditing Guide](database/spring-data-auditing-guide.md)

- Connection Pools
  - 📗 [HikariCP Guide](database/hikaricp-guide.md)
  - [HikariCP Properties](database/hikaricp-properties.md)

- Non Relational Databases
  - 🆕 [NoSQL](database/nosql.md)
    - 📦 [BASE - NoSQL](database/base-principles.md)
    - 🆕 [Cassandra](database/cassandra.md) - real-time streams

- Storage
  - 🆕 📗 [Large File Storage](database/large-file-storage.md)

# Database Scaling

- [DB Scaling](database/database-scaling.md)
- [Db Indexing](database/db-indexing.md)
  - 🆕 📗 [Database Keys Guide](database/database-keys-guide.md)
  - 🆕 📗 [Composite Index Guide](database/composite-index-guide.md)
- 🔴 [ID generation](database/id-generation.md)
- 🔴 [ID generation strategy](database/jpa-id-gen-strategy.md)
- [Db Replication](database/db-replication.md) - solves READ scaling without splitting data
- 🔴 [Caching](database/caching.md) - reduces load on the database for hot reads
  - [Cache Eviction](database/cache-eviction.md)
  - 🆕 [Spring Cache](database/spring-cache.md)
- [Partitioning](database/partitioning.md) - solves query/maintenance pain within one instance
- [Sharding](database/sharding.md) - data split on multiple instances

- 🟠 📗 [Slow Query Guide](database/slow-query-guide.md)

# API

- 🟠 [HTTP protocols](api/http-protocols.md)

- API Styles
  - 🔴 [API](api/_api.md)
  - 🔴 [REST](api/rest.md)
  - [gRPC](api/grpc.md)
    - 🆕 [Protobuf](api/protobuf.md)
  - [GraphQL](api/graphql.md)
  - [Websockets](api/websockets.md)

- API Design
  - 🆕 🔴 [OpenAPI](openapi.md)
  - 🟠 [Validation](api/validation.md)
  - 🟠 [Error Handling](api/error-handling.md)
  - 🔴 [API idempotency](api/api-idempotency.md)
  - 🔴 [Deduplication](api/deduplication.md)
  - [API latency tiers](api/api-latency-tiers.md)
  - 🔴 📗 [API performance Guide](api/api-performance-guide.md)

- External Communication
  - 🆕 [Rest Client](api/spring-rest-client.md)
  - 🆕 [Custom Rest Client](api/custom-rest-client.md)
  - 🆕 [Sring Cloud Open Feign](layer-infrastructure/openfeign.md)

- API integration patterns
  - [Webhooks](api/webhooks.md)

- 🆕 [Web Servers](api/web-servers.md)
  - 🆕 🟠 📗 [Tomcat Guide](api/tomcat-guide.md)
  - 🆕 📗 [Jetty Guide](api/jetty-guide.md)

# Testing

- 🔴 [Unit Tests](test/test-unit.md)
  - [JUnit](test/junit.md)
  - [Mockito](test/mockito.md)
    - [Spy vs Mock](test/spy-vs-mock.md)
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
  - 🆕 📗 [Variable Names Conventions](database/conventions-variable-name.md)
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
  - 🆕 [Spring Cloud Eureka](layer-infrastructure/eureka.md)
- 📦 [Pattern: Internal Secret Management](layer-infrastructure/internal-secret-management.md)
- 📦 [Pattern: Externalized Configuration](layer-infrastructure/externalized-configuration.md)
  - 🆕 [Spring Cloud Config](layer-infrastructure/spring-cloud-config.md)
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
      - 🆕 [Spring Sleuth](layer-infrastructure/sleuth.md)
  - [Alert as Code](layer-infrastructure/alerting.md) Prometheus + Grafana
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

- 🎯 [Spring Framework](api/spring.md)
  - 🔴 📗 [Spring Bean Scope guide](layer-application/spring-bean-scope-guide.md)
  - 🆕 🔴 [Spring AOP](layer-application/spring-aop.md)

# Data Layer

- Data
  - 📦 [Database per Service](microservices/database-per-service.md)

- Processing
  - 🎯 [Batch Processing](processing/batch-processing.md) Spring Batch
    - 🆕 [Spring Batch](processing/spring-batch.md)
  - 🎯 [Data Streaming](processing/data-streaming.md) Kafka Streams
  - 🆕 [Big Data Concepts](processing/big-data-concepts.md)
  - 🎯 [Big Data](processing/big-data.md)
    - Hadoop - data lake (historical data)
    - Spark - analytical engine
    - Hive
  - 🔴 📗 [Serialization Guide](processing/serialization-guide.md)
  - 📗 [Serialization Avro Guide](processing/avro-guide.md)
  - 📗 [Video Streaming](processing/video-streaming.md)

- Messaging
  - 🔴 🎯 [Messaging](processing/_messaging.md)
  - 🆕 [Messaging Protocols](processing/messaging-protocols.md)
  - 🔴 📦 [Pub/Sub](processing/pub-sub.md)
  - 🔴 📦 [Message Queues](processing/message-queues.md)
  - [Backpressure](processing/backpressure.md)
  - 🆕 [Notification types](processing/notification-types.md)

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
  - Stateless
    - 🆕 [SSO (Single Sign On)](security/sso.md)
    - OIDC
    - [Access Token](security/access-token.md)
    - 🆕 [Passkey Auth](security/passkey-authentication.md)
  - Session
    - [Session Management](security/session-management.md)
    - 📗 [Session Management Guide](security/session-context-management-guide.md)

- 🆕 🔴 [Authorization](security/authorization.md)
  - OAuth2

- Encryption
  - [Encryption](security/encryption.md) Symmetric vs Asymmetric
  - [SSL/TLS, HTTPS](security/ssl-tls-https.md)
  - [PKCS Public-Key Cryptography Standards](security/pkcs.md)
  - 🆕 📗 [Password Storage Guide](security/password-storage-guide.md)

- 🔴 [Spring Security](security/spring-security.md)
- [CORS](security/cors.md)

# Microservices

- 🎯 [Distributed Systems](microservices/distributed-system.md)
- [Microservices Patterns](microservices/microservices-patterns.md)

- Decomposition
  - 📦 [Decompose by business capability](microservices/decompose-by-capability.md)
  - 📦 [Decompose by subdomain](microservices/decompose-by-subdomain.md)

# 🔴 Resilience

- 📦 [Load Balancing](resilience/load-balancing.md)
- 📦 [Circuit Breaker](resilience/circuit-breaker.md) stops calling something that's clearly broken - Resilience4j
  - 🆕 [Spring Hystrix](microservices/hystrix.md) - circuit breaker

- Downstream
  - 📦 [Timeout](resilience/timeout.md)
  - 📦 [Retry/Fallback](resilience/retry-fallback.md) handle recovery
- Upstream
  - [Load Shedding vs. Rate Limiting](resilience/load-shed-vs-rate-limit.md)
  - 📦 [Load Shedding](resilience/load-shedding.md) reject requests based on the service's own real-time health
  - 📦 [Rate Limiter / Time Limiter / Bulkhead](resilience/rate-time-limiter.md) cap volume / duration / concurrent capacity
  - [API Throttling](resilience/api-throttling.md)
  - 📦 [Health check API](resilience/healthcheck-api.md)

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

- 🔴 [Spring Cloud](microservices/spring-cloud.md)

# DevOps and CI/CD

- Build Tools
  - 🆕 [Gradle](devops/gradle.md)
  - 🆕 [Maven](devops/maven.md)

- 🔴 [CI/CD](devops/ci-cd.md)
  - [Continous Deployment](devops/continuous-deployment.md)
  - [CI/CD Pipeline Guide](devops/ci-cd-pipeline-guide.md)
  - 🆕 [Github](devops/github.md)

- [Containers](devops/docker.md) Docker
- [Orchestration](devops/kubernetes.md) Kubernetes
  - [Helm](devops/helm.md)
- GitOps
  - [GitOps](devops/gitops.md) ArgoCD
- IaC (Infrastructure as Code)
  - 🆕 [Terraform](devops/terraform.md)

# AI

- [AI agents](ai/ai-agents.md)
- [Cursor Agent](ai/cursor-agent.md)
- 🆕 📗 [MCP Servers Guide](ai/mcp-servers-guide.md)
