# JDBC — Java Database Connectivity

> **Learning Path Position:** JDBC → Spring JDBC → Spring Data JDBC → JPA → Hibernate → Spring Data JPA
>
> JDBC is the **foundation of everything**. Every ORM, every Spring abstraction ultimately calls JDBC under the hood. Understanding it deeply is what separates engineers who debug production issues from those who can't.

---

## 1. Overview — What is it? Why does it exist?

JDBC (Java Database Connectivity) is a **Java API** (part of `java.sql` and `javax.sql` packages) that defines a standard interface for connecting Java applications to relational databases.

Before JDBC (pre-1997), every database vendor had its own proprietary API. Java code written for Oracle couldn't talk to MySQL. JDBC solved this with a **driver-based abstraction**:

- Your code talks to **JDBC interfaces** (standard API).
- The **JDBC Driver** (vendor-provided) translates those calls to the database's native protocol.
- Swapping databases = swapping the driver JAR + connection URL. Your code stays the same.

**Real-world analogy:** JDBC is like a power adapter standard. Your laptop (Java code) has a standard plug (JDBC API). The adapter (JDBC Driver) knows how to convert that to the wall socket format (Oracle/MySQL/Postgres wire protocol).

---

## 2. JDBC Architecture

```
+------------------+
|   Java App Code  |
+--------+---------+
         |  uses
+--------v---------+
|   JDBC API       |  <- java.sql.*, javax.sql.*  (interfaces only)
| (DriverManager,  |
|  Connection,     |
|  Statement,      |
|  ResultSet...)   |
+--------+---------+
         |  delegates to
+--------v---------+
|   JDBC Driver    |  <- vendor-provided JAR (e.g. mysql-connector-java)
| (Type 4 - Thin)  |
+--------+---------+
         |  TCP/IP (native protocol)
+--------v---------+
|    Database      |  <- MySQL / PostgreSQL / Oracle / H2
+------------------+
```

### JDBC Driver Types

| Type | Name | How it works | Used today? |
|------|------|-------------|-------------|
| Type 1 | JDBC-ODBC Bridge | Converts JDBC calls → ODBC calls | No (removed in Java 8) |
| Type 2 | Native-API Driver | Calls DB native client libraries via JNI | Rarely |
| Type 3 | Network Protocol Driver | Routes through a middleware server | Rarely |
| **Type 4** | **Thin Driver (Pure Java)** | Direct TCP/IP to DB using native protocol | **Yes — all modern drivers** |

> **Interview tip:** Always say "Type 4 — pure Java thin driver" when asked what JDBC driver type is used in production. MySQL Connector/J, PostgreSQL JDBC, Oracle Thin — all Type 4.

---

## 3. Key Classes & Interfaces

| Class / Interface | Package | Role |
|-------------------|---------|------|
| `DriverManager` | `java.sql` | Manages registered drivers; creates `Connection` objects |
| `DataSource` | `javax.sql` | Preferred over `DriverManager`; supports connection pooling |
| `Connection` | `java.sql` | Represents a single DB connection; unit of transaction |
| `Statement` | `java.sql` | Executes static SQL; vulnerable to SQL injection |
| `PreparedStatement` | `java.sql` | Pre-compiled parameterized SQL; safe & faster |
| `CallableStatement` | `java.sql` | Executes stored procedures |
| `ResultSet` | `java.sql` | Cursor over query results |
| `ResultSetMetaData` | `java.sql` | Metadata about columns in a `ResultSet` |
| `DatabaseMetaData` | `java.sql` | Metadata about the DB itself (version, supported features) |
| `SQLException` | `java.sql` | Checked exception for all JDBC errors |
| `Savepoint` | `java.sql` | Partial transaction rollback point |

---

## 4. The JDBC Connection Lifecycle

Every JDBC operation follows this lifecycle:

```
1. Load Driver (auto in JDBC 4.0+)
        ↓
2. Get Connection  ←─── most expensive step (TCP handshake + auth)
        ↓
3. Create Statement / PreparedStatement
        ↓
4. Execute Query / Update
        ↓
5. Process ResultSet
        ↓
6. Close Resources (ResultSet → Statement → Connection)
```

### 4.1 Getting a Connection

**Old way (avoid in production):**
```java
// DriverManager — creates a new physical connection every time
Connection conn = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/ecommerce",
    "root",
    "secret"
);
```

**Production way — DataSource with Connection Pool:**
```java
// HikariCP — the fastest connection pool, default in Spring Boot
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/ecommerce");
config.setUsername("app_user");
config.setPassword("secret");
config.setMaximumPoolSize(20);           // max 20 connections in pool
config.setMinimumIdle(5);               // keep 5 idle connections ready
config.setConnectionTimeout(30000);     // 30s to get a connection from pool
config.setIdleTimeout(600000);          // remove idle connection after 10 min

DataSource dataSource = new HikariDataSource(config);

// This does NOT create a new TCP connection — it borrows from pool
Connection conn = dataSource.getConnection();
```

> **Why DataSource over DriverManager?**
> Creating a real DB connection involves: TCP handshake, authentication, session setup — ~50-150ms. A connection pool pre-creates connections and **lends** them. Getting a pooled connection takes ~microseconds.

---

## 5. Statement Types

### 5.1 `Statement` — Static SQL (Avoid for user input)

```java
Statement stmt = conn.createStatement();
// NEVER DO THIS WITH USER INPUT — SQL Injection risk!
ResultSet rs = stmt.executeQuery(
    "SELECT * FROM orders WHERE customer_id = " + customerId
);
```

**SQL Injection example:**
If `customerId = "1 OR 1=1"`, the query becomes:
```sql
SELECT * FROM orders WHERE customer_id = 1 OR 1=1
-- Returns ALL orders from ALL customers!
```

### 5.2 `PreparedStatement` — Parameterized SQL (Always use this)

```java
// SQL is compiled ONCE, parameters are substituted safely
String sql = "SELECT * FROM orders WHERE customer_id = ? AND status = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setLong(1, customerId);       // parameter index is 1-based
pstmt.setString(2, "DELIVERED");

ResultSet rs = pstmt.executeQuery();
```

**Why PreparedStatement is faster:**
1. The DB parses and compiles the SQL once (creates an execution plan).
2. Subsequent calls with different parameters reuse the cached plan.
3. Parameters are **escaped** by the driver — no SQL injection possible.

### 5.3 `CallableStatement` — Stored Procedures

```java
// Calling a stored procedure: calculate_discount(customer_id, OUT discount)
CallableStatement cstmt = conn.prepareCall("{call calculate_discount(?, ?)}");
cstmt.setLong(1, customerId);
cstmt.registerOutParameter(2, Types.DECIMAL);  // OUT parameter

cstmt.execute();
BigDecimal discount = cstmt.getBigDecimal(2);
```

### 5.4 Execute Methods

| Method | Returns | Use for |
|--------|---------|---------|
| `executeQuery(sql)` | `ResultSet` | SELECT |
| `executeUpdate(sql)` | `int` (rows affected) | INSERT, UPDATE, DELETE, DDL |
| `execute(sql)` | `boolean` (true = ResultSet) | When you don't know the type |
| `executeBatch()` | `int[]` (rows affected per batch) | Batch inserts/updates |

---

## 6. ResultSet — Processing Query Results

```java
String sql = "SELECT id, name, price, created_at FROM products WHERE category = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "ELECTRONICS");

ResultSet rs = pstmt.executeQuery();

while (rs.next()) {  // cursor starts BEFORE first row
    long id         = rs.getLong("id");
    String name     = rs.getString("name");
    BigDecimal price = rs.getBigDecimal("price");
    LocalDate date  = rs.getDate("created_at").toLocalDate();

    System.out.printf("Product: %s, Price: %s%n", name, price);
}
```

### ResultSet Types

```java
// Default: forward-only, read-only (most efficient)
conn.createStatement(
    ResultSet.TYPE_FORWARD_ONLY,       // can only call rs.next()
    ResultSet.CONCUR_READ_ONLY
);

// Scrollable: can move forward/backward
conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE, // scroll but won't see live DB changes
    ResultSet.CONCUR_READ_ONLY
);
// Now you can: rs.first(), rs.last(), rs.absolute(5), rs.previous()

// Updatable: can update rows in-place (rare in modern code)
conn.createStatement(
    ResultSet.TYPE_SCROLL_SENSITIVE,
    ResultSet.CONCUR_UPDATABLE
);
```

> **Interview tip:** Always use `TYPE_FORWARD_ONLY` + `CONCUR_READ_ONLY` unless you specifically need scrollability or updatability. Other types consume more memory and DB resources.

---

## 7. Transaction Management in JDBC

By default, JDBC runs in **auto-commit mode** — every statement is immediately committed.

```java
conn.setAutoCommit(false);  // BEGIN TRANSACTION

try {
    // In an e-commerce order placement:
    PreparedStatement deductStock = conn.prepareStatement(
        "UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?"
    );
    deductStock.setInt(1, 2);
    deductStock.setLong(2, productId);
    deductStock.executeUpdate();

    PreparedStatement insertOrder = conn.prepareStatement(
        "INSERT INTO orders (customer_id, product_id, quantity, total) VALUES (?, ?, ?, ?)"
    );
    insertOrder.setLong(1, customerId);
    insertOrder.setLong(2, productId);
    insertOrder.setInt(3, 2);
    insertOrder.setBigDecimal(4, total);
    insertOrder.executeUpdate();

    conn.commit();   // both succeed → COMMIT

} catch (SQLException e) {
    conn.rollback(); // either fails → ROLLBACK (no orphaned stock deduction)
    throw e;
} finally {
    conn.setAutoCommit(true);
}
```

### Transaction Isolation Levels

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

| Constant | Level | Prevents |
|----------|-------|---------|
| `TRANSACTION_READ_UNCOMMITTED` | 1 | Nothing — dirty reads possible |
| `TRANSACTION_READ_COMMITTED` | 2 | Dirty reads |
| `TRANSACTION_REPEATABLE_READ` | 4 | Dirty reads + Non-repeatable reads |
| `TRANSACTION_SERIALIZABLE` | 8 | All anomalies (slowest) |

> Most production apps use `READ_COMMITTED` (PostgreSQL default) or `REPEATABLE_READ` (MySQL InnoDB default).

### Savepoints — Partial Rollback

```java
conn.setAutoCommit(false);

// Insert order header
stmt.executeUpdate("INSERT INTO order_header ...");

Savepoint afterHeader = conn.setSavepoint("AFTER_HEADER");

try {
    // Try to apply a coupon
    stmt.executeUpdate("UPDATE coupon SET used = true WHERE code = 'SAVE10'");
} catch (SQLException e) {
    // Coupon failed — roll back only this part, keep the order header
    conn.rollback(afterHeader);
}

conn.commit(); // commits order header, coupon attempt rolled back
```

---

## 8. Batch Processing

When inserting/updating thousands of rows, individual round-trips kill performance. Batching groups multiple statements into one network call.

```java
// Scenario: Importing 10,000 products from a CSV feed
conn.setAutoCommit(false);

String sql = "INSERT INTO products (sku, name, price, category) VALUES (?, ?, ?, ?)";
PreparedStatement pstmt = conn.prepareStatement(sql);

int batchSize = 500;

for (int i = 0; i < products.size(); i++) {
    Product p = products.get(i);
    pstmt.setString(1, p.getSku());
    pstmt.setString(2, p.getName());
    pstmt.setBigDecimal(3, p.getPrice());
    pstmt.setString(4, p.getCategory());
    pstmt.addBatch();  // adds to batch, does NOT send to DB yet

    if ((i + 1) % batchSize == 0) {
        pstmt.executeBatch();  // send 500 rows in ONE network call
        conn.commit();
        pstmt.clearBatch();
    }
}
// Handle remaining rows
pstmt.executeBatch();
conn.commit();
```

**Performance difference:**
- 10,000 individual inserts: ~10,000 round trips → maybe 30–60 seconds
- 10,000 rows in batches of 500: 20 round trips → ~1–2 seconds

---

## 9. Resource Management — Closing Connections

This is the #1 source of production bugs with raw JDBC. **Connections MUST be closed**, or you leak pool connections and the app hangs.

### Wrong way (pre-Java 7):
```java
Connection conn = null;
PreparedStatement pstmt = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    // ...
} finally {
    // if rs.close() throws, pstmt and conn are never closed!
    if (rs != null) rs.close();
    if (pstmt != null) pstmt.close();
    if (conn != null) conn.close();
}
```

### Correct way — try-with-resources (Java 7+):
```java
// All AutoCloseable resources closed in LIFO order, even if exceptions thrown
try (Connection conn = dataSource.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(
         "SELECT * FROM customers WHERE email = ?")) {

    pstmt.setString(1, email);

    try (ResultSet rs = pstmt.executeQuery()) {
        while (rs.next()) {
            // process rows
        }
    }
    // rs closed here automatically
}
// pstmt and conn closed here automatically
```

> **What does `conn.close()` do with a pool?** It does NOT close the physical TCP connection. It returns the connection back to the HikariCP pool, ready for the next borrower.

---

## 10. Metadata APIs

### ResultSetMetaData — Column info at runtime

```java
ResultSet rs = pstmt.executeQuery();
ResultSetMetaData meta = rs.getMetaData();

int columnCount = meta.getColumnCount();
for (int i = 1; i <= columnCount; i++) {
    System.out.printf("Column %d: name=%s, type=%s, nullable=%b%n",
        i,
        meta.getColumnName(i),
        meta.getColumnTypeName(i),
        meta.isNullable(i) == ResultSetMetaData.columnNullable
    );
}
```

**Use case:** Generic CSV export utility, dynamic query builders, ORMs (Hibernate uses this to map result columns to entity fields).

### DatabaseMetaData — DB capabilities

```java
DatabaseMetaData dbMeta = conn.getMetaData();
System.out.println("DB Product: " + dbMeta.getDatabaseProductName());
System.out.println("DB Version: " + dbMeta.getDatabaseProductVersion());
System.out.println("Driver: " + dbMeta.getDriverName());
System.out.println("Max Connections: " + dbMeta.getMaxConnections());

// Check if stored procedures supported
System.out.println("Supports stored procs: " + dbMeta.supportsStoredProcedures());
```

---

## 11. Connection Pooling Deep Dive

### Why connection pooling exists

Creating a real DB connection:
1. DNS resolution
2. TCP 3-way handshake
3. TLS handshake (if SSL)
4. DB authentication
5. Session initialization

This takes **50–200ms** per connection. A web app handling 1000 req/sec cannot afford this on every request.

### HikariCP (the standard in Spring Boot)

```
App Thread 1 ──→ borrow conn ──→ use ──→ return ──→ pool
App Thread 2 ──→ borrow conn ──→ use ──→ return ──→ pool
App Thread 3 ──→ wait (pool full) ──→ timeout → HikariTimeoutException
                 ↑
     connection-timeout = 30s (default)
```

Key HikariCP settings:

```properties
# Spring Boot application.properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000  # warn if connection held > 60s
```

**How to size your pool:**
```
Pool size = (core_count * 2) + number_of_spindle_disks
```
For a 4-core server with SSD: `(4 * 2) + 1 = 9` → use 10.

> **HikariCP fact:** It uses a lock-free `ConcurrentBag` internally instead of a `BlockingQueue`. This is why it benchmarks 2–5x faster than c3p0 or DBCP2.

---

## 12. SQLException Handling

`SQLException` is a **checked exception** and is also a **linked list** — a single database operation can produce a chain of errors.

```java
try {
    // ...
} catch (SQLException e) {
    // Walk the exception chain
    while (e != null) {
        System.err.println("SQLState: " + e.getSQLState());   // 5-char code per SQL standard
        System.err.println("Error Code: " + e.getErrorCode()); // vendor-specific
        System.err.println("Message: " + e.getMessage());
        e = e.getNextException();  // JDBC-specific chaining
    }
}
```

**Common SQLState codes:**
| SQLState | Meaning |
|----------|---------|
| `23000` | Integrity constraint violation (duplicate key, FK) |
| `42000` | Syntax error or access violation |
| `08001` | Connection failure |
| `40001` | Deadlock detected (serialization failure) |
| `HY000` | General error (vendor-specific, check error code) |

### SQLException subclasses (Java 6+):

```java
catch (SQLIntegrityConstraintViolationException e) {
    // Duplicate key, FK violation — don't retry, it's a logic error
} catch (SQLTransientConnectionException e) {
    // DB temporarily unavailable — safe to retry
} catch (SQLTimeoutException e) {
    // Query exceeded Statement.setQueryTimeout() — possible full table scan?
} catch (SQLRecoverableException e) {
    // Connection dropped mid-operation — reconnect and retry
}
```

---

## 13. Auto-Generated Keys

```java
// Get the DB-generated primary key after INSERT
String sql = "INSERT INTO orders (customer_id, total) VALUES (?, ?)";
PreparedStatement pstmt = conn.prepareStatement(
    sql,
    Statement.RETURN_GENERATED_KEYS  // tell JDBC to capture generated keys
);
pstmt.setLong(1, customerId);
pstmt.setBigDecimal(2, total);
pstmt.executeUpdate();

try (ResultSet generatedKeys = pstmt.getGeneratedKeys()) {
    if (generatedKeys.next()) {
        long orderId = generatedKeys.getLong(1);
        System.out.println("Created order with ID: " + orderId);
    }
}
```

---

## 14. Complete Real-World Example

A `ProductRepository` using raw JDBC in a realistic e-commerce context:

```java
@Repository
public class ProductJdbcRepository {

    private final DataSource dataSource;

    public ProductJdbcRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Optional<Product> findById(long id) {
        String sql = "SELECT id, sku, name, price, stock_count, category " +
                     "FROM products WHERE id = ? AND deleted = false";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setLong(1, id);

            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapRow(rs));
                }
                return Optional.empty();
            }
        } catch (SQLException e) {
            throw new DataAccessException("Failed to find product by id: " + id, e);
        }
    }

    public long save(Product product) {
        String sql = "INSERT INTO products (sku, name, price, stock_count, category) " +
                     "VALUES (?, ?, ?, ?, ?)";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(
                     sql, Statement.RETURN_GENERATED_KEYS)) {

            pstmt.setString(1, product.getSku());
            pstmt.setString(2, product.getName());
            pstmt.setBigDecimal(3, product.getPrice());
            pstmt.setInt(4, product.getStockCount());
            pstmt.setString(5, product.getCategory());
            pstmt.executeUpdate();

            try (ResultSet keys = pstmt.getGeneratedKeys()) {
                if (keys.next()) {
                    return keys.getLong(1);
                }
                throw new DataAccessException("No generated key returned");
            }
        } catch (SQLException e) {
            throw new DataAccessException("Failed to save product", e);
        }
    }

    public boolean deductStock(long productId, int quantity, Connection conn) throws SQLException {
        // Accepts an external Connection — caller controls the transaction
        String sql = "UPDATE products SET stock_count = stock_count - ? " +
                     "WHERE id = ? AND stock_count >= ?";

        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setInt(1, quantity);
            pstmt.setLong(2, productId);
            pstmt.setInt(3, quantity);  // optimistic check: only update if enough stock
            return pstmt.executeUpdate() == 1;
        }
    }

    public List<Product> findByCategory(String category) {
        String sql = "SELECT id, sku, name, price, stock_count, category " +
                     "FROM products WHERE category = ? ORDER BY name";

        List<Product> results = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setString(1, category);

            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    results.add(mapRow(rs));
                }
            }
        } catch (SQLException e) {
            throw new DataAccessException("Failed to find products by category", e);
        }
        return results;
    }

    private Product mapRow(ResultSet rs) throws SQLException {
        return new Product(
            rs.getLong("id"),
            rs.getString("sku"),
            rs.getString("name"),
            rs.getBigDecimal("price"),
            rs.getInt("stock_count"),
            rs.getString("category")
        );
    }
}
```

---

## 15. Scenario Walkthrough

### Scenario 1: Flash sale — 1000 users try to buy the last unit simultaneously

**Problem:** Without careful JDBC transaction management:
- Thread A reads stock = 1 ✓
- Thread B reads stock = 1 ✓
- Thread A updates stock = 0, inserts order ✓
- Thread B updates stock = -1, inserts order ← **oversell!**

**JDBC-level fix (optimistic locking via conditional UPDATE):**

```java
public boolean purchaseProduct(long customerId, long productId, int qty) {
    conn.setAutoCommit(false);
    try {
        // Step 1: Try to deduct stock atomically — only succeeds if enough stock exists
        String deductSql = "UPDATE products SET stock_count = stock_count - ? " +
                           "WHERE id = ? AND stock_count >= ?";
        PreparedStatement deduct = conn.prepareStatement(deductSql);
        deduct.setInt(1, qty);
        deduct.setLong(2, productId);
        deduct.setInt(3, qty);
        int rowsAffected = deduct.executeUpdate();

        if (rowsAffected == 0) {
            conn.rollback();
            return false; // out of stock
        }

        // Step 2: Insert the order (stock is already reserved above)
        String orderSql = "INSERT INTO orders (customer_id, product_id, qty) VALUES (?, ?, ?)";
        PreparedStatement order = conn.prepareStatement(orderSql);
        order.setLong(1, customerId);
        order.setLong(2, productId);
        order.setInt(3, qty);
        order.executeUpdate();

        conn.commit();
        return true;
    } catch (SQLException e) {
        conn.rollback();
        throw new RuntimeException(e);
    }
}
```

The `WHERE stock_count >= ?` in the UPDATE is the key — the DB enforces the constraint **atomically** at the row level. Even if 1000 threads hit this simultaneously, only the ones that find `stock_count >= qty` will succeed.

### Scenario 2: Connection pool exhaustion under load

**Symptoms in production:**
- App starts throwing `HikariPool-1 - Connection is not available, request timed out after 30000ms`
- Response times spike suddenly at ~30s

**Root causes and fixes:**

```java
// BAD: connection held while doing slow external API call
Connection conn = dataSource.getConnection();  // borrowed from pool
ProductInfo info = externalCatalogApi.fetch(productId);  // 5 second external call!
PreparedStatement pstmt = conn.prepareStatement("INSERT INTO...");
// Pool is starved for 5s per request

// GOOD: get connection only when needed for DB work
ProductInfo info = externalCatalogApi.fetch(productId);  // external call BEFORE borrowing
try (Connection conn = dataSource.getConnection()) {      // borrow only for DB work
    // quick DB operation
}
```

Set `leak-detection-threshold=60000` in HikariCP to log stack traces of connections held > 60s.

---

## 16. Common Interview Questions

**Q1: What is the difference between `Statement`, `PreparedStatement`, and `CallableStatement`?**

`Statement` executes static SQL strings directly — concatenating user input into SQL strings is a SQL injection vulnerability. `PreparedStatement` pre-compiles the SQL with `?` placeholders — the DB creates one execution plan reused across calls, parameters are escaped by the driver (no injection). `CallableStatement` extends `PreparedStatement` for stored procedure calls and supports `OUT`/`INOUT` parameters. In production, always use `PreparedStatement` or higher.

---

**Q2: What is SQL injection and how does JDBC prevent it?**

SQL injection is when user-supplied input alters the SQL structure. Example: username `admin' --` in a `Statement`-based login would comment out the password check. `PreparedStatement` prevents this because parameters are sent separately from the SQL structure — the driver escapes special characters. The DB sees the parameter as a **value**, never as SQL syntax.

---

**Q3: What is the difference between `DriverManager` and `DataSource`?**

`DriverManager.getConnection()` creates a new physical TCP connection every call — fine for small scripts, completely unacceptable in web apps (50–200ms overhead, no upper bound on connections). `DataSource` is an abstraction over a connection pool — it hands out pre-established connections from a pool and reclaims them on close. Spring Boot auto-configures `HikariDataSource` when you add a JDBC dependency.

---

**Q4: What happens when you call `connection.close()` on a pooled connection?**

It does NOT close the underlying TCP socket. The `HikariProxyConnection` wrapper intercepts the call and returns the physical connection back to `HikariCP`'s pool for the next borrower. The pool resets the connection state (rolls back uncommitted transactions, resets auto-commit, clears warnings) before returning it.

---

**Q5: What is auto-commit and when would you set it to false?**

Auto-commit (default `true`) means each `executeUpdate()` call is immediately committed as its own transaction. Set it to `false` when you need **multi-statement atomicity** — e.g., deducting inventory AND creating an order must either both succeed or both fail. With auto-commit on, if the order insert fails after the inventory deduction succeeds, you've lost stock with no order.

---

**Q6: What are the JDBC transaction isolation levels and what anomaly does each prevent?**

- `READ_UNCOMMITTED`: No protection — threads can read uncommitted (dirty) data from other transactions.
- `READ_COMMITTED`: Prevents dirty reads. A row is only visible after its transaction commits. (**PostgreSQL default**)
- `REPEATABLE_READ`: Prevents dirty reads AND non-repeatable reads — if you read a row twice in one transaction, you get the same value. (**MySQL InnoDB default**)
- `SERIALIZABLE`: Full isolation — transactions execute as if serial. Prevents phantom reads too. Highest correctness, lowest concurrency.

---

**Q7: What is a connection leak and how do you detect/fix it?**

A connection leak is when your code borrows a connection from the pool but never returns it (never calls `close()`). The pool fills up and new requests time out. Causes: exception thrown before `conn.close()` in the `finally` block, long-running code holding the connection. Fix: **always use try-with-resources**. Detection: set `hikari.leak-detection-threshold` — HikariCP logs a stack trace if a connection is held longer than the threshold.

---

**Q8: How does batch processing work in JDBC and when should you use it?**

Batch processing collects multiple `PreparedStatement` executions client-side via `addBatch()`, then sends them all to the DB in one network round-trip via `executeBatch()`. Use it when inserting/updating large numbers of rows (100+). For 10,000 rows with batch size 500: 20 round trips vs 10,000. The DB also optimizes the execution since it receives operations in bulk. Combine with `setAutoCommit(false)` and manual `commit()` per batch for transaction control.

---

## 17. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **Not closing connections** | Pool exhaustion, app hangs | Always use try-with-resources |
| **Using `Statement` with user input** | SQL injection vulnerability | Always use `PreparedStatement` |
| **Auto-commit in multi-step operations** | Partial updates committed, data inconsistency | Set `autoCommit=false`, explicitly commit/rollback |
| **Holding connections during external calls** | Pool starvation under load | Only borrow connection for DB work, not surrounding logic |
| **Not handling `SQLException` chain** | Missing root cause in logs | Walk `e.getNextException()` chain |
| **Wrong ResultSet type** | Scrollable RS uses server-side cursor, high memory | Use `TYPE_FORWARD_ONLY` unless you need scrollability |
| **`rs.getInt()` returns 0 for NULL** | Silent data corruption | Use `rs.wasNull()` after `rs.getInt()` for nullable columns |
| **`setDate()` uses `java.sql.Date`** | Timezone issues, loss of time portion | Use `setObject()` with `LocalDate` / `LocalDateTime` (JDBC 4.2+) |
| **Not setting fetch size** | Full table loaded into memory | `pstmt.setFetchSize(100)` for large result sets |

### The `wasNull()` gotcha:
```java
int stock = rs.getInt("stock_count");  // returns 0 if column is NULL
if (rs.wasNull()) {
    // stock_count was actually NULL in DB, not zero!
    stock = -1; // sentinel value for "not set"
}
```

### Fetch size for large results:
```java
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM audit_log WHERE created_at > ?",
    ResultSet.TYPE_FORWARD_ONLY,
    ResultSet.CONCUR_READ_ONLY
);
pstmt.setFetchSize(200); // JDBC fetches 200 rows at a time from DB, not all at once
```
Without `setFetchSize`, some drivers load ALL rows into memory. For a 10M-row audit log, that's an OOM.

---

## 18. Summary — Quick Revision

- **JDBC = standard interface**, Type 4 drivers implement it directly via native DB protocol. Never Type 1/2/3 in production.
- **Always use `PreparedStatement`** — prevents SQL injection, improves performance via execution plan caching.
- **Always use try-with-resources** — guarantees `ResultSet → Statement → Connection` are closed in order, even on exceptions.
- **`DriverManager` is for scripts, `DataSource` (HikariCP) is for production** — connection pooling is non-negotiable in web apps.
- **`conn.close()` on a pooled connection = return to pool**, not TCP disconnect.
- **Disable auto-commit for multi-statement atomicity** — `setAutoCommit(false)` + explicit `commit()`/`rollback()`.
- **Batch inserts** (`addBatch()` + `executeBatch()`) can yield 10–100x speedup for bulk operations.
- **`setFetchSize()`** prevents OOM when streaming large result sets.

---

## What's Next

You now understand raw JDBC. The next layer — **Spring JDBC** (`JdbcTemplate`) — eliminates all the boilerplate (try-with-resources, connection management, exception translation) while keeping you in full control of SQL. That's what we cover next.

> **Follow-up questions to think about:**
> 1. How would you implement pagination using raw JDBC? (`LIMIT`/`OFFSET` vs keyset pagination)
> 2. If two threads call `deductStock()` simultaneously on the same product — using only JDBC and no ORM — how do you guarantee exactly-once deduction?
> 3. What is the difference between `java.sql.Date`, `java.sql.Timestamp`, and `java.sql.Time`? Why are they problematic with Java 8 `LocalDate`/`LocalDateTime`?
