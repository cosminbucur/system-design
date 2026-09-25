Auditing means recording who did what, to what, when, and with what result — durably enough to reconstruct exactly what happened after the fact, for compliance, security investigation, or dispute resolution. This is distinct from the operational logging: observability logs exist to help _you_ debug the system and are often sampled, rotated, or discarded after weeks; an audit trail exists to answer a question a regulator, auditor, or customer asks _later_ — sometimes years later — and must be complete, tamper-evident, and retained on its own schedule regardless of what happens to ordinary application logs. This matters especially in a core banking context, where "who approved this transaction limit change, and when" is a real regulatory question, not a hypothetical.

## 1. What an Audit Record Actually Needs to Capture

A minimal audit record answers five questions — missing any one of them makes the record far less useful during an actual investigation.

| Field     | Answers                                                                           |
| --------- | --------------------------------------------------------------------------------- |
| Actor     | Who performed the action (user ID, service account, or system process)            |
| Action    | What was done (`ACCOUNT_LIMIT_CHANGED`, `TRANSFER_APPROVED`, `USER_ROLE_GRANTED`) |
| Target    | The specific entity/resource affected (account ID, transaction ID)                |
| Timestamp | When it happened, in UTC, with enough precision to establish ordering             |
| Outcome   | Succeeded, failed, or was denied — and why, for a denial                          |

```java
public record AuditEvent(
    String actorId,
    String actorType,       // USER, SERVICE_ACCOUNT, SYSTEM
    String action,
    String targetType,
    String targetId,
    Instant timestamp,
    AuditOutcome outcome,
    Map<String, Object> beforeState,
    Map<String, Object> afterState,
    String correlationId    // ties this event back to the originating request
) {}
```

Capturing both `beforeState` and `afterState` (not just "the limit was changed") is what makes an audit record actually useful during a dispute — "the daily withdrawal limit changed from $500 to $5,000 at 14:32 UTC, approved by user X" is answerable; "the limit was changed" is not.

## 2. Application-Level Auditing: Spring Data JPA Auditing

For the common case — knowing who created/modified a row and when — Spring Data JPA's built-in auditing support requires almost no boilerplate.

```java
@EnableJpaAuditing
@Configuration
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName); // pulls the current authenticated principal
    }
}
```

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Account {
    @Id
    private Long id;

    @CreatedBy
    private String createdBy;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedBy
    private String lastModifiedBy;

    @LastModifiedDate
    private Instant lastModifiedAt;

    // ... business fields
}
```

This is enough for "who touched this row last" but not for a full history — it overwrites `lastModifiedBy`/`lastModifiedAt` on every update, so it can't answer "who changed the limit _before_ the most recent change." For that, you need either a full revision history or an explicit, append-only audit event log.

## 3. Full Revision History: Hibernate Envers

Envers automatically maintains a complete version history of every change to an annotated entity — not just the latest state, but every historical revision, queryable independently.

```java
@Entity
@Audited // every insert/update/delete on this entity is versioned automatically
public class AccountLimit {
    @Id
    private Long id;

    private Long accountId;
    private BigDecimal dailyWithdrawalLimit;
    private BigDecimal dailyTransferLimit;
}
```

```java
AuditReader reader = AuditReaderFactory.get(entityManager);

// Full history of every revision to this specific entity
List<Number> revisionNumbers = reader.getRevisions(AccountLimit.class, limitId);

for (Number rev : revisionNumbers) {
    AccountLimit atThatRevision = reader.find(AccountLimit.class, limitId, rev);
    DefaultRevisionEntity revisionInfo = reader.findRevision(DefaultRevisionEntity.class, rev);
    System.out.println("At revision " + rev + " (" + revisionInfo.getRevisionDate() + "): "
        + atThatRevision.getDailyWithdrawalLimit());
}
```

```java
// Custom revision entity — attach WHO made each revision, not just when
@Entity
@RevisionEntity(AuditorRevisionListener.class)
public class CustomRevisionEntity extends DefaultRevisionEntity {
    private String modifiedBy;
    // getters/setters
}

public class AuditorRevisionListener implements RevisionListener {
    @Override
    public void newRevision(Object revisionEntity) {
        ((CustomRevisionEntity) revisionEntity).setModifiedBy(
            SecurityContextHolder.getContext().getAuthentication().getName());
    }
}
```

Envers is the right tool when you need to reconstruct an entity's _entire_ history (every intermediate limit value, not just the current one and who last touched it) and you're already using JPA/Hibernate. It stores history in a parallel `_AUD` table per audited entity, generated and migrated the same way as your regular schema.

## 4. Domain-Event-Based Audit Trail

For business-meaningful actions specifically (not every field update, but "a transfer was approved," "a user's role was changed"), the more robust and intentional approach is to publish an explicit domain event for the action itself and have a dedicated consumer persist it to an append-only audit store — the same domain event mechanism from DDD, applied specifically to compliance/audit needs rather than just cross-aggregate consistency.

```java
public record TransferApprovedEvent(
    String approverId,
    String transferId,
    BigDecimal amount,
    Instant occurredAt,
    String correlationId
) {}

@Service
public class TransferApprovalService {

    @Transactional
    public void approve(String transferId, String approverId) {
        Transfer transfer = transferRepository.findById(transferId).orElseThrow();
        transfer.approve(approverId); // domain logic enforces who/whether this is even allowed

        eventPublisher.publishEvent(new TransferApprovedEvent(
            approverId, transferId, transfer.getAmount(), Instant.now(), MDC.get("correlationId")));
    }
}

@Component
public class AuditEventListener {

    @EventListener
    public void onTransferApproved(TransferApprovedEvent event) {
        auditLogRepository.save(new AuditLogEntry(
            event.approverId(), "TRANSFER_APPROVED", event.transferId(),
            event.occurredAt(), AuditOutcome.SUCCESS, toJson(event), event.correlationId()));
        // written to a dedicated, append-only audit table/store
    }
}
```

This approach scales naturally to a message-queue-based architecture too: instead of an in-process `@EventListener`, publish the audit event to a durable topic and let a separate audit service consume and persist it — decoupling "recording the audit trail" from "processing the transfer," so a slow or temporarily-unavailable audit store never blocks the actual business operation (the same control-plane/data-plane reasoning from microservices-patterns applies here: the audit trail is off the hot path of the transaction itself).

## 5. Making the Audit Log Append-Only and Tamper-Evident

An audit trail that the same application/database role can freely `UPDATE`/`DELETE` isn't trustworthy as an audit trail — anyone with write access to the operational database could rewrite history. A few concrete measures, in increasing order of rigor:

```sql
-- A dedicated, separate table (or schema/database) for audit records —
-- never the same table the audited entity itself lives in
CREATE TABLE audit_log (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    actor_id VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    target_type VARCHAR(100) NOT NULL,
    target_id VARCHAR(100) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    outcome VARCHAR(20) NOT NULL,
    details JSONB,
    correlation_id VARCHAR(100),
    prev_hash VARCHAR(64),   -- hash of the previous record — see the tamper-evidence note below
    record_hash VARCHAR(64) NOT NULL
);

-- The application's normal database role should have INSERT only on this table —
-- no UPDATE, no DELETE — enforced at the database level, not just by application code
REVOKE UPDATE, DELETE ON audit_log FROM app_role;
GRANT INSERT, SELECT ON audit_log TO app_role;
```

```java
public class AuditLogEntry {
    // ...
    public String computeHash(String previousRecordHash) {
        String content = actorId + action + targetId + occurredAt + previousRecordHash;
        return DigestUtils.sha256Hex(content); // each record's hash depends on the previous one
    }
}
```

Chaining each record's hash to the previous record's hash (a lightweight version of the same idea behind a blockchain's tamper-evidence, without needing any of a blockchain's other machinery) means altering or deleting a historical record breaks the hash chain for every subsequent record — detectable by recomputing and comparing hashes, rather than preventable outright. For genuinely regulated environments, pair this with database-level protections (a role with `INSERT`-only privileges, as above) or write-once storage (e.g., an object store bucket configured for retention/legal-hold, or a dedicated audit-logging service) rather than relying on the hash chain alone as the only safeguard.

## 6. Auditing Cross-Cutting Concerns With AOP

Rather than manually publishing an audit event inside every service method that needs one, a custom annotation plus a Spring AOP aspect centralizes the concern — similar in spirit to how clean code treats error handling as a distinct, separable concern from business logic.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
    String action();
}

@Aspect
@Component
public class AuditAspect {

    @Around("@annotation(auditable)")
    public Object audit(ProceedingJoinPoint joinPoint, Auditable auditable) throws Throwable {
        String actorId = currentActor();
        try {
            Object result = joinPoint.proceed();
            auditService.record(actorId, auditable.action(), joinPoint.getArgs(), AuditOutcome.SUCCESS);
            return result;
        } catch (Exception ex) {
            auditService.record(actorId, auditable.action(), joinPoint.getArgs(), AuditOutcome.FAILURE);
            throw ex; // audit failures too — a denied/failed action is often the MORE important record
        }
    }
}
```

```java
@Auditable(action = "ACCOUNT_LIMIT_CHANGED")
public void updateLimit(String accountId, BigDecimal newLimit) { ... }
```

Auditing failed/denied attempts, not just successful ones, matters more than it might first seem: "user X attempted to raise the transfer limit to $50,000 and was denied due to insufficient authorization" is frequently the single most important audit record in a security investigation — a successful action is often less interesting than a rejected one.

## 7. Correlating Audit Records With the Rest of the System

An audit record is far more useful when it can be tied back to the full request context — the correlation/trace ID from observability that ties it to application logs, and (for a security-relevant action) the authentication context from security.

```java
auditService.record(new AuditEvent(
    actorId, "USER", "TRANSFER_APPROVED", "TRANSFER", transferId,
    Instant.now(), AuditOutcome.SUCCESS, beforeState, afterState,
    MDC.get("correlationId") // the SAME correlation ID that appears in the operational logs for this request
));
```

During an investigation, this lets you go from "the audit log shows transfer X was approved by user Y at 14:32" directly to "here are the exact application logs, including the IP address and request details, for that same request" — without the correlation ID, those two systems are unlinked, and reconstructing the full picture becomes far harder.

## 8. Retention and Compliance Considerations

Audit retention requirements are typically driven by regulation, not engineering preference — financial services commonly require multi-year retention (e.g., several years under various banking/AML regulations), independent of how long operational logs are kept (which observability data in observability is often rotated within weeks). This has direct consequences for how audit storage is designed:

- **Store audit data separately from operational data**, on its own retention policy — don't let a generic "delete logs older than 30 days" job accidentally sweep up audit records.
- **GDPR's "right to be forgotten" creates real tension with immutable audit logs** — the common resolution is to pseudonymize the actor/subject reference in the audit record (replace a raw user ID with a reversible token held separately, or a one-way hash) rather than deleting the audit record itself, preserving the audit trail's integrity while still being able to sever the link to a specific person's identity on request.
- **Access to the audit log itself needs its own authorization** — not everyone who can read application data should be able to read (or, especially, write to) the audit trail; treat "who can view audit records" as its own permission, since the audit log is frequently what an internal investigation needs to be protected _from_ insiders, not just external attackers.

## 9. Best Practices

| Practice                                                                                        | Recommendation                                                                                                                                                              |
| ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Capture before/after state, not just "something changed"                                        | "Limit changed from $500 to $5,000" is investigable; "limit changed" is not.                                                                                                |
| Store audit records in a separate, append-only table/store                                      | The application's normal DB role should have `INSERT`-only access — no `UPDATE`/`DELETE` — enforced at the database level.                                                  |
| Audit failed and denied actions, not just successes                                             | A rejected privilege escalation attempt is frequently the most important record in a security investigation.                                                                |
| Propagate the same correlation ID into both audit and operational logs                          | Lets an investigation move seamlessly between "what the audit trail shows" and "what actually happened in the request".                                                     |
| Separate audit retention policy from operational log retention                                  | Regulatory retention requirements (often years) are unrelated to how long you keep debug/observability logs.                                                                |
| Use domain events for business-meaningful audit trails, not field-level auditing for everything | Envers/JPA auditing suits "who touched this row" — an explicit domain event better captures "what business action occurred and why it mattered."                            |
| Restrict who can read the audit log itself                                                      | The audit trail is often meant to protect against insider risk — treat read access to it as its own permission, not something everyone with data access automatically gets. |
| Resolve GDPR erasure requests by pseudonymizing, not deleting, audit records                    | Preserves the audit trail's integrity while still honoring a legitimate erasure request against the specific identity.                                                      |
