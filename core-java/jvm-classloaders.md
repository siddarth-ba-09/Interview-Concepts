# JVM ClassLoaders

---

## 1. Overview

A **ClassLoader** is the JVM subsystem responsible for finding, loading, linking, and initializing `.class` files into the JVM at runtime. It bridges the file system (or network, database, memory) and the JVM's runtime representation of a type.

**Why it matters:**
- Controls *which classes are available* and *when* they are loaded.
- Enables **isolation** — multiple versions of the same class can coexist in the same JVM if loaded by different ClassLoaders (application servers, OSGi, hot-reload frameworks all rely on this).
- ClassLoader bugs are the root cause of `ClassNotFoundException`, `NoClassDefFoundError`, `ClassCastException` across loader boundaries, and Metaspace/PermGen memory leaks.

---

## 2. Core Concepts

### 2.1 What "Loading a Class" Actually Means

Loading a class is a three-phase process called **Linking**. Together with Loading and Initialization they form the full class lifecycle:

```
┌───────────────────────────────────────────────────────────┐
│                  Class Lifecycle                          │
│                                                           │
│  1. Loading      → find .class bytes, create Class object │
│  2. Linking                                               │
│     a. Verification  → validate bytecode safety           │
│     b. Preparation   → allocate static fields (defaults)  │
│     c. Resolution    → symbolic refs → direct refs        │
│  3. Initialization   → run <clinit> (static blocks)       │
└───────────────────────────────────────────────────────────┘
```

#### Loading
- ClassLoader reads the raw bytes of the `.class` file.
- JVM creates an instance of `java.lang.Class` representing that type.
- The `Class` object lives in the heap; the class metadata lives in Metaspace.

#### Verification
- Bytecode verifier checks structural correctness: valid opcodes, type safety, no stack underflow/overflow, proper `final` usage.
- Cannot be skipped for untrusted code. Can be disabled with `-Xverify:none` (dangerous, never in production).

#### Preparation
- JVM allocates memory for all static fields and sets them to **default values** (`0`, `false`, `null`).
- No user code runs here — static initializers run in Initialization, not Preparation.

#### Resolution
- Symbolic references in the constant pool (e.g., `com/example/Foo.bar()V`) are replaced with direct references (memory pointers or method table offsets).
- Can be **eager** (resolved at load time) or **lazy** (resolved on first use) — JVM spec allows either; HotSpot uses lazy resolution.

#### Initialization
- JVM runs the class's `<clinit>` method (static initializer blocks + static field assignments, in textual order).
- Happens exactly **once**, on the first active use of the class.
- Thread-safe: JVM guarantees that `<clinit>` is run by exactly one thread; other threads block until it completes.

---

### 2.2 Active vs. Passive Use

Initialization is only triggered by **active use**:

| Active Use (triggers initialization) | Passive Use (does NOT trigger initialization) |
|--------------------------------------|-----------------------------------------------|
| `new MyClass()` | Referencing a `static final` compile-time constant |
| Calling a static method | Creating an array of the type (`new MyClass[10]`) |
| Reading/writing a non-constant static field | A subclass being initialized (superclass IS initialized) |
| `Class.forName("com.example.Foo")` | `ClassLoader.loadClass("com.example.Foo")` |
| JVM main class | Referencing an inherited static field via subclass |

---

### 2.3 Class Identity = Class + ClassLoader

Two `Class` objects are considered the **same type** only if they have the same binary name **and** were loaded by the same ClassLoader instance. This is critical:

```java
ClassLoader cl1 = new CustomClassLoader();
ClassLoader cl2 = new CustomClassLoader();

Class<?> c1 = cl1.loadClass("com.example.Foo");
Class<?> c2 = cl2.loadClass("com.example.Foo");

System.out.println(c1 == c2);       // false — different loaders → different classes
System.out.println(c1.equals(c2));  // false

Object o1 = c1.getDeclaredConstructor().newInstance();
// ClassCastException if you try to cast o1 to Foo loaded by a third ClassLoader
```

This is why objects passed across ClassLoader boundaries cause `ClassCastException` even when the class names match.

---

## 3. Built-In ClassLoader Hierarchy

### Java 8 and Earlier

```
Bootstrap ClassLoader  (native C++, no Java object)
        │
        ▼
Extension ClassLoader  (sun.misc.Launcher$ExtClassLoader)
        │
        ▼
Application ClassLoader (sun.misc.Launcher$AppClassLoader)
        │
        ▼
  [User-defined ClassLoaders]
```

| ClassLoader | What It Loads | Java Class |
|---|---|---|
| **Bootstrap** | `java.lang.*`, `java.util.*`, `java.io.*` — JDK core classes from `rt.jar` | None (native C++) — `getClassLoader()` returns `null` |
| **Extension** | JDK extension JARs from `$JAVA_HOME/lib/ext/` | `sun.misc.Launcher$ExtClassLoader` |
| **Application (System)** | Your app JARs + classpath (`-cp`) | `sun.misc.Launcher$AppClassLoader` |

### Java 9+ (Module System)

```
Bootstrap ClassLoader   (loads java.base module)
        │
        ▼
Platform ClassLoader    (replaces Extension; loads platform modules)
        │
        ▼
Application ClassLoader (loads app module-path / classpath)
        │
        ▼
  [User-defined ClassLoaders]
```

Key change: Extension ClassLoader replaced by **Platform ClassLoader** (`java.lang.ClassLoader` subclass), which loads JDK modules like `java.sql`, `java.xml`, etc.

---

## 4. Parent Delegation Model

### 4.1 How It Works

Every ClassLoader (except Bootstrap) has a **parent**. When asked to load a class, a ClassLoader follows this algorithm:

```
loadClass("com.example.Foo") called on ApplicationClassLoader
        │
        ▼
1. Check local cache: already loaded? → return cached Class object
        │
        ▼
2. Delegate to parent (Platform/Extension ClassLoader)
        │
        ▼
   Parent delegates to ITS parent (Bootstrap ClassLoader)
        │
        ▼
   Bootstrap tries to load — fails (not a JDK class)
        │
        ▼
   Platform tries to load — fails (not a platform class)
        │
        ▼
3. Application ClassLoader tries to load from classpath → SUCCESS
        │
        ▼
4. Returns Class object
```

**Pseudocode:**
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 1. Check already-loaded cache
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            // 2. Delegate to parent first
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                // parent is null → we are child of Bootstrap
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // parent couldn't load — try ourselves
        }
        if (c == null) {
            // 3. Find the class ourselves
            c = findClass(name); // override this in custom loaders
        }
    }
    if (resolve) resolveClass(c);
    return c;
}
```

### 4.2 Why Parent Delegation?

1. **Security:** Prevents user code from shadowing JDK classes. You cannot create a `java.lang.String` that the JVM uses instead of the real one.
2. **Consistency:** Ensures `java.lang.Object` loaded by Bootstrap is the same `Object` everywhere — avoids type identity chaos.
3. **Avoids duplicate loading:** Once Bootstrap loads `java.util.ArrayList`, no child loader ever loads it again.

### 4.3 Where Parent Delegation Is Broken (Intentionally)

Some legitimate use cases require loading from the child *before* the parent — this is done by overriding `loadClass()`:

| Use Case | Mechanism |
|---|---|
| **OSGi** | Each bundle has its own class space; delegation goes to the bundle's wiring, not just parent |
| **Tomcat / App servers** | Each WAR has its own `WebAppClassLoader` — loads webapp classes before shared parent to achieve isolation |
| **JDBC / JNDI SPI** | Thread Context ClassLoader (TCCL) used to let Bootstrap-loaded code access app-level drivers |
| **Hot-reload** (Spring DevTools, JRebel) | Discards old ClassLoader, creates a new one — "reloading" is actually re-loading classes via a new loader |

---

## 5. Thread Context ClassLoader (TCCL)

### 5.1 The Problem

The SPI (Service Provider Interface) pattern (JDBC drivers, JAXP parsers, etc.) is loaded by Bootstrap, but the actual implementations are in app JARs (loaded by Application ClassLoader). Bootstrap cannot delegate *down* to Application ClassLoader in the normal delegation model.

### 5.2 The Solution: TCCL

Every Java thread carries a **Thread Context ClassLoader**, set by whoever creates the thread (typically the framework):

```java
// Getting TCCL
ClassLoader tccl = Thread.currentThread().getContextClassLoader();

// Setting TCCL
Thread.currentThread().setContextClassLoader(myClassLoader);
```

Framework code (Bootstrap-loaded) uses TCCL to load app-level implementations:

```java
// Inside DriverManager (Bootstrap-loaded) — simplified
ServiceLoader<Driver> drivers = ServiceLoader.load(Driver.class,
    Thread.currentThread().getContextClassLoader()); // uses TCCL to find app drivers
```

### 5.3 Production Pattern: Temporarily Switch TCCL

```java
ClassLoader original = Thread.currentThread().getContextClassLoader();
try {
    Thread.currentThread().setContextClassLoader(targetClassLoader);
    // code that relies on SPI to find the right implementations
} finally {
    Thread.currentThread().setContextClassLoader(original); // always restore
}
```

---

## 6. Custom ClassLoaders

### 6.1 When to Write One

- Load classes from a non-standard source (database, network URL, encrypted JAR, in-memory byte array).
- Implement class isolation (multiple versions of the same library in the same JVM).
- Hot-reload / plugin systems.
- Bytecode transformation at load time (instrumentation agents, AOP weaving).

### 6.2 The Correct Extension Point

**Always override `findClass()`, not `loadClass()`.**

Overriding `loadClass()` breaks parent delegation. Override `findClass()` — it is called *after* the parent has already been given a chance to load:

```java
public class DatabaseClassLoader extends ClassLoader {

    private final DataSource dataSource;

    public DatabaseClassLoader(ClassLoader parent, DataSource dataSource) {
        super(parent); // always pass a parent
        this.dataSource = dataSource;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadBytesFromDatabase(name);
        if (classBytes == null) {
            throw new ClassNotFoundException("Class not found in DB: " + name);
        }
        return defineClass(name, classBytes, 0, classBytes.length);
    }

    private byte[] loadBytesFromDatabase(String className) {
        String resourceName = className.replace('.', '/') + ".class";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT bytecode FROM class_store WHERE name = ?")) {
            ps.setString(1, resourceName);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) return rs.getBytes("bytecode");
        } catch (SQLException e) {
            throw new RuntimeException("DB error loading class: " + className, e);
        }
        return null;
    }
}
```

### 6.3 Bytecode Transformation at Load Time

```java
public class InstrumentingClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] rawBytes = loadRawBytes(name); // read from file/classpath
        byte[] transformed = transform(rawBytes); // apply ASM/Javassist transformation
        return defineClass(name, transformed, 0, transformed.length);
    }
}
```

This is how **AOP weaving** (AspectJ LTW), **JaCoCo code coverage**, and **Mockito byte-buddy instrumentation** work.

---

## 7. `Class.forName()` vs `ClassLoader.loadClass()`

| | `Class.forName(name)` | `ClassLoader.loadClass(name)` |
|---|---|---|
| Initializes class? | **Yes** (runs `<clinit>`) | **No** |
| Default ClassLoader | Caller's ClassLoader | The ClassLoader instance itself |
| Overloaded form | `Class.forName(name, initialize, loader)` | N/A |
| Common use | JDBC driver registration, reflection | Custom loader pipelines |

```java
// JDBC traditional: Class.forName initializes the driver,
// which registers itself with DriverManager via static block
Class.forName("com.mysql.cj.jdbc.Driver");

// Just load the class without initializing (no static blocks run)
ClassLoader.getSystemClassLoader().loadClass("com.example.HeavyClass");
```

---

## 8. ClassLoader Memory Leak

### 8.1 Why It Happens

A ClassLoader is garbage-collected only when:
1. No `Class` objects it loaded are referenced.
2. No instances of those classes are alive.
3. The ClassLoader itself has no live references.

If any static field, thread-local, shutdown hook, or JVM-global registry holds a reference to an object whose class was loaded by a custom ClassLoader — **the entire ClassLoader and all its classes are pinned in Metaspace forever**.

### 8.2 Common Leak Causes in Production

| Cause | Explanation |
|---|---|
| `ThreadLocal` not removed | Thread pool threads are long-lived; ThreadLocal value holds a class → pins its ClassLoader |
| Static field in loaded class holding an object | `static Logger log = LoggerFactory.getLogger(...)` — the Logger factory may be in the parent loader; the Class with the static field pins the child loader |
| JDBC driver not deregistered | Driver registered with `DriverManager` (parent-loaded) holds a reference to the driver class (child-loaded) |
| Shutdown hooks | `Runtime.getRuntime().addShutdownHook(thread)` — the hook's class pins its ClassLoader |
| `java.beans.Introspector` cache | Caches `BeanInfo` keyed by `Class`; must call `Introspector.flushCaches()` on undeploy |
| RMI, MBeans registered in MBeanServer | Hold references to loaded classes |

### 8.3 Detecting a ClassLoader Leak

```bash
# Heap dump analysis
jmap -dump:live,format=b,file=heap.hprof <pid>

# Use Eclipse MAT:
# 1. OQL: SELECT * FROM java.lang.ClassLoader
# 2. Check "Retained Heap" for each ClassLoader instance
# 3. Look for WebAppClassLoader instances that should have been GC'd after undeploy
```

### 8.4 Fix Pattern: Safe Cleanup on Undeploy

```java
// In a ServletContextListener or Spring's DisposableBean:
@PreDestroy
public void cleanup() {
    // 1. Deregister JDBC drivers loaded by this ClassLoader
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
        Driver driver = drivers.nextElement();
        if (driver.getClass().getClassLoader() == cl) {
            try { DriverManager.deregisterDriver(driver); }
            catch (SQLException e) { /* log */ }
        }
    }

    // 2. Shut down any ThreadPools this classloader owns
    executorService.shutdownNow();

    // 3. Clear ThreadLocals
    myThreadLocal.remove();
}
```

---

## 9. Scenario Walkthroughs

### 9.1 Tomcat: Two WARs with the Same Library

**Setup:** Two WARs (app1.war, app2.war) both bundle `jackson-databind-2.15.jar`. Tomcat must isolate them.

```
Bootstrap ClassLoader
        │
Platform ClassLoader
        │
Application ClassLoader (Tomcat's own classes)
        ├──► WebAppClassLoader (app1) ← loads jackson 2.15 from app1/WEB-INF/lib
        └──► WebAppClassLoader (app2) ← loads jackson 2.15 from app2/WEB-INF/lib
```

Tomcat's `WebAppClassLoader` overrides `loadClass()` to check WEB-INF/lib **before** delegating to the parent. This means app1 and app2 each get their own isolated copy of Jackson classes. If a class is in both the WAR and the shared Tomcat lib, the WAR's version wins.

**What happens on undeploy:** Tomcat removes the reference to `WebAppClassLoader(app1)`. If no static/ThreadLocal/driver references pin it, GC can collect the loader and all 2000 Jackson classes with it, freeing Metaspace.

---

### 9.2 JDBC Driver Registration (SPI + TCCL)

**Setup:** JDBC's `DriverManager` is loaded by Bootstrap. `com.mysql.cj.jdbc.Driver` is in your app's classpath, loaded by Application ClassLoader.

**Flow:**
1. JVM starts. Bootstrap loads `DriverManager`.
2. `Class.forName("com.mysql.cj.jdbc.Driver")` is called (or Java 6+ SPI auto-discovery via `ServiceLoader`).
3. MySQL driver's `static {}` block calls `DriverManager.registerDriver(new Driver())`.
4. `DriverManager` (Bootstrap-loaded) now holds a reference to the MySQL `Driver` instance (Application-loaded).
5. When you call `DriverManager.getConnection()`, it iterates registered drivers — works because the driver instance is already registered.

**The leak:** If you undeploy the app without calling `DriverManager.deregisterDriver()`, DriverManager (global, Bootstrap) holds the driver instance, which pins the Application ClassLoader → Metaspace leak.

---

### 9.3 Hot-Reload with Custom ClassLoader

**Setup:** A plugin system where plugins can be updated without restarting the JVM.

```java
class PluginManager {
    private volatile ClassLoader pluginLoader;

    void reload() throws Exception {
        // Discard old loader — if nothing else holds a reference, GC can collect it
        pluginLoader = new URLClassLoader(
            new URL[]{ pluginJar.toURI().toURL() },
            getClass().getClassLoader() // parent = app loader
        );
    }

    Plugin loadPlugin(String className) throws Exception {
        Class<?> clazz = pluginLoader.loadClass(className);
        return (Plugin) clazz.getDeclaredConstructor().newInstance();
    }
}
```

**Caveat:** The `Plugin` interface must be loaded by the **parent** (app) ClassLoader — not the plugin loader — otherwise the cast fails with `ClassCastException` due to the class identity rule (same name, different loader = different type).

---

## 10. Common Pitfalls & Gotchas

### `ClassNotFoundException` vs `NoClassDefFoundError`

| Error | When Thrown | Root Cause |
|---|---|---|
| `ClassNotFoundException` | At load time, by `Class.forName()` or `ClassLoader.loadClass()` | Class does not exist on classpath at all |
| `NoClassDefFoundError` | At runtime, during linking/initialization | Class existed at compile time but is missing at runtime; OR class initialization (`<clinit>`) failed — JVM marks it broken and throws `NoClassDefFoundError` on every subsequent access |

**Tricky case:**
```java
class Foo {
    static { throw new RuntimeException("init failed"); }
}

// First call — RuntimeException wrapped in ExceptionInInitializerError
try { new Foo(); } catch (ExceptionInInitializerError e) { }

// Every subsequent call — NoClassDefFoundError (class is permanently broken)
new Foo(); // throws NoClassDefFoundError
```

---

### `ClassCastException` Across ClassLoader Boundaries

```java
// Plugin loader child of app loader
Object obj = pluginLoader.loadClass("com.example.Foo")
                          .getDeclaredConstructor().newInstance();

// If Foo was loaded by pluginLoader and the cast target Foo was loaded by appLoader:
com.example.Foo foo = (com.example.Foo) obj; // ClassCastException!
```

**Fix:** Define the shared interface/superclass in code loaded by the **parent** ClassLoader. Only implementations live in the child loader. Cast to the interface, not the implementation class.

---

### `Class.forName()` Uses the Caller's ClassLoader

```java
// In a class loaded by Bootstrap (e.g., inside DriverManager):
Class.forName("com.mysql.Driver");  // Bootstrap cannot see app classpath → ClassNotFoundException
```

Use the three-argument form to specify a ClassLoader explicitly:
```java
Class.forName("com.mysql.Driver", true, Thread.currentThread().getContextClassLoader());
```

---

### Static Field Initialization Order Dependency

If class A's static initializer references class B, and B's static initializer references A, you have a **circular initialization dependency**. The JVM will not deadlock (it's single-threaded per class), but one of them will see the other's static fields at default values (`null`, `0`), leading to subtle bugs.

---

### `getClassLoader()` Returns `null` for Bootstrap-Loaded Classes

```java
String.class.getClassLoader();          // null — loaded by Bootstrap (native)
ArrayList.class.getClassLoader();       // null — loaded by Bootstrap
MyService.class.getClassLoader();       // ApplicationClassLoader instance
```

`null` does not mean "no class loader" — it means the Bootstrap ClassLoader, which has no Java object representation.

---

## 11. Miscellaneous & Advanced Details

### Parallel-Capable ClassLoaders (Java 7+)

Before Java 7, `loadClass()` was synchronized on the ClassLoader instance — loading two classes simultaneously on different threads would serialize. Java 7 introduced `ClassLoader.registerAsParallelCapable()` which uses per-class-name locking:

```java
public class MyLoader extends ClassLoader {
    static {
        ClassLoader.registerAsParallelCapable(); // enable fine-grained locking
    }
}
```

All built-in loaders (Bootstrap, Platform, Application) are parallel-capable since Java 7.

---

### `URLClassLoader` — Most Common Custom Loader

```java
URL[] urls = { new URL("file:///opt/plugins/myplugin.jar") };
URLClassLoader loader = new URLClassLoader(urls, getClass().getClassLoader());
Class<?> clazz = loader.loadClass("com.plugin.PluginImpl");

// Important: close when done (Java 7+) to release JAR file handles
loader.close();
```

`URLClassLoader.close()` was added in Java 7 to release file handles (especially on Windows, where open file handles prevent deletion/replacement of JARs).

---

### Secure ClassLoader & Code Source

`SecureClassLoader` extends `ClassLoader` to associate a `CodeSource` (URL + code signing certificates) with each class. `URLClassLoader` extends `SecureClassLoader`. This is the foundation of Java's sandboxing (now mostly legacy after Java Security Manager was deprecated in Java 17).

---

### Module System Impact (Java 9+)

In the module system, class loading is further constrained by module boundaries:
- A class is accessible only if the module that owns it **exports** the package.
- Even if two classes are on the same ClassLoader, if they are in different modules and the package is not exported, reflection access is restricted.
- `--add-opens` and `--add-exports` flags are JVM arguments to open/export module packages at runtime (common in frameworks needing deep reflection).

---

## 12. Interview Questions — Standard

**Q1: What is a ClassLoader and what does it do?**
A ClassLoader is a Java object responsible for finding `.class` file bytes, loading them into the JVM, and creating a `java.lang.Class` object representing the type. The full process includes loading, verification, preparation, resolution, and initialization.

**Q2: Explain the parent delegation model.**
When a ClassLoader is asked to load a class, it first delegates to its parent before attempting to load it itself. This chain goes all the way to the Bootstrap ClassLoader. A loader only loads a class if its entire parent chain fails. This guarantees JDK classes are always loaded by Bootstrap, preventing spoofing.

**Q3: What is the difference between `Class.forName()` and `ClassLoader.loadClass()`?**
`Class.forName()` loads and **initializes** the class (runs `<clinit>`). `ClassLoader.loadClass()` loads the class but does **not** initialize it. `Class.forName()` is used for JDBC driver registration precisely because the driver registers itself in a static block — initialization is required.

**Q4: What are the three built-in ClassLoaders?**
Bootstrap (native C++, loads JDK core), Platform/Extension (loads JDK platform modules or `lib/ext`), and Application/System (loads app classpath). Each is the parent of the next.

**Q5: Why does `String.class.getClassLoader()` return `null`?**
`String` is loaded by the Bootstrap ClassLoader, which is implemented in native C++ and has no Java object representation. By convention, `null` from `getClassLoader()` means the Bootstrap ClassLoader.

**Q6: What should you override when writing a custom ClassLoader — `loadClass()` or `findClass()`?**
Always override `findClass()`. Overriding `loadClass()` risks breaking the parent delegation model. `findClass()` is called only after the parent delegation chain has already failed — it is the correct extension point for custom class discovery logic.

**Q7: What is the Thread Context ClassLoader and why does it exist?**
The TCCL is a ClassLoader stored on each thread, accessible via `Thread.currentThread().getContextClassLoader()`. It was introduced to allow code loaded by a parent ClassLoader (e.g., Bootstrap-loaded JNDI, JDBC) to access classes from child ClassLoaders (e.g., app-provided drivers). It is a workaround for the upward-only parent delegation limitation.

---

## 13. Interview Questions — Tricky / Scenario-Based

**Q1: Two classes named `com.example.Foo` are loaded by different ClassLoaders. Can you cast an instance of one to the other?**
No. In Java, class identity = binary name + ClassLoader. `Foo` loaded by `LoaderA` is a completely different type from `Foo` loaded by `LoaderB`. Attempting the cast throws `ClassCastException`. The fix is to share an interface loaded by a common parent ClassLoader and cast to that interface.

**Q2: You redeploy a WAR on Tomcat but still see old behavior. What could cause this?**
A ClassLoader leak — the old `WebAppClassLoader` has not been GC'd because something is still referencing it (e.g., an un-deregistered JDBC driver, a running thread from a ThreadPoolExecutor, a resident ThreadLocal, or an MBean still registered in JMX). The new deployment creates a new ClassLoader, but the old one is still alive and may interfere.

**Q3: A class's `static {}` block threw an exception. What happens on the second attempt to use that class?**
The first attempt throws `ExceptionInInitializerError` wrapping the original exception. The JVM marks the class as erroneously initialized. Every subsequent attempt to use that class throws `NoClassDefFoundError`, even if the original problem is transient. The only fix is to restart the JVM (or the ClassLoader that loaded the class).

**Q4: Why can Bootstrap-loaded code (like `DriverManager`) not directly use `Class.forName()` to load a JDBC driver from the app classpath?**
`Class.forName(name)` (single-argument form) uses the **caller's** ClassLoader. Since `DriverManager` is loaded by Bootstrap, the caller's ClassLoader is Bootstrap, which cannot see the app classpath. The three-argument form `Class.forName(name, true, tccl)` with the Thread Context ClassLoader solves this.

**Q5: What happens if you override `loadClass()` in a custom ClassLoader and always load the class yourself, skipping parent delegation?**
You break the security guarantee. Your loader could load its own `java.lang.String`, but it would be a different type from the JDK's `String`. Code passing your `String` to JDK methods would get `ClassCastException`. Worse, if you shadow `java.lang.Object`, the JVM rejects it with a `SecurityException` because JDK classes cannot be redefined by custom loaders.

**Q6: How does Spring Boot's DevTools achieve hot-reload without restarting the JVM?**
DevTools uses **two ClassLoaders**: a base ClassLoader for JARs that don't change (third-party dependencies), and a **restart ClassLoader** for your app's classes. On code change, DevTools discards the restart ClassLoader, creates a new one, and re-loads your application classes. The JVM never restarts — only the restart ClassLoader and everything it loaded is thrown away (assuming no leaks).

---

## 14. Quick Revision Summary

- Java is always pass-by-value; ClassLoaders are the mechanism by which `.class` bytes become `Class` objects in the JVM.
- Class lifecycle: **Loading → Verification → Preparation → Resolution → Initialization**.
- `<clinit>` (static initializers) runs in Initialization, is thread-safe, and runs exactly once.
- Three built-in loaders: **Bootstrap → Platform → Application**. Each is parent of the next.
- **Parent delegation**: always try the parent first. Guarantees JDK classes are never shadowed.
- Class identity = **binary name + ClassLoader**. Same name, different loader = different type = `ClassCastException` on cross-boundary casts.
- **TCCL** exists so Bootstrap-loaded SPI code can access app-level implementations.
- Override **`findClass()`**, not `loadClass()`, in custom ClassLoaders to preserve delegation.
- **`ClassNotFoundException`** = class not found at load time. **`NoClassDefFoundError`** = found at compile time, missing at runtime OR class initialization previously failed.
- ClassLoader leaks: a single static/ThreadLocal/shutdown hook reference to a class pins the entire ClassLoader in Metaspace — always clean up on undeploy.
- `getClassLoader()` returning `null` = class loaded by Bootstrap ClassLoader.
- Java 9+ module system adds access control on top of ClassLoader visibility.
