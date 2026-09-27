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

- API thorttling vs rate limiting
  - token bucket

# security

- sticky sessions
- PCKE (pixie)
- passkeys (passwordless authentication)
  - Storage: Website stores the public key in its database.
  - Unlocking: User device holds the private key, unlocked locally via biometrics.

# fundamentals

- switch
- Object.requireNonNull() - lighter than Optional
- records - immutable data classes, dtos, return multiple values from methods
- default methods (interfaces) - backward compatibility, not forcing implementation of a new method

# logs

- stdout vs stderr

# system design

https://www.systemdesignhandbook.com/guides/design-a-stock-exchange-system/#non-functional-requirements-that-drive-the-design

https://algomaster.io/learn/system-design-interviews/design-notification-service
