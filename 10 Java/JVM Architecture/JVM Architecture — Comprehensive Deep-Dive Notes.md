# JVM Architecture — Comprehensive Deep-Dive Notes
### Targeted at 6 YOE Java Interview Candidates | Modern JVM (Java 8–21)

---

## QUICK REFERENCE: Master Summary Table

| JVM Spec (Abstract) | HotSpot Implementation | Memory Location             | Managed By             | Cleaned By                | Per-Thread or Shared |
| ------------------- | ---------------------- | --------------------------- | ---------------------- | ------------------------- | -------------------- |
| Method Area         | **Metaspace**          | Off-heap (native OS memory) | OS + JVM allocator     | Class unloading / Full GC | Shared               |
| Heap                | Young Gen + Old Gen    | JVM-managed heap            | JVM                    | Garbage Collector         | Shared               |
| JVM Stack           | JVM Stack (frames)     | Thread-local stack          | JVM                    | Auto (frame pop/push)     | Per-Thread           |
| PC Register         | PC Register            | CPU register / memory       | JVM                    | Auto (thread lifecycle)   | Per-Thread           |
| Native Method Stack | C Stack                | Thread-local native stack   | OS + JVM               | Auto (thread lifecycle)   | Per-Thread           |
| *(Not in JVM Spec)* | **Code Cache**         | Off-heap (native OS memory) | JVM Code Cache Sweeper | Code Cache Flusher / JVM  | Shared               |
|                     |                        |                             |                        |                           |                      |

---
## 1. JVM High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                          J V M  A R C H I T E C T U R E                │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                   CLASS LOADER SUBSYSTEM                       │    │
│  │   Bootstrap CL  →  Platform CL  →  Application CL              │    │
│  │         [ Loading → Linking → Initialization ]                 │    │
│  └────────────────────────┬───────────────────────────────────────┘    │
│                           │ Loaded .class bytecode                     │
│                           ▼                                            │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    RUNTIME DATA AREAS                          │    │
│  │                                                                │    │
│  │  ┌─────────────────┐  ┌────────────────────────────────────┐   │    │
│  │  │   METHOD AREA   │  │             HEAP AREA              │   │    │
│  │  │  (Metaspace)    │  │  [Young Gen]  |  [Old Gen]         │   │    │
│  │  │  OFF-HEAP       │  │  Eden|S0|S1   |  Tenured           │   │    │
│  │  └─────────────────┘  └────────────────────────────────────┘   │    │
│  │           ◄──── SHARED ACROSS ALL THREADS ────►                │    │
│  │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─       │    │
│  │           ◄─────── PER-THREAD AREAS ──────────►                │    │
│  │  ┌──────────────┐  ┌─────────────┐  ┌─────────────────────┐    │    │
│  │  │  JVM STACK   │  │ PC REGISTER │  │ NATIVE METHOD STACK │    │    │
│  │  │ (T1)         │  │ (T1)        │  │ (T1)                │    │    │
│  │  │  JVM STACK   │  │ PC REGISTER │  │ NATIVE METHOD STACK │    │    │
│  │  │ (T2) ...     │  │ (T2) ...    │  │ (T2) ...            │    │    │
│  │  └──────────────┘  └─────────────┘  └─────────────────────┘    │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                           │                                            │
│                           ▼                                            │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     EXECUTION ENGINE                           │    │
│  │  ┌─────────────┐  ┌─────────────────┐  ┌────────────────────┐  │    │
│  │  │ Interpreter │  │  JIT Compiler   │  │ Garbage Collector  │  │    │
│  │  │             │  │  C1 (Client)    │  │ G1 / ZGC /         │  │    │
│  │  │             │  │  C2 (Server)    │  │ Parallel / Serial  │  │    │
│  │  └─────────────┘  └────────┬────────┘  └────────────────────┘  │    │
│  │                            │ compiled native code              │    │
│  │                            ▼                                   │    │
│  │                    ┌───────────────┐                           │    │
│  │                    │  CODE CACHE   │  (Off-heap)               │    │
│  │                    └───────────────┘                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                           │                                            │
│                           ▼                                            │
│  ┌──────────────────────────────┐   ┌──────────────────────────────┐   │
│  │  Native Method Interface     │──▶│  Native Method Libraries     │   │
│  │  (JNI)                       │   │  (.dll / .so / .dylib)       │   │
│  └──────────────────────────────┘   └──────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

---
## 2. Class Loader Subsystem

The Class Loader Subsystem is responsible for the **dynamic loading, linking, and initialization** of `.class` files into the JVM at runtime — not at compile time.

```
┌───────────────────────────────────────────────────────────────┐
│                   CLASS LOADER SUBSYSTEM                      │
│                                                               │
│  ┌──────────┐     ┌───────────────────────────────────┐       │
│  │ .class   │────▶│  PHASE 1: LOADING                 │       │
│  │  files   │     │  Reads bytecode → creates         │       │
│  │  (disk / │     │  java.lang.Class object in Heap   │       │
│  │  network)│     └─────────────────┬─────────────────┘       │
│  └──────────┘                       │                         │
│                                     ▼                         │
│                      ┌──────────────────────────────┐         │
│                      │  PHASE 2: LINKING            │         │
│                      │                              │         │
│                      │  ┌────────────────────────┐  │         │
│                      │  │ 2a. VERIFICATION       │  │         │
│                      │  │ bytecode validity check│  │         │
│                      │  └───────────┬────────────┘  │         │
│                      │              ▼               │         │
│                      │  ┌────────────────────────┐  │         │
│                      │  │ 2b. PREPARATION        │  │         │
│                      │  │ Allocate static vars   │  │         │
│                      │  │ → assign default values│  │         │
│                      │  └───────────┬────────────┘  │         │
│                      │              ▼               │         │
│                      │  ┌────────────────────────┐  │         │
│                      │  │ 2c. RESOLUTION         │  │         │
│                      │  │ Symbolic refs →        │  │         │
│                      │  │ Direct memory refs     │  │         │
│                      │  └────────────────────────┘  │         │
│                      └──────────────────────────────┘         │
│                                     │                         │
│                                     ▼                         │
│                      ┌──────────────────────────────┐         │
│                      │  PHASE 3: INITIALIZATION     │         │
│                      │  Execute <clinit>()          │         │
│                      │  (static blocks + static     │         │
│                      │   field assignments)         │         │
│                      └──────────────────────────────┘         │
└───────────────────────────────────────────────────────────────┘
```
### 2.1 Three Phases in Detail

#### Phase 1 — Loading
- Reads the binary `.class` file (from disk, network, jar, etc.)
- Creates a `java.lang.Class` object in the **Heap** representing that class
- Class is identified by: **fully qualified name + ClassLoader instance** (two classes loaded by different class loaders are considered different)
#### Phase 2 — Linking

| Sub-Phase | What Happens | Example |
|---|---|---|
| **Verification** | Validates bytecode structure against JVM spec — prevents corrupted/malicious code | Magic number check (0xCAFEBABE), type checks, stack depth checks |
| **Preparation** | Allocates memory for class-level (static) variables and assigns **JVM default values** (not programmer-assigned values) | `static int x = 5;` → x gets `0` here, not `5` |
| **Resolution** | Converts symbolic references in the constant pool to direct memory references | `java/lang/String` → actual pointer to String class in Metaspace |
> ⚠️ **Resolution is lazy by default** — symbols are resolved only when first used unless `-XX:+EagerResolutionEnabled` is set.
#### Phase 3 — Initialization
- JVM executes the `<clinit>()` method (compiler-generated from static initializer blocks and static field assignments)
- **Guaranteed thread-safe** by JVM — uses class-level locking
- Happens only **once per class** per ClassLoader
- Order: parent class `<clinit>()` → child class `<clinit>()`

```java
class Example {
    static int x = 10;          // ← assigned during Initialization
    static { x = 20; }          // ← <clinit>() runs this
    // During Preparation: x = 0 (default)
    // During Initialization: x = 10, then x = 20
}
```

---
### 2.2 Class Loader Types & Hierarchy

```
                  ┌──────────────────────────────────────────────────┐
                  │         Bootstrap Class Loader                   │
                  │  • Written in native C++                         │
                  │  • Loads: java.base module                       │
                  │    (java.lang.*, java.util.*, java.io.*, etc.)   │
                  │  • Java 8: loads rt.jar                          │
                  │  • Java 9+: loads $JAVA_HOME/lib/*.jmod          │
                  │  • Returns null when queried (not a Java object) │
                  └───────────────────┬──────────────────────────────┘
                                      │ parent of
                                      ▼
                  ┌─────────────────────────────────────────────────┐
                  │   Platform Class Loader (Java 9+)               │
                  │   [was: Extension Class Loader in Java 8]       │
                  │  • Loads: java.sql, java.xml, java.naming, etc. │
                  │  • Java 8: loads $JAVA_HOME/jre/lib/ext/        │
                  │  • Java 9+: loads platform modules              │
                  └───────────────────┬─────────────────────────────┘
                                      │ parent of
                                      ▼
                  ┌─────────────────────────────────────────────────┐
                  │         Application Class Loader                │
                  │           (System Class Loader)                 │
                  │  • Loads: application classes from              │
                  │    -classpath / -cp / CLASSPATH env var         │
                  │  • Default loader for your code                 │
                  └───────────────────┬─────────────────────────────┘
                                      │ parent of
                                      ▼
                  ┌─────────────────────────────────────────────────┐
                  │       User-Defined Class Loaders                │
                  │  • Extend java.lang.ClassLoader                 │
                  │  • @Override findClass(String name)             │
                  │  • Examples: OSGi, Tomcat WebApp CL,            │
                  │    Spring DevTools restart CL                   │
                  └─────────────────────────────────────────────────┘
```

>[!important]- User-Defined Class Loaders
> When creating a user-defined class loader, you extend the `java.lang.ClassLoader` class and **override the `findClass(String name)` method**. 
> ### 🛠️ Why `findClass` is Overridden
> Officially subclasses are explicitly encouraged to override `findClass(String name)` rather than `loadClass(String name, boolean resolve)`. 
> - **Preserves Delegation**: Overriding `findClass` ensures your custom loader complies with the standard **Parent Delegation Model**.
> - **Fallback Execution**: The default `loadClass` implementation will check if the class is already loaded, delegate to the parent class loader, and only > call `findClass` if the parent cannot find it. 
> ### 📋 Key Step Implementation
> Inside your overridden `findClass` method, you must follow two vital steps to convert raw data into a live Java class: 
> - **Fetch the bytecode**: Read the raw `.class` file data into a `byte[]` array from your specific source (e.g., a custom file system, network path, or encrypted > file). 
> - **Define the class**: Invoke the `defineClass(String name, byte[] b, int off, int len)` method. This is a `final` method provided by `java.lang.> ClassLoader` that natively registers the byte array as a valid `Class` object in the JVM.
>
> ```java
> // 💻 Code Example
> public class MyCustomClassLoader extends ClassLoader {
>
>     @Override
>     protected Class<?> findClass(String name) throws ClassNotFoundException {
>         // 1. Fetch the raw bytecode array from your custom source
>         byte[] classBytes = loadClassBytesFromCustomSource(name);
>        
>         if (classBytes == null) {
>             throw new ClassNotFoundException("Could not find class: " + name);
>         }
>
>         // 2. Convert the byte array into a JVM Class instance
>         return defineClass(name, classBytes, 0, classBytes.length);
>     }
>
>     private byte[] loadClassBytesFromCustomSource(String name) {
>         // Custom logic to fetch your byte array (database, network, etc.)
>         return null;
>     }
> }
> ```
> ### ⚠️ When to Override `loadClass` Instead
>
> You should only override `loadClass(String name, boolean resolve)` if you explicitly want to **break the Parent Delegation Model**. This is rarer and typically > utilized for: 
> - **Class Isolation**: Forcing specific plugin classes to load independently of system-wide dependencies.
> - **Hot-Swapping**: Reloading updated classes dynamically at runtime while bypassing parent caches

### 2.3 Parent Delegation Model

```
  Request to load "com.example.MyClass"
       │
       ▼
  ApplicationCL
  ┌─────────────────────────────────────────────┐
  │ 1. Check local cache → NOT FOUND            │
  │ 2. Delegate to parent (PlatformCL)  ──────▶ │
  └─────────────────────────────────────────────┘
                           │
                           ▼
                     PlatformCL
              ┌──────────────────────┐
              │ Check cache → MISS   │
              │ Delegate to parent ─▶│
              └──────────────────────┘
                           │
                           ▼
                    BootstrapCL
              ┌──────────────────────┐
              │ Check rt.jar/modules │
              │ → NOT FOUND          │
              │ Returns control ◀────│
              └──────────────────────┘
                           │ (not found, bubble back)
                           ▼
                     PlatformCL
              ┌──────────────────────┐
              │ Try to load → MISS   │
              │ Returns control ◀────│
              └──────────────────────┘
                           │ (not found, bubble back)
                           ▼
                   ApplicationCL
              ┌──────────────────────┐
              │ Loads from classpath │
              │ → SUCCESS ✓          │
              └──────────────────────┘
```

**Why Parent Delegation?**
1. **Security**: Prevents user code from overriding `java.lang.String` or `java.lang.Object`
2. **Uniqueness**: Ensures only one copy of a class loaded by a given loader chain
3. **Consistency**: Core classes always loaded by Bootstrap CL

> 💡 **Breaking Parent Delegation**: Thread Context ClassLoader (TCCL) — used in frameworks like JNDI, JDBC, JCE to allow the bootstrap-loaded code to load application-specific implementations. OSGi also breaks it intentionally for module isolation.

---
## 3. Runtime Data Areas

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RUNTIME DATA AREAS                          │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              PROCESS-LEVEL SHARED MEMORY                      │  │
│  │                                                               │  │
│  │   ┌───────────────────────┐   ┌──────────────────────────┐    │  │
│  │   │    METHOD AREA        │   │       HEAP AREA          │    │  │
│  │   │  (Metaspace)          │   │                          │    │  │
│  │   │  Off-heap native mem  │   │  JVM-managed memory      │    │  │
│  │   └───────────────────────┘   └──────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                 PER-THREAD MEMORY (×N threads)                │  │
│  │                                                               │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐  │  │
│  │   │  JVM Stack  │  │ PC Register │  │ Native Method Stack  │  │  │
│  │   │  (T1)       │  │  (T1)       │  │  (T1)                │  │  │
│  │   └─────────────┘  └─────────────┘  └──────────────────────┘  │  │
│  │   ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐  │  │
│  │   │  JVM Stack  │  │ PC Register │  │ Native Method Stack  │  │  │
│  │   │  (T2)       │  │  (T2)       │  │  (T2)                │  │  │
│  │   └─────────────┘  └─────────────┘  └──────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---
### 3.1 Method Area → Metaspace

**Abstract Concept (JVM Spec):** Method Area — shared memory area storing per-class structures  
**HotSpot Implementation:** `Metaspace` (Java 8+) — replaces `PermGen` (Java 7 and below)
#### Key Distinction
| | PermGen (Java ≤ 7) | Metaspace (Java 8+) |
|---|---|---|
| Location | JVM heap (fixed size) | Native OS memory (off-heap) |
| Default Max | 64–85 MB (`-XX:MaxPermSize`) | No limit (until OS runs out) |
| OOM cause | Easy to hit fixed cap | Rare; requires `-XX:MaxMetaspaceSize` to cap |
| GC trigger | Collected with Full GC | Full GC or class unloading |

```
┌────────────────────────────────────────────────────────────────┐
│                    METASPACE (Off-Heap Native Memory)          │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Class Metadata                                        │    │
│  │  • Class name, superclass, interfaces                  │    │
│  │  • Field names, types, access modifiers                │    │
│  │  • Method signatures, access modifiers                 │    │
│  │  • Annotations metadata                                │    │
│  └────────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Method Bytecodes                                      │    │
│  │  • Compiled .class bytecode instructions               │    │
│  │  • Exception tables                                    │    │
│  └────────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Runtime Constant Pool (per class)                     │    │
│  │  • Literal constants (int, long, float values)         │    │
│  │  • Symbolic references (class names, method refs)      │    │
│  │    → resolved to direct refs during Resolution phase   │    │
│  └────────────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Virtual Method Table (vtable) & Interface Table (itable)│  │
│  │  • Used for dynamic dispatch (polymorphism)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ⚠️ NOTE: java.lang.Class OBJECT (mirror) lives in HEAP        │
│  ⚠️ NOTE: Static field VALUES live in HEAP (since Java 8)      │
│          (as part of the Class object in Heap)                 │
└────────────────────────────────────────────────────────────────┘
```

**Memory Manager:** OS kernel (native memory), but JVM's Metaspace allocator manages it  
**Content Type:** Class structures, bytecodes, constant pools, vtable/itable  
**Cleaned By:** Class unloading (when ClassLoader is GC'd) → triggers Full GC or incremental cleanup

**Key JVM Flags:**
```
-XX:MetaspaceSize=64m          # initial commit size (not a hard limit)
-XX:MaxMetaspaceSize=256m      # cap metaspace growth
-XX:MinMetaspaceFreeRatio=40   # min % free space before shrink
-XX:MaxMetaspaceFreeRatio=70   # max % free space before grow
```

**OOM for Metaspace:**
```
java.lang.OutOfMemoryError: Metaspace
```
Causes: excessive class generation (reflection frameworks, Groovy, CGLib proxies, JSP recompilation)

---
### 3.2 Heap Area

**Abstract Concept (JVM Spec):** Heap — runtime data area where objects are allocated  
**HotSpot Implementation:** Generational Heap (Young + Old) for most GCs; Region-based for G1/ZGC

**Memory Manager:** JVM (via Garbage Collector)  
**Cleaned By:** Garbage Collector (GC) — see Section 4.3

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HEAP AREA (JVM Managed)                          │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                   YOUNG GENERATION                             │ │
│  │                                                                │ │
│  │  ┌───────────────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │      EDEN SPACE       │ │ SURVIVOR 0  │ │ SURVIVOR 1  │     │ │
│  │  │                       │ │  (S0/From)  │ │  (S1/To)    │     │ │
│  │  │ • New object alloc    │ │             │ │             │     │ │
│  │  │   (new keyword)       │ │ • Survived  │ │ • Survived  │     │ │
│  │  │ • TLAB allocation     │ │   1+ GC     │ │   2+ GC     │     │ │
│  │  │   (Thread-Local       │ │ • Age:1     │ │ • Age:1-14  │     │ │
│  │  │    Alloc Buffer)      │ │             │ │             │     │ │
│  │  │                       │ │  [One is    │ │   always    │     │ │
│  │  │ Ratio: 8:1:1          │ │   always    │ │   empty]    │     │ │
│  │  │ (default Eden:S0:S1)  │ │   empty]    │ │             │     │ │
│  │  └───────────────────────┘ └─────────────┘ └─────────────┘     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                   OLD GENERATION (Tenured Space)               │ │
│  │                                                                │ │
│  │  • Objects that survived N minor GCs (default age ≥ 15)        │ │
│  │  • Large objects that don't fit in Young Gen                   │ │
│  │  • Long-lived objects (caches, singletons, static refs)        │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              STRING POOL (Heap, since Java 7)                  │ │
│  │                                                                │ │
│  │  • String literals: "hello" — interned automatically           │ │
│  │  • String.intern() — moves to pool                             │ │
│  │  • new String("hello") — creates in Eden, NOT in pool          │ │
│  │  • Implemented as a fixed-size HashMap (Java 8: 60013 buckets) │ │
│  │  • JVM flag: -XX:StringTableSize=65536 (tune bucket count)     │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```
#### Complete Content Inventory of Heap

| Content | Where in Heap | Notes |
|---|---|---|
| Objects created with `new` | Eden → S0/S1 → Old Gen | Standard lifecycle |
| Arrays | Eden → S0/S1 → Old Gen | Treated like objects |
| String literals | String Pool (Heap) | Since Java 7 |
| `String.intern()` results | String Pool (Heap) | Deduplication |
| Static field **values** | Part of `Class` object in Heap | Since Java 8 |
| `java.lang.Class` mirror objects | Heap | One per loaded class |
| Lambda captures (closures) | Heap | As anonymous class instances |
| Exception objects | Heap | At point of `throw` |

> ⚠️ **Static fields in Java 8+**: The *value* of a static field (e.g., `static int count = 5`) is stored inside the `java.lang.Class` object which resides in the **Heap**. The *metadata* about the field (its name, type, modifiers) is stored in **Metaspace**.

**Key JVM Flags:**
```
-Xms512m                    # initial heap size
-Xmx2g                      # max heap size
-XX:NewRatio=2              # Old:Young ratio = 2:1
-XX:SurvivorRatio=8         # Eden:Survivor ratio = 8:1:1
-XX:MaxTenuringThreshold=15 # age before promotion to Old Gen
```

**OOM types:**
```
java.lang.OutOfMemoryError: Java heap space            # heap full
java.lang.OutOfMemoryError: GC overhead limit exceeded # GC spending >98% time
```

---
### 3.3 JVM Stack (Per-Thread)

**Abstract Concept (JVM Spec):** JVM Stack — stores frames for each method invocation  
**HotSpot Implementation:** Per-thread stack, grows/shrinks with method calls  
**Memory Location:** Thread-local stack memory (in process memory, not heap)  
**Managed By:** JVM automatically — frame pushed on method call, popped on return  
**Cleaned By:** Automatic (LIFO frame push/pop); entire stack freed when thread dies

```
┌──────────────────────────────────────────────────────────────────┐
│              JVM STACK — Thread T1 (Per-Thread, LIFO)            │
│                                                                  │
│  Stack grows downward ▼                                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  STACK FRAME 3 (current: methodC())          ← TOP/Active  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  LOCAL VARIABLE ARRAY (LVA)                          │  │  │
│  │  │  Index 0: 'this' reference (for instance methods)    │  │  │
│  │  │  Index 1: method parameter int x = 5                 │  │  │
│  │  │  Index 2: local int result = 0                       │  │  │
│  │  │  Index 3: object reference → points to Heap object   │  │  │
│  │  │  (stores primitives by VALUE, refs by REFERENCE)     │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  OPERAND STACK                                       │  │  │
│  │  │  • JVM's working area for computations               │  │  │
│  │  │  • Bytecode instructions push/pop values here        │  │  │
│  │  │  • e.g., iadd: pops two ints, pushes sum             │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  FRAME DATA                                          │  │  │
│  │  │  • Constant Pool Reference (→ Metaspace CP)          │  │  │
│  │  │  • Return Address (where to resume caller)           │  │  │
│  │  │  • Exception Table Reference                         │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  STACK FRAME 2 (waiting: methodB())                        │  │
│  │  [same structure as above]                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  STACK FRAME 1 (waiting: methodA() / main())               │  │
│  │  [same structure as above]                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ← Stack Bottom (main() frame, first pushed)                     │
└──────────────────────────────────────────────────────────────────┘
```
#### Content of JVM Stack (What's Stored)

| Content | Location in Frame | Type |
|---|---|---|
| Primitive values (`int`, `long`, `boolean`, etc.) | Local Variable Array | Stored by value |
| Object references | Local Variable Array | Address/pointer → Heap object |
| Method parameters | Local Variable Array (index 1+) | Including primitives by value |
| `this` reference | Local Variable Array (index 0) | For instance methods |
| Intermediate computation results | Operand Stack | Bytecode operations |
| Return address of calling method | Frame Data | To restore caller's PC |
| Exception handler info | Frame Data | Jump targets for try-catch |

> ⚠️ The **JVM Stack stores references to objects, not the objects themselves**. Objects live in the Heap; the stack holds the memory address of where the object is.

**Key JVM Flags:**
```
-Xss512k     # stack size per thread (default: 512KB–1MB depending on OS/JVM)
```

**Errors:**
```
java.lang.StackOverflowError     # stack full (deep/infinite recursion)
java.lang.OutOfMemoryError: unable to create new native thread  # OOM creating thread stacks
```

---
### 3.4 PC Register (Program Counter — Per-Thread)

**Abstract Concept (JVM Spec):** PC (Program Counter) Register — holds address of current instruction  
**HotSpot Implementation:** CPU register or small memory location per thread  
**Memory Location:** Per-thread; essentially free (register-level)  
**Managed By:** JVM automatically  
**Cleaned By:** Freed automatically when thread terminates

```
┌─────────────────────────────────────────────────────┐
│              PC REGISTER (Per-Thread)                │
│                                                     │
│  Thread 1: PC → 0x1A3F  (executing methodA bytecode)│
│  Thread 2: PC → 0x2B7C  (executing methodB bytecode)│
│  Thread 3: PC → undefined (running native method)   │
│                                                     │
│  • For Java methods: holds address of current JVM   │
│    bytecode instruction being executed              │
│  • For native methods: PC is undefined (native      │
│    code manages its own instruction pointer)        │
│  • Updated after every bytecode instruction         │
│  • Enables thread context switching (OS saves/      │
│    restores PC for each thread)                     │
└─────────────────────────────────────────────────────┘
```

**Content Type:** Single value — memory address of the bytecode instruction currently executing

---
### 3.5 Native Method Stack (Per-Thread)

**Abstract Concept (JVM Spec):** Native Method Stack — like JVM Stack but for native (C/C++) methods  
**HotSpot Implementation:** Uses the C-language runtime stack (the OS process stack)  
**Memory Location:** Per-thread, in native process memory  
**Managed By:** OS + JVM  
**Cleaned By:** Automatically freed when the native method returns or thread ends

```
┌──────────────────────────────────────────────────────┐
│          NATIVE METHOD STACK (Per-Thread)            │
│                                                      │
│  • Activated when a Java method calls a native       │
│    method via JNI (e.g., Thread.sleep(), I/O ops)    │
│  • Stores C/C++ stack frames (local vars, return     │
│    addresses) for native code execution              │
│  • Separate from JVM Stack                           │
│  • Can use same memory region as JVM Stack in        │
│    some JVM implementations                          │
│                                                      │
│  Example Call Chain:                                 │
│  Java: main() → Java: Thread.sleep(1000)             │
│    → JVM Stack frame for sleep()                     │
│      → JNI call → Native C code                      │
│        → Native Method Stack frame (OS syscall)      │
└──────────────────────────────────────────────────────┘
```

**Content Type:** C/C++ stack frames — native local variables, native return addresses, OS-level data

**Error:**
```
java.lang.StackOverflowError   # native stack overflow
```

---

## 4. Execution Engine

```
┌─────────────────────────────────────────────────────────────────────┐
│                         EXECUTION ENGINE                            │
│                                                                     │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐    │
│  │    INTERPRETER      │    │         JIT COMPILER             │    │
│  │                     │    │                                  │    │
│  │ • Reads bytecode    │    │  ┌─────────────┐ ┌────────────┐  │    │
│  │   instruction by    │    │  │ C1 Compiler │ │ C2 Compiler│  │    │
│  │   instruction       │    │  │ (Client)    │ │ (Server)   │  │    │
│  │ • No optimization   │    │  │ Fast compile│ │ Heavy opt. │  │    │
│  │ • Slow but          │    │  │ + profiling │ │ Peak perf  │  │    │
│  │   immediate         │    │  └──────┬──────┘ └─────┬──────┘  │    │
│  │                     │    │         │               │        │    │
│  │                     │    │         └───────┬───────┘        │    │
│  └──────────┬──────────┘    │                 ▼                │    │
│             │               │          ┌────────────┐          │    │
│             │               │          │ CODE CACHE │          │    │
│             │               │          │ (off-heap) │          │    │
│             └───────────────┼──────────┤            │          │    │
│                             │          └────────────┘          │    │
│                             └──────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    GARBAGE COLLECTOR                        │    │
│  │  Serial | Parallel | G1 (default) | ZGC | Shenandoah        │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---
### 4.1 Interpreter

- **Function:** Fetches bytecode instructions one at a time from the method bytecodes (in Metaspace), decodes, and executes them
- **Speed:** Slow — each instruction is re-interpreted every time (no caching of execution logic)
- **When used:** At startup and for cold (infrequently called) methods
- **Works with:** Template Interpreter in HotSpot — pre-generated native code snippets for each bytecode, slightly faster than pure software interpretation
- **Advantage:** No warm-up time; starts executing immediately

---
### 4.2 JIT Compiler (Just-In-Time)

The JIT compiler compiles frequently executed bytecode ("hot code") into **native machine code**, which is then cached in the **Code Cache** and executed directly by the CPU — bypassing the interpreter.
#### How "Hot" Code is Detected — Profiling Counters

```
Each method/loop has two counters in HotSpot:

  Method Invocation Counter       Back-Edge Counter
  ┌────────────────────────┐     ┌────────────────────────────┐
  │ Incremented each time  │     │ Incremented each loop      │
  │ method is called       │     │ iteration (back-edge of    │
  │                        │     │ bytecode = loop)           │
  └────────────────────────┘     └────────────────────────────┘
         │                               │
         └───────────────┬───────────────┘
                         ▼
              Sum > CompileThreshold?
              → Submit to JIT for compilation
              → Default: -XX:CompileThreshold=10000 (C2)
```

#### Tiered Compilation (Default since Java 8)

```
   Method first called
         │
         ▼
  ┌──────────────┐
  │  Level 0     │  Interpreted (no profiling)
  │  Interpreter │
  └──────┬───────┘
         │ invocation count > tier1_threshold
         ▼
  ┌──────────────┐
  │  Level 1     │  C1 compiled, no profiling
  │  C1 Simple   │  (fastest C1 — simple methods)
  └──────┬───────┘
         │ or needs more info
         ▼
  ┌──────────────┐
  │  Level 2     │  C1 compiled + limited profiling
  │  C1 Limited  │  (invocation/back-edge counters only)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Level 3     │  C1 compiled + FULL profiling
  │  C1 Full     │  (branch profiling, type profiling)
  │  Profiled    │  ← Most time spent here collecting data
  └──────┬───────┘
         │ profiling data ripe → hot enough
         ▼
  ┌──────────────┐
  │  Level 4     │  C2 compiled (server optimizer)
  │  C2 Fully    │  Maximum optimization using profiling data
  │  Optimized   │  Inlining, loop unrolling, escape analysis,
  └──────────────┘  dead code elimination, etc.
```
#### C1 vs C2 Compiler Comparison

| Feature | C1 (Client Compiler) | C2 (Server Compiler) |
|---|---|---|
| Compile speed | Fast | Slow (heavy analysis) |
| Code quality | Moderate | Best — peak performance |
| Optimizations | Basic inlining, constant folding | Aggressive: loop unrolling, escape analysis, scalar replacement, vectorization |
| Profiling | Instruments code to collect data | Uses C1 profiling data to optimize |
| Threshold | ~1,500–2,000 invocations (tiered) | ~10,000–15,000 invocations |
#### Key JIT Optimizations (C2)
- **Inlining**: Replaces method call with method body — reduces call overhead
- **Escape Analysis**: If an object doesn't escape a method/thread → allocate on stack (not heap) or eliminate entirely
- **Loop Unrolling**: Replicate loop body to reduce branch overhead
- **Dead Code Elimination**: Remove code that can never execute
- **Constant Folding**: `3 * 4` → `12` at compile time
- **Deoptimization**: If assumptions are violated (e.g., a new subclass is loaded), JIT reverts to interpreted mode

---
### 4.3 Code Cache

**Not part of JVM Spec** — HotSpot implementation detail  
**Location:** Off-heap native memory  
**Content:** JIT-compiled native machine code  
**Managed By:** JVM's Code Cache Sweeper (background thread)  
**Cleaned By:** Code Cache Flusher (removes old/unused compiled code when cache fills)

```
┌──────────────────────────────────────────────────────────────────┐
│              CODE CACHE (Off-Heap, Segmented since Java 9)       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  NON-METHOD SEGMENT                                        │  │
│  │  • JVM internal code (stubs, adapters)                     │  │
│  │  • Interpreter generated code                              │  │
│  │  • Non-nmethod code (JVM runtime)                          │  │
│  │  • Default size: 5–7 MB                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  PROFILED-CODE SEGMENT                                     │  │
│  │  • C1-compiled methods (Level 1–3)                         │  │
│  │  • Contains instrumentation/profiling hooks                │  │
│  │  • Temporary — will be replaced by C2 output               │  │
│  │  • Default: ~120 MB                                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  NON-PROFILED CODE SEGMENT                                 │  │
│  │  • C2-compiled methods (Level 4, fully optimized)          │  │
│  │  • Long-lived — these are the "hottest" methods            │  │
│  │  • Default: ~120 MB                                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Total default: ~240 MB (-XX:ReservedCodeCacheSize=240m)         │
└──────────────────────────────────────────────────────────────────┘
```

**Key JVM Flags:**
```
-XX:ReservedCodeCacheSize=256m   # total code cache size
-XX:+UseCodeCacheFlushing        # enable flushing when full (default: true)
-XX:+PrintCodeCache              # print code cache stats at shutdown
```

**Error when Code Cache is full:**
```
# JVM will log: "CodeCache is full. Compiler has been disabled."
# Performance degrades back to interpreted mode
```

---
### 4.4 Garbage Collector

#### GC Memory Regions Quick View

```
  OBJECT LIFECYCLE IN HEAP:

  new MyObj()
      │
      ▼
  ┌──────────┐   Minor GC  ┌─────────┐   Minor GC    ┌─────────┐
  │  EDEN    │────────────▶│  S0/S1  │──────────────▶│ OLD GEN │
  │          │  if alive   │  age+1  │ if age >= 15  │         │
  └──────────┘             └─────────┘               └─────────┘
  
  Minor GC:  Eden + active Survivor → other Survivor (STW, fast)
  Major GC:  Old Generation (STW or concurrent depending on GC)
  Full GC:   Young + Old + Metaspace (STW, most expensive)
```
#### Modern GC Algorithms Comparison

| GC              | Flag                   | Default Since      | STW Pauses    | Throughput | Latency  | Heap Size      |
| --------------- | ---------------------- | ------------------ | ------------- | ---------- | -------- | -------------- |
| **Serial GC**   | `-XX:+UseSerialGC`     | Client JVM         | Long STW      | Low        | High     | Small (<100MB) |
| **Parallel GC** | `-XX:+UseParallelGC`   | Java 7 default     | Long STW      | Highest    | High     | Medium         |
| **G1 GC**       | `-XX:+UseG1GC`         | **Java 9 default** | Short STW     | Good       | Low      | 4GB–10GB       |
| **ZGC**         | `-XX:+UseZGC`          | Java 15 (prod)     | Sub-ms (<1ms) | Good       | Very Low | 8MB–16TB       |
| **Shenandoah**  | `-XX:+UseShenandoahGC` | OpenJDK only       | Sub-ms        | Good       | Very Low | Large          |
> ❌ **CMS (Concurrent Mark Sweep)** — Deprecated in Java 9, removed in Java 14. Do not use or mention as current.
#### G1 GC — Deep Dive (Default GC)

G1 (Garbage First) breaks the heap into equal-sized **regions** (1–32 MB each) and collects the regions with the most garbage first.

```
┌────────────────────────────────────────────────────────────────────┐
│                     G1 GC HEAP LAYOUT                              │
│                                                                    │
│  Each cell = one Region (1MB–32MB, power of 2, configurable)       │
│                                                                    │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ E │ E │ S │ O │ O │ E │ H │ H │ O │ E │ S │ O │ E │ E │ O │     │
│  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤     │
│  │ O │ E │ O │ E │ S │ E │ O │ O │ E │ O │ O │ E │ S │ O │ E │     │
│  ├───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┤     │
│  │ F │ E │ O │ H │ H │ O │ E │ O │ E │ S │ O │ F │ E │ O │ E │     │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘     │
│                                                                    │
│  E = Eden    S = Survivor    O = Old    H = Humongous    F = Free  │
│                                                                    │
│  Humongous regions: for objects > 50% of region size               │
│  Allocated directly in Old Gen; collected in Cleanup or Full GC    │
└────────────────────────────────────────────────────────────────────┘
```
#### G1 GC Phases

```
Phase 1: YOUNG-ONLY PHASE (default, ongoing)
  │
  ├── Young GC (Minor GC) — STW
  │   • Evacuates Eden + Survivor regions to new Survivor regions
  │   • Tenures objects to Old Gen if age threshold reached
  │
  └── Concurrent Marking Cycle (triggered when Old Gen fills ~45%)
        │
        ├── Initial Mark (STW, piggybacks on Young GC)
        │   • Mark GC roots
        │
        ├── Root Region Scan (Concurrent)
        │   • Scan survivor regions for refs into old gen
        │
        ├── Concurrent Mark (Concurrent)
        │   • Traverse object graph, mark live objects
        │   • Uses SATB (Snapshot At The Beginning)
        │
        ├── Remark (STW)
        │   • Finalize marking
        │   • Process SATB buffers
        │
        └── Cleanup (STW + Concurrent)
            • Accounting: determine which regions are mostly garbage
            • Free empty regions immediately
            • Select regions for Mixed GC

Phase 2: SPACE RECLAMATION / MIXED GC
  • Collects Young regions + selected Old regions (most garbage)
  • STW, like Young GC but includes old regions in CSet
  • Continues until no profitable old regions remain or heap shrinks
  • Then back to Young-Only Phase
```

**Key G1 Flags:**
```
-XX:MaxGCPauseMillis=200               # target max pause (default: 200ms)
-XX:G1HeapRegionSize=16m               # region size (1–32m, power of 2)
-XX:G1NewSizePercent=5                 # min young gen %
-XX:G1MaxNewSizePercent=60             # max young gen %
-XX:InitiatingHeapOccupancyPercent=45  # trigger concurrent marking
-XX:G1MixedGCCountTarget=8             # mixed GC cycles to space reclamation
```
#### ZGC — Key Concepts (Java 15+ Production)

```
┌──────────────────────────────────────────────────────────────┐
│  ZGC KEY DESIGN PRINCIPLES                                   │
│                                                              │
│  1. COLORED POINTERS                                         │
│     64-bit pointer = [metadata bits 4] + [address bits 44]   │
│     Metadata bits: Marked0, Marked1, Remapped, Finalizable   │
│     → GC reads/sets color bits via LOAD BARRIERS             │
│                                                              │
│  2. LOAD BARRIERS (not write barriers)                       │
│     Injected by JIT into every object reference load         │
│     → If pointer "bad color" → fix pointer concurrently      │
│     → Allows relocation while app threads run                │
│                                                              │
│  3. NO GENERATIONAL (pre-Java 21)                            │
│     Single generation heap; all objects treated equally      │
│     Java 21: Generational ZGC introduced (better throughput) │
│                                                              │
│  4. PAUSE TIMES: O(1) — independent of heap size             │
│     Typical: < 1ms                                           │
│     Only STW phases: initial mark, remark                    │
└──────────────────────────────────────────────────────────────┘
```

---
## 5. Native Method Interface (JNI)

```
┌────────────────────────────────────────────────────────────────┐
│                 JNI — Java Native Interface                    │
│                                                                │
│  Java World                   │  Native World (C/C++)          │
│  ─────────────────────────────│──────────────────────────────  │
│  public native void sort();   │  JNIEXPORT void JNICALL        │
│  System.loadLibrary("mylib")  │  Java_MyClass_sort(JNIEnv*,    │
│                               │    jobject this) { ... }       │
│                               │                                │
│  JNI Functions:               │  Accessed via:                 │
│  • NewObject(), GetField()    │  env->FindClass()              │
│  • CallMethod(), NewArray()   │  env->GetMethodID()            │
│  • GetStringUTFChars()        │  env->CallObjectMethod()       │
│                               │                                │
│  Used by: File I/O, sockets,  │  Security concern: native      │
│  cryptography, GPU/OpenGL,    │  code bypasses JVM safety      │
│  java.lang.System             │  checks & GC boundaries        │
└────────────────────────────────────────────────────────────────┘
```

- **JNI is the bridge** between Java bytecode (JVM) and native machine code (OS/C libraries)
- `System.loadLibrary("awt")` loads a `.dll` / `.so` / `.dylib` into the process
- JNI calls are expensive — context switch between JVM and native frame, GC roots must be tracked at boundaries

---
## 6. Common OOM Errors — Root Cause Map

```
java.lang.OutOfMemoryError: Java heap space
  └── Too many live objects; GC cannot reclaim enough
  └── Memory leak: objects held by static refs, caches, listeners
  └── Fix: -Xmx, heap dump analysis, fix leaks

java.lang.OutOfMemoryError: GC overhead limit exceeded
  └── GC is spending >98% of time but reclaiming <2% heap
  └── Effectively: heap full but GC is thrashing
  └── Fix: same as above; reduce allocation rate

java.lang.OutOfMemoryError: Metaspace
  └── Too many classes loaded and not unloaded
  └── CGLib/ByteBuddy/reflection generating classes at runtime
  └── Fix: -XX:MaxMetaspaceSize, fix classloader leaks

java.lang.OutOfMemoryError: unable to create new native thread
  └── Too many threads; each thread needs stack memory
  └── OS limit on thread count reached
  └── Fix: reduce thread count, reduce -Xss size

java.lang.OutOfMemoryError: Direct buffer memory
  └── NIO DirectByteBuffer exhausted native memory
  └── Fix: -XX:MaxDirectMemorySize

java.lang.StackOverflowError
  └── JVM Stack depth exceeded (infinite/deep recursion)
  └── Fix: reduce recursion depth; increase -Xss (cautiously)

java.lang.OutOfMemoryError: request <size> bytes for <type>. Out of swap space?
  └── JVM cannot allocate off-heap native memory
  └── Process total memory exceeds available RAM + swap
```

---
## 7. JVM Flags Cheat Sheet (Interview-Ready)

```
HEAP
  -Xms2g                         # min heap
  -Xmx4g                         # max heap
  -XX:NewRatio=2                 # Old:Young = 2:1
  -XX:SurvivorRatio=8            # Eden:Survivor = 8:1:1

GC SELECTION
  -XX:+UseG1GC                   # G1 (default Java 9+)
  -XX:+UseZGC                    # ZGC (production Java 15+)
  -XX:+UseParallelGC             # Throughput GC
  -XX:+UseSerialGC               # Serial (single-threaded)

G1 TUNING
  -XX:MaxGCPauseMillis=200       # pause goal
  -XX:G1HeapRegionSize=16m       # region size

METASPACE
  -XX:MetaspaceSize=128m         # initial
  -XX:MaxMetaspaceSize=512m      # hard cap

JIT / CODE CACHE
  -XX:ReservedCodeCacheSize=256m
  -XX:+TieredCompilation         # on by default
  -XX:CompileThreshold=10000     # C2 threshold

GC LOGGING (Java 9+)
  -Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m

GC DIAGNOSTICS
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/tmp/heapdump.hprof
  -XX:+PrintGCDetails             # Java 8 style
  -verbose:gc

THREAD STACK
  -Xss512k                       # stack size per thread

STRING TABLE
  -XX:StringTableSize=65536      # hash buckets for String pool
```

---
## 8. Full Mental Model — How They All Connect

```
Source Code (.java)
       │
       │ javac
       ▼
  Bytecode (.class)
       │
       │ JVM starts
       ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ CLASS LOADER: finds .class → loads bytecode → creates       │
  │ java.lang.Class object in HEAP, class metadata in METASPACE │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ EXECUTION ENGINE: main thread starts                        │
  │                                                             │
  │ Thread created:                                             │
  │   • JVM Stack allocated (per thread)                        │
  │   • PC Register initialized (per thread)                   │
  │   • Native Method Stack allocated (per thread)              │
  │                                                             │
  │ Method called → Stack Frame pushed onto JVM Stack           │
  │   • LVA: local vars + params stored                        │
  │   • Operand Stack: computation workspace                    │
  │                                                             │
  │ Bytecode executed:                                          │
  │   → INTERPRETER: first few thousand executions             │
  │   → C1 JIT: after invocation threshold → compiled to       │
  │             native code + profiling instrumented           │
  │   → C2 JIT: after profiling data ripe → peak optimized     │
  │             native code stored in CODE CACHE               │
  │                                                             │
  │ new Object() → allocated in HEAP (Eden/TLAB)               │
  │ "hello" → stored in STRING POOL (part of Heap)             │
  │ static int x = 5 → value in Class object in HEAP           │
  │                                                             │
  │ When heap fills → GC runs:                                  │
  │   Minor GC: Eden + Survivor → copy live to other Survivor  │
  │   Mixed GC (G1): Young + some Old regions                  │
  │   Full GC: all heap + Metaspace                             │
  │                                                             │
  │ Method returns → Stack Frame popped (automatic cleanup)     │
  │ Thread ends → Stack + PC + NativeStack freed                │
  └─────────────────────────────────────────────────────────────┘
```

---
## Key Interview Distinctions (Summary)

| Question | Answer |
|---|---|
| Where do `static` field **values** live? | **Heap** (inside `Class` object) since Java 8 |
| Where does class **metadata** live? | **Metaspace** (off-heap) since Java 8 |
| Where does `String pool` live? | **Heap** since Java 7 |
| What replaced PermGen? | **Metaspace** in Java 8 |
| Who manages Metaspace? | **OS** (native memory), with JVM's allocator on top |
| Who manages Heap? | **JVM** via Garbage Collector |
| Who manages JVM Stack? | **JVM** automatically (LIFO frame management) |
| What's stored in JVM Stack? | Local vars, method params, object **references** (not objects), operand stack, return address |
| What's stored in Code Cache? | JIT-compiled **native machine code** |
| Is Code Cache in JVM Spec? | **No** — HotSpot-specific implementation detail |
| What causes `StackOverflowError`? | JVM Stack full — deep/infinite recursion |
| What causes `OutOfMemoryError: Metaspace`? | Too many classes loaded — classloader leak / proxy generation |
| What is TLAB? | Thread-Local Allocation Buffer — each thread has its own buffer in Eden for lock-free object allocation |
| What is tiered compilation? | Levels 0→4: Interpreter → C1 (simple) → C1 (profiled) → C2 (peak) |
| What does escape analysis do? | Allows stack allocation / scalar replacement of objects that don't escape method/thread |
