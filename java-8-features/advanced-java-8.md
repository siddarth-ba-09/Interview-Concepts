# Advanced Java 8 Concepts

---

## 1. Overview

Java 8 (released March 2014) is the most transformative Java release since Java 5. It brought functional programming constructs into the language without breaking backward compatibility.

**Key additions:**
- Lambda expressions and method references
- Functional interfaces (`java.util.function`)
- Stream API
- `Optional<T>`
- Default and static interface methods
- `CompletableFuture` (async programming)
- New Date/Time API (`java.time`)
- `Collectors` and new `Map` methods

---

## 2. Lambda Expressions

### 2.1 Syntax

A lambda is an anonymous function — a concise way to implement a **functional interface** (an interface with exactly one abstract method).

```
(parameters) -> expression
(parameters) -> { statements; }
```

```java
// Before Java 8
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("Hello"); }
};

// Java 8 lambda
Runnable r = () -> System.out.println("Hello");

// With parameters
Comparator<String> c = (a, b) -> a.compareTo(b);

// With block body
Comparator<String> c2 = (a, b) -> {
    int result = a.compareTo(b);
    return result;
};
```

### 2.2 Variable Capture

Lambdas can capture variables from the enclosing scope, but captured variables must be **effectively final** (never reassigned after first assignment):

```java
String prefix = "Hello"; // effectively final
List<String> names = List.of("Alice", "Bob");
names.forEach(name -> System.out.println(prefix + " " + name)); // OK

prefix = "Hi"; // compile error — makes prefix no longer effectively final
```

**Why effectively final?** The lambda may outlive the method that created it (e.g., passed to another thread). If the variable could change, the lambda might read stale or inconsistent data. Effectively-final captures a snapshot — the value at lambda creation time is copied into the lambda's closure.

**Static and instance fields** are not subject to this restriction — they are accessed via the object reference, not captured by value:

```java
private String prefix = "Hello"; // instance field — mutable from lambda

void process(List<String> names) {
    names.forEach(name -> System.out.println(prefix + " " + name)); // OK
    prefix = "Hi"; // OK — instance field, not captured
}
```

### 2.3 `this` in Lambdas vs Anonymous Classes

Inside a lambda, `this` refers to the **enclosing class instance** — the same as outside the lambda. Inside an anonymous class, `this` refers to the anonymous class instance itself.

```java
class Printer {
    String name = "Printer";

    void demo() {
        Runnable lambda = () -> System.out.println(this.name); // this = Printer
        Runnable anon = new Runnable() {
            String name = "Anon";
            @Override
            public void run() { System.out.println(this.name); } // this = Anon
        };
    }
}
```

---

## 3. Functional Interfaces

A **functional interface** has exactly one abstract method (SAM — Single Abstract Method). `@FunctionalInterface` annotation is optional but recommended — it makes the compiler enforce the SAM constraint.

### 3.1 Core `java.util.function` Interfaces

| Interface | Method | Takes | Returns | Example use |
|-----------|--------|-------|---------|-------------|
| `Function<T,R>` | `R apply(T t)` | T | R | Transform, map |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | T, U | R | Merge two inputs |
| `UnaryOperator<T>` | `T apply(T t)` | T | T (same type) | In-place transform |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | T, T | T | Reduce two values |
| `Predicate<T>` | `boolean test(T t)` | T | boolean | Filter |
| `BiPredicate<T,U>` | `boolean test(T t, U u)` | T, U | boolean | Two-input filter |
| `Consumer<T>` | `void accept(T t)` | T | void | Side effects |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | T, U | void | Side effects |
| `Supplier<T>` | `T get()` | — | T | Lazy value provider |
| `Runnable` | `void run()` | — | void | No-arg no-result task |

Primitive specializations (avoid boxing overhead):

| Interface | Method |
|-----------|--------|
| `IntFunction<R>`, `LongFunction<R>`, `DoubleFunction<R>` | `R apply(int/long/double)` |
| `ToIntFunction<T>`, `ToLongFunction<T>`, `ToDoubleFunction<T>` | `int/long/double applyAsX(T)` |
| `IntUnaryOperator`, `LongUnaryOperator`, `DoubleUnaryOperator` | `int/long/double applyAsX(int/long/double)` |
| `IntConsumer`, `LongConsumer`, `DoubleConsumer` | `void accept(int/long/double)` |
| `IntSupplier`, `LongSupplier`, `DoubleSupplier` | `int/long/double getAsX()` |
| `IntPredicate`, `LongPredicate`, `DoublePredicate` | `boolean test(int/long/double)` |

### 3.2 Composing Functions

```java
Function<Integer, Integer> times2 = x -> x * 2;
Function<Integer, Integer> plus10 = x -> x + 10;

// andThen: apply times2 first, then plus10
Function<Integer, Integer> times2ThenPlus10 = times2.andThen(plus10);
times2ThenPlus10.apply(3); // (3*2) + 10 = 16

// compose: apply plus10 first, then times2
Function<Integer, Integer> plus10ThenTimes2 = times2.compose(plus10);
plus10ThenTimes2.apply(3); // (3+10) * 2 = 26
```

### 3.3 Composing Predicates

```java
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> startsWithA = s -> s.startsWith("A");

Predicate<String> nonEmptyStartsWithA = notEmpty.and(startsWithA);
Predicate<String> emptyOrStartsWithA  = notEmpty.negate().or(startsWithA);
Predicate<String> notStartsWithA      = startsWithA.negate();
Predicate<String> alwaysTrue          = Predicate.not(String::isEmpty); // Java 11
```

### 3.4 Custom Functional Interface

```java
@FunctionalInterface
public interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}

TriFunction<String, Integer, Boolean, String> formatter =
    (name, age, active) -> name + ":" + age + ":" + active;
```

---

## 4. Method References

A shorthand for a lambda that simply calls an existing method. Four types:

| Type | Syntax | Lambda equivalent |
|------|--------|-------------------|
| Static method | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` |
| Instance method on specific instance | `instance::method` | `(args) -> instance.method(args)` |
| Instance method on arbitrary instance of type | `ClassName::instanceMethod` | `(obj, args) -> obj.method(args)` |
| Constructor reference | `ClassName::new` | `(args) -> new ClassName(args)` |

```java
// Static method
Function<String, Integer> parse = Integer::parseInt;
parse.apply("42"); // 42

// Instance method on specific instance
String prefix = "Hello ";
Function<String, String> greet = prefix::concat;
greet.apply("World"); // "Hello World"

// Instance method on arbitrary instance of type
Function<String, String> upper = String::toUpperCase; // (str) -> str.toUpperCase()
Comparator<String> compare = String::compareTo;       // (a, b) -> a.compareTo(b)

// Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<Integer, ArrayList<String>> sized = ArrayList::new; // new ArrayList<>(capacity)
```

---

## 5. Stream API

A **Stream** is a sequence of elements supporting sequential or parallel aggregate operations. Streams are lazy — no work is done until a terminal operation is invoked.

```
Source → [Intermediate Operations] → Terminal Operation
         (lazy, chainable, stateless/stateful)   (eager, triggers execution)
```

### 5.1 Stream Sources

```java
// From collections
List.of("a", "b", "c").stream()
Set.of(1, 2, 3).stream()

// From arrays
Arrays.stream(new int[]{1, 2, 3})
Stream.of("a", "b", "c")

// Infinite streams
Stream.iterate(0, n -> n + 1)            // 0, 1, 2, 3, ...
Stream.iterate(0, n -> n < 100, n -> n+1) // Java 9: with termination predicate
Stream.generate(Math::random)             // infinite random doubles

// Primitive streams
IntStream.range(0, 10)        // [0, 9]
IntStream.rangeClosed(1, 10)  // [1, 10]
LongStream.of(1L, 2L, 3L)
DoubleStream.of(1.1, 2.2)

// From String
"hello world".chars()         // IntStream of char values
Stream.of("a,b,c".split(","))

// From file (lazy line reading)
Files.lines(Path.of("file.txt"), StandardCharsets.UTF_8)
```

### 5.2 Intermediate Operations

Intermediate operations are **lazy** and return a new Stream. They do not execute until a terminal operation is called.

#### `filter(Predicate)`
```java
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
```

#### `map(Function)` — transform elements
```java
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());
```

#### `mapToInt` / `mapToLong` / `mapToDouble` — avoid boxing
```java
int total = orders.stream()
    .mapToInt(Order::getQuantity) // returns IntStream — no Integer boxing
    .sum();
```

#### `flatMap(Function<T, Stream<R>>)` — flatten nested structures
```java
// Each Order has a List<Item>. Get all items across all orders:
List<Item> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream()) // Order → Stream<Item>
    .collect(Collectors.toList());

// Flatten a List<List<String>>
List<List<String>> nested = List.of(List.of("a","b"), List.of("c","d"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream) // List<String> → Stream<String>
    .collect(Collectors.toList()); // ["a","b","c","d"]
```

#### `distinct()` — deduplicate (uses `equals`/`hashCode`)
```java
stream.distinct()
```

#### `sorted()` / `sorted(Comparator)` — stateful, bufferes all elements
```java
stream.sorted()
stream.sorted(Comparator.comparing(User::getName).reversed())
stream.sorted(Comparator.comparing(User::getAge)
    .thenComparing(User::getName))
```

#### `peek(Consumer)` — side-effect without consuming (for debugging)
```java
stream
    .filter(n -> n > 0)
    .peek(n -> System.out.println("After filter: " + n))
    .map(n -> n * 2)
    .peek(n -> System.out.println("After map: " + n))
    .collect(Collectors.toList());
```

#### `limit(n)` / `skip(n)` — pagination
```java
// Page 3, page size 10
stream.skip(20).limit(10)
```

#### `takeWhile(Predicate)` / `dropWhile(Predicate)` — Java 9+
```java
// Take elements while condition holds (stops at first false)
Stream.of(1, 2, 3, 4, 1, 2)
    .takeWhile(n -> n < 4) // [1, 2, 3]

// Drop elements while condition holds, then take the rest
Stream.of(1, 2, 3, 4, 1, 2)
    .dropWhile(n -> n < 4) // [4, 1, 2]
```

### 5.3 Terminal Operations

Terminal operations are **eager** — they trigger the pipeline and produce a result.

#### `collect(Collector)` — most powerful
See Section 6 for deep `Collectors` coverage.

#### `forEach(Consumer)` / `forEachOrdered(Consumer)`
```java
stream.forEach(System.out::println);          // unordered in parallel
stream.forEachOrdered(System.out::println);   // maintains encounter order
```

#### `count()`
```java
long activeUsers = users.stream().filter(User::isActive).count();
```

#### `findFirst()` / `findAny()` — returns `Optional<T>`
```java
Optional<User> first = users.stream()
    .filter(User::isAdmin)
    .findFirst(); // deterministic — first in encounter order

Optional<User> any = users.parallelStream()
    .filter(User::isAdmin)
    .findAny(); // faster for parallel — returns any matching element
```

#### `anyMatch` / `allMatch` / `noneMatch` — short-circuit
```java
boolean hasAdmin  = users.stream().anyMatch(User::isAdmin);   // stops at first match
boolean allActive = users.stream().allMatch(User::isActive);  // stops at first false
boolean noBlocked = users.stream().noneMatch(User::isBlocked);
```

#### `reduce(identity, BinaryOperator)` — fold to single value
```java
// Sum with identity
int sum = IntStream.rangeClosed(1, 10).reduce(0, Integer::sum); // 55

// Product
int product = Stream.of(1, 2, 3, 4).reduce(1, (a, b) -> a * b); // 24

// Without identity — returns Optional (stream may be empty)
Optional<Integer> max = Stream.of(3, 1, 4, 1, 5).reduce(Integer::max); // Optional[5]

// Three-arg reduce for parallel (combiner merges partial results)
int parallelSum = Stream.of(1, 2, 3, 4, 5)
    .parallel()
    .reduce(0, Integer::sum, Integer::sum); // identity, accumulator, combiner
```

#### `min` / `max` — returns `Optional`
```java
Optional<User> youngest = users.stream()
    .min(Comparator.comparing(User::getAge));

Optional<Integer> maxVal = stream.max(Integer::compare);
```

#### `toArray()`
```java
String[] arr = stream.toArray(String[]::new);
Object[] raw = stream.toArray();
```

#### `iterator()` / `spliterator()` — escape hatches
```java
Iterator<String> it = stream.iterator(); // pull-based iteration
```

### 5.4 Lazy Evaluation — How the Pipeline Really Executes

The pipeline is a single pass — each element flows through all operations before the next element is processed:

```java
Stream.of(1, 2, 3, 4, 5)
    .filter(n -> { System.out.println("filter: " + n); return n % 2 == 0; })
    .map(n -> { System.out.println("map: " + n); return n * 10; })
    .findFirst();

// Output:
// filter: 1
// filter: 2      ← 2 passes filter
// map: 2         ← immediately mapped
// (stream stops — findFirst() has its result)
// 3, 4, 5 never processed
```

This is why `limit()` + `findFirst()` on an infinite stream works:
```java
Stream.iterate(0, n -> n + 1)
    .filter(n -> n % 7 == 0)
    .limit(5)
    .forEach(System.out::println); // 0, 7, 14, 21, 28
```

### 5.5 Stateless vs. Stateful Operations

| Stateless | Stateful |
|-----------|----------|
| `filter`, `map`, `flatMap`, `peek` | `distinct`, `sorted`, `limit`, `skip` |
| Process each element independently | Must see all (or many) elements before proceeding |
| Safe for parallel streams | Can significantly reduce parallel performance |

`distinct()` and `sorted()` must buffer elements — `sorted()` buffers everything before outputting any. Avoid them in the middle of long parallel pipelines.

### 5.6 Parallel Streams

```java
// Convert to parallel
list.parallelStream()
stream.parallel()

// Convert back to sequential
parallelStream.sequential()

// Check
stream.isParallel()
```

Parallel streams use the **ForkJoinPool.commonPool()** (shared across the JVM) with `Runtime.getRuntime().availableProcessors() - 1` threads by default.

**Custom thread pool for parallel streams:**
```java
ForkJoinPool customPool = new ForkJoinPool(4);
List<Integer> result = customPool.submit(() ->
    largeList.parallelStream()
             .filter(n -> n % 2 == 0)
             .collect(Collectors.toList())
).get();
customPool.shutdown();
```

**When parallel streams help:**
- Large data sets (> ~10,000 elements as a rough threshold)
- Computationally expensive per-element operations
- No shared mutable state, no I/O in the pipeline

**When parallel streams hurt or are wrong:**
- Small collections (thread overhead > computation)
- Stateful operations (`sorted`, `distinct`) — may require full buffering
- I/O-bound operations (common pool thread starvation)
- Operations with side effects on shared state
- Ordered operations where order matters (`forEachOrdered` kills parallelism benefit)

---

## 6. `Collectors` — Deep Dive

`Collectors` is a factory class for `Collector` implementations.

### 6.1 Collecting to Collections

```java
// List (mutable, not guaranteed ArrayList)
List<String> list = stream.collect(Collectors.toList());

// Guaranteed ArrayList
List<String> arrayList = stream.collect(Collectors.toCollection(ArrayList::new));

// Unmodifiable List (Java 10+)
List<String> immutable = stream.collect(Collectors.toUnmodifiableList());

// Set
Set<String> set = stream.collect(Collectors.toSet());

// Sorted TreeSet
TreeSet<String> sorted = stream.collect(Collectors.toCollection(TreeSet::new));
```

### 6.2 `toMap`

```java
// keyMapper, valueMapper
Map<Long, User> byId = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

// With merge function (handles duplicate keys)
Map<String, Integer> wordCounts = words.stream()
    .collect(Collectors.toMap(
        Function.identity(),
        w -> 1,
        Integer::sum   // merge: add counts for duplicate keys
    ));

// With specific Map implementation
Map<Long, User> linkedMap = users.stream()
    .collect(Collectors.toMap(
        User::getId,
        Function.identity(),
        (existing, duplicate) -> existing, // keep first on duplicate key
        LinkedHashMap::new                 // preserve insertion order
    ));
```

> **Gotcha:** `toMap` throws `IllegalStateException` on duplicate keys unless you provide a merge function.

### 6.3 `groupingBy`

```java
// Group users by department
Map<String, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));

// Group + downstream collector
Map<String, Long> countByDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment, Collectors.counting()));

Map<String, Double> avgAgeByDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment,
        Collectors.averagingInt(User::getAge)));

Map<String, Optional<User>> oldestByDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment,
        Collectors.maxBy(Comparator.comparing(User::getAge))));

// Multi-level grouping (nested)
Map<String, Map<String, List<User>>> byDeptAndRole = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment,
        Collectors.groupingBy(User::getRole)));

// Group to specific Map type
Map<String, List<User>> treeMap = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment,
        TreeMap::new, Collectors.toList()));
```

### 6.4 `partitioningBy`

```java
// Splits into true/false groups — always produces exactly two entries
Map<Boolean, List<User>> activeInactive = users.stream()
    .collect(Collectors.partitioningBy(User::isActive));

List<User> active   = activeInactive.get(true);
List<User> inactive = activeInactive.get(false);

// With downstream
Map<Boolean, Long> counts = users.stream()
    .collect(Collectors.partitioningBy(User::isActive, Collectors.counting()));
```

### 6.5 String Joining

```java
String csv  = stream.collect(Collectors.joining(", "));
String full = stream.collect(Collectors.joining(", ", "[", "]")); // prefix, suffix
// ["Alice", "Bob", "Charlie"]
```

### 6.6 Statistical Collectors

```java
IntSummaryStatistics stats = orders.stream()
    .collect(Collectors.summarizingInt(Order::getQuantity));

stats.getCount();   // number of elements
stats.getSum();
stats.getMin();
stats.getMax();
stats.getAverage();

// Individual stats
long   count = stream.collect(Collectors.counting());
double avg   = stream.collect(Collectors.averagingInt(Order::getQuantity));
int    sum   = stream.collect(Collectors.summingInt(Order::getQuantity));
Optional<Order> max = stream.collect(Collectors.maxBy(Comparator.comparing(Order::getAmount)));
```

### 6.7 `teeing` (Java 12+) — Two Collectors, One Pass

```java
// Compute both min and max in a single stream pass
record MinMax(Optional<Integer> min, Optional<Integer> max) {}

MinMax result = Stream.of(3, 1, 4, 1, 5, 9)
    .collect(Collectors.teeing(
        Collectors.minBy(Integer::compare),
        Collectors.maxBy(Integer::compare),
        MinMax::new
    ));
```

### 6.8 Custom `Collector`

```java
// Collector that collects to an ImmutableList
Collector<String, List<String>, List<String>> toImmutable = Collector.of(
    ArrayList::new,                    // supplier
    List::add,                         // accumulator
    (l1, l2) -> { l1.addAll(l2); return l1; }, // combiner (for parallel)
    Collections::unmodifiableList,     // finisher
    Collector.Characteristics.UNORDERED
);
```

---

## 7. `Optional<T>`

`Optional` is a container that may or may not hold a non-null value. It is a return type signal: "this method might not return a value." It is **not** a general-purpose null replacement.

### 7.1 Creating

```java
Optional<String> present = Optional.of("hello");       // throws NPE if null
Optional<String> nullable = Optional.ofNullable(null); // safe — wraps null as empty
Optional<String> empty    = Optional.empty();
```

### 7.2 Consuming

```java
Optional<User> user = findUser(id);

// Check and get (verbose — prefer the methods below)
if (user.isPresent()) { process(user.get()); }
if (user.isEmpty())   { handleMissing(); }   // Java 11+

// ifPresent — run action only if value present
user.ifPresent(u -> sendEmail(u));

// ifPresentOrElse (Java 9+)
user.ifPresentOrElse(
    u -> process(u),
    () -> log.warn("User not found")
);

// get() — avoid unless you've checked isPresent()
// Throws NoSuchElementException if empty
user.get();
```

### 7.3 Providing Defaults

```java
// orElse — ALWAYS evaluates the argument (even if present)
String name = user.map(User::getName).orElse("Anonymous");

// orElseGet — lazy, only evaluated if empty (prefer for expensive defaults)
String name = user.map(User::getName).orElseGet(() -> generateDefaultName());

// orElseThrow (Java 10+) — default throws NoSuchElementException
User u = user.orElseThrow();

// orElseThrow with custom exception
User u = user.orElseThrow(() -> new UserNotFoundException(id));
```

> **Key difference:** `orElse(value)` evaluates `value` regardless. `orElseGet(supplier)` only calls the supplier if the Optional is empty. For expensive computations (DB calls, network), always use `orElseGet`.

### 7.4 Transforming

```java
// map — transform the value if present
Optional<String> name = user.map(User::getName);
Optional<Integer> nameLength = user.map(User::getName).map(String::length);

// flatMap — when the mapping function itself returns Optional (avoids Optional<Optional<T>>)
Optional<Address> address = user.flatMap(User::getAddress); // User::getAddress returns Optional<Address>

// filter — keep value only if predicate passes
Optional<User> admin = user.filter(User::isAdmin);

// or — provide alternative Optional if empty (Java 9+)
Optional<User> resolved = findInCache(id).or(() -> findInDatabase(id));

// stream — convert Optional to a Stream (0 or 1 elements) — Java 9+
List<String> names = optionalUsers.stream()
    .flatMap(Optional::stream) // each Optional contributes 0 or 1 elements
    .collect(Collectors.toList());
```

### 7.5 Anti-Patterns

```java
// WRONG — Optional as a field (unnecessary overhead, Serializable issues)
class User {
    private Optional<String> middleName; // bad
}

// WRONG — Optional as method parameter (use overloading or @Nullable instead)
void process(Optional<String> name) { } // bad

// WRONG — isPresent() + get() is verbose; use map/orElse
if (opt.isPresent()) return opt.get().toUpperCase(); // verbose
return opt.map(String::toUpperCase).orElse(""); // clean

// WRONG — orElse with a method call (always executes)
User u = findUser(id).orElse(createUser(id)); // createUser always runs!
User u = findUser(id).orElseGet(() -> createUser(id)); // lazy — only if empty
```

---

## 8. Default and Static Interface Methods

### 8.1 Default Methods

Allow interfaces to have method implementations. Enables adding new methods to interfaces without breaking existing implementors.

```java
public interface Auditable {
    Instant getCreatedAt();
    Instant getUpdatedAt();

    // Default implementation — all implementors inherit this
    default boolean isStale(Duration maxAge) {
        return Duration.between(getUpdatedAt(), Instant.now()).compareTo(maxAge) > 0;
    }
}
```

### 8.2 Diamond Problem Resolution

When a class implements two interfaces with the same default method:

```java
interface A { default String hello() { return "A"; } }
interface B { default String hello() { return "B"; } }

class C implements A, B {
    @Override
    public String hello() {
        return A.super.hello(); // explicitly call A's default — required by compiler
    }
}
```

**Priority rules:**
1. Class/superclass method always wins over interface default.
2. More specific interface wins (if one extends the other).
3. Otherwise: compile error — must override and resolve explicitly.

### 8.3 Static Interface Methods

Utility methods that belong to the interface's namespace:

```java
public interface Validator<T> {
    boolean validate(T value);

    static <T> Validator<T> of(Predicate<T> predicate) {
        return predicate::test;
    }

    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }
}

Validator<String> notEmpty  = s -> !s.isEmpty();
Validator<String> notNull   = s -> s != null;
Validator<String> combined  = notNull.and(notEmpty);
```

### 8.4 `Comparator` Default Methods (Key Example)

`Comparator` is the best example of default methods in the JDK:

```java
Comparator<User> byAge = Comparator.comparing(User::getAge);

// reversed()
Comparator<User> byAgeDesc = Comparator.comparing(User::getAge).reversed();

// thenComparing — secondary sort
Comparator<User> byAgeThenName = Comparator.comparing(User::getAge)
    .thenComparing(User::getName);

// nullsFirst / nullsLast
Comparator<User> nullSafe = Comparator.nullsFirst(
    Comparator.comparing(User::getName));

// Comparing with key extractor + comparator
Comparator<User> byNameLength = Comparator.comparing(
    User::getName, Comparator.comparingInt(String::length));
```

---

## 9. `CompletableFuture` — Async Programming

`CompletableFuture<T>` is an implementation of `Future<T>` that supports non-blocking composition of async tasks.

### 9.1 Creating

```java
// Run async task, no result
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> doWork());

// Run async task, with result
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchData());

// Custom thread pool (avoid common pool for I/O)
ExecutorService pool = Executors.newFixedThreadPool(10);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> fetchData(), pool);

// Already-completed future (testing/mocking)
CompletableFuture<String> done = CompletableFuture.completedFuture("result");
```

### 9.2 Transforming the Result

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUserId())        // async: returns Long
    .thenApply(id -> fetchUser(id))          // sync transform: Long → User (runs on same thread)
    .thenApply(user -> user.getName())       // sync: User → String
    .thenApply(String::toUpperCase);

// thenApplyAsync — transform runs on a different thread (from ForkJoinPool or custom pool)
    .thenApplyAsync(String::toUpperCase)
    .thenApplyAsync(String::toUpperCase, customPool)
```

### 9.3 Consuming the Result

```java
cf.thenAccept(result -> System.out.println("Got: " + result)); // Consumer, returns Void CF
cf.thenRun(() -> System.out.println("Done"));                   // Runnable, result ignored
```

### 9.4 Chaining Async Tasks (flatMap equivalent)

```java
// thenCompose — when the next step is itself async (avoids CompletableFuture<CompletableFuture<T>>)
CompletableFuture<Order> order = CompletableFuture
    .supplyAsync(() -> fetchUserId())
    .thenCompose(id -> fetchUserAsync(id))      // returns CompletableFuture<User>
    .thenCompose(user -> fetchOrderAsync(user)); // returns CompletableFuture<Order>
```

| Method | Analogy | Returns |
|--------|---------|---------|
| `thenApply` | `Stream.map` | `CompletableFuture<R>` |
| `thenCompose` | `Stream.flatMap` | `CompletableFuture<R>` (unwrapped) |
| `thenAccept` | `Stream.forEach` | `CompletableFuture<Void>` |

### 9.5 Combining Multiple Futures

```java
CompletableFuture<String> userFuture    = fetchUserAsync(userId);
CompletableFuture<String> productFuture = fetchProductAsync(productId);

// thenCombine — combine results of two independent futures
CompletableFuture<String> combined = userFuture.thenCombine(
    productFuture,
    (user, product) -> user + " bought " + product
);

// allOf — wait for ALL to complete
CompletableFuture<Void> all = CompletableFuture.allOf(
    fetchA(), fetchB(), fetchC()
);
// Collect results after all complete
CompletableFuture<List<String>> results = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0])
).thenApply(v -> futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList()));

// anyOf — complete when ANY one completes (returns Object — loses type safety)
CompletableFuture<Object> first = CompletableFuture.anyOf(fetchA(), fetchB(), fetchC());
```

### 9.6 Error Handling

```java
// exceptionally — recover from failure (like catch)
CompletableFuture<String> cf = fetchDataAsync()
    .exceptionally(ex -> {
        log.error("Fetch failed", ex);
        return "default-value"; // recovery value
    });

// handle — transform result OR exception (runs in both cases)
CompletableFuture<String> cf2 = fetchDataAsync()
    .handle((result, ex) -> {
        if (ex != null) return "fallback";
        return result.toUpperCase();
    });

// whenComplete — side effect on both success and failure (doesn't transform)
cf.whenComplete((result, ex) -> {
    if (ex != null) metrics.recordFailure();
    else metrics.recordSuccess();
});
```

### 9.7 Timeout (Java 9+)

```java
CompletableFuture<String> withTimeout = fetchDataAsync()
    .orTimeout(2, TimeUnit.SECONDS)          // throws TimeoutException after 2s
    .completeOnTimeout("default", 2, TimeUnit.SECONDS); // returns default after 2s
```

### 9.8 Production Pattern — Parallel API Calls

```java
// Call three services in parallel, combine results
CompletableFuture<UserDto>    userFuture    = userService.getUserAsync(userId);
CompletableFuture<List<Order>> ordersFuture = orderService.getOrdersAsync(userId);
CompletableFuture<Address>    addressFuture = addressService.getAddressAsync(userId);

CompletableFuture<UserProfile> profile = userFuture
    .thenCombine(ordersFuture,   (user, orders) -> new PartialProfile(user, orders))
    .thenCombine(addressFuture,  (partial, addr) -> partial.withAddress(addr))
    .thenApply(PartialProfile::build)
    .orTimeout(3, TimeUnit.SECONDS)
    .exceptionally(ex -> UserProfile.empty(userId));
```

---

## 10. New Date/Time API (`java.time`)

Replaces the broken, mutable, non-thread-safe `java.util.Date` and `Calendar`.

### 10.1 Core Classes

| Class | Represents | Has time? | Has timezone? |
|-------|-----------|-----------|---------------|
| `LocalDate` | Date only (2024-05-17) | No | No |
| `LocalTime` | Time only (14:30:00) | Yes | No |
| `LocalDateTime` | Date + time (2024-05-17T14:30) | Yes | No |
| `ZonedDateTime` | Date + time + timezone | Yes | Yes |
| `OffsetDateTime` | Date + time + UTC offset | Yes | Offset only |
| `Instant` | Point on UTC timeline (epoch nanos) | Yes | UTC only |
| `YearMonth` | Year + month | No | No |
| `MonthDay` | Month + day (birthday) | No | No |

### 10.2 Creating

```java
// Current date/time
LocalDate today      = LocalDate.now();
LocalTime now        = LocalTime.now();
LocalDateTime nowDT  = LocalDateTime.now();
ZonedDateTime zonedNow = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));
Instant nowInstant   = Instant.now();

// Specific date/time
LocalDate date = LocalDate.of(2024, Month.MAY, 17);
LocalDate date = LocalDate.of(2024, 5, 17);          // month as int
LocalTime time = LocalTime.of(14, 30, 0);
LocalDateTime dt = LocalDateTime.of(2024, 5, 17, 14, 30);

// Parsing
LocalDate parsed = LocalDate.parse("2024-05-17");
LocalDate custom  = LocalDate.parse("17/05/2024", DateTimeFormatter.ofPattern("dd/MM/yyyy"));
```

### 10.3 Manipulation — Immutable (returns new instance)

```java
LocalDate date = LocalDate.of(2024, 5, 17);

date.plusDays(7);         // 2024-05-24
date.minusMonths(2);      // 2024-03-17
date.withYear(2025);      // 2025-05-17
date.withDayOfMonth(1);   // 2024-05-01

// Adjust to first day of month, last day of month, next Monday
date.with(TemporalAdjusters.firstDayOfMonth());  // 2024-05-01
date.with(TemporalAdjusters.lastDayOfMonth());   // 2024-05-31
date.with(TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY));
date.with(TemporalAdjusters.firstDayOfNextYear());
```

### 10.4 `Duration` vs `Period`

```java
// Duration — time-based (hours, minutes, seconds, nanos)
Duration twoHours   = Duration.ofHours(2);
Duration ninetyMins = Duration.ofMinutes(90);
Duration between    = Duration.between(startTime, endTime);

between.toHours();
between.toMinutes();
between.getSeconds();
between.toMillis();
between.toNanos();

// Period — date-based (years, months, days)
Period twoYears  = Period.ofYears(2);
Period sixMonths = Period.ofMonths(6);
Period between   = Period.between(startDate, endDate);

between.getYears();
between.getMonths();
between.getDays();

// ChronoUnit — compute difference in a single unit
long daysBetween   = ChronoUnit.DAYS.between(start, end);
long monthsBetween = ChronoUnit.MONTHS.between(start, end);
```

### 10.5 Formatting

```java
LocalDate date = LocalDate.of(2024, 5, 17);

// Predefined formatters
date.format(DateTimeFormatter.ISO_LOCAL_DATE);      // "2024-05-17"
date.format(DateTimeFormatter.BASIC_ISO_DATE);      // "20240517"

// Custom pattern
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd MMM yyyy");
date.format(fmt);        // "17 May 2024"

// Locale-aware
DateTimeFormatter localeFmt = DateTimeFormatter
    .ofPattern("d MMMM yyyy", Locale.FRENCH);
date.format(localeFmt);  // "17 mai 2024"

// Thread-safe: DateTimeFormatter instances are immutable — safe to share as static constants
private static final DateTimeFormatter API_DATE_FORMAT =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

### 10.6 Timezone Handling

```java
ZoneId kolkata = ZoneId.of("Asia/Kolkata");
ZoneId utc     = ZoneId.of("UTC");
ZoneId nyc     = ZoneId.of("America/New_York");

// Instant (UTC epoch) → ZonedDateTime in specific zone
Instant instant = Instant.now();
ZonedDateTime inKolkata = instant.atZone(kolkata);
ZonedDateTime inNYC     = instant.atZone(nyc);

// LocalDateTime → ZonedDateTime (attaches timezone)
LocalDateTime localDT = LocalDateTime.of(2024, 5, 17, 10, 0);
ZonedDateTime zoned   = localDT.atZone(kolkata);

// Convert between zones
ZonedDateTime asNYC = zoned.withZoneSameInstant(nyc); // same instant, different display

// Always store as Instant or UTC ZonedDateTime in databases
// Convert to local zone only for display

// Legacy interop
Date legacy = Date.from(instant);
Instant back = legacy.toInstant();
```

### 10.7 Legacy `Date`/`Calendar` Interop

```java
// java.util.Date → Instant
Instant i = legacyDate.toInstant();

// Instant → java.util.Date
Date d = Date.from(instant);

// java.sql.Date ↔ LocalDate
LocalDate ld  = sqlDate.toLocalDate();
java.sql.Date sd = java.sql.Date.valueOf(localDate);

// java.sql.Timestamp ↔ LocalDateTime
LocalDateTime ldt = timestamp.toLocalDateTime();
java.sql.Timestamp ts = java.sql.Timestamp.valueOf(localDateTime);
```

---

## 11. New `Map` Methods (Java 8)

```java
Map<String, Integer> scores = new HashMap<>();

// getOrDefault — avoid null check
int score = scores.getOrDefault("Alice", 0);

// putIfAbsent — only put if key not present (returns existing value)
scores.putIfAbsent("Bob", 100);

// computeIfAbsent — compute and store only if key absent (lazy initialization)
// Returns the (new or existing) value
List<String> list = mapOfLists.computeIfAbsent("key", k -> new ArrayList<>());
list.add("item"); // modify the returned list — it's in the map

// computeIfPresent — update only if key exists
scores.computeIfPresent("Alice", (k, v) -> v + 10);

// compute — compute regardless (return null to remove the entry)
scores.compute("Alice", (k, v) -> (v == null) ? 1 : v + 1); // increment or initialize

// merge — combine existing value with new value; remove entry if result is null
scores.merge("Alice", 1, Integer::sum); // add 1 to Alice's score; initialize to 1 if absent

// replaceAll — transform all values in place
scores.replaceAll((k, v) -> v * 2);

// forEach — iterate entries
scores.forEach((k, v) -> System.out.println(k + "=" + v));
```

**`computeIfAbsent` vs `putIfAbsent`:**
- `putIfAbsent` always evaluates the value argument (even if key exists).
- `computeIfAbsent` only invokes the function if the key is absent — critical for expensive operations like creating collections.

---

## 12. `Stream` API — Advanced Patterns

### 12.1 Building a Frequency Map

```java
Map<String, Long> frequency = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

### 12.2 Finding Duplicates

```java
Set<String> seen = new HashSet<>();
List<String> duplicates = list.stream()
    .filter(e -> !seen.add(e)) // add returns false if already present
    .collect(Collectors.toList());
```

### 12.3 Zipping Two Lists

Java doesn't have a built-in zip, but:
```java
List<String> names  = List.of("Alice", "Bob");
List<Integer> scores = List.of(90, 85);

List<String> zipped = IntStream.range(0, Math.min(names.size(), scores.size()))
    .mapToObj(i -> names.get(i) + ":" + scores.get(i))
    .collect(Collectors.toList()); // ["Alice:90", "Bob:85"]
```

### 12.4 Batching a List into Pages

```java
List<Integer> data = IntStream.rangeClosed(1, 100).boxed().collect(Collectors.toList());
int batchSize = 10;

List<List<Integer>> batches = IntStream.range(0, (data.size() + batchSize - 1) / batchSize)
    .mapToObj(i -> data.subList(i * batchSize, Math.min((i + 1) * batchSize, data.size())))
    .collect(Collectors.toList());
```

### 12.5 Collecting to Multimap

```java
// Map<Department, List<Employee>>
Map<String, List<Employee>> multimap = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Google Guava alternative: ImmutableListMultimap.toImmutableListMultimap(...)
```

---

## 13. Common Pitfalls & Gotchas

### Stream Cannot Be Reused

```java
Stream<String> s = list.stream();
s.forEach(System.out::println);
s.forEach(System.out::println); // IllegalStateException: stream has already been operated upon or closed
```

Create a new stream for each pipeline.

---

### `peek()` Is Not for Side Effects in Production

`peek()` is intended for debugging — its execution depends on whether the terminal operation actually pulls elements. With `findFirst()` or short-circuiting operations, `peek()` may only run on a subset of elements. Never rely on `peek()` for business logic.

---

### `orElse` vs `orElseGet` Performance

```java
// This ALWAYS creates a new User, even if the Optional is present
Optional<User> user = findUser(id).orElse(new User()); // User() always constructed

// This only creates a new User if needed
Optional<User> user = findUser(id).orElseGet(User::new); // User() lazy
```

---

### `reduce` Identity Must Be True Identity

```java
// WRONG — 0 is not the identity for max
int wrong = Stream.of(1, 2, 3).reduce(0, Integer::max); // 3 — happens to work
int wrong = Stream.of(-3, -2, -1).reduce(0, Integer::max); // 0 — wrong! 0 is not in the stream

// CORRECT — use max() with Optional
OptionalInt correct = IntStream.of(-3, -2, -1).max(); // OptionalInt[-1]
```

---

### Parallel Stream on Ordered Source Can Be Slow

```java
// Ordered parallel stream — must maintain encounter order, limiting parallelism
list.parallelStream().forEachOrdered(...); // largely sequential in practice
list.parallelStream().forEach(...);        // unordered, truly parallel
```

---

### `LocalDateTime` Is Not a Point in Time

`LocalDateTime` has no timezone — it's just a date-time representation. Two `LocalDateTime` instances with the same value in different timezones represent different instants. For timestamps, events, and anything stored in a database, use `Instant` (UTC) or `ZonedDateTime`.

---

## 14. Interview Questions — Standard

**Q1: What is a functional interface? Can it have default and static methods?**
A functional interface has exactly one abstract method (SAM). It can have any number of `default` and `static` methods — those do not count toward the abstract method constraint. `@FunctionalInterface` tells the compiler to enforce this. Examples: `Runnable`, `Callable`, `Comparator`, `Function`, `Predicate`.

**Q2: What is the difference between `map()` and `flatMap()` in streams?**
`map()` applies a function to each element, producing a one-to-one transformation: `Stream<T>` → `Stream<R>`. `flatMap()` applies a function that returns a `Stream<R>` for each element, then flattens all resulting streams into one: `Stream<T>` → `Stream<Stream<R>>` → `Stream<R>`. Use `flatMap()` to flatten nested collections or when the mapping function itself returns a stream.

**Q3: What is lazy evaluation in streams and why does it matter?**
Intermediate operations (`filter`, `map`, `flatMap`, etc.) do not execute when called — they build a pipeline description. Execution begins only when a terminal operation (`collect`, `findFirst`, `count`, etc.) is invoked. Each element flows through the entire pipeline before the next element is processed. This enables short-circuiting (e.g., `findFirst()` stops after the first match) and processing infinite streams.

**Q4: What is the difference between `orElse()` and `orElseGet()` in `Optional`?**
`orElse(value)` always evaluates its argument — the value is computed even if the Optional is present. `orElseGet(supplier)` is lazy — the supplier is only invoked when the Optional is empty. For constants, both are equivalent. For expensive operations (DB calls, object construction), always use `orElseGet` to avoid unnecessary work.

**Q5: What is the difference between `thenApply()` and `thenCompose()` in `CompletableFuture`?**
`thenApply()` is like `map()` — it transforms the result synchronously: `T → R`, returning `CompletableFuture<R>`. `thenCompose()` is like `flatMap()` — it takes a function that returns a `CompletableFuture<R>`, and unwraps it. Use `thenCompose` when the next step is itself asynchronous, to avoid `CompletableFuture<CompletableFuture<R>>`.

**Q6: Why is `DateTimeFormatter` thread-safe but `SimpleDateFormat` is not?**
`DateTimeFormatter` is immutable — all state is set at construction time and never modified. `SimpleDateFormat` maintains mutable internal state (a `Calendar` object) that is modified during `format()` and `parse()`, making concurrent access produce incorrect results or throw exceptions. Always use `DateTimeFormatter` (declare as `static final`) and never share `SimpleDateFormat` instances across threads.

---

## 15. Interview Questions — Tricky / Scenario-Based

**Q1: A `Stream` operation throws an exception inside a lambda. What happens?**
The exception propagates out of the terminal operation that triggered the pipeline. For unchecked exceptions, they propagate normally. For checked exceptions, you must wrap them — lambdas in standard functional interfaces (`Function`, `Consumer`, etc.) cannot declare `throws` checked exceptions. The common pattern is to wrap in a `RuntimeException` or use a custom functional interface that declares `throws`.

**Q2: You call `list.stream().filter(x -> x > 0).sorted().findFirst()`. Does `sorted()` sort all elements before `findFirst()` gets one?**
Yes — `sorted()` is a stateful intermediate operation that must consume the entire upstream before emitting any element. It buffers all elements matching `filter()` into a data structure, sorts them, then hands the first element to `findFirst()`. The short-circuit of `findFirst()` only applies after `sorted()` has completed. For large streams where only the minimum/maximum is needed, use `min()`/`max()` instead.

**Q3: What is the difference between `Stream.of(list)` and `list.stream()`?**
`Stream.of(list)` creates a `Stream<List<String>>` — a stream containing a single element (the list itself). `list.stream()` creates a `Stream<String>` — a stream of the list's elements. This is a common mistake when trying to create a stream from a collection.

**Q4: Two interface default methods have the same signature. The class doesn't override it. What happens?**
Compile error: "class inherits unrelated defaults." The class must override the method and explicitly choose which interface's default to call using `InterfaceA.super.method()` syntax. If one interface extends the other, the more specific interface's default wins automatically.

**Q5: `CompletableFuture.allOf()` returns `CompletableFuture<Void>`. How do you get the results of all the individual futures?**
`allOf()` itself doesn't collect results — you hold references to the original futures. After `allOf(...).join()` or in a `thenApply()` on the result, call `.join()` on each original future (they are all complete at this point, so `.join()` returns immediately). Use `futures.stream().map(CompletableFuture::join).collect(Collectors.toList())` to gather all results.

**Q6: You use a parallel stream to insert into a `HashSet`. Is this safe?**
No. `HashSet` is not thread-safe. Concurrent modifications from parallel stream threads will cause data corruption or `ConcurrentModificationException`. Use `Collectors.toSet()` in a terminal `collect()` operation — `Collectors` are designed to handle parallel stream merging correctly.

---

## 16. Quick Revision Summary

- Lambdas capture variables **by value** — captured variables must be effectively final. Instance and static fields are exempt.
- In lambdas, `this` refers to the **enclosing class**. In anonymous classes, `this` refers to the anon class.
- `@FunctionalInterface` — enforces SAM constraint; can have `default` and `static` methods freely.
- `flatMap` = flatten nested streams (one-to-many); `map` = transform one-to-one.
- Streams are **lazy** — nothing executes until a terminal operation. `sorted()` and `distinct()` are stateful and buffer elements.
- `Collectors.toMap()` throws `IllegalStateException` on duplicate keys — always provide a merge function.
- `groupingBy` + downstream collectors enable powerful aggregations in a single pass.
- `Optional.orElse()` always evaluates its argument. `orElseGet()` is lazy — prefer for expensive computations.
- Never use `Optional` as a field, constructor parameter, or method parameter — only as a return type.
- `thenApply` ≈ `map`; `thenCompose` ≈ `flatMap` in `CompletableFuture`. `exceptionally` ≈ `catch`.
- Parallel streams use `ForkJoinPool.commonPool()` — use a custom pool for I/O-bound parallel work.
- `LocalDateTime` has no timezone — use `Instant` or `ZonedDateTime` for timestamps stored in databases.
- `DateTimeFormatter` is **immutable and thread-safe** — declare as `static final`. `SimpleDateFormat` is not.
- `Duration` is time-based (seconds/nanos); `Period` is date-based (years/months/days).
- `Map.computeIfAbsent()` is lazy — only invokes the function if key absent. Critical for initializing nested collections.
