# Java: Pass by Value vs Pass by Reference

---

## 1. The One Rule

> **Java is strictly pass-by-value. Always. No exceptions.**

This is one of the most misunderstood statements in Java. The confusion arises because *what is passed* differs between primitives and objects — but the *mechanism* is always the same: **a copy of the value is handed to the method**.

---

## 2. Primitives — Simple Copy

When a primitive (`int`, `long`, `double`, `boolean`, `char`, `byte`, `short`, `float`) is passed to a method, the method receives a **copy of the bits**. Modifying it inside the method has zero effect on the caller.

```java
void doubleIt(int x) {
    x = x * 2;         // modifies the local copy only
}

int a = 5;
doubleIt(a);
System.out.println(a); // still 5
```

**Memory picture:**

```
Caller stack frame:      Method stack frame:
┌────────────┐           ┌────────────┐
│  a = 5     │  ──copy──►│  x = 5     │
└────────────┘           │  x = 10    │  (x changes, a is unaffected)
                         └────────────┘
```

---

## 3. Objects — The Reference Is Copied, Not the Object

When an object is passed, the method receives a **copy of the reference** (the memory address pointing to the object on the heap). Both the caller and the method now hold two separate variables that point to the **same object**.

This means:
- Mutating the object's **state** through the copied reference **IS** visible to the caller.
- Reassigning the reference variable itself is **NOT** visible to the caller.

### 3.1 Mutating State — Visible to Caller

```java
void addItem(List<String> list) {
    list.add("hello");      // modifies the object both references point to
}

List<String> names = new ArrayList<>();
addItem(names);
System.out.println(names); // [hello] — the caller sees the change
```

**Memory picture:**

```
Caller stack frame:          Heap:
┌──────────────────┐         ┌──────────────┐
│ names = ref@100  │──────►  │ ArrayList@100│
└──────────────────┘    ┌──► │ [ ]          │
                        │    └──────────────┘
Method stack frame:     │
┌──────────────────┐    │
│ list  = ref@100  │────┘    list.add("hello") mutates the same object
└──────────────────┘
```

### 3.2 Reassigning the Reference — NOT Visible to Caller

```java
void reassign(List<String> list) {
    list = new ArrayList<>();   // only the local copy of the reference changes
    list.add("world");
}

List<String> names = new ArrayList<>();
names.add("hello");
reassign(names);
System.out.println(names); // [hello] — caller's reference unchanged
```

**Memory picture:**

```
Caller stack frame:          Heap:
┌──────────────────┐         ┌──────────────┐
│ names = ref@100  │──────►  │ ArrayList@100│
└──────────────────┘         │ ["hello"]    │
                             └──────────────┘
Method stack frame (after reassignment):
┌──────────────────┐         ┌──────────────┐
│ list  = ref@200  │──────►  │ ArrayList@200│  ← brand new object
└──────────────────┘         │ ["world"]    │    caller never sees this
                             └──────────────┘
```

---

## 4. Strings — The Immutability Trap

`String` is an object, so its reference is passed by value. But `String` is **immutable** — every "modification" (concat, replace, trim, etc.) creates a **new String object** rather than mutating the existing one. This makes it look like pass-by-value for primitives.

```java
void shout(String s) {
    s = s.toUpperCase();   // creates a new String, rebinds local 's'
}

String name = "alice";
shout(name);
System.out.println(name); // "alice" — completely unchanged
```

```java
void appendSuffix(String s) {
    s = s + "_v2";         // s += "_v2" is also a reassignment, not mutation
}

String id = "user123";
appendSuffix(id);
System.out.println(id);   // "user123"
```

**Rule of thumb:** Because `String` is immutable, it *behaves* as if it were a primitive — you can never modify the caller's string through a method parameter.

---

## 5. Wrapper Types — Same as String

`Integer`, `Long`, `Double`, `Boolean`, etc. are also immutable. All arithmetic operations produce new objects.

```java
void increment(Integer n) {
    n = n + 1;  // unboxes → adds 1 → boxes into new Integer → rebinds local n
}

Integer count = 10;
increment(count);
System.out.println(count); // 10 — unchanged
```

**Production gotcha:** This is why you **cannot** use an `Integer` field as a mutable counter shared across method calls without returning the result or wrapping it in a mutable container.

---

## 6. Arrays — Mutable, But Reference Is Still Copied

Arrays are objects. Their contents can be mutated through the passed reference, but the array variable itself cannot be made to point to a new array.

```java
void fillFirst(int[] arr) {
    arr[0] = 99;       // mutates the array — caller sees this
}

void replaceArray(int[] arr) {
    arr = new int[]{1, 2, 3};  // rebinds local reference — caller does NOT see this
}

int[] data = {0, 0, 0};
fillFirst(data);
System.out.println(data[0]);  // 99

replaceArray(data);
System.out.println(data[0]);  // still 99
```

---

## 7. Edge Cases

### 7.1 Swapping Two Variables

A classic interview trap — swapping two object references inside a method does not work:

```java
void swap(StringBuilder a, StringBuilder b) {
    StringBuilder temp = a;
    a = b;
    b = temp;
    // only the local copies of the references are swapped
}

StringBuilder x = new StringBuilder("X");
StringBuilder y = new StringBuilder("Y");
swap(x, y);
System.out.println(x); // "X" — unchanged
System.out.println(y); // "Y" — unchanged
```

To actually swap, you must swap the *contents* of the objects:

```java
void swapContents(StringBuilder a, StringBuilder b) {
    String temp = a.toString();
    a.replace(0, a.length(), b.toString());
    b.replace(0, b.length(), temp);
}
```

---

### 7.2 Mutable vs. Immutable Object Parameter

```java
// Mutable — caller sees mutations
void appendTo(StringBuilder sb) {
    sb.append(" World");
}
StringBuilder s = new StringBuilder("Hello");
appendTo(s);
System.out.println(s); // "Hello World"

// Immutable — caller sees nothing
void appendTo(String s) {
    s = s + " World";  // new String object, local rebind
}
String t = "Hello";
appendTo(t);
System.out.println(t); // "Hello"
```

---

### 7.3 Varargs

Varargs (`int... nums`) are compiled to an array. The same rules apply — array contents can be mutated, but the array reference cannot be replaced.

```java
void zeroOut(int... nums) {
    for (int i = 0; i < nums.length; i++) nums[i] = 0;
}

int[] data = {1, 2, 3};
zeroOut(data);
System.out.println(Arrays.toString(data)); // [0, 0, 0] — mutated
```

---

### 7.4 Generic Collections and Type Erasure

Type parameters do not change pass-by-value semantics. Adding to a `List<T>` inside a method is always visible to the caller; reassigning the list variable is never visible.

```java
<T> void addDefault(List<T> list, T item) {
    list.add(item);   // visible — mutation through shared reference
}
```

---

### 7.5 `final` Parameter Keyword

`final` on a method parameter prevents *reassignment of the local reference* inside the method. It does **not** make the object immutable.

```java
void process(final List<String> list) {
    list.add("ok");          // ALLOWED — mutating the object
    list = new ArrayList<>(); // COMPILE ERROR — cannot reassign final parameter
}
```

`final` on a parameter is a coding style choice (prevents accidental reassignment) and has no effect on the caller.

---

### 7.6 Nested Object Mutation (Deep vs. Shallow)

When an object holds references to other objects, passing the outer object lets you mutate the inner objects as well:

```java
class Order {
    List<String> items = new ArrayList<>();
}

void addItem(Order order, String item) {
    order.items.add(item);  // mutates the inner list — visible to caller
}

Order o = new Order();
addItem(o, "laptop");
System.out.println(o.items); // [laptop]
```

**Defensive copy pattern** — used to prevent callers from mutating internal state:

```java
class ImmutableOrder {
    private final List<String> items;

    ImmutableOrder(List<String> items) {
        this.items = new ArrayList<>(items); // defensive copy — own internal list
    }

    List<String> getItems() {
        return Collections.unmodifiableList(items); // prevent mutation via getter
    }
}
```

---

## 8. Production Use Cases

### 8.1 Returning Multiple Values from a Method

Since you cannot modify primitive parameters to "return" values through them, use:

**Option A: Return a wrapper/record (preferred in modern Java):**
```java
record MinMax(int min, int max) {}

MinMax findMinMax(int[] arr) {
    // compute...
    return new MinMax(min, max);
}
```

**Option B: Mutate a passed-in result object:**
```java
class Result {
    int min, max;
}

void findMinMax(int[] arr, Result out) {
    out.min = ...; out.max = ...;
}
```

**Option C: Single-element array hack (common in lambda contexts):**
```java
int[] counter = {0};
list.forEach(item -> counter[0]++);  // array content is mutable — works inside lambda
```

---

### 8.2 Lambdas and Effectively Final Variables

Variables captured by lambdas/anonymous classes must be **effectively final** (never reassigned after initialization). Because of pass-by-value, even if they were not final, a lambda capturing a local primitive would get a snapshot — not a live reference.

```java
// Compile error — i is not effectively final (it is reassigned)
int i = 0;
Runnable r = () -> System.out.println(i); // ERROR if i is mutated later
i++;

// Workaround: use AtomicInteger for a mutable counter inside lambdas
AtomicInteger count = new AtomicInteger(0);
list.forEach(item -> count.incrementAndGet()); // AtomicInteger is mutable object
```

---

### 8.3 Thread Safety — Shared Mutable State

Two threads can hold copies of a reference to the same object. Both can mutate it concurrently — this is the root of many race conditions.

```java
List<String> sharedList = new ArrayList<>();

// Thread A
sharedList.add("A");

// Thread B (concurrent) — ConcurrentModificationException or data corruption
sharedList.add("B");
```

Fix: use `Collections.synchronizedList()`, `CopyOnWriteArrayList`, or `ConcurrentLinkedQueue`.

**Key insight:** Pass-by-value copies the reference — it does **not** copy the object or provide any thread safety.

---

### 8.4 Defensive Copies in APIs

When your class exposes a mutable object (like a `Date` or a `List`), callers can mutate your internal state if you return the raw reference:

```java
// UNSAFE — caller can modify internal date
class Event {
    private Date startTime;
    public Date getStartTime() { return startTime; } // leaks internal reference
}

// SAFE — return a defensive copy
class Event {
    private Date startTime;
    public Date getStartTime() { return new Date(startTime.getTime()); }
}
```

Modern practice: Use `LocalDate`/`LocalDateTime` (immutable by design) or `Instant` instead of `java.util.Date`.

---

### 8.5 Builder Pattern and Method Chaining

Builders mutate a shared builder object and return `this`:

```java
public Builder name(String name) {
    this.name = name;  // mutates the builder object — caller sees this
    return this;       // returns the same reference for chaining
}
```

This works precisely because the caller and the method share a reference to the same builder object.

---

### 8.6 Spring / Framework Dependency Injection

Spring injects beans by passing their references. Because references are shared, all components holding the same singleton bean reference are operating on the same object:

```java
@Service
class OrderService {
    private final OrderRepository repo; // Spring passes the same singleton reference to all dependents

    OrderService(OrderRepository repo) { this.repo = repo; }
}
```

This is safe for stateless beans. It becomes dangerous when a **prototype-scoped** (stateful) bean is injected into a **singleton-scoped** bean — the singleton holds one copy of the prototype reference forever, defeating the prototype scope.

---

### 8.7 Stream API and Mutation

Streams are designed for functional, non-mutating operations. Mutating an external collection from inside a stream pipeline is **explicitly prohibited** by the Stream contract (interference) and unsafe:

```java
List<String> source = new ArrayList<>(List.of("a", "b", "c"));

// WRONG — modifying source while iterating it
source.stream().filter(s -> {
    source.remove(s);   // ConcurrentModificationException
    return true;
}).collect(Collectors.toList());

// CORRECT — collect to a new list
List<String> result = source.stream()
    .filter(s -> !s.equals("b"))
    .collect(Collectors.toList());
```

---

## 9. Summary Table

| Passed value | What the method receives | Can method mutate caller's data? | Can method rebind caller's variable? |
|---|---|---|---|
| `int`, `long`, etc. (primitive) | Copy of the bits | No | No |
| `String` | Copy of reference → immutable object | No (immutable) | No |
| `Integer`, `Long`, etc. (wrapper) | Copy of reference → immutable object | No (immutable) | No |
| Mutable object (`List`, `StringBuilder`, custom POJO) | Copy of reference → same object | **Yes** (via the shared reference) | No |
| Array | Copy of reference → same array | **Yes** (element assignment) | No |

---

## 10. Interview Questions

**Q: Is Java pass-by-reference?**
No. Java is always pass-by-value. For objects, the value that is passed is a copy of the reference (memory address). The object itself is not copied.

**Q: If I pass an object and modify it inside the method, why does the caller see the change?**
Because both the caller's variable and the method's parameter hold copies of the *same reference* pointing to the *same object on the heap*. Mutations go through that shared reference to the same object.

**Q: Can I write a swap method in Java?**
You cannot swap the caller's two reference variables through a method (it would require pass-by-reference). You can swap the *contents* of two mutable objects passed to the method.

**Q: Why can't I use an `int` parameter to "return" a result from a method?**
Because the method only receives a copy of the int's value. Modifying that copy does not affect the caller's variable. Use a return value, an output object, an array wrapper, or an `AtomicInteger` instead.

**Q: What is a defensive copy and why is it needed?**
A defensive copy is a new object created from another to avoid sharing the reference. It is used in constructors and getters of value objects to prevent callers from mutating internal state — important for correctness and immutability guarantees.

**Q: How does pass-by-value interact with `final` parameters?**
`final` on a parameter prevents reassigning the local reference variable inside the method. It does not make the referenced object immutable, and it has absolutely no effect on the caller.
