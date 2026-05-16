# Spring JDBC

> **Learning Path Position:** Raw JDBC → **Spring JDBC** → Spring Data JDBC → JPA/Hibernate → Spring Data JPA
>
> Spring JDBC is the first abstraction layer over raw JDBC. It keeps you in **full control of SQL** while eliminating all the boilerplate: connection management, statement creation, exception handling, and resource cleanup. If you know SQL and want complete query control without an ORM, Spring JDBC is the right tool.

---

## 1. Overview — What is it? Why does it exist?

Raw JDBC requires ~15 lines of boilerplate for a single `SELECT`:

```java
// Raw JDBC: find a product by ID
Connection conn = null;
PreparedStatement pstmt = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    pstmt = conn.prepareStatement("SELECT * FROM products WHERE id = ?");
    pstmt.setLong(1, id);
    rs = pstmt.executeQuery();
    if (rs.next()) {
        return new Product(rs.getLong("id"), rs.getString("name"), rs.getBigDecimal("price"));
    }
    return null;
} catch (SQLException e) {
    throw new RuntimeException(e);  // checked → unchecked, losing SQLState info
} finally {
    // close rs, pstmt, conn individually, each can throw...
}
```

Spring JDBC collapses this to:

```java
// Spring JDBC: same operation
return jdbcTemplate.queryForObject(
    "SELECT * FROM products WHERE id = ?",
    productRowMapper,
    id
);
```

**What Spring JDBC handles for you:**
1. Opens and closes connections (via `DataSource`)
2. Creates and closes `PreparedStatement`
3. Iterates `ResultSet`
4. Catches `SQLException` and translates to meaningful `DataAccessException` subclasses
5. Manages transactions (if you use Spring's `@Transactional`)

**What you still control:**
- Every SQL query — no magic generation
- How result rows map to objects (`RowMapper`)
- When and how to batch operations

---

## 2. Core Classes & Interfaces

| Class / Interface | Role |
|-------------------|------|
| `JdbcTemplate` | The central class. Covers 95% of use cases. |
| `NamedParameterJdbcTemplate` | Like `JdbcTemplate` but uses `:name` params instead of `?` |
| `SimpleJdbcInsert` | Fluent builder for INSERT statements |
| `SimpleJdbcCall` | Fluent builder for stored procedure calls |
| `RowMapper<T>` | Maps a single `ResultSet` row → object `T` |
| `ResultSetExtractor<T>` | Full control over `ResultSet` iteration, returns single result |
| `RowCallbackHandler` | Processes each row without a return value (streaming) |
| `DataAccessException` | Root of Spring's unchecked DB exception hierarchy |
| `JdbcOperations` | Interface that `JdbcTemplate` implements (good for mocking) |

---

## 3. JdbcTemplate — The Core

### 3.1 Setup

```java
@Configuration
public class DataConfig {

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        JdbcTemplate template = new JdbcTemplate(dataSource);
        template.setFetchSize(100);          // don't load entire ResultSet into memory
        template.setMaxRows(10_000);         // safety cap
        template.setQueryTimeout(30);        // 30s query timeout
        return template;
    }
}
```

In Spring Boot — `JdbcTemplate` is **auto-configured** the moment you have `spring-boot-starter-jdbc` and a `DataSource`. Just `@Autowired` it.

---

### 3.2 Query Methods

#### `queryForObject` — Single row, single value

```java
// Scalar value
int count = jdbcTemplate.queryForObject(
    "SELECT COUNT(*) FROM orders WHERE status = ?",
    Integer.class,
    "PENDING"
);

// Single object
Product product = jdbcTemplate.queryForObject(
    "SELECT id, name, price FROM products WHERE id = ?",
    (rs, rowNum) -> new Product(
        rs.getLong("id"),
        rs.getString("name"),
        rs.getBigDecimal("price")
    ),
    productId
);
```

> **Gotcha:** `queryForObject` throws `EmptyResultDataAccessException` if no rows found, and `IncorrectResultSizeDataAccessException` if more than one row found. Use `query()` + check size, or wrap in try-catch for optional semantics.

```java
// Safe Optional wrapper
public Optional<Product> findById(long id) {
    try {
        return Optional.ofNullable(
            jdbcTemplate.queryForObject(
                "SELECT * FROM products WHERE id = ?",
                productRowMapper,
                id
            )
        );
    } catch (EmptyResultDataAccessException e) {
        return Optional.empty();
    }
}
```

#### `query` — Multiple rows

```java
// Returns List<T>
List<Product> products = jdbcTemplate.query(
    "SELECT id, name, price FROM products WHERE category = ? ORDER BY name",
    productRowMapper,
    category
);

// With pagination
List<Order> orders = jdbcTemplate.query(
    "SELECT * FROM orders WHERE customer_id = ? ORDER BY created_at DESC LIMIT ? OFFSET ?",
    orderRowMapper,
    customerId, pageSize, pageSize * pageNumber
);
```

#### `queryForList` — When you want raw Maps (dynamic queries)

```java
// Each row is a Map<String, Object>
List<Map<String, Object>> rows = jdbcTemplate.queryForList(
    "SELECT * FROM audit_log WHERE entity_type = ?",
    "PRODUCT"
);
rows.forEach(row -> {
    String action = (String) row.get("action");
    Timestamp ts = (Timestamp) row.get("created_at");
});
```

---

### 3.3 Update Methods

```java
// INSERT
int rowsAffected = jdbcTemplate.update(
    "INSERT INTO products (sku, name, price, category) VALUES (?, ?, ?, ?)",
    product.getSku(), product.getName(), product.getPrice(), product.getCategory()
);

// UPDATE
int updated = jdbcTemplate.update(
    "UPDATE products SET price = ?, updated_at = NOW() WHERE id = ?",
    newPrice, productId
);

// DELETE
int deleted = jdbcTemplate.update(
    "DELETE FROM cart_items WHERE session_id = ? AND expires_at < NOW()",
    sessionId
);
```

### 3.4 INSERT with Generated Keys

```java
KeyHolder keyHolder = new GeneratedKeyHolder();

jdbcTemplate.update(connection -> {
    PreparedStatement ps = connection.prepareStatement(
        "INSERT INTO orders (customer_id, total, status) VALUES (?, ?, ?)",
        Statement.RETURN_GENERATED_KEYS
    );
    ps.setLong(1, order.getCustomerId());
    ps.setBigDecimal(2, order.getTotal());
    ps.setString(3, "PENDING");
    return ps;
}, keyHolder);

long generatedId = keyHolder.getKey().longValue();
```

---

## 4. RowMapper — The Object Mapping Contract

`RowMapper<T>` is a `@FunctionalInterface` with one method: `T mapRow(ResultSet rs, int rowNum)`.

### Reusable RowMapper as a constant

```java
@Repository
public class ProductRepository {

    // Define once, reuse across all query methods
    private static final RowMapper<Product> PRODUCT_ROW_MAPPER = (rs, rowNum) ->
        Product.builder()
            .id(rs.getLong("id"))
            .sku(rs.getString("sku"))
            .name(rs.getString("name"))
            .price(rs.getBigDecimal("price"))
            .stockCount(rs.getInt("stock_count"))
            .category(Category.valueOf(rs.getString("category")))
            .createdAt(rs.getTimestamp("created_at").toInstant())
            .active(rs.getBoolean("active"))
            .build();

    private final JdbcTemplate jdbcTemplate;

    public ProductRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<Product> findById(long id) {
        try {
            return Optional.of(jdbcTemplate.queryForObject(
                "SELECT * FROM products WHERE id = ? AND active = true",
                PRODUCT_ROW_MAPPER, id));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    public List<Product> findByCategory(Category category) {
        return jdbcTemplate.query(
            "SELECT * FROM products WHERE category = ? AND active = true ORDER BY name",
            PRODUCT_ROW_MAPPER,
            category.name()
        );
    }
}
```

### BeanPropertyRowMapper — Auto-mapping by column name

```java
// Maps column names to bean properties (snake_case → camelCase)
// customer_id → customerId, order_date → orderDate
RowMapper<Order> autoMapper = new BeanPropertyRowMapper<>(Order.class);

List<Order> orders = jdbcTemplate.query(
    "SELECT id, customer_id, order_date, total FROM orders",
    autoMapper
);
```

> **Gotcha:** `BeanPropertyRowMapper` uses reflection and requires a no-arg constructor. It's convenient but slower than a hand-written `RowMapper` and doesn't handle complex types well. Avoid it in performance-critical code.

---

## 5. ResultSetExtractor — Full ResultSet Control

Use when you need to process the `ResultSet` yourself — especially for one-to-many joins where multiple rows map to one object.

```java
// Scenario: load an Order with its LineItems in a single query (no N+1 problem)
private static final ResultSetExtractor<Order> ORDER_WITH_ITEMS_EXTRACTOR = rs -> {
    Order order = null;
    List<LineItem> items = new ArrayList<>();

    while (rs.next()) {
        if (order == null) {
            order = new Order(
                rs.getLong("o_id"),
                rs.getLong("o_customer_id"),
                rs.getBigDecimal("o_total"),
                rs.getString("o_status")
            );
        }
        // Each row has one line item (JOIN produces duplicate order columns)
        long itemId = rs.getLong("li_id");
        if (itemId > 0) {  // guard against outer join NULLs
            items.add(new LineItem(
                itemId,
                rs.getLong("li_product_id"),
                rs.getString("li_product_name"),
                rs.getInt("li_quantity"),
                rs.getBigDecimal("li_unit_price")
            ));
        }
    }
    if (order != null) order.setItems(items);
    return order;
};

public Order findOrderWithItems(long orderId) {
    String sql = """
        SELECT
            o.id        AS o_id,
            o.customer_id AS o_customer_id,
            o.total     AS o_total,
            o.status    AS o_status,
            li.id       AS li_id,
            li.product_id AS li_product_id,
            p.name      AS li_product_name,
            li.quantity AS li_quantity,
            li.unit_price AS li_unit_price
        FROM orders o
        LEFT JOIN line_items li ON li.order_id = o.id
        LEFT JOIN products p   ON p.id = li.product_id
        WHERE o.id = ?
        ORDER BY li.id
        """;

    return jdbcTemplate.query(sql, ORDER_WITH_ITEMS_EXTRACTOR, orderId);
}
```

This is the clean alternative to the N+1 problem without an ORM's lazy loading.

---

## 6. RowCallbackHandler — Streaming Large Results

When processing millions of rows (e.g., ETL, report generation), loading into a `List<T>` causes OOM. `RowCallbackHandler` processes row-by-row without accumulation.

```java
// Scenario: Nightly export of all transactions to a CSV file
public void exportTransactionsToCsv(OutputStream out) throws IOException {
    jdbcTemplate.setFetchSize(500);  // stream 500 rows at a time from DB
    BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(out));
    writer.write("id,customer_id,amount,status,created_at\n");

    jdbcTemplate.query(
        "SELECT id, customer_id, amount, status, created_at FROM transactions " +
        "WHERE created_at >= CURRENT_DATE - INTERVAL '1 day' ORDER BY id",
        rs -> {
            // Called once per row — no List accumulation
            writer.write(String.format("%d,%d,%.2f,%s,%s\n",
                rs.getLong("id"),
                rs.getLong("customer_id"),
                rs.getBigDecimal("amount"),
                rs.getString("status"),
                rs.getTimestamp("created_at")
            ));
        }
    );
    writer.flush();
}
```

---

## 7. NamedParameterJdbcTemplate — Named Parameters

`?` parameters become unmanageable with many parameters (easy to mix up position). `NamedParameterJdbcTemplate` uses `:paramName` syntax.

```java
@Repository
public class OrderRepository {

    private final NamedParameterJdbcTemplate namedJdbc;

    public OrderRepository(NamedParameterJdbcTemplate namedJdbc) {
        this.namedJdbc = namedJdbc;
    }

    public List<Order> searchOrders(OrderSearchCriteria criteria) {
        String sql = """
            SELECT * FROM orders
            WHERE customer_id = :customerId
              AND status IN (:statuses)
              AND total BETWEEN :minTotal AND :maxTotal
              AND created_at BETWEEN :from AND :to
            ORDER BY created_at DESC
            LIMIT :limit OFFSET :offset
            """;

        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("customerId", criteria.getCustomerId())
            .addValue("statuses", criteria.getStatuses())  // IN clause — List supported!
            .addValue("minTotal", criteria.getMinTotal())
            .addValue("maxTotal", criteria.getMaxTotal())
            .addValue("from", criteria.getFrom())
            .addValue("to", criteria.getTo())
            .addValue("limit", criteria.getPageSize())
            .addValue("offset", criteria.getPageSize() * criteria.getPage());

        return namedJdbc.query(sql, params, ORDER_ROW_MAPPER);
    }

    // BeanPropertySqlParameterSource — reads from bean properties automatically
    public int save(Order order) {
        String sql = """
            INSERT INTO orders (customer_id, total, status, created_at)
            VALUES (:customerId, :total, :status, :createdAt)
            """;
        // Reads order.getCustomerId(), order.getTotal(), etc.
        return namedJdbc.update(sql, new BeanPropertySqlParameterSource(order));
    }
}
```

> **Key advantage of named params:** `IN (:statuses)` with a `List` — Spring automatically expands `(:statuses)` to `(?, ?, ?)` based on list size. This is not possible with plain `JdbcTemplate`.

---

## 8. SimpleJdbcInsert — Fluent INSERT Builder

Eliminates the need to write INSERT SQL for simple cases.

```java
@Repository
public class CustomerRepository {

    private final SimpleJdbcInsert insertCustomer;

    public CustomerRepository(DataSource dataSource) {
        this.insertCustomer = new SimpleJdbcInsert(dataSource)
            .withTableName("customers")
            .usingGeneratedKeyColumns("id")  // auto-capture generated PK
            .usingColumns("name", "email", "phone", "tier"); // explicit columns (safe)
    }

    public long save(Customer customer) {
        Map<String, Object> params = new HashMap<>();
        params.put("name", customer.getName());
        params.put("email", customer.getEmail());
        params.put("phone", customer.getPhone());
        params.put("tier", customer.getTier().name());

        Number key = insertCustomer.executeAndReturnKey(params);
        return key.longValue();
    }
}
```

---

## 9. SimpleJdbcCall — Stored Procedure Calls

```java
@Repository
public class ReportRepository {

    private final SimpleJdbcCall calculateRevenue;

    public ReportRepository(DataSource dataSource) {
        this.calculateRevenue = new SimpleJdbcCall(dataSource)
            .withProcedureName("calculate_revenue")
            .declareParameters(
                new SqlParameter("p_from_date", Types.DATE),
                new SqlParameter("p_to_date", Types.DATE),
                new SqlOutParameter("p_total_revenue", Types.DECIMAL),
                new SqlOutParameter("p_order_count", Types.INTEGER)
            );
    }

    public RevenueReport getRevenue(LocalDate from, LocalDate to) {
        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("p_from_date", Date.valueOf(from))
            .addValue("p_to_date", Date.valueOf(to));

        Map<String, Object> result = calculateRevenue.execute(params);

        return new RevenueReport(
            (BigDecimal) result.get("p_total_revenue"),
            (Integer) result.get("p_order_count")
        );
    }
}
```

---

## 10. Batch Operations

### JdbcTemplate.batchUpdate

```java
// Scenario: Bulk price update from a pricing engine (10,000 products)
public void bulkUpdatePrices(List<PriceUpdate> updates) {
    jdbcTemplate.batchUpdate(
        "UPDATE products SET price = ?, updated_at = NOW() WHERE sku = ?",
        new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                PriceUpdate u = updates.get(i);
                ps.setBigDecimal(1, u.getNewPrice());
                ps.setString(2, u.getSku());
            }
            @Override
            public int getBatchSize() {
                return updates.size();
            }
        }
    );
}
```

### NamedParameterJdbcTemplate batch

```java
// More readable with named params
public void bulkInsertAuditLog(List<AuditEvent> events) {
    String sql = """
        INSERT INTO audit_log (entity_type, entity_id, action, actor_id, occurred_at)
        VALUES (:entityType, :entityId, :action, :actorId, :occurredAt)
        """;

    SqlParameterSource[] batchParams = events.stream()
        .map(BeanPropertySqlParameterSource::new)
        .toArray(SqlParameterSource[]::new);

    namedJdbc.batchUpdate(sql, batchParams);
}
```

### Chunked batch with transaction control

```java
// Scenario: Migrating 2 million old records to new schema
@Transactional
public void migrateOrdersInChunks() {
    int chunkSize = 1000;
    int offset = 0;
    int migrated;

    do {
        final int currentOffset = offset;
        List<Map<String, Object>> chunk = jdbcTemplate.queryForList(
            "SELECT * FROM orders_legacy ORDER BY id LIMIT ? OFFSET ?",
            chunkSize, currentOffset
        );

        if (chunk.isEmpty()) break;

        jdbcTemplate.batchUpdate(
            "INSERT INTO orders_v2 (id, customer_id, total, status, created_at) " +
            "VALUES (?, ?, ?, ?, ?)",
            new BatchPreparedStatementSetter() {
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    Map<String, Object> row = chunk.get(i);
                    ps.setLong(1, (Long) row.get("id"));
                    ps.setLong(2, (Long) row.get("customer_id"));
                    ps.setBigDecimal(3, (BigDecimal) row.get("total"));
                    ps.setString(4, (String) row.get("status"));
                    ps.setTimestamp(5, (Timestamp) row.get("created_at"));
                }
                public int getBatchSize() { return chunk.size(); }
            }
        );

        migrated = chunk.size();
        offset += migrated;
        log.info("Migrated {} records (total: {})", migrated, offset);

    } while (true);
}
```

---

## 11. DataAccessException — Spring's Exception Hierarchy

This is one of the most important things Spring JDBC does: translates opaque `SQLException` (with vendor-specific error codes) into a meaningful, **unchecked** exception hierarchy you can catch selectively.

```
DataAccessException (unchecked — RuntimeException)
├── NonTransientDataAccessException  ← retrying won't help
│   ├── DataIntegrityViolationException   ← duplicate key, FK violation, NOT NULL
│   ├── BadSqlGrammarException            ← SQL syntax error
│   ├── TypeMismatchDataAccessException   ← wrong type in result
│   └── PermissionDeniedDataAccessException ← missing DB permission
├── TransientDataAccessException     ← retrying may help
│   ├── TransientDataAccessResourceException ← DB temporarily unavailable
│   ├── ConcurrencyFailureException        ← deadlock or serialization failure
│   └── QueryTimeoutException              ← query exceeded timeout
├── RecoverableDataAccessException   ← can recover by reconnecting
└── EmptyResultDataAccessException   ← queryForObject returned 0 rows
    IncorrectResultSizeDataAccessException ← queryForObject returned >1 rows
```

### In practice:

```java
@Service
public class ProductService {

    public void createProduct(Product product) {
        try {
            productRepository.save(product);
        } catch (DataIntegrityViolationException e) {
            // Duplicate SKU — a business logic concern, not a crash
            throw new DuplicateSkuException("SKU already exists: " + product.getSku(), e);
        } catch (QueryTimeoutException e) {
            // DB under load — return a user-friendly error, alert monitoring
            log.error("Product save timed out for SKU: {}", product.getSku());
            throw new ServiceUnavailableException("Database is busy, try again shortly");
        }
        // No need to catch SQLException — Spring translates it automatically
    }
}
```

### How translation works internally

```
Spring uses SQLErrorCodeSQLExceptionTranslator:
1. Reads sql-error-codes.xml (bundled in spring-jdbc jar)
2. Matches e.getErrorCode() (vendor-specific int) → DataAccessException subclass
3. Falls back to SQLStateSQLExceptionTranslator using e.getSQLState() (standard 5-char code)

Examples:
  MySQL 1062 (Duplicate entry)  → DataIntegrityViolationException
  MySQL 1205 (Lock wait timeout) → TransientDataAccessResourceException
  Postgres 23505 (unique_violation) → DataIntegrityViolationException
  Postgres 40001 (serialization_failure) → ConcurrencyFailureException
```

---

## 12. Transaction Management with Spring JDBC

Spring JDBC integrates seamlessly with `@Transactional`. The `JdbcTemplate` automatically participates in the current Spring transaction — it borrows the transaction-bound connection from the `DataSourceTransactionManager`.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;
    private final InventoryRepository inventoryRepo;
    private final PaymentRepository paymentRepo;

    @Transactional
    public OrderResult placeOrder(OrderRequest request) {
        // All three repository calls share THE SAME connection/transaction
        // JdbcTemplate detects the active transaction and uses its connection

        // 1. Deduct inventory (fails if insufficient stock)
        boolean deducted = inventoryRepo.deductStock(
            request.getProductId(), request.getQuantity()
        );
        if (!deducted) throw new OutOfStockException(request.getProductId());

        // 2. Create order record
        long orderId = orderRepo.createOrder(
            request.getCustomerId(), request.getProductId(),
            request.getQuantity(), request.getTotalAmount()
        );

        // 3. Record payment intent
        paymentRepo.createPaymentRecord(orderId, request.getTotalAmount());

        return new OrderResult(orderId, "CREATED");
        // @Transactional commits all 3 operations atomically
        // Any exception → all 3 rolled back
    }
}
```

```java
@Repository
public class InventoryRepository {

    private final JdbcTemplate jdbcTemplate;

    public boolean deductStock(long productId, int quantity) {
        // JdbcTemplate auto-joins the @Transactional transaction
        // Uses TransactionSynchronizationManager to get bound connection
        int rows = jdbcTemplate.update(
            "UPDATE inventory SET quantity = quantity - ? " +
            "WHERE product_id = ? AND quantity >= ?",
            quantity, productId, quantity
        );
        return rows == 1;
    }
}
```

### Programmatic transaction management (when @Transactional isn't enough)

```java
@Service
public class BulkImportService {

    private final TransactionTemplate transactionTemplate;
    private final ProductRepository productRepo;

    public BulkImportService(PlatformTransactionManager txManager, ProductRepository productRepo) {
        this.transactionTemplate = new TransactionTemplate(txManager);
        this.productRepo = productRepo;
    }

    public ImportResult importProducts(List<ProductCsv> csvRows) {
        List<String> failures = new ArrayList<>();
        int successCount = 0;

        for (ProductCsv row : csvRows) {
            try {
                // Each product in its OWN transaction — failure of one doesn't affect others
                transactionTemplate.executeWithoutResult(status -> {
                    productRepo.save(row.toProduct());
                    productRepo.saveAttributes(row.toAttributes());
                });
                successCount++;
            } catch (DataIntegrityViolationException e) {
                failures.add("SKU " + row.getSku() + ": duplicate");
            }
        }
        return new ImportResult(successCount, failures);
    }
}
```

---

## 13. Complete Production Example — E-commerce Product Service

```java
@Repository
public class ProductRepository {

    private static final RowMapper<Product> ROW_MAPPER = (rs, n) ->
        Product.builder()
            .id(rs.getLong("id"))
            .sku(rs.getString("sku"))
            .name(rs.getString("name"))
            .description(rs.getString("description"))
            .price(rs.getBigDecimal("price"))
            .stockCount(rs.getInt("stock_count"))
            .category(rs.getString("category"))
            .brand(rs.getString("brand"))
            .active(rs.getBoolean("active"))
            .createdAt(rs.getTimestamp("created_at").toInstant())
            .build();

    private final JdbcTemplate jdbc;
    private final NamedParameterJdbcTemplate namedJdbc;
    private final SimpleJdbcInsert insertProduct;

    public ProductRepository(JdbcTemplate jdbc,
                             NamedParameterJdbcTemplate namedJdbc,
                             DataSource dataSource) {
        this.jdbc = jdbc;
        this.namedJdbc = namedJdbc;
        this.insertProduct = new SimpleJdbcInsert(dataSource)
            .withTableName("products")
            .usingGeneratedKeyColumns("id");
    }

    // ── READ ─────────────────────────────────────────────────────────────────

    public Optional<Product> findById(long id) {
        try {
            return Optional.of(jdbc.queryForObject(
                "SELECT * FROM products WHERE id = ? AND active = true",
                ROW_MAPPER, id));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    public Optional<Product> findBySku(String sku) {
        try {
            return Optional.of(jdbc.queryForObject(
                "SELECT * FROM products WHERE sku = ?",
                ROW_MAPPER, sku));
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    public Page<Product> search(ProductSearchRequest req) {
        String countSql = """
            SELECT COUNT(*) FROM products
            WHERE (:category IS NULL OR category = :category)
              AND (:brand IS NULL OR brand = :brand)
              AND (:minPrice IS NULL OR price >= :minPrice)
              AND (:maxPrice IS NULL OR price <= :maxPrice)
              AND (:keyword IS NULL OR name ILIKE '%' || :keyword || '%')
              AND active = true
            """;

        String dataSql = countSql.replace("COUNT(*)", "id, sku, name, description, " +
                                          "price, stock_count, category, brand, active, created_at") +
            " ORDER BY " + req.getSortField() + " " + req.getSortDir() +
            " LIMIT :limit OFFSET :offset";

        MapSqlParameterSource params = new MapSqlParameterSource()
            .addValue("category", req.getCategory())
            .addValue("brand", req.getBrand())
            .addValue("minPrice", req.getMinPrice())
            .addValue("maxPrice", req.getMaxPrice())
            .addValue("keyword", req.getKeyword())
            .addValue("limit", req.getPageSize())
            .addValue("offset", req.getPageSize() * req.getPage());

        int total = namedJdbc.queryForObject(countSql, params, Integer.class);
        List<Product> items = namedJdbc.query(dataSql, params, ROW_MAPPER);

        return new Page<>(items, total, req.getPage(), req.getPageSize());
    }

    // ── WRITE ────────────────────────────────────────────────────────────────

    public long save(Product product) {
        Map<String, Object> params = new HashMap<>();
        params.put("sku", product.getSku());
        params.put("name", product.getName());
        params.put("description", product.getDescription());
        params.put("price", product.getPrice());
        params.put("stock_count", product.getStockCount());
        params.put("category", product.getCategory());
        params.put("brand", product.getBrand());
        params.put("active", true);
        return insertProduct.executeAndReturnKey(params).longValue();
    }

    public boolean deductStock(long productId, int quantity) {
        return jdbc.update(
            "UPDATE products SET stock_count = stock_count - ? " +
            "WHERE id = ? AND stock_count >= ? AND active = true",
            quantity, productId, quantity
        ) == 1;
    }

    public void bulkDeactivate(List<Long> productIds) {
        namedJdbc.update(
            "UPDATE products SET active = false, updated_at = NOW() WHERE id IN (:ids)",
            new MapSqlParameterSource("ids", productIds)
        );
    }

    // ── BATCH ────────────────────────────────────────────────────────────────

    public void bulkUpdatePrices(List<PriceUpdate> updates) {
        namedJdbc.batchUpdate(
            "UPDATE products SET price = :newPrice, updated_at = NOW() WHERE sku = :sku",
            updates.stream()
                .map(BeanPropertySqlParameterSource::new)
                .toArray(SqlParameterSource[]::new)
        );
    }
}
```

---

## 14. Scenario Walkthroughs

### Scenario 1: N+1 problem — solved with a JOIN + ResultSetExtractor

**Wrong approach (N+1):**
```java
// 1 query for orders + N queries for line items = N+1 queries
List<Order> orders = orderRepo.findByCustomerId(customerId); // 1 query
orders.forEach(o ->
    o.setItems(lineItemRepo.findByOrderId(o.getId()))        // N queries
);
```

**Right approach with Spring JDBC:**
```java
// 1 query total using JOIN + ResultSetExtractor
public List<Order> findOrdersWithItemsByCustomer(long customerId) {
    String sql = """
        SELECT
            o.id AS o_id, o.total AS o_total, o.status AS o_status,
            li.id AS li_id, li.product_id, li.quantity, li.unit_price, p.name AS product_name
        FROM orders o
        LEFT JOIN line_items li ON li.order_id = o.id
        LEFT JOIN products p ON p.id = li.product_id
        WHERE o.customer_id = ?
        ORDER BY o.id, li.id
        """;

    return jdbc.query(sql, rs -> {
        Map<Long, Order> orderMap = new LinkedHashMap<>();
        while (rs.next()) {
            long orderId = rs.getLong("o_id");
            Order order = orderMap.computeIfAbsent(orderId, id ->
                new Order(id, rs.getBigDecimal("o_total"), rs.getString("o_status"), new ArrayList<>())
            );
            long itemId = rs.getLong("li_id");
            if (itemId > 0) {
                order.getItems().add(new LineItem(
                    itemId, rs.getLong("product_id"),
                    rs.getString("product_name"),
                    rs.getInt("quantity"),
                    rs.getBigDecimal("unit_price")
                ));
            }
        }
        return new ArrayList<>(orderMap.values());
    }, customerId);
}
```

### Scenario 2: Handling duplicate key gracefully in a product import API

```java
@Service
public class ProductImportService {

    @Transactional
    public ImportResult importFromCsv(MultipartFile file) {
        List<ProductCsv> rows = parseCsv(file);
        int inserted = 0, skipped = 0;

        for (ProductCsv row : rows) {
            try {
                productRepo.save(row.toProduct());
                inserted++;
            } catch (DataIntegrityViolationException e) {
                // SKU already exists — skip, don't fail entire import
                log.warn("Skipping duplicate SKU: {}", row.getSku());
                skipped++;
            }
        }
        return new ImportResult(inserted, skipped);
    }
}
```

---

## 15. Common Interview Questions

**Q1: What does `JdbcTemplate` do that raw JDBC doesn't?**

It handles the entire lifecycle of connection acquisition, `PreparedStatement` creation, `ResultSet` iteration, exception translation (from `SQLException` → `DataAccessException`), and resource cleanup — all in try-with-resources-style internally. You provide only the SQL and the mapping logic.

---

**Q2: What is the difference between `RowMapper`, `ResultSetExtractor`, and `RowCallbackHandler`?**

- `RowMapper<T>`: called once per row, returns an object. Spring collects results into a `List<T>`. Best for simple row-to-object mapping.
- `ResultSetExtractor<T>`: you get the full `ResultSet` cursor and return a single result (could be an object, a map, a list). Best for one-to-many JOIN results where multiple rows represent one aggregate.
- `RowCallbackHandler`: called once per row with no return value. Best for streaming processing (writing to a file/stream) where accumulating into a List would cause OOM.

---

**Q3: How does `JdbcTemplate` participate in a `@Transactional` transaction?**

`JdbcTemplate` uses `DataSourceUtils.getConnection(dataSource)` to get a connection. `DataSourceUtils` checks the `TransactionSynchronizationManager` for a transaction-bound connection on the current thread. If `@Transactional` is active, a connection is bound to the thread — `JdbcTemplate` reuses it. If not, it gets a fresh connection from the pool. This is how multiple repositories share one transaction transparently.

---

**Q4: What is `DataAccessException`? Why is it unchecked?**

It's the root of Spring's database exception hierarchy. It's a `RuntimeException` (unchecked) for two reasons: (1) `SQLException` is checked but most callers can't meaningfully recover from it, so forcing every caller to declare `throws SQLException` is pure boilerplate; (2) Spring's hierarchy is meaningful — `DataIntegrityViolationException`, `QueryTimeoutException`, `ConcurrencyFailureException` — giving callers the option to catch specific cases when needed, without forcing it everywhere.

---

**Q5: What is the advantage of `NamedParameterJdbcTemplate` over `JdbcTemplate`?**

Named parameters (`:name`) are readable and order-independent — unlike positional `?` where index bugs are common. The critical advantage is `IN` clause support: passing a `List` for `:ids` automatically expands to `(?, ?, ?, ...)`. With plain `JdbcTemplate` you'd have to manually build the placeholder string.

---

**Q6: When would you use `TransactionTemplate` instead of `@Transactional`?**

When transaction boundaries are dynamic — e.g., when you want to commit each item in a bulk import independently (so one failure doesn't roll back the entire batch), or when you're writing code in a context where AOP proxies don't apply (calling a method on `this`, utility classes not managed by Spring, or lambda-based processing). `TransactionTemplate` gives programmatic control of the exact commit/rollback boundary.

---

## 16. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **`queryForObject` on 0 rows** | Throws `EmptyResultDataAccessException` | Wrap in try-catch and return `Optional.empty()` |
| **`queryForObject` on >1 rows** | Throws `IncorrectResultSizeDataAccessException` | Use `query()` and check list size |
| **Not setting `fetchSize`** | Large queries load all rows into memory → OOM | `jdbcTemplate.setFetchSize(N)` or per-query via `PreparedStatementSetter` |
| **Self-invocation breaks `@Transactional`** | Calling a `@Transactional` method on `this` bypasses the proxy — no transaction | Inject self or use `TransactionTemplate` programmatically |
| **`BeanPropertyRowMapper` with `is` prefix** | Booleans with `isActive()` may not map correctly depending on Spring version | Use explicit `RowMapper` for reliability |
| **Dynamic sort field in SQL** | `ORDER BY :sortField` doesn't work — column names can't be parameters | Whitelist allowed values and concatenate safely: `"ORDER BY " + ALLOWED_SORT_FIELDS.get(field)` |
| **SQL injection via string concatenation** | Dynamic column names/table names can't use `?` | Always validate against a whitelist before concatenating |
| **Forgetting `namedJdbc` has no `batchUpdate` with `BatchPreparedStatementSetter`** | Compile error | Use `SqlParameterSource[]` array with `namedJdbc.batchUpdate()` |

---

## 17. Summary — Quick Revision

- `JdbcTemplate` = raw JDBC minus all boilerplate. You write SQL, it handles connections, statements, and resource cleanup.
- Use `RowMapper` for row-to-object, `ResultSetExtractor` for join aggregation, `RowCallbackHandler` for streaming large result sets.
- `NamedParameterJdbcTemplate` for readable params and `IN` clause support with `List`.
- `DataAccessException` hierarchy = meaningful, unchecked exceptions that let you catch only what you can handle.
- `JdbcTemplate` automatically participates in `@Transactional` — multiple repos share one connection via `TransactionSynchronizationManager`.
- Batch via `batchUpdate()` — 10–100x faster than individual updates for bulk operations.
- `SimpleJdbcInsert` for fluent INSERT with auto-generated key capture.
- Spring JDBC is the right choice when you want full SQL control, complex joins, or are working with a legacy schema that doesn't map cleanly to JPA entities.

---

> **What's next:** Spring Data JDBC — adds a repository abstraction (`CrudRepository`, `@Query`) on top of Spring JDBC, with domain-driven design concepts (Aggregate Root, one-way relationships) but deliberately no lazy loading, no dirty tracking, no Hibernate magic.

> **Follow-up questions to think about:**
> 1. How would you unit test a `@Repository` class that uses `JdbcTemplate`? What do you mock?
> 2. Two `@Transactional` service methods call each other — both use `JdbcTemplate`. How many DB connections are used?
> 3. How do you safely build a dynamic `ORDER BY` clause without risking SQL injection?
