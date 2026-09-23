# architecture

- hexagonal architecture - adapters and ports to separate core from infrastructure (db, api)
- when domain is complex (banking, insurance, pricing)

- event sourcing

# message queues

- rabbitmq
- kafka

# microservices

- API gateway
- API thorttling vs rate limiting
  - token bucket

- deduplication (prevent execution)
- idempotency (safe effects)

# security

- sticky sessions
- pkce

# fundamentals

- switch
- Object.requireNonNull() - lighter than Optional
- records - immutable data classes, dtos, return multiple values from methods
- default methods (interfaces) - backward compatibility, not forcing implementation of a new method

# logs

- stdout vs stderr
- structured logging
