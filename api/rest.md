API design is the contract between your service and everyone who calls it — once clients depend on it, changing it is expensive, so the goal is to get the shape right early and evolve it deliberately. This note covers REST conventions specifically; it complements exception handling (error response shape), security (auth), and pagination (list endpoints), which already cover their pieces of the contract in depth.

## 1. Resource Naming — Nouns, Not Verbs

A REST API models resources (things), and HTTP methods express the action — the URL itself should never contain a verb.

```
GOOD:
GET    /accounts/123
GET    /accounts/123/transactions
POST   /accounts/123/transactions

BAD:
GET    /getAccount?id=123
POST   /createTransaction
POST   /accounts/123/deleteTransaction/456
```

Conventions:

- Plural nouns for collections (`/accounts`, not `/account`) — consistent regardless of whether the collection currently has 0, 1, or many items.
- Nesting reflects genuine ownership (`/accounts/123/transactions` — a transaction belongs to an account), not just "related to." Don't nest more than 2-3 levels deep — `/customers/1/accounts/2/transactions/3/lines/4` becomes unwieldy; consider a top-level `/transaction-lines/4` with a filter instead once nesting gets that deep.
- Use kebab-case for multi-word resource names (`/payment-methods`, not `/paymentMethods` or `/payment_methods`) — most REST API style guides converge here, and it avoids ambiguity across case-sensitive vs. case-insensitive clients.

## 2. HTTP Methods and Their Semantics

| Method | Purpose                                                   | Safe?                 | Idempotent?                                     |
| ------ | --------------------------------------------------------- | --------------------- | ----------------------------------------------- |
| GET    | Retrieve a resource/collection                            | Yes (no side effects) | Yes                                             |
| POST   | Create a new resource, or trigger a non-idempotent action | No                    | No                                              |
| PUT    | Replace a resource entirely                               | No                    | Yes                                             |
| PATCH  | Partially update a resource                               | No                    | Not guaranteed (depends on the patch semantics) |
| DELETE | Remove a resource                                         | No                    | Yes                                             |

"Safe" means the method causes no server-side change (so it's OK to prefetch/cache/retry freely); "idempotent" means calling it N times has the same effect as calling it once (important for safe client-side retries after a timeout). A common mistake: using `POST` for an update that's actually a full replace (should be `PUT`) or using `GET` with side effects (e.g., `GET /accounts/123/send-statement` triggering an email) — breaks caching and retry assumptions clients and proxies rely on.

```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {

    @GetMapping("/{id}")
    public AccountResponse get(@PathVariable Long id) { ... }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public AccountResponse create(@RequestBody CreateAccountRequest request) { ... }

    @PutMapping("/{id}")
    public AccountResponse replace(@PathVariable Long id, @RequestBody UpdateAccountRequest request) { ... }

    @PatchMapping("/{id}")
    public AccountResponse partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> updates) { ... }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { ... }
}
```

## 3. Status Codes — Say What Actually Happened

Reserve specific codes for specific meanings so clients can branch on them programmatically, not just on the response body.

| Code                      | Meaning                                                 | Common use                                                             |
| ------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------- |
| 200 OK                    | Success, response has a body                            | `GET`, `PUT`, `PATCH`                                                  |
| 201 Created               | Resource created, `Location` header points to it        | `POST` that creates something                                          |
| 202 Accepted              | Request accepted for async processing, not yet complete | Kicking off a long-running job                                         |
| 204 No Content            | Success, no body                                        | `DELETE`, or a `PUT` with nothing meaningful to return                 |
| 400 Bad Request           | Malformed request / validation failure                  | Missing required field, invalid format                                 |
| 401 Unauthorized          | No valid authentication provided                        | Missing/invalid token                                                  |
| 403 Forbidden             | Authenticated, but not allowed to do this               | Valid token, insufficient permissions                                  |
| 404 Not Found             | Resource doesn't exist                                  | Wrong ID, or deliberately hiding existence from an unauthorized caller |
| 409 Conflict              | Request conflicts with current state                    | Optimistic lock version mismatch, duplicate unique key                 |
| 422 Unprocessable Entity  | Well-formed request, but semantically invalid           | Passes JSON schema validation but violates a business rule             |
| 429 Too Many Requests     | Rate limit exceeded                                     | Rate limit                                                             |
| 500 Internal Server Error | Unexpected failure                                      | Never expose the stack trace                                           |

`201 Created` should include a `Location` header pointing at the new resource:

```java
@PostMapping
public ResponseEntity<AccountResponse> create(@RequestBody CreateAccountRequest request) {
    Account account = accountService.create(request);
    URI location = URI.create("/api/accounts/" + account.getId());
    return ResponseEntity.created(location).body(AccountResponse.from(account));
}
```

## 4. Request/Response Contracts — DTOs, Not Entities

Never return a JPA entity directly from a controller — map to a dedicated DTO that controls exactly what's exposed.

```java
// Entity has internal fields (password hash, internal flags) that should never reach a client
public record AccountResponse(Long id, String ownerName, BigDecimal balance, String status) {
    public static AccountResponse from(Account account) {
        return new AccountResponse(account.getId(), account.getOwnerName(),
            account.getBalance(), account.getStatus().name());
    }
}
```

Benefits: the API contract stays stable even if the internal entity's structure changes (decoupling), you control exactly what's exposed (never leak internal-only fields), and you avoid the lazy-loading trap of serializing an entity whose associations haven't been fetched.

## 5. Versioning — Because Breaking Changes Are Inevitable

Once external clients depend on a response shape, you can't change it out from under them — versioning gives you a controlled way to evolve.

| Strategy                             | Example                                                | Tradeoff                                                                                                                                |
| ------------------------------------ | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| URI versioning                       | `/api/v1/accounts`, `/api/v2/accounts`                 | Most visible/explicit, easy to route at a gateway, but "pollutes" the URL and implies the whole resource changed even for a small tweak |
| Header versioning                    | `Accept: application/vnd.example.v2+json`              | Keeps URLs clean, but less discoverable (harder to just paste a URL and see what version you're hitting)                                |
| No versioning, only additive changes | Add optional fields, never remove/rename existing ones | Avoids versioning entirely, but requires strict discipline and doesn't cover every kind of breaking change                              |

Whichever strategy you pick, the actual discipline that matters more than the mechanism: **never remove or repurpose a field that existing clients might depend on** — adding new optional fields is (almost) always safe; removing or changing the meaning of an existing field is a breaking change requiring a new version, full stop.

## 6. Filtering, Sorting, and Pagination

Collection endpoints need a consistent convention for narrowing/ordering results — inventing a bespoke query syntax per endpoint makes the API harder to learn.

```
GET /api/accounts?status=ACTIVE&balance_gt=1000&sort=-createdAt&limit=20&cursor=eyJpZCI6MTAwfQ
```

- Filtering: `field=value` for equality; a suffix convention (`_gt`, `_lt`, `_in`) for other operators.
- Sorting: `sort=field` for ascending, `sort=-field` for descending; support multiple fields as a comma-separated list (`sort=-createdAt,ownerName`).
- Pagination: prefer cursor-based pagination for anything beyond small/static collections.

## 7. Idempotency for Non-Idempotent Operations

`POST` isn't idempotent by default — if a client's connection drops after the server processed the request but before the response arrived, a naive retry can create a duplicate resource (e.g., a duplicate payment). An idempotency key lets the client safely retry.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody CreatePaymentRequest request) {

    Optional<Payment> existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(PaymentResponse.from(existing.get())); // same key → return the original result, don't reprocess
    }

    Payment payment = paymentService.process(request, idempotencyKey);
    return ResponseEntity.status(CREATED).body(PaymentResponse.from(payment));
}
```

The client generates a unique key (typically a UUID) once per logical operation and sends the same key on every retry of that same operation — the server treats a repeated key as "already handled," returning the original result rather than processing it again. This matters enormously for anything money-related, mirroring the at-least-once + idempotent-consumer principle.

## 8. Async APIs for Long-Running Operations

Some operations (a large report generation, a batch import) can't complete within a normal HTTP request/response cycle — model them as "accepted, check back later" rather than holding the connection open.

```java
@PostMapping("/reports")
public ResponseEntity<Void> generateReport(@RequestBody ReportRequest request) {
    String jobId = reportService.startAsync(request);
    return ResponseEntity.accepted()
        .location(URI.create("/api/reports/jobs/" + jobId))
        .build(); // 202 Accepted — the client polls or gets notified separately
}

@GetMapping("/reports/jobs/{jobId}")
public ReportJobStatus getStatus(@PathVariable String jobId) {
    return reportJobRepository.findStatus(jobId); // PENDING, RUNNING, COMPLETED, FAILED
}
```

For genuinely event-driven integration (rather than polling), consider webhooks (you call the client back when done — see webhooks for signing, retries, and idempotency) or a message queue-based notification instead of making the client poll indefinitely.

## 9. Rate Limiting

Protects the API from being overwhelmed by a single client (accidental retry storm, or a genuinely abusive caller) — communicate limits explicitly so well-behaved clients can self-throttle instead of hammering `429` responses.

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1757930400
```

`Retry-After` tells the client exactly how long to back off — clients that respect it avoid a thundering-herd retry pattern reminiscent of the cache stampede problem.

## 10. API Documentation — OpenAPI/Swagger

Hand-written API docs drift out of sync with the actual code. Generating the spec directly from annotated controller code keeps documentation and implementation from diverging.

```java
@Operation(summary = "Get an account by ID")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Account found"),
    @ApiResponse(responseCode = "404", description = "Account not found")
})
@GetMapping("/{id}")
public AccountResponse get(@PathVariable Long id) { ... }
```

`springdoc-openapi` generates a live OpenAPI spec (and a browsable Swagger UI) directly from these annotations — treat the generated spec as the actual source of truth for API consumers, not a separately maintained wiki page that can silently go stale.

## 11. Best Practices

| Practice                                                                     | Recommendation                                                                                        |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Model resources as nouns, actions as HTTP methods                            | `/accounts/123`, not `/getAccount` — lets the method (`GET`/`POST`/etc.) carry the verb.              |
| Never return entities directly                                               | Map to DTOs so the API contract is decoupled from internal persistence structure.                     |
| Only add fields, never remove/repurpose them, without a new version          | The one rule that matters more than which versioning scheme you pick.                                 |
| Support idempotency keys on `POST` for anything with real-world consequences | Payments, transfers, order creation — anywhere a client retry could otherwise double-process.         |
| Use cursor pagination for list endpoints beyond trivial size                 | Offset pagination degrades and produces inconsistent pages at scale.                                  |
| Reserve specific status codes for specific meanings                          | Lets clients branch programmatically on the status code, not just parse the error body.               |
| Generate API docs from code, not a separate wiki                             | An OpenAPI spec generated from annotations can't silently drift from the actual implementation.       |
| Communicate rate limits explicitly                                           | `Retry-After` and `X-RateLimit-*` headers let well-behaved clients self-throttle instead of guessing. |
