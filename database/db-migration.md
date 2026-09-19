Database migration tools version-control schema changes the same way Git version-controls code — every change to a table, column, or index is captured as an explicit, ordered, repeatable script, applied automatically and consistently across every environment (local, CI, staging, production) instead of someone manually running ad-hoc SQL against production and hoping every environment ends up in the same state. Liquibase is one of the two dominant Java-ecosystem tools for this (the other being Flyway); this note uses Liquibase as the primary example.

## 1. Why Not Just Run SQL Manually

Manually applied schema changes drift: one environment gets a column added by hand and someone forgets to apply it to another, or the order two changes were applied in differs between environments in a way that matters. A migration tool guarantees:

- **Ordering**: changes apply in a defined, deterministic sequence — never accidentally out of order.
- **Idempotency**: a migration that's already been applied is never re-applied — the tool tracks exactly which changes have already run, per environment.
- **Auditability**: the full history of schema changes is in version control, reviewable in a PR the same way application code is.
- **Repeatability**: spinning up a fresh environment (a new developer's machine, a CI database, a disaster-recovery restore) reaches the exact same schema state by simply replaying the same ordered migrations — the same principle behind "Git is the source of truth," applied to schema instead of infrastructure.

## 2. Liquibase Core Concepts

| Concept                       | Meaning                                                                                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Changelog                     | The top-level file listing all changesets, in order, that make up the schema's history                                                                |
| Changeset                     | One atomic, uniquely-identified unit of change (e.g., "add a column") — the smallest thing Liquibase tracks and applies independently                 |
| `DATABASECHANGELOG` table     | A table Liquibase creates and maintains in your actual database, recording which changesets have already been applied                                 |
| `DATABASECHANGELOGLOCK` table | A lock table preventing two concurrent Liquibase runs (e.g., two service instances starting simultaneously) from applying migrations at the same time |

```xml
<!-- changelog-master.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog">
    <include file="changelog/001-create-accounts-table.xml"/>
    <include file="changelog/002-add-email-to-accounts.xml"/>
    <include file="changelog/003-create-orders-table.xml"/>
</databaseChangeLog>
```

## 3. Writing a Changeset

Liquibase supports XML, YAML, JSON, or plain SQL for changelogs — XML/YAML are more common because they support Liquibase's built-in rollback generation and cross-database abstraction; plain SQL is often preferred for its directness and familiarity, at the cost of writing rollback logic manually.

```xml
<!-- 001-create-accounts-table.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog">
    <changeSet id="001-create-accounts-table" author="cosmin">
        <createTable tableName="accounts">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="owner_name" type="VARCHAR(100)">
                <constraints nullable="false"/>
            </column>
            <column name="balance" type="NUMERIC(19,2)" defaultValueNumeric="0">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

```yaml
# Equivalent in YAML — 002-add-email-to-accounts.yaml
databaseChangeLog:
  - changeSet:
      id: 002-add-email-to-accounts
      author: cosmin
      changes:
        - addColumn:
            tableName: accounts
            columns:
              - column:
                  name: email
                  type: VARCHAR(255)
```

```sql
-- Formatted SQL changelog — 003-create-orders-table.sql
-- liquibase formatted sql

-- changeset cosmin:003-create-orders-table
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES accounts(id),
    total NUMERIC(19,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Each changeset's `id` + `author` (+ changelog file path) together form its unique identity — Liquibase uses this combination to know whether a given changeset has already been applied, so **never edit an already-applied changeset's content** after it's run anywhere (including just locally) — Liquibase computes and stores a checksum of each changeset, and by default refuses to proceed if a previously-applied changeset's content has changed underneath it, since that would mean different environments ran genuinely different SQL under the same recorded identity.

## 4. Running Migrations — Spring Boot Integration

Spring Boot auto-runs Liquibase migrations on application startup if the dependency is on the classpath — schema changes ship as part of the same deployable artifact as the application code that depends on them.

```yaml
# application.yml
spring:
  liquibase:
    change-log: classpath:db/changelog/changelog-master.xml
```

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

This runs automatically before the application context finishes starting — by the time your `@Repository`/`@Service` beans are ready to serve traffic, the schema they expect already exists. For more controlled rollout (e.g., running migrations as a separate CI/CD step before deploying new application instances, rather than coupling it to app startup), disable auto-run and invoke the Liquibase CLI/Maven plugin explicitly as its own pipeline stage instead.

## 5. Rollback

Liquibase can auto-generate a rollback for many structural changes (adding a column, creating a table) — for anything it can't infer safely, you write the rollback explicitly.

```xml
<changeSet id="002-add-email-to-accounts" author="cosmin">
    <addColumn tableName="accounts">
        <column name="email" type="VARCHAR(255)"/>
    </addColumn>
    <rollback>
        <dropColumn tableName="accounts" columnName="email"/>
    </rollback>
</changeSet>
```

```bash
liquibase rollback-count 1   # rolls back the most recently applied changeset
liquibase rollback-to-tag v1.4.0
```

Rollback is genuinely safe only for purely structural, reversible changes — a rollback that would **drop a column containing real data already written by the new code** is not actually safe to run against production without a separate, deliberate data-preservation plan; auto-generated rollback should never be trusted blindly for anything that could destroy live data, only reasoned about explicitly per change.

## 6. Liquibase vs. Flyway

| Aspect               | Liquibase                                                                                  | Flyway                                                                             |
| -------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Changelog format     | XML, YAML, JSON, or SQL                                                                    | Primarily SQL (with a Java-based migration escape hatch for complex logic)         |
| Rollback support     | Built-in, auto-generated for many change types                                             | Rollback is a paid/Enterprise feature in Flyway; community edition is forward-only |
| Database abstraction | Changelogs can be database-agnostic (e.g., `createTable` generates correct SQL per vendor) | Migrations are typically vendor-specific raw SQL                                   |
| Learning curve       | More concepts (changelog/changeset/context/label), more powerful                           | Simpler mental model — "just SQL files in order"                                   |

Neither is objectively "better" — Flyway's simplicity (plain, ordered SQL files) appeals to teams that want minimal abstraction and are comfortable writing vendor-specific SQL directly; Liquibase's structured changelogs and free built-in rollback appeal to teams wanting more tooling around the migration process itself, or genuine multi-database-vendor support.

## 7. Zero-Downtime Migrations: The Expand/Contract Pattern

In a system with rolling deployments, old and new versions of the application run **simultaneously** for a period during rollout — a migration that both adds a new column and immediately requires it (`NOT NULL`, no default) will break the still-running old application instances the moment it applies, because they don't know about the new column yet.

The safe pattern splits a breaking-looking change into multiple, individually-safe deployments:

```
Expand:    Add the new column as NULLABLE (or with a default). Deploy this migration alone.
           Old app instances ignore the new column; they keep working unmodified.

Migrate:   Deploy application code that writes to BOTH the old and new column
           (or backfills the new column from existing data). Both old and new
           app versions can coexist during this rollout, because nothing required
           has changed yet from the old version's perspective.

Contract:  Once every instance is confirmed running the new code, deploy a follow-up
           migration that finally makes the new column NOT NULL, drops the old
           column, etc. — the destructive/tightening step, run only after nothing
           depends on the old shape anymore.
```

```sql
-- Expand (safe to deploy immediately, old code unaffected)
ALTER TABLE accounts ADD COLUMN email VARCHAR(255);

-- ... deploy app code that writes email on every account creation/update ...
-- ... optionally backfill existing rows ...

-- Contract (only once every running instance writes email correctly)
ALTER TABLE accounts ALTER COLUMN email SET NOT NULL;
```

Renaming a column is the same problem in disguise — never rename directly; add the new column, dual-write during the migrate phase, then drop the old column in a later contract step. This pattern is exactly why "only add fields, never remove/repurpose them without a version" discipline exists at the schema level too — a migration is a contract with every currently-running instance of your application, not just the next deploy.

## 8. Best Practices

| Practice                                                                   | Recommendation                                                                                                                             |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Never edit an already-applied changeset                                    | Add a new changeset instead — editing history that's already run elsewhere causes checksum mismatches and environment drift.               |
| Keep changesets small and single-purpose                                   | One structural change per changeset — easier to review, easier to reason about rollback for.                                               |
| Use the expand/contract pattern for breaking schema changes                | A rolling deployment runs old and new code simultaneously; a same-step add-and-require column change breaks the old instances mid-rollout. |
| Never trust auto-generated rollback for anything that could drop live data | Reason explicitly about data-destructive changes; prefer expand/contract over a rollback that would need to resurrect deleted data.        |
| Run migrations as an explicit, auditable step in CI/CD                     | Whether coupled to app startup or a separate pipeline stage, treat schema changes with the same review rigor as application code changes.  |
| Keep the changelog under version control alongside the application         | The schema's history should be reviewable in the same PR process as the code that depends on it.                                           |
| Pick Liquibase or Flyway based on actual team needs, not habit             | Liquibase for built-in rollback/multi-vendor abstraction; Flyway for a simpler, plain-SQL-first workflow.                                  |
