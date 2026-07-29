# Visibility Problem — Complete Tutorial
---
## The Story Before the Definition

```css
Imagine a large office.

There is ONE master whiteboard at the front of the room.
That is the OFFICIAL record of everything.

But the office is big — so each employee has a
PERSONAL STICKY NOTE on their own desk.
When they need a value, they copy it from the
master whiteboard to their sticky note.
Then they work from their sticky note — it is faster
than walking to the whiteboard every time.

Now Employee A updates a value on the master whiteboard.
"Counter is now 42."

Employee B is sitting at their desk.
Their sticky note still says "Counter = 0."
Employee B has NOT walked to the whiteboard to re-read it.
Employee B is working from a STALE copy.

Employee A's write is DONE.
But Employee B cannot SEE it.

This is the Visibility Problem.

The master whiteboard = Main Memory (RAM)
The personal sticky note = CPU Cache / Working Memory
Employee = Thread
```

---

## The Hardware Reality — Why This Happens

```cs
MODERN CPU ARCHITECTURE
════════════════════════════════════════════════════════════════════

                   ┌──────────────────────────────────┐
                   │         MAIN MEMORY (RAM)        │
                   │                                  │
                   │  counter  = 0                    │
                   │  flag     = false                │
                   │  value    = 0                    │
                   │                                  │
                   │  (authoritative master copy)     │
                   └──────────────┬───────────────────┘
                                  │
              ┌───────────────────┴────────────────────┐
              │  Memory Bus (slow ~100-300 CPU cycles) │
              └───────────────────┬────────────────────┘
                                  │
              ┌───────────────────┴─────────────────────┐
              │                                         │
   ┌──────────▼───────────┐              ┌──────────────▼────────┐
   │      CPU CORE 1      │              │      CPU CORE 2       │
   │                      │              │                       │
   │  ┌────────────────┐  │              │  ┌─────────────────┐  │
   │  │ L3 Cache       │  │              │  │ L3 Cache        │  │
   │  │ (shared)       │  │              │  │ (shared)        │  │
   │  └────────────────┘  │              │  └─────────────────┘  │
   │  ┌────────────────┐  │              │  ┌─────────────────┐  │
   │  │ L2 Cache       │  │              │  │ L2 Cache        │  │
   │  │ counter = 0    │  │              │  │ counter = 0     │  │
   │  └────────────────┘  │              │  └─────────────────┘  │
   │  ┌────────────────┐  │              │  ┌─────────────────┐  │
   │  │ L1 Cache       │  │              │  │ L1 Cache        │  │
   │  │ counter = 42   │◄─┼─ Thread A    │  │ counter = 0     │◄─┼─ Thread B
   │  │ (UPDATED)      │  │  writes here │  │ (STALE)         │  │  reads this!
   │  └────────────────┘  │              │  └─────────────────┘  │
   │  ┌────────────────┐  │              │  ┌─────────────────┐  │
   │  │ Registers      │  │              │  │ Registers       │  │
   │  └────────────────┘  │              │  └─────────────────┘  │
   └──────────────────────┘              └───────────────────────┘

THE PROBLEM:
  Thread A writes counter = 42 → goes to Core 1's L1 cache
  Main memory still has counter = 0
  Thread B reads counter → reads from Core 2's L1 cache → gets 0!

  Thread A's write is REAL and DONE.
  But it has NOT propagated to main memory yet.
  Thread B cannot see it.
  Thread B is working with STALE data.
```

---
## Why CPU Caches Exist (The Necessary Evil)
```css
Speed comparison of memory hierarchy:

  ┌─────────────────────┬────────────────┬──────────────────────────┐
  │ Memory Level        │ Access Time    │ Size (typical)           │
  ├─────────────────────┼────────────────┼──────────────────────────┤
  │ CPU Register        │ 1 cycle        │ ~bytes                   │
  │ L1 Cache            │ 4 cycles       │ 32 KB per core           │
  │ L2 Cache            │ 12 cycles      │ 256 KB per core          │
  │ L3 Cache            │ 30-40 cycles   │ 8-32 MB shared           │
  │ Main Memory (RAM)   │ 100-300 cycles │ 8-64 GB                  │
  │ SSD Storage         │ 100,000 cycles │ 256 GB - 4 TB            │
  └─────────────────────┴────────────────┴──────────────────────────┘

  Without caches: every variable access = 100-300 cycles
  With caches:    most accesses = 1-4 cycles

  CPUs are 100x faster than RAM.
  Without caches, CPUs spend 99% of time waiting for RAM.
  Caches are NOT optional — they are what makes modern CPUs fast.

  BUT: caches on DIFFERENT cores are NOT automatically synchronized.
  That is the visibility problem.
```
---
## The Simplest Visibility Bug
```java
// ═══════════════════════════════════════════════════════════════
//  THE CLASSIC VISIBILITY BUG
//  One thread writes. Another thread never sees the write.
// ═══════════════════════════════════════════════════════════════
public class ClassicVisibilityBug {

    // Plain boolean — no visibility guarantee between threads
    // Each thread may work from its own CPU cache copy
    private static boolean stopRequested = false;

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker: started spinning...");

            // Worker reads stopRequested ONCE into its CPU register/cache
            // Then loops using its CACHED copy
            // It may NEVER re-read from main memory
            // This loop may run FOREVER
            while (!stopRequested) {
                // spin spin spin...
                // JIT compiler may optimize this to:
                //   if (!stopRequested) { while (true) {} }
                // Because it sees stopRequested never changes
                // in this thread's view — it HOISTS the read OUT of the loop!
            }

            System.out.println("Worker: stop signal seen — exiting.");
        });

        worker.start();
        Thread.sleep(1000); // let worker run for 1 second

        stopRequested = true; // Main thread writes to main memory (or its cache)
        System.out.println("Main: set stopRequested = true");

        worker.join(3000); // wait max 3 seconds for worker

        if (worker.isAlive()) {
            System.out.println("Main: worker STILL RUNNING after 3 seconds!");
            System.out.println("Main: worker never saw the update — VISIBILITY BUG!");
            worker.interrupt();
        }
    }
}
```

**What happens:**
```cs
Worker: started spinning...
Main  : set stopRequested = true
[1 second passes]
[2 seconds pass]
[3 seconds pass]
Main  : worker STILL RUNNING after 3 seconds!
Main  : worker never saw the update — VISIBILITY BUG!

Worker thread loops FOREVER because:
  1. Main thread wrote stopRequested=true to ITS CPU cache
  2. Worker thread reads from ITS OWN CPU cache → still sees false
  3. JIT compiler may have hoisted the read out of the loop entirely
  4. Worker never re-reads from main memory — no reason to
     (no synchronization action forcing it)
```
---
## The JIT Compiler Makes It Worse — Hoisting

```java
public class JITHoisting {

    static boolean flag = false;

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {

            // What YOU wrote:
            while (!flag) { }

            // What JIT compiler may TRANSFORM this into
            // (after seeing flag never changes in this thread):
            //
            //   if (!flag) {
            //       while (true) { }   ← infinite loop — flag check removed!
            //   }
            //
            // JIT reasoning:
            //   "flag is only written by main thread.
            //    From THIS thread's perspective, flag never changes.
            //    I can cache it in a register permanently.
            //    I'll read it once and loop forever if it's false."
            //
            // This is a LEGAL optimization under JMM
            // because there is NO happens-before between the
            // main thread's write and this thread's read.
            // Without synchronization, JIT is FREE to do this.

            System.out.println("Exited loop");
        });

        worker.start();
        Thread.sleep(500);
        flag = true; // JIT doesn't know about this — optimized it away!
        System.out.println("Set flag = true");
        worker.join(2000);

        if (worker.isAlive()) {
            System.out.println("Worker still running — JIT hoisted the read!");
        }
    }
}
```

```css
JIT HOISTING — before and after:

  YOUR CODE:                      JIT COMPILED CODE:
  ─────────────────────           ─────────────────────────────
  while (!flag) {                 // JIT sees: flag never written
      // spin                     // in this thread → cache it
  }                               boolean cachedFlag = flag; // read once
                                  if (!cachedFlag) {
                                      while (true) { } // ← FOREVER
                                  }

  The JIT is NOT wrong — it is following the JMM rules correctly.
  JMM says: without synchronization, no visibility guarantee.
  JIT exploits this to optimize.
  YOU must use synchronization to prevent this.
```
---
## Two Dimensions of the Visibility Problem

```js
The Visibility Problem has TWO distinct dimensions.
Both must be understood.

DIMENSION 1: WRITE VISIBILITY
══════════════════════════════
  A thread writes a value.
  Other threads may NOT see that write.
  The write is trapped in the writing thread's CPU cache.

  Thread A writes: counter = 42
  Thread B reads:  counter → sees 0   ← write not visible

DIMENSION 2: WRITE ORDERING (Reordering)
══════════════════════════════════════════
  Even when writes BECOME visible to another thread,
  they may appear in a DIFFERENT ORDER than written.

  Thread A writes (in this order):
    value = 42;      // write 1
    flag  = true;    // write 2

  Thread B reads (may see):
    flag  = true     // sees write 2 FIRST
    value = 0        // but write 1 NOT YET VISIBLE!

  The CPU can reorder writes for performance.
  Thread B may see flag=true but value still 0.
  This is LEGAL under JMM without synchronization.
```
---
## Dimension 1 — Write Visibility — Full Example

```java
public class WriteVisibility {

    private static int  value        = 0;
    private static boolean ready     = false;

    public static void main(String[] args) throws InterruptedException {

        Thread writer = new Thread(() -> {
            value = 42;       // write value first
            ready = true;     // write flag second
            System.out.println("Writer: wrote value=42, ready=true");
        });

        Thread reader = new Thread(() -> {
            // Spin until ready is true, then read value
            int attempts = 0;
            while (!ready) {
                attempts++;
                if (attempts > 1_000_000_000) {
                    System.out.println("Reader: gave up waiting — ready never seen!");
                    return;
                }
            }
            // We saw ready=true — what is value?
            System.out.println("Reader: saw ready=true after "
                    + attempts + " attempts");
            System.out.println("Reader: value = " + value);
            // EXPECTED: 42
            // POSSIBLE: 0  ← visibility bug — value write not yet visible
            //              even though ready write IS visible
        });

        reader.start();
        Thread.sleep(100); // let reader start spinning
        writer.start();

        writer.join();
        reader.join();
    }
}
```

```css
THREE POSSIBLE OUTCOMES (all legal under JMM without sync):

  Outcome 1 (correct — both writes propagated together):
    Reader: saw ready=true
    Reader: value = 42   ✓

  Outcome 2 (visibility bug — ready propagated, value did not):
    Reader: saw ready=true
    Reader: value = 0    ← value write not yet visible!
                            ready=true seen but value=42 NOT seen
                            writes propagated in different order

  Outcome 3 (visibility bug — ready never propagates):
    Reader: gave up waiting — ready never seen!
    ← ready=true write never left writer's CPU cache

  All three are legal.
  JMM does NOT guarantee WHICH one happens.
  Without synchronization, all are possible.
```
---
## Dimension 2 — Write Reordering — Full Example

```java
public class WriteReordering {

    static int    x = 0;
    static int    y = 0;
    static int    result1 = -1;
    static int    result2 = -1;

    public static void main(String[] args) throws Exception {
        int reorderingsSeen = 0;

        for (int trial = 0; trial < 1_000_000; trial++) {
            x = 0; y = 0; result1 = -1; result2 = -1;

            CyclicBarrier barrier = new CyclicBarrier(2);

            Thread t1 = new Thread(() -> {
                try { barrier.await(); } catch (Exception e) {}
                x = 1;           // write x first
                result2 = y;     // then read y into result2
            });

            Thread t2 = new Thread(() -> {
                try { barrier.await(); } catch (Exception e) {}
                y = 1;           // write y first
                result1 = x;     // then read x into result1
            });

            t1.start(); t2.start();
            t1.join();  t2.join();

            // Logically, at least ONE of these must be 1:
            //   result1 = 1  means t2 read x AFTER t1 wrote x=1
            //   result2 = 1  means t1 read y AFTER t2 wrote y=1
            //
            // result1=0 AND result2=0 means BOTH reads happened
            // BEFORE either write — only possible if CPU reordered:
            //   t1: result2 = y  EXECUTED BEFORE  x = 1
            //   t2: result1 = x  EXECUTED BEFORE  y = 1
            if (result1 == 0 && result2 == 0) {
                reorderingsSeen++;
            }
        }

        System.out.println("Reorderings observed: " + reorderingsSeen
                + " / 1,000,000");
        // This WILL print a non-zero number on multi-core hardware
        // Proving that CPU reordered the instructions
    }
}
```

```
WHY result1=0 AND result2=0 IS POSSIBLE:

  YOUR CODE order:                CPU ACTUAL execution order:
  ─────────────────────────       ──────────────────────────────────
  Thread 1:                       Thread 1 (REORDERED by CPU):
    x = 1;           ┐              result2 = y;  ← moved BEFORE x=1
    result2 = y;     ┘              x = 1;

  Thread 2:                       Thread 2 (REORDERED by CPU):
    y = 1;           ┐              result1 = x;  ← moved BEFORE y=1
    result1 = x;     ┘              y = 1;

  With reordering:
    result2 = y → reads y=0 (y not yet written)
    result1 = x → reads x=0 (x not yet written)
    x = 1
    y = 1

  Final: result1=0, result2=0 — the "impossible" outcome
  Proves CPU reordered writes.
```

---

## The JMM Abstract Model — How Java Formalizes Visibility

```
Java Memory Model (JMM) defines an ABSTRACT model:

┌─────────────────────────────────────────────────────────────────────┐
│                         MAIN MEMORY                                 │
│                    (shared by all threads)                          │
│                                                                     │
│   counter = 0    flag = false    value = 0                          │
│   (master copies — authoritative)                                   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              ┌────────────────┴─────────────────────┐
              │                                      │
   ┌──────────▼─────────────┐           ┌────────────▼──────────────┐
   │  WORKING MEMORY        │           │  WORKING MEMORY           │
   │  of Thread A           │           │  of Thread B              │
   │  (thread-local copy)   │           │  (thread-local copy)      │
   │                        │           │                           │
   │  counter = 42  ← new   │           │  counter = 0   ← stale    │
   │  flag    = true← new   │           │  flag    = false← stale   │
   │  value   = 99  ← new   │           │  value   = 0   ← stale    │
   └────────────────────────┘           └───────────────────────────┘
          ↑                                      ↑
     Thread A                              Thread B
     (wrote new values)                    (reading stale values)

JMM RULE (without synchronization):
  A thread is NOT required to write its working memory
  to main memory at any specific time.
  A thread is NOT required to refresh its working memory
  from main memory at any specific time.

  This means:
  Thread A's writes may NEVER reach Thread B
  unless a synchronization action forces it.

JMM RULE (with synchronization):
  A synchronized unlock forces ALL writes to main memory.
  A synchronized lock forces a fresh read from main memory.
  This is how visibility is GUARANTEED.
```

---

## The Happens-Before Guarantee — The JMM's Answer

```
JMM's solution to visibility is the HAPPENS-BEFORE (hb) relationship.

  If Action A  happens-before  Action B:
    ALL writes done by A (and everything before A)
    are GUARANTEED to be visible to B.

  If there is NO happens-before:
    No visibility guarantee whatsoever.

HAPPENS-BEFORE RULES that establish visibility:

  Rule 1: Same thread
  ────────────────────
    Within ONE thread, everything happens in program order.
    Line 1 hb Line 2 hb Line 3 ...
    (single-thread visibility is always guaranteed)

  Rule 2: volatile write → volatile read
  ──────────────────────────────────────
    Write to volatile variable V  hb  subsequent read of V
    AND all writes before the volatile write
    are also visible to the reader.

  Rule 3: synchronized unlock → synchronized lock
  ────────────────────────────────────────────────
    Unlock of monitor M  hb  subsequent lock of M
    Everything written inside synchronized block
    is visible to next thread that acquires same lock.

  Rule 4: Thread.start() → thread body
  ──────────────────────────────────────
    t.start()  hb  any action in thread t
    Everything written before t.start()
    is visible to thread t when it starts.

  Rule 5: thread body → Thread.join()
  ─────────────────────────────────────
    Any action in thread t  hb  t.join() returns
    Everything written by thread t
    is visible to the thread that called t.join().

  Rule 6: Transitivity
  ──────────────────────
    A hb B  AND  B hb C  →  A hb C
```

---

## The Three Fixes for Visibility Problem

### Fix 1 — `volatile`

```java
// ═══════════════════════════════════════════════════════════════
//  FIX 1: volatile — the visibility keyword
// ═══════════════════════════════════════════════════════════════
public class VolatileFix {

    // volatile guarantees:
    //   1. Every WRITE is immediately written to MAIN MEMORY
    //      (bypasses CPU cache for writes)
    //   2. Every READ reads from MAIN MEMORY
    //      (bypasses CPU cache for reads)
    //   3. Write to volatile hb read of same volatile
    //   4. Prevents JIT from hoisting the read out of loops
    //   5. All writes BEFORE volatile write are also flushed
    //   6. All reads AFTER volatile read see the latest values

    private static volatile boolean stopRequested = false; // ← volatile added
    private static volatile int     value         = 0;     // ← volatile added

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker: started");

            while (!stopRequested) {
                // Now: every iteration reads stopRequested from MAIN MEMORY
                // JIT CANNOT hoist this out of the loop
                // volatile read prevents the optimization
            }

            // Guaranteed: if we see stopRequested=true,
            // we also see value = 42 (because value was written before stopRequested)
            System.out.println("Worker: stopped. value=" + value);
            // GUARANTEED to print 42 — not 0
        });

        worker.start();
        Thread.sleep(1000);

        value         = 42;          // ordinary write — flushed with volatile write below
        stopRequested = true;        // volatile write → flushes ALL pending writes to main memory
                                     // happens-before any thread's volatile read of stopRequested
        System.out.println("Main: set stop signal");

        worker.join();
        System.out.println("Main: worker done");
    }
}
```

**volatile as a memory fence — visualized:**

```
WITHOUT volatile:
══════════════════════════════════════════════════════════════════

  Thread A writes:              CPU Store Buffer          Main Memory
  ─────────────────             ─────────────────         ───────────
  value = 42;    ─────────────► [buffered]                value = 0
  stopRequested  ─────────────► [buffered]                flag  = false
    = true

  Thread B reads from its cache → sees old values forever


WITH volatile on stopRequested:
══════════════════════════════════════════════════════════════════

  Thread A writes:              Store Buffer              Main Memory
  ─────────────────             ─────────────             ───────────
  value = 42;    ─────────────► [buffered, NOT yet in mem]
  stopRequested                 ← volatile write acts as STORE FENCE
    = true ─────────────────────►FLUSHES ALL PENDING ──►  value = 42
                                 WRITES TO MAIN MEM        flag  = true

  Thread B reads stopRequested (volatile READ = LOAD FENCE):
    reads from main memory → sees true
    ALL writes before the volatile write also visible
    reads value → guaranteed to see 42

VOLATILE FENCE PICTURE:
══════════════════════════════════════════════════════════════════

  Thread A                                    Thread B
  ────────                                    ────────
  value = 42;          ┐
  ... other writes ...  │ these all get        │
  stopRequested=true;  ◄┘ flushed BEFORE       │
                         volatile write        │
                                               ▼
                    ┌──────────────────────────────────────┐
                    │  HAPPENS-BEFORE FENCE                │
                    │  volatile write hb volatile read     │
                    └──────────────────────────────────────┘
                                               │
                                               ▼
                         All of Thread A's    if (stopRequested) ← volatile read
                         writes visible here  value → sees 42  ✓
```

**What volatile does NOT fix:**

```java
public class WhatVolatileDoesNotFix {

    // volatile guarantees VISIBILITY but NOT ATOMICITY
    private static volatile int counter = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
                // ↑ volatile guarantees every READ and WRITE
                //   hits main memory.
                //
                // BUT counter++ is still:
                //   READ  counter from main memory → register  (step 1)
                //   ADD   1                                     (step 2)
                //   WRITE result back to main memory            (step 3)
                //
                // Thread 2 can still run between step 1 and step 3!
                // Both threads read 5, both compute 6, both write 6
                // One increment LOST — despite volatile
                //
                // volatile fixes visibility (step 1 and 3 go to main memory)
                // volatile does NOT fix atomicity (3 steps still exist)
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
            }
        });

        t1.start(); t2.start();
        t1.join();  t2.join();

        System.out.println("Expected: 20000");
        System.out.println("Actual  : " + counter); // still wrong! e.g. 17843
        System.out.println("volatile fixes VISIBILITY, NOT ATOMICITY");
    }
}
```

```
VOLATILE — WHAT IT FIXES AND WHAT IT DOESN'T:

╔══════════════════╦═══════════════════════════════════════════════════╗
║ PROBLEM          ║ VOLATILE FIX?                                     ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ Stale read       ║ ✅ YES — forces read from main memory             ║
║ (while(!flag))   ║         prevents JIT hoisting                    ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ Write not seen   ║ ✅ YES — forces write to main memory              ║
║ by other thread  ║         visible to all threads immediately       ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ Write reordering ║ ✅ YES — acts as memory fence                     ║
║ (writes seen out ║         all writes before volatile write         ║
║  of order)       ║         are visible before the volatile write    ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ counter++ race   ║ ❌ NO  — counter++ is 3 steps, not atomic        ║
║ (read-modify-    ║         need synchronized or AtomicInteger        ║
║  write)          ║                                                   ║
╠══════════════════╬═══════════════════════════════════════════════════╣
║ Compound actions ║ ❌ NO  — check-then-act still not atomic         ║
║ (check-then-act) ║         need synchronized                        ║
╚══════════════════╩═══════════════════════════════════════════════════╝

GOLDEN RULE for volatile:
  Use volatile ONLY when:
    → Exactly ONE thread WRITES the variable
    → One or more threads READ the variable
    → The write operation is a SINGLE ASSIGNMENT
       (not a read-modify-write like ++)

  Examples where volatile is CORRECT:
    volatile boolean stopFlag = false;      ← one writer, many readers ✓
    volatile int     state    = INIT;       ← state machine transitions ✓
    volatile Config  config   = initial;    ← reference replacement ✓

  Examples where volatile is WRONG:
    volatile int counter = 0;  counter++;   ← read-modify-write ✗
    volatile int balance = 0;  balance += x; ← read-modify-write ✗
```

### Fix 2 — `synchronized`

```java
// ═══════════════════════════════════════════════════════════════
//  FIX 2: synchronized — fixes BOTH visibility AND atomicity
// ═══════════════════════════════════════════════════════════════
public class SynchronizedFix {

    private static boolean stopRequested = false; // NOT volatile — sync handles it
    private static int     value         = 0;     // NOT volatile — sync handles it
    private static int     counter       = 0;     // NOT volatile — sync handles it

    private static final Object lock = new Object();

    // ── Visibility fix via synchronized ───────────────────────────
    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("Worker: started");

            while (true) {
                boolean stop;
                synchronized (lock) {
                    // ACQUIRING lock:
                    //   JVM flushes all changes from main memory
                    //   to this thread's working memory
                    //   i.e., thread reads FRESH values from main memory
                    stop = stopRequested; // fresh read guaranteed
                }
                if (stop) break;
            }

            // Read value — guaranteed visible because of happens-before
            synchronized (lock) {
                System.out.println("Worker: stopped. value=" + value);
                // guaranteed to see 42 — unlock(main) hb lock(worker)
            }
        });

        worker.start();
        Thread.sleep(1000);

        synchronized (lock) {
            // RELEASING lock:
            //   JVM flushes ALL writes to main memory
            //   before releasing the lock
            //   Establishes happens-before with next lock acquire
            value         = 42;    // written to main memory on unlock
            stopRequested = true;  // written to main memory on unlock
        }

        worker.join();
    }

    // ── synchronized fixes counter++ too ──────────────────────────
    public static synchronized void safeIncrement() {
        counter++; // atomic: only one thread at a time, fresh read, flushed write
    }
}
```

```
HOW synchronized FIXES VISIBILITY:

  synchronized block does TWO memory operations:

  ON ENTERING (acquiring lock):
  ┌────────────────────────────────────────────────────────┐
  │  Thread refreshes its working memory from main memory  │
  │  All variables read inside block are FRESH             │
  │  No stale cache values                                 │
  └────────────────────────────────────────────────────────┘

  ON EXITING (releasing lock):
  ┌────────────────────────────────────────────────────────┐
  │  Thread flushes ALL writes to main memory              │
  │  ALL writes inside block are visible to next locker    │
  │  Establishes happens-before with next lock acquire     │
  └────────────────────────────────────────────────────────┘

  Thread A: synchronized(lock) { value=42; flag=true; }
                                                        ← lock released → flush
  Thread B: synchronized(lock) { // acquire → refresh
                int v = value; // guaranteed 42
                boolean f = flag; // guaranteed true
            }
```

### Fix 3 — `AtomicReference` and `AtomicInteger`

```java
// ═══════════════════════════════════════════════════════════════
//  FIX 3: Atomic classes — visibility + atomicity, lock-free
// ═══════════════════════════════════════════════════════════════
public class AtomicFix {

    // AtomicBoolean — volatile semantics + atomic ops
    private static final AtomicBoolean stopRequested = new AtomicBoolean(false);

    // AtomicInteger — volatile semantics + atomic increment
    private static final AtomicInteger counter = new AtomicInteger(0);

    // AtomicReference — volatile semantics for object references
    private static final AtomicReference<String> message
            = new AtomicReference<>("initial");

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            // AtomicBoolean.get() reads from main memory (volatile semantics)
            // No stale reads
            while (!stopRequested.get()) {
                // safe — volatile read every iteration
            }
            System.out.println("Worker: stopped");
        });

        // AtomicInteger.incrementAndGet() is atomic (CAS instruction)
        // No lost increments — fixes both visibility AND atomicity
        Thread counter1 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter.incrementAndGet(); // atomic read-add-write
            }
        });

        Thread counter2 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter.incrementAndGet();
            }
        });

        worker.start();
        counter1.start(); counter2.start();

        Thread.sleep(500);

        // AtomicBoolean.set() writes to main memory (volatile semantics)
        stopRequested.set(true);  // visible to worker immediately

        // AtomicReference.compareAndSet() — atomic check-then-update
        boolean updated = message.compareAndSet("initial", "updated");
        System.out.println("Message updated: " + updated);
        System.out.println("Message: " + message.get());

        worker.join();
        counter1.join(); counter2.join();

        System.out.println("Counter: " + counter.get()); // 20000 guaranteed
    }
}
```

---

## The Visibility Problem in Spring Boot — Real Example

```java
// ═══════════════════════════════════════════════════════════════
//  VISIBILITY BUG IN SPRING BOOT
//  Singleton bean with non-volatile flag field
// ═══════════════════════════════════════════════════════════════

// ── BROKEN ────────────────────────────────────────────────────
@Service
public class FeatureFlagService_BROKEN {

    // Plain boolean — not volatile
    // Main thread may update this via admin endpoint
    // Request-handling threads may NEVER see the update
    private boolean featureEnabled = false;

    // Called by admin thread via /admin/feature endpoint
    public void enableFeature() {
        featureEnabled = true;   // write — may stay in admin thread's cache!
    }

    // Called by 200 Tomcat request-handling threads
    public boolean isFeatureEnabled() {
        return featureEnabled;   // read — may return stale false FOREVER
    }
}

// ── FIXED with volatile ────────────────────────────────────────
@Service
public class FeatureFlagService_FIXED {

    // volatile — one writer (admin thread), many readers (request threads)
    // Perfect use case for volatile
    private volatile boolean featureEnabled = false;

    public void enableFeature() {
        featureEnabled = true;   // volatile write → visible to ALL threads
    }

    public boolean isFeatureEnabled() {
        return featureEnabled;   // volatile read → always fresh value
    }
}

// ── ANOTHER REAL EXAMPLE: configuration reload ────────────────
@Service
public class ConfigService_BROKEN {

    // Non-volatile reference to config object
    // Admin thread replaces config — request threads may see old config
    private Config config = Config.loadFromFile();

    // Called by admin thread
    public void reloadConfig() {
        config = Config.loadFromFile(); // new object — non-volatile write
        // Request threads may NEVER see the new Config object
        // They keep using the old one from their CPU cache
    }

    // Called by 200 request threads
    public String getHost() {
        return config.getHost(); // reading potentially stale config reference
    }
}

@Service
public class ConfigService_FIXED {

    // volatile reference — replacing the reference is a single write
    // volatile is CORRECT here (single assignment, not read-modify-write)
    private volatile Config config = Config.loadFromFile();

    public void reloadConfig() {
        config = Config.loadFromFile(); // volatile write → visible to all threads
    }

    public String getHost() {
        return config.getHost(); // fresh read of config reference guaranteed
    }
}
```

---

## Comparing All Three Fixes

```java
public class AllThreeFixes {

    // ─────────────────────────────────────────────────────────────
    // SCENARIO: background thread sets a flag, many threads read it
    // ─────────────────────────────────────────────────────────────

    // FIX 1: volatile — best for this scenario
    private static volatile boolean flag1 = false;

    // FIX 2: synchronized — works but heavier
    private static          boolean flag2 = false;
    private static final Object     lock  = new Object();

    // FIX 3: AtomicBoolean — lock-free, good alternative
    private static final AtomicBoolean flag3 = new AtomicBoolean(false);


    // WRITING the flag:
    void writeWithVolatile()     { flag1 = true; }       // single line, fast
    void writeWithSynchronized() {
        synchronized(lock) { flag2 = true; }             // needs lock overhead
    }
    void writeWithAtomic()       { flag3.set(true); }    // clean, fast


    // READING the flag:
    boolean readWithVolatile()     { return flag1; }         // simple
    boolean readWithSynchronized() {
        synchronized(lock) { return flag2; }                 // lock every read
    }
    boolean readWithAtomic()       { return flag3.get(); }   // simple


    // WHEN TO USE WHICH:
    // ─────────────────────────────────────────────────────────────
    //
    // volatile:
    //   ✓ ONE writer, MANY readers
    //   ✓ Write is a SINGLE assignment (not ++)
    //   ✓ No compound operations needed
    //   ✓ Fastest — no lock acquisition
    //   ✗ Does NOT fix counter++ style operations
    //
    // synchronized:
    //   ✓ MULTIPLE writers AND readers
    //   ✓ Compound operations (counter++, check-then-act)
    //   ✓ Multiple fields must be consistent together
    //   ✓ Fixes both visibility AND atomicity
    //   ✗ Slower — requires lock acquisition and release
    //   ✗ Risk of deadlock if locks nested improperly
    //
    // AtomicInteger/AtomicBoolean/AtomicReference:
    //   ✓ MULTIPLE writers AND readers for SINGLE variable
    //   ✓ Atomic compound ops (compareAndSet, incrementAndGet)
    //   ✓ Lock-free — no deadlock risk
    //   ✓ Faster than synchronized under high contention
    //   ✗ Only works for single-variable operations
    //   ✗ Cannot coordinate multiple fields together
}
```

---

## The Visibility Problem — Complete Mental Model

```vbnet
THE STICKY NOTE ANALOGY — COMPLETE:
════════════════════════════════════════════════════════════════════

  MASTER WHITEBOARD = Main Memory
  PERSONAL STICKY NOTE = CPU Cache (per-thread working memory)
  EMPLOYEE = Thread

  READING:
    Without sync:  Employee reads from their own sticky note (stale!)
    With volatile: Employee always reads from master whiteboard (fresh!)
    With sync:     Employee updates sticky from whiteboard on lock acquire

  WRITING:
    Without sync:  Employee writes to sticky note (may never reach whiteboard)
    With volatile: Employee writes DIRECTLY to master whiteboard (always visible)
    With sync:     Employee updates whiteboard when releasing lock (visible to next locker)

  ORDERING:
    Without sync:  Employee can reorder their own sticky note updates
                   Other employees may see updates in different order
    With volatile: Volatile write = employee puts pen down and ANNOUNCES
                   "I've updated the whiteboard — everyone come refresh"
                   All writes before announcement are guaranteed visible
    With sync:     Lock release = employee finishes section, hands off marker
                   Next employee to pick up marker sees everything up to handoff


THE CANONICAL RULE:
════════════════════════════════════════════════════════════════════

  A thread is NEVER guaranteed to see a write by another thread
  UNLESS there is a happens-before relationship between
  the write and the read.

  Happens-before is established by:
    volatile write → volatile read (same variable)
    synchronized unlock → synchronized lock (same monitor)
    thread.start() → thread body
    thread body → thread.join()


DECISION FLOWCHART:
════════════════════════════════════════════════════════════════════

  Is data SHARED between threads?
       NO → no problem (stack vars, ThreadLocal)
       YES ↓

  Is data MUTABLE (written after creation)?
       NO → no problem (final, immutable objects, constants)
       YES ↓

  Is the write a SINGLE ASSIGNMENT?
       YES → consider volatile (if one writer)
       NO  → need synchronized or Atomic*

  Are MULTIPLE FIELDS written together that must be consistent?
       YES → need synchronized (volatile can't group operations)
       NO  → volatile or Atomic* may be sufficient


SUMMARY TABLE:
════════════════════════════════════════════════════════════════════

╔══════════════════════╦══════════╦══════════════╦═════════════════╗
║ SYMPTOM              ║ VOLATILE ║ SYNCHRONIZED ║ ATOMIC*         ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ Stale read (flag)    ║ ✅ Fix   ║ ✅ Fix       ║ ✅ Fix          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ JIT hoisting loop    ║ ✅ Fix   ║ ✅ Fix       ║ ✅ Fix          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ Write not visible    ║ ✅ Fix   ║ ✅ Fix       ║ ✅ Fix          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ Write reordering     ║ ✅ Fix   ║ ✅ Fix       ║ ✅ Fix          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ counter++ lost update║ ❌ No    ║ ✅ Fix       ║ ✅ Fix          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ check-then-act       ║ ❌ No    ║ ✅ Fix       ║ ✅ CAS          ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ Multi-field atomicity║ ❌ No    ║ ✅ Fix       ║ ❌ No           ║
╠══════════════════════╬══════════╬══════════════╬═════════════════╣
║ Performance          ║ 🚀 Fast  ║ 🐢 Slower   ║ 🚀 Fast         ║
╚══════════════════════╩══════════╩══════════════╩═════════════════╝


THE ONE SENTENCE TO REMEMBER:

  "The Visibility Problem is when one thread's write
   to shared memory is not seen by another thread —
   because the write is trapped in the writer's CPU cache
   and never flushed to main memory where others can read it."
```