# Java Serialization

---

## 1. Overview

**Serialization** is the process of converting an object's state into a byte stream so it can be:
- Persisted to disk or a database.
- Transmitted over a network (RMI, sockets, message queues).
- Cached externally (Redis, Memcached).
- Deep-copied in memory.

**Deserialization** is the reverse: reconstructing an object from the byte stream.

Java provides built-in serialization via `java.io.Serializable`. However, Java's native serialization is widely considered **dangerous and obsolete** in modern systems. Production systems overwhelmingly use JSON (Jackson), binary formats (Protobuf, Avro, Kryo), or custom serializers. Understanding native serialization is still essential for interviews and for debugging legacy systems.

```
Object (heap) ──► ObjectOutputStream ──► byte[] ──► file / network / cache

byte[] ──► ObjectInputStream ──► Object (heap)
```

---

## 2. `java.io.Serializable`

`Serializable` is a **marker interface** — it has no methods. Its sole purpose is to signal to the JVM that objects of this class may be serialized.

```java
import java.io.Serializable;

public class Product implements Serializable {

    private static final long serialVersionUID = 1L;

    private Long id;
    private String name;
    private double price;
    private transient String cachedDescription; // excluded from serialization

    // constructors, getters, setters
}
```

If you attempt to serialize an object whose class does not implement `Serializable`, an `ObjectOutputStream.writeObject()` call throws `java.io.NotSerializableException`.

---

## 3. `serialVersionUID`

### 3.1 What It Is

`serialVersionUID` is a `static final long` field that acts as a **version fingerprint** for the serialized form of a class. It is embedded in the byte stream during serialization and checked during deserialization.

```java
private static final long serialVersionUID = 1L;
```

### 3.2 Why It Exists

When you deserialize a byte stream, Java checks that the `serialVersionUID` in the stream matches the `serialVersionUID` of the class currently loaded in the JVM. If they differ:

```
java.io.InvalidClassException: com.example.Product;
    local class incompatible: stream classdesc serialVersionUID = 1, local class serialVersionUID = 2
```

### 3.3 Auto-Generated vs. Explicit

If you don't declare `serialVersionUID`, the JVM generates one at runtime by hashing class metadata: class name, interfaces, fields, methods. **Any change to the class** (adding/removing a field, changing a method signature) changes the hash → deserialization of old data fails immediately.

**Explicit rule of thumb:**
- Always declare it explicitly to control compatibility.
- Increment it (e.g., `1L` → `2L`) when you make a **breaking change** to the serialized form.
- Keep the same value when adding optional fields or making backward-compatible changes.

### 3.4 Backward Compatibility Rules

| Change | Compatible? | Notes |
|--------|------------|-------|
| Add a new field | Yes | New field gets default value when deserializing old stream |
| Remove a field | Yes | Missing field in stream is ignored |
| Change a field's type | **No** | Breaks deserialization |
| Change a field from non-transient to transient | Partial | Field disappears from stream; old data loses value |
| Change class hierarchy | **No** | Class must have same superclass chain |

---

## 4. `transient` Keyword

Fields marked `transient` are excluded from serialization. The field is set to its **default value** (`null`, `0`, `false`) during deserialization.

Use `transient` for:
- Sensitive data (passwords, tokens, keys) — security.
- Derived/computed fields that can be recalculated — efficiency.
- Non-serializable fields (database connections, thread handles, streams) — correctness.
- Caches.

```java
public class UserSession implements Serializable {
    private static final long serialVersionUID = 1L;

    private String username;
    private transient String password;         // security — never serialize
    private transient Connection dbConn;       // not serializable
    private transient int cachedAge;           // derived — recalculate on load

    // Custom readObject to rebuild transient state if needed
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        this.cachedAge = computeAge(); // rebuild after deserialization
    }
}
```

**Static fields are also NOT serialized** — static state belongs to the class, not the instance.

---

## 5. Serialization Mechanics — How It Works

### 5.1 Writing (Serialization)

```java
Product product = new Product(1L, "Laptop", 999.99);

try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("product.ser"))) {
    oos.writeObject(product);
}
```

What happens internally:
1. JVM checks `product`'s class implements `Serializable` (else `NotSerializableException`).
2. JVM writes the stream header (`STREAM_MAGIC`, `STREAM_VERSION`).
3. JVM writes class descriptor: class name, `serialVersionUID`, field names and types.
4. JVM recursively serializes all non-transient, non-static fields.
5. Object references are tracked — if the same object is referenced twice, only one copy is written (object identity is preserved using a **handle table**).

### 5.2 Reading (Deserialization)

```java
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("product.ser"))) {
    Product product = (Product) ois.readObject();
}
```

What happens internally:
1. JVM reads class descriptor, resolves the class, checks `serialVersionUID`.
2. JVM allocates memory for the object **without calling any constructor** (bypasses constructors entirely).
3. JVM reads field values from the stream and injects them directly into the object.
4. `transient` and `static` fields are left at their default values.

> **Critical:** The **constructor is NOT called** during deserialization. Any validation or initialization in the constructor is bypassed. This is a major security concern.

### 5.3 The Handle Table (Object Identity)

The `ObjectOutputStream` maintains a table of all objects already written. If the same object instance is encountered again (via multiple references), it writes a **back reference handle** instead of the full object. This:
- Preserves `==` equality after deserialization.
- Handles circular references without infinite loops.
- Means modifying an object after writing part of it and writing it again produces stale data in the stream.

```java
List<String> list = new ArrayList<>();
list.add("a");
oos.writeObject(list); // writes full list
list.add("b");
oos.writeObject(list); // writes back-reference to same handle — "b" is NOT in stream!
// Both deserialized references point to the same list with only ["a"]
```

**Fix:** Call `oos.reset()` before writing the modified object to clear the handle table.

---

## 6. Custom Serialization

### 6.1 `writeObject` / `readObject`

Override the default serialization by declaring these **private methods** (the JVM finds them via reflection — they are not from an interface):

```java
public class BankAccount implements Serializable {
    private static final long serialVersionUID = 1L;

    private String accountNumber;
    private double balance;
    private transient String encryptedPin; // stored transient; we handle it ourselves

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject(); // serialize normal fields first
        // Manually write encrypted PIN
        String encrypted = encrypt(encryptedPin);
        oos.writeObject(encrypted);
    }

    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject(); // deserialize normal fields first
        // Manually read and decrypt PIN
        String encrypted = (String) ois.readObject();
        this.encryptedPin = decrypt(encrypted);
    }

    private String encrypt(String value) { /* AES encrypt */ return value; }
    private String decrypt(String value) { /* AES decrypt */ return value; }
}
```

### 6.2 `readObjectNoData`

Called when a superclass is added to the hierarchy during deserialization of an old stream that didn't have it:

```java
private void readObjectNoData() throws ObjectStreamException {
    // Set safe defaults when this class wasn't in the original stream
    this.balance = 0.0;
}
```

### 6.3 `writeReplace` / `readResolve`

These allow an object to **substitute itself** during serialization/deserialization:

```java
// writeReplace: substitute a different object to be serialized
protected Object writeReplace() throws ObjectStreamException {
    return new SerializedProxy(this); // serialize a proxy instead of this
}

// readResolve: substitute a different object after deserialization
protected Object readResolve() throws ObjectStreamException {
    return this; // used in Singletons to return the existing instance
}
```

**Singleton pattern — protecting it from deserialization:**

```java
public class AppConfig implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final AppConfig INSTANCE = new AppConfig();

    private AppConfig() {}

    public static AppConfig getInstance() { return INSTANCE; }

    // Without readResolve, deserialization creates a NEW instance, breaking Singleton
    protected Object readResolve() throws ObjectStreamException {
        return INSTANCE; // always return the existing singleton
    }
}
```

---

## 7. `Externalizable` Interface

`Externalizable` extends `Serializable` and gives **complete manual control** over the serialized form. Unlike `Serializable` where the JVM handles everything, `Externalizable` requires you to write and read every field yourself.

```java
public class Order implements Externalizable {

    private Long id;
    private String customerName;
    private List<String> items;

    // Required: public no-arg constructor (called by JVM during deserialization)
    public Order() {}

    public Order(Long id, String customerName, List<String> items) {
        this.id = id;
        this.customerName = customerName;
        this.items = items;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeLong(id);
        out.writeUTF(customerName);
        out.writeInt(items.size());
        for (String item : items) out.writeUTF(item);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        this.id = in.readLong();
        this.customerName = in.readUTF();
        int size = in.readInt();
        this.items = new ArrayList<>(size);
        for (int i = 0; i < size; i++) items.add(in.readUTF());
    }
}
```

### `Serializable` vs `Externalizable`

| | `Serializable` | `Externalizable` |
|---|---|---|
| Methods to implement | None (marker interface) | `writeExternal()`, `readExternal()` |
| Constructor called on deserialization | No | **Yes — public no-arg constructor required** |
| Control over serialized form | Custom via `writeObject`/`readObject` | Complete manual control |
| Performance | Slower (reflection-based) | Faster (no reflection) |
| `transient` keyword respected | Yes | Irrelevant — you control everything |
| Use today | Legacy | Rarely; prefer Jackson/Protobuf |

---

## 8. Inheritance and Serialization

### 8.1 Superclass is `Serializable`

If the superclass implements `Serializable`, all subclasses are automatically serializable. The superclass fields are serialized as part of the object graph.

### 8.2 Superclass is NOT `Serializable`

If the superclass does not implement `Serializable`, during deserialization the JVM calls the **superclass's no-arg constructor** to initialize the superclass portion. If no such constructor exists, deserialization fails with `InvalidClassException`.

```java
class Animal {           // NOT Serializable
    String species;
    Animal() { this.species = "Unknown"; } // required — called during deserialization
    Animal(String s) { this.species = s; }
}

class Dog extends Animal implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
}

// When Dog is deserialized:
// 1. Animal() no-arg constructor is called → species = "Unknown"
// 2. Dog's serialized fields (name) are injected from stream
// The original species value is LOST
```

### 8.3 Subclass Serialization Control

A subclass can prevent serialization even if its superclass is `Serializable`:

```java
public class SensitiveData extends BaseData {
    private void writeObject(ObjectOutputStream oos) throws IOException {
        throw new NotSerializableException("SensitiveData must not be serialized");
    }
}
```

---

## 9. Deep Copy via Serialization

Serializing and immediately deserializing an object graph produces a **deep copy** — every object in the graph is a new instance, sharing no references with the original.

```java
@SuppressWarnings("unchecked")
public static <T extends Serializable> T deepCopy(T original) {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(bos)) {

        oos.writeObject(original);

        try (ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
             ObjectInputStream ois = new ObjectInputStream(bis)) {
            return (T) ois.readObject();
        }
    } catch (IOException | ClassNotFoundException e) {
        throw new RuntimeException("Deep copy failed", e);
    }
}

// Usage
Order original = new Order(1L, "Alice", List.of("Laptop", "Mouse"));
Order copy = deepCopy(original);
// copy is a completely independent object — modifying copy doesn't affect original
```

**Caveats:**
- Slower than manual copying (reflection + I/O overhead).
- All classes in the graph must be `Serializable`.
- For production deep copies, prefer copy constructors, `clone()` (carefully), or MapStruct for DTOs.

---

## 10. Security Concerns

Java's native deserialization is one of the most dangerous attack vectors in the Java ecosystem. **Deserialization of untrusted data = Remote Code Execution (RCE)**.

### 10.1 Why It's Dangerous

During deserialization, the JVM:
1. Instantiates classes (bypassing constructors).
2. Calls `readObject()` on any class in the stream.
3. Does all of this before the application code gets the result.

An attacker can craft a byte stream containing a **gadget chain** — a sequence of `readObject()` calls on classes already in the JVM classpath (Apache Commons Collections, Spring, etc.) that, when triggered, execute arbitrary code.

**Famous attacks:** Apache Commons Collections exploit (CVE-2015-4852), Log4Shell pre-chain, Spring Framework RCE.

### 10.2 Mitigations

**1. Never deserialize untrusted data.** The safest rule.

**2. Use an `ObjectInputFilter` (Java 9+) to whitelist allowed classes:**
```java
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filterInfo -> {
    Class<?> clazz = filterInfo.serialClass();
    if (clazz == null) return ObjectInputFilter.Status.UNDECIDED;
    // Only allow known safe classes
    if (clazz == Product.class || clazz == ArrayList.class) {
        return ObjectInputFilter.Status.ALLOWED;
    }
    return ObjectInputFilter.Status.REJECTED; // reject everything else
});
```

Set a JVM-wide default filter:
```bash
-Djdk.serialFilter=com.example.**;java.util.*;!*
```

**3. Use safer serialization formats** instead of Java native serialization:
- JSON (Jackson) — cannot execute code, only data.
- Protobuf, Avro, Thrift — schema-first, type-safe, no arbitrary class loading.
- Kryo (with class whitelist) — faster binary, but needs whitelisting.

**4. OWASP A08 — Software and Data Integrity Failures** lists deserialization of untrusted data as a top risk.

---

## 11. Modern Alternatives

### 11.1 Jackson (JSON — most common in Spring apps)

```java
ObjectMapper mapper = new ObjectMapper();

// Serialize
String json = mapper.writeValueAsString(product);
// {"id":1,"name":"Laptop","price":999.99}

// Deserialize
Product product = mapper.readValue(json, Product.class);
```

Key annotations:

| Annotation | Purpose |
|---|---|
| `@JsonIgnore` | Exclude a field (like `transient`) |
| `@JsonProperty("product_name")` | Map field to different JSON key |
| `@JsonSerialize` / `@JsonDeserialize` | Custom serializer/deserializer |
| `@JsonInclude(NON_NULL)` | Exclude null fields from output |
| `@JsonTypeInfo` | Polymorphic type info (replaces class descriptors) |

### 11.2 Protocol Buffers (Protobuf)

Schema-first binary format. Extremely compact, fast, language-neutral.

```protobuf
// product.proto
message Product {
    int64 id = 1;
    string name = 2;
    double price = 3;
}
```

```java
Product product = Product.newBuilder()
    .setId(1L)
    .setName("Laptop")
    .setPrice(999.99)
    .build();

byte[] bytes = product.toByteArray();
Product deserialized = Product.parseFrom(bytes);
```

### 11.3 Kryo

Fast binary serialization, often used for Spark, Akka, and caching:

```java
Kryo kryo = new Kryo();
kryo.register(Product.class); // whitelist classes — security

Output output = new Output(new FileOutputStream("product.bin"));
kryo.writeObject(output, product);
output.close();

Input input = new Input(new FileInputStream("product.bin"));
Product deserialized = kryo.readObject(input, Product.class);
```

### Comparison

| Format | Human-readable | Schema required | Size | Speed | Security |
|--------|---------------|-----------------|------|-------|----------|
| Java native | No | No | Large | Slow | **Dangerous** |
| JSON (Jackson) | Yes | No | Medium | Moderate | Safe |
| Protobuf | No | Yes | **Smallest** | **Fastest** | Safe |
| Avro | No | Yes | Small | Fast | Safe |
| Kryo | No | No | Small | Very fast | Safe (with whitelist) |

---

## 12. Common Pitfalls & Gotchas

### No Constructor Called on Deserialization

The biggest surprise: the constructor is bypassed. Any validation, initialization, or side effects in the constructor are skipped. Use `readObject()` to validate after deserialization:

```java
private void readObject(ObjectInputStream ois)
        throws IOException, ClassNotFoundException {
    ois.defaultReadObject();
    if (price < 0) throw new InvalidObjectException("Price cannot be negative");
    if (name == null) throw new InvalidObjectException("Name cannot be null");
}
```

---

### `serialVersionUID` Not Declared → Fragile Class

Without an explicit `serialVersionUID`, any change to the class (even adding a method) generates a new UID, breaking all previously serialized data:

```java
// Serialized with this class:
public class Config implements Serializable {
    private String host;
}

// Later, added a field:
public class Config implements Serializable {
    private String host;
    private int port; // ← auto-generated serialVersionUID changed → old data unreadable
}
```

**Always declare `private static final long serialVersionUID`.**

---

### Singleton Broken by Deserialization

Without `readResolve`, deserializing a Singleton creates a second instance, breaking the pattern and potentially creating inconsistent state. Always add `readResolve()` returning `INSTANCE` to any Singleton that is `Serializable`.

---

### `ArrayList` Serialization and Capacity

`ArrayList`'s internal `elementData` array is declared `transient`. It implements its own `writeObject()` to serialize only the actual elements (not the full capacity). If you extend `ArrayList` with your own `writeObject()` and forget `oos.defaultWriteObject()` + manually handling `elementData`, you'll serialize an empty list.

---

### Serializing Inner Classes

Non-static inner classes hold an implicit reference to their outer class instance. Serializing an inner class instance therefore also serializes the entire outer class — often unintentional and expensive. Always use **static nested classes** for serializable types.

```java
// Dangerous
class Outer implements Serializable {
    class Inner implements Serializable { // implicitly holds reference to Outer
        int value;
    }
}

// Safe
class Outer {
    static class Inner implements Serializable { // no implicit outer reference
        int value;
    }
}
```

---

### `ObjectOutputStream` Handle Table Caching

If you write the same object twice to a stream, the second write is a back-reference. If you modified the object between writes, the modification is NOT in the stream:

```java
List<String> list = new ArrayList<>();
oos.writeObject(list);
list.add("new item");
oos.writeObject(list); // writes a back-reference, NOT the updated list

// Fix: reset the stream to clear handle table
oos.reset();
oos.writeObject(list); // now writes the full updated list
```

---

## 13. Scenario Walkthroughs

### 13.1 Caching a Domain Object in Redis

**Setup:** An e-commerce app caches `Product` objects in Redis using Spring's `RedisTemplate`. Products must survive service restarts.

**Approach 1 — Java serialization (not recommended):**
```java
// Spring RedisTemplate with JdkSerializationRedisSerializer (default)
// Stored as opaque binary — only Java apps can read it, version-sensitive
redisTemplate.opsForValue().set("product:1", product);
```

Problems: binary is opaque, any class change breaks existing cache entries, deserialization RCE risk if Redis is exposed.

**Approach 2 — Jackson JSON (recommended):**
```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        Jackson2JsonRedisSerializer<Object> serializer =
            new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);

        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        template.afterPropertiesSet();
        return template;
    }
}
```

Human-readable, language-neutral, survives class additions.

---

### 13.2 Versioning a Serialized Class Without Breaking Old Data

**Setup:** `Order` objects are serialized to disk for audit logs. You need to add a `currency` field.

```java
// Version 1 — in production
public class Order implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private double amount;
}

// Version 2 — adding a new field
public class Order implements Serializable {
    private static final long serialVersionUID = 1L; // SAME UID — compatible change
    private Long id;
    private double amount;
    private String currency; // new field — old streams set this to null on deserialization

    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        if (currency == null) currency = "USD"; // safe default for old streams
    }
}
```

Keep `serialVersionUID = 1L` — the same. Old `Order` streams deserialize successfully; `currency` gets `null`, then `readObject` applies the default.

---

## 14. Interview Questions — Standard

**Q1: What is `serialVersionUID` and what happens if you don't declare it?**
`serialVersionUID` is a `static final long` field that versions the serialized form of a class. During deserialization, the JVM checks that the UID in the stream matches the UID of the loaded class. If you don't declare it, the JVM auto-generates one by hashing the class structure — any change to the class (adding a field, method, etc.) changes the hash, making all previously serialized data incompatible with `InvalidClassException`.

**Q2: Is the constructor called during deserialization?**
No. The JVM bypasses the constructor entirely and allocates memory directly, then injects field values from the stream using reflection. This means constructor-based validation is skipped — use `readObject()` to validate post-deserialization. If the superclass is non-serializable, its no-arg constructor IS called for the superclass portion.

**Q3: What is the purpose of the `transient` keyword?**
`transient` excludes a field from serialization. When the object is deserialized, the transient field is set to its default value (`null`, `0`, `false`). Used for: sensitive data (passwords), non-serializable fields (connections, threads), and derived/cached fields that can be recalculated via `readObject()`.

**Q4: What is the difference between `Serializable` and `Externalizable`?**
`Serializable` is a marker interface — the JVM handles serialization automatically. `Externalizable` requires implementing `writeExternal()` and `readExternal()` — you control every byte. `Externalizable` is faster (no reflection), but requires a public no-arg constructor and significantly more code. Both are legacy; modern code uses Jackson or Protobuf.

**Q5: How do you protect a Singleton from deserialization creating a second instance?**
Implement `readResolve()` returning the existing singleton instance. The JVM calls `readResolve()` after deserialization and substitutes its return value for the newly created object. The newly created object is then discarded.

**Q6: What are suppressed exceptions in the context of `try-with-resources` and how does it relate to `Closeable` (serialization streams)?**
When `ObjectInputStream.close()` fails after the body throws, the close exception is added as a suppressed exception to the primary exception. This prevents close-failure exceptions from masking the more important body exception.

---

## 15. Interview Questions — Tricky / Scenario-Based

**Q1: Two references point to the same object. You serialize the containing object. After deserialization, are those two references still pointing to the same instance?**
Yes — Java's serialization preserves object identity within a single serialization stream via the handle table. The second reference is written as a back-reference to the same handle. After deserialization, both fields point to the same object instance (`==` returns true).

**Q2: You serialize an `ArrayList`. It implements `Serializable`, but `elementData` is `transient`. Why doesn't serializing an ArrayList produce an empty list?**
`ArrayList` provides a custom `writeObject()` that manually writes the list size and each element, even though `elementData` is transient. The `readObject()` re-creates and populates `elementData` from the manually written data. This is a perfect example of custom serialization to avoid serializing the array's excess capacity while preserving all list contents.

**Q3: A class has `serialVersionUID = 1L`. You rename a field. Old serialized data exists. What happens?**
The old field name in the stream doesn't match the new field name in the class. Java's serialization matches fields by name and type. The renamed field is treated as a new field (gets default value), and the old field in the stream is ignored. Effectively, the data in that field is silently lost. This is a backward compatibility break that `serialVersionUID` alone does not prevent — `serialVersionUID` only controls whether deserialization is attempted, not whether field values map correctly.

**Q4: Why is deserializing untrusted data dangerous? What is a gadget chain?**
Java's `readObject()` is called on every object in the stream before the application code receives control. Attackers craft streams containing objects from commonly available libraries (Apache Commons Collections, Spring, etc.) whose `readObject()` methods chain together to invoke `Runtime.exec()` or similar. This is RCE without any vulnerability in the application code itself — just the presence of vulnerable library classes on the classpath is enough. Mitigation: `ObjectInputFilter` whitelisting, or never using Java native deserialization for untrusted input.

**Q5: You use `ObjectOutputStream` to write the same mutable object twice — before and after modifying it. What do both deserialized objects look like?**
Both are identical and reflect the **first write's state**. The second write emits a back-reference to the first write's handle — it does not write the updated state. Both deserialized references point to the same object with the original pre-modification state. Fix with `oos.reset()` before the second write to clear the handle table.

**Q6: Can a non-serializable class have a serializable subclass? What are the constraints?**
Yes. When the non-serializable superclass is deserialized, the JVM calls its **public or protected no-arg constructor** to initialize the superclass portion. If no such constructor exists, `InvalidClassException` is thrown. The non-serializable superclass's fields are initialized only by that constructor — any state set in a parameterized constructor is lost during deserialization.

---

## 16. Quick Revision Summary

- `Serializable` is a **marker interface** — no methods; just signals the JVM that the class is serializable.
- `serialVersionUID` is a version fingerprint — always declare it explicitly to control compatibility.
- **Constructor is NOT called** during deserialization — validation must be in `readObject()`.
- `transient` fields are excluded; they get default values (`null`, `0`) after deserialization.
- **Static fields are never serialized** — they belong to the class, not the instance.
- `try-with-resources` is the correct way to use `ObjectInputStream`/`ObjectOutputStream`.
- Custom serialization: override private `writeObject()`/`readObject()` to extend default behavior; `Externalizable` for full manual control.
- `readResolve()` is mandatory on Singletons that implement `Serializable` — prevents second instance creation.
- `writeReplace()` lets an object substitute itself with a proxy during serialization (Serialization Proxy Pattern).
- Handle table: same object written twice → second write is a back-reference. Use `oos.reset()` to clear it.
- Non-static inner classes hold an implicit outer reference — always use **static nested classes** for serializable types.
- **Java native deserialization of untrusted data = RCE risk**. Use `ObjectInputFilter` whitelisting or switch to JSON/Protobuf.
- Modern production: use **Jackson** (JSON) or **Protobuf/Avro** (binary) — never Java native serialization for new systems.
