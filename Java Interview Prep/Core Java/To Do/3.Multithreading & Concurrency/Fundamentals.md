- **Concurrency** = dealing with multiple tasks at the same time (conceptually).  
    Tasks may _interleave_ on a single CPU core.
- **Parallelism** = actually executing multiple tasks at the same time.  
    Requires multiple CPU cores (or processors).
Simple analogy
- **Concurrency**: One chef cooking 3 dishes by switching between them.
- **Parallelism**: Three chefs each cooking one dish at the same time.

Concurrency is about _dealing with_ multiple things at once.  
Parallelism is about _doing_ multiple things at once.

**Nagarro Softwares Pvt. Ltd.**   Gurugram, India _Junior Associate, Technology_ 

![[concurrency_parallelism.mp4]]


| Concept      | About Structure? | About Hardware? | About Waiting?       |
| ------------ | ---------------- | --------------- | -------------------- |
| Concurrency  | ✅ Yes            | ❌ Not required  | ❌ Not necessarily    |
| Parallelism  | ❌ Not design     | ✅ Yes           | ❌ Not about waiting  |
| Asynchronous | ✅ Yes            | ❌ No            | ✅ Yes (non-blocking) |


# 🧩 Simple Mental Model

- **Concurrency** → “Many things in progress.”
- **Parallelism** → “Many things happening at the exact same time.”
- **Asynchronous** → “Don’t wait; come back later.”



![[MultiTh-Concurrency TERMs.png]]


IMP PACKAGES : 

java.lang
- Why: basic threading primitives and monitor methods live here.
	- java.lang.Thread, 
	- java.lang.Runnable, 
	- java.lang.ThreadGroup, 
	- java.lang.ThreadLocal, 
	- java.lang.InterruptedException, 
	- Object.wait/notify/notifyAll (monitor methods)


- [ ] Block from low to high to master concurrency in java
- [ ] Producer/Consumer Queue


# Thread 🧵Lifecycle 🔁

six core states in the Java thread lifecycle:
1. `NEW`, 
2. `RUNNABLE`, 
3. `BLOCKED*`, 
4. `WAITING`, 
5. `TIMED_WAITING`, and 
6. `TERMINATED`

**Mnemonic** : **N**ow **R**unning, **B**ut **W**aiting **T**ime **T**erminated.
acronym **NRBWTT** (pronounced _N-R-Bew-T-T_)  

 Think of a thread as a worker: it is **N**ewborn, starts **R**unning, gets **B**locked by a door, decides to **W**ait, takes a **T**imed break, and finally **T**erminates.
 
 ```js
 Think of a thread 🧵 as a worker: 
    it is Newborn 👶, 
				starts Running 🏃, 
						🚪gets Blocked by a door 🏃‍♀️, 
									decides to Wait ⏳,
											takes a Timed break ⏱️, and
													finally Terminates ☠️💀
 ```

`BLOCKED` is the **only state a thread enters involuntarily**.
The JVM manages the `BLOCKED` state behind the scenes. You cannot "ask" a thread to be blocked; it happens because the thread **attempted** to do something (enter a synchronized block/method) and the "bouncer" (the monitor lock) turned it away.

|**State**|**How it's reached**|**Control**|
|---|---|---|
|**BLOCKED**|**Implicitly** by the JVM when a thread fails to acquire a monitor lock.|**External:** Depends on other threads releasing the lock.|
|**WAITING**|**Explicitly** calling `wait()` or `join()`.|**Internal:** The thread chooses to pause.|
|**TIMED_WAITING**|**Explicitly** calling `sleep(ms)`, `wait(ms)`, or `join(ms)`.|**Internal:** The thread chooses to pause for a specific duration.|

### The Subtle Difference: Waiting vs. Blocked
This is where many developers get confused. Here is the "Dance" analogy:
- **BLOCKED (The Queue):** You are standing at the club door. You _want_ to be inside dancing, but the club is at capacity. You are actively trying to get in the moment someone leaves.
- **WAITING (The Lounge):** You are already inside the club, but you've stepped off the dance floor to sit in the lounge. You told the DJ, "Don't call me back until the next song starts (`notify()`)." You aren't even trying to dance right now.

---

# Methods

## wait()

## sleep()

## join()

## yield()
The `Thread.yield()` method in Java is ==a **static native method** that acts as a **hint** to the thread scheduler==, suggesting that the currently executing thread is willing to pause its execution and allow other threads of the same or higher priority to run.
```java
public class YieldDemo {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 1; i <= 5; i++) {
                System.out.println(Thread.currentThread().getName() + " is running: " + i);
                // Hint to the scheduler that this thread is willing to yield
                Thread.yield(); 
            }
        };

        Thread t1 = new Thread(task, "Child-Thread-1");
        Thread t2 = new Thread(task, "Child-Thread-2");

        t1.start();
        t2.start();
    }
}

```

----
## User vs Daemon Thread
*what is daemon thread (java). how is it different from other threads. clarify through examples.*

In the Java universe, there are two types of threads: **User Threads** (high priority, the stars of the show) and **Daemon Threads** (the background crew).
The best way to think of a **Daemon Thread** is as a "service provider." Its only purpose is to support user threads.
### 1. The Core Difference: Survival
The JVM will continue to run as long as at least **one** User Thread is alive. However, the JVM **does not care** about Daemon threads. The moment the last User Thread finishes, the JVM will abruptly kill all Daemon threads and exit.

|**Feature**|**User Thread**|**Daemon Thread**|
|---|---|---|
|**Purpose**|To execute main application logic.|To provide background support (e.g., GC).|
|**Priority**|Higher.|Lower.|
|**JVM Exit**|JVM waits for it to finish.|JVM terminates it immediately on exit.|
|**Example**|`main` thread, request processors.|Garbage Collector, Finalizers.|
### 2. Code Example: The "Abrupt End"

In this example, the child thread is a "Cleaner." If we don't make it a daemon, the program will run forever. If we **do** make it a daemon, it dies the moment `main` finishes.

```Java
public class DaemonDemo {
    public static void main(String[] args) {
        Thread cleanerThread = new Thread(() -> {
            while (true) {
                System.out.println("Cleaning up background junk...");
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            }
        });

        // MUST be set before starting the thread
        cleanerThread.setDaemon(true); 
        cleanerThread.start();

        System.out.println("Main thread is doing important work...");
        try { Thread.sleep(2000); } catch (InterruptedException e) {}
        
        System.out.println("Main thread (User Thread) is finishing.");
        // At this point, cleanerThread is killed instantly by the JVM.
    }
}
```
### 3. Critical Rules for the "Dance"
1. **The Point of No Return:** You must call `setDaemon(true)` **before** the thread is started. If you call it after `start()`, Java will throw an `IllegalThreadStateException`.
2. **Inheritance:** If a thread is a Daemon, any "child" threads it creates will **automatically** be Daemons too.
3. **No `finally` guarantee:** Because the JVM kills Daemon threads abruptly, `finally` blocks inside a Daemon thread might **not** execute. Never use a Daemon thread for persistent operations like writing to a database or closing a file.
### Real-World "Daemons"
You are already using them! The **Garbage Collector (GC)** is the most famous Daemon thread. It runs in the background, cleaning up your memory, but it doesn't stop your program from closing when you're done.

---
## Parent vs Child Thread
*A thread under which some new thread is created/started is called __ ? and newly spawned thread is called ___  ? are they called parent thread and child thread? i want to know Java universe terminology.*
### 1. The Terminology
- **Parent Thread:** The thread that executes the `new Thread()` constructor and the `.start()` method.
- **Child Thread:** The newly created and started thread.
In Java, **every thread has a parent**. Even the very first thread that runs your `public static void main` is created by the Java Virtual Machine (JVM) and is typically called the **"main" thread**.
### 2. The Relationship "Rules"
While we use the terms Parent and Child, the relationship in Java is flatter than in some other operating systems (like Unix processes). Here is how the "dance" works:
- **Independence:** Once a child thread is started, it is **independent** of its parent. If the parent thread finishes its execution, the child thread keeps running until its own `run()` method completes.
- **Inheritance:** The child thread inherits certain properties from its parent at the moment of creation:
    - **Priority:** It starts with the same priority level as the parent.
    - **Daemon Status:** If the parent is a Daemon thread (a background "service" thread), the child will be a Daemon thread by default.
    - **Context ClassLoader:** It inherits the parent's class loader.
### 3. The Exception: Daemon Threads
There is one major way a "Parent" (the JVM/Main process) affects its children.
If you mark a child thread as a **Daemon**, it will be forcibly terminated by the JVM as soon as all **non-daemon** threads (like the main thread) have finished.

```Java
Thread child = new Thread(() -> {
    while(true) {
        System.out.println("Child dancing...");
    }
});

child.setDaemon(true); // This makes it a background helper
child.start();

System.out.println("Main thread (Parent) is finishing.");
// Once main ends, the Daemon child is killed instantly by the JVM.
```

---
### 4. Comparison Table

|**Feature**|**Parent Thread**|**Child Thread**|
|---|---|---|
|**Creation**|Invokes `start()`.|Created via `new Thread()`.|
|**Lifecycle**|Can exit while child runs.|Can continue after parent exits.|
|**Hierarchy**|Often the "Main" thread.|Often a "Worker" thread.|

> **Note:** Even though we call them Parent and Child, there is no built-in `getParent()` method in the standard `Thread` class. Java treats them as peers once they are both in the **RUNNABLE** state.




# Thread Synchronization in Java

There are multiple ways of controlling thread lifecycle (inter-thread synchronization ) : 
- wait() and notify() : SHARED OBJECT (monitor)
- join()
# wait() vs join()
*So difference between `wait()` & `join()` - no argument variation(for simplicity) can be said to be as follows :*

*`wait()`: the JOINING Thread waits() until TARGET thread manually notifies using notify() or notifyAll().*  
*while in*
*`join()` : the JOINING Thread waits until TARGET thread reaches TERMINATED state (and responsibility of notifying to JOINING thread is that of JVM) .*

*correct if any error in above observation* 

## Your Observation — What is Correct and What Needs Fixing

### The join() Part — ✅ Perfectly Correct

```js
join(): the JOINING Thread waits until TARGET thread reaches
        TERMINATED state (and responsibility of notifying
        the JOINING thread is that of JVM)

→ 100% accurate. Nothing to fix here.
```

---

### The wait() Part — ❌ Two Errors

Your statement:
```
wait(): the JOINING Thread waits() until TARGET thread
        manually notifies using notify() or notifyAll()
```

**Error 1 — Wrong terminology: "JOINING Thread"**

```
"JOINING Thread" and "TARGET Thread" are terms
that belong EXCLUSIVELY to join().

In wait(), the thread that calls wait() is called:

  → WAITING Thread   (not Joining Thread)

"Joining Thread" implies join() is involved.
wait() has nothing to do with joining.
```

**Error 2 — Wrong concept: "TARGET thread notifies"**

```
In wait(), there is NO concept of a TARGET thread.

wait() does NOT wait for a SPECIFIC THREAD.
wait() waits for a CONDITION on a SHARED OBJECT (monitor).

The notify() can be called by ANY thread
that holds the lock on that SAME OBJECT.
It does not have to be one specific "target" thread.

This is the deepest conceptual difference between the two.
```

---
## The Corrected Statement

```vbnet
wait(): the WAITING Thread waits on a SHARED OBJECT's monitor
        until ANY other thread that holds the lock on that
        SAME OBJECT manually calls notify() or notifyAll().

join(): the JOINING Thread waits until the TARGET Thread
        reaches TERMINATED state (JVM is responsible for
        the notification automatically).
```

---
The join method is defined in the _[Thread](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html#join\(\))_ class:

> _public final void join() throws InterruptedException_  
> _Waits for this thread to die._

## The Core Distinction in One Line Each

```sql
wait()  →  thread-AGNOSTIC  — waits for a CONDITION on an OBJECT
                              (any thread can satisfy it)

join()  →  thread-SPECIFIC  — waits for ONE SPECIFIC THREAD to die
                              (only that thread's death satisfies it)
```

---
## Visualized

```vbnet
wait() — Object centric
═══════════════════════

  WAITING Thread          Shared Object          ANY other Thread
  ──────────────          ─────────────          ────────────────
  calls wait()  ────────► releases lock
  suspended                on Object
                                │
                                │             acquires lock on Object
                                │             changes condition
                                │             calls notify() ◄────────
                                │
  wakes up ◄────────────────────┘
  re-acquires lock
  checks condition


join() — Thread centric
════════════════════════

  JOINING Thread          TARGET Thread
  ──────────────          ─────────────
  calls join()  ────────► waits for THIS
  suspended               specific thread
                                │
                                │  reaches TERMINATED
                                │  JVM auto-notifies ◄── no other thread
                                │                         needed
  wakes up ◄────────────────────┘
  TARGET confirmed dead


KEY:
  wait()  — shared object is the center of coordination
  join()  — the target thread itself is the center of coordination
```

---

## Summary Table

```
╔══════════════════╦══════════════════════════╦══════════════════════════╗
║                  ║        wait()            ║        join()            ║
╠══════════════════╬══════════════════════════╬══════════════════════════╣
║ Waiting thread   ║ WAITING Thread           ║ JOINING Thread           ║
║ is called        ║                          ║                          ║
╠══════════════════╬══════════════════════════╬══════════════════════════╣
║ Other side       ║ ANY thread holding       ║ TARGET Thread            ║
║ is called        ║ the same lock            ║ (specific one)           ║
╠══════════════════╬══════════════════════════╬══════════════════════════╣
║ Waiting for      ║ A CONDITION on           ║ ONE SPECIFIC thread      ║
║                  ║ a shared OBJECT          ║ to TERMINATE             ║
╠══════════════════╬══════════════════════════╬══════════════════════════╣
║ Who notifies     ║ ANY thread manually      ║ JVM automatically        ║
║                  ║ calling notify()         ║ on thread death          ║
╠══════════════════╬══════════════════════════╬══════════════════════════╣
║ Centric to       ║ OBJECT (monitor)         ║ THREAD (lifecycle)       ║
╚══════════════════╩══════════════════════════╩══════════════════════════╝
```
