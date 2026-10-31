# Java Exceptions

## Core Java

### What is the difference between checked and unchecked exceptions?

Java exceptions are commonly divided into checked and unchecked exceptions.

- **Checked exceptions:** Checked by the compiler. A method must catch them or declare them using `throws`.
- **Unchecked exceptions:** Subclasses of `RuntimeException`. The compiler does not require callers to catch or declare them.
- **`Error`:** Also unchecked, but represents serious JVM or environment problems that application code generally should not try to recover from.

**Mental model:** Checked exceptions represent conditions the caller may reasonably be expected to handle. Unchecked exceptions commonly represent programming errors, invalid input, or failures expressed through an application's runtime contract. This is a design convention, not a strict rule enforced by Java.

**Code example:**

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class SettlementFileReader {

    // IOException is checked, so it is declared.
    public String read(Path path) throws IOException {
        return Files.readString(path);
    }

    // IllegalArgumentException is unchecked.
    public void validateAmount(long amountInCents) {
        if (amountInCents <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
}
```

**Real-world production usage:** Reading a settlement file can fail because the file is missing or inaccessible. The caller may decide to retry, mark the settlement batch as failed, or alert an operator. An invalid payment amount is usually a rejected request or programming/input error, so an unchecked exception is often appropriate.

### What is the difference between `throw` and `throws`?

`throw` is used inside a method to actually throw an exception. `throws` appears in a method declaration and states that the method may pass an exception to its caller.

**Mental model:**

- `throw`: Perform the action.
- `throws`: Declare the method's exception contract.

**Code example:**

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class SettlementFileService {

    public String loadFile(Path path) throws IOException {
        if (path == null) {
            throw new IllegalArgumentException("Path is required");
        }

        return Files.readString(path);
    }
}
```

**Real-world production usage:** A low-level settlement-file adapter can declare `throws IOException`. A service above it decides what that failure means to the application: retry the read, fail the batch, or translate it into a domain-specific exception.

### How does `try`, `catch`, and `finally` work?

- `try` contains code that may fail.
- `catch` handles a matching exception.
- `finally` runs when control leaves the try/catch construct, whether normally or because an exception was thrown. It may not run if the JVM or process is forcibly terminated.

Put more specific catch blocks before more general ones. Avoid returning or throwing from `finally`, because it can hide the original return value or exception.

**Code example:**

```java
public void processSettlementFile() {
    try {
        validateFile();
        importTransactions();
    } catch (InvalidSettlementFileException e) {
        markBatchRejected(e.getMessage());
    } catch (RuntimeException e) {
        markBatchFailed();
        throw e;
    } finally {
        recordProcessingAttempt();
    }
}
```

**Real-world production usage:** A settlement import may need to record that processing was attempted regardless of success. `finally` is suitable for simple cleanup or bookkeeping, but resource cleanup is generally better expressed with try-with-resources.

### How does exception propagation work in Java?

If a method does not handle an exception, the exception propagates to its caller. This continues up the call stack until a matching handler is found. If no handler handles it, it reaches the thread's uncaught-exception handler and the operation or thread may terminate.

For checked exceptions, each method in the call chain must catch the exception or declare it.

**Code example:**

```java
public class SettlementController {
    private final SettlementService service;

    public SettlementController(SettlementService service) {
        this.service = service;
    }

    public void importFile(String fileName) throws SettlementFileException {
        service.importFile(fileName);
    }
}

class SettlementService {
    private final SettlementFileRepository repository;

    SettlementService(SettlementFileRepository repository) {
        this.repository = repository;
    }

    void importFile(String fileName) throws SettlementFileException {
        repository.read(fileName);
    }
}

class SettlementFileRepository {
    void read(String fileName) throws SettlementFileException {
        // Read and parse the file.
    }
}

class SettlementFileException extends Exception {
    SettlementFileException(String message) {
        super(message);
    }
}
```

**Real-world production usage:** A repository may detect a file-read failure, a service may decide whether it should fail the batch, and an API boundary may translate the failure into an HTTP response. Handle exceptions at the layer that has enough context to make a meaningful decision—not by catching and ignoring them at every layer.

### When should you create custom exceptions?

Create a custom exception when it communicates a meaningful application or domain failure that callers need to distinguish. Avoid creating a new exception class for every minor variation.

A useful distinction:

- **Domain failure:** A payment or settlement cannot be processed because a business rule was not met.
- **Technical failure:** A dependency, file system, database, or network operation failed.

**Code example:**

```java
public class PaymentNotFoundException extends RuntimeException {
    public PaymentNotFoundException(String paymentId) {
        super("Payment not found: " + paymentId);
    }
}

public class PaymentGatewayException extends RuntimeException {
    public PaymentGatewayException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class PaymentService {

    private final PaymentRepository repository;

    public PaymentService(PaymentRepository repository) {
        this.repository = repository;
    }

    public Payment findPayment(String paymentId) {
        return repository.findById(paymentId)
                .orElseThrow(() -> new PaymentNotFoundException(paymentId));
    }
}
```

**Real-world production usage:** A payment API may distinguish a missing payment from a gateway communication failure. The first might map to HTTP 404; the second may map to a retryable server-side response. Do not put card numbers, credentials, tokens, or other sensitive data into exception messages.

### What is exception wrapping and why should you preserve the cause?

Exception wrapping translates a low-level exception into a higher-level exception that makes sense to the calling layer. Preserve the original exception as the cause so logs and diagnostics retain the underlying failure and stack trace.

**Code example:**

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class SettlementFileAdapter {

    public String read(Path path) {
        try {
            return Files.readString(path);
        } catch (IOException e) {
            throw new SettlementFileReadException(
                    "Unable to read settlement file", e);
        }
    }
}

class SettlementFileReadException extends RuntimeException {
    SettlementFileReadException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**Real-world production usage:** The file adapter knows that an `IOException` occurred. The settlement service can reason in terms of a settlement-file failure without depending on file-system details. The original `IOException` remains available through `getCause()` for troubleshooting. Do not replace the original exception with a new one that discards the cause.

### What is try-with-resources?

Try-with-resources automatically closes resources that implement `AutoCloseable`, such as streams, readers, and many database-related resources. Resources are closed when the block exits, including when an exception occurs, and are closed in the reverse order in which they were declared.

**Code example:**

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class SettlementFileReader {

    public void read(Path path) throws IOException {
        try (BufferedReader reader = Files.newBufferedReader(path)) {
            String line;

            while ((line = reader.readLine()) != null) {
                processLine(line);
            }
        }
    }

    private void processLine(String line) {
        // Parse and process one record.
    }
}
```

If the body throws an exception and closing the resource also throws, the body exception remains the primary exception and the close exception is recorded as a suppressed exception. Suppressed exceptions can be inspected with `getSuppressed()`.

**Real-world production usage:** A settlement-file import should not leave file handles open when parsing fails. Try-with-resources makes cleanup predictable and reduces resource leaks during repeated batch processing.

### How do you handle checked exceptions in lambdas and streams?

Most standard Java functional interfaces used by streams—such as `Function`, `Consumer`, and `Predicate`—do not declare checked exceptions. A lambda passed to them cannot directly throw a checked exception unless it catches it or translates it into an unchecked exception.

**Code example:**

```java
import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

public class SettlementFiles {

    public List<String> readAll(List<Path> paths) {
        return paths.stream()
                .map(path -> {
                    try {
                        return Files.readString(path);
                    } catch (IOException e) {
                        throw new UncheckedIOException(
                                "Unable to read " + path, e);
                    }
                })
                .toList();
    }
}
```

`UncheckedIOException` is useful when an I/O operation must be used in an API that does not accept checked exceptions. It preserves the original `IOException` as its cause.

**Real-world production usage:** For a small collection of settlement files, a stream may be convenient. For a large batch where each file needs independent retry, status tracking, or error reporting, a regular `for` loop can be clearer. Decide whether one failed item should fail the entire operation or be recorded while processing continues.

### How does `Optional.orElseThrow()` work?

`Optional.orElseThrow()` returns the value when it is present. If it is empty, it throws an exception. It is a concise way to express that a value is required at that point in the code.

**Code example:**

```java
public Payment getPayment(String paymentId) {
    return paymentRepository.findById(paymentId)
            .orElseThrow(() -> new PaymentNotFoundException(paymentId));
}
```

The supplier is evaluated only when the `Optional` is empty.

**Real-world production usage:** A payment lookup can return `Optional<Payment>` when the record may not exist. The service converts the absence into a meaningful `PaymentNotFoundException`, which the API exception handler can map to a 404 response. Do not use `Optional` as a replacement for every nullable field or method parameter. It is most commonly useful as a return type to express possible absence.

### How do exceptions work with `CompletableFuture`?

A `CompletableFuture` can complete normally or exceptionally. An exception thrown inside an asynchronous stage causes that stage to complete exceptionally, and dependent stages generally observe that failure unless they handle it.

Important methods:

- `exceptionally`: Recover from a failure by supplying a replacement result.
- `handle`: Process either the result or the exception and produce a new result.
- `whenComplete`: Observe the result or exception, typically for logging or metrics, without intentionally converting the outcome.
- `join`: Wait for completion and report failure using an unchecked `CompletionException`.
- `get`: Wait for completion and report failure using `ExecutionException`; it also declares `InterruptedException`.

**Code example:**

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

public class PaymentService {

    private final PaymentGateway gateway;
    private final Executor paymentExecutor;

    public PaymentService(PaymentGateway gateway, Executor paymentExecutor) {
        this.gateway = gateway;
        this.paymentExecutor = paymentExecutor;
    }

    public CompletableFuture<PaymentResult> authorize(PaymentRequest request) {
        return CompletableFuture
                .supplyAsync(
                        () -> gateway.authorize(request),
                        paymentExecutor)
                .whenComplete((result, error) -> {
                    if (error != null) {
                        recordFailure(error);
                    }
                })
                .exceptionally(error ->
                        PaymentResult.failed("Authorization failed"));
    }

    private void recordFailure(Throwable error) {
        // Record a correlation ID and safe diagnostic details.
    }
}
```

The example uses an explicit executor rather than implicitly using the common `ForkJoinPool`. In production, choose an executor appropriate for the work and configure its capacity and queueing behavior.

**Real-world production usage:** A payment gateway call may time out even though the gateway actually processed the authorization. A timeout does not prove that the payment failed. Before retrying, use an idempotency key and/or query the gateway for the transaction's status. Blind retries can create duplicate financial operations.

Also remember that `exceptionally` handles an upstream failure for the stage where it is attached; a later stage can fail independently and needs its own handling strategy.

## Spring Boot exception handling

### How do you handle exceptions globally in Spring Boot?

`@RestControllerAdvice` allows exception handling to be centralized across controllers. Methods annotated with `@ExceptionHandler` define how selected exceptions are translated into HTTP responses. This keeps controller methods focused on request handling and business orchestration instead of repeating response-mapping logic.

**Code example:**

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(PaymentNotFoundException.class)
    public ProblemDetail handleNotFound(PaymentNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND,
                "The requested payment was not found");

        problem.setTitle("Payment not found");
        return problem;
    }

    @ExceptionHandler(PaymentGatewayException.class)
    public ProblemDetail handleGatewayFailure(PaymentGatewayException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.SERVICE_UNAVAILABLE,
                "The payment service is temporarily unavailable");

        problem.setTitle("Payment service unavailable");
        return problem;
    }
}
```

`ProblemDetail` is available with Spring Framework 6+ (commonly used with Spring Boot 3+). In older Spring versions, return a suitable response DTO or `ResponseEntity` instead.

**Real-world production usage:** A payment API might map a missing payment to 404, invalid request data to 400, and a temporary downstream gateway failure to 503. The HTTP status and response body are part of the API contract, so clients can make appropriate decisions.

Do not expose stack traces, SQL statements, internal hostnames, credentials, access tokens, or card data in API responses. Log useful diagnostic details securely and include a correlation or trace ID so an incident can be investigated.

### How do you handle validation exceptions in Spring Boot?

Spring MVC can validate request bodies annotated with `@Valid` or `@Validated`, provided the appropriate validation setup is present. If validation fails, Spring raises an exception such as `MethodArgumentNotValidException` for a typical `@Valid` request body. A global exception handler can convert the validation details into a consistent 400 response.

**Code example:**

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

public class PaymentRequest {

    @NotBlank
    private String merchantId;

    @Positive
    private long amountInCents;

    // Getters and setters omitted.
}
```

```java
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/payments")
public class PaymentController {

    @PostMapping
    public String createPayment(@Valid @RequestBody PaymentRequest request) {
        return "Payment request accepted";
    }
}
```

**Real-world production usage:** A payment endpoint can reject a blank merchant ID or non-positive amount before the request reaches the payment service. Keep request-shape validation separate from deeper business rules—for example, whether a merchant is enabled or whether a transaction is allowed for a particular account.

### Why might `@RestControllerAdvice` not catch Spring Security authentication failures?

Spring Security processes requests through a filter chain before they reach a controller. Authentication and authorization failures can therefore occur before controller invocation and outside the normal MVC exception-handling path.

Spring Security provides dedicated extension points:

- `AuthenticationEntryPoint` handles authentication failures, commonly returning HTTP 401.
- `AccessDeniedHandler` handles authorization failures for an authenticated user, commonly returning HTTP 403.

**Code example:**

```java
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;

import java.io.IOException;

public class ApiAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException exception) throws IOException {

        response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                "Authentication is required");
    }
}
```

A corresponding `AccessDeniedHandler` should produce a 403 response when an authenticated caller lacks permission. In a real API, both handlers should follow the API's consistent error-response format rather than exposing internal exception details.

**Real-world production usage:** For a payment API, an expired or invalid JWT normally results in 401. A validly authenticated user who is not authorized to refund a transaction normally receives 403. Authentication answers “Who are you?”; authorization answers “Are you allowed to do this?”

## Senior-level design considerations

### How should you decide where an exception should be handled?

Handle an exception at the layer that has enough context to make a meaningful decision. Lower layers should generally report or translate failures without silently deciding business policy. The API boundary should translate application failures into stable client-facing responses.

A practical way to think about the layers:

1. **Adapter / repository:** Understands technical details such as SQL, HTTP, file I/O, and serialization.
2. **Service / domain:** Understands business meaning, such as payment rejected, settlement batch invalid, or retry permitted.
3. **API boundary:** Understands HTTP status codes and the public error contract.
4. **Operational layer:** Records diagnostics, metrics, traces, and alerts.

**Code example:**

```java
public Payment authorize(PaymentRequest request) {
    try {
        return gateway.authorize(request);
    } catch (GatewayTimeoutException e) {
        throw new PaymentGatewayException(
                "Payment authorization outcome is unknown", e);
    }
}
```

**Real-world production usage:** For financial operations, distinguish a definitive decline from an unknown outcome. A decline is a known business result. A timeout may mean the gateway did not respond even though it processed the request. The latter requires idempotency and status reconciliation—not an automatic assumption that the payment failed.
