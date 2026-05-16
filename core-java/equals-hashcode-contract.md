# `equals()` and `hashCode()` Contract in Java

---

## 1. Overview

Every Java object inherits `equals()` and `hashCode()` from `java.lang.Object`. The **default** implementations use **reference identity** — two variables are equal only if they point to the same object in memory.

That works for identity comparison, but almost every real-world domain object needs **value equality** (two `User` objects with the same `id` should be considered equal). That's when you override both methods — and you *must* override both, not just one.

The contract that binds them is one of the most important, most violated rules in Java.

---

## 2. The Contract (What the JDK Spec Says)

### `equals()` Contract (from `Object.java` Javadoc)

| Property | Meaning |
|---|---|
| **Reflexive** | `x.equals(x)` must be `true` |
| **Symmetric** | `x.equals(y)` ↔ `y.equals(x)` |
| **Transitive** | `x.equals(y)` and `y.equals(z)` → `x.equals(z)` |
| **Consistent** | Multiple calls return the same result if nothing changed |
| **Null-safe** | `x.equals(null)` must return `false`, never throw NPE |

### `hashCode()` Contract

| Rule | Meaning |
|---|---|
| **Consistency** | Same object → same hash across multiple calls in one JVM run |
| **equals → same hash** | If `x.equals(y)` is `true`, then `x.hashCode() == y.hashCode()` **MUST** hold |
| **Not equal → hash CAN differ** | If `x.equals(y)` is `false`, the hashes *may* differ (but don't have to) |

### The Golden Rule (memorize this)

> **If you override `equals()`, you MUST override `hashCode()`.**
> If two objects are equal, they must have the same hash code.
> The reverse is NOT required — unequal objects can have the same hash.

---

## 3. How It Works Internally

### HashMap bucket lookup (the key insight)

```
put(key, value):
  1. bucket = key.hashCode() & (capacity - 1)   // which bucket?
  2. for each existing key in that bucket:
       if existing.equals(key) → overwrite value  // same key?
  3. if no match → insert new entry

get(key):
  1. bucket = key.hashCode() & (capacity - 1)
  2. for each entry in that bucket:
       if entry.equals(key) → return value
  3. return null
```

Both methods are used in **sequence**:
- `hashCode()` narrows down the bucket (O(1) lookup)
- `equals()` confirms the exact match within that bucket

**Breaking the contract breaks this lookup entirely.**

---

## 4. Real-World Example — Correct Implementation

```java
import java.util.Objects;

public class User {
    private final Long id;
    private String name;
    private String email;

    public User(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    // Business key: two users are the same if they share the same id
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                  // reflexivity shortcut
        if (!(o instanceof User)) return false;      // null-safe + type check
        User other = (User) o;
        return Objects.equals(this.id, other.id);    // null-safe field comparison
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);                     // consistent with equals
    }
}
```

> **Why `Objects.equals()` and `Objects.hash()`?**
> Both handle `null` fields gracefully without throwing NPE.

---

## 5. How It Is Crucial in Collections

### 5.1 HashMap / HashSet

```java
Set<User> set = new HashSet<>();
User u1 = new User(1L, "Alice", "alice@example.com");
User u2 = new User(1L, "Alice Changed", "alice@example.com");

set.add(u1);
set.contains(u2); // true — because id is the same and equals/hashCode agree
```

Without override:
```java
// default equals = reference equality
set.add(u1);
set.contains(u2); // FALSE — different object reference, even if logically same user
```

### 5.2 TreeMap / TreeSet — does NOT use hashCode

`TreeMap`/`TreeSet` use `compareTo()` (via `Comparable`) or a `Comparator`. `hashCode` is irrelevant here. `equals` is still used in some edge cases.

### 5.3 ArrayList.contains() / remove()

```java
List<User> users = new ArrayList<>();
users.add(new User(1L, "Alice", "alice@example.com"));
users.contains(new User(1L, "Alice", "alice@example.com")); // true if equals is overridden
```

`ArrayList` uses `equals()` for `contains()`, `indexOf()`, `remove(Object)`. No `hashCode()` involved.

### 5.4 Hibernate / JPA Entity Identity

Hibernate uses `equals`/`hashCode` to manage its **first-level cache** (identity map). Entities stored in `Set` relationships **must** have correct implementations to avoid duplicate rows or stale associations.

---

## 6. Scenario Walkthroughs

### Scenario 1 — The "Ghost Key" Bug

**Context:** An e-commerce app caches product lookups in a `HashMap<Product, PriceInfo>`.

```java
public class Product {
    private Long id;
    private String sku;
    // equals/hashCode NOT overridden — using Object defaults
}

Map<Product, PriceInfo> priceCache = new HashMap<>();
Product p1 = new Product(1L, "SKU-100");
priceCache.put(p1, new PriceInfo(99.99));

// Later, fetch from DB — same id, same sku, but new object reference
Product p2 = productRepo.findById(1L); // new Product object
priceCache.get(p2); // returns NULL — cache miss every time!
```

**Result:** The cache is completely useless. Every request hits the database.

**Fix:** Override `equals` and `hashCode` using `id`.

---

### Scenario 2 — Mutable Key Mutation

**Context:** An order management system uses `Order` as a HashMap key.

```java
public class Order {
    private String orderId;
    private String status; // MUTABLE

    @Override
    public boolean equals(Object o) { ... orderId + status ... }
    @Override
    public int hashCode() { return Objects.hash(orderId, status); }
}

Map<Order, List<Item>> orderMap = new HashMap<>();
Order o = new Order("ORD-001", "PENDING");
orderMap.put(o, items);

o.setStatus("SHIPPED"); // Mutation changes the hash!

orderMap.get(o); // NULL — bucket has changed, key is now a ghost!
```

**Result:** Memory leak + unreachable keys — classic production disaster.

**Fix:** Never include mutable fields in `equals`/`hashCode`. Use immutable business keys only (natural IDs, UUIDs).

---

### Scenario 3 — Hibernate Set Duplicate Entities

**Context:** A `User` has a `Set<Role>` roles.

```java
@Entity
public class Role {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    // equals/hashCode NOT overridden
}

user.getRoles().add(new Role("ADMIN"));
user.getRoles().add(new Role("ADMIN")); // duplicate! Set doesn't catch it
```

Without overriding `equals`/`hashCode`, the `Set` allows duplicates because `Object.equals` compares references. Both `ADMIN` roles get persisted.

**Fix:**
```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof Role)) return false;
    Role r = (Role) o;
    return Objects.equals(this.name, r.name);
}

@Override
public int hashCode() {
    return Objects.hash(name);
}
```

---

## 7. Miscellaneous Topics

### 7.1 The `instanceof` vs `getClass()` Debate

```java
// Approach 1: instanceof (allows subclass equality)
if (!(o instanceof User)) return false;

// Approach 2: getClass() (strict — no subclass equality)
if (o == null || getClass() != o.getClass()) return false;
```

- Use `instanceof` when you expect subclassing and Liskov substitution matters.
- Use `getClass()` for strict equality where subtypes are not considered equal.
- **Bloch (Effective Java)** recommends `instanceof` for most cases.
- **Warning:** Mixing both in a hierarchy breaks **symmetry**. If `A.equals(B)` uses `instanceof` but `B.equals(A)` uses `getClass()`, symmetry breaks.

---

### 7.2 Java Records (Java 16+)

Records auto-generate correct `equals`/`hashCode` based on all components:

```java
public record Point(int x, int y) {}

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
p1.equals(p2); // true — auto-generated
p1.hashCode() == p2.hashCode(); // true
```

Records are immutable by design, so no mutable-key danger.

---

### 7.3 Lombok `@EqualsAndHashCode`

```java
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @EqualsAndHashCode.Include
    private Long id;

    private String name; // excluded
}
```

**Gotcha with `@Data`:** `@Data` auto-includes ALL fields in `equals`/`hashCode`, which is almost never what you want for JPA entities. Always use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` explicitly.

---

### 7.4 `hashCode()` Quality and Performance

A bad hash function causes excessive collisions, degrading HashMap from O(1) to O(n):

```java
// Terrible: all instances hash to same bucket
@Override
public int hashCode() { return 1; }

// Better: spread across buckets
@Override
public int hashCode() { return Objects.hash(id, name); }
```

Java 8+ mitigates worst-case collision attack by converting long chains into **red-black trees** (threshold = 8), making worst case O(log n) instead of O(n).

---

### 7.5 `hashCode()` Across JVM Runs

`Object.hashCode()` (the default) is not consistent **across JVM restarts**. It's based on memory address (or a random seed in modern JVMs). Do NOT persist or transmit `hashCode()` values across JVM runs.

---

### 7.6 `String.hashCode()` Is Consistent

`String` hardcodes its hash function (`31 * h + c` algorithm), making it consistent across JVM runs. This is why `String` is safe as a map key even after serialization.

---

### 7.7 Caching `hashCode()` — Immutable Objects

For expensive hash computations on immutable objects, cache the result:

```java
public final class ComplexKey {
    private final String part1;
    private final String part2;
    private int cachedHash; // lazily computed, 0 = not computed

    @Override
    public int hashCode() {
        if (cachedHash == 0) {
            cachedHash = Objects.hash(part1, part2);
        }
        return cachedHash;
    }
}
```

This is exactly how `String` does it internally.

---

## 8. Common Interview Questions

**Q1. If you only override `equals()` but not `hashCode()`, what happens?**

Two objects that are `equals()` will have different hash codes (default `Object.hashCode()` is reference-based). `HashMap`/`HashSet` will:
- Put the first object in bucket X.
- When you look up the second (equal) object, it computes a different bucket Y.
- The key is not found. The set allows "duplicates". Cache misses everywhere.

This is the most common violation and the root of several production bugs.

---

**Q2. Can two unequal objects have the same `hashCode()`?**

Yes. This is called a **hash collision** and is perfectly legal. The contract only mandates: equal objects → same hash. It says nothing about the reverse.

---

**Q3. Why should you NOT use mutable fields in `hashCode()`?**

Once an object is placed in a `HashMap` or `HashSet`, mutating a field that participates in `hashCode()` changes the object's bucket. The collection cannot find it anymore — it becomes a **phantom key**. This is a memory leak and a correctness bug.

---

**Q4. What is the problem with using `id` from a JPA entity in `equals`/`hashCode`?**

New entities have `id = null` before persisting. Two new unsaved entities would both have `null` id:
```java
new User(null, "Alice").equals(new User(null, "Bob")); // TRUE — wrong!
```

Solutions:
- Use a **natural business key** (email, username, UUID assigned before persist).
- Use a UUID generated in the constructor, not by the DB.
- Fall back to `super.equals()` when id is null (not recommended for Sets).

---

**Q5. How does `HashSet.add()` use both methods?**

```
add(element):
  bucket = element.hashCode() % capacity
  for each e in bucket:
    if e.hashCode() == element.hashCode() && e.equals(element):
      return false  // duplicate, not added
  insert into bucket
  return true
```

Both `hashCode()` and `equals()` participate. This is why both must agree.

---

**Q6. What happens in a `HashMap` with all keys having the same `hashCode()`?**

All entries land in one bucket, forming a linked list (O(n) lookup). In Java 8+, after 8 entries in one bucket, the list converts to a **red-black tree** (O(log n)). Still far worse than a well-distributed hash (O(1)).

---

**Q7. Can `hashCode()` return negative values?**

Yes. `int` range is −2^31 to 2^31−1. HashMap handles this by masking: `(n - 1) & hash` where `n` is a power of 2, ensuring a non-negative bucket index.

---

**Q8. Does `equals()` need to handle the case where the argument is `null`?**

Yes. `x.equals(null)` must return `false` and must **never** throw `NullPointerException`. The `instanceof` check naturally handles this: `null instanceof User` is `false`.

---

## 9. Common Production Pitfalls

| Pitfall | What Happens | Fix |
|---|---|---|
| Override `equals` but not `hashCode` | Silent HashMap/HashSet failures | Always override both |
| Mutable fields in `hashCode` | Keys become unreachable after mutation | Use only immutable fields |
| Using all fields in JPA entity `equals` | Two new unsaved entities with same data conflict | Use business key or UUID |
| Lombok `@Data` on JPA entity | Includes all fields including mutable ones | Use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` |
| `instanceof` + inheritance | Symmetry violation when subclass overrides `equals` | Be consistent throughout hierarchy |
| Persisting `Object.hashCode()` to DB | Different values on next JVM start | Never store reference-based hash |
| Comparing `Double` fields with `==` in `equals` | Floating-point precision issues | Use `Double.compare()` or `Objects.equals()` |
| Forgetting `super.equals()` in inheritance chains | Parent fields not compared | Explicitly call `super.equals()` when appropriate |

---

## 10. Summary — Quick Revision

- **If `a.equals(b)` is true → `a.hashCode() == b.hashCode()` MUST be true.** The reverse is not required.
- **Always override both** — one without the other breaks collections silently.
- **Use immutable fields** (business keys, UUIDs) in `equals`/`hashCode` — never mutable state.
- **JPA entities** are the trickiest: avoid generated IDs in `equals`; prefer natural keys or pre-assigned UUIDs.
- `HashMap` uses `hashCode` to find the bucket, then `equals` to confirm the key — break either and the map fails.
- Java Records auto-generate correct, component-based `equals`/`hashCode` — prefer them for value objects.
