# CPU Caching and Reordering — Complete Tutorial

---
## The Story Before Everything Else

```
It is 1985.
A CPU runs at 5 MHz.
RAM runs at 5 MHz.
They are equal in speed.
CPU asks for data. RAM delivers instantly.
Life is simple.

Fast forward to 2024.
A modern CPU runs at 4,000 MHz (4 GHz).
RAM still delivers data at roughly the same latency as 1985
(measured in nanoseconds — maybe 5x faster in absolute terms,
but CPU got 800x faster).

The gap:
  CPU can execute 1 instruction in 0.25 nanoseconds.
  RAM takes 60-100 nanoseconds to deliver data.
  That is 240 to 400 CPU cycles of WAITING.

  CPU: "Give me variable x."
  RAM: [240 cycles of silence]
  RAM: "Here is x."
  CPU: "I could have done 240 other instructions while waiting."

This gap is called the MEMORY WALL.
It would make modern CPUs useless if not addressed.

The solution: CPU CACHES.
Small, fast memory sitting between CPU and RAM.
Close to the CPU core. Built on the same chip.
Holds recently used data.

The problem: multiple CPU cores have separate caches.
They can disagree about the current value of a variable.
And CPUs can reorder operations to fill the waiting time.

These two things — caching and reordering —
are the root cause of every concurrency bug
at the hardware level.

This tutorial explains both from the ground up.
```
---
## Part 1 — CPU Caches

### The Cache Hierarchy
```
MODERN CPU MEMORY HIERARCHY
════════════════════════════════════════════════════════════════════

   ┌─────────────────────────────────────────────────────────┐
   │                     CPU CHIP                            │
   │                                                         │
   │  ┌──────────────────┐      ┌──────────────────┐        │
   │  │    CORE 0        │      │    CORE 1        │        │
   │  │                  │      │                  │        │
   │  │  ┌────────────┐  │      │  ┌────────────┐  │        │
   │  │  │ Registers  │  │      │  │ Registers  │  │        │
   │  │  │ ~16 slots  │  │      │  │ ~16 slots  │  │        │
   │  │  │ 1 cycle    │  │      │  │ 1 cycle    │  │        │
   │  │  └─────┬──────┘  │      │  └─────┬──────┘  │        │
   │  │        │         │      │        │         │        │
   │  │  ┌─────▼──────┐  │      │  ┌─────▼──────┐  │        │
   │  │  │ L1 Cache   │  │      │  │ L1 Cache   │  │        │
   │  │  │ 32 KB      │  │      │  │ 32 KB      │  │        │
   │  │  │ 4 cycles   │  │      │  │ 4 cycles   │  │        │
   │  │  │ PRIVATE    │  │      │  │ PRIVATE    │  │        │
   │  │  └─────┬──────┘  │      │  └─────┬──────┘  │        │
   │  │        │         │      │        │         │        │
   │  │  ┌─────▼──────┐  │      │  ┌─────▼──────┐  │        │
   │  │  │ L2 Cache   │  │      │  │ L2 Cache   │  │        │
   │  │  │ 256 KB     │  │      │  │ 256 KB     │  │        │
   │  │  │ 12 cycles  │  │      │  │ 12 cycles  │  │        │
   │  │  │ PRIVATE    │  │      │  │ PRIVATE    │  │        │
   │  │  └─────┬──────┘  │      │  └─────┬──────┘  │        │
   │  └────────┼─────────┘      └────────┼─────────┘        │
   │           │                         │                   │
   │  ┌────────▼─────────────────────────▼──────────┐       │
   │  │              L3 Cache (LLC)                  │       │
   │  │              8-32 MB                         │       │
   │  │              30-40 cycles                    │       │
   │  │              SHARED between cores            │       │
   │  └──────────────────────┬───────────────────────┘       │
   └─────────────────────────┼───────────────────────────────┘
                             │  Memory Bus
                             │  ~100-300 cycles
                             ▼
   ┌─────────────────────────────────────────────────────────┐
   │                   MAIN MEMORY (RAM)                     │
   │                   8-64 GB                               │
   │                   ~100 ns latency                       │
   └─────────────────────────────────────────────────────────┘


SPEED COMPARISON TABLE:
════════════════════════════════════════════════════════════════

  ┌─────────────────┬──────────────────┬────────────────┬───────────┐
  │ Memory Level    │ Access Time      │ Size           │ Per Core? │
  ├─────────────────┼──────────────────┼────────────────┼───────────┤
  │ Register        │ 1 cycle (0.25ns) │ ~bytes         │ Yes       │
  │ L1 Cache        │ 4 cycles (1ns)   │ 32 KB          │ Yes       │
  │ L2 Cache        │ 12 cycles (3ns)  │ 256 KB         │ Yes       │
  │ L3 Cache        │ 30-40 cycles     │ 8-32 MB        │ Shared    │
  │ Main Memory     │ 100-300 cycles   │ 8-64 GB        │ Shared    │
  │ SSD             │ 100,000+ cycles  │ TB             │ Shared    │
  └─────────────────┴──────────────────┴────────────────┴───────────┘

  L1 cache hit: 4 cycles
  RAM access:   300 cycles
  RAM is 75x SLOWER than L1 cache

  A program that fits in L1/L2 cache runs
  75x faster than one that constantly hits RAM.
  This is why cache-friendly code matters enormously.
```

### What is a Cache Line

```
CPUs do NOT load individual bytes from memory.
They load data in chunks called CACHE LINES.

Cache Line Size:
  Modern CPUs: 64 bytes per cache line
  (has been 64 bytes since ~2000)

When CPU reads variable x (4 bytes):
  It loads x PLUS the surrounding 60 bytes
  into the cache line.

Why: Spatial Locality principle
  "If you accessed address N, you will probably
   access addresses near N soon."
  Arrays, object fields, local variables —
  all tend to be spatially close.
  Loading 64 bytes at once is much more efficient
  than loading 4 bytes N separate times.

CACHE LINE PICTURE:
════════════════════════════════════════════════════════════════

  Memory:
  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  │    │    │ x  │    │    │    │ y  │    │    │    │    │    │
  │    │    │    │    │    │    │    │    │    │    │    │    │
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  │◄──────────────── 64 bytes (one cache line) ────────────────►│

  CPU reads x → loads entire 64-byte cache line into L1 cache
  CPU then reads y → ALREADY IN CACHE (y is in same cache line)
  CACHE HIT — no need to go to RAM

  If x and y were in DIFFERENT cache lines:
  CPU reads x → loads cache line 1 (64 bytes)
  CPU reads y → loads cache line 2 (64 bytes) — separate RAM access
```
### How Cache Lines Cause the Visibility Problem

```java
// ═══════════════════════════════════════════════════════════════
//  HOW CACHE LINES CAUSE VISIBILITY PROBLEMS
// ═══════════════════════════════════════════════════════════════
public class CacheLineVisibility {

    // Both counter and flag are likely in the SAME cache line
    // (they are adjacent fields in the same object)
    static int     counter = 0;
    static boolean flag    = false;

    public static void main(String[] args) throws InterruptedException {

        Thread writer = new Thread(() -> {
            counter = 42;     // W1: write counter
            flag    = true;   // W2: write flag
            // HARDWARE REALITY:
            // Core 0 writes to its L1 cache copy of the cache line
            // containing both counter and flag.
            // This modified cache line is NOT immediately
            // written back to L3/RAM.
            // Core 0's L1 says: counter=42, flag=true
            // Core 1's L1 says: counter=0,  flag=false  ← STALE
        });

        Thread reader = new Thread(() -> {
            while (!flag) { }         // spinning — reads from Core 1's L1
                                      // Core 1's L1 has flag=false
                                      // It never sees flag=true
                                      // because Core 0's cache line
                                      // modification hasn't propagated
            System.out.println("counter = " + counter);
            // Even if flag becomes visible:
            // counter may still be 0 in Core 1's cache
            // Core 1 might have loaded the cache line at different time
        });

        reader.start();
        Thread.sleep(100);
        writer.start();
        writer.join();
        reader.join();
    }
}
```

```
CACHE STATE DURING EXECUTION:

INITIAL STATE:
  Core 0 L1 Cache:                Core 1 L1 Cache:
  ┌────────────────────────┐      ┌────────────────────────┐
  │ Cache line 0xABC0:     │      │ Cache line 0xABC0:     │
  │   counter = 0          │      │   counter = 0          │
  │   flag    = false      │      │   flag    = false      │
  │   State: SHARED        │      │   State: SHARED        │
  └────────────────────────┘      └────────────────────────┘
                    Main Memory (RAM):
                    ┌────────────────────────┐
                    │ Cache line 0xABC0:     │
                    │   counter = 0          │
                    │   flag    = false      │
                    └────────────────────────┘

AFTER writer writes counter=42, flag=true:
  Core 0 L1 Cache:                Core 1 L1 Cache:
  ┌────────────────────────┐      ┌────────────────────────┐
  │ Cache line 0xABC0:     │      │ Cache line 0xABC0:     │
  │   counter = 42  ← NEW  │      │   counter = 0   ← STALE│
  │   flag    = true← NEW  │      │   flag    = false←STALE│
  │   State: MODIFIED      │      │   State: INVALID/STALE │
  └────────────────────────┘      └────────────────────────┘
                    Main Memory (RAM):
                    ┌────────────────────────┐
                    │ Cache line 0xABC0:     │
                    │   counter = 0  ← OLD   │
                    │   flag    = false←OLD  │
                    └────────────────────────┘
                    (not yet written back from Core 0)

WITHOUT SYNCHRONIZATION:
  Core 0's changes stay in its L1/L2 cache.
  Core 1 reads from its own stale cache.
  Core 1's reader never sees flag=true.
  PROGRAM HANGS or reads wrong values.
```
### The MESI Cache Coherence Protocol

```
How CPUs manage cache consistency:
MESI Protocol (Modified, Exclusive, Shared, Invalid)

Every cache line has a STATE:

╔═══════════════╦══════════════════════════════════════════════════╗
║ STATE         ║ MEANING                                          ║
╠═══════════════╬══════════════════════════════════════════════════╣
║ M (Modified)  ║ Cache line was modified.                         ║
║               ║ Only this cache has the valid copy.              ║
║               ║ Main memory is STALE — must write back.          ║
╠═══════════════╬══════════════════════════════════════════════════╣
║ E (Exclusive) ║ Cache line is clean, only in this cache.         ║
║               ║ No other core has it. Main memory is up-to-date. ║
╠═══════════════╬══════════════════════════════════════════════════╣
║ S (Shared)    ║ Cache line exists in multiple caches.            ║
║               ║ All copies are CLEAN (same as main memory).      ║
║               ║ Can be read freely. Must invalidate others        ║
║               ║ before writing.                                  ║
╠═══════════════╬══════════════════════════════════════════════════╣
║ I (Invalid)   ║ Cache line is not valid / not present.           ║
║               ║ Must fetch from another cache or main memory.    ║
╚═══════════════╩══════════════════════════════════════════════════╝


MESI STATE TRANSITIONS — write to shared variable:
════════════════════════════════════════════════════════════════

STEP 1: Both cores have counter=0 in SHARED state
  Core 0 L1: [counter=0, state=S]
  Core 1 L1: [counter=0, state=S]
  Main Mem:  [counter=0]

STEP 2: Core 0 writes counter=42
  Core 0 sends "INVALIDATE 0xABC0" message on memory bus
  ↓
  Core 1 receives message → marks its cache line as INVALID (I)
  ↓
  Core 0 updates its copy → state becomes MODIFIED (M)

  Core 0 L1: [counter=42, state=M]   ← has the only valid copy
  Core 1 L1: [counter=?,  state=I]   ← invalid, must fetch
  Main Mem:  [counter=0]             ← stale, not yet written back

STEP 3: Core 1 tries to read counter
  Core 1 L1: INVALID → cache miss
  Core 1 requests from memory bus
  Core 0 "snoops" the bus — sees request for its Modified line
  Core 0 writes its Modified line back to main memory
  Core 0's state: M → S
  Core 1 loads fresh data from main memory
  Core 1's state: I → S

  Core 0 L1: [counter=42, state=S]   ← written back to main mem
  Core 1 L1: [counter=42, state=S]   ← loaded fresh value
  Main Mem:  [counter=42]            ← updated

THE PROBLEM:
  Steps 2 and 3 are NOT instantaneous.
  The invalidation message takes time to propagate.
  Core 1 can read counter BEFORE the invalidation arrives.
  It reads the STALE value from its SHARED state cache line.
  This is the visibility window — the root of the problem.

  volatile and synchronized force the invalidation/write-back
  to happen at a SPECIFIC POINT — closing the visibility window.
```
### False Sharing — The Performance Killer
```java
// ═══════════════════════════════════════════════════════════════
//  FALSE SHARING — when unrelated variables share a cache line
//  and KILL performance
// ═══════════════════════════════════════════════════════════════
public class FalseSharing {

    // ── THE FALSE SHARING PROBLEM ──────────────────────────────────
    // counter1 and counter2 are independent.
    // Thread 1 ONLY touches counter1.
    // Thread 2 ONLY touches counter2.
    // They should not interfere with each other.
    // BUT they are adjacent in memory — SAME CACHE LINE!

    static class BadCounters {
        volatile long counter1 = 0; // Thread 1 writes this
        volatile long counter2 = 0; // Thread 2 writes this
        // Both are 8 bytes each = 16 bytes total
        // Both fit in ONE 64-byte cache line
        // Writing counter1 INVALIDATES counter2's cache line on other core
        // Even though Thread 2 never touched counter1!
    }

    // ── WHAT HAPPENS ─────────────────────────────────────────────
    // Thread 1 writes counter1:
    //   → Core 0 L1: [counter1=N+1, counter2=0, state=M]
    //   → Sends INVALIDATE to Core 1
    //   → Core 1 L1: [counter1=?, counter2=0, state=I] ← INVALIDATED!
    //
    // Thread 2 wants to write counter2:
    //   → Core 1's cache line is INVALID (invalidated by Thread 1)
    //   → Core 1 must FETCH the cache line from Core 0 or memory
    //   → This is a CACHE MISS — expensive!
    //   → Core 1 fetches, writes counter2, marks as Modified
    //   → Sends INVALIDATE to Core 0
    //   → Core 0's cache line invalidated!
    //
    // Thread 1 tries to increment counter1 again:
    //   → Cache miss! Must fetch from Core 1.
    //   → Cache line ping-pongs between cores!
    //
    // Result: performance of two counter increments is WORSE
    //         than a single-threaded version!
    //         Threads are "sharing" cache lines they don't need to.
    //         Hence: FALSE sharing.

    // ── BENCHMARK: With vs Without False Sharing ──────────────────
    static long ITERATIONS = 500_000_000L;

    static void withFalseSharing() throws InterruptedException {
        BadCounters counters = new BadCounters();

        Thread t1 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) {
                counters.counter1++;
            }
        });

        Thread t2 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) {
                counters.counter2++;
            }
        });

        long start = System.currentTimeMillis();
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("With false sharing: "
                + (System.currentTimeMillis() - start) + "ms");
        // Typical: ~5000ms
    }

    // ── FIX 1: Padding — manually add bytes to push to separate lines
    static class PaddedCounters {
        volatile long counter1 = 0;
        // Padding: 7 longs × 8 bytes = 56 bytes + 8 bytes counter1 = 64 bytes
        // Forces counter1 to occupy its own cache line
        long p1, p2, p3, p4, p5, p6, p7; // padding

        volatile long counter2 = 0;
        // counter2 now starts at the next cache line
        long p8, p9, p10, p11, p12, p13, p14; // padding
    }

    // ── FIX 2: @Contended annotation (Java 8+) ────────────────────
    // Tells JVM to pad this field to its own cache line
    // Must run with JVM flag: -XX:-RestrictContended
    static class ContendedCounters {
        @jdk.internal.vm.annotation.Contended
        volatile long counter1 = 0; // JVM pads this to its own 64-byte line

        @jdk.internal.vm.annotation.Contended
        volatile long counter2 = 0; // JVM pads this to its own 64-byte line
    }

    // ── FIX 3: LongAdder (Java 8) — striped per-thread cells ──────
    // LongAdder uses @Contended internally on its Cell class
    // Best solution for high-contention counters
    static java.util.concurrent.atomic.LongAdder adder1
            = new java.util.concurrent.atomic.LongAdder();
    static java.util.concurrent.atomic.LongAdder adder2
            = new java.util.concurrent.atomic.LongAdder();
}
```

```
FALSE SHARING — CACHE LINE PING-PONG PICTURE:
════════════════════════════════════════════════════════════════

  counter1 and counter2 in SAME cache line:

  Core 0 writes counter1:         Core 1 writes counter2:
  ┌──────────────────────┐        ┌──────────────────────┐
  │ [c1=N, c2=0, M]      │        │ [INVALID]            │
  └──────────────────────┘        └──────────────────────┘
          │ invalidate                      │
          └─────────────────────────────────┘
                    Core 1 MISS → fetch
  ┌──────────────────────┐        ┌──────────────────────┐
  │ [c1=N, c2=0, S]      │        │ [c1=N, c2=N, M]      │
  └──────────────────────┘        └──────────────────────┘
          │ invalidated by Core 1
  ┌──────────────────────┐        ┌──────────────────────┐
  │ [INVALID]            │        │ [c1=N, c2=N, M]      │
  └──────────────────────┘        └──────────────────────┘
  Core 0 MISS → fetch ← cache line bounces back and forth
  Every write = cache miss on other core = SLOW!

  SAME cache line + different writers = FALSE SHARING
  = performance disaster

  SEPARATE cache lines + different writers:
  ┌──────────────────────┐        ┌──────────────────────┐
  │ [c1=N, state=M]      │        │ [c2=N, state=M]      │
  └──────────────────────┘        └──────────────────────┘
  Each core modifies ITS OWN cache line
  No invalidation traffic between cores = FAST!
```
---
## Part 2 — CPU Reordering

### Why CPUs Reorder Instructions

```css
CPUs are pipelined.
While one instruction executes, the next is already being fetched.
Modern CPUs have pipelines 15-25 stages deep.

If instruction N has to wait for data from RAM:
  Without reordering: entire pipeline stalls for 300 cycles
  With reordering:    CPU finds independent instruction N+5
                      executes N+5 while waiting for N's data
                      Hides the memory latency

Example:
  1. load  x from memory  ← takes 100 cycles (cache miss)
  2. add   x, 1           ← depends on step 1 — must wait
  3. store result to y    ← depends on step 2 — must wait
  4. load  z from memory  ← INDEPENDENT of 1,2,3!
  5. multiply z by 2      ← depends on step 4

  Without reordering:   1 → (wait) → 2 → 3 → 4 → (wait) → 5
  With reordering:      1 → (while waiting) → 4 → 2 → 3 → 5
                        Steps 4 and 5 overlap with step 1's wait
                        Better CPU utilization!

The CPU keeps a "to-do list" (reorder buffer).
It executes things out of order but COMMITS results in order.
Within ONE thread: the result is identical to in-order execution.
Across threads: the out-of-order execution is VISIBLE.
```
### The Four Types of Reordering
```c
FOUR TYPES OF MEMORY OPERATION REORDERING:
════════════════════════════════════════════════════════════════

  W = Write (Store)    R = Read (Load)

  ┌──────────────────────┬────────────────────────────────────────┐
  │ REORDERING TYPE      │ DESCRIPTION                            │
  ├──────────────────────┼────────────────────────────────────────┤
  │ LoadLoad             │ Read followed by Read                  │
  │ R1 → R2 becomes R2→R1│ Second read moved before first        │
  │                      │ Rare on x86, common on ARM/POWER       │
  ├──────────────────────┼────────────────────────────────────────┤
  │ StoreStore           │ Write followed by Write                │
  │ W1 → W2 becomes W2→W1│ Second write made visible before first │
  │                      │ Possible on ARM/POWER                  │
  ├──────────────────────┼────────────────────────────────────────┤
  │ LoadStore            │ Read followed by Write                 │
  │ R → W becomes W → R  │ Write becomes visible before the read  │
  │                      │ Rare, hardware dependent               │
  ├──────────────────────┼────────────────────────────────────────┤
  │ StoreLoad  ← MOST    │ Write followed by Read                 │
  │ COMMON AND DANGEROUS │ Read can see value from BEFORE write   │
  │ W → R becomes R → W  │ Very common on x86, ARM, POWER        │
  └──────────────────────┴────────────────────────────────────────┘

  StoreLoad reordering is the most common and dangerous:
    Thread A writes x=1, then reads y
    Thread B writes y=1, then reads x
    Both reads may return 0 even though both writes happened!
```

### Three Sources of Reordering in Java

```java
// ═══════════════════════════════════════════════════════════════
//  THREE SOURCES OF REORDERING
// ═══════════════════════════════════════════════════════════════
public class ThreeSourcesOfReordering {

    // ── SOURCE 1: COMPILER REORDERING ─────────────────────────────
    // javac and JIT rearrange instructions
    // Visible in bytecode and native code — not in source

    static int a = 0, b = 0;

    static void compilerReorderingExample() {
        // Source code (what you write):
        a = 1;   // store a
        b = 2;   // store b — independent of a

        // Compiler MAY generate:
        b = 2;   // compiler reordered — no data dependency
        a = 1;   //

        // On single thread: IDENTICAL result
        // On multi-thread: another thread may see b=2 but a=0
        // even though source code writes a before b
    }

    // ── SOURCE 2: CPU OUT-OF-ORDER EXECUTION ──────────────────────
    // CPU hardware executes instructions out of program order
    // to avoid pipeline stalls and memory latency

    // This is HARDWARE-level — JVM cannot fully control it
    // JVM inserts MEMORY BARRIERS to prevent specific reorderings
    // Memory barriers are CPU instructions that enforce ordering

    // SOURCE 3: STORE BUFFER REORDERING ───────────────────────────
    // Each CPU core has a STORE BUFFER between the core and cache
    // Writes go to store buffer first, drained to cache asynchronously

    // STORE BUFFER PURPOSE:
    //   Core wants to write x=42
    //   Cache line containing x may be SHARED by other cores
    //   Must wait for INVALIDATE acknowledgements from other cores
    //   While waiting: core puts write in store buffer
    //   Core continues executing other instructions
    //   When invalidation confirmed: store drained from buffer to cache

    // STORE BUFFER REORDERING:
    //   Thread A:
    //     STORE x=1 → goes to store buffer (not yet in cache)
    //     LOAD  y   → reads directly from cache (bypasses store buffer)
    //                 may see OLD value of y even though x=1 was "written"
    //
    //   This creates StoreLoad reordering:
    //   From another thread's perspective, the load of y
    //   happened BEFORE the store of x
}
```
### The Store Buffer — Deep Dive
```css
THE STORE BUFFER — the #1 source of reordering:
════════════════════════════════════════════════════════════════

  Core Hardware:
  ┌─────────────────────────────────────────────────────────────┐
  │                    CPU CORE                                 │
  │                                                             │
  │  ┌─────────────────┐    ┌──────────────────────────────┐   │
  │  │  Execution      │    │       STORE BUFFER           │   │
  │  │  Unit           │───►│  [addr: x, value: 42]        │   │
  │  │                 │    │  [addr: y, value: 99]        │   │
  │  │  LOAD  x ──────►│    │  [addr: z, value: 7 ]        │   │
  │  │  (checks store  │    └──────────────┬───────────────┘   │
  │  │   buffer first) │                   │ drain              │
  │  └─────────────────┘                   │ asynchronously     │
  └───────────────────────────────────────┼─────────────────────┘
                                          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                      L1 CACHE                               │
  │  [x=0]  [y=0]  [z=0]  ← values BEFORE store buffer drains  │
  └─────────────────────────────────────────────────────────────┘

  SCENARIO: Thread A on Core 0, Thread B on Core 1
  Both variables x and y start at 0.

  Thread A (Core 0):               Thread B (Core 1):
  ─────────────────                ─────────────────
  STORE x = 1                      STORE y = 1
    → goes to store buffer           → goes to store buffer
  LOAD  y                          LOAD  x
    → L1 cache: y = 0 ← STALE!      → L1 cache: x = 0 ← STALE!
    → store buffer not yet drained   → store buffer not yet drained

  Result: Thread A reads y=0, Thread B reads x=0
  Even though BOTH wrote before reading!
  This is the STORE-LOAD reordering caused by store buffer.

  Result = 0,0 — seems impossible without reordering.
  But it DOES HAPPEN on real hardware.

  MEMORY BARRIER fixes this:
  Thread A:                        Thread B:
    STORE x = 1                      STORE y = 1
    [SFENCE/MFENCE]                  [SFENCE/MFENCE]
    ↑ forces store buffer drain      ↑ forces store buffer drain
    LOAD y → sees 1 ✓                LOAD x → sees 1 ✓
```
### Memory Barriers — How Java Prevents Reordering
```java
// ═══════════════════════════════════════════════════════════════
//  MEMORY BARRIERS — how JVM prevents specific reorderings
// ═══════════════════════════════════════════════════════════════
public class MemoryBarriers {

    // Memory barriers (fences) are CPU instructions
    // that prevent specific types of reordering.
    //
    // YOU never write memory barriers in Java.
    // JVM inserts them when you use:
    //   volatile, synchronized, AtomicXxx
    //
    // But understanding them explains WHY these tools work.

    // ── FOUR TYPES OF MEMORY BARRIERS ─────────────────────────────
    //
    // LoadLoad  barrier:
    //   No load before the barrier can be reordered
    //   with any load after the barrier.
    //   LOAD A → [LoadLoad] → LOAD B
    //   Guarantees: A loads before B loads.
    //
    // StoreStore barrier:
    //   No store before the barrier can be reordered
    //   with any store after the barrier.
    //   STORE A → [StoreStore] → STORE B
    //   Guarantees: A stored/visible before B stored/visible.
    //
    // LoadStore barrier:
    //   No load before the barrier can be reordered
    //   with any store after the barrier.
    //   LOAD A → [LoadStore] → STORE B
    //   Guarantees: A loaded before B stored.
    //
    // StoreLoad barrier (MFENCE — most expensive):
    //   All stores before are visible before any load after.
    //   Drains store buffer completely.
    //   STORE A → [StoreLoad] → LOAD B
    //   Guarantees: A written and visible before B read.
    //   This is the strongest barrier — equivalent to full fence.

    // ── HOW volatile MAPS TO BARRIERS ─────────────────────────────
    volatile int v = 0;
    int x = 0, y = 0;

    void volatileWriteBarriers() {
        x = 1;          // ordinary store
        y = 2;          // ordinary store
        // [StoreStore barrier] ← JVM inserts here
        v = 42;         // volatile store
        // [StoreLoad barrier]  ← JVM inserts here (most expensive!)
    }
    // StoreStore before volatile write:
    //   x=1 and y=2 CANNOT be reordered to execute AFTER v=42
    //   Their writes are visible BEFORE v=42's write
    //
    // StoreLoad after volatile write:
    //   v=42 write drains the store buffer
    //   Any subsequent read by any thread sees v=42

    void volatileReadBarriers() {
        int local = v;  // volatile load
        // [LoadLoad barrier]   ← JVM inserts here
        // [LoadStore barrier]  ← JVM inserts here
        int a = x;      // ordinary load — sees latest value
        int b = y;      // ordinary load — sees latest value
    }
    // LoadLoad after volatile read:
    //   a=x and b=y CANNOT be reordered to execute BEFORE volatile load
    //   They happen AFTER the volatile load
    //
    // LoadStore after volatile read:
    //   Any stores after cannot be reordered before the volatile load

    // ── HOW synchronized MAPS TO BARRIERS ─────────────────────────
    Object lock = new Object();

    void synchronizedBarriers() {
        synchronized (lock) {
            // ON ACQUIRE (LOCK):
            // [LoadLoad barrier]   ← all subsequent loads are fresh
            // [LoadStore barrier]  ← all subsequent loads+stores ordered

            int read = x;    // fresh read from memory ✓
            x = read + 1;    // write inside sync block

            // ON RELEASE (UNLOCK):
            // [StoreStore barrier] ← all previous stores committed
            // [StoreLoad barrier]  ← store buffer drained completely
        }
        // After exiting: ALL writes inside are visible to next locker
    }

    // ── BARRIER COSTS ──────────────────────────────────────────────
    //
    // StoreLoad (MFENCE) is MOST EXPENSIVE:
    //   Must drain the entire store buffer
    //   Other cores must acknowledge cache invalidations
    //   CPU pipeline may stall
    //   Cost: ~20-100 ns on modern hardware
    //   (vs 1 ns for a regular instruction)
    //
    // This is why volatile reads/writes are slower than plain reads/writes.
    // And why synchronized is slower than volatile.
    // But they are NECESSARY for correctness.
}
```

---
## Part 3 — Caching + Reordering Together

### The Double-Checked Locking Bug — Root Cause
```java
// ═══════════════════════════════════════════════════════════════
//  HOW CACHING + REORDERING COMBINE TO BREAK DCL
//  Root cause analysis at the hardware level
// ═══════════════════════════════════════════════════════════════
public class DCLRootCause {

    static Object instance = null; // non-volatile — BROKEN

    // BROKEN double-checked locking:
    public static Object getBroken() {
        if (instance == null) {              // Check 1 (no lock)
            synchronized (DCLRootCause.class) {
                if (instance == null) {      // Check 2 (with lock)
                    instance = new Object(); // BROKEN!
                }
            }
        }
        return instance;
    }

    // WHY IT BREAKS — machine code level analysis:
    //
    // instance = new Object() compiles to roughly:
    //
    //   1. r1 = allocate_memory(Object.class)  // get address
    //   2. r1.initialize()                     // run constructor — WRITE fields
    //   3. instance = r1                       // publish reference — WRITE address
    //
    // CPU/Compiler can REORDER steps 2 and 3:
    //   1. r1 = allocate_memory(Object.class)
    //   3. instance = r1    ← REORDERED: write address BEFORE initializing!
    //   2. r1.initialize()  ← constructor runs AFTER reference published!
    //
    // Thread B on different core:
    //   Reads instance (from its L1 cache or main memory)
    //   Sees instance != null (step 3 already written and visible)
    //   Returns instance
    //   BUT: r1.initialize() (step 2) NOT YET DONE!
    //   Thread B is working with UNINITIALIZED OBJECT
    //   CRASH or wrong behavior!
    //
    // STORE BUFFER CONTRIBUTION:
    //   Core 0: step 3 (instance=r1) in store buffer, draining to cache
    //   Core 0: step 2 still in execution unit or store buffer too
    //   Core 1: sees instance write from Core 0's store buffer drain
    //   Core 1: doesn't see step 2's writes yet (different order)
    //
    // volatile FIXES THIS:
    //   [StoreStore barrier] before volatile write of instance
    //   Forces: step 2 (initialize) MUST complete and be visible
    //           BEFORE step 3 (write reference) is visible
    //   Prevents the reordering entirely.

    static volatile Object fixedInstance = null; // volatile fixes it

    public static Object getFixed() {
        if (fixedInstance == null) {
            synchronized (DCLRootCause.class) {
                if (fixedInstance == null) {
                    fixedInstance = new Object();
                    // volatile write:
                    // StoreStore barrier before this write
                    // Constructor MUST finish before reference stored
                    // Thread B sees reference → constructor is done ✓
                }
            }
        }
        return fixedInstance;
    }
}
```
### The Classic Dekker's Algorithm — Reordering in Action
```java
// ═══════════════════════════════════════════════════════════════
//  DEKKER'S ALGORITHM BROKEN BY REORDERING
//  Classic proof that hardware reorders across threads
// ═══════════════════════════════════════════════════════════════
public class DekkersReordering {

    // These look like they should prevent simultaneous entry
    // into the critical section.
    static boolean wants1 = false; // Thread 1 wants to enter
    static boolean wants2 = false; // Thread 2 wants to enter

    // Thread 1 code (simplified Dekker):
    static void thread1CriticalSection() {
        wants1 = true;              // "I want to enter"
        while (wants2) { }         // wait if Thread 2 also wants
        // --- CRITICAL SECTION ---
        System.out.println("Thread 1 in critical section");
        // --- END CRITICAL SECTION ---
        wants1 = false;            // "I'm done"
    }

    // Thread 2 code:
    static void thread2CriticalSection() {
        wants2 = true;             // "I want to enter"
        while (wants1) { }        // wait if Thread 1 also wants
        // --- CRITICAL SECTION ---
        System.out.println("Thread 2 in critical section");
        // --- END CRITICAL SECTION ---
        wants2 = false;            // "I'm done"
    }

    // THIS IS BROKEN DUE TO REORDERING!
    //
    // Thread 1 executes:         Thread 2 executes:
    //   wants1 = true;             wants2 = true;
    //   while (wants2) { }         while (wants1) { }
    //
    // Due to StoreLoad reordering:
    //   Thread 1: wants1=true goes to store buffer
    //   Thread 1: reads wants2 BEFORE wants1 store drains
    //   Thread 1: sees wants2=false → enters critical section!
    //
    //   Thread 2: wants2=true goes to store buffer
    //   Thread 2: reads wants1 BEFORE wants2 store drains
    //   Thread 2: sees wants1=false → enters critical section!
    //
    // BOTH threads enter critical section simultaneously!
    // The algorithm that LOOKED correct is BROKEN by reordering.
    //
    // FIX: make wants1 and wants2 volatile
    //      volatile enforces StoreLoad ordering:
    //      write must be visible BEFORE the subsequent read

    static volatile boolean wants1Fixed = false;
    static volatile boolean wants2Fixed = false;
    // volatile writes flush store buffer
    // volatile reads bypass store buffer
    // StoreLoad reordering prevented → algorithm works correctly
}
```

---
## Part 4 — Java's Tools Against Caching and Reordering
```java
// ═══════════════════════════════════════════════════════════════
//  ALL JAVA TOOLS AND HOW THEY ADDRESS CACHING + REORDERING
// ═══════════════════════════════════════════════════════════════
public class JavaConcurrencyTools {

    // ── TOOL 1: volatile ──────────────────────────────────────────
    volatile int v = 0;
    int x = 0;

    void volatileDemo() {
        x = 99;          // ordinary write
        v = 1;           // volatile write

        // WHAT JVM DOES (machine code):
        //   MOV [x], 99
        //   [StoreStore fence] ← x=99 must be written BEFORE v=1
        //   MOV [v], 1         ← volatile write
        //   [StoreLoad fence]  ← store buffer drained (MFENCE on x86)

        // CACHE EFFECT:
        //   The MFENCE forces Core's store buffer to drain to L1 cache
        //   Cache coherence protocol then propagates to other cores
        //   All other cores see v=1 (and x=99) after their volatile read

        // REORDERING PREVENTION:
        //   x=99 cannot move after v=1 (StoreStore fence)
        //   Any load after v=1's MFENCE sees latest values (StoreLoad fence)
    }

    void volatileReadDemo() {
        int local = v;    // volatile read

        // WHAT JVM DOES:
        //   [LoadLoad fence]  ← read below cannot move above
        //   MOV local, [v]   ← volatile read — bypasses store buffer
        //   [LoadStore fence] ← store below cannot move above

        // CACHE EFFECT:
        //   Volatile read bypasses store buffer
        //   Reads from L1 cache (but cache is coherent via MESI)
        //   If another core wrote v and its store drained to L3/RAM:
        //   MESI invalidation ensures this core's cache is refreshed
        int y = x;        // guaranteed to see latest x after volatile read
    }

    // ── TOOL 2: synchronized ──────────────────────────────────────
    Object lock = new Object();

    void synchronizedDemo() {
        synchronized (lock) {
            // ON LOCK ACQUIRE:
            // x86: lock prefix on compare-and-swap instruction
            //      acts as full memory fence
            // JVM: [LoadLoad + LoadStore barriers]
            //      cache is refreshed — all reads see latest values

            int read = x;     // fresh from memory/cache ✓

            // ON LOCK RELEASE:
            // JVM: [StoreStore + StoreLoad barriers]
            //      store buffer drained completely
            //      all writes visible to next thread that acquires lock

            x = read + 1;     // will be visible to next locker ✓
        }
    }

    // ── TOOL 3: AtomicInteger — CAS instruction ───────────────────
    java.util.concurrent.atomic.AtomicInteger atomic
            = new java.util.concurrent.atomic.AtomicInteger(0);

    void atomicDemo() {
        atomic.incrementAndGet();

        // WHAT CPU DOES:
        //   LOCK CMPXCHG [atomic.value], expectedValue, newValue
        //   ↑ LOCK prefix = full memory fence (like MFENCE)
        //   ↑ CMPXCHG = Compare-And-Swap atomic instruction
        //
        // CAS operation:
        //   1. Read current value from memory (not cache)
        //   2. Compare with expected value
        //   3. If equal: write new value atomically
        //   4. If not equal: return false, retry
        //   Steps 1-4 are ONE ATOMIC CPU INSTRUCTION
        //   No other CPU can interrupt between read and write
        //   The LOCK prefix also acts as a memory fence
        //
        // VISIBILITY:
        //   LOCK prefix invalidates other cores' cache lines
        //   for the written address — they must fetch fresh copy
        //
        // REORDERING:
        //   LOCK prefix = full fence — no reordering across it
    }

    // ── TOOL 4: VarHandle (Java 9+) ───────────────────────────────
    // Fine-grained control over memory access modes
    // Access modes map to specific memory barrier combinations

    // Access modes:
    //   PLAIN        — no barriers (same as plain read/write)
    //   OPAQUE       — prevents JIT from caching in register
    //                  no cross-thread ordering guarantee
    //   ACQUIRE      — acts as LoadLoad + LoadStore barrier
    //                  everything after sees latest writes by another
    //                  thread that did a RELEASE write
    //   RELEASE      — acts as StoreStore + LoadStore barrier
    //                  everything before is visible to ACQUIRE reader
    //   VOLATILE     — acts as full fence (same as volatile keyword)
    //   compareAndSet— acts as full fence (same as AtomicXxx.CAS)
}
```

---
## The Complete Picture — Everything Together
```css
WHAT HAPPENS WHEN Thread A writes x=42 and Thread B reads x:
════════════════════════════════════════════════════════════════════

WITHOUT any synchronization:
─────────────────────────────

  Core 0 (Thread A):              Core 1 (Thread B):
  ┌──────────────────────┐        ┌──────────────────────┐
  │ x = 42               │        │ read x               │
  │ → Core 0 registers   │        │ ← Core 1 L1 cache    │
  │ → Core 0 store buffer│        │   x = 0 (stale)      │
  │ → Core 0 L1 cache    │        │   sees 0 ← WRONG      │
  │ (stays here — not    │        └──────────────────────┘
  │  yet propagated)     │
  └──────────────────────┘

  Core 1 may NEVER see x=42.
  JVM is not obligated to flush Core 0's cache.
  Result: visibility bug.

WITH volatile:
──────────────

  Core 0 (Thread A):              Core 1 (Thread B):
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │ [StoreStore fence]                                   │
  │ STORE x = 42  ←─ volatile write                      │
  │ [StoreLoad fence = MFENCE]                           │
  │   ↑ MFENCE drains store buffer                       │
  │   ↑ Sends INVALIDATE to Core 1 for cache line with x │
  │   ↑ Waits for Core 1 to acknowledge INVALIDATE       │
  └──────────────────────────────────────────────────────┘

                    Cache coherence protocol:
                    Core 1 receives INVALIDATE for x's cache line
                    Core 1 marks its cache line as INVALID (I)

  Core 1 (Thread B):
  ┌──────────────────────────────────────────────────────┐
  │ [LoadLoad fence]                                     │
  │ LOAD x  ←── volatile read                            │
  │   ↑ cache line is INVALID → must fetch fresh copy    │
  │   ↑ fetches from Core 0's cache or main memory       │
  │   ↑ sees x = 42 ← CORRECT ✓                         │
  │ [LoadStore fence]                                    │
  └──────────────────────────────────────────────────────┘

  Core 1 sees x=42 because:
    1. MFENCE drained Core 0's store buffer
    2. INVALIDATE message sent to Core 1
    3. Core 1's cache miss → fetches fresh value


CPU REORDERING PREVENTION — volatile fence picture:
════════════════════════════════════════════════════════════════════

  Thread A writes:       Machine code:           Reordering allowed?
  ────────────────       ─────────────────────   ───────────────────
  x = 1;                 STORE [x], 1            Can be reordered ↕
  y = 2;                 STORE [y], 2            Can be reordered ↕
  v = 3; (volatile)      [StoreStore FENCE]      ← FENCE: nothing
                         STORE [v], 3              above can move below
                         [StoreLoad FENCE]        ← FENCE: nothing
  a = b; (read)          LOAD [a], [b]             below can move above

  GUARANTEED ORDER (as seen by any thread):
    x=1 and y=2 HAPPEN BEFORE v=3
    v=3 HAPPENS BEFORE read of a
    Any thread that reads v=3 ALSO sees x=1 and y=2
```

---
## Real-World Impact — Why You MUST Know This
```java
// ═══════════════════════════════════════════════════════════════
//  REAL WORLD BUG CAUSED BY CACHING + REORDERING
//  A Spring Boot service that loses updates silently
// ═══════════════════════════════════════════════════════════════

// BROKEN Service — mutable state without proper JMM protection
@Service
public class SessionManager_BROKEN {

    // Non-volatile map — updates may not be visible to other threads
    private Map<String, Session> sessions = new HashMap<>();

    // Non-volatile counter — atomicity + visibility both broken
    private int activeCount = 0;

    // Called by Thread Pool Thread 1
    public void createSession(String userId) {
        Session s = new Session(userId);
        sessions.put(userId, s);  // HashMap not thread-safe!
        activeCount++;            // read-modify-write — broken
        // BUGS:
        //   1. HashMap.put() can corrupt internal array on concurrent access
        //   2. Other threads may not see the new session (cache not flushed)
        //   3. activeCount++ loses increments (race condition)
        //   4. Reordering: another thread may see activeCount++ 
        //      before sessions.put() is visible
    }

    // Called by Thread Pool Thread 2
    public Session getSession(String userId) {
        return sessions.get(userId);
        // May return null even if createSession was called:
        //   1. Thread 1's sessions.put() in its CPU cache, not visible here
        //   2. Reordering: sessions map reference may appear
        //      before the Session object's fields are initialized
    }

    // Called by Thread Pool Thread 3
    public int getActiveCount() {
        return activeCount; // may be stale — thread's CPU cache
    }
}

// FIXED Service — proper JMM protection
@Service
public class SessionManager_FIXED {

    // ConcurrentHashMap: thread-safe + volatile-like visibility
    private final ConcurrentHashMap<String, Session> sessions
            = new ConcurrentHashMap<>();

    // AtomicInteger: visibility + atomicity
    private final AtomicInteger activeCount = new AtomicInteger(0);

    public void createSession(String userId) {
        Session s = new Session(userId);
        sessions.put(userId, s);     // thread-safe, visible immediately ✓
        activeCount.incrementAndGet(); // atomic + visible ✓
    }

    public Session getSession(String userId) {
        return sessions.get(userId); // always sees latest state ✓
    }

    public int getActiveCount() {
        return activeCount.get();    // volatile read — always fresh ✓
    }
}
```
---
## The Mental Model — Cache Line Journey

```cs
COMPLETE JOURNEY OF A WRITE: x = 42 (with volatile)
════════════════════════════════════════════════════════════════════

  STEP 1: Thread A executes x = 42 (volatile write)
  ──────────────────────────────────────────────────
  Thread A running on Core 0:
  Execution unit computes value 42
    ↓
  [StoreStore fence] — previous stores committed
    ↓
  Store issued to L1 cache
    ↓
  L1 cache: [cache line for x: state=M, value=42]
    ↓
  [StoreLoad fence = MFENCE on x86]
    ↓
  Store buffer drained to L1

  STEP 2: Cache coherence kicks in (MESI)
  ────────────────────────────────────────
  Core 0's cache controller broadcasts on memory bus:
  "I have MODIFIED the cache line containing x"
  "All other cores: INVALIDATE your copies"
    ↓
  Core 1 receives invalidation signal
  Core 1 L1 cache: [cache line for x: state=I (Invalid)]
  Core 1 acknowledges invalidation to Core 0
    ↓
  Core 0 receives acknowledgement
  Core 0: write propagates to L3 cache (shared)
  L3 cache: [cache line for x: state=S, value=42]

  STEP 3: Thread B executes read of x (volatile read)
  ────────────────────────────────────────────────────
  Thread B running on Core 1:
  [LoadLoad fence]
    ↓
  Load request for x issued to L1 cache
  L1 cache: state=I (Invalid) → CACHE MISS
    ↓
  Request forwarded to L3 cache
  L3 cache: state=S, value=42 → CACHE HIT
    ↓
  Cache line loaded into Core 1 L1: [state=S, value=42]
    ↓
  Thread B reads x = 42 ✓ CORRECT!
  [LoadStore fence]


TIMELINE:
══════════════════════════════════════════════════════════════

  Time ──────────────────────────────────────────────────────►

  Core 0:  [write x=42]→[MFENCE]→[INVALIDATE broadcast]
                                          ↓
  Core 1:                        [ACK invalidate]→[MISS]→[LOAD from L3]
                                                          ↓
                                                   [sees x=42] ✓

  WITHOUT volatile:
  Core 0:  [write x=42 to store buffer] (sits there indefinitely)
  Core 1:  [reads x] → L1 hit → x=0 ← STALE! (no INVALIDATE sent)
```

---
## Summary — The Complete Picture
```css
╔═════════════════════════════════════════════════════════════════════════╗
║          CPU CACHING AND REORDERING — COMPLETE CHEAT SHEET              ║
╠═════════════════════════════════════════════════════════════════════════╣
║                                                                         ║
║  WHY CACHES EXIST:                                                      ║
║    CPU (4 GHz) is 300x faster than RAM.                                 ║
║    Without caches: CPU waits 300 cycles for every variable.             ║
║    With caches: most accesses served in 4 cycles.                       ║
║    Cost: each core has its own cache → they can disagree.               ║
║                                                                         ║
║  CACHE LINE:                                                            ║
║    Unit of cache transfer = 64 bytes.                                   ║
║    Reading one byte loads 64 bytes into cache.                          ║
║    False sharing: unrelated vars in same cache line                     ║
║    → writes by different threads invalidate each other's lines.         ║
║                                                                         ║
║  MESI PROTOCOL:                                                         ║
║    Modified / Exclusive / Shared / Invalid.                             ║
║    Write to Shared line → broadcasts INVALIDATE to other cores.         ║
║    Other cores must fetch fresh copy on next read.                      ║
║    Propagation is NOT instant → visibility window.                      ║
║                                                                         ║
║  WHY REORDERING HAPPENS:                                                ║
║    1. Compiler: reorders independent instructions for IPC               ║
║    2. CPU out-of-order execution: fills pipeline gaps                   ║
║    3. Store buffer: writes drain asynchronously                         ║
║    Within one thread: invisible (appears sequential).                   ║
║    Across threads: visible and dangerous.                               ║
║                                                                         ║
║  FOUR REORDERING TYPES:                                                 ║
║    LoadLoad, StoreStore, LoadStore, StoreLoad (most dangerous).         ║
║    StoreLoad: write may not be visible before subsequent read.          ║
║                                                                         ║
║  MEMORY BARRIERS (FENCES):                                              ║
║    CPU instructions that prevent specific reorderings.                  ║
║    LoadLoad, StoreStore, LoadStore, StoreLoad (full/MFENCE).            ║
║    JVM inserts these when you use: volatile, synchronized, Atomic.      ║
║                                                                         ║
║  HOW JAVA TOOLS MAP TO BARRIERS:                                        ║
║  ┌──────────────────┬───────────────────────────────────────────────┐   ║
║  │ volatile WRITE   │ [StoreStore] STORE [StoreLoad]                │   ║
║  │ volatile READ    │ [LoadLoad] LOAD [LoadStore]                   │   ║
║  │ synchronized     │ [LoadLoad+LoadStore] on acquire               │   ║
║  │  ENTER           │ reads fresh from cache                        │   ║
║  │ synchronized     │ [StoreStore+StoreLoad] on release             │   ║
║  │  EXIT            │ drains store buffer, flushes to main memory   │   ║
║  │ AtomicXxx CAS    │ LOCK CMPXCHG = full MFENCE                    │   ║
║  └──────────────────┴───────────────────────────────────────────────┘   ║
║                                                                         ║
║  PRACTICAL RULES:                                                       ║
║    1. Shared mutable variable + multiple threads = use volatile/sync    ║
║    2. volatile = visibility + ordering, NOT atomicity                   ║
║    3. synchronized = visibility + ordering + atomicity + exclusion      ║
║    4. False sharing: put hot vars on separate cache lines (@Contended)  ║
║    5. Immutable objects: no caching/reordering concerns (no writes)     ║
║    6. final fields: special JMM fence at constructor end                ║
║                                                                         ║
║  THE ONE MENTAL MODEL:                                                  ║
║    CPU caches cause VISIBILITY problems (stale reads).                  ║
║    CPU reordering causes ORDERING problems (wrong sequence).            ║
║    volatile/synchronized insert MEMORY BARRIERS that:                   ║
║      → Force cache flushes/invalidations (fix visibility)               ║
║      → Prevent instruction reordering (fix ordering)                    ║
║    Without barriers: JVM and CPU are free to optimize in ways           ║
║    that break multi-threaded code.                                      ║
╚═════════════════════════════════════════════════════════════════════════╝


THE ONE SENTENCE TO REMEMBER:

  "CPU caches cause visibility problems because each core
   has its own private copy of memory that may be stale,
   and CPUs reorder instructions for performance — both are
   legal optimizations for single-threaded code but break
   multi-threaded correctness unless memory barriers
   (inserted by volatile, synchronized, or Atomic operations)
   force cache coherence and instruction ordering at the
   precise points you need them."
```