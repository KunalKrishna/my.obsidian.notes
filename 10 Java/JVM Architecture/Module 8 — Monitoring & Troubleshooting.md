it's about diagnosing real production problems, which is exactly what distinguishes an experienced engineer from someone who only knows theory. The tools are stable across versions; where `jcmd` has superseded older tools I've noted it. 

---
## JDK Command-Line Tools

**One line:** The JDK ships CLI tools to inspect a running JVM — `jps` (list JVMs), `jstat` (live GC stats), `jmap` (heap dump), `jstack` (thread dump), and `jcmd` (the modern all-in-one). These are your first responders when production misbehaves.

Every tool needs a target JVM's **PID** (process ID), so you always start with `jps`.

**`jps` — list running Java processes** (finds the PID):

```bash
jps -l
# Output:
# 4521 com.example.MyApp      <- PID 4521 is your app
# 4600 jdk.jcmd/sun.tools.jps.Jps
```

**`jstat` — live GC statistics** (watch GC behavior without stopping the app):

```bash
jstat -gc 4521 1000
# ^gc stats for PID 4521, refreshed every 1000ms
# Shows Eden/Survivor/Old usage, plus YGC/YGCT (young GC count/time)
#   and FGC/FGCT (FULL GC count/time)

jstat -gcutil 4521 1000    # same data as percentages — easier to read
```

The key thing to watch: **FGC** (Full GC count). If it's climbing rapidly, you have a problem (leak or undersized heap — see the OOM note). This is often your _first_ signal.

**`jmap` — heap information and heap dumps:**

```bash
jmap -histo 4521           # histogram: which classes have the most instances/bytes
jmap -dump:live,format=b,file=heap.hprof 4521   # full heap dump for offline analysis
#          ^live = run GC first, dump only reachable objects
```

**`jstack` — thread dump** (snapshot of every thread's stack):

```bash
jstack 4521 > threads.txt   # capture all thread states — the tool for hangs/deadlocks
```

**`jcmd` — the modern Swiss Army knife** (recommended today; consolidates most of the above):

```bash
jcmd 4521 help                          # list everything jcmd can do for this PID
jcmd 4521 GC.heap_dump heap.hprof       # heap dump (jmap replacement)
jcmd 4521 Thread.print                  # thread dump (jstack replacement)
jcmd 4521 GC.class_histogram            # class histogram (jmap -histo replacement)
jcmd 4521 VM.flags                      # see the JVM flags in effect
jcmd 4521 GC.run                        # suggest a GC
```

Modern practice: **reach for `jcmd` first** — it's the officially preferred, actively maintained entry point, and several older tools are considered legacy. Still know `jmap`/`jstack`/`jstat` by name, because they're everywhere in existing docs, scripts, and older environments.

_Interview Q: "A prod service is slow — what tools do you reach for?"_ → `jps` to find the PID, `jstat -gcutil` to watch GC (is it Full-GC-thrashing?), `jstack`/`jcmd Thread.print` for a thread dump if it's hung, and `jmap`/`jcmd GC.heap_dump` to capture a heap dump for offline analysis. Mention `jcmd` as the modern unified tool.

---
## Heap Dumps & Thread Dumps

**One line:** A **heap dump** is a snapshot of every object in memory (used to find _memory_ problems / leaks); a **thread dump** is a snapshot of every thread's stack (used to find _hangs, deadlocks, and high CPU_). Different tools for different failures.

Know which dump answers which question — mixing them up is a common stumble.

**Heap Dump — "what's eating my memory?"**

A binary snapshot (`.hprof`) of all objects, their fields, and their references at one instant. You analyze it _offline_ to find what's retaining memory.

Capture it three ways:

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>    # on demand
jcmd <pid> GC.heap_dump heap.hprof                # on demand (modern)
# OR automatically at the moment of failure (best for prod):
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/
```

Analyze it in **Eclipse MAT (Memory Analyzer Tool)** — the standard — or VisualVM. MAT's two killer features:

- **Dominator tree** — shows which objects retain the most memory (the biggest memory hogs).
- **"Path to GC Roots"** — for a suspect object, shows the exact reference chain keeping it alive. _This is how you pinpoint a leak:_ you find the object that shouldn't still be there, then trace why it's still reachable (recall Module 5 — a leak is unwanted reachability). Often it traces back to a `static` collection or a stray listener.

**Thread Dump — "why is it stuck / burning CPU?"**

A text snapshot of every thread and what it's currently executing (its stack) plus its state (`RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`).

```bash
jstack <pid> > threads.txt
jcmd <pid> Thread.print > threads.txt     # modern equivalent
```

Use it to diagnose:

- **Deadlocks** — `jstack` explicitly detects and prints "Found one Java-level deadlock" with the two threads each holding a lock the other wants.
- **Hangs** — many threads `BLOCKED` on the same lock reveals contention on a bottleneck.
- **High CPU** — capture a few dumps a second apart; a thread stuck `RUNNABLE` in the same method across all of them is your CPU hog. (Cross-reference the OS thread ID from `top -H` to the dump.)

Pro tip worth stating: **take 2–3 thread dumps a few seconds apart.** A single dump is a still photo; comparing several tells you what's _stuck_ (same stack every time) versus what's just passing through (changing stacks).

_Interview Q: "How do you capture and analyze a heap dump / thread dump?"_ → Heap dump via `jmap`/`jcmd GC.heap_dump` (or auto on OOM), analyzed in Eclipse MAT using the dominator tree + path-to-GC-roots to find leaks. Thread dump via `jstack`/`jcmd Thread.print` for deadlocks (auto-detected), contention, and high CPU — take several a few seconds apart to see what's truly stuck.

---
## OutOfMemoryError Types

**One line:** `OutOfMemoryError` isn't one error — the _message_ tells you _which_ memory area ran out, and each points to a different cause and fix. Reading the specific message is the whole game.

A junior says "I got an OOM." A senior reads the message and knows _where_ and _why_. The common variants:

**1. `java.lang.OutOfMemoryError: Java heap space`**

- **Meaning:** the heap filled with reachable objects and GC couldn't free enough.
- **Causes:** a genuine **memory leak** (objects accumulating — Module 5), or a heap simply too small for the workload.
- **Diagnosis:** heap dump → Eclipse MAT → find the dominator / path to GC roots.
- **Fix:** fix the leak, or raise `-Xmx` if the workload legitimately needs more.

**2. `java.lang.OutOfMemoryError: GC overhead limit exceeded`**

- **Meaning:** the JVM is spending **>98% of time in GC while reclaiming <2% of heap** — it's thrashing, technically alive but doing almost no real work.
- **Cause:** essentially a slower-motion version of #1 — the heap is nearly full, so GC runs constantly for tiny gains.
- **Fix:** same as heap space — find the leak or resize. (This error often precedes a full `Java heap space` OOM.)

**3. `java.lang.OutOfMemoryError: Metaspace`**

- **Meaning:** class _metadata_ space is exhausted (Module 3 — native memory, not the heap).
- **Causes:** loading too many classes, or a **classloader leak** — classic on repeated app redeploys in app servers, where old classloaders never get collected.
- **Fix:** find the classloader leak (heap dump helps); cap/raise with `-XX:MaxMetaspaceSize`. _(Note: pre-Java 8 this said `PermGen space` — seeing that message means an old JVM. Don't cite it as current.)_

**4. `java.lang.OutOfMemoryError: unable to create new native thread`**

- **Meaning:** the OS refused to create another thread — you've hit an OS/process thread limit or run out of native memory for thread stacks.
- **Causes:** a thread leak (threads created but never terminated — e.g., a leaked executor per request), or too many threads for the machine.
- **Fix:** find where threads are created without bound (thread dump shows the count and their stacks); use bounded thread pools; check OS `ulimit`. Counterintuitively, a _smaller_ `-Xss` can help by leaving more native memory for more thread stacks.

**5. `java.lang.OutOfMemoryError: Requested array size exceeds VM limit`** _(rarer)_

- **Meaning:** code tried to allocate an array larger than the JVM's max (near `Integer.MAX_VALUE`).
- **Cause:** usually a bug — an overflowed size calculation.

The one-line diagnostic reflex: **read the text after the colon** — `Java heap space` → object leak/undersized heap; `Metaspace` → classloader/class leak; `unable to create new native thread` → thread leak/OS limit. That single habit is the answer.

_Interview Q: "You get an OOM — how do you diagnose it?"_ → Read the specific message first. `Java heap space`/`GC overhead limit` → heap leak or undersized heap → heap dump + MAT. `Metaspace` → classloader leak (often redeploys). `unable to create new native thread` → thread leak / OS limit. Each message points to a different area and fix.

---
## `StackOverflowError` vs `OutOfMemoryError`

**One line:** Both are `Error`s about exhausted memory, but different memory: `StackOverflowError` = a single thread's **stack** is full (too-deep call chain); `OutOfMemoryError` = a **shared area** (usually heap) is full. Different cause, different area.

A clean comparison question that tests whether you truly understand Module 3's stack-vs-heap split.

|x|`StackOverflowError`|`OutOfMemoryError`|
|---|---|---|
|**Which memory**|Thread's **stack** (per-thread)|**Heap** (or Metaspace/native — shared)|
|**Cause**|Call chain too deep — too many stacked frames|Too many/too-large live objects; can't reclaim enough|
|**Classic trigger**|Unbounded recursion (no base case)|Memory leak; undersized heap; huge allocations|
|**Related flag**|`-Xss` (stack size)|`-Xmx` (heap size)|
|**Type**|`Error` (subclass of `VirtualMachineError`)|`Error` (subclass of `VirtualMachineError`)|

**`StackOverflowError`** — one thread pushed more frames than its stack can hold. The textbook cause is infinite recursion:

```java
static int recurse(int n) {
    return recurse(n + 1);   // no base case -> frames never pop -> StackOverflowError
}
```

Fix: add/verify a base case, or convert deep recursion to iteration. Raising `-Xss` only delays it — the real fix is bounding the depth.

**`OutOfMemoryError`** — a shared memory area (heap, Metaspace, native) can't satisfy an allocation. Causes and fixes are the previous note's whole story:

```java
List<byte[]> list = new ArrayList<>();
while (true) {
    list.add(new byte[10_000_000]);   // objects stay reachable via 'list'
}                                      // heap fills -> OutOfMemoryError: Java heap space
```

The distinction to state crisply: **stack overflow is about call _depth_ (per-thread), heap OOM is about object _volume_ (shared).** One is fixed by bounding recursion; the other by fixing a leak or sizing memory.

One nuance worth a sentence: both extend `VirtualMachineError`, so both are `Error`s, not `Exception`s — you generally **don't catch** them, because they signal the JVM is in a bad state rather than a recoverable condition.

_Interview Q: "`StackOverflowError` vs `OutOfMemoryError`?"_ → Both are `Error`s but hit different memory: SOF = a thread's stack overflowed (usually unbounded recursion — depth problem, `-Xss`); OOM = a shared area like the heap is exhausted (leak/undersized/huge allocation — volume problem, `-Xmx`). Neither should normally be caught.

---

That completes Module 8 — and the entire JVM syllabus. This module is where everything converges: the tools _observe_ the memory areas (Module 3), diagnose the GC behavior (Module 5), and act on the flags (Module 7). For your MOC, link **CLI Tools** ↔ **Heap/Thread Dumps** (the tools produce the dumps) ↔ **OOM Types** (the dumps diagnose the OOMs), and link **OOM Types** back to Module 5's _Memory Leaks_ and Module 3's _Metaspace_ and _Stack_ notes — those cross-links mirror exactly how interviewers chain questions ("you have a leak → how do you find it? → what tool? → what does the dump show?").

---

**The full picture — all 8 modules are now done.** Here's how the syllabus hangs together as a study map:

- **Modules 1–2** (Architecture, Class Loading) — how code gets _into_ the JVM.
- **Modules 3–4** (Memory Areas, Object Storage) — _where_ everything lives.
- **Module 5** (GC) — how memory is _reclaimed_. The heaviest interview area; give it the most time.
- **Module 6** (Execution Engine/JIT) — how code _runs fast_.
- **Module 7** (Tuning/Flags) — the _knobs_ over Modules 3 & 5.
- **Module 8** (Monitoring) — _diagnosing_ it all in production. Your 6-YOE differentiator.

A suggested two-day pass, given the depth here: **Day 1** — Modules 1–4 plus start Module 5 fundamentals (they're foundational and build linearly). **Day 2** — finish Module 5 (collectors + troubleshooting), then 6–8, which lean on the memory model you've already built.

The four version-currency traps are handled correctly throughout — **static fields on the heap** ([[Module 4 — Object & Variable Storage|Module 4]]), **PermGen→Metaspace** ([[Module 3 — Runtime Data Areas (Memory Structure)|Module 3]]), **CMS removed in 14** ([[Module 5 — Garbage Collection|Module 5]]), and **`finalize()` deprecated for removal** (Module 5) — plus the container-awareness and tiered-compilation-default points. 

