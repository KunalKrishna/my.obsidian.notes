# Thread Lifecycle — 6 States — Comprehensive Notes

---

## The Story Before the States

```
A thread is not always running.
A thread is born, lives through different phases, and eventually dies.

Think of a contractor (thread) hired for a project:

  NEW          → Hired but not yet shown up for work
  RUNNABLE     → At work — either working or waiting for a desk (CPU)
  BLOCKED      → Waiting outside a locked room (waiting for a lock)
  WAITING      → Sitting in a waiting room — told "wait until called"
  TIMED_WAITING→ Sitting in waiting room — "wait for max 1 hour"
  TERMINATED   → Work is done. Contractor has left the building.

These are the EXACT 6 states Java defines in Thread.State enum.
Every thread in the JVM is always in exactly ONE of these 6 states.
Understanding transitions between them is the foundation of
understanding every concurrency primitive that exists.
```

---

## The 6 States — Quick Reference First

```
java.lang.Thread.State  (enum)

  NEW             → Thread created, start() NOT yet called
  RUNNABLE        → start() called, running or waiting for CPU
  BLOCKED         → Waiting to acquire a synchronized lock
  WAITING         → Waiting indefinitely for notification
  TIMED_WAITING   → Waiting for specified duration
  TERMINATED      → run() has returned or threw uncaught exception

Checking state in code:
  Thread t = new Thread(() -> {});
  System.out.println(t.getState()); // Thread.State enum value
  System.out.println(t.isAlive());  // true for RUNNABLE, BLOCKED,
                                    //          WAITING, TIMED_WAITING
                                    // false for NEW, TERMINATED
```

---

## The Complete State Machine

```
╔═════════════════════════════════════════════════════════════════════════════╗
║                    THREAD LIFECYCLE — COMPLETE STATE MACHINE                ║
╚═════════════════════════════════════════════════════════════════════════════╝

                         new Thread()
                              │
                              ▼
                    ┌─────────────────┐
                    │      NEW        │
                    │                 │
                    │ Thread object   │
                    │ exists, no OS   │
                    │ thread yet      │
                    └────────┬────────┘
                             │
                             │  t.start()
                             │  ↓ OS thread created
                             │  ↓ Stack allocated
                             │  ↓ Scheduler notified
                             ▼
    ┌───────────────────────────────────────────────┐
    │                  RUNNABLE                     │
    │                                               │
    │   ┌─────────────────┐   ┌──────────────────┐  │
    │   │    RUNNING      │   │  READY (queued)  │  │
    │   │  (on CPU)       │◄──►  (waiting for    │  │
    │   │                 │   │   CPU timeslice) │  │
    │   └─────────────────┘   └──────────────────┘  │
    │             ↑   JVM combines these into ONE   │
    │             │   RUNNABLE state                │
    └──────┬──────┴──────┬──────────────┬───────────┘
           │             │              │
           │             │              │
     synchronized    wait()         sleep(n)
     lock not        join()         wait(n)
     available       LockSupport    join(n)
           │         .park()     LockSupport
           │             │        .parkNanos()
           ▼             ▼              ▼
    ┌──────────┐  ┌──────────────┐  ┌────────────────┐
    │ BLOCKED  │  │   WAITING    │  │ TIMED_WAITING  │
    │          │  │              │  │                │
    │ waiting  │  │ waiting      │  │ waiting for    │
    │ for lock │  │ indefinitely │  │ specified time │
    └────┬─────┘  └──────┬───────┘  └───────┬────────┘
         │               │                  │
         │  lock       notify()           timeout
         │  available  notifyAll()        expires  OR
         │             interrupt()        notify()
         │             unpark()           interrupt()
         │               │                  │
         └───────────────┴──────────────────┘
                         │
                         ▼
                    RUNNABLE ◄─────────────────────────────────┐
                         │                                     │
                         │  run() returns                      │
                         │  OR                                 │
                         │  uncaught exception thrown          │
                         ▼                                     │
                  ┌──────────────┐                             │
                  │  TERMINATED  │  ← final state              │
                  │              │     cannot go back ─────────┘
                  │  Thread dead │     to any state (X)
                  └──────────────┘
```

---

## State 1 — NEW

### What It Is

```
A thread in NEW state is:
  - A Java Thread object exists in memory (on the Heap)
  - NO corresponding OS thread has been created yet
  - No Stack has been allocated for this thread
  - The thread has NEVER been started
  - Thread.isAlive() returns FALSE

It is just a regular Java object — nothing special about it yet.
The actual "thread" in the OS sense does not exist.
```

### Code

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 1: NEW
// ═══════════════════════════════════════════════════════════════
public class NewStateDemo {

    public static void main(String[] args) {

        // Thread created — in NEW state immediately
        Thread t1 = new Thread(() -> System.out.println("Hello"));

        // Thread is NEW — not yet started
        System.out.println(t1.getState());   // NEW
        System.out.println(t1.isAlive());    // false

        // At this point:
        //   - Thread object exists on Heap ✓
        //   - OS thread: NOT created ✗
        //   - Stack for t1: NOT allocated ✗
        //   - t1 is NOT scheduled by OS ✗

        // ── NEW thread properties ─────────────────────────────────
        System.out.println(t1.getName());     // Thread-0 (auto-assigned)
        System.out.println(t1.getId());       // unique JVM-assigned ID
        System.out.println(t1.getPriority()); // 5 (NORM_PRIORITY)
        System.out.println(t1.isDaemon());    // false (inherits from creator)

        // ── Things you can do in NEW state ───────────────────────
        t1.setName("my-worker");              // ✓ allowed
        t1.setPriority(Thread.MAX_PRIORITY);  // ✓ allowed
        t1.setDaemon(true);                   // ✓ allowed (MUST do before start)

        // ── Things you CANNOT do in NEW state ─────────────────────
        // t1.join();    // ← does nothing useful (returns immediately)
        //               //   thread is not alive — nothing to wait for
        // t1.interrupt(); // ← sets flag but thread not alive yet
        //                 //   has no effect

        // ── Transition: NEW → RUNNABLE via start() ────────────────
        t1.start();
        // NOW: OS thread created, Stack allocated, scheduler notified
        // t1 moves to RUNNABLE state
        System.out.println(t1.getState());   // RUNNABLE (probably)
        System.out.println(t1.isAlive());    // true

        // ── Cannot start twice ────────────────────────────────────
        try {
            t1.start(); // IllegalThreadStateException!
        } catch (IllegalThreadStateException e) {
            System.out.println("Cannot start thread twice: " + e.getMessage());
        }
    }
}
```

```
NEW STATE — MEMORY PICTURE:

  HEAP:
  ┌──────────────────────────────────────┐
  │  Thread object (t1)                  │
  │    name     = "Thread-0"             │
  │    priority = 5                      │
  │    daemon   = false                  │
  │    state    = NEW                    │
  │    target   = ref→ lambda Runnable   │
  │    threadId = 1                      │
  │    stackSize= 0 (not allocated yet)  │
  └──────────────────────────────────────┘

  OS THREAD TABLE:
  (t1 does not appear here yet — no OS thread exists)

  AFTER start():
  ┌──────────────────────────────────────┐
  │  Thread object (t1)                  │
  │    state    = RUNNABLE               │
  │    eetop    = [OS thread handle]     │  ← pointer to OS thread
  └──────────────────────────────────────┘

  OS THREAD TABLE:
  ┌─────────────────────────────────────────────────────┐
  │  OS Thread ID: 7823  ← created by start()           │
  │  Stack: [1MB allocated]                             │
  │  State: RUNNABLE                                    │
  │  Priority: 5                                        │
  └─────────────────────────────────────────────────────┘
```

---

## State 2 — RUNNABLE

### What It Is

```
RUNNABLE is the most nuanced state because it covers TWO sub-states
that the OS distinguishes but Java's JVM combines into ONE:

  Sub-state 1: RUNNING
    Thread IS executing on a CPU core right now.
    CPU instructions are being executed.

  Sub-state 2: READY (also called READY-TO-RUN)
    Thread is NOT on CPU right now.
    Thread is in the OS scheduler's ready queue.
    Waiting for a CPU time slice.
    Could run the moment a CPU core becomes available.

WHY Java combines them:
  Java does not expose OS-level scheduling details.
  The transition between RUNNING and READY happens thousands
  of times per second (context switching).
  It is below the level of abstraction Java provides.
  To Java: if it COULD be running → it IS RUNNABLE.

Thread.isAlive() returns TRUE in RUNNABLE state.
```

### Code

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 2: RUNNABLE
// ═══════════════════════════════════════════════════════════════
public class RunnableStateDemo {

    public static void main(String[] args) throws InterruptedException {

        // ── RUNNABLE: actively computing ──────────────────────────
        Thread cpuBound = new Thread(() -> {
            // CPU-intensive work — stays RUNNABLE throughout
            long sum = 0;
            for (long i = 0; i < Long.MAX_VALUE; i++) {
                sum += i;
                if (Thread.currentThread().isInterrupted()) break;
            }
        }, "cpu-bound-thread");

        cpuBound.start();
        Thread.sleep(50); // give it time to start running

        System.out.println(cpuBound.getState()); // RUNNABLE
        System.out.println(cpuBound.isAlive());  // true

        cpuBound.interrupt(); // stop the loop
        cpuBound.join();

        // ── RUNNABLE: doing I/O ───────────────────────────────────
        // IMPORTANT: threads doing I/O (file read, network call)
        // are STILL in RUNNABLE state from JVM's perspective!
        // The OS may internally block the thread on I/O,
        // but JVM reports it as RUNNABLE.
        // This is a common source of confusion.

        Thread ioBound = new Thread(() -> {
            try {
                // Reading from a file, network socket, DB —
                // JVM still reports RUNNABLE even while "waiting" for I/O
                // because from JVM's view: thread is not blocked on a LOCK
                // and not in wait() — it's just executing native I/O code
                new java.net.URL("http://example.com").openStream();
            } catch (Exception e) {
                // ignore for demo
            }
        }, "io-bound-thread");

        ioBound.start();
        Thread.sleep(50);
        System.out.println(ioBound.getState()); // RUNNABLE (even during I/O!)
        ioBound.join();

        // ── Thread.yield() — voluntary RUNNING → READY ────────────
        Thread yielder = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Working: " + i);
                Thread.yield();
                // Hint to scheduler: "I'm willing to give up CPU"
                // Moves from RUNNING to READY sub-state
                // Stays RUNNABLE — just gives other threads a chance
                // Scheduler may IGNORE this hint entirely
            }
        }, "yielder-thread");

        yielder.start();
        yielder.join();
    }
}
```

```
RUNNABLE — THE TWO SUB-STATES:

  What OS sees:                What JVM/Java sees:
  ─────────────                ───────────────────

  ┌─────────────────────┐
  │  RUNNING            │ ────────────────────────────────────────────►
  │  (on CPU)           │                                          RUNNABLE
  └─────────────────────┘ ────────────────────────────────────────────►
  ┌─────────────────────┐
  │  READY              │  Both combined into ONE state
  │  (in queue)         │  JVM does not distinguish them
  └─────────────────────┘

  Why?
  Context switching between RUNNING ↔ READY
  happens every 1-10 milliseconds (configurable OS time slice).
  At that granularity, tracking it in Java would be meaningless noise.
  The JVM abstraction says: "this thread is eligible to run" = RUNNABLE.
```

---

## State 3 — BLOCKED

### What It Is

```
A thread is BLOCKED when:
  It tried to enter a synchronized block or method
  AND another thread is currently holding that monitor lock.

The thread is physically STOPPED.
It is in the OS's BLOCKED queue for that monitor.
It uses 0% CPU.
It will automatically move to RUNNABLE when the lock is released.
NO explicit notification needed — OS handles it automatically.

BLOCKED is the ONLY state caused exclusively by synchronized.
BLOCKED is different from WAITING:
  BLOCKED: waiting for a LOCK (synchronized)
  WAITING: waiting for an explicit SIGNAL (wait/notify)
```

### Code

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 3: BLOCKED
// ═══════════════════════════════════════════════════════════════
public class BlockedStateDemo {

    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {

        // Thread 1: acquires lock and holds it for a long time
        Thread lockHolder = new Thread(() -> {
            synchronized (lock) {          // acquires lock
                System.out.println("LockHolder: I have the lock");
                try {
                    Thread.sleep(5000);    // holds lock for 5 seconds
                    // IMPORTANT: sleep() does NOT release the lock!
                    // lock is HELD the entire 5 seconds
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("LockHolder: releasing lock");
            }                              // releases lock here
        }, "lock-holder");

        // Thread 2: tries to acquire the same lock — will BLOCK
        Thread blocker = new Thread(() -> {
            System.out.println("Blocker: trying to get lock...");
            synchronized (lock) {          // ← BLOCKS HERE if lock not available
                System.out.println("Blocker: got lock!");
                // Only reaches here after lockHolder releases
            }
        }, "blocker");

        // Thread 3: another thread trying for same lock — also BLOCKS
        Thread blocker2 = new Thread(() -> {
            System.out.println("Blocker2: trying to get lock...");
            synchronized (lock) {          // ← ALSO BLOCKS
                System.out.println("Blocker2: got lock!");
            }
        }, "blocker-2");

        lockHolder.start();
        Thread.sleep(100);   // ensure lockHolder gets the lock first
        blocker.start();
        blocker2.start();
        Thread.sleep(200);   // give blockers time to reach synchronized

        // Check states
        System.out.println("lockHolder: " + lockHolder.getState()); // TIMED_WAITING
                                                                     // (sleeping)
        System.out.println("blocker  : " + blocker.getState());    // BLOCKED
        System.out.println("blocker2 : " + blocker2.getState());   // BLOCKED

        lockHolder.join();
        blocker.join();
        blocker2.join();
    }
}
```

```
BLOCKED — WHAT HAPPENS INTERNALLY:

  Monitor (lock object) has:
    1. Owner field     → which thread currently holds it (null if free)
    2. Entry Set       → threads BLOCKED waiting to acquire it
    3. Wait Set        → threads in wait() (WAITING state)

  ┌──────────────────────────────────────────────────────────┐
  │   Monitor: 'lock' object                                 │
  │                                                          │
  │   Owner:    lockHolder thread ← holds the lock           │
  │                                                          │
  │   Entry Set (BLOCKED threads):                           │
  │     ├── blocker  thread   ← BLOCKED, waiting for lock    │
  │     └── blocker2 thread   ← BLOCKED, waiting for lock    │
  │                                                          │
  │   Wait Set (WAITING threads):                            │
  │     (empty for now)                                      │
  └──────────────────────────────────────────────────────────┘

  When lockHolder releases lock:
    OS picks ONE thread from Entry Set (blocker or blocker2)
    That thread moves: BLOCKED → RUNNABLE → acquires lock
    Other stays BLOCKED

  Thread selection from entry set:
    Not guaranteed to be FIFO!
    OS decides — may be random or priority-based
    For fairness guarantees → use ReentrantLock(true)
    synchronized keyword has NO fairness guarantee


BLOCKED vs WAITING — the KEY difference:

  BLOCKED:  waiting for a LOCK to be released
            OS monitors the lock — wakes thread automatically
            No explicit signal needed
            Happens ONLY with synchronized keyword

  WAITING:  waiting for an explicit SIGNAL (notify/notifyAll)
            Nobody wakes it automatically — must call notify()
            Happens with wait(), join(), LockSupport.park()
```

---

## State 4 — WAITING

### What It Is

```
A thread is WAITING when:
  It has voluntarily given up CPU and is waiting INDEFINITELY
  for some other thread to explicitly wake it up.

"Indefinitely" = no timeout — could wait forever if nobody signals.

Causes of WAITING state:
  object.wait()           → waiting on an object's monitor
  thread.join()           → waiting for target thread to die
  LockSupport.park()      → waiting for unpark() call

To exit WAITING:
  object.notify()         → owner calls on same object
  object.notifyAll()      → owner calls on same object
  LockSupport.unpark(t)   → some thread calls this
  t.interrupt()           → interrupt the waiting thread
                            (throws InterruptedException)

A thread in WAITING is not consuming any CPU.
It is removed from the CPU scheduler queue entirely.
It sits in the monitor's WAIT SET until woken.
```

### Code — wait() and WAITING state

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 4: WAITING — via wait()
// ═══════════════════════════════════════════════════════════════
public class WaitingStateDemo {

    static final Object lock  = new Object();
    static boolean      ready = false;

    public static void main(String[] args) throws InterruptedException {

        // ── WAITING via wait() ────────────────────────────────────
        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                System.out.println("Waiter: acquired lock");
                while (!ready) {
                    try {
                        System.out.println("Waiter: going to WAITING state");
                        lock.wait();
                        // ↑ THREE things happen atomically:
                        //   1. Releases the lock on 'lock' object
                        //   2. Suspends this thread (WAITING state)
                        //   3. Moves thread from Entry Set → Wait Set
                        //
                        // Thread is now in WAITING state.
                        // Uses 0% CPU.
                        // Lock is available for others to acquire.
                        System.out.println("Waiter: woken up from WAITING");
                        // Re-acquired lock before returning from wait()
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
                System.out.println("Waiter: condition is ready, proceeding");
            }
        }, "waiter-thread");

        // ── WAITING via join() ────────────────────────────────────
        Thread longTask = new Thread(() -> {
            try {
                Thread.sleep(3000);
                System.out.println("LongTask: done");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "long-task");

        Thread joiner = new Thread(() -> {
            try {
                System.out.println("Joiner: waiting for longTask to finish");
                longTask.join();
                // ↑ joiner thread enters WAITING state here
                //   waiting on the longTask Thread object's monitor
                //   woken automatically when longTask TERMINATES
                //   (JVM calls notifyAll() on Thread object on termination)
                System.out.println("Joiner: longTask is done");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "joiner-thread");

        // ── Start threads and observe states ──────────────────────
        waiter.start();
        Thread.sleep(100); // give waiter time to enter wait()

        System.out.println("waiter state: " + waiter.getState()); // WAITING

        longTask.start();
        joiner.start();
        Thread.sleep(100);

        System.out.println("joiner state: " + joiner.getState());   // WAITING
        System.out.println("longTask state: " + longTask.getState()); // TIMED_WAITING

        // ── Wake the waiter ───────────────────────────────────────
        synchronized (lock) {
            ready = true;
            lock.notifyAll();
            // ↑ Moves waiter from Wait Set → Entry Set
            //   Waiter tries to re-acquire lock
            //   When it does: WAITING → BLOCKED → RUNNABLE
        }

        waiter.join();
        joiner.join();
        longTask.join();
    }
}
```

```
WAITING — INTERNAL MONITOR PICTURE:

  BEFORE wait():
  ┌──────────────────────────────────────────────────────────┐
  │   Monitor: 'lock' object                                 │
  │   Owner:    waiter thread  ← holds lock                  │
  │   Entry Set: (empty)                                     │
  │   Wait Set:  (empty)                                     │
  └──────────────────────────────────────────────────────────┘

  DURING wait() — after lock.wait() called:
  ┌──────────────────────────────────────────────────────────┐
  │   Monitor: 'lock' object                                 │
  │   Owner:    null  ← lock RELEASED by waiter              │
  │   Entry Set: (empty)                                     │
  │   Wait Set:  waiter thread  ← moved here by wait()       │
  └──────────────────────────────────────────────────────────┘

  AFTER notifyAll() called:
  ┌──────────────────────────────────────────────────────────┐
  │   Monitor: 'lock' object                                 │
  │   Owner:    main thread  ← still holding lock (notifier) │
  │   Entry Set: waiter thread  ← moved from Wait Set here   │
  │   Wait Set:  (empty)                                     │
  └──────────────────────────────────────────────────────────┘
  State: waiter is now BLOCKED (trying to re-acquire)

  AFTER notifier releases lock (exits synchronized):
  ┌──────────────────────────────────────────────────────────┐
  │   Monitor: 'lock' object                                 │
  │   Owner:    waiter thread  ← re-acquired lock            │
  │   Entry Set: (empty)                                     │
  │   Wait Set:  (empty)                                     │
  └──────────────────────────────────────────────────────────┘
  State: waiter is RUNNABLE again, inside synchronized block
```

---

## State 5 — TIMED_WAITING

### What It Is

```
TIMED_WAITING is identical to WAITING with ONE addition:
  The thread has a TIMEOUT.
  If nobody wakes it before the timeout → it wakes up automatically.

This is the "wait, but not forever" state.

Causes of TIMED_WAITING state:
  Thread.sleep(millis)          → most common
  Thread.sleep(millis, nanos)
  object.wait(millis)           → wait with timeout
  object.wait(millis, nanos)
  thread.join(millis)           → join with timeout
  thread.join(millis, nanos)
  LockSupport.parkNanos(nanos)  → park with timeout
  LockSupport.parkUntil(millis) → park until absolute time

To exit TIMED_WAITING:
  Timeout expires                → automatic, no signal needed
  notify() / notifyAll()         → for wait(n) variant
  LockSupport.unpark(t)          → for parkNanos variant
  thread.interrupt()             → throws InterruptedException
```

### Code

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 5: TIMED_WAITING — all causes
// ═══════════════════════════════════════════════════════════════
public class TimedWaitingStateDemo {

    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {

        // ── TIMED_WAITING via Thread.sleep() ──────────────────────
        Thread sleeper = new Thread(() -> {
            try {
                System.out.println("Sleeper: going to sleep for 5 seconds");
                Thread.sleep(5000);
                // Thread is in TIMED_WAITING for 5 seconds
                // Does NOT release any locks it holds
                // Wakes up automatically after 5000ms
                // OR immediately if interrupted (InterruptedException)
                System.out.println("Sleeper: woke up");
            } catch (InterruptedException e) {
                System.out.println("Sleeper: interrupted early");
                Thread.currentThread().interrupt();
            }
        }, "sleeper-thread");

        sleeper.start();
        Thread.sleep(100);
        System.out.println("sleeper: " + sleeper.getState()); // TIMED_WAITING

        sleeper.interrupt(); // wake it early
        sleeper.join();

        // ── TIMED_WAITING via wait(millis) ────────────────────────
        Thread timedWaiter = new Thread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("TimedWaiter: waiting with timeout");
                    lock.wait(3000);
                    // Releases lock AND waits for UP TO 3 seconds
                    // Exits if:
                    //   a) notify() is called on lock → woken early
                    //   b) 3000ms elapses → woken automatically
                    //   c) interrupt() called → InterruptedException
                    System.out.println("TimedWaiter: wait ended");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "timed-waiter-thread");

        timedWaiter.start();
        Thread.sleep(100);
        System.out.println("timedWaiter: " + timedWaiter.getState()); // TIMED_WAITING
        timedWaiter.join();

        // ── TIMED_WAITING via join(millis) ─────────────────────────
        Thread longRunner = new Thread(() -> {
            try { Thread.sleep(10_000); } catch (InterruptedException e) {}
        }, "long-runner");

        Thread timedJoiner = new Thread(() -> {
            try {
                longRunner.join(2000);
                // Waits for longRunner to die, but max 2 seconds
                // State: TIMED_WAITING while waiting
                // After 2 seconds → returns regardless
                if (longRunner.isAlive()) {
                    System.out.println("TimedJoiner: gave up waiting");
                } else {
                    System.out.println("TimedJoiner: longRunner finished in time");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "timed-joiner-thread");

        longRunner.start();
        timedJoiner.start();
        Thread.sleep(100);
        System.out.println("timedJoiner: " + timedJoiner.getState()); // TIMED_WAITING
        timedJoiner.join();
        longRunner.interrupt();
        longRunner.join();
    }
}
```

```
TIMED_WAITING — TIMEOUT COMPARISON:

  ┌─────────────────────────────────────────────────────────────┐
  │  METHOD          │  RELEASES LOCK? │  WOKEN BY              │
  ├──────────────────┼─────────────────┼────────────────────────┤
  │  sleep(n)        │  NO             │  timeout / interrupt   │
  ├──────────────────┼─────────────────┼────────────────────────┤
  │  wait(n)         │  YES            │  timeout / notify /    │
  │                  │                 │  notifyAll / interrupt │
  ├──────────────────┼─────────────────┼────────────────────────┤
  │  join(n)         │  NO (no lock)   │  timeout / target dies │
  │                  │                 │  / interrupt           │
  ├──────────────────┼─────────────────┼────────────────────────┤
  │  parkNanos(n)    │  NO (no lock)   │  timeout / unpark /    │
  │                  │                 │  interrupt             │
  └─────────────────────────────────────────────────────────────┘

  CRITICAL: sleep() does NOT release locks!
  wait(n)   DOES release locks!
  This is the most important difference between sleep and wait.
```

---

## State 6 — TERMINATED

### What It Is

```
A thread reaches TERMINATED when:
  1. run() method returns normally
  2. run() method throws an UncaughtException
  3. The thread is stopped (deprecated Thread.stop() — never use)

TERMINATED is the FINAL state.
A terminated thread CANNOT be restarted.
t.start() on a TERMINATED thread → IllegalThreadStateException.

Thread.isAlive() returns FALSE for TERMINATED.
All threads waiting on t.join() are notified automatically.
The Thread object still exists in memory (on Heap) until GC collects it.
But the OS thread is GONE — Stack and OS resources are freed.
```

### Code

```java
// ═══════════════════════════════════════════════════════════════
//  STATE 6: TERMINATED — both normal and exceptional
// ═══════════════════════════════════════════════════════════════
public class TerminatedStateDemo {

    public static void main(String[] args) throws InterruptedException {

        // ── TERMINATED: normal completion ─────────────────────────
        Thread normal = new Thread(() -> {
            System.out.println("Normal: doing work");
            System.out.println("Normal: work done, run() returning");
            // run() returns → thread TERMINATES
        }, "normal-thread");

        normal.start();
        normal.join(); // wait for termination
        System.out.println("normal: " + normal.getState()); // TERMINATED
        System.out.println("normal alive: " + normal.isAlive()); // false

        // ── TERMINATED: uncaught exception ────────────────────────
        Thread exceptional = new Thread(() -> {
            System.out.println("Exceptional: about to throw");
            throw new RuntimeException("Something went wrong!");
            // run() exits via uncaught exception → thread TERMINATES
            // Exception goes to UncaughtExceptionHandler
        }, "exceptional-thread");

        // Register handler BEFORE start
        exceptional.setUncaughtExceptionHandler((thread, ex) -> {
            System.out.println("UncaughtHandler: thread '"
                    + thread.getName() + "' threw: " + ex.getMessage());
            // This handler runs on a different thread (not exceptional)
            // Thread is already TERMINATED when this runs
        });

        exceptional.start();
        exceptional.join(); // join still works — waits for termination
        System.out.println("exceptional: " + exceptional.getState()); // TERMINATED

        // ── Cannot restart a TERMINATED thread ────────────────────
        try {
            normal.start(); // IllegalThreadStateException!
        } catch (IllegalThreadStateException e) {
            System.out.println("Cannot restart: " + e.getClass().getSimpleName());
        }

        // ── Thread object still exists after TERMINATED ────────────
        // The Java object is still in memory
        // Can still read its properties
        System.out.println("Name after termination: " + normal.getName());
        System.out.println("ID after termination  : " + normal.getId());
        System.out.println("State after termination: " + normal.getState()); // TERMINATED
        // Just cannot restart it
        // Object will be GC'd when no more references to it

        // ── JVM exits when last non-daemon thread terminates ──────
        // main() returning = main thread TERMINATES
        // If no other non-daemon threads are alive → JVM exits
        System.out.println("Main: done");
    }
}
```

---

## Complete State Transition Table

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                 ALL STATE TRANSITIONS — COMPLETE REFERENCE                   ║
╠══════════════════╦═════════════════╦═════════════════════════════════════════╣
║  FROM            ║  TO             ║  CAUSED BY                              ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  NEW             ║  RUNNABLE       ║  t.start()                              ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  RUNNABLE        ║  BLOCKED        ║  Trying to enter synchronized block/    ║
║                  ║                 ║  method where another thread holds lock ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  RUNNABLE        ║  WAITING        ║  object.wait()                          ║
║                  ║                 ║  thread.join() (no timeout)             ║
║                  ║                 ║  LockSupport.park()                     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  RUNNABLE        ║  TIMED_WAITING  ║  Thread.sleep(n)                        ║
║                  ║                 ║  object.wait(n)                         ║
║                  ║                 ║  thread.join(n)                         ║
║                  ║                 ║  LockSupport.parkNanos(n)               ║
║                  ║                 ║  LockSupport.parkUntil(deadline)        ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  RUNNABLE        ║  TERMINATED     ║  run() returns normally                 ║
║                  ║                 ║  run() throws uncaught exception        ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  BLOCKED         ║  RUNNABLE       ║  Lock becomes available                 ║
║                  ║                 ║  (owning thread exits synchronized)     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  WAITING         ║  BLOCKED        ║  notify() called → moves to Entry Set   ║
║  (via wait())    ║                 ║  notifyAll() → all move to Entry Set    ║
║                  ║                 ║  Must re-acquire lock → becomes BLOCKED ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  WAITING         ║  RUNNABLE       ║  LockSupport.unpark(t) called           ║
║  (via park())    ║                 ║  interrupt() called                     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  WAITING         ║  RUNNABLE       ║  Target thread terminates (for join())  ║
║  (via join())    ║                 ║  interrupt() called                     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  TIMED_WAITING   ║  RUNNABLE       ║  Timeout expires (for sleep, parkNanos) ║
║  (via sleep)     ║                 ║  interrupt() → InterruptedException     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  TIMED_WAITING   ║  BLOCKED        ║  notify()/notifyAll() called before     ║
║  (via wait(n))   ║                 ║  timeout — must re-acquire lock         ║
║                  ╠═════════════════╬═════════════════════════════════════════╣
║                  ║  RUNNABLE       ║  Timeout expires                        ║
║                  ║                 ║  interrupt() → InterruptedException     ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  TIMED_WAITING   ║  RUNNABLE       ║  Target dies / timeout / interrupt()    ║
║  (via join(n))   ║                 ║                                         ║
╠══════════════════╬═════════════════╬═════════════════════════════════════════╣
║  TERMINATED      ║  (none)         ║  Final state — no transitions out       ║
╚══════════════════╩═════════════════╩═════════════════════════════════════════╝
```

---

## Observing All 6 States in One Program

```java
// ═══════════════════════════════════════════════════════════════
//  ALL 6 STATES IN ONE PROGRAM — complete demonstration
// ═══════════════════════════════════════════════════════════════
public class AllSixStatesDemo {

    static final Object lock       = new Object();
    static final Object waitLock   = new Object();
    static       boolean condition = false;

    public static void main(String[] args) throws InterruptedException {

        // ─────────────────────────────────────────────────────────
        // STATE 1: NEW
        // ─────────────────────────────────────────────────────────
        Thread t = new Thread(() -> {
            // This thread will be used to demonstrate all states
            try {
                // Will be RUNNABLE when start() is called

                // Will become TIMED_WAITING
                Thread.sleep(2000);

                // Will become WAITING
                synchronized (waitLock) {
                    while (!condition) {
                        waitLock.wait();
                    }
                }

                // CPU work — RUNNABLE
                long sum = 0;
                for (long i = 0; i < 100_000_000L; i++) sum += i;
                System.out.println("Sum: " + sum);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // run() returns → TERMINATED
        }, "demo-thread");

        // STATE 1: NEW — thread object created, not started
        printState("After new Thread()", t);
        // Output: NEW    isAlive: false

        // ─────────────────────────────────────────────────────────
        // STATE 2: RUNNABLE (very briefly) then TIMED_WAITING
        // ─────────────────────────────────────────────────────────
        t.start();
        // Immediately after start() → RUNNABLE
        // Then thread calls sleep(2000) → TIMED_WAITING
        printState("Right after start()", t);
        // Output: RUNNABLE or TIMED_WAITING (depends on timing)

        Thread.sleep(100); // let thread reach sleep()

        // ─────────────────────────────────────────────────────────
        // STATE 5: TIMED_WAITING — via sleep()
        // ─────────────────────────────────────────────────────────
        printState("During sleep(2000)", t);
        // Output: TIMED_WAITING    isAlive: true

        Thread.sleep(2100); // wait for sleep to finish

        // Thread woke from sleep, now in wait()
        Thread.sleep(100);  // let thread reach wait()

        // ─────────────────────────────────────────────────────────
        // STATE 4: WAITING — via wait()
        // ─────────────────────────────────────────────────────────
        printState("During wait()", t);
        // Output: WAITING    isAlive: true

        // Signal the waiting thread
        synchronized (waitLock) {
            condition = true;
            waitLock.notifyAll();
        }

        Thread.sleep(50); // let thread re-acquire lock and start CPU work

        // ─────────────────────────────────────────────────────────
        // STATE 2: RUNNABLE — doing CPU work
        // ─────────────────────────────────────────────────────────
        printState("During CPU work", t);
        // Output: RUNNABLE    isAlive: true

        t.join(); // wait for thread to die

        // ─────────────────────────────────────────────────────────
        // STATE 6: TERMINATED — run() returned
        // ─────────────────────────────────────────────────────────
        printState("After run() returned", t);
        // Output: TERMINATED    isAlive: false

        // ─────────────────────────────────────────────────────────
        // STATE 3: BLOCKED — demonstrated separately
        // ─────────────────────────────────────────────────────────
        demonstrateBlocked();
    }

    static void demonstrateBlocked() throws InterruptedException {
        final Object sharedLock = new Object();

        Thread holder = new Thread(() -> {
            synchronized (sharedLock) {
                try { Thread.sleep(3000); }
                catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "lock-holder");

        Thread blocked = new Thread(() -> {
            synchronized (sharedLock) { /* just needs the lock */ }
        }, "blocked-thread");

        holder.start();
        Thread.sleep(100);
        blocked.start();
        Thread.sleep(100);

        // ─────────────────────────────────────────────────────────
        // STATE 3: BLOCKED
        // ─────────────────────────────────────────────────────────
        printState("blocked-thread waiting for lock", blocked);
        // Output: BLOCKED    isAlive: true

        holder.interrupt();
        blocked.join();
    }

    // ── Helper ────────────────────────────────────────────────────
    static void printState(String label, Thread t) {
        System.out.printf("%-40s  State: %-15s  Alive: %s%n",
                label, t.getState(), t.isAlive());
    }
}
```

**Expected Output:**
```
After new Thread()                        State: NEW              Alive: false
Right after start()                       State: RUNNABLE         Alive: true
During sleep(2000)                        State: TIMED_WAITING    Alive: true
During wait()                             State: WAITING          Alive: true
During CPU work                           State: RUNNABLE         Alive: true
After run() returned                      State: TERMINATED       Alive: false
blocked-thread waiting for lock           State: BLOCKED          Alive: true
```

---

## The Interruption Model — Critical for Understanding State Transitions

```java
// ═══════════════════════════════════════════════════════════════
//  INTERRUPTION — how it interacts with each state
// ═══════════════════════════════════════════════════════════════
public class InterruptionAndStates {

    public static void main(String[] args) throws InterruptedException {

        // ── interrupt() when thread is RUNNABLE ───────────────────
        Thread runnable = new Thread(() -> {
            System.out.println("Runnable: started");
            // Busy loop — stays RUNNABLE
            while (!Thread.currentThread().isInterrupted()) {
                // No blocking calls — interrupt just sets the FLAG
                // Thread must CHECK the flag manually
                // If thread never checks → interrupt has NO effect!
            }
            System.out.println("Runnable: saw interrupt flag, stopping");
            // Thread decides what to do with the interrupt
        }, "runnable-thread");

        runnable.start();
        Thread.sleep(100);
        runnable.interrupt();
        // Sets interrupt flag → thread exits while loop naturally
        runnable.join();
        System.out.println("runnable: " + runnable.getState()); // TERMINATED

        // ── interrupt() when thread is TIMED_WAITING (sleeping) ───
        Thread sleeper = new Thread(() -> {
            try {
                Thread.sleep(10_000); // sleeping for 10 seconds
            } catch (InterruptedException e) {
                // interrupt() while sleeping:
                //   1. Thread is woken from TIMED_WAITING → RUNNABLE
                //   2. Interrupt FLAG is CLEARED
                //   3. InterruptedException is thrown HERE
                System.out.println("Sleeper: interrupted while sleeping");
                Thread.currentThread().interrupt(); // restore the flag
            }
        }, "sleeper-thread");

        sleeper.start();
        Thread.sleep(100);
        sleeper.interrupt(); // wake it early
        sleeper.join();
        System.out.println("sleeper: " + sleeper.getState()); // TERMINATED

        // ── interrupt() when thread is WAITING ────────────────────
        final Object lock = new Object();

        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait(); // WAITING indefinitely
                } catch (InterruptedException e) {
                    // interrupt() while waiting:
                    //   1. Thread moved from Wait Set → Entry Set
                    //   2. Thread acquires lock → RUNNABLE
                    //   3. Interrupt FLAG is CLEARED
                    //   4. InterruptedException thrown HERE
                    System.out.println("Waiter: interrupted while waiting");
                    Thread.currentThread().interrupt(); // restore flag
                }
            }
        }, "waiter-thread");

        waiter.start();
        Thread.sleep(100);
        waiter.interrupt();
        waiter.join();
        System.out.println("waiter: " + waiter.getState()); // TERMINATED

        // ── interrupt() when thread is BLOCKED ────────────────────
        // IMPORTANT: interrupt() does NOT unblock a BLOCKED thread!
        // It sets the flag. When thread EVENTUALLY acquires the lock
        // and calls a blocking method → THEN InterruptedException

        final Object blockLock = new Object();

        Thread blocker = new Thread(() -> {
            synchronized (blockLock) { // will BLOCK here
                // Gets here only AFTER holder releases lock
                // At which point interrupt flag is checked
                // by subsequent blocking calls
                System.out.println("Blocker: finally got lock");
                try {
                    Thread.sleep(1); // will throw InterruptedException
                    // because interrupt flag was set while we were BLOCKED
                } catch (InterruptedException e) {
                    System.out.println("Blocker: interrupt flag was set!");
                }
            }
        }, "blocker-thread");

        Thread holder = new Thread(() -> {
            synchronized (blockLock) {
                try { Thread.sleep(2000); }
                catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "holder-thread");

        holder.start();
        Thread.sleep(100);
        blocker.start();   // will BLOCK immediately
        Thread.sleep(100);
        blocker.interrupt(); // sets flag, but blocker is BLOCKED — can't exit!
        System.out.println("blocker: " + blocker.getState()); // still BLOCKED!
        holder.join();     // holder releases lock
        blocker.join();    // blocker eventually runs and sees interrupt flag
    }
}
```

```
INTERRUPT BEHAVIOR BY STATE:

  ╔═══════════════════╦══════════════════════════════════════════════════╗
  ║ Thread State      ║ What happens when interrupt() is called          ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ NEW               ║ Flag set, no immediate effect (thread not alive) ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ RUNNABLE          ║ Flag SET only. Thread must check                 ║
  ║                   ║ isInterrupted() manually. Nothing automatic.     ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ BLOCKED           ║ Flag SET only. Thread STAYS BLOCKED.             ║
  ║                   ║ Will see flag when it eventually acquires lock   ║
  ║                   ║ and makes a blocking call.                       ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ WAITING           ║ Thread WOKEN immediately.                        ║
  ║ (via wait())      ║ Moved to Entry Set. Must re-acquire lock.        ║
  ║                   ║ Flag CLEARED. InterruptedException thrown.       ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ TIMED_WAITING     ║ Thread WOKEN immediately.                        ║
  ║ (via sleep/wait)  ║ Flag CLEARED. InterruptedException thrown.       ║
  ╠═══════════════════╬══════════════════════════════════════════════════╣
  ║ TERMINATED        ║ No effect.                                       ║
  ╚═══════════════════╩══════════════════════════════════════════════════╝

  KEY RULE: InterruptedException CLEARS the interrupt flag!
  Best practice: always restore the flag in catch:
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // restore flag
    }
```

---

## Thread State in Real Applications

```java
// ═══════════════════════════════════════════════════════════════
//  REAL APPLICATION: monitoring thread states
//  Useful for debugging, health checks, metrics
// ═══════════════════════════════════════════════════════════════
public class ThreadStateMonitor {

    public static void main(String[] args) throws InterruptedException {

        ExecutorService pool = Executors.newFixedThreadPool(5);

        // Submit a mix of different tasks
        pool.submit(() -> {  // CPU bound
            long s = 0;
            for (long i = 0; i < Long.MAX_VALUE; i++) s += i;
        });
        pool.submit(() -> {  // sleeping
            try { Thread.sleep(Long.MAX_VALUE); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        pool.submit(() -> {  // waiting for lock
            synchronized (pool) {
                try { pool.wait(); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        });

        Thread.sleep(500); // let tasks get into their states

        // Get ALL threads in JVM
        Map<Thread.State, List<String>> stateMap = new EnumMap<>(Thread.State.class);
        for (Thread.State s : Thread.State.values()) {
            stateMap.put(s, new ArrayList<>());
        }

        // Enumerate all threads
        ThreadGroup root = Thread.currentThread().getThreadGroup();
        while (root.getParent() != null) root = root.getParent();
        Thread[] allThreads = new Thread[root.activeCount() + 10];
        root.enumerate(allThreads, true);

        for (Thread t : allThreads) {
            if (t != null) {
                stateMap.get(t.getState()).add(t.getName());
            }
        }

        // Print summary
        System.out.println("=== THREAD STATE SUMMARY ===");
        for (Thread.State state : Thread.State.values()) {
            List<String> threads = stateMap.get(state);
            if (!threads.isEmpty()) {
                System.out.printf("%-15s (%d): %s%n",
                        state, threads.size(), threads);
            }
        }

        pool.shutdownNow();
    }
}
```

**Sample Output:**
```
=== THREAD STATE SUMMARY ===
RUNNABLE        (4): [main, pool-1-thread-1, GC Thread#0, C2 CompilerThread0]
TIMED_WAITING   (3): [pool-1-thread-2, Finalizer, Reference Handler]
WAITING         (2): [pool-1-thread-3, Signal Dispatcher]
```

---

## The sleep() vs wait() vs join() State Comparison

```
WHICH STATE DOES EACH METHOD CAUSE?

  Thread.sleep(n)   → TIMED_WAITING
  Thread.sleep()    → no zero-arg version

  object.wait()     → WAITING
  object.wait(n)    → TIMED_WAITING

  t.join()          → WAITING       (caller thread waits for t to die)
  t.join(n)         → TIMED_WAITING (caller waits max n ms)

  LockSupport.park()          → WAITING
  LockSupport.parkNanos(n)    → TIMED_WAITING
  LockSupport.parkUntil(dl)   → TIMED_WAITING

  synchronized(lock) { }      → BLOCKED (if lock is taken)

  ── DOES THE METHOD RELEASE LOCKS? ───────────────────────────────

  sleep()           → NO  — holds all locks while sleeping
  wait()            → YES — releases the monitor lock
  join()            → N/A — no lock involved (unless called in synchronized)
  LockSupport.park()→ NO  — does not release locks

  ── IS THERE A TIMEOUT VARIANT? ──────────────────────────────────

  sleep():   YES — sleep(n), sleep(n, nanos)
  wait():    YES — wait(n), wait(n, nanos)
  join():    YES — join(n), join(n, nanos)
  park():    YES — parkNanos(n), parkUntil(deadline)
```

---

## Common Mistakes Related to Thread States

```java
// ═══════════════════════════════════════════════════════════════
//  COMMON MISTAKES ABOUT THREAD STATES
// ═══════════════════════════════════════════════════════════════
public class ThreadStateMistakes {

    static final Object lock = new Object();

    // ── MISTAKE 1: Confusing BLOCKED and WAITING ──────────────────
    // BLOCKED  = waiting for a LOCK (synchronized)
    // WAITING  = waiting for a SIGNAL (wait/notify)
    // They look similar but are completely different mechanisms.
    // Interrupt does NOT unblock a BLOCKED thread.
    // Interrupt DOES unblock a WAITING thread (via InterruptedException).

    // ── MISTAKE 2: Thinking sleep() releases the lock ─────────────
    static void sleepMistake() throws InterruptedException {
        Thread t = new Thread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("Sleeping while holding lock");
                    Thread.sleep(5000); // DOES NOT release lock!
                    // Other threads trying synchronized(lock) will BLOCK
                    // for the entire 5 seconds!
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        t.start();
        // If you want to release lock while waiting → use wait(), NOT sleep()
    }

    // ── MISTAKE 3: Thinking RUNNABLE means actively using CPU ─────
    // RUNNABLE includes threads doing I/O, waiting for network,
    // waiting for disk — they appear RUNNABLE but aren't on CPU.
    // Only way to know if truly on CPU: profiler, CPU usage metrics.

    // ── MISTAKE 4: Calling getState() and acting on result ─────────
    static void stateMistake(Thread t) {
        // WRONG: checking state and acting on it
        if (t.getState() == Thread.State.WAITING) {
            // By the time you act on this — state may have changed!
            // This is a RACE CONDITION on the state check itself.
            // State can change between the check and your action.
        }

        // RIGHT: use proper synchronization mechanisms
        // Don't base logic on thread state — use locks, conditions, futures.
        // getState() is for MONITORING/DEBUGGING only, not control flow.
    }

    // ── MISTAKE 5: Expecting TERMINATED state immediately after run()─
    static void terminatedMistake() throws InterruptedException {
        Thread t = new Thread(() -> {
            System.out.println("Task done");
        });
        t.start();
        // t.getState() here might be RUNNABLE (not yet run)
        // or TERMINATED (already ran and finished)
        // Depends on scheduling — non-deterministic!

        t.join(); // use join() to GUARANTEE thread has terminated
        System.out.println(t.getState()); // NOW guaranteed TERMINATED
    }
}
```

---

## Visualizing States in a Thread Dump (jstack)

```
REAL THREAD DUMP OUTPUT — jstack <pid>
Shows the state of every thread in the JVM.
This is what you read during production debugging.

"main" #1 prio=5 os_prio=0 tid=0x... nid=0x1234 waiting on condition
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at com.example.MyService.processRequest(MyService.java:45)
        at com.example.Main.main(Main.java:12)
   → Thread is in TIMED_WAITING due to Thread.sleep()

"worker-1" #12 prio=5 os_prio=0 tid=0x... nid=0x5678 waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.OrderService.processOrder(OrderService.java:23)
        - waiting to lock <0x000000076b3e1f60> (a java.lang.Object)
        at com.example.Worker.run(Worker.java:15)
   → Thread is BLOCKED trying to acquire a synchronized lock
   → The lock address is shown: 0x000000076b3e1f60

"worker-2" #13 prio=5 os_prio=0 tid=0x... nid=0x9abc in Object.wait()
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x000000076b3e2000> (a java.util.LinkedList)
        at java.lang.Object.wait(Object.java:502)
        at com.example.ProducerConsumer.consume(ProducerConsumer.java:34)
   → Thread is WAITING indefinitely in wait()
   → The object it's waiting on is shown

DETECTING DEADLOCK IN THREAD DUMP:
══════════════════════════════════

  "Thread-1" waiting to lock <0xAAA> held by "Thread-2"
  "Thread-2" waiting to lock <0xBBB> held by "Thread-1"

  Found 1 deadlock:
    Thread-1 ── wants lock 0xAAA ──► held by Thread-2
    Thread-2 ── wants lock 0xBBB ──► held by Thread-1

  jstack explicitly prints "Found N deadlock(s)" section at the bottom.
```

---

## Interview Questions on Thread States

```
Q1: How many states does a Java thread have? Name them.
────────────────────────────────────────────────────────
A:  6 states defined in Thread.State enum:
    NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED

────────────────────────────────────────────────────────
Q2: What is the difference between BLOCKED and WAITING?
────────────────────────────────────────────────────────
A:  BLOCKED:
      Caused by: trying to enter a synchronized block/method
                 when another thread holds the lock.
      Woken by:  the lock being released (automatic, no code needed).
      Uses:      synchronized keyword only.

    WAITING:
      Caused by: object.wait(), thread.join(), LockSupport.park()
      Woken by:  explicit notify()/notifyAll(), unpark(), or interrupt().
      Without notification: waits forever.

    Key difference:
      BLOCKED = waiting for a LOCK (passive, automatic)
      WAITING = waiting for a SIGNAL (must be explicitly woken)
      Interrupt() wakes WAITING threads but does NOT unblock BLOCKED threads.

────────────────────────────────────────────────────────
Q3: What is the difference between WAITING and TIMED_WAITING?
────────────────────────────────────────────────────────
A:  WAITING:       waits INDEFINITELY — only woken by explicit signal.
    TIMED_WAITING: waits for a MAXIMUM DURATION — wakes automatically
                   when timeout expires, OR when explicitly signaled.

    TIMED_WAITING causes:
      Thread.sleep(n), object.wait(n), thread.join(n),
      LockSupport.parkNanos(n), LockSupport.parkUntil(deadline)

────────────────────────────────────────────────────────
Q4: Does sleep() release the lock?
────────────────────────────────────────────────────────
A:  NO. Thread.sleep() does NOT release any locks.
    Thread holds all its locks while sleeping.
    Other threads trying to acquire those locks will BLOCK
    for the entire sleep duration.
    If you want to release the lock while waiting: use wait().

────────────────────────────────────────────────────────
Q5: What happens to a thread's state when interrupt() is called?
────────────────────────────────────────────────────────
A:  Depends on current state:
    RUNNABLE:  interrupt FLAG is set. State stays RUNNABLE.
               Thread must check isInterrupted() manually.
    BLOCKED:   interrupt FLAG is set. State STAYS BLOCKED.
               Thread remains blocked until it acquires the lock.
    WAITING:   Thread is WOKEN. Moves to BLOCKED (to re-acquire lock).
               InterruptedException thrown. Flag CLEARED.
    TIMED_WAITING: Thread is WOKEN. Moves to RUNNABLE.
               InterruptedException thrown. Flag CLEARED.
    TERMINATED: No effect.

    CRITICAL: InterruptedException CLEARS the interrupt flag.
    Best practice: call Thread.currentThread().interrupt() in catch.

────────────────────────────────────────────────────────
Q6: Can a TERMINATED thread be restarted?
────────────────────────────────────────────────────────
A:  NO. TERMINATED is the final state.
    Calling start() on a TERMINATED thread:
      → IllegalThreadStateException
    Must create a new Thread object if you need to run the task again.
    This is one reason to prefer Runnable/Callable over extending Thread —
    the same task can be submitted multiple times without this issue.

────────────────────────────────────────────────────────
Q7: What is the state of a thread during I/O operations?
────────────────────────────────────────────────────────
A:  RUNNABLE — even though it appears "blocked" on I/O.
    JVM reports RUNNABLE for all cases where the thread is not
    waiting on a Java-level lock/condition.
    I/O blocking at the OS level is transparent to JVM.
    The OS may internally put the thread on an I/O wait queue,
    but JVM's Thread.State shows RUNNABLE.

────────────────────────────────────────────────────────
Q8: How does join() cause WAITING state?
────────────────────────────────────────────────────────
A:  When thread A calls thread B's join():
    A enters WAITING state (indefinite join) or TIMED_WAITING (join(n)).
    Internally: join() calls wait() on the Thread object of B.
    When B terminates, JVM calls notifyAll() on B's Thread object.
    This wakes thread A from WAITING → RUNNABLE.
    join() is literally: while (isAlive()) { wait(); }
    implemented in the Thread class itself.
```

---

## The Complete Summary

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    6 STATES — COMPLETE SUMMARY                           ║
╠══════════════╦═══════════════╦════════════════╦══════════════════════════╣
║  STATE       ║ isAlive()     ║ CPU USAGE      ║ EXITS VIA                ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ NEW          ║ false         ║ 0%             ║ start()                  ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ RUNNABLE     ║ true          ║ 0-100%         ║ blocking op /            ║
║              ║               ║ (on or off CPU)║ lock attempt / run() end ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ BLOCKED      ║ true          ║ 0%             ║ lock released            ║
║              ║               ║                ║ (automatic)              ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ WAITING      ║ true          ║ 0%             ║ notify / notifyAll /     ║
║              ║               ║                ║ unpark / interrupt /     ║
║              ║               ║                ║ target dies (join)       ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ TIMED_WAITING║ true          ║ 0%             ║ timeout / notify /       ║
║              ║               ║                ║ interrupt / target dies  ║
╠══════════════╬═══════════════╬════════════════╬══════════════════════════╣
║ TERMINATED   ║ false         ║ 0%             ║ FINAL — no exit          ║
╚══════════════╩═══════════════╩════════════════╩══════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════╗
║             WHICH METHOD → WHICH STATE                                   ║
╠═══════════════════════════════╦══════════════════════════════════════════║
║  Method called                ║  State entered                           ║
╠═══════════════════════════════╬══════════════════════════════════════════║
║  t.start()                    ║  RUNNABLE                                ║
║  synchronized(lock){ }        ║  BLOCKED (if lock taken)                 ║
║  object.wait()                ║  WAITING                                 ║
║  t.join()                     ║  WAITING                                 ║
║  LockSupport.park()           ║  WAITING                                 ║
║  Thread.sleep(n)              ║  TIMED_WAITING                           ║
║  object.wait(n)               ║  TIMED_WAITING                           ║
║  t.join(n)                    ║  TIMED_WAITING                           ║
║  LockSupport.parkNanos(n)     ║  TIMED_WAITING                           ║
║  run() returns / throws       ║  TERMINATED                              ║
╚═══════════════════════════════╩══════════════════════════════════════════╝

THE ONE RULE TO REMEMBER ALL STATES:

  NEW         → "I exist but haven't started"
  RUNNABLE    → "I am running or ready to run"
  BLOCKED     → "I need a lock someone else has"
  WAITING     → "I'm waiting to be called — indefinitely"
  TIMED_WAITING→"I'm waiting to be called — but max N ms"
  TERMINATED  → "I'm done — forever"
```