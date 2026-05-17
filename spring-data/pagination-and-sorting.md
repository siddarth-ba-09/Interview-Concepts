# Pagination & Sorting in Spring Data JPA — Complete Guide

> **Why this matters in interviews:** Every production app that lists data must paginate. Interviewers test whether you understand the difference between `Page` and `Slice`, know about the N+1 count-query problem, can implement cursor-based pagination, and can safely expose sorting to API callers without opening injection vectors.

---

## Table of Contents

1. [Core Abstractions](#1-core-abstractions)
2. [Pageable — Building Page Requests](#2-pageable)
3. [Sort — Building Sort Specifications](#3-sort)
4. [Page vs Slice vs List](#4-page-vs-slice-vs-list)
5. [Repository Methods — Derived Queries](#5-repository-methods)
6. [@Query with Pageable](#6-query-with-pageable)
7. [Specifications + Pagination](#7-specifications--pagination)
8. [REST Controller Integration](#8-rest-controller-integration)
9. [Cursor-Based / Keyset Pagination](#9-cursor-based--keyset-pagination)
10. [Count Query Optimisation](#10-count-query-optimisation)
11. [Sorting Pitfalls & Security](#11-sorting-pitfalls--security)
12. [Production Patterns & Pitfalls](#12-production-patterns--pitfalls)
13. [Interview Questions & Answers](#13-interview-questions)

---

## 1. Core Abstractions

Spring Data JPA pagination is built on four interfaces/classes:

```
Pageable            ← input: which page, how many rows, in what order
  └── PageRequest   ← concrete implementation

Page<T>             ← output: content + total element count + metadata
Slice<T>            ← output: content + hasNext flag (NO count query)

Sort                ← standalone ordering spec (can be embedded in Pageable)
  └── Sort.Order    ← single field ordering
```

---

## 2. Pageable

`Pageable` is the **request object** you pass into repository methods. It carries:
- **Page number** (0-indexed)
- **Page size** (number of records)
- **Sort** (optional)

### 2.1 Creating Pageable Instances

```java
// Page 0, 20 records per page, no sort
Pageable pageable = PageRequest.of(0, 20);

// Page 1, 10 records, sorted by lastName ASC
Pageable pageable = PageRequest.of(1, 10, Sort.by("lastName"));

// Multi-field sort
Pageable pageable = PageRequest.of(0, 10,
    Sort.by(Sort.Direction.ASC, "lastName")
        .and(Sort.by(Sort.Direction.DESC, "createdAt")));

// Using Sort object separately
Sort sort = Sort.by("price").descending().and(Sort.by("name").ascending());
Pageable pageable = PageRequest.of(0, 25, sort);
```

### 2.2 Pageable Internals

```
PageRequest {
    page:   0              ← zero-based index
    size:   20
    sort:   price DESC
}

Translates to SQL:
    SELECT * FROM product
    ORDER BY price DESC
    LIMIT 20 OFFSET 0
```

### 2.3 Unpaged — Disable Pagination

```java
// Returns all results with a sort but no LIMIT/OFFSET
Pageable all = Pageable.unpaged();
Pageable allSorted = PageRequest.of(Pageable.unpaged(), sort); // Spring 3.x
```

Useful when you want to reuse a pageable-aware repository method without actually paginating (e.g., internal batch jobs).

---

## 3. Sort

`Sort` can be used **standalone** (without pagination) or embedded inside `Pageable`.

### 3.1 Building Sort Objects

```java
// Single field, default ASC
Sort sort = Sort.by("name");

// Explicit direction
Sort sort = Sort.by(Sort.Direction.DESC, "createdAt");

// Multiple fields — chained
Sort sort = Sort.by("category").ascending()
               .and(Sort.by("price").descending());

// Multiple fields — varargs shortcut (all same direction)
Sort sort = Sort.by(Sort.Direction.ASC, "lastName", "firstName");

// Null handling
Sort sort = Sort.by(Sort.Order.by("price")
    .with(Sort.NullHandling.NULLS_LAST));

// Case-insensitive sort
Sort sort = Sort.by(Sort.Order.by("name").ignoreCase());
```

### 3.2 Sort.Direction

```java
Sort.Direction.ASC    // ascending (default)
Sort.Direction.DESC   // descending
Sort.Direction.fromString("asc")   // parse from string
Sort.Direction.fromOptionalString("DESC")  // returns Optional
```

### 3.3 Sort.Order Properties

| Property | Default | Description |
|---|---|---|
| `property` | required | The entity field name (NOT column name) |
| `direction` | `ASC` | Sort direction |
| `ignoreCase` | `false` | `LOWER(column)` wrapping in SQL |
| `nullHandling` | `NATIVE` | `NULLS FIRST`, `NULLS LAST`, or DB default |

---

## 4. Page vs Slice vs List

This is one of the most important distinctions for performance.

### 4.1 `Page<T>` — Full Pagination Metadata

```java
Page<Product> page = productRepository.findAll(pageable);

page.getContent();           // List<Product> — the actual data
page.getTotalElements();     // total record count across ALL pages
page.getTotalPages();        // total pages = ceil(totalElements / pageSize)
page.getNumber();            // current page number (0-indexed)
page.getSize();              // page size
page.getNumberOfElements();  // records in THIS page (may be < size on last page)
page.hasNext();              // is there a next page?
page.hasPrevious();          // is there a previous page?
page.isFirst();              // is this the first page?
page.isLast();               // is this the last page?
page.nextPageable();         // Pageable for the next page
page.previousPageable();     // Pageable for the previous page
page.getPageable();          // Pageable used to get this page
```

**What Spring generates:**
```sql
-- Data query
SELECT * FROM product ORDER BY price DESC LIMIT 20 OFFSET 0;

-- Count query (ALWAYS fired for Page<T>)
SELECT COUNT(*) FROM product;
```

**Use when:** You need total page count for a UI paginator ("Page 3 of 47").

---

### 4.2 `Slice<T>` — Next-Page-Only

```java
Slice<Product> slice = productRepository.findByActiveTrue(pageable);

slice.getContent();          // List<Product>
slice.hasNext();             // fetches size+1 rows to determine this
slice.hasPrevious();
slice.getNumber();
slice.getSize();
slice.getNumberOfElements();
// NO: getTotalElements(), getTotalPages() — these don't exist on Slice
```

**What Spring generates:**
```sql
-- fetches size+1 rows; if more than size rows returned → hasNext = true
SELECT * FROM product WHERE active = true ORDER BY price DESC LIMIT 21 OFFSET 0;

-- NO count query fired
```

**Use when:** Infinite scroll, "Load More" buttons, mobile feeds — you only need to know if more data exists, not how much total.

**Performance advantage:** Eliminates the often-expensive `COUNT(*)` query.

---

### 4.3 `List<T>` — No Metadata

```java
List<Product> products = productRepository.findByCategory(category, pageable);
// or
List<Product> products = productRepository.findByCategory(category, sort);
```

**What Spring generates:**
```sql
SELECT * FROM product WHERE category_id = ? ORDER BY price DESC LIMIT 20 OFFSET 0;
-- NO count query; no hasNext check
```

**Use when:** Internal service calls where you just want a sorted/limited list without needing page metadata.

---

### 4.4 Comparison Table

| Feature | `Page<T>` | `Slice<T>` | `List<T>` |
|---|---|---|---|
| Data content | Yes | Yes | Yes |
| Total count | Yes | **No** | **No** |
| Total pages | Yes | **No** | **No** |
| `hasNext()` | Yes | Yes | **No** |
| Count query | **Always fired** | **Never fired** | **Never fired** |
| Overhead | Highest | Low | Lowest |
| UI paginator | Best fit | Not suitable | Not suitable |
| Infinite scroll | Overkill | **Best fit** | Works |
| Internal batch | Overkill | Good | **Best fit** |

---

## 5. Repository Methods — Derived Queries

### 5.1 Adding Pageable to Derived Methods

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Page — with count query
    Page<Product> findByCategory(Category category, Pageable pageable);

    // Slice — no count query
    Slice<Product> findByActiveTrue(Pageable pageable);

    // List — no metadata
    List<Product> findByPriceLessThan(BigDecimal maxPrice, Pageable pageable);

    // With Sort only (no pagination)
    List<Product> findByCategory(Category category, Sort sort);

    // Top/First — implicit limit, no Pageable needed
    List<Product> findTop5ByOrderByCreatedAtDesc();
    Optional<Product> findFirstByOrderByPriceAsc();
}
```

### 5.2 Top/First Limiting Keywords

```java
// Limit by keyword — returns List (no metadata)
List<Product>        findTop10ByCategory(Category c);
List<Product>        findFirst10ByCategory(Category c);    // synonymous with Top

// Combined with sort keyword
List<Product>        findTop5ByOrderByPriceAsc();
Optional<Product>    findTopByOrderByCreatedAtDesc();       // most recent single record
```

`Top` and `First` generate `LIMIT N` in SQL — these do NOT fire a count query and do NOT accept a `Pageable`.

### 5.3 Distinct Pagination

```java
Page<Product> findDistinctByTagsIn(List<Tag> tags, Pageable pageable);
```

Caution: `DISTINCT` combined with `COUNT(*)` can be expensive and may give surprising results with JOINs — see [Count Query Optimisation](#10-count-query-optimisation).

---

## 6. @Query with Pageable

### 6.1 Basic Usage

```java
@Query("SELECT p FROM Product p WHERE p.price > :minPrice AND p.active = true")
Page<Product> findActivePricedAbove(@Param("minPrice") BigDecimal minPrice, Pageable pageable);
```

Spring automatically appends the `ORDER BY` and `LIMIT/OFFSET` from the `Pageable` to your JPQL query.

**The count query Spring auto-generates:**
```jpql
SELECT COUNT(p) FROM Product p WHERE p.price > :minPrice AND p.active = true
```

### 6.2 Custom Count Query

When the auto-generated count query is wrong (e.g., with JOINs, DISTINCT, or subqueries), provide your own:

```java
@Query(
    value       = "SELECT p FROM Product p JOIN p.tags t WHERE t.name = :tag",
    countQuery  = "SELECT COUNT(DISTINCT p.id) FROM Product p JOIN p.tags t WHERE t.name = :tag"
)
Page<Product> findByTag(@Param("tag") String tag, Pageable pageable);
```

### 6.3 Native Queries with Pagination

```java
@Query(
    value       = "SELECT * FROM product WHERE category_id = :catId AND active = 1",
    countQuery  = "SELECT COUNT(*) FROM product WHERE category_id = :catId AND active = 1",
    nativeQuery = true
)
Page<Product> findNativeByCategory(@Param("catId") Long catId, Pageable pageable);
```

**Important:** With native queries, Spring **cannot** automatically inject the `ORDER BY` from `Pageable.getSort()` — you must handle sorting manually or use a fixed `ORDER BY` in the query string.

**Workaround for dynamic sorting in native queries:**

```java
// Option A: Inject sort via SpEL (Spring Data supports #{#pageable})
@Query(
    value       = "SELECT * FROM product /**#pageable**/",
    countQuery  = "SELECT COUNT(*) FROM product",
    nativeQuery = true
)
Page<Product> findAllNative(Pageable pageable);

// Option B: Use JpaSort (type-safe, works with native)
Sort sort = JpaSort.unsafe(Sort.Direction.DESC, "price");   // marks as trusted
```

### 6.4 Sorting with @Query on JPQL

When using `@Query` with JPQL, you can sort by aliased fields and joined paths:

```java
@Query("SELECT p, AVG(r.rating) AS avgRating FROM Product p LEFT JOIN p.reviews r GROUP BY p")
Page<Object[]> findProductsWithAvgRating(Pageable pageable);
// Pageable sort on 'avgRating' works with JPQL aliases
```

---

## 7. Specifications + Pagination

When you need **dynamic filters** (query parameters that may or may not be present), use `JpaSpecificationExecutor`:

```java
public interface ProductRepository
        extends JpaRepository<Product, Long>,
                JpaSpecificationExecutor<Product> { }
```

```java
// Building specifications
public class ProductSpecs {

    public static Specification<Product> hasCategory(Category category) {
        return (root, query, cb) ->
            category == null ? null : cb.equal(root.get("category"), category);
    }

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) ->
            cb.between(root.get("price"), min, max);
    }

    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }
}

// Service layer — combine specs + pageable
public Page<Product> search(ProductFilter filter, Pageable pageable) {
    Specification<Product> spec = Specification.where(isActive())
        .and(hasCategory(filter.getCategory()))
        .and(priceBetween(filter.getMinPrice(), filter.getMaxPrice()));

    return productRepository.findAll(spec, pageable);
}
```

**Available methods on `JpaSpecificationExecutor`:**
```java
Optional<T>  findOne(Specification<T> spec);
List<T>      findAll(Specification<T> spec);
Page<T>      findAll(Specification<T> spec, Pageable pageable);
List<T>      findAll(Specification<T> spec, Sort sort);
long         count(Specification<T> spec);
```

---

## 8. REST Controller Integration

### 8.1 `Pageable` from Request Parameters (Spring MVC)

Spring MVC's `PageableHandlerMethodArgumentResolver` automatically maps HTTP request params to a `Pageable` object:

```
GET /products?page=0&size=10&sort=price,desc&sort=name,asc
```

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @GetMapping
    public Page<ProductDto> list(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {

        return productService.findAll(pageable)
                             .map(ProductDto::from);
    }
}
```

**Default parameter names:**

| Param | Meaning | Default |
|---|---|---|
| `page` | Page number (0-indexed) | 0 |
| `size` | Page size | 20 |
| `sort` | `field,direction` — repeatable | none |

### 8.2 @PageableDefault

```java
@PageableDefault(
    page      = 0,
    size      = 10,
    sort      = { "lastName", "firstName" },
    direction = Sort.Direction.ASC
)
Pageable pageable
```

### 8.3 @SortDefault — Separate Sort Defaults

```java
@SortDefault.SortDefaults({
    @SortDefault(sort = "lastName",  direction = Direction.ASC),
    @SortDefault(sort = "createdAt", direction = Direction.DESC)
})
Pageable pageable
```

### 8.4 Customising Parameter Names

```java
// application.properties
spring.data.web.pageable.page-parameter=page
spring.data.web.pageable.size-parameter=size
spring.data.web.pageable.default-page-size=20
spring.data.web.pageable.max-page-size=100       // important — cap to prevent abuse
spring.data.web.pageable.one-indexed-parameters=false
```

### 8.5 PagedModel for HATEOAS

If your API includes HATEOAS links:

```java
@GetMapping
public PagedModel<EntityModel<ProductDto>> list(Pageable pageable,
        PagedResourcesAssembler<ProductDto> assembler) {

    Page<ProductDto> page = productService.findAll(pageable).map(ProductDto::from);
    return assembler.toModel(page);
}
```

Response includes `_links.next`, `_links.prev`, `_links.first`, `_links.last` with full URLs.

### 8.6 Returning a DTO Page Response (Without HATEOAS)

```java
@GetMapping
public ResponseEntity<PageResponse<ProductDto>> list(
        @PageableDefault(size = 20) Pageable pageable) {

    Page<ProductDto> page = productService.findAll(pageable).map(ProductDto::from);
    return ResponseEntity.ok(PageResponse.from(page));
}

// Custom response wrapper
public record PageResponse<T>(
    List<T>  content,
    int      page,
    int      size,
    long     totalElements,
    int      totalPages,
    boolean  last
) {
    public static <T> PageResponse<T> from(Page<T> p) {
        return new PageResponse<>(p.getContent(), p.getNumber(), p.getSize(),
                                  p.getTotalElements(), p.getTotalPages(), p.isLast());
    }
}
```

---

## 9. Cursor-Based / Keyset Pagination

### 9.1 The Problem with Offset Pagination

**Offset pagination** (`LIMIT N OFFSET M`) has two fundamental problems at scale:

1. **Performance degrades with large offsets.** `OFFSET 100000` still causes the DB to scan and discard 100,000 rows before returning your 20.

2. **Data drift.** If rows are inserted/deleted between page fetches, records shift — you get duplicates or skip records.

```sql
-- At page 100 with 20 per page:
SELECT * FROM product ORDER BY created_at DESC LIMIT 20 OFFSET 2000;
-- DB must scan and discard 2000 rows — O(offset) cost
```

### 9.2 Keyset / Cursor-Based Pagination

Instead of telling the DB "skip N rows", you tell it "give me rows AFTER this specific row" using the last seen value of the sort key.

**Concept:**
```sql
-- First page
SELECT * FROM product ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page (using last row's values as the cursor)
SELECT * FROM product
WHERE (created_at, id) < ('2024-03-15 10:00:00', 5432)   -- keyset condition
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**This uses an index seek (O(log N))** instead of a full scan.

### 9.3 Implementing Keyset Pagination in Spring Data JPA

Spring Data 3.1+ introduced `Window<T>` and `ScrollPosition` for this:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    Window<Product> findByActiveTrue(Pageable pageable, ScrollPosition position);
}
```

```java
// Service
public Window<Product> firstPage() {
    Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending()
                                                  .and(Sort.by("id").descending()));
    return productRepository.findByActiveTrue(pageable, ScrollPosition.keyset());
}

public Window<Product> nextPage(Window<Product> previous) {
    return productRepository.findByActiveTrue(
        previous.getPageable(),
        previous.positionAt(previous.size() - 1)  // position after last element
    );
}
```

### 9.4 Manual Keyset Pagination (pre-Spring Data 3.1)

```java
@Query("""
    SELECT p FROM Product p
    WHERE p.active = true
      AND (p.createdAt < :lastCreatedAt
           OR (p.createdAt = :lastCreatedAt AND p.id < :lastId))
    ORDER BY p.createdAt DESC, p.id DESC
    """)
List<Product> findNextPage(
    @Param("lastCreatedAt") Instant lastCreatedAt,
    @Param("lastId")        Long lastId,
    Pageable pageable   // only size matters here; ORDER BY is in query
);
```

The client receives the cursor as an opaque token:
```json
{
  "content": [...],
  "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI0LTAzLTE1VDEwOjAwOjAwWiIsImlkIjo1NDMyfQ=="
}
```

### 9.5 Offset vs Keyset — When to Use Each

| Factor | Offset | Keyset |
|---|---|---|
| UI paginator (jump to page N) | Yes | Not suitable |
| Infinite scroll / feeds | Suitable | **Preferred** |
| Large datasets (> 100k rows) | Degrades | Scales linearly |
| Data drift (inserts during scroll) | Possible | Immune |
| Complexity | Low | Higher |
| Multi-column sort requirement | Easy | Requires tie-breaking ID |

---

## 10. Count Query Optimisation

### 10.1 The Hidden COUNT(*) Problem

Every `Page<T>` result fires **two queries**: the data query and a `COUNT` query. On large, complex queries the count can be orders of magnitude more expensive than the data fetch.

**When this hurts:**
- Queries with many JOINs
- Queries with `DISTINCT`
- Full-text search queries
- Aggregations

### 10.2 Providing a Custom Count Query

```java
@Query(
    value      = """
        SELECT o FROM Order o
        JOIN FETCH o.customer c
        JOIN FETCH o.items i
        WHERE c.id = :customerId
        """,
    countQuery = "SELECT COUNT(o) FROM Order o WHERE o.customer.id = :customerId"
)
Page<Order> findByCustomer(@Param("customerId") Long customerId, Pageable pageable);
```

The count query avoids the expensive JOINs — it only needs to count orders, not fetch all joined data.

### 10.3 Use Slice When Total Count Isn't Needed

```java
// Before (fires COUNT every page)
Page<Notification> findByUserId(Long userId, Pageable pageable);

// After (no COUNT — only need hasNext for infinite scroll)
Slice<Notification> findByUserId(Long userId, Pageable pageable);
```

### 10.4 Approximate Counts for Large Tables

For tables with tens of millions of rows where approximate counts are acceptable:

```java
@Query(value = "SELECT reltuples::BIGINT FROM pg_class WHERE relname = 'orders'", nativeQuery = true)
Long approximateOrderCount();  // PostgreSQL statistics — O(1) lookup
```

---

## 11. Sorting Pitfalls & Security

### 11.1 Never Trust Raw Sort Strings from Clients

Passing user-provided sort field names directly into `Sort.by()` is a **SQL injection vector** — the field name ends up in an `ORDER BY` clause.

```java
// DANGEROUS — never do this
String sortField = request.getParam("sort");  // user input
Sort sort = Sort.by(sortField);               // injected into ORDER BY
```

### 11.2 Safe Sorting Pattern — Whitelist Approach

```java
public enum ProductSortField {
    PRICE("price"),
    NAME("name"),
    CREATED_AT("createdAt"),
    RATING("averageRating");

    private final String entityField;

    ProductSortField(String entityField) {
        this.entityField = entityField;
    }

    public String getEntityField() { return entityField; }

    public static ProductSortField fromString(String value) {
        return Arrays.stream(values())
            .filter(f -> f.name().equalsIgnoreCase(value))
            .findFirst()
            .orElse(CREATED_AT);  // safe default
    }
}

// Controller
@GetMapping
public Page<ProductDto> list(
        @RequestParam(defaultValue = "CREATED_AT") String sortBy,
        @RequestParam(defaultValue = "DESC")        String direction,
        @PageableDefault(size = 20)                 Pageable pageable) {

    ProductSortField field = ProductSortField.fromString(sortBy);   // validated
    Sort.Direction dir     = Sort.Direction.fromOptionalString(direction)
                                           .orElse(Sort.Direction.DESC);

    Pageable safePage = PageRequest.of(pageable.getPageNumber(), pageable.getPageSize(),
                                       Sort.by(dir, field.getEntityField()));
    return productService.findAll(safePage).map(ProductDto::from);
}
```

### 11.3 Limit Max Page Size

```java
// application.properties
spring.data.web.pageable.max-page-size=100
```

Without this, a caller can request `size=100000` and effectively do a table scan.

### 11.4 Sorting by Non-Indexed Columns

Sorting by columns without an index causes a **filesort** — the DB must sort the full result set in memory or on disk before returning the page. Always add a DB index on commonly sorted columns.

```sql
CREATE INDEX idx_product_price     ON product(price);
CREATE INDEX idx_product_createdat ON product(created_at DESC);
-- Composite index for combined filter + sort
CREATE INDEX idx_product_cat_price ON product(category_id, price DESC);
```

### 11.5 Deterministic Sort — Always Include a Tie-Breaker

Paginating without a unique tie-breaking field causes **non-deterministic ordering** — the same record may appear on multiple pages if the DB chooses a different execution plan or rows have equal sort values.

```java
// Potentially non-deterministic: two products with same price could swap pages
Sort sort = Sort.by("price").descending();

// Deterministic: price DESC then id ASC ensures strict ordering
Sort sort = Sort.by("price").descending().and(Sort.by("id").ascending());
```

---

## 12. Production Patterns & Pitfalls

### 12.1 Pitfall: `findAll()` Without Pageable

```java
// Loads entire table into memory — catastrophic on large tables
List<Product> all = productRepository.findAll();
```

Always use `findAll(pageable)` for user-facing APIs.

### 12.2 Pitfall: N+1 with Paginated Queries

Paginating entities with lazy `@OneToMany` collections causes N+1 queries:

```java
Page<Order> orders = orderRepository.findAll(pageable);
// ↑ 1 query for orders
orders.getContent().forEach(o -> o.getItems().size());
// ↑ N queries for items (one per order) = N+1
```

**Fix with `@EntityGraph`:**

```java
@EntityGraph(attributePaths = { "items", "customer" })
Page<Order> findAll(Pageable pageable);
```

**However**, using `JOIN FETCH` with `Page<T>` generates a Hibernate warning:
```
HibernateJpaDialect: "firstResult/maxResults specified with collection fetch; applying in memory!"
```

This means Hibernate fetches **all rows** into memory and paginates there — defeating the purpose.

**Correct fix — two-query approach:**

```java
@Query("""
    SELECT DISTINCT o.id FROM Order o
    JOIN o.items i
    WHERE o.customerId = :customerId
    ORDER BY o.createdAt DESC
    """)
Page<Long> findOrderIdsByCustomer(@Param("customerId") Long id, Pageable pageable);

// Then fetch full entities for just those IDs
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findByIdsWithItems(@Param("ids") List<Long> ids);
```

Or use the **`@EntityGraph` + count query** pattern:

```java
@EntityGraph(attributePaths = "items")
@Query(value       = "SELECT o FROM Order o WHERE o.customerId = :cid",
       countQuery  = "SELECT COUNT(o) FROM Order o WHERE o.customerId = :cid")
Page<Order> findByCustomerWithItems(@Param("cid") Long cid, Pageable pageable);
```

### 12.3 Pitfall: Exposing Raw `Page<Entity>` from REST API

```java
// Bad — exposes internal entity fields, lazy-loading can fail in controller
return orderRepository.findAll(pageable);

// Good — map to DTO before the Hibernate session closes
return orderRepository.findAll(pageable).map(OrderDto::from);
```

### 12.4 Pattern: Stable Pagination for Exports

When generating a large export, use `Slice` in a loop:

```java
public void exportOrders(OutputStream out) {
    Pageable page = PageRequest.of(0, 500, Sort.by("id").ascending());
    Slice<Order> slice;

    do {
        slice = orderRepository.findByStatus(OrderStatus.COMPLETED, page);
        slice.getContent().forEach(order -> writeToCsv(out, order));
        page = slice.nextPageable();
    } while (slice.hasNext());
}
```

### 12.5 Pattern: Async Pagination for Long-Running Queries

```java
@Async
public CompletableFuture<Page<ReportRow>> generateReport(Pageable pageable) {
    return CompletableFuture.completedFuture(
        reportRepository.findAll(pageable)
    );
}
```

### 12.6 Pitfall: Pageable in `@Transactional` Tests

```java
@Test
@Transactional
void testPagination() {
    // save test data in same TX as the query
    productRepository.saveAll(testProducts);
    productRepository.flush();   // ← must flush before paginated query

    Page<Product> page = productRepository.findAll(PageRequest.of(0, 5));
    assertThat(page.getTotalElements()).isEqualTo(10);
}
```

Without `flush()`, saved entities may not be visible to the count query.

---

## 13. Interview Questions

**Q: What is the difference between `Page<T>` and `Slice<T>`?**

> `Page<T>` fires two SQL queries: the data query and a `COUNT(*)` query. It gives you total elements, total pages, and navigation metadata. `Slice<T>` fires only the data query (fetching `size + 1` rows to determine `hasNext()`). Use `Page<T>` when you need to show a page count in the UI; use `Slice<T>` for infinite scroll or "load more" patterns where the total count isn't needed and you want to save the `COUNT(*)` overhead.

---

**Q: Why does Hibernate warn "firstResult/maxResults specified with collection fetch; applying in memory" and how do you fix it?**

> This happens when you use `JOIN FETCH` (or `@EntityGraph` on a collection) combined with pagination. Hibernate cannot push `LIMIT/OFFSET` to the DB when a collection join multiplies rows, so it loads all matching rows into memory and paginates there — which defeats pagination entirely. The fix is a two-query approach: first page the parent IDs with a scalar query (no join fetch), then load the full entities with their collections using `WHERE id IN (?)`.

---

**Q: How does Spring MVC auto-bind HTTP request parameters to `Pageable`?**

> Via `PageableHandlerMethodArgumentResolver`, which is registered by `@EnableSpringDataWebSupport` (auto-applied in Spring Boot). It maps `?page=0&size=20&sort=price,desc` to a `PageRequest`. Defaults can be overridden with `@PageableDefault`. The max page size should be capped in properties to prevent abuse.

---

**Q: What is the security risk of accepting sort fields directly from API callers?**

> Sort field names are injected into the SQL `ORDER BY` clause. Without validation, a caller can provide arbitrary SQL fragments (SQL injection). The fix is to whitelist allowed sort fields using an enum and map from the API-facing name to the internal entity field name before constructing the `Sort` object.

---

**Q: What is keyset (cursor-based) pagination and when should you use it over offset pagination?**

> Offset pagination uses `LIMIT N OFFSET M` which requires the DB to scan and discard M rows — performance degrades as offset grows. It also suffers data drift when rows are inserted during scrolling. Keyset pagination uses a `WHERE` clause based on the last seen value of the sort key (`WHERE created_at < ? AND id < ?`) which hits an index and runs in O(log N). Use keyset for infinite-scroll, large datasets, or feeds where jumping to an arbitrary page isn't needed.

---

**Q: How do you provide a custom count query for `Page<T>`?**

> Use the `countQuery` attribute of `@Query`. This is needed when the auto-derived count query is incorrect or expensive — for example, queries with JOINs where the count query should avoid the joins, or `DISTINCT` queries where the count must be `COUNT(DISTINCT p.id)` rather than `COUNT(p)`.

---

**Q: Why should you always include a tie-breaking field in your sort?**

> Without a unique tie-breaker, rows with equal sort values can appear in any order, and that order may change between page fetches if the DB query planner varies its execution plan. This causes records to appear on multiple pages or be skipped entirely. Always append a unique field (typically the primary key) as the final sort criterion.

---

**Q: What does `readOnly = true` do for paginated query methods in Spring Data?**

> For methods declared on the repository, Spring Data applies `@Transactional(readOnly = true)` by default on all query methods. This tells Hibernate to skip dirty-checking (no snapshot comparison for each loaded entity), set `FlushMode.MANUAL`, and optionally route the connection to a read replica. For paginated queries loading many entities, this is a significant performance optimisation.

---

**Q: What happens if you use `Pageable.unpaged()`?**

> The repository method executes without a `LIMIT/OFFSET` clause — it returns all matching records. The sort from the `Pageable` (if any) is still applied. This is useful for internal service calls or exports where you want to reuse a pageable-accepting method without actually paginating. Be careful on large tables as this loads all rows.

---

**Q: How would you paginate a report that produces 10 million rows without running out of memory?**

> Use `Slice<T>` in a loop with a fixed page size (e.g., 500–1000 rows), writing each chunk to output as it's processed. Clear the `EntityManager` (`em.clear()`) or use a `StatelessSession` after each chunk to prevent the first-level cache from accumulating entities. Use `Sort.by("id")` with a cursor approach for O(log N) page fetches at high offsets.

---

*Last updated: May 2026*
