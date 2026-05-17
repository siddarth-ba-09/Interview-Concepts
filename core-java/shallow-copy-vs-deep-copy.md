# Shallow Copy vs Deep Copy in Java

---

## 1. Overview

**Copying an object** means creating a new object that is a duplicate of the original. In Java, there are two fundamentally different kinds of copies:

| | Shallow Copy | Deep Copy |
|--|--|--|
| **New object created?** | Yes | Yes |
| **Primitive fields** | Copied by value | Copied by value |
| **Reference fields** | Copied by reference (same object) | Copied recursively (new object) |
| **Independence** | Partial — changes to nested objects affect both | Full — original and copy are completely independent |

**Why it matters:** Most bugs involving copies in Java stem from assuming you have a deep copy when you only have a shallow one — mutating the copy silently mutates the original.

**Assignment is NOT a copy:**
```java
Person a = new Person("Alice");
Person b = a; // NOT a copy — both variables point to the same heap object
b.setName("Bob");
System.out.println(a.getName()); // "Bob" — a and b are the same object
```

A copy always produces a **new object on the heap**. Assignment just copies the reference variable.

---

## 2. Core Concepts & Sub-Topics

### 2.1 Object Graph

Every Java object may contain references to other objects, forming an **object graph** — a directed graph where nodes are objects and edges are reference fields.

```
Order
 ├── customerId: long          (primitive — value)
 ├── Customer customer         (reference → Customer object)
 │     ├── name: String        (reference → String object)
 │     └── address: Address    (reference → Address object)
 └── List<Item> items          (reference → ArrayList object)
       └── Item[]              (references → Item objects)
```

- **Shallow copy** duplicates only the root node and copies all edges as-is (same target objects).
- **Deep copy** duplicates the root node AND recursively all reachable objects in the graph.

### 2.2 Primitives vs References in Copies

```java
class Employee {
    int age;          // primitive — always copied by value
    String name;      // reference — shallow: same String object; deep: new String (though String is immutable)
    Address address;  // reference — shallow: same Address; deep: new Address
    List<String> tags;// reference — shallow: same ArrayList; deep: new ArrayList with copied contents
}
```

For **immutable objects** (`String`, `Integer`, `LocalDate`), shallow and deep copy are effectively equivalent — you cannot mutate them, so sharing them is safe. The distinction only matters for **mutable reference fields**.

### 2.3 Mechanisms for Shallow Copy

| Mechanism | How |
|-----------|-----|
| `Object.clone()` with `Cloneable` | Override `clone()`, call `super.clone()` |
| Copy constructor | `new MyClass(other)` — manually copy fields |
| `new ArrayList<>(original)` | Copies list structure, same element references |
| `Arrays.copyOf()` / `System.arraycopy()` | Shallow array copy |
| `Map.putAll()` | Shallow map copy |
| `Collections.copy()` | Shallow copy of list elements |
| `Stream.toList()` / `Collectors.toList()` | New list, same element objects |

### 2.4 Mechanisms for Deep Copy

| Mechanism | How | Pros | Cons |
|-----------|-----|------|------|
| Manual copy constructor/factory | Each class implements its own deep-copy logic | Full control, no deps | Tedious, error-prone as fields grow |
| `Serializable` round-trip | Serialize to bytes → deserialize | Works on any `Serializable` graph | Slow, requires all classes `Serializable`, security risk |
| Reflection-based libraries | Apache Commons `BeanUtils.cloneBean()` | Less boilerplate | Slow, shallow by default unless configured |
| `Cloneable` + override every level | Each class overrides `clone()` deeply | Standard Java | Fragile `Cloneable` contract, verbose |
| Jackson JSON round-trip | Serialize to JSON → deserialize | Works across types | Requires no-arg constructor, slow |
| Constructor chaining | Each class copies its own fields, nesting constructors | Clean, explicit | Must maintain as fields change |

---

## 3. How It Works Internally

### 3.1 `Object.clone()` Internals

`Object.clone()` is a **native method** implemented in the JVM. When called on an object:

1. Allocates a new object of the same class on the heap.
2. Copies all field values from the original to the new object **bit-for-bit** (field by field, including reference values).
3. Returns the new object.

This is inherently a **shallow copy** — reference fields hold the same memory addresses as the original.

```
BEFORE clone():
original ──────► [ age=30 | name─────► "Alice" | address─────► Address{city="NYC"} ]

AFTER super.clone():
copy     ──────► [ age=30 | name─────► "Alice" | address─────► Address{city="NYC"} ]
                                        ▲                        ▲
                                   same String              same Address object
```

Both `original.address` and `copy.address` point to the **same Address object** on the heap.

```java
// Mutating copy's address ALSO mutates original's address
copy.address.setCity("LA");
System.out.println(original.address.getCity()); // "LA" — same object!
```

### 3.2 Deep Copy Memory Model

```
BEFORE deep copy:
original ──► [ age=30 | name─► "Alice" | address─► Address{city="NYC"} ]

AFTER deep copy:
copy     ──► [ age=30 | name─► "Alice" | address─► Address{city="NYC"} ]  ← new address object
                                                         ↑
                                             different heap object — independent

copy.address.setCity("LA");
System.out.println(original.address.getCity()); // still "NYC"
```

### 3.3 Serialization Deep Copy Internals

```
Object graph → ObjectOutputStream → byte[] → ObjectInputStream → reconstructed graph
```

Serialization traverses the entire object graph following non-transient, non-static fields. Deserialization reconstructs the graph by creating new objects (bypassing constructors). This achieves a deep copy. The critical constraint: every object in the graph must implement `Serializable`.

### 3.4 Array Copying

| Method | Type | Notes |
|--------|------|-------|
| `array = original` | Reference copy | No array created |
| `Arrays.copyOf(arr, len)` | Shallow | New array, same element references |
| `System.arraycopy(src,0,dst,0,len)` | Shallow | Fastest; no bounds check overhead |
| `array.clone()` | Shallow | Equivalent to `Arrays.copyOf` |
| Manual loop | Deep (if elements are deep-copied) | Full control |

```
int[] primitives = {1, 2, 3};
int[] copy = Arrays.copyOf(primitives, 3); // TRUE deep copy — primitives are values

String[] strings = {"a", "b"};
String[] shallowCopy = Arrays.copyOf(strings, 3); // shallow — same String references
// (acceptable: String is immutable)

Address[] addresses = {new Address("NYC"), new Address("LA")};
Address[] shallowCopy = Arrays.copyOf(addresses, 2); // shallow — same Address objects!
```

---

## 4. Key APIs — Annotations / Classes / Interfaces / Methods

| Name | Type | Purpose | Notes |
|------|------|---------|-------|
| `Cloneable` | Interface (marker) | Signals that `Object.clone()` is allowed | Empty interface; not implementing it causes `CloneNotSupportedException` |
| `Object.clone()` | Method (protected) | Returns shallow copy of the object | Must override as `public`; calls native bit-copy |
| `CloneNotSupportedException` | Checked exception | Thrown by `Object.clone()` if not `Cloneable` | Almost always caught and wrapped |
| `System.arraycopy(src, srcPos, dst, dstPos, length)` | Static method | Fast native shallow array copy | Fastest array copy mechanism |
| `Arrays.copyOf(original, newLength)` | Static method | Shallow array copy; pads or truncates | Returns new array |
| `Arrays.copyOfRange(original, from, to)` | Static method | Shallow partial array copy | |
| `Collections.unmodifiableList(list)` | Static method | Wraps a list — NOT a copy | Same elements; writes throw `UnsupportedOperationException` |
| `List.copyOf(list)` | Static (Java 10+) | Shallow unmodifiable copy | Elements are same references |
| `Map.copyOf(map)` | Static (Java 10+) | Shallow unmodifiable copy | |
| `ObjectOutputStream` / `ObjectInputStream` | Classes | Serialization — deep copy via byte stream | All objects must be `Serializable` |
| `SerializationUtils.clone()` | Apache Commons | Serialization-based deep copy utility | Convenience wrapper |
| `BeanUtils.copyProperties()` | Apache Commons / Spring | Shallow property copy between beans | Only copies top-level field values |

---

## 5. Configuration & Setup

No special Spring Boot or Maven configuration is required for basic copy mechanisms. For serialization-based deep copy via Apache Commons:

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.14.0</version>
</dependency>
```

For Jackson-based deep copy:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.0</version>
</dependency>
```

Jackson deep copy (most practical for Spring Boot projects that already have Jackson):
```java
ObjectMapper mapper = new ObjectMapper();
MyClass deepCopy = mapper.readValue(mapper.writeValueAsString(original), MyClass.class);
// Requires: no-arg constructor, getter/setter or @JsonProperty
```

Or more efficiently using Jackson's built-in:
```java
MyClass deepCopy = mapper.convertValue(original, MyClass.class);
// Internally: serialize to JsonNode → deserialize — avoids String allocation
```

---

## 6. Real-World Production Code Example

**Domain: E-commerce Order Processing**

An order is created in the cart and then "confirmed" — producing a confirmed order snapshot. The confirmed order must be completely independent of the cart (the cart can be modified after checkout without affecting the confirmed order).

```java
// ── Domain Classes ──────────────────────────────────────────────────────────

@Getter
@Setter
public class Address {
    private String street;
    private String city;
    private String zip;

    public Address(String street, String city, String zip) {
        this.street = street;
        this.city   = city;
        this.zip    = zip;
    }

    // Deep copy constructor
    public Address(Address other) {
        this.street = other.street; // String — immutable, safe to share
        this.city   = other.city;
        this.zip    = other.zip;
    }
}

@Getter
@Setter
public class OrderItem {
    private String productId;
    private int    quantity;
    private BigDecimal unitPrice;

    public OrderItem(String productId, int quantity, BigDecimal unitPrice) {
        this.productId = productId;
        this.quantity  = quantity;
        this.unitPrice = unitPrice; // BigDecimal is immutable — safe to share
    }

    // Deep copy constructor
    public OrderItem(OrderItem other) {
        this.productId = other.productId;
        this.quantity  = other.quantity;
        this.unitPrice = other.unitPrice;
    }
}

@Getter
@Setter
public class Order {
    private String orderId;
    private Address shippingAddress;       // mutable — must deep copy
    private List<OrderItem> items;         // mutable list with mutable items

    public Order(String orderId, Address shippingAddress, List<OrderItem> items) {
        this.orderId         = orderId;
        this.shippingAddress = shippingAddress;
        this.items           = items;
    }

    // ── Shallow copy (via Cloneable) — DANGEROUS for mutable fields ──────────
    @Override
    public Order clone() {
        try {
            return (Order) super.clone(); // copies reference values — same Address, same List!
        } catch (CloneNotSupportedException e) {
            throw new AssertionError("Should never happen", e);
        }
    }

    // ── Deep copy constructor — CORRECT approach ──────────────────────────────
    public Order deepCopy() {
        return new Order(
            this.orderId,
            new Address(this.shippingAddress),                   // deep copy address
            this.items.stream()
                      .map(OrderItem::new)                       // deep copy each item
                      .collect(Collectors.toList())              // new list
        );
    }
}

// ── Service Layer ─────────────────────────────────────────────────────────────

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    /**
     * Demonstrates why deep copy matters in checkout flow.
     */
    public void demonstrateCopyProblem(Order cartOrder) {

        // ❌ WRONG — shallow clone: confirmed order shares Address and List with cart
        Order wrongConfirmed = cartOrder.clone();

        // Simulate user updating their cart address after checkout
        cartOrder.getShippingAddress().setCity("Los Angeles");

        // BUG: confirmed order address also changed!
        System.out.println(wrongConfirmed.getShippingAddress().getCity()); // "Los Angeles" ← wrong

        // ✅ CORRECT — deep copy: confirmed order is fully independent
        Order correctConfirmed = cartOrder.deepCopy();

        cartOrder.getShippingAddress().setCity("Chicago");

        System.out.println(correctConfirmed.getShippingAddress().getCity()); // "Los Angeles" — unchanged ✓
    }

    /**
     * Checkout flow — save a snapshot of the order.
     */
    public Order checkout(Order cartOrder) {
        // Deep copy ensures persisted order is a point-in-time snapshot
        Order confirmedOrder = cartOrder.deepCopy();
        confirmedOrder.setOrderId(UUID.randomUUID().toString());
        return orderRepository.save(confirmedOrder);
    }
}

// ── Serialization-Based Deep Copy Utility ────────────────────────────────────

public class DeepCopyUtils {

    /**
     * Deep copies any Serializable object graph.
     * All objects in the graph must implement Serializable.
     */
    @SuppressWarnings("unchecked")
    public static <T extends Serializable> T deepCopy(T original) {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            try (ObjectOutputStream oos = new ObjectOutputStream(bos)) {
                oos.writeObject(original);
            }
            byte[] bytes = bos.toByteArray();
            try (ObjectInputStream ois = new ObjectInputStream(
                    new ByteArrayInputStream(bytes))) {
                return (T) ois.readObject();
            }
        } catch (IOException | ClassNotFoundException e) {
            throw new RuntimeException("Deep copy failed", e);
        }
    }
}

// ── Array Shallow vs Deep Copy ────────────────────────────────────────────────

public class ArrayCopyDemo {

    public static void main(String[] args) {
        // Primitive array — Arrays.copyOf is effectively a deep copy
        int[] original = {1, 2, 3};
        int[] copy = Arrays.copyOf(original, original.length);
        copy[0] = 99;
        System.out.println(original[0]); // 1 — unaffected ✓

        // Object array — Arrays.copyOf is SHALLOW
        Address[] addresses = {
            new Address("123 Main St", "NYC", "10001"),
            new Address("456 Oak Ave", "LA", "90001")
        };
        Address[] shallowCopy = Arrays.copyOf(addresses, addresses.length);
        shallowCopy[0].setCity("Chicago");
        System.out.println(addresses[0].getCity()); // "Chicago" — same object! ✗

        // Object array — manual deep copy
        Address[] deepCopy = Arrays.stream(addresses)
                                   .map(Address::new)   // copy constructor
                                   .toArray(Address[]::new);
        deepCopy[0].setCity("Boston");
        System.out.println(addresses[0].getCity()); // still "Chicago" ✓
    }
}
```

---

## 7. Production Use-Cases & Best Practices

### Use-Case 1: Caching — Return Defensive Copies

When a cache stores objects and callers may mutate what they receive, return deep copies to protect the cached state:

```java
@Component
public class ProductCache {
    private final Map<String, Product> cache = new ConcurrentHashMap<>();

    public Optional<Product> get(String id) {
        Product cached = cache.get(id);
        return cached == null
            ? Optional.empty()
            : Optional.of(cached.deepCopy()); // caller gets an independent copy
    }
}
```

### Use-Case 2: Undo/Redo History

Maintaining a history stack for undo/redo requires a deep copy snapshot at each state change:

```java
Deque<DocumentState> history = new ArrayDeque<>();

void onUserEdit(DocumentState current) {
    history.push(current.deepCopy()); // save snapshot before applying change
    applyChange(current);
}

DocumentState undo() {
    return history.pop(); // restore previous independent snapshot
}
```

### Use-Case 3: Immutable DTOs from Mutable Entities

When converting a JPA entity (mutable) to a DTO for the API layer, deep-copy nested collections so the DTO is truly detached:

```java
public OrderDto toDto(Order entity) {
    return new OrderDto(
        entity.getOrderId(),
        entity.getItems().stream().map(ItemDto::new).collect(Collectors.toList())
    );
}
```

### Use-Case 4: Thread Safety — Sharing State Between Threads

Passing mutable objects between threads is unsafe. Deep copy before handing off:

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

void processAsync(Order order) {
    Order snapshot = order.deepCopy(); // independent copy for the worker thread
    executor.submit(() -> processOrder(snapshot));
    // Main thread can safely continue modifying 'order'
}
```

### Use-Case 5: Prototype Design Pattern

Spring's `prototype` bean scope creates a new bean instance per request, but the bean's internal mutable fields may still share references with other beans. Use deep copy in factories:

```java
@Scope("prototype")
@Component
public class ReportTemplate implements Cloneable {
    private List<ReportSection> sections; // mutable

    public ReportTemplate copy() {
        return new ReportTemplate(
            sections.stream().map(ReportSection::new).collect(Collectors.toList())
        );
    }
}
```

### Use-Case 6: Defensive Parameter Copying in Constructors

Prevent callers from mutating the internal state of your object after construction:

```java
public class Payroll {
    private final List<Employee> employees;

    public Payroll(List<Employee> employees) {
        // Defensive shallow copy at minimum — deep copy if elements are mutable
        this.employees = new ArrayList<>(employees);
    }

    public List<Employee> getEmployees() {
        return Collections.unmodifiableList(employees); // read-only view
    }
}
```

### Best Practices Summary

- **Prefer copy constructors over `Cloneable`** — copy constructors are cleaner, don't require casting, and work with inheritance naturally.
- **Immutable objects don't need deep copying** — `String`, `Integer`, `BigDecimal`, `LocalDate` are safe to share.
- **Document whether your `clone()`/copy is shallow or deep** — ambiguity causes bugs.
- **For Jackson-based projects**, `mapper.convertValue()` is the most pragmatic deep-copy mechanism — no `Serializable` required.
- **Avoid serialization deep copy in hot paths** — it is 10–100x slower than copy constructors.
- **Return `Collections.unmodifiableList()` from getters** when deep copying is too expensive — prevent external mutation.

---

## 8. Scenario Walkthroughs

### Scenario 1: Cart Modification After Checkout

**Setup:** An e-commerce service uses `Object.clone()` to create a confirmed order from the cart.

```java
Order cart = new Order("cart-1", new Address("5th Ave", "NYC", "10001"),
    new ArrayList<>(List.of(new OrderItem("P001", 2, new BigDecimal("49.99")))));

Order confirmed = cart.clone(); // SHALLOW clone
```

**What happens step-by-step:**
1. `super.clone()` creates a new `Order` object.
2. `shippingAddress` field in `confirmed` holds the **same reference** as in `cart` — points to the same `Address` heap object.
3. `items` field in `confirmed` holds the **same reference** as in `cart` — points to the same `ArrayList`.
4. User goes back and updates cart shipping address: `cart.getShippingAddress().setCity("LA")`.
5. Since `confirmed.shippingAddress` is the same object, `confirmed.getShippingAddress().getCity()` now returns `"LA"` — the confirmed order is corrupted.
6. User removes an item from cart: `cart.getItems().remove(0)`.
7. `confirmed.getItems()` is now empty too — same ArrayList.

**Correct handling:**
```java
Order confirmed = cart.deepCopy(); // copy constructor recursively copies Address and each Item into a new List
```
Now cart and confirmed are fully independent.

---

### Scenario 2: Shallow Copy of a List — The "Fixed" List That Isn't

**Setup:** A developer "copies" a list before modifying it:

```java
List<Address> original = new ArrayList<>(List.of(
    new Address("1 Main St", "NYC", "10001"),
    new Address("2 Oak Ave", "LA", "90001")
));

List<Address> copy = new ArrayList<>(original); // shallow copy of the list
copy.get(0).setCity("Chicago");                 // mutates Address object

System.out.println(original.get(0).getCity()); // "Chicago" — SURPRISE
```

**What happened:**
- `new ArrayList<>(original)` creates a new `ArrayList` with the same element references.
- `copy.get(0)` and `original.get(0)` return the **same Address object**.
- Mutating it through `copy` is the same as mutating it through `original`.

**What IS safe with a shallow list copy:**
- `copy.add(new Address(...))` — adds to `copy`, not `original` (different List object).
- `copy.remove(0)` — removes from `copy`, not `original` (different List object).
- Structural operations (add/remove/clear) on the list are independent.
- Mutating the elements themselves is NOT independent.

**Correct deep copy:**
```java
List<Address> deepCopy = original.stream()
    .map(a -> new Address(a.getStreet(), a.getCity(), a.getZip()))
    .collect(Collectors.toList());
```

---

### Scenario 3: Singleton Breaks with `clone()`

**Setup:** A class is designed as a singleton but implements `Cloneable`:

```java
public class AppConfig implements Cloneable {
    private static final AppConfig INSTANCE = new AppConfig();
    private String dbUrl = "jdbc:postgresql://prod-db/myapp";

    private AppConfig() {}
    public static AppConfig getInstance() { return INSTANCE; }

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone(); // DANGEROUS for a singleton!
    }
}

AppConfig config = AppConfig.getInstance();
AppConfig clone  = (AppConfig) config.clone(); // creates a SECOND instance!
clone.setDbUrl("jdbc:h2:mem:test");            // doesn't affect INSTANCE, but singleton is broken
System.out.println(clone == AppConfig.getInstance()); // false — two instances exist
```

**Correct handling:**
Override `clone()` to throw `CloneNotSupportedException`, or return the same singleton instance:
```java
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Singleton — cloning not allowed");
    // OR: return INSTANCE;
}
```
Similarly, override `readResolve()` for Serializable singletons to prevent serialization from creating a second instance.

---

## 9. Common Pitfalls & Gotchas

### Pitfall 1: Assuming `new ArrayList<>(original)` is a Deep Copy

**Problem:** Developers see "new ArrayList" and assume independence. It's only the list structure that's new — the elements are shared.

**Why it happens:** The ArrayList constructor copies element references, not elements.

**Fix:** Stream + copy constructor for each element, or `List.copyOf()` (still shallow but at least makes the list unmodifiable).

---

### Pitfall 2: `Object.clone()` is Always Shallow

**Problem:** Overriding `clone()` as `public` and calling `super.clone()` gives a shallow copy. Many developers don't realize they need to recursively clone mutable fields.

**Why it happens:** The JDK documentation for `clone()` says "returns a copy" without being explicit that it's shallow.

**Fix:** Override `clone()` to also clone every mutable reference field:
```java
@Override
public Employee clone() {
    try {
        Employee copy = (Employee) super.clone();
        copy.address = this.address.clone(); // clone mutable nested object
        copy.tags    = new ArrayList<>(this.tags); // shallow-copy list of Strings (OK — immutable elements)
        return copy;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(e);
    }
}
```

---

### Pitfall 3: `Cloneable` is a Broken API

**Problem:** `Cloneable` is a marker interface but the method `clone()` is in `Object`, not `Cloneable`. Implementing `Cloneable` doesn't give you a `clone()` method — it just changes the behavior of `Object.clone()` (removes the `CloneNotSupportedException`). The method is `protected` in `Object`, so you must override it as `public`.

**Why it happens:** A design mistake in the JDK from Java 1.0.

**Fix:** Prefer copy constructors or static factory `copy()` methods. Joshua Bloch's _Effective Java_ (Item 13) recommends avoiding `Cloneable` entirely.

---

### Pitfall 4: `String` Fields Don't Need Deep Copying — But Mutable Wrappers Do

**Problem:** String is immutable, so `copy.name = this.name` is safe. But `StringBuilder`, `Date`, `byte[]`, and custom mutable classes must be deep-copied.

```java
class Record {
    String name;             // immutable — safe to share
    Date createdAt;          // MUTABLE — must deep copy
    byte[] data;             // MUTABLE array — must deep copy
    StringBuilder notes;    // MUTABLE — must deep copy
}

// Deep copy:
copy.createdAt = new Date(this.createdAt.getTime());
copy.data      = Arrays.copyOf(this.data, this.data.length);
copy.notes     = new StringBuilder(this.notes);
```

---

### Pitfall 5: Cyclic Object Graphs Break Naive Deep Copy

**Problem:** If your object graph has cycles (A → B → A), a naive recursive deep copy will throw a `StackOverflowError`.

```java
class Node {
    Node next;
    Node prev; // doubly linked — creates cycles
}
```

**Fix:** Use a `Map<Object, Object>` to track already-copied objects during the deep copy traversal:
```java
Map<Object, Object> visited = new IdentityHashMap<>();
// Before copying an object: check if it's already in 'visited'
// If yes, return the already-copied version
// If no, create the copy, put in visited, then copy fields
```
Serialization handles cycles automatically (via the handle table in `ObjectOutputStream`).

---

### Pitfall 6: `Collections.unmodifiableList()` Is NOT a Copy

`Collections.unmodifiableList(list)` returns a **view** of the original list — it wraps the same backing list. Modifying the original list through the original reference still reflects in the "unmodifiable" view:

```java
List<String> original = new ArrayList<>(List.of("a", "b"));
List<String> view = Collections.unmodifiableList(original);

view.add("c"); // UnsupportedOperationException — view prevents modification through itself
original.add("c"); // works! view now also contains "c"
System.out.println(view.size()); // 3 — surprising if you thought view was a copy
```

For a true read-only copy, use `List.copyOf(list)` (Java 10+) — creates a new, unmodifiable list.

---

### Pitfall 7: `Arrays.copyOf` on 2D Arrays

```java
int[][] matrix = {{1, 2}, {3, 4}};
int[][] copy   = Arrays.copyOf(matrix, matrix.length); // shallow!

copy[0][0] = 99;
System.out.println(matrix[0][0]); // 99 — same inner array!

// True deep copy of 2D array:
int[][] deepCopy = new int[matrix.length][];
for (int i = 0; i < matrix.length; i++) {
    deepCopy[i] = Arrays.copyOf(matrix[i], matrix[i].length);
}
```

---

## 10. Miscellaneous & Advanced Details

### 10.1 `record` Classes and Copying (Java 16+)

Records are **immutable** — all fields are `final`. There's no need for defensive copying because the record's state cannot change. However, if a record field holds a mutable object (e.g., `List`), that object can still be mutated through the reference:

```java
record Order(String id, List<String> items) {
    // Compact constructor — defensive copy on construction
    Order {
        items = List.copyOf(items); // unmodifiable snapshot at creation time
    }
}

Order o = new Order("1", new ArrayList<>(List.of("A", "B")));
// o.items() returns the unmodifiable List.copyOf — caller cannot mutate it
```

### 10.2 `Object.clone()` vs Copy Constructor — Comparison

| | `clone()` | Copy Constructor |
|--|--|--|
| Type safety | Requires cast — `(MyClass) super.clone()` | No cast needed |
| Works with `final` fields | Cannot set — bit-copy bypasses constructors | Can set via constructor |
| Inheritance | Tricky — `super.clone()` must be overridden all the way up | Straightforward |
| Interface-based copying | `Cloneable` broken design | Clean, no marker interface needed |
| Exception handling | Must handle `CloneNotSupportedException` | No exceptions |
| Recommendation | Avoid unless required by framework | **Prefer this** |

### 10.3 Copy vs Defensive Copy vs Protective Copy

- **Copy** — general term for creating a duplicate.
- **Defensive copy** — copy made by a class of its inputs or outputs to protect its invariants from external mutation:
  ```java
  // In constructor: defensive copy of input
  public Period(Date start, Date end) {
      this.start = new Date(start.getTime()); // prevent caller from mutating
  }
  // In getter: defensive copy of output
  public Date getStart() { return new Date(start.getTime()); } // prevent receiver from mutating
  ```
- **Protective copy** — copy made by a caller before passing an object somewhere it might be mutated.

### 10.4 Performance Considerations

| Copy Method | Approximate Speed | Notes |
|-------------|------------------|-------|
| Copy constructor / manual | Fastest | Zero overhead — direct field assignment |
| `Arrays.copyOf` / `System.arraycopy` | Fastest for arrays | Native method — very fast |
| `Object.clone()` | Fast | Native method, but shallow |
| Jackson `convertValue()` | Moderate | Reflection + intermediate JsonNode |
| Serialization round-trip | Slow | I/O streams, full graph traversal |

For objects cloned frequently (cache entries, request DTOs), use copy constructors or factory methods. Avoid serialization in hot paths.

### 10.5 Spring's `BeanUtils.copyProperties()`

Spring's `BeanUtils.copyProperties(source, target)` copies matching property values (by name and type) between two beans. It is **always shallow**:

```java
OrderDto dto = new OrderDto();
BeanUtils.copyProperties(entity, dto);
// dto.getItems() and entity.getItems() are the SAME List object
```

Use this only when all fields are immutable (String, primitives, wrappers) or when you explicitly deep-copy nested collections afterward.

### 10.6 Java 9+ `List.copyOf()`, `Map.copyOf()`, `Set.copyOf()`

These produce **shallow, unmodifiable** copies:
- New collection is created with the same element references.
- Attempting to mutate via add/remove throws `UnsupportedOperationException`.
- The elements themselves are still mutable if they are mutable types.

```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> copy = List.copyOf(original);
original.add("d");
System.out.println(copy.size()); // 3 — structural change not reflected
// But: if elements were mutable objects, their mutations would still be visible
```

---

## 11. Interview Questions — Standard

**Q1: What is the difference between a shallow copy and a deep copy in Java?**

A shallow copy creates a new object and copies all field values from the original. For primitive fields, the actual values are copied. For reference fields, only the reference (memory address) is copied — the original and the copy point to the same heap objects. A deep copy creates a new object and recursively copies all objects reachable through reference fields, so the original and copy are completely independent. The distinction only matters for mutable reference fields — immutable objects (String, Integer) are safe to share.

**Q2: What does `Object.clone()` do? Is it shallow or deep?**

`Object.clone()` is a native JVM method that allocates a new object of the same class and copies all field values bit-for-bit from the original — it is always a **shallow copy**. To get a deep copy via `clone()`, you must override `clone()` in every class that has mutable reference fields and explicitly clone those fields. The class must also implement `Cloneable` (a marker interface), otherwise `clone()` throws `CloneNotSupportedException`.

**Q3: Why is `Cloneable` considered a broken or poorly designed API?**

Several reasons: (1) `Cloneable` is a marker interface, but the actual method (`clone()`) is on `Object`, not on `Cloneable` — there is no way to call `clone()` polymorphically on a `Cloneable` reference. (2) `Object.clone()` is `protected`, so you must override it as `public` in every class. (3) It bypasses constructors entirely, which means `final` fields cannot be set and any constructor-level validation is skipped. (4) It is inherently shallow. Joshua Bloch's _Effective Java_ recommends avoiding `Cloneable` and using copy constructors or static factory methods instead.

**Q4: Is `new ArrayList<>(original)` a shallow copy or deep copy?**

Shallow copy. It creates a new `ArrayList` object with the same element references. Structural operations on the copy (add, remove, clear) do not affect the original. However, mutating the objects referenced by the elements through either the copy or the original affects the same underlying objects — they share the same element instances.

**Q5: How do you perform a deep copy using Java serialization?**

Serialize the object to a `ByteArrayOutputStream` using `ObjectOutputStream`, then deserialize it back from a `ByteArrayInputStream` using `ObjectInputStream`. The deserialization process creates new objects for the entire graph. The requirement is that all objects in the graph implement `Serializable`. This is significantly slower than copy constructors and should not be used in hot paths.

**Q6: What is a defensive copy and why is it important?**

A defensive copy is a copy made by a class of its inputs (in constructors) or outputs (in getters) to protect its internal state from external mutation. In a constructor, if you store a mutable parameter directly, the caller can later mutate it and change the object's state unexpectedly. In a getter, if you return a direct reference to a mutable internal field, the caller can mutate it. Defensive copies prevent both problems. This is essential for building truly immutable classes.

**Q7: What is `Collections.unmodifiableList()` and how is it different from a copy?**

`Collections.unmodifiableList(list)` returns a **view** (wrapper) around the original list, not a copy. It delegates all read operations to the backing list and throws `UnsupportedOperationException` for write operations. If the original list is modified through the original reference, the view reflects those changes. For a true independent copy, use `List.copyOf(list)` (Java 10+) or `new ArrayList<>(list)`.

---

## 12. Interview Questions — Tricky / Scenario-Based

**Q1: A developer writes `copy.items = new ArrayList<>(original.items)` and says "I made a deep copy." Are they correct?**

Only partially. The `ArrayList` itself is new — adding or removing from `copy.items` won't affect `original.items`. However, the elements inside the list are shared — both lists point to the same `Item` objects. If `Item` is mutable and someone calls `copy.items.get(0).setPrice(99)`, `original.items.get(0).getPrice()` also returns `99`. This is a classic **shallow copy misconception**. A true deep copy would also copy each `Item`.

**Q2: A class has a `final` field. Can you deep-copy it using `Object.clone()`?**

`Object.clone()` copies field values bit-for-bit (including `final` fields) directly in memory — it bypasses the constructor and the `final` field contract is not violated from the JVM's perspective. However, if you need to set a **different** value for the `final` field in the copy (e.g., a new ID or a deep-copied nested object), `clone()` cannot do it — `final` fields cannot be assigned outside of constructors. This is one of the key reasons copy constructors are preferred over `clone()`.

**Q3: You have a `List<List<Integer>>` (a 2D list). You do `new ArrayList<>(original)`. Is this a deep copy?**

No — it's a one-level shallow copy. The outer `ArrayList` is new, but the inner `List<Integer>` objects are shared. Adding to `copy.get(0)` would also add to `original.get(0)`. A true deep copy requires copying each inner list:
```java
List<List<Integer>> deep = original.stream()
    .map(ArrayList::new) // copy each inner list
    .collect(Collectors.toList());
```
Even this is only two levels deep — if elements were mutable objects, you'd need another level.

**Q4: Can a Singleton be broken by `clone()`? How do you prevent it?**

Yes. If a Singleton class implements `Cloneable` and doesn't override `clone()` to prevent it, anyone can create a second instance by calling `instance.clone()`. Prevention: override `clone()` to throw `CloneNotSupportedException` or return the singleton instance itself. Similarly, a `Serializable` Singleton must override `readResolve()` to return the singleton instance during deserialization, or each deserialization creates a new instance.

**Q5: `String` is immutable — do you need to deep-copy a `String` field?**

No. Since `String` is immutable, sharing the same `String` reference between original and copy is perfectly safe — neither can change the `String` object. Shallow copy of `String` fields is always correct. This applies to all immutable types: `Integer`, `Long`, `Double`, `BigDecimal`, `LocalDate`, `LocalDateTime`, `UUID`. The deep copy concern only applies to **mutable** reference types.

**Q6: What happens to `transient` fields during a serialization-based deep copy?**

`transient` fields are **not serialized** and therefore not deserialized — the copy gets the default value for those fields (`null` for objects, `0` for numbers, `false` for booleans). If a `transient` field holds important state (e.g., a cached computed value or a connection pool), the copy will be missing it. This is a significant limitation of serialization-based deep copy — you may need to post-process the copy to reinitialize `transient` fields.

**Q7: Is this a deep copy or shallow copy? `new HashMap<>(original)` where values are `List<String>`.**

Shallow copy. The new `HashMap` has the same key-value pairs. The `List<String>` values are **shared** — `copy.get("key")` and `original.get("key")` return the same `List<String>` object. Mutating `copy.get("key").add("x")` also affects `original.get("key")`. A deep copy would require copying each value list:
```java
Map<String, List<String>> deep = original.entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> new ArrayList<>(e.getValue()) // copy each list
    ));
```

---

## 13. Quick Revision Summary

- **Assignment (`b = a`)** copies the reference — no new object, same heap object. NOT a copy.
- **Shallow copy** — new object, but mutable reference fields still point to the same nested objects.
- **Deep copy** — new object, AND all mutable nested objects are recursively duplicated.
- **Immutable objects** (`String`, `Integer`, `BigDecimal`, `LocalDate`) are safe to share — shallow copy is sufficient for them.
- **`Object.clone()` is always shallow** — override every mutable reference field to make it deep.
- **`Cloneable` is a broken API** — prefer copy constructors or `deepCopy()` factory methods.
- **`new ArrayList<>(list)`** = shallow list copy: structural independence (add/remove safe), element-level dependence (mutations to elements affect both).
- **`List.copyOf(list)` / `Collections.unmodifiableList(list)`** — `List.copyOf` creates a shallow unmodifiable copy; `unmodifiableList` is just a view (not a copy at all).
- **Serialization deep copy** — works for any `Serializable` graph, handles cycles; but `transient` fields get default values and it's significantly slower than copy constructors.
- **2D arrays / nested collections** require explicit multi-level deep copy — `Arrays.copyOf` on a `int[][]` only copies outer array references.
- **`BeanUtils.copyProperties()`** (Spring/Apache) is always shallow — only use when fields are immutable or you copy nested objects manually afterward.
- **Defensive copy rule** — copy mutable inputs in constructors; return copies (or unmodifiable views) from getters of mutable fields.
