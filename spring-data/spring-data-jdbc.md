# Spring Data JDBC

> **Learning Path Position:** Raw JDBC → Spring JDBC → **Spring Data JDBC** → JPA/Hibernate → Spring Data JPA
>
> Spring Data JDBC is a **deliberate, opinionated simplification** of the Spring Data family. It gives you the repository abstraction (`CrudRepository`, `@Query`) from Spring Data, but makes a hard architectural choice: **no lazy loading, no dirty tracking, no session, no proxy magic**. What you see is what you get. This makes it fast, predictable, and easy to reason about — at the cost of some flexibility.

---

## 1. Overview — What is it? Why does it exist?

Spring Data JPA (with Hibernate) is powerful but complex. It has:
- **Lazy loading** — objects that secretly hit the DB when you access a field
- **Dirty checking** — Hibernate tracks every managed entity for changes
- **First-level cache (session)** — entities are cached per transaction
- **Proxy objects** — your `@Entity` might actually be a CGLIB subclass
- **Session lifecycle** — `EntityManager`, `@Transactional`, `LazyInitializationException`

These features solve real problems but introduce hidden complexity, subtle bugs, and hard-to-predict behavior.

**Spring Data JDBC philosophy:**
> *"An ORM should not be smarter than the developer using it."*

Spring Data JDBC gives you:
- Repository abstraction (same `CrudRepository`, `JpaRepository`-style API)
- Automatic SQL generation for CRUD
- `@Query` for custom SQL
- Domain-driven design support (Aggregate Roots)
- Spring application events for hooks
- **No lazy loading. No dirty tracking. No session. No proxy. No surprises.**

**When to choose Spring Data JDBC over Spring Data JPA:**
- You want explicit control over every SQL that runs
- Your domain model follows DDD (Aggregate Roots)
- You're building microservices with small, focused aggregates
- You've been burned by `LazyInitializationException` or N+1 in JPA
- You need predictable, measurable performance

---

## 2. Architecture — How it Works Internally

```
Spring Data JDBC Architecture:

Your Code
    │
    ▼
Repository Interface (CrudRepository<Product, Long>)
    │
    ▼  Spring generates implementation at startup (JDK proxy)
SimpleJdbcRepository  ← default implementation
    │
    ▼
RelationalEntityInformation + MappingContext
    │  (knows table name, column mappings, relationships from your entity class)
    ▼
DataAccessStrategy (DefaultDataAccessStrategy)
    │
    ▼
JdbcConverter  ← converts Java types ↔ JDBC types
    │
    ▼
NamedParameterJdbcOperations  ← uses NamedParameterJdbcTemplate under the hood
    │
    ▼
DataSource (HikariCP)
    │
    ▼
Database
```

**Key design decisions:**
1. **Load everything eagerly** — when you load an aggregate, all referenced entities are loaded with it.
2. **Save the entire aggregate** — `save()` always replaces the aggregate (delete children, re-insert).
3. **No identity map** — the same row loaded twice = two separate Java objects.
4. **No proxy objects** — your entity class is your entity class.

---

## 3. Dependency Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
    username: app_user
    password: secret
    hikari:
      maximum-pool-size: 10
  sql:
    init:
      mode: always  # run schema.sql on startup (dev only)
```

---

## 4. Key Annotations & Classes

| Annotation / Class | Role |
|--------------------|------|
| `@Table("table_name")` | Maps entity to a DB table (optional if name matches) |
| `@Id` | Marks the primary key field |
| `@Column("column_name")` | Maps field to a specific column name |
| `@Transient` | Field is NOT persisted to DB |
| `@MappedCollection` | Maps a `List` or `Set` of child entities (one-to-many) |
| `@Embedded` | Embeds a value object's fields into the parent table |
| `@ReadOnlyProperty` | Field is read from DB but never written |
| `@Version` | Optimistic locking version field |
| `@CreatedDate`, `@LastModifiedDate` | Auditing (with `@EnableJdbcAuditing`) |
| `@CreatedBy`, `@LastModifiedBy` | Auditing with user info |
| `@Query` | Custom SQL on repository methods |
| `@Modifying` | Required on `@Query` methods that mutate data |
| `CrudRepository<T, ID>` | Basic CRUD operations |
| `PagingAndSortingRepository<T, ID>` | Adds pagination and sorting |
| `ListCrudRepository<T, ID>` | Returns `List` instead of `Iterable` (Spring Data 3+) |
| `JdbcAggregateTemplate` | Low-level programmatic access to Spring Data JDBC ops |

---

## 5. Domain Model — The Aggregate Root Concept

This is the most important concept to understand. Spring Data JDBC is built around **DDD (Domain-Driven Design) Aggregates**.

### Aggregate Root
- The **entry point** to a cluster of related entities.
- Only the root has a repository.
- All access to child entities goes **through the root**.
- **Saving the root saves the entire aggregate.**
- **Deleting the root deletes the entire aggregate.**

```
                  ┌─────────────────────┐
                  │   Order (ROOT)      │  ← has a repository
                  │   id, customerId,   │
                  │   status, total     │
                  └────────┬────────────┘
                           │ 1:N
              ┌────────────▼─────────────┐
              │      LineItem            │  ← NO separate repository
              │  orderId, productId,     │     accessed only via Order
              │  quantity, unitPrice     │
              └──────────────────────────┘
```

```java
// Schema
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status      VARCHAR(50) NOT NULL,
    total       DECIMAL(19, 2) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE line_item (
    id         BIGSERIAL PRIMARY KEY,
    order      BIGINT NOT NULL REFERENCES orders(id),  -- column name must match root entity field name
    product_id BIGINT NOT NULL,
    quantity   INT NOT NULL,
    unit_price DECIMAL(19, 2) NOT NULL
);
```

> **Important:** The FK column in `line_item` is named `order` — Spring Data JDBC uses the field name in the aggregate root to construct the FK column name. If your root field is `private List<LineItem> items`, the FK column would be `items`.

```java
// Aggregate Root
@Table("orders")
public class Order {

    @Id
    private Long id;

    private Long customerId;

    private String status;

    private BigDecimal total;

    private Instant createdAt;

    // One-to-many: Spring Data JDBC owns this collection
    // Line items have no separate repository — they belong to this aggregate
    @MappedCollection(idColumn = "order", keyColumn = "position")
    private List<LineItem> items = new ArrayList<>();

    // Domain behavior — keep logic in the aggregate
    public void addItem(LineItem item) {
        this.items.add(item);
        recalculateTotal();
    }

    public void cancel() {
        if ("SHIPPED".equals(this.status)) {
            throw new IllegalStateException("Cannot cancel a shipped order");
        }
        this.status = "CANCELLED";
    }

    private void recalculateTotal() {
        this.total = items.stream()
            .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Child entity — NOT a root, no repository
public class LineItem {
    @Id
    private Long id;
    private Long productId;
    private int quantity;
    private BigDecimal unitPrice;
    // No back-reference to Order — DDD style
}
```

---

## 6. Repository Interface

```java
// Basic: extend CrudRepository
public interface OrderRepository extends CrudRepository<Order, Long> {
    // Spring Data JDBC auto-generates SQL for basic CRUD:
    // save(), findById(), findAll(), deleteById(), existsById(), count()
}

// With pagination and sorting
public interface ProductRepository extends
        PagingAndSortingRepository<Product, Long>,
        CrudRepository<Product, Long> {

    // Derived query methods (Spring Data parses method name → SQL)
    List<Product> findByCategory(String category);
    List<Product> findByCategoryAndActiveTrue(String category);
    Optional<Product> findBySku(String sku);
    boolean existsBySku(String sku);
    long countByCategory(String category);
    List<Product> findTop10ByOrderByCreatedAtDesc();

    // Pagination
    Page<Product> findByCategory(String category, Pageable pageable);
}
```

### CRUD operations:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepo;

    @Transactional
    public Order placeOrder(PlaceOrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setStatus("PENDING");
        order.setCreatedAt(Instant.now());

        request.getItems().forEach(item ->
            order.addItem(new LineItem(item.getProductId(), item.getQuantity(), item.getPrice()))
        );

        // save() → INSERT into orders + INSERT into line_item for each item
        return orderRepo.save(order);
    }

    @Transactional
    public Order cancelOrder(long orderId) {
        Order order = orderRepo.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.cancel();  // domain logic in aggregate

        // save() → UPDATE orders SET status='CANCELLED'
        // Spring Data JDBC diffs the aggregate:
        //   - order status changed → UPDATE orders
        //   - line items unchanged → no re-insert of line items
        return orderRepo.save(order);
    }

    @Transactional
    public Order addItemToOrder(long orderId, AddItemRequest request) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.addItem(new LineItem(request.getProductId(), request.getQuantity(), request.getPrice()));

        // save() → UPDATE orders (new total), INSERT INTO line_item (new item)
        // Spring Data JDBC figures out the diff for you
        return orderRepo.save(order);
    }
}
```

---

## 7. How `save()` Works Internally

This is critical to understand — it's very different from JPA.

```
save(order) call:

1. Is order.getId() null?
   YES → INSERT  (new entity)
   NO  → UPDATE + replace children

For UPDATE of an aggregate with children:
  a. UPDATE orders SET status=?, total=? WHERE id=?
  b. DELETE FROM line_item WHERE order=?   ← DELETE ALL existing children
  c. INSERT INTO line_item ...              ← RE-INSERT all current children

Spring Data JDBC does NOT track which line items changed.
It always deletes all and re-inserts all.
```

**Implication:** If your aggregate has 1000 child records and you update one field on the root, Spring Data JDBC will delete 1000 rows and re-insert 1000 rows. For large aggregates, this is a performance concern.

**Fix:** Keep aggregates small. A `Cart` with 50 items is fine. An `Order` with 10,000 historical events is not a good aggregate boundary.

---

## 8. Custom Queries with `@Query`

```java
public interface ProductRepository extends CrudRepository<Product, Long> {

    // Custom SELECT — returns domain objects
    @Query("SELECT * FROM products WHERE category = :category AND active = true ORDER BY name")
    List<Product> findActiveByCategory(@Param("category") String category);

    // Pagination with custom query
    @Query("SELECT * FROM products WHERE price BETWEEN :min AND :max AND active = true")
    Page<Product> findByPriceRange(
        @Param("min") BigDecimal min,
        @Param("max") BigDecimal max,
        Pageable pageable
    );

    // Projection — return only specific columns (use interface projection)
    @Query("SELECT id, sku, name, price FROM products WHERE brand = :brand")
    List<ProductSummary> findSummaryByBrand(@Param("brand") String brand);

    // Custom UPDATE — requires @Modifying
    @Modifying
    @Query("UPDATE products SET stock_count = stock_count - :qty WHERE id = :id AND stock_count >= :qty")
    int deductStock(@Param("id") long id, @Param("qty") int qty);

    // Custom DELETE
    @Modifying
    @Query("DELETE FROM products WHERE active = false AND created_at < :cutoff")
    int deleteInactiveOlderThan(@Param("cutoff") Instant cutoff);

    // Native SQL with complex JOINs
    @Query("""
        SELECT p.id, p.sku, p.name, p.price,
               COUNT(oi.id) AS total_sold
        FROM products p
        LEFT JOIN order_items oi ON oi.product_id = p.id
        WHERE p.category = :category
        GROUP BY p.id, p.sku, p.name, p.price
        ORDER BY total_sold DESC
        LIMIT :limit
        """)
    List<ProductSalesView> findTopSellersByCategory(
        @Param("category") String category,
        @Param("limit") int limit
    );
}

// Projection interface — only the fields you need
public interface ProductSummary {
    Long getId();
    String getSku();
    String getName();
    BigDecimal getPrice();
}
```

---

## 9. Derived Query Methods

Spring Data JDBC supports method name parsing just like Spring Data JPA:

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {

    // findBy + field name
    Optional<Customer> findByEmail(String email);
    List<Customer> findByLastName(String lastName);

    // AND / OR
    List<Customer> findByFirstNameAndLastName(String first, String last);
    List<Customer> findByEmailOrPhone(String email, String phone);

    // Comparison operators
    List<Customer> findByAgeLessThan(int age);
    List<Customer> findByAgeGreaterThanEqual(int age);
    List<Customer> findByCreatedAtBetween(Instant from, Instant to);

    // Null checks
    List<Customer> findByPhoneIsNull();
    List<Customer> findByEmailIsNotNull();

    // String matching
    List<Customer> findByLastNameLike(String pattern);           // LIKE :pattern
    List<Customer> findByEmailContaining(String substring);      // LIKE %substring%
    List<Customer> findByFirstNameStartingWith(String prefix);   // LIKE prefix%

    // Boolean
    List<Customer> findByActiveTrue();
    List<Customer> findByActiveFalse();

    // IN
    List<Customer> findByTierIn(List<String> tiers);

    // Limiting
    List<Customer> findTop5ByOrderByCreatedAtDesc();
    Optional<Customer> findFirstByEmailOrderByCreatedAtDesc(String email);

    // Existence and count
    boolean existsByEmail(String email);
    long countByTier(String tier);

    // Delete
    void deleteByEmail(String email);
    int deleteByActiveFalse();
}
```

---

## 10. Embedded Value Objects

Map a value object's fields into the parent table:

```java
// Value object — no @Id, not a separate table
public class Address {
    private String street;
    private String city;
    private String state;
    private String zipCode;
    private String country;
}

@Table("customers")
public class Customer {
    @Id
    private Long id;
    private String name;
    private String email;

    // Address fields are stored in the customers table:
    // street, city, state, zip_code, country
    @Embedded(onEmpty = Embedded.OnEmpty.USE_NULL)
    private Address address;
}

// Table: customers(id, name, email, street, city, state, zip_code, country)
```

With column prefix:
```java
// If columns are prefixed: billing_street, billing_city, etc.
@Embedded(onEmpty = Embedded.OnEmpty.USE_NULL, prefix = "billing_")
private Address billingAddress;

@Embedded(onEmpty = Embedded.OnEmpty.USE_NULL, prefix = "shipping_")
private Address shippingAddress;
```

---

## 11. One-to-Many Relationships

```java
// Schema:
// CREATE TABLE shopping_cart (id BIGSERIAL PRIMARY KEY, customer_id BIGINT, ...)
// CREATE TABLE cart_item (
//     id       BIGSERIAL PRIMARY KEY,
//     cart     BIGINT REFERENCES shopping_cart(id),  ← FK = root field name
//     product_id BIGINT,
//     quantity INT
// )

@Table("shopping_cart")
public class ShoppingCart {

    @Id
    private Long id;
    private Long customerId;
    private Instant updatedAt;

    // List: ordered, keyColumn tracks position (0, 1, 2...)
    @MappedCollection(idColumn = "cart", keyColumn = "position")
    private List<CartItem> items = new ArrayList<>();

    // Set: unordered, no keyColumn needed
    // @MappedCollection(idColumn = "cart")
    // private Set<CartItem> items = new HashSet<>();
}

public class CartItem {
    @Id
    private Long id;
    private Long productId;
    private int quantity;
    private BigDecimal unitPrice;
}
```

---

## 12. References Between Aggregates (AggregateReference)

Spring Data JDBC does NOT support navigating from one aggregate root to another like JPA does (`@ManyToOne Product product`). Instead, you store a reference by ID.

```java
// WRONG (JPA-style — not supported in Spring Data JDBC)
public class Order {
    @ManyToOne
    private Customer customer;  // ← Spring Data JDBC can't lazy-load this
}

// CORRECT — store reference by ID only
@Table("orders")
public class Order {
    @Id
    private Long id;

    // Reference to Customer aggregate — just an ID
    private AggregateReference<Customer, Long> customerId;

    // Usage:
    // order.getCustomerId().getId()  → the raw Long ID
}

// In service layer — load separately when needed
@Service
public class OrderQueryService {
    private final OrderRepository orderRepo;
    private final CustomerRepository customerRepo;

    public OrderDetailView getOrderDetail(long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        Customer customer = customerRepo.findById(order.getCustomerId().getId()).orElseThrow();
        return new OrderDetailView(order, customer);
    }
}
```

This is intentional — it enforces aggregate boundaries and prevents the N+1 lazy loading trap.

---

## 13. Optimistic Locking with `@Version`

```java
@Table("products")
public class Product {

    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    private int stockCount;

    @Version
    private Long version;  // auto-incremented on every save
}
```

```sql
-- What Spring Data JDBC generates under the hood:
-- On UPDATE:
UPDATE products SET name=?, price=?, stock_count=?, version=version+1
WHERE id=? AND version=?    ← if version doesn't match → 0 rows affected → exception

-- If 0 rows affected → throws OptimisticLockingFailureException
```

```java
@Service
public class PricingService {

    @Transactional
    public Product updatePrice(long productId, BigDecimal newPrice) {
        try {
            Product product = productRepo.findById(productId).orElseThrow();
            product.setPrice(newPrice);
            return productRepo.save(product);
            // If another thread updated this product between our read and save
            // → version mismatch → OptimisticLockingFailureException
        } catch (OptimisticLockingFailureException e) {
            // Retry logic or return conflict response
            throw new ConflictException("Product was updated concurrently, please retry");
        }
    }
}
```

---

## 14. Auditing

```java
// 1. Enable auditing
@Configuration
@EnableJdbcAuditing
public class JdbcConfig { }

// 2. Provide AuditorAware (who is the current user)
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext())
        .map(ctx -> ctx.getAuthentication())
        .filter(auth -> auth != null && auth.isAuthenticated())
        .map(auth -> auth.getName());
}

// 3. Add auditing fields to your entity
@Table("products")
public class Product {

    @Id
    private Long id;
    private String name;
    private BigDecimal price;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}
```

```sql
-- Schema must have these columns:
ALTER TABLE products
ADD COLUMN created_at TIMESTAMP,
ADD COLUMN updated_at TIMESTAMP,
ADD COLUMN created_by VARCHAR(100),
ADD COLUMN updated_by VARCHAR(100);
```

---

## 15. Application Events — Lifecycle Hooks

Spring Data JDBC publishes events around entity persistence. Use these instead of JPA's `@PrePersist`, `@PostLoad`, etc.

```java
@Component
public class OrderEventHandler {

    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    // Called BEFORE order is saved (INSERT or UPDATE)
    @EventListener
    public void onBeforeSave(BeforeSaveEvent<Order> event) {
        Order order = event.getEntity();
        if (order.getId() == null) {
            // Only on INSERT — set defaults
            order.setCreatedAt(Instant.now());
            order.setStatus("PENDING");
        }
    }

    // Called AFTER order is saved
    @EventListener
    public void onAfterSave(AfterSaveEvent<Order> event) {
        Order order = event.getEntity();
        if ("PLACED".equals(order.getStatus())) {
            // Trigger async notification after successful save
            notificationService.sendOrderConfirmation(order.getCustomerId(), order.getId());
        }
    }

    // Called BEFORE order is deleted
    @EventListener
    public void onBeforeDelete(BeforeDeleteEvent<Order> event) {
        Order order = event.getEntity();
        // Restore inventory when order is cancelled/deleted
        order.getItems().forEach(item ->
            inventoryService.restoreStock(item.getProductId(), item.getQuantity())
        );
    }

    // Called AFTER entity is loaded from DB
    @EventListener
    public void onAfterLoad(AfterLoadEvent<Order> event) {
        // Post-load processing — decrypt sensitive fields, etc.
    }
}
```

---

## 16. Custom Row Mapping with `RowMapper`

When Spring Data JDBC's default mapping isn't enough:

```java
@Configuration
public class JdbcMappingConfig {

    // Custom converter: DB String → Java Enum
    @Bean
    public JdbcCustomConversions jdbcCustomConversions() {
        return new JdbcCustomConversions(List.of(
            new Converter<String, OrderStatus>() {
                public OrderStatus convert(String source) {
                    return OrderStatus.fromCode(source);
                }
            },
            new Converter<OrderStatus, String>() {
                public String convert(OrderStatus source) {
                    return source.getCode();
                }
            }
        ));
    }
}
```

---

## 17. `JdbcAggregateTemplate` — Low-Level Access

When you need operations that repositories don't expose:

```java
@Service
public class ProductBulkService {

    private final JdbcAggregateTemplate jdbcTemplate;

    // Batch insert without individual save() calls
    public void bulkInsert(List<Product> products) {
        // Inserts all at once, returns saved entities with IDs
        Iterable<Product> saved = jdbcTemplate.insertAll(products);
    }

    // Update without loading first
    public void updateStatus(long orderId, String status) {
        Order order = new Order();
        order.setId(orderId);
        order.setStatus(status);
        // Only updates fields present — doesn't touch what's not set
        jdbcTemplate.update(order);
    }

    // Count with criteria
    public long countActiveProducts() {
        return jdbcTemplate.count(Query.query(
            Criteria.where("active").is(true)
        ), Product.class);
    }

    // Query with Criteria API
    public List<Product> findExpensiveProducts(BigDecimal minPrice) {
        return jdbcTemplate.findAll(
            Query.query(Criteria.where("price").greaterThan(minPrice))
                 .sort(Sort.by("price").descending())
                 .limit(100),
            Product.class
        );
    }
}
```

---

## 18. Complete Real-World Example — E-commerce Order Management

```java
// Schema
/*
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status      VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total       DECIMAL(19,2) NOT NULL DEFAULT 0,
    version     BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP
);

CREATE TABLE line_item (
    id          BIGSERIAL PRIMARY KEY,
    orders      BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    position    INT NOT NULL,
    product_id  BIGINT NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity    INT NOT NULL,
    unit_price  DECIMAL(19,2) NOT NULL
);
*/

// Aggregate Root
@Table("orders")
@Data
public class Order {

    @Id
    private Long id;
    private Long customerId;
    private String status;
    private BigDecimal total;

    @Version
    private Long version;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @MappedCollection(idColumn = "orders", keyColumn = "position")
    private List<LineItem> items = new ArrayList<>();

    public static Order create(Long customerId) {
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setStatus("PENDING");
        order.setTotal(BigDecimal.ZERO);
        return order;
    }

    public void addItem(long productId, String productName, int qty, BigDecimal unitPrice) {
        items.add(new LineItem(null, productId, productName, qty, unitPrice));
        recalcTotal();
    }

    public void removeItem(long productId) {
        items.removeIf(i -> i.getProductId().equals(productId));
        recalcTotal();
    }

    public void confirm() {
        if (items.isEmpty()) throw new IllegalStateException("Cannot confirm empty order");
        this.status = "CONFIRMED";
    }

    public void ship() {
        if (!"CONFIRMED".equals(status)) throw new IllegalStateException("Order must be confirmed before shipping");
        this.status = "SHIPPED";
    }

    private void recalcTotal() {
        this.total = items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

@Data
@AllArgsConstructor
public class LineItem {
    @Id
    private Long id;
    private Long productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
}

// Repository
public interface OrderRepository extends CrudRepository<Order, Long> {

    List<Order> findByCustomerId(Long customerId);
    List<Order> findByStatus(String status);
    Page<Order> findByCustomerIdOrderByCreatedAtDesc(Long customerId, Pageable pageable);

    @Query("SELECT * FROM orders WHERE customer_id = :customerId AND status = :status " +
           "AND created_at >= :since ORDER BY created_at DESC")
    List<Order> findRecentByCustomerAndStatus(
        @Param("customerId") Long customerId,
        @Param("status") String status,
        @Param("since") Instant since
    );

    @Modifying
    @Query("UPDATE orders SET status = :status, updated_at = NOW() WHERE id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") String status);

    @Query("SELECT COUNT(*) FROM orders WHERE status = 'PENDING' AND created_at < :cutoff")
    int countStaleOrders(@Param("cutoff") Instant cutoff);
}

// Service
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepo;
    private final ProductRepository productRepo;

    public Order placeOrder(PlaceOrderRequest req) {
        Order order = Order.create(req.getCustomerId());

        req.getItems().forEach(item -> {
            Product product = productRepo.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));

            order.addItem(
                product.getId(),
                product.getName(),
                item.getQuantity(),
                product.getPrice()
            );
        });

        order.confirm();
        return orderRepo.save(order);
        // → INSERT INTO orders, INSERT INTO line_item for each item
    }

    public Order addItemToOrder(long orderId, AddItemRequest req) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        Product product = productRepo.findById(req.getProductId()).orElseThrow();

        order.addItem(product.getId(), product.getName(), req.getQuantity(), product.getPrice());

        return orderRepo.save(order);
        // → UPDATE orders (new total), DELETE FROM line_item WHERE orders=?, re-INSERT all items
    }

    @Transactional(readOnly = true)
    public Page<Order> getOrderHistory(long customerId, int page, int size) {
        return orderRepo.findByCustomerIdOrderByCreatedAtDesc(
            customerId,
            PageRequest.of(page, size)
        );
    }

    public void shipOrder(long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.ship();
        orderRepo.save(order);
    }
}
```

---

## 19. Spring Data JDBC vs Spring Data JPA — Comparison

| Feature | Spring Data JDBC | Spring Data JPA (Hibernate) |
|---------|-----------------|------------------------------|
| **Lazy loading** | No — always eager | Yes — `FetchType.LAZY` |
| **Dirty checking** | No — always explicit save | Yes — Hibernate tracks changes |
| **First-level cache** | No | Yes — EntityManager per transaction |
| **Proxy objects** | No — plain Java objects | Yes — CGLIB proxies for lazy |
| **Session lifecycle** | None | EntityManager / PersistenceContext |
| **N+1 risk** | Low — you control joins | High — easy to trigger with LAZY |
| **`LazyInitializationException`** | Impossible | Very common |
| **Save behavior** | Full aggregate replace | Dirty tracking (only changed fields) |
| **Complex joins** | `@Query` with full SQL | JPQL / `@EntityGraph` |
| **Inheritance mapping** | Not supported | `@Inheritance`, `@DiscriminatorColumn` |
| **Second-level cache** | Not supported | Ehcache, Caffeine, Redis |
| **Schema generation** | No | `spring.jpa.hibernate.ddl-auto` |
| **Learning curve** | Low | High |
| **Predictability** | High | Low (magic can surprise you) |
| **Performance** | Predictable, no hidden queries | Variable, depends on fetch strategy |
| **Best for** | Microservices, DDD, simple aggregates | Complex domains, legacy schemas, reporting |

---

## 20. Scenario Walkthroughs

### Scenario 1: Shopping Cart — Concurrent item additions

**Problem:** Two browser tabs add items to the same cart simultaneously.

```java
@Table("shopping_cart")
public class ShoppingCart {
    @Id private Long id;
    private Long customerId;

    @Version
    private Long version;  // optimistic locking

    @MappedCollection(idColumn = "cart", keyColumn = "position")
    private List<CartItem> items = new ArrayList<>();
}

@Service
public class CartService {

    @Transactional
    @Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
    public ShoppingCart addItem(long cartId, AddItemRequest request) {
        ShoppingCart cart = cartRepo.findById(cartId).orElseThrow();
        cart.addItem(request.getProductId(), request.getQuantity(), request.getPrice());

        try {
            return cartRepo.save(cart);
            // If another tab saved the cart between our read and save:
            // → version mismatch → OptimisticLockingFailureException
            // @Retryable re-reads and retries up to 3 times
        } catch (OptimisticLockingFailureException e) {
            throw e;  // let @Retryable handle it
        }
    }
}
```

### Scenario 2: Why `save()` deletes and re-inserts children — and when it matters

**System:** An order with 500 line items. User changes the shipping address (stored in `orders` table, not in `line_item`).

```java
// This seems harmless:
order.setShippingAddress(newAddress);
orderRepo.save(order);

// What actually happens:
// UPDATE orders SET shipping_address=? WHERE id=?  ← OK
// DELETE FROM line_item WHERE orders=?              ← deletes 500 rows!
// INSERT INTO line_item ...                         ← re-inserts 500 rows!
```

**Solution for large aggregates:**

```java
// Use a targeted @Modifying query for partial updates
public interface OrderRepository extends CrudRepository<Order, Long> {
    @Modifying
    @Query("UPDATE orders SET shipping_street = :street, shipping_city = :city " +
           "WHERE id = :id")
    int updateShippingAddress(@Param("id") Long id,
                              @Param("street") String street,
                              @Param("city") String city);
}

// Call the targeted query instead of save()
orderRepo.updateShippingAddress(orderId, newAddress.getStreet(), newAddress.getCity());
// → 1 UPDATE, no DELETE/INSERT on line_item
```

**Lesson:** For aggregates with many children, use `@Modifying @Query` for partial updates. Reserve `save()` for when you're actually changing the aggregate structure.

---

## 21. Common Interview Questions

**Q1: What is the key architectural difference between Spring Data JDBC and Spring Data JPA?**

Spring Data JPA wraps Hibernate which uses a "Unit of Work" pattern: it maintains an identity map (first-level cache), tracks all entity changes (dirty checking), and flushes changes at transaction commit. Spring Data JDBC has none of this — there is no session, no dirty tracking, no proxy objects. Every `save()` is an explicit operation. Every load returns a plain Java object. This makes behavior completely predictable but requires the developer to be more deliberate about when and what they save.

---

**Q2: What is an Aggregate Root in Spring Data JDBC and why does it matter?**

An Aggregate Root is the single entry point to a cluster of related entities. In Spring Data JDBC, only aggregate roots have repositories. Child entities (like `LineItem` inside `Order`) can only be accessed and persisted via their root. When you `save(order)`, Spring Data JDBC also persists all `LineItem`s inside it. When you `delete(order)`, it deletes the `LineItem`s too. This enforces DDD boundaries — you can't accidentally modify a child entity independently of its root.

---

**Q3: Explain what happens internally when you call `save()` on an aggregate that already exists.**

Spring Data JDBC checks if `getId()` returns null (INSERT) or a non-null value (UPDATE). For UPDATE on an aggregate with child collections: it runs an `UPDATE` on the root table, then `DELETE FROM child_table WHERE root_fk = ?` to remove all existing children, then re-`INSERT`s all current children. It does NOT diff the children. This means `save()` on a large aggregate is expensive — it always replaces the entire child collection.

---

**Q4: `LazyInitializationException` is very common in Spring Data JPA. Can it happen in Spring Data JDBC?**

No, it's architecturally impossible. Spring Data JDBC has no lazy loading — when you load an aggregate, all referenced child entities are loaded immediately (eagerly) in the same operation. There is no proxy, no CGLIB subclass, no session to check. Your entity is a plain Java object with all fields populated. The trade-off is that large aggregates always load everything — but you avoid the entire class of bugs caused by accessing lazily-loaded collections outside a session.

---

**Q5: How does `@Version` work in Spring Data JDBC?**

`@Version` annotates a `Long` field that Spring Data JDBC includes in every `UPDATE` statement: `WHERE id = ? AND version = ?`. After the update, it increments the version in the DB. If the `WHERE` clause matches 0 rows (because another transaction already incremented the version), Spring Data JDBC throws `OptimisticLockingFailureException`. The developer is responsible for catching this and retrying or returning a 409 Conflict response.

---

**Q6: What is `AggregateReference` and why does Spring Data JDBC use it instead of direct object references?**

`AggregateReference<T, ID>` is a wrapper around a foreign-key ID. Instead of `private Customer customer`, you write `private AggregateReference<Customer, Long> customerId`. This is intentional — Spring Data JDBC enforces aggregate boundaries by not allowing one aggregate to contain another aggregate's entity. Cross-aggregate relationships must go through IDs. This prevents the lazy-loading problem: there's no way to accidentally navigate from Order to Customer and trigger an unexpected DB query.

---

**Q7: When would you choose Spring Data JDBC over Spring Data JPA?**

Choose Spring Data JDBC when: (1) your domain model naturally maps to small, well-defined aggregates; (2) you want predictable, explicit DB access with no hidden queries; (3) you're building microservices where each service owns a few aggregates; (4) your team has been burned by Hibernate's complexity (`LazyInitializationException`, N+1, dirty tracking bugs, session management). Choose Spring Data JPA when: you have a complex legacy schema, need second-level caching, require inheritance mapping, or need the full ORM feature set for a complex domain.

---

## 22. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **`save()` on large aggregates** | Deletes and re-inserts all children every time | Use `@Modifying @Query` for partial updates |
| **FK column name mismatch** | Children not linked to parent — foreign key defaults to field name | Name FK column exactly as the parent's field name, or use `idColumn` in `@MappedCollection` |
| **Cross-aggregate reference as object** | Spring Data JDBC doesn't fetch related aggregates automatically | Use `AggregateReference<T, ID>` and load separately |
| **Expecting lazy loading** | All collections loaded eagerly — performance issue for deep graphs | Keep aggregates small and focused |
| **Missing `@Modifying` on UPDATE/DELETE queries** | Spring Data throws exception: "Executing an update/delete query" | Always add `@Modifying` to mutation `@Query` methods |
| **No `@Transactional` on mutations** | Multiple SQL statements (save root + children) run without a transaction | Add `@Transactional` to service methods, not repository methods |
| **`@Version` without retry** | `OptimisticLockingFailureException` crashes the request | Catch the exception and retry or return 409 |
| **Complex reporting queries** | Spring Data JDBC is for writes; complex multi-join reports are awkward | Use `JdbcTemplate` or `NamedParameterJdbcTemplate` directly for read-side queries |
| **Assuming schema is auto-created** | Spring Data JDBC does NOT generate/validate schema | Write `schema.sql` manually or use Flyway/Liquibase |

---

## 23. Summary — Quick Revision

- Spring Data JDBC = Spring Data's repository abstraction + Spring JDBC, with **no lazy loading, no dirty tracking, no session, no proxy**.
- Built around **Aggregate Roots** — only roots have repositories; children are owned by the root.
- `save()` on an existing aggregate = UPDATE root + DELETE all children + re-INSERT all children.
- Use `@Version` for optimistic locking; handle `OptimisticLockingFailureException` with retry.
- Use `AggregateReference<T, ID>` for cross-aggregate references (foreign key by ID, not object reference).
- `@Query` with `:namedParams` for custom SQL; `@Modifying` required for mutations.
- Application events (`BeforeSaveEvent`, `AfterSaveEvent`) replace JPA's `@PrePersist` / `@PostPersist`.
- For large aggregates or partial updates, bypass `save()` and use `@Modifying @Query` directly.
- Spring Data JDBC is the right choice for microservices, DDD-style domains, and teams that want predictable data access without ORM magic.

---

> **What's next:** JPA and Hibernate — the full ORM, with lazy loading, dirty checking, EntityManager lifecycle, JPQL, criteria API, and all the power (and pitfalls) that come with it.

> **Follow-up questions to think about:**
> 1. You have an `Order` aggregate with `@Version`. Two concurrent requests both add an item to the same order. Walk through exactly what happens at the DB level with optimistic locking.
> 2. How would you implement a "soft delete" pattern (set `deleted=true` instead of actually deleting) in Spring Data JDBC?
> 3. Spring Data JDBC `save()` always re-inserts child entities. How would you design your aggregate boundary for an entity with 10,000 child records?
