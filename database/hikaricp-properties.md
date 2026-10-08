### 1. Essential Properties

These are the bare minimum required to establish a connection.

```sh
# The JDBC URL for the database
hikari.jdbcUrl=jdbc:mysql://localhost:3306/your_db

# Database credentials
hikari.username=admin
hikari.password=your_password

# The driver class name (optional for modern JDBC drivers)
hikari.driverClassName=com.mysql.cj.jdbc.Driver
```

### 2. Pool Sizing and Timeouts

These properties control the performance and resource usage of the pool.

```sh
# Maximum number of actual connections to the backend
hikari.maximumPoolSize=10

# Minimum number of idle connections HikariCP tries to maintain in the pool
hikari.minimumIdle=5

# Maximum amount of time that a connection is allowed to sit idle in the pool (ms)
hikari.idleTimeout=600000

# Maximum lifetime of a connection in the pool (ms)
hikari.maxLifetime=1800000

# Maximum number of milliseconds that a client will wait for a connection from the pool
hikari.connectionTimeout=30000
```

### 3. Advanced Settings

Useful for debugging and specific database requirements.

```sh
# User-defined name for the connection pool (appears in logging)
hikari.poolName=MyHikariPool

# Query executed to test if a connection is still alive (often not needed for modern drivers)
hikari.connectionTestQuery=SELECT 1

# Whether or not connections obtained from the pool are in auto-commit mode by default
hikari.autoCommit=true
```

### Summary of Key Parameters

| Property            | Default       | Description                                                                               |
| :------------------ | :------------ | :---------------------------------------------------------------------------------------- |
| `maximumPoolSize`   | 10            | Max size the pool can reach, including idle and in-use connections.                       |
| `minimumIdle`       | same as max   | Minimum number of idle connections maintained.                                            |
| `connectionTimeout` | 30000 (30s)   | How long to wait for a connection before throwing an exception.                           |
| `idleTimeout`       | 600000 (10m)  | Max time a connection can sit idle before being retired.                                  |
| `maxLifetime`       | 1800000 (30m) | Max life of a connection. Should be shorter than any database or infrastructure timeouts. |

You can apply these changes to your `src/main/resources/application.properties` or `hibernate.cfg.xml` by clicking the **Apply** button on the code blocks above if they match your file structure.
