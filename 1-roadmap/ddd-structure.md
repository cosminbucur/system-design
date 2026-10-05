```
# Microservie Architecture DDD + Hexagonal

src/main/java/com/company/bookservice/
├── BookServiceApplication.java
│
├── domain/                                 # Pure business logic (No Spring/JPA annotations)
│   ├── model/
│   │   ├── book/                           # Aggregate Root
│   │   │   ├── Book.java
│   │   │   ├── BookId.java                 # Value Object (Record or Wrapper)
│   │   │   ├── ISBN.java                   # Value Object
│   │   │   ├── Price.java                  # Value Object
│   │   │   └── BookCreatedEvent.java       # Domain Event
│   │   └── author/                         # Related Entity or separate Aggregate if complex
│   │       ├── Author.java
│   │       └── AuthorId.java
│   ├── repository/                         # Domain Interfaces (Port)
│   │   ├── BookRepository.java
│   │   └── AuthorRepository.java
│   ├── service/                            # Domain services (logic spanning multiple aggregates)
│   │   └── BookPricingDomainService.java
│   └── exception/
│       └── BookDomainException.java
│
├── application/                            # Use cases & application orchestration
│   ├── usecase/
│   │   ├── CreateBookUseCase.java
│   │   └── GetBookByIdUseCase.java
│   ├── dto/                                # Data Transfer Objects
│   │   ├── CreateBookCommand.java
│   │   └── BookResponseDto.java
│   └── internal/
│       └── BookApplicationService.java
│
├── infrastructure/                         # Technical implementations (Adapters & Frameworks)
│   ├── persistence/
│   │   ├── SpringDataBookRepository.java   # Spring Data JPA interface
│   │   ├── BookRepositoryAdapter.java      # Implements domain.repository.BookRepository
│   │   └── entity/
│   │       └── BookJpaEntity.java          # Database entity model
│   ├── observability/                      # Telemetry & monitoring configs
│   │    ├── MetricsConfig.java             # Micrometer / Prometheus custom meters
│   │    ├── TracingConfig.java             # OpenTelemetry or distributed tracing setup
│   │    └── LoggingMdcFilter.java          # Servlet filter to inject Correlation IDs / MDC
│   ├── messaging/
│   │   ├── kafka/
│   │   │   ├── KafkaEventPublisher.java    # Produces domain events
│   │   │   └── KafkaOrderConsumer.java     # Consumes incoming events from other services
│   ├── outbox/                             # Transactional Outbox Pattern
│   │   ├── OutboxMessage.java
│   │   └── OutboxScheduler.java
│   ├── cache/                              # Redis / Caffeine caching setup
│   │   └── CacheConfig.java
│   ├── webhook/                            # Outbound webhook dispatchers (HTTP senders)
│   │   └── PartnerWebhookPublisher.java
│   ├── external/                           # Third-party APIs (e.g., ISBN lookup service)
│   │   └── OpenLibraryRestClient.java
│   ├── resilience/                         # Resilience4j configurations
│   │   └── BookClientCircuitBreaker.java
│   ├── batch/                              # Spring Batch jobs, readers, writers, & configs
│   │   ├── BookImportJobConfig.java
│   │   ├── BookItemReader.java
│   │   └── BookItemWriter.java
│   ├── bigdata/
│   │   ├── spark/
│   │   │   └── SparkJobClient.java
│   │   └── hadoop/
│   │       └── HdfsStorageAdapter.java
│   └── config/                             # Low-level technical beans
│       ├── JsonMapperConfig.java           # Jackson / ObjectMapper setup
│       ├── RestClientConfig.java           # Spring RestClient beans
│       └── OpenApiConfig.java              # Swagger / Springdoc OpenAPI
│
└── presentation/                           # Entry points
    ├── rest/
    │   └── BookController.java             # Spring MVC REST Controller
    ├── security/                           # Spring Security & Web security concerns
    │   ├── SecurityConfig.java             # Spring Security filter chain
    │   └── CorsConfig.java                 # CORS configuration
    ├── exception/                          # Translates exceptions to HTTP responses
    │   └── GlobalExceptionHandler.java     # @ControllerAdvice
    ├── grpc/
    │   └── BookGrpcService.java            # Inbound gRPC service implementation
    ├── graphql/                            # GraphQL API layer
    │   ├── BookGraphqlController.java      # Or @QueryMapping / @MutationMapping beans
    │   └── scalar/                         # Custom scalar definitions (e.g., ISBN, DateTime)
    │       └── IsbnScalar.java
    ├── webhook/                            # Inbound webhook controllers (HTTP receivers)
    │   └── SupplierWebhookController.java
    └── websocket/
        ├── BookWebSocketController.java    # Real-time WebSocket handler
        └── WebSocketConfig.java            # STOMP / WebSocket configuration

# Tests

src/test/java/com/company/bookservice/
├── architecture/
│   └── ArchUnitTests.java                  # Enforces DDD package dependency rules
├── integration/
│   └── BookRepositoryIntegrationTest.java  # Testcontainers DB tests
└── unit/
    └── BookDomainTest.java                 # Pure unit tests for aggregate logic

# Resources

src/main/resources/
├── application.yml
└── db/
    └── migration/                          # Flyway / Liquibase versioned SQL scripts
    └── V1__create_book_table.sql

# Infrastructure

book-service/                               # Repository Root
├── .github/                                # CI/CD pipelines (GitHub Actions)
│   └── workflows/
│       └── ci-cd.yml                       # Runs static analysis, tests, & builds container
├── config/                                 # Static analysis & code style configs
│   ├── checkstyle/
│   │   └── checkstyle.xml                  # Code style rules
│   └── sonarqube/
│       └── sonar-project.properties        # SonarQube quality gate config
├── terraform/                              # IaC (Infrastructure as Code)
│   ├── main.tf                             # Provisions AWS/GCP resources (PostgreSQL, Kafka, EKS)
│   ├── variables.tf
│   └── outputs.tf
├── deploy/                                 # Kubernetes & GitOps manifests
│   ├── helm/                               # Helm chart for the book service
│   │   └── book-service/
│   ├── argocd/                             # ArgoCD Application definition manifests
│   │   └── book-service-app.yaml
│   └── monitoring/                         # Alerting & Observability configs
│       ├── prometheus-rules.yaml           # Prometheus alert rules (e.g., High Error Rate, CPU)
│       └── grafana-dashboard.json          # Custom Grafana dashboard for the microservice
├── e2e/                                    # Playwright E2E Tests (Node.js/TypeScript)
│   ├── tests/                              # Playwright test specs (.spec.ts)
│   ├── playwright.config.ts                # Playwright configuration
│   └── package.json                        # Playwright dependencies
├── load-tests/                             # Performance & Load Tests (e.g., k6 scripts)
│   └── book-catalog-load-test.js
├── profiling/                              # Profiling tools & JFR configs
│   ├── jfr-profile.jfc                     # Java Flight Recorder profile template
│   └── generate-flamegraph.sh              # Script to pull and parse flame graphs
├── Dockerfile                              # Multi-stage Docker build for the Spring Boot app
├── docker-compose.yml                      # Local dev setup (App + PostgreSQL + Redis + Kafka)
├── build.gradle.kts (or pom.xml)

# External Analytics

├── spark-recommendation-job/               # Standalone Spark Scala/Python project for book recommendations
├── hadoop-etl-pipelines/                   # Hadoop map-reduce or Hive scripts
```
