# Thread Lifecycle — Deep Dive

---

## Table of Contents

1. [What is Thread Lifecycle?](#1-what-is-thread-lifecycle)
2. [The 6 States — Simple Definitions](#2-the-6-states)
3. [Full State Transition Diagram](#3-state-transition-diagram)
4. [Each State — Deep Explanation](#4-each-state-deep-explanation)
   - [NEW](#41-new)
   - [RUNNABLE](#42-runnable)
   - [BLOCKED](#43-blocked)
   - [WAITING](#44-waiting)
   - [TIMED_WAITING](#45-timed_waiting)
   - [TERMINATED](#46-terminated)
5. [How to Observe Thread States at Runtime](#5-how-to-observe-thread-states)
6. [Real-Life Scenario Walkthroughs](#6-real-life-scenario-walkthroughs)
7. [Common Interview Questions](#7-common-interview-questions)
8. [Common Production Pitfalls](#8-common-production-pitfalls)
9. [Quick Revision Summary](#9-quick-revision-summary)

---

## 1. What is Thread Lifecycle?

### Simple Language

Think of a thread like an **employee at a company**. The employee:
- Gets **hired** (created).
- Gets **assigned to a desk** and starts working (running).
- Sometimes has to **wait at the printer** because someone else is using it (blocked).
- Sometimes sits in the **break room waiting for a colleague to call them** (waiting).
- Sometimes sets a **15-minute timer and waits** for something (timed waiting).
- Eventually **finishes their shift and goes home** (terminated).

That complete journey — from hire to leaving — is the **Thread Lifecycle**.

### Formal Definition

The Thread Lifecycle is the set of **well-defined states** a thread passes through from the moment it is created (`new Thread()`) until it finishes execution. Java formally defines these states in the `Thread.State` enum.

```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED
}
```

### What Problem Does This Solve?

Without a defined lifecycle:
- You wouldn't know if a thread is ready to run, waiting for a resource, or dead.
- You couldn't build tools (debuggers, profilers, monitoring dashboards) to understand thread health.
- The JVM couldn't manage CPU scheduling, resource acquisition, and cleanup efficiently.
- Developers couldn't diagnose issues like deadlocks, starvation, or zombie threads.

The lifecycle gives both the **JVM scheduler** and **developers** a shared vocabulary for understanding what every thread is doing at any point in time.

---

## 2. The 6 States

| State | One-Line Definition |
|---|---|
| `NEW` | Thread object created, `start()` not yet called |
| `RUNNABLE` | Thread is either running on CPU or ready and waiting for CPU time |
| `BLOCKED` | Thread is waiting to acquire a `synchronized` lock held by another thread |
| `WAITING` | Thread is indefinitely suspended — explicitly waiting for another thread to wake it |
| `TIMED_WAITING` | Thread is suspended for a specific duration — wakes automatically after timeout |
| `TERMINATED` | Thread has finished execution (normally or due to exception) |

---

## 3. State Transition Diagram

```
                    ┌─────────────────────────────────────────────────────────────────────┐
                    │                                                                     │
   new Thread()     │  start()                                                           │
  ┌───────────┐     │  ┌─────────────────────────────────────────────────────────────┐   │
  │           │     │  │                                                             │   │
  │    NEW    ├─────▼──▶                    RUNNABLE                                │   │
  │           │        │  (scheduler decides: actually ON CPU or waiting for CPU)   │   │
  └───────────┘        └──┬────────────────────┬──────────────────────┬─────────────┘   │
                          │                    │                      │                  │
                          │ tries to enter     │ calls                │ calls            │
                          │ synchronized block │ wait()/join()        │ sleep(n)/        │
                          │ but lock is taken  │ LockSupport.park()   │ wait(n)/join(n)  │
                          │                    │                      │                  │
                          ▼                    ▼                      ▼                  │
                   ┌────────────┐    ┌──────────────────┐    ┌─────────────────────┐    │
                   │            │    │                  │    │                     │    │
                   │  BLOCKED   │    │    WAITING       │    │   TIMED_WAITING     │    │
                   │            │    │                  │    │                     │    │
                   └──────┬─────┘    └────────┬─────────┘    └──────────┬──────────┘    │
                          │                   │                         │               │
                          │ lock released,    │ notify()/notifyAll()    │ timeout       │
                          │ thread wins lock  │ interrupt()             │ expires /     │
                          │                  │ join target finishes     │ interrupt()   │
                          │                   │                         │               │
                          └───────────────────┴─────────────────────────┘               │
                                                        │                               │
                                                        │ back to RUNNABLE              │
                                                        ▼                               │
                                              ┌──────────────────┐                     │
                                              │                  │                     │
                                              │   RUNNABLE       │─────────────────────┘
                                              │                  │    run() returns
                                              └────────┬─────────┘    or throws uncaught exception
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │                  │
                                              │   TERMINATED     │
                                              │                  │
                                              └──────────────────┘
```

---

## 4. Each State — Deep Explanation

---

### 4.1 NEW

#### What it is

The thread object has been **created in memory** using `new Thread(...)` but `start()` has **not been called** yet. The thread exists as a Java object but is completely unknown to the OS and the JVM scheduler. No OS thread has been created yet.

#### How it works internally

```java
Thread t = new Thread(() -> System.out.println("hello"));
// At this point:
// - A Thread object is on the heap
// - Thread.State = NEW
// - No OS thread exists yet
// - The thread has no stack yet

System.out.println(t.getState()); // prints: NEW
```

The OS thread is only created when `start()` is called. The JVM calls a native method (`start0()`) which asks the OS to create a thread.

#### Real-life analogy

You've filled out a **job application** for a new employee — the paperwork exists, but the person hasn't shown up for their first day yet.

#### Common Pitfall

```java
Thread t = new Thread(() -> processOrder());
// ... somewhere else in code
t.run(); // BUG: calls run() directly, NOT start()
// This executes on the CURRENT thread, not a new thread!
// Thread state stays NEW, then goes to TERMINATED on the current thread's stack.
```

`t.run()` is just a regular method call. `t.start()` is what actually creates a new thread.

---

### 4.2 RUNNABLE

#### What it is

This is the **active state**. A RUNNABLE thread is either:
- **Currently executing** on a CPU core, OR
- **Ready to execute** but waiting for the OS scheduler to give it CPU time.

Java does NOT distinguish between these two sub-states. Both are `RUNNABLE` from Java's perspective. This is because the distinction is handled entirely by the OS kernel, not the JVM.

#### How it works internally

When `start()` is called:
1. JVM calls native `start0()`.
2. OS creates a real kernel thread.
3. OS scheduler puts the thread in its **run queue**.
4. The OS preemptively time-slices CPU across all RUNNABLE threads.
5. When the thread's time slice expires, it goes back to the run queue — still RUNNABLE, but not currently on CPU.

```java
Thread t = new Thread(() -> {
    System.out.println(Thread.currentThread().getState()); 
    // Can't print RUNNING — Java has no RUNNING state
    // This will print RUNNABLE
    while (true) { /* busy loop */ }
});
t.start();
Thread.sleep(100);
System.out.println(t.getState()); // prints: RUNNABLE
```

#### Real-life analogy

An employee is at their **desk working** OR is in the **queue for the coffee machine** — either way, they're "on the clock" and ready to work. The manager (OS scheduler) decides who actually gets to use the coffee machine right now.

#### Key points
- A thread can be RUNNABLE and not actually running (waiting for CPU).
- The OS may switch a thread off CPU hundreds of times per second — it's still RUNNABLE throughout.
- Calling `Thread.yield()` moves the thread from "currently on CPU" back to the run queue — still RUNNABLE.

---

### 4.3 BLOCKED

#### What it is

A thread enters BLOCKED state when it tries to **enter a `synchronized` block or method** but **another thread currently holds that lock**.

This is purely about **mutual exclusion via the `synchronized` keyword**. It has nothing to do with I/O blocking or `wait()`.

#### How it works internally

```java
// Account object's intrinsic lock
public class BankAccount {
    private double balance = 1000.0;

    public synchronized void withdraw(double amount) {
        // Only ONE thread at a time can be inside here
        balance -= amount;
        // Simulate some processing
        try { Thread.sleep(5000); } catch (InterruptedException e) {}
    }
}

BankAccount account = new BankAccount();

Thread t1 = new Thread(() -> account.withdraw(100)); // acquires lock, enters synchronized
Thread t2 = new Thread(() -> account.withdraw(200)); // tries to acquire lock → BLOCKED

t1.start();
Thread.sleep(100); // give t1 a head start
t2.start();
Thread.sleep(100); // give t2 time to reach the synchronized block

System.out.println(t2.getState()); // prints: BLOCKED
```

When `t1` releases the lock (exits the `synchronized` block), the JVM moves `t2` from BLOCKED back to RUNNABLE, and `t2` can try to acquire the lock again.

#### Important distinction: BLOCKED vs WAITING

| | BLOCKED | WAITING |
|---|---|---|
| Caused by | Trying to enter `synchronized` | Calling `wait()`, `join()`, `park()` |
| Waiting for | A lock to be released | An explicit signal/notification |
| Can be interrupted? | **No** — must wait for lock | **Yes** — `interrupt()` works |
| `ReentrantLock.lock()` | Thread goes to **WAITING**, not BLOCKED | — |

This is a very common interview trick. `ReentrantLock.lock()` puts a thread in **WAITING** state (via `LockSupport.park()`), NOT BLOCKED. BLOCKED is exclusively for `synchronized`.

#### Real-life analogy

You walk up to the **single-occupancy bathroom** at work. Someone is inside (lock is held). You stand right outside the door waiting. You're not doing anything else — just waiting for the door to open. That's BLOCKED.

#### Common Production Pitfall

A spike in threads in BLOCKED state means **lock contention** — many threads competing for the same `synchronized` block. This is a performance bottleneck. You'd see it in a thread dump as:

```
"http-thread-45" - BLOCKED on <0x00000000c1234567> (BankAccount.class)
   waiting for thread "http-thread-12" to release the lock
```

Fix: reduce the scope of `synchronized`, use finer-grained locking, or switch to `ConcurrentHashMap`/`ReentrantLock`.

---

### 4.4 WAITING

#### What it is

A thread enters WAITING when it **voluntarily suspends itself indefinitely**, waiting for **another thread to explicitly wake it up**. The thread is not using any CPU and is not runnable until signaled.

#### What causes WAITING state

| Method Called | How to Wake Up |
|---|---|
| `Object.wait()` | Another thread calls `notify()` or `notifyAll()` on the same object |
| `Thread.join()` (no timeout) | The target thread terminates |
| `LockSupport.park()` | Another thread calls `LockSupport.unpark(thread)` |

#### How it works internally — `wait()` example

```java
public class OrderQueue {
    private final Queue<Order> queue = new LinkedList<>();

    // Consumer — called by worker threads
    public synchronized Order consume() throws InterruptedException {
        while (queue.isEmpty()) {
            System.out.println(Thread.currentThread().getName() + " → WAITING");
            wait();
            // What happens at wait():
            // 1. Thread atomically RELEASES the lock on 'this'
            // 2. Thread's state changes to WAITING
            // 3. Thread is added to the wait set of this object
            // 4. Thread suspends — uses NO CPU
        }
        return queue.poll();
    }

    // Producer — called when a new order arrives
    public synchronized void produce(Order order) {
        queue.add(order);
        notifyAll(); // moves ALL threads from wait set → BLOCKED (they now compete for lock)
    }
}
```

#### Step-by-step breakdown of `wait()`

```
Thread calls wait() inside synchronized(obj)
    │
    ├─ 1. Thread releases the lock on 'obj'         ← KEY: lock is released!
    ├─ 2. Thread.State changes to WAITING
    ├─ 3. Thread enters the WAIT SET of 'obj'
    └─ 4. Thread suspends (no CPU usage)

Another thread calls obj.notify() or obj.notifyAll()
    │
    ├─ notify():    ONE arbitrary thread moves from WAIT SET → BLOCKED (ready to re-acquire lock)
    └─ notifyAll(): ALL threads move from WAIT SET → BLOCKED

When the signaled thread re-acquires the lock:
    └─ Thread.State changes back to RUNNABLE
    └─ Thread continues from the line AFTER wait()
```

#### `join()` example

```java
Thread t1 = new Thread(() -> {
    // do heavy computation
    Thread.sleep(3000);
});
t1.start();

// Main thread calls join() — enters WAITING state
System.out.println(Thread.currentThread().getState()); // This can't print itself accurately
t1.join(); // main thread → WAITING until t1 terminates
System.out.println("t1 is done");
```

```java
// Check from another thread:
Thread main = Thread.currentThread();
Thread watcher = new Thread(() -> {
    Thread.sleep(500);
    System.out.println(main.getState()); // prints: WAITING
});
watcher.start();
t1.join();
```

#### Real-life analogy

An employee sends an email to a colleague and says "Let me know when the report is ready." They then **sit idle, doing nothing**, waiting for a ping. They're not blocked by a door — they're just voluntarily waiting for a signal. That's WAITING.

---

### 4.5 TIMED_WAITING

#### What it is

Exactly like WAITING, but with a **built-in expiry timer**. The thread wakes up automatically when the timer runs out, even if no one signals it. It can also be woken up early by a signal or interrupt.

#### What causes TIMED_WAITING state

| Method Called | Wakes up when |
|---|---|
| `Thread.sleep(n)` | After `n` milliseconds |
| `Object.wait(n)` | After `n` ms OR `notify()` — whichever comes first |
| `Thread.join(n)` | After `n` ms OR target thread finishes |
| `LockSupport.parkNanos(n)` | After `n` nanoseconds |
| `LockSupport.parkUntil(deadline)` | After absolute deadline |
| `Condition.await(n, unit)` | After timeout OR `signal()` |

#### `Thread.sleep()` deep-dive

```java
Thread t = new Thread(() -> {
    System.out.println("Starting 5-second sleep...");
    try {
        Thread.sleep(5000); // → TIMED_WAITING for 5 seconds
        // IMPORTANT: sleep does NOT release any lock it holds!
    } catch (InterruptedException e) {
        System.out.println("Sleep interrupted early!");
        Thread.currentThread().interrupt(); // restore interrupt flag
    }
    System.out.println("Awake!");
});

t.start();
Thread.sleep(100);
System.out.println(t.getState()); // prints: TIMED_WAITING
```

#### `wait(timeout)` — the hybrid

```java
synchronized (lock) {
    lock.wait(3000); // wait for signal, but at most 3 seconds
    // Wakes up if:
    // - notify()/notifyAll() is called before 3s → RUNNABLE
    // - 3 seconds pass with no notify → RUNNABLE (spurious wake)
    // - thread.interrupt() called → throws InterruptedException
}
```

#### Real-life analogy

You put a **15-minute timer on your phone** and take a nap on the couch. You'll wake up when the timer goes off — but your colleague can also wake you up earlier if needed. Compare to WAITING (indefinite nap with no alarm — only your colleague can wake you).

#### sleep() vs wait() — the most common interview question

| | `Thread.sleep(n)` | `Object.wait(n)` |
|---|---|---|
| Releases lock? | **NO** | **YES** |
| Must be in `synchronized`? | No | **YES** |
| Belongs to | `Thread` class | `Object` class |
| Interrupt handling | Throws `InterruptedException` | Throws `InterruptedException` |
| State | `TIMED_WAITING` | `TIMED_WAITING` |

---

### 4.6 TERMINATED

#### What it is

The thread has **finished its life**. The `run()` method has returned (either normally or due to an uncaught exception). The thread object still exists in memory (you can still call `t.getState()`, `t.getName()`, etc.) but the thread can **never be restarted**.

#### How it happens

```java
Thread t = new Thread(() -> {
    System.out.println("Working...");
    // run() returns normally
});
t.start();
t.join(); // wait for t to finish
System.out.println(t.getState()); // prints: TERMINATED
```

```java
// Also terminated via uncaught exception
Thread t = new Thread(() -> {
    throw new RuntimeException("Crash!");
});
t.setUncaughtExceptionHandler((thread, ex) -> {
    System.out.println(thread.getName() + " died: " + ex.getMessage());
});
t.start();
t.join();
System.out.println(t.getState()); // prints: TERMINATED
```

#### Can a TERMINATED thread be restarted?

**No. Never.** Calling `start()` on a TERMINATED thread throws `IllegalThreadStateException`.

```java
t.start(); // first start — OK
t.join();  // wait for it to finish
t.start(); // throws IllegalThreadStateException — thread is TERMINATED
```

This is a common bug in code that tries to "restart" threads. The fix is to create a new `Thread` object, or better, use an `ExecutorService` which handles thread lifecycle management for you.

#### Real-life analogy

The employee's **contract has ended** and they've left the company. Their employee file still exists in HR (the Java object still in memory, eligible for GC if no references exist), but you can't call them back to work the same contract. You'd need to hire someone new.

#### TERMINATED vs Daemon Thread Death

When a daemon thread's carrier process (JVM) shuts down, daemon threads are killed abruptly — they may not reach TERMINATED state cleanly. Their `finally` blocks may NOT run.

```java
Thread daemon = new Thread(() -> {
    try {
        while (true) { /* background work */ }
    } finally {
        System.out.println("Cleanup!"); // MAY NOT PRINT if JVM shuts down
    }
});
daemon.setDaemon(true);
daemon.start();
// JVM shuts down → daemon thread killed without cleanup
```

---

## 5. How to Observe Thread States at Runtime

### Programmatically

```java
// Get state of any thread
Thread t = new Thread(() -> { /* ... */ });
System.out.println(t.getState()); // Thread.State enum value

// Get ALL thread states in the JVM
ThreadMXBean tmx = ManagementFactory.getThreadMXBean();
ThreadInfo[] infos = tmx.dumpAllThreads(true, true);
for (ThreadInfo info : infos) {
    System.out.printf("%-40s %s%n", info.getThreadName(), info.getThreadState());
}
```

### Thread Dump (Production Debugging)

```bash
# Get PID of Java process
jps -l

# Take thread dump
jstack <pid>

# Or send SIGQUIT signal (Unix)
kill -3 <pid>

# Or via jcmd (modern)
jcmd <pid> Thread.print
```

**Sample thread dump output:**

```
"http-nio-8080-exec-5" #42 daemon prio=5 os_prio=0 cpu=1.23ms elapsed=120.4s tid=0x00007f... nid=0x3a2b
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.BankAccount.withdraw(BankAccount.java:15)
        - waiting to lock <0x00000000c1abc123> (a com.example.BankAccount)
        - locked by "http-nio-8080-exec-3"

"http-nio-8080-exec-3" #40 daemon prio=5 os_prio=0 cpu=850ms elapsed=120.4s tid=0x00007f... nid=0x3a29
   java.lang.Thread.State: RUNNABLE
        at com.example.BankAccount.withdraw(BankAccount.java:15)
        - locked <0x00000000c1abc123> (a com.example.BankAccount)
```

This is exactly how you diagnose BLOCKED threads in production.

### Monitoring Tools
- **VisualVM**: GUI thread timeline — see state changes over time.
- **JConsole**: Live thread state view.
- **Async Profiler**: Flame graph — shows where threads spend time.
- **Datadog / New Relic**: APM tools alert on thread pool saturation (too many BLOCKED/WAITING threads).

---

## 6. Real-Life Scenario Walkthroughs

---

### Scenario 1: E-Commerce Order Processing

**Setup:** Tomcat handles 200 concurrent HTTP requests. Each request processes an order via a `synchronized` method that calls a slow payment gateway.

```java
@RestController
public class OrderController {

    @PostMapping("/order")
    public ResponseEntity<String> placeOrder(@RequestBody OrderRequest req) {
        orderService.processOrder(req); // this calls synchronized method
        return ResponseEntity.ok("Order placed");
    }
}

@Service
public class OrderService {
    // BAD: entire method synchronized — one order at a time!
    public synchronized void processOrder(OrderRequest req) {
        paymentGateway.charge(req); // takes 2 seconds
        inventory.deduct(req);
        emailService.sendConfirmation(req);
    }
}
```

**What happens under load:**

```
Thread states under 200 concurrent requests:
- Thread 1:  RUNNABLE  → executing processOrder()
- Thread 2:  BLOCKED   → waiting for lock on orderService
- Thread 3:  BLOCKED   → waiting for lock on orderService
- ...
- Thread 200: BLOCKED  → waiting for lock on orderService

All 199 threads are BLOCKED. Throughput = 1 order / 2 seconds = 0.5 orders/sec.
```

**The BLOCKED threads in a thread dump would show:**

```
"http-nio-8080-exec-15" BLOCKED
  waiting to lock <0x...> (OrderService)
  locked by "http-nio-8080-exec-1"
```

**Fix:** Remove `synchronized` from the method, use per-order locking or remove the need for synchronization entirely.

---

### Scenario 2: Report Generation Service

**Setup:** A report generation service spawns worker threads to fetch data from 3 different data sources in parallel, then merges them.

```java
public Report generateReport(String userId) throws InterruptedException {
    List<DataSection> sections = new ArrayList<>();

    Thread t1 = new Thread(() -> sections.add(fetchSalesData(userId)));
    Thread t2 = new Thread(() -> sections.add(fetchInventoryData(userId)));
    Thread t3 = new Thread(() -> sections.add(fetchFinancialData(userId)));

    t1.start(); // t1: NEW → RUNNABLE
    t2.start(); // t2: NEW → RUNNABLE
    t3.start(); // t3: NEW → RUNNABLE

    // Main thread calls join() → WAITING for each
    t1.join();  // main: RUNNABLE → WAITING until t1 is TERMINATED
    t2.join();  // main: RUNNABLE → WAITING until t2 is TERMINATED
    t3.join();  // main: RUNNABLE → WAITING until t3 is TERMINATED

    // main thread resumes: WAITING → RUNNABLE
    return merge(sections);
}
```

**Thread state timeline:**

```
Time 0ms:   main=RUNNABLE, t1=NEW, t2=NEW, t3=NEW
Time 1ms:   main=RUNNABLE, t1=RUNNABLE, t2=RUNNABLE, t3=RUNNABLE (all started)
Time 2ms:   main=WAITING,  t1=RUNNABLE, t2=RUNNABLE, t3=RUNNABLE (main called t1.join())
Time 500ms: main=WAITING,  t1=TIMED_WAITING(inside sleep/IO), t2=RUNNABLE, t3=RUNNABLE
Time 2000ms: t1=TERMINATED → main: WAITING → WAITING (now waiting for t2)
Time 2100ms: t2=TERMINATED → main: WAITING → WAITING (now waiting for t3)
Time 2300ms: t3=TERMINATED → main: WAITING → RUNNABLE
Time 2305ms: merge() runs, report returned
```

---

### Scenario 3: Ride-Sharing — Driver Location Updates

**Setup:** Background thread polls GPS every 2 seconds.

```java
public class LocationTracker implements Runnable {
    private volatile boolean active = true;

    @Override
    public void run() {
        while (active) {
            // State: RUNNABLE (doing real work)
            GPS location = gpsService.getCurrentLocation();
            locationRepository.save(location);

            // State: TIMED_WAITING for 2 seconds
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break; // graceful shutdown
            }
            // State: back to RUNNABLE after sleep expires
        }
        // State: TERMINATED
    }

    public void stop() {
        active = false;
        // Note: if thread is sleeping, it won't check 'active' for up to 2 seconds
        // To wake it up immediately:
        trackerThread.interrupt(); // moves from TIMED_WAITING → InterruptedException thrown
    }
}
```

**Why interrupt() matters here:**  
Without `interrupt()`, calling `stop()` while the thread sleeps means it waits up to 2 more seconds before actually stopping. With `interrupt()`, it wakes from `TIMED_WAITING` immediately.

---

## 7. Common Interview Questions

### Q1: What is the difference between BLOCKED and WAITING?

**BLOCKED** is specifically about waiting for a `synchronized` lock. It cannot be interrupted — the thread must wait until the lock is released.

**WAITING** is a voluntary suspension via `wait()`, `join()`, or `park()`. It CAN be interrupted via `thread.interrupt()`, which causes `InterruptedException` to be thrown.

Key distinction: `ReentrantLock.lock()` puts a thread in **WAITING** (not BLOCKED), because it uses `LockSupport.park()` internally.

---

### Q2: How many threads can be in RUNNABLE state simultaneously?

All of them can be in RUNNABLE state simultaneously. RUNNABLE means "eligible to run or currently running." The OS scheduler decides which ones are actually on CPU at any instant. On an 8-core machine, at most 8 threads actually execute simultaneously, but hundreds can be RUNNABLE (queued to run).

---

### Q3: What happens if you call `start()` twice on the same thread?

```java
Thread t = new Thread(() -> System.out.println("hi"));
t.start(); // OK: NEW → RUNNABLE → TERMINATED
t.start(); // Throws IllegalThreadStateException
```

Once a thread reaches TERMINATED, it can never be restarted. You must create a new `Thread` object. This is one reason `ExecutorService` is preferred — it handles thread reuse internally.

---

### Q4: Does `Thread.sleep()` release the lock?

**No.** `sleep()` pauses the thread but holds all locks. This is a classic trap:

```java
synchronized (lock) {
    Thread.sleep(10000); // TIMED_WAITING but lock is STILL HELD
    // All other threads trying to enter this synchronized block are BLOCKED for 10 seconds!
}
```

`wait()` releases the lock. `sleep()` does not. Never use `sleep()` inside a synchronized block to "wait for a condition."

---

### Q5: Can a TERMINATED thread be garbage collected?

Yes, IF no other object holds a reference to the `Thread` object. The thread's native OS resources are cleaned up when it terminates. The Java `Thread` object itself is GC-eligible once no references remain.

However, threads stored in fields (like a service class field `private Thread workerThread`) will keep the Thread object alive even after TERMINATED — no memory leak, but the object stays in heap until the holding class is GC'd.

---

### Q6: What is a zombie thread?

Informally, a "zombie thread" is a TERMINATED thread whose object is still referenced somewhere, preventing GC. Unlike OS-level zombie processes, Java zombie threads consume no CPU or meaningful resources — just the ~100 bytes of the Thread object on heap. Usually not a concern unless you store millions of terminated thread references.

---

### Q7: Why does Java's RUNNABLE combine "ready" and "running" into one state?

Because the split between "ready in run queue" and "currently executing on CPU" is an **OS-level distinction**, not a JVM-level one. The JVM doesn't always know (or need to know) whether its thread is actually on CPU right now — that's the OS kernel's domain. Exposing this as a single `RUNNABLE` state keeps the Java memory model clean and portable across different OSes.

---

## 8. Common Production Pitfalls

### Pitfall 1: Thread Dump Shows Hundreds of BLOCKED Threads

**Symptom:** API latency spikes, thread pool exhaustion, requests timing out.  
**Cause:** Too many threads competing for the same `synchronized` lock.  
**Diagnosis:** `jstack <pid>` → look for threads in BLOCKED state all pointing to same lock.  
**Fix:** Reduce critical section size, use `ConcurrentHashMap`, `ReentrantLock` with finer granularity, or redesign to avoid contention.

---

### Pitfall 2: Thread Stuck in WAITING Forever (Missed Notification)

```java
// Thread A
synchronized (obj) {
    obj.wait(); // waiting for notification
}

// Thread B (runs BEFORE Thread A even calls wait)
synchronized (obj) {
    obj.notify(); // signal sent to nobody!
}
// Thread A calls wait() AFTER notify() → waits forever (missed signal)
```

**Fix:** Always use a **state variable** with `wait()` in a `while` loop:

```java
// Thread A
synchronized (obj) {
    while (!workAvailable) { // check actual condition
        obj.wait();
    }
    doWork();
}

// Thread B
synchronized (obj) {
    workAvailable = true;
    obj.notify();
}
// Now even if notify() runs first, Thread A sees workAvailable=true and skips wait()
```

---

### Pitfall 3: Sleeping Inside `synchronized` (Holding Locks Too Long)

```java
public synchronized void processRequest() {
    Thread.sleep(5000); // TERRIBLE: holds lock for 5 seconds, blocks all other threads
    doWork();
}
```

**Fix:** Release lock before sleeping, or restructure to not hold the lock during waits.

---

### Pitfall 4: Not Restoring Interrupt Flag

```java
// WRONG
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // silently swallowed! Interrupt flag is now cleared.
    // Any caller that checks isInterrupted() will see false.
}

// CORRECT
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore the flag
    return; // or propagate
}
```

---

### Pitfall 5: Calling `run()` Instead of `start()`

```java
Thread t = new Thread(() -> heavyComputation());
t.run();   // runs on the CURRENT thread — no new thread created!
// Thread state: t stays NEW, then goes TERMINATED immediately on current thread's stack
// No concurrency benefit at all
```

---

### Pitfall 6: `join()` Without Timeout in Production

```java
workerThread.join(); // waits FOREVER if workerThread hangs
```

Always use a timeout in production:

```java
workerThread.join(5000); // wait at most 5 seconds
if (workerThread.isAlive()) {
    log.warn("Worker did not finish in time, interrupting");
    workerThread.interrupt();
}
```

---

### Pitfall 7: Daemon Threads Skipping Cleanup

```java
Thread daemon = new Thread(() -> {
    try {
        writeToDatabase();
    } finally {
        closeConnection(); // MAY NOT RUN if JVM shuts down
    }
});
daemon.setDaemon(true);
daemon.start();
```

If the JVM exits while this daemon thread is running, `closeConnection()` may never execute → resource leak. Only use daemon threads for truly non-critical background work (metrics, health checks, etc.).

---

## 9. Quick Revision Summary

- **6 states**: NEW → RUNNABLE ↔ BLOCKED / WAITING / TIMED_WAITING → TERMINATED.
- **NEW**: object created, OS thread not yet created, `start()` not called.
- **RUNNABLE**: on CPU or in OS run queue — Java doesn't distinguish between the two.
- **BLOCKED**: waiting for a `synchronized` lock — cannot be interrupted.
- **WAITING**: voluntarily sleeping, waiting for `notify()` / `join()` / `unpark()` — CAN be interrupted.
- **TIMED_WAITING**: like WAITING but with a timeout — wakes up automatically.
- **TERMINATED**: `run()` finished; thread object stays in memory but can never restart.
- `sleep()` → TIMED_WAITING; does **NOT** release locks.
- `wait()` → WAITING or TIMED_WAITING; **DOES** release the lock.
- `BLOCKED` is only for `synchronized`; `ReentrantLock.lock()` → WAITING.
- Diagnosing production issues: use `jstack <pid>` to read thread states and stack traces.
- Always use `while` (not `if`) around `wait()` to guard against spurious wakeups.
- Always use `join(timeout)` (not `join()`) in production to avoid infinite waits.
