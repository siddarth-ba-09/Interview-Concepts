# Spring Data JPA — Complete Reference

> **Learning Path Position:** Raw JDBC → Spring JDBC → Spring Data JDBC → Hibernate → JPA → **Spring Data JPA**
>
> Spring Data JPA is the repository abstraction layer that sits on top of JPA (Hibernate). It eliminates the boilerplate `EntityManager` code, generates queries from method names, and provides pagination, sorting, auditing, projections, and specifications — all with minimal code. Understanding *what it generates* and *when it doesn't do what you expect* is what matters for production and interviews.

---

## 1. Overview — What is Spring Data JPA?

```
Your Code
  └── Spring Data JPA  (JpaRepository, derived queries, Specification, Auditing)
        └── JPA API    (EntityManager, JPQL, Criteria API)
              └── Hibernate
                    └── JDBC
                          └── Database
```

Spring Data JPA:
- **Generates repository implementations at startup** — you write interfaces, Spring generates the implementation beans
- **Parses method names** into JPQL queries at startup (fail-fast)
- **Provides common CRUD + pagination operations** via `JpaRepository`
- **Adds `Specification` API** for type-safe dynamic queries (wraps Criteria API)
- **Manages `EntityManager` lifecycle** — you never call `em.persist()` / `em.find()` directly
- **Integrates auditing** — auto-populates `createdAt`, `updatedAt`, `createdBy`

---

## 2. Repository Hierarchy

```
Repository<T, ID>                              ← Marker interface (no methods)
  └── CrudRepository<T, ID>                   ← save, findById, findAll, delete, count, exists
        └── PagingAndSortingRepository<T, ID> ← findAll(Pageable), findAll(Sort)
              └── JpaRepository<T, ID>        ← flush, saveAll, findAllById, deleteAllInBatch
                    └── JpaSpecificationExecutor<T>  ← (separate interface) Specification support
```

```java
// CrudRepository — core operations
save(S entity)                     // INSERT or UPDATE (merge by default)
saveAll(Iterable<S> entities)      // batch save
findById(ID id)                    // returns Optional<T>
existsById(ID id)                  // SELECT COUNT
findAll()                          // SELECT all — DANGEROUS on large tables
findAllById(Iterable<ID> ids)      // SELECT ... WHERE id IN (...)
count()                            // SELECT COUNT(*)
deleteById(ID id)                  // findById first, then em.remove() — 2 queries
delete(T entity)                   // em.remove() — 1 query
deleteAll()                        // loads all, then deletes one by one — N+1 disaster
deleteAllById(Iterable<ID> ids)    // same — loads first then deletes

// JpaRepository additions
flush()                            // explicit flush to DB within current transaction
saveAndFlush(T entity)             // save + immediate flush (useful in tests)
deleteAllInBatch()                 // single DELETE FROM table — no load, no cascade
deleteAllInBatch(Iterable<T>)      // DELETE ... WHERE id IN (...)
deleteAllByIdInBatch(Iterable<ID>) // DELETE ... WHERE id IN (...) — preferred for bulk deletes
getOne(ID id)                      // deprecated — use getReferenceById
getReferenceById(ID id)            // em.getReference() — proxy, no SELECT unless accessed
findAll(Sort sort)                 // findAll with ordering
findAll(Pageable pageable)         // paginated findAll
```

> **`deleteAll()` vs `deleteAllInBatch()`:** `deleteAll()` loads all entities into memory (N+1), calls `em.remove()` on each, fires N DELETE statements. `deleteAllInBatch()` issues a single `DELETE FROM table` — no entity loading, no cascade, no lifecycle callbacks. For bulk deletes, always use `InBatch` variants. Caveat: `InBatch` bypasses cascade and `@PreRemove` callbacks.

---

## 3. Setting Up Spring Data JPA

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>  <!-- or mysql-connector-j -->
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    open-in-view: false             # ALWAYS false — see pitfalls
    show-sql: false                 # true in dev only
    hibernate:
      ddl-auto: validate            # production: validate or none
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 25  # global @BatchSize for all lazy collections
        jdbc:
          batch_size: 50            # enable JDBC batch inserts
          order_inserts: true       # reorder inserts for better batching
          order_updates: true
        generate_statistics: false  # true only for performance analysis
```

```java
// Enable JPA repositories (auto-detected by Spring Boot, but explicit for clarity)
@EnableJpaRepositories(basePackages = "com.app.repository")
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
@SpringBootApplication
public class Application { }
```

---

## 4. Defining Repositories

```java
// Standard — extend JpaRepository<EntityClass, IdType>
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Spring Data generates the implementation at startup
    // All CrudRepository + JpaRepository methods available for free
}

// Add JpaSpecificationExecutor for dynamic queries
public interface ProductRepository
        extends JpaRepository<Product, Long>, JpaSpecificationExecutor<Product> {
}

// Read-only repository — prevents accidental writes
@NoRepositoryBean  // prevents Spring from trying to instantiate this
public interface ReadOnlyRepository<T, ID> extends Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
}

public interface CategoryReadRepository extends ReadOnlyRepository<Category, Long> {
}

// Composite ID (embeddable key)
public interface OrderItemRepository extends JpaRepository<OrderItem, OrderItemId> {
}
```

---

## 5. Derived Query Methods — Method Name Parsing

Spring Data parses the method name and generates JPQL at startup. If the method name is invalid, the application fails to start — a great fail-fast feature.

### Keyword Reference

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── findBy ───────────────────────────────────────────────────────────────
    List<Product> findByActive(boolean active);
    // SELECT p FROM Product p WHERE p.active = ?1

    Optional<Product> findBySku(String sku);
    // SELECT p FROM Product p WHERE p.sku = ?1

    List<Product> findByCategory(Category category);
    // SELECT p FROM Product p WHERE p.category = ?1

    // ─── Nested property traversal ────────────────────────────────────────────
    List<Product> findByCategoryName(String categoryName);
    // SELECT p FROM Product p WHERE p.category.name = ?1

    List<Product> findByCategoryNameAndActive(String categoryName, boolean active);
    // WHERE p.category.name = ?1 AND p.active = ?2

    // ─── Comparison operators ─────────────────────────────────────────────────
    List<Product> findByPriceGreaterThan(BigDecimal price);         // price > ?
    List<Product> findByPriceLessThanEqual(BigDecimal price);       // price <= ?
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);  // BETWEEN
    List<Product> findByStockCountGreaterThanEqual(int minStock);   // >=

    // ─── String operators ─────────────────────────────────────────────────────
    List<Product> findByNameContaining(String keyword);             // LIKE %keyword%
    List<Product> findByNameStartingWith(String prefix);            // LIKE prefix%
    List<Product> findByNameEndingWith(String suffix);              // LIKE %suffix
    List<Product> findByNameLike(String pattern);                   // LIKE (you supply %)
    List<Product> findByNameIgnoreCase(String name);                // LOWER comparison
    List<Product> findByNameContainingIgnoreCase(String keyword);   // LIKE lower case

    // ─── Null checks ──────────────────────────────────────────────────────────
    List<Product> findByDeletedAtIsNull();                          // IS NULL
    List<Product> findByDeletedAtIsNotNull();                       // IS NOT NULL

    // ─── Collection / IN ──────────────────────────────────────────────────────
    List<Product> findByIdIn(Collection<Long> ids);                 // WHERE id IN (...)
    List<Product> findByStatusNotIn(Collection<Status> statuses);   // NOT IN

    // ─── Boolean ──────────────────────────────────────────────────────────────
    List<Product> findByActiveTrue();                               // WHERE active = true
    List<Product> findByActiveFalse();                              // WHERE active = false

    // ─── Logical operators ────────────────────────────────────────────────────
    List<Product> findByCategoryAndActiveTrue(Category category);
    List<Product> findByCategoryOrFeaturedTrue(Category category);

    // ─── Ordering ─────────────────────────────────────────────────────────────
    List<Product> findByCategoryOrderByPriceAsc(Category category);
    List<Product> findByActiveOrderByCreatedAtDesc(boolean active);
    List<Product> findTop10ByActiveTrueOrderByCreatedAtDesc();      // TOP N

    // ─── Limiting results ─────────────────────────────────────────────────────
    Optional<Product> findFirstByOrderByCreatedAtDesc();            // latest product
    List<Product> findTop5ByCategoryOrderByPriceAsc(Category cat);  // Top 5

    // ─── Count, Exists, Delete ────────────────────────────────────────────────
    long countByActive(boolean active);                             // SELECT COUNT
    boolean existsBySku(String sku);                                // SELECT COUNT > 0
    long deleteByActiveFalseAndCreatedAtBefore(Instant cutoff);     // bulk DELETE (returns count)

    // ─── With Pageable parameter ──────────────────────────────────────────────
    Page<Product> findByCategory(Category category, Pageable pageable);
    Slice<Product> findByActiveTrue(Pageable pageable);             // Slice: no count query
    List<Product> findByCategory(Category category, Sort sort);     // with sorting only

    // ─── Stream ───────────────────────────────────────────────────────────────
    @Transactional(readOnly = true)
    Stream<Product> findAllByActiveTrue();  // streams result set — close the stream!
}
```

### Method naming keywords table

| Keyword | JPQL equivalent | Example |
|---------|----------------|---------|
| `And` | `AND` | `findByNameAndActive` |
| `Or` | `OR` | `findByNameOrSku` |
| `Is`, `Equals` | `=` | `findByNameIs`, `findByNameEquals` |
| `Between` | `BETWEEN` | `findByPriceBetween` |
| `LessThan` | `<` | `findByAgeLessThan` |
| `LessThanEqual` | `<=` | `findByAgeLessThanEqual` |
| `GreaterThan` | `>` | `findByAgeGreaterThan` |
| `GreaterThanEqual` | `>=` | `findByAgeGreaterThanEqual` |
| `After` | `>` (date) | `findByCreatedAtAfter` |
| `Before` | `<` (date) | `findByCreatedAtBefore` |
| `IsNull`, `Null` | `IS NULL` | `findByDeletedAtIsNull` |
| `IsNotNull`, `NotNull` | `IS NOT NULL` | `findByDeletedAtNotNull` |
| `Like` | `LIKE` | `findByNameLike` |
| `NotLike` | `NOT LIKE` | `findByNameNotLike` |
| `StartingWith` | `LIKE 'x%'` | `findByNameStartingWith` |
| `EndingWith` | `LIKE '%x'` | `findByNameEndingWith` |
| `Containing` | `LIKE '%x%'` | `findByNameContaining` |
| `OrderBy` | `ORDER BY` | `findByActiveOrderByNameAsc` |
| `Not` | `<>` | `findByStatusNot` |
| `In` | `IN` | `findByIdIn` |
| `NotIn` | `NOT IN` | `findByStatusNotIn` |
| `True` | `= true` | `findByActiveTrue` |
| `False` | `= false` | `findByActiveFalse` |
| `IgnoreCase` | `LOWER(...)` | `findByNameIgnoreCase` |

---

## 6. `@Query` Annotation — Custom JPQL and Native Queries

When derived method names become too long or can't express the query, use `@Query`.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ─── JPQL query ───────────────────────────────────────────────────────────
    @Query("SELECT o FROM Order o WHERE o.customer.id = :customerId AND o.status = :status")
    List<Order> findByCustomerAndStatus(
        @Param("customerId") Long customerId,
        @Param("status") OrderStatus status
    );

    // ─── JOIN FETCH (solve N+1) ───────────────────────────────────────────────
    @Query("""
        SELECT DISTINCT o FROM Order o
        JOIN FETCH o.customer c
        JOIN FETCH o.items i
        JOIN FETCH i.product
        WHERE o.status = :status
        AND o.createdAt >= :since
        """)
    List<Order> findWithDetailsAfter(
        @Param("status") OrderStatus status,
        @Param("since") Instant since
    );

    // ─── DTO projection via constructor expression ─────────────────────────────
    @Query("""
        SELECT new com.app.dto.OrderSummaryDto(
            o.id, o.total, o.status, o.createdAt,
            c.name, c.email,
            COUNT(i)
        )
        FROM Order o
        JOIN o.customer c
        LEFT JOIN o.items i
        WHERE o.status = :status
        GROUP BY o.id, o.total, o.status, o.createdAt, c.name, c.email
        ORDER BY o.createdAt DESC
        """)
    List<OrderSummaryDto> findSummariesByStatus(@Param("status") OrderStatus status);

    // ─── With Pageable ────────────────────────────────────────────────────────
    // countQuery is required when the main query has JOIN FETCH (Hibernate can't derive count)
    @Query(
        value = """
            SELECT o FROM Order o
            JOIN FETCH o.customer
            WHERE o.customer.id = :customerId
            """,
        countQuery = "SELECT COUNT(o) FROM Order o WHERE o.customer.id = :customerId"
    )
    Page<Order> findByCustomerId(@Param("customerId") Long customerId, Pageable pageable);

    // ─── Native SQL ───────────────────────────────────────────────────────────
    @Query(
        value = """
            SELECT o.*, c.name AS customer_name
            FROM orders o
            JOIN customers c ON c.id = o.customer_id
            WHERE o.created_at >= NOW() - INTERVAL '7 days'
            ORDER BY o.total DESC
            LIMIT :limit
            """,
        nativeQuery = true
    )
    List<Object[]> findTopOrdersNative(@Param("limit") int limit);

    // ─── Native with projection interface (cleaner) ────────────────────────────
    @Query(
        value = "SELECT o.id, o.total, o.status, c.name AS customerName " +
                "FROM orders o JOIN customers c ON c.id = o.customer_id " +
                "WHERE o.status = :status",
        nativeQuery = true
    )
    List<OrderProjection> findNativeProjection(@Param("status") String status);

    // ─── Subquery ─────────────────────────────────────────────────────────────
    @Query("""
        SELECT p FROM Product p
        WHERE p.price > (
            SELECT AVG(p2.price) FROM Product p2 WHERE p2.category = p.category
        )
        AND p.active = true
        """)
    List<Product> findAboveAveragePriceInCategory();

    // ─── EXISTS subquery ──────────────────────────────────────────────────────
    @Query("""
        SELECT c FROM Customer c
        WHERE EXISTS (
            SELECT o FROM Order o WHERE o.customer = c AND o.createdAt >= :since
        )
        """)
    List<Customer> findActiveCustomersSince(@Param("since") Instant since);
}
```

---

## 7. `@Modifying` — Bulk Updates and Deletes

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── Bulk UPDATE ──────────────────────────────────────────────────────────
    @Modifying
    @Query("UPDATE Product p SET p.active = false WHERE p.stockCount = 0")
    int deactivateOutOfStockProducts();
    // Returns int = number of rows affected
    // Does NOT update the first-level cache — call em.clear() or use clearAutomatically

    // ─── clearAutomatically clears the EntityManager after execution ──────────
    @Modifying(clearAutomatically = true)
    @Query("UPDATE Product p SET p.price = p.price * :multiplier WHERE p.category = :category")
    int applyPriceMultiplier(
        @Param("multiplier") BigDecimal multiplier,
        @Param("category") Category category
    );

    // ─── flushAutomatically flushes pending changes before the bulk query ─────
    @Modifying(flushAutomatically = true, clearAutomatically = true)
    @Query("UPDATE Order o SET o.status = :newStatus WHERE o.status = :oldStatus AND o.createdAt < :cutoff")
    int expireStaleOrders(
        @Param("oldStatus") OrderStatus oldStatus,
        @Param("newStatus") OrderStatus newStatus,
        @Param("cutoff") Instant cutoff
    );

    // ─── Bulk DELETE ──────────────────────────────────────────────────────────
    @Modifying(clearAutomatically = true)
    @Query("DELETE FROM AuditLog a WHERE a.createdAt < :cutoff")
    int deleteOldAuditLogs(@Param("cutoff") Instant cutoff);

    // ─── Native bulk UPDATE ───────────────────────────────────────────────────
    @Modifying
    @Query(
        value = "UPDATE products SET search_vector = to_tsvector('english', name || ' ' || description) WHERE id = :id",
        nativeQuery = true
    )
    void updateSearchVector(@Param("id") Long id);
}
```

> **Critical:** `@Modifying` queries bypass Hibernate's dirty checking and first-level cache. After a bulk update, entities already loaded in the session will have stale data. Use `clearAutomatically = true` to clear the persistence context automatically, or call `em.clear()` manually. `flushAutomatically = true` ensures any pending dirty writes are flushed to DB before the bulk query runs (so you don't overwrite changes you haven't committed yet).

---

## 8. Pagination and Sorting

### Pageable, Page, Slice

```java
// ─── Creating Pageable ────────────────────────────────────────────────────────
// Page 0, size 20, sorted by price desc, then name asc
Pageable pageable = PageRequest.of(
    0,                                   // page number (0-indexed)
    20,                                  // page size
    Sort.by(
        Sort.Order.desc("price"),
        Sort.Order.asc("name")
    )
);

// Simple sort
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());

// Unsorted page
Pageable pageable = PageRequest.of(0, 20);

// ─── Page<T> — includes total count ──────────────────────────────────────────
Page<Product> page = productRepo.findByCategory(Category.ELECTRONICS, pageable);

page.getContent();       // List<Product> — the actual data
page.getTotalElements(); // long — total matching records (runs COUNT query)
page.getTotalPages();    // int — ceil(totalElements / pageSize)
page.getNumber();        // current page number (0-indexed)
page.getSize();          // page size
page.isFirst();          // is this the first page?
page.isLast();           // is this the last page?
page.hasNext();          // more pages available?
page.hasPrevious();      // previous page available?
page.nextPageable();     // Pageable for the next page
page.previousPageable(); // Pageable for the previous page

// ─── Slice<T> — no total count (faster) ──────────────────────────────────────
// Page fires 2 queries: data + COUNT. Slice fires only the data query.
// Use Slice for infinite scroll (mobile apps, feed) where total count isn't needed.
Slice<Product> slice = productRepo.findByActiveTrue(pageable);
slice.getContent();
slice.hasNext();         // checks if there's a next page (fetches pageSize+1 to check)
// slice.getTotalElements() — NOT available (no count query)

// ─── List<T> + Sort — no pagination, just ordering ───────────────────────────
Sort sort = Sort.by(Sort.Direction.DESC, "createdAt");
List<Product> products = productRepo.findByCategory(Category.ELECTRONICS, sort);

// ─── REST controller pattern ──────────────────────────────────────────────────
@GetMapping("/products")
public Page<ProductDto> getProducts(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "createdAt") String sortBy,
    @RequestParam(defaultValue = "desc") String direction
) {
    // Whitelist allowed sort fields to prevent injection
    Set<String> allowedSortFields = Set.of("createdAt", "price", "name");
    if (!allowedSortFields.contains(sortBy)) {
        throw new InvalidSortFieldException(sortBy);
    }

    Sort sort = direction.equalsIgnoreCase("asc")
        ? Sort.by(sortBy).ascending()
        : Sort.by(sortBy).descending();

    Pageable pageable = PageRequest.of(page, Math.min(size, 100), sort); // cap at 100

    return productRepo.findByActiveTrue(pageable)
                      .map(ProductDto::from);  // Page.map() transforms content
}
```

### Sorting by nested property

```java
// Sort by a related entity's field
Sort sort = Sort.by("category.name").ascending();

// Sort by multiple fields with JpaSort (safe for function-based sorts)
Sort sort = JpaSort.unsafe("LENGTH(name)").and(Sort.by("name"));
// JpaSort.unsafe() for expressions not representable as a simple property path
```

---

## 9. Projections — Return Only What You Need

Projections let you fetch a subset of columns without loading the full entity. Three types: interface projections, DTO class projections, and dynamic projections.

### 9.1 Interface Projections (Closed)

```java
// Closed projection — fixed set of fields
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();          // maps to field "price" or getter getPrice()
    String getCategoryName();       // maps to nested: category.name
    // Spring Data generates a proxy that implements this interface
}

public interface ProductRepository extends JpaRepository<Product, Long> {
    List<ProductSummary> findByActiveTrue();
    // SELECT p.id, p.name, p.price, c.name FROM products p JOIN categories c
    // Only fetches the needed columns — not SELECT *
}

// Usage
List<ProductSummary> summaries = productRepo.findByActiveTrue();
summaries.get(0).getName();        // "Laptop Pro"
summaries.get(0).getCategoryName(); // "Electronics"
```

### 9.2 Open Projections (`@Value` with SpEL)

```java
// Open projection — computed/derived fields using SpEL
public interface ProductDisplayProjection {
    String getName();
    BigDecimal getPrice();
    String getCurrency();

    // @Value with SpEL — computed, but loads ALL columns (open projection = no optimization)
    @Value("#{target.name + ' — ' + target.currency + target.price}")
    String getDisplayLabel();

    // Default method — computed without loading extra columns (preferred)
    default String getDisplayLabel() {
        return getName() + " — " + getPrice();
    }
}
```

### 9.3 Class-Based (DTO) Projections — Best Performance

```java
// DTO record (Java 16+) — constructor expression in JPQL
public record ProductDto(Long id, String name, BigDecimal price, String categoryName) {}

// Spring Data auto-maps if constructor params match field names
// OR use @Query with constructor expression explicitly:
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Spring Data can auto-project to DTO record if field names match
    List<ProductDto> findByActiveTrue();

    // Or explicit JPQL constructor:
    @Query("""
        SELECT new com.app.dto.ProductDto(p.id, p.name, p.price, c.name)
        FROM Product p JOIN p.category c
        WHERE p.active = true AND p.category.id = :catId
        """)
    List<ProductDto> findByCategoryAsDto(@Param("catId") Long catId);
}
```

### 9.4 Dynamic Projections — Choose at Runtime

```java
// One repository method, multiple return types
public interface ProductRepository extends JpaRepository<Product, Long> {
    <T> List<T> findByCategory(Category category, Class<T> type);
}

// Caller decides what to project:
List<ProductSummary> summaries = repo.findByCategory(cat, ProductSummary.class);
List<Product>        entities  = repo.findByCategory(cat, Product.class);
List<ProductDto>     dtos      = repo.findByCategory(cat, ProductDto.class);
```

### Projection Types Summary

| Type | SQL optimization | Computed fields | Nested associations |
|------|-----------------|-----------------|-------------------|
| Closed interface | ✅ Only projected columns | ❌ | ✅ via naming convention |
| Open interface (`@Value`) | ❌ Loads all columns | ✅ SpEL | ✅ |
| DTO / record class | ✅ JPQL constructor expr | ✅ (in query) | ✅ via explicit JOIN |
| Dynamic | Same as chosen type | — | — |

---

## 10. `@EntityGraph` in Spring Data JPA

`@EntityGraph` controls fetch behavior per query method — without changing entity annotations.

```java
@Entity
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "itemWithProduct")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "itemWithProduct",
            attributeNodes = @NamedAttributeNode("product")
        )
    }
)
public class Order { ... }

public interface OrderRepository extends JpaRepository<Order, Long> {

    // Use defined NamedEntityGraph
    @EntityGraph("Order.withCustomerAndItems")
    Optional<Order> findById(Long id);

    // Inline entity graph — attributePaths style (simpler)
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    List<Order> findByStatus(OrderStatus status);

    // fetchgraph (default) vs loadgraph
    @EntityGraph(value = "Order.withCustomerAndItems", type = EntityGraph.EntityGraphType.FETCH)
    // FETCH: only listed attributes are EAGER, all others are LAZY
    // LOAD:  listed attributes are EAGER, others use their declared fetch type
    Optional<Order> findWithGraphById(Long id);

    // Combine with @Query
    @EntityGraph(attributePaths = {"customer"})
    @Query("SELECT o FROM Order o WHERE o.createdAt >= :since ORDER BY o.createdAt DESC")
    List<Order> findRecentWithCustomer(@Param("since") Instant since);

    // Combine with Pageable
    @EntityGraph(attributePaths = {"customer"})
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
}
```

> **Gotcha:** Using `@EntityGraph` with `@Query` that has `JOIN FETCH` causes a `HibernateException`: you can't use both at the same time. Choose one strategy per method.

> **Gotcha:** Using `@EntityGraph` with a `Page` query that joins a collection causes a Hibernate warning: "HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!" — Hibernate fetches ALL rows into memory and paginates in Java, not in SQL. Fix: use DTO projection or a separate query for the collection.

---

## 11. Specification API — Dynamic Queries with Type Safety

`Specification<T>` wraps the JPA Criteria API into composable predicates. Use for search filters where conditions are optional.

```java
// 1. Repository must extend JpaSpecificationExecutor
public interface ProductRepository
        extends JpaRepository<Product, Long>, JpaSpecificationExecutor<Product> {
}

// 2. Define Specification methods (static factory methods — best practice)
public class ProductSpecifications {

    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<Product> hasCategory(Category category) {
        return (root, query, cb) ->
            category == null ? null : cb.equal(root.get("category"), category);
        // returning null = no predicate added (Spring Data skips null specs)
    }

    public static Specification<Product> nameLike(String keyword) {
        return (root, query, cb) ->
            keyword == null ? null :
            cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%");
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("price"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("price"), min);
            return cb.between(root.get("price"), min, max);
        };
    }

    public static Specification<Product> inStock() {
        return (root, query, cb) -> cb.greaterThan(root.get("stockCount"), 0);
    }

    public static Specification<Product> createdAfter(Instant from) {
        return (root, query, cb) ->
            from == null ? null : cb.greaterThanOrEqualTo(root.get("createdAt"), from);
    }

    // Specification with JOIN
    public static Specification<Product> hasCategoryName(String categoryName) {
        return (root, query, cb) -> {
            if (categoryName == null) return null;
            Join<Product, Category> catJoin = root.join("category", JoinType.INNER);
            return cb.equal(cb.lower(catJoin.get("name")), categoryName.toLowerCase());
        };
    }

    // Specification with subquery (EXISTS)
    public static Specification<Product> hasRecentReviews(Instant since) {
        return (root, query, cb) -> {
            Subquery<Long> subquery = query.subquery(Long.class);
            Root<Review> reviewRoot = subquery.from(Review.class);
            subquery.select(cb.count(reviewRoot))
                    .where(
                        cb.equal(reviewRoot.get("product"), root),
                        cb.greaterThanOrEqualTo(reviewRoot.get("createdAt"), since)
                    );
            return cb.greaterThan(subquery, 0L);
        };
    }
}

// 3. Compose Specifications
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductQueryService {

    private final ProductRepository productRepo;

    public Page<ProductDto> search(ProductFilter filter, Pageable pageable) {
        // Compose predicates — null specs are ignored
        Specification<Product> spec = Specification
            .where(ProductSpecifications.isActive())
            .and(ProductSpecifications.hasCategory(filter.getCategory()))
            .and(ProductSpecifications.nameLike(filter.getKeyword()))
            .and(ProductSpecifications.priceBetween(filter.getMinPrice(), filter.getMaxPrice()))
            .and(ProductSpecifications.inStock());

        if (filter.isOnlyWithRecentReviews()) {
            spec = spec.and(ProductSpecifications.hasRecentReviews(
                Instant.now().minus(30, DAYS)
            ));
        }

        return productRepo.findAll(spec, pageable)
                          .map(ProductDto::from);
    }
}

// 4. Specification operators
Specification<Product> activeAndElectronics =
    isActive().and(hasCategory(Category.ELECTRONICS));

Specification<Product> cheapOrElectronics =
    priceBetween(null, new BigDecimal("100"))
    .or(hasCategory(Category.ELECTRONICS));

Specification<Product> notActive = Specification.not(isActive());
```

> **When to use Specification vs `@Query`:**
> - Use `Specification` when the query has many optional filters (admin search, filter pages)
> - Use `@Query` when the query is fixed — simpler, more readable, easier to optimize
> - For very complex queries (window functions, CTEs, full-text search), use `@Query(nativeQuery=true)` or drop to `EntityManager`/`JdbcTemplate`

---

## 12. Custom Repository Implementation

When you need logic beyond what Spring Data can generate, add a custom implementation.

```java
// 1. Define a custom interface
public interface ProductRepositoryCustom {
    List<ProductDto> searchWithFacets(ProductFilter filter);
    void updateStockBulk(Map<Long, Integer> productIdToStock);
}

// 2. Implement it — class name MUST be <RepositoryInterface>Impl
@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    private final JdbcTemplate jdbcTemplate;  // can mix JDBC and JPA

    @Override
    @Transactional(readOnly = true)
    public List<ProductDto> searchWithFacets(ProductFilter filter) {
        // Complex query logic here
        CriteriaBuilder cb = em.getCriteriaBuilder();
        // ... full criteria API logic
        return em.createQuery(cq).getResultList();
    }

    @Override
    @Transactional
    public void updateStockBulk(Map<Long, Integer> productIdToStock) {
        // Use JDBC for bulk update — much faster than entity-based updates
        List<Object[]> params = productIdToStock.entrySet().stream()
            .map(e -> new Object[]{e.getValue(), e.getKey()})
            .collect(Collectors.toList());

        jdbcTemplate.batchUpdate(
            "UPDATE products SET stock_count = ? WHERE id = ?",
            params
        );
    }
}

// 3. Extend both — Spring Data merges them
public interface ProductRepository
        extends JpaRepository<Product, Long>,
                JpaSpecificationExecutor<Product>,
                ProductRepositoryCustom {
    // Derived queries and @Query methods here
    List<Product> findByActiveTrue();
}

// All methods available on the same interface:
productRepo.findByActiveTrue();                    // Spring Data generated
productRepo.findAll(spec, pageable);               // JpaSpecificationExecutor
productRepo.searchWithFacets(filter);              // Custom implementation
productRepo.updateStockBulk(stockMap);             // Custom JDBC
```

---

## 13. Locking in Spring Data JPA

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── Pessimistic write lock ───────────────────────────────────────────────
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
    // SQL: SELECT * FROM products WHERE id = ? FOR UPDATE

    // ─── With timeout (prevents indefinite blocking) ───────────────────────────
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdateWithTimeout(@Param("id") Long id);

    // ─── Pessimistic read lock (shared) ───────────────────────────────────────
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForShare(@Param("id") Long id);

    // ─── Optimistic lock — use @Version on entity, no special repo method needed
    // em.find() automatically checks version on flush
}
```

---

## 14. `@QueryHints`

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Read-only hint — Hibernate marks entities as read-only (no dirty checking)
    // Slight memory and CPU saving for read-only queries
    @QueryHints(value = {
        @QueryHint(name = "org.hibernate.readOnly", value = "true"),
        @QueryHint(name = "org.hibernate.fetchSize", value = "200"),  // JDBC fetch size
        @QueryHint(name = "jakarta.persistence.query.timeout", value = "5000")  // 5 sec timeout
    })
    @Query("SELECT p FROM Product p WHERE p.active = true")
    List<Product> findAllActiveReadOnly();

    // Cache query result (second-level query cache)
    @QueryHints(value = {
        @QueryHint(name = "org.hibernate.cacheable", value = "true"),
        @QueryHint(name = "org.hibernate.cacheRegion", value = "product.byCategory")
    })
    List<Product> findByCategory(Category category);

    // Comment in SQL (helpful for DB-level query identification / slow query logs)
    @QueryHints(@QueryHint(name = "org.hibernate.comment", value = "catalog-search"))
    List<Product> findByNameContaining(String keyword);
}
```

---

## 15. Auditing

```java
// ─── 1. Enable in Spring Boot app ────────────────────────────────────────────
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
@SpringBootApplication
public class Application { }

// ─── 2. AuditorAware — who is the current user? ──────────────────────────────
@Bean("auditorProvider")
public AuditorAware<String> auditorProvider() {
    return () -> {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated() || auth instanceof AnonymousAuthenticationToken) {
            return Optional.of("system");
        }
        return Optional.of(auth.getName());
    };
}

// ─── 3. Auditable base class (reuse across entities) ──────────────────────────
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}

// ─── 4. Entities extend Auditable ────────────────────────────────────────────
@Entity
@Table(name = "products")
public class Product extends Auditable {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_id_seq", allocationSize = 50)
    private Long id;

    // ... fields
}

// ─── 5. @Version for optimistic locking (often paired with auditing) ──────────
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class VersionedAuditable extends Auditable {
    @Version
    private Long version;
}
```

---

## 16. Soft Delete Pattern

Spring Data JPA has no built-in soft delete, but it's a common production pattern.

```java
// ─── Approach 1: @Where (Hibernate-specific, global filter) ──────────────────
@Entity
@Table(name = "products")
@SQLDelete(sql = "UPDATE products SET deleted_at = NOW() WHERE id = ?")  // override DELETE
@Where(clause = "deleted_at IS NULL")   // appended to all queries automatically
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;
    private Instant deletedAt;
}

// All JPA queries automatically add WHERE deleted_at IS NULL
// productRepo.findAll()  → SELECT ... FROM products WHERE deleted_at IS NULL
// productRepo.delete(product) → UPDATE products SET deleted_at = NOW() WHERE id = ?

// ─── Approach 2: Manual with @Query (more explicit, more portable) ────────────
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue private Long id;
    private String name;
    private boolean deleted = false;
    private Instant deletedAt;

    public void softDelete() {
        this.deleted = true;
        this.deletedAt = Instant.now();
    }
}

public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByDeletedFalse();                     // all active
    Optional<Product> findByIdAndDeletedFalse(Long id);     // by id, not deleted
    Page<Product> findByDeletedFalse(Pageable pageable);    // paginated active

    @Modifying
    @Query("UPDATE Product p SET p.deleted = true, p.deletedAt = :now WHERE p.id = :id")
    void softDeleteById(@Param("id") Long id, @Param("now") Instant now);
}

@Service
@RequiredArgsConstructor
@Transactional
public class ProductService {
    private final ProductRepository productRepo;

    public void delete(Long id) {
        Product product = productRepo.findByIdAndDeletedFalse(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        product.softDelete();  // dirty checking handles the update
    }
}

// ─── Approach 3: @FilterDef + @Filter (Hibernate, conditional) ───────────────
@Entity
@FilterDef(name = "notDeleted", parameters = @ParamDef(name = "deleted", type = Boolean.class))
@Filter(name = "notDeleted", condition = "deleted = :deleted")
public class Product { ... }

// Activate the filter per session (can be toggled)
@Autowired
private EntityManager em;

Session session = em.unwrap(Session.class);
session.enableFilter("notDeleted").setParameter("deleted", false);
// Now all queries on this session filter out deleted records
session.disableFilter("notDeleted");
```

---

## 17. `@Transactional` in Spring Data JPA

```java
// Spring Data repository methods already have @Transactional declared:
// - save(), delete(), findAll() etc. are @Transactional
// - findById(), findAll(Pageable) etc. are @Transactional(readOnly = true)

// Override in your own repository:
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Override readOnly for specific queries
    @Override
    @Transactional(readOnly = true)
    List<Product> findAll();

    // Add timeout
    @Transactional(timeout = 10)
    @Query("SELECT p FROM Product p WHERE p.active = true")
    List<Product> findActiveWithTimeout();
}

// In service — always control transaction at service layer:
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepo;

    // Read — mark readOnly for performance
    @Transactional(readOnly = true)
    public Page<ProductDto> getProducts(Pageable pageable) {
        return productRepo.findAll(pageable).map(ProductDto::from);
    }

    // Write — default @Transactional
    @Transactional
    public ProductDto create(CreateProductCommand cmd) {
        Product product = new Product(cmd);
        productRepo.save(product);
        return ProductDto.from(product);
    }

    // Multi-step write — single transaction wraps multiple repo calls
    @Transactional
    public void transferStock(Long fromId, Long toId, int quantity) {
        Product from = productRepo.findById(fromId).orElseThrow();
        Product to   = productRepo.findById(toId).orElseThrow();

        if (from.getStockCount() < quantity) throw new InsufficientStockException();

        from.setStockCount(from.getStockCount() - quantity);
        to.setStockCount(to.getStockCount() + quantity);
        // Dirty checking flushes both updates in a single transaction on commit
        // No explicit save() needed
    }
}
```

---

## 18. Streaming Large Results

```java
// Use Stream<T> for large result sets — avoids loading all into memory
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE p.active = true")
    @Transactional(readOnly = true)
    Stream<Product> streamAllActive();
}

@Service
@RequiredArgsConstructor
public class ProductExportService {

    private final ProductRepository productRepo;

    // MUST be @Transactional — stream uses an open Hibernate cursor
    @Transactional(readOnly = true)
    public void exportToCsv(OutputStream out) {
        try (
            Stream<Product> stream = productRepo.streamAllActive();
            PrintWriter writer = new PrintWriter(out)
        ) {
            writer.println("id,sku,name,price");
            stream
                .map(p -> p.getId() + "," + p.getSku() + "," + p.getName() + "," + p.getPrice())
                .forEach(writer::println);
        }
        // Stream MUST be closed — it holds an open JDBC ResultSet
        // try-with-resources ensures closure
    }
}

// Also set JDBC fetch size for true streaming (otherwise Hibernate loads all at once):
@QueryHints(@QueryHint(name = "org.hibernate.fetchSize", value = "1000"))
@Query("SELECT p FROM Product p WHERE p.active = true")
@Transactional(readOnly = true)
Stream<Product> streamAllActive();
```

---

## 19. Testing with `@DataJpaTest`

```java
@DataJpaTest
// Loads ONLY: @Repository, @Entity, DataSource, JPA config
// Does NOT load: @Service, @Controller, @Component
// Replaces DataSource with H2 in-memory (by default)
// Each test is @Transactional + rollback — data doesn't leak between tests
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepo;

    @Autowired
    private TestEntityManager em;  // wrapper for EntityManager in tests

    @Test
    void shouldFindBySkuIgnoringCase() {
        // Arrange — persist test data
        Product product = new Product("SKU-001", "Laptop", new BigDecimal("999.99"));
        em.persistAndFlush(product);  // persist + flush to DB
        em.clear();                   // clear first-level cache (simulate real query)

        // Act
        Optional<Product> found = productRepo.findBySku("SKU-001");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Laptop");
    }

    @Test
    void shouldReturnPageOfActiveProducts() {
        em.persist(new Product("SKU-001", "Laptop", true));
        em.persist(new Product("SKU-002", "Mouse",  true));
        em.persist(new Product("SKU-003", "Old",    false));
        em.flush();
        em.clear();

        Page<Product> page = productRepo.findByActiveTrue(PageRequest.of(0, 10));

        assertThat(page.getTotalElements()).isEqualTo(2);
        assertThat(page.getContent()).extracting(Product::getName)
                                     .containsExactlyInAnyOrder("Laptop", "Mouse");
    }

    @Test
    void shouldDetectNPlusOneWithStatistics() {
        // Seed orders with customers
        for (int i = 0; i < 5; i++) {
            Customer c = em.persist(new Customer("Customer " + i));
            em.persist(new Order(c, new BigDecimal("100")));
        }
        em.flush();
        em.clear();

        // Get Hibernate statistics
        SessionFactory sf = em.getEntityManager()
                              .getEntityManagerFactory()
                              .unwrap(SessionFactory.class);
        sf.getStatistics().clear();

        // Execute method under test
        List<Order> orders = orderRepo.findAllWithCustomer();

        // Assert: should be 1 query (JOIN FETCH), not 1+N
        assertThat(sf.getStatistics().getQueryExecutionCount())
            .as("Should use JOIN FETCH, not N+1")
            .isEqualTo(1);
    }
}

// Real database test (don't replace with H2)
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class ProductRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ProductRepository productRepo;

    @Test
    void shouldUsePostgresSpecificFeatures() {
        // Test with real PostgreSQL — e.g., full-text search, UUID, jsonb
    }
}
```

---

## 20. Complete Real-World Example — E-Commerce Platform

```java
// ─── Entity ───────────────────────────────────────────────────────────────────

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_products_sku",      columnList = "sku",            unique = true),
    @Index(name = "idx_products_category", columnList = "category_id, active"),
    @Index(name = "idx_products_active",   columnList = "active, created_at")
})
@EntityListeners(AuditingEntityListener.class)
@NamedEntityGraph(
    name = "Product.withCategoryAndImages",
    attributeNodes = {
        @NamedAttributeNode("category"),
        @NamedAttributeNode("images")
    }
)
public class Product extends Auditable {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_id_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true, length = 50, updatable = false)
    private String sku;

    @Column(nullable = false)
    private String name;

    @Column(precision = 19, scale = 4, nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private int stockCount;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @OrderBy("sortOrder ASC")
    private List<ProductImage> images = new ArrayList<>();

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name = "product_tags", joinColumns = @JoinColumn(name = "product_id"))
    @Column(name = "tag")
    private Set<String> tags = new HashSet<>();

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ProductStatus status = ProductStatus.DRAFT;

    @Column(nullable = false)
    private boolean active = false;

    @Version
    private Long version;
}

// ─── Projections ──────────────────────────────────────────────────────────────

public interface ProductCatalogView {
    Long getId();
    String getSku();
    String getName();
    BigDecimal getPrice();
    int getStockCount();
    String getCategoryName();    // nested: category.name
}

public record ProductSearchResult(
    Long id, String sku, String name,
    BigDecimal price, int stockCount,
    String categoryName, boolean active
) {}

// ─── Repository ───────────────────────────────────────────────────────────────

public interface ProductRepository
        extends JpaRepository<Product, Long>,
                JpaSpecificationExecutor<Product>,
                ProductRepositoryCustom {

    // Catalog listing — projection
    @EntityGraph("Product.withCategoryAndImages")
    Optional<Product> findByIdAndActiveTrue(Long id);

    // Catalog listing — interface projection (SELECT only needed cols)
    List<ProductCatalogView> findByCategoryAndActiveTrue(Category category, Sort sort);
    Page<ProductCatalogView> findByCategoryAndActiveTrue(Category category, Pageable pageable);

    // Admin lookup
    @EntityGraph(attributePaths = {"category", "images", "tags"})
    Optional<Product> findBySku(String sku);

    // Existence checks
    boolean existsBySku(String sku);
    boolean existsBySkuAndIdNot(String sku, Long id);  // for update uniqueness check

    // Stock-related
    List<ProductCatalogView> findByStockCountLessThanAndActiveTrue(int threshold);

    // Bulk status update
    @Modifying(clearAutomatically = true)
    @Query("UPDATE Product p SET p.active = :active WHERE p.id IN :ids")
    int updateActiveStatus(@Param("ids") List<Long> ids, @Param("active") boolean active);

    // Revenue report
    @Query("""
        SELECT new com.app.dto.ProductRevenueDto(
            p.id, p.sku, p.name, SUM(oi.quantity), SUM(oi.unitPrice * oi.quantity)
        )
        FROM Product p
        JOIN OrderItem oi ON oi.product = p
        JOIN oi.order o
        WHERE o.status = 'COMPLETED'
        AND o.completedAt BETWEEN :from AND :to
        GROUP BY p.id, p.sku, p.name
        ORDER BY SUM(oi.unitPrice * oi.quantity) DESC
        """)
    List<ProductRevenueDto> findRevenueReport(
        @Param("from") Instant from,
        @Param("to") Instant to,
        Pageable pageable
    );

    // Streaming for export
    @Query("SELECT p FROM Product p WHERE p.active = true ORDER BY p.id")
    @QueryHints(@QueryHint(name = "org.hibernate.fetchSize", value = "500"))
    @Transactional(readOnly = true)
    Stream<Product> streamAllActive();
}

// ─── Custom Repository Impl ───────────────────────────────────────────────────

public interface ProductRepositoryCustom {
    void bulkUpdatePrices(Map<Long, BigDecimal> priceMap);
}

@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom {

    private final JdbcTemplate jdbcTemplate;

    @Override
    @Transactional
    public void bulkUpdatePrices(Map<Long, BigDecimal> priceMap) {
        List<Object[]> params = priceMap.entrySet().stream()
            .map(e -> new Object[]{e.getValue(), e.getKey()})
            .collect(Collectors.toList());

        jdbcTemplate.batchUpdate("UPDATE products SET price = ? WHERE id = ?", params);
    }
}

// ─── Specifications ───────────────────────────────────────────────────────────

public final class ProductSpec {

    private ProductSpec() {}

    public static Specification<Product> active() {
        return (r, q, cb) -> cb.isTrue(r.get("active"));
    }

    public static Specification<Product> inCategory(Long categoryId) {
        return (r, q, cb) -> categoryId == null ? null :
            cb.equal(r.join("category", JoinType.INNER).get("id"), categoryId);
    }

    public static Specification<Product> nameLike(String kw) {
        return (r, q, cb) -> kw == null ? null :
            cb.like(cb.lower(r.get("name")), "%" + kw.toLowerCase() + "%");
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (r, q, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.le(r.get("price"), max);
            if (max == null) return cb.ge(r.get("price"), min);
            return cb.between(r.get("price"), min, max);
        };
    }

    public static Specification<Product> hasTag(String tag) {
        return (r, q, cb) -> {
            if (tag == null) return null;
            Join<Product, String> tagJoin = r.join("tags", JoinType.INNER);
            return cb.equal(tagJoin, tag);
        };
    }
}

// ─── Service ──────────────────────────────────────────────────────────────────

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepo;
    private final CategoryRepository categoryRepo;

    @Transactional(readOnly = true)
    public ProductDetailDto getProductDetail(Long id) {
        Product product = productRepo.findByIdAndActiveTrue(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        return ProductDetailDto.from(product);
        // EntityGraph ensures category + images loaded in 1 query — no N+1
    }

    @Transactional(readOnly = true)
    public Page<ProductCatalogView> catalog(CatalogRequest req, Pageable pageable) {
        if (req.getCategoryId() != null) {
            Category cat = categoryRepo.getReferenceById(req.getCategoryId());
            return productRepo.findByCategoryAndActiveTrue(cat, pageable);
        }
        // Dynamic spec for advanced search
        Specification<Product> spec = Specification
            .where(ProductSpec.active())
            .and(ProductSpec.nameLike(req.getKeyword()))
            .and(ProductSpec.priceBetween(req.getMinPrice(), req.getMaxPrice()))
            .and(ProductSpec.hasTag(req.getTag()));

        return productRepo.findAll(spec, pageable)
                          .map(p -> new ProductCatalogView() { /* anonymous impl */ });
    }

    @Transactional
    public ProductDto create(CreateProductCommand cmd) {
        if (productRepo.existsBySku(cmd.getSku())) {
            throw new DuplicateSkuException(cmd.getSku());
        }
        Category category = categoryRepo.findById(cmd.getCategoryId())
            .orElseThrow(() -> new CategoryNotFoundException(cmd.getCategoryId()));

        Product product = new Product(cmd.getSku(), cmd.getName(), cmd.getPrice(), category);
        productRepo.save(product);
        return ProductDto.from(product);
    }

    @Transactional
    public ProductDto update(Long id, UpdateProductCommand cmd) {
        Product product = productRepo.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));

        if (!product.getSku().equals(cmd.getSku()) && productRepo.existsBySkuAndIdNot(cmd.getSku(), id)) {
            throw new DuplicateSkuException(cmd.getSku());
        }

        product.update(cmd);  // dirty checking handles the UPDATE on commit
        return ProductDto.from(product);
        // No productRepo.save() needed — dirty checking tracks the change
    }

    @Transactional
    public void publish(Long id) {
        Product product = productRepo.findById(id).orElseThrow();
        product.publish();  // domain method validates and changes status
    }

    @Transactional(readOnly = true)
    public void exportCsv(HttpServletResponse response) throws IOException {
        response.setContentType("text/csv");
        response.setHeader("Content-Disposition", "attachment; filename=products.csv");

        try (
            Stream<Product> stream = productRepo.streamAllActive();
            PrintWriter writer = response.getWriter()
        ) {
            writer.println("id,sku,name,price,stock");
            stream.map(p -> String.join(",",
                    p.getId().toString(), p.getSku(), p.getName(),
                    p.getPrice().toPlainString(), String.valueOf(p.getStockCount())
            )).forEach(writer::println);
        }
    }
}
```

---

## 21. Scenario Walkthroughs

### Scenario 1: Pagination with JOIN FETCH — The "In Memory Pagination" Trap

**System:** Admin panel lists orders with their customer name. Page size 20.

```java
// ❌ Trap:
@EntityGraph(attributePaths = {"customer"})
Page<Order> findAll(Pageable pageable);
// Hibernate log: "HHH90003004: firstResult/maxResults specified with collection fetch;
//                applying in memory!"
// Hibernate fetches ALL orders with JOIN, then paginates in Java heap — not in DB!
// On 1 million orders → OOM.

// ❌ Also broken for collections (items):
@EntityGraph(attributePaths = {"items"})
Page<Order> findAll(Pageable pageable);  // same problem — items is a collection

// ✅ Fix 1: Use DTO projection — no entity, no join issues
@Query("""
    SELECT new com.app.dto.OrderSummaryDto(
        o.id, o.total, o.status, o.createdAt, c.name, c.email
    )
    FROM Order o JOIN o.customer c
    """)
Page<OrderSummaryDto> findAllSummaries(Pageable pageable);
// Single query, paginated in SQL (LIMIT/OFFSET), no entity overhead

// ✅ Fix 2: Use @ManyToOne (not a collection) — safe for pagination
@EntityGraph(attributePaths = {"customer"})  // customer is @ManyToOne — not a collection
Page<Order> findAll(Pageable pageable);
// @EntityGraph on @ManyToOne is safe — no collection = no Cartesian product
// This works correctly.

// ✅ Fix 3: Two-query approach for true pagination with collection
@Transactional(readOnly = true)
public Page<OrderDto> getOrdersWithItems(Pageable pageable) {
    // Query 1: paginate order IDs only (fast, uses DB LIMIT/OFFSET)
    Page<Long> orderIds = orderRepo.findAllIds(pageable);

    if (orderIds.isEmpty()) return Page.empty(pageable);

    // Query 2: load full data for those IDs (no pagination — already narrowed)
    List<Order> orders = orderRepo.findAllWithItemsByIdIn(orderIds.getContent());

    // Sort to match original order
    Map<Long, Order> orderMap = orders.stream()
        .collect(Collectors.toMap(Order::getId, o -> o));
    List<OrderDto> content = orderIds.getContent().stream()
        .map(id -> OrderDto.from(orderMap.get(id)))
        .collect(Collectors.toList());

    return new PageImpl<>(content, pageable, orderIds.getTotalElements());
}

@Query("SELECT o.id FROM Order o ORDER BY o.createdAt DESC")
Page<Long> findAllIds(Pageable pageable);

@EntityGraph(attributePaths = {"items", "items.product", "customer"})
@Query("SELECT DISTINCT o FROM Order o WHERE o.id IN :ids")
List<Order> findAllWithItemsByIdIn(@Param("ids") List<Long> ids);
```

**Interviewer's intent:** Tests whether you understand the Hibernate in-memory pagination warning and can distinguish `@ManyToOne` (safe) from `@OneToMany`/`@ManyToMany` (dangerous) when combined with paginated queries.

---

### Scenario 2: Derived Query Ambiguity and Fail-Fast Validation

**System:** A new developer writes `findByCustomerAddress` but `Customer` has two embedded addresses — `billingAddress` and `shippingAddress`.

```java
// ❌ Ambiguous — Spring Data can't resolve "Address" — which address?
List<Order> findByCustomerAddress(Address address);
// Throws: PropertyReferenceException at startup
// "No property 'address' found for type 'Customer'! Traversed: Order.customer"

// ✅ Be explicit — use the exact field path
List<Order> findByCustomerBillingAddress(Address address);
List<Order> findByCustomerShippingAddressCity(String city);

// ✅ Or use @Query for clarity
@Query("SELECT o FROM Order o WHERE o.customer.billingAddress.zipCode = :zip")
List<Order> findByBillingZipCode(@Param("zip") String zipCode);
```

This is actually a **positive** Spring Data JPA feature: invalid method names cause startup failure — not runtime exceptions. This means you catch naming mistakes immediately in CI, not in production.

---

### Scenario 3: `save()` Returns a New Object — A Subtle Bug

**System:** Service persists an entity and immediately uses the returned object.

```java
// Entity with @Version and @CreatedDate
@Entity
public class Product {
    @Id @GeneratedValue private Long id;
    @Version private Long version;
    @CreatedDate @Column(updatable = false) private Instant createdAt;
    private String name;
}

// ❌ Common mistake:
@Transactional
public void createProduct(String name) {
    Product product = new Product(name);
    productRepo.save(product);

    // What is product.getId() here?
    // If strategy = IDENTITY → flush happened, id is populated ✓
    // If strategy = SEQUENCE → depends on timing

    // What is product.getVersion() here?
    // If strategy = SEQUENCE → save() called em.persist(), NOT em.merge()
    //   → version is populated only after flush ✓
    // But: product.getCreatedAt() might be null if @CreatedDate is set in @PrePersist
    //   and no flush happened yet...

    log.info("Created: id={}, version={}", product.getId(), product.getVersion());
    // Safest: call saveAndFlush() for immediate consistency
}

// ✅ For update (detached entity):
@Transactional
public Product updateProduct(Product detachedProduct) {
    Product saved = productRepo.save(detachedProduct);
    // save() on entity with non-null ID calls em.merge()
    // em.merge() returns a NEW persistent copy — the original is still detached!

    // ❌ Wrong: using detachedProduct after merge
    detachedProduct.setName("changed");  // this change is NOT tracked

    // ✅ Correct: use 'saved' (the returned managed copy)
    saved.setName("changed");            // this is tracked by dirty checking

    return saved;
}
```

**Interviewer's intent:** Tests whether you know that `save()` on an existing entity calls `merge()`, which returns a different object, and that the argument to `save()` remains detached.

---

## 22. Common Interview Questions

**Q1: What is the difference between `CrudRepository`, `JpaRepository`, and `JpaSpecificationExecutor`?**

`CrudRepository` is a Spring Data interface providing basic CRUD (`save`, `findById`, `deleteById`, etc.) for any store. `JpaRepository` extends `PagingAndSortingRepository` (which adds paginated and sorted `findAll`) and adds JPA-specific methods: `flush()`, `saveAndFlush()`, `deleteAllInBatch()`, `getReferenceById()`. `JpaSpecificationExecutor` is a separate interface you mix in alongside `JpaRepository` — it adds `findAll(Specification)`, `findOne(Specification)`, `count(Specification)` to enable Criteria API-based dynamic queries. In practice, extend `JpaRepository + JpaSpecificationExecutor` for entities that need both pagination and dynamic filtering.

---

**Q2: How does Spring Data JPA generate the query for a method named `findByCustomerEmailAndStatusOrderByCreatedAtDesc`?**

At application startup, Spring Data parses the method name using a keyword parser. It strips `findBy`, then splits on keywords (`And`, `Or`, `OrderBy`). For each segment, it resolves the property path against the entity's type (`customer.email`, `status`). For `OrderBy`, it creates an `ORDER BY` clause. It generates JPQL: `SELECT o FROM Order o WHERE o.customer.email = ?1 AND o.status = ?2 ORDER BY o.createdAt DESC`. If any property name is invalid (e.g., `customer.typo`), Spring throws `PropertyReferenceException` at startup — not at query execution. This is the "fail-fast" benefit of derived queries.

---

**Q3: What is the difference between `Page<T>` and `Slice<T>`?**

Both represent a page of results from a paginated query. `Page` fires two queries: the data query (with `LIMIT`/`OFFSET`) and a `COUNT(*)` query to calculate `totalElements` and `totalPages`. `Slice` fires only the data query — it doesn't know the total count. `Slice.hasNext()` works by requesting `pageSize + 1` rows and checking if more than `pageSize` were returned. Use `Page` when you need to display "Page 3 of 47" (total count required). Use `Slice` for infinite scroll / "load more" UI patterns (only need to know if there's a next page), because the `COUNT` query can be expensive on large tables.

---

**Q4: What happens when you call `productRepo.save(existingProduct)` where `existingProduct` has a non-null ID?**

`save()` in `SimpleJpaRepository` calls `em.persist()` if the entity is new (ID is null or entity implements `Persistable` and `isNew()` returns true), otherwise calls `em.merge()`. For an entity with a non-null ID: `em.merge(existingProduct)` is called. This copies the state from `existingProduct` onto a persistent copy and returns that copy. The returned object is managed (tracked by dirty checking). The original `existingProduct` remains detached — changes to it after `save()` are NOT tracked. Always use the return value of `save()` when the entity may be detached.

---

**Q5: When would you use `@Query` instead of a derived query method?**

Use `@Query` when: (1) the derived method name would be too long or unreadable; (2) you need `JOIN FETCH` to solve N+1; (3) you need aggregate functions (`COUNT`, `SUM`, `GROUP BY`); (4) you need a subquery or `EXISTS`; (5) you need a `HAVING` clause; (6) you need a database-specific function (use `nativeQuery=true`); (7) you need a DTO constructor expression. Derived queries are great for simple, single-table lookups. Complex multi-join queries are better expressed explicitly so they're readable and optimizable.

---

**Q6: What is the `open-in-view` anti-pattern? Why must it be disabled?**

Open-in-view (OSIV) is a Spring MVC filter (enabled by default in Spring Boot) that opens a Hibernate session at the start of an HTTP request and closes it when the response is fully written. This allows lazy associations to be accessed in the controller and view layer without `LazyInitializationException`. It must be disabled (`spring.jpa.open-in-view=false`) because: (1) it silently enables lazy loading anywhere in the request, hiding N+1 bugs that only appear under load; (2) it holds a DB connection for the full duration of the request — including slow JSON serialization, Jackson processing, or template rendering — reducing connection pool throughput; (3) it encourages leaking entities into the web layer (entities should be converted to DTOs inside the service layer). The correct pattern is: explicit fetch strategies in the service layer, DTO return types, no entities in controllers.

---

**Q7: How do Specifications compose? Give an example of AND + OR + NOT composition.**

`Specification<T>` implements the Composite pattern. `and()`, `or()`, and `not()` return new `Specification` instances combining predicates:

```java
Specification<Product> spec = Specification
    .where(ProductSpec.active())                              // active = true
    .and(ProductSpec.priceBetween(null, new BigDecimal("500")))  // AND price <= 500
    .and(
        ProductSpec.inCategory(1L)                           // AND (category = 1
        .or(ProductSpec.hasTag("featured"))                  //      OR has tag 'featured')
    )
    .and(Specification.not(ProductSpec.nameLike("discontinued")));  // AND name NOT LIKE
```

Each `Specification` is a functional interface `(Root<T>, CriteriaQuery<?>, CriteriaBuilder) → Predicate`. Returning `null` from a specification means "no predicate" — Spring Data ignores it. This makes conditionally adding predicates trivial: just return `null` for absent filter values.

---

**Q8: What is `@Modifying(clearAutomatically = true)` and when do you need it?**

`@Modifying` is required on `@Query` methods that perform `UPDATE` or `DELETE` — without it, Spring Data throws an exception. `clearAutomatically = true` tells Spring Data to call `em.clear()` on the persistence context after executing the bulk operation. This is necessary because bulk JPQL `UPDATE`/`DELETE` bypasses Hibernate's first-level cache: if you load a `Product`, then run a bulk `UPDATE products SET price = ...`, the product object in your cache still has the old price. `clearAutomatically = true` evicts all cached entities after the bulk statement, so subsequent `findById` calls re-fetch from DB. Combine with `flushAutomatically = true` when there are pending dirty changes that the bulk query needs to see.

---

## 23. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **`spring.jpa.open-in-view=true` (default)** | Hidden N+1, DB connections held for full HTTP request | Set `open-in-view=false`; use explicit fetch in service; return DTOs |
| **`@EntityGraph` on a `Page` with `@OneToMany`** | Hibernate paginates in memory (HHH90003004 warning); OOM on large data | Use DTO projection, or two-query approach (paginate IDs, then fetch) |
| **Using `findAll()` without conditions** | Loads entire table — destructive on large tables | Always filter by at least one indexed column; add `@Query` limits |
| **`deleteAll()` vs `deleteAllInBatch()`** | `deleteAll()` loads all entities first (N+1), then N DELETE statements | Use `deleteAllInBatch()` or `deleteAllByIdInBatch()` for bulk deletes |
| **Not using the return value of `save()` on a detached entity** | `save()` on detached calls `merge()` which returns a new managed copy; original stays detached | Always reassign: `product = productRepo.save(product)` |
| **Returning entities from service layer** | Jackson serializes lazy collections → N+1, circular refs, `LazyInitializationException` | Return DTOs; never let entities cross the service boundary |
| **`@Modifying` without `clearAutomatically = true`** | First-level cache returns stale data after bulk update | Add `clearAutomatically = true` to `@Modifying` |
| **Derived query method resolves wrong property** | Ambiguous paths (e.g., two nested `address` fields) cause startup failure | Use explicit `@Query` or fully-qualified property paths |
| **Sorting by user-supplied field without whitelist** | SQL injection via `Sort` parameter | Whitelist allowed sort fields before building `Sort` / `Pageable` |
| **`JOIN FETCH` with `Page`** | Need `countQuery` in `@Query`; without it, Spring Data derives a wrong count query from the JOIN FETCH | Always provide explicit `countQuery` when using `JOIN FETCH` in `@Query` |
| **Multiple `JOIN FETCH` on collections** | `MultipleBagFetchException` | Only one collection per `JOIN FETCH`; use `@BatchSize` or separate queries for additional collections |
| **`Stream<T>` not closed** | Holds JDBC `ResultSet` and `Connection` open → resource leak | Always use try-with-resources: `try (Stream<T> s = repo.stream...)` |
| **`@Transactional(readOnly=true)` on `save()`** | `FlushMode.MANUAL` → update never flushed | Use default `@Transactional` for write operations |
| **Testing with H2 but running PostgreSQL** | H2 silently ignores PostgreSQL-specific SQL → false test passes | Use `@AutoConfigureTestDatabase(replace=NONE)` + Testcontainers |
| **`@Query` with named params and `@Param` mismatch** | `IllegalArgumentException` at startup or runtime: param name not found | Use `-parameters` compiler flag or always use `@Param("name")` |
| **Using `JpaRepository.getOne()` (deprecated)** | `getOne()` is deprecated since Spring Data 2.7; throws `EntityNotFoundException` on access if ID not found | Use `getReferenceById()` (proxy) or `findById()` (returns Optional) |

---

## 24. Summary — Quick Revision

- **Spring Data JPA = repository abstraction over JPA.** Write an interface extending `JpaRepository`; Spring generates the implementation at startup. Derived queries are parsed and validated at startup — fail fast.
- **Always `@Transactional(readOnly=true)` on reads.** Sets `FlushMode.MANUAL` — no dirty checking, no accidental writes, read replica hint. Apply at service layer.
- **Return DTOs, never entities.** Entities must not cross the service boundary. Use interface projections (SELECT specific columns), DTO constructor expressions, or `record` projections.
- **`@EntityGraph` for fetch control.** Solves N+1 without changing entity annotations. Safe for `@ManyToOne`/`@OneToOne`. Dangerous for `@OneToMany` with pagination — use two-query approach or DTO instead.
- **`Specification` for dynamic filters.** Composable, type-safe, null-safe predicates. Use for admin search and filter pages. For fixed queries, `@Query` is simpler and more readable.
- **`deleteAllInBatch()` for bulk deletes.** Never use `deleteAll()` for more than a handful of records. `deleteAllInBatch()` = single SQL statement, no entity loading, no cascade.
- **Disable OSIV.** `spring.jpa.open-in-view=false`. Without exception.

---

> **What's next:** You now have the full data layer picture — JDBC → Spring JDBC → Spring Data JDBC → Hibernate → JPA → Spring Data JPA. From here, natural next topics are: Spring Data JPA with Querydsl for even more type-safe queries, Hibernate Envers for entity versioning/audit history, and Spring Cache abstraction for L2-equivalent caching at the application layer.

> **Follow-up questions to think about:**
> 1. You have a `Page<Order>` query with `@EntityGraph(attributePaths = {"items"})`. The `items` is a `@OneToMany` collection. Spring Boot logs a HHH90003004 warning. What exactly is Hibernate doing wrong, and what are your two options to fix it?
> 2. A `ProductRepository` method is annotated `@Transactional(readOnly = true)`. A developer calls `productRepo.save(product)` from within a service method also annotated `@Transactional(readOnly = true)`. What happens at runtime?
> 3. You have a `Specification<Product>` that joins the `reviews` collection to filter products with 5-star reviews. When you call `productRepo.findAll(spec, pageable)`, the result has duplicate `Product` entries. Why, and how do you fix it within the Specification?
