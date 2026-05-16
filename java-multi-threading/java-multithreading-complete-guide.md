# Java Multi-Threading — Complete Interview Guide

---

## Table of Contents

1. [What is Multi-threading? Why does it exist?](#1-what-is-multi-threading)
2. [Process vs Thread vs Fiber](#2-process-vs-thread-vs-fiber)
3. [Thread Lifecycle](#3-thread-lifecycle)
4. [Creating Threads](#4-creating-threads)
5. [Thread Methods — `sleep`, `join`, `yield`, `interrupt`](#5-thread-methods)
6. [The Memory Visibility Problem — `volatile` & Happens-Before](#6-volatile--happens-before)
7. [`synchronized` Keyword — Intrinsic Locks](#7-synchronized--intrinsic-locks)
8. [`wait()`, `notify()`, `notifyAll()`](#8-wait-notify-notifyall)
9. [`ReentrantLock`, `ReadWriteLock`, `StampedLock`](#9-reentrantlock-readwritelock-stampedlock)
10. [Executor Framework — `ExecutorService`, `ThreadPoolExecutor`](#10-executor-framework)
11. [`Callable` and `Future`](#11-callable-and-future)
12. [`CompletableFuture` — Async Non-Blocking Pipelines](#12-completablefuture)
13. [Synchronization Utilities](#13-synchronization-utilities)
14. [Atomic Classes & CAS](#14-atomic-classes--cas)
15. [Thread-Safe Collections](#15-thread-safe-collections)
16. [`ThreadLocal`](#16-threadlocal)
17. [`ForkJoinPool` and Work Stealing](#17-forkjoinpool--work-stealing)
18. [Deadlock, Livelock, Starvation](#18-deadlock-livelock-starvation)
19. [Virtual Threads — Project Loom (Java 21)](#19-virtual-threads--project-loom)
20. [Common Interview Questions (Q&A)](#20-common-interview-questions)
21. [Common Pitfalls & Gotchas](#21-common-pitfalls--gotchas)
22. [Quick Revision Summary](#22-quick-revision-summary)

---

## 1. What is Multi-threading?

**Multi-threading** is the ability of a CPU (or a single process) to execute multiple threads concurrently.

- A **thread** is the smallest unit of execution within a process.
- A **process** has its own memory space; threads within the same process **share heap memory** but each has its own **stack**.

**Why does it exist?**

| Problem | Without Multi-threading | With Multi-threading |
|---|---|---|
| CPU-bound computation | Sequential, slow | Parallelized across cores |
| I/O-bound operations | Thread blocks waiting for disk/network | Other threads can run |
| Responsiveness | UI freezes during background work | UI thread stays free |
| Resource utilization | Only 1 of 8 cores used | All cores utilized |

**Real-world analogy:**  
A restaurant kitchen. One chef (single thread) takes an order, cooks it start-to-finish, then takes the next order. Multiple chefs (multi-threading) work simultaneously — one handles prep, one handles grilling, one handles plating. The kitchen throughput is far higher.

---

## 2. Process vs Thread vs Fiber

| Aspect | Process | Thread (Platform) | Virtual Thread (Fiber) |
|---|---|---|---|
| Memory | Own heap & stack | Shared heap, own stack | Shared heap, tiny stack (few KB) |
| OS resource | Heavyweight | Moderate (~1MB stack) | Very lightweight (managed by JVM) |
| Context switch cost | Expensive | Moderate | Very cheap |
| Creation cost | High | Moderate | Near zero |
| Communication | IPC (sockets, pipes) | Shared memory | Shared memory |
| Java support | `ProcessBuilder` | `Thread` | `Thread.ofVirtual()` (Java 21) |

---

## 3. Thread Lifecycle

```
                                        ┌──────────────────────────────────────────┐
                                        │            Thread.start()                 │
         new Thread()                   ▼                                          │
  ┌──────────────┐             ┌──────────────────┐                               │
  │     NEW      │────start()──▶    RUNNABLE       │                               │
  └──────────────┘             │  (Ready to run /  │                               │
                                │  currently runs) │                               │
                                └────────┬─────────┘                               │
                                         │                                          │
               ┌─────────────────────────┼──────────────────────────┐              │
               │                         │                          │              │
               ▼                         ▼                          ▼              │
  ┌─────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐    │
  │  BLOCKED            │  │  WAITING             │  │  TIMED_WAITING       │    │
  │ (waiting for lock)  │  │ (wait(), join())      │  │ (sleep(n), wait(n))  │    │
  └──────────┬──────────┘  └──────────┬───────────┘  └──────────┬───────────┘    │
             │                        │                          │                 │
             └────────────────────────┴──────────────────────────┘                 │
                                         │                                          │
                                         ▼                                          │
                                ┌──────────────────┐                               │
                                │   TERMINATED     │◀──────────────────────────────┘
                                │  (run() complete │
                                │   or exception)  │
                                └──────────────────┘
```

**Key distinctions:**
- **BLOCKED**: waiting to acquire a `synchronized` lock held by another thread.
- **WAITING**: indefinitely waiting — needs explicit `notify()` / `interrupt()` to wake.
- **TIMED_WAITING**: waiting with a timeout — wakes up automatically after time expires.

---

## 4. Creating Threads

### Method 1: Extend `Thread`

**What it is:** You subclass `Thread` and override `run()`. The task logic and the thread machinery live in the same class.

```java
public class DownloadThread extends Thread {
    private final String url;

    public DownloadThread(String url) {
        this.url = url;
        setName("Downloader-" + url); // always name your threads!
        setDaemon(true);              // daemon: JVM won't wait for it to finish
    }

    @Override
    public void run() {
        System.out.println(getName() + " downloading: " + url);
        // ... download logic
    }
}

// Usage
new DownloadThread("https://example.com/file.zip").start();
```

---

### Method 2: Implement `Runnable`

**What it is:** You implement the `Runnable` interface (just the `run()` method) and pass it to a `Thread` object. The task and the thread are two separate objects.

```java
public class OrderProcessor implements Runnable {
    private final Order order;

    public OrderProcessor(Order order) {
        this.order = order;
    }

    @Override
    public void run() {
        processOrder(order);
    }
}

// Usage
Thread t = new Thread(new OrderProcessor(order), "OrderProcessor-" + order.getId());
t.start();
```

---

### Thread vs Runnable — Full Trade-off Analysis

This is one of the most common Java interview questions. The answer goes deeper than "Java doesn't have multiple inheritance."

#### 1. The Single Inheritance Problem

Java allows extending only **one** class. If your task class extends `Thread`, it can never extend anything else.

```java
// PROBLEM: DownloadTask already needs to extend BaseTask for shared logic
public class DownloadTask extends BaseTask { // already using the one inheritance slot
    // Cannot also extend Thread!
}

// With Runnable — no problem, inheritance slot is free
public class DownloadTask extends BaseTask implements Runnable {
    @Override
    public void run() { /* task logic */ }
}
```

In real enterprise code, task classes often extend framework base classes (`BaseJob`, `AbstractProcessor`, etc.). `Runnable` keeps this possible; `Thread` blocks it.

---

#### 2. Separation of Concerns (the design principle reason)

Extending `Thread` **violates the Single Responsibility Principle**. The class is responsible for both:
- *What* to do (the task logic), AND
- *How* to run it (thread lifecycle, naming, priority, daemon flag)

With `Runnable`, these are cleanly separated:
- `OrderProcessor` → knows only what to do.
- `Thread` / `ExecutorService` → knows how and when to run it.

```java
// Extending Thread: tight coupling — task IS a thread
public class ReportGenerator extends Thread {
    public void run() { generateReport(); }
    // Now this class is also a Thread — you can call start(), interrupt(), join() on it directly
    // That's not ReportGenerator's job!
}

// Runnable: loose coupling — task is just a task
public class ReportGenerator implements Runnable {
    public void run() { generateReport(); }
    // ReportGenerator knows nothing about threads. Clean.
}
```

---

#### 3. Reusability — Same Task, Different Executors

A `Runnable` is a plain task object. You can submit it to any executor or run it in any context:

```java
Runnable task = new OrderProcessor(order);

// Option 1: run on a new raw thread
new Thread(task).start();

// Option 2: submit to a thread pool (most common in production)
executorService.submit(task);

// Option 3: run on the current thread (for testing, or sequential fallback)
task.run();

// Option 4: schedule it
scheduledExecutor.schedule(task, 5, TimeUnit.MINUTES);

// Option 5: wrap in a virtual thread (Java 21)
Thread.ofVirtual().start(task);
```

A class that extends `Thread` can only be used as a `Thread`. You can't submit a `Thread` subclass directly to `ExecutorService.submit()` as a task in a clean, decoupled way.

---

#### 4. Object Overhead

When you extend `Thread`, every instance of your task class **carries the full weight of a Thread object** — including the native thread handle, stack reference, thread group, etc. — even before `start()` is ever called.

With `Runnable`, the task object is lightweight. The `Thread` object (and its native resources) only exists while the thread is running.

```java
// Each DownloadThread object carries Thread's full state (~hundreds of bytes + native handle)
List<DownloadThread> tasks = new ArrayList<>();
for (String url : urls) tasks.add(new DownloadThread(url)); // heavy objects

// Each OrderProcessor is a plain POJO — tiny
List<Runnable> tasks = new ArrayList<>();
for (Order o : orders) tasks.add(new OrderProcessor(o)); // lightweight
```

---

#### 5. Testability

A `Runnable` is just an interface — you can unit test `run()` directly without spinning up OS threads:

```java
// Test the task logic in isolation — no threads involved
@Test
void testOrderProcessorLogic() {
    OrderProcessor processor = new OrderProcessor(mockOrder);
    processor.run(); // just a method call, no thread created
    verify(mockOrder).markProcessed();
}
```

Testing a `Thread` subclass is messier — you deal with thread state, `start()`, `join()`, timing issues, and test frameworks that don't play well with raw thread creation.

---

#### 6. When Extending `Thread` IS Justified

There are rare cases where extending `Thread` makes sense:

```java
// Justified: you genuinely need to customize Thread behavior itself
public class InstrumentedThread extends Thread {
    private final Metrics metrics;

    public InstrumentedThread(Runnable r, String name, Metrics metrics) {
        super(r, name);
        this.metrics = metrics;
    }

    @Override
    public void run() {
        long start = System.nanoTime();
        try {
            super.run();
        } finally {
            metrics.record(getName(), System.nanoTime() - start);
        }
    }
}
// Here you're customizing the Thread itself, not just writing a task — this is valid.
```

But even here, a custom `ThreadFactory` passed to `ThreadPoolExecutor` is the more modern approach.

---

#### Summary Table

| Aspect | `extends Thread` | `implements Runnable` |
|---|---|---|
| Inheritance | Burns the one slot | Free for other use |
| Separation of concerns | Violated (task + thread = same class) | Clean (task separate from executor) |
| Reusability | Tightly tied to Thread | Submittable to any executor |
| Object weight | Heavy (carries Thread state) | Lightweight POJO |
| Testability | Harder (thread lifecycle involved) | Easy (just call `run()`) |
| Works with `ExecutorService` | Awkward | Native fit |
| Works with virtual threads | No direct path | `Thread.ofVirtual().start(runnable)` |
| **Verdict** | Avoid in application code | **Always prefer this** |

> **Interview one-liner:** "Prefer `Runnable` because it separates *what* to do from *how* to run it, preserves the single inheritance slot, works natively with `ExecutorService`, and produces lightweight, testable task objects."

---

### Method 3: Lambda (most modern, inline tasks)

```java
Thread t = new Thread(() -> System.out.println("Processing payment..."), "PaymentThread");
t.start();
```

---

### Method 4: `Callable` + `Future` (returns result / throws checked exceptions)

```java
Callable<Double> priceCalculator = () -> {
    // can throw checked exceptions!
    return fetchLatestPrice("AAPL");
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Double> future = executor.submit(priceCalculator);
Double price = future.get(); // blocks until result is ready
```

---

### Daemon Threads

```java
Thread monitor = new Thread(() -> {
    while (true) {
        logMemoryUsage();
        Thread.sleep(5000);
    }
});
monitor.setDaemon(true); // MUST be called before start()
monitor.start();
// JVM exits without waiting for this thread
```

A **daemon thread** is a background service thread (e.g., GC thread, Finalizer). The JVM shuts down when **all non-daemon threads finish**. Daemon threads are then killed abruptly — never use them for tasks requiring cleanup or I/O commits.

---

## 5. Thread Methods

### `Thread.sleep(millis)`

```java
// Rider-tracking service polling location every 2 seconds
while (!rideComplete) {
    updateRiderLocation();
    Thread.sleep(2000); // releases NO locks, just pauses
}
```

- **Does NOT release any lock it holds** — unlike `wait()`.
- Throws `InterruptedException` — always handle or propagate it.
- Use `TimeUnit.SECONDS.sleep(2)` for readability.

---

### `thread.join()`

```java
// In a ride-sharing app, wait for all ETA calculations before responding
Thread t1 = new Thread(() -> calculateETA("Route A"));
Thread t2 = new Thread(() -> calculateETA("Route B"));
t1.start();
t2.start();

t1.join(); // current thread waits until t1 finishes
t2.join(); // then waits for t2

System.out.println("All ETAs calculated, sending response");
```

- `join(timeout)` — wait at most `timeout` ms; don't wait forever in production.
- The calling thread enters **WAITING** state.

---

### `Thread.yield()`

```java
Thread.yield(); // hint to scheduler: "I can pause, let others run"
```

- Just a **hint** to the OS scheduler; not guaranteed to do anything.
- Used in spin-wait loops to reduce CPU burning.
- Almost never used in application code.

---

### `thread.interrupt()`

#### What is it?

Interruption is Java's **cooperative cancellation mechanism** for threads. It does NOT forcibly kill a thread — it sets a single boolean flag (the **interrupt flag**) on the target thread and says "please stop when you get a chance." The thread itself decides what to do with that signal.

Think of it like tapping someone on the shoulder: you're not dragging them away, you're signalling them. They choose to respond.

#### What problem does it solve?

Before interrupts, the only way to stop a thread was `Thread.stop()` — which was deprecated because it forcibly killed the thread at any point, potentially corrupting shared state (e.g., stop mid-write to a file). Interruption is **safe** because the thread can finish what it's doing, release locks, flush buffers, and then stop cleanly.

#### How it works internally

Every `Thread` object has a private boolean field: the **interrupt flag** (also called interrupt status). It is `false` by default.

```
thread.interrupt()  →  sets thread's interrupt flag to true

Thread.currentThread().isInterrupted()  →  reads the flag (does NOT clear it)
Thread.interrupted()  →  reads the flag AND clears it to false
```

When a thread is sleeping/waiting (in WAITING or TIMED_WAITING state) and another thread calls `interrupt()` on it:
1. The JVM **clears the interrupt flag** (sets it back to false).
2. Throws `InterruptedException` in the sleeping/waiting thread.

This is why you must restore the flag after catching `InterruptedException` — the JVM cleared it for you.

#### The 3 Key Methods

| Method | Who calls it | What it does |
|---|---|---|
| `thread.interrupt()` | External thread (caller/manager) | Sets the target thread's interrupt flag to `true` |
| `Thread.currentThread().isInterrupted()` | The thread itself | Reads the flag; does **NOT** clear it |
| `Thread.interrupted()` (static) | The thread itself | Reads the flag AND **clears it** to `false` |

#### Case 1: Thread is RUNNABLE (busy computing)

The interrupt flag is set, but nothing throws. The thread must periodically **check the flag itself**:

```java
// CPU-bound task — no blocking calls, so no InterruptedException will ever be thrown
// Must check the flag manually in the loop
public class PriceCalculator implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) { // check on every iteration
            // heavy computation: simulate Monte Carlo pricing
            double price = monteCarloPricing(option);
            resultAccumulator.add(price);
        }
        // loop exits cleanly when interrupted
        System.out.println("Calculation cancelled. Partial results saved.");
    }
}

Thread calcThread = new Thread(new PriceCalculator());
calcThread.start();

// 2 seconds later, user cancels the request
Thread.sleep(2000);
calcThread.interrupt(); // sets flag → loop condition fails → exits cleanly
```

**Key point:** For a RUNNABLE thread, interruption only works if the thread cooperates by checking `isInterrupted()`. If the loop never checks the flag, the thread runs forever regardless of interrupts.

#### Case 2: Thread is WAITING / TIMED_WAITING (blocked in sleep/wait)

When the thread is sleeping or waiting, the JVM intervenes immediately:

```java
public class RetryTask implements Runnable {
    @Override
    public void run() {
        try {
            for (int attempt = 1; attempt <= 5; attempt++) {
                boolean success = callExternalApi();
                if (success) return;

                System.out.println("Attempt " + attempt + " failed. Waiting 10 seconds...");
                Thread.sleep(10_000); // ← if interrupt() called here:
                                      //   1. flag is cleared by JVM
                                      //   2. InterruptedException is thrown
                                      //   3. thread jumps to catch block
            }
        } catch (InterruptedException e) {
            // JVM already cleared the flag when throwing this exception
            Thread.currentThread().interrupt(); // RESTORE the flag
            System.out.println("Retry cancelled while waiting. Shutting down cleanly.");
            // release resources, close connections, etc.
        }
    }
}
```

The same applies to `wait()`, `join()`, `BlockingQueue.take()`, `Lock.lockInterruptibly()`, `Semaphore.acquire()`, and `Condition.await()` — all of them wake up with `InterruptedException` when interrupted.

#### Case 3: Thread is BLOCKED on `synchronized` (waiting for a lock)

**`interrupt()` has NO effect** on a thread waiting for a `synchronized` lock. The thread will stay BLOCKED until the lock is released — it cannot be interrupted out of a `synchronized` wait.

```java
// Thread 2 is blocked waiting for this lock
synchronized (sharedResource) {
    Thread.sleep(30_000); // Thread 1 holds this lock for 30 seconds
}

// Meanwhile:
thread2.interrupt(); // USELESS — thread2 is BLOCKED, not WAITING
                     // It will stay BLOCKED until thread1 releases the lock
                     // The interrupt flag will be set, but nothing happens until thread2 gets the lock
```

This is one reason to prefer `ReentrantLock.lockInterruptibly()` over `synchronized` when you need cancellability:

```java
// With ReentrantLock — CAN be interrupted while waiting for lock
lock.lockInterruptibly(); // throws InterruptedException if interrupted while waiting
try {
    doWork();
} finally {
    lock.unlock();
}
```

#### The Golden Rule: Always Restore the Interrupt Flag

```java
// WRONG #1 — swallowed, flag is lost
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // do nothing — interrupt signal is GONE forever
    // any caller checking isInterrupted() sees false — they'll never know
}

// WRONG #2 — logging but not restoring
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    log.warn("Interrupted"); // logged but flag still not restored
}

// CORRECT — restore the flag
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore flag so callers see it
    return; // exit the current operation
}

// ALSO CORRECT — propagate upward (let the caller handle it)
public void doWork() throws InterruptedException {
    Thread.sleep(1000); // just let InterruptedException bubble up
}
```

**Why does restoring matter?**  
Your code runs inside a framework (Spring, Tomcat, ExecutorService). After your code returns, the framework checks `isInterrupted()` to decide whether to shut down the thread, drain the queue, or cancel the task. If you swallow the interrupt, the framework never sees it and the thread keeps running when it should stop.

#### Real-life Scenario: Order Processing Service Shutdown

```java
@Service
public class OrderProcessingService {

    private final BlockingQueue<Order> orderQueue = new LinkedBlockingQueue<>();
    private Thread workerThread;

    @PostConstruct
    public void start() {
        workerThread = new Thread(this::processOrders, "OrderProcessor");
        workerThread.start();
    }

    private void processOrders() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                // take() blocks until an order arrives OR interrupted
                Order order = orderQueue.take(); // throws InterruptedException when interrupted
                processOrder(order);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore flag
            log.info("Order processor shutting down. Draining remaining orders...");
            // drain remaining orders from queue before exiting
            orderQueue.forEach(this::processOrder);
        }
        log.info("Order processor stopped cleanly.");
    }

    @PreDestroy
    public void stop() {
        workerThread.interrupt(); // signals the worker to stop
        try {
            workerThread.join(5000); // wait up to 5 seconds for clean shutdown
            if (workerThread.isAlive()) {
                log.warn("Worker did not stop in time!");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**What happens during shutdown:**
1. Spring calls `@PreDestroy` → `stop()` is called.
2. `workerThread.interrupt()` sets the flag.
3. If worker is blocked in `orderQueue.take()` → `InterruptedException` thrown immediately.
4. Worker catches it, restores flag, drains remaining orders.
5. Worker exits `processOrders()` → thread terminates.
6. `join(5000)` in `stop()` returns.
7. Clean shutdown complete.

#### `interrupt()` vs `volatile boolean` flag — Which to use?

Both are valid cancellation patterns. Here's when each fits:

```java
// Pattern 1: volatile flag
// PROBLEM: doesn't wake up a thread blocked in sleep/wait/take
private volatile boolean running = true;

public void run() {
    while (running) {
        // if thread is sleeping here when running=false, it won't see it until sleep ends
        Thread.sleep(10_000); // up to 10 seconds delay!
    }
}
public void stop() { running = false; }

// Pattern 2: interrupt()
// BENEFIT: immediately wakes thread from sleep/wait/take
public void run() {
    try {
        while (!Thread.currentThread().isInterrupted()) {
            Thread.sleep(10_000);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
public void stop() { thread.interrupt(); } // wakes immediately

// Pattern 3: BOTH (belt and suspenders, common in production)
private volatile boolean running = true;

public void run() {
    try {
        while (running && !Thread.currentThread().isInterrupted()) {
            Thread.sleep(10_000);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
public void stop() {
    running = false;
    thread.interrupt(); // ensure it wakes up immediately if sleeping
}
```

| Approach | Wakes from sleep/wait? | Works for CPU-only loops? | Framework compatibility |
|---|---|---|---|
| `volatile boolean` | No (waits for sleep to expire) | Yes | Partial |
| `interrupt()` | **Yes — immediately** | Yes (if flag is checked) | **Full** |
| Both combined | Yes | Yes | Best |

**CRITICAL:** When you catch `InterruptedException`, always call `Thread.currentThread().interrupt()` to restore the interrupted flag. Otherwise, the interruption signal is swallowed and callers that check `isInterrupted()` won't know.

---

### Thread Priority

```java
thread.setPriority(Thread.MAX_PRIORITY); // 10
thread.setPriority(Thread.NORM_PRIORITY); // 5 (default)
thread.setPriority(Thread.MIN_PRIORITY);  // 1
```

- Just a **hint** to the OS scheduler — not guaranteed.
- Priority inversion is a real production bug (low-priority thread holding a lock needed by a high-priority thread).
- **Don't rely on priority** for correctness.

---

## 6. `volatile` & Happens-Before

### The Visibility Problem

```java
// WITHOUT volatile — BUG in multi-core systems
class StopFlag {
    private boolean running = true; // each core may cache this!

    public void stop() { running = false; }

    public void run() {
        while (running) { // thread may read stale cached value from its L1 cache!
            doWork();
        }
    }
}
```

On a multi-core CPU, each core has its own L1/L2 cache. A thread on Core 1 may never see the write to `running` made by a thread on Core 2 because caches are not synchronized.

### `volatile` to the Rescue

```java
class StopFlag {
    private volatile boolean running = true; // writes go to main memory, reads bypass cache

    public void stop() { running = false; }

    public void run() {
        while (running) { // always reads from main memory
            doWork();
        }
    }
}
```

**What `volatile` guarantees:**
1. **Visibility**: Any write to a volatile variable is immediately visible to all other threads.
2. **Happens-Before**: A write to `volatile x` happens-before every subsequent read of `volatile x`.
3. **No caching**: Reads/writes go directly to main memory.

**What `volatile` does NOT guarantee:**
- **Atomicity** for compound operations.

```java
volatile int counter = 0;

// STILL NOT THREAD-SAFE! Two steps: read + write
counter++; // read counter, increment, write back — race condition!
```

Use `AtomicInteger` for this.

### Happens-Before Guarantee

The JMM (Java Memory Model) defines a happens-before relationship. If action A happens-before action B, then the effects of A are visible to B.

Key happens-before rules:
- Within a thread: each statement HB the next.
- `Thread.start()` HB any action in the started thread.
- All actions in a thread HB `Thread.join()` returning.
- Unlock of a monitor HB every subsequent lock of that monitor.
- Write to `volatile` HB every subsequent read of that variable.
- `x.notify()` HB a thread waking from `x.wait()`.

---

## 7. `synchronized` — Intrinsic Locks

Every Java object has an **intrinsic lock** (monitor). `synchronized` uses this lock to ensure mutual exclusion.

### Synchronized Method

```java
// Bank account — only one thread should modify balance at a time
public class BankAccount {
    private double balance;

    public synchronized void deposit(double amount) {
        // only one thread executes this at a time — uses 'this' as lock
        balance += amount;
    }

    public synchronized void withdraw(double amount) {
        if (balance >= amount) balance -= amount;
    }

    public synchronized double getBalance() {
        return balance;
    }
}
```

### Synchronized Block (finer granularity, better performance)

```java
public class OrderService {
    private final List<Order> pendingOrders = new ArrayList<>();
    private final List<Order> completedOrders = new ArrayList<>();
    private final Object pendingLock = new Object();   // dedicated lock object
    private final Object completedLock = new Object(); // separate lock

    public void addPending(Order o) {
        synchronized (pendingLock) {   // only lock what's needed
            pendingOrders.add(o);
        }
    }

    public void markCompleted(Order o) {
        synchronized (pendingLock) {
            pendingOrders.remove(o);
        }
        synchronized (completedLock) {
            completedOrders.add(o);
        }
    }
}
```

### Static Synchronized (locks the Class object)

```java
public class ConnectionPool {
    private static int activeConnections = 0;

    public static synchronized void checkOut() {
        // locks ConnectionPool.class, not any instance
        activeConnections++;
    }
}
```

### Key Properties

| Property | Description |
|---|---|
| **Mutual Exclusion** | Only one thread holds the lock at a time |
| **Reentrant** | Same thread can acquire the same lock multiple times (no self-deadlock) |
| **Visibility** | On lock release, all writes are flushed to main memory; on lock acquire, latest values are read |
| **Not interruptible** | A thread waiting for a `synchronized` lock **cannot** be interrupted |

### Reentrancy Example

```java
public synchronized void methodA() {
    methodB(); // same thread, re-acquires same lock — allowed!
}

public synchronized void methodB() {
    // works fine, same thread holds the lock already
}
```

---

## 8. `wait()`, `notify()`, `notifyAll()`

These are **Object methods** used for inter-thread communication via the monitor.

**The Producer-Consumer problem:**

```java
public class OrderQueue {
    private final Queue<Order> queue = new LinkedList<>();
    private final int MAX_SIZE = 10;

    // Called by Producer thread
    public synchronized void produce(Order order) throws InterruptedException {
        while (queue.size() == MAX_SIZE) {
            wait(); // releases lock AND waits — thread goes to WAITING state
        }
        queue.add(order);
        notifyAll(); // wake up all waiting threads (consumers)
    }

    // Called by Consumer thread
    public synchronized Order consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait(); // releases lock AND waits
        }
        Order order = queue.poll();
        notifyAll(); // wake up producers
        return order;
    }
}
```

### `wait()` vs `sleep()`

| Aspect | `wait()` | `sleep()` |
|---|---|---|
| Releases lock? | **YES** | **NO** |
| Must be in `synchronized`? | **YES** (else IllegalMonitorStateException) | No |
| Woken by | `notify()` / `notifyAll()` | Timeout expires |
| Belongs to | `Object` | `Thread` |

### Why `while` loop instead of `if`?

Always use `while` around `wait()`. The reason: **spurious wakeups** — a thread can wake from `wait()` without being notified (platform-level, rare but real). The `while` re-checks the condition.

```java
// CORRECT
while (queue.isEmpty()) {
    wait();
}

// WRONG — can proceed with empty queue due to spurious wakeup
if (queue.isEmpty()) {
    wait();
}
```

### `notify()` vs `notifyAll()`

- `notify()` — wakes **one** arbitrary waiting thread. Use only when all waiting threads are equivalent (same condition, same action).
- `notifyAll()` — wakes **all** waiting threads. Safer default — all re-check their condition and the right one proceeds.

---

## 9. `ReentrantLock`, `ReadWriteLock`, `StampedLock`

`java.util.concurrent.locks` package provides explicit lock objects with more features than `synchronized`.

### `ReentrantLock`

```java
import java.util.concurrent.locks.ReentrantLock;

public class TicketBookingService {
    private final ReentrantLock lock = new ReentrantLock();
    private int availableSeats = 100;

    public boolean bookSeat(String userId) {
        lock.lock(); // explicit lock
        try {
            if (availableSeats > 0) {
                availableSeats--;
                return true;
            }
            return false;
        } finally {
            lock.unlock(); // ALWAYS in finally block to avoid lock leaks
        }
    }

    // TryLock — non-blocking attempt
    public boolean tryBookSeat(String userId) {
        if (lock.tryLock()) { // doesn't block, returns false if locked
            try {
                availableSeats--;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false; // seat was busy, don't wait
    }

    // Interruptible lock — can be cancelled
    public void bookWithCancellation() throws InterruptedException {
        lock.lockInterruptibly(); // unlike synchronized, CAN be interrupted
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    }
}
```

### `ReentrantLock` Fairness

#### What is Fairness?

**Fairness** in a lock context means: "In what order do waiting threads acquire the lock?"

- **Fair lock**: Threads acquire the lock in **FIFO (First-In, First-Out) order** — the same order they requested it. No thread gets "cut in line." The thread that has been waiting the longest gets the lock next.
- **Unfair lock** (default): Any waiting thread can grab the lock — it's a **free-for-all**. A thread that just arrived might jump ahead of threads that have been waiting for milliseconds or seconds.

**Real-world analogy:**
- Fair lock = a bank queue where people stand in line and are served in order.
- Unfair lock = a chaotic bakery where anyone can shout "I'm next!" and the cashier picks whoever is loudest.

#### Why Would You Want an Unfair Lock?

**Throughput.** Unfair locks are significantly faster (often 2-3x higher throughput) because:
1. No overhead tracking who's first in the queue.
2. The thread that just released the lock can immediately re-acquire it (cache-friendly).
3. Context switching is reduced — threads on the same CPU core are preferred.

```java
// Unfair lock: throughput-optimized
ReentrantLock unfairLock = new ReentrantLock(false); // or just new ReentrantLock()

// Benchmark-style example: 8 threads hammering a counter
AtomicInteger fairCounter = new AtomicInteger(0);
AtomicInteger unfairCounter = new AtomicInteger(0);
ReentrantLock fairLock = new ReentrantLock(true);
ReentrantLock unfairLock = new ReentrantLock(false);

// With fair lock: ~500,000 increments/sec (slower due to FIFO overhead)
// With unfair lock: ~2,000,000 increments/sec (cache locality, less context switching)
```

#### Why Would You Want a Fair Lock?

**Predictability and starvation prevention.**

With an unfair lock, a thread can theoretically wait forever if other threads keep "cutting in line." Imagine:

```java
// Unfair lock - potential starvation scenario
ReentrantLock unfairLock = new ReentrantLock(false);

// Thread A holds lock, does work for 1ms, releases
// Threads B, C, D are waiting
// Thread B wakes up but... Thread E (just arrived) grabs lock first
// B goes back to waiting
// A finishes, re-acquires, does more work
// Still B, C, D waiting; Thread F grabs it
// Threads B, C, D might wait for a very long time or even indefinitely

while (workAvailable) {
    unfairLock.lock();
    try {
        doWork(); // 1ms
    } finally {
        unfairLock.unlock();
    }
}
```

With a fair lock, once a thread requests the lock, it **will** get it within a predictable timeframe because threads ahead of it will acquire and release in order.

#### Internal Implementation — How Does Fairness Work?

**Unfair lock:**
```
Lock state: FREE
Thread A calls lock() → acquires immediately, state becomes LOCKED
Thread B calls lock() → BLOCKED (A holds lock)
Thread C calls lock() → BLOCKED (A holds lock)
Thread A calls unlock() → lock state becomes FREE
  
  Thread B and C wake up. Who gets it?
  In unfair mode: a thread can "barge" in — whoever the OS scheduler wakes first
  OR if a new Thread D calls lock() at exactly this moment, D might grab it first!
```

**Fair lock (internal queue):**
```
Lock state: FREE, Queue: empty
Thread A calls lock() → acquires immediately, state becomes LOCKED
Thread B calls lock() → added to FIFO queue, BLOCKED
Thread C calls lock() → added to FIFO queue (behind B), BLOCKED
Thread D calls lock() → added to FIFO queue (behind C), BLOCKED
Thread A calls unlock() → lock state becomes FREE
  
  Only B (front of queue) is allowed to acquire
  B acquires, Thread E cannot barge ahead even if it calls lock() now
  B will use the lock, then release
  Then C acquires (next in queue)
```

This internal queue is what makes fair locks slower — every lock/unlock operation must interact with the queue data structure.

#### Real-world Scenario: Ticket Booking System

```java
// Unfair lock — high throughput but customers might get angry
public class TicketBookingServiceUnfair {
    private final ReentrantLock lock = new ReentrantLock(false); // unfair
    private int availableSeats = 100;

    public boolean bookSeat(String userId) {
        lock.lock();
        try {
            if (availableSeats > 0) {
                availableSeats--;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
}

// What happens: 1000 users clicking "Book" simultaneously
// Some users see availability and book successfully
// Others see "no seats" even though seats exist (unfair queue jumping)
// Same user might retry and succeed if they "barge in"

// Fair lock — ensures every request is processed in order
public class TicketBookingServiceFair {
    private final ReentrantLock lock = new ReentrantLock(true); // fair
    private int availableSeats = 100;

    public boolean bookSeat(String userId) {
        lock.lock();
        try {
            if (availableSeats > 0) {
                availableSeats--;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
}

// What happens: 1000 users click "Book" simultaneously
// Users are processed in strict FIFO order
// First-come, first-served guarantee
// But latency is slightly higher (maybe 10ms vs 5ms per request)
```

#### Fair vs Unfair — The Trade-off Table

| Aspect | Fair Lock | Unfair Lock |
|---|---|---|
| **Order guarantee** | FIFO — thread gets lock in requested order | No order — any thread can grab it |
| **Throughput** | Lower (50-70% of unfair) | Higher (baseline) |
| **Latency variance** | Predictable (bounded) | High variance (some threads very fast, others slow) |
| **Starvation risk** | None — if you request, you'll get it | Possible — a thread can be skipped repeatedly |
| **CPU cache efficiency** | Lower — less likely to reuse same CPU core | Higher — threads stay on same core |
| **Use case** | Fairness-critical systems (banking, medical) | High-throughput systems (e.g., web servers) |
| **Default** | Must explicitly set `new ReentrantLock(true)` | Default (`new ReentrantLock()` or `new ReentrantLock(false)`) |

#### Benchmarking Example: Fair vs Unfair in Action

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class FairnessComparison {

    static class CounterWithLock {
        private final ReentrantLock lock;
        private long count = 0;

        CounterWithLock(boolean fair) {
            this.lock = new ReentrantLock(fair);
        }

        void increment() {
            lock.lock();
            try {
                count++;
            } finally {
                lock.unlock();
            }
        }

        long getCount() {
            lock.lock();
            try {
                return count;
            } finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // Unfair benchmark
        long start = System.nanoTime();
        CounterWithLock unfairCounter = new CounterWithLock(false);
        runBenchmark(unfairCounter, 8, 1_000_000);
        long unfairTime = System.nanoTime() - start;

        // Fair benchmark
        start = System.nanoTime();
        CounterWithLock fairCounter = new CounterWithLock(true);
        runBenchmark(fairCounter, 8, 1_000_000);
        long fairTime = System.nanoTime() - start;

        System.out.printf("Unfair: %d ns, Fair: %d ns, Fair is %.2fx slower%n",
            unfairTime, fairTime, (double) fairTime / unfairTime);
        // Output: Unfair: 400ms, Fair: 1200ms, Fair is 3.0x slower
    }

    static void runBenchmark(CounterWithLock counter, int threads, int operationsPerThread)
            throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                for (int j = 0; j < operationsPerThread; j++) {
                    counter.increment();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
    }
}
```

**Output:**
```
Unfair: 400ms, Fair: 1200ms, Fair is 3.0x slower
```

The fair lock's overhead comes from:
1. Maintaining the FIFO queue.
2. Every thread that calls `lock()` must check the queue (is it my turn?).
3. Less cache locality — threads don't repeat on the same CPU core.

#### When to Use Each

**Use UNFAIR (default):**
- High-throughput systems where microseconds matter (web servers, databases, trading systems).
- Starvation is not a concern because work is short-lived and repeated.
- You're not fighting over a scarce resource.

```java
// E-commerce checkout system — very high throughput, millisecond-scale operations
private final ReentrantLock checkoutLock = new ReentrantLock(); // unfair, default
```

**Use FAIR:**
- Fairness is a business/regulatory requirement (banking, medical devices, auditing).
- You want predictable, bounded latency for each request.
- Preventing starvation is critical.
- Work duration is highly variable (some tasks take 1ms, others take 100ms).

```java
// Medical device controller — must process requests in order, predictably
private final ReentrantLock deviceLock = new ReentrantLock(true); // fair
```

**Use NEITHER (use something else):**
- `ConcurrentHashMap` or lock-free data structures for ultra-high throughput.
- `Semaphore` if fairness is optional and you want guaranteed throughput.
- `StampedLock` for read-heavy workloads.

#### Production Pitfalls with Fairness

**Pitfall 1: Fair lock doesn't prevent starvation of the process**

```java
// A fair lock ensures FIFO, but it doesn't help if threads never release
ReentrantLock fairLock = new ReentrantLock(true);

fairLock.lock();
try {
    Thread.sleep(60_000); // hold the lock for 60 seconds
} finally {
    fairLock.unlock();
}
// All waiting threads are starved for 60 seconds, fair or not
// Fairness only helps when lock holders eventually release
```

**Pitfall 2: Fair locks under extreme contention**

```java
// With 1000 threads all competing for a fair lock
// Context switches happen constantly: A gets lock, does work, releases, B gets lock...
// Fair lock may become a **bottleneck** because of queue overhead
// Solution: Use a thread pool with fewer threads, or partition the lock across buckets (striped locking)
```

**Pitfall 3: Mixing fair and unfair in the same code**

```java
// WRONG: inconsistent fairness policy
ReentrantLock lock1 = new ReentrantLock(true);   // fair
ReentrantLock lock2 = new ReentrantLock(false);  // unfair

// If you always acquire lock1 then lock2, thread starvation can happen on lock2
// while lock1 seems fair
```

#### Interview Question: "Should I use a fair or unfair lock?"

**Best answer:**
> "**Unfair by default** — it has better throughput and is the JVM default. Switch to **fair only if you can demonstrate starvation in production** or have a specific business requirement. Measure both before deciding. In most cases, starvation doesn't happen because operations are short-lived and repeated; fairness overhead often outweighs the benefit."

### `ReentrantLock` with `Condition` (replaces `wait`/`notify`)

```java
import java.util.concurrent.locks.Condition;

public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity)
                notFull.await(); // like wait(), releases lock
            queue.add(item);
            notEmpty.signal(); // like notify(), only for notEmpty condition
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty())
                notEmpty.await();
            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

**Advantage:** Multiple `Condition` objects on one lock — you can signal specifically the producers OR specifically the consumers. With `notifyAll()`, you'd wake everyone.

---

### `ReentrantReadWriteLock`

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

// Product catalog: reads are frequent, writes are rare
public class ProductCatalog {
    private final Map<String, Product> catalog = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public Product getProduct(String id) {
        rwLock.readLock().lock(); // multiple threads can read simultaneously
        try {
            return catalog.get(id);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void updateProduct(String id, Product product) {
        rwLock.writeLock().lock(); // exclusive — blocks all readers and writers
        try {
            catalog.put(id, product);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

**Rules:**
- Multiple threads can hold **read lock** simultaneously (read-read = compatible).
- **Write lock** is exclusive — blocks all other readers AND writers.
- A thread holding a **write lock** can downgrade to a read lock (NOT vice versa).

---

### `StampedLock` (Java 8+) — Optimistic Reads

```java
import java.util.concurrent.locks.StampedLock;

public class FlightSeatMap {
    private final int[] seats = new int[200];
    private final StampedLock sl = new StampedLock();

    public int getSeatStatus(int seatNo) {
        long stamp = sl.tryOptimisticRead(); // NO lock acquired!
        int status = seats[seatNo];
        if (!sl.validate(stamp)) {           // check if a write happened
            stamp = sl.readLock();           // fall back to real read lock
            try {
                status = seats[seatNo];
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return status; // fast path: no lock needed if no concurrent write
    }

    public void reserveSeat(int seatNo) {
        long stamp = sl.writeLock();
        try {
            seats[seatNo] = 1;
        } finally {
            sl.unlockWrite(stamp);
        }
    }
}
```

**Key trade-off:**
- `StampedLock` is the most performant for read-heavy workloads.
- It is **NOT reentrant** — same thread cannot acquire it twice.
- It does NOT support `Condition` objects.

| Lock | Reentrant | Condition | Optimistic Read | Fairness |
|---|---|---|---|---|
| `synchronized` | Yes | `wait/notify` | No | No |
| `ReentrantLock` | Yes | Yes | No | Optional |
| `ReadWriteLock` | Yes | On write lock | No | Optional |
| `StampedLock` | **No** | No | **Yes** | No |

---

## 10. Executor Framework

Never create threads manually in production — use the Executor framework. It manages thread lifecycle, pooling, and queuing.

### Hierarchy

```
Executor
  └── ExecutorService
        ├── AbstractExecutorService
        │     └── ThreadPoolExecutor   ← THE core implementation
        │           └── ScheduledThreadPoolExecutor
        └── ForkJoinPool
```

### `ThreadPoolExecutor` Deep Dive

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                          // corePoolSize: threads always kept alive
    20,                         // maximumPoolSize: max threads when queue is full
    60L, TimeUnit.SECONDS,      // keepAliveTime: idle threads above coreSize are terminated
    new LinkedBlockingQueue<>(100), // work queue: holds tasks when all core threads busy
    new ThreadFactory() {       // custom thread factory (name your threads!)
        private final AtomicInteger count = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "OrderWorker-" + count.incrementAndGet());
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

### Task Submission Flow

```
New task submitted
        │
        ▼
corePool threads busy? ──No──▶ Assign to idle core thread
        │
       Yes
        ▼
Work queue full? ──No──▶ Put task in queue
        │
       Yes
        ▼
Max pool reached? ──No──▶ Create new thread (above coreSize, up to max)
        │
       Yes
        ▼
Apply RejectedExecutionHandler
```

### Rejection Policies

| Policy | Behavior | Use When |
|---|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` | Want to know when overloaded |
| `CallerRunsPolicy` | The **submitting thread** runs the task | Natural backpressure |
| `DiscardPolicy` | Silently drops the task | Task loss is acceptable |
| `DiscardOldestPolicy` | Drops the oldest queued task, retries | Most recent tasks matter more |

### Pre-built Pools via `Executors` Factory

```java
// 1. Fixed pool — good for CPU-bound tasks
ExecutorService fixed = Executors.newFixedThreadPool(8); // 8 = Runtime.getRuntime().availableProcessors()

// 2. Cached pool — good for short-lived, many async I/O tasks
// WARNING: unbounded thread creation! Can OOM under high load
ExecutorService cached = Executors.newCachedThreadPool();

// 3. Single thread — sequential processing with a thread per task abstraction
ExecutorService single = Executors.newSingleThreadExecutor();

// 4. Scheduled pool — run tasks after delay or periodically
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.scheduleAtFixedRate(() -> syncInventory(), 0, 15, TimeUnit.MINUTES);
scheduled.schedule(() -> sendReminder(userId), 30, TimeUnit.MINUTES);
```

**WARNING about `Executors.newFixedThreadPool`:** It uses an **unbounded `LinkedBlockingQueue`**. If producers outpace consumers, the queue grows without limit → OOM. In production, always use `ThreadPoolExecutor` directly with a bounded queue.

### Graceful Shutdown

```java
executor.shutdown(); // stops accepting new tasks, finishes in-flight tasks

try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow(); // interrupt all running tasks
        if (!executor.awaitTermination(10, TimeUnit.SECONDS))
            log.error("Executor did not terminate cleanly");
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

## 11. `Callable` and `Future`

### Why `Callable`?

`Runnable` was the original way to submit work to a thread. It has two hard limitations:
1. Returns `void` — you cannot get a result back.
2. Cannot throw checked exceptions.

`Callable<V>` solves both. It returns a typed result and can throw any exception.

```java
// Runnable — no return, no checked exceptions
Runnable r = () -> System.out.println("done");

// Callable — returns a result, can throw checked exceptions
Callable<Integer> c = () -> {
    return computeExpensiveResult(); // can throw IOException, etc.
};
```

You submit a `Callable` to an `ExecutorService` and get back a `Future<V>`.

---

### `Future<V>` — The "Result Ticket"

**Mental model:** Think of ordering food at a restaurant counter.

1. You place the order (submit the task).
2. The cashier gives you a **numbered token** (the `Future`).
3. You go sit down, chat, check your phone — you don't stand frozen at the counter.
4. When your number is called (or you walk back up), you hand in the token and get the food (`future.get()`).

The `Future<T>` is **not the result**. It is a **handle to claim the result when you need it**.

---

### How `Future` Works — Step by Step

```
Main Thread                        Worker Thread
     │                                  │
     │  executor.submit(callable)       │
     │ ─────────────────────────────►  │ Task starts running
     │                                  │
     │  [Future returned immediately]   │ (still running...)
     │                                  │
     │  doSomeOtherWork()               │ (still running...)
     │                                  │
     │  future.get()  ◄── BLOCKS here  │
     │ ◄─────────────────────────────  │ Task completes, result returned
     │                                  │
     │  [continues with result]         │
```

**Key insight:** Submitting the task and retrieving the result are **decoupled**. You can submit many tasks, do other work, and only block when you actually need each result.

---

### Code Example — Parallel Checks in a Loan Application

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// All three submitted immediately — they run IN PARALLEL on separate threads
Future<String> creditCheck = executor.submit(() -> runCreditCheck(userId));
Future<String> fraudCheck  = executor.submit(() -> runFraudCheck(userId));
Future<Double> creditLimit = executor.submit(() -> calculateCreditLimit(userId));

// While they run, do something useful on the main thread
logApplicationReceived(userId);
validateRequestSignature(request);

// NOW collect results — blocks only if the task hasn't finished yet
String creditResult = creditCheck.get(5, TimeUnit.SECONDS);
String fraudResult  = fraudCheck.get(5, TimeUnit.SECONDS);
Double limit        = creditLimit.get(5, TimeUnit.SECONDS);
```

If all three take 2 seconds each, total time is ~2 seconds (parallel), not 6 seconds (sequential).

---

### `Future` API

| Method | What it does | Production note |
|---|---|---|
| `get()` | Block forever until result | **Never use without timeout** — can hang a thread forever |
| `get(timeout, unit)` | Block up to timeout, throws `TimeoutException` | Always prefer this |
| `isDone()` | Non-blocking: is it complete? | Use for polling patterns |
| `isCancelled()` | Was it cancelled? | Check before using result |
| `cancel(mayInterrupt)` | Try to cancel; `true` = interrupt if running | Task must check `Thread.interrupted()` to respond |

**Exception handling with `Future`:**

```java
try {
    String result = future.get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    // Task took too long
} catch (ExecutionException e) {
    // Task itself threw — get the real cause:
    Throwable realCause = e.getCause(); // don't ignore this!
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // always restore interrupt flag
}
```

---

### `invokeAll` and `invokeAny`

```java
List<Callable<PriceQuote>> tasks = suppliers.stream()
    .map(s -> (Callable<PriceQuote>) () -> s.getQuote(productId))
    .collect(toList());

// invokeAll: submit all, block until ALL complete (or timeout)
List<Future<PriceQuote>> results = executor.invokeAll(tasks, 3, TimeUnit.SECONDS);

// invokeAny: submit all, return as soon as the FIRST ONE succeeds (cancels the rest)
PriceQuote bestQuote = executor.invokeAny(tasks, 3, TimeUnit.SECONDS);
```

---

### The Hard Limit of `Future` — Why It Falls Short

`Future` is fine for simple "submit one task, get one result" patterns. The moment you need to **chain** results or **react** when a result is ready without blocking, it completely falls apart.

**Problem 1 — You cannot chain without blocking:**

```java
// Goal: fetch user → fetch their profile → fetch recommendations
// With Future, you are forced to block at every step:

Future<User>    userFuture    = executor.submit(() -> fetchUser(id));
User user = userFuture.get();                                    // BLOCK #1 — must wait

Future<Profile> profileFuture = executor.submit(() -> fetchProfile(user.getId()));
Profile profile = profileFuture.get();                           // BLOCK #2 — must wait

Future<List<Rec>> recsFuture  = executor.submit(() -> fetchRecs(profile));
List<Rec> recs = recsFuture.get();                               // BLOCK #3 — must wait

// You've serialised three calls. Zero parallelism benefit.
// Might as well have called them synchronously.
```

**Problem 2 — You cannot combine two parallel results without blocking:**

```java
// Goal: fetch user AND account in parallel, then combine them
Future<User>    userFuture    = executor.submit(() -> fetchUser(id));
Future<Account> accountFuture = executor.submit(() -> fetchAccount(id));

// To combine, you MUST block on both — no other option
User    user    = userFuture.get();    // block
Account account = accountFuture.get(); // block
Dashboard dashboard = new Dashboard(user, account);
// There is no way to say: "when BOTH are done, call this function automatically"
```

**Problem 3 — No built-in timeout on the whole pipeline, no fallback.**

These are exactly the problems `CompletableFuture` was built to solve.

---

## 12. `CompletableFuture` — Non-Blocking Async Pipelines

### What Is It, and Why Did It Come?

`CompletableFuture<T>` (Java 8+) is a `Future<T>` that you can also:

1. **Attach callbacks to** — "when this finishes, run this function automatically"
2. **Chain stages on** — build a pipeline where each step feeds into the next
3. **Combine with other futures** — fan-out/fan-in without blocking
4. **Complete externally** — another thread can call `.complete(value)` to resolve it

**The name itself:**
- "Future" — it holds a value that will arrive in the future.
- "Completable" — it can be explicitly completed, and it supports chaining on completion.

---

### Mental Model — A Factory Assembly Line

Think of a production line. Each station takes a part, does its work, and passes it to the next station — **automatically**, the moment it's ready. No station stands idle waiting; it just gets activated when the previous one finishes.

```
[supplyAsync]    [thenApply]    [thenCompose]    [thenApply]
  Fetch User  ──►  Validate  ──►  Fetch Profile ──►  Format
  (Thread A)       (Thread A)     (Thread B)          (Thread B)
```

With `Future`, you'd have to block a thread at every station to pick up the result and pass it forward. `CompletableFuture` does that handoff automatically.

---

### Starting a `CompletableFuture`

```java
// No return value (fire and forget)
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> sendEmail(userId));

// With return value — most common
CompletableFuture<User> cf2 = CompletableFuture.supplyAsync(() -> fetchUser(userId));

// IMPORTANT: always pass a custom executor for I/O-bound work
// Default uses ForkJoinPool.commonPool() — a shared, limited-size pool
// Blocking I/O on it can starve other async tasks across the entire JVM
ExecutorService ioPool = Executors.newFixedThreadPool(20);
CompletableFuture<User> cf3 = CompletableFuture.supplyAsync(() -> fetchUser(userId), ioPool);

// Already have a value? Wrap it (useful in tests or for cache-hit fast paths)
CompletableFuture<User> cf4 = CompletableFuture.completedFuture(cachedUser);
```

---

### Chaining — `thenApply`, `thenCompose`, `thenCombine`

These three are the backbone of `CompletableFuture`. Understanding the difference between them is critical.

---

#### `thenApply` — Transform the result (like `map` on a Stream)

**Use when:** your next step is a simple, **synchronous function** that maps the current result to a new value.

```
CF<A>  ──thenApply(A → B)──►  CF<B>
```

```java
CompletableFuture<String> userIdCF = CompletableFuture.supplyAsync(() -> "user-42");

// Each step transforms the value synchronously
CompletableFuture<User>   userCF   = userIdCF.thenApply(id -> userRepo.findById(id));
CompletableFuture<String> nameCF   = userCF.thenApply(user -> user.getFullName());
```

---

#### `thenCompose` — Flat-map (like `flatMap` on a Stream)

**Use when:** your next step **itself returns a `CompletableFuture`**.

If you use `thenApply` when the lambda returns a `CompletableFuture`, you get `CF<CF<R>>` — a nested future that is awkward to work with. `thenCompose` unwraps it to `CF<R>`.

```
CF<A>  ──thenApply(A → CF<B>)──►    CF<CF<B>>   ❌ nested, wrong
CF<A>  ──thenCompose(A → CF<B>)──►  CF<B>        ✅ flat, correct
```

```java
// fetchProfile(userId) returns CompletableFuture<Profile>
// So we MUST use thenCompose, not thenApply

CompletableFuture<User> userCF =
    CompletableFuture.supplyAsync(() -> fetchUser(id));

CompletableFuture<Profile> profileCF =
    userCF.thenCompose(user -> fetchProfile(user.getId())); // fetchProfile returns CF<Profile>

// If we had used thenApply here, we'd get CF<CF<Profile>> — useless nested type
```

**Intuition rule:** Is the lambda returning a plain value? → `thenApply`. Is it returning a `CompletableFuture`? → `thenCompose`.

---

#### `thenCombine` — Merge two INDEPENDENT futures

**Use when:** two async tasks run in parallel and you want to combine their results when **both** are done.

```
CF<A>  ────────────────────────────┐
                                    ├──► thenCombine((a, b) → C) ──► CF<C>
CF<B>  ────────────────────────────┘
```

```java
ExecutorService pool = Executors.newFixedThreadPool(10);

// These two start running in parallel immediately
CompletableFuture<User>    userCF    = CompletableFuture.supplyAsync(() -> fetchUser(id), pool);
CompletableFuture<Account> accountCF = CompletableFuture.supplyAsync(() -> fetchAccount(id), pool);

// thenCombine fires automatically when BOTH are done — no blocking
CompletableFuture<Dashboard> dashboardCF =
    userCF.thenCombine(accountCF, (user, account) -> new Dashboard(user, account));
```

This is the solution to the `Future` "Problem 2" we saw above — and it requires zero blocking.

---

### Quick Reference: Which Method to Use?

| Scenario | Method | Example |
|---|---|---|
| Synchronous transform of result | `thenApply` | `.thenApply(user -> user.getName())` |
| Next step also returns a `CF` | `thenCompose` | `.thenCompose(user -> fetchProfile(user.getId()))` |
| Merge 2 independent parallel tasks | `thenCombine` | `userCF.thenCombine(accountCF, Dashboard::new)` |
| Consume result (no return value) | `thenAccept` | `.thenAccept(order -> audit(order))` |
| Run action after (don't need result) | `thenRun` | `.thenRun(() -> log("pipeline done"))` |

---

### Sync vs Async Variants — Which Thread Runs the Callback?

Every chain method has an `Async` variant. This controls **which thread executes the callback**.

| Method | Thread running the callback |
|---|---|
| `thenApply(fn)` | The same thread that completed the previous stage |
| `thenApplyAsync(fn)` | ForkJoinPool.commonPool() |
| `thenApplyAsync(fn, executor)` | Your custom executor |

```java
// Scenario: fetch user (DB pool thread), then call profile service (I/O)

// BAD — callback runs on the DB pool thread, which now blocks on a network call
CompletableFuture<Profile> cf1 = CompletableFuture
    .supplyAsync(() -> fetchUser(id), dbPool)
    .thenApply(user -> callProfileService(user)); // runs on dbPool thread — tying up a DB thread!

// GOOD — move the blocking I/O callback to the right pool
CompletableFuture<Profile> cf2 = CompletableFuture
    .supplyAsync(() -> fetchUser(id), dbPool)
    .thenApplyAsync(user -> callProfileService(user), ioPool); // runs on ioPool
```

**Rule:** Use `thenApplyAsync` when the callback does blocking I/O or expensive CPU work. Use `thenApply` only for fast, cheap transformations.

---

### Fan-out / Fan-in — `allOf` and `anyOf`

**`allOf` — wait for ALL futures, then collect results:**

```java
List<CompletableFuture<PriceQuote>> futures = suppliers.stream()
    .map(s -> CompletableFuture.supplyAsync(() -> s.getQuote(productId), pool))
    .collect(toList());

// allOf returns CF<Void> — you still need to collect results manually
CompletableFuture<List<PriceQuote>> allQuotesCF =
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream()
                               .map(CompletableFuture::join) // safe: all are already done
                               .collect(toList()));
```

**`anyOf` — return as soon as the FIRST one finishes:**

```java
// Try primary + backup in parallel, take whoever responds first
CompletableFuture<User> primary = CompletableFuture.supplyAsync(() -> primaryService.fetch(id));
CompletableFuture<User> backup  = CompletableFuture.supplyAsync(() -> backupService.fetch(id));

CompletableFuture<User> fastest = CompletableFuture
    .anyOf(primary, backup)
    .thenApply(result -> (User) result); // anyOf returns CF<Object>, cast is needed
```

---

### Error Handling in Pipelines

When an exception is thrown at any stage, it propagates through the pipeline — unless you handle it.

```
supplyAsync → thenApply → thenCompose  → [Exception thrown here]
                                               │
                                      ┌────────▼────────┐
                                      │  exceptionally  │  recover, return fallback
                                      │  handle         │  handle both success + failure
                                      │  whenComplete   │  side-effect only
                                      └─────────────────┘
```

```java
CompletableFuture<Order> result = CompletableFuture
    .supplyAsync(() -> fetchOrder(id))          // might throw

    .exceptionally(ex -> {
        // Recover — return a fallback value; the pipeline continues normally from here
        log.warn("fetchOrder failed: {}", ex.getMessage());
        return Order.fallback(id);              // must return same type as the stage
    })

    .handle((order, ex) -> {
        // ALWAYS called, whether success or failure (like try-catch-finally)
        // ex is null on success, non-null on failure
        if (ex != null) return handleError(ex);
        return enrichOrder(order);
    })

    .whenComplete((order, ex) -> {
        // Side-effect only — does NOT change the result flowing through the pipeline
        // Use for: logging, metrics, audit trails
        auditLog.record(order, ex);
    });
```

**Which to use:**
- `exceptionally` — recover from failure, return a default value.
- `handle` — handle both success and failure in one place (when logic is shared).
- `whenComplete` — logging, metrics, audit. Does not alter the result.

---

### Full Production Example — Dashboard API (3 parallel service calls)

```java
@GetMapping("/dashboard/{userId}")
public Dashboard getDashboard(@PathVariable String userId) {

    // All three start immediately and run in parallel
    CompletableFuture<UserProfile> profileCF =
        CompletableFuture.supplyAsync(() -> profileService.fetch(userId), ioPool)
            .exceptionally(ex -> UserProfile.empty(userId));   // degrade gracefully

    CompletableFuture<Account> accountCF =
        CompletableFuture.supplyAsync(() -> accountService.fetch(userId), ioPool)
            .exceptionally(ex -> Account.empty(userId));

    CompletableFuture<List<Order>> ordersCF =
        CompletableFuture.supplyAsync(() -> orderService.recent(userId, 10), ioPool)
            .exceptionally(ex -> Collections.emptyList());

    // Combine all three when done, with a 1.5s timeout on the whole pipeline
    return profileCF
        .thenCombine(accountCF,  (profile, account) -> new PartialDashboard(profile, account))
        .thenCombine(ordersCF,   (partial, orders)  -> partial.withOrders(orders))
        .orTimeout(1500, TimeUnit.MILLISECONDS)          // kills the pipeline if too slow
        .exceptionally(ex -> Dashboard.fallback(userId)) // final safety net
        .join(); // safe here: Tomcat request thread can block; we have an overall timeout
}
```

---

### `Future` vs `CompletableFuture` — Side-by-Side Comparison

| Dimension | `Future` | `CompletableFuture` |
|---|---|---|
| Get result | `get()` — always blocks | `join()` or attach callbacks (non-blocking) |
| Chain next step | Not possible — must block first | `thenApply`, `thenCompose` |
| Combine two parallel results | Must block on both, then combine manually | `thenCombine(cf2, fn)` |
| Wait for all / any | Manual with loops | `allOf(...)` / `anyOf(...)` |
| Error handling | Catch `ExecutionException` in `get()` | `exceptionally`, `handle` |
| Timeout | `get(5, SECONDS)` — blocks with timeout | `orTimeout(5, SECONDS)` — pipeline-level |
| Complete from outside | Not possible | `.complete(value)` |
| Truly non-blocking | No | Yes (if you don't call `join()` too early) |
| Java version | Java 5+ | Java 8+ |

---

### Common Pitfalls

| Pitfall | What goes wrong | Fix |
|---|---|---|
| Using `ForkJoinPool.commonPool()` for I/O | Starves all other async tasks in the JVM | Always pass a dedicated custom executor |
| Calling `join()` too early | Converts the whole pipeline into blocking code | Call `join()` only at the final aggregation point |
| No `exceptionally` or `handle` | Silent failures; `join()` throws `CompletionException` | Every pipeline must have a terminal error handler |
| `thenApply` when lambda returns `CF` | You get `CF<CF<R>>` — nested future, type error | If your lambda returns a `CF`, use `thenCompose` |
| MDC / security context loss across async hops | Log correlation IDs and auth info disappear | Wrap supplier to copy context before async call |

---

## 13. Synchronization Utilities

### `CountDownLatch` — Wait for N events to complete

```java
// E-commerce: wait for inventory, payment, and fraud checks before confirming order
int checks = 3;
CountDownLatch latch = new CountDownLatch(checks);

executor.submit(() -> { checkInventory(); latch.countDown(); });
executor.submit(() -> { processPayment(); latch.countDown(); });
executor.submit(() -> { runFraudCheck(); latch.countDown(); });

latch.await(10, TimeUnit.SECONDS); // main thread waits
confirmOrder(); // only reached when all 3 count down to 0

// KEY: CountDownLatch is ONE-TIME use. Cannot be reset.
```

**Real-world use:** Service initialization gates — don't start serving traffic until all dependent services are ready.

---

### `CyclicBarrier` — N threads meet at a checkpoint together

```java
// Risk calculation: 4 threads each compute a sub-portfolio; combine results at the barrier
CyclicBarrier barrier = new CyclicBarrier(4, () -> {
    // This runs when all 4 threads reach the barrier
    combineRiskResults();
    System.out.println("Round complete, starting next round...");
});

for (int i = 0; i < 4; i++) {
    int portfolioSlice = i;
    executor.submit(() -> {
        while (marketOpen) {
            computeRisk(portfolioSlice);
            barrier.await(); // wait for all 4 to finish this round
            // barrier resets automatically — can use for next round!
        }
    });
}

// KEY: CyclicBarrier is REUSABLE. Resets after each use.
```

**Difference from CountDownLatch:**

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Direction | Threads count DOWN, main waits | Threads wait FOR EACH OTHER |
| Reusable | No | Yes |
| Completion action | No built-in | Runnable on completion |
| Use case | Wait for N events | N threads sync at checkpoint |

---

### `Semaphore` — Control access to N resources

```java
// Rate limiting: only 10 concurrent DB connections allowed
Semaphore dbConnections = new Semaphore(10); // 10 permits

public Data queryDatabase(String sql) throws InterruptedException {
    dbConnections.acquire(); // blocks if 0 permits available
    try {
        return db.execute(sql);
    } finally {
        dbConnections.release(); // ALWAYS in finally
    }
}

// Tryacquire — non-blocking
if (dbConnections.tryAcquire(500, TimeUnit.MILLISECONDS)) {
    try { return db.execute(sql); }
    finally { dbConnections.release(); }
} else {
    throw new ServiceUnavailableException("DB connection pool exhausted");
}
```

**Use cases:** Connection pool throttling, API rate limiting, parking lot simulation (classic interview problem).

---

### `Phaser` — Flexible multi-phase coordination (Java 7+)

```java
// ETL pipeline: load → transform → validate → store (each phase needs all threads)
Phaser phaser = new Phaser(4); // 4 parties (threads)

for (int i = 0; i < 4; i++) {
    int worker = i;
    executor.submit(() -> {
        loadData(worker);
        phaser.arriveAndAwaitAdvance(); // Phase 1 complete

        transformData(worker);
        phaser.arriveAndAwaitAdvance(); // Phase 2 complete

        validateData(worker);
        phaser.arriveAndAwaitAdvance(); // Phase 3 complete

        storeData(worker);
        phaser.arriveAndDeregister();   // done, unregister
    });
}
```

`Phaser` is like a reusable `CyclicBarrier` where parties can dynamically register/deregister.

---

### `Exchanger` — Two threads swap data at a rendezvous point

```java
// Filling and draining pipeline: producer fills a buffer, swaps with consumer
Exchanger<List<Order>> exchanger = new Exchanger<>();

// Producer thread
executor.submit(() -> {
    List<Order> buffer = new ArrayList<>();
    while (true) {
        buffer.add(getNextOrder());
        if (buffer.size() == 100) {
            buffer = exchanger.exchange(buffer); // swap with consumer, get empty buffer back
            buffer.clear();
        }
    }
});

// Consumer thread
executor.submit(() -> {
    List<Order> buffer = new ArrayList<>();
    while (true) {
        buffer = exchanger.exchange(buffer); // wait for producer, get full buffer
        processAll(buffer);
    }
});
```

---

## 14. Atomic Classes & CAS

### The Problem with `volatile` for Compound Operations

```java
private volatile int ticketsSold = 0;

// BUG: read-increment-write is NOT atomic
public void sellTicket() {
    ticketsSold++; // multiple threads can read the same stale value
}
```

### CAS — Compare And Swap

#### What Is CAS?

CAS is a **single, indivisible CPU instruction** (`CMPXCHG` on x86) that does three things **atomically** — meaning the CPU completes all three steps with no other thread able to interrupt in between:

1. Read the current value at a memory address.
2. Compare it to an **expected** value you provide.
3. **If they match** → write a **new** value to that address → return `true`.
4. **If they don't match** → do nothing → return `false`.

No OS lock, no mutex, no kernel call — this is **pure hardware atomicity**.

---

#### Real-Life Analogy — The Version-Stamped Google Doc

Imagine two people editing the same Google Doc simultaneously.

- The doc is at **Version 5**.
- Person A opens it (reads version 5), drafts a change.
- Person B also opens it (reads version 5), makes a change first, and saves → doc is now **Version 6**.
- Person A tries to save. Google checks: "You based your edit on Version 5, but the doc is now Version 6 — someone else changed it."
- Google **rejects** Person A's save. Person A must **re-read the current version**, re-apply their change, and try again.

That's CAS. The "version number" is the expected value. If it still matches, your write goes through. If someone else changed it first, you retry.

---

#### Another Analogy — The Atomic Ticket Machine

A cinema has a single ticket dispenser showing the current available seat count. Two people press "Book" at exactly the same moment.

With a **lock (mutex):** The machine puts up a "BUSY" sign, serves one person, removes the sign, then serves the next. One person waits.

With **CAS:** Both read "50 seats available" simultaneously. Both try to atomically change `50 → 49`. The CPU allows only **one** to succeed. The other gets back "expected 50, but actual is now 49 — retry." That person re-reads (49), tries to change `49 → 48` — succeeds this time. No lock, no waiting, no sign.

---

#### How CAS Works at the CPU Level

```
// Pseudocode for what CMPXCHG does atomically:
fun CAS(memoryAddress, expected, newValue):
    // The following block is ONE indivisible CPU instruction:
    actual = read(memoryAddress)
    if actual == expected:
        write(memoryAddress, newValue)
        return true          // success: we changed it
    else:
        return false         // failure: someone else changed it first
```

In Java, `AtomicInteger.compareAndSet(expected, update)` directly maps to this CPU instruction via `sun.misc.Unsafe.compareAndSwapInt()`.

---

#### The CAS Retry Loop (Optimistic Locking)

Because CAS can fail, you wrap it in a **retry loop** — keep trying until your update wins:

```java
AtomicInteger counter = new AtomicInteger(0);

// Manual CAS loop — exactly what AtomicInteger.incrementAndGet() does internally
public int increment() {
    int expected;
    int newValue;
    do {
        expected = counter.get();       // Step 1: read current value
        newValue = expected + 1;        // Step 2: compute desired new value
    } while (!counter.compareAndSet(expected, newValue));
    //         └─ Step 3: atomically: if current still == expected, write newValue
    //                    if another thread changed it, loop and try again
    return newValue;
}
```

**What happens under high contention (many threads retrying):**
```
Thread A: read 0, compute 1, CAS(0→1) → SUCCESS
Thread B: read 0, compute 1, CAS(0→1) → FAIL (value is now 1) → retry
Thread B: read 1, compute 2, CAS(1→2) → SUCCESS
Thread C: read 0, compute 1, CAS(0→1) → FAIL → retry
Thread C: read 2, compute 3, CAS(2→3) → SUCCESS
```

No thread is blocked. They all make progress. This is called **optimistic locking** — you optimistically assume no contention, and retry only if you were wrong.

---

#### CAS vs `synchronized` — The Core Trade-off

| Aspect | `synchronized` (Pessimistic) | CAS (Optimistic) |
|---|---|---|
| **Assumption** | "Conflict will happen — lock first" | "Conflict is rare — try without locking" |
| **OS involvement** | Yes — mutex involves kernel | No — pure CPU instruction |
| **Thread state** | Blocked threads sleep (context switch) | Failed threads spin and retry (stays RUNNABLE) |
| **Overhead (no contention)** | Lock acquire + release overhead | Near zero — just one CPU instruction |
| **Overhead (high contention)** | Threads queue up, one runs at a time | Many threads spin → CPU waste |
| **Best for** | High-contention, long critical sections | Low-to-medium contention, tiny atomic ops |
| **Starvation risk** | No (JVM fair queueing) | Possible under extreme contention |

**Rule of thumb:** CAS wins when contention is low and the critical section is tiny (increment, swap a reference). `synchronized` wins when contention is high or the critical section is complex (multi-step logic).

---

#### The ABA Problem — CAS's Hidden Danger

**Scenario:**
1. Thread 1 reads value `A` from memory.
2. Thread 2 changes `A → B`.
3. Thread 2 changes `B → A` (back to original).
4. Thread 1's CAS: "Is it still `A`? Yes!" → succeeds — but the value **was changed and changed back**. Thread 1 has no idea.

**Why it matters in a linked list / stack:**

```
Initial stack: top → [A] → [B] → [C]

Thread 1: reads top = A, plans to pop A (new top should be B)
Thread 2: pops A (top = B), pops B (top = C), pushes A back (top = A → C)

Thread 1's CAS: "top is still A, so change to B" — SUCCEEDS
Result: top → [B]... but B was already removed! Dangling reference → memory corruption.
```

**Solution: `AtomicStampedReference`** — carry a version counter alongside the value:

```java
AtomicStampedReference<Node> top = new AtomicStampedReference<>(headNode, 0);

// CAS now requires BOTH value AND stamp to match
// Even if value goes A→B→A, the stamp goes 0→1→2, so it never matches the original (A, 0)
boolean success = top.compareAndSet(
    expectedNode, newNode,
    expectedStamp, expectedStamp + 1
);
```

Now step 4 above fails: Thread 1 expected stamp `0`, but the stamp is now `2` → retry.

---

No lock needed — purely hardware-level atomic.

### `AtomicInteger` and `AtomicLong`

```java
import java.util.concurrent.atomic.AtomicInteger;

public class TicketCounter {
    private final AtomicInteger sold = new AtomicInteger(0);
    private final int totalSeats = 500;

    public boolean sellTicket() {
        int current;
        do {
            current = sold.get();
            if (current >= totalSeats) return false; // sold out
        } while (!sold.compareAndSet(current, current + 1)); // CAS loop
        return true;
    }

    // Simpler: uses built-in CAS internally
    public int incrementAndGet() {
        return sold.incrementAndGet();
    }
}
```

### `AtomicReference`

```java
// Lock-free stack
public class LockFreeStack<T> {
    private static class Node<T> {
        T val; Node<T> next;
        Node(T val, Node<T> next) { this.val = val; this.next = next; }
    }

    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    public void push(T val) {
        Node<T> newNode;
        do {
            newNode = new Node<>(val, top.get()); // read current top
        } while (!top.compareAndSet(newNode.next, newNode)); // retry if top changed
    }
}
```

### `AtomicStampedReference` — Solving the ABA Problem

**ABA Problem:**
1. Thread 1 reads value `A`.
2. Thread 2 changes `A → B → A`.
3. Thread 1's CAS sees `A` and succeeds — but the value has changed and changed back.

```java
// Stamped reference includes a version stamp to detect intermediate changes
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);

int[] stampHolder = new int[1];
String current = ref.get(stampHolder); // get value AND stamp
int stamp = stampHolder[0];

// CAS succeeds only if BOTH value AND stamp match
boolean success = ref.compareAndSet(current, "B", stamp, stamp + 1);
```

### `LongAdder` / `LongAccumulator` (Java 8+)

```java
// For HIGH contention counters (e.g., page view count)
// LongAdder maintains separate counters per thread, merges on sum()
LongAdder pageViews = new LongAdder();

pageViews.increment(); // much lower contention than AtomicLong
long total = pageViews.sum(); // merge all thread-local counts
```

Use `LongAdder` over `AtomicLong` when the counter is updated by many threads but read infrequently.

---

## 15. Thread-Safe Collections

### `ConcurrentHashMap`

```java
ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// Java 8+: atomic compute operations
wordCount.merge(word, 1, Integer::sum);           // atomic: add 1 to existing or put 1
wordCount.compute(word, (k, v) -> v == null ? 1 : v + 1);
wordCount.computeIfAbsent(key, k -> new ArrayList<>()); // safe lazy init
```

#### How ConcurrentHashMap Uses CAS Internally

`ConcurrentHashMap` is the most important real-world example of CAS in the JDK. It uses **three different synchronization strategies** for three different situations — no single lock covers everything.

**Structure:** An array of buckets (`Node[] table`). Each bucket is the head of a linked list (or a Red-Black Tree when > 8 entries).

```
table[]
  [0] → null
  [1] → Node("apple",3) → Node("apricot",1) → null
  [2] → null
  [3] → Node("banana",7) → null
  ...
```

---

##### Case 1: Inserting into an EMPTY bucket — Pure CAS, zero locking

When the target bucket is `null`, `ConcurrentHashMap` uses CAS to atomically insert the new node:

```java
// Simplified from ConcurrentHashMap source (putVal method)
if (casTabAt(tab, index, null, new Node(hash, key, value))) {
    break; // CAS succeeded — node inserted with NO lock
}
// If another thread inserted into this bucket between our null-check
// and our CAS attempt, casTabAt returns false → we loop and retry
```

`casTabAt` internally calls `Unsafe.compareAndSwapObject(table, offset, null, newNode)` — a single CPU instruction. If the bucket is still `null`, the node is placed. If another thread beat us there, we re-enter the loop.

**Why CAS here and not `synchronized`?**  
Empty bucket inserts are the most common case (especially when the map isn't full). Using a lock for every one of them would be wasteful. CAS is a single CPU instruction — it's essentially free when there's no contention.

---

##### Case 2: Inserting / Updating an EXISTING bucket — `synchronized` on the bucket head

When the bucket already has a node, CAS alone can't work — you need to traverse the linked list and potentially modify multiple pointers. CAS only atomically modifies one memory location. So `ConcurrentHashMap` locks **just the head node of that bucket**:

```java
// Simplified from putVal source
Node headNode = tabAt(tab, index); // volatile read — no lock needed

synchronized (headNode) {          // lock ONLY this one bucket, not the whole map
    // traverse the linked list, update existing or append new node
    for (Node e = headNode; e != null; e = e.next) {
        if (e.hash == hash && e.key.equals(key)) {
            e.val = value;  // update existing
            break;
        }
    }
    // OR: append new node at the end of the list
}
```

**Key insight:** Only threads hashing to the **same bucket** compete for this lock. Threads in different buckets run completely in parallel — no contention at all. With a default capacity of 16 and a good hash function, the probability that two random keys land in the same bucket is ~6%. That means ~94% of concurrent writes don't block each other.

---

##### Case 3: Reads — Lock-free using `volatile`

All `Node` fields in `ConcurrentHashMap` are `volatile`:

```java
static class Node<K, V> {
    final int hash;
    final K key;
    volatile V val;       // ← volatile: any write is immediately visible to readers
    volatile Node<K,V> next; // ← volatile: next pointer changes are immediately visible
}
```

A `get()` never acquires any lock — it reads the volatile `val` directly. This means reads are **completely non-blocking**, regardless of how many threads are writing simultaneously.

---

##### Case 4: Treeification — CAS + synchronized

When a bucket's linked list exceeds 8 entries, `ConcurrentHashMap` converts it to a Red-Black Tree for O(log n) lookup instead of O(n). The conversion:
1. Uses `synchronized` on the bucket head (safe structural change).
2. Uses CAS to swap the head pointer from `Node` to `TreeBin` in the table.

---

#### The Three Strategies — Summary

```
┌──────────────────────────────────────────────────────────────────┐
│              ConcurrentHashMap Synchronization Strategy           │
├─────────────────────┬────────────────────────────────────────────┤
│ Operation           │ Strategy                                    │
├─────────────────────┼────────────────────────────────────────────┤
│ Read (get)          │ Volatile read — no lock, no CAS            │
│ Insert (empty slot) │ CAS on the table array slot                │
│ Insert/Update       │ synchronized on bucket head node           │
│ (non-empty slot)    │                                             │
│ Table resize        │ CAS + cooperative migration                │
│ Size counting       │ CAS on base count + LongAdder-style cells  │
└─────────────────────┴────────────────────────────────────────────┘
```

#### Why Not Just Lock the Whole Map?

Compare `ConcurrentHashMap` vs old `Hashtable` (which locks the entire map):

```java
// Hashtable: every operation locks the ENTIRE map
public synchronized V get(Object key) { ... }  // ALL threads wait, even for different keys
public synchronized V put(K key, V value) { ... } // same global lock

// ConcurrentHashMap: only the specific bucket is locked for writes, nothing for reads
// 16 threads can write to 16 different buckets simultaneously — no contention at all
```

**Throughput difference:** Under 16 threads, `ConcurrentHashMap` can be **10-100x faster** than `Hashtable` depending on the workload, because most operations run truly in parallel.

---

#### Production Usage Patterns

```java
ConcurrentHashMap<String, List<Order>> userOrders = new ConcurrentHashMap<>();

// WRONG: check-then-act is not atomic, race condition
if (!userOrders.containsKey(userId)) {
    userOrders.put(userId, new ArrayList<>()); // another thread might have put between check and put
}

// CORRECT: computeIfAbsent is a single atomic operation using CAS internally
List<Order> orders = userOrders.computeIfAbsent(userId, k -> new ArrayList<>());

// CORRECT: merge for counters — atomically adds or initialises
userOrders.merge(userId, newOrder, (existing, addition) -> {
    existing.add(addition.get(0));
    return existing;
});
```

---

### `CopyOnWriteArrayList` / `CopyOnWriteArraySet`

```java
// Event listeners: reads are very frequent, writes (add/remove) are rare
CopyOnWriteArrayList<EventListener> listeners = new CopyOnWriteArrayList<>();

listeners.add(new AuditListener());   // creates a FULL COPY of the array!
listeners.remove(oldListener);        // creates a FULL COPY of the array!

// Reads are lock-free — iteration uses the snapshot taken at iterator creation
for (EventListener l : listeners) {
    l.onEvent(event); // even if another thread adds/removes, no ConcurrentModificationException
}
```

**Use when:** Reads >> writes, small collection size.  
**Avoid when:** Large collection with frequent writes (O(n) copy per write).

---

### `BlockingQueue` Implementations

```java
// ArrayBlockingQueue: bounded, array-backed, FIFO
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);

// LinkedBlockingQueue: optionally bounded (default: Integer.MAX_VALUE!)
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(500); // always bound it!

// PriorityBlockingQueue: unbounded, priority-ordered
BlockingQueue<Order> orders = new PriorityBlockingQueue<>(100,
    Comparator.comparing(Order::getPriority).reversed());

// SynchronousQueue: NO buffer — each put() blocks until a take() matches
BlockingQueue<Task> handoff = new SynchronousQueue<>();

// DelayQueue: elements become available only after a delay
DelayQueue<ScheduledTask> delayed = new DelayQueue<>();
```

**BlockingQueue methods:**

| Operation | Throws Exception | Returns Special Value | Blocks | Times Out |
|---|---|---|---|---|
| Insert | `add(e)` | `offer(e)` | `put(e)` | `offer(e, t, u)` |
| Remove | `remove()` | `poll()` | `take()` | `poll(t, u)` |
| Examine | `element()` | `peek()` | — | — |

---

### `ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`

Lock-free, non-blocking queue based on CAS. Use for high-throughput, non-blocking producer-consumer where you don't need backpressure (no `put`/`take` blocking).

---

## 16. `ThreadLocal`

### What is it?

`ThreadLocal<T>` provides a variable that has a **separate copy for each thread**. No sharing → no synchronization needed.

```java
// Each request gets its own database connection
public class DatabaseConnectionHolder {
    private static final ThreadLocal<Connection> connectionHolder =
        ThreadLocal.withInitial(() -> dataSource.getConnection());

    public static Connection getConnection() {
        return connectionHolder.get();
    }

    public static void releaseConnection() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            conn.close();
            connectionHolder.remove(); // CRITICAL: always remove!
        }
    }
}
```

### Real-World Spring Use: `RequestContextHolder`

Spring uses `ThreadLocal` internally to store the current `HttpServletRequest`:

```java
// Spring's RequestContextHolder stores request per thread
HttpServletRequest request = ((ServletRequestAttributes)
    RequestContextHolder.getRequestAttributes()).getRequest();
```

### `ThreadLocal` Memory Leak — The Biggest Gotcha

In thread pool scenarios (e.g., Tomcat, Spring), threads are **reused**. If you store something in `ThreadLocal` and **forget to call `remove()`**, the next request handled by the same thread sees the old value.

Worse: `ThreadLocalMap` uses **weak keys** (the `ThreadLocal` object) but **strong values**. If the `ThreadLocal` is GC'd but the thread is not, the value lives forever → **memory leak**.

```java
// ALWAYS use try-finally pattern
public void handleRequest(Request req) {
    threadLocal.set(req.getUser()); // set
    try {
        processRequest(req);
    } finally {
        threadLocal.remove(); // ALWAYS remove in finally
    }
}
```

### `InheritableThreadLocal`

Child threads inherit the value from the parent thread:

```java
static InheritableThreadLocal<String> tenantId = new InheritableThreadLocal<>();

tenantId.set("tenant-A");
Thread child = new Thread(() -> System.out.println(tenantId.get())); // prints "tenant-A"
child.start();
```

**Problem:** With thread pools, new threads in the pool are created once and inherit from whoever created the pool — not per request. Use `TransmittableThreadLocal` (TTL library) for thread pool context propagation.

---

## 17. `ForkJoinPool` & Work Stealing

### Purpose

`ForkJoinPool` is designed for **recursive, divide-and-conquer** tasks — split a large task into smaller subtasks, solve recursively, merge results.

### Work Stealing

Each thread in the ForkJoinPool has its **own deque** (double-ended queue) of tasks. When a thread runs out of work, it **steals tasks from the tail of another thread's deque**. Maximizes CPU utilization.

### `RecursiveTask<V>` (returns result)

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

// Parallel sum of a large array
public class ParallelSum extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int start, end;

    public ParallelSum(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // BASE CASE: small enough to compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        // RECURSIVE CASE: split
        int mid = (start + end) / 2;
        ParallelSum left  = new ParallelSum(array, start, mid);
        ParallelSum right = new ParallelSum(array, mid, end);

        left.fork();         // submit left to pool asynchronously
        long rightResult = right.compute(); // compute right on this thread
        long leftResult  = left.join();     // wait for left result

        return leftResult + rightResult;
    }
}

// Usage
ForkJoinPool pool = new ForkJoinPool(); // uses available processors by default
long sum = pool.invoke(new ParallelSum(data, 0, data.length));
```

### `RecursiveAction` (no result)

```java
// Parallel image processing
public class ImageFilter extends RecursiveAction {
    @Override
    protected void compute() {
        if (imageSize <= THRESHOLD) {
            applyFilter();
        } else {
            invokeAll(new ImageFilter(topHalf), new ImageFilter(bottomHalf));
        }
    }
}
```

### Common ForkJoinPool (Java 8+)

```java
// Stream's parallel() uses the common ForkJoinPool
long sum = LongStream.rangeClosed(1, 10_000_000)
    .parallel()
    .sum();

// WARNING: using the common pool for blocking I/O tasks can starve
// other parallel streams across the entire JVM!
// Use a custom ForkJoinPool for blocking tasks:
ForkJoinPool custom = new ForkJoinPool(4);
custom.submit(() ->
    stream.parallel().forEach(this::processWithBlockingIO)
).get();
```

---

## 18. Deadlock, Livelock, Starvation

### Deadlock

Four **Coffman conditions** — ALL must hold for deadlock:
1. **Mutual Exclusion**: Resource held in non-shareable mode.
2. **Hold and Wait**: Thread holds one lock and waits for another.
3. **No Preemption**: Locks can't be forcibly taken away.
4. **Circular Wait**: Thread A waits for B, B waits for A.

```java
// Classic deadlock: two accounts, transfer in different directions simultaneously
public class DeadlockExample {
    public void transfer(Account from, Account to, double amount) {
        synchronized (from) {         // Thread 1 locks accountA
            synchronized (to) {       // Thread 1 waits for accountB
                from.debit(amount);   // Thread 2 locked accountB, waits for accountA
                to.credit(amount);    // DEADLOCK!
            }
        }
    }
}
```

**Fix 1: Lock Ordering** — always acquire locks in the same global order

```java
public void transfer(Account from, Account to, double amount) {
    Account first  = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Fix 2: `tryLock` with timeout** — give up if can't acquire both

```java
public boolean transfer(Account from, Account to, double amount) throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return true;
                    } finally { to.lock.unlock(); }
                }
            } finally { from.lock.unlock(); }
        }
        Thread.sleep(10 + new Random().nextInt(10)); // random backoff
    }
}
```

---

### Livelock

Threads are not blocked — they keep responding to each other but make **no progress**.

```java
// Two threads keep "politely" yielding to each other and never proceed
// Like two people in a hallway both stepping to the same side repeatedly
while (!acquired) {
    if (otherThread.isWorking()) {
        Thread.yield(); // give way
    } else {
        doWork();
    }
}
```

**Fix:** Introduce **randomized backoff** — add random sleep so they don't step in sync.

---

### Starvation

A thread is perpetually denied CPU time because other threads always get priority.

**Causes:**
- Thread priority set too low.
- A `synchronized` block holds a lock for too long, starving other threads.
- Unfair lock (a few threads always win CAS).

**Fix:** Use fair locks (`new ReentrantLock(true)`), avoid long critical sections, monitor thread activity.

---

### Detecting Deadlocks Programmatically

```java
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
if (deadlockedThreads != null) {
    ThreadInfo[] info = threadMXBean.getThreadInfo(deadlockedThreads, true, true);
    for (ThreadInfo ti : info) {
        System.out.println("Deadlocked: " + ti.getThreadName());
    }
}
```

Also: `jstack <pid>` in the terminal — prints all thread stacks and deadlock detection summary.

---

## 19. Virtual Threads — Project Loom (Java 21)

### The Problem with Platform Threads

In traditional Java, each `Thread` maps 1:1 to an OS thread. OS threads are expensive:
- ~1MB stack per thread.
- Context switching is slow (kernel mode switch).
- A typical JVM can handle only a few thousand OS threads.

In reactive/async style (`CompletableFuture`, WebFlux), code becomes complex and hard to debug.

### Virtual Threads

Virtual threads are **JVM-managed, extremely lightweight threads**:
- ~few KB per virtual thread (vs ~1MB for platform thread).
- JVM mounts virtual threads onto a small pool of **carrier threads** (platform threads).
- When a virtual thread **blocks** (I/O, `sleep`, `wait`), it is **unmounted** from the carrier thread. The carrier thread picks up another virtual thread. **The OS thread is never blocked.**

```java
// Java 21 — create virtual threads
Thread vt = Thread.ofVirtual().name("VT-1").start(() -> {
    System.out.println("Running on: " + Thread.currentThread());
    // blocking I/O here is fine! JVM will unmount this VT during the wait
    String result = httpClient.send(request).body(); // does NOT block the carrier thread
});

// Via executor (recommended)
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        int taskId = i;
        executor.submit(() -> processRequest(taskId)); // 1 million virtual threads — fine!
    }
} // auto-close shuts down executor
```

### Structured Concurrency (Java 21 Preview)

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<User>    user    = scope.fork(() -> fetchUser(id));
    Future<Account> account = scope.fork(() -> fetchAccount(id));

    scope.join();           // wait for both
    scope.throwIfFailed();  // propagate any failure

    return new Dashboard(user.resultNow(), account.resultNow());
}
// If either task fails, the other is cancelled automatically
```

### Virtual Threads vs Platform Threads

| Aspect | Platform Thread | Virtual Thread |
|---|---|---|
| Managed by | OS | JVM |
| Stack size | ~1MB | ~few KB |
| Max count | ~thousands | Millions |
| Blocking I/O | Wastes OS thread | JVM unmounts, no waste |
| Context switch | Expensive (kernel) | Cheap (JVM) |
| Use case | CPU-bound, small count | I/O-bound, high concurrency |

### Pinning — The Virtual Thread Gotcha

A virtual thread is **pinned** (cannot unmount) when it's inside:
1. A `synchronized` block/method.
2. A native method or foreign function call.

If pinned and blocked, it **blocks the carrier thread** — losing the benefit. Fix: replace `synchronized` with `ReentrantLock`.

```java
// BEFORE (pins carrier thread if blocked inside)
synchronized (lock) {
    Thread.sleep(100);
}

// AFTER (virtual thread can unmount)
reentrantLock.lock();
try {
    Thread.sleep(100); // or any blocking op
} finally {
    reentrantLock.unlock();
}
```

---

## 20. Common Interview Questions

### Q1: What is the difference between `synchronized` and `ReentrantLock`?

| Aspect | `synchronized` | `ReentrantLock` |
|---|---|---|
| API | Keyword (implicit) | Class (explicit) |
| Lock release | Automatic | Must call `unlock()` in `finally` |
| Interruptible | No | Yes (`lockInterruptibly()`) |
| Trylock | No | Yes (`tryLock()`) |
| Fairness | No | Optional (`new ReentrantLock(true)`) |
| Condition objects | One (`wait/notify`) | Multiple (`newCondition()`) |
| Performance | Similar (JVM optimized) | Slightly higher overhead |

Use `synchronized` by default. Use `ReentrantLock` when you need tryLock, interruptibility, or multiple conditions.

---

### Q2: What is the happens-before guarantee and why does it matter?

The JMM allows the compiler and CPU to reorder instructions for performance. **Happens-before** is a formal guarantee that a write will be visible to a subsequent read. Without it, you can write to a field in one thread and another thread reads a stale/uninitialized value.

Key example: Double-Checked Locking bug pre-Java 5.

```java
// BROKEN in Java < 5 (no volatile)
class Singleton {
    private static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null)
                    instance = new Singleton(); // reordering bug: reference published before constructor finishes
            }
        }
        return instance;
    }
}

// CORRECT with volatile (Java 5+)
class Singleton {
    private static volatile Singleton instance; // volatile ensures full construction before publish
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

---

### Q3: What is the difference between `wait()` and `sleep()`?

`wait()` releases the lock and suspends the thread (requires `synchronized`); `sleep()` pauses without releasing any lock. `wait()` is for inter-thread communication; `sleep()` is for timed pauses.

---

### Q4: What is a ThreadLocal memory leak and how do you prevent it?

In thread pool environments (servers, Spring), threads are reused. If you call `threadLocal.set(value)` and never call `threadLocal.remove()`, the value persists for the thread's lifetime. Since `ThreadLocalMap` holds strong references to values, they're never GC'd as long as the thread lives. Always call `threadLocal.remove()` in a `finally` block.

---

### Q5: What is the difference between `CountDownLatch`, `CyclicBarrier`, and `Semaphore`?

- `CountDownLatch`: one-time countdown. Main thread waits for N events. Fire-and-forget from worker threads.
- `CyclicBarrier`: reusable. N threads all wait for each other at a sync point. Used for iterative phases.
- `Semaphore`: limits concurrent access to N permits. Used for resource pool throttling.

---

### Q6: Why is `ConcurrentHashMap` preferred over `Hashtable` or `Collections.synchronizedMap()`?

`Hashtable` and `synchronizedMap` lock the **entire map** for every operation — only one thread can read OR write at any time.

`ConcurrentHashMap` (Java 8+) uses:
- CAS for inserts into empty buckets (no lock at all).
- Lock only on the **individual bucket head** for writes.
- Lock-free reads using volatile.

Result: ~16x higher throughput under contention. Also, iteration on `ConcurrentHashMap` never throws `ConcurrentModificationException`.

---

### Q7: Explain the ForkJoinPool work-stealing algorithm.

Each thread in the pool maintains a **deque** of tasks. When a thread creates subtasks via `fork()`, tasks are added to its own deque. When a thread is idle, it **steals tasks from the tail of another thread's deque** (the original thread pops from its own head). This minimizes contention (two different ends of the deque) and maximizes CPU utilization.

---

### Q8: What is a spurious wakeup and how do you handle it?

A thread in `wait()` can wake up **without being notified** — this is allowed by the JVM/OS specification for performance reasons. Handle it by always wrapping `wait()` in a `while` loop that re-checks the condition:

```java
while (!conditionMet) {
    obj.wait();
}
```

---

### Q9: What is the difference between `notify()` and `notifyAll()`?

- `notify()` wakes **one arbitrary** waiting thread. Use only when all waiters are equivalent (same condition, identical action).
- `notifyAll()` wakes **all** waiting threads. Safer — each thread re-checks its condition (via `while` loop) and the correct one proceeds. Use `notifyAll()` by default unless you have proven performance issues.

---

### Q10: What is the ABA problem in CAS? How is it solved?

Thread 1 reads value `A`. Thread 2 changes it `A → B → A`. Thread 1's CAS sees `A` and succeeds, not knowing the value changed and reverted. This can cause silent bugs in data structures (e.g., wrong pointer in a lock-free linked list).

Solution: `AtomicStampedReference<V>` — pairs the value with an integer stamp (version). CAS checks BOTH the value AND the stamp.

---

## 21. Common Pitfalls & Gotchas

1. **Calling `start()` twice on a Thread** — throws `IllegalThreadStateException`. A thread can only be started once.

2. **Using `stop()`, `suspend()`, `resume()`** — deprecated and unsafe. Use `interrupt()` + `isInterrupted()` for cancellation.

3. **Swallowing `InterruptedException`** — catching it and doing nothing loses the interrupt signal. Always restore with `Thread.currentThread().interrupt()`.

4. **Using `Executors.newFixedThreadPool` in production** — uses unbounded `LinkedBlockingQueue`. A slow consumer causes the queue to grow without limit → OOM. Use `ThreadPoolExecutor` with bounded queue.

5. **Using `Executors.newCachedThreadPool` under high load** — unbounded thread creation. If tasks arrive faster than they complete, millions of threads → OOM. Only use for short-lived tasks with known bounded arrival rates.

6. **Not calling `shutdown()` on ExecutorService** — threads keep running, preventing JVM shutdown and leaking resources.

7. **`synchronized` on non-final fields**:
   ```java
   private Object lock = new Object(); // BAD: lock reference can be reassigned!
   synchronized (lock) { ... }

   private final Object lock = new Object(); // GOOD
   ```

8. **Double-checked locking without `volatile`** — object reference can be published before the constructor completes due to instruction reordering. Always use `volatile` for the instance field.

9. **`ThreadLocal` with thread pools** — values persist between requests unless `remove()` is called. Silent data leakage between requests.

10. **Using `parallel()` stream with blocking operations** — blocks the common `ForkJoinPool`, starving other parallel streams. Use a custom `ForkJoinPool` for blocking parallel operations.

11. **Virtual thread pinning with `synchronized`** — if a virtual thread blocks inside a `synchronized` block, the carrier thread is pinned (blocked). Use `ReentrantLock` instead.

12. **Checking `isDone()` in a spin loop** — burns CPU. Use `future.get()` with a timeout or `CompletableFuture.whenComplete()`.

13. **`HashMap` in multi-threaded code** — `HashMap` is NOT thread-safe. Under concurrent `put` operations, it can cause **infinite loops** (Java 6/7) or data loss. Always use `ConcurrentHashMap` or synchronize externally.

14. **Large `CopyOnWriteArrayList` with frequent writes** — each write copies the entire array. O(n) per write. Only suitable for read-heavy, small collections.

---

## 22. Quick Revision Summary

- **Thread states**: NEW → RUNNABLE ↔ BLOCKED/WAITING/TIMED_WAITING → TERMINATED.
- **`volatile`**: visibility + happens-before, NOT atomicity for compound ops.
- **`synchronized`**: mutual exclusion, visibility, reentrant, not interruptible.
- **`wait()`** releases the lock; **`sleep()`** does not.
- Always use **`while` loop** with `wait()` for spurious wakeup safety.
- **`ReentrantLock`** adds tryLock, interruptibility, fairness, and multiple `Condition`s.
- **Use `ThreadPoolExecutor` directly** in production — never unbounded queues.
- **`CompletableFuture`** for non-blocking async pipelines; `thenCompose` for flat-map, `thenCombine` for merging.
- **`CountDownLatch`** = one-shot gate; **`CyclicBarrier`** = reusable rendezvous; **`Semaphore`** = N-permit throttle.
- **CAS** = hardware atomic compare-and-swap; `AtomicInteger`/`AtomicReference` use it.
- **`ConcurrentHashMap`**: CAS + per-bucket lock, lock-free reads — far better than `Hashtable`.
- **`ThreadLocal` leaks**: always `remove()` in `finally` in thread pool contexts.
- **`ForkJoinPool`**: work-stealing for recursive divide-and-conquer; backs parallel streams.
- **Deadlock prevention**: consistent lock ordering or `tryLock` with timeout.
- **Virtual threads (Java 21)**: millions of lightweight threads; JVM unmounts on blocking I/O; avoid `synchronized` (causes pinning).
