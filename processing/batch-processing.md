Batch processing handles a large volume of data as a scheduled, bounded job — reading, transforming, and writing many records as one unit of work — rather than reacting to each record individually as it arrives. It's the right tool when data doesn't need to be processed the instant it appears (end-of-day interest accrual, a nightly reconciliation report, a bulk data migration) and the wrong tool when it does (a single customer-facing request needs an answer now, not at the next scheduled run).

## 1. Batch vs. Stream Processing

| | Batch | Stream (message queues, reactive pipelines) |
| --- | --- | --- |
| Trigger | Scheduled, or manually invoked | Continuous, reacts to each event as it arrives |
| Latency | Minutes to hours (until the next run) | Near real-time |
| Data volume per unit of work | Large, bounded ("all of yesterday's transactions") | Small, one event at a time |
| Failure handling | Retry/resume the whole job or a chunk of it | Retry/DLQ per individual message |
| Typical use case | Interest accrual, billing runs, data warehouse loads, bulk migrations | Order placed notifications, real-time fraud scoring, live dashboards |

The two aren't mutually exclusive within one system — a core banking platform commonly runs both: real-time transaction posting via events, and a nightly batch job for interest accrual and statement generation that operates on the whole day's accumulated data at once.

## 2. The Chunk-Oriented Model: Read → Process → Write

The standard shape of a batch job (and what Spring Batch is built around) processes data in fixed-size chunks rather than one record at a time or the entire dataset at once — this bounds memory usage and gives natural checkpoints to resume from after a failure.

```java
@Bean
public Step accrueInterestStep(JobRepository jobRepository, PlatformTransactionManager txManager,
                                 ItemReader<Account> reader, ItemProcessor<Account, InterestAccrual> processor,
                                 ItemWriter<InterestAccrual> writer) {
    return new StepBuilder("accrueInterestStep", jobRepository)
        .<Account, InterestAccrual>chunk(500, txManager) // read/process/write 500 records per transaction
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .faultTolerant()
        .skipLimit(10)
        .skip(InvalidAccountException.class)
        .build();
}
```

```java
@Bean
public ItemReader<Account> accountReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Account>()
        .dataSource(dataSource)
        .sql("SELECT * FROM accounts WHERE status = 'ACTIVE'")
        .rowMapper(new AccountRowMapper())
        .build();
}

public class InterestAccrualProcessor implements ItemProcessor<Account, InterestAccrual> {
    @Override
    public InterestAccrual process(Account account) {
        BigDecimal interest = calculateInterest(account);
        return new InterestAccrual(account.getId(), interest);
    }
}
```

Each chunk commits as its own transaction — if the job fails partway through, only the current in-flight chunk is rolled back, and a restarted job can resume from the last successfully committed chunk rather than starting over from the very first record.

## 3. Why Chunk Size Matters

| Chunk size | Effect |
| --- | --- |
| Too small | More transaction overhead (commit cost paid far more often), slower overall throughput |
| Too large | More memory held in-flight per chunk, and a failure loses/re-does more work when the job resumes |
| Right-sized | Balances transaction overhead against memory footprint and how much work a restart has to redo |

There's no universal correct number — it depends on record size, available memory, and how expensive a re-run of a partial chunk actually is; tune it against real data volumes rather than guessing a value.

## 4. Restart, Skip, and Retry Within a Job

A long-running batch job over a large dataset will eventually hit a bad record or a transient failure, and the framework-level handling of that moment is what separates a robust batch pipeline from one that requires manual intervention on every failure.

| Mechanism | Behavior |
| --- | --- |
| Skip | A record that fails processing (a malformed row, a business-rule violation) is logged and skipped, up to a configured limit, rather than failing the entire job |
| Retry | A record that fails due to a likely-transient error (a brief DB connectivity blip) is retried a bounded number of times before being treated as a skip or a hard failure |
| Restart | If the job itself is interrupted (crash, deploy), it can resume from the last completed chunk rather than reprocessing the entire dataset from scratch |

```java
.faultTolerant()
.skipLimit(10)
.skip(InvalidAccountException.class)  // skip up to 10 bad records, don't fail the whole run
.retryLimit(3)
.retry(TransientDataAccessException.class) // retry a likely-transient DB error before giving up on that record
```

Getting skip/retry limits right matters for a job touching financial data specifically: too permissive a skip limit can silently drop legitimate records that needed attention; too strict, and one bad record blocks an entire nightly run that thousands of unrelated, valid records were waiting on.

## 5. Idempotent Batch Runs

A batch job that's restarted, rerun manually, or accidentally triggered twice needs to produce the same end state either way — the same idempotency concern that applies to any retried operation, just at the scale of an entire job instead of a single API call.

```java
// WRONG: re-running the job a second time double-accrues interest
public void accrueInterest(Account account) {
    account.setBalance(account.getBalance().add(calculateInterest(account)));
}

// RIGHT: record which accrual date/run already happened, and skip it on a duplicate run
public void accrueInterest(Account account, LocalDate accrualDate) {
    if (accrualRepository.existsByAccountIdAndDate(account.getId(), accrualDate)) {
        return; // already accrued for this date — a rerun is a no-op, not a double-accrual
    }
    accrualRepository.save(new InterestAccrual(account.getId(), accrualDate, calculateInterest(account)));
}
```

This is the same underlying pattern as an idempotency key on an API call — track what's already been done, keyed by something stable (here, the account plus the accrual date), so a rerun is safe rather than destructive.

## 6. Parallelizing a Batch Job

A large batch job can be sped up by partitioning the dataset and processing partitions concurrently, as long as the partitions don't need to coordinate with each other.

```java
@Bean
public Step partitionedStep(JobRepository jobRepository, Step workerStep, Partitioner partitioner) {
    return new StepBuilder("partitionedStep", jobRepository)
        .partitioner("workerStep", partitioner) // splits the account range into N partitions
        .step(workerStep)
        .gridSize(8) // run 8 partitions concurrently
        .taskExecutor(new SimpleAsyncTaskExecutor())
        .build();
}
```

A common partitioning strategy is a range split (account IDs 1-1000, 1001-2000, ...) or a hash-based split, similar in spirit to choosing a shard key for a database — the same principle of picking a partitioning key that distributes work evenly applies here too.

## 7. Best Practices

| Practice | Recommendation |
| --- | --- |
| Process in bounded chunks, not row-by-row or all-at-once | Bounds memory usage and gives a natural, resumable checkpoint if the job fails partway through. |
| Make every batch job idempotent | A rerun (manual retry, an accidental double-trigger) must produce the same end state, not duplicate side effects. |
| Set skip/retry limits deliberately, not permissively by default | Especially for financial data — too loose silently drops bad records; too strict blocks an entire valid run over one bad one. |
| Tune chunk size against real data volume | No universal right answer — balance transaction overhead, memory footprint, and how much work a restart re-does. |
| Design for restart from the last committed chunk | A job interrupted by a crash or deploy should resume, not require reprocessing the entire dataset from the start. |
| Partition only when partitions are genuinely independent | Parallelizing a job whose partitions actually depend on each other's results just reintroduces coordination complexity for no real speedup. |
| Log and alert on skipped records, don't let them silently disappear | A skipped record was still a real failure for that specific piece of data — it needs visibility, not just a suppressed exception. |
