A multi-tenant core banking platform runs one deployed application serving multiple distinct financial institutions ("tenants") from shared infrastructure, while keeping each tenant's data isolated from every other tenant's — not as a nice-to-have, but typically as a contractual and regulatory requirement, since one bank's customer data ending up visible to another bank is a severe compliance failure, not just a bug. This note works through a concrete design, pulling together ddd, jpa-persistence, db-migrations, sharding/partitioning, caching, security, and auditing into one worked example, rather than introducing new isolated concepts.

## 1. Why Isolation Is the Central Constraint

Everything else in this design serves one requirement: **a request scoped to Tenant A must never read or write Tenant B's data, under any code path, even a buggy one.** This shapes decisions differently than a typical single-tenant SaaS product would:

- Regulatory and contractual obligations often mandate data segregation between institutions, sometimes down to encryption keys or physical storage — "logically separated by a `WHERE` clause" is not always sufficient.
- A single missed tenant filter in one query is a data breach, not a minor defect — so isolation needs to be enforced at more than one layer (defense in depth), not solely trusted to application code remembering to filter correctly every time.
- Tenants vary enormously in size — a large national bank and a small community credit union have very different data volumes and infrastructure requirements, which pushes toward a tiered isolation strategy rather than one-size-fits-all.

## 2. Three Isolation Models, and Why a Hybrid Is Usually Right

| Model                                                 | Isolation strength                                                                           | Operational cost at scale                                                                                                 | Fits                                                                                   |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Database per tenant                                   | Strongest — physically separate database, can even live in a different region/instance       | High — connection pool and migration fan-out grow linearly with tenant count                                              | Large tenants with strict regulatory/data-residency requirements                       |
| Schema per tenant (one DB instance, many schemas)     | Strong — same logical isolation as separate databases, shared connection/instance overhead   | Moderate — the same migration-fan-out pattern already used for parallel test isolation, now applied to production tenancy | Mid-sized tenants; the common default for most institutions                            |
| Shared schema with a `tenant_id` discriminator column | Weakest by default — isolation depends entirely on every query correctly filtering by tenant | Lowest — one schema, trivial to add a new tenant                                                                          | A large number of very small tenants, where per-tenant schema overhead isn't justified |

A realistic core banking platform tiers these: large/regulated institutions get a dedicated database, mid-sized tenants get a dedicated schema in a shared cluster, and a long tail of small tenants share one schema with strict, enforced row-level isolation. This mirrors the same "not everything needs the same treatment" instinct as choosing sharding strategy per access pattern in sharding.

## 3. Resolving Tenant Context Per Request

Every request needs to know which tenant it belongs to before touching any data — resolved once, early, and propagated the same way a correlation ID is in observability.

```java
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void set(String tenantId) { CURRENT_TENANT.set(tenantId); }
    public static String get() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); } // always clear — thread pools reuse threads, same caution as MDC in observability
}

@Component
public class TenantResolvingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        try {
            // resolved from a verified JWT claim, NEVER from a client-supplied,
            // unverified header alone — a tenant ID a caller could simply set themselves is not isolation
            String tenantId = extractTenantFromVerifiedToken(request);
            TenantContext.set(tenantId);
            MDC.put("tenantId", tenantId); // tenant ID appears in every log line for this request too
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
            MDC.clear();
        }
    }
}
```

The critical detail: the tenant ID must come from something cryptographically verified (a validated JWT claim from the OIDC/OAuth2 flow) — a subdomain or a client-supplied header alone is a routing hint at best, and trusting it as the isolation boundary means any caller can simply claim to be a different tenant.

## 4. Routing to the Correct Data Source (Schema/Database per Tenant)

For the schema- or database-per-tenant tiers, Spring's `AbstractRoutingDataSource` picks the right physical connection per request based on the resolved tenant context.

```java
public class TenantRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.get(); // the same tenant ID resolved, now used to pick a DataSource
    }
}
```

```java
@Bean
public DataSource tenantRoutingDataSource(Map<Object, Object> tenantDataSources, DataSource defaultDataSource) {
    TenantRoutingDataSource routingDataSource = new TenantRoutingDataSource();
    routingDataSource.setTargetDataSources(tenantDataSources); // one entry per tenant's schema/DB
    routingDataSource.setDefaultTargetDataSource(defaultDataSource);
    return routingDataSource;
}
```

Every JPA/JDBI call made during this request transparently uses the correct tenant's schema — application/domain code never needs to know or pass the tenant ID explicitly into a query, because the isolation happens at the connection level, beneath the ORM.

## 5. Enforcing Isolation in the Shared-Schema Tier

For the small-tenant shared-schema tier, isolation depends on every query being scoped by `tenant_id` — Hibernate's native multi-tenancy support (`@TenantId`, Hibernate 6.4+) applies this automatically, rather than relying on every repository method remembering to add a `WHERE tenant_id = ?` clause by hand.

```java
@Entity
public class Account {
    @Id
    private Long id;

    @TenantId
    private String tenantId; // Hibernate automatically adds this to every WHERE clause and every INSERT

    private BigDecimal balance;
    // ... Hibernate transparently scopes every query issued through this session to TenantContext.get()
}
```

**Defense in depth**: even with `@TenantId`, add Postgres row-level security as a second, independent enforcement layer beneath the ORM — so a bug in application code (a raw native query that forgets the filter, an admin tool bypassing the repository layer) still can't leak across tenants.

```sql
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON accounts
    USING (tenant_id = current_setting('app.current_tenant')::text);
```

```java
// Set the session-level variable RLS checks against, at the start of each request/transaction
jdbcTemplate.execute("SET app.current_tenant = '" + TenantContext.get() + "'");
```

This is the same "don't trust a single layer" principle as validating both at the API boundary and in the domain — Hibernate's `@TenantId` and Postgres RLS are two independent checks, so one missing the filter doesn't mean data actually leaks.

## 6. Provisioning a New Tenant: Migrations Per Schema

Onboarding a new tenant in the schema-per-tenant tier means running the full migration history against a brand-new schema — literally the same schema-per-isolation-unit technique already covered for parallel test execution, now used for production tenant provisioning instead of test isolation.

```java
@Service
public class TenantProvisioningService {

    public void provisionTenant(String tenantId) {
        String schemaName = "tenant_" + tenantId;
        jdbcTemplate.execute("CREATE SCHEMA " + schemaName);

        Liquibase liquibase = new Liquibase(
            "db/changelog/changelog-master.xml",
            new ClassLoaderResourceAccessor(),
            new JdbcConnection(dataSourceFor(schemaName).getConnection())
        );
        liquibase.update(new Contexts(), new LabelExpression());

        tenantRegistry.register(tenantId, schemaName); // makes it visible to the routing DataSource
    }
}
```

## 7. The Cross-Tenant Bug Class to Watch For: Cache Keys

The single most common way tenant isolation actually breaks in practice isn't the database layer above — it's a cache that forgets to scope its keys by tenant.

```java
// WRONG: a cache key that doesn't include the tenant ID — Tenant B's request can
// receive Tenant A's cached account balance if they happen to use the same account number
String cacheKey = "account:" + accountNumber;

// CORRECT: tenant ID is part of the cache key, not just the database query
String cacheKey = "tenant:" + TenantContext.get() + ":account:" + accountNumber;
```

The same bug shows up anywhere state is held in memory across requests: an in-process cache, a rate limiter's counter key, a feature-flag evaluation cache — anywhere a key is derived from business data (an account number, a customer ID) without also including the tenant ID, because business identifiers are often only unique _within_ one tenant, not globally.

## 8. Tenant-Scoped Auditing

Every audit record needs a `tenantId` field, and — just as important — access to a tenant's audit trail must itself be restricted to that tenant's own authorized users/auditors, not visible platform-wide by default.

```java
public record AuditEvent(
    String tenantId,   // added to every field
    String actorId,
    String action,
    String targetId,
    Instant timestamp,
    AuditOutcome outcome
) {}
```

## 9. Noisy Neighbor: Per-Tenant Rate Limiting and Resource Isolation

On shared infrastructure, one tenant's traffic spike (a batch job, an unusually busy day) shouldn't degrade service for every other tenant sharing the same instance — apply rate limiting and connection pool sizing per tenant, not just globally.

```java
@Bean
public RateLimiter rateLimiterFor(String tenantId) {
    return RateLimiter.of("tenant-" + tenantId, RateLimiterConfig.custom()
        .limitForPeriod(tenantTierConfig.requestsPerSecond(tenantId)) // larger tenants get a higher limit
        .build());
}
```

For the database-per-tenant tier, this is naturally solved by physical separation; for the shared-schema tier, it needs explicit enforcement — the same reasoning as the Bulkhead pattern, applied per tenant instead of per downstream dependency.

## 10. Control Plane vs. Data Plane, Applied to Tenancy

**Tenant provisioning/configuration is the control plane** (rarely touched, can tolerate being slow — creating a schema and running migrations for a new tenant takes seconds, not milliseconds), while **transaction processing scoped to a resolved tenant is the data plane** (every single request, must be fast, must never depend on the provisioning service being available). A transaction processing request should never call out to the tenant-provisioning service synchronously — it should only ever consult the already-resolved, locally-cached tenant routing information.

## 11. Best Practices

| Practice                                                                                  | Recommendation                                                                                                                                          |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Resolve tenant identity from a verified token claim, never a client-supplied header alone | A caller-supplied "tenant ID" with no verification isn't an isolation boundary.                                                                         |
| Enforce tenant isolation at more than one layer                                           | Hibernate `@TenantId` plus Postgres row-level security — one missing filter shouldn't mean an actual cross-tenant leak.                                 |
| Always include the tenant ID in cache keys, rate-limit keys, and any in-memory state      | The most common real-world isolation break is a cache or counter keyed only by a business ID that's merely unique within one tenant.                    |
| Tier isolation strategy by tenant size/regulatory need, not one model for everyone        | Database-per-tenant for large/regulated institutions, schema-per-tenant for mid-size, shared-schema with enforced RLS for a long tail of small tenants. |
| Add `tenantId` to every audit record, and restrict audit-log access per tenant            | An audit trail spanning all tenants visible to everyone defeats the isolation guarantee just as much as a data leak would.                              |
| Isolate resource consumption per tenant, not just data                                    | Rate limits and connection pool sizing should prevent one tenant's load spike from degrading others sharing the same infrastructure tier.               |
| Never let transaction processing depend synchronously on the tenant-provisioning service  | Provisioning is control-plane (rare, can be slow); transaction processing is data-plane (constant, must be fast).                                       |
