# Hibernate — Complete Reference

> **Learning Path Position:** Raw JDBC → Spring JDBC → Spring Data JDBC → **Hibernate (JPA)** → Spring Data JPA
>
> Hibernate is the most widely used ORM (Object-Relational Mapper) in the Java ecosystem. Understanding it deeply — not just the annotations, but the internals — is what separates engineers who produce efficient, correct code from those who create subtle production bugs with hidden queries and memory leaks.

---

## 1. Overview — What is Hibernate? Why does it exist?

Hibernate solves the **Object-Relational Impedance Mismatch** — the structural gap between how Java models data (objects, inheritance, associations) and how relational databases model data (tables, foreign keys, rows).

Without an ORM, you write SQL, map `ResultSet` rows to objects manually, manage `Connection` lifecycle, handle transactions, and deal with caching yourself.

With Hibernate:
- Java classes map to tables (`@Entity`)
- Fields map to columns (`@Column`)
- Java references map to foreign keys (`@ManyToOne`, `@OneToMany`)
- Hibernate generates SQL, manages the `Connection`, tracks changes, and handles the session lifecycle

**Hibernate vs JPA:**
```
JPA (Jakarta Persistence API)
  = a SPECIFICATION (just interfaces + annotations, no code)
  = defined in jakarta.persistence.*

Hibernate
  = the most popular IMPLEMENTATION of JPA
  = also has its own extended API (Session, Criteria, etc.) beyond JPA

Spring Data JPA
  = a layer on top of JPA (Hibernate)
  = adds repository abstraction, derived queries, etc.

Stack:
  Your Code
    → Spring Data JPA (repositories)
      → JPA API (EntityManager)
        → Hibernate (implementation)
          → JDBC
            → Database
```

---

## 2. Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Hibernate Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application Layer                                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  EntityManager (JPA) / Session (Hibernate native API)    │   │
│  │  • First-level cache (identity map)                      │   │
│  │  • Dirty checking (tracks entity state changes)          │   │
│  │  • Unit of Work (batches DB ops until flush)             │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │                                            │
│  Factory Layer      │                                            │
│  ┌──────────────────▼───────────────────────────────────────┐   │
│  │  EntityManagerFactory / SessionFactory                   │   │
│  │  • Created ONCE at startup (expensive)                   │   │
│  │  • Thread-safe, shared across all threads                │   │
│  │  • Holds: Mapping metadata, Connection pool reference    │   │
│  │  • Second-level cache lives HERE                         │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │                                            │
│  Infrastructure     │                                            │
│  ┌──────────────────▼───────────────────────────────────────┐   │
│  │  Connection Pool (HikariCP) + JDBC Driver                │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │                                            │
│                     ▼                                            │
│                  Database                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Key classes:

| Class / Interface | API | Role |
|-------------------|-----|------|
| `EntityManagerFactory` | JPA | Application-scoped factory; created once at startup |
| `EntityManager` | JPA | Request/transaction-scoped; the "session" in JPA terms |
| `SessionFactory` | Hibernate | Hibernate's equivalent of `EntityManagerFactory` |
| `Session` | Hibernate | Hibernate's extended `EntityManager` (extra methods) |
| `Transaction` | JPA/Hibernate | Controls commit/rollback |
| `Query` / `TypedQuery` | JPA | JPQL query execution |
| `Criteria` / `CriteriaBuilder` | JPA | Type-safe programmatic queries |
| `StatelessSession` | Hibernate | No first-level cache — for bulk operations |

---

## 3. Entity Lifecycle — The Most Important Concept

Every entity in Hibernate is in one of **4 states**. This determines when SQL is generated.

```
                    new MyEntity()
                         │
                         ▼
              ┌─────────────────────┐
              │     TRANSIENT        │
              │  • Not known to any  │
              │    Hibernate session │
              │  • No DB row exists  │
              └──────────┬──────────┘
                         │  em.persist(entity)
                         │
                         ▼
              ┌─────────────────────┐
              │     PERSISTENT       │◄──────────────────┐
              │  • Managed by session│                   │
              │  • Has DB row        │  em.merge(entity) │
              │  • Dirty checking ON │◄──────────────────┘
              │  • Changes auto-     │  (from DETACHED)
              │    flushed on commit │
              └──────────┬──────────┘
                  │      │
     tx commit /  │      │  session.close() /
     em.detach()  │      │  em.clear() /
         │        │      │  end of @Transactional
         │        │      ▼
         │        │  ┌──────────────────────┐
         │        │  │     DETACHED          │
         │        │  │  • Not tracked anymore│
         │        │  │  • Changes NOT synced │
         │        │  │    to DB              │
         │        │  │  • Can be re-attached │
         │        │  │    with em.merge()    │
         │        │  └──────────────────────┘
         │        │
         ▼        │  em.remove(entity)
         │        ▼
         │  ┌──────────────────────┐
         │  │      REMOVED          │
         │  │  • Scheduled for      │
         │  │    DELETE on flush    │
         └─►│  • Still in session   │
            └──────────────────────┘
```

```java
// Transient — just a Java object, Hibernate doesn't know about it
Product product = new Product("SKU-001", "Laptop", new BigDecimal("999.99"));

// Persistent — managed by EntityManager, dirty checking is active
em.persist(product);   // now managed, INSERT will happen on flush

// Changes to persistent entity are automatically detected
product.setPrice(new BigDecimal("899.99"));  // NO explicit save() needed
// Hibernate will detect this change and issue UPDATE on flush/commit

// Detached — after transaction ends
// product is now detached — changes are NOT tracked
product.setPrice(new BigDecimal("799.99"));  // this change will NOT be saved

// Re-attach — merge returns a new persistent copy
Product managed = em.merge(product);
// now managed is persistent, product is still detached
managed.setPrice(new BigDecimal("799.99"));  // THIS will be saved

// Removed
em.remove(managed);  // scheduled for DELETE on flush
```

---

## 4. Entity Mapping

### 4.1 Basic Mapping

```java
@Entity                          // marks this as a JPA entity
@Table(
    name = "products",
    schema = "catalog",
    indexes = {
        @Index(name = "idx_products_sku", columnList = "sku", unique = true),
        @Index(name = "idx_products_category", columnList = "category, active")
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_products_sku", columnNames = "sku")
    }
)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // DB auto-increment
    private Long id;

    @Column(
        name = "sku",
        nullable = false,
        length = 50,
        unique = true
    )
    private String sku;

    @Column(name = "name", nullable = false, length = 255)
    private String name;

    @Column(name = "description", columnDefinition = "TEXT")
    private String description;

    @Column(name = "price", precision = 19, scale = 2)
    private BigDecimal price;

    @Column(name = "stock_count", nullable = false)
    private int stockCount;

    @Enumerated(EnumType.STRING)   // stores "ELECTRONICS" not 0,1,2
    @Column(name = "category", nullable = false, length = 50)
    private Category category;

    @Column(name = "active", nullable = false)
    private boolean active = true;

    @Column(name = "created_at", updatable = false)  // updatable=false → never in UPDATE
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    @Transient  // NOT mapped to DB
    private String computedDisplayPrice;

    @Version    // optimistic locking
    private Long version;

    @PrePersist
    private void prePersist() {
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    @PreUpdate
    private void preUpdate() {
        this.updatedAt = Instant.now();
    }
}
```

### 4.2 Primary Key Generation Strategies

```java
// IDENTITY — relies on DB auto-increment (MySQL, Postgres SERIAL/BIGSERIAL)
// Issue: disables JDBC batch inserts (must flush to get ID)
@GeneratedValue(strategy = GenerationType.IDENTITY)

// SEQUENCE — uses a DB sequence (PostgreSQL preferred)
// Best for batch inserts — Hibernate can pre-allocate a range of IDs
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
@SequenceGenerator(
    name = "product_seq",
    sequenceName = "products_id_seq",
    allocationSize = 50  // Hibernate pre-allocates 50 IDs per DB round-trip
)

// TABLE — uses a separate table to track ID sequences (DB-agnostic, slow, avoid)
@GeneratedValue(strategy = GenerationType.TABLE)

// UUID — good for distributed systems, no DB round-trip needed
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

// Custom generator
@Id
@GeneratedValue(generator = "custom-sku-generator")
@GenericGenerator(name = "custom-sku-generator", strategy = "com.app.SkuGenerator")
private String sku;
```

> **Interview tip:** `IDENTITY` is the most common but prevents Hibernate from batching inserts. `SEQUENCE` with `allocationSize` is faster for high-volume inserts. Never use `TABLE` in production.

---

## 5. Relationships

### 5.1 `@ManyToOne` (the most common)

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // MANY orders belong to ONE customer
    @ManyToOne(fetch = FetchType.LAZY)   // ALWAYS lazy for @ManyToOne
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Column(name = "total")
    private BigDecimal total;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;
}
```

> **Rule:** Always use `FetchType.LAZY` on `@ManyToOne`. The default is `FetchType.EAGER`, which is a performance trap — loading an order will automatically load the entire Customer object, even when you don't need it.

### 5.2 `@OneToMany` (the N+1 minefield)

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ONE order has MANY line items
    @OneToMany(
        mappedBy = "order",           // "order" = field name in LineItem
        cascade = CascadeType.ALL,    // persist/remove LineItems with Order
        orphanRemoval = true,         // delete LineItem if removed from this list
        fetch = FetchType.LAZY        // default for @OneToMany — don't change this
    )
    private List<LineItem> items = new ArrayList<>();

    // Helper methods to maintain bidirectional consistency
    public void addItem(LineItem item) {
        items.add(item);
        item.setOrder(this);          // MUST set both sides of bidirectional
    }

    public void removeItem(LineItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "line_items")
public class LineItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;              // "order" — this is mappedBy target

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    private int quantity;
    private BigDecimal unitPrice;
}
```

### 5.3 `@OneToOne`

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    // OneToOne — FK lives in user_profiles table (mappedBy side)
    @OneToOne(
        mappedBy = "user",
        cascade = CascadeType.ALL,
        fetch = FetchType.LAZY,       // MUST explicitly set LAZY — default is EAGER!
        optional = false
    )
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;               // owns the FK

    private String bio;
    private String avatarUrl;
    private LocalDate dateOfBirth;
}
```

> **Gotcha:** `@OneToOne` defaults to `EAGER` fetch. This means loading a `User` **always** loads the `UserProfile`, even if you only need the email. Always set `fetch = FetchType.LAZY` explicitly.
>
> **Gotcha:** Hibernate can't truly lazy-load the non-owning side of `@OneToOne` (the `mappedBy` side) without bytecode instrumentation. It uses a proxy, but in practice, it may issue an extra SELECT. Use `@OneToOne(optional = false)` to help Hibernate optimize.

### 5.4 `@ManyToMany`

```java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();  // Use Set, not List (avoids duplicate join issues)
}

@Entity
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany(mappedBy = "tags", fetch = FetchType.LAZY)
    private Set<Product> products = new HashSet<>();
}
```

> **Production advice:** Avoid `@ManyToMany` with `List` — Hibernate deletes all rows in the join table and re-inserts them when any change is made (catastrophic for large collections). Use `Set`. Better yet, model the join table as an explicit `@Entity` with its own `@ManyToOne` on both sides (gives you control + ability to add columns to the join table).

```java
// Better: explicit join entity
@Entity
@Table(name = "product_tags")
public class ProductTag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "tag_id")
    private Tag tag;

    private Instant taggedAt;  // extra column — impossible with plain @ManyToMany
}
```

---

## 6. Fetch Types — The Core of Performance

### EAGER vs LAZY

```
FetchType.EAGER:
  Hibernate loads the association in the SAME query (or immediately after)
  as the parent entity — even if you never use it.

FetchType.LAZY:
  Hibernate loads the association ONLY when you first access the field.
  Represented as a PROXY object until then.

Default fetch types:
  @ManyToOne  → EAGER  ← CHANGE THIS TO LAZY
  @OneToOne   → EAGER  ← CHANGE THIS TO LAZY
  @OneToMany  → LAZY   ← keep as-is
  @ManyToMany → LAZY   ← keep as-is
```

### How LAZY loading works

```java
// Inside @Transactional — session is open
Order order = em.find(Order.class, 1L);
// SQL: SELECT * FROM orders WHERE id=1   (no JOIN to line_items yet)

// order.getItems() returns a Hibernate proxy (PersistentBag/PersistentList)
// The proxy has not fired a SQL yet

List<LineItem> items = order.getItems();
// Still no SQL — you have the proxy reference

LineItem first = items.get(0);
// NOW Hibernate fires: SELECT * FROM line_items WHERE order_id=1
// This is the "lazy loading trigger"
```

### LazyInitializationException — the most common Hibernate bug

```java
// @Transactional method — session is open here
public Order findOrder(long id) {
    return orderRepo.findById(id).get();
}   // ← transaction ends HERE, session closes, entity becomes DETACHED

// Controller — no session here
Order order = orderService.findOrder(1L);
order.getItems().size();  // BOOM! LazyInitializationException
// "could not initialize proxy - no Session"
```

**Fixes:**

```java
// Fix 1: Load what you need inside the transaction (BEST)
@Transactional(readOnly = true)
public Order findOrderWithItems(long id) {
    Order order = orderRepo.findById(id).orElseThrow();
    Hibernate.initialize(order.getItems());  // forces load while session is open
    return order;
}

// Fix 2: JPQL JOIN FETCH (most efficient — single query)
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findWithItems(@Param("id") Long id);

// Fix 3: @EntityGraph (Spring Data JPA style)
@EntityGraph(attributePaths = {"items", "items.product"})
Optional<Order> findById(Long id);

// Fix 4: DTO projection (best performance — no entity overhead)
@Query("SELECT new com.app.dto.OrderSummary(o.id, o.total, o.status) FROM Order o WHERE o.id = :id")
Optional<OrderSummary> findSummaryById(@Param("id") Long id);

// Anti-fix: open-session-in-view (AVOID in production — see pitfalls)
```

---

## 7. First-Level Cache (Session Cache)

The first-level cache is **per-session (per-transaction)**, always on, and cannot be disabled.

```java
@Transactional
public void demonstrateFirstLevelCache() {
    // SQL: SELECT * FROM products WHERE id=1
    Product p1 = em.find(Product.class, 1L);

    // NO SQL — returned from first-level cache (identity map)
    Product p2 = em.find(Product.class, 1L);

    System.out.println(p1 == p2);  // TRUE — same object reference from cache

    // JPQL bypasses first-level cache check!
    Product p3 = em.createQuery("FROM Product WHERE id = 1", Product.class)
                   .getSingleResult();
    // SQL: SELECT * FROM products WHERE id=1  ← runs the query
    // But the result is merged with p1/p2 from identity map
    System.out.println(p1 == p3);  // TRUE — same object (identity map merge)
}
```

### First-level cache and bulk operations

```java
@Transactional
public void processBatch() {
    for (int i = 0; i < 100_000; i++) {
        em.persist(new LogEntry("event-" + i));

        if (i % 50 == 0) {
            em.flush();   // write to DB
            em.clear();   // clear first-level cache ← PREVENTS MEMORY LEAK
            // Without em.clear(), all 100,000 LogEntry objects stay in cache
            // causing gradual memory exhaustion
        }
    }
}
```

---

## 8. Second-Level Cache (Shared Cache)

The second-level cache is **per-SessionFactory** (shared across all sessions/transactions), **optional**, and must be explicitly enabled.

```
First-level cache:   per-session  → gone when session closes
Second-level cache:  per-factory  → survives across sessions
Query cache:         caches query results (query string + params → result IDs)
```

```java
// 1. Enable in application.properties
// spring.jpa.properties.hibernate.cache.use_second_level_cache=true
// spring.jpa.properties.hibernate.cache.region.factory_class=
//     org.hibernate.cache.jcache.JCacheRegionFactory
// spring.jpa.properties.javax.cache.provider=
//     org.ehcache.jsr107.EhcacheCachingProvider

// 2. Mark entities as cacheable
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // for mutable entities
public class Category {
    @Id
    private Long id;
    private String name;
    private String description;
}

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)   // for immutable reference data
public class Country {
    @Id
    private String code;
    private String name;
}

// Cache strategies:
// READ_ONLY        → for data that never changes (countries, currencies)
// NONSTRICT_READ_WRITE → may briefly serve stale data, higher throughput
// READ_WRITE       → strict consistency, uses soft locks
// TRANSACTIONAL   → full XA transaction support (JTA required)
```

```java
// 3. Cache collections too
@OneToMany(mappedBy = "product", fetch = FetchType.LAZY)
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
private List<ProductImage> images;
```

**In production:**

```java
// Check cache statistics
Statistics stats = sessionFactory.getStatistics();
stats.setStatisticsEnabled(true);

// After running queries:
long secondLevelHitCount  = stats.getSecondLevelCacheHitCount();  // cache hits
long secondLevelMissCount = stats.getSecondLevelCacheMissCount(); // cache misses
double hitRatio = (double) secondLevelHitCount /
                  (secondLevelHitCount + secondLevelMissCount);
```

---

## 9. Dirty Checking — How Hibernate Detects Changes

When a session flushes (on commit, or explicitly), Hibernate compares every **persistent** entity against its **snapshot** (the state when it was first loaded).

```
Load entity → Hibernate stores a SNAPSHOT of initial state
                    │
                    ▼
Your code modifies the entity
                    │
                    ▼
Session flush triggered (commit or explicit flush())
                    │
                    ▼
Hibernate compares current state vs snapshot field-by-field
  Changed fields → include in UPDATE SET clause
  Unchanged      → omit (unless using @DynamicUpdate)
                    │
                    ▼
SQL UPDATE generated and executed
```

```java
// What Hibernate does internally:
@Transactional
public void updateProductPrice(long id, BigDecimal newPrice) {
    Product product = em.find(Product.class, id);
    // Hibernate stores snapshot: {id:1, price:100, name:"Laptop", stock:50, ...}

    product.setPrice(newPrice);
    // No SQL yet. Just a Java field change.

    // On transaction commit → flush:
    // Hibernate compares: price changed (100 → 89.99)
    // SQL: UPDATE products SET price=89.99, updated_at=... WHERE id=1 AND version=0
    // No explicit save() needed!
}
```

### `@DynamicUpdate` — Only changed columns in UPDATE

```java
@Entity
@DynamicUpdate  // generates UPDATE with only changed columns
public class Product {
    // Without @DynamicUpdate: UPDATE products SET sku=?, name=?, price=?, stock_count=?, ... WHERE id=?
    // With @DynamicUpdate:    UPDATE products SET price=? WHERE id=?   (if only price changed)
    // Useful for wide tables or high-contention rows
}
```

### `@DynamicInsert` — Only non-null columns in INSERT

```java
@Entity
@DynamicInsert  // omits NULL columns from INSERT, letting DB defaults apply
public class AuditLog {
    // Without: INSERT INTO audit_log (id, action, actor, details, created_at) VALUES (...)
    // With:    INSERT INTO audit_log (id, action, actor) VALUES (...)  if details is null
}
```

---

## 10. Flush Modes

Flush = Hibernate writing pending changes from session to DB (not committing).

```java
// FlushMode.AUTO (default):
//   Hibernate flushes BEFORE executing a query if the query might be affected
//   by pending changes. Also flushes on commit.

// FlushMode.COMMIT:
//   Only flushes on explicit commit. Queries might see stale data.
//   Useful for read-heavy transactions to avoid unnecessary flushes.

// FlushMode.MANUAL:
//   Never auto-flushes. Must call em.flush() explicitly.
//   Used for special scenarios like read-only reporting transactions.

// FlushMode.ALWAYS:
//   Flushes before every query. Safe but slow.

// Setting flush mode:
em.setFlushMode(FlushModeType.COMMIT);

// In Spring:
@Transactional(readOnly = true)
// readOnly=true sets FlushMode.MANUAL (never flushes) + read-only connection hint
```

---

## 11. Cascade Types

Cascade means "propagate this operation from parent to children".

```java
@OneToMany(
    mappedBy = "order",
    cascade = CascadeType.ALL,   // propagate ALL operations
    orphanRemoval = true
)
private List<LineItem> items;
```

| CascadeType | What it propagates |
|-------------|-------------------|
| `PERSIST` | `em.persist(parent)` → persists children too |
| `MERGE` | `em.merge(parent)` → merges children too |
| `REMOVE` | `em.remove(parent)` → removes children too |
| `REFRESH` | `em.refresh(parent)` → refreshes children from DB |
| `DETACH` | `em.detach(parent)` → detaches children too |
| `ALL` | All of the above |

```java
// Cascade.PERSIST example:
Order order = new Order(customerId);
order.addItem(new LineItem(product, 2, price));
order.addItem(new LineItem(product2, 1, price2));

em.persist(order);
// Cascade PERSIST → automatically persists both LineItems too
// INSERT INTO orders ...
// INSERT INTO line_items ...  (twice)

// Cascade.REMOVE example:
em.remove(order);
// Cascade REMOVE → deletes LineItems first (FK constraint), then Order
// DELETE FROM line_items WHERE order_id=1
// DELETE FROM orders WHERE id=1

// orphanRemoval example:
order.getItems().remove(lineItem);  // remove from collection
// With orphanRemoval=true:
// DELETE FROM line_items WHERE id=<lineItem.id>
// Without orphanRemoval: line item remains in DB with orphaned FK
```

> **Rule of thumb:** Use `cascade = CascadeType.ALL` + `orphanRemoval = true` for parent-owns-children (Order → LineItems). Use `cascade = {PERSIST, MERGE}` for shared references (Product referenced by multiple entities). Never cascade `REMOVE` to shared entities.

---

## 12. N+1 Problem — The #1 Hibernate Performance Issue

**What it is:** Loading N entities, then for each entity, Hibernate fires 1 additional query to load an association = 1 + N queries total.

```java
// Example: Load all orders and print customer name
List<Order> orders = em.createQuery("FROM Order", Order.class).getResultList();
// SQL: SELECT * FROM orders   → returns 100 orders

for (Order order : orders) {
    System.out.println(order.getCustomer().getName());
    // EACH access to order.getCustomer() fires a separate query!
    // SQL: SELECT * FROM customers WHERE id=?  → 100 times!
}
// Total: 1 + 100 = 101 queries for 100 orders. N+1 problem.
```

### Solutions

**Solution 1: JOIN FETCH (most common)**

```java
// JPQL JOIN FETCH — loads association in the same query
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findByStatusWithCustomer(@Param("status") String status);
// SQL: SELECT o.*, c.* FROM orders o JOIN customers c ON c.id=o.customer_id
// 1 query total ✓
```

**Solution 2: `@EntityGraph` (Spring Data JPA)**

```java
// Define graph on entity
@Entity
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items-with-product")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items-with-product",
            attributeNodes = @NamedAttributeNode("product")
        )
    }
)
public class Order { ... }

// Use in repository
@EntityGraph("Order.withCustomerAndItems")
Optional<Order> findById(Long id);

// Or inline (Spring Data JPA)
@EntityGraph(attributePaths = {"customer", "items", "items.product"})
List<Order> findByStatus(String status);
```

**Solution 3: Batch fetching (for large collections)**

```java
// Hibernate fetches associations in batches — reduces N+1 to ceil(N/batchSize) queries
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@BatchSize(size = 25)  // loads 25 orders' line items at a time
private List<LineItem> items;

// Or globally in properties:
// spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

**Solution 4: DTO Projection (best for read-only)**

```java
// Custom DTO — no entities, no proxies, no lazy loading
@Query("""
    SELECT new com.app.dto.OrderSummaryDto(
        o.id, o.total, o.status, c.name, c.email
    )
    FROM Order o
    JOIN o.customer c
    WHERE o.status = :status
    """)
List<OrderSummaryDto> findOrderSummaries(@Param("status") String status);
// Single query, only needed columns, no entity overhead
```

**Solution 5: Subselect fetching**

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@Fetch(FetchMode.SUBSELECT)
// Fires: SELECT * FROM line_items WHERE order_id IN (SELECT id FROM orders WHERE ...)
// 2 queries instead of N+1
private List<LineItem> items;
```

---

## 13. JPQL — Java Persistence Query Language

JPQL operates on **entity classes and fields**, not table names and column names. Hibernate translates to SQL.

```java
// Basic SELECT
TypedQuery<Product> query = em.createQuery(
    "SELECT p FROM Product p WHERE p.active = true",
    Product.class
);
List<Product> products = query.getResultList();

// WHERE with parameters
TypedQuery<Order> q = em.createQuery(
    "SELECT o FROM Order o WHERE o.customerId = :customerId AND o.status = :status",
    Order.class
);
q.setParameter("customerId", customerId);
q.setParameter("status", "PENDING");
List<Order> orders = q.getResultList();

// JOIN
List<Order> orders = em.createQuery(
    "SELECT o FROM Order o JOIN o.customer c WHERE c.email = :email",
    Order.class
).setParameter("email", email).getResultList();

// JOIN FETCH (solves N+1)
List<Order> orders = em.createQuery(
    "SELECT DISTINCT o FROM Order o " +
    "JOIN FETCH o.items i " +
    "JOIN FETCH i.product " +
    "WHERE o.customerId = :id",
    Order.class
).setParameter("id", customerId).getResultList();
// DISTINCT needed — JOIN FETCH with collections produces duplicate Order objects

// Aggregates
Long count = em.createQuery(
    "SELECT COUNT(o) FROM Order o WHERE o.status = 'PENDING'",
    Long.class
).getSingleResult();

BigDecimal revenue = em.createQuery(
    "SELECT SUM(o.total) FROM Order o " +
    "WHERE o.createdAt BETWEEN :from AND :to",
    BigDecimal.class
).setParameter("from", from).setParameter("to", to).getSingleResult();

// GROUP BY
List<Object[]> report = em.createQuery(
    "SELECT o.status, COUNT(o), SUM(o.total) " +
    "FROM Order o GROUP BY o.status",
    Object[].class
).getResultList();

// UPDATE (bulk update — bypasses dirty checking)
int updated = em.createQuery(
    "UPDATE Product p SET p.active = false WHERE p.stockCount = 0"
).executeUpdate();
// WARNING: does NOT update first-level cache — do em.clear() after bulk updates

// DELETE (bulk delete)
int deleted = em.createQuery(
    "DELETE FROM AuditLog a WHERE a.createdAt < :cutoff"
).setParameter("cutoff", cutoff).executeUpdate();

// Pagination
List<Product> page = em.createQuery("FROM Product p ORDER BY p.name", Product.class)
    .setFirstResult(pageNumber * pageSize)  // OFFSET
    .setMaxResults(pageSize)                // LIMIT
    .getResultList();
```

---

## 14. Criteria API — Type-Safe Programmatic Queries

Use when query conditions are dynamic (unknown at compile time).

```java
@Repository
public class ProductCriteriaRepository {

    @PersistenceContext
    private EntityManager em;

    public List<Product> search(ProductSearchRequest req) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (req.getCategory() != null) {
            predicates.add(cb.equal(root.get("category"), req.getCategory()));
        }
        if (req.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), req.getMinPrice()));
        }
        if (req.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), req.getMaxPrice()));
        }
        if (req.getKeyword() != null) {
            predicates.add(cb.like(
                cb.lower(root.get("name")),
                "%" + req.getKeyword().toLowerCase() + "%"
            ));
        }
        predicates.add(cb.isTrue(root.get("active")));

        cq.where(predicates.toArray(new Predicate[0]));

        // Sorting
        if ("price".equals(req.getSortField())) {
            cq.orderBy(req.isAscending()
                ? cb.asc(root.get("price"))
                : cb.desc(root.get("price")));
        } else {
            cq.orderBy(cb.asc(root.get("name")));
        }

        return em.createQuery(cq)
            .setFirstResult(req.getPage() * req.getPageSize())
            .setMaxResults(req.getPageSize())
            .getResultList();
    }

    // Count query for pagination (avoid duplicating predicates)
    public long count(ProductSearchRequest req) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Long> cq = cb.createQuery(Long.class);
        Root<Product> root = cq.from(Product.class);

        cq.select(cb.count(root));
        // ... same predicates ...

        return em.createQuery(cq).getSingleResult();
    }
}
```

---

## 15. Native Queries

When JPQL can't express the query (DB-specific functions, CTEs, window functions):

```java
// Simple native query
@Query(value = "SELECT * FROM products WHERE SIMILARITY(name, :keyword) > 0.3 " +
               "ORDER BY SIMILARITY(name, :keyword) DESC",
       nativeQuery = true)
List<Product> findBySimilarity(@Param("keyword") String keyword);

// Native query with pagination
@Query(
    value = "SELECT * FROM products WHERE category = :cat ORDER BY created_at DESC",
    countQuery = "SELECT COUNT(*) FROM products WHERE category = :cat",
    nativeQuery = true
)
Page<Product> findByCategory(@Param("cat") String category, Pageable pageable);

// Projection with native query
@Query(value = """
    SELECT p.id, p.sku, p.name, p.price,
           COALESCE(SUM(oi.quantity), 0) AS total_sold
    FROM products p
    LEFT JOIN order_items oi ON oi.product_id = p.id
    GROUP BY p.id, p.sku, p.name, p.price
    ORDER BY total_sold DESC
    LIMIT :limit
    """, nativeQuery = true)
List<ProductSalesProjection> findTopSellers(@Param("limit") int limit);

// Projection interface — Spring maps columns by name
public interface ProductSalesProjection {
    Long getId();
    String getSku();
    String getName();
    BigDecimal getPrice();
    Long getTotalSold();
}

// EntityManager — named native query
@SqlResultSetMapping(
    name = "ProductSalesMapping",
    classes = @ConstructorResult(
        targetClass = ProductSalesDto.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "name"),
            @ColumnResult(name = "total_sold", type = Long.class)
        }
    )
)
@NamedNativeQuery(
    name = "Product.findTopSellers",
    query = "SELECT id, name, SUM(quantity) as total_sold FROM ...",
    resultSetMapping = "ProductSalesMapping"
)
```

---

## 16. Locking

### Optimistic Locking (`@Version`)

```java
@Entity
public class Product {

    @Id
    private Long id;

    @Version
    private Long version;  // Hibernate manages this

    private int stockCount;
}

@Transactional
public boolean deductStock(long productId, int qty) {
    Product product = em.find(Product.class, productId);
    if (product.getStockCount() < qty) return false;

    product.setStockCount(product.getStockCount() - qty);
    // On flush: UPDATE products SET stock_count=?, version=version+1
    //           WHERE id=? AND version=?   ← if version changed → 0 rows → exception

    return true;
    // If two threads both read version=5, first to commit succeeds (version→6)
    // Second gets OptimisticLockException (version 5 no longer matches)
}
```

### Pessimistic Locking

```java
// PESSIMISTIC_WRITE — exclusive lock, blocks all other reads and writes
Product product = em.find(
    Product.class, productId,
    LockModeType.PESSIMISTIC_WRITE
);
// SQL: SELECT * FROM products WHERE id=? FOR UPDATE

// PESSIMISTIC_READ — shared lock, blocks writes but allows reads
Product product = em.find(
    Product.class, productId,
    LockModeType.PESSIMISTIC_READ
);
// SQL: SELECT * FROM products WHERE id=? FOR SHARE

// In Spring Data JPA repository:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdForUpdate(@Param("id") Long id);

// With timeout (avoid indefinite blocking)
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Product> findByIdForUpdate(@Param("id") Long id);
```

**When to use each:**
- **Optimistic** — low contention, mostly reads; requires retry on failure
- **Pessimistic** — high contention, critical sections (flash sales, seat booking); requires careful timeout to avoid deadlocks

---

## 17. Inheritance Mapping

Three strategies — each with different trade-offs:

### Strategy 1: SINGLE_TABLE (default, best performance)

```java
// All subclasses share ONE table with a discriminator column
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {
    @Id @GeneratedValue private Long id;
    private BigDecimal amount;
    private Instant paidAt;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    private String maskedCardNumber;
    private String cardNetwork;      // VISA, MASTERCARD — columns NULL for other types
}

@Entity
@DiscriminatorValue("UPI")
public class UpiPayment extends Payment {
    private String upiId;            // NULL for CREDIT_CARD rows
}

// Table: payments(id, payment_type, amount, paid_at, masked_card_number, card_network, upi_id)
// Columns for other subtypes are NULL — violates normalization but very fast
```

### Strategy 2: TABLE_PER_CLASS (avoid)

```java
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
// Each concrete class gets its own FULL table
// Querying the parent type uses UNION ALL → expensive
// No FK to base table → poor relational integrity
```

### Strategy 3: JOINED (normalized, slower)

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {
    @Id @GeneratedValue private Long id;
    private BigDecimal amount;
    private Instant paidAt;
}

@Entity
@PrimaryKeyJoinColumn(name = "payment_id")
public class CreditCardPayment extends Payment {
    private String maskedCardNumber;
    // Table: credit_card_payments(payment_id FK→payments, masked_card_number)
}

// Querying CreditCardPayment → JOIN payments ON payments.id = credit_card_payments.payment_id
// Fully normalized, but every query joins parent + child table
```

**Comparison:**

| Strategy | Performance | Normalization | Nullable columns |
|----------|------------|---------------|-----------------|
| SINGLE_TABLE | Best | Poor | Yes (subtype columns) |
| JOINED | Moderate | Best | No |
| TABLE_PER_CLASS | Worst (UNION ALL) | Moderate | No |

---

## 18. Embeddable — Value Objects

```java
@Embeddable
public class Address {
    @Column(name = "street", nullable = false)
    private String street;

    @Column(name = "city", nullable = false)
    private String city;

    @Column(name = "state")
    private String state;

    @Column(name = "zip_code")
    private String zipCode;

    @Column(name = "country", nullable = false)
    private String country;
}

@Entity
public class Customer {
    @Id @GeneratedValue private Long id;
    private String name;

    @Embedded
    private Address billingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street",  column = @Column(name = "ship_street")),
        @AttributeOverride(name = "city",    column = @Column(name = "ship_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "ship_zip_code")),
        @AttributeOverride(name = "country", column = @Column(name = "ship_country"))
    })
    private Address shippingAddress;
}
// Table: customers(id, name, street, city, state, zip_code, country,
//                  ship_street, ship_city, ship_zip_code, ship_country)
```

---

## 19. Lifecycle Callbacks

```java
@Entity
public class Order {

    @PrePersist    // called before INSERT
    private void onPrePersist() {
        this.createdAt = Instant.now();
        this.status = OrderStatus.PENDING;
    }

    @PostPersist   // called after INSERT (ID now set)
    private void onPostPersist() {
        log.info("Order created with id: {}", this.id);
    }

    @PreUpdate     // called before UPDATE
    private void onPreUpdate() {
        this.updatedAt = Instant.now();
    }

    @PostUpdate    // called after UPDATE
    private void onPostUpdate() { }

    @PreRemove     // called before DELETE
    private void onPreRemove() {
        log.info("Deleting order: {}", this.id);
    }

    @PostRemove    // called after DELETE
    private void onPostRemove() { }

    @PostLoad      // called after SELECT (entity loaded)
    private void onPostLoad() {
        // Decrypt sensitive fields, compute derived values
        this.displayTotal = formatCurrency(this.total);
    }
}

// Or use a separate EntityListener class (keeps entity clean)
@EntityListeners(AuditEntityListener.class)
@Entity
public class Product { ... }

public class AuditEntityListener {
    @PrePersist
    public void prePersist(Object entity) { /* ... */ }
    @PreUpdate
    public void preUpdate(Object entity) { /* ... */ }
}
```

---

## 20. Hibernate Statistics & Performance Monitoring

```java
// Enable statistics
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG

// In code — detect N+1 at test time
@Autowired
private EntityManagerFactory emf;

@Test
void detectNPlusOne() {
    SessionFactory sf = emf.unwrap(SessionFactory.class);
    sf.getStatistics().setStatisticsEnabled(true);
    sf.getStatistics().clear();

    orderService.findAllOrdersWithCustomers();  // the method under test

    long queryCount = sf.getStatistics().getQueryExecutionCount();
    assertThat(queryCount).isLessThanOrEqualTo(2);  // fail if N+1 occurs
}
```

```yaml
# Log all SQL (dev only)
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true  # adds JPQL as comment above SQL

# Slow query log
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE  # logs bound parameter values
```

---

## 21. Schema Generation / Validation

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate   # production setting — validates schema against entities
# Options:
# none     → do nothing (safest, use with Flyway/Liquibase)
# validate → validate schema matches entities, fail if not (good for CI)
# update   → add new columns/tables but never drops (use with extreme caution)
# create   → drop and recreate on startup (dev only)
# create-drop → create on start, drop on stop (test only)
```

**Production approach:** `ddl-auto=none` + Flyway or Liquibase for versioned schema migrations.

---

## 22. StatelessSession — Bulk Operations Without Cache

```java
// StatelessSession has NO first-level cache, NO dirty checking, NO lifecycle callbacks
// Use for bulk imports/exports

@Autowired
private SessionFactory sessionFactory;

public void bulkImportProducts(List<ProductDto> dtos) {
    StatelessSession session = sessionFactory.openStatelessSession();
    Transaction tx = session.beginTransaction();

    try {
        for (int i = 0; i < dtos.size(); i++) {
            Product product = dtos.get(i).toEntity();
            session.insert(product);  // immediately writes to DB, no cache accumulation

            if (i % 500 == 0) {
                tx.commit();
                tx = session.beginTransaction();
            }
        }
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        session.close();
    }
}
```

---

## 23. Complete Real-World Example — E-commerce Order System

```java
// Entities
@Entity
@Table(name = "customers")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    @BatchSize(size = 20)
    private List<Order> orders = new ArrayList<>();
}

@Entity
@Table(name = "orders")
@DynamicUpdate
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<LineItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private BigDecimal total;

    @Version
    private Long version;

    @CreatedDate
    @Column(updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    public void addItem(Product product, int qty) {
        LineItem item = new LineItem(this, product, qty, product.getPrice());
        items.add(item);
        recalcTotal();
    }

    private void recalcTotal() {
        this.total = items.stream()
            .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

@Entity
@Table(name = "line_items")
public class LineItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    private int quantity;
    private BigDecimal unitPrice;
}

// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"items", "items.product", "customer"})
    Optional<Order> findWithDetailsById(Long id);

    @Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
    List<Order> findByStatusWithCustomer(@Param("status") OrderStatus status);

    @Query("""
        SELECT new com.app.dto.OrderSummaryDto(
            o.id, o.total, o.status, o.createdAt, c.name
        )
        FROM Order o JOIN o.customer c
        WHERE o.customer.id = :customerId
        ORDER BY o.createdAt DESC
        """)
    Page<OrderSummaryDto> findSummariesByCustomer(
        @Param("customerId") Long customerId,
        Pageable pageable
    );

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);
}

// Service
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepo;
    private final ProductRepository productRepo;
    private final InventoryService inventoryService;

    public Order placeOrder(PlaceOrderRequest req) {
        Customer customer = /* load customer */;
        Order order = new Order(customer);

        for (OrderItemRequest item : req.getItems()) {
            Product product = productRepo.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));

            // Pessimistic lock on inventory for flash-sale scenario
            boolean deducted = inventoryService.deductWithLock(product.getId(), item.getQuantity());
            if (!deducted) throw new OutOfStockException(product.getSku());

            order.addItem(product, item.getQuantity());
        }

        return orderRepo.save(order);
        // Hibernate: INSERT INTO orders, INSERT INTO line_items (x N)
        // Cascade.ALL on items handles the line item inserts
    }

    @Transactional(readOnly = true)
    public OrderDetailView getOrderDetail(long orderId) {
        Order order = orderRepo.findWithDetailsById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        // Single query via @EntityGraph — no N+1
        return OrderDetailView.from(order);
    }

    public void cancelOrder(long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        // @DynamicUpdate → only UPDATE orders SET status=?, updated_at=? WHERE id=?
        // No need for save() — dirty checking handles it
        // Cascade REMOVE + orphanRemoval NOT needed for cancel, items stay for audit
    }
}
```

---

## 24. Scenario Walkthroughs

### Scenario 1: Flash Sale — Race Condition on Last Item

**System:** 500 concurrent users try to buy the last laptop at 12:00 PM.

**Without locking (broken):**
```
Thread A: SELECT stock=1 FROM products WHERE id=1  → reads 1
Thread B: SELECT stock=1 FROM products WHERE id=1  → reads 1
Thread A: UPDATE products SET stock=0 WHERE id=1   → succeeds
Thread B: UPDATE products SET stock=0 WHERE id=1   → succeeds (stock is already 0, but returns 1 row affected!)
Result: 2 orders created, stock=-1. Oversell.
```

**With `@Version` (optimistic locking):**
```java
@Transactional
public boolean purchase(long productId, long userId, int qty) {
    try {
        Product product = productRepo.findById(productId).orElseThrow();
        // SQL: SELECT * FROM products WHERE id=? → version=5

        if (product.getStockCount() < qty) return false;
        product.setStockCount(product.getStockCount() - qty);
        // Flush: UPDATE products SET stock_count=0, version=6 WHERE id=? AND version=5
        // Thread B's flush: WHERE version=5 finds 0 rows → OptimisticLockException

        orderRepo.save(new Order(userId, productId, qty));
        return true;

    } catch (OptimisticLockException | StaleObjectStateException e) {
        // Expected under contention — return false, frontend retries
        return false;
    }
}
```

**With pessimistic locking (for guaranteed exactly-once):**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "2000"))
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdForUpdate(@Param("id") Long id);

@Transactional
public boolean purchase(long productId, long userId, int qty) {
    Product product = productRepo.findByIdForUpdate(productId).orElseThrow();
    // SQL: SELECT * FROM products WHERE id=? FOR UPDATE  ← blocks other threads

    if (product.getStockCount() < qty) return false;
    product.setStockCount(product.getStockCount() - qty);
    orderRepo.save(new Order(userId, productId, qty));
    return true;
    // Lock released on commit/rollback
}
```

**Interviewer's intent:** Tests understanding of dirty checking, version-based locking, `PESSIMISTIC_WRITE`, and the practical trade-offs between optimistic and pessimistic strategies.

---

### Scenario 2: LazyInitializationException in a REST API

**System:** A product catalog API returns product details including category info.

```java
// Service
@Transactional
public Product getProduct(long id) {
    return productRepo.findById(id).orElseThrow();
}  // ← transaction ends, product is DETACHED here

// Controller
@GetMapping("/products/{id}")
public ProductResponse getProduct(@PathVariable long id) {
    Product product = productService.getProduct(id);
    return ProductResponse.from(product);  // tries to access product.getCategory().getName()
    // LazyInitializationException: could not initialize proxy - no Session
}
```

**Fix 1 — Keep transaction open until serialization (OSIV — avoid):**
```
spring.jpa.open-in-view=true  (Spring Boot default!)
This keeps the Hibernate session open for the entire HTTP request lifecycle.
It solves LazyInitializationException but hides N+1 bugs and holds DB connections longer.
TURN IT OFF: spring.jpa.open-in-view=false
```

**Fix 2 — Correct fix — DTO projection:**
```java
@Transactional(readOnly = true)
public ProductDetailDto getProduct(long id) {
    // Load what you need inside the transaction, return a DTO (not an entity)
    return productRepo.findProjectionById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
}

@Query("""
    SELECT new com.app.dto.ProductDetailDto(
        p.id, p.name, p.price, p.description, c.name, c.slug
    )
    FROM Product p JOIN p.category c
    WHERE p.id = :id
    """)
Optional<ProductDetailDto> findProjectionById(@Param("id") Long id);
```

**Interviewer's intent:** Tests understanding of entity lifecycle, OSIV, and the correct pattern for REST APIs (never return entities from services, always return DTOs; keep transactions in the service layer).

---

### Scenario 3: Bulk Update — Why Dirty Checking is a Problem

**System:** Nightly job that resets all cart items older than 30 days.

**Wrong approach:**
```java
@Transactional
public void clearExpiredCarts() {
    List<CartItem> expired = cartItemRepo.findByCreatedAtBefore(thirtyDaysAgo);
    // Loads 50,000 CartItem entities into memory and first-level cache

    for (CartItem item : expired) {
        item.setActive(false);
        // Hibernate tracks every one — 50,000 dirty-check comparisons on flush
    }
    // Flush: 50,000 individual UPDATE statements
    // Memory: 50,000 entities in first-level cache → OOM risk
}
```

**Correct approach:**
```java
@Transactional
public void clearExpiredCarts() {
    // Bulk UPDATE bypasses entity loading, dirty checking, and first-level cache
    int updated = cartItemRepo.deactivateExpired(Instant.now().minus(30, DAYS));
    // SQL: UPDATE cart_items SET active=false WHERE created_at < ? AND active=true
    // 1 query, 50,000 rows, no memory overhead
    log.info("Deactivated {} expired cart items", updated);
}

@Modifying
@Query("UPDATE CartItem c SET c.active = false WHERE c.createdAt < :cutoff AND c.active = true")
int deactivateExpired(@Param("cutoff") Instant cutoff);
```

---

## 25. Common Interview Questions

**Q1: What is the Hibernate first-level cache? Can it be disabled?**

The first-level cache is an identity map maintained per `Session` (per `EntityManager`). When you load an entity by ID, Hibernate stores it in this cache. Subsequent loads of the same ID return the same Java object from cache — no DB roundtrip. This guarantees object identity within a session (`em.find(Product.class, 1L) == em.find(Product.class, 1L)` is always true). It cannot be disabled — it's fundamental to Hibernate's dirty checking and identity guarantees. However, you can evict specific entities with `em.detach(entity)` or clear the whole session with `em.clear()`.

---

**Q2: Explain dirty checking. When does Hibernate generate an UPDATE?**

When an entity is loaded (becomes persistent), Hibernate takes a snapshot of its field values. On session flush (which happens before queries under `FlushMode.AUTO`, or on transaction commit), Hibernate compares each persistent entity's current state against its snapshot. If any field changed, Hibernate generates an `UPDATE` for that entity. No explicit `save()` or `update()` call is needed — this is the "Unit of Work" pattern. The `UPDATE` is only sent during flush, not at the moment the field is changed.

---

**Q3: What is the N+1 problem? How do you detect and fix it?**

N+1 occurs when loading N entities and then accessing a LAZY association on each one, triggering N additional queries (one per entity) — totalling N+1. It's the most common Hibernate performance bug. Detection: enable `hibernate.generate_statistics` and assert query count in tests. Fixes (in order of preference): (1) `JOIN FETCH` in JPQL — one query with a JOIN; (2) `@EntityGraph` — declarative, works with Spring Data JPA; (3) `@BatchSize` — reduces N queries to ceil(N/batchSize) by batching IN clauses; (4) DTO projections — skip entity loading entirely.

---

**Q4: What is the difference between `persist`, `merge`, `detach`, and `refresh`?**

- `persist(entity)` — takes a TRANSIENT entity and makes it PERSISTENT (schedules INSERT).
- `merge(entity)` — takes a DETACHED entity, copies its state onto a PERSISTENT copy (or creates a new persistent one), and returns the persistent copy. The original entity remains DETACHED.
- `detach(entity)` — moves a PERSISTENT entity to DETACHED state. Changes are no longer tracked.
- `refresh(entity)` — reloads the entity's state from DB, discarding any in-memory changes. Useful when another process may have updated the DB outside your session.

---

**Q5: What is `CascadeType.REMOVE` vs `orphanRemoval = true`? Are they the same?**

Not the same. `CascadeType.REMOVE` means: when `em.remove(parent)` is called, also call `em.remove()` on each child. `orphanRemoval = true` means: if a child is removed from the parent's collection (`parent.getItems().remove(item)`), automatically call `em.remove(item)` — it treats removal from the collection as orphaning. Both together (`CascadeType.ALL` + `orphanRemoval`) is the typical parent-owns-children setup. `orphanRemoval` also implies `CascadeType.REMOVE`, so setting both is redundant for the remove case.

---

**Q6: What is `open-session-in-view` and why should it be disabled?**

OSIV (Open Session In View) is a Spring pattern (enabled by default in Spring Boot) that keeps the Hibernate session open for the entire duration of an HTTP request — through the controller, view rendering, and response writing. This allows lazy associations to be accessed outside `@Transactional` service methods without `LazyInitializationException`. It should be disabled (`spring.jpa.open-in-view=false`) because: (1) it hides N+1 bugs — lazy loading in views is invisible; (2) it holds a DB connection open for the full request duration, even during slow operations (JSON serialization, template rendering); (3) it encourages leaking entities into the web layer (entities should never leave the service layer — return DTOs).

---

**Q7: How does Hibernate handle `@OneToMany` with a `List` vs a `Set`? Why does it matter for `@ManyToMany`?**

With a `List` (an ordered `@OneToMany`), Hibernate tracks element positions. For `@ManyToMany` with a `List`, Hibernate uses a `PersistentBag` — when any change occurs to the collection, it deletes ALL rows from the join table and re-inserts all of them (including unchanged ones). With a `Set`, Hibernate can compute the exact diff — it only deletes removed elements and inserts new ones. For `@ManyToMany`, always use `Set` to avoid the delete-all-and-reinsert behavior.

---

**Q8: What is `@Transactional(readOnly = true)` and what does it actually do?**

`readOnly = true` on a Spring transaction: (1) sets `FlushMode.MANUAL` on the Hibernate session, so Hibernate never flushes (no dirty-check overhead, no accidental updates); (2) passes a read-only hint to the JDBC driver — some drivers optimize (e.g., route to a read replica); (3) signals intent to developers that this transaction should not modify data. It does NOT prevent you from calling `save()` or making entity changes — it just means they won't be flushed automatically. Use for all read operations, especially in query-heavy APIs.

---

## 26. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **`open-session-in-view` enabled (Spring Boot default)** | Lazy loading works anywhere → hidden N+1 bugs, connection held for full request | Set `spring.jpa.open-in-view=false`, use DTO projections or explicit fetching |
| **`@ManyToOne` default is EAGER** | Loading any entity with a `@ManyToOne` eagerly loads the related entity — even when not needed | Always explicitly set `fetch = FetchType.LAZY` on `@ManyToOne` and `@OneToOne` |
| **N+1 with `@OneToMany`** | Accessing a LAZY collection inside a loop fires N queries | Use `JOIN FETCH`, `@EntityGraph`, or `@BatchSize` |
| **`@ManyToMany` with `List`** | Any collection change → DELETE all + INSERT all in join table | Use `Set` |
| **Returning entities from service layer** | Entities get serialized by Jackson, triggering lazy loading outside transaction | Return DTOs, never entities from service/controller |
| **Bulk update without `em.clear()`** | First-level cache holds stale entities after bulk `UPDATE`/`DELETE` JPQL | Call `em.clear()` after bulk mutations |
| **`IDENTITY` generation strategy with batch inserts** | Hibernate can't batch INSERTs — must flush after each to get the generated ID | Use `SEQUENCE` with `allocationSize` |
| **`CascadeType.REMOVE` on shared entities** | Deleting a parent cascades removal to an entity referenced by other parents | Only cascade REMOVE to entities owned exclusively by this parent |
| **Missing `mappedBy`** | Hibernate creates TWO foreign key columns (or a join table) instead of one | The non-owning side of a bidirectional relationship MUST have `mappedBy` |
| **Forgetting to set both sides of bidirectional** | In-memory object graph is inconsistent — parent's collection doesn't reflect what was added | Add helper methods (`addItem`, `removeItem`) that set both sides |
| **Long-running transactions** | Hibernate session holds DB connection; dirty checking on thousands of entities is slow | Keep transactions short; don't call external services inside `@Transactional` |
| **`@Transactional` self-invocation** | Calling a `@Transactional` method from within the same bean bypasses the proxy | Inject self, use `ApplicationContext.getBean()`, or restructure code |
| **Entity equals/hashCode using `id`** | `id` is null before persist; using id in `hashCode` breaks `HashSet` behavior mid-transaction | Use business key (e.g., `sku`) for equals/hashCode, or use UUID IDs |
| **JPQL bulk update bypasses cache** | `em.createQuery("UPDATE Product...")` doesn't update first-level or second-level cache | Call `em.clear()` after bulk JPQL updates to prevent stale reads |
| **`schema.sql` + `ddl-auto=create`** | Both try to create schema → error | Use only one: either `ddl-auto` or `schema.sql`/Flyway |

---

## 27. Summary — Quick Revision

- **Entity lifecycle:** Transient → Persistent (tracked, dirty-checked) → Detached (not tracked) → Removed (scheduled DELETE). Only persistent entities participate in dirty checking.
- **Dirty checking:** Hibernate stores a snapshot on load; compares on flush; generates UPDATE for changed fields only. No explicit `save()` needed for persistent entities.
- **First-level cache:** Per-session identity map. `em.find(X, id)` twice → same object, no second query. Clear with `em.clear()` in batch operations.
- **N+1:** The #1 Hibernate performance bug. Detect with `generate_statistics`. Fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.
- **Always LAZY:** Set `fetch = FetchType.LAZY` on all `@ManyToOne` and `@OneToOne`. The defaults (EAGER) are wrong for production.
- **Disable OSIV:** `spring.jpa.open-in-view=false`. Return DTOs from services, not entities.
- **Optimistic locking (`@Version`):** For low-contention scenarios. Requires retry on `OptimisticLockException`. Pessimistic (`FOR UPDATE`) for high-contention, critical sections — always set a timeout.
- **Bulk operations:** Use JPQL bulk `UPDATE`/`DELETE` or `StatelessSession` — never load-modify-save 100,000 entities.

---

> **What's next:** Spring Data JPA — adds the repository abstraction (`JpaRepository`), derived queries, `@Query`, specifications, projections, and pagination on top of Hibernate/JPA.

> **Follow-up questions to think about:**
> 1. Two `@Transactional` methods call each other. The inner method has `propagation = REQUIRES_NEW`. How many Hibernate sessions are active? How many DB connections?
> 2. You have `@OneToMany` with `CascadeType.ALL` and `orphanRemoval = true`. You call `order.getItems().clear()` inside a `@Transactional` method. What SQL runs and when?
> 3. Hibernate generates `SELECT ... WHERE id IN (?, ?, ?, ...)` instead of individual queries for your `@OneToMany`. Which configuration caused this and why is it better than N+1?
