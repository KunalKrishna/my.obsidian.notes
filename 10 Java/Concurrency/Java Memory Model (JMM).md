# Java Memory Model (JMM) — Complete Tutorial

---
## The Story Before the Model

```
It is 2004. A senior Java developer named Doug is debugging
a production issue that has been haunting his team for weeks.

The bug:
  A configuration object is updated by an admin thread.
  200 request-handling threads read that configuration.
  Sometimes — randomly, unpredictably — the request threads
  read a PARTIALLY INITIALIZED configuration.
  Half the fields are new. Half are old.
  The application crashes.

Doug reads every Java book available.
None of them precisely explains WHEN one thread's write
becomes visible to another thread.
The language specification says:
  "synchronized and volatile provide some guarantees"
but never defines EXACTLY what those guarantees are.

Doug is not alone.
Thousands of Java developers are fighting the same ghost.

The Java Community Process forms JSR-133.
Brian Goetz, Jeremy Manson, Bill Pugh and others
spend years formally defining exactly how Java threads
interact with memory.

The result: The Java Memory Model (JMM).
Introduced in Java 5.
The formal contract between the developer and the JVM.
The answer to Doug's question.
And the foundation of everything in Java concurrency.
```

---
## What JMM Actually Is

```
JMM is NOT:
  ✗ A description of physical CPU caches
  ✗ A description of how the OS schedules threads
  ✗ A performance optimization guide
  ✗ A description of what the hardware actually does

JMM IS:
  ✓ A formal specification (Chapter 17 of JLS)
  ✓ A contract between the Java developer and the JVM
  ✓ A set of rules defining when writes become visible
  ✓ A set of rules defining what orderings are guaranteed
  ✓ Platform-independent (works regardless of CPU architecture)
  ✓ The answer to: "Is my multi-threaded code correct?"

THE CONTRACT:
  Developer's side:
    "I will use the synchronization mechanisms
     you have provided (synchronized, volatile, etc.)"

  JVM's side:
    "If you do that, I GUARANTEE your code behaves
     as you intended — on ANY hardware, ANY OS,
     ANY JVM implementation."

  Neither side of the contract: 
    No synchronization → no guarantee → anything can happen.
```

---
## The Three Problems JMM Must Solve

```
╔═════════════════════════════════════════════════════════════════════╗
║         THREE PROBLEMS JMM MUST FORMALLY ADDRESS                    ║
╠══════════════╦══════════════════════════════════════════════════════╣
║  PROBLEM     ║  DESCRIPTION                                         ║
╠══════════════╬══════════════════════════════════════════════════════╣
║  VISIBILITY  ║  Thread A writes x=42.                               ║
║              ║  Thread B reads x → may still see 0.                 ║
║              ║  Write trapped in A's CPU cache.                     ║
║              ║  B reads from its own stale cache.                   ║
╠══════════════╬══════════════════════════════════════════════════════╣
║  ATOMICITY   ║  counter++ looks like one step.                      ║
║              ║  It is actually READ + ADD + WRITE.                  ║
║              ║  Another thread can run between these steps.         ║
║              ║  Result: lost updates, incorrect final values.       ║
╠══════════════╬══════════════════════════════════════════════════════╣
║  ORDERING    ║  JVM and CPU reorder instructions for speed.         ║
║              ║  Your code runs in a different order than written.   ║
║              ║  Invisible on one thread but catastrophic when       ║
║              ║  observed from another thread.                       ║
╚══════════════╩══════════════════════════════════════════════════════╝
```

---
## The JMM Abstract Memory Model

```
JMM defines an abstract model — NOT tied to physical hardware.
Every JVM implementation on every platform must honour it.

╔══════════════════════════════════════════════════════════════════════════╗
║                    JMM ABSTRACT MEMORY MODEL                            ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║   ┌──────────────────────────────────────────────────────────────────┐  ║
║   │                      MAIN MEMORY                                 │  ║
║   │              (shared — all threads can access)                   │  ║
║   │                                                                  │  ║
║   │   counter = 0    flag = false    obj = null                      │  ║
║   │   (MASTER COPIES — the authoritative values)                     │  ║
║   └──────────────────────────────────────────────────────────────────┘  ║
║          │                    │                    │                     ║
║          │ read/write         │ read/write         │ read/write          ║
║          │ (not always        │ (not always        │ (not always         ║
║          │  immediate)        │  immediate)        │  immediate)         ║
║          ▼                    ▼                    ▼                     ║
║   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 ║
║   │  WORKING    │    │  WORKING    │    │  WORKING    │                  ║
║   │  MEMORY     │    │  MEMORY     │    │  MEMORY     │                  ║
║   │  Thread 1   │    │  Thread 2   │    │  Thread 3   │                  ║
║   │             │    │             │    │             │                  ║
║   │ counter=42  │    │ counter=0   │    │ counter=0   │                  ║
║   │ (updated)   │    │ (stale)     │    │ (stale)     │                  ║
║   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                 ║
║          │                  │                  │                         ║
║   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐                 ║
║   │  Thread 1   │    │  Thread 2   │    │  Thread 3   │                  ║
║   │  (running)  │    │  (running)  │    │  (running)  │                  ║
║   └─────────────┘    └─────────────┘    └─────────────┘                 ║
║                                                                          ║
║  KEY RULES of this abstract model:                                       ║
║  1. All shared variables live in MAIN MEMORY                             ║
║  2. Each thread has its own WORKING MEMORY (CPU cache abstraction)       ║
║  3. Thread can ONLY operate on its working memory directly               ║
║  4. JMM does NOT define WHEN working memory syncs with main memory       ║
║     ← THIS IS THE CORE OF THE PROBLEM                                    ║
║  5. Synchronization actions FORCE the sync at defined points             ║
╚══════════════════════════════════════════════════════════════════════════╝

The abstract model maps to physical hardware like this:

  JMM concept         →  Physical reality
  ──────────────────      ─────────────────────────────────────
  Main Memory         →  RAM (DRAM)
  Working Memory      →  CPU L1/L2/L3 caches + registers
  Sync action         →  Memory barrier / fence instruction
  volatile write      →  SFENCE (store fence) instruction
  volatile read       →  LFENCE (load fence) instruction
  synchronized        →  MFENCE (full fence) + lock prefix
```

---

## Problem 1 — Visibility — Code and Internals

```java
// ═══════════════════════════════════════════════════════════════
//  VISIBILITY PROBLEM — in all its forms
// ═══════════════════════════════════════════════════════════════
public class JMM_Visibility {

    // ── FORM 1: Simple stale read ─────────────────────────────────
    static int  value      = 0;
    static boolean ready   = false;

    static void visibilityBug() throws InterruptedException {
        Thread writer = new Thread(() -> {
            value = 42;      // W1: write to value
            ready = true;    // W2: write to ready
        });

        Thread reader = new Thread(() -> {
            while (!ready) { } // spin — may never see ready=true
            System.out.println(value); // may print 0 even if ready=true
            // TWO visibility problems:
            //   1. ready=true may never be seen (stale read)
            //   2. value=42 may not be seen even if ready=true is seen
            //      (writes can become visible in DIFFERENT ORDER)
        });

        reader.start(); writer.start();
        writer.join(); reader.join();
    }

    // ── FORM 2: Stale object reference ────────────────────────────
    static class Config {
        String host = "localhost";
        int    port = 8080;
    }

    static Config config = null; // non-volatile reference

    static void objectVisibilityBug() throws InterruptedException {
        Thread writer = new Thread(() -> {
            Config c = new Config(); // create object
            c.host = "prod-server";  // initialize fields
            c.port = 443;
            config = c;              // publish reference — NON-VOLATILE WRITE
            // Problem: on some hardware/JVM:
            //   1. config reference becomes visible to reader
            //   2. BUT fields host and port may NOT be visible yet
            //   Reader sees config != null but config.host = "localhost"
            //   This is a PARTIALLY CONSTRUCTED OBJECT read
        });

        Thread reader = new Thread(() -> {
            while (config == null) { }
            // config is non-null — but are its fields initialized?
            // JMM says: NO GUARANTEE without synchronization
            System.out.println(config.host); // may print "localhost" !
            System.out.println(config.port); // may print 8080 !
        });

        writer.start(); reader.start();
        writer.join(); reader.join();
    }
}
```

```
WHY WRITES BECOME VISIBLE OUT OF ORDER:

  Thread A writes (in this source order):
    1. value = 42
    2. ready = true

  CPU Store Buffer (between core and cache):
    CPU doesn't write directly to cache/RAM.
    It writes to a local STORE BUFFER first.
    Store buffer is flushed asynchronously.

  CPU may flush them in DIFFERENT order:
    2. ready = true   ← flushed first (smaller, moved up)
    3. value = 42     ← flushed second

  Thread B reads:
    sees ready = true  ← from flushed store
    reads value        ← from its own cache = 0 ← STALE!

  This is called a STORE-LOAD REORDERING.
  It is legal on modern CPUs (x86, ARM, etc.)
  JMM permits this without synchronization.

  Fix: use volatile on ready
       volatile write acts as STORE FENCE
       forces value=42 to be flushed BEFORE ready=true
```

---

## Problem 2 — Atomicity — Code and Internals

```java
// ═══════════════════════════════════════════════════════════════
//  ATOMICITY PROBLEM — operations that look atomic but aren't
// ═══════════════════════════════════════════════════════════════
public class JMM_Atomicity {

    static int counter = 0;

    // ── counter++ is NOT atomic ───────────────────────────────────
    // Bytecode for counter++:
    //   getstatic  #counter    ← READ  from working memory
    //   iconst_1               ← push constant 1
    //   iadd                   ← ADD
    //   putstatic  #counter    ← WRITE to working memory
    //
    // Four bytecode instructions — any thread switch between them
    // causes a race condition

    // ── Long and Double are NOT atomic on 32-bit JVM ──────────────
    static long  longValue   = 0L;
    static double doubleValue = 0.0;

    // On 32-bit JVM:
    //   long and double are 64-bit — written as TWO 32-bit operations
    //   Thread can read between the two 32-bit writes
    //   Resulting in a "torn" value — high 32 bits from one write,
    //   low 32 bits from another write
    //
    // On 64-bit JVM: long and double writes ARE atomic (implementation detail)
    // JMM does NOT guarantee this — use volatile long/double to be safe

    static void longAtomicityBug() throws InterruptedException {
        long value1 = 0xAAAAAAAABBBBBBBBL;
        long value2 = 0xCCCCCCCCDDDDDDDDL;

        Thread writer = new Thread(() -> {
            while (true) {
                longValue = value1;  // may not be atomic on 32-bit!
                longValue = value2;
            }
        });

        Thread reader = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                long v = longValue;
                // On 32-bit JVM could see: 0xAAAAAAAADDDDDDDDL
                // High 32 bits from value1, low 32 bits from value2
                // A "torn" read — impossible with atomic write
                if (v != value1 && v != value2) {
                    System.out.println("TORN READ: " + Long.toHexString(v));
                }
            }
        });

        writer.start(); reader.start();
        Thread.sleep(100);
        writer.interrupt(); reader.interrupt();
    }

    // ── What IS atomic in JMM ──────────────────────────────────────
    // JMM guarantees these are ALWAYS atomic
    // (even without volatile or synchronized):
    //
    //   Read/write of:   int, char, byte, short, boolean, float
    //                    object references (32 or 64 bit)
    //   Read/write of:   long and double WITH volatile keyword
    //
    // Atomic here means:
    //   Another thread cannot see a "half-written" value
    //   BUT it does NOT mean the read sees the latest value (visibility separate)
}
```

---

## Problem 3 — Ordering / Reordering — Code and Internals

```java
// ═══════════════════════════════════════════════════════════════
//  ORDERING PROBLEM — instructions executed in unexpected order
// ═══════════════════════════════════════════════════════════════
public class JMM_Ordering {

    // ── COMPILER REORDERING ───────────────────────────────────────
    // Java compiler (javac + JIT) can reorder instructions
    // when they appear independent to a single thread

    static int  a = 0, b = 0;
    static int  x = 0, y = 0;

    // Thread 1 writes:    Thread 2 writes:
    //   a = 1;              b = 1;
    //   x = b;              y = a;

    // After both finish, possible values:
    //   x=1, y=1  ← Thread 1 ran before Thread 2
    //   x=0, y=1  ← Thread 2 ran before Thread 1
    //   x=1, y=0  ← Threads interleaved
    //   x=0, y=0  ← IMPOSSIBLE? NO! Legal with reordering.
    //               JIT may reorder to: x=b; a=1; in Thread 1
    //               AND:                y=a; b=1; in Thread 2
    //               Both read 0 before either writes 1

    // ── THREE SOURCES OF REORDERING ──────────────────────────────
    //
    //  1. COMPILER REORDERING
    //     javac and JIT reorder bytecode/machine instructions
    //     when they see no data dependency between them
    //
    //  2. CPU OUT-OF-ORDER EXECUTION
    //     Modern CPUs execute instructions out of order
    //     to maximize pipeline utilization
    //     Results are "committed" in order within one thread
    //     but become visible out of order to other threads
    //
    //  3. STORE BUFFER REORDERING
    //     CPU has a write buffer between core and cache
    //     Writes drain from buffer to cache asynchronously
    //     Another core may see writes in a different order
    //     than they were issued

    // ── WITHIN-THREAD REORDERING IS INVISIBLE ─────────────────────
    static void withinThreadReordering() {
        int localA = 1;    // ┐ JVM may execute these
        int localB = 2;    // ┘ in either order
        int sum = localA + localB; // sum = 3 always

        // To THIS thread: always looks like program order (3)
        // The reordering is invisible within one thread
        System.out.println(sum); // always 3
    }

    // ── CROSS-THREAD REORDERING IS VISIBLE AND DANGEROUS ──────────
    static int  result = 0;
    static boolean published = false;

    static void crossThreadReorderingBug() throws InterruptedException {
        Thread writer = new Thread(() -> {
            result    = 99;        // W1
            published = true;      // W2

            // JIT sees W1 and W2 as independent (no data flow between them)
            // JIT may REORDER to: published=true; result=99;
            // On SINGLE THREAD: result is always 99 when published=true
            //   (reads happen after writes in same thread)
            // On MULTI THREAD: reader may see published=true but result=0!
        });

        Thread reader = new Thread(() -> {
            while (!published) { }
            // Sees published=true — but result may be 0!
            // Writer's W1 (result=99) may not have happened yet
            // (from reader's perspective — reordering)
            System.out.println("result = " + result); // may be 0!
        });

        reader.start(); writer.start();
        writer.join(); reader.join();
    }
}
```

```
REORDERING RULES — What JMM PERMITS vs FORBIDS:

  ╔══════════════════════════╦══════════════╦═══════════════════════╗
  ║  REORDERING TYPE         ║  PERMITTED?  ║  REASON               ║
  ╠══════════════════════════╬══════════════╬═══════════════════════╣
  ║  Within same thread      ║  ✓ YES       ║  Invisible to thread  ║
  ║  (independent ops)       ║  (if result  ║  itself — result same ║
  ║                          ║   same)      ║                       ║
  ╠══════════════════════════╬══════════════╬═══════════════════════╣
  ║  Volatile write before   ║  ✗ NO        ║  Volatile is fence    ║
  ║  volatile read           ║              ║  cannot cross it      ║
  ╠══════════════════════════╬══════════════╬═══════════════════════╣
  ║  Into/out of             ║  ✗ NO        ║  Monitor ordering     ║
  ║  synchronized block      ║              ║  must be maintained   ║
  ╠══════════════════════════╬══════════════╬═══════════════════════╣
  ║  Writes before           ║  ✗ NO        ║  start() hb thread    ║
  ║  Thread.start()          ║              ║  body                 ║
  ╠══════════════════════════╬══════════════╬═══════════════════════╣
  ║  Normal writes across    ║  ✓ YES       ║  No happens-before    ║
  ║  threads (no sync)       ║              ║  to prevent it        ║
  ╚══════════════════════════╩══════════════╩═══════════════════════╝
```

---

## JMM's Solution — Synchronization Actions and Memory Barriers

```
JMM's solution to all three problems is the concept of
SYNCHRONIZATION ACTIONS combined with MEMORY BARRIERS.

SYNCHRONIZATION ACTIONS (what you write in Java):
  synchronized block (acquire/release)
  volatile read/write
  Thread.start()
  Thread.join()
  Thread.interrupt()

MEMORY BARRIERS (what JVM inserts in machine code):
  LoadLoad   barrier: no load can be reordered before this barrier
  StoreStore barrier: no store can be reordered after this barrier
  LoadStore  barrier: no load before + no store after can be reordered
  StoreLoad  barrier: all stores before are visible before any load after
                      (most expensive — full fence)

HOW THEY MAP:

  volatile WRITE → StoreStore barrier BEFORE + StoreLoad barrier AFTER
  volatile READ  → LoadLoad barrier AFTER + LoadStore barrier AFTER
  synchronized   → full fence on entry (LOAD) + full fence on exit (STORE)


MEMORY BARRIER PICTURE:

  Thread A code:                 Machine code (after JIT with volatile):
  ──────────────                 ────────────────────────────────────────
  value = 42;                    STORE value, 42
  flag  = true; (volatile)       [StoreStore BARRIER] ← prevents reordering
                                 STORE flag, true
                                 [StoreLoad BARRIER]  ← flushes all stores

  Thread B code:                 Machine code:
  ──────────────                 ──────────────────────────────────────
  while(!flag) {}  (volatile)    [LoadLoad BARRIER]   ← fresh load guaranteed
                                 LOAD flag
                                 [LoadStore BARRIER]
  value → read                   LOAD value            ← sees 42 guaranteed
```

---

## JMM's Formal Tool — The Happens-Before Relationship

```
JMM's formal mechanism for defining visibility and ordering
is the HAPPENS-BEFORE (hb) relationship.

JMM says:
  Action A HAPPENS-BEFORE action B means:
  1. All writes by A are visible to B
  2. A appears to execute before B

  No happens-before = no guarantee = data race = undefined

THE 8 HB RULES (already covered in depth — summarized):

  1. Program Order  : Within thread, each action hb next action
  2. Monitor Lock   : unlock(M) hb lock(M) — same monitor M
  3. Volatile       : write(V) hb read(V) — same volatile V
  4. Thread Start   : start() hb thread body
  5. Thread Join    : thread body hb join() returns
  6. Interruption   : interrupt() hb interrupt detection
  7. Finalizer      : constructor end hb finalizer start
  8. Transitivity   : A hb B AND B hb C → A hb C

THE ONE QUESTION TO ALWAYS ASK:
  "Is there a hb chain between this WRITE and this READ?"
  YES → visible, safe ✓
  NO  → data race, unsafe ✗
```

---

## JMM's Synchronization Mechanisms — Deep Dive

### synchronized — The Complete Guarantee

```java
// ═══════════════════════════════════════════════════════════════
//  synchronized — solves ALL THREE JMM problems
// ═══════════════════════════════════════════════════════════════
public class JMM_Synchronized {

    private int     counter  = 0;
    private String  state    = "INIT";
    private boolean active   = false;
    private final Object lock = new Object();

    // synchronized fixes:
    //   VISIBILITY  : unlock flushes all writes to main memory
    //                 lock refreshes all reads from main memory
    //   ATOMICITY   : mutual exclusion — only one thread inside
    //   ORDERING    : cannot reorder across lock/unlock boundaries

    public void update() {
        synchronized (lock) {
            // ── ON ENTERING (LOCK) ─────────────────────────────────
            // JVM inserts LOAD FENCE:
            //   All reads inside see FRESH values from main memory
            //   No stale cached values from before the lock

            counter++;            // atomic relative to other sync blocks
            state   = "RUNNING";  // visible to next thread that acquires lock
            active  = true;       // visible to next thread that acquires lock

            // ── ON EXITING (UNLOCK) ────────────────────────────────
            // JVM inserts STORE FENCE:
            //   All writes are flushed to main memory
            //   Establishes happens-before with next lock acquisition
        }
    }

    public void read() {
        synchronized (lock) {
            // Same lock → happens-before with update()'s unlock
            System.out.println(counter); // guaranteed fresh
            System.out.println(state);   // guaranteed fresh
            System.out.println(active);  // guaranteed fresh
        }
    }

    // ── THREE FORMS OF synchronized ───────────────────────────────

    // Form 1: Synchronized INSTANCE method
    //         Lock = 'this' (the instance)
    public synchronized void instanceMethod() {
        counter++;
        // Lock is acquired on 'this' object
        // Two threads with SAME instance → one waits
        // Two threads with DIFFERENT instances → both proceed (different locks!)
    }

    // Form 2: Synchronized STATIC method
    //         Lock = Class object (ClassName.class)
    public static synchronized void staticMethod() {
        // Lock is acquired on JMM_Synchronized.class
        // All instances share this lock
        // Any thread calling this method on ANY instance will use same lock
    }

    // Form 3: Synchronized BLOCK (most flexible — preferred)
    //         Lock = specified object
    public void blockForm() {
        synchronized (lock) {    // can be any object, fine-grained control
            counter++;
        }
    }

    // ── REENTRANCE ────────────────────────────────────────────────
    // synchronized is REENTRANT:
    //   A thread that already holds a lock can re-enter
    //   synchronized blocks on the SAME lock without deadlocking

    public synchronized void outer() {
        System.out.println("outer");
        inner(); // calls another synchronized method on same lock
                 // does NOT deadlock — same thread can re-acquire its own lock
    }

    public synchronized void inner() {
        System.out.println("inner");
        // JVM keeps a REENTRY COUNT:
        //   outer() acquires lock → count = 1
        //   inner() re-acquires   → count = 2
        //   inner() returns       → count = 1
        //   outer() returns       → count = 0 → lock released
    }
}
```

```
synchronized MEMORY SEMANTICS — complete picture:

  Thread A:                          Thread B:
  ─────────                          ─────────
  synchronized(lock) {
    // LOCK ACQUIRE:                  [waiting for lock]
    // ─────────────────────────────────────────────────
    // 1. Acquire monitor on 'lock'
    // 2. LOAD FENCE inserted:
    //    all reads from now on
    //    read from main memory
    // ─────────────────────────────────────────────────

    counter  = 42;   // write 1
    message  = "hi"; // write 2

    // LOCK RELEASE:
    // ─────────────────────────────────────────────────
    // 1. STORE FENCE inserted:
    //    all writes flushed to main memory
    // 2. Release monitor on 'lock'
    // 3. Happens-before established with next locker
    // ─────────────────────────────────────────────────
  }                                  synchronized(lock) {
                                       // LOCK ACQUIRE:
                                       // Monitor acquired
                                       // LOAD FENCE:
                                       // reads from main memory

                                       // unlock(A) hb lock(B)
                                       counter  → 42    ✓ (GUARANTEED)
                                       message  → "hi"  ✓ (GUARANTEED)
                                     }
```

### volatile — The Lightweight Visibility Tool

```java
// ═══════════════════════════════════════════════════════════════
//  volatile — solves visibility and ordering but NOT atomicity
// ═══════════════════════════════════════════════════════════════
public class JMM_Volatile {

    // volatile guarantees:
    //   VISIBILITY  : write immediately visible to all threads
    //   ORDERING    : acts as memory fence (prevents reordering)
    //   NOT ATOMIC  : counter++ still broken with volatile

    private volatile boolean flag  = false;
    private          int     value = 0;    // not volatile

    // ── CORRECT USE: flag/sentinel pattern ────────────────────────
    // One writer, many readers
    // Write is a single assignment

    public void writer() {
        value = 42;       // ordinary write
        flag  = true;     // volatile write → STORE FENCE
        //                  ↑
        //                  This volatile write ensures:
        //                  1. value=42 is written to main memory
        //                     BEFORE flag=true is written
        //                  2. Cannot be reordered: value=42 cannot
        //                     move AFTER the volatile write of flag
    }

    public void reader() {
        if (flag) {         // volatile read → LOAD FENCE
            //                ↑
            //                This volatile read ensures:
            //                1. flag is read from main memory (fresh)
            //                2. ALL reads after this point see fresh values
            //                3. value=42 guaranteed visible here
            //                   (because value write hb volatile write hb volatile read)
            System.out.println(value); // guaranteed 42
        }
    }

    // ── volatile FENCE SEMANTICS ──────────────────────────────────
    //
    //  WRITE to volatile:
    //    [StoreStore fence] ← no prior writes can move AFTER this
    //    VOLATILE STORE
    //    [StoreLoad fence]  ← all stores visible before any subsequent loads
    //
    //  READ of volatile:
    //    [LoadLoad fence]   ← no subsequent reads can move BEFORE this
    //    VOLATILE LOAD
    //    [LoadStore fence]  ← all loads before no subsequent stores before

    // ── volatile for reference replacement ────────────────────────
    // Replacing a reference atomically (NOT modifying fields)
    private volatile Config config;

    public void reloadConfig() {
        // Create new config object
        Config newConfig = new Config("newhost", 9090);
        // volatile write: reference replacement is atomic + visible
        config = newConfig;
        // Any thread reading config after this volatile write
        // GUARANTEED to see the new Config reference
        // AND the new Config's FINAL fields (if they are final)
    }

    static class Config {
        final String host;
        final int port;
        Config(String host, int port) {
            this.host = host;
            this.port = port;
        }
    }

    // ── volatile DOES NOT FIX ──────────────────────────────────────
    private volatile int brokenCounter = 0;

    public void brokenIncrement() {
        brokenCounter++;
        // STILL broken! Even with volatile.
        // volatile ensures every read/write is to main memory
        // BUT counter++ is:
        //   LOAD  brokenCounter  (from main memory — volatile load)
        //   ADD   1
        //   STORE brokenCounter  (to main memory — volatile store)
        //
        // Thread A and B can BOTH do the LOAD before EITHER does the STORE
        // Both see 5, both compute 6, both store 6 → one increment lost
        // volatile != atomic for compound operations
    }
}
```

### final Fields — The Strongest Guarantee

```java
// ═══════════════════════════════════════════════════════════════
//  final fields — special JMM guarantee beyond volatile
// ═══════════════════════════════════════════════════════════════
public class JMM_FinalFields {

    // JMM FINAL FIELD GUARANTEE:
    // "An object is considered to be completely initialized
    //  when its constructor finishes. A thread that can only
    //  see a reference to an object after that object has been
    //  completely initialized is guaranteed to see the correctly
    //  initialized values for that object's final fields."
    //
    // In plain terms:
    //   If an object has final fields,
    //   any thread that gets a reference to the object
    //   AFTER the constructor completes
    //   is guaranteed to see the final fields' initialized values.
    //   NO synchronization needed for the fields themselves.

    static class ImmutablePoint {
        final int x;    // ← final field
        final int y;    // ← final field

        ImmutablePoint(int x, int y) {
            this.x = x; // written in constructor
            this.y = y; // written in constructor
            // JMM inserts a StoreStore fence at end of constructor
            // for final fields
            // This prevents: reference publication before fields written
        }
    }

    // Non-volatile reference — normally unsafe!
    static ImmutablePoint point;

    static void publisher() {
        point = new ImmutablePoint(3, 4);
        // Even though 'point' reference is not volatile,
        // the FINAL FIELD GUARANTEE ensures:
        // Any thread that reads point != null will see x=3, y=4
        // The final field fence prevents the constructor's
        // writes from being reordered after reference publication
    }

    static void reader() {
        ImmutablePoint p = point;
        if (p != null) {
            // Final field guarantee:
            System.out.println(p.x); // GUARANTEED: 3
            System.out.println(p.y); // GUARANTEED: 4
            // This is safe WITHOUT volatile on 'point'
            // BECAUSE x and y are final
        }
    }

    // ── CONTRAST WITH NON-FINAL FIELDS ────────────────────────────
    static class MutablePoint {
        int x;    // NOT final
        int y;    // NOT final

        MutablePoint(int x, int y) {
            this.x = x;
            this.y = y;
            // NO JMM fence at end of constructor for non-final fields!
        }
    }

    static MutablePoint mutablePoint;

    static void unsafePublisher() {
        mutablePoint = new MutablePoint(3, 4);
        // NON-final fields — NO final field guarantee
        // reference publication can be reordered before field writes
        // Another thread may see mutablePoint != null
        // but x=0, y=0 (default values) ← PARTIALLY CONSTRUCTED OBJECT
    }

    // ── THE FINAL FIELD FENCE (JMM internals) ─────────────────────
    //
    // JVM inserts a StoreStore FENCE
    // at the end of every constructor that has final fields:
    //
    // Constructor:                   Machine code (simplified):
    //   this.x = 3;                    STORE x, 3
    //   this.y = 4;                    STORE y, 4
    //   [end of constructor]     →     [StoreStore FENCE]
    //                                  STORE reference, this
    //
    // This fence ensures:
    //   x and y writes cannot move AFTER the reference store
    //   Any thread reading the reference will always see
    //   the initialized values of x and y
}
```

---

## Safe Publication — A Critical JMM Application

```java
// ═══════════════════════════════════════════════════════════════
//  SAFE PUBLICATION — making an object visible correctly
// ═══════════════════════════════════════════════════════════════
public class JMM_SafePublication {

    // "Publishing" an object = making its reference visible
    //   to other threads

    // UNSAFE PUBLICATION — three ways it can fail:
    static class UnsafePublicationExamples {

        // Failure 1: non-volatile, non-synchronized reference
        static Helper helperA = null; // not volatile

        static void failureOne() {
            helperA = new Helper(42);
            // Another thread may:
            // a) See helperA == null (write not yet visible)
            // b) See helperA != null but Helper(42) not initialized
            //    (reference visible before constructor completes — reordering)
        }

        // Failure 2: this escaping from constructor
        static Helper escapedHelper;

        static class BadHelper {
            int value;
            BadHelper(int value) {
                escapedHelper = this; // ← 'this' escapes BEFORE constructor done!
                this.value = value;   // ← written AFTER escape
                // Another thread reads escapedHelper.value → may see 0!
                // The final field guarantee doesn't help here
                // because value is not final AND this escapes mid-constructor
            }
        }

        // Failure 3: partial construction via thread interleaving
        static int x, y;
        static boolean published;

        static void failureThree() {
            x = 1;             // W1
            y = 2;             // W2
            published = true;  // W3 — non-volatile flag
            // CPU may reorder: W3 before W1 or W2
            // Reader sees published=true but x=0 or y=0
        }
    }

    // ── FOUR PATTERNS FOR SAFE PUBLICATION ────────────────────────

    static class SafePublicationPatterns {

        // PATTERN 1: static final (class loading guarantee)
        // Safest — classloader synchronizes class initialization
        static final Helper STATIC_FINAL_HELPER = new Helper(42);
        // Any thread accessing STATIC_FINAL_HELPER after class loads
        // GUARANTEED to see fully initialized Helper

        // PATTERN 2: volatile reference
        static volatile Helper volatileHelper;

        static void publishWithVolatile() {
            Helper h = new Helper(42); // create + initialize locally
            volatileHelper = h;        // volatile write → visible + ordered
        }

        // PATTERN 3: synchronized publication
        static Helper syncHelper;
        static final Object LOCK = new Object();

        static void publishWithSync() {
            synchronized (LOCK) {
                syncHelper = new Helper(42); // flush on unlock
            }
        }

        static Helper getWithSync() {
            synchronized (LOCK) {
                return syncHelper; // fresh read on lock
            }
        }

        // PATTERN 4: final fields (immutable objects)
        static class ImmutableHelper {
            final int value;            // final — safe publication guarantee
            ImmutableHelper(int value) { this.value = value; }
        }

        static ImmutableHelper immutableHelper; // even non-volatile is safe!

        static void publishImmutable() {
            immutableHelper = new ImmutableHelper(42);
            // Even without volatile:
            // final field guarantee ensures any thread reading
            // immutableHelper.value will see 42 (not 0)
            // PROVIDED they get the reference after constructor completes
        }
    }

    static class Helper {
        int value;
        Helper(int value) { this.value = value; }
    }
}
```

---

## Double-Checked Locking — The Famous JMM Trap

```java
// ═══════════════════════════════════════════════════════════════
//  DOUBLE-CHECKED LOCKING — JMM's most famous gotcha
//  Understanding this proves you understand JMM deeply
// ═══════════════════════════════════════════════════════════════
public class JMM_DoubleCheckedLocking {

    // ── VERSION 1: BROKEN — pre Java 5 idiom ──────────────────────
    private static JMM_DoubleCheckedLocking brokenInstance;

    public static JMM_DoubleCheckedLocking getBroken() {
        if (brokenInstance == null) {          // Check 1 (no lock)
            synchronized (JMM_DoubleCheckedLocking.class) {
                if (brokenInstance == null) {   // Check 2 (with lock)
                    brokenInstance = new JMM_DoubleCheckedLocking();
                    // new JMM_DoubleCheckedLocking() does:
                    //   1. Allocate memory for object
                    //   2. Write reference to brokenInstance ← STEP 2
                    //   3. Initialize object (constructor)   ← STEP 3
                    //
                    // JIT can REORDER steps 2 and 3:
                    //   1. Allocate memory
                    //   2. Write reference  ← brokenInstance != null NOW
                    //   3. Initialize object ← NOT DONE YET
                    //
                    // Thread B does Check 1:
                    //   brokenInstance != null → returns it!
                    //   But constructor not yet run → partially constructed!
                }
            }
        }
        return brokenInstance; // may return half-constructed object
    }

    // ── VERSION 2: FIXED — with volatile ──────────────────────────
    private static volatile JMM_DoubleCheckedLocking fixedInstance;
    //                ↑
    //                volatile prevents the reordering of steps 2 and 3
    //                volatile write acts as StoreStore fence:
    //                  Constructor MUST complete before reference is written
    //                  Steps 2 and 3 CANNOT be reordered

    public static JMM_DoubleCheckedLocking getFixed() {
        if (fixedInstance == null) {           // Check 1: no lock (fast path)
            synchronized (JMM_DoubleCheckedLocking.class) {
                if (fixedInstance == null) {    // Check 2: with lock (slow path)
                    fixedInstance = new JMM_DoubleCheckedLocking();
                    // volatile write:
                    //   Constructor GUARANTEED to complete BEFORE
                    //   reference is written to fixedInstance
                    //   StoreStore fence prevents reordering
                }
            }
        }
        return fixedInstance; // guaranteed fully constructed ✓
    }

    // ── VERSION 3: CLEANEST — Initialization-on-demand holder ─────
    // No volatile needed — uses classloader guarantee (Rule 4-like)
    private static class Holder {
        // Class loaded lazily (when getClean() first called)
        // Class initialization synchronized by JVM classloader
        // static final = safe publication
        static final JMM_DoubleCheckedLocking INSTANCE =
                new JMM_DoubleCheckedLocking();
    }

    public static JMM_DoubleCheckedLocking getClean() {
        return Holder.INSTANCE;
        // Thread-safe, lazy initialization
        // No volatile, no synchronized needed
        // The classloader handles synchronization
    }
}
```

```
WHY volatile FIXES DCL — MACHINE CODE LEVEL:

WITHOUT volatile:                   WITH volatile:
──────────────────────────          ──────────────────────────────────
new Object():                       new Object():
  1. malloc → address                 1. malloc → address
  2. write ref = address  ┐           2. constructor runs
  3. constructor runs     │ REORDERED    - initialize fields ✓
                          ▼           3. [StoreStore FENCE] ← volatile
  ref != null but         ↓           4. write ref = address
  constructor not done!               ref != null AND
  PARTIALLY CONSTRUCTED               constructor ALWAYS done ✓

THE FENCE FORCES ORDERING:
  StoreStore fence at volatile write means:
  "All stores before me MUST complete before I execute"
  So constructor (store to fields) MUST complete
  before the volatile store of the reference.
  This makes it impossible to see a partially constructed object.
```

---

## JMM in Spring Boot — Real World Applications

```java
// ═══════════════════════════════════════════════════════════════
//  JMM IN SPRING BOOT CONTEXT
//  Understanding JMM is critical for Spring Boot developers
// ═══════════════════════════════════════════════════════════════

// ── SCENARIO 1: Singleton bean shared by 200 request threads ──

// BROKEN: mutable fields without JMM protection
@Service
public class CounterService_BROKEN {
    private int    requestCount = 0;    // ← JMM visibility + atomicity issue
    private String lastRequest  = null; // ← JMM visibility issue

    public void recordRequest(String path) {
        requestCount++;           // read-modify-write — broken
        lastRequest = path;       // visibility — may be stale
    }

    public int getCount()        { return requestCount; } // may be stale
    public String getLast()      { return lastRequest;  } // may be stale
}

// FIXED: proper JMM protection
@Service
public class CounterService_FIXED {

    // AtomicInteger: visibility + atomicity for counter
    private final AtomicInteger requestCount = new AtomicInteger(0);

    // volatile: visibility for single-writer reference
    private volatile String lastRequest = null;

    public void recordRequest(String path) {
        requestCount.incrementAndGet(); // atomic ✓
        lastRequest = path;             // volatile write ✓
    }

    public int    getCount() { return requestCount.get(); } // atomic read ✓
    public String getLast()  { return lastRequest;        } // volatile read ✓
}

// ── SCENARIO 2: Immutable configuration bean ──────────────────

// BEST PRACTICE: immutable Spring beans
@Service
public class AppConfig {

    // All final — set once in constructor
    // final field guarantee: visible to any thread after construction
    // No synchronization needed anywhere
    private final String  dbUrl;
    private final int     maxConnections;
    private final boolean debugMode;

    // Spring injects via constructor (preferred injection)
    public AppConfig(
            @Value("${db.url}")         String  dbUrl,
            @Value("${db.maxConn}")     int     maxConnections,
            @Value("${app.debug}")      boolean debugMode) {

        this.dbUrl          = dbUrl;
        this.maxConnections = maxConnections;
        this.debugMode      = debugMode;
        // Constructor completes → final field fence inserted by JVM
        // All threads reading these fields see correct values
    }

    // Getters only — no setters
    // This bean is IMMUTABLE → 100% thread-safe under JMM
    public String  getDbUrl()         { return dbUrl; }
    public int     getMaxConnections(){ return maxConnections; }
    public boolean isDebugMode()      { return debugMode; }
}

// ── SCENARIO 3: @Async and visibility ─────────────────────────

@Service
public class AsyncService {

    private volatile String status = "PENDING"; // volatile for visibility

    @Async("taskExecutor")
    public CompletableFuture<String> processAsync(String input) {
        // This runs in a thread pool thread
        // ExecutorService.submit() establishes happens-before:
        //   Everything before submit() is visible here
        //   (ExecutorService uses internal synchronization)

        status = "PROCESSING"; // volatile write → visible to all threads
        String result = doHeavyWork(input);
        status = "DONE";       // volatile write → visible to all threads
        return CompletableFuture.completedFuture(result);
    }

    public String getStatus() {
        return status; // volatile read → always fresh
    }

    private String doHeavyWork(String input) {
        return input.toUpperCase();
    }
}
```

---

## JMM Quick-Reference Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════════╗
║                    JMM COMPLETE CHEAT SHEET                             ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  THE THREE PROBLEMS:                                                     ║
║  ─────────────────────────────────────────────────────────────────────  ║
║  VISIBILITY  → write not seen by other thread (CPU cache)               ║
║  ATOMICITY   → multi-step op interrupted (counter++)                    ║
║  ORDERING    → instructions reordered (CPU/compiler optimization)       ║
║                                                                          ║
║  THE FIXES:                                                              ║
║  ─────────────────────────────────────────────────────────────────────  ║
║                                                                          ║
║  ┌────────────────┬───────────────┬───────────────┬───────────────────┐ ║
║  │                │   volatile    │  synchronized │  Atomic*/final    │ ║
║  ├────────────────┼───────────────┼───────────────┼───────────────────┤ ║
║  │ Visibility     │ ✅ Yes        │ ✅ Yes        │ ✅ Yes            │ ║
║  │ Ordering       │ ✅ Yes        │ ✅ Yes        │ ✅ Yes            │ ║
║  │ Atomicity      │ ❌ No         │ ✅ Yes        │ ✅ Yes (CAS)      │ ║
║  │ Mutual Excl.   │ ❌ No         │ ✅ Yes        │ ❌ No             │ ║
║  │ Multi-field    │ ❌ No         │ ✅ Yes        │ ❌ No             │ ║
║  │ Performance    │ 🚀 Fast       │ 🐢 Medium     │ 🚀 Fast           │ ║
║  └────────────────┴───────────────┴───────────────┴───────────────────┘ ║
║                                                                          ║
║  WHEN TO USE WHAT:                                                       ║
║  ─────────────────────────────────────────────────────────────────────  ║
║  volatile:      1 writer, N readers, single assignment (flag/ref)       ║
║  synchronized:  multiple writers, compound ops, multi-field consistency ║
║  AtomicInteger: multiple writers, single variable counter/flag          ║
║  AtomicRef:     multiple writers, single reference replacement          ║
║  final:         immutable after construction — best performance          ║
║                                                                          ║
║  SAFE PUBLICATION PATTERNS:                                              ║
║  ─────────────────────────────────────────────────────────────────────  ║
║  1. static final field (class loading guarantee)                        ║
║  2. volatile reference field                                            ║
║  3. synchronized get/set methods on same lock                           ║
║  4. final fields in immutable object                                    ║
║                                                                          ║
║  HAPPENS-BEFORE QUICK RULES:                                             ║
║  ─────────────────────────────────────────────────────────────────────  ║
║  synchronized unlock → lock (same monitor)                              ║
║  volatile write → volatile read (same variable)                         ║
║  start() → thread body                                                  ║
║  thread body → join() returns                                           ║
║  constructor end → finalizer start                                      ║
║  A hb B AND B hb C → A hb C (transitivity)                              ║
║                                                                          ║
║  THE GOLDEN RULE:                                                        ║
║  ─────────────────────────────────────────────────────────────────────  ║
║  Shared + Mutable + Concurrent + No-Sync = DATA RACE = BUG             ║
║  Remove ANY one of these four = no race                                 ║
╚══════════════════════════════════════════════════════════════════════════╝


THE ONE MENTAL MODEL TO RULE THEM ALL:
════════════════════════════════════════════════════════════════════

  JMM is the rulebook.
  Happens-before is the guarantee mechanism.
  Synchronization actions are the tools.

  WITHOUT following the rulebook:
    CPU caches lie to you (visibility)
    Instructions run out of order (ordering)
    Multi-step ops get interrupted (atomicity)
    Your program is broken in ways you cannot reproduce

  WITH following the rulebook:
    JVM inserts the right memory barriers
    CPU caches are synchronized at the right moments
    Instructions are ordered as you intend
    Your program behaves correctly on ALL hardware


THE ONE SENTENCE TO REMEMBER:

  "The Java Memory Model is the formal contract between
   the developer and the JVM that defines exactly when
   a thread's write to shared memory becomes visible
   to other threads — and the answer is always:
   only when a happens-before relationship exists,
   established by synchronized, volatile, start(),
   join(), or their transitivity."
```