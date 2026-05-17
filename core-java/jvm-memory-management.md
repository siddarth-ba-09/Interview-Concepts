# JVM Memory Management

---

## 1. JVM Memory Areas — Full Map

The JVM divides memory into distinct **runtime data areas**. Some are shared across all threads; others are created per-thread.

```
┌────────────────────────────────────────────────────────────────┐
│                        JVM Process Memory                      │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                        HEAP  (shared)                   │  │
│  │                                                         │  │
│  │   ┌──────────────────────┐   ┌────────────────────────┐ │  │
│  │   │    Young Generation  │   │    Old Generation      │ │  │
│  │   │  ┌──────┬──────────┐ │   │  (Tenured Space)       │ │  │
│  │   │  │ Eden │Survivor  │ │   │                        │ │  │
│  │   │  │      │ S0 | S1  │ │   │                        │ │  │
│  │   │  └──────┴──────────┘ │   │                        │ │  │
│  │   └──────────────────────┘   └────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────┐                                  │
│  │  Metaspace  (off-heap)   │  class metadata, static vars    │
│  └──────────────────────────┘                                  │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ JVM Stack    │  │ JVM Stack    │  │ JVM Stack    │ ...      │
│  │ (Thread 1)   │  │ (Thread 2)   │  │ (Thread 3)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                │
│  ┌──────────────┐  ┌──────────────────────────────────────┐   │
│  │ PC Register  │  │  Native Method Stack (JNI)           │   │
│  │ (per thread) │  │  (per thread)                        │   │
│  └──────────────┘  └──────────────────────────────────────┘   │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Code Cache  (JIT-compiled native code)                  │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Each Memory Area in Detail

### 2.1 Heap

- **What lives here:** Every object and array allocated with `new`.
- **Shared:** Yes — all threads read/write the same heap.
- **GC-managed:** Yes — the garbage collector reclaims unreachable objects here.
- Divided into **Young Generation** and **Old Generation** (details in Section 3).

Key JVM flags:

| Flag | Meaning |
|------|---------|
| `-Xms512m` | Initial heap size |
| `-Xmx2g` | Maximum heap size |
| `-XX:NewRatio=2` | Ratio of Old:Young (default 2 → Young is 1/3 of heap) |
| `-XX:SurvivorRatio=8` | Ratio of Eden:Survivor within Young (default 8 → Eden is 8/10) |

> **Interview tip:** Setting `-Xms` == `-Xmx` prevents the JVM from repeatedly resizing the heap, which reduces GC pause overhead in production.

---

### 2.2 JVM Stack (Thread Stack)

- **Created:** One per thread, at thread creation time.
- **What lives here:** **Stack frames** — one frame pushed per method call, popped on return.
- Each frame contains:
  - **Local variable array** — method parameters and local variables (primitives stored by value, object references stored here, actual object on heap)
  - **Operand stack** — working memory for bytecode instructions (like a CPU register)
  - **Reference to constant pool** of the owning class

```
Thread Stack
┌──────────────────────┐   ← top (currently executing frame)
│  Frame: methodC()    │
│  locals: x=5, y=10   │
│  operand stack: [15] │
├──────────────────────┤
│  Frame: methodB()    │
│  locals: obj=ref     │
├──────────────────────┤
│  Frame: methodA()    │
│  locals: a=1         │
├──────────────────────┤
│  Frame: main()       │
└──────────────────────┘   ← bottom (entry point)
```

Key JVM flag:

| Flag | Meaning |
|------|---------|
| `-Xss512k` | Stack size per thread (default ~512KB–1MB depending on OS/JVM) |

---

### 2.3 Metaspace (Java 8+) / PermGen (Java 7 and earlier)

- **What lives here:** Class metadata — class name, method signatures, bytecode, constant pool, static variables, interned strings (moved to heap in Java 7+).
- **Shared:** Yes — across all threads.
- **GC-managed:** Partially — class metadata is reclaimed when a `ClassLoader` and all its loaded classes become unreachable (class unloading).

| | PermGen (≤ Java 7) | Metaspace (Java 8+) |
|-|---|---|
| Location | JVM heap (fixed region) | Native OS memory (off-heap) |
| Default max | Fixed (e.g., 64MB–256MB) | Effectively unlimited (bounded by OS RAM) |
| Common error | `OutOfMemoryError: PermGen space` | `OutOfMemoryError: Metaspace` |
| Tuning | `-XX:MaxPermSize=256m` | `-XX:MaxMetaspaceSize=256m` |

> **Why was PermGen replaced?** It was notoriously hard to tune correctly. Application servers doing hot-reload (e.g., Tomcat) would exhaust PermGen because old class metadata was not collected fast enough. Metaspace grows dynamically and avoids this cliff.

---

### 2.4 PC Register (Program Counter)

- One per thread.
- Holds the address of the **currently executing JVM instruction** (bytecode offset).
- If the current method is `native`, the PC is undefined.
- Very small; never causes memory errors in practice.

---

### 2.5 Native Method Stack

- One per thread.
- Used when a Java method calls into native code via **JNI** (C/C++ libraries).
- Behaves like a native C call stack.
- A `StackOverflowError` can also originate here if native call depth is exceeded.

---

### 2.6 Code Cache

- Stores **JIT-compiled native machine code** produced by the C1 (client) and C2 (server) compilers.
- Not GC-managed in the traditional sense — compiled code is evicted when classes are unloaded or JIT decides to deoptimize.
- If exhausted: `Java HotSpot(TM) ... warning: CodeCache is full. Compiler has been disabled.`

Flag: `-XX:ReservedCodeCacheSize=256m`

---

## 3. Heap Generations and Garbage Collection

### 3.1 Why Generations?

The **Generational Hypothesis**: most objects die young. Separating short-lived from long-lived objects lets the GC collect the cheap-to-collect young area frequently without touching the expensive old area.

### 3.2 Heap Layout

```
┌──────────────────────────────────────────────────────────┐
│                        HEAP                              │
│                                                          │
│  ┌───────────────────────────────┐  ┌─────────────────┐  │
│  │       Young Generation        │  │  Old Generation  │  │
│  │                               │  │  (Tenured)       │  │
│  │  ┌──────────┐  ┌───┐  ┌───┐  │  │                  │  │
│  │  │   Eden   │  │S0 │  │S1 │  │  │  Long-lived       │  │
│  │  │ (new obj)│  │   │  │   │  │  │  objects          │  │
│  │  └──────────┘  └───┘  └───┘  │  │                  │  │
│  └───────────────────────────────┘  └─────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

| Region | Purpose |
|--------|---------|
| **Eden** | All new objects are allocated here first |
| **Survivor S0 / S1** | Objects that survived at least one Minor GC |
| **Old (Tenured)** | Objects that survived enough Minor GCs (threshold: `MaxTenuringThreshold`, default 15) |

### 3.3 Minor GC (Young GC)

Triggered when **Eden is full**.

1. Mark all live objects in Eden + active Survivor space (e.g., S0).
2. Copy live objects to the other Survivor space (S1). Increment their age counter.
3. Objects that exceed `MaxTenuringThreshold` are **promoted** to Old Gen.
4. Eden and S0 are wiped clean.
5. S0 and S1 roles swap for the next cycle.

- **Stop-The-World (STW):** Yes, but very short (typically < 50ms) because Young Gen is small.

### 3.4 Major GC / Full GC

- **Major GC:** Collects Old Generation only (rare, collector-dependent).
- **Full GC:** Collects both Young + Old + Metaspace. Very expensive — can pause for seconds.

Triggered by:
- Old Gen filling up (promotion failure).
- `System.gc()` call (hint, not guaranteed).
- Metaspace filling up.
- Explicit GC from JMX/tooling.

---

## 4. Garbage Collectors

### 4.1 Serial GC
- Single-threaded, STW for all phases.
- Flag: `-XX:+UseSerialGC`
- Use case: small single-core environments, batch jobs.

### 4.2 Parallel GC (Throughput Collector)
- Multiple threads for Minor and Major GC. Still STW.
- Default in Java 8 for server JVMs.
- Flag: `-XX:+UseParallelGC`
- Use case: batch processing where throughput > latency.

### 4.3 CMS (Concurrent Mark Sweep) — Deprecated/Removed
- Concurrent marking phases to reduce pause time. Still needs STW for initial mark + remark.
- Suffered from **fragmentation** (no compaction) → eventually caused `OutOfMemoryError` even with free space.
- Removed in Java 14.

### 4.4 G1 GC (Garbage First) — Default since Java 9
- Divides heap into equal-sized **regions** (typically 1MB–32MB each) rather than fixed Young/Old areas.
- Collects the regions with the most garbage first (hence "Garbage First").
- Concurrent marking + incremental compaction → **predictable low-pause** GC.
- Flag: `-XX:+UseG1GC`
- Key tuning flag: `-XX:MaxGCPauseMillis=200` (soft target, not a hard guarantee)

```
Heap with G1:
┌──┬──┬──┬──┬──┬──┬──┬──┐
│E │E │S │O │O │E │H │O │  E=Eden, S=Survivor, O=Old, H=Humongous
└──┴──┴──┴──┴──┴──┴──┴──┘
Regions are dynamically assigned roles — no fixed Young/Old boundaries.
```

**Humongous objects:** Objects > 50% of region size are allocated directly in the Old region to avoid Eden → Survivor → Old copying overhead.

### 4.5 ZGC (Z Garbage Collector) — Production-ready since Java 15
- Sub-millisecond pause times regardless of heap size (tested up to TB-scale heaps).
- All phases concurrent except a very brief STW root scan.
- Uses **load barriers** and **colored pointers** (stores GC metadata in pointer bits).
- Flag: `-XX:+UseZGC`

### 4.6 Shenandoah
- Also targets ultra-low pauses; concurrent compaction.
- Developed by Red Hat; available in OpenJDK.
- Flag: `-XX:+UseShenandoahGC`

### Summary Table

| Collector | Pause Type | Best For | Default |
|-----------|-----------|---------|---------|
| Serial | Full STW | Small/single-core | Java < 5 server |
| Parallel | Full STW (multi-thread) | High throughput | Java 8 server |
| G1 | Short STW + concurrent | Balanced (latency + throughput) | Java 9+ |
| ZGC | Sub-ms STW | Ultra-low latency, large heaps | Opt-in |
| Shenandoah | Sub-ms STW | Ultra-low latency | Opt-in |

---

## 5. Object Lifecycle in Memory

```
1. new MyObject()
       │
       ▼
   Allocated in Eden (Young Gen)
       │
       │ Eden fills up → Minor GC triggered
       ▼
   Live? ──No──► Collected (memory freed)
       │
      Yes
       ▼
   Copied to Survivor S0 (age = 1)
       │
       │ Next Minor GC
       ▼
   Live? ──No──► Collected
       │
      Yes (age incremented each survival)
       ▼
   age >= MaxTenuringThreshold (15)?
       │
      Yes
       ▼
   Promoted to Old Generation
       │
       │ Old Gen fills → Major/Full GC
       ▼
   Live? ──No──► Collected
       │
      Yes
       ▼
   Stays in Old Gen until unreachable
```

---

## 6. Garbage Collection Roots (GC Roots)

The GC starts from **GC roots** — objects that are always considered reachable — and marks everything reachable transitively.

GC roots include:
- Local variables and method parameters on all thread stacks
- Active threads themselves
- Static fields of loaded classes
- JNI references (native code holding Java object references)
- Objects referenced by synchronized monitors

Anything NOT reachable from a GC root is **eligible for collection**.

---

## 7. `StackOverflowError`

### 7.1 What It Is

`java.lang.StackOverflowError` is thrown when the JVM stack for a thread runs out of space. It is an `Error` (not an `Exception`), meaning it signals an unrecoverable JVM-level condition.

```
java.lang.StackOverflowError
    at com.example.Foo.recurse(Foo.java:10)
    at com.example.Foo.recurse(Foo.java:10)
    at com.example.Foo.recurse(Foo.java:10)
    ... (hundreds more frames)
```

### 7.2 Root Cause

Each method call pushes a **stack frame** onto the thread's stack. The stack has a fixed maximum size (`-Xss`). When the call depth exceeds what fits, the JVM throws `StackOverflowError`.

**Most common causes:**

1. **Unbounded recursion** — the most frequent cause:
   ```java
   // Missing base case → infinite recursion
   public int factorial(int n) {
       return n * factorial(n - 1); // never reaches 0
   }
   ```

2. **Mutual recursion** — A calls B calls A calls B...:
   ```java
   void methodA() { methodB(); }
   void methodB() { methodA(); }
   ```

3. **Very deep (but finite) call chains** — legitimate deep recursion on large data structures (deep XML/JSON parsing, deep tree traversal) with a small stack size.

4. **toString() / equals() / hashCode() cycle** on objects with circular references (e.g., a bidirectional JPA relationship without proper `@ToString` exclusion).

### 7.3 Stack Frame Size Factors

The deeper each frame, the fewer frames fit before overflow. Frame size grows with:
- Number of local variables
- Size of local primitives (`long`/`double` take 2 slots)
- Number of synchronized blocks (adds monitor references)

### 7.4 How to Fix

| Cause | Fix |
|-------|-----|
| Missing base case | Add the correct base/termination condition |
| Deep tree/list recursion | Rewrite iteratively using an explicit `Deque` stack |
| Legitimate deep recursion on large input | Increase stack size: `-Xss2m` |
| Circular object graph in `toString()` | Use Lombok `@ToString(exclude=...)` or manual exclusion |
| Mutual recursion causing logical infinite loop | Redesign using iteration or trampolining |

**Iterative rewrite example:**
```java
// Recursive — can overflow for deep lists
int sum(Node node) {
    if (node == null) return 0;
    return node.val + sum(node.next);
}

// Iterative — safe regardless of depth
int sum(Node node) {
    int total = 0;
    while (node != null) {
        total += node.val;
        node = node.next;
    }
    return total;
}
```

### 7.5 Can You Catch `StackOverflowError`?

Yes — because `Error` extends `Throwable`, you can `catch (StackOverflowError e)`. However, doing so is almost always wrong, as the stack is in an indeterminate state and the frame that catches it may itself cause further overflow. Only catch it at a very high-level boundary (e.g., a plugin sandbox) to prevent one bad call from crashing the entire application.

---

## 8. `OutOfMemoryError`

### 8.1 What It Is

`java.lang.OutOfMemoryError` is thrown when the JVM cannot allocate memory for a new object or JVM structure. Like `StackOverflowError`, it is an `Error`. It comes with a descriptive **message** that identifies exactly which memory area was exhausted — this is the most important diagnostic signal.

### 8.2 Variants and Their Meanings

#### `OutOfMemoryError: Java heap space`

```
java.lang.OutOfMemoryError: Java heap space
```

- **Cause:** The JVM heap (Young + Old Gen) is full and GC cannot reclaim enough space.
- **Most common cause:** Memory leak — objects remain referenced (directly or indirectly via static collections, caches, listeners) and are never collected.
- **Other causes:** Heap simply too small for the workload; loading very large data sets into memory at once.

**How to investigate:**
1. Take a heap dump: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof`
2. Analyze with Eclipse MAT, VisualVM, or IntelliJ's heap analyzer.
3. Look for the **Dominator Tree** — the objects retaining the most memory.

**Common leak patterns:**

| Pattern | Example |
|---------|---------|
| Static collection grows unbounded | `static List<Event> auditLog = new ArrayList<>()` never trimmed |
| Cache without eviction | `static Map<String, Object> cache` — never cleared |
| Listener/callback not deregistered | GUI listeners holding references to large objects |
| ThreadLocal not removed | `ThreadLocal` variables in thread pools — thread never dies, value never removed |
| Classloader leak | App server redeploying wars; old classloaders held alive by static references |

**ThreadLocal leak example:**
```java
// DANGEROUS in thread pools (threads are reused)
private static final ThreadLocal<byte[]> BUFFER = ThreadLocal.withInitial(() -> new byte[1024 * 1024]);

void process() {
    byte[] buf = BUFFER.get();
    // ... use buf ...
    // MISSING: BUFFER.remove() — buf stays referenced by the pooled thread forever
}
```

**Fix:** Always call `ThreadLocal.remove()` in a `finally` block when using thread pools.

---

#### `OutOfMemoryError: GC overhead limit exceeded`

```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

- **Cause:** The JVM is spending > 98% of time in GC but recovering < 2% of heap — it gives up and throws this error rather than spinning forever.
- **Indicates:** Severe heap pressure; effectively a slow-motion heap exhaustion.
- **Fix:** Same as heap space — find the leak or increase heap.

To disable this check (not recommended): `-XX:-UseGCOverheadLimit`

---

#### `OutOfMemoryError: Metaspace`

```
java.lang.OutOfMemoryError: Metaspace
```

- **Cause:** Metaspace (class metadata region) is full.
- **Most common cause:** Too many classes loaded — typically due to dynamic class generation (reflection proxies, CGLIB/ByteBuddy, Groovy/JRuby scripts, JSP compilation) or classloader leaks in app servers.
- **Fix:**
  - Set `-XX:MaxMetaspaceSize=512m` to give more room.
  - If the metaspace grows without bound, it is a **classloader leak** — find what is preventing old `ClassLoader` instances from being GC'd.

---

#### `OutOfMemoryError: Direct buffer memory`

```
java.lang.OutOfMemoryError: Direct buffer memory
```

- **Cause:** Off-heap direct ByteBuffers (`ByteBuffer.allocateDirect()`) exhausted the native memory limit.
- **Common in:** Netty-based applications, NIO servers, Kafka clients.
- **Fix:** `-XX:MaxDirectMemorySize=512m`; investigate where direct buffers are allocated and not released.

---

#### `OutOfMemoryError: Unable to create new native thread`

```
java.lang.OutOfMemoryError: unable to create new native thread
```

- **Cause:** The OS cannot create another OS thread. This is a system-level limit, not a heap issue.
- **Causes:**
  - Too many threads created (thread leak — threads started but never terminated).
  - OS per-process thread limit (`ulimit -u`) reached.
  - Not enough virtual address space to allocate more thread stacks (each thread needs `-Xss` of virtual memory).
- **Fix:** Reduce number of threads; use thread pools; check for thread leaks; increase OS limit.

---

#### `OutOfMemoryError: Compressed class space`

```
java.lang.OutOfMemoryError: Compressed class space
```

- **Cause:** When using compressed class pointers (`-XX:+UseCompressedClassPointers`, default with heap < 32GB), the compressed class space — a fixed-size region within Metaspace for class structure data — is exhausted.
- **Fix:** `-XX:CompressedClassSpaceSize=1g`

---

### 8.3 OOM Diagnostic Flags

```bash
# Dump heap to file on OOM (invaluable in production)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heap-$(date +%s).hprof

# Run a script/command on OOM (e.g., alert + restart)
-XX:OnOutOfMemoryError="kill -9 %p; systemctl restart myapp"

# GC logging (Java 11+)
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

---

## 9. Memory Leak Detection Workflow

```
Symptom: heap grows over time, eventually OOM
          │
          ▼
1. Enable GC logging → look for shrinking "after-GC" heap floor
          │
          ▼
2. Enable heap dump on OOM: -XX:+HeapDumpOnOutOfMemoryError
          │
          ▼
3. Open .hprof in Eclipse MAT or VisualVM
          │
          ▼
4. Check "Leak Suspects" report (MAT auto-detects)
          │
          ▼
5. Examine Dominator Tree → find top memory retainers
          │
          ▼
6. Trace reference chain (GC root → retaining object)
          │
          ▼
7. Identify the code holding the reference → fix retention
```

---

## 10. Quick Reference: Errors vs. Causes vs. Fixes

| Error | Memory Area | Typical Root Cause | Quick Fix |
|-------|-------------|-------------------|-----------|
| `StackOverflowError` | JVM Stack | Infinite/deep recursion | Fix recursion; increase `-Xss` |
| `OOM: Java heap space` | Heap (Old Gen) | Memory leak or undersized heap | Fix leak; increase `-Xmx` |
| `OOM: GC overhead limit exceeded` | Heap | Extreme heap pressure | Fix leak; increase `-Xmx` |
| `OOM: Metaspace` | Metaspace | Too many classes / classloader leak | Fix leak; increase `-XX:MaxMetaspaceSize` |
| `OOM: Direct buffer memory` | Off-heap | Unreleased direct ByteBuffers | Fix release; increase `-XX:MaxDirectMemorySize` |
| `OOM: unable to create native thread` | OS memory | Thread leak / OS limit | Fix thread leak; tune `ulimit` |
| `OOM: Compressed class space` | Metaspace (compressed) | Too many class definitions | Increase `-XX:CompressedClassSpaceSize` |

---

## 11. Key JVM Memory Flags Summary

```bash
# Heap
-Xms512m                          # initial heap
-Xmx4g                            # max heap
-XX:NewRatio=2                    # Old:Young ratio
-XX:SurvivorRatio=8               # Eden:Survivor ratio

# Stack
-Xss512k                          # thread stack size

# Metaspace
-XX:MetaspaceSize=128m            # initial metaspace (triggers GC when reached)
-XX:MaxMetaspaceSize=512m         # hard cap on metaspace

# GC selection
-XX:+UseG1GC                      # G1 (default Java 9+)
-XX:+UseZGC                       # ZGC (ultra-low pause)
-XX:MaxGCPauseMillis=200          # G1 pause target

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
-Xlog:gc*:file=/tmp/gc.log        # GC log (Java 11+)
-XX:+PrintGCDetails               # GC log (Java 8)
```

---

## 12. Interview Questions

**Q: Where are local variables stored?**
In the stack frame's local variable array. If the variable is an object reference, the reference is on the stack but the actual object lives on the heap.

**Q: Can objects be allocated off the heap?**
Yes — through **escape analysis**, the JIT compiler can allocate objects on the stack (stack allocation) if it proves the object does not escape the method. This avoids GC pressure entirely. Also, `ByteBuffer.allocateDirect()` allocates in native off-heap memory.

**Q: What is a memory leak in Java?**
A situation where objects that are no longer needed remain **reachable** from a GC root (e.g., held in a static collection), so the GC cannot collect them. Java prevents *dangling pointer* bugs but does not prevent logical memory leaks.

**Q: What triggers a Full GC?**
Promotion failure (Old Gen full when trying to promote from Young), `System.gc()`, Metaspace exhaustion, or explicit tooling requests. Full GC is expensive — reducing its frequency is a key performance goal.

**Q: What is the difference between `StackOverflowError` and `OutOfMemoryError`?**
- `StackOverflowError` → the **thread stack** exceeded its size limit, almost always due to excessive recursion.
- `OutOfMemoryError` → a JVM memory region (heap, Metaspace, native memory) ran out of space due to too many/large allocations or a memory leak.

**Q: Why should `ThreadLocal` be removed in thread pools?**
Thread pool threads are reused. If `ThreadLocal.remove()` is not called, the value remains bound to the thread indefinitely — even after the request that set it is complete. This is both a memory leak (the value is never GC'd) and a correctness bug (a later request on the same thread might see a stale value).

**Q: What is the Metaspace and why was PermGen removed?**
Metaspace is the successor to PermGen (removed in Java 8). PermGen was a fixed-size JVM heap region for class metadata; it was easy to exhaust on app servers with hot-reload. Metaspace uses native OS memory and grows dynamically, eliminating the PermGen space error under normal conditions.
