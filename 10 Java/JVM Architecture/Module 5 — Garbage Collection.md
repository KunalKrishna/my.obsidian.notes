# 1. Fundamentals
These are stable, version-independent concepts (reachability, generations, mark-sweep-compact haven't changed), so there's no outdated-fact risk here — I've saved the version-sensitive material (collectors, `finalize`) for Part 2, where it belongs, and I'll verify current state there. Six notes.

Core Philosophy : **The Weak Generational Hypothesis** - states that **most objects die young**.

---
## GC Basics & Why Automatic Memory Management

**One line:** The JVM automatically reclaims memory from objects you no longer use, so you never manually free memory — eliminating a whole class of bugs.

In languages like C, you allocate and free memory by hand:

```c
// C — manual memory management
int* data = malloc(100 * sizeof(int));  // you allocate
// ... use it ...
free(data);                             // you MUST free it yourself
```

This is error-prone in three classic ways:

- **Memory leak** — you forget to `free()`, memory is never reclaimed.
- **Dangling pointer** — you `free()` too early, then use freed memory → crash/corruption.
- **Double free** — you `free()` twice → corruption.

Java removes the manual step. You allocate with `new`; you **never** free:

```java
public class GCBasics {
    public static void main(String[] args) {
        String s = new String("temp");  // allocated on heap
        s = null;                        // object now unreachable
        // No free() call. The Garbage Collector will reclaim it automatically,
        // whenever it decides to run — you don't control exactly when.
    }
}
```

The **Garbage Collector (GC)** is a background process in the JVM that periodically finds objects no longer in use and reclaims their memory. The trade-off: you gain safety and simplicity, but give up precise control over _when_ memory is freed, and GC work costs some CPU and can pause your application briefly _(pauses covered in Part 2)_.

_Interview Q: "Why does Java have garbage collection?"_ → It automates memory reclamation, preventing manual-management bugs (leaks, dangling pointers, double frees). Trade-off: less control over timing + some runtime overhead.

---
## GC Roots & Reachability 

**One line:** An object is "garbage" if it's **unreachable from any GC Root**. The GC keeps everything reachable and collects everything else. 

The GC doesn't track "is this object still needed?" directly — it can't read your intent. Instead it uses **reachability**: starting from a set of always-alive references called **GC Roots**, it follows every reference chain. Any object it can reach is _live_; anything it can't reach is _garbage_. 

**📍 The Starting Point: GC Roots**
The JVM maintains a list of special, always-alive memory pointers called **GC Roots**. These are the foundational starting points of your entire application. 
![[JVM GC reachability.png|485]]
**GC Roots** are the starting points — references the JVM knows are active :

- **Local variables** on any thread's stack (references in active method frames/ **Stack Frame variables**).
- **Static fields** of loaded classes.
- **Active Thread Objects** themselves.
- **System Classes:** Core classes loaded by the Bootstrap ClassLoader (like `java.lang.Object`)
- **JNI references** from native code.

![[jvm gc root.png|683]]

The algorithm is essentially a graph traversal: mark everything reachable from the roots; whatever's left unmarked is collectible.

🪵 The Tree Analogy (Mark and Sweep)
1. **The Marking Phase:** The JVM starts at these **GC Roots** and follows every single reference path, branch by branch, like tracing a tree from the roots to the leaves. Every object it can touch is marked as **"Alive"**.
2. **The Sweeping Phase:** Any object left on the Heap that could _not_ be reached from any GC Root tree path is flagged as garbage and marked for immediate destruction.

```java
public class Reachability {
    static Object rootObj = new Object();   // reachable via a STATIC field (a GC root)

    public static void main(String[] args) {
        Object local = new Object();        // reachable via a LOCAL variable (a GC root)

        Object a = new Object();
        Object b = new Object();
        a = null;   // the object formerly in 'a' is now unreachable -> garbage
                    // (nothing points to it from any root)

        // 'local', 'rootObj', and 'b' remain reachable -> kept alive
    }
}
```

![[jvm gc Mark and Sweep.gif]]

Crucially, this handles **circular references** correctly — a trap for naive reference-counting schemes:

```java
class Node { Node ref; }

Node x = new Node();
Node y = new Node();
x.ref = y;   // x -> y
y.ref = x;   // y -> x  (they point at each other)
x = null;
y = null;    // both are now unreachable from any ROOT...
             // even though they still reference EACH OTHER.
             // Reachability-based GC correctly collects both.
```

This is why Java uses reachability, not reference counting: two objects pointing at each other but detached from all roots are still garbage, and reachability catches that.

_Interview Q: "How does the JVM decide an object is garbage?"_ → Reachability from GC Roots (stack locals, statics, active threads, JNI refs). Unreachable = garbage. This correctly collects cyclic references, which reference-counting cannot.

---
## Reference Types (Strong, Soft, Weak, Phantom)

**One line:** Four strengths of reference tell the GC how eagerly it may collect the referent — from "never while I hold it" (strong) down to "collect freely" (weak/phantom).

By default every reference you write is **strong**. But `java.lang.ref` lets you deliberately hold an object _weakly_, so the GC can reclaim it under memory pressure. This matters for caches and memory-sensitive structures.

**1. Strong Reference** — the normal kind. As long as a strong reference exists, the object is **never** collected.

```java
Object o = new Object();   // strong reference; o's object is safe from GC
```

**2. Soft Reference** — collected **only when the JVM is low on memory**. Ideal for memory-sensitive caches: keep the object as long as there's room, discard it before an `OutOfMemoryError`.

```java
import java.lang.ref.SoftReference;

SoftReference<byte[]> cache = new SoftReference<>(new byte[10_000_000]);
byte[] data = cache.get();        // returns the array, or null if GC reclaimed it
if (data == null) {
    // it was collected under memory pressure — recompute/reload
}
```

**3. Weak Reference** — collected at the **next GC cycle** once no strong references remain (doesn't wait for memory pressure). The backbone of `WeakHashMap`, where entries vanish once their keys are otherwise unreferenced.

```java
import java.lang.ref.WeakReference;

Object strong = new Object();
WeakReference<Object> weak = new WeakReference<>(strong);
System.out.println(weak.get());   // prints the object (still strongly held by 'strong')

strong = null;                    // remove the only strong reference
System.gc();                      // suggest a GC
System.out.println(weak.get());   // likely null — collected, since only weakly reachable
```

**4. Phantom Reference** — the weakest. `get()` **always returns null**; you can't retrieve the object. It's used purely to get a notification (via a `ReferenceQueue`) _after_ an object has been finalized, enabling safer post-mortem cleanup than the deprecated `finalize()`. Niche — know it exists and that `get()` always returns null.

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);
System.out.println(phantom.get());   // ALWAYS null, by design
// When the referent is reclaimed, 'phantom' is enqueued in 'queue' — your cue to clean up.
```

Strength ladder (strongest → weakest): **Strong → Soft → Weak → Phantom**.

|Type|Collected when|Typical use|
|---|---|---|
|Strong|Never (while ref held)|Normal code|
|Soft|Only under memory pressure|Memory-sensitive caches|
|Weak|Next GC once no strong refs|`WeakHashMap`, canonical maps|
|Phantom|After finalization; `get()`=null|Post-mortem cleanup|

_Interview Q: "Difference between soft and weak references?"_ → Soft: reclaimed only when memory is low (good for caches). Weak: reclaimed at the next GC once no strong refs remain (good for `WeakHashMap`). Phantom is weakest — can't retrieve the object, used for cleanup notification.

---
## Generational GC & the Weak Generational Hypothesis

**One line:** GC splits the heap by object age because most objects die young — so collecting the "young" area frequently and cheaply reclaims the vast majority of garbage.

This is the _reason_ the heap has generations (from Module 3). It rests on an empirical observation called the **Weak Generational Hypothesis**:

> **Most objects die young.** The overwhelming majority of objects become unreachable shortly after creation (loop temporaries, method-local objects, intermediate results). Relatively few live a long time.

A second observation: **old objects rarely reference young ones.**

Given these, collecting the _whole_ heap every time would be wasteful — you'd repeatedly re-scan long-lived objects that are almost never garbage. So the GC divides the heap:

- **Young Generation** (Eden + Survivors) — where objects are born. Collected **frequently** and **quickly**, because it's small and most of it is dead each time.
- **Old Generation** — objects that survived long enough to be **promoted**. Collected **rarely**, because these tend to stay alive.

```
new objects ──▶ EDEN ──(survive)──▶ SURVIVOR ──(survive enough cycles)──▶ OLD GEN
                 │                      │                                    │
            collected often        collected often                    collected rarely
            (Minor GC)             (Minor GC)                          (Major/Full GC)
```

The payoff: because ~90%+ of new objects are already dead by the first collection, a Young GC frees a lot of memory by scanning only a small region — fast and cheap. Long-lived objects get quietly promoted out of the way so they aren't repeatedly re-examined.

```java
public class Generational {
    public static void main(String[] args) {
        // These die almost immediately — classic "young" garbage.
        for (int i = 0; i < 1_000_000; i++) {
            String temp = "value-" + i;   // born in Eden, unreachable next iteration
        }                                  // collected quickly by frequent Minor GCs

        // This lives for the whole program — gets promoted to Old Gen.
        static_cache = new java.util.HashMap<>();
    }
    static java.util.Map<String,String> static_cache;
}
```

_Interview Q: "Why generational garbage collection?"_ → The weak generational hypothesis: most objects die young. Splitting the heap lets the GC collect the small Young gen frequently and cheaply (reclaiming most garbage) while scanning the Old gen rarely.

---
## Minor vs Major vs Full GC

**One line:** Minor GC cleans the Young gen (frequent, fast); Major GC cleans the Old gen; Full GC cleans the entire heap (plus metadata) — the most expensive.

These terms describe _which region_ is being collected. Precise definitions matter because interviewers use them loosely and want to see if _you_ know the distinction.

- **Minor GC (Young GC)** — collects **only the Young Generation** (Eden + Survivors). Triggered when Eden fills up. Frequent but fast, because the region is small and mostly dead. Survivors get moved/promoted; dead objects reclaimed. Involves a brief stop-the-world pause, but a short one.
- **Major GC** — collects the **Old Generation**. Triggered as the Old gen fills (from promotions). Less frequent but slower, because long-lived objects must be examined. _(Note: the term is used a bit inconsistently across sources — some equate "Major" with "Full." What matters is the region: Old gen.)_
- **Full GC** — collects the **entire heap** (Young **+** Old) and typically also cleans up other areas like Metaspace. The most expensive, with the longest stop-the-world pause. Triggered when the Old gen is full, on promotion failure, or when `System.gc()` is called.

```
Minor GC:  [ Eden | S0 | S1 ]                 <- Young only
Major GC:                     [ Old Gen ]      <- Old only
Full GC:   [ Eden | S0 | S1 ] [ Old Gen ] + Metaspace   <- everything
```

**A common performance red flag:** frequent **Full GCs** usually signal a problem — the Old gen filling too fast, often from a memory leak or an undersized heap. This is a classic diagnostic thread interviewers pull on _(troubleshooting is Part 2)_.

Note `System.gc()`:

```java
System.gc();   // a REQUEST for a (typically Full) GC — not a guarantee.
               // The JVM may ignore it. Avoid in production code.
```

Interviewers like to hear that `System.gc()` is only a _hint_, not a command, and that relying on it is an anti-pattern.

_Interview Q: "Minor vs Major vs Full GC?"_ → Minor = Young gen only (frequent, fast); Major = Old gen; Full = entire heap + metadata (rare, slowest, longest pause). Frequent Full GCs suggest a leak or undersized heap.

---
## Mark–Sweep–Compact

**One line:** The core three-phase algorithm underlying GC: **Mark** the live objects, **Sweep** away the dead, **Compact** the survivors to remove fragmentation.

This is the fundamental algorithm most collectors build on. Understand the three phases and _why_ the compaction phase is needed.

**Phase 1 — Mark.** Starting from the GC Roots, traverse all reachable objects and mark them as "live." (This is the reachability scan from the earlier note.)

**Phase 2 — Sweep.** Scan the heap and reclaim the memory of every **unmarked** (unreachable) object, adding it back to the pool of free space.

After just Mark-and-Sweep, memory works but is **fragmented** — free gaps are scattered between surviving objects. A large new object might not fit in any single gap even if total free memory is sufficient:

```
After Mark-Sweep (fragmented):
[ live ][ free ][ live ][ free ][ free ][ live ][ free ]
          ^ scattered gaps — a big object may not fit in any single one
```

![[jvm gc Mark and Sweep 2.gif]]

**Phase 3 — Compact.** Move all surviving objects together to one end of the memory region, consolidating free space into one contiguous block:

```
After Compact (defragmented):
[ live ][ live ][ live ][ ---------- free ---------- ]
                          ^ one large contiguous free block; fast allocation
```

Compaction has a cost — moving objects means updating all references to them — but it makes future allocation trivially fast (just bump a pointer) and prevents the "can't allocate despite free memory" failure.

```java
// Conceptually, across a collection cycle:
// 1. MARK    — walk from GC Roots, flag every reachable object as live
// 2. SWEEP   — free memory of everything not flagged
// 3. COMPACT — slide survivors together so free space is one contiguous block
```

![[Mark–Sweep–Compact.png]]

Different real collectors implement these phases differently (some copy instead of compact, some do phases concurrently), which is exactly the territory of Part 2.

**The 3 Core Algorithm Types** : 
Behind the scenes, different JVM collectors use a combination of three algorithmic steps to physically manipulate computer RAM:
1. **Mark and Sweep:** The GC starts at the roots and **marks** everything reachable as "alive". Then, it walks the entire heap memory floor and **sweeps** (deletes) anything that wasn't marked. _Problem: It leaves holes in memory, creating fragmentations._ 
2. **Mark, Sweep, and Compact:** Same as above, but after sweeping, it slides all surviving objects to the absolute beginning of the memory block. This leaves a massive, continuous open space for new allocations.  
3. **Copying:** The GC splits a zone into two halves, marks alive objects in Half A, and **copies** them cleanly in a neat row directly over to Half B, instantly wiping Half A clean. (This is exactly how Eden and Survivor spaces work).

_Interview Q: "Explain mark-sweep-compact."_ → Mark live objects from GC roots; sweep (free) the unmarked dead; compact survivors together to eliminate fragmentation so allocation stays fast.

### **Q. How is static reference variable (primitive / object)  stored in memory and compare their gc ?**

The core principle is the same: the **reference variable** for a static object behaves exactly like a static primitive because it is attached to the `java.lang.Class` instance. However, there is a massive trap regarding the **actual target object** it points to.
For a static object, memory layout and garbage collection are split into two distinct parts: **The Remote Pointer** and **The Heavy Object Data**.

📊 **1. Memory Space Comparison**
When you declare static fields in a class, the JVM splits their physical memory footprint across the Heap differently based on their data type: 

```java
public static int counter = 42;
public static User admin = new User("Alice");
```

```
[ MAIN JVM HEAP ]
 ├──> [ java.lang.Class Mirror Object ]
 │     ├──> counter: [ 42 ]  (Value is embedded directly inside the Class instance)
 │     └──> admin:   [ 0x7FFA ] (Only a 4/8-byte memory pointer is inside the Class instance)
 │
 └──> [ Independent User Object (0x7FFA) ]
       └──> name: "Alice"    (The actual heavy data sitting out on the open Heap)
```

- **Static Primitive (`counter`):** The actual byte value (`42`) is stored **directly inside** the `java.lang.Class` object layout. It has no external dependencies.
- **Static Object (`admin`):** Only a tiny 4-byte or 8-byte **reference address pointer** is stored inside the `java.lang.Class` object. The physical, heavy payload of that `User` object (along with its fields like `"Alice"`) sits as an independent memory block elsewhere out on the open Heap floor.

🔄 **2. Garbage Collection (GC) Comparison**
This structural split creates a fundamental difference in how the Garbage Collector treats them.

**A. Static Primitive Variable**
- **Normal GC Lifecycle:** The value is completely locked inside the Class object. Normal GC cycles ignore it.
- **The End of Life:** It is removed **only** during **Class Unloading**, when the entire Class object and its Metaspace blueprints are wiped clean. 

**B. Static Object Variable (The Two-Part Lifecycle)**
The **Pointer** and the **Target Object** have completely separate rules:
1. **The Pointer (`admin` variable):** This behaves exactly like a primitive. It is an unmovable anchor attached to the Class instance. It will **only** be garbage collected during Class Unloading. 
2. **The Target Object (`new User()`):** This object is completely exposed to standard, everyday Garbage Collection cycles. Its survival depends entirely on **what the pointer is currently doing**.

🚨 **How the Target Object Dies _Without_ Class Unloading**

While a static primitive can never change its memory slot, a static object pointer can be manipulated by your code. You can kill a static object without unloading the class in two ways:

**Case 1: Reassigning to Null**
If you clear the static anchor, the target object instantly loses its GC Root chain.
```java
DemoClass.admin = null; 
// The pointer inside the Class object becomes null.
// The "Alice" User object has NO roots left. 
// Standard Minor/Major GC will vaporize "Alice" on the next run!
```

**Case 2: Reassigning to a New Object**
If you overwrite the static anchor, the old object becomes an orphan garbage candidate.

```java
DemoClass.admin = new User("Bob");
// The pointer changes to the new "Bob" object address.
// The old "Alice" object is cut loose and will be garbage collected normally.
```

---
📊 Summary Matrix for Interviews

|Feature|Static Primitive Variable|Static Object Variable|
|---|---|---|
|**Pointer Location**|N/A (Value is embedded)|Attached to `java.lang.Class` on Heap|
|**Data Payload Location**|Attached to `java.lang.Class` on Heap|Independent block out on open Heap|
|**When is the Anchor Cleared?**|Only during **Class Unloading**|Only during **Class Unloading**|
|**When is the Data Cleared?**|Only during **Class Unloading**|**Anytime** if reassigned, nullified, or over-written|
|**Risk Category**|No risk (Fixed size footprint)|High risk for **Memory Leaks** if items accumulate|

---

That's Part 1 — the fundamentals. Natural MOC links: **GC Roots** ↔ **Mark-Sweep-Compact** (marking _is_ the reachability scan), **Reference types** ↔ **GC Roots** (they're deliberately _non_-root references), and **Generational** ↔ **Minor/Major/Full** (generations are what those GC types operate on). These all feed forward into Part 2, where the collectors are just different engineering trade-offs on top of these same primitives.

Ready for Part 2 — the collectors (Serial, Parallel, G1, ZGC, Shenandoah, and how CMS fits in historically) plus troubleshooting (leaks, OOM types, `finalize`, and the diagnostic tools). This is the part where version-currency matters most, so I'll verify the current defaults and deprecations before writing. Go ahead?

# 2. Collectors and Troubleshooting

---
## GC Collectors Overview

**One line:** The JVM ships several collectors, each a different trade-off between **throughput** (total work done) and **latency** (pause length). You pick one with a flag; **G1 is the default**.

Java offers multiple collectors because applications have different needs — a batch job wants maximum throughput and doesn't care about pauses, while a trading system needs pauses measured in microseconds. The core tension:
- **Throughput** — total useful work per unit time. High-throughput collectors accept longer, rarer pauses in exchange for doing less total GC overhead.
- **Latency** — length of individual pauses (especially stop-the-world pauses). Low-latency collectors do more work _concurrently_ (alongside your app) to keep pauses tiny, at some throughput cost.

|Collector|Optimizes for|Enable with|
|---|---|---|
|**Serial**|Small heaps, single thread|`-XX:+UseSerialGC`|
|**Parallel** (Throughput)|Maximum throughput|`-XX:+UseParallelGC`|
|**G1** (Garbage-First)|Balance — **default**|`-XX:+UseG1GC`|
|**ZGC**|Ultra-low latency, huge heaps|`-XX:+UseZGC`|
|**Shenandoah**|Ultra-low latency|`-XX:+UseShenandoahGC`|

G1 has been the default since Java 9. Each of the next notes covers one collector. The historical **CMS** collector is covered separately since it's been removed and interviewers use it to check how current you are.

_Interview Q: "What garbage collectors does Java have and how do you choose?"_ → Serial (small/simple), Parallel (throughput), G1 (balanced, default), ZGC/Shenandoah (ultra-low latency). Choice = throughput vs latency trade-off for your workload.

---
## Serial GC

**One line:** The simplest collector — a single thread does all GC work, stopping the entire application while it runs. Best for small heaps and single-core environments.

- Uses **one thread** for collection.
- **Stops the world** for the entire collection — the app is fully paused.
- Minimal overhead and memory footprint precisely because it's so simple.
- Uses mark-sweep-compact (from Part 1) in the Old gen, copying in the Young gen.

**When it fits:** small applications, limited-memory containers, single-core machines, or short-lived JVMs (CLI tools) where GC tuning isn't worth it. It's often the automatic choice in tiny containers.

```bash
java -XX:+UseSerialGC -jar app.jar
```

_Interview Q: "When would you use Serial GC?"_ → Small heaps / single-core / low-overhead scenarios (small containers, CLI tools) where simplicity beats parallelism.

---
## Parallel GC (Throughput Collector)

**One line:** Uses **multiple threads** to do GC work faster, maximizing total throughput. Still stops the world, but finishes the pause quicker by parallelizing.

- Multiple GC threads work simultaneously (hence "parallel").
- Still **stop-the-world** — the app pauses during collection — but the pause is _shorter_ than Serial's because many threads share the work.
- Optimizes for **throughput**: get GC done fast, spend maximum time running your actual code. Doesn't try to minimize individual pause length below what parallelism gives.
- Was the **default before Java 9** (worth knowing as a version fact).

**When it fits:** batch processing, data crunching, and back-end jobs where overall throughput matters more than any single pause — nobody's waiting on a UI, so a longer occasional pause is fine if total work is maximized.

```bash
java -XX:+UseParallelGC -jar app.jar
# Related tuning knobs:
java -XX:+UseParallelGC -XX:MaxGCPauseMillis=200 -XX:GCTimeRatio=99 -jar app.jar
```

_Interview Q: "Parallel vs Serial GC?"_ → Both stop-the-world, but Parallel uses multiple threads for shorter pauses and higher throughput. Parallel suits throughput-oriented batch work; it was the pre-Java-9 default.

---
## G1 GC (Garbage-First) — the default

**One line:** Divides the heap into many equal-size **regions** and collects the ones with the most garbage _first_, working mostly concurrently to hit a **target pause time** you set. The default since Java 9 and the one to know best.

G1 was designed as the balanced successor to CMS. Its key innovations:

**Region-based heap.** Instead of large fixed Young/Old areas, G1 splits the heap into many equal-size regions (typically 1–32 MB each). A region is _dynamically_ labeled Eden, Survivor, or Old — the generational roles still exist, but they're now a set of regions rather than contiguous blocks.

```
Traditional heap:   [ Eden    ][ Survivor ][ ------- Old ------- ]

G1 heap (regions):  [E][O][S][E][O][O][E][S][O][E][O][O][E][S][O] ...
                     each box = one region, role assigned dynamically
```

**Garbage-First strategy.** G1 tracks how much garbage each region holds and collects the **fullest-of-garbage regions first** — that's the "Garbage-First" name. This <mark style="background-color: #1EFF00; color: black">maximizes memory reclaimed per unit of pause time</mark>.

**Predictable pause target.** You tell G1 the pause time you're willing to tolerate, and it collects only as many regions as it can within that budget:

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar
# "keep pauses around 200ms" — G1 sizes its work to try to honor this
```

**Mostly concurrent.** Much of G1's marking happens concurrently with the application; it still has stop-the-world evacuation pauses, but keeps them bounded. It also does **mixed collections** (Young + some Old regions together) to chip away at the Old gen without a giant Full GC.

Recent JDK releases keep improving it — G1 picked up meaningful JDK 25 improvements around remembered-set memory and pause-time spikes — so it remains actively developed, not legacy.

_Interview Q: "How does G1 work / why is it the default?"_ → Region-based heap; collects highest-garbage regions first; targets a configurable max pause (`-XX:MaxGCPauseMillis`); does concurrent marking + mixed collections. It's the default because it balances throughput and latency well for typical server workloads.

---
## ZGC & Shenandoah (Ultra-Low-Latency Collectors)

**One line:** Two collectors engineered for **sub-millisecond pauses even on huge (multi-terabyte) heaps**, by doing almost all work _concurrently_ with the application. Know them at a conceptual level.

For latency-critical systems (trading, real-time services, large caches), even G1's pauses can be too long. ZGC and Shenandoah push nearly all GC work to run _alongside_ the app so stop-the-world pauses stay tiny regardless of heap size.

**ZGC** (`-XX:+UseZGC`):

- Goal: pause times that stay **sub-millisecond**, independent of heap size — scales to terabyte heaps.
- Uses **colored pointers** and **load barriers** to relocate objects concurrently while the app runs.
- Version arc worth knowing: introduced experimentally in Java 11, made production-ready in Java 15, and Java 21 introduced Generational ZGC, which maintains separate generations for young and old objects. As of **Java 23+, generational mode is the default**, so the old `-XX:+ZGenerational` flag is no longer needed.

**Shenandoah** (`-XX:+UseShenandoahGC`):

- Same goal — ultra-low, heap-size-independent pauses — using concurrent evacuation with a technique called Brooks pointers/load barriers.
- Generational Shenandoah was experimental in JDK 24 and is no longer experimental as of Java 25.

The high-level story for an interview: both ZGC and Shenandoah have gone generational, combining ultra-low latency with the efficiency of the generational hypothesis (Part 1). You don't need internals — know that they trade some throughput/CPU and memory overhead for dramatically smaller pauses, and you pick them when latency is the top priority.

```bash
java -XX:+UseZGC -jar app.jar          # ultra-low latency; generational by default on Java 23+
java -XX:+UseShenandoahGC -jar app.jar # ultra-low latency alternative
```

_Interview Q: "When would you use ZGC or Shenandoah?"_ → When you need very low, predictable pauses (sub-millisecond) on large heaps and can trade some throughput for it. Both are concurrent and now generational.

---
## CMS — Historical Context (Deprecated & Removed)

**One line:** CMS (Concurrent Mark Sweep) was an early low-pause collector. It is **deprecated and removed** — G1 replaced it. Know it _only_ as history; never present it as a current option.

This note exists purely to handle the interview trap "tell me about CMS" — the right answer demonstrates you know it's gone and _why_.

**What CMS was:** the Concurrent Mark Sweep collector aimed to reduce Old-gen pause times by doing most of its marking and sweeping **concurrently** with the application — an early attempt at low-latency GC before ZGC/Shenandoah existed.

**Why it was retired:**

- It did **not compact** the Old gen, so it suffered **memory fragmentation** over time, which could eventually force an expensive full stop-the-world compacting collection anyway.
- It was complex and hard to maintain.
- G1 was designed as its successor.

**The version facts (this is what interviewers check):** CMS was deprecated in JDK 11 and removed from JDK 14. As of JDK 14, requesting it with `-XX:+UseConcMarkSweepGC` is ignored with a message that support was removed. The removal (JEP 363) happened because no contributors stepped up to maintain it, while ZGC, Shenandoah, and improvements to G1 had made it safe to drop.

> **How to answer:** "CMS was a concurrent low-pause collector, but it didn't compact so it fragmented, and it was **deprecated in Java 9/11 and removed in Java 14**. Its role was taken over by G1, and for ultra-low latency today you'd use ZGC or Shenandoah." Saying "you'd use CMS for low pauses" as if it's current is the wrong answer that dates your knowledge.

_Interview Q: "What about CMS?"_ → Old concurrent collector; didn't compact (fragmentation); deprecated Java 9/11, removed Java 14; replaced by G1, with ZGC/Shenandoah for modern low-latency needs.

---
## Stop-the-World Pauses

**One line:** A "stop-the-world" (STW) pause is when the JVM **halts all application threads** so the GC can work safely. Every collector has _some_ STW; they differ in how long and how often.
**What is a "Stop-The-World" (STW) pause?** A state where the JVM freezes your application's business threads entirely so the GC can safely move memory objects around without data corruption.

To safely mark reachability or move objects, the GC sometimes needs the object graph to hold still — so it pauses every application thread. During an STW pause, your app does nothing: no requests served, no work done.

- **Serial / Parallel** — the _entire_ collection is STW (Parallel just uses more threads to shorten it).
- **G1** — much marking is concurrent, but evacuation pauses are still STW, bounded by your pause target.
- **ZGC / Shenandoah** — almost everything is concurrent; STW pauses are kept sub-millisecond regardless of heap size.

The reason low-latency collectors exist is entirely to shrink these pauses. A long STW pause in a user-facing service shows up as latency spikes / timeouts; in a game server, as dropped frames.

```
App threads:  ▓▓▓▓▓▓░░░░░░▓▓▓▓▓▓▓▓░░░░░▓▓▓▓▓▓
                    ▲STW▲        ▲STW▲
              ▓ = running   ░ = paused for GC
```

_Interview Q: "What is a stop-the-world pause?"_ → GC halts all app threads to work safely. All collectors have some; low-latency ones (ZGC/Shenandoah) minimize it by doing most work concurrently.

---
## Memory Leaks in Java

**One line:** Yes — Java _can_ leak memory. A leak here means objects stay **reachable** (so GC won't collect them) even though your program no longer needs them. Know the classic causes and be ready with an example.

A common misconception is "Java has GC, so no leaks." Wrong — GC only reclaims _unreachable_ objects. If you accidentally keep a reference alive, the object is reachable forever and accumulates. That's a leak.

The classic causes (memorize a couple to name instantly):

**1. Unbounded static collections** — a `static` collection lives for the whole program (it's a GC root), so anything you add and never remove is retained forever:

```java
public class LeakyCache {
    // static -> always reachable -> nothing added here is ever GC'd
    private static final Map<String, byte[]> CACHE = new HashMap<>();

    public void store(String key, byte[] data) {
        CACHE.put(key, data);   // never removed -> grows without bound -> leak
    }
}
```

**2. Unclosed resources** — streams, connections, sockets not closed hold native memory and objects. Fix with try-with-resources:

```java
// Leak-prone: if an exception occurs, close() may never run
FileInputStream in = new FileInputStream("f");
// ... use in ...

// Correct: try-with-resources auto-closes, even on exception
try (FileInputStream in2 = new FileInputStream("f")) {
    // ... use in2 ...
}   // in2.close() called automatically
```

**3. Listeners/callbacks never unregistered** — you register a listener but never remove it; the publisher keeps a strong reference to your object indefinitely.

**4. `ThreadLocal` misuse** — values not removed on long-lived (pooled) threads stay reachable via the thread. Always `remove()` when done, especially in thread pools.

**5. Classloader leaks** — a lingering reference into an undeployed app's classloader keeps the entire classloader (and all its classes' metadata in Metaspace) alive. Classic in app servers on redeploy, and a cause of `OutOfMemoryError: Metaspace`.

The through-line: every leak is **unintended reachability**. Fixing a leak means finding and cutting the reference chain (often via a heap dump — next notes).

_Interview Q: "Can Java have memory leaks? Give an example."_ → Yes — objects that stay reachable but are no longer needed. Classic: an ever-growing `static` collection, unclosed resources, un-removed listeners, `ThreadLocal` in pooled threads, classloader leaks. GC can't help because the objects are still reachable.

---

## `finalize()` and Modern Cleanup

**One line:** `finalize()` was meant to run cleanup before an object is collected, but it's **deprecated for removal** and should never be used. Use **try-with-resources** (`AutoCloseable`) or **`Cleaner`** instead.

Another version-currency trap. `Object.finalize()` let you write code to run when the GC was about to reclaim an object. It's fundamentally broken:

- **No guarantee it ever runs**, or _when_ — you can't rely on it for releasing resources.
- **Hurts performance** and can resurrect objects, delaying collection.
- **Security and reliability risks**.

Status to state confidently: JEP 421 deprecated finalization for removal; it remains enabled by default for now but will be disabled and eventually removed in a future release. The migration path started back in Java 9. So the correct answer is _don't use it_ and know the replacements.

**Modern replacement 1 — try-with-resources** (for anything with a clear scope). Implement `AutoCloseable`:
```java
class Resource implements AutoCloseable {
    Resource() { System.out.println("opened"); }
    @Override public void close() { System.out.println("closed"); } // deterministic
}

try (Resource r = new Resource()) {
    // use r
}   // r.close() runs deterministically here — no reliance on GC timing
```

**Modern replacement 2 — `Cleaner`** (for a last-resort safety net on native resources, replacing the `finalize()` use case):
```java
import java.lang.ref.Cleaner;

class NativeThing implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    NativeThing() {
        // cleanup action must NOT reference the outer object (would keep it alive)
        this.cleanable = CLEANER.register(this, () -> System.out.println("native freed"));
    }
    @Override public void close() { cleanable.clean(); }
}
```

_Interview Q: "Why is `finalize()` discouraged, and what replaces it?"_ → It's unreliable (no guarantee it runs or when), slow, and unsafe — deprecated for removal (JEP 421). Use try-with-resources / `AutoCloseable` for deterministic cleanup, and `Cleaner` as a native-resource safety net.

---
## Module 8 Preview — GC Troubleshooting Toolkit

**One line:** When GC/memory goes wrong in production, you diagnose with **heap dumps**, **thread dumps**, and the JDK CLI tools. (Full detail is Module 8; this note connects the concepts.)

Tying the GC theory to real diagnosis :

- **Symptom: frequent Full GCs / rising Old gen** → likely a **memory leak** or undersized heap. Capture a **heap dump** (`jmap` / `jcmd`) and analyze it in Eclipse MAT to find what's retaining memory (the reference chain back to a GC root).
- **Symptom: `OutOfMemoryError: Java heap space`** → the heap genuinely filled — leak, or heap too small for the workload. Heap dump + MAT.
- **Symptom: `OutOfMemoryError: Metaspace`** → too many classes / a **classloader leak** (often redeploy-related).
- **Symptom: long pauses / high latency** → analyze **GC logs**; consider switching collector (e.g., to G1/ZGC) or tuning pause targets.

The essential tools (detailed in Module 8): `jps`, `jstat` (live GC stats), `jmap` (heap dump), `jstack` (thread dump), `jcmd` (does most of the above), plus **Eclipse MAT** / **VisualVM** for analysis.

```bash
# quick previews (full coverage in Module 8)
jstat -gc <pid> 1000        # print GC stats every 1s
jmap -dump:live,format=b,file=heap.hprof <pid>   # capture a heap dump
```

_Interview Q: "A prod service is doing constant Full GCs — how do you diagnose it?"_ → Confirm via GC logs/`jstat`, capture a heap dump (`jmap`/`jcmd`), analyze in Eclipse MAT to find the dominating retained objects and their reference chain to a GC root — usually revealing a leak or an undersized heap.

---

That completes Module 5. The whole GC story now reads as one arc: **fundamentals** (reachability → generations → mark-sweep-compact) explain _what_ GC does, and **collectors** are just different engineering trade-offs (throughput ↔ latency) on those same primitives, ending in **troubleshooting** that turns theory into diagnosis.

For your MOC, the highest-value links: **Collectors Overview** as a hub to each collector note; **CMS** ↔ **G1** (successor relationship) and **CMS** ↔ **Stop-the-World** (why concurrent collectors arose); **Memory Leaks** ↔ **GC Roots** from Part 1 (a leak _is_ unwanted root-reachability) ↔ **Troubleshooting Toolkit**; and **`finalize()`** ↔ **Phantom References** from Part 1 (both about post-mortem cleanup).

Three version-currency traps are now correctly handled across the module — CMS (removed in 14), `finalize()` (deprecated for removal), and static-field storage (heap, from your Module 4 correction) — which are precisely the ones interviewers use to date a candidate's knowledge.

