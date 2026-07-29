Here's Module 7 — Tuning & Flags. The core sizing flags (`-Xms`, `-Xmx`, `-Xss`) are stable across versions; where a _default_ is version-sensitive I've flagged it. Two notes.

---
## Essential JVM Flags

**One line:** A handful of flags control the two things you tune most — **heap size** (`-Xms`/`-Xmx`), **stack size** (`-Xss`) — plus Metaspace and GC selection. Know these cold; they're the practical payoff of Modules 3 and 5.

JVM flags come in a few syntactic families — recognizing the shape helps you read any flag:

- **`-X` flags** — non-standard but stable (e.g., `-Xmx`). Set core sizes.
- **`-XX:` flags** — advanced/experimental tuning. Two forms:
    - **Boolean:** `-XX:+Flag` turns _on_, `-XX:-Flag` turns _off_ (note the `+`/`-`).
    - **Value:** `-XX:Flag=value` sets a number/option.

**The heap flags (the ones you'll actually use most):**

```bash
-Xms512m      # initial (starting) heap size
-Xmx2g        # MAXIMUM heap size — the single most important flag
```

- `-Xmx` caps the heap. Hit the cap with live objects still growing → `OutOfMemoryError: Java heap space`.
- `-Xms` sets the starting size. **Common production practice: set `-Xms` = `-Xmx`.** This pre-allocates the full heap up front, avoiding the cost and pauses of the JVM resizing the heap at runtime.

```bash
java -Xms2g -Xmx2g -jar app.jar   # fixed 2 GB heap, no resizing
```

**The stack flag:**

```bash
-Xss512k      # stack size PER THREAD (Module 3's stack)
```

- Bigger `-Xss` → deeper recursion before `StackOverflowError`, but each thread costs more memory (fewer total threads possible). It's a per-thread multiplier, so raising it in a many-threaded app adds up fast.

**The Metaspace flags** (recall Module 3 — Metaspace holds class _metadata_, auto-grows by default):

```bash
-XX:MetaspaceSize=128m       # initial threshold that triggers the first Metaspace GC
-XX:MaxMetaspaceSize=512m    # optional HARD CAP (default is unlimited)
```

- By default Metaspace grows freely, so a classloader leak can consume native memory until the machine is starved. Setting `-XX:MaxMetaspaceSize` caps it so the failure surfaces as a catchable `OutOfMemoryError: Metaspace` instead. _(Reminder: pre-Java 8 this was `-XX:MaxPermSize` for PermGen — using that flag today does nothing, a knowledge-currency tell.)_

**The GC selection flags** (from Module 5):

```bash
-XX:+UseG1GC             # G1 (this is already the default since Java 9)
-XX:+UseZGC              # ZGC (ultra-low latency; generational by default on Java 23+)
-XX:+UseParallelGC       # Parallel (throughput)
-XX:+UseSerialGC         # Serial (small heaps)
-XX:MaxGCPauseMillis=200 # pause-time target (G1/ZGC honor this)
```

**Diagnostic flags worth knowing** (they save you in production):

```bash
-XX:+HeapDumpOnOutOfMemoryError          # auto-capture a heap dump when OOM hits
-XX:HeapDumpPath=/var/logs/heapdump.hprof # where to write it
-Xlog:gc*                                 # unified GC logging (Java 9+); older syntax was -XX:+PrintGCDetails
```

`-XX:+HeapDumpOnOutOfMemoryError` is a must-have in production — it captures the evidence _at the moment of failure_ so you can diagnose the leak afterward in Eclipse MAT (Module 8), instead of trying to reproduce it.

A realistic production command pulling it together:

```bash
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/logs/ \
     -Xlog:gc*:file=/var/logs/gc.log \
     -jar app.jar
```

_Interview Q: "How do you set max heap? How do you pick a GC?"_ → `-Xmx` sets the max heap (`-Xms` the initial; set them equal in prod to avoid resizing). Pick the GC with `-XX:+UseG1GC` / `-XX:+UseZGC` / etc. Also mention `-XX:+HeapDumpOnOutOfMemoryError` for production diagnostics.

---
## Heap Sizing & GC Selection Heuristics

**One line:** Sizing and collector choice come down to your workload's priority — **throughput vs latency** — and its heap size and object lifetime profile. There's no universal "best"; there are sensible defaults and trade-offs.

This note is about _judgment_, which is exactly what separates a 6-YOE answer from a junior one. Interviewers want reasoning, not memorized numbers.

**Heap sizing heuristics:**

- **Don't over-size the heap.** A bigger heap means _rarer but longer_ GC pauses (more to scan/compact when GC finally runs). Bigger isn't automatically better.
- **Don't under-size it either.** Too small → frequent GC and, worse, frequent **Full GCs** as the Old gen fills → throughput collapses, then `OutOfMemoryError`.
- **Set `-Xms` = `-Xmx` in production** to avoid runtime resizing pauses (as above).
- **Leave headroom for non-heap memory.** Total process memory = heap + Metaspace + thread stacks + JIT code cache + direct/native buffers. In a **container**, if you set `-Xmx` too close to the container memory limit, the _non-heap_ usage can push the process over the limit and the OS/orchestrator kills it (an OOM-kill, distinct from a Java `OutOfMemoryError`). Modern JVMs are container-aware and respect cgroup limits — prefer `-XX:MaxRAMPercentage=75.0` in containers so the JVM sizes the heap as a fraction of the _container's_ memory rather than a hardcoded value.

```bash
# Container-friendly heap sizing (Java 10+ is container-aware):
java -XX:MaxRAMPercentage=75.0 -jar app.jar
```

**GC selection heuristics** — match the collector to the priority:

|Your priority / situation|Reasonable choice|
|---|---|
|Balanced, typical server app (don't overthink)|**G1** (the default — often just leave it)|
|Maximum throughput, pauses don't matter (batch)|**Parallel**|
|Ultra-low latency, large heap (trading, real-time)|**ZGC** or **Shenandoah**|
|Small heap / container / single core|**Serial**|

The decision tree in words: _start with the default (G1)_; only change if you have a measured problem. If profiling shows pauses are your bottleneck and you have a large heap → try ZGC. If it's a throughput-only batch job with no latency concern → Parallel. If it's a tiny memory-constrained service → Serial.

**The cardinal rule — measure, don't guess:**

> Tune based on **GC logs and profiling data**, not hunches. Capture GC logs (`-Xlog:gc*`), observe actual pause times and frequencies, change _one_ thing, and re-measure. Premature GC tuning without data usually makes things worse.

This is the mature-engineer framing interviewers reward: the best answer to "how would you tune this?" is often "first I'd measure — capture GC logs and a profile to see whether the problem is pause length, pause frequency, or allocation rate — _then_ decide."

_Interview Q: "How do you size the heap / choose a GC?"_ → It depends on the throughput-vs-latency priority and workload. Start with G1 and default sizing; set `-Xms`=`-Xmx` (or `MaxRAMPercentage` in containers) with headroom for non-heap memory; only switch collectors (ZGC for latency, Parallel for throughput) based on **measured** GC-log/profiling data — never guess.

---

That's Module 7. It's the applied layer over Modules 3 and 5: those taught the memory _structure_ and GC _mechanics_; this turns them into knobs you actually turn. For your MOC, link **Essential Flags** ↔ Module 3 (heap/stack/Metaspace are what the size flags target) and ↔ Module 5's _Collectors Overview_ (the `-XX:+Use...GC` flags); link **Sizing Heuristics** ↔ Module 8 (the "measure first" rule leads straight into the monitoring tools).

Two version-currency notes now handled here: **tiered compilation defaults** (Module 6) and, in this module, **container-aware heap sizing** (`MaxRAMPercentage`, Java 10+) plus the dead `-XX:MaxPermSize` flag — both things a candidate who stopped learning at Java 8 would get wrong.

Last one: Module 8 (Monitoring & Troubleshooting) — the CLI tools, heap/thread dumps, and OOM diagnosis. 