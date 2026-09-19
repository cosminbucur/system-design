Validation confirms that data meets required constraints before it's acted on — the goal is to fail fast, at the boundary, with a clear message, rather than let bad data propagate deeper into the system where it causes a confusing failure far from its actual cause (a `NullPointerException` three layers down instead of a clear "email is required" at the API edge). Java's Bean Validation (Jakarta Validation, implemented by Hibernate Validator) is the standard mechanism for this at the API/DTO layer; it's a distinct — and complementary.

## 1. Bean Validation Basics

Annotate a DTO's fields with constraints; Spring triggers validation automatically when the parameter is annotated `@Valid`.

```java
public record CreateAccountRequest(
    @NotBlank(message = "Owner name is required")
    String ownerName,

    @Email(message = "Must be a valid email address")
    String email,

    @NotNull
    @DecimalMin(value = "0.00", message = "Initial balance cannot be negative")
    BigDecimal initialBalance,

    @Pattern(regexp = "\\d{3}-\\d{2}-\\d{4}", message = "Invalid routing number format")
    String routingNumber
) {}
```

```java
@RestController
public class AccountController {

    @PostMapping("/api/accounts")
    public ResponseEntity<AccountResponse> create(@Valid @RequestBody CreateAccountRequest request) {
        // if any constraint fails, this method body never executes —
        // Spring throws MethodArgumentNotValidException before the controller method runs
        return ResponseEntity.ok(AccountResponse.from(accountService.create(request)));
    }
}
```

Common built-in constraints: `@NotNull`/`@NotBlank`/`@NotEmpty` (null vs. blank string vs. empty collection — pick the one matching the actual type), `@Size(min=, max=)`, `@Min`/`@Max`/`@DecimalMin`/`@DecimalMax`, `@Email`, `@Pattern` (regex), `@Past`/`@Future` (dates), `@Positive`/`@PositiveOrZero`/`@Negative`.

## 2. Wiring Validation Failures Into the API Error Contract

A failed `@Valid` check throws `MethodArgumentNotValidException` — handle it in the centralized exception handler to return a structured, field-level error response instead of a generic 500.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();

        return ResponseEntity.badRequest()
            .body(new ValidationErrorResponse(Instant.now(), 400, "VALIDATION_FAILED", errors));
    }
}

public record FieldError(String field, String message) {}
public record ValidationErrorResponse(Instant timestamp, int status, String code, List<FieldError> errors) {}
```

```json
{
  "timestamp": "2026-09-15T10:30:00Z",
  "status": 400,
  "code": "VALIDATION_FAILED",
  "errors": [
    { "field": "email", "message": "Must be a valid email address" },
    {
      "field": "initialBalance",
      "message": "Initial balance cannot be negative"
    }
  ]
}
```

Returning every failing field at once (not just the first) matters for usability — a client fixing one field at a time and resubmitting to discover the _next_ error is a frustrating, avoidable round-trip.

## 3. Cross-Field Validation

Some rules involve more than one field and can't be expressed as a single-field annotation — write a class-level constraint instead.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "endDate must be after startDate";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class DateRangeValidator implements ConstraintValidator<ValidDateRange, ReportRequest> {
    @Override
    public boolean isValid(ReportRequest request, ConstraintValidatorContext context) {
        if (request.startDate() == null || request.endDate() == null) {
            return true; // let @NotNull on the individual fields handle absence — don't duplicate that concern here
        }
        return request.endDate().isAfter(request.startDate());
    }
}

@ValidDateRange
public record ReportRequest(@NotNull LocalDate startDate, @NotNull LocalDate endDate) {}
```

A custom validator can also target a single field with more complex logic than a regex allows (e.g., checking a value against a database, or validating an IBAN's checksum) — implement `ConstraintValidator<YourAnnotation, FieldType>` the same way, just targeting the field type instead of the whole class.

## 4. Nested Object Validation

`@Valid` only validates the annotated object's own direct fields — it does **not** automatically cascade into nested objects unless the nested field is _also_ annotated `@Valid`.

```java
public record CreateOrderRequest(
    @NotNull Long customerId,

    @NotEmpty(message = "Order must have at least one line")
    @Valid // without this, OrderLineRequest's own constraints are silently skipped
    List<OrderLineRequest> lines
) {}

public record OrderLineRequest(
    @NotNull String sku,
    @Positive(message = "Quantity must be greater than zero") int quantity
) {}
```

Forgetting the nested `@Valid` is one of the most common Bean Validation mistakes — the outer object validates cleanly, but every constraint inside `lines` is silently never checked, and invalid nested data (e.g., a zero quantity) reaches the service layer looking pre-validated.

## 5. Validation Groups — Different Rules for Different Operations

The same DTO shape sometimes needs different validation rules depending on context (e.g., `id` must be absent on create, but present on update) — groups let one class support multiple constraint sets.

```java
public interface OnCreate {}
public interface OnUpdate {}

public record AccountRequest(
    @Null(groups = OnCreate.class, message = "id must not be provided on create")
    @NotNull(groups = OnUpdate.class, message = "id is required on update")
    Long id,

    @NotBlank
    String ownerName
) {}
```

```java
@PostMapping
public AccountResponse create(@Validated(OnCreate.class) @RequestBody AccountRequest request) { ... }

@PutMapping("/{id}")
public AccountResponse update(@Validated(OnUpdate.class) @RequestBody AccountRequest request) { ... }
```

Groups add real complexity — reach for them only when two operations genuinely share the same DTO shape with diverging rules; otherwise, two separate request records (`CreateAccountRequest`, `UpdateAccountRequest`) are usually simpler to read and maintain than one shared record threaded through multiple validation groups.

## 6. Method-Level Validation on Services

`@Validated` (Spring's variant, supporting groups) on a `@Service` class enables constraint annotations directly on method parameters — useful for validating inputs to internal service methods, not just HTTP controller boundaries.

```java
@Service
@Validated
public class TransferService {

    public void transfer(@NotNull Long fromAccountId,
                          @NotNull Long toAccountId,
                          @Positive BigDecimal amount) {
        // throws ConstraintViolationException before the method body runs, if any parameter is invalid
    }
}
```

This is a reasonable second line of defense for internal callers that don't go through a validated DTO (e.g., an internal batch job or another service calling this method directly), but it shouldn't be the _only_ validation for anything reachable from an external HTTP request.

## 7. Where Validation Actually Belongs — Boundary vs Domain

Bean Validation checks a request's _shape and format_ at the API boundary — is this field present, is this a valid email pattern, is this number non-negative. This is a genuinely different concern from the _business invariants_ an entity/aggregate root enforces internally — e.g., "a submitted order must have at least one line" or "a withdrawal can't exceed the current balance" are domain rules that don't depend on how the request arrived (HTTP, a message queue, an internal service call) and should be enforced by the domain object itself, not solely by an annotation on a controller-facing DTO.

```java
// Boundary validation (Bean Validation) — checks the REQUEST is well-formed
public record WithdrawRequest(
    @NotNull Long accountId,
    @Positive BigDecimal amount // "is this a sane, positive number" — a shape/format concern
) {}

// Domain validation — enforces the BUSINESS RULE, regardless of caller
public class Account {
    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) {
            throw new InsufficientFundsException(id, amount, balance); // a business rule, not a format check
        }
        this.balance = this.balance.subtract(amount);
    }
}
```

Skipping Bean Validation because "the domain will catch it anyway" produces worse error messages (a generic `InsufficientFundsException` stack trace instead of a clean 400 for a malformed request) and does unnecessary work before failing (constructing objects, starting a transaction) for input that was never going to be valid in the first place. Skipping domain validation because "the DTO was already validated" is worse: it leaves the business rule unenforced for every other caller that doesn't go through that specific controller (an internal service call, a batch job, a message consumer) — exactly the kind of gap that record compact constructors and aggregate roots are designed to close by making invalid domain states structurally impossible to construct, not just impossible to submit through one specific API.

## 8. Programmatic Validation

Sometimes you need to invoke the validator directly rather than relying on `@Valid`'s automatic interception — e.g., validating an object built from something other than an HTTP request body (a message queue payload, a batch-imported row).

```java
private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

public void processImportedRow(AccountImportRow row) {
    Set<ConstraintViolation<AccountImportRow>> violations = validator.validate(row);
    if (!violations.isEmpty()) {
        String messages = violations.stream()
            .map(v -> v.getPropertyPath() + ": " + v.getMessage())
            .collect(Collectors.joining("; "));
        throw new InvalidImportRowException(messages);
    }
    // ... proceed with a validated row
}
```

This is the same underlying validator Spring wires up automatically for `@Valid` — useful specifically for the message queues consumer case, where there's no HTTP request/controller to trigger validation for you.

## 9. Best Practices

| Practice                                                                                       | Recommendation                                                                                                                                          |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Validate at every entry point, not just HTTP controllers                                       | A message consumer or batch job bypasses `@Valid` entirely unless you invoke the `Validator` programmatically.                                 |
| Always add `@Valid` on nested objects/collections                                              | Without it, an outer object's constraints pass while every nested field's constraints are silently skipped.                                             |
| Return all validation errors at once, not just the first                                       | Avoids a frustrating fix-one-resubmit-see-the-next-error loop for API clients.                                                                          |
| Keep boundary (Bean Validation) and domain (entity/aggregate) validation both in place         | They check different things — format/shape at the edge, business invariants inside the domain.                                                 |
| Reach for validation groups only when genuinely justified                                      | Two separate request DTOs are usually simpler than one shared DTO threaded through multiple groups.                                                     |
| Prefer a custom `ConstraintValidator` over ad-hoc validation logic scattered in the controller | Keeps the rule reusable, testable in isolation, and declared right next to the field/class it constrains.                                               |
| Never rely solely on client-side validation                                                    | Anything enforced only in a frontend can be bypassed by any other caller (a direct API call, a script) — server-side validation is the actual contract. |
