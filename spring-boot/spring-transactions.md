# Spring Transactions — Complete Guide

> **Why this matters in interviews:** Spring Transactions is one of the most frequently tested topics for mid-to-senior Java/Spring engineers. Interviewers probe not just *what* `@Transactional` does, but *how it works under the hood*, *what can silently break it*, and *how you'd handle edge cases in production*. A weak answer lists propagation types. A strong answer explains proxy mechanics, classifies pitfalls by root cause, and compares locking strategies with concrete examples.

---

## Table of Contents

1. [Core Concepts — What is a Transaction?](#1-core-concepts)
2. [Spring Transaction Abstraction](#2-spring-transaction-abstraction)
3. [Declarative Transactions — @Transactional](#3-declarative-transactions)
4. [Transaction Propagation](#4-transaction-propagation)
5. [Transaction Isolation Levels](#5-transaction-isolation-levels)
6. [Rollback Rules](#6-rollback-rules)
7. [Read-Only Transactions](#7-read-only-transactions)
8. [Programmatic Transaction Management](#8-programmatic-transaction-management)
9. [Locking Mechanisms — Optimistic vs Pessimistic](#9-locking-mechanisms)
10. [Distributed Transactions & Two-Phase Commit](#10-distributed-transactions)
11. [Transaction Internals — How the Proxy Works](#11-transaction-internals)
12. [Production Use Cases](#12-production-use-cases)
13. [Common Pitfalls](#13-common-pitfalls)
14. [Miscellaneous Topics](#14-miscellaneous-topics)
15. [Interview Questions & Answers](#15-interview-questions)

---

## 1. Core Concepts

### 1.1 ACID Properties

| Property | Meaning |
|---|---|
| **Atomicity** | All operations in a transaction succeed or all are rolled back — never partial. |
| **Consistency** | The database moves from one valid state to another valid state. Constraints, triggers, and rules are never violated. |
| **Isolation** | Concurrent transactions do not interfere with each other. One transaction's intermediate state is invisible to others (to the degree the isolation level permits). |
| **Durability** | Once committed, the data survives crashes, power failures, etc. (via WAL / redo logs). |

### 1.2 Why Transactions Are Hard

- **Concurrency** — multiple threads/processes hit the same rows simultaneously.
- **Partial failures** — a network blip mid-operation leaves the DB in an inconsistent state.
- **Cascading effects** — one service's rollback must undo work done by downstream services it called.

---

## 2. Spring Transaction Abstraction

Spring provides a **platform-agnostic transaction management layer** so the same `@Transactional` annotation works regardless of the underlying technology (JDBC, JPA/Hibernate, JTA, R2DBC, etc.).

### 2.1 Key Interfaces

```
PlatformTransactionManager          (imperative stack)
  └── DataSourceTransactionManager  (plain JDBC)
  └── JpaTransactionManager         (JPA / Hibernate)
  └── JtaTransactionManager         (XA / distributed)

ReactiveTransactionManager          (reactive stack — R2DBC)
```

**`PlatformTransactionManager`** — the central SPI:
```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

**`TransactionDefinition`** — carries the metadata you set on `@Transactional`:
- Propagation behavior
- Isolation level
- Timeout
- Read-only flag
- Transaction name

**`TransactionStatus`** — represents the current running transaction handle:
- `isNewTransaction()` — is this the outermost transaction or a participant?
- `setRollbackOnly()` — mark for rollback without throwing
- `hasSavepoint()` — is there a nested savepoint?

### 2.2 Auto-Configuration

In a Spring Boot app, `DataSourceTransactionManagerAutoConfiguration` (for JDBC) or `JpaBaseConfiguration` (for JPA) automatically registers the correct `PlatformTransactionManager` bean. You rarely need to declare one manually.

---

## 3. Declarative Transactions — @Transactional

### 3.1 Basic Usage

```java
@Service
public class OrderService {

    @Transactional                      // applies default settings
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryService.deductStock(order.getItems());
        paymentService.charge(order.getPayment());
        // if any line above throws RuntimeException → full rollback
    }
}
```

### 3.2 Full Annotation Reference

```java
@Transactional(
    propagation    = Propagation.REQUIRED,       // default
    isolation      = Isolation.DEFAULT,           // default (DB default)
    timeout        = 30,                          // seconds; -1 = unlimited
    readOnly       = false,                       // default
    rollbackFor    = { Exception.class },         // override rollback rules
    noRollbackFor  = { NotFoundException.class }  // suppress rollback
)
```

### 3.3 Where to Place @Transactional

| Placement | Effect |
|---|---|
| **Method** | Only that method is transactional. Most granular. |
| **Class** | All public methods are transactional with these settings. Methods can override. |
| **Interface** | Works, but not recommended — Spring proxies the class, not the interface; Spring Data repos use this as a convention. |

> **Rule:** Prefer method-level annotation for explicitness. Class-level is fine for repository/service classes where nearly everything needs a transaction.

### 3.4 Default Rollback Behavior

- **Rolls back** on unchecked exceptions (`RuntimeException` and its subclasses, and `Error`).
- **Does NOT roll back** on checked exceptions (`Exception` subclasses that are not `RuntimeException`).

This is a deliberate design decision: checked exceptions represent expected business conditions (e.g., `InsufficientFundsException`), so Spring assumes you want to handle them and commit. Change this with `rollbackFor`.

---

## 4. Transaction Propagation

Propagation defines **what happens to the transaction when a transactional method calls another transactional method**.

### 4.1 All Seven Propagation Types

```
Caller has:      REQUIRED  REQUIRES_NEW  SUPPORTS  NOT_SUPPORTED  MANDATORY  NEVER  NESTED
─────────────────────────────────────────────────────────────────────────────────────────────
No transaction   Creates   Creates       None      None           Exception  None   Creates
Has transaction  Joins     Suspends+New  Joins     Suspends+None  Joins      Exc.   Savepoint
```

---

#### `REQUIRED` (default)

- **Has existing transaction?** → Join it. Same physical transaction.
- **No transaction?** → Start a new one.

```java
// OuterService
@Transactional(propagation = Propagation.REQUIRED)
public void outer() {
    innerService.inner();  // joins the same transaction
}

// InnerService
@Transactional(propagation = Propagation.REQUIRED)
public void inner() {
    // same connection, same transaction — if inner throws and is caught in outer,
    // the transaction is still marked rollback-only!
}
```

**Key trap:** If `inner()` throws and the exception is caught in `outer()`, the transaction is still **marked rollback-only**. Attempting to commit will throw `UnexpectedRollbackException`.

---

#### `REQUIRES_NEW`

- **Always** starts a fresh transaction, suspending any existing one.
- The two transactions are **completely independent** — outer committing/rolling back does NOT affect inner, and vice versa.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(AuditEvent event) {
    // must always persist, even if the outer business transaction rolls back
    auditRepository.save(event);
}
```

**Production use:** Audit logging, email notification records, outbox pattern entries that must survive the parent transaction's rollback.

**Warning:** Each `REQUIRES_NEW` call checks out a separate connection from the pool. Nested calls can exhaust the connection pool.

---

#### `SUPPORTS`

- **Has transaction?** → Join it.
- **No transaction?** → Run without any transaction (plain JDBC call).

```java
@Transactional(propagation = Propagation.SUPPORTS)
public List<Product> findAll() {
    // works fine with or without a surrounding transaction
}
```

---

#### `NOT_SUPPORTED`

- **Has transaction?** → Suspend it, run non-transactionally, resume after.
- **No transaction?** → Run non-transactionally.

Used when calling legacy code or operations that must never be wrapped in a transaction (e.g., some stored procedures).

---

#### `MANDATORY`

- **Has transaction?** → Join it.
- **No transaction?** → Throw `IllegalTransactionStateException`.

Used to enforce that a method is always called from within a transactional context. Good for helper methods that must not be called standalone.

```java
@Transactional(propagation = Propagation.MANDATORY)
public void validateAndApply(Voucher v) {
    // calling this from non-transactional code is a programming error
}
```

---

#### `NEVER`

- **Has transaction?** → Throw `IllegalTransactionStateException`.
- **No transaction?** → Execute normally.

Used for operations that must never run inside a transaction (e.g., long-running batch reads that intentionally avoid locking).

---

#### `NESTED`

- **Has transaction?** → Create a **savepoint** inside the existing transaction.
  - If the nested portion fails → roll back to savepoint only (outer continues).
  - If the outer fails → everything (including inner commits) rolls back.
- **No transaction?** → Behave like `REQUIRED`.

```java
@Transactional(propagation = Propagation.NESTED)
public void processLineItem(LineItem item) {
    // if this fails, only this line item is rolled back — outer loop continues
}
```

**Requires savepoint support** — works with JDBC `DataSourceTransactionManager`. **Does NOT work with JPA/Hibernate** (JPA spec has no savepoint API). For JPA, use `REQUIRES_NEW` as an alternative.

---

### 4.2 Propagation Decision Tree

```
Do I need the operation to always commit independently of caller?
  YES → REQUIRES_NEW

Am I writing a helper that should fail if called outside a transaction?
  YES → MANDATORY

Does this operation intentionally bypass transactions (legacy/batch)?
  YES → NOT_SUPPORTED or NEVER

Do I want partial rollback on failure but still within the same outer TX?
  YES → NESTED (JDBC only)

Default behavior (join if exists, create if not)?
  → REQUIRED
```

---

## 5. Transaction Isolation Levels

> Cross-reference: see `databases/transaction-isolation-levels.md` for full anomaly diagrams.

### 5.1 Quick Reference

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible | Highest |
| `READ_COMMITTED` | Prevented | Possible | Possible | High (PostgreSQL default) |
| `REPEATABLE_READ` | Prevented | Prevented | Possible | Medium (MySQL InnoDB default) |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | Lowest |
| `DEFAULT` | Uses DB default | — | — | — |

### 5.2 Setting Isolation in Spring

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processPayment(PaymentRequest req) { ... }
```

### 5.3 Production Guidance

| Scenario | Recommended Level |
|---|---|
| Simple CRUD, reporting, no concurrency sensitivity | `READ_COMMITTED` |
| Financial calculations requiring consistent balance reads | `REPEATABLE_READ` |
| Seat reservation, ticket booking (phantom prevention critical) | `SERIALIZABLE` or pessimistic locking |
| High-throughput analytics (tolerate stale reads) | `READ_UNCOMMITTED` (rare; mostly read replicas) |

---

## 6. Rollback Rules

### 6.1 Default Rules

```
RuntimeException & subclasses  → ROLLBACK
Error & subclasses             → ROLLBACK
Checked exceptions             → COMMIT (no rollback)
```

### 6.2 Custom Rules

```java
// Roll back on checked exception
@Transactional(rollbackFor = IOException.class)
public void uploadFile(MultipartFile file) throws IOException { ... }

// Roll back on any Exception (checked or not)
@Transactional(rollbackFor = Exception.class)
public void criticalOperation() throws Exception { ... }

// Never roll back on this specific exception
@Transactional(noRollbackFor = OptimisticLockingFailureException.class)
public void retryableUpdate(Entity e) { ... }
```

### 6.3 Programmatic Rollback Without Exception

```java
@Transactional
public void conditionalRollback(TransactionStatus status) {
    // ... do work ...
    if (someBusinessCondition) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        // transaction will roll back when the method returns, even without an exception
    }
}
```

### 6.4 The "swallowed exception" trap

```java
@Transactional
public void outer() {
    try {
        inner();  // inner throws RuntimeException → marks TX rollback-only
    } catch (RuntimeException e) {
        log.warn("caught", e);  // exception swallowed
    }
    // method returns normally → Spring tries to commit
    // → throws UnexpectedRollbackException because TX is rollback-only
}
```

**Fix:** Either let the exception propagate, use `REQUIRES_NEW` for `inner()`, or check `TransactionAspectSupport.currentTransactionStatus().isRollbackOnly()`.

---

## 7. Read-Only Transactions

```java
@Transactional(readOnly = true)
public List<Order> findOrdersByCustomer(Long customerId) {
    return orderRepository.findByCustomerId(customerId);
}
```

### 7.1 What `readOnly = true` Actually Does

| Effect | Detail |
|---|---|
| **Hibernate optimization** | Skips dirty-checking on loaded entities (no snapshot comparison at flush time → faster). |
| **Flush mode** | Sets `FlushMode.MANUAL` — Hibernate will not auto-flush before queries. |
| **JDBC hint** | Some drivers/pools set the connection to read-only mode (can route to read replica). |
| **DB hint** | Some DBs (PostgreSQL) can use this hint to avoid acquiring write locks on scanned rows. |

> `readOnly = true` does NOT enforce that writes are impossible at the application level. It's an optimization hint, not a constraint.

### 7.2 When to Use

- All `findBy*`, `getAll*`, reporting queries.
- Eliminates dirty-checking overhead for entities loaded in the session — significant performance gain when loading many entities.

---

## 8. Programmatic Transaction Management

Use when you need **fine-grained control** that declarative annotations cannot express (e.g., conditional transactions, transactions inside lambdas, dynamic propagation).

### 8.1 TransactionTemplate (Preferred)

```java
@Service
public class PaymentService {

    private final TransactionTemplate txTemplate;

    public PaymentService(PlatformTransactionManager txManager) {
        this.txTemplate = new TransactionTemplate(txManager);
        this.txTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        this.txTemplate.setTimeout(10);
    }

    public PaymentResult processPayment(PaymentRequest req) {
        return txTemplate.execute(status -> {
            // all code here runs in a transaction
            Account account = accountRepository.findById(req.getAccountId())
                .orElseThrow();
            account.debit(req.getAmount());
            accountRepository.save(account);
            return new PaymentResult("SUCCESS");
        });
        // auto-commit on normal return; auto-rollback on exception
    }
}
```

### 8.2 PlatformTransactionManager Directly (Lower Level)

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);

TransactionStatus status = txManager.getTransaction(def);
try {
    // ... work ...
    txManager.commit(status);
} catch (Exception e) {
    txManager.rollback(status);
    throw e;
}
```

### 8.3 When to Choose Programmatic Over Declarative

| Use Programmatic When | Use Declarative When |
|---|---|
| Transaction logic inside a lambda/stream | Standard service method boundary |
| Conditional transaction (start only if X) | Method always needs a transaction |
| Fine-grained per-operation control | Consistent policy across a class |
| Testing transaction behavior in isolation | Production service code |

---

## 9. Locking Mechanisms

### 9.1 Optimistic Locking

**Philosophy:** Assume conflicts are rare. Let multiple transactions read freely. Detect conflicts at commit time.

**How it works:**
1. Each row has a `version` column (integer or timestamp).
2. When a transaction reads a row, it stores the current version.
3. When updating, the `WHERE` clause includes the original version.
4. If another transaction updated the row (version changed), the `UPDATE` affects 0 rows.
5. JPA/Hibernate detects 0 rows affected → throws `OptimisticLockException`.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version                          // ← the magic annotation
    private int version;
}
```

**JPA generates:**
```sql
UPDATE product SET stock = 9, version = 2
WHERE id = 1 AND version = 1;   -- fails if another TX already bumped version
```

**Handling the exception:**

```java
@Transactional
public void deductStock(Long productId, int qty) {
    try {
        Product p = productRepository.findById(productId).orElseThrow();
        p.setStock(p.getStock() - qty);
        productRepository.save(p);
    } catch (OptimisticLockingFailureException e) {
        // retry logic or return conflict response
        throw new StockConflictException("Concurrent update detected, please retry");
    }
}
```

**Lock modes for read operations:**

```java
// Force version check even on read (protects against lost update)
Product p = em.find(Product.class, id, LockModeType.OPTIMISTIC);

// Increment version on read — prevents others from committing any write to this row
Product p = em.find(Product.class, id, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
```

**When to use:**
- Low contention scenarios (most reads, rare writes to same row).
- Web apps where users edit their own data.
- Read-heavy systems where locking would be a bottleneck.
- Microservices that can retry on conflict.

---

### 9.2 Pessimistic Locking

**Philosophy:** Assume conflicts are likely. Lock the row on read — prevent others from modifying it until you're done.

**Lock types:**

| Lock Type | SQL Equivalent | Others Can Read? | Others Can Write? |
|---|---|---|---|
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` | Yes | No |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` | No (most DBs) | No |
| `PESSIMISTIC_FORCE_INCREMENT` | `SELECT ... FOR UPDATE` + version bump | No | No |

**Usage in Spring Data JPA:**

```java
// Repository method
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdForUpdate(@Param("id") Long id);

// Service
@Transactional
public void reserveStock(Long productId, int qty) {
    Product p = productRepository.findByIdForUpdate(productId)
        .orElseThrow();
    if (p.getStock() < qty) {
        throw new InsufficientStockException();
    }
    p.setStock(p.getStock() - qty);
}
```

**Usage with EntityManager:**

```java
@Transactional
public void updateBalance(Long accountId, BigDecimal amount) {
    Account account = em.find(Account.class, accountId, LockModeType.PESSIMISTIC_WRITE);
    account.credit(amount);
}
```

**Timeout for pessimistic locks:**

```java
Map<String, Object> hints = Map.of("javax.persistence.lock.timeout", 5000); // ms
Account a = em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE, hints);
```

**When to use:**
- High contention on specific rows (e.g., bank account balance, seat reservation).
- Financial transactions where the cost of a conflict/retry is high.
- Short transactions that release locks quickly.
- You cannot retry (e.g., user is waiting synchronously).

---

### 9.3 Optimistic vs Pessimistic — Decision Matrix

| Factor | Optimistic | Pessimistic |
|---|---|---|
| **Contention** | Low | High |
| **Read/write ratio** | Read-heavy | Write-heavy on same rows |
| **Retry feasibility** | Easy | Difficult or unacceptable |
| **Performance (no contention)** | Better | Worse (lock overhead) |
| **Performance (high contention)** | Worse (many retries) | Better (queue writers) |
| **Deadlock risk** | None | Yes (if multiple locks acquired) |
| **Distributed systems** | Natural fit | Hard (distributed locks needed) |

---

### 9.4 Deadlock Prevention with Pessimistic Locking

Always acquire multiple locks **in the same order** across all transactions:

```java
// SAFE — always lock account A before account B (order by ID)
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Long firstId  = Math.min(fromId, toId);
    Long secondId = Math.max(fromId, toId);

    Account first  = accountRepo.findByIdForUpdate(firstId).orElseThrow();
    Account second = accountRepo.findByIdForUpdate(secondId).orElseThrow();

    // now perform transfer using correct references
}
```

---

## 10. Distributed Transactions

### 10.1 The Problem

When a single business operation touches **multiple databases or message brokers**, ACID across them requires coordination. A failure between two commits can leave one system committed and another rolled back.

### 10.2 XA / Two-Phase Commit (2PC)

The classical solution. A **Transaction Coordinator** orchestrates:

1. **Phase 1 (Prepare):** Coordinator asks all participants: "Can you commit?"
2. **Phase 2 (Commit/Abort):** If all say yes → coordinator sends commit. If any says no → all rollback.

```
         ┌─────────────────┐
         │  TX Coordinator │
         └────────┬────────┘
          ┌───────┴───────┐
          ▼               ▼
    ┌──────────┐    ┌──────────┐
    │  DB-1    │    │  DB-2    │
    └──────────┘    └──────────┘
```

**Spring Boot + JTA (Atomikos / Bitronix):**

```java
@Bean
public UserTransactionManager atomikosTransactionManager() {
    UserTransactionManager utm = new UserTransactionManager();
    utm.setForceShutdown(false);
    return utm;
}

// application.properties
spring.jta.atomikos.datasource.xa-data-source-class-name=com.mysql.cj.jdbc.MysqlXADataSource
```

**Downsides of XA:**
- Blocking protocol — coordinator failure leaves participants in-doubt.
- Significant performance overhead.
- Rarely used in modern microservices.

### 10.3 SAGA Pattern (Modern Approach)

Break a distributed transaction into a **sequence of local transactions**, each publishing an event/message. On failure, execute **compensating transactions**.

**Choreography-based SAGA:**
```
OrderService    → publishes OrderCreated
InventoryService → listens → reserves stock → publishes StockReserved
PaymentService  → listens → charges card  → publishes PaymentCharged
OrderService    → listens → confirms order

On failure:
PaymentService  → publishes PaymentFailed
InventoryService → listens → releases stock
OrderService    → listens → cancels order
```

**Orchestration-based SAGA:**
```
OrderSaga (Orchestrator) → calls InventoryService
                         → calls PaymentService
                         → on failure: calls compensating endpoints
```

**Outbox Pattern** (ensures at-least-once delivery with SAGA):

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    // write the event to outbox table in SAME transaction
    outboxRepository.save(new OutboxEvent("ORDER_CREATED", order.getId(), toJson(order)));
    // a separate poller reads outbox and publishes to Kafka/RabbitMQ
}
```

---

## 11. Transaction Internals — How the Proxy Works

### 11.1 The AOP Proxy Mechanism

Spring's `@Transactional` is implemented via **AOP (Aspect-Oriented Programming)**. When a `@Transactional` bean is injected, Spring wraps it in a **proxy object**.

```
Client Code
    │
    ▼
┌─────────────────────────────────┐
│  Spring Proxy (TX Interceptor)  │
│  1. Begin / Join transaction    │
│  2. ──────────────────────────► │──► Target Bean Method
│  3. Commit or Rollback          │◄── (actual business logic)
└─────────────────────────────────┘
```

Two proxy types:
- **JDK Dynamic Proxy** — used when the bean implements an interface. Proxy implements the same interface.
- **CGLIB Proxy** — used when the bean does NOT implement an interface (or `proxyTargetClass=true`). Proxy subclasses the bean.

### 11.2 The Self-Invocation Problem

Because the proxy sits **outside** the target bean, a method calling another method **on the same object** bypasses the proxy entirely:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        // ...
        this.sendConfirmationEmail(order);  // ← calls 'this', not the proxy!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendConfirmationEmail(Order order) {
        // This @Transactional is IGNORED — running in placeOrder's transaction
    }
}
```

**Fixes:**

1. **Inject self (not recommended, can cause circular dependency issues):**
   ```java
   @Autowired
   private OrderService self;  // Spring injects the proxy
   self.sendConfirmationEmail(order);
   ```

2. **Extract to a separate bean (cleanest):**
   ```java
   @Service
   public class EmailService {
       @Transactional(propagation = Propagation.REQUIRES_NEW)
       public void sendConfirmationEmail(Order order) { ... }
   }
   ```

3. **Use `AopContext.currentProxy()` (requires `exposeProxy = true`):**
   ```java
   @EnableAspectJAutoProxy(exposeProxy = true)  // on config class

   ((OrderService) AopContext.currentProxy()).sendConfirmationEmail(order);
   ```

### 11.3 Why Private Methods Don't Work

CGLIB proxies work by **subclassing** the target class. Private methods cannot be overridden by a subclass → Spring cannot intercept them. `@Transactional` on a private method is silently ignored.

```java
@Transactional  // ← SILENTLY IGNORED for private methods
private void doInternalWork() { }
```

Same issue: `@Transactional` on `final` methods or `final` classes (CGLIB cannot subclass them).

---

## 12. Production Use Cases

### 12.1 Financial Transfer — Pessimistic Locking

```java
@Transactional(isolation = Isolation.READ_COMMITTED, timeout = 30)
public TransferResult transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    // Always lock lower ID first to prevent deadlocks
    Long firstId  = Math.min(fromAccountId, toAccountId);
    Long secondId = Math.max(fromAccountId, toAccountId);

    Account from = accountRepo.findByIdForUpdate(firstId).orElseThrow();
    Account to   = accountRepo.findByIdForUpdate(secondId).orElseThrow();

    // Ensure correct assignment after ordering
    Account source = from.getId().equals(fromAccountId) ? from : to;
    Account target = from.getId().equals(toAccountId)   ? from : to;

    if (source.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }

    source.debit(amount);
    target.credit(amount);

    return TransferResult.success(source.getBalance());
}
```

### 12.2 E-Commerce Order Placement — Optimistic Locking

```java
@Transactional
public Order placeOrder(OrderRequest req) {
    // optimistic locking via @Version on Product entity
    for (OrderItem item : req.getItems()) {
        Product product = productRepo.findById(item.getProductId()).orElseThrow();
        if (product.getStock() < item.getQty()) {
            throw new OutOfStockException(product.getName());
        }
        product.setStock(product.getStock() - item.getQty());
        // OptimisticLockException thrown here if concurrent update occurred
    }
    Order order = new Order(req);
    return orderRepo.save(order);
}
// Caller catches OptimisticLockingFailureException and retries
```

### 12.3 Audit Logging — REQUIRES_NEW

```java
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action, Long entityId, String details) {
        // Independent transaction — persists even if caller rolls back
        AuditLog entry = new AuditLog(action, entityId, details, Instant.now());
        auditLogRepo.save(entry);
    }
}
```

### 12.4 Batch Processing — Chunk-Level Transactions

```java
// Spring Batch style — commit every N items rather than one giant transaction
@Transactional
public void processBatch(List<Record> records) {
    int batchSize = 100;
    for (int i = 0; i < records.size(); i += batchSize) {
        List<Record> chunk = records.subList(i, Math.min(i + batchSize, records.size()));
        processChunk(chunk);
        entityManager.flush();
        entityManager.clear();  // prevent memory overflow from first-level cache
    }
}
```

### 12.5 Event-Driven Outbox Pattern

```java
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    Order order = new Order(cmd);
    orderRepo.save(order);

    // Atomically write event to outbox — same DB transaction
    OutboxEvent event = OutboxEvent.of("order.created", order.getId(), serialize(order));
    outboxRepo.save(event);

    // Separate process (CDC / polling) reads outbox and publishes to Kafka
}
```

---

## 13. Common Pitfalls

### 13.1 Self-Invocation

**Problem:** `@Transactional` bypassed because the method is called via `this`.  
**Fix:** Extract to a separate Spring-managed bean.

### 13.2 Exception Swallowing

**Problem:** Catching a `RuntimeException` inside a `@Transactional` method → `UnexpectedRollbackException` at commit.  
**Fix:** Don't catch and swallow exceptions in transactional methods; let them propagate; or use `REQUIRES_NEW` if isolation is needed.

### 13.3 Transaction Not Started (Non-Spring-Managed Bean)

**Problem:** `@Transactional` on a bean that is `new`-ed directly (not injected by Spring) → proxy is never created → no transaction.  
**Fix:** Always inject beans via Spring context (`@Autowired`, constructor injection, etc.).

### 13.4 Lazy Loading Outside Transaction

**Problem:** Entity with a lazy collection loaded after the transaction closes → `LazyInitializationException`.  
**Fix options:**
- Extend transaction scope (keep transaction open while view renders — Open Session in View — **generally bad practice**).
- Use `@EntityGraph` or `JOIN FETCH` to eagerly load needed data.
- Use DTOs and project only required data.
- Use `@Transactional` on the calling method to keep the session alive.

### 13.5 Long-Running Transactions Holding Locks

**Problem:** A transaction that calls external REST APIs, sends emails, or does heavy computation holds DB locks the entire time → contention, timeouts.  
**Fix:** 
- Perform all DB work first, commit, then do I/O operations.
- Use `@Transactional(timeout = N)` to cap long-running transactions.
- Break the operation: read outside transaction, process, write in a short transaction.

### 13.6 Connection Pool Exhaustion with REQUIRES_NEW

**Problem:** Each `REQUIRES_NEW` suspends the current connection and acquires a new one. Nested calls can exhaust the pool.  
**Fix:** Size the connection pool appropriately. Avoid deeply nested `REQUIRES_NEW` chains.

### 13.7 Incorrect Rollback on Checked Exceptions

**Problem:** A service method throws a checked exception (e.g., `IOException`) but the transaction commits because checked exceptions don't trigger rollback by default.  
**Fix:** Use `@Transactional(rollbackFor = IOException.class)` or wrap in a `RuntimeException`.

### 13.8 @Transactional on Private / Final Methods

**Problem:** Spring silently ignores `@Transactional` on private or final methods (CGLIB cannot proxy them).  
**Fix:** Make the method `public` and non-final, or (better) refactor so the transaction boundary is on a public method.

### 13.9 Multiple Transaction Managers — Wrong One Used

**Problem:** App has JpaTransactionManager and JmsTransactionManager. `@Transactional` with no qualifier uses the primary one — JMS operations don't participate.  
**Fix:** Use the `transactionManager` attribute:
```java
@Transactional("jmsTransactionManager")
public void sendMessage(String msg) { ... }
```

### 13.10 Optimistic Lock Version Not Exposed in API

**Problem:** Client fetches an entity, edits it, posts back without the version — Hibernate has no version to compare against → silent data overwrite (last-write-wins).  
**Fix:** Always include the `@Version` field in DTOs and API payloads when using optimistic locking.

---

## 14. Miscellaneous Topics

### 14.1 @TransactionalEventListener

Standard `@EventListener` runs in the publisher's transaction. `@TransactionalEventListener` binds execution to a transaction phase:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    // runs only if the publishing transaction committed
    emailService.sendOrderConfirmation(event.getOrderId());
}
```

**Phases:**
| Phase | When it fires |
|---|---|
| `AFTER_COMMIT` (default) | After the transaction commits. |
| `AFTER_ROLLBACK` | After the transaction rolls back. |
| `AFTER_COMPLETION` | After commit or rollback. |
| `BEFORE_COMMIT` | Just before commit (still inside transaction). |

> If no active transaction exists, the event is **discarded by default**. Set `fallbackExecution = true` to fire anyway.

### 14.2 @Transactional on @Repository vs @Service

Spring Data repositories already have `@Transactional` applied by default (read-only on queries, read-write on mutations). When you call a repository from a service that has its own `@Transactional`, the repository method **joins the service's transaction** (REQUIRED propagation). You don't need to add `@Transactional` to repositories yourself.

### 14.3 Flush Modes

| Flush Mode | Behavior |
|---|---|
| `AUTO` (default) | Hibernate flushes before query execution to ensure the query sees pending changes. |
| `COMMIT` | Hibernate flushes only at transaction commit. More efficient but changes invisible to queries within same TX. |
| `MANUAL` | Application controls when flush happens. Used with `readOnly = true`. |
| `ALWAYS` | Flush before every query. Very safe, potentially slow. |

```java
entityManager.setFlushMode(FlushModeType.COMMIT);
```

### 14.4 Transaction Synchronization

Spring's `TransactionSynchronizationManager` allows registering callbacks tied to transaction lifecycle:

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCommit() {
        // e.g., invalidate cache after commit
        cache.evict(key);
    }
});
```

### 14.5 Test Transactions — @Transactional in Tests

```java
@SpringBootTest
@Transactional                    // each test method runs in a transaction
class OrderServiceTest {

    @Test
    void testPlaceOrder() {
        // DB changes made here are automatically rolled back after the test
        // No need for cleanup logic
    }

    @Test
    @Commit                       // override — commit this test's changes
    void testAuditLog() { }

    @Test
    @Rollback(false)              // same as @Commit
    void testPersistence() { }
}
```

**Important:** If the service being tested uses `REQUIRES_NEW`, those inner transactions commit independently and will **not** be rolled back by the test framework — you need manual cleanup.

### 14.6 Reactive Transactions

```java
@Transactional  // uses ReactiveTransactionManager
public Mono<Order> placeOrder(OrderRequest req) {
    return orderRepository.save(new Order(req))
        .flatMap(order -> inventoryService.deductStock(order));
}
```

Reactive transactions use thread-local-free propagation via Project Reactor's `Context`. The `@Transactional` annotation works on reactive return types (`Mono`, `Flux`) if a `ReactiveTransactionManager` is configured (e.g., R2DBC).

### 14.7 Savepoints

Savepoints allow partial rollback within a transaction without aborting the entire transaction:

```java
@Transactional
public void processWithSavepoint() {
    Object savepoint = TransactionAspectSupport.currentTransactionStatus().createSavepoint();
    try {
        riskyOperation();
    } catch (Exception e) {
        TransactionAspectSupport.currentTransactionStatus().rollbackToSavepoint(savepoint);
        // outer transaction continues
    } finally {
        TransactionAspectSupport.currentTransactionStatus().releaseSavepoint(savepoint);
    }
}
```

This is the programmatic equivalent of `NESTED` propagation.

### 14.8 `spring.jpa.open-in-view`

- Default: **true** in Spring Boot (Open Session in View filter).
- Keeps the Hibernate `Session` open for the entire HTTP request lifecycle — allows lazy loading in view layer.
- **Downside:** DB connection held open for the full request duration → resource pressure; lazy loading in view makes hidden N+1 queries easy to miss.
- **Best practice for production:** Set `spring.jpa.open-in-view=false` and handle lazy loading explicitly.

---

## 15. Interview Questions

### 15.1 Core Mechanics

**Q: What does `@Transactional` actually do under the hood?**

> Spring wraps the bean in an AOP proxy (JDK dynamic proxy or CGLIB). When a transactional method is invoked through the proxy, the proxy calls `PlatformTransactionManager.getTransaction()` to begin or join a transaction, delegates to the real method, then calls `commit()` or `rollback()` based on whether an exception was thrown. The transaction context (connection, session) is stored in a `ThreadLocal` via `TransactionSynchronizationManager`.

---

**Q: Why doesn't `@Transactional` work when calling a method from within the same class?**

> Spring's transaction management is proxy-based. When `methodA()` calls `this.methodB()`, the call bypasses the proxy and goes directly to the target object. The interceptor never runs, so no transaction management is applied. The fix is to extract `methodB()` into a separate Spring bean and inject it.

---

**Q: What happens if you throw a checked exception inside a `@Transactional` method?**

> By default, Spring does NOT roll back for checked exceptions — the transaction commits. This is because Spring assumes checked exceptions represent expected business conditions. Override this with `rollbackFor = CheckedException.class`.

---

**Q: Explain `REQUIRED` vs `REQUIRES_NEW` propagation.**

> `REQUIRED` joins an existing transaction if one exists, or creates a new one if there isn't one — both methods share the same connection and unit of work. `REQUIRES_NEW` always starts a completely independent transaction, suspending the current one (if any). The two transactions can commit or roll back independently. Use `REQUIRES_NEW` when an operation (like audit logging) must always persist regardless of whether the calling transaction succeeds.

---

**Q: What is `UnexpectedRollbackException` and when does it occur?**

> When an inner method participating in a `REQUIRED` transaction throws a `RuntimeException`, Spring marks the transaction `rollbackOnly`. If the caller catches that exception and the method returns normally, Spring tries to commit — but the transaction is already marked for rollback, so it throws `UnexpectedRollbackException`. The fix is to not swallow exceptions that cause rollback, or to use `REQUIRES_NEW` to isolate the inner operation.

---

### 15.2 Isolation & Locking

**Q: What is the difference between optimistic and pessimistic locking?**

> Optimistic locking assumes conflicts are rare. It adds a `@Version` column; conflicts are detected at commit time when the `UPDATE ... WHERE version = ?` affects 0 rows, triggering `OptimisticLockException`. No DB locks are held during reads — high concurrency.
> 
> Pessimistic locking acquires DB-level locks (`SELECT FOR UPDATE`) immediately on read. Other transactions block until the lock is released. Higher consistency guarantee but lower throughput and risk of deadlocks.

---

**Q: When would you choose pessimistic locking over optimistic?**

> When contention on specific rows is genuinely high (e.g., a shared bank account balance), retrying is expensive or not feasible (synchronous user operation), or you cannot tolerate even one conflict slipping through.

---

**Q: How do you prevent deadlocks with pessimistic locking?**

> Always acquire multiple locks in a consistent order (e.g., sort rows by primary key before locking). This prevents the circular wait condition that causes deadlocks.

---

**Q: What anomalies do the four isolation levels protect against?**

> `READ_UNCOMMITTED` — none. `READ_COMMITTED` — dirty reads. `REPEATABLE_READ` — dirty reads + non-repeatable reads. `SERIALIZABLE` — all three (dirty reads, non-repeatable reads, phantom reads). Each higher level adds protection at the cost of concurrency.

---

### 15.3 Advanced

**Q: How does Spring propagate the transaction context across methods?**

> Via `ThreadLocal`. `TransactionSynchronizationManager` stores the current `Connection` (or `EntityManager`) bound to the current thread. As long as the same thread processes the call chain, all methods participate in the same transaction. This breaks with `@Async` (different thread) and reactive programming (no thread-local).

---

**Q: Does `@Transactional` work with `@Async`?**

> Not automatically. Since `@Async` runs in a different thread, it has no access to the caller's thread-local transaction context. `@Transactional` on the async method starts a new, independent transaction for that thread.

---

**Q: What is the SAGA pattern and when do you use it?**

> SAGA is a pattern for managing long-lived distributed transactions across microservices without XA/2PC. Each step of the business process is a local transaction. On failure, compensating transactions undo previous steps. Used when a single business operation spans multiple services/databases and XA is impractical (performance, coupling).

---

**Q: How does `@TransactionalEventListener` differ from `@EventListener`?**

> `@EventListener` fires immediately when the event is published, still inside the publisher's transaction. `@TransactionalEventListener` (with `AFTER_COMMIT` phase) fires only after the transaction commits — ensuring the listener doesn't act on data that might be rolled back.

---

**Q: What does `readOnly = true` actually do?**

> It sets a `readOnly` hint on the transaction. Hibernate sets `FlushMode.MANUAL` (skips dirty-checking, won't write to DB accidentally). Some JDBC drivers and connection pools use the hint to route reads to a read replica. Some databases use it to skip acquiring write locks on scanned rows. It's an optimization hint, not a hard write-prevention mechanism.

---

**Q: What is the Open Session in View (OSIV) pattern and why is it controversial?**

> OSIV keeps the Hibernate `Session` (and therefore DB connection) open for the entire HTTP request, allowing lazy loading in the view/controller layer. It's enabled by default in Spring Boot (`spring.jpa.open-in-view=true`). It's controversial because it silently holds DB connections for the full request lifecycle and makes lazy-loading N+1 queries easy to overlook. Best practice for production is to disable it and manage fetch strategies explicitly.

---

**Q: What happens to a `@Transactional` method called from a non-Spring context (e.g., a scheduler not managed by Spring)?**

> If the scheduler is a Spring-managed bean, `@Transactional` works normally. If it's not (e.g., Quartz job instantiated by Quartz itself), the bean is not proxied by Spring, and `@Transactional` is silently ignored. Solution: use `SpringBeanJobFactory` to let Spring manage Quartz job beans.

---

**Q: Can you have multiple `@Transactional` annotations on the same method?**

> No. Only one `@Transactional` annotation is effective per method. You can use `transactionManager` attribute to specify which transaction manager to use. For participating in multiple transactions, use programmatic transaction management.

---

*Last updated: May 2026*
