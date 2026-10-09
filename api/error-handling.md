In REST APIs, exception handling maps backend Java errors into standardized HTTP responses with structured JSON payloads. Instead of exposing raw stack traces or SQL errors, you return proper HTTP Status Codes alongside machine-readable Error Codes and human-readable messages.

## Why error codes?

Error codes serve several concrete purposes beyond the HTTP status code:

- **Programmatic handling**: Clients can branch on a stable code (`if (error.code === 'trip_too_short')`) rather than parsing fragile human-readable strings. This is the single biggest reason — it makes client logic deterministic and testable.
- **Fine-grained categorization**: HTTP status codes are coarse (e.g., `400` covers a huge range of issues). Application-level error codes distinguish _which_ specific validation failed, _which_ resource wasn't found, _what_ conflict occurred, etc.
- **Retry & resilience decisions**: Precise codes let clients and intermediaries implement appropriate strategies — retry with backoff on 5xx, don't retry on 404, etc.
- **Localization**: A code can be mapped to a localized message per client locale without the server needing to know the user's language.
- **Consistency & self-documentation**: A stable, documented code is a contract. It's the one piece of an error response that won't change when you reword a message, so client code built against it stays stable.

In short: HTTP status codes tell the client _the category_ of failure; error codes tell it _exactly what went wrong_ so it can react correctly without guessing from prose.

## 1. Standard Error Response Structure

Every error returned by your API should follow a consistent JSON schema. RFC 7807 (Problem Details for HTTP APIs) is the standard blueprint:

```json
{
  "timestamp": "2026-09-15T10:30:00Z",
  "status": 400,
  "error": "BAD_REQUEST",
  "code": "INSUFFICIENT_FUNDS",
  "message": "Cannot withdraw $150.00. Current balance is $100.00.",
  "path": "/api/v1/accounts/123/withdraw"
}
```

HTTP Status Code (e.g., 400): Communicates the error class to generic HTTP clients, proxies, and web infrastructure.
Custom Error Code (e.g., INSUFFICIENT_FUNDS): Provides precise domain-specific context so frontends or API consumers can react programmatically.

## 2. Centralized Exception Handling in Spring Boot

Use @ControllerAdvice (or @RestControllerAdvice) and @ExceptionHandler to intercept exceptions across all controllers in one place.

Step 1: Define an Error Code Enum

```java
public enum ErrorCode {
USER_NOT_FOUND("USR-4041", HttpStatus.NOT_FOUND, "User profile not found"),
INSUFFICIENT_FUNDS("ACC-4001", HttpStatus.BAD_REQUEST, "Account balance is insufficient"),
INVALID_PAYLOAD("REQ-4000", HttpStatus.BAD_REQUEST, "Request payload validation failed"),
INTERNAL_ERROR("SYS-5000", HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected system error occurred");

    private final String code;
    private final HttpStatus httpStatus;
    private final String defaultMessage;

    ErrorCode(String code, HttpStatus httpStatus, String defaultMessage) {
        this.code = code;
        this.httpStatus = httpStatus;
        this.defaultMessage = defaultMessage;
    }

    public String getCode() { return code; }
    public HttpStatus getHttpStatus() { return httpStatus; }
    public String getDefaultMessage() { return defaultMessage; }

}
```

Step 2: Custom Base Exception

```java
public class BaseApiException extends RuntimeException {
private final ErrorCode errorCode;

    public BaseApiException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() { return errorCode; }

}
```

Step 3: Centralized Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BaseApiException.class)
    public ResponseEntity<ErrorResponse> handleApiException(BaseApiException ex, HttpServletRequest request) {
        ErrorCode errorCode = ex.getErrorCode();

        ErrorResponse body = new ErrorResponse(
            Instant.now(),
            errorCode.getHttpStatus().value(),
            errorCode.getCode(),
            ex.getMessage(),
            request.getRequestURI()
        );

        return new ResponseEntity<>(body, errorCode.getHttpStatus());
    }

    // Fallback for unhandled unexpected exceptions (Security & Hygiene)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex, HttpServletRequest request) {
        // Log actual exception internally (never expose internal details to caller)
        org.slf4j.LoggerFactory.getLogger(GlobalExceptionHandler.class)
            .error("Unhandled exception: ", ex);

        ErrorCode errorCode = ErrorCode.INTERNAL_ERROR;
        ErrorResponse body = new ErrorResponse(
            Instant.now(),
            errorCode.getHttpStatus().value(),
            errorCode.getCode(),
            errorCode.getDefaultMessage(),
            request.getRequestURI()
        );

        return new ResponseEntity<>(body, errorCode.getHttpStatus());
    }

}
```

3. Best Practices for REST Exception Handling
   | Practice | Recommendation |
   |---|---|
   | Never expose stack traces | Hide internal stack traces in production to prevent leaking system infrastructure details. |
   | Align HTTP & Domain Codes | Match HTTP status broad categories (4xx for client error, 5xx for server error) with specific internal string/numeric codes. |
   | Handle Validation Errors | Override Spring's MethodArgumentNotValidException to return field-by-field validation failures. |
   | Log 5xx Errors, Not 4xx | Treat 4xx errors as business logic branches; reserve high-priority alerts for unhandled 5xx exceptions.
