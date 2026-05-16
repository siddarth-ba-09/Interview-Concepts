# JDK, JRE, JVM Architecture & How Java is Compiled

---

## 1. Overview

Java's "Write Once, Run Anywhere" promise is delivered by a three-layer platform:

- **JVM (Java Virtual Machine)** — an abstract machine that executes Java bytecode on any OS/hardware.
- **JRE (Java Runtime Environment)** — JVM + standard class libraries needed to *run* Java programs.
- **JDK (Java Development Kit)** — JRE + compiler (`javac`) + developer tools needed to *develop* Java programs.

> **Java 9+ note:** Oracle stopped shipping a standalone JRE download. `jlink` can produce a custom minimal runtime. The conceptual layering still holds.

---

## 2. How It Works Internally

### 2.1 The Three Layers

```
┌─────────────────────────────────────────┐
│                  JDK                    │  ← Developer tool
│  (javac, javadoc, jar, jdb, jlink...)   │
│  ┌───────────────────────────────────┐  │
│  │              JRE                  │  │  ← Runtime environment
│  │  (class libraries, modules)       │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │           JVM               │  │  │  ← Execution engine
│  │  │  (ClassLoader, JIT, GC...)  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

| Component | What it is | Who needs it |
|-----------|-----------|--------------|
| **JVM** | Abstract machine that executes bytecode | End-user (runtime only) |
| **JRE** | JVM + standard class libraries | End-user running Java apps |
| **JDK** | JRE + compiler (`javac`) + dev tools | Developers |

---

### 2.2 JVM Architecture — Full Breakdown

```
┌──────────────────────────────────────────────────────────┐
│                        JVM                               │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Class Loader Subsystem               │   │
│  │  Bootstrap → Platform → Application              │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │ loads .class files             │
│  ┌──────────────────────▼───────────────────────────┐   │
│  │                 Runtime Data Areas                │   │
│  │                                                   │   │
│  │  ┌────────────┐  ┌──────────┐  ┌──────────────┐  │   │
│  │  │  Method    │  │  Heap    │  │ PC Register  │  │   │
│  │  │  Area      │  │          │  │ (per thread) │  │   │
│  │  │(Metaspace) │  │(Objects) │  └──────────────┘  │   │
│  │  └────────────┘  └──────────┘  ┌──────────────┐  │   │
│  │  ┌────────────────────────┐    │ Native Method│  │   │
│  │  │  Stack (per thread)    │    │ Stack        │  │   │
│  │  │  [Frame][Frame][Frame] │    └──────────────┘  │   │
│  │  └────────────────────────┘                      │   │
│  └──────────────────────┬───────────────────────────┘   │
│                         │                                │
│  ┌──────────────────────▼───────────────────────────┐   │
│  │              Execution Engine                     │   │
│  │   Interpreter → JIT Compiler (C1/C2) → GC        │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │         Native Method Interface (JNI)              │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

### 2.3 Class Loader Subsystem

Three built-in class loaders forming a **parent-delegation chain**:

| Loader | Loads | Parent |
|--------|-------|--------|
| **Bootstrap** | `java.lang.*`, `java.util.*` (JDK core) | None (native C++) |
| **Platform** (was Extension pre-Java 9) | JDK platform modules (`javax.*`, etc.) | Bootstrap |
| **Application** | Your classes + classpath JARs | Platform |

**Parent Delegation Model:**
When `Application` loader is asked to load `com.myapp.Foo`, it first delegates to `Platform`, which delegates to `Bootstrap`. Only if they cannot find it does the child loader handle it itself.

- **Why?** Security — prevents user code from shadowing `java.lang.String` or other core classes.

Three phases per class:

1. **Loading** — reads `.class` bytes from disk/network/URL
2. **Linking**
   - *Verify* — checks bytecode is structurally valid and safe
   - *Prepare* — allocates memory for static fields, sets default values
   - *Resolve* — replaces symbolic references with direct references
3. **Initialization** — runs `<clinit>` (static initializers and static blocks), top-down

> **Key gotcha:** Class loading is **lazy** — a class is loaded only when first referenced at runtime, not at JVM startup. `Class.forName()` loads AND initializes; `ClassLoader.loadClass()` loads but does NOT initialize.

---

### 2.4 Runtime Data Areas (Memory)

#### Method Area / Metaspace
- Stores **class metadata**: class name, superclass, field/method descriptors, bytecode, constant pool, **static variables**.
- **Shared across all threads.**
- Pre-Java 8: **PermGen** — fixed max size, caused frequent `OutOfMemoryError: PermGen space` in app servers with hot-reloading.
- Java 8+: replaced by **Metaspace** — lives in native (off-heap) memory, grows dynamically.
  - Still tunable: `-XX:MaxMetaspaceSize=256m`

#### Heap
- Where **all objects and arrays** live.
- **Shared across all threads** — object allocation is thread-safe via **TLAB (Thread Local Allocation Buffer)**: each thread gets a private chunk of Eden to allocate into without locking.
- Divided for GC purposes:

```
Heap
├── Young Generation
│   ├── Eden Space        ← new objects born here (Minor GC)
│   ├── Survivor S0       ← objects that survived 1 GC
│   └── Survivor S1
└── Old Generation        ← long-lived objects promoted here (Major/Full GC)
```

#### JVM Stack (per thread)
- Each thread has its own private stack — **no sharing, no synchronization needed.**
- Holds **frames** — one frame pushed per method call, popped on return.
- Each frame contains:
  - **Local variable array** — method parameters + local variables
  - **Operand stack** — working area for computation (push/pop operands)
  - **Reference to constant pool** of the declaring class
- `StackOverflowError` = stack depth exceeded (e.g., unbounded recursion)

#### PC Register (per thread)
- Holds the address of the **currently executing JVM instruction**.
- Undefined when executing a native method.

#### Native Method Stack
- Supports native (C/C++) methods called via **JNI (Java Native Interface)**.

---

### 2.5 Execution Engine

#### Interpreter
- Reads and executes bytecode instructions one at a time.
- **Fast startup**, but slow for frequently-called code (re-interprets every time).

#### JIT Compiler (Just-In-Time)
- Monitors execution. When a method crosses the **hot threshold** (~10,000 invocations, tunable via `-XX:CompileThreshold`), it is JIT-compiled to native machine code.
- Compiled native code is cached — subsequent calls run at near-native speed.
- HotSpot JVM has two compilers:

| Compiler | Name | Strategy |
|----------|------|----------|
| **C1** | Client Compiler | Fast compile, light optimizations — good for short-lived apps |
| **C2** | Server Compiler | Slow compile, aggressive optimizations (inlining, loop unrolling, escape analysis) |

- **Tiered Compilation** (default since Java 7): C1 first, promote to C2 for very hot paths.

**Key JIT optimizations:**
- **Method inlining** — replaces method call with the method body (eliminates call overhead)
- **Escape analysis** — if an object doesn't escape a method scope, JIT may allocate it on the stack instead of heap (avoids GC)
- **Loop unrolling** — reduces loop overhead for tight loops
- **Dead code elimination**

> Use `-XX:+PrintCompilation` to see which methods get JIT-compiled at runtime.

#### Garbage Collector
- Reclaims heap memory for objects that are no longer reachable.
- Common GC algorithms:

| GC | Default for | Characteristic |
|----|-------------|----------------|
| Serial | Small single-core apps | Stop-the-world, single-threaded |
| Parallel | Throughput-focused | Multi-threaded GC, stop-the-world |
| G1 | Java 9+ default | Region-based, low pause, concurrent |
| ZGC | Java 15+ production | Sub-millisecond pauses, fully concurrent |
| Shenandoah | RedHat, OpenJDK | Concurrent compaction |

---

## 3. How Java is Compiled — End to End

```
MyApp.java
    │
    │  javac (source compiler)
    ▼
MyApp.class  (platform-neutral bytecode)
    │
    │  java launcher → JVM starts
    ▼
ClassLoader: Loading → Linking → Initialization
    │
    ▼
Interpreter begins executing bytecode
    │
    │  (method called ~10,000 times → "hot")
    ▼
JIT Compiler: bytecode → native machine code (cached)
    │
    ▼
CPU executes native code directly
```

### Phase 1 — `javac`: Source → Bytecode

```bash
javac MyApp.java   # produces MyApp.class
```

Stages inside `javac`:
1. **Lexical analysis** — tokenizes source into tokens
2. **Parsing** — builds an Abstract Syntax Tree (AST)
3. **Semantic analysis** — type checking, name resolution, overload resolution
4. **Annotation processing** — runs APT processors (e.g., Lombok, MapStruct generate new source)
5. **Bytecode generation** — emits `.class` file

The `.class` file contains:
- Magic bytes `0xCAFEBABE` (first 4 bytes — JVM sanity check on load)
- Class file version (major.minor, e.g., 65.0 for Java 21)
- Constant pool — interned strings, class names, method signatures, numeric literals
- Field and method descriptors
- Method bytecode
- Attributes (debug info, annotations, etc.)

> Note: `javac` does **zero JIT optimization** — it produces straightforward bytecode. All optimization happens inside the JVM at runtime.

### Phase 2 — JVM: Bytecode → Native

```java
// Source
int add(int a, int b) { return a + b; }
```

```
// javap -c output (bytecode)
0: iload_1       // push local var 1 ('a') onto operand stack
1: iload_2       // push local var 2 ('b') onto operand stack
2: iadd          // pop both, add integers, push result
3: ireturn       // return top of operand stack to caller
```

This bytecode is **platform-neutral**. The JVM on each OS JIT-compiles it to the native ISA of the host:

```
MyApp.class
      │
      ├── JVM on Windows (x86)  →  x86 native instructions
      ├── JVM on Linux  (x86)   →  x86 native instructions
      └── JVM on macOS  (ARM)   →  ARM64 native instructions
```

The `.class` file is **identical** on all platforms. Each JVM handles the translation.

---

## 4. Key Annotations / Classes / Interfaces

| Class/Tool | Purpose |
|-----------|---------|
| `ClassLoader` | Abstract base; extend to create custom loaders |
| `Class<T>` | Runtime representation of a loaded class |
| `Class.forName(name)` | Loads and initializes a class by name |
| `ClassLoader.loadClass(name)` | Loads but does NOT initialize |
| `java.lang.instrument` | JVM instrumentation API (used by APM agents) |
| `javac` | Source-to-bytecode compiler |
| `javap` | Bytecode disassembler (inspect `.class` files) |
| `jconsole` / `jvisualvm` | JVM monitoring tools |
| `jstack` | Thread dump (stack trace of all threads) |
| `jmap` | Heap dump |

---

## 5. Real-World Example

```java
// OrderService.java
@Service
public class OrderService {

    public BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
                    .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQty())))
                    .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

**What happens when this runs in production (high traffic):**

1. `OrderService` class is loaded by the Application ClassLoader on first use.
2. `calculateTotal` is initially interpreted.
3. After ~10,000 calls (flash sale scenario), JIT kicks in:
   - Inlines `stream()`, `map()`, `reduce()` lambda calls.
   - May eliminate intermediate `Stream` object allocation via escape analysis.
4. Subsequent calls run as native machine code — orders of magnitude faster.

---

## 6. Scenario Walkthroughs

### Scenario A: PermGen OutOfMemoryError in a Tomcat App Server

**Context:** An e-commerce app is deployed to Tomcat 7 (Java 7). During development, the app is hot-reloaded multiple times.

**What happens:**
1. Each hot-reload causes Tomcat to create a new `WebAppClassLoader` and load all classes fresh.
2. Old class metadata is supposed to be GC'd from PermGen.
3. But static references (e.g., in a ThreadLocal, JDBC driver registry, or logging framework) hold a reference to the old classloader.
4. Old classloader cannot be GC'd → its class metadata stays in PermGen.
5. After enough reloads: `OutOfMemoryError: PermGen space`.

**Fix in Java 8+:** Metaspace (native memory) auto-grows, and the GC is better at cleaning up classloaders. The underlying classloader leak is still a bug, but it manifests as native OOM much later.

---

### Scenario B: JIT Deoptimization in a Payment Service

**Context:** A payment service processes millions of transactions. The JIT has compiled `validateCard()` assuming all cards are `VisaCard` (monomorphic call site). Suddenly, `MasterCard` instances start arriving.

**What happens:**
1. JIT inlined `validateCard()` based on profiling data (only `VisaCard` seen so far).
2. `MasterCard` arrives — the inlined assumption is violated.
3. JVM **deoptimizes** — throws away the compiled native code, falls back to interpreter.
4. After enough `MasterCard` calls, JIT recompiles with a polymorphic dispatch.
5. Throughput temporarily dips, then recovers.

This is why **JVM warmup time** matters in production — performance benchmarks should be measured after warmup (use JMH for benchmarking).

---

## 7. Common Interview Questions

**Q1: What is the difference between JDK, JRE, and JVM?**

JVM is the abstract execution engine for bytecode. JRE is JVM + standard libraries needed to run Java programs. JDK is JRE + compiler + dev tools. A developer needs JDK; a server running a pre-built JAR only needs a JRE (or JDK, since standalone JREs no longer exist in Java 9+).

---

**Q2: Explain the parent delegation model in class loading. Why does it exist?**

When asked to load a class, a classloader first delegates to its parent before attempting to load it itself. The chain is: Application → Platform → Bootstrap. Bootstrap handles core JDK classes. This ensures `java.lang.String` is always loaded by Bootstrap — user code cannot substitute a malicious `String` class into the runtime. It also ensures classes are loaded only once and shared across classloaders in the same hierarchy.

---

**Q3: What is the difference between PermGen and Metaspace?**

Both store class metadata (method area). PermGen (pre-Java 8) was a fixed-size region inside the JVM heap — common cause of `OutOfMemoryError: PermGen space` in app servers with classloader leaks. Metaspace (Java 8+) lives in native memory outside the heap, grows dynamically. It can still OOM but is tunable via `-XX:MaxMetaspaceSize`. Static variables that were in PermGen are now in Metaspace too.

---

**Q4: Is Java interpreted or compiled?**

Both. `javac` compiles source to bytecode at build time. The JVM initially interprets bytecode. After a method becomes "hot" (~10k invocations), the JIT compiler compiles it to native machine code. So Java is a **hybrid** — interpreted startup, JIT-compiled steady state. Long-running Java apps (like Spring Boot services) approach native C++ performance for hot paths.

---

**Q5: Why is the JVM Stack thread-safe without synchronization?**

Each thread has its own private JVM stack. Local variables and method parameters live in the current frame on that stack. No other thread can access another thread's stack. This is why local variables are inherently thread-safe. Only shared state (heap objects, static fields) requires synchronization.

---

**Q6: What is escape analysis and how does it help performance?**

Escape analysis is a JIT optimization where the JVM determines if an object "escapes" its creating method — i.e., is it referenced from outside (returned, stored in a field, passed to another thread). If not, the JVM can:
- Allocate it **on the stack** instead of the heap (cheaper, no GC pressure)
- **Eliminate the object entirely** if its fields can be represented as local variables (scalar replacement)

This is why creating short-lived objects in hot methods isn't always as expensive as it looks — the JIT may never put them on the heap at all.

---

**Q7: What is TLAB and why does it matter?**

Thread Local Allocation Buffer. Each thread is given a private chunk of Eden space. Object allocation within a TLAB is a simple pointer bump — no locking required. When the TLAB is exhausted, the thread requests a new one (that part needs synchronization). This makes allocation in multi-threaded apps very fast despite the heap being shared.

---

**Q8: What does `Class.forName()` do vs `ClassLoader.loadClass()`?**

`Class.forName("com.Foo")` loads and **initializes** the class (runs static initializers). Used by JDBC to register drivers (`Class.forName("com.mysql.cj.jdbc.Driver")`).

`ClassLoader.loadClass("com.Foo")` loads the class but does NOT initialize it — static blocks do not run. Useful when you want to check class existence without triggering side effects.

---

## 8. Common Pitfalls & Gotchas

| Pitfall | Detail |
|---------|--------|
| **"Static fields are on the heap"** | Wrong. Static variables are stored in Metaspace (Method Area). The *objects* they reference live on the heap. |
| **"Java is just interpreted"** | Wrong. JIT compiles hot code to native. Don't dismiss Java performance. |
| **Classloader leak** | Holding a reference to a class (via static field, ThreadLocal, etc.) keeps its entire classloader alive → Metaspace leak in app servers. |
| **JVM warmup** | Benchmarking Java without warmup gives misleading results. Use JMH; always measure after steady state. |
| **`-XX:PermSize` on Java 8+** | PermGen flags are ignored on Java 8+ (no PermGen). Use `-XX:MetaspaceSize` / `-XX:MaxMetaspaceSize`. |
| **Stack vs Heap confusion** | Primitive local variables → stack. All objects (even `new int[10]`) → heap. |
| **`Class.forName()` triggers static blocks** | Can cause unexpected side effects. Know the difference from `loadClass()`. |

---

## 9. Summary

- **JDK ⊃ JRE ⊃ JVM** — each layer wraps the previous one.
- The JVM has five memory areas: Metaspace (class metadata), Heap (objects), Stack (frames per thread), PC Register (per thread), Native Method Stack.
- Class loading follows **parent delegation**: Bootstrap → Platform → Application, ensuring core classes are never overridden.
- Java compilation is **two-phase**: `javac` produces platform-neutral bytecode; the JVM JIT-compiles hot bytecode to native machine code at runtime.
- **Metaspace replaced PermGen** in Java 8 — lives in native memory and grows dynamically.
- JIT optimizations (inlining, escape analysis, loop unrolling) make long-running Java apps approach native performance — warmup time is the tradeoff.
