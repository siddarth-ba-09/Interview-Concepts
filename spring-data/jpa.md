# JPA (Jakarta Persistence API) — Complete Reference

> **Learning Path Position:** Raw JDBC → Spring JDBC → Spring Data JDBC → Hibernate → **JPA** → Spring Data JPA
>
> Hibernate is the *engine*. JPA is the *contract*. Understanding JPA as a specification — its API, rules, and guarantees — is what lets you reason about code correctly regardless of which JPA provider is under the hood, and answer interview questions with precision.

---

## 1. Overview — What is JPA?

JPA (Jakarta Persistence API, formerly Java Persistence API) is a **specification** — a set of interfaces, annotations, and rules defined by a Jakarta EE standard (JSR-338). It does not contain any implementation code.

```
JPA Specification (javax.persistence / jakarta.persistence)
  └── Hibernate (most popular)      ← what Spring Boot uses by default
  └── EclipseLink (Reference Implementation)
  └── OpenJPA
  └── DataNucleus
```

**Why it exists:** Before JPA, every ORM had its own proprietary API. JPA standardized ORM in Java so that:
- Application code depends on the JPA API (interfaces), not a specific ORM
- You can theoretically switch from Hibernate to EclipseLink without changing application code (in practice, Hibernate-specific hints/features creep in)
- Java EE / Spring can integrate with any JPA-compliant provider

**Package changes:**
```
Java EE 8 / Spring Boot 2.x: javax.persistence.*
Jakarta EE 9+ / Spring Boot 3.x: jakarta.persistence.*  ← use this
```

---

## 2. JPA Core Concepts — The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JPA Architecture                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │             Persistence Unit (persistence.xml / Spring config) │ │
│  │  • Named unit with a set of entity classes                     │ │
│  │  • DataSource reference                                        │ │
│  │  • Provider class (Hibernate)                                  │ │
│  │  • Cache settings, transaction type, etc.                      │ │
│  └─────────────────────────┬──────────────────────────────────────┘ │
│                            │  Persistence.createEntityManagerFactory │
│                            ▼                                         │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              EntityManagerFactory (EMF)                        │ │
│  │  • One per persistence unit                                    │ │
│  │  • Thread-safe, application-scoped                             │ │
│  │  • Expensive to create (holds metadata, connection pool ref)   │ │
│  │  • Holds the Second-Level Cache                                │ │
│  └─────────────────────────┬──────────────────────────────────────┘ │
│                            │  emf.createEntityManager()             │
│                            ▼                                         │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              EntityManager (EM)                                │ │
│  │  • One per transaction / request                               │ │
│  │  • NOT thread-safe — never share between threads               │ │
│  │  • Wraps a Hibernate Session internally                        │ │
│  │  • Contains the Persistence Context (first-level cache)        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Persistence Context

The **Persistence Context** is the in-memory workspace that the `EntityManager` maintains — it's essentially the first-level cache. Every entity loaded or persisted through an `EntityManager` lives in its persistence context.

```
Persistence Context:
  • An identity map: Map<(EntityClass, PrimaryKey), EntityInstance>
  • Tracks ALL managed entities and their snapshots
  • Guarantees object identity: em.find(Product, 1L) == em.find(Product, 1L)
  • Participates in dirty checking (snapshot comparison on flush)
  • Closed when the EntityManager is closed (end of @Transactional method in Spring)
```

### Persistence Context Types

```java
// 1. TRANSACTION scoped (default in Spring — @PersistenceContext)
// The persistence context lives for the duration of the transaction.
// Created when @Transactional begins, closed when it ends.
// This is what you use 99% of the time.

@PersistenceContext   // Spring injects a transaction-scoped proxy
private EntityManager em;

// 2. EXTENDED scope (used in stateful EJBs / long-lived contexts)
// The persistence context survives across multiple transactions.
// Rarely used in Spring applications.
@PersistenceContext(type = PersistenceContextType.EXTENDED)
private EntityManager em;
```

### How Spring manages EntityManager

In Spring, you inject `EntityManager` using `@PersistenceContext`. Spring does NOT inject the actual `EntityManager` — it injects a **thread-safe proxy**. Each call on this proxy is delegated to the real `EntityManager` bound to the **current transaction**. This is why you can declare `EntityManager` as a field in a Spring singleton bean without thread-safety issues.

```java
@Repository
public class ProductRepository {

    @PersistenceContext
    private EntityManager em;  // This is a proxy — thread-safe
                               // The underlying EM is transaction-scoped and NOT shared

    @Transactional
    public Product save(Product product) {
        em.persist(product);   // Proxy delegates to the real EM bound to this thread's tx
        return product;
    }
}
```

---

## 4. EntityManager API — Every Method Explained

### 4.1 Persistence Operations

```java
// persist(entity)
// Transitions entity from TRANSIENT to PERSISTENT.
// INSERT will happen on flush.
// Returns void — the passed entity becomes the managed copy.
Product product = new Product("SKU-001", "Laptop", new BigDecimal("999"));
em.persist(product);
// After flush: product.getId() is populated (if IDENTITY strategy)

// merge(entity)
// Takes a DETACHED entity, copies its state onto a managed entity, returns managed copy.
// The ORIGINAL object remains detached — always use the returned reference.
Product detached = getDetachedProduct();          // e.g., from a DTO
Product managed = em.merge(detached);             // returns new PERSISTENT copy
managed.setPrice(new BigDecimal("899"));          // safe — tracked
detached.setPrice(new BigDecimal("799"));         // NOT tracked — detached is unchanged

// find(Class, id)
// Looks up by primary key. Checks first-level cache first, then second-level cache, then DB.
// Returns null if not found. Does NOT throw an exception.
Product product = em.find(Product.class, 1L);
if (product == null) throw new NotFoundException();

// getReference(Class, id)
// Returns a PROXY (hollow object) without hitting the DB.
// Use when you only need the ID for a relationship FK (avoids unnecessary SELECT).
// Throws EntityNotFoundException only when the proxy is first accessed.
Order order = new Order();
order.setCustomer(em.getReference(Customer.class, customerId));  // no DB hit
em.persist(order);   // only INSERTs order with customer_id set — no customer SELECT

// remove(entity)
// Transitions PERSISTENT entity to REMOVED. DELETE on flush.
// Entity MUST be in persistent state (use find first if you have just an ID).
Product product = em.find(Product.class, 1L);  // persistent
em.remove(product);                             // scheduled for DELETE

// Shortcut with getReference (no SELECT then DELETE):
em.remove(em.getReference(Product.class, 1L));

// refresh(entity)
// Reloads entity state from DB, discarding any in-memory changes.
// Useful when another process may have changed the DB row.
Product product = em.find(Product.class, 1L);
product.setPrice(new BigDecimal("0"));  // accidental change
em.refresh(product);                    // price restored from DB
```

### 4.2 Session / Cache Management

```java
// flush()
// Writes pending changes to DB immediately (within the current transaction).
// Does NOT commit. Used to get generated IDs, or order operations.
em.persist(order);
em.flush();  // INSERT issued, order.getId() now has a value

// clear()
// Detaches ALL entities from the persistence context.
// Resets the identity map. Use in batch operations to prevent memory bloat.
em.flush();
em.clear();  // all entities are now detached, memory freed

// detach(entity)
// Detaches a specific entity only. Changes to it are no longer tracked.
em.detach(product);  // product is now DETACHED

// contains(entity)
// Returns true if the entity is managed (persistent) in this context.
boolean isManaged = em.contains(product);

// isOpen()
boolean open = em.isOpen();
```

### 4.3 Query Operations

```java
// JPQL — entity-oriented query language
TypedQuery<Product> q = em.createQuery(
    "SELECT p FROM Product p WHERE p.category = :cat AND p.active = true",
    Product.class
);
q.setParameter("cat", Category.ELECTRONICS);
q.setMaxResults(20);
q.setFirstResult(0);
List<Product> results = q.getResultList();

// Single result (throws NoResultException if 0, NonUniqueResultException if >1)
Product p = em.createQuery("SELECT p FROM Product p WHERE p.sku = :sku", Product.class)
              .setParameter("sku", "SKU-001")
              .getSingleResult();

// Optional single result (JPA 2.2+)
Optional<Product> opt = em.createQuery("FROM Product p WHERE p.sku = :sku", Product.class)
                           .setParameter("sku", "SKU-001")
                           .getResultStream()
                           .findFirst();

// Native SQL
List<Product> products = em.createNativeQuery(
    "SELECT * FROM products WHERE category = ? AND active = true",
    Product.class
).setParameter(1, "ELECTRONICS").getResultList();

// Named query (defined on entity)
List<Product> results = em.createNamedQuery("Product.findByCategory", Product.class)
                           .setParameter("category", Category.ELECTRONICS)
                           .getResultList();

// Criteria API
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Product> cq = cb.createQuery(Product.class);
Root<Product> root = cq.from(Product.class);
cq.where(cb.equal(root.get("active"), true));
List<Product> products = em.createQuery(cq).getResultList();
```

### 4.4 Transaction Management (non-Spring)

```java
// In plain JPA (no Spring), you manage transactions manually:
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myUnit");
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();

try {
    tx.begin();
    Product product = new Product("SKU-001", "Laptop", new BigDecimal("999"));
    em.persist(product);
    tx.commit();
} catch (Exception e) {
    if (tx.isActive()) tx.rollback();
    throw e;
} finally {
    em.close();
}
// In Spring: @Transactional handles begin/commit/rollback automatically
```

---

## 5. Entity Annotations — Complete Reference

### 5.1 Class-Level Annotations

```java
@Entity                         // Required. Marks class as JPA-managed entity.
@Table(
    name = "products",          // table name (defaults to class name)
    schema = "catalog",         // DB schema
    catalog = "my_db",          // DB catalog (rarely needed)
    indexes = {
        @Index(name = "idx_sku",      columnList = "sku",            unique = true),
        @Index(name = "idx_cat_name", columnList = "category, name", unique = false)
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_sku", columnNames = {"sku"})
    }
)
@NamedQuery(
    name = "Product.findByCategory",
    query = "SELECT p FROM Product p WHERE p.category = :category AND p.active = true"
)
@NamedQueries({
    @NamedQuery(name = "Product.findActive",    query = "FROM Product WHERE active = true"),
    @NamedQuery(name = "Product.findBySku",     query = "FROM Product WHERE sku = :sku")
})
@NamedNativeQuery(
    name = "Product.findTopSellers",
    query = "SELECT * FROM products ORDER BY total_sold DESC LIMIT :limit",
    resultClass = Product.class
)
@EntityListeners(AuditListener.class)      // external lifecycle listener class
@Cacheable(true)                           // enable second-level cache
@Access(AccessType.FIELD)                  // annotations on fields, not getters (default)
// AccessType.PROPERTY = annotations on getters — Hibernate reads via getters
public class Product { ... }
```

### 5.2 Field-Level Annotations

```java
@Entity
public class Product {

    // --- Primary Key ---
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    // --- Basic column ---
    @Column(
        name = "sku",
        nullable = false,
        unique = true,
        length = 50,            // for VARCHAR (default: 255)
        precision = 10,         // for NUMERIC/DECIMAL: total digits
        scale = 2,              // for NUMERIC/DECIMAL: decimal digits
        insertable = true,      // include in INSERT (default: true)
        updatable = false,      // never include in UPDATE (for immutable columns)
        columnDefinition = "VARCHAR(50) DEFAULT 'UNKNOWN'"  // raw DDL — use sparingly
    )
    private String sku;

    // --- Enum ---
    @Enumerated(EnumType.STRING)    // stores "ELECTRONICS" — prefer over ORDINAL
    // @Enumerated(EnumType.ORDINAL) // stores 0,1,2 — dangerous: enum order changes break data
    @Column(name = "category", nullable = false)
    private Category category;

    // --- Large objects ---
    @Lob                            // maps to CLOB/BLOB in DB
    @Column(name = "description")
    private String description;     // CLOB

    @Lob
    @Column(name = "thumbnail")
    private byte[] thumbnail;       // BLOB

    // --- Temporal (legacy — prefer Instant/LocalDate with JPA 2.2+) ---
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at")
    private Date createdAt;         // old way

    // Modern way (JPA 2.2+ natively supports java.time):
    @Column(name = "created_at")
    private Instant createdAt;      // no @Temporal needed

    @Column(name = "expiry_date")
    private LocalDate expiryDate;

    // --- Transient (not persisted) ---
    @Transient
    private String computedDisplayPrice;

    // --- Version (optimistic locking) ---
    @Version
    private Long version;           // Long, Integer, Short, Timestamp supported

    // --- Embedded value object ---
    @Embedded
    private Address address;

    // --- Element collection (a collection of basic types or embeddables) ---
    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id")
    )
    @Column(name = "tag")
    private Set<String> tags = new HashSet<>();
    // Table: product_tags(product_id FK, tag VARCHAR)
    // No separate entity needed for simple value collections

    // --- Map collection ---
    @ElementCollection
    @CollectionTable(name = "product_attributes", joinColumns = @JoinColumn(name = "product_id"))
    @MapKeyColumn(name = "attr_key")
    @Column(name = "attr_value")
    private Map<String, String> attributes = new HashMap<>();
}
```

### 5.3 `@Embedded` and `@Embeddable`

```java
@Embeddable                     // marks a class as embeddable (no @Id)
public class Money {
    @Column(name = "amount", precision = 19, scale = 4, nullable = false)
    private BigDecimal amount;

    @Column(name = "currency", length = 3, nullable = false)
    private String currency;

    // No @Id, no @Table — this is a value object, not an entity

    // Good practice: override equals/hashCode for value objects
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money m = (Money) o;
        return Objects.equals(amount, m.amount) && Objects.equals(currency, m.currency);
    }
}

@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @Embedded
    private Money total;                // columns: amount, currency in orders table

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount",   column = @Column(name = "shipping_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "shipping_currency"))
    })
    private Money shippingCost;         // re-use same embeddable with different column names

    @AttributeOverrides({
        @AttributeOverride(name = "amount",   column = @Column(name = "discount_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "discount_currency"))
    })
    @Embedded
    private Money discount;
}
// Single table: orders(id, amount, currency, shipping_amount, shipping_currency,
//                       discount_amount, discount_currency)
```

---

## 6. JPA Relationships — Deep Dive

### 6.1 Relationship Annotations Reference

| Annotation | Multiplicity | Default Fetch | Owns FK? |
|------------|-------------|---------------|----------|
| `@ManyToOne` | Many → One | **EAGER** (change to LAZY!) | Yes — FK in this entity's table |
| `@OneToMany` | One → Many | LAZY | No — unless `@JoinColumn`, else join table |
| `@OneToOne` | One → One | **EAGER** (change to LAZY!) | Depends on `@JoinColumn` placement |
| `@ManyToMany` | Many → Many | LAZY | No — always a join table |

### 6.2 Unidirectional vs Bidirectional

```java
// ─── Unidirectional @ManyToOne ───────────────────────────────────────────────
// Simplest. FK in this table. Only Order knows about Customer — not vice versa.
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")  // column in orders table
    private Customer customer;
}

// ─── Bidirectional @OneToMany / @ManyToOne ───────────────────────────────────
// Both sides know about each other. mappedBy MUST be on the @OneToMany side.
// The @ManyToOne side OWNS the FK — it's the one that actually maps the column.
@Entity
public class Order {
    @Id @GeneratedValue private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")  // owner — FK column here
    private Customer customer;
}

@Entity
public class Customer {
    @Id @GeneratedValue private Long id;

    @OneToMany(
        mappedBy = "customer",         // "customer" = field name in Order
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    private List<Order> orders = new ArrayList<>();

    // ALWAYS provide helper methods for bidirectional consistency
    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);   // MUST set both sides while in-memory
    }

    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}

// ─── Unidirectional @OneToMany (with @JoinColumn, no join table) ─────────────
// Hibernate creates a @JoinColumn directly on the "many" table.
// Causes spurious UPDATE statements — Hibernate inserts the "many" row first,
// then updates it to set the FK. Less efficient than bidirectional @ManyToOne.
// Avoid unless the relationship MUST be unidirectional.
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name = "order_id")  // FK column in line_items table
    private List<LineItem> items;
    // Creates: INSERT INTO line_items(id,...) VALUES(...)  — FK is NULL
    //          UPDATE line_items SET order_id=? WHERE id=?  — spurious update
}

// ─── Unidirectional @ManyToMany (default join table) ─────────────────────────
@Entity
public class Student {
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();  // Set, not List!
}
```

### 6.3 `@JoinColumn` vs `mappedBy` — The Most Confused Topic

```
Rule:
  Exactly ONE side owns the relationship (has @JoinColumn or @JoinTable).
  The other side uses mappedBy to reference the OWNING side's field name.

  Owning side     = has @JoinColumn / @JoinTable → controls the FK column
  Non-owning side = has mappedBy → mirrors the relationship, no FK control

  For @OneToMany / @ManyToOne:
    @ManyToOne side is ALWAYS the owner (FK is in its table)
    @OneToMany side is ALWAYS the non-owner → must have mappedBy

  For @OneToOne:
    Whichever side has @JoinColumn is the owner
    The other side must have mappedBy

  Critical: If you forget mappedBy on @OneToMany, Hibernate creates a JOIN TABLE
  (a separate table) even though you only wanted a FK column.
```

---

## 7. JPA Query Language (JPQL) — Deep Dive

JPQL is SQL-like but operates on **entity names and field names**, not table/column names.

```java
// ─── SELECT ───────────────────────────────────────────────────────────────────

// Full entity
List<Product> products = em.createQuery(
    "SELECT p FROM Product p WHERE p.active = true",
    Product.class
).getResultList();

// Single field
List<String> skus = em.createQuery(
    "SELECT p.sku FROM Product p WHERE p.active = true",
    String.class
).getResultList();

// Multiple fields → Object[]
List<Object[]> rows = em.createQuery(
    "SELECT p.id, p.name, p.price FROM Product p"
).getResultList();
for (Object[] row : rows) {
    Long id = (Long) row[0];
    String name = (String) row[1];
    BigDecimal price = (BigDecimal) row[2];
}

// Constructor expression (DTO — no entity overhead)
List<ProductDto> dtos = em.createQuery(
    "SELECT new com.app.dto.ProductDto(p.id, p.name, p.price) " +
    "FROM Product p WHERE p.active = true",
    ProductDto.class
).getResultList();
// ProductDto must have a matching constructor

// DISTINCT
List<Customer> customers = em.createQuery(
    "SELECT DISTINCT c FROM Order o JOIN o.customer c WHERE o.status = 'PLACED'",
    Customer.class
).getResultList();

// ─── WHERE ────────────────────────────────────────────────────────────────────

// Named parameter (preferred)
.setParameter("status", OrderStatus.PLACED)
"WHERE o.status = :status"

// Positional parameter (legacy, fragile — avoid)
"WHERE o.status = ?1"
.setParameter(1, OrderStatus.PLACED)

// IN clause
List<Long> ids = List.of(1L, 2L, 3L);
em.createQuery("FROM Product p WHERE p.id IN :ids", Product.class)
  .setParameter("ids", ids)
  .getResultList();

// LIKE
"WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))"

// IS NULL / IS NOT NULL
"WHERE o.completedAt IS NULL"

// BETWEEN
"WHERE o.createdAt BETWEEN :from AND :to"

// SIZE (collection size)
"WHERE SIZE(o.items) > 0"

// MEMBER OF (element in collection)
"WHERE :product MEMBER OF o.items"

// EXISTS subquery
"WHERE EXISTS (SELECT r FROM Review r WHERE r.product = p AND r.rating >= 4)"

// ─── JOINs ────────────────────────────────────────────────────────────────────

// INNER JOIN (entities without the association are excluded)
"SELECT o FROM Order o JOIN o.items i WHERE i.product.id = :productId"

// LEFT JOIN (include orders even with no items)
"SELECT o FROM Order o LEFT JOIN o.items i GROUP BY o"

// JOIN FETCH (solves N+1 — loads association in same query)
"SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status"

// Multiple JOIN FETCH — be careful with collections
"SELECT DISTINCT o FROM Order o " +
"JOIN FETCH o.customer " +
"JOIN FETCH o.items i " +         // can only JOIN FETCH one collection at a time!
"JOIN FETCH i.product"
// DISTINCT needed to deduplicate Order objects from the Cartesian product

// ─── Aggregates & GROUP BY ────────────────────────────────────────────────────

"SELECT o.status, COUNT(o), SUM(o.total) FROM Order o GROUP BY o.status"
"SELECT o.status, COUNT(o) FROM Order o GROUP BY o.status HAVING COUNT(o) > 10"
"SELECT MAX(p.price), MIN(p.price), AVG(p.price) FROM Product p WHERE p.active = true"

// ─── ORDER BY ─────────────────────────────────────────────────────────────────

"FROM Product p ORDER BY p.price DESC, p.name ASC"
"FROM Product p ORDER BY p.category ASC NULLS LAST"

// ─── UPDATE / DELETE (bulk operations) ───────────────────────────────────────

// Bypass dirty checking — directly UPDATE/DELETE in DB
// WARNING: First-level cache is NOT updated — call em.clear() after
int count = em.createQuery(
    "UPDATE Product p SET p.active = false WHERE p.stockCount = 0"
).executeUpdate();
em.clear();  // synchronize first-level cache

int deleted = em.createQuery(
    "DELETE FROM AuditLog a WHERE a.createdAt < :cutoff"
).setParameter("cutoff", cutoff).executeUpdate();
em.clear();

// ─── Pagination ───────────────────────────────────────────────────────────────

TypedQuery<Product> q = em.createQuery("FROM Product p ORDER BY p.id", Product.class);
q.setFirstResult(page * pageSize);   // OFFSET
q.setMaxResults(pageSize);           // LIMIT
List<Product> page = q.getResultList();

// Count query for total pages
Long total = em.createQuery(
    "SELECT COUNT(p) FROM Product p WHERE p.active = true",
    Long.class
).getSingleResult();

// ─── Named Queries ────────────────────────────────────────────────────────────

// Defined on entity class (compiled and validated at startup by Hibernate)
@NamedQuery(
    name = "Product.findByCategory",
    query = "SELECT p FROM Product p WHERE p.category = :category AND p.active = true " +
            "ORDER BY p.name"
)
@Entity
public class Product { ... }

// Usage
List<Product> products = em.createNamedQuery("Product.findByCategory", Product.class)
                            .setParameter("category", Category.ELECTRONICS)
                            .getResultList();
// Named queries are pre-parsed at startup — fail fast + slightly faster execution
```

---

## 8. JPA Criteria API — Complete Guide

Use Criteria API when query conditions are **dynamic** (you don't know them at compile time).

```java
@Repository
@RequiredArgsConstructor
public class ProductCriteriaDao {

    @PersistenceContext
    private EntityManager em;

    // ─── Basic criteria query ─────────────────────────────────────────────────
    public List<Product> findActive() {
        CriteriaBuilder cb = em.getCriteriaBuilder();  // factory for predicates
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);   // FROM Product p

        cq.select(root)                                // SELECT p
          .where(cb.isTrue(root.get("active")));       // WHERE p.active = true

        return em.createQuery(cq).getResultList();
    }

    // ─── Dynamic search ───────────────────────────────────────────────────────
    public Page<Product> search(ProductFilter filter) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);

        List<Predicate> predicates = buildPredicates(cb, root, filter);

        cq.where(predicates.toArray(new Predicate[0]));

        // Sorting
        cq.orderBy(filter.isSortAsc()
            ? cb.asc(root.get(filter.getSortField()))
            : cb.desc(root.get(filter.getSortField())));

        // Data query
        List<Product> content = em.createQuery(cq)
            .setFirstResult(filter.getPage() * filter.getPageSize())
            .setMaxResults(filter.getPageSize())
            .getResultList();

        // Count query (reuse predicates)
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<Product> countRoot = countQuery.from(Product.class);
        List<Predicate> countPredicates = buildPredicates(cb, countRoot, filter);
        countQuery.select(cb.count(countRoot)).where(countPredicates.toArray(new Predicate[0]));
        long total = em.createQuery(countQuery).getSingleResult();

        return new PageImpl<>(content, PageRequest.of(filter.getPage(), filter.getPageSize()), total);
    }

    private List<Predicate> buildPredicates(CriteriaBuilder cb, Root<Product> root, ProductFilter filter) {
        List<Predicate> predicates = new ArrayList<>();

        if (filter.getCategory() != null) {
            predicates.add(cb.equal(root.get("category"), filter.getCategory()));
        }
        if (filter.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), filter.getMinPrice()));
        }
        if (filter.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), filter.getMaxPrice()));
        }
        if (StringUtils.hasText(filter.getKeyword())) {
            predicates.add(cb.like(
                cb.lower(root.get("name")),
                "%" + filter.getKeyword().toLowerCase() + "%"
            ));
        }
        if (filter.getInStock() != null && filter.getInStock()) {
            predicates.add(cb.greaterThan(root.get("stockCount"), 0));
        }
        predicates.add(cb.isTrue(root.get("active")));

        return predicates;
    }

    // ─── Join in criteria ─────────────────────────────────────────────────────
    public List<Order> findOrdersByCustomerCity(String city) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> orderRoot = cq.from(Order.class);

        Join<Order, Customer> customerJoin = orderRoot.join("customer", JoinType.INNER);
        Join<Customer, Address> addressJoin = customerJoin.join("address", JoinType.LEFT);

        cq.where(cb.equal(addressJoin.get("city"), city));

        return em.createQuery(cq).getResultList();
    }

    // ─── Aggregate with criteria ──────────────────────────────────────────────
    public List<CategoryRevenueDto> revenueByCategory() {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<CategoryRevenueDto> cq = cb.createQuery(CategoryRevenueDto.class);
        Root<Order> root = cq.from(Order.class);
        Join<Order, LineItem> itemJoin = root.join("items");
        Join<LineItem, Product> productJoin = itemJoin.join("product");

        cq.multiselect(
            productJoin.get("category"),
            cb.sum(cb.prod(itemJoin.get("unitPrice"),
                           itemJoin.get("quantity").as(BigDecimal.class)))
        ).groupBy(productJoin.get("category"))
         .orderBy(cb.desc(cb.sum(cb.prod(
             itemJoin.get("unitPrice"),
             itemJoin.get("quantity").as(BigDecimal.class)
         ))));

        return em.createQuery(cq).getResultList();
    }
}
```

---

## 9. Entity Graphs — Fine-Grained Fetch Control

Entity Graphs let you declare what to fetch **per query**, without changing entity annotation defaults. They are the cleanest solution to N+1 when using Spring Data JPA.

```java
// ─── Define on entity (NamedEntityGraph) ──────────────────────────────────────
@Entity
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),                         // fetch customer
        @NamedAttributeNode(value = "items", subgraph = "itemSubgraph")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "itemSubgraph",
            attributeNodes = { @NamedAttributeNode("product") } // fetch items.product
        )
    }
)
public class Order { ... }

// ─── Usage with EntityManager ──────────────────────────────────────────────────
EntityGraph<?> graph = em.getEntityGraph("Order.withCustomerAndItems");
Map<String, Object> hints = Map.of("jakarta.persistence.fetchgraph", graph);
Order order = em.find(Order.class, orderId, hints);
// fetchgraph = ONLY the specified attributes are fetched (LAZY for everything else)
// loadgraph  = specified attributes are fetched + any EAGER defaults

// ─── FETCH vs LOAD graph type ──────────────────────────────────────────────────
// fetchgraph: only listed attributes are eagerly fetched; all others use LAZY
// loadgraph:  listed attributes are eagerly fetched; others use their default fetch type

// ─── Usage in Spring Data JPA ──────────────────────────────────────────────────
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Using named entity graph
    @EntityGraph("Order.withCustomerAndItems")
    Optional<Order> findById(Long id);

    // Inline entity graph (no annotation on entity needed)
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    List<Order> findByStatus(OrderStatus status);

    // Read-only + entity graph for reports
    @Transactional(readOnly = true)
    @EntityGraph(attributePaths = {"customer"})
    @Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :from AND :to")
    List<Order> findBetween(@Param("from") Instant from, @Param("to") Instant to);
}
```

---

## 10. JPA Callbacks & Entity Listeners

```java
// ─── Inline callbacks ─────────────────────────────────────────────────────────
@Entity
public class Product {

    @PrePersist
    void onPrePersist() {
        // Called before INSERT
        // Good for: default values, validation, audit fields
        if (this.active == null) this.active = true;
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    @PostPersist
    void onPostPersist() {
        // Called after INSERT (ID is now populated)
        // Good for: publishing domain events, audit logging
    }

    @PreUpdate
    void onPreUpdate() {
        // Called before UPDATE
        this.updatedAt = Instant.now();
    }

    @PostUpdate
    void onPostUpdate() {
        // Called after UPDATE
    }

    @PreRemove
    void onPreRemove() {
        // Called before DELETE — entity is still persistent
        // Good for: audit trail, soft-delete logic
    }

    @PostRemove
    void onPostRemove() {
        // Called after DELETE
    }

    @PostLoad
    void onPostLoad() {
        // Called after SELECT (entity fully loaded)
        // Good for: decrypting sensitive fields, computing derived properties
        this.displayPrice = String.format("%s %s", currency, price.toPlainString());
    }
}

// ─── External Entity Listener (keeps entity clean) ───────────────────────────
@EntityListeners(ProductAuditListener.class)
@Entity
public class Product { ... }

@Component  // Spring component — can inject beans if using Spring-aware listener
public class ProductAuditListener {

    // Note: Spring DI in listeners requires special setup via
    // @EnableJpaAuditing + Spring Data auditing, or a static ApplicationContext accessor

    @PrePersist
    public void prePersist(Product product) {
        product.setCreatedBy(SecurityContextHolder.getContext()
            .getAuthentication().getName());
    }

    @PreUpdate
    public void preUpdate(Product product) {
        product.setUpdatedBy(SecurityContextHolder.getContext()
            .getAuthentication().getName());
    }
}
```

### Spring Data JPA Auditing (easiest approach)

```java
// 1. Enable
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
@SpringBootApplication
public class App { }

// 2. Implement AuditorAware
@Bean("auditorProvider")
public AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                         .map(Authentication::getName);
}

// 3. Use on entity
@Entity
@EntityListeners(AuditingEntityListener.class)  // Spring Data's built-in listener
public class Product {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}
```

---

## 11. JPA Transactions

### `@Transactional` — Every Attribute Explained

```java
@Transactional(
    propagation  = Propagation.REQUIRED,     // join existing tx or create new
    isolation    = Isolation.READ_COMMITTED,  // DB isolation level
    readOnly     = false,                     // true for read operations
    timeout      = 30,                        // seconds before timeout (rolls back)
    rollbackFor  = { Exception.class },       // rollback on these (default: RuntimeException)
    noRollbackFor= { BusinessException.class} // don't rollback on these
)
public void processOrder(long orderId) { }
```

### Transaction Propagation Types

```java
// REQUIRED (default):
//   Join existing transaction if one exists.
//   Create a new one if none exists.
//   Most common. Inner method participates in outer transaction.
@Transactional(propagation = Propagation.REQUIRED)
public void innerMethod() { }

// REQUIRES_NEW:
//   ALWAYS create a new transaction.
//   Suspend the outer transaction if one exists.
//   Use for operations that must commit/rollback independently.
//   Real use case: audit logging — must persist even if the main tx rolls back.
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(AuditEntry entry) {
    auditRepo.save(entry);   // commits independently of outer tx
}

// NESTED:
//   Create a nested transaction within the outer one (uses savepoints).
//   On rollback: only the nested part rolls back, outer continues.
//   Only works with JDBC DataSourceTransactionManager (not JTA).
@Transactional(propagation = Propagation.NESTED)
public void riskyOperation() { }

// SUPPORTS:
//   If a transaction exists, join it. If not, execute non-transactionally.
@Transactional(propagation = Propagation.SUPPORTS)
public List<Product> findAll() { }

// NOT_SUPPORTED:
//   Suspend any existing transaction, execute without one.
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void longRunningReport() { }

// NEVER:
//   Throw exception if a transaction exists.
@Transactional(propagation = Propagation.NEVER)
public void noTxAllowed() { }

// MANDATORY:
//   Must have an existing transaction. Throw exception if none.
@Transactional(propagation = Propagation.MANDATORY)
public void mustBeCalledInTx() { }
```

### Isolation Levels

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)  // dirty reads allowed — almost never use
@Transactional(isolation = Isolation.READ_COMMITTED)    // default for PostgreSQL — prevents dirty reads
@Transactional(isolation = Isolation.REPEATABLE_READ)   // default for MySQL — prevents non-repeatable reads
@Transactional(isolation = Isolation.SERIALIZABLE)      // highest — prevents all anomalies, lowest concurrency
@Transactional(isolation = Isolation.DEFAULT)           // use DB default (most common)
```

### Self-Invocation Problem — The Most Common `@Transactional` Bug

```java
@Service
public class OrderService {

    // ❌ BROKEN — calling processAll from within the same bean
    public void processAll() {
        for (Long id : pendingIds) {
            processOrder(id);   // calls through 'this' — bypasses Spring proxy!
            // @Transactional on processOrder is IGNORED
        }
    }

    @Transactional
    public void processOrder(long id) {
        // This @Transactional works when called from OUTSIDE the bean
        // But NOT when called via this.processOrder() — no proxy
    }

    // ✅ Fix 1: Split into two beans
    // ✅ Fix 2: Self-inject the proxy
    @Autowired
    private OrderService self;

    public void processAll() {
        for (Long id : pendingIds) {
            self.processOrder(id);   // goes through proxy — @Transactional works
        }
    }

    // ✅ Fix 3: Use ApplicationContext
    // ✅ Fix 4: Use @Async (different proxy mechanism)
    // ✅ Fix 5: Use TransactionTemplate programmatically
}
```

### Programmatic Transactions

```java
@Service
@RequiredArgsConstructor
public class MigrationService {

    private final TransactionTemplate transactionTemplate;
    private final TransactionTemplate requiresNewTemplate;

    @Autowired
    public MigrationService(PlatformTransactionManager txManager) {
        this.transactionTemplate = new TransactionTemplate(txManager);
        this.requiresNewTemplate = new TransactionTemplate(txManager);
        this.requiresNewTemplate.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );
    }

    public void migrate(List<Record> records) {
        for (Record record : records) {
            // Each record in its own transaction — failure in one doesn't affect others
            requiresNewTemplate.execute(status -> {
                try {
                    processRecord(record);
                    return null;
                } catch (Exception e) {
                    status.setRollbackOnly();  // mark for rollback
                    log.error("Failed to process record {}: {}", record.getId(), e.getMessage());
                    return null;
                }
            });
        }
    }
}
```

---

## 12. JPA Locking

### Optimistic Locking (`@Version`)

```java
@Entity
public class Seat {
    @Id private Long id;

    @Version
    private Long version;     // Hibernate auto-increments on every UPDATE

    @Enumerated(EnumType.STRING)
    private SeatStatus status;  // AVAILABLE, RESERVED, BOOKED
}

@Transactional
public boolean reserveSeat(Long seatId, Long userId) {
    Seat seat = em.find(Seat.class, seatId);
    // SQL: SELECT id, status, version FROM seats WHERE id=?  → version=3

    if (seat.getStatus() != SeatStatus.AVAILABLE) return false;
    seat.setStatus(SeatStatus.RESERVED);

    // On flush: UPDATE seats SET status='RESERVED', version=4 WHERE id=? AND version=3
    // If another thread committed first (version is now 4): 0 rows updated
    // → Hibernate throws OptimisticLockException

    return true;
}

// Handle in caller:
try {
    boolean reserved = seatService.reserveSeat(seatId, userId);
} catch (OptimisticLockException | StaleObjectStateException e) {
    // Retry, or return "seat no longer available" to the user
}
```

### Explicit Lock Mode on find/query

```java
// Upgrade to write lock after finding
Seat seat = em.find(Seat.class, seatId, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
// Increments version even if entity was not modified — use when you want to signal "I touched this"

// Pessimistic locks
Product p = em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE);
// SQL: SELECT ... FOR UPDATE

Product p = em.find(Product.class, id, LockModeType.PESSIMISTIC_READ);
// SQL: SELECT ... FOR SHARE

Product p = em.find(Product.class, id, LockModeType.PESSIMISTIC_FORCE_INCREMENT);
// SQL: SELECT ... FOR UPDATE  + increments @Version

// Timeout hint
Map<String, Object> hints = Map.of("jakarta.persistence.lock.timeout", 3000L);
Product p = em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE, hints);
// Throws LockTimeoutException if lock not acquired within 3 seconds

// Lock on query
em.createQuery("FROM Seat s WHERE s.status = 'AVAILABLE'", Seat.class)
  .setLockMode(LockModeType.PESSIMISTIC_WRITE)
  .setHint("jakarta.persistence.lock.timeout", 5000L)
  .getResultList();
```

### LockModeType Reference

| LockMode | Type | SQL | Use Case |
|----------|------|-----|----------|
| `NONE` | — | No lock | Default |
| `OPTIMISTIC` | Optimistic | Read version at end, fail if changed | Validate no one changed between read and verify |
| `OPTIMISTIC_FORCE_INCREMENT` | Optimistic | Increment version even without modification | Signal ownership of entity |
| `PESSIMISTIC_READ` | Pessimistic | `SELECT ... FOR SHARE` | Read-heavy, blocks writes |
| `PESSIMISTIC_WRITE` | Pessimistic | `SELECT ... FOR UPDATE` | Exclusive access before update |
| `PESSIMISTIC_FORCE_INCREMENT` | Pessimistic | `FOR UPDATE` + version++ | Combine pessimistic + version tracking |

---

## 13. Second-Level Cache (JPA Level)

```java
// JPA defines the API; provider (Hibernate + Ehcache/Caffeine/Redis) implements it.

// application.properties / application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory

// Entity-level caching
@Entity
@Cacheable          // JPA annotation
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Hibernate annotation
public class Category {
    @Id private Long id;
    private String name;
}

// Cache interaction
SharedCacheMode mode = emf.getCache().unwrap(Cache.class).contains(Product.class, 1L)
    ? SharedCacheMode.ALL : SharedCacheMode.NONE;

// Evict from second-level cache
emf.getCache().evict(Product.class, productId);   // evict specific entity
emf.getCache().evictAll();                         // clear entire L2 cache

// Query result cache
TypedQuery<Product> q = em.createQuery("FROM Product p WHERE p.active = true", Product.class);
q.setHint("org.hibernate.cacheable", true);
q.setHint("org.hibernate.cacheRegion", "product.active");
// Query cache stores result IDs; for each ID, hits L2 cache or DB
// Best for queries that run very frequently with the same params and rarely change
```

---

## 14. Complete Real-World Example — E-Commerce Platform

```java
// ─── Domain model ─────────────────────────────────────────────────────────────

@Entity
@Table(name = "products")
@EntityListeners(AuditingEntityListener.class)
@NamedEntityGraph(
    name = "Product.withCategoryAndImages",
    attributeNodes = {
        @NamedAttributeNode("category"),
        @NamedAttributeNode("images")
    }
)
@NamedQuery(
    name = "Product.findActiveByCategoryOrderedByPrice",
    query = "SELECT p FROM Product p WHERE p.category.id = :catId " +
            "AND p.active = true ORDER BY p.price ASC"
)
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "products_id_seq", allocationSize = 50)
    private Long id;

    @Column(name = "sku", nullable = false, unique = true, length = 50, updatable = false)
    private String sku;

    @Column(name = "name", nullable = false)
    private String name;

    @Lob
    @Column(name = "description")
    private String description;

    @Embedded
    private Money price;

    @Column(name = "stock_count", nullable = false)
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
    @Column(name = "status", nullable = false)
    private ProductStatus status = ProductStatus.DRAFT;

    @Version
    private Long version;

    @CreatedDate @Column(updatable = false) private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy  @Column(updatable = false) private String createdBy;
    @LastModifiedBy private String updatedBy;

    @Column(name = "active", nullable = false)
    private boolean active = false;

    public void addImage(ProductImage image) { images.add(image); image.setProduct(this); }
    public void removeImage(ProductImage image) { images.remove(image); image.setProduct(null); }

    public void publish() {
        if (images.isEmpty()) throw new BusinessException("Cannot publish product without images");
        this.active = true;
        this.status = ProductStatus.ACTIVE;
    }
}

// ─── Repository ───────────────────────────────────────────────────────────────

@Repository
@RequiredArgsConstructor
public class ProductJpaRepository {

    @PersistenceContext
    private EntityManager em;

    public Product save(Product product) {
        if (product.getId() == null) {
            em.persist(product);
            return product;
        }
        return em.merge(product);
    }

    public Optional<Product> findById(Long id) {
        return Optional.ofNullable(em.find(Product.class, id));
    }

    public Optional<Product> findByIdWithDetails(Long id) {
        EntityGraph<?> graph = em.getEntityGraph("Product.withCategoryAndImages");
        Map<String, Object> hints = Map.of("jakarta.persistence.fetchgraph", graph);
        return Optional.ofNullable(em.find(Product.class, id, hints));
    }

    public Optional<Product> findBySku(String sku) {
        try {
            return Optional.of(
                em.createQuery("FROM Product p WHERE p.sku = :sku", Product.class)
                  .setParameter("sku", sku)
                  .getSingleResult()
            );
        } catch (NoResultException e) {
            return Optional.empty();
        }
    }

    public List<Product> findByCategoryForCatalog(Long categoryId, int page, int size) {
        return em.createNamedQuery("Product.findActiveByCategoryOrderedByPrice", Product.class)
            .setParameter("catId", categoryId)
            .setFirstResult(page * size)
            .setMaxResults(size)
            .getResultList();
    }

    public Page<ProductSearchDto> search(ProductFilter filter, Pageable pageable) {
        // Criteria API for dynamic search — returns DTO, not entities
        CriteriaBuilder cb = em.getCriteriaBuilder();

        // Data query
        CriteriaQuery<ProductSearchDto> cq = cb.createQuery(ProductSearchDto.class);
        Root<Product> root = cq.from(Product.class);
        Join<Product, Category> catJoin = root.join("category", JoinType.LEFT);

        List<Predicate> predicates = buildSearchPredicates(cb, root, catJoin, filter);

        cq.multiselect(
            root.get("id"), root.get("sku"), root.get("name"),
            root.get("price"), root.get("stockCount"), catJoin.get("name")
        ).where(predicates.toArray(new Predicate[0]))
         .orderBy(cb.desc(root.get("createdAt")));

        List<ProductSearchDto> content = em.createQuery(cq)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize())
            .getResultList();

        // Count query
        CriteriaQuery<Long> countCq = cb.createQuery(Long.class);
        Root<Product> countRoot = countCq.from(Product.class);
        Join<Product, Category> countCatJoin = countRoot.join("category", JoinType.LEFT);
        List<Predicate> countPredicates = buildSearchPredicates(cb, countRoot, countCatJoin, filter);
        countCq.select(cb.count(countRoot)).where(countPredicates.toArray(new Predicate[0]));
        long total = em.createQuery(countCq).getSingleResult();

        return new PageImpl<>(content, pageable, total);
    }

    @Transactional
    public int deductStock(Long productId, int quantity) {
        return em.createQuery(
            "UPDATE Product p SET p.stockCount = p.stockCount - :qty " +
            "WHERE p.id = :id AND p.stockCount >= :qty"
        ).setParameter("qty", quantity)
         .setParameter("id", productId)
         .executeUpdate();
        // Returns 0 if insufficient stock (stockCount < qty) → idiomatic out-of-stock check
    }

    private List<Predicate> buildSearchPredicates(
            CriteriaBuilder cb, Root<Product> root,
            Join<Product, Category> catJoin, ProductFilter filter) {
        List<Predicate> predicates = new ArrayList<>();

        predicates.add(cb.isTrue(root.get("active")));

        if (filter.getCategoryId() != null)
            predicates.add(cb.equal(catJoin.get("id"), filter.getCategoryId()));

        if (StringUtils.hasText(filter.getKeyword()))
            predicates.add(cb.like(cb.lower(root.get("name")),
                "%" + filter.getKeyword().toLowerCase() + "%"));

        if (filter.getMinPrice() != null)
            predicates.add(cb.greaterThanOrEqualTo(
                root.get("price").get("amount"), filter.getMinPrice()));

        if (filter.getMaxPrice() != null)
            predicates.add(cb.lessThanOrEqualTo(
                root.get("price").get("amount"), filter.getMaxPrice()));

        return predicates;
    }
}

// ─── Service ──────────────────────────────────────────────────────────────────

@Service
@RequiredArgsConstructor
@Transactional
public class ProductService {

    private final ProductJpaRepository productRepo;
    private final EventPublisher eventPublisher;

    @Transactional(readOnly = true)
    public ProductDetailDto getProduct(Long id) {
        // Always return DTO from service — never entities
        Product product = productRepo.findByIdWithDetails(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        return ProductDetailDto.from(product);
    }

    @Transactional(readOnly = true)
    public Page<ProductSearchDto> search(ProductFilter filter, Pageable pageable) {
        return productRepo.search(filter, pageable);
    }

    public ProductDto create(CreateProductCommand cmd) {
        if (productRepo.findBySku(cmd.getSku()).isPresent()) {
            throw new DuplicateSkuException(cmd.getSku());
        }

        Product product = new Product(cmd.getSku(), cmd.getName(), cmd.getPrice());
        productRepo.save(product);  // id populated after this

        eventPublisher.publish(new ProductCreatedEvent(product.getId(), product.getSku()));

        return ProductDto.from(product);
    }

    public void updateStock(Long productId, int delta) {
        // Use pessimistic lock for inventory — high contention
        Product product = productRepo.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        // Already in @Transactional — dirty checking will flush the update
        product.setStockCount(product.getStockCount() + delta);
    }
}
```

---

## 15. Scenario Walkthroughs

### Scenario 1: Multi-Tenant Product Import (Bulk + Audit)

**System:** SaaS B2B platform. Each tenant uploads a CSV of 20,000 products via REST API.

**Problem:** Loading all 20,000 as entities causes OOM. Each product must be audited. Failures in one product must not affect others.

```java
@Service
@RequiredArgsConstructor
public class ProductImportService {

    @PersistenceContext
    private EntityManager em;

    private final AuditService auditService;

    // Outer method: not transactional — manages its own tx per batch
    public ImportResult importProducts(Long tenantId, List<ProductImportRow> rows) {
        int success = 0, failed = 0;
        List<String> errors = new ArrayList<>();

        for (int i = 0; i < rows.size(); i++) {
            try {
                importRow(tenantId, rows.get(i));  // each row in its own tx
                success++;
            } catch (Exception e) {
                failed++;
                errors.add("Row " + i + ": " + e.getMessage());
            }
        }

        return new ImportResult(success, failed, errors);
    }

    // Each row is independent — REQUIRES_NEW so failures don't cascade
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void importRow(Long tenantId, ProductImportRow row) {
        // Validate
        if (!row.isValid()) throw new InvalidRowException(row.getErrors());

        // Upsert pattern using merge-by-business-key
        Optional<Product> existing = findBySku(tenantId, row.getSku());
        Product product = existing.orElseGet(() -> new Product(tenantId, row.getSku()));

        product.updateFrom(row);  // apply changes
        em.merge(product);        // INSERT if new, UPDATE if existing

        // Audit in separate tx (must commit even if outer fails)
        auditService.recordImport(tenantId, row.getSku(), existing.isEmpty() ? "CREATE" : "UPDATE");
    }

    // REQUIRES_NEW: audit logs must be committed regardless of outer tx outcome
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordImport(Long tenantId, String sku, String action) {
        em.persist(new ImportAuditLog(tenantId, sku, action, Instant.now()));
    }
}
```

**Interviewer's intent:** Tests understanding of `REQUIRES_NEW` propagation, independent transactions per record, and why you'd use it for audit trails.

---

### Scenario 2: Reporting Query — Avoiding N+1 and Cartesian Product

**System:** Admin dashboard showing "orders with customer info and item count for today".

```java
// ❌ Wrong: N+1 + potential Cartesian product
List<Order> orders = em.createQuery(
    "FROM Order o WHERE o.createdAt >= :today", Order.class
).setParameter("today", today).getResultList();
// 1 query for orders

orders.forEach(o -> {
    String customerName = o.getCustomer().getName();  // N queries (LazyInit if OSIV off)
    int itemCount = o.getItems().size();              // N more queries
});
// Total: 1 + N + N = 2N+1 queries

// ✅ Correct: DTO projection — single query
@Query(value = """
    SELECT new com.app.dto.OrderDashboardDto(
        o.id,
        o.total,
        o.status,
        o.createdAt,
        c.name,
        c.email,
        COUNT(i)
    )
    FROM Order o
    JOIN o.customer c
    LEFT JOIN o.items i
    WHERE o.createdAt >= :today
    GROUP BY o.id, o.total, o.status, o.createdAt, c.name, c.email
    ORDER BY o.createdAt DESC
    """)
List<OrderDashboardDto> findTodayOrdersForDashboard(@Param("today") Instant today);
// 1 query, no entities, no lazy loading, no N+1
```

**Interviewer's intent:** Tests knowing when to use DTO projections vs entity graphs, and how GROUP BY solves the Cartesian product from joining a collection.

---

### Scenario 3: `@Transactional` Rollback Rules

**System:** Payment service. A payment is processed, then a notification is sent. If the notification fails, should the payment roll back?

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRepository paymentRepo;
    private final NotificationService notificationService;

    // ❌ Problem: EmailException (checked) does NOT roll back by default
    @Transactional
    public void processPayment(PaymentRequest req) throws EmailException {
        Payment payment = new Payment(req);
        paymentRepo.save(payment);

        notificationService.sendConfirmation(payment);  // throws EmailException (checked)
        // @Transactional only auto-rolls back for RuntimeException and Error
        // EmailException is checked → payment is COMMITTED even though email failed
    }

    // ✅ Fix 1: rollbackFor covers checked exceptions
    @Transactional(rollbackFor = Exception.class)
    public void processPayment(PaymentRequest req) throws Exception { ... }

    // ✅ Fix 2: Don't couple notification to payment transaction
    // Payment commits, notification failure is handled separately (retry queue)
    @Transactional
    public Payment processPayment(PaymentRequest req) {
        Payment payment = new Payment(req);
        paymentRepo.save(payment);
        return payment;
    }

    public void processPaymentAndNotify(PaymentRequest req) {
        Payment payment = processPayment(req);   // separate tx — commits independently
        try {
            notificationService.sendConfirmation(payment);
        } catch (EmailException e) {
            // Log failure, push to retry queue — don't fail the whole operation
            retryQueue.enqueue(payment.getId(), NotificationType.PAYMENT_CONFIRMATION);
        }
    }
}
```

**Interviewer's intent:** Tests understanding of Spring's rollback rules (`RuntimeException` only by default), and architectural knowledge about decoupling side effects from core transactions.

---

## 16. Common Interview Questions

**Q1: What is a persistence context? How long does it live in a Spring application?**

The persistence context is the first-level cache and the unit of work tracked by an `EntityManager`. It holds all managed entities and their snapshots. In Spring, by default, a persistence context lives for the duration of a single `@Transactional` method. It's created when `@Transactional` begins (Spring opens the EM or reuses an existing one) and closed when the method returns (which triggers a final flush, then commits or rolls back). After the method exits, all entities that were managed become DETACHED. In EXTENDED scope (used in stateful beans), the context can survive across multiple transactions, but this is rare in Spring applications.

---

**Q2: What is the difference between `em.persist()` and `em.merge()`? When would you use each?**

`persist()` takes a TRANSIENT entity (no ID, not known to JPA) and makes it PERSISTENT. The entity itself becomes the managed copy — you keep using the same reference. `merge()` takes a DETACHED entity (has an ID, was previously managed or loaded from elsewhere), copies its state onto a PERSISTENT copy, and returns that copy. The original entity remains DETACHED. Use `persist()` for new entities. Use `merge()` when re-attaching a detached entity (e.g., a DTO was mapped to an entity, or an entity passed across a transaction boundary). A common mistake is to call `merge()` on a new entity — this works (Hibernate treats it as persist if no ID), but `persist()` is the correct semantic.

---

**Q3: What is the self-invocation problem with `@Transactional`? How do you fix it?**

Spring implements `@Transactional` via a proxy — when you call a method on a Spring bean from outside, the call goes through the proxy, which starts/joins the transaction. When you call `this.method()` from within the same bean, you're calling directly on the object — the proxy is bypassed, so `@Transactional` has no effect. Fix: split the method into a different bean; or self-inject the bean (`@Autowired private MyService self`) and call `self.method()` to go through the proxy; or use `TransactionTemplate` programmatically.

---

**Q4: What is the difference between `FetchType.EAGER` and `FetchType.LAZY`? What are the defaults?**

`EAGER` means Hibernate loads the association immediately when the parent entity is loaded (usually via a JOIN or a second query right after). `LAZY` means Hibernate creates a proxy and only loads the association when it's first accessed. Defaults: `@ManyToOne` and `@OneToOne` are `EAGER` by default (a production trap — always override to `LAZY`); `@OneToMany` and `@ManyToMany` are `LAZY` by default. The problem with eager loading in a real application: if you have 5 `@ManyToOne` on an entity all set to EAGER, loading any list of that entity fires joins or secondary queries for all 5 associations — even when you only needed the ID.

---

**Q5: A `@Transactional` method on Service A calls a `@Transactional(propagation = REQUIRES_NEW)` method on Service B. What happens? How many DB connections are held?**

Service A's method starts Transaction 1 (acquires DB Connection 1). When it calls Service B's method, Spring suspends Transaction 1, starts Transaction 2 (acquires a second DB Connection 2). Both connections are held simultaneously until Service B's method commits/rolls back (Transaction 2 ends, Connection 2 returned to pool), then Transaction 1 resumes on Connection 1. This means 2 connections from the pool are held at the same time. In a high-throughput system with limited pool size, this can cause connection pool exhaustion — something interview candidates rarely think about.

---

**Q6: What happens when you call `em.find()` on an entity that doesn't exist vs `em.getReference()`?**

`em.find()` hits the DB immediately and returns `null` if the entity doesn't exist. `em.getReference()` returns a PROXY without hitting the DB. The proxy appears non-null. If you later access any field on the proxy (other than the ID), and the entity doesn't actually exist in DB, Hibernate throws `EntityNotFoundException`. Use `getReference()` only for setting up FK relationships when you have the ID and know the entity exists — it saves an unnecessary SELECT.

---

**Q7: Explain the difference between `CascadeType.REMOVE` and `orphanRemoval = true`.**

`CascadeType.REMOVE` propagates `em.remove(parent)` to child entities — removing the parent also removes children via JPA's remove operation. `orphanRemoval = true` deletes a child entity when it's **removed from the parent's collection** (`parent.getChildren().remove(child)`), treating removal from the collection as the entity becoming an orphan. The two are complementary: `CascadeType.REMOVE` handles explicit parent deletion; `orphanRemoval` handles collection-level disassociation. `orphanRemoval = true` implies `CascadeType.REMOVE`, so setting both is redundant for that case. Never use `CascadeType.REMOVE` (or `orphanRemoval`) on entities that are referenced from multiple parents — you'll get FK violations or unexpected deletions.

---

**Q8: What does `@Transactional(readOnly = true)` actually do under the hood?**

It sets the flush mode to `MANUAL` (or `NEVER`) on the Hibernate session — the session never auto-flushes, so no dirty checking overhead and no accidental writes. It passes a read-only hint to the JDBC connection, which some databases/drivers use to route to a read replica or use a lighter lock mode. It signals intent to developers that this method should not write. It does NOT prevent you from calling write methods — those calls simply won't flush. Use it on every query method to eliminate dirty-checking overhead, which can be significant for large transactions loading many entities.

---

## 17. JPA vs Hibernate — Which API to Use?

```
Use JPA API (EntityManager, @Entity, @NamedQuery, etc.) for:
  ✓ Code that should be portable across JPA providers
  ✓ Standard CRUD, JPQL, Criteria API operations
  ✓ When Spring Data JPA handles most of the boilerplate

Use Hibernate-specific API (Session, @BatchSize, @DynamicUpdate, @Cache, etc.) for:
  ✓ Bulk operations (StatelessSession)
  ✓ Fine-grained performance tuning (@BatchSize, @Fetch, @DynamicUpdate)
  ✓ Hibernate-specific second-level cache configuration
  ✓ Multi-tenancy configurations
  ✓ Hibernate Envers (entity auditing/versioning)

In practice:
  Spring Boot + Spring Data JPA → use JPA API 95% of the time
  Unwrap Hibernate session for specific optimizations:
    Session session = em.unwrap(Session.class);
    StatelessSession stateless = sessionFactory.openStatelessSession();
```

---

## 18. Common Pitfalls & Gotchas

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| **`@ManyToOne` / `@OneToOne` default EAGER** | Every entity load pulls in all associated entities | Always set `fetch = FetchType.LAZY` explicitly |
| **Returning entities from controller/service** | Jackson serializes entity; triggers lazy loading outside tx; `LazyInitializationException`; circular refs | Always return DTOs; never expose entities in the web layer |
| **Forgetting `mappedBy`** | Hibernate creates a join table (for `@OneToMany`) or two FK columns (for `@OneToOne`) | `mappedBy` is mandatory on the non-owning side of bidirectional relations |
| **Not setting both sides of bidirectional** | In-memory object graph is inconsistent | Add `addX()`/`removeX()` helper methods that set both sides |
| **Self-invocation with `@Transactional`** | Proxy is bypassed; transaction doesn't start/join | Refactor into separate bean or self-inject |
| **Checked exceptions not rolled back** | `@Transactional` only rolls back on `RuntimeException`; checked exceptions silently commit | Use `rollbackFor = Exception.class` or only throw `RuntimeException` subtypes |
| **`em.merge()` on new entity with null ID** | Works (Hibernate treats as insert) but wrong semantics — confusing | Use `em.persist()` for new entities |
| **Bulk `UPDATE`/`DELETE` bypasses first-level cache** | Post-bulk-update reads return stale data from cache | Call `em.clear()` after bulk JPQL updates |
| **Using `List` in `@ManyToMany`** | Any change → Hibernate deletes all join table rows and re-inserts | Use `Set` for `@ManyToMany` |
| **`JOIN FETCH` with multiple collections** | `MultipleBagFetchException` or Cartesian product | Fetch only one collection per query; use multiple queries or `@BatchSize` for others |
| **OSIV enabled in Spring Boot** | Lazy loading works accidentally; hides N+1 bugs; DB connections held for full HTTP request | Set `spring.jpa.open-in-view=false`; use explicit fetch strategies |
| **`CascadeType.ALL` on shared entities** | `em.remove(parent)` deletes entities shared by other parents → FK violation or data loss | Cascade only to child entities owned exclusively by this parent |
| **`ddl-auto = update` in production** | Hibernate adds columns/tables but never drops — schema drift accumulates | Use `ddl-auto = validate` or `none` with Flyway/Liquibase |
| **Comparing entities with `==` after `merge()`** | `merge()` returns a new persistent copy; original is still detached | Always use the returned copy from `merge()` |
| **`@Enumerated(EnumType.ORDINAL)`** | Reordering enum values corrupts all stored data | Always use `EnumType.STRING` |
| **Entity not a `public` class / missing no-arg constructor** | Hibernate can't proxy the entity → error at startup | Entities must be non-final, public, with no-arg constructor (can be `protected`) |
| **Long-running transaction calling external services** | DB connection held for the duration of the HTTP call → pool exhaustion under load | Never call external APIs inside `@Transactional`; load data, close tx, call API, open new tx to save |

---

## 19. JPA Specification — Key Versions

| Version | Key Additions |
|---------|--------------|
| JPA 1.0 (2006) | Initial: `EntityManager`, JPQL, Criteria API basics |
| JPA 2.0 (2009) | Full Criteria API, `@ElementCollection`, `@OrderColumn`, L2 cache API |
| JPA 2.1 (2013) | `@NamedStoredProcedureQuery`, `@Convert`/`AttributeConverter`, bulk UPDATE/DELETE, entity graphs |
| JPA 2.2 (2017) | Java 8 `java.time` support, `getResultStream()`, `@Repeatable` on annotations |
| Jakarta Persistence 3.0 (2020) | Package rename: `javax.persistence` → `jakarta.persistence` |
| Jakarta Persistence 3.1 (2022) | Math functions in JPQL, UUID as id type, numeric functions |
| Jakarta Persistence 3.2 (2024) | `find()` with multiple IDs, `@ManyToOne` on records (preview) |

---

## 20. `AttributeConverter` — Custom Type Mapping

```java
// Map a custom Java type to a DB column
@Converter(autoApply = true)  // autoApply = register globally for all matching fields
public class MoneyConverter implements AttributeConverter<Money, String> {

    @Override
    public String convertToDatabaseColumn(Money money) {
        if (money == null) return null;
        return money.getAmount() + " " + money.getCurrency();  // e.g., "99.99 USD"
    }

    @Override
    public Money convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        String[] parts = dbData.split(" ");
        return new Money(new BigDecimal(parts[0]), parts[1]);
    }
}

// Without autoApply, use @Convert on the field:
@Convert(converter = MoneyConverter.class)
private Money price;

// Encrypting sensitive data
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    @Override
    public String convertToDatabaseColumn(String attribute) {
        return EncryptionUtil.encrypt(attribute);
    }
    @Override
    public String convertToEntityAttribute(String dbData) {
        return EncryptionUtil.decrypt(dbData);
    }
}

@Convert(converter = EncryptedStringConverter.class)
private String socialSecurityNumber;  // stored encrypted in DB
```

---

## 21. Summary — Quick Revision

- **JPA is a specification.** Hibernate is the most common implementation. Spring Boot auto-configures Hibernate as the JPA provider.
- **EntityManager lifecycle:** Created per transaction in Spring (`@PersistenceContext` injects a thread-safe proxy). Persistence context (first-level cache) lives for the transaction duration.
- **Entity states:** Transient (new, unknown) → Persistent (managed, tracked) → Detached (untracked, changes not saved) → Removed (scheduled for delete). Dirty checking only works on Persistent entities.
- **Fetch type rule:** Always `LAZY` on `@ManyToOne` and `@OneToOne`. The defaults are EAGER — a performance trap in every real application.
- **OSIV must be disabled.** `spring.jpa.open-in-view=false`. Use DTO projections and explicit fetch strategies instead.
- **Transactions:** Only roll back on `RuntimeException` by default. `readOnly = true` eliminates dirty-check overhead on reads. Self-invocation bypasses the proxy.
- **Cascade carefully.** `ALL + orphanRemoval` for exclusive parent-child. Never cascade `REMOVE` to shared entities.
- **Always return DTOs from services.** Entities must not leave the transaction boundary.

---

> **What's next:** Spring Data JPA — the repository abstraction (`JpaRepository`, derived queries, `@Query`, `Specification`, projections, pagination) that sits on top of JPA and Hibernate.

> **Follow-up questions to think about:**
> 1. You call `em.merge(detachedEntity)` and then modify the returned object. Later you call `em.merge(detachedEntity)` again in the same transaction. What happens? Is there one `UPDATE` or two?
> 2. A `@Transactional(propagation = REQUIRED)` service method calls another `@Transactional(propagation = REQUIRED)` method on a different bean. One of them reads a row, the other updates it. Which isolation level is needed to guarantee the first read sees the updated value in the same transaction?
> 3. You have `@OneToMany(cascade = ALL, orphanRemoval = true)`. You call `order.getItems().clear()` and then add 3 new items. How many SQL statements does Hibernate generate and in what order?
