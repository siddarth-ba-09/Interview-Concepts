# Java Exception Handling

---

## 1. Overview

Exception handling is Java's mechanism for dealing with abnormal conditions that disrupt the normal flow of program execution. Java uses a class hierarchy to model these conditions, a `try-catch-finally` construct to handle them, and a checked/unchecked distinction to enforce handling at compile time.

**Core goals:**
- Separate error-handling code from normal business logic.
- Propagate errors up the call stack without every method manually testing return codes.
- Ensure resources (files, DB connections, sockets) are always released even when errors occur.

---

## 2. The Exception Hierarchy

```
java.lang.Throwable
        │
        ├── java.lang.Error                    (JVM-level, unrecoverable)
        │       ├── OutOfMemoryError
        │       ├── StackOverflowError
        │       ├── AssertionError
        │       └── LinkageError
        │               ├── NoClassDefFoundError
        │               └── ExceptionInInitializerError
        │
        └── java.lang.Exception                (application-level)
                ├── IOException                (checked)
                │       ├── FileNotFoundException
                │       └── SocketException
                ├── SQLException               (checked)
                ├── ReflectiveOperationException (checked)
                │       └── ClassNotFoundException
                ├── CloneNotSupportedException (checked)
                └── RuntimeException           (unchecked)
                        ├── NullPointerException
                        ├── IllegalArgumentException
                        │       └── NumberFormatException
                        ├── IllegalStateException
                        ├── IndexOutOfBoundsException
                        │       ├── ArrayIndexOutOfBoundsException
                        │       └── StringIndexOutOfBoundsException
                        ├── ClassCastException
                        ├── ArithmeticException
                        ├── UnsupportedOperationException
                        ├── ConcurrentModificationException
                        └── NoSuchElementException
```

### Key Nodes

| Node | Nature | Compiler check | Typical source |
|------|--------|----------------|----------------|
| `Error` | Unrecoverable JVM condition | Unchecked | JVM internals |
| `Exception` (non-Runtime) | Recoverable, anticipated | **Checked** | I/O, network, DB |
| `RuntimeException` | Programming bugs | **Unchecked** | Null deref, bad casts |

---

## 3. Checked vs. Unchecked Exceptions

### 3.1 Checked Exceptions

- Subclass of `Exception` but NOT `RuntimeException`.
- Compiler **forces** you to either `catch` them or declare `throws` in the method signature.
- Represent conditions outside the programmer's control: file not found, network timeout, DB unavailable.

```java
// Must handle or declare — compiler enforces this
public String readFile(String path) throws IOException {
    return Files.readString(Path.of(path));
}

// Or handle it
public String readFile(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        log.error("Could not read file: {}", path, e);
        return "";
    }
}
```

### 3.2 Unchecked Exceptions (`RuntimeException`)

- No compile-time enforcement.
- Represent programming mistakes: null dereference, invalid arguments, index out of bounds.
- Should generally NOT be caught unless you have a specific recovery strategy.

```java
// No need to declare or catch — compiles fine
public int divide(int a, int b) {
    return a / b; // throws ArithmeticException if b == 0
}
```

### 3.3 The Checked vs. Unchecked Debate

| Argument for Checked | Argument for Unchecked |
|---|---|
| Forces callers to think about error handling | Checked exceptions pollute method signatures throughout the call chain |
| Self-documenting API | Hard to handle at the right level — often just `throws Exception` bubbling |
| Part of the contract | Spring, Hibernate, modern frameworks all use unchecked exceptions |

**Modern practice (Spring, Clean Code):** Use **unchecked exceptions** for application-level errors. Reserve checked exceptions for truly recoverable, caller-must-handle scenarios (e.g., a library that reads from an InputStream the caller provided).

---

## 4. `try-catch-finally` Mechanics

### 4.1 Basic Structure

```java
try {
    // code that may throw
} catch (SpecificException e) {
    // handle specific type first
} catch (AnotherException e) {
    // handle another type
} catch (Exception e) {
    // broader catch — always after specific ones
} finally {
    // ALWAYS runs: whether exception thrown or not, caught or not
    // even if a return statement is hit inside try or catch
}
```

### 4.2 Multi-catch (Java 7+)

When two exception types require identical handling, merge them with `|`:

```java
try {
    process();
} catch (IOException | SQLException e) {
    // e is effectively final here — cannot be reassigned
    log.error("Data access failure", e);
    throw new ServiceException("Failed to process", e);
}
```

**Restriction:** The exception types in a multi-catch cannot be in an inheritance relationship (compiler error — the subtype is redundant).

### 4.3 `finally` Guarantees and Edge Cases

`finally` runs in all scenarios except:
- `System.exit()` is called.
- The JVM crashes (segfault, OOM in a native library).
- The thread is killed with `Thread.stop()` (deprecated).

**Gotcha — `finally` suppresses the exception if it throws:**
```java
try {
    throw new IllegalStateException("original");
} finally {
    throw new RuntimeException("finally"); // original exception is LOST silently
}
// Only RuntimeException propagates — IllegalStateException is gone
```

**Gotcha — `return` in `finally` suppresses the exception:**
```java
int compute() {
    try {
        throw new RuntimeException("error");
    } finally {
        return 42; // exception is silently swallowed, method returns 42
    }
}
```

**Rule:** Never `throw` or `return` from a `finally` block. Only use it for cleanup.

### 4.4 Variable Scope

Variables declared inside `try` are not visible in `catch` or `finally`:

```java
Connection conn = null; // declare outside to use in finally
try {
    conn = dataSource.getConnection();
    // ...
} catch (SQLException e) {
    // handle
} finally {
    if (conn != null) conn.close(); // conn is visible here
}
```

---

## 5. `try-with-resources` (Java 7+)

Any object implementing `java.lang.AutoCloseable` (or its subtype `java.io.Closeable`) can be declared in the `try(...)` header. The JVM guarantees `close()` is called automatically when the block exits — normally or via exception.

```java
// Old style — verbose and error-prone
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader(path));
    return reader.readLine();
} finally {
    if (reader != null) reader.close();
}

// try-with-resources — clean and safe
try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
    return reader.readLine();
} catch (IOException e) {
    throw new ServiceException("Could not read " + path, e);
}
```

### 5.1 Multiple Resources

Declare multiple resources separated by semicolons. They are closed in **reverse declaration order** (LIFO):

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(SQL);
     ResultSet rs = ps.executeQuery()) {

    while (rs.next()) { /* process */ }
}
// Close order: rs → ps → conn
```

### 5.2 Suppressed Exceptions

If both the `try` body and `close()` throw, the `close()` exception is **suppressed** (attached to the primary exception) rather than replacing it. This preserves the original failure:

```java
try (MyResource r = new MyResource()) {
    throw new RuntimeException("body exception");
    // r.close() also throws → added as suppressed
}

// Retrieving suppressed exceptions:
try {
    // ...
} catch (RuntimeException e) {
    Throwable[] suppressed = e.getSuppressed(); // inspect close() failures
}
```

### 5.3 Making Your Own Resource

```java
public class ManagedTransaction implements AutoCloseable {
    private final Connection conn;
    private boolean committed = false;

    public ManagedTransaction(DataSource ds) throws SQLException {
        this.conn = ds.getConnection();
        this.conn.setAutoCommit(false);
    }

    public void commit() throws SQLException {
        conn.commit();
        committed = true;
    }

    @Override
    public void close() throws SQLException {
        if (!committed) conn.rollback(); // auto-rollback on any exception
        conn.close();
    }
}

// Usage
try (ManagedTransaction tx = new ManagedTransaction(dataSource)) {
    orderRepo.save(order);
    inventoryRepo.deduct(order.getItemId(), order.getQty());
    tx.commit();
} // auto-rollback if any exception occurs before commit
```

---

## 6. `throw` and `throws`

### `throw` — raise an exception

```java
void validate(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
}
```

### `throws` — declare checked exceptions in a method signature

```java
public User findById(Long id) throws UserNotFoundException {
    User user = repo.findById(id).orElse(null);
    if (user == null) throw new UserNotFoundException("User not found: " + id);
    return user;
}
```

**Widening throws:** A method can declare `throws Exception` even if it throws only `IOException` — legal but bad practice (forces all callers to handle `Exception`).

**Overriding rules:** An overriding method cannot declare new or broader checked exceptions than the parent:

```java
class Parent {
    void process() throws IOException { }
}

class Child extends Parent {
    @Override
    void process() throws FileNotFoundException { } // OK — subtype of IOException
    // void process() throws Exception { } // COMPILE ERROR — broader than IOException
    // void process() throws SQLException { } // COMPILE ERROR — unrelated checked type
}
```

---

## 7. Exception Chaining

Wrap lower-level exceptions in higher-level ones to build a meaningful stack trace without losing the root cause:

```java
public Order findOrder(Long id) {
    try {
        return orderRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Order not found: " + id));
    } catch (DataAccessException e) {
        // Wrap infrastructure exception in domain exception, preserve cause
        throw new OrderServiceException("Failed to retrieve order: " + id, e);
    }
}
```

Always pass the original exception as the `cause` parameter. Without it, the root cause is lost and debugging becomes extremely hard.

```java
// BAD — root cause lost
catch (IOException e) {
    throw new ServiceException("Failed");
}

// GOOD — root cause preserved
catch (IOException e) {
    throw new ServiceException("Failed to read config", e);
}
```

Retrieve the cause chain programmatically:
```java
Throwable cause = e.getCause();
while (cause != null) {
    System.out.println(cause.getMessage());
    cause = cause.getCause();
}
```

---

## 8. Custom Exceptions

### 8.1 Design Principles

- Extend `RuntimeException` for domain/application errors (modern standard).
- Extend `Exception` only when you genuinely want to force callers to handle it.
- Add domain-specific fields (error codes, entity IDs) for programmatic handling.
- Always provide constructors that accept a `cause`.

### 8.2 Standard Custom Exception Template

```java
public class OrderNotFoundException extends RuntimeException {

    private final Long orderId;

    public OrderNotFoundException(Long orderId) {
        super("Order not found with id: " + orderId);
        this.orderId = orderId;
    }

    public OrderNotFoundException(Long orderId, Throwable cause) {
        super("Order not found with id: " + orderId, cause);
        this.orderId = orderId;
    }

    public Long getOrderId() {
        return orderId;
    }
}
```

### 8.3 Exception with Error Code (API-friendly)

```java
public class ApiException extends RuntimeException {

    private final String errorCode;
    private final int httpStatus;

    public ApiException(String errorCode, String message, int httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public ApiException(String errorCode, String message, int httpStatus, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() { return errorCode; }
    public int getHttpStatus()   { return httpStatus; }
}

// Usage
throw new ApiException("ORDER_NOT_FOUND", "Order 123 does not exist", 404);
```

### 8.4 Exception Hierarchy for a Domain

```
AppException (base, RuntimeException)
        ├── NotFoundException       (404)
        │       ├── OrderNotFoundException
        │       └── UserNotFoundException
        ├── ValidationException     (400)
        ├── ConflictException       (409)
        └── ServiceUnavailableException (503)
```

---

## 9. Exception Handling in Spring MVC

### 9.1 `@ExceptionHandler` — Controller-Scoped

Handles exceptions thrown by a specific controller:

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        return orderService.findById(id); // throws OrderNotFoundException
    }

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(OrderNotFoundException ex) {
        return new ErrorResponse("ORDER_NOT_FOUND", ex.getMessage());
    }
}
```

### 9.2 `@RestControllerAdvice` — Global Handler

Handles exceptions across all controllers:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(NotFoundException ex) {
        return new ErrorResponse(ex.getErrorCode(), ex.getMessage());
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(ValidationException ex) {
        return new ErrorResponse("VALIDATION_ERROR", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleBeanValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ErrorResponse("INVALID_INPUT", message);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleUnexpected(Exception ex, HttpServletRequest req) {
        log.error("Unexpected error at {}", req.getRequestURI(), ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}

record ErrorResponse(String code, String message) {}
```

### 9.3 `ResponseEntity` for Fine-Grained Control

```java
@ExceptionHandler(ConflictException.class)
public ResponseEntity<ErrorResponse> handleConflict(ConflictException ex) {
    ErrorResponse body = new ErrorResponse(ex.getErrorCode(), ex.getMessage());
    return ResponseEntity.status(HttpStatus.CONFLICT)
                         .header("X-Error-Code", ex.getErrorCode())
                         .body(body);
}
```

---

## 10. Production Use Cases & Best Practices

### 10.1 Log at the Right Level, Once

```java
// WRONG — logs at service, then again at controller, duplicate stack traces in logs
catch (OrderNotFoundException e) {
    log.error("Order not found", e); // log here
    throw e;                          // AND propagate — causes double logging
}

// CORRECT — let it propagate, log once at the boundary (@ControllerAdvice)
catch (OrderNotFoundException e) {
    throw e; // don't log — ControllerAdvice will log it centrally
}
```

**Rule:** Log an exception exactly once, at the point where it is **handled** (not re-thrown). `@ControllerAdvice` is the natural boundary.

### 10.2 Include Context in Exception Messages

```java
// BAD — useless in production logs
throw new RuntimeException("Not found");

// GOOD — immediately actionable
throw new OrderNotFoundException("Order not found: orderId=" + orderId
    + ", customerId=" + customerId);
```

### 10.3 Don't Swallow Exceptions

```java
// CATASTROPHIC — failure silently ignored
try {
    processPayment(order);
} catch (Exception e) {
    // nothing here
}

// ACCEPTABLE — log it, but silent failure is intentional and documented
try {
    sendAnalyticsEvent(event);
} catch (Exception e) {
    log.warn("Analytics event failed (non-critical): {}", e.getMessage());
}
```

### 10.4 Never Catch `Throwable` or `Error`

```java
// DANGEROUS — catches OutOfMemoryError, StackOverflowError, etc.
// JVM is in an undefined state; continuing is worse than crashing
try {
    ...
} catch (Throwable t) { // almost never correct
    ...
}
```

Only acceptable at JVM shutdown hooks or top-level server request handlers to prevent one request from killing the process — and even then, re-throw or force restart.

### 10.5 Use `Optional` Instead of Null / Exception for Expected Absence

```java
// Throwing an exception for an expected "not found" in a query is expensive and noisy
// Optional is appropriate when absence is a normal, expected outcome

Optional<User> findByEmail(String email); // normal — user may or may not exist

// Exception appropriate when absence is unexpected / a contract violation
User getById(Long id); // throws UserNotFoundException — id must exist
```

### 10.6 Fail Fast with Preconditions

```java
import java.util.Objects;

public void processOrder(Order order, Long userId) {
    Objects.requireNonNull(order, "order must not be null");
    Objects.requireNonNull(userId, "userId must not be null");
    if (order.getItems().isEmpty()) {
        throw new IllegalArgumentException("Order must have at least one item");
    }
    // ... rest of method
}
```

### 10.7 Transaction Rollback and Exceptions

Spring `@Transactional` only rolls back on `RuntimeException` and `Error` by default. Checked exceptions do NOT trigger rollback unless explicitly configured:

```java
// Will NOT roll back on IOException by default
@Transactional
public void saveReport(Report report) throws IOException {
    reportRepo.save(report);
    fileWriter.write(report); // throws IOException → no rollback!
}

// CORRECT — declare rollback for checked exceptions
@Transactional(rollbackFor = IOException.class)
public void saveReport(Report report) throws IOException {
    reportRepo.save(report);
    fileWriter.write(report);
}

// Or better — wrap in unchecked
@Transactional
public void saveReport(Report report) {
    try {
        reportRepo.save(report);
        fileWriter.write(report);
    } catch (IOException e) {
        throw new ReportException("Failed to write report", e); // RuntimeException → rollback
    }
}
```

---

## 11. Common Pitfalls & Gotchas

### Catching a Supertype When You Need a Subtype

```java
try {
    process();
} catch (Exception e) {          // catches everything — too broad
    log.error("Error", e);
    throw new ServiceException(); // wraps it, original type lost to caller
}

// Better — catch specifically what you can handle
try {
    process();
} catch (FileNotFoundException e) {
    // handle missing file specifically
} catch (IOException e) {
    // handle other I/O errors
}
```

---

### Exception in `catch` Block Loses Original

```java
try {
    riskyOperation();
} catch (IOException e) {
    cleanup(); // if cleanup() throws, original IOException e is gone
}

// Fix
try {
    riskyOperation();
} catch (IOException e) {
    try { cleanup(); } catch (Exception cleanupEx) {
        e.addSuppressed(cleanupEx); // attach cleanup failure to original
    }
    throw new ServiceException("Operation failed", e);
}
```

---

### `NullPointerException` with No Message (Pre-Java 14)

Before Java 14's **Helpful NullPointerExceptions** (enabled by default in Java 15+), NPE messages were empty strings. In Java 15+:

```
NullPointerException: Cannot invoke "String.length()" because "str" is null
```

Enable on older JVMs: `-XX:+ShowCodeDetailsInExceptionMessages`

---

### `printStackTrace()` in Production

```java
catch (Exception e) {
    e.printStackTrace(); // outputs to System.err, bypasses your logging framework
}

// Always use your logger
catch (Exception e) {
    log.error("Operation failed", e); // goes through SLF4J → logback/log4j
}
```

---

### Catching `InterruptedException` Without Re-Interrupting

```java
// WRONG — swallows the interrupted status, thread never stops cleanly
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    log.warn("Interrupted"); // status cleared, thread loop continues forever
}

// CORRECT — restore interrupted status so the caller can respond
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore interrupted status
    throw new RuntimeException("Thread interrupted", e); // or return/break
}
```

---

### Stack Trace Performance

Creating an exception captures a full stack trace, which is **expensive** (profiling shows it can take microseconds to milliseconds depending on stack depth). If you throw exceptions in hot paths (e.g., using exceptions for flow control), override `fillInStackTrace()`:

```java
public class FlowControlException extends RuntimeException {
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this; // skip stack trace capture — much faster
    }
}
```

Use only for internal flow-control exceptions, not for errors you'll log.

---

## 12. Scenario Walkthroughs

### 12.1 Exception Propagation Through Layers

**Scenario:** A REST endpoint calls a service, which calls a repository. The DB is down.

```
[HTTP Request]
      │
      ▼
OrderController.getOrder(id)
      │
      ▼ calls
OrderService.findById(id)
      │
      ▼ calls
OrderRepository.findById(id)
      │
      throws DataAccessException (Spring wraps SQLException)
      │
      ▼ propagates to
OrderService.findById(id)
      │ catches DataAccessException
      │ throws OrderServiceException("DB unavailable", cause)
      │
      ▼ propagates to
OrderController.getOrder(id)
      │ no catch → propagates to DispatcherServlet
      │
      ▼
GlobalExceptionHandler.handleServiceException()
      │
      ▼ returns
HTTP 503 {"code":"SERVICE_UNAVAILABLE","message":"DB unavailable"}
```

Each layer re-wraps with a domain-appropriate exception type, preserving the cause. The original `SQLException` is accessible via `getCause().getCause()`.

---

### 12.2 `finally` vs `try-with-resources` Resource Race

**Scenario:** Old code releases a connection in `finally`, but `close()` can throw.

```java
// Old style — close() exception in finally can hide the try-block exception
Connection conn = dataSource.getConnection();
try {
    conn.prepareStatement("BAD SQL").execute(); // throws SQLException A
} finally {
    conn.close(); // throws SQLException B → SQLException A is lost forever
}

// try-with-resources — B is suppressed, A propagates
try (Connection conn = dataSource.getConnection()) {
    conn.prepareStatement("BAD SQL").execute(); // throws A
    // conn.close() throws B → B added as suppressed to A
}
// catch (SQLException A) { A.getSuppressed()[0] is B }
```

**Takeaway:** Always prefer `try-with-resources` for any `AutoCloseable`. It handles the exception suppression correctly where manual `finally` silently drops exceptions.

---

### 12.3 `@Transactional` Self-Invocation and Rollback

**Scenario:** An internal method is annotated `@Transactional` but called from the same bean.

```java
@Service
public class PaymentService {

    public void processOrder(Order order) {
        chargeCard(order); // NOT proxied — direct call bypasses Spring's proxy
    }

    @Transactional
    public void chargeCard(Order order) {
        paymentRepo.save(new Payment(order));
        throw new RuntimeException("Payment gateway down");
        // Transaction does NOT roll back because the proxy was never invoked
    }
}
```

Because Spring's `@Transactional` works via proxy, calling `chargeCard()` from within the same class bypasses the proxy — no transaction is started, no rollback occurs. The `RuntimeException` propagates to `processOrder()` but there is no transaction to roll back.

**Fix:** Inject the bean into itself (Spring 4.3+), or extract `chargeCard()` into a separate `@Service`.

---

## 13. Interview Questions — Standard

**Q1: What is the difference between `Error` and `Exception`?**
`Error` represents unrecoverable JVM-level conditions (`OutOfMemoryError`, `StackOverflowError`). Applications should not catch Errors except at a top-level boundary. `Exception` represents application-level conditions — either anticipated recoverable failures (checked) or programming bugs (unchecked `RuntimeException`).

**Q2: What is the difference between checked and unchecked exceptions?**
Checked exceptions extend `Exception` (but not `RuntimeException`) and must be declared in `throws` or caught — the compiler enforces this. Unchecked exceptions extend `RuntimeException` (or `Error`) and require no compiler enforcement. Checked exceptions are used for conditions the caller is expected to handle (I/O, network); unchecked for programming errors (null, illegal args). Modern frameworks prefer unchecked exceptions.

**Q3: What does `finally` guarantee?**
The `finally` block always executes after the `try` block exits, whether normally, via an exception, or via a `return` statement. Exceptions: `System.exit()`, JVM crash, or `Thread.stop()`. A `throw` or `return` inside `finally` itself will suppress any exception propagating from the `try` block — a critical gotcha.

**Q4: What is `try-with-resources` and what interface must a resource implement?**
`try-with-resources` (Java 7+) automatically calls `close()` on declared resources when the block exits. Resources must implement `java.lang.AutoCloseable`. Resources are closed in reverse declaration order. If both the body and `close()` throw, the close exception is suppressed and attached to the body exception.

**Q5: What is exception chaining and why is it important?**
Exception chaining is wrapping a caught exception inside a new one and passing it as the `cause` parameter. It preserves the full root cause in the stack trace while allowing higher layers to use domain-appropriate exception types. Without it, the original error is lost, making debugging in production very difficult.

**Q6: Why should you never catch `InterruptedException` and do nothing with it?**
`InterruptedException` is thrown when a thread is interrupted (e.g., during `Thread.sleep()`, `wait()`, or blocking I/O). When caught, the thread's interrupted status is cleared. Swallowing it without calling `Thread.currentThread().interrupt()` permanently prevents the thread from ever responding to future interruption requests, breaking cooperative thread termination.

**Q7: Can you catch `StackOverflowError`?**
Yes — `StackOverflowError` extends `Error` extends `Throwable`. You can `catch (StackOverflowError e)`. However, the stack is in a degraded state at that point. Catching it is only safe at a very high-level boundary (e.g., plugin sandbox) to isolate bad plugins without crashing the whole JVM. Doing so in normal code is almost always a bug.

**Q8: What are suppressed exceptions?**
Suppressed exceptions (Java 7+) are exceptions that would have been thrown but were suppressed to allow another exception to propagate. `try-with-resources` uses this: if the body throws exception A and `close()` throws exception B, A propagates and B is attached to A via `addSuppressed()`. Retrieved with `getSuppressed()`.

---

## 14. Interview Questions — Tricky / Scenario-Based

**Q1: What happens if both `try` and `finally` blocks have `return` statements?**
The `finally` block's `return` wins and the `try` block's `return` value is discarded. Any exception thrown in `try` is also silently discarded. Example: `try { return 1; } finally { return 2; }` returns `2`. This is one of the worst gotchas in Java — never `return` from `finally`.

**Q2: A `@Transactional` method catches an exception internally and doesn't re-throw. Does the transaction roll back?**
No. Spring's transaction infrastructure only rolls back if the exception propagates out of the `@Transactional` proxy. If the exception is caught and swallowed inside the method, Spring never sees it and commits the transaction normally — even if the business operation was only partially complete. Always re-throw (as unchecked) or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` if you must catch without throwing.

**Q3: What happens when a checked exception is thrown inside a lambda?**
Lambdas implementing standard functional interfaces (`Runnable`, `Consumer`, `Function`, etc.) cannot throw checked exceptions — those interfaces don't declare `throws`. You either wrap in a `RuntimeException`, or define your own `@FunctionalInterface` that declares `throws`:

```java
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}
```

Otherwise, the common ugly pattern `catch (Exception e) { throw new RuntimeException(e); }` inside the lambda is required.

**Q4: You have `catch (IOException | SQLException e)`. Can you reassign `e` inside the catch block?**
No. In a multi-catch block, the caught exception variable `e` is implicitly `final` (effectively final). You cannot assign a different value to it. This is because the compiler needs to know the precise type for type inference — since `e` can be either `IOException` or `SQLException`, reassignment is disallowed.

**Q5: What is the difference between `throw new RuntimeException(e)` and `throw new RuntimeException(e.getMessage())`?**
A significant difference: `new RuntimeException(e)` wraps the original exception as the cause — the full original stack trace is preserved and accessible via `getCause()`. `new RuntimeException(e.getMessage())` creates a new exception with only the message string — the original type, stack trace, and cause chain are all **permanently lost**. Always use the `Throwable cause` constructor form when wrapping.

**Q6: Can a subclass override a method and throw a broader checked exception?**
No. The Liskov Substitution Principle requires that an overriding method cannot throw new or broader checked exceptions than the parent method. It can throw narrower checked exceptions (subtypes) or unchecked exceptions freely, or declare no exceptions at all. This is enforced by the compiler.

---

## 15. Quick Revision Summary

- **Throwable → Error (unchecked, unrecoverable) or Exception (checked or unchecked RuntimeException).**
- Checked exceptions must be caught or declared in `throws`; unchecked need neither.
- Modern Java: prefer **unchecked** (RuntimeException) for application errors; Spring, Hibernate, JPA all do this.
- `finally` always runs — but never `throw` or `return` from it (swallows propagating exceptions).
- **`try-with-resources`** is always preferred over manual `finally` cleanup for `AutoCloseable` resources.
- Multi-catch: `catch (A | B e)` — `e` is effectively final; types cannot be in an inheritance relationship.
- **Always chain exceptions**: `throw new ServiceException("msg", causeException)` — never lose the root cause.
- Log exceptions once, at the handling boundary — not at every re-throw.
- `@Transactional` only rolls back on `RuntimeException`/`Error` by default — use `rollbackFor` for checked.
- `InterruptedException`: always restore thread interrupted status with `Thread.currentThread().interrupt()`.
- `Class.forName()` triggers class initialization; `ClassLoader.loadClass()` does not.
- Suppressed exceptions: `try-with-resources` attaches `close()` failures to the body exception — never lost.
- Never use exceptions for normal control flow in hot paths — constructing an exception is expensive (stack trace capture).
