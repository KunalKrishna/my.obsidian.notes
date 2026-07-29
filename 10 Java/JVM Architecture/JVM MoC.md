---
type: concept
area: java
topic: JVM
status: learning
confidence: 2
last-reviewed: 2026-07-23
tags:
  - high-yield
---
### [[Module 1 — JVM Fundamentals & Architecture]]
- **JDK vs JRE vs JVM** (P1) — what each contains and the layering. _Q: "Difference between JDK, JRE, and JVM?"_
- **Compilation & execution flow** (P1) — `.java` → `javac` → `.class` bytecode → loaded → interpreted/JIT-compiled → native. _Q: "What happens when you run `java MyClass`?"_
- **JVM architecture overview** (P1) — the three subsystems: ClassLoader, Runtime Data Areas, Execution Engine. Make this a hub note that links to Modules 2, 3, 6.
- **Platform independence / bytecode / WORA** (P2) — why bytecode + per-OS JVM = portability.
### [[Module 2 — Class Loading]]
- **ClassLoader types** (P1) — Bootstrap, Platform/Extension, Application/System. _Q: "Which classloader loads `String` vs your own class?"_
- **Class loading phases** (P1) — Loading → Linking (Verification, Preparation, Resolution) → Initialization. Know what happens in each.
- **Parent delegation model** (P1) — how a request goes up before down, and why (security, avoiding duplicate core classes). _Q: "Why parent delegation?"_
- **`Class.forName()` vs `ClassLoader.loadClass()`** (P2) — the initialization difference (one runs static blocks, one doesn't).
- **Custom classloaders** (P3) — when/why (plugins, hot reload, app servers). Conceptual only.
### [[Module 3 — Runtime Data Areas (Memory Structure)| Module 3 — Runtime Data Areas]]
- **Heap structure** (P1) — Young gen (Eden, Survivor S0/S1) and Old gen. _Q: "Walk me through the heap layout."_
- **Stack & stack frames** (P1) — per-thread, holds frames with local variables and operand stack; `StackOverflowError`.
- **Metaspace vs PermGen** (P1) — the Java 8 change, why PermGen was removed, where it lives now (native memory). _Q: "What changed with PermGen in Java 8?"_
- **PC Register & Native Method Stack** (P2) — short note, know they exist and their role.
- **Heap vs Stack** (P1) — what's stored where, thread-sharing, lifetime. Very common comparison question.
### [[Module 4 — Object & Variable Storage]]
- **Where things live** (P1) — objects on heap, references and primitives-in-methods on stack, static fields in Metaspace. _Q: "Where is a local `int` stored vs a `new` object?"_
- **String pool / `intern()`** (P2) — literal pool, `new String()` vs literal, `.==`  vs `.equals()`. Frequently asked.
### [[Module 5 — Garbage Collection]] (the heaviest interview area — spend the most time here)
- **GC basics & why automatic memory management** (P1) — what GC solves vs manual `free()`.
- **GC roots & reachability** (P1) — how the JVM decides what's alive. _Q: "How does JVM know an object is garbage?"_
- **Reference types** (P1) — Strong, Soft, Weak, Phantom, and when each matters (caches, `WeakHashMap`). _Q: "Difference between soft and weak references?"_
- **Generational GC & weak generational hypothesis** (P1) — "most objects die young," why generations exist.
- **Minor vs Major vs Full GC** (P1) — triggers and scope of each. _Q: "Minor vs Full GC?"_
- **Mark-Sweep-Compact** (P1) — the core algorithm phases.
- **GC collectors** (P1) — Serial, Parallel, CMS (deprecated), **G1** (know this well — it's the default since Java 9), ZGC & Shenandoah (low-latency, know at a high level). _Q: "How does G1 work / why is it the default?"_
- **Stop-the-world pauses** (P2) — what they are, which collectors minimize them.
- **Memory leaks in Java** (P1) — yes, they happen: static collections, unclosed resources, listeners, `ThreadLocal`, classloader leaks. _Q: "Can Java have memory leaks? Give an example."_
- **`finalize()` and why it's deprecated** (P3) — plus `Cleaner`/try-with-resources as the modern approach.
### [[Module 6 — Execution Engine & JIT]]
- **Interpreter vs JIT** (P1) — why both exist, warm-up. _Q: "What does the JIT compiler do?"_
- **HotSpot, C1/C2 & tiered compilation** (P2) — client vs server compiler, hot-method optimization, inlining.
### [[Module 7 — JVM Tuning & Flags]]
- **Essential flags** (P1) — `-Xms`, `-Xmx`, `-Xss`, `-XX:MetaspaceSize`, and GC-selection flags like `-XX:+UseG1GC`. _Q: "How do you set max heap? How do you pick a GC?"_
- **Heap sizing & GC selection heuristics** (P2) — throughput vs latency trade-offs.
### [[Module 8 — Monitoring & Troubleshooting]] (strong signal for 6 YOE — shows real-world experience)
- **CLI tools** (P1) — `jps`, `jstat`, `jmap`, `jstack`, `jcmd`. Know what each is for. _Q: "A prod service is slow — what tools do you reach for?"_
- **Heap dumps & thread dumps** (P1) — how to capture (`jmap`/`jcmd`, `jstack`) and analyze (Eclipse MAT, VisualVM).
- **OutOfMemoryError types** (P1) — Java heap space, GC overhead limit exceeded, Metaspace, unable to create new native thread. Know how to distinguish. _Q: "You get an OOM — how do you diagnose it?"_
- **`StackOverflowError` vs `OutOfMemoryError`** (P2) — cause and difference.


