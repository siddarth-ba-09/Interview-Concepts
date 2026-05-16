# Transaction Isolation Levels

> **Why this matters in interviews:** Isolation levels are one of the most misunderstood concepts in backend engineering. Interviewers ask about them in the context of data consistency bugs, performance trade-offs, and system design. A strong answer covers not just *what* each level is, but *what anomaly it prevents, how the DB enforces it internally, and when to use it in production*.

---

## 1. Overview — What is it? Why does it exist?

A **database transaction** must satisfy the **ACID** properties:
- **A**tomicity — all-or-nothing
- **C**onsistency — DB moves from one valid state to another
- **I**solation — concurrent transactions don't interfere with each other
- **D**urability — committed data survives crashes

**Isolation** is the tricky one. Full isolation (every transaction sees a perfectly consistent snapshot, as if running serially) is expensive. Relaxed isolation improves concurrency but introduces anomalies.

**Transaction Isolation Levels** are a spectrum — a contract between you and the DB about how much concurrency-related anomaly you're willing to tolerate in exchange for performance.

```
← Less Isolation, More Concurrency, Higher Performance
READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
     More Isolation, Less Concurrency, Lower Performance →
```

---

## 2. The Four Concurrency Anomalies

Before understanding the levels, you must understand what they're protecting against.

### 2.1 Dirty Read

**Definition:** Transaction A reads data written by Transaction B that has **not yet committed**. If B rolls back, A has read data that never officially existed.

```
Time  Transaction A (Read)          Transaction B (Write)
────  ──────────────────────────    ──────────────────────────
T1                                  UPDATE accounts SET balance = 5000
                                    WHERE id = 42;   -- not committed yet
T2    SELECT balance FROM accounts
      WHERE id = 42;
      → reads 5000  ← DIRTY READ
T3                                  ROLLBACK;  -- balance reverts to original
T4    A uses 5000 for a decision    -- but it was wrong data!
```

**Real world:** A banking app reads an account balance mid-transfer. The transfer rolls back, but the app already approved a loan based on the dirty balance.

---

### 2.2 Non-Repeatable Read

**Definition:** Transaction A reads the same row **twice** and gets **different values** because Transaction B committed an update between the two reads.

```
Time  Transaction A (Read)          Transaction B (Write)
────  ──────────────────────────    ──────────────────────────
T1    SELECT price FROM products
      WHERE id = 1;
      → price = 100
T2                                  UPDATE products SET price = 150
                                    WHERE id = 1;
                                    COMMIT;
T3    SELECT price FROM products
      WHERE id = 1;
      → price = 150  ← DIFFERENT VALUE!
      (same transaction, same query, different result)
```

**Real world:** An e-commerce order service reads product price for validation, another service updates the price, the order service reads again for the invoice — inconsistent prices in one order flow.

---

### 2.3 Phantom Read

**Definition:** Transaction A executes the **same range query twice** and sees **different rows** (new rows appeared or disappeared) because Transaction B inserted/deleted rows in that range between the reads.

```
Time  Transaction A (Read)          Transaction B (Write)
────  ──────────────────────────    ──────────────────────────
T1    SELECT COUNT(*) FROM orders
      WHERE customer_id = 7;
      → count = 3
T2                                  INSERT INTO orders (customer_id, ...)
                                    VALUES (7, ...);
                                    COMMIT;
T3    SELECT COUNT(*) FROM orders
      WHERE customer_id = 7;
      → count = 4  ← PHANTOM ROW appeared!
```

**Real world:** A fraud detection system counts transactions per customer in one report — the count changes mid-report because new transactions are being inserted concurrently.

---

### 2.4 Lost Update (Not in SQL standard, but critical in practice)

**Definition:** Two transactions read the same value, both compute a new value based on the old one, both write — the second write **overwrites** the first.

```
Time  Transaction A                 Transaction B
────  ──────────────────────────    ──────────────────────────
T1    SELECT stock FROM products
      WHERE id = 1;  → stock = 10
T2                                  SELECT stock FROM products
                                    WHERE id = 1;  → stock = 10
T3    UPDATE products SET stock = 9  -- A deducted 1
      WHERE id = 1;
      COMMIT;
T4                                  UPDATE products SET stock = 9  -- B also deducted 1
                                    WHERE id = 1;  -- B read 10, not 9!
                                    COMMIT;
      -- stock should be 8, but it's 9. One deduction was LOST.
```

---

## 3. The Four Isolation Levels

Defined by the **SQL-92 standard** (`java.sql.Connection` constants).

### Anomaly Prevention Matrix

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update |
|----------------|:---------:|:------------------:|:-----------:|:-----------:|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| READ COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible | ✅ Possible |
| REPEATABLE READ | ❌ Prevented | ❌ Prevented | ⚠️ Possible* | ❌ Prevented |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |

> *MySQL InnoDB's REPEATABLE READ also prevents phantom reads via gap locks. PostgreSQL's REPEATABLE READ prevents phantoms via MVCC snapshots. The SQL standard says REPEATABLE READ does NOT prevent phantoms — but both major DBs go beyond the standard here.

---

### 3.1 READ UNCOMMITTED

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_UNCOMMITTED);
```

**What it means:** A transaction can read rows modified by other transactions **even before they commit**.

**How DB implements it:** No read locks at all. Writers acquire write locks but readers don't even check them.

**When to use:** Almost never. Theoretically useful for approximate aggregations where dirty data is acceptable (e.g., a dashboard showing rough row counts), but even then, READ COMMITTED is usually preferred.

**Database defaults:** No major DB defaults to this.

```sql
-- Session A
BEGIN;
UPDATE products SET stock = 0 WHERE id = 1;
-- NOT committed yet

-- Session B (READ UNCOMMITTED)
SELECT stock FROM products WHERE id = 1;
-- Returns 0  ← DIRTY READ (A might rollback!)

-- Session A
ROLLBACK;

-- Session B already made a decision based on stock=0 — wrong!
```

---

### 3.2 READ COMMITTED

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

**What it means:** A transaction only sees data that has been **committed**. Each `SELECT` within the transaction sees the latest committed snapshot **at the moment of that SELECT**.

**How DB implements it (two approaches):**

**Lock-based (SQL Server, older Oracle):**
- Reader acquires a **shared lock** on the row, reads it, **immediately releases** the lock.
- Writer holds an **exclusive lock** until commit.
- Readers never block writers (lock released immediately after read).

**MVCC-based (PostgreSQL, MySQL InnoDB):**
```
MVCC = Multi-Version Concurrency Control

Each row has hidden columns: xmin (created by txn), xmax (deleted by txn)

Row versions in storage:
  products id=1: [xmin=100, xmax=null, stock=10]  ← current

Session A (txn 101) updates:
  products id=1: [xmin=100, xmax=101, stock=10]   ← old version
  products id=1: [xmin=101, xmax=null, stock=5]   ← new version (not committed)

Session B (txn 102) reads with READ COMMITTED:
  Sees only rows where xmax is null OR xmax belongs to an aborted txn
  → reads stock=10 (the committed version)

Session A commits (txn 101 marked committed):
  Session B reads again:
  → now reads stock=5 (the newly committed version)
  ← THIS is a Non-Repeatable Read — same query, different result
```

**Default for:** PostgreSQL, Oracle, SQL Server, DB2.

**Use for:** Most OLTP workloads. REST APIs, microservices. Acceptable non-repeatable reads if your logic doesn't depend on reading the same row twice within one transaction.

---

### 3.3 REPEATABLE READ

```java
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);
```

**What it means:** Within a transaction, if you read a row, you get the **same value every time you read it**, regardless of concurrent commits. The snapshot is **fixed at the start of the transaction**.

**How DB implements it:**

**Lock-based:**
- Shared locks on **every row read** are held until transaction ends (not released after read).
- This prevents other transactions from updating those rows.
- Expensive on lock table for long transactions.

**MVCC-based (PostgreSQL, MySQL):**
```
Transaction snapshot is taken at START of transaction, not at each statement.

Session B (txn 102) starts → snapshot taken: "see only txn ≤ 101"

Session A (txn 103) updates products id=1, stock=5, COMMITS.

Session B reads products id=1:
  → still sees stock=10 (snapshot says: ignore txn 103, it started after me)

Session B reads again:
  → still sees stock=10 ← Repeatable! No non-repeatable read.
```

**MySQL InnoDB's REPEATABLE READ also prevents phantom reads** via **gap locks**:
```sql
-- Session B
SELECT * FROM orders WHERE amount > 1000 FOR UPDATE;
-- InnoDB places gap locks on the index range (1000, +∞)

-- Session A tries to INSERT INTO orders (amount = 2000)
-- → BLOCKED by Session B's gap lock until B commits
```

**Default for:** MySQL InnoDB.

**Use for:** Reports that run multi-query aggregations, financial calculations that read multiple related rows (e.g., sum of all line items for an invoice), anywhere consistency across multiple reads in one transaction matters.

---

### 3.4 SERIALIZABLE

```java
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

**What it means:** Transactions execute as if they were **completely serial** — one after another. No concurrency anomaly is possible.

**How DB implements it:**

**Lock-based:**
- Full table/range locks on all reads. Read and write locks held until commit.
- Essentially no concurrency for conflicting transactions.

**SSI — Serializable Snapshot Isolation (PostgreSQL 9.1+):**
```
A more sophisticated approach: let transactions run concurrently,
track read/write dependencies between them, and ABORT a transaction
if a dependency cycle is detected (would violate serializability).

Tx A reads rows that Tx B writes  +  Tx B reads rows that Tx A writes
→ Dependency cycle → PostgreSQL aborts one of them with:
   ERROR: could not serialize access due to read/write dependencies
   SQLSTATE: 40001

Your code must RETRY on SQLSTATE 40001.
```

**Performance cost:**
- Lock-based SERIALIZABLE: ~50–80% throughput reduction under contention.
- PostgreSQL SSI: ~10–30% throughput reduction (much better than lock-based).

**Use for:** Financial double-entry bookkeeping, seat reservation (exactly one person gets the last seat), anything where correctness is more important than throughput. Not for general CRUD.

---

## 4. How Databases Actually Implement This — MVCC Deep Dive

Understanding MVCC is what separates a mid-level from a senior-level answer.

```
┌─────────────────────────────────────────────────────────┐
│                 PostgreSQL MVCC Internals                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Physical row storage (heap):                           │
│  ┌────────┬──────┬──────┬─────────────────────────┐    │
│  │ t_xmin │t_xmax│t_cmin│        data             │    │
│  │  (who  │ (who │(cmd  │                         │    │
│  │created)│ del) │ seq) │                         │    │
│  └────────┴──────┴──────┴─────────────────────────┘    │
│                                                         │
│  Multiple versions of same row coexist in heap.         │
│  Visibility determined by transaction snapshot.         │
│                                                         │
│  READ COMMITTED snapshot: "latest committed at each     │
│                            statement execution"         │
│                                                         │
│  REPEATABLE READ snapshot: "latest committed at         │
│                             transaction start"          │
│                                                         │
│  Old versions cleaned up by VACUUM (background process) │
└─────────────────────────────────────────────────────────┘
```

### MVCC Key Properties:
- **Readers never block writers** — no shared locks on reads.
- **Writers never block readers** — readers see old version, writer creates new version.
- **Only writer-writer conflicts** cause blocking (two transactions trying to update the same row).

This is why PostgreSQL and MySQL can handle high read throughput — reads and writes don't contend.

---

## 5. In JDBC

```java
// Set before any statements are executed in the transaction
conn.setAutoCommit(false);
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);

try {
    // All statements here run under REPEATABLE READ
    PreparedStatement ps1 = conn.prepareStatement("SELECT balance FROM accounts WHERE id = ?");
    ps1.setLong(1, accountId);
    ResultSet rs1 = ps1.executeQuery();
    // ...

    PreparedStatement ps2 = conn.prepareStatement("SELECT balance FROM accounts WHERE id = ?");
    ps2.setLong(1, accountId);
    ResultSet rs2 = ps2.executeQuery();
    // Under REPEATABLE READ: rs1 and rs2 return the same value, even if another
    // transaction committed a change between these two reads.

    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

### JDBC Isolation Level Constants

```java
Connection.TRANSACTION_NONE             // 0 - no transaction support (rare drivers)
Connection.TRANSACTION_READ_UNCOMMITTED // 1
Connection.TRANSACTION_READ_COMMITTED   // 2
Connection.TRANSACTION_REPEATABLE_READ  // 4
Connection.TRANSACTION_SERIALIZABLE     // 8
```

Check if the DB supports a level:
```java
DatabaseMetaData meta = conn.getMetaData();
if (meta.supportsTransactionIsolationLevel(Connection.TRANSACTION_SERIALIZABLE)) {
    conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
}
```

---

## 6. In Spring — `@Transactional(isolation = ...)`

Spring maps directly to JDBC isolation levels:

```java
@Service
public class OrderService {

    // Default — uses DB's default isolation (READ_COMMITTED for Postgres, REPEATABLE_READ for MySQL)
    @Transactional
    public void placeOrder(OrderRequest request) { ... }

    // Explicit READ_COMMITTED: good for most CRUD operations
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public List<Order> getOrderHistory(long customerId) { ... }

    // REPEATABLE_READ: generating an invoice that reads multiple line items
    // — ensure price consistency across all reads in this transaction
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public Invoice generateInvoice(long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        List<LineItem> items = lineItemRepo.findByOrderId(orderId);
        // Even if prices are updated concurrently, we see a consistent snapshot
        return buildInvoice(order, items);
    }

    // SERIALIZABLE: seat reservation — only one person can book the last seat
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Booking reserveSeat(long eventId, int seatNumber, long userId) {
        Seat seat = seatRepo.findByEventAndNumber(eventId, seatNumber);
        if (seat.isBooked()) throw new SeatAlreadyBookedException();
        seat.setBooked(true);
        seat.setBookedBy(userId);
        return seatRepo.save(seat);  // concurrent transactions serialized at DB level
    }
}
```

### Spring `Isolation` enum:

| Spring Enum | SQL Standard | `Connection` Constant |
|-------------|-------------|----------------------|
| `Isolation.DEFAULT` | DB default | — |
| `Isolation.READ_UNCOMMITTED` | READ UNCOMMITTED | `TRANSACTION_READ_UNCOMMITTED` |
| `Isolation.READ_COMMITTED` | READ COMMITTED | `TRANSACTION_READ_COMMITTED` |
| `Isolation.REPEATABLE_READ` | REPEATABLE READ | `TRANSACTION_REPEATABLE_READ` |
| `Isolation.SERIALIZABLE` | SERIALIZABLE | `TRANSACTION_SERIALIZABLE` |

> **Important:** `Isolation.DEFAULT` tells Spring to not call `setTransactionIsolation()` at all — it uses whatever the connection pool's default is, which is typically whatever the DB default is.

---

## 7. Database-Specific Defaults & Behaviours

| Database | Default Isolation | Notes |
|----------|-----------------|-------|
| PostgreSQL | READ COMMITTED | SERIALIZABLE uses SSI (very efficient) |
| MySQL InnoDB | REPEATABLE READ | Gap locks prevent phantom reads |
| Oracle | READ COMMITTED | Has no READ UNCOMMITTED; uses MVCC |
| SQL Server | READ COMMITTED | Can switch to SNAPSHOT isolation (similar to MVCC) |
| H2 (testing) | READ COMMITTED | |
| SQLite | SERIALIZABLE | Single writer at a time |

### SQL Server — SNAPSHOT Isolation (bonus)

SQL Server adds a non-standard level: `READ_COMMITTED_SNAPSHOT` — READ COMMITTED semantics but using row versioning (like MVCC) instead of locks. This eliminates reader-writer blocking without needing REPEATABLE READ:

```sql
-- Enable at DB level (SQL Server)
ALTER DATABASE MyDB SET READ_COMMITTED_SNAPSHOT ON;
```

This is why SQL Server apps can often run on READ COMMITTED without heavy lock contention.

---

## 8. Real-World Scenario Walkthroughs

### Scenario 1: Banking — Transfer Between Accounts

**System:** A banking app that transfers ₹10,000 from Account A to Account B.

**The bug with READ COMMITTED:**
```
Time  Tx A (Transfer ₹10,000)       Tx B (Balance Report)
────  ──────────────────────────    ──────────────────────────
T1                                  SELECT balance FROM accounts
                                    WHERE id = 'A';  → ₹50,000
T2    UPDATE accounts SET balance
      = balance - 10000
      WHERE id = 'A';  COMMIT;
                                    -- A's balance is now ₹40,000
T3                                  SELECT balance FROM accounts
                                    WHERE id = 'B';
                                    -- B's balance not yet updated
                                    → ₹20,000
T4    UPDATE accounts SET balance
      = balance + 10000
      WHERE id = 'B';  COMMIT;

Result: Report shows A=₹50,000, B=₹20,000  (total=₹70,000)
        But actual: A=₹40,000, B=₹30,000  (total=₹70,000)
        The report captured an inconsistent mid-transfer state!
```

**Fix: REPEATABLE READ (or SERIALIZABLE)**
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public BalanceReport generateReport() {
    // Snapshot fixed at start of this transaction
    // Sees either PRE-transfer or POST-transfer state, never in-between
    BigDecimal balanceA = accountRepo.getBalance("A");
    BigDecimal balanceB = accountRepo.getBalance("B");
    return new BalanceReport(balanceA, balanceB);  // always consistent
}
```

---

### Scenario 2: Flash Sale — Last Item

**System:** 1000 concurrent users try to buy the last unit of a product.

**The bug with REPEATABLE READ without FOR UPDATE:**
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public boolean purchase(long productId, long userId) {
    Product p = productRepo.findById(productId);  // reads stock=1
    if (p.getStock() <= 0) return false;
    p.setStock(p.getStock() - 1);                 // computes stock=0
    productRepo.save(p);                          // writes stock=0
    // PROBLEM: 10 threads all read stock=1, all compute stock=0, all write stock=0
    // Last-write-wins → lost updates for 9 of 10 threads
}
```

**Fix Option 1: SERIALIZABLE**
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public boolean purchase(long productId, long userId) {
    Product p = productRepo.findById(productId);
    if (p.getStock() <= 0) return false;
    p.setStock(p.getStock() - 1);
    productRepo.save(p);
    // DB detects conflict, aborts 9 of 10 transactions → retry logic needed
}
```

**Fix Option 2: Atomic UPDATE (better performance, any isolation level)**
```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public boolean purchase(long productId, long userId) {
    // Single atomic UPDATE — no read-modify-write race
    int updated = productRepo.deductStock(productId);
    // Repository:
    // UPDATE products SET stock = stock - 1 WHERE id = ? AND stock > 0
    return updated == 1;
}
```
Option 2 is preferred in production — the DB engine handles the concurrency atomically at the row level without needing higher isolation.

---

### Scenario 3: Fraud Detection — Phantom Read

**System:** Fraud system checks: "Has this user made more than 5 transactions > ₹10,000 in the last hour?"

```
Time  Tx Fraud (Check)              Tx Payment (Insert)
────  ──────────────────────────    ──────────────────────────
T1    SELECT COUNT(*) FROM txns
      WHERE user=7 AND amount>10000
      AND time > now()-1hr;
      → count = 4  (safe, < 5)
T2                                  INSERT INTO txns (user=7, amount=15000)
                                    COMMIT;
T3    SELECT COUNT(*) FROM txns
      WHERE user=7 AND amount>10000
      AND time > now()-1hr;
      → count = 5  ← PHANTOM ROW
      -- Now flagged as fraud? But first check said safe.
```

**Fix: REPEATABLE READ (MySQL prevents this with gap locks) or SERIALIZABLE**

On PostgreSQL under REPEATABLE READ, the second SELECT still returns 4 (snapshot isolation). On MySQL InnoDB under REPEATABLE READ, gap locks prevent the INSERT from committing until the fraud check transaction ends.

---

## 9. Common Interview Questions

**Q1: What is a dirty read? Give a real-world example where it causes a bug.**

A dirty read occurs when Transaction A reads data written by Transaction B before B commits. If B rolls back, A has used phantom data. Real-world: a payment service reads an account balance that a concurrent transfer wrote but then rolled back due to a downstream failure. The payment service approved a withdrawal based on money that never existed in the account.

---

**Q2: What is the difference between a non-repeatable read and a phantom read?**

Both involve reading different data in the same transaction. A **non-repeatable read** is about a **specific row changing** — you read row with id=1, another transaction updates it and commits, you read id=1 again and get a different value. A **phantom read** is about **rows appearing or disappearing in a range** — you count rows matching a condition, another transaction inserts matching rows, you count again and get a different number. REPEATABLE READ prevents the first but not necessarily the second (though MySQL InnoDB does prevent phantoms via gap locks).

---

**Q3: What is MVCC and how does it help?**

Multi-Version Concurrency Control maintains multiple versions of each row. Readers see a snapshot of the DB at a point in time (transaction start for REPEATABLE READ, statement start for READ COMMITTED) without acquiring read locks. This means **readers never block writers and writers never block readers** — only writer-writer conflicts cause blocking. This is why PostgreSQL and MySQL have excellent read throughput even under write load.

---

**Q4: Your app is on MySQL. You're on the default isolation level. A phantom read occurs. Is that expected?**

No — MySQL InnoDB's default `REPEATABLE READ` prevents phantom reads via **gap locks** and **next-key locks**. This goes beyond the SQL-92 standard (which says REPEATABLE READ does not prevent phantoms). If a phantom read is occurring on MySQL InnoDB under REPEATABLE READ, check: are you using `FOR UPDATE` or `LOCK IN SHARE MODE`? Are you using a storage engine other than InnoDB?

---

**Q5: When would you actually use SERIALIZABLE in production?**

Use SERIALIZABLE when lost updates or phantom reads would cause real financial or correctness damage — airline/event seat booking (exactly one person books the last seat), financial double-entry bookkeeping (debits must exactly equal credits), inventory systems where overselling has real cost. Combine with retry logic on serialization failure (SQLState `40001`). For most CRUD APIs, READ COMMITTED with atomic SQL updates is a better trade-off.

---

**Q6: What is the "lost update" problem? Is it prevented by REPEATABLE READ?**

Lost update: two transactions read the same value, both increment it independently, both write — one write overwrites the other. Example: two threads both read stock=10, compute 9, write 9. Stock should be 8 but is 9. REPEATABLE READ alone does NOT prevent lost updates with ORM-style read-modify-write. Fix options: (a) use atomic SQL (`UPDATE ... SET stock = stock - 1 WHERE stock > 0`), (b) use `SELECT FOR UPDATE` (pessimistic lock), (c) use optimistic locking (`@Version` in JPA / `WHERE version = ?`), (d) use SERIALIZABLE.

---

**Q7: What is the difference between `Isolation.DEFAULT` and `Isolation.READ_COMMITTED` in Spring?**

`Isolation.DEFAULT` instructs Spring's transaction infrastructure to not set any isolation level — the connection uses whatever default the pool/DB has. `Isolation.READ_COMMITTED` explicitly calls `connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED)`. Functionally they may be the same if the DB default is READ COMMITTED (PostgreSQL), but `Isolation.DEFAULT` is more portable and avoids overriding pool-level settings. Be careful: if you deploy the same code on MySQL (where default is REPEATABLE READ) and PostgreSQL, behavior may differ under `Isolation.DEFAULT`.

---

**Q8: What happens at the DB level when you set `@Transactional(isolation = Isolation.REPEATABLE_READ)` in Spring?**

Spring's `PlatformTransactionManager` intercepts the method call (via AOP proxy), borrows a connection from the pool, calls `connection.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ)`, then calls `connection.setAutoCommit(false)`. After the method returns, it calls `connection.commit()` or `connection.rollback()`. When the connection is returned to HikariCP, HikariCP resets the isolation level back to the pool default to avoid state leaking to the next borrower.

---

## 10. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **Assuming REPEATABLE READ prevents phantoms everywhere** | PostgreSQL REPEATABLE READ DOES prevent phantoms (MVCC snapshot). MySQL InnoDB REPEATABLE READ prevents them via gap locks. But this isn't guaranteed by SQL standard. | Know your DB's actual behavior, don't rely on standard alone. |
| **Using SERIALIZABLE without retry logic** | On serialization failure (SQLState `40001`), the transaction is aborted. App gets an exception and never retries → user sees error. | Implement retry with exponential backoff for `40001`. |
| **Holding long transactions at high isolation** | REPEATABLE READ with long transactions bloats MVCC version chains (PostgreSQL), holds gap locks (MySQL), reducing concurrency. | Keep transactions short. Do not do external API calls inside a transaction. |
| **Setting isolation on a nested transaction** | Spring's nested/required propagation reuses the outer connection — you cannot change isolation mid-connection in most drivers. | Only set isolation on the outermost transaction. |
| **Not knowing your DB's default** | Behaviour differs between PostgreSQL (READ_COMMITTED) and MySQL (REPEATABLE_READ). Same Spring code behaves differently on different DBs. | Always explicitly set isolation on critical transactions instead of relying on `DEFAULT`. |
| **Assuming higher isolation = more locking** | PostgreSQL SERIALIZABLE (SSI) is lock-free — it uses dependency tracking. It's much more concurrent than lock-based SERIALIZABLE. | Don't avoid SERIALIZABLE blindly; measure actual throughput on your DB. |
| **Lost update with `@Version` (JPA optimistic locking)** | `@Version` raises `OptimisticLockingFailureException` on conflict, but you must catch it and retry at the application level. | Always handle optimistic lock failures with retry in the service layer. |

---

## 11. Summary — Quick Revision

- **4 anomalies:** Dirty Read → Non-Repeatable Read → Phantom Read → Lost Update (severity increases left to right).
- **4 levels:** READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE (protection increases, concurrency decreases).
- **READ COMMITTED** is the right default for most web API CRUD — prevents dirty reads, acceptable non-repeatable reads.
- **REPEATABLE READ** for multi-read consistency (reports, invoices) — snapshot fixed at transaction start (MVCC) or row locks held till commit (lock-based).
- **SERIALIZABLE** for exact-once semantics (seat booking, financial ledger) — requires retry on `40001` serialization failures.
- **MVCC** (PostgreSQL, MySQL InnoDB) means readers never block writers — isolation is achieved via row versioning, not shared locks.
- **Know your DB defaults:** PostgreSQL = READ COMMITTED, MySQL InnoDB = REPEATABLE READ.
- In Spring: `@Transactional(isolation = Isolation.X)` → Spring sets it on the connection before the transaction starts, HikariCP resets it after.

---

> **Follow-up questions to think about:**
> 1. Two Spring `@Transactional` methods with different isolation levels call each other. What isolation level runs?
> 2. How does PostgreSQL's SSI (Serializable Snapshot Isolation) detect conflicts without using locks?
> 3. In MySQL, what is a "next-key lock" and how does it prevent phantom reads under REPEATABLE READ?
