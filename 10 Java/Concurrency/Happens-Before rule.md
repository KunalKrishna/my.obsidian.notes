# Happens-Before Rule — Complete Tutorial
---
## The Story Before the Definition

```cs
Imagine a relay race.

Runner A runs the first leg and passes the baton to Runner B.
Runner B runs the second leg and passes to Runner C.

There is ONE absolute rule in relay racing:
  Runner B CANNOT start running until Runner A hands over the baton.
  Runner C CANNOT start running until Runner B hands over the baton.

When Runner A hands the baton to B:
  Everything A did (her entire run, all her effort)
  is COMPLETE and KNOWN to B before B takes a single step.
  B does not need to ask "did A finish her leg?"
  The BATON HANDOFF is the guarantee.

This is Happens-Before.

The baton handoff = the synchronization action
Runner A's work   = writes by Thread A
Runner B seeing it = Thread B's guaranteed visibility

If there is NO baton handoff (runners run simultaneously
without coordination) — chaos. B might start before A finishes.
Nobody knows who did what in what order.
That is the world WITHOUT happens-before.
```

---
## Why Happens-Before Was Invented

```css
Before Java 5 (JSR-133), Java had NO formal memory model.

The language said:
  "synchronized and volatile provide some guarantees"
  but never precisely defined WHAT those guarantees were.

This led to:
  - JVM implementations doing different things
  - Code that worked on one JVM crashing on another
  - Code that worked on single-core failing on multi-core
  - Impossible-to-debug race conditions

Java 5 (JSR-133) introduced the formal Java Memory Model (JMM).
The core concept: HAPPENS-BEFORE.

JMM's promise:
  "We will not tell you HOW the hardware implements things.
   We will tell you EXACTLY what guarantees you have.
   If you establish a happens-before relationship,
   your writes WILL be visible.
   If you don't — no guarantee whatsoever."
```

---
## The Formal Definition — Made Simple

```javascript
FORMAL DEFINITION (JLS Chapter 17):

  If action A  happens-before  action B:
    1. A is ORDERED before B
       (A appears to execute before B)
    2. A's results are VISIBLE to B
       (everything A wrote, B is guaranteed to see)

  Written as: A hb B

WHAT IT IS NOT:

  hb is NOT about wall-clock time.
  hb does NOT mean A physically executes before B on the CPU.
  hb is about GUARANTEED VISIBILITY and ORDERING.

  Two actions can happen at the exact same nanosecond
  on two different CPU cores — and still have a
  happens-before relationship if a synchronization
  action connects them.

THE SIMPLEST MENTAL MODEL:

  A happens-before B means:
  "B is GUARANTEED to see everything A did."

  No happens-before means:
  "B may or may not see what A did — undefined."
```
---
## The Big Picture — All 8 Happens-Before Rules

```cs
╔══════════════════════════════════════════════════════════════════════════╗
║              ALL HAPPENS-BEFORE RULES IN JAVA                            ║
╠══════╦═══════════════════════════════════════════════════════════════════╣
║  #   ║  RULE                                                             ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  1   ║  PROGRAM ORDER RULE                                               ║
║      ║  Within a thread, each action hb every subsequent action          ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  2   ║  MONITOR LOCK RULE                                                ║
║      ║  Unlock of monitor M  hb  every subsequent lock of M              ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  3   ║  VOLATILE VARIABLE RULE                                           ║
║      ║  Write to volatile V  hb  every subsequent read of V              ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  4   ║  THREAD START RULE                                                ║
║      ║  Thread.start()  hb  every action in the started thread           ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  5   ║  THREAD TERMINATION RULE                                          ║
║      ║  Every action in thread T  hb  T.join() returns                   ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  6   ║  THREAD INTERRUPTION RULE                                         ║
║      ║  Thread.interrupt()  hb  interrupted thread detects interrupt     ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  7   ║  FINALIZER RULE                                                   ║
║      ║  End of constructor  hb  start of finalizer                       ║
╠══════╬═══════════════════════════════════════════════════════════════════╣
║  8   ║  TRANSITIVITY RULE                                                ║
║      ║  If A hb B  AND  B hb C  →  A hb C                                ║
╚══════╩═══════════════════════════════════════════════════════════════════╝

These 8 rules are the COMPLETE set.
Every synchronization guarantee in Java
derives from one or more of these rules.
```
---
## Rule 1 — Program Order Rule

```
RULE: Within a single thread, each action happens-before
      every subsequent action in program order.

This is the rule that makes single-threaded code WORK.
Without it, even a single-threaded program would be unpredictable.
```

```java
public class ProgramOrderRule {

    public static void main(String[] args) {

        // All of this runs in the main thread.
        // Program Order Rule guarantees:
        // Line N  hb  Line N+1  hb  Line N+2  ...

        int x = 5;         // action 1  hb
        int y = x + 3;     // action 2  ← guaranteed to see x=5
                           //             hb
        int z = y * 2;     // action 3  ← guaranteed to see y=8
                           //             hb
        System.out.println(z); // action 4 ← guaranteed to see z=16

        // Output: 16  (always, guaranteed, never wrong)
        // Because: action 1 hb action 2 hb action 3 hb action 4
    }
}
```

```
CRITICAL SUBTLETY — Program Order Rule does NOT prevent reordering:

  Within ONE thread, the JVM CAN reorder instructions
  as long as the APPEARANCE of program order is maintained
  to THAT THREAD.

  int a = 1;    ┐  These two are independent.
  int b = 2;    ┘  JVM can execute b=2 before a=1.
                   But to THIS THREAD it appears sequential.

  The reordering is invisible within one thread.
  It only becomes visible (and dangerous) to OTHER threads.

  Program Order Rule guarantees:
  "Within your thread, things appear to happen in order."
  It does NOT guarantee that other threads see the same order.
  That is what the other rules are for.
```
---
## Rule 2 — Monitor Lock Rule (synchronized)

```
RULE: An unlock on monitor M happens-before every
      subsequent lock on that SAME monitor M.

This is the most important rule for multi-threaded code.
It is the guarantee behind the synchronized keyword.
```

```java
public class MonitorLockRule {

    private static int     counter = 0;
    private static String  message = "initial";
    private static boolean ready   = false;

    private static final Object lock = new Object(); // the monitor

    public static void main(String[] args) throws InterruptedException {

        Thread writer = new Thread(() -> {
            synchronized (lock) {
                // Writer thread acquires lock on 'lock' object
                // Does work inside synchronized block
                counter = 42;
                message = "updated";
                ready   = true;
                System.out.println("Writer: wrote counter=42, message=updated, ready=true");
                // Writer releases lock on 'lock' object
                // ← UNLOCK happens here (exiting synchronized block)
                // UNLOCK of 'lock' hb LOCK of 'lock' (by any subsequent locker)
            }
        });

        Thread reader = new Thread(() -> {
            synchronized (lock) {
                // Reader thread acquires lock on 'lock' object
                // ← LOCK happens here
                // Because: UNLOCK(writer) hb LOCK(reader)
                // Everything writer did BEFORE unlock is visible here

                System.out.println("Reader: counter = " + counter); // 42 GUARANTEED
                System.out.println("Reader: message = " + message); // "updated" GUARANTEED
                System.out.println("Reader: ready   = " + ready);   // true GUARANTEED
            }
        });

        writer.start();
        writer.join(); // ensure writer finishes first (so unlock happens before lock)
        reader.start();
        reader.join();
    }
}
```

```
MONITOR LOCK RULE — VISUALIZED:

  Thread Writer                          Thread Reader
  ─────────────────────────────          ──────────────────────────
  synchronized(lock) {                   synchronized(lock) {
    counter = 42;          ┐               // LOCK acquired HERE
    message = "updated";   │ hb            counter → 42      ✓
    ready   = true;        │ ──────────►   message → "updated"✓
  } ← UNLOCK HERE          ┘               ready   → true    ✓
                                         }

  The UNLOCK acts as a "memory flush":
    All writes inside the block are flushed to main memory.

  The LOCK acts as a "memory refresh":
    All reads inside the block read fresh from main memory.

  SAME MONITOR requirement:
    Writer uses synchronized(lock)
    Reader uses synchronized(lock)
    SAME object 'lock' = SAME monitor = hb established ✓

    Writer uses synchronized(lockA)
    Reader uses synchronized(lockB)  ← DIFFERENT monitor
    NO hb established → NO visibility guarantee ✗


THE MEMORY FLUSH/REFRESH PICTURE:

  Writer:                   Main Memory              Reader:
  ────────                  ───────────              ────────
  acquire lock              counter = 0              acquire lock
    counter=42                                         ← flushes main
    message="updated"                                    memory to
    ready=true                                           working memory
  release lock ────FLUSH───► counter = 42
                             message = "updated"     read counter → 42  ✓
                             ready   = true ◄─REFRESH─ read message     ✓
                                                     read ready         ✓
                                                     release lock
```
### The Same Monitor Requirement — A Common Mistake

```java
public class SameMonitorMistake {

    private static int value = 0;

    private static final Object lockA = new Object();
    private static final Object lockB = new Object(); // DIFFERENT object!

    public static void main(String[] args) throws InterruptedException {

        Thread writer = new Thread(() -> {
            synchronized (lockA) {  // ← locks on lockA
                value = 42;
            } // ← UNLOCKS lockA
        });

        Thread reader = new Thread(() -> {
            synchronized (lockB) {  // ← locks on lockB — DIFFERENT monitor!
                // NO happens-before between lockA unlock and lockB lock
                // value may still be 0 here!
                System.out.println("value = " + value); // NOT GUARANTEED to be 42
            }
        });

        writer.start(); writer.join();
        reader.start(); reader.join();
    }
}
```

```
WHY DIFFERENT MONITORS BREAK THE GUARANTEE:

  Monitor Lock Rule states:
  "Unlock of M hb subsequent lock of M"
            ↑                       ↑
         SAME M                  SAME M

  lockA.unlock  hb  lockA.lock  ← YES: same monitor ✓
  lockA.unlock  hb  lockB.lock  ← NO:  different monitors ✗

  Think of it like this:
    Two separate rooms, each with their own key.
    Leaving Room A (unlocking A) tells nothing to someone
    trying to enter Room B (locking B).
    They are completely independent rooms.
```
---
## Rule 3 — Volatile Variable Rule

```
RULE: A write to a volatile variable V happens-before
      every subsequent read of that SAME volatile variable V.

AND (crucially):
  ALL writes before the volatile write are also visible
  to the thread that reads the volatile variable.
```

```java
public class VolatileVariableRule {

    private static int     value      = 0;    // NOT volatile
    private static String  message    = null; // NOT volatile
    private static volatile boolean ready = false; // ONLY this is volatile

    public static void main(String[] args) throws InterruptedException {

        Thread writer = new Thread(() -> {
            value   = 42;        // ordinary write (1)
            message = "hello";   // ordinary write (2)
            ready   = true;      // VOLATILE WRITE  (3)
            // The volatile write to 'ready' acts as a STORE FENCE:
            // Writes (1) and (2) are GUARANTEED to be flushed
            // to main memory BEFORE write (3).
            // Nothing can be reordered to execute AFTER (3).
        });

        Thread reader = new Thread(() -> {
            while (!ready) { }   // VOLATILE READ of 'ready'
            // The volatile read of 'ready' acts as a LOAD FENCE:
            // ALL writes that happened before the volatile write of 'ready'
            // are guaranteed visible here.

            // Volatile Variable Rule:
            // write(ready=true) hb read(ready=true)
            // AND by Program Order Rule + Transitivity:
            // write(value=42)   hb write(ready=true)  hb read(ready=true)
            // THEREFORE:
            // write(value=42)   hb read(ready=true)

            System.out.println("value   = " + value);   // GUARANTEED: 42
            System.out.println("message = " + message); // GUARANTEED: "hello"
        });

        reader.start();
        Thread.sleep(100);
        writer.start();

        writer.join(); reader.join();
    }
}
```

```
VOLATILE RULE + TRANSITIVITY — THE CHAIN:

  Thread Writer:
    value   = 42;          action W1
    message = "hello";     action W2
    ready   = true;        action W3  ← volatile write

  Thread Reader:
    while (!ready) { }     action R1  ← volatile read
    print(value);          action R2
    print(message);        action R3

  Happens-Before chain:
    W1 hb W2               (Program Order Rule)
    W2 hb W3               (Program Order Rule)
    W3 hb R1               (Volatile Variable Rule — same variable 'ready')
    R1 hb R2               (Program Order Rule)
    R1 hb R3               (Program Order Rule)

  By Transitivity:
    W1 hb R2               ← value=42  visible when printing value
    W2 hb R3               ← message="hello" visible when printing message

  RESULT:
    print(value)   → 42      ✓ guaranteed
    print(message) → "hello" ✓ guaranteed


VOLATILE FENCE PICTURE:
══════════════════════════════════════════════════════════════════

  Thread Writer (Core 1)              Thread Reader (Core 2)
  ──────────────────────              ──────────────────────
  value   = 42           ┐
  message = "hello"      │ all writes
                         │ above the
  ready   = true;  ──────┘ volatile write
  ← STORE FENCE             are flushed
    (all preceding            to main
     writes flushed           memory
     to main memory)
                              LOAD FENCE ──► while (!ready) {}
                              (all subsequent    ← volatile read
                               reads from        forces fresh read
                               main memory)      from main memory
                                             value → 42    ✓
                                             message → "hello" ✓
```
---
## Rule 4 — Thread Start Rule

```
RULE: Thread.start() happens-before every action
      in the newly started thread.

Everything the parent thread does BEFORE calling start()
is guaranteed to be visible to the child thread.
```

```java
public class ThreadStartRule {

    private static int    config    = 0;
    private static String serverUrl = null;

    public static void main(String[] args) throws InterruptedException {

        // Setup BEFORE start()
        config    = 8080;               // action 1
        serverUrl = "http://localhost"; // action 2

        // These writes are BEFORE start()
        // Thread Start Rule: start() hb any action in thread
        // Therefore: action 1 hb thread body, action 2 hb thread body
        Thread worker = new Thread(() -> {
            // Thread Start Rule guarantees:
            // Everything written BEFORE start() is visible here

            System.out.println("config    = " + config);    // GUARANTEED: 8080
            System.out.println("serverUrl = " + serverUrl); // GUARANTEED: "http://localhost"
            // No synchronization needed — Thread Start Rule covers this
        });

        worker.start(); // ← Thread Start Rule fires here
        worker.join();
    }
}
```

```
THREAD START RULE — VISUALIZED:

  Main Thread                          Worker Thread
  ───────────                          ─────────────
  config = 8080;           ┐
  serverUrl = "http://..."; │ all these
  ... more setup ...        │ writes
  worker.start(); ──────────┘ happen-before

                            ┌─────────────────────────────────┐
                            │  Thread body starts             │
                            │  All pre-start writes visible   │
                            │  config → 8080         ✓        │
                            │  serverUrl → "http://...) ✓     │
                            └─────────────────────────────────┘

COMMON MISTAKE — writing AFTER start():

  Thread worker = new Thread(() -> {
      System.out.println(config); // may see 0! (write happened after start)
  });

  worker.start();  ← start() hb thread body (for writes BEFORE start())

  config = 8080;   ← AFTER start() — NO hb with thread body!
                     worker may or may not see this write.
```
---
## Rule 5 — Thread Termination Rule (join)

```
RULE: Every action in thread T happens-before
      the return of T.join() in any other thread.

Everything the child thread does BEFORE it terminates
is visible to the thread that called join() AFTER join() returns.
```

```java
public class ThreadTerminationRule {

    private static int    result    = 0;
    private static String status    = null;
    private static boolean success  = false;

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            // Worker does heavy computation
            result  = computeHeavyResult();  // action 1
            status  = "COMPLETED";           // action 2
            success = true;                  // action 3
            System.out.println("Worker: done");
            // Thread terminates here
        });

        worker.start();
        worker.join();
        // ← join() BLOCKS main thread until worker terminates
        // Thread Termination Rule:
        //   Every action in worker  hb  join() returns
        //   action 1, 2, 3  hb  everything after join()

        // Everything worker wrote is GUARANTEED visible here
        System.out.println("result  = " + result);  // GUARANTEED: computed value
        System.out.println("status  = " + status);  // GUARANTEED: "COMPLETED"
        System.out.println("success = " + success); // GUARANTEED: true
    }

    static int computeHeavyResult() {
        int sum = 0;
        for (int i = 0; i < 1_000_000; i++) sum += i;
        return sum;
    }
}
```

```
THREAD TERMINATION RULE — VISUALIZED:

  Worker Thread                        Main Thread
  ─────────────                        ───────────
  result  = computeHeavy()   ┐
  status  = "COMPLETED"      │ all these
  success = true             │ writes
  [thread terminates] ───────┘ happen-before

                             worker.join(); ← main blocked here
                             [join() returns when worker terminated]

                             ┌────────────────────────────────────┐
                             │  All worker writes visible here    │
                             │  result  → computed value   ✓      │
                             │  status  → "COMPLETED"      ✓      │
                             │  success → true             ✓      │
                             └────────────────────────────────────┘

READING RESULTS WITHOUT join() IS UNSAFE:

  worker.start();
  // NO join() here!
  System.out.println(result);  // may be 0 — worker may not be done
                               // no hb between worker writes and this read
  System.out.println(status);  // may be null
  System.out.println(success); // may be false
```
---
## Rule 6 — Thread Interruption Rule

```
RULE: Thread.interrupt() on a thread T happens-before
      the interrupted thread detects the interrupt.
      (via InterruptedException, isInterrupted(), or interrupted())
```

```java
public class ThreadInterruptionRule {

    private static String reason = null;

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(10_000); // long sleep
            } catch (InterruptedException e) {
                // Thread Interruption Rule:
                // interrupt() call hb this catch block
                // 'reason' written BEFORE interrupt() is visible here

                System.out.println("Worker: interrupted! Reason: " + reason);
                // GUARANTEED to see "shutdown" — hb established
                Thread.currentThread().interrupt(); // restore interrupt flag
            }
        });

        worker.start();
        Thread.sleep(100);

        reason = "shutdown";         // write BEFORE interrupt()
        worker.interrupt();          // interrupt() hb detection in worker
        // By Transitivity:
        // write(reason="shutdown") hb interrupt() hb detection
        // Therefore: write(reason) hb detection → visible in catch block

        worker.join();
    }
}
```

---
## Rule 7 — Finalizer Rule

```
RULE: The end of a constructor happens-before
      the start of the finalizer for that object.

Ensures that whatever a constructor initializes
is visible in the finalizer.
This is the least commonly discussed rule in interviews.
Know it exists but rarely needs explicit coding.
```

```java
public class FinalizerRule {

    private final int value;

    public FinalizerRule(int value) {
        this.value = value;
        // Constructor end hb finalizer start
        // 'value' is guaranteed visible in finalize()
    }

    @Override
    protected void finalize() throws Throwable {
        // Finalizer Rule: constructor end hb this method
        System.out.println("Finalizing with value: " + value);
        // value is GUARANTEED to be the constructed value
        // not 0 (uninitialized) — constructor hb finalizer
        super.finalize();
    }
}
```
---
## Rule 8 — Transitivity Rule

```
RULE: If A happens-before B,
      and B happens-before C,
      then A happens-before C.

This is the MOST POWERFUL rule.
It lets you CHAIN happens-before relationships
to build visibility guarantees across complex scenarios.
```

```java
public class TransitivityRule {

    private static int    data1  = 0;
    private static int    data2  = 0;
    private static volatile boolean phase1Done = false;
    private static volatile boolean phase2Done = false;

    public static void main(String[] args) throws InterruptedException {

        // Thread A: does phase 1 work
        Thread threadA = new Thread(() -> {
            data1       = 100;        // write 1 (W1)
            data2       = 200;        // write 2 (W2)
            phase1Done  = true;       // volatile write (W3)
        }, "Thread-A");

        // Thread B: waits for phase 1, does phase 2 work
        Thread threadB = new Thread(() -> {
            while (!phase1Done) { }   // volatile read (R1)
            // R1 sees W3 → Volatile Rule: W3 hb R1
            // Transitivity: W1 hb W2 hb W3 hb R1
            // Therefore: W1 hb R1 and W2 hb R1
            // data1 and data2 are guaranteed visible here

            int localData1 = data1;   // guaranteed 100
            int localData2 = data2;   // guaranteed 200

            // Now thread B does its own work
            data1      = localData1 + 50;  // write (W4) = 150
            data2      = localData2 + 50;  // write (W5) = 250
            phase2Done = true;              // volatile write (W6)
        }, "Thread-B");

        // Thread C: waits for phase 2, reads final results
        Thread threadC = new Thread(() -> {
            while (!phase2Done) { }   // volatile read (R2)
            // R2 sees W6 → Volatile Rule: W6 hb R2
            // Program Order: W4 hb W5 hb W6
            // Transitivity: W4 hb W6 hb R2  →  W4 hb R2
            //               W5 hb W6 hb R2  →  W5 hb R2
            //
            // But what about W1 and W2 (Thread A's writes)?
            // W1 hb R1 (via transitivity shown above)
            // R1 hb W4 (Program Order in Thread B)
            // W4 hb R2 (shown above)
            // Therefore: W1 hb R1 hb W4 hb R2
            // By Transitivity: W1 hb R2
            // Thread A's writes are ALSO visible to Thread C!

            System.out.println("data1 = " + data1); // GUARANTEED: 150
            System.out.println("data2 = " + data2); // GUARANTEED: 250
        }, "Thread-C");

        threadA.start();
        threadB.start();
        threadC.start();

        threadA.join(); threadB.join(); threadC.join();
    }
}
```

```
TRANSITIVITY — THE CHAIN VISUALIZED:

  Thread A writes:
    W1: data1 = 100      ─┐
    W2: data2 = 200       │ Program Order Rule
    W3: phase1Done=true  ◄┘

  Thread B:
    R1: read phase1Done  ◄─── Volatile Rule: W3 hb R1
    W4: data1 = 150      ─┐
    W5: data2 = 250       │ Program Order Rule
    W6: phase2Done=true  ◄┘

  Thread C:
    R2: read phase2Done  ◄─── Volatile Rule: W6 hb R2

  THE CHAIN (using transitivity):
    W1 hb W2 hb W3  (Program Order in A)
                W3 hb R1  (Volatile Rule)
                      R1 hb W4 hb W5 hb W6  (Program Order in B)
                                        W6 hb R2  (Volatile Rule)

  Full chain by transitivity:
    W1 hb W2 hb W3 hb R1 hb W4 hb W5 hb W6 hb R2

  Meaning:
    Thread C reading after R2 sees ALL of Thread A's and B's writes ✓
```

---
## Happens-Before Does NOT Mean "Runs First"

```java
public class HBNotAboutTime {

    static volatile int x = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(() -> {
            x = 1; // volatile write
        });

        Thread t2 = new Thread(() -> {
            int local = x; // volatile read
            System.out.println("x = " + local);
        });

        // Scenario 1: t1 starts before t2
        t1.start();
        Thread.sleep(100); // let t1 finish
        t2.start();
        // t1's write hb t2's read (t1 finished, then t2 started and read)
        // t2 GUARANTEED to see x=1

        t1.join(); t2.join();

        // Scenario 2: t2 starts first (reads BEFORE t1 writes)
        x = 0; // reset
        t2 = new Thread(() -> {
            int local = x; // may read 0 — t1 hasn't written yet
            System.out.println("x = " + local);
        });
        t2.start(); // t2 starts FIRST
        Thread.sleep(100); // let t2 finish reading
        t1 = new Thread(() -> { x = 1; });
        t1.start();
        t1.join(); t2.join();
    }
}
```

```
THE KEY INSIGHT:

  hb is not about which thread starts first.
  hb is about WHETHER the read can see the write.

  For write W to be visible to read R:
    W must happen-before R (in hb terms)

  W hb R requires:
    Some synchronization ACTION linking W and R.
    (volatile write/read, lock/unlock, start/join, etc.)

  If no synchronization action connects them:
    Even if W physically executed before R on the CPU clock
    → R is NOT guaranteed to see W
    → JMM permits R to see a stale value

  COUNTERINTUITIVE EXAMPLE:
    Thread A: x = 42;  (runs at time 0.000001s)
    Thread B: read x;  (runs at time 0.001s)
    B runs 1000x later than A on wall clock.
    BUT if no synchronization → NOT guaranteed to see 42.
    CPU cache may still have x=0 in B's core.
```

---
## No Happens-Before — The Data Race

```java
public class NoHappensBefore {

    static int x = 0;
    static int y = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(() -> {
            x = 1; // write to x
            y = 1; // write to y
        });

        Thread t2 = new Thread(() -> {
            int ry = y; // read y
            int rx = x; // read x
            System.out.println("y=" + ry + " x=" + rx);
        });

        t1.start();
        t2.start();
        t1.join(); t2.join();
    }
}
```

```
POSSIBLE OUTPUTS (all legal under JMM):

  "y=0 x=0"  ← t2 read both before t1 wrote either
  "y=0 x=1"  ← t2 read y before t1's write, but saw t1's x write
  "y=1 x=0"  ← CPU reordering: t1 wrote y before x (reordered)
               t2 read y=1 but x=0
  "y=1 x=1"  ← t2 read both after t1 wrote both

  "y=1 x=0" looks IMPOSSIBLE by looking at t1's code.
  t1 writes x=1 BEFORE y=1.
  How can t2 see y=1 but x=0?

  ANSWER: CPU REORDERING.
  CPU can execute y=1 BEFORE x=1 in t1
  because they are independent (no data dependency between them).
  This is legal because there is NO hb between t1 and t2.
  JMM permits this outcome.

  Without hb → JMM says the program has a DATA RACE.
  Data race → behavior is unpredictable.
  Any output is legal including seemingly impossible ones.

  THE RULE:
    If two threads access the same variable
    and at least one access is a write
    and there is NO happens-before between them
    = DATA RACE
    = UNDEFINED BEHAVIOR under JMM
    = Fix it with synchronization
```

---
## Happens-Before in Real Code Scenarios

```java
// ═══════════════════════════════════════════════════════════════
//  REAL SCENARIO 1: Thread pool task sees submitted data
// ═══════════════════════════════════════════════════════════════
public class ExecutorHappensBefore {

    public static void main(String[] args) throws Exception {

        int importantData = 42;              // set before submit
        String config     = "production";    // set before submit

        ExecutorService pool = Executors.newSingleThreadExecutor();

        // submit() establishes happens-before:
        // Everything before submit() hb task execution
        // (ExecutorService implementations guarantee this via internal synchronization)
        Future<?> future = pool.submit(() -> {
            // importantData and config are guaranteed visible here
            System.out.println("data   = " + importantData); // 42 guaranteed
            System.out.println("config = " + config);         // "production" guaranteed
        });

        future.get(); // future.get() hb everything after it
        // task completion hb future.get() returns
        pool.shutdown();
    }
}

// ═══════════════════════════════════════════════════════════════
//  REAL SCENARIO 2: Spring Boot request handling
// ═══════════════════════════════════════════════════════════════
@Service
public class RequestService {

    // Final fields — set once in constructor, never changed
    // Constructor end hb any use (Finalizer Rule generalization)
    // Safe to read from any thread without synchronization
    private final String appName;
    private final int    maxRetries;

    // Static final — set at class initialization
    // Class init hb first use (static init rule)
    private static final int TIMEOUT = 5000;

    public RequestService(String appName, int maxRetries) {
        this.appName    = appName;    // ← set in constructor
        this.maxRetries = maxRetries; // ← set in constructor
    }

    // Called by any Tomcat thread — safe because fields are final
    public String process(String input) {
        return appName + ": " + input + " (max " + maxRetries + " retries)";
        // appName and maxRetries are FINAL — immutable after construction
        // No synchronization needed — safe from any thread
    }
}
```
---
## The Complete Happens-Before Chain — A Full Example

```java
// ═══════════════════════════════════════════════════════════════
//  PUTTING IT ALL TOGETHER
//  Build a happens-before chain using multiple rules
// ═══════════════════════════════════════════════════════════════
public class CompleteHBChain {

    static int         setup    = 0;
    static int         result   = 0;
    static volatile boolean done = false;
    static final Object lock     = new Object();

    public static void main(String[] args) throws InterruptedException {

        // ── PHASE 1: Setup (Thread Start Rule) ────────────────────
        setup = 100; // W1: written before worker.start()
                     // Thread Start Rule: W1 hb anything in worker

        Thread worker = new Thread(() -> {

            // ── PHASE 2: Computation (Program Order Rule) ──────────
            int local = setup;       // R1: sees 100 (Thread Start Rule)
            result    = local * 2;   // W2: result = 200
            done      = true;        // W3: volatile write
            // Program Order: R1 hb W2 hb W3

        });

        worker.start(); // Thread Start Rule: setup=100 hb worker body
        worker.join();  // Thread Termination Rule: worker body hb here

        // ── PHASE 3: Reading (Thread Termination Rule) ─────────────
        // All worker writes hb join() returns
        // Transitivity: W2 hb W3 hb join() returns hb read result
        System.out.println("result = " + result); // GUARANTEED: 200
        System.out.println("done   = " + done);   // GUARANTEED: true

        // ── PHASE 4: Lock-based visibility ─────────────────────────
        result = 0; // reset
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                result = 999; // W4
            } // UNLOCK lock: W4 hb subsequent lock of 'lock'
        });

        Thread t2 = new Thread(() -> {
            synchronized (lock) { // LOCK lock
                // Monitor Lock Rule: t1's unlock hb this lock
                // Transitivity: W4 hb unlock hb this lock hb read result
                System.out.println("result = " + result); // GUARANTEED: 999
            }
        });

        t1.start(); t1.join(); // ensure t1 unlocks before t2 locks
        t2.start(); t2.join();
    }
}
```
---
## The Mental Model — Happens-Before as a Contract
The 8 Happens-Before Rule : 

```c
HAPPENS-BEFORE IS A CONTRACT BETWEEN YOU AND THE JVM:

  YOU promise:
    "I will establish a happens-before relationship
     between my writes and the reads of those values."

  JVM promises:
    "If you do that, I GUARANTEE your writes are visible.
     I will insert the necessary memory barriers.
     I will prevent illegal reorderings.
     Your program will behave as you intended."

  YOU break the contract (no hb):
    JVM promises nothing.
    Writes may be invisible.
    Instructions may be reordered.
    Results are undefined.
    Bugs may appear only in production, only under load,
    only on specific hardware — the worst kind of bugs.


HAPPENS-BEFORE — THE COMPLETE PICTURE:

╔═════════════════════════════════════════════════════════════════════╗
║  RULE               WHAT ESTABLISHES IT   WHAT IT GUARANTEES        ║
╠═════════════════════════════════════════════════════════════════════╣
║  Program Order      Sequential execution  Within-thread ordering    ║
║                     in same thread        always maintained         ║
╠═════════════════════════════════════════════════════════════════════╣
║  Monitor Lock       synchronized          Everything in sync block  ║
║                     unlock → lock         visible to next locker    ║
╠═════════════════════════════════════════════════════════════════════╣
║  Volatile           volatile write        Write + all prior writes  ║
║                     → volatile read       visible to reader         ║
╠═════════════════════════════════════════════════════════════════════╣
║  Thread Start       start()               Pre-start writes visible  ║
║                     → thread body         inside the thread         ║
╠═════════════════════════════════════════════════════════════════════╣
║  Thread Termination thread body           All thread writes visible ║
║                     → join() returns      after join() returns      ║
╠═════════════════════════════════════════════════════════════════════╣
║  Interruption       interrupt()           Pre-interrupt writes      ║
║                     → detection           visible on detection      ║
╠═════════════════════════════════════════════════════════════════════╣
║  Finalizer          constructor end       Constructor writes        ║
║                     → finalizer start     visible in finalizer      ║
╠═════════════════════════════════════════════════════════════════════╣
║  Transitivity       A hb B  AND  B hb C   A's writes visible to C   ║
║                     → A hb C              builds chains             ║
╚═════════════════════════════════════════════════════════════════════╝


THE GOLDEN QUESTION FOR EVERY SHARED VARIABLE:

  "Is there a happens-before chain connecting
   the WRITE to the READ of this variable?"

  YES → write is guaranteed visible to reader → safe ✓
  NO  → data race → behavior undefined → fix it ✗


THE ONE SENTENCE TO REMEMBER:

  "Happens-before is Java's formal guarantee that
   if action A happens-before action B,
   everything A wrote is guaranteed to be visible to B —
   and it is established ONLY by specific synchronization
   actions: synchronized, volatile, start(), join(),
   and interrupt()."
```