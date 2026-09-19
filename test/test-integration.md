Integration tests verify that a slice of the real system — a controller, a repository, several collaborating beans — works correctly wired to real infrastructure (a real database, a real message broker), rather than a fully mocked-out unit. They sit in the middle of the test pyramid: slower than unit tests, but able to catch wiring bugs (wrong SQL, mapping issues, misconfigured beans) that a fully-mocked unit test structurally cannot see.

## 1. Spring Boot Test Slices

Loading the full Spring context for every test is slow. Test "slices" load only the beans relevant to one layer.

```java
@WebMvcTest(AccountController.class) // loads only the web layer, not the full context
class AccountControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean // replaces the real bean in the Spring context with a Mockito mock
    private AccountService accountService;

    @Test
    void shouldReturn404WhenAccountNotFound() throws Exception {
        when(accountService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/accounts/99"))
            .andExpect(status().isNotFound());
    }
}

@DataJpaTest // loads only JPA-related beans, uses an embedded/test DB, rolls back after each test
class AccountRepositoryTest {

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void shouldFindByOwnerName() {
        accountRepository.save(new Account("Cosmin", BigDecimal.TEN));
        assertThat(accountRepository.findByOwnerName("Cosmin")).isPresent();
    }
}
```

Other slices: `@JsonTest` (serialization only), `@RestClientTest` (outbound HTTP client code). Full `@SpringBootTest` (loads everything) is for true end-to-end tests — reach for a slice whenever you're only testing one layer.

## 2. Testcontainers: Real Infrastructure, Not Mocks

For integration tests, an in-memory DB (H2) can hide real bugs — vendor-specific SQL, real driver behavior, actual constraint enforcement. Testcontainers spins up the real thing in Docker for the test's lifetime.

```java
@SpringBootTest
@Testcontainers
class AccountRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void shouldPersistAndRetrieveAccount() {
        Account saved = accountRepository.save(new Account("Cosmin", BigDecimal.TEN));
        assertThat(accountRepository.findById(saved.getId())).isPresent();
    }
}
```

The container starts once per test class (or reuse it across classes with a shared static container for speed), runs real Postgres, and is torn down automatically. Same approach works for Kafka, Redis, RabbitMQ, etc. — test against what production actually uses.

## 3. Running Integration Tests in Parallel Without Manual Database Cleanup

The default most teams reach for — run integration tests serially, `DELETE FROM` every table (or truncate) in `@AfterEach` — works, but becomes the actual bottleneck as the suite grows: cleanup queries run once per test, and serial execution means total suite time grows linearly with test count. The fix isn't "clean up faster," it's removing the need for cleanup and the need for serialization at the same time, by making tests genuinely unable to see each other's data.

### Technique 1: Schema-per-Test-Class on One Shared Container

The most robust fix: give every test class its own database schema on the _same_ Testcontainers instance — parallel test classes then physically cannot see each other's rows, so there's nothing to clean up, and no serialization is needed to avoid collisions.

```java
@SpringBootTest
@Testcontainers
class AccountRepositoryIntegrationTest {

    // One container, started once, reused across every test class in the suite —
    // avoids paying container startup cost per class
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withReuse(true);

    private static final String SCHEMA = "test_" + UUID.randomUUID().toString().replace("-", "");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", () -> postgres.getJdbcUrl() + "&currentSchema=" + SCHEMA);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.schemas", () -> SCHEMA); // migrations run fresh, against this schema only
        registry.add("spring.jpa.properties.hibernate.default_schema", () -> SCHEMA);
    }

    @Autowired
    private AccountRepository accountRepository;

    @Test
    void shouldPersistAndRetrieveAccount() {
        Account saved = accountRepository.save(new Account("Cosmin", BigDecimal.TEN));
        assertThat(accountRepository.findById(saved.getId())).isPresent();
        // no @AfterEach cleanup needed — this schema is unique to this test class,
        // and no other test class (running concurrently or not) ever touches it
    }
}
```

Each test class gets a fresh schema at startup (via Flyway/Liquibase migrations scoped to that schema), so every test starts from a known-empty state without an explicit cleanup step — the "cleanup" already happened implicitly, by simply never reusing the schema. Old schemas can be dropped opportunistically (a cleanup job, or just left for the container's lifetime, which usually ends with the test run anyway) rather than needing to be torn down per-test.

### Technique 2: Unique Test Data Instead of Infrastructure Isolation

A lighter-weight complement (or alternative, for a smaller suite where schema-per-class is overkill): instead of isolating tests infrastructurally, make each test generate and assert only against uniquely-keyed data, so tests never collide even on a fully shared schema.

```java
@Test
void shouldFindAccountByOwnerName() {
    String uniqueOwnerName = "owner-" + UUID.randomUUID(); // never collides with another concurrently-running test
    accountRepository.save(new Account(uniqueOwnerName, BigDecimal.TEN));

    assertThat(accountRepository.findByOwnerName(uniqueOwnerName)).isPresent();
    // never assert on a full-table COUNT(*) or "list everything" — that WOULD see
    // other tests' concurrently-inserted rows and become flaky under parallel execution
}
```

The rule this technique depends on: never assert on shared/global state (a total row count, "the first row returned," an unfiltered `findAll()`) — only ever assert on data your own test created and can address by its own unique key. This is far cheaper to set up than schema-per-class, but it's weaker: it only protects against _read_ collisions on uniquely-keyed lookups, not against a test that legitimately needs to reason about the whole table's contents (e.g., "there are now exactly 3 active accounts") — that kind of test still needs real isolation (Technique 1) or must stay serialized.

### Technique 3: `@Transactional` Rollback — Fast, But With Real Limits

`@DataJpaTest` already rolls back each test's transaction automatically — no manual `DELETE` needed, and it's effectively free. The limitation that matters for parallelism: this only isolates _within_ one test's own transaction. It doesn't isolate one test's transaction from a _different, concurrently-running_ test's transaction against the same shared schema — two parallel tests both inserting into the same table can still see each other's uncommitted-but-visible rows depending on isolation level, or collide on a unique constraint. `@Transactional` rollback and Technique 1/2 solve different problems and are often combined: rollback avoids leftover data after a _single_ test, while schema-per-class or unique keys prevent _concurrent_ tests from interfering with each other in the first place.

Also worth knowing: `@Transactional` rollback silently breaks for code that manages its own transactions (e.g., a service method annotated `REQUIRES_NEW`, or anything that runs on a separate thread/connection) — those commits are real and won't be rolled back by the test's outer transaction, which is a common source of "cleanup isn't actually happening" confusion.

### Enabling Parallel Execution in JUnit 5

None of the above matters if the test runner itself still executes everything serially — enable parallel execution explicitly:

```properties
# junit-platform.properties (on the classpath, e.g. src/test/resources)
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.mode.classes.default = concurrent
```

For the rare test that genuinely can't run concurrently with others (e.g., one that intentionally asserts on a full-table count, or mutates genuinely global state), mark it explicitly rather than disabling parallelism suite-wide:

```java
@Isolated // this class always runs alone, never concurrently with any other test
class GlobalSequenceCounterTest { ... }

@ResourceLock("accounts_table") // serializes only against other tests declaring the same lock name
class AccountBulkOperationTest { ... }
```

`@Isolated`/`@ResourceLock` let you keep the whole suite parallel by default and carve out narrow, explicit exceptions — rather than the far more common anti-pattern of leaving the entire suite serial "just in case" because one or two tests need it.

## 4. Best Practices

| Practice | Recommendation |
| --- | --- |
| Prefer real infrastructure over mocks for integration tests | Testcontainers over H2/in-memory fakes when testing persistence — catches real SQL/driver bugs. |
| Use test slices, not full context, for layer-specific tests | `@WebMvcTest`/`@DataJpaTest` over `@SpringBootTest` when you're only testing one layer — much faster feedback loop. |
| Isolate integration tests by schema or unique data, not by running serially | Schema-per-class (or uniquely-keyed test data) removes the need for both manual cleanup and serialized execution. |
| Never assert on unfiltered/global state in a parallel-safe test | A full-table `COUNT(*)` or unfiltered `findAll()` will see other concurrently-running tests' rows — assert only on data your test itself created and can address by a unique key. |
| Know the real limits of `@Transactional` rollback | It doesn't isolate concurrent tests from each other, and silently doesn't apply to code managing its own transactions. |
| Reuse containers across test classes where possible | Avoids paying container startup cost per class while still getting real infrastructure behavior. |
