# Object-Oriented Programming (OOP) Concepts in Java & Spring Boot

---

## 1. Overview

Object-Oriented Programming is a programming paradigm that organises software around **objects** — data structures that bundle **state** (fields) and **behaviour** (methods) together. Java is class-based and object-oriented at its core.

**Four pillars of OOP:**
1. **Encapsulation** — hide internal state; expose only via a controlled interface
2. **Abstraction** — hide implementation complexity; show only what is necessary
3. **Inheritance** — derive new types from existing ones, promoting code reuse
4. **Polymorphism** — the same interface can refer to different underlying types

**Why it matters for Spring Boot:**  
Spring's entire architecture is built on OOP and layered with Design Patterns. The IoC container, AOP, Spring MVC's DispatcherServlet, and Spring Data repositories are all living examples of OOP principles in action. Understanding OOP deeply is a prerequisite for understanding Spring.

---

## 2. Core Concepts & Sub-Topics

### 2.1 Encapsulation

**Definition:** Bundling data (fields) and methods that operate on that data within one class, and restricting direct access to the fields from outside.

**How it is achieved in Java:**
- Declare fields `private`
- Provide `public` getter/setter methods (or use `@Getter/@Setter` from Lombok)
- Use access modifiers: `private` → `package-private` (default) → `protected` → `public`

```java
public class BankAccount {
    private double balance;  // hidden state
    private String accountNumber;

    public double getBalance() { return balance; }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;
    }
    // No setter for balance — only controlled mutation via deposit/withdraw
}
```

**Benefits:**
- Prevents invalid state (validation in setters/methods)
- Changes to internals don't break callers
- Easier testing and maintenance

**Access Modifier Summary:**

| Modifier        | Same Class | Same Package | Subclass | Any Class |
|----------------|------------|--------------|----------|-----------|
| `private`       | ✅         | ❌           | ❌       | ❌        |
| (default)       | ✅         | ✅           | ❌       | ❌        |
| `protected`     | ✅         | ✅           | ✅       | ❌        |
| `public`        | ✅         | ✅           | ✅       | ✅        |

---

### 2.2 Abstraction

**Definition:** Exposing only the *what* (interface/contract), hiding the *how* (implementation).

**Mechanisms in Java:**
- **Abstract classes** — `abstract` keyword; can have abstract and concrete methods; cannot be instantiated
- **Interfaces** — pure contract (Java 8+ allows `default` and `static` methods)

```java
// Abstraction via interface
public interface PaymentGateway {
    PaymentResult charge(Order order, PaymentDetails details);
    PaymentResult refund(String transactionId, double amount);
}

// Two concrete implementations
public class StripeGateway implements PaymentGateway { ... }
public class PayPalGateway implements PaymentGateway { ... }
```

**Abstract Class vs Interface:**

| Feature                      | Abstract Class                  | Interface                           |
|-----------------------------|---------------------------------|-------------------------------------|
| Instantiation                | No                              | No                                  |
| Multiple inheritance         | No (single)                     | Yes (a class implements many)       |
| Fields                       | Any type                        | `public static final` only          |
| Constructors                 | Yes                             | No                                  |
| Methods                      | Abstract + concrete             | Abstract + `default` + `static`     |
| Access modifiers on methods  | Any                             | `public` by default                 |
| `IS-A` relationship          | Stronger (base type)            | Contract/capability                 |
| When to use                  | Shared base implementation      | Define a capability/contract        |

**In Spring Boot:**  
Spring's `JpaRepository`, `CrudRepository` are interfaces; their implementations (`SimpleJpaRepository`) are injected by Spring Data at runtime — pure abstraction.

---

### 2.3 Inheritance

**Definition:** A class (subclass/child) inherits fields and methods from another class (superclass/parent), extending or specialising it.

```java
public class Vehicle {
    protected String brand;
    protected int speed;

    public void accelerate(int amount) { this.speed += amount; }
    public void brake(int amount) { this.speed = Math.max(0, this.speed - amount); }
}

public class Car extends Vehicle {
    private int numberOfDoors;

    @Override
    public void accelerate(int amount) {
        // Cars have a speed limit
        this.speed = Math.min(this.speed + amount, 200);
    }
}
```

**Key rules:**
- Java supports **single inheritance** for classes (a class can extend only one class)
- A class can implement **multiple interfaces**
- `super` keyword refers to the parent class
- `final` class cannot be extended; `final` method cannot be overridden
- Constructors are not inherited, but `super()` can call parent constructor

**IS-A relationship:**
- `Car IS-A Vehicle` → `Car extends Vehicle` ✅
- Inheritance models a true IS-A relationship; if you cannot say "X is a Y", prefer composition

**Inheritance vs Composition (Prefer Composition Over Inheritance):**

```java
// Inheritance (tight coupling — breaks Liskov if misused)
class Stack extends ArrayList { ... } // BAD: Stack IS-NOT an ArrayList

// Composition (better)
class Stack {
    private final ArrayList<Object> storage = new ArrayList<>();
    public void push(Object item) { storage.add(item); }
    public Object pop() { return storage.remove(storage.size() - 1); }
}
```

**In Spring Boot:**  
`@RestController` extends `@Controller`. `SimpleJpaRepository` extends `JpaRepositoryImplementation`. Spring beans frequently use abstract base classes to share common behavior (e.g., a common `BaseEntity` with `createdAt`/`updatedAt`).

---

### 2.4 Polymorphism

**Definition:** The ability of an object to take many forms — the same method call can behave differently depending on the actual type of the object at runtime.

**Two types:**

#### Compile-time Polymorphism (Method Overloading)
Same method name, different parameter signatures in the **same class**. Resolved at compile time.

```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}
```

#### Runtime Polymorphism (Method Overriding)
A subclass provides its own implementation of a parent method. Resolved at **runtime** via dynamic dispatch.

```java
public class NotificationService {
    public void send(String message) {
        System.out.println("Generic notification: " + message);
    }
}

public class EmailNotificationService extends NotificationService {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

public class SMSNotificationService extends NotificationService {
    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

// Runtime polymorphism in action
NotificationService service = new EmailNotificationService();
service.send("Hello"); // → "Email: Hello"
service = new SMSNotificationService();
service.send("Hello"); // → "SMS: Hello"
```

**Rules for Overriding:**
- Return type must be the same or a **covariant** (subtype)
- Access modifier must be the same or **more permissive** (cannot narrow visibility)
- Cannot override `final`, `static`, or `private` methods
- `@Override` annotation is recommended (compiler check)
- Overriding method can declare fewer checked exceptions, not more

**Overloading vs Overriding:**

| Feature           | Overloading             | Overriding                         |
|------------------|-------------------------|------------------------------------|
| Defined in       | Same class              | Subclass                           |
| Resolution       | Compile time            | Runtime (JVM vtable)               |
| `@Override`      | N/A                     | Recommended                        |
| Return type      | Can differ              | Same or covariant                  |
| `static` methods | Can be overloaded       | Cannot be overridden (hidden)      |
| `private` methods| N/A                     | Cannot be overridden               |

**In Spring Boot:**  
Spring's AOP proxies intercept method calls and dispatch to advice implementations — runtime polymorphism. `@Transactional` works because Spring wraps your bean in a proxy that overrides your methods.

---

### 2.5 SOLID Principles

#### S — Single Responsibility Principle (SRP)
> A class should have only one reason to change.

```java
// BAD: UserService handles both user logic AND email
class UserService {
    void registerUser(User user) { /* save user */ }
    void sendWelcomeEmail(User user) { /* send email */ } // separate concern!
}

// GOOD: separate responsibilities
@Service class UserService {
    void registerUser(User user) { userRepository.save(user); }
}
@Service class EmailService {
    void sendWelcomeEmail(User user) { /* email logic */ }
}
```

#### O — Open/Closed Principle (OCP)
> Open for extension, closed for modification.

```java
// BAD: adding a new discount requires modifying this class
double applyDiscount(String type, double price) {
    if (type.equals("SEASONAL")) return price * 0.9;
    if (type.equals("LOYALTY")) return price * 0.85;
    // need to add more ifs for every new type
}

// GOOD: extend via polymorphism
public interface DiscountStrategy {
    double apply(double price);
}
@Component("seasonal") class SeasonalDiscount implements DiscountStrategy { ... }
@Component("loyalty")  class LoyaltyDiscount  implements DiscountStrategy { ... }
// Adding a new discount = new class, no change to existing code
```

#### L — Liskov Substitution Principle (LSP)
> Objects of a subclass must be substitutable for objects of their superclass without breaking behaviour.

```java
// VIOLATION: Square extends Rectangle but breaks expected behaviour
class Rectangle {
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    @Override void setWidth(int s) { this.width = s; this.height = s; }
    @Override void setHeight(int s) { this.width = s; this.height = s; }
}
// A consumer that sets w=5, h=3 and expects area=15 gets area=9 — broken!
```

**Fix:** `Square` and `Rectangle` should be separate types, or share an interface `Shape`.

#### I — Interface Segregation Principle (ISP)
> Clients should not be forced to depend on interfaces they do not use.

```java
// BAD: fat interface
interface Animal {
    void eat();
    void fly();   // not all animals fly!
    void swim();  // not all animals swim!
}

// GOOD: segregated interfaces
interface Eatable  { void eat(); }
interface Flyable  { void fly(); }
interface Swimmable{ void swim(); }

class Duck implements Eatable, Flyable, Swimmable { ... }
class Dog  implements Eatable, Swimmable          { ... }
class Eagle implements Eatable, Flyable           { ... }
```

#### D — Dependency Inversion Principle (DIP)
> Depend on abstractions, not concretions.

```java
// BAD: high-level module depends on low-level module
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository(); // concrete!
}

// GOOD: depend on interface (Spring DI enforces this naturally)
@Service
class OrderService {
    private final OrderRepository repo; // interface!

    @Autowired
    OrderService(OrderRepository repo) { this.repo = repo; }
}
```

**Spring Boot IS DIP in practice.** Spring's IoC container injects the correct implementation at runtime; your service only knows about the interface.

---

### 2.6 Other Key OOP Concepts

#### Coupling & Cohesion
- **Coupling:** Degree to which one class depends on another. Aim for **loose coupling** (via interfaces, DI).
- **Cohesion:** Degree to which a class's methods relate to its single purpose. Aim for **high cohesion** (each class does one thing well).

#### Association, Aggregation, Composition

| Relationship | Description                                         | Lifetime dependency |
|-------------|-----------------------------------------------------|---------------------|
| Association  | A uses B (no ownership)                            | Independent         |
| Aggregation  | A has B (weak ownership, B can exist without A)    | Independent         |
| Composition  | A has B (strong ownership, B dies when A dies)     | Dependent           |

```java
// Association: Employee works at Company (both exist independently)
class Employee { private Company company; }

// Aggregation: Department has Employees (employees can belong to other depts)
class Department { private List<Employee> members; }

// Composition: Order has OrderItems (items don't exist outside an order)
class Order {
    private final List<OrderItem> items = new ArrayList<>(); // created inside
}
```

#### `instanceof` and Type Casting
```java
Animal animal = new Dog();
if (animal instanceof Dog dog) {  // pattern matching (Java 16+)
    dog.bark();
}
```

#### `final` keyword
- `final class` → cannot be extended (e.g., `String`, `Integer`)
- `final method` → cannot be overridden
- `final variable` → value cannot be reassigned (for objects, the reference is immutable, not the object itself)

#### `static` keyword and OOP
- `static` members belong to the class, not instances — they break object-level encapsulation if overused
- `static` methods cannot be overridden (they are hidden in subclasses)
- Anti-pattern: making everything static defeats OOP and makes testing harder

#### Interfaces vs Abstract Classes (Deeper)

```java
// Abstract class: shared state + template method pattern
abstract class ReportGenerator {
    // Template method — defines algorithm skeleton
    public final Report generate(Data data) {
        Data validated = validate(data);      // hook
        List<Row> rows = extractRows(validated); // hook
        return format(rows);                   // hook
    }
    protected abstract Data validate(Data data);
    protected abstract List<Row> extractRows(Data data);
    protected abstract Report format(List<Row> rows);
}

// Interface: capability contract
public interface Auditable {
    default void audit(String action) {
        AuditLog.record(this.getClass().getSimpleName(), action);
    }
}
```

---

## 3. How It Works Internally

### JVM and OOP Internals

#### Object Memory Layout (JVM Heap)
Every Java object in heap memory has:
1. **Object header** (~16 bytes): Mark word (hash, GC flags, lock state) + class pointer (reference to metadata)
2. **Instance fields**: actual data
3. **Padding**: alignment to 8-byte boundary

```
[Mark Word 8B][Class Pointer 4-8B][field1][field2]...[padding]
```

#### Method Dispatch — vtable (Virtual Method Table)
Runtime polymorphism is implemented via the **vtable**:
1. Every class has a vtable — an array of method pointers.
2. When a subclass overrides a method, it replaces the pointer in its vtable.
3. At runtime, the JVM looks up the actual type's vtable to find the right method.

```
Vehicle vtable:
  [0] → Vehicle::accelerate
  [1] → Vehicle::brake

Car vtable:
  [0] → Car::accelerate    ← replaced
  [1] → Vehicle::brake     ← inherited
```

When you call `vehicle.accelerate()`, the JVM follows the reference to the actual object, reads its class metadata, looks up index [0] in the vtable, and calls the right implementation. This is **invokevirtual** bytecode instruction.

#### `invokevirtual` vs `invokeinterface` vs `invokespecial`

| Bytecode          | Used for                                              |
|------------------|-------------------------------------------------------|
| `invokevirtual`  | Instance methods on classes (vtable lookup)           |
| `invokeinterface`| Methods declared in interfaces (itable lookup)        |
| `invokespecial`  | Constructors, `private` methods, `super` method calls |
| `invokestatic`   | Static methods                                        |

#### Class Loading and Inheritance
When the JVM loads `Car`:
1. First loads `Vehicle` (parent first)
2. Merges the vtable of parent into child
3. Replaces entries for overridden methods

#### Spring Proxy and OOP
Spring creates proxy objects around your beans to implement AOP (`@Transactional`, `@Cacheable`, etc.):
- **JDK Dynamic Proxy**: If bean implements an interface → proxy implements the same interface, overrides methods
- **CGLIB Proxy**: If bean is a concrete class → subclass is generated at runtime, overrides methods

This is pure OOP — Spring exploits inheritance/polymorphism to intercept method calls without modifying your source code.

```
Client → [Spring CGLIB Proxy (subclass of OrderService)]
                   ↓
           [OrderService (your code)]

OrderService proxy overrides placeOrder() to:
  1. begin transaction
  2. call super.placeOrder()  ← your actual code
  3. commit/rollback transaction
```

---

## 4. Key APIs — Annotations / Classes / Interfaces / Methods

| Name                  | Type           | Purpose                                                                 |
|-----------------------|----------------|-------------------------------------------------------------------------|
| `Object`              | Class          | Root of all Java classes; provides `equals`, `hashCode`, `toString`, `clone`, `wait`, `notify` |
| `abstract`            | Keyword        | Declare abstract class or method; cannot be instantiated                |
| `interface`           | Keyword        | Define a pure contract                                                  |
| `extends`             | Keyword        | Class inheritance / interface extension                                 |
| `implements`          | Keyword        | A class implementing an interface                                       |
| `super`               | Keyword        | Reference to parent class constructor or method                         |
| `final`               | Keyword        | Prevent extension/override/reassignment                                 |
| `instanceof`          | Operator       | Runtime type check                                                      |
| `@Override`           | Annotation     | Compiler check that a method truly overrides a parent method            |
| `@FunctionalInterface`| Annotation     | Marks an interface as having exactly one abstract method                |
| `Comparable<T>`       | Interface      | Natural ordering via `compareTo`                                        |
| `Cloneable`           | Interface      | Marker interface; enables `Object.clone()`                              |
| `Serializable`        | Interface      | Marker interface; enables Java serialization                            |
| `Enum`                | Class          | Base class for all enums; special form of class                         |
| `record`              | Java 16+       | Immutable data carrier; generates constructor, getters, equals, hashCode, toString |
| `sealed`              | Java 17+       | Restricts which classes can extend/implement                            |
| `@Component`          | Spring Annotation | Registers a class as a Spring-managed bean                           |
| `@Service`            | Spring Annotation | Specialisation of `@Component` for service layer                     |
| `@Repository`         | Spring Annotation | Specialisation of `@Component` for DAO layer; enables exception translation |
| `@Transactional`      | Spring Annotation | Declarative transaction management; applied to methods/classes      |
| `BeanPostProcessor`   | Spring Interface | Hook to customise beans after instantiation                          |
| `ApplicationContext`  | Spring Interface | Central IoC container; extends `BeanFactory`                         |

---

## 5. Configuration & Setup

OOP concepts are language-level and require no special dependencies. For Spring Boot:

**Maven dependency (Spring Boot starter — includes all core Spring):**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

**Lombok (reduces boilerplate for encapsulation):**
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

**Using Lombok for encapsulation:**
```java
@Data           // generates getters, setters, equals, hashCode, toString
@Builder        // builder pattern
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
}
```

**`application.properties` — relevant to OOP-driven Spring configuration:**
```properties
# Bean scope
# Configured via @Scope annotation, not properties

# Lazy initialization (affects singleton bean creation)
spring.main.lazy-initialization=true

# Proxy mode default
spring.aop.proxy-target-class=true  # CGLIB for all beans (not just those implementing interfaces)
```

---

## 6. Real-World Production Code Example

**Domain: E-commerce Order Processing System**

This example demonstrates all four OOP pillars + SOLID in a realistic Spring Boot application.

```java
// ──────────────────────────────────────────────────────
// ABSTRACTION + INTERFACE SEGREGATION
// ──────────────────────────────────────────────────────
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerId(Long customerId);
    Optional<Order> findByOrderNumber(String orderNumber);
}

public interface InventoryService {
    boolean reserve(Long productId, int quantity);
    void release(Long productId, int quantity);
}

public interface PaymentService {
    PaymentResult charge(String customerId, BigDecimal amount, String currency);
    void refund(String paymentIntentId, BigDecimal amount);
}

public interface NotificationService {
    void notify(Long customerId, String message);
}

// ──────────────────────────────────────────────────────
// ENCAPSULATION — Domain Entity
// ──────────────────────────────────────────────────────
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String orderNumber;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();  // COMPOSITION

    private BigDecimal totalAmount;
    private Long customerId;
    private LocalDateTime createdAt;

    // Private constructor — use factory method
    private Order() {}

    // Factory method — controlled creation
    public static Order create(Long customerId) {
        Order order = new Order();
        order.orderNumber = "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.totalAmount = BigDecimal.ZERO;
        return order;
    }

    // Behaviour on the entity — not anemic domain model
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot add items to a non-PENDING order");
        }
        OrderItem item = new OrderItem(this, product, quantity);
        items.add(item);
        recalculateTotal();
    }

    public void confirm() {
        if (items.isEmpty()) throw new IllegalStateException("Order has no items");
        this.status = OrderStatus.CONFIRMED;
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel a " + status + " order");
        }
        this.status = OrderStatus.CANCELLED;
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Only getters exposed — no public setters on business fields
    public Long getId()             { return id; }
    public String getOrderNumber()  { return orderNumber; }
    public OrderStatus getStatus()  { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public Long getCustomerId()     { return customerId; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
}

// ──────────────────────────────────────────────────────
// INHERITANCE — Discount Strategy (OCP + Polymorphism)
// ──────────────────────────────────────────────────────
public interface DiscountStrategy {
    BigDecimal apply(Order order);
    boolean isApplicable(Order order);
}

@Component
public class SeasonalDiscount implements DiscountStrategy {
    private static final BigDecimal RATE = new BigDecimal("0.10");

    @Override
    public BigDecimal apply(Order order) {
        return order.getTotalAmount().multiply(RATE);
    }

    @Override
    public boolean isApplicable(Order order) {
        int month = LocalDate.now().getMonthValue();
        return month == 12 || month == 1; // holiday season
    }
}

@Component
public class LoyaltyDiscount implements DiscountStrategy {
    private final CustomerRepository customerRepository;

    public LoyaltyDiscount(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    @Override
    public BigDecimal apply(Order order) {
        int points = customerRepository.getLoyaltyPoints(order.getCustomerId());
        BigDecimal rate = points > 1000 ? new BigDecimal("0.15") : new BigDecimal("0.05");
        return order.getTotalAmount().multiply(rate);
    }

    @Override
    public boolean isApplicable(Order order) {
        return customerRepository.getLoyaltyPoints(order.getCustomerId()) > 100;
    }
}

// ──────────────────────────────────────────────────────
// SRP — OrderService: orchestrates, doesn't implement everything
// ──────────────────────────────────────────────────────
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    private final List<DiscountStrategy> discountStrategies; // Spring injects all implementations

    @Transactional
    public Order placeOrder(Long customerId, List<OrderItemRequest> itemRequests) {
        Order order = Order.create(customerId);

        // Reserve inventory
        for (OrderItemRequest req : itemRequests) {
            if (!inventoryService.reserve(req.getProductId(), req.getQuantity())) {
                throw new InsufficientInventoryException(req.getProductId());
            }
            order.addItem(productRepository.findById(req.getProductId()).orElseThrow(), req.getQuantity());
        }

        // Apply best available discount (POLYMORPHISM — calls correct isApplicable/apply per type)
        BigDecimal maxDiscount = discountStrategies.stream()
            .filter(strategy -> strategy.isApplicable(order))
            .map(strategy -> strategy.apply(order))
            .max(Comparator.naturalOrder())
            .orElse(BigDecimal.ZERO);

        BigDecimal finalAmount = order.getTotalAmount().subtract(maxDiscount);

        // Charge payment
        PaymentResult result = paymentService.charge(
            String.valueOf(customerId), finalAmount, "USD"
        );
        if (!result.isSuccess()) {
            throw new PaymentFailedException(result.getErrorMessage());
        }

        order.confirm();
        Order saved = orderRepository.save(order);

        notificationService.notify(customerId, "Order " + saved.getOrderNumber() + " placed successfully!");
        return saved;
    }

    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.cancel();
        // Release inventory
        order.getItems().forEach(item ->
            inventoryService.release(item.getProduct().getId(), item.getQuantity())
        );
        orderRepository.save(order);
        notificationService.notify(order.getCustomerId(), "Order " + order.getOrderNumber() + " cancelled.");
    }
}

// ──────────────────────────────────────────────────────
// POLYMORPHISM in notification — different channels
// ──────────────────────────────────────────────────────
@Component
public class EmailNotificationService implements NotificationService {
    private final JavaMailSender mailSender;

    @Override
    public void notify(Long customerId, String message) {
        // send email
    }
}

@Component
@Primary // Primary bean used when injecting NotificationService
public class CompositeNotificationService implements NotificationService {
    private final List<NotificationService> delegates;

    // Spring injects all NotificationService beans — fan-out
    public CompositeNotificationService(
        @Qualifier("emailNotificationService") NotificationService email,
        @Qualifier("smsNotificationService") NotificationService sms
    ) {
        this.delegates = List.of(email, sms);
    }

    @Override
    public void notify(Long customerId, String message) {
        delegates.forEach(d -> d.notify(customerId, message)); // polymorphic dispatch
    }
}
```

---

## 7. Production Use-Cases & Best Practices

### 1. Prefer Constructor Injection Over Field Injection
Spring recommends constructor injection for mandatory dependencies. It makes dependencies explicit, supports immutability (`final`), and makes testing without Spring possible.

```java
// BAD
@Service
public class OrderService {
    @Autowired private OrderRepository repo; // mutable, hidden dependency
}

// GOOD
@Service
public class OrderService {
    private final OrderRepository repo; // immutable, explicit

    public OrderService(OrderRepository repo) { this.repo = repo; }
}
```

### 2. Program to Interfaces, Not Implementations
Always type fields as the interface:

```java
// BAD
private final MySQLOrderRepository repo;

// GOOD
private final OrderRepository repo; // Spring injects the right impl
```

### 3. Domain Model Should Not Be Anemic
An anemic domain model has entities with only getters/setters and all logic in services. This violates encapsulation and OOP. Rich domain models put behaviour on the entity.

```java
// ANEMIC (bad OOP)
order.setStatus(OrderStatus.CONFIRMED); // anyone can set anything

// RICH DOMAIN MODEL (good OOP)
order.confirm(); // validates business rules internally
```

### 4. Use Polymorphism Instead of `instanceof` Chains
```java
// BAD: violates OCP + LSP
if (payment instanceof CreditCardPayment) { ... }
else if (payment instanceof UPIPayment) { ... }

// GOOD: polymorphism
payment.process(); // each type knows how to process itself
```

### 5. Abstract Base Entity for Auditing
```java
@MappedSuperclass
public abstract class BaseEntity {
    @CreatedDate  private LocalDateTime createdAt;
    @LastModifiedDate private LocalDateTime updatedAt;
    @CreatedBy   private String createdBy;
}

@Entity
public class Product extends BaseEntity { ... } // INHERITANCE for cross-cutting state
```

### 6. Strategy Pattern for Interchangeable Algorithms
Use the Strategy pattern (OCP + DIP) for pricing, discounts, routing, sorting. Inject all implementations via `List<StrategyInterface>` in Spring.

### 7. Composition Over Inheritance in Spring Services
Never extend another service class. Use composition (inject the other service).

```java
// BAD
@Service public class AdvancedOrderService extends OrderService { ... }

// GOOD
@Service public class AdvancedOrderService {
    private final OrderService orderService; // composed
    private final FraudDetectionService fraudService;
}
```

### 8. Use Records for DTOs/Value Objects (Java 16+)
```java
public record OrderResponse(String orderNumber, BigDecimal total, OrderStatus status) {}
// Immutable, encapsulated, auto-generates equals/hashCode/toString
```

---

## 8. Scenario Walkthroughs

### Scenario 1: Self-Invocation and Spring AOP Proxy

**Setup:** `OrderService` has two methods: `placeOrder()` and a `@Transactional validateAndPlace()` that internally calls `placeOrder()`.

```java
@Service
public class OrderService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void validateAndPlace(OrderRequest req) {
        validate(req);
        placeOrder(req);  // calling own method directly
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void placeOrder(OrderRequest req) { ... }
}
```

**What happens step by step:**
1. Client calls `orderService.validateAndPlace()` — but `orderService` is a **Spring CGLIB proxy**.
2. The proxy intercepts the call, starts a transaction for `validateAndPlace`.
3. Inside `validateAndPlace`, `this.placeOrder(req)` is called — `this` refers to the **real object**, NOT the proxy.
4. The proxy is bypassed! `placeOrder`'s `REQUIRES_NEW` transaction annotation is completely ignored.
5. `placeOrder` runs in the **same transaction** as `validateAndPlace`.

**Correct approach:** Inject a self-reference or extract `placeOrder` to a separate bean.

```java
// Option 1: Extract to separate service
@Service class OrderPlacementService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void placeOrder(OrderRequest req) { ... }
}
@Service class OrderService {
    @Autowired OrderPlacementService placementService;
    @Transactional
    public void validateAndPlace(OrderRequest req) {
        validate(req);
        placementService.placeOrder(req); // goes through proxy ✅
    }
}
```

**Interviewer's intent:** Tests understanding of Spring's proxy mechanism and how OOP (self-reference bypasses polymorphic dispatch) interacts with AOP.

---

### Scenario 2: LSP Violation Causing Production Bug

**Setup:** `PremiumOrderRepository` extends `StandardOrderRepository` and overrides `findAll()` to only return orders with value > ₹10,000 (to save load).

```java
class StandardOrderRepository {
    List<Order> findAll() { return db.queryAll(); }
}
class PremiumOrderRepository extends StandardOrderRepository {
    @Override
    List<Order> findAll() { return db.queryAll().stream()
        .filter(o -> o.getValue() > 10000).collect(toList()); }
}
```

**What goes wrong:**
- Code consuming `StandardOrderRepository` expects `findAll()` to return ALL orders.
- When `PremiumOrderRepository` is substituted, audit jobs and reconciliation code silently miss low-value orders.
- This is a **Liskov Substitution Principle violation** — the subtype breaks the parent's contract.

**Fix:** Do not override `findAll()` to change its fundamental contract. Add a new method:
```java
class PremiumOrderRepository extends StandardOrderRepository {
    // Do NOT override findAll()
    public List<Order> findHighValueOrders() { ... } // new method
}
```

**Interviewer's intent:** Tests whether candidate recognises LSP violations and understands the real-world consequences.

---

### Scenario 3: Polymorphic Bean Injection in Spring

**Setup:** You have three implementations of `ShippingProvider` — `FedExProvider`, `DHLProvider`, `BlueDartProvider`. The `OrderService` should pick the provider based on order weight.

```java
public interface ShippingProvider {
    boolean supports(double weightKg);
    ShippingQuote getQuote(Order order);
}

@Component public class BlueDartProvider implements ShippingProvider {
    @Override public boolean supports(double w) { return w < 5; }
    @Override public ShippingQuote getQuote(Order order) { ... }
}

@Component public class DHLProvider implements ShippingProvider {
    @Override public boolean supports(double w) { return w >= 5 && w < 20; }
    @Override public ShippingQuote getQuote(Order order) { ... }
}

@Component public class FedExProvider implements ShippingProvider {
    @Override public boolean supports(double w) { return w >= 20; }
    @Override public ShippingQuote getQuote(Order order) { ... }
}

@Service
public class ShippingService {
    private final List<ShippingProvider> providers; // Spring injects ALL implementations

    public ShippingService(List<ShippingProvider> providers) {
        this.providers = providers;
    }

    public ShippingQuote getBestQuote(Order order, double weightKg) {
        return providers.stream()
            .filter(p -> p.supports(weightKg))    // polymorphic filter
            .map(p -> p.getQuote(order))           // polymorphic call
            .min(Comparator.comparing(ShippingQuote::getPrice))
            .orElseThrow(() -> new NoProviderException(weightKg));
    }
}
// Adding a new provider = new class only. ShippingService never changes. — OCP ✅
```

**Interviewer's intent:** Tests practical use of polymorphism + OCP + Spring DI together.

---

## 9. Common Pitfalls & Gotchas

### 1. `equals()` / `hashCode()` Not Overridden
**Problem:** Two `Order` objects with the same ID are not equal when added to a `Set` or used as `Map` keys.  
**Why:** `Object.equals()` compares references by default.  
**Fix:** Always override `equals()` and `hashCode()` using business key (ID or natural key). Use `@EqualsAndHashCode` from Lombok or `@Override` manually.

---

### 2. Mutable Subclass of Immutable Parent
**Problem:** Parent `Money` is immutable. Subclass `MutableMoney` overrides `add()` to mutate state.  
**Why:** This violates LSP — code treating `Money` as immutable is broken.  
**Fix:** Mark immutable classes `final` to prevent extension.

---

### 3. Overriding `static` Methods
**Problem:** Developer thinks they are overriding `static` methods but is actually *hiding* them.  
**Why:** `static` methods belong to the class, not the instance — vtable lookup is not used.  
**Example:**
```java
class Parent { static void greet() { System.out.println("Parent"); } }
class Child extends Parent { static void greet() { System.out.println("Child"); } }

Parent p = new Child();
p.greet(); // prints "Parent" — NOT "Child"! Static methods are resolved at compile time.
```

---

### 4. Calling Overridable Methods in Constructors
**Problem:** Parent constructor calls `init()` which is overridden by subclass. The subclass's `init()` runs before subclass fields are initialised.

```java
class Parent {
    Parent() { init(); } // BAD: calling overridable method
    void init() { System.out.println("Parent init"); }
}
class Child extends Parent {
    private String name = "Alice";
    @Override void init() { System.out.println(name); } // prints null!
}
```

**Fix:** Never call overridable methods in constructors. Use `final` methods or factory methods.

---

### 5. Spring Bean is Singleton — OOP State Caution
**Problem:** Developers add instance state to a `@Service` (e.g., a counter, a buffer) forgetting it's a singleton shared across all threads.  
**Fix:** Spring beans should be stateless. Any state goes in the method-local variables or request-scoped beans.

---

### 6. Field Injection Breaks Testability
**Problem:** `@Autowired` on a field means you cannot create the object without Spring context in tests.  
**Fix:** Constructor injection — allows `new OrderService(mockRepo)` in unit tests.

---

### 7. Covariant Return Type Confusion
```java
class Animal { Animal create() { return new Animal(); } }
class Dog extends Animal {
    @Override Dog create() { return new Dog(); } // covariant — OK
}
```
This is allowed and is a valid override. But the covariant return type must be a **subtype**.

---

### 8. Forgetting `@Override` Leads to Overloading Instead of Overriding
```java
class Animal {
    public boolean equals(Object o) { ... }
}
class Dog extends Animal {
    public boolean equals(Dog d) { ... } // OVERLOAD, not OVERRIDE — different parameter!
    // equals(Object) is NOT overridden — bugs in Collections
}
```

Always use `@Override` — the compiler will catch this mistake.

---

### 9. `instanceof` Before Java 16 — Verbose and Error-Prone
```java
// Pre-Java 16
if (animal instanceof Dog) {
    Dog dog = (Dog) animal; // redundant cast
    dog.bark();
}

// Java 16+ Pattern Matching — cleaner
if (animal instanceof Dog dog) {
    dog.bark();
}
```

---

### 10. Tight Coupling via `new` Inside Methods
```java
@Service class OrderService {
    public void place(Order o) {
        EmailSender sender = new EmailSender(); // tight coupling!
        sender.send("Order placed");
    }
}
```
**Fix:** Inject `EmailSender` (or its interface) via DI. Never instantiate collaborators with `new` inside services.

---

## 10. Miscellaneous & Advanced Details

### Java Records (Java 16+)
Records are a concise, immutable value-carrier class. They automatically generate:
- Canonical constructor
- Getters (no `get` prefix: `record.name()`)
- `equals()`, `hashCode()`, `toString()`

```java
public record ProductDTO(Long id, String name, BigDecimal price) {}
// Immutable, final, perfect for DTOs in Spring MVC responses
```

Records **cannot** extend classes (only implement interfaces). They are implicitly `final`.

---

### Sealed Classes (Java 17+)
Restrict which classes can extend a class/interface:

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public final class Circle    implements Shape { ... }
public final class Rectangle implements Shape { ... }
public final class Triangle  implements Shape { ... }
// No other class can implement Shape
```

Useful with pattern matching switch (Java 21):
```java
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
```

---

### Marker Interfaces
Interfaces with no methods — used to signal a capability to the JVM or frameworks:
- `Serializable` — JVM serialization
- `Cloneable` — enables `Object.clone()`
- `Remote` (RMI)

Modern Java prefers annotations over marker interfaces.

---

### Default Methods in Interfaces (Java 8+)
Allows backward-compatible extension of interfaces:
```java
public interface Validator<T> {
    void validate(T obj) throws ValidationException;

    default boolean isValid(T obj) {
        try { validate(obj); return true; }
        catch (ValidationException e) { return false; }
    }
}
// Existing implementations don't need to change
```

**Diamond problem with default methods:** If two interfaces provide conflicting defaults, the implementing class must override:
```java
class MyClass implements A, B {
    @Override public void hello() { A.super.hello(); } // must resolve explicitly
}
```

---

### Java 21 Virtual Threads and OOP
Virtual threads (`Thread.ofVirtual()`) do not change OOP semantics — they change scheduling. Your classes and objects remain the same. The implication for Spring:
- `@Async` can be configured to use a virtual thread executor
- Blocking I/O in services is no longer a scalability concern

---

### Covariance and Contravariance in Generics
```java
// Covariant: List<? extends Animal> — can read, cannot write
List<? extends Animal> animals = new ArrayList<Dog>();
Animal a = animals.get(0); // OK
// animals.add(new Cat()); // COMPILE ERROR

// Contravariant: List<? super Dog> — can write, cannot safely read
List<? super Dog> dogs = new ArrayList<Animal>();
dogs.add(new Dog()); // OK
// Dog d = dogs.get(0); // COMPILE ERROR (returns Object)
```

---

### Spring's Use of OOP Patterns

| Pattern               | Where Used in Spring                                         |
|-----------------------|--------------------------------------------------------------|
| Proxy                 | `@Transactional`, `@Cacheable`, `@Async`, AOP               |
| Template Method       | `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`       |
| Factory               | `BeanFactory`, `ApplicationContext`                         |
| Singleton             | Default bean scope                                          |
| Observer/Event        | `ApplicationEvent`, `@EventListener`                        |
| Strategy              | `HandlerMapping`, `MessageConverter`, `DiscountStrategy`    |
| Decorator             | `FilterChain` in Spring Security                            |
| Chain of Responsibility| `FilterChain`, `HandlerInterceptor`                        |
| Composite             | Composite `NotificationService` (shown above)               |

---

## 11. Interview Questions — Standard

**Q1. What are the four pillars of OOP? Give a Spring example for each.**

**A:** Encapsulation (private fields with public methods — Spring entity/bean), Abstraction (interfaces like `JpaRepository` hide SQL), Inheritance (Spring's `@RestController` extends `@Controller`, entity extends `BaseEntity`), Polymorphism (Spring injects different implementations of an interface — `@Transactional` proxy overrides your methods at runtime).

---

**Q2. What is the difference between method overloading and method overriding?**

**A:** Overloading is compile-time polymorphism — same method name, different parameters, within the same class. The compiler resolves which method to call based on the argument types at compile time. Overriding is runtime polymorphism — a subclass provides its own implementation of a parent class method with the same signature. The JVM resolves which implementation to call at runtime via the vtable. `@Override` annotation should always be used when overriding to ensure correctness; it has no effect on overloading.

---

**Q3. Can we override a `private` or `static` method? What is method hiding?**

**A:** No, `private` methods cannot be overridden because they are not visible to subclasses. `static` methods cannot be overridden either — if a subclass defines a static method with the same signature, it **hides** (not overrides) the parent's method. Method hiding means the method called is determined by the reference type at compile time, not the actual object type at runtime. This means `Parent p = new Child(); p.staticMethod()` calls `Parent.staticMethod()` even though `p` refers to a `Child` instance.

---

**Q4. What is the `IS-A` vs `HAS-A` relationship? When do you use each?**

**A:** IS-A is modelled via inheritance (`extends`/`implements`) and means one type is a specific kind of another. HAS-A is modelled via composition (a field referencing another object) and means one type contains another. Prefer composition over inheritance unless a true IS-A relationship exists. Inheritance creates tight coupling — a change in the parent can break all children. Composition is more flexible; you can replace a component without changing the container. Rule of thumb: if "X IS-A Y" is true now but might not be true in the future (due to evolving requirements), use composition.

---

**Q5. Explain the `equals()` and `hashCode()` contract.**

**A:** If `a.equals(b)` is `true`, then `a.hashCode()` must equal `b.hashCode()`. The reverse is not required — two objects can have the same hash code but not be equal (hash collision). Failing to override both methods together breaks all collections that use hashing (`HashMap`, `HashSet`, `Hashtable`). For example, if you put an object in a `HashMap` and override `equals()` but not `hashCode()`, the object may not be found on lookup because it will be placed in a different bucket than expected. Always override both together. Java 7+ provides `Objects.equals()` and `Objects.hash()` utilities.

---

**Q6. What is the difference between abstract class and interface? When would you choose one over the other?**

**A:** An abstract class can have constructors, instance fields, and both abstract and concrete methods. An interface (Java 8+) can only have abstract methods, `default` methods, `static` methods, and `public static final` constants. A class can extend only one abstract class but implement many interfaces. Choose an abstract class when you want to share common implementation and state among closely related types (template method pattern). Choose an interface when you want to define a contract/capability that unrelated classes can fulfil. In Spring, almost every abstraction is an interface — `ApplicationContext`, `JpaRepository`, `MessageConverter` — because it allows multiple unrelated implementations to be injected.

---

**Q7. What is the Liskov Substitution Principle? Give a real example of its violation.**

**A:** LSP states that anywhere a parent type is used, a subtype must work correctly without the caller knowing the difference. A classic violation: `Stack extends Vector` in Java — `Stack` inherits all `Vector` methods including `add(index, element)` and `set(index, element)`, which allow inserting/modifying elements at arbitrary positions, violating the LIFO contract of a stack. Another example: a `ReadOnlyFile extends File` that overrides `write()` to throw `UnsupportedOperationException` — any code that calls `file.write()` expecting it to work will fail. The fix: don't extend if the subtype cannot honour the parent's full contract; use interfaces or composition instead.

---

**Q8. How does Spring's proxy mechanism relate to OOP?**

**A:** Spring AOP creates proxy objects that extend (CGLIB) or implement (JDK Dynamic Proxy) your beans. The proxy overrides/intercepts methods annotated with `@Transactional`, `@Cacheable`, `@Async`, etc. This is runtime polymorphism in action — the caller holds a reference typed as `OrderService` (or its interface), but the actual object is a subclass/proxy generated by Spring at startup. This proxy pattern allows cross-cutting concerns to be applied declaratively without modifying the business logic. The implication is that `self-invocation` (calling a method on `this`) bypasses the proxy because it doesn't go through the polymorphic dispatch chain.

---

**Q9. What is the difference between `Comparable` and `Comparator`?**

**A:** `Comparable<T>` defines the *natural ordering* of a class — implemented by the class itself via `compareTo(T other)`. It bakes the ordering into the class. `Comparator<T>` is an external comparison strategy — a separate object or lambda that defines ordering independently of the class. Use `Comparable` when there is one obvious natural ordering (e.g., numbers, dates). Use `Comparator` when you need multiple orderings (sort products by price, by name, by date) or when you cannot modify the class. In Java 8+, `Comparator.comparing()` makes this concise: `Comparator.comparing(Product::getPrice).thenComparing(Product::getName)`.

---

**Q10. What is the difference between composition and aggregation?**

**A:** Both are HAS-A relationships. In **composition**, the child object's lifecycle is owned by the parent — if the parent is destroyed, so are the children. Example: `Order` contains `OrderItems`; deleting an `Order` deletes all its `OrderItems` (reflected in JPA via `cascade = ALL, orphanRemoval = true`). In **aggregation**, the child can exist independently of the parent. Example: a `Department` has `Employees`, but employees can exist without the department. The distinction matters in domain modelling, database design (foreign key with ON DELETE CASCADE vs without), and JPA mapping.

---

**Q11. What is the SOLID principle most commonly violated in Spring Boot applications?**

**A:** SRP and DIP are most commonly violated. SRP is violated when service classes grow to hundreds of lines handling multiple concerns (validation, business logic, persistence, notifications). DIP is violated when services create dependencies with `new` instead of injecting interfaces. OCP is violated when `if-else` or `switch` chains handle type-specific logic that should be polymorphic strategies. ISP is violated with fat repository interfaces that expose methods the consumer never uses.

---

**Q12. What is covariant return type?**

**A:** A covariant return type allows an overriding method to return a more specific type than the parent method. Introduced in Java 5. This is valid:

```java
class Animal { Animal create() { return new Animal(); } }
class Dog extends Animal { @Override Dog create() { return new Dog(); } }
```

It maintains polymorphism — `Dog.create()` can be used wherever `Animal.create()` is expected — while being more specific in the subclass. Widely used in builder patterns and factory methods.

---

## 12. Interview Questions — Tricky / Scenario-Based

**Q1. An interviewer says: "You have a `@Service` class. Can I add an instance variable to track how many requests it has processed?" What do you say?**

**Trap:** Interviewee says "yes" without thinking.  
**Answer:** No, not safely. Spring `@Service` beans are **singletons** by default — shared across all HTTP threads. An unsynchronised instance variable is a race condition. If you need a counter, use `AtomicLong` or Micrometer metrics, or keep the state in a database/cache. Spring beans should be **stateless**; state goes in method-local variables, the database, or thread-safe utilities. This is an OOP design question masked as a Spring question.

---

**Q2. Why does `@Transactional` not work when a method calls another `@Transactional` method in the same class?**

**Trap:** Candidate knows about self-invocation but cannot explain why.  
**Answer:** Spring wraps your bean in a proxy. All calls from external callers go through the proxy, which starts/joins transactions. When `methodA` calls `this.methodB()`, the call bypasses the proxy entirely — `this` is the raw object. The proxy has no chance to intercept. This is fundamentally an OOP behaviour: `this` inside an instance refers to the actual object, not any wrapper around it. Fix: extract `methodB` to a separate bean, or use `@Autowired`-self injection (inject a reference to the same bean so calls go through the proxy).

---

**Q3. You override `equals()` in your `User` entity but not `hashCode()`. What breaks?**

**Trap:** Candidate says "nothing serious."  
**Answer:** The `equals`/`hashCode` contract is broken. If two `User` objects with the same ID are equal (via overridden `equals`), but their `hashCode` values differ (from `Object`), placing them in a `HashSet` will result in **both being stored** (they land in different buckets). A `HashMap` lookup will fail to find a key even though `equals` returns true. Hibernate's first-level cache also relies on `equals`/`hashCode` for entity identity — broken contract leads to duplicate entities in `Set`-based relationships. Always override both together.

---

**Q4. A class implements two interfaces that both define a `default` method with the same signature. What happens?**

**Trap:** Candidate doesn't know about the diamond problem with default methods.  
**Answer:** A compile error — the compiler cannot decide which default implementation to use. The class **must** override the method and explicitly resolve the conflict:

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override public void hello() {
        A.super.hello(); // explicitly choosing A's default
    }
}
```

If the method is not overridden, it's a compile-time error. This is Java's answer to the C++ diamond problem.

---

**Q5. Can a `final` class implement an interface?**

**Trap:** Candidates confuse "final" with "cannot have contracts."  
**Answer:** Yes, absolutely. `final` only prevents a class from being **extended** (subclassed). It can still implement any number of interfaces. Example: `String` is `final` and implements `Serializable`, `Comparable<String>`, `CharSequence`. `Integer` is `final` and implements `Comparable<Integer>`, `Number`. Making a class `final` is a deliberate API decision: the designer says "this type's behaviour is sealed; you cannot specialise it." It doesn't restrict the class's own interface implementations.

---

**Q6. What is the difference between hiding and shadowing in Java?**

**Trap:** Candidates know overriding but not hiding/shadowing.  
**Answer:**  
- **Method hiding** (static methods): A subclass defines a `static` method with the same signature as a parent's `static` method. The subclass method "hides" the parent's. Which one is called depends on the **reference type** at compile time, not the runtime type. It's not polymorphism.
- **Variable shadowing**: A local variable or parameter has the same name as an instance field. The local variable takes precedence within its scope. Classic bug: constructor parameter `name` shadows field `name` — use `this.name = name`.
- **Variable hiding**: A subclass field has the same name as a parent field. The subclass field hides (not overrides) the parent field. Accessed by casting: `((Parent) child).field`.

---

**Q7. Can an abstract class have a constructor? If it can't be instantiated, why would you need one?**

**Trap:** Candidates say "no" or don't know the purpose.  
**Answer:** Yes, abstract classes can and frequently do have constructors — they just cannot be invoked with `new AbstractClass()`. The constructor is called via `super(...)` from the subclass constructor. The purpose is to initialise the **common state** defined in the abstract class. Example:

```java
abstract class BaseRepository<T> {
    private final EntityManager em;
    protected BaseRepository(EntityManager em) { this.em = em; }
    protected EntityManager getEm() { return em; }
}
class OrderRepository extends BaseRepository<Order> {
    OrderRepository(EntityManager em) { super(em); } // calls abstract class constructor
}
```

---

**Q8. You have `List<Object> list = new ArrayList<String>()`. Does this compile?**

**Trap:** Candidates confuse generics with polymorphism.  
**Answer:** **No, it does not compile.** This is a common mistake. Even though `String IS-A Object`, `ArrayList<String>` is **NOT** an `ArrayList<Object>`. Java generics are **invariant** by default — there is no covariance between parameterised types. If this were allowed, you could do `list.add(new Integer(5))` which would break type safety. To allow this, use wildcards: `List<? extends Object> list = new ArrayList<String>()` (read-only) or `List<? super String> list = new ArrayList<Object>()` (write-only for strings). This is related to Liskov but at the generic type level.

---

**Q9. Is it possible to achieve multiple inheritance in Java? How does Spring take advantage of this?**

**Answer:** Java classes support only single inheritance (one `extends`). However, a class can implement **multiple interfaces**, achieving multiple-type inheritance. Java 8+ default methods give interfaces concrete behaviour, making this a form of multiple inheritance of implementation. Spring extensively uses this: `JpaRepository` extends `PagingAndSortingRepository` which extends `CrudRepository` which extends `Repository` — all interfaces. A `SimpleJpaRepository` implements all of them in one class. Spring beans can implement multiple capability interfaces (`BeanNameAware`, `InitializingBean`, `DisposableBean`, `ApplicationContextAware`) simultaneously — each adds a specific capability without restricting the class hierarchy.

---

**Q10. What happens if a subclass tries to override a `@Transactional` method with a non-`@Transactional` override?**

**Trap:** Candidates assume the annotation is inherited.  
**Answer:** Spring reads `@Transactional` from the actual method being invoked on the proxy. If the subclass overrides the method **without** `@Transactional`, the overriding method does not inherit the annotation automatically (Spring checks the method's declared annotations, not the parent's). The transaction will **not be started** for calls through the subclass. Always re-annotate `@Transactional` if you override a transactional method. Exception: if calling through an interface and the `@Transactional` is on the interface method, it will be picked up regardless.

---

## 13. Quick Revision Summary

- **Encapsulation**: Private fields + public methods. Prevents invalid state. Spring beans should be stateless singletons.
- **Abstraction**: Interface = contract. Abstract class = shared template + state. Spring's power comes from programming to interfaces.
- **Inheritance**: Single class inheritance, multiple interface implementation. Prefer composition over inheritance when in doubt.
- **Polymorphism**: Overloading = compile-time (method signature). Overriding = runtime (vtable). `@Transactional` proxy is pure runtime polymorphism.
- **SOLID**: SRP → one class, one job. OCP → extend, don't modify. LSP → subtypes keep parent contracts. ISP → small focused interfaces. DIP → depend on abstractions (Spring DI enforces this).
- **Spring Proxy** intercepts method calls via subclassing (CGLIB) or interface implementation (JDK). `this.method()` bypasses the proxy — key source of `@Transactional` bugs.
- **`equals()` + `hashCode()` contract**: always override both together. Violating it breaks all hash-based collections and Hibernate entity identity.
- **Generics are invariant**: `List<Dog>` is NOT a `List<Animal>`. Use `? extends` (covariant, read) or `? super` (contravariant, write).
- **`static` methods are hidden, not overridden**. Reference type (compile-time) determines which static method is called.
- **Abstract classes can have constructors** — invoked via `super()` from subclass constructors to initialise common state.
- **Records (Java 16+)** are immutable, final data carriers. Perfect for DTOs, value objects, and Spring MVC response/request objects.
- **Sealed classes (Java 17+)** restrict extension to a defined set of subtypes — enables exhaustive pattern matching.
