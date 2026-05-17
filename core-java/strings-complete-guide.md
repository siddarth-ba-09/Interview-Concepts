# Java Strings — Complete Guide

---

## Table of Contents

1. [What is a String in Java?](#1-what-is-a-string-in-java)
2. [String Immutability](#2-string-immutability)
3. [String Pool (String Intern Pool)](#3-string-pool-string-intern-pool)
4. [String Memory Management](#4-string-memory-management)
5. [String Creation — new vs Literal](#5-string-creation--new-vs-literal)
6. [String Interning](#6-string-interning)
7. [String Methods — Deep Dive](#7-string-methods--deep-dive)
8. [StringBuilder](#8-stringbuilder)
9. [StringBuffer](#9-stringbuffer)
10. [String vs StringBuilder vs StringBuffer](#10-string-vs-stringbuilder-vs-stringbuffer)
11. [String Concatenation Internals](#11-string-concatenation-internals)
12. [Character Encoding in Strings](#12-character-encoding-in-strings)
13. [Compact Strings (Java 9+)](#13-compact-strings-java-9)
14. [Miscellaneous Concepts](#14-miscellaneous-concepts)
15. [Production Use Cases and Pitfalls](#15-production-use-cases-and-pitfalls)
16. [Interview Questions](#16-interview-questions)
17. [Tricky Questions and Gotchas](#17-tricky-questions-and-gotchas)

---

## 1. What is a String in Java?

- `String` is a **class** in `java.lang` package, not a primitive.
- A String object represents an **immutable sequence of Unicode characters**.
- Because it is in `java.lang`, it is automatically imported in every Java class.
- Internally, before Java 9, characters were stored in a `char[]` (UTF-16). From Java 9+, it uses a `byte[]` with a `coder` field (Compact Strings).

```java
String s = "Hello"; // s is a reference to a String object
```

---

## 2. String Immutability

### What does immutability mean?

Once a `String` object is created, its **content cannot be changed**. Any operation that appears to modify it actually creates a **new String object**.

```java
String s = "Hello";
s = s + " World"; // "Hello" object is unchanged; a new object "Hello World" is created
```

### Why is String immutable?

| Reason | Explanation |
|---|---|
| **String Pool safety** | Multiple references can safely point to the same object |
| **Thread safety** | Immutable objects are inherently thread-safe without synchronization |
| **Security** | Used in class loading, network connections, file paths — mutability would be a security risk |
| **Hashcode caching** | `hashCode()` is computed once and cached, enabling efficient use as HashMap keys |

### How immutability is enforced

```java
public final class String {           // final: cannot be subclassed
    private final byte[] value;       // final: reference cannot change
    private final byte coder;
    private int hash;                 // cached hash
}
```

The `final` on the class prevents subclasses from breaking the contract. The `final` on `value` prevents the array reference from being reassigned, and the array itself is never exposed externally.

---

## 3. String Pool (String Intern Pool)

### What is the String Pool?

The **String Pool** (also called the **String Constant Pool** or **String Intern Pool**) is a special memory region inside the **Heap** that stores a single copy of each distinct string literal.

When the JVM encounters a string literal, it checks the pool:
- **If found**: returns the reference to the existing object.
- **If not found**: creates a new object in the pool and returns its reference.

```java
String a = "Java";
String b = "Java";
System.out.println(a == b); // true — same object in the pool
```

### Location of the String Pool

| JVM Version | Location |
|---|---|
| **Before Java 7** | **PermGen** (Permanent Generation) — fixed size, could cause `OutOfMemoryError: PermGen space` |
| **Java 7+** | **Heap** — subject to GC, dynamically sized, `-XX:StringTableSize` controls the hash table bucket count |

Moving to the Heap allowed interned strings to be **garbage collected** when no longer referenced, solving a major memory leak issue.

### String Pool as a Hash Table

Internally, the pool is a **fixed-size hash table** (array of buckets). The default size is:
- Java 7: 1009 buckets
- Java 8: 60013 buckets
- Java 11+: 65536 buckets

You can tune it: `-XX:StringTableSize=200003` (use a prime number for better distribution).

---

## 4. String Memory Management

### Object layout

```
Heap
├── String Pool (since Java 7)
│   ├── "Hello"   ← literal "Hello" lives here
│   ├── "World"
│   └── "Java"
└── Regular Heap
    └── new String("Hello")  ← new keyword bypasses the pool
```

### GC and String Pool

- String pool entries **can be garbage collected** (Java 7+) once no live references point to them.
- Interned strings have their references held by the pool's internal table, so they survive as long as the pool itself does.
- `WeakReference` semantics: the JVM uses weak references internally so that pool entries can be GC'd.

### Memory impact of large pools

```java
// Anti-pattern: loading 1 million unique strings into the pool
for (String line : lines) {
    line.intern(); // pollutes the pool, increases GC pressure
}
```

---

## 5. String Creation — `new` vs Literal

```java
String s1 = "Hello";           // Uses pool
String s2 = "Hello";           // Same pool object as s1
String s3 = new String("Hello"); // Forces a new heap object (NOT in pool)
String s4 = new String("Hello"); // Another new heap object

System.out.println(s1 == s2);  // true  (same pool reference)
System.out.println(s1 == s3);  // false (different objects)
System.out.println(s3 == s4);  // false (different objects)
System.out.println(s1.equals(s3)); // true (same content)
```

### How many objects are created?

```java
String s = new String("Hello");
```
- If `"Hello"` is not in the pool: **2 objects** — one in the pool (from the literal), one on the heap (from `new`).
- If `"Hello"` is already in the pool: **1 object** — only the heap object.

---

## 6. String Interning

`intern()` manually places a string into the pool and returns the pool reference.

```java
String s1 = new String("Hello");
String s2 = s1.intern();         // puts "Hello" in pool (if not already there), returns pool ref
String s3 = "Hello";             // pool ref

System.out.println(s2 == s3);   // true
System.out.println(s1 == s2);   // false — s1 is the heap object, s2 is the pool ref
```

### When to use `intern()`

- When you have a **large number of repeated strings** (e.g., parsing CSV with repeated column values) and want to reduce memory by sharing objects.
- **Benchmark first**: `intern()` has overhead due to the native hash table lookup.

---

## 7. String Methods — Deep Dive

### Comparison

```java
s1.equals(s2)           // content equality
s1.equalsIgnoreCase(s2) // case-insensitive
s1.compareTo(s2)        // lexicographic: negative, 0, positive
s1.compareToIgnoreCase(s2)
```

### Searching

```java
s.charAt(index)          // char at position
s.indexOf("sub")         // first occurrence, -1 if not found
s.lastIndexOf("sub")
s.contains("sub")        // boolean
s.startsWith("pre")
s.endsWith("suf")
s.matches("regex")       // full match
```

### Transformation

```java
s.toUpperCase()
s.toLowerCase()
s.trim()                 // removes leading/trailing ASCII whitespace (<=\u0020)
s.strip()                // Unicode-aware trim (Java 11+)
s.stripLeading()
s.stripTrailing()
s.replace('a', 'b')      // char replacement
s.replace("old", "new")  // sequence replacement
s.replaceAll("regex", "replacement")
s.replaceFirst("regex", "replacement")
```

### Extraction

```java
s.substring(beginIndex)
s.substring(beginIndex, endIndex) // [begin, end)
s.split("delim")
s.split("delim", limit)           // limit controls number of parts
s.toCharArray()
s.getBytes()
s.getBytes(StandardCharsets.UTF_8)
```

### Building / Joining

```java
String.valueOf(42)               // int/long/double/boolean/char/Object → String
String.format("Hello %s", name)  // formatted string
String.join("-", "a", "b", "c")  // "a-b-c"
String.join(", ", list)          // works with Iterable
```

### Java 11+ Additions

```java
s.isBlank()              // true if empty or only whitespace
s.repeat(3)              // "ab".repeat(3) → "ababab"
s.lines()                // Stream<String> split by line terminators
s.stripIndent()          // removes common leading whitespace (text blocks)
s.translateEscapes()     // interprets escape sequences
```

### Java 15+ Text Blocks

```java
String json = """
        {
            "name": "Java",
            "version": 21
        }
        """;
// Indentation is stripped; trailing newline is included
```

### `hashCode()` contract

`String.hashCode()` is computed as:
$$h = s[0] \cdot 31^{n-1} + s[1] \cdot 31^{n-2} + \ldots + s[n-1]$$

The result is cached after first computation. This makes `String` efficient as a `HashMap` key.

---

## 8. StringBuilder

`StringBuilder` is a **mutable**, **non-thread-safe** sequence of characters. It is the go-to class for building strings dynamically in single-threaded code.

### Internal Structure

```java
// Simplified internals
class StringBuilder {
    byte[] value;  // character storage
    int count;     // current length
    // default initial capacity: 16
}
```

- Starts with capacity **16** (can be overridden in constructor).
- When capacity is exceeded: new array of size `(oldCapacity * 2) + 2` is allocated.
- Content is copied into the new array.

### Common Methods

```java
StringBuilder sb = new StringBuilder();
sb.append("Hello");          // returns `this` — chainable
sb.append(' ');
sb.append("World");
sb.insert(5, ",");           // inserts at index
sb.delete(5, 6);             // deletes [5, 6)
sb.deleteCharAt(5);
sb.replace(0, 5, "Hi");
sb.reverse();
sb.indexOf("World");
sb.charAt(0);
sb.setCharAt(0, 'h');
sb.length();
sb.capacity();
sb.ensureCapacity(100);      // pre-allocate to avoid resizing
sb.trimToSize();             // shrink backing array to current length
String result = sb.toString();
```

### Pre-allocating capacity

```java
// Anti-pattern: no capacity hint
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) sb.append(i);

// Better: estimate and pre-allocate
StringBuilder sb = new StringBuilder(50000);
for (int i = 0; i < 10000; i++) sb.append(i);
```

---

## 9. StringBuffer

`StringBuffer` is a **mutable**, **thread-safe** sequence of characters. It is the older (Java 1.0) synchronized counterpart to `StringBuilder`.

### How Thread Safety is Achieved

Every public method is synchronized on `this`:

```java
public synchronized StringBuffer append(String str) { ... }
public synchronized StringBuffer insert(int offset, String str) { ... }
public synchronized StringBuffer delete(int start, int end) { ... }
```

### API

The API is **identical** to `StringBuilder`. They both extend `AbstractStringBuilder`.

```java
StringBuffer sb = new StringBuffer(64);
sb.append("thread-safe");
sb.reverse();
String result = sb.toString();
```

### When to use StringBuffer

- In **legacy code** or when a mutable string must be shared across threads.
- In **modern code**, prefer `StringBuilder` in single-threaded contexts; use explicit synchronization or `ConcurrentLinkedQueue<String>` + join for multi-threaded string building.

---

## 10. String vs StringBuilder vs StringBuffer

| Feature | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutability | Immutable | Mutable | Mutable |
| Thread Safety | Thread-safe (immutable) | Not thread-safe | Thread-safe (synchronized) |
| Performance | Slowest for repeated concat | Fastest | Slower than SB (sync overhead) |
| Storage | String Pool / Heap | Heap | Heap |
| Since | Java 1.0 | Java 1.5 | Java 1.0 |
| Use case | General, keys, literals | Single-thread string building | Multi-thread string building |

### Performance benchmark intuition

```java
// Worst: O(n²) — each + creates a new String
String result = "";
for (int i = 0; i < 10000; i++) result += i;

// Best: O(n) — single backing array with amortized resizing
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) sb.append(i);
String result = sb.toString();
```

---

## 11. String Concatenation Internals

### Compile-time constant folding

```java
String s = "Hello" + " " + "World";
// Compiler folds to: String s = "Hello World";
// Only ONE string literal in the pool
```

### Runtime concatenation pre-Java 9

```java
String s = a + b + c;
// Compiled to:
new StringBuilder().append(a).append(b).append(c).toString()
```

### Runtime concatenation Java 9+ (invokedynamic)

Java 9 introduced `StringConcatFactory` via `invokedynamic`. The JVM can now pick the optimal strategy at runtime (e.g., directly building the byte array without a `StringBuilder` intermediary). This avoids the overhead of `StringBuilder` construction for simple cases.

```
// Bytecode instruction (Java 9+)
invokedynamic #5, makeConcatWithConstants
```

### The `+` in a loop problem

```java
// BAD — compiler still creates a new StringBuilder per iteration in older JVMs
String result = "";
for (String s : list) {
    result = result + s;  // new StringBuilder each iteration
}

// GOOD
StringBuilder sb = new StringBuilder();
for (String s : list) sb.append(s);
String result = sb.toString();
```

---

## 12. Character Encoding in Strings

- Java `String` uses **UTF-16** encoding internally (before Java 9).
- Most ASCII characters fit in a single `char` (2 bytes).
- Supplementary characters (code points > U+FFFF) require **two `char`s** (a surrogate pair).

```java
String emoji = "😀";
System.out.println(emoji.length());      // 2 (two chars — surrogate pair)
System.out.println(emoji.codePointCount(0, emoji.length())); // 1 (one code point)

// Iterating code points correctly
emoji.codePoints().forEach(cp -> System.out.println(Character.toString(cp)));
```

### Conversion to bytes

```java
byte[] utf8Bytes = "Hello".getBytes(StandardCharsets.UTF_8);
String restored = new String(utf8Bytes, StandardCharsets.UTF_8);
```

**Never use `getBytes()` without a charset** — it uses the platform default charset, which varies by OS.

---

## 13. Compact Strings (Java 9+)

Before Java 9, every `char` took 2 bytes (UTF-16). Since most Western text is Latin-1 (fits in 1 byte), half the space was wasted.

Java 9 introduced **Compact Strings**:

```java
// Simplified
class String {
    byte[] value;
    byte coder; // LATIN1 = 0, UTF16 = 1
}
```

- If all characters fit in Latin-1: stored as 1 byte/char → **50% memory reduction** for typical strings.
- If any character needs UTF-16: stored as 2 bytes/char.
- Transparent to the developer; no API changes.
- Can be disabled with `-XX:-CompactStrings` (not recommended).

---

## 14. Miscellaneous Concepts

### `==` vs `.equals()` vs `.compareTo()`

```java
String a = "test";
String b = new String("test");

a == b           // false: reference comparison
a.equals(b)      // true:  content comparison
a.compareTo(b)   // 0:     lexicographic comparison, 0 means equal
```

### `null` handling

```java
String s = null;
s.length();          // NullPointerException
"hello".equals(s)    // false — safe
s.equals("hello")    // NullPointerException — UNSAFE
Objects.equals(s, "hello") // false — null-safe
```

**Best practice**: use `"literal".equals(variable)` (Yoda condition) or `Objects.equals()`.

### String as a HashMap key

String is the ideal `HashMap` key because:
1. **Immutable** — hash cannot change after insertion, preventing key loss.
2. **Cached hashCode** — computed once, reused.
3. **`equals` is well-defined** — based on content.

### `intern()` and `==` after interning

```java
String a = new String("hello").intern();
String b = "hello";
System.out.println(a == b); // true
```

### String length vs codePointCount

```java
String s = "A😀B";
s.length()                       // 4 — counts chars (UTF-16 code units)
s.codePointCount(0, s.length())  // 3 — counts Unicode code points
```

### `String.format` vs `MessageFormat` vs text blocks

| API | Best for |
|---|---|
| `String.format` / `formatted()` | Simple interpolation, logging |
| `MessageFormat` | Internationalization (i18n) with locale-aware formatting |
| Text blocks (Java 15+) | Multi-line strings (SQL, JSON, HTML) |
| `String.valueOf()` | Safe null → "null" conversion |

### Regular expressions and String

```java
// Compiled pattern (reuse for performance)
Pattern p = Pattern.compile("\\d+");
Matcher m = p.matcher("abc123def456");
while (m.find()) System.out.println(m.group()); // 123, 456

// Avoid: compiles pattern on every call
"abc123".matches("\\d+"); // re-compiles each time
```

### `substring()` memory note (pre-Java 7u6)

Before Java 7u6, `substring()` shared the backing `char[]` of the original string, meaning a small substring could prevent a large string from being GC'd. This was fixed in Java 7u6 — `substring()` now always copies.

---

## 15. Production Use Cases and Pitfalls

### Use Case 1 — Building SQL/JSON dynamically

```java
// BAD: String concatenation with user input → SQL Injection risk
String query = "SELECT * FROM users WHERE name = '" + userInput + "'";

// GOOD: PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, userInput);
```

### Use Case 2 — CSV/log line building

```java
// GOOD: pre-allocated StringBuilder
public String buildCsvLine(List<String> fields) {
    StringBuilder sb = new StringBuilder(fields.size() * 20); // estimate
    for (int i = 0; i < fields.size(); i++) {
        if (i > 0) sb.append(',');
        sb.append(fields.get(i));
    }
    return sb.toString();
}
```

### Use Case 3 — Splitting and parsing

```java
// split() compiles regex every call — cache if called in a loop
private static final Pattern COMMA = Pattern.compile(",");
String[] parts = COMMA.split(line); // faster than line.split(",")
```

### Use Case 4 — Large string processing (streams)

```java
// For very large strings, use streams to avoid loading everything into memory
try (Stream<String> lines = Files.lines(Paths.get("large.txt"), StandardCharsets.UTF_8)) {
    lines.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}
```

### Pitfall 1 — String concatenation in loops

```java
// Creates O(n²) objects
String result = "";
for (int i = 0; i < 100_000; i++) result += i;

// Fix
StringBuilder sb = new StringBuilder(700_000);
for (int i = 0; i < 100_000; i++) sb.append(i);
```

### Pitfall 2 — `split()` with special regex characters

```java
"a.b.c".split(".")    // WRONG: "." is regex "any char" → returns empty array
"a.b.c".split("\\.")  // CORRECT: escaped dot
"a|b|c".split("\\|") // CORRECT: escaped pipe
```

### Pitfall 3 — `substring()` on very large strings

```java
// Even though pre-Java 7u6 issue is fixed, be aware of large copies
String huge = readEntireFile(); // 100 MB
String tiny = huge.substring(0, 10); // copies 10 chars — fine
huge = null; // now the 100 MB can be GC'd
```

### Pitfall 4 — Charset assumptions

```java
// WRONG: platform-dependent
String s = new String(bytes);
byte[] b = s.getBytes();

// CORRECT: explicit charset
String s = new String(bytes, StandardCharsets.UTF_8);
byte[] b = s.getBytes(StandardCharsets.UTF_8);
```

### Pitfall 5 — Using `StringBuffer` unnecessarily

```java
// Unnecessary synchronization overhead in single-threaded code
StringBuffer sb = new StringBuffer(); // don't do this unless you need thread safety
StringBuilder sb = new StringBuilder(); // prefer this
```

### Pitfall 6 — Interning all strings indiscriminately

```java
// DON'T intern everything — pollutes the pool, increases GC scan time
for (String s : hugeList) s.intern(); // bad

// DO intern only high-cardinality repeated strings where memory savings are clear
```

### Pitfall 7 — `trim()` vs `strip()` for Unicode whitespace

```java
String s = "\u2000Hello\u2000"; // Unicode En Quad space
s.trim().equals("Hello")   // FALSE — trim() only removes chars ≤ \u0020
s.strip().equals("Hello")  // TRUE  — strip() uses Character.isWhitespace()
```

### Pitfall 8 — Comparing strings with `==`

```java
// Classic bug — works in some cases due to pool, breaks with new/runtime strings
if (userInput == "admin") { ... }  // WRONG
if ("admin".equals(userInput)) { ... } // CORRECT
```

### Pitfall 9 — `String.format` performance in hot paths

```java
// String.format() is ~5-10x slower than StringBuilder due to regex parsing of format string
// Avoid in tight loops; prefer StringBuilder or use a logging framework's lazy evaluation
log.debug("Processing {} items", count); // SLF4J lazy eval — no concatenation if DEBUG disabled
```

### Pitfall 10 — Text blocks trailing whitespace

```java
String s = """
        Hello   
        World
        """; // trailing spaces on "Hello   " line are preserved — use \s or trim
```

---

## 16. Interview Questions

### Q1. What is the difference between `String`, `StringBuilder`, and `StringBuffer`?

**Answer**: `String` is immutable; any modification creates a new object. `StringBuilder` is mutable and not synchronized — best for single-threaded string manipulation. `StringBuffer` is mutable and synchronized — best when shared across threads. In practice, `StringBuilder` is preferred in modern single-threaded code due to better performance.

---

### Q2. Why is String immutable in Java?

**Answer**: For security (used in class loading, network connections), thread safety (no synchronization needed), String Pool efficiency (safe sharing of literals), and hashCode caching (String is an ideal HashMap key).

---

### Q3. Where is the String Pool located and can strings in it be garbage collected?

**Answer**: Before Java 7, the pool was in PermGen (fixed size, not GC'd easily). From Java 7 onwards, it moved to the **Heap**, allowing interned strings to be garbage collected when they have no live references. This solved common `OutOfMemoryError: PermGen space` issues.

---

### Q4. What does `intern()` do?

**Answer**: It adds the string to the pool (if not already present) and returns the canonical pool reference. After interning, two strings with the same content will have the same reference (`==` will be `true`).

---

### Q5. How many String objects are created by `String s = new String("hello")`?

**Answer**: Potentially **2** — one literal `"hello"` in the pool (if not already there) and one new object on the heap created by `new`. If `"hello"` already exists in the pool, only **1** new object is created on the heap.

---

### Q6. Why is `String` a good HashMap key?

**Answer**: Because it is immutable (hash cannot change after insertion, preventing key loss), its `hashCode()` is cached after first computation, and `equals()` is based on content.

---

### Q7. What is the internal storage of String in Java 9+ vs earlier?

**Answer**: Before Java 9: `char[]` (UTF-16, 2 bytes per char). Java 9+: `byte[]` with a `coder` field. If all characters are Latin-1, 1 byte per char is used (Compact Strings), cutting memory usage roughly in half for typical text.

---

### Q8. What happens when you concatenate strings using `+` in a loop?

**Answer**: Each `+` in a loop body creates a new `StringBuilder`, appends, and calls `toString()`, resulting in O(n²) object creation and copying. The fix is to create one `StringBuilder` outside the loop and `append()` inside.

---

### Q9. What is the difference between `equals()` and `==` for strings?

**Answer**: `==` compares **references** (memory addresses). `equals()` compares **content** (character by character). Two strings with the same content created with `new` will have `==` return `false` but `equals()` return `true`.

---

### Q10. Is `String` thread-safe? Why?

**Answer**: Yes, because it is **immutable**. Since no thread can modify its state after creation, there are no race conditions. No synchronization is required.

---

### Q11. What is `String.format()` and when should you avoid it?

**Answer**: `String.format()` builds a string using a format string with placeholders. It is slower than `StringBuilder` because it parses the format string using regular expressions internally. Avoid it in **tight loops** or **hot code paths**; prefer `StringBuilder` or logging framework lazy evaluation.

---

### Q12. What is the difference between `trim()` and `strip()`?

**Answer**: `trim()` removes characters with code point ≤ `\u0020` (ASCII whitespace). `strip()` (Java 11+) uses `Character.isWhitespace()` which recognizes all Unicode whitespace characters. Prefer `strip()` for modern internationalized code.

---

## 17. Tricky Questions and Gotchas

### T1. Output of this code?

```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = new String("Hello");

System.out.println(s1 == s2);       // true  — same pool reference
System.out.println(s1 == s3);       // false — s3 is heap object
System.out.println(s1.equals(s3));  // true  — same content
System.out.println(s3 == s3.intern()); // true IF "Hello" was already in pool
                                        // (s3.intern() returns pool ref, s3 is heap ref → false)
```

**Correct answer for last line**: `false`. `s3` is on the heap; `s3.intern()` returns the pool reference — these are different objects.

---

### T2. What is the output?

```java
String a = "Hello" + "World";     // compile-time constant folding → "HelloWorld" in pool
String b = "Hello" + "World";
System.out.println(a == b);        // true — same pool object after folding
```

---

### T3. What about this?

```java
String x = "Hello";
String y = "World";
String z = x + y;                  // runtime concatenation — new heap object
String w = "HelloWorld";           // pool object
System.out.println(z == w);        // false — z is a new heap object
```

---

### T4. Final variables and string pool

```java
final String x = "Hello";
final String y = "World";
String z = x + y;                  // compile-time constant (finals) → "HelloWorld" in pool
String w = "HelloWorld";
System.out.println(z == w);        // true — compiler inlines final variables
```

`final` local variables with literal values are treated as **compile-time constants**, so the concatenation is folded.

---

### T5. How long is this string?

```java
String emoji = "😀";
System.out.println(emoji.length()); // 2 — NOT 1!
```

Emoji are supplementary Unicode characters requiring two UTF-16 code units (a surrogate pair). `length()` returns the number of `char`s, not code points.

---

### T6. What does `"".equals(null)` return?

```java
"".equals(null); // false — no NPE, equals() handles null
null == "";      // false
"".equals("");   // true
```

---

### T7. Does this cause a NullPointerException?

```java
String s = null;
String result = "prefix_" + s;    // "prefix_null" — no NPE!
// String + null appends the literal text "null"
```

---

### T8. StringBuilder is not thread-safe — what happens if used across threads?

```java
StringBuilder sb = new StringBuilder();
// Two threads calling sb.append() concurrently can:
// 1. Corrupt the internal array (interleaved writes)
// 2. Get wrong count value
// 3. Throw ArrayIndexOutOfBoundsException during resize
```

Use `StringBuffer` or guard with `synchronized` / `ReentrantLock` if sharing across threads.

---

### T9. What does `String.valueOf(null)` return?

```java
String.valueOf(null);  // NullPointerException!
// The compiler chooses valueOf(char[]) overload, which throws NPE on null
// contrast with:
Object o = null;
String.valueOf(o);     // "null" — safe, uses valueOf(Object) overload
```

---

### T10. Tricky split behavior

```java
"a,,b,,".split(",")     // ["a", "", "b"] — trailing empty strings discarded by default
"a,,b,,".split(",", -1) // ["a", "", "b", "", ""] — negative limit keeps trailing empties
```

---

### T11. `hashCode()` consistency

```java
String s = "Aa";
String t = "BB";
s.hashCode() == t.hashCode(); // true! — hash collision (classic Java example)
// 'A'=65, 'a'=97 → 65*31+97 = 2112
// 'B'=66, 'B'=66 → 66*31+66 = 2112
```

This is why `equals()` must be checked after a hash match.

---

### T12. Can you subclass String?

No. `String` is declared `final`. Attempting to extend it results in a **compile-time error**:

```java
class MyString extends String {} // compile error: cannot inherit from final 'java.lang.String'
```

---

### T13. What happens to the original string in `replace()`?

```java
String s = "Hello";
s.replace('l', 'r'); // returns "Herro" but s is UNCHANGED
System.out.println(s); // "Hello"

// Must assign the result:
s = s.replace('l', 'r');
```

A common beginner mistake — forgetting to assign the result of any String transformation.

---

*Last updated: May 2026*
