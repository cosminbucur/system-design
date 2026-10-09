To bound a cache in Java using **Caffeine**, you configure the **`Caffeine`** builder with size or weight constraints and time-based expiration policies.

**Size-Based Bounding**
Limit the cache by the number of entries or the weighted size of values:

- **`maximumSize(long)`**: Sets the maximum number of entries the cache will hold.
- **`maximumWeight(long)`** and **`weigher(Weigher)`**: Sets a maximum weight limit, requiring a custom weigher to calculate the weight of each key-value pair.

**Time-Based Bounding**
Entries can be automatically removed after a specified duration:

- **`expireAfterWrite(long, TimeUnit)`**: Expires entries a fixed time after they are written.
- **`expireAfterAccess(long, TimeUnit)`**: Expires entries a fixed time after their last read or write access.
- **`expireAfter(Expiry)`**: Allows for a custom expiry policy implementation.

**Programmatic Configuration Example**

```java
Cache<String, DataObject> cache = Caffeine.newBuilder()
    .maximumSize(10_000)           // Bound by entry count
    .maximumWeight(100_000)        // Bound by weight (requires weigher)
    .weigher((key, value) -> value.size()) // Define weight logic
    .expireAfterWrite(10, TimeUnit.MINUTES)// Expire 10 mins after write
    .expireAfterAccess(5, TimeUnit.MINUTES)// Expire 5 mins after last access
    .build();
```

**Spring Boot Configuration**
In **Spring Boot**, you can bind these limits via properties or Java configuration:

- **Properties**: `spring.cache.caffeine.spec=maximumSize=500,expireAfterAccess=10m`
- **Java Config**: Create a `Caffeine` bean using `Caffeine.newBuilder().maximumSize(500).expireAfterAccess(10, TimeUnit.MINUTES)` and inject it into a `CaffeineCacheManager`.
