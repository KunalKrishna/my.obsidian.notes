
# JVM: Stack, Heap, and Method Area — Complete Tutorial

---

## Why This Matters for Concurrency (Before We Start)

```
Every concurrency bug you will ever face —
race conditions, visibility problems, memory leaks —
traces back to ONE question:

  "Where does this piece of data live in memory,
   and which threads can see it?"

  The answer is always:
    Stack  → only ONE thread can see it   (private)
    Heap   → ALL threads can see it       (shared)
    Method Area → ALL threads can see it  (shared)

  If data is on the Stack   → no concurrency problem possible
  If data is on the Heap    → concurrency problem is possible
  If data is in Method Area → concurrency problem is possible

  That is the entire foundation.
  Everything else in concurrency is about protecting Heap data.
```

---

## The Big Picture — JVM Memory Layout

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         JVM MEMORY LAYOUT                                ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐    ║
║   │                        HEAP                                     │    ║
║   │              (shared by ALL threads)                            │    ║
║   │                                                                 │    ║
║   │   ┌─────────────────────────────┐  ┌────────────────────────┐   │    ║
║   │   │       Young Generation      │  │    Old Generation      │   │    ║
║   │   │  ┌──────────┐ ┌──────────┐  │  │    (Tenured Space)     │   │    ║
║   │   │  │  Eden    │ │Survivor  │  │  │                        │   │    ║
║   │   │  │  Space   │ │S0  │  S1 │  │  │  Long-lived objects    │   │    ║
║   │   │  └──────────┘ └──────────┘  │  └────────────────────────┘   │    ║
║   │   └─────────────────────────────┘                               │    ║
║   └─────────────────────────────────────────────────────────────────┘    ║
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐    ║
║   │                    METHOD AREA (Metaspace)                      │    ║
║   │                  (shared by ALL threads)                        │    ║
║   │                                                                 │    ║
║   │   Class definitions · Static variables · Constants              │    ║
║   │   Method bytecode  · Runtime constant pool                      │    ║
║   └─────────────────────────────────────────────────────────────────┘    ║
║                                                                          ║
║   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   ║
║   │  JVM STACK   │  │  JVM STACK   │  │  JVM STACK   │  ...              ║
║   │  Thread 1    │  │  Thread 2    │  │  Thread 3    │                   ║
║   │  (private)   │  │  (private)   │  │  (private)   │                   ║
║   │              │  │              │  │              │                   ║
║   │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │                   ║
║   │ │  Frame 3 │ │  │ │  Frame 2 │ │  │ │  Frame 1 │ │                   ║
║   │ ├──────────┤ │  │ ├──────────┤ │  │ ├──────────┤ │                   ║
║   │ │  Frame 2 │ │  │ │  Frame 1 │ │  │ │  Frame 0 │ │                   ║
║   │ ├──────────┤ │  │ ├──────────┤ │  │ └──────────┘ │                   ║
║   │ │  Frame 1 │ │  │ │  Frame 0 │ │  │              │                   ║
║   │ └──────────┘ │  │ └──────────┘ │  │              │                   ║
║   └──────────────┘  └──────────────┘  └──────────────┘                   ║
║                                                                          ║
║   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   ║
║   │   PC Reg     │  │   PC Reg     │  │   PC Reg     │  one per thread   ║
║   │   Thread 1   │  │   Thread 2   │  │   Thread 3   │                   ║
║   └──────────────┘  └──────────────┘  └──────────────┘                   ║
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐    ║
║   │              NATIVE METHOD STACK                                │    ║
║   │         (for native/JNI method calls per thread)                │    ║
║   └─────────────────────────────────────────────────────────────────┘    ║
╚══════════════════════════════════════════════════════════════════════════╝

SHARED  (all threads can read/write): Heap + Method Area
PRIVATE (only owning thread):         JVM Stack + PC Register
```

---

## Part 1 — The JVM Stack

### What It Is

```
Every thread in the JVM gets its OWN private stack.
Created when the thread is created.
Destroyed when the thread dies.
No other thread can access it.

The stack holds STACK FRAMES.
One stack frame is pushed every time a method is called.
One stack frame is popped  every time a method returns.

Think of it as:
  Stack = the call history of a thread
  Frame = one method call's entire local universe
```

### What Is Inside a Stack Frame

```
Each Stack Frame contains exactly THREE things:

┌─────────────────────────────────────────────────────────────┐
│                      STACK FRAME                            │
│                   (one per method call)                     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. LOCAL VARIABLE ARRAY                             │  │
│  │                                                      │  │
│  │     Slot 0: 'this' reference (for instance methods)  │  │
│  │     Slot 1: first parameter                          │  │
│  │     Slot 2: second parameter                         │  │
│  │     Slot 3: first local variable                     │  │
│  │     Slot 4: second local variable                    │  │
│  │     ...                                              │  │
│  │                                                      │  │
│  │  Stores: int, long, float, double, boolean, char,    │  │
│  │          byte, short (primitives — stored BY VALUE)  │  │
│  │          Object references (stored BY REFERENCE,     │  │
│  │          actual object is on the Heap)               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  2. OPERAND STACK                                    │  │
│  │                                                      │  │
│  │     Working area for bytecode instructions.          │  │
│  │     JVM pushes operands here, executes operations,   │  │
│  │     pops results.                                    │  │
│  │     Think of it as the CPU's scratch pad.            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  3. FRAME DATA (Reference Data)                      │  │
│  │                                                      │  │
│  │     Reference to runtime constant pool               │  │
│  │     Return address (where to go after method returns)│  │
│  │     Exception table reference                        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Stack in Action — Tracing Through Code

```java
public class StackDemo {

    public static void main(String[] args) {    // Frame 1 pushed
        int x = 10;                             // x lives in Frame 1
        int y = 20;                             // y lives in Frame 1
        int result = add(x, y);                 // Frame 2 pushed
        System.out.println(result);             // Frame 3 pushed, popped
    }                                           // Frame 1 popped

    public static int add(int a, int b) {       // Frame 2
        int sum = a + b;                        // sum lives in Frame 2
        return sum;                             // Frame 2 popped
    }                                           // sum is GONE — memory reclaimed
}
```

**Stack state at each point:**

```
POINT 1: main() starts
═══════════════════════

  Thread Stack (main thread)
  ┌──────────────────────┐  ← TOP
  │  Frame: main()       │
  │  local vars:         │
  │    args = ref→[]     │
  │    x    = (not set)  │
  │    y    = (not set)  │
  │    result=(not set)  │
  └──────────────────────┘

POINT 2: x=10, y=20 assigned, add(x,y) called
═══════════════════════════════════════════════

  Thread Stack (main thread)
  ┌──────────────────────┐  ← TOP (new frame pushed)
  │  Frame: add()        │
  │  local vars:         │
  │    a   = 10          │  ← copy of x (pass by value)
  │    b   = 20          │  ← copy of y (pass by value)
  │    sum = (not set)   │
  └──────────────────────┘
  ┌──────────────────────┐
  │  Frame: main()       │
  │  local vars:         │
  │    args   = ref→[]   │
  │    x      = 10       │
  │    y      = 20       │
  │    result = (waiting)│
  └──────────────────────┘

POINT 3: sum = a + b computed inside add()
═══════════════════════════════════════════

  Thread Stack (main thread)
  ┌──────────────────────┐  ← TOP
  │  Frame: add()        │
  │  local vars:         │
  │    a   = 10          │
  │    b   = 20          │
  │    sum = 30          │
  └──────────────────────┘
  ┌──────────────────────┐
  │  Frame: main()       │
  └──────────────────────┘

POINT 4: add() returns — Frame popped
══════════════════════════════════════

  Thread Stack (main thread)
  ┌──────────────────────┐  ← TOP
  │  Frame: main()       │
  │  local vars:         │
  │    args   = ref→[]   │
  │    x      = 10       │
  │    y      = 20       │
  │    result = 30       │  ← return value captured here
  └──────────────────────┘

  Frame: add() is GONE.
  Variables a, b, sum are GONE.
  Memory instantly reclaimed — no GC needed for stack frames.
```

### References on the Stack — The Critical Concept

```java
public class StackHeapInteraction {

    public static void main(String[] args) {

        // primitive — VALUE lives on Stack directly
        int age = 25;
        //  Stack Frame:
        //    age = 25    ← the number 25 is stored right here on stack

        // object — REFERENCE lives on Stack, OBJECT lives on Heap
        String name = new String("Alice");
        //  Stack Frame:
        //    name = 0x7f3a  ← memory address (reference/pointer)
        //                      the actual String object is on the HEAP
        //                      at address 0x7f3a

        Person person = new Person("Bob", 30);
        //  Stack Frame:
        //    person = 0x8b2c  ← reference to Person object on Heap

        // ── PASSING TO METHOD ──────────────────────────────────────
        modifyPrimitive(age);
        // age is COPIED to new frame
        // original age on THIS frame unchanged

        modifyObject(person);
        // person REFERENCE is copied to new frame
        // BOTH frames now have a reference to SAME Person object on Heap
        // modifications to the object AFFECT the Heap object
        // (both references point to same thing)
    }

    static void modifyPrimitive(int a) {
        a = 999;
        // a is a local copy in THIS frame
        // original age in main's frame: UNCHANGED
        // When this frame is popped: a is gone, nothing affected
    }

    static void modifyObject(Person p) {
        p.name = "Charlie";
        // p is a reference pointing to same Person on Heap
        // p.name = "Charlie" MODIFIES the Heap object
        // When main() reads person.name → sees "Charlie"
        // The Heap object was changed — both references see the change
    }
}
```

```
STACK vs HEAP for variables:

  int age = 25;
  ┌────────────────────────────────────────────────────┐
  │ STACK FRAME                                        │
  │   age ──────────────────► 25                       │
  │                           (value lives here)       │
  └────────────────────────────────────────────────────┘

  Person person = new Person("Bob", 30);
  ┌────────────────────────────────────────────────────┐
  │ STACK FRAME                                        │
  │   person ───────────────► 0x8b2c (reference/addr) │
  └──────────────────────────────┬─────────────────────┘
                                 │ points to
                                 ▼
  ┌────────────────────────────────────────────────────┐
  │ HEAP                                               │
  │   [0x8b2c]  Person object                         │
  │               name = "Bob"                        │
  │               age  = 30                           │
  └────────────────────────────────────────────────────┘
```

### StackOverflowError — What It Means

```java
public class StackOverflow {

    // Each recursive call pushes a NEW frame onto the stack
    // Stack has a fixed size (default ~512KB - 1MB per thread, tunable with -Xss)
    // Eventually: no more space → StackOverflowError

    public static void infiniteRecursion(int n) {
        // Each call: new frame pushed with local var n
        System.out.println("Depth: " + n);
        infiniteRecursion(n + 1);  // ← keeps pushing frames
        // Eventually stack runs out of space
        // JVM throws: java.lang.StackOverflowError
    }

    public static void main(String[] args) {
        infiniteRecursion(0);
    }
}
```

```
STACK GROWING — frame by frame:

  infiniteRecursion(0) called:
  ┌────────────────┐
  │ frame: n=0     │
  └────────────────┘

  infiniteRecursion(1) called:
  ┌────────────────┐  ← TOP
  │ frame: n=1     │
  ├────────────────┤
  │ frame: n=0     │
  └────────────────┘

  ... thousands of frames later ...

  ┌────────────────┐  ← TOP
  │ frame: n=9873  │
  ├────────────────┤
  │ frame: n=9872  │
  ├────────────────┤
  │ ...            │
  ├────────────────┤
  │ frame: n=0     │
  └────────────────┘
         ↓
  StackOverflowError!  (no more stack space)

JVM flag to change stack size per thread:
  -Xss512k    → 512 KB stack per thread
  -Xss2m      → 2 MB stack per thread
```

### Stack and Concurrency — The Key Point

```java
public class StackConcurrencySafety {

    // This method is called by 10 threads simultaneously
    // Is it thread-safe?
    public int calculateTax(int income) {
        int taxRate  = 30;             // local — goes on THAT THREAD'S stack
        int tax      = income * taxRate / 100; // local — on THAT THREAD'S stack
        int afterTax = income - tax;   // local — on THAT THREAD'S stack
        return afterTax;
        // ALL these variables are on the calling thread's private stack frame
        // NO sharing between threads
        // NO synchronization needed
        // 100% thread-safe by design
    }
}
```

```
Thread 1 calling calculateTax(50000):    Thread 2 calling calculateTax(80000):

  STACK of Thread 1:                       STACK of Thread 2:
  ┌──────────────────────┐                 ┌──────────────────────┐
  │ frame: calculateTax  │                 │ frame: calculateTax  │
  │   income   = 50000   │                 │   income   = 80000   │
  │   taxRate  = 30      │                 │   taxRate  = 30      │
  │   tax      = 15000   │                 │   tax      = 24000   │
  │   afterTax = 35000   │                 │   afterTax = 56000   │
  └──────────────────────┘                 └──────────────────────┘
           ↑                                         ↑
    Thread 1's private                        Thread 2's private
    stack — invisible                         stack — invisible
    to Thread 2                               to Thread 1

CONCLUSION:
  Local variables and method parameters
  are ALWAYS thread-safe — they live on the private stack.
  No synchronization needed for these EVER.
```

---

## Part 2 — The Heap

### What It Is

```
The Heap is the LARGEST memory area in the JVM.
It is created when the JVM starts.
It is shared by ALL threads.
It stores ALL objects and arrays created with 'new'.
It is managed by the Garbage Collector.

The Heap is where concurrency problems live.
Every race condition, every visibility bug,
every data corruption — happens on the Heap.
```

### What Lives on the Heap

```java
public class WhatLivesOnHeap {

    // ── INSTANCE VARIABLES (fields) → ALWAYS on Heap ─────────────
    // These live ON the object which lives on the Heap
    private int    instanceCount = 0;       // on Heap (part of object)
    private String instanceName  = "test";  // ref on Heap, String obj on Heap
    private int[]  instanceArray = new int[10]; // both ref and array on Heap

    // ── STATIC VARIABLES → in Method Area (discussed later) ───────
    private static int staticCount = 0;     // NOT on Heap — in Method Area

    public void demonstrate() {

        // ── LOCAL VARIABLES → on Stack ────────────────────────────
        int localPrimitive = 42;        // value on Stack
        String localRef    = "hello";   // reference on Stack,
                                        // String object on Heap

        // ── OBJECTS CREATED WITH 'new' → ALWAYS on Heap ───────────
        Person person = new Person();   // Person object on Heap
                                        // reference 'person' on Stack

        int[] array   = new int[5];     // array on Heap
                                        // reference 'array' on Stack

        // ── IMPORTANT: even if reference is local (stack),
        //    the OBJECT it points to is on the Heap
        //    and SHARED if multiple references point to same object
    }
}
```

```
HEAP CONTENTS — visual:

┌──────────────────────────────────────────────────────────────────┐
│                            HEAP                                  │
│                                                                  │
│  [0x1000] Person object                                          │
│             name = ref→[0x2000]                                  │
│             age  = 25                                            │
│             salary = 50000.0                                     │
│                                                                  │
│  [0x2000] String object "Alice"                                  │
│             value = ref→[0x3000] (char array)                    │
│             hash  = 0                                            │
│                                                                  │
│  [0x3000] char[] {'A','l','i','c','e'}                           │
│                                                                  │
│  [0x4000] int[] {0, 0, 0, 0, 0}                                  │
│                                                                  │
│  [0x5000] BankAccount object                                     │
│             balance  = 1000.0   ← shared mutable state!          │
│             owner    = ref→[0x1000]                              │
│                                                                  │
│  ... thousands more objects ...                                  │
└──────────────────────────────────────────────────────────────────┘

Objects on Heap are accessible from:
  - Stack references (local variables pointing to them)
  - Other Heap objects (fields in objects pointing to other objects)
  - Method Area (static fields pointing to objects)
```

### Heap Structure — Young and Old Generation

```
HEAP STRUCTURE (how GC manages it)
════════════════════════════════════════════════════════════════════

┌───────────────────────────────────────────────────────────────────┐
│                           HEAP                                    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   YOUNG GENERATION                         │  │
│  │          (most objects die young — short-lived)            │  │
│  │                                                            │  │
│  │  ┌──────────────────┐  ┌────────────┐  ┌───────────────┐  │  │
│  │  │   EDEN SPACE     │  │ SURVIVOR 0 │  │  SURVIVOR 1   │  │  │
│  │  │                  │  │  (S0/From) │  │  (S1/To)      │  │  │
│  │  │  New objects     │  │            │  │               │  │  │
│  │  │  born here       │  │  Survived  │  │  Survived     │  │  │
│  │  │  first           │  │  1+ GC     │  │  2+ GC        │  │  │
│  │  └──────────────────┘  └────────────┘  └───────────────┘  │  │
│  │                                                            │  │
│  │  GC here = Minor GC (fast, frequent, stops only young gen) │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   OLD GENERATION                           │  │
│  │               (Tenured Space)                              │  │
│  │         (long-lived objects promoted here)                 │  │
│  │                                                            │  │
│  │  Objects that survived many GC cycles in Young Gen        │  │
│  │  Large objects (may go directly here)                     │  │
│  │  Static variable referenced objects often here            │  │
│  │                                                            │  │
│  │  GC here = Major GC / Full GC (slow, infrequent)          │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘

OBJECT LIFECYCLE ON HEAP:

  1. Object created with 'new'
       ↓
  2. Born in Eden Space
       ↓
  3. Minor GC happens — if object still referenced → moves to S0
       ↓
  4. Next Minor GC — if still referenced → moves to S1
       ↓
  5. After threshold (default 15 GC cycles) → promoted to Old Gen
       ↓
  6. Eventually unreachable (no references pointing to it)
       ↓
  7. Garbage Collected — memory reclaimed
```

### Heap Concurrency Problem — In Code

```java
// ═══════════════════════════════════════════════════════════════
//  THE HEAP SHARING PROBLEM
//  Two threads modifying the SAME object on the Heap
// ═══════════════════════════════════════════════════════════════
public class HeapSharingProblem {

    // BankAccount object lives on the HEAP
    // SHARED by Thread 1 and Thread 2
    static BankAccount account = new BankAccount(1000);

    static class BankAccount {
        double balance;  // this field lives on the HEAP (part of object)

        BankAccount(double initialBalance) {
            this.balance = initialBalance;
        }

        // NOT thread-safe — balance is on Heap, multiple threads can modify
        public void deposit(double amount) {
            balance += amount;  // READ balance from Heap, ADD, WRITE to Heap
                                // three operations — not atomic!
        }

        public void withdraw(double amount) {
            balance -= amount;  // same problem
        }
    }

    public static void main(String[] args) throws InterruptedException {

        Thread depositor = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                account.deposit(10);    // modifying Heap object
            }
        });

        Thread withdrawer = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                account.withdraw(10);   // modifying SAME Heap object
            }
        });

        depositor.start();
        withdrawer.start();
        depositor.join();
        withdrawer.join();

        // Expected: 1000 (deposits and withdrawals cancel out)
        // Actual:   unpredictable — could be anything
        System.out.println("Balance: " + account.balance);
    }
}
```

```
WHY IT FAILS — both threads share the SAME Heap object:

  HEAP:
  ┌──────────────────────┐
  │  BankAccount object  │
  │    balance = 1000.0  │ ← BOTH threads read/write this
  └──────────────────────┘
         ▲           ▲
         │           │
  Thread 1       Thread 2
  Stack:         Stack:
  ref→account    ref→account  ← SAME reference to SAME Heap object

  Thread 1: reads balance=1000  (from Heap / working memory)
  Thread 2: reads balance=1000  (from Heap / working memory)
  Thread 1: computes 1000+10=1010
  Thread 2: computes 1000-10=990
  Thread 1: writes balance=1010 (to Heap)
  Thread 2: writes balance=990  (to Heap) ← OVERWRITES Thread 1's write!
  Net:      balance=990  instead of 1000  ← WRONG
```

### OutOfMemoryError — What It Means

```java
public class HeapExhaustion {

    public static void main(String[] args) {

        // Keep creating objects — never releasing references
        // GC cannot collect them (we hold references)
        // Eventually Heap runs out of space
        // JVM throws: java.lang.OutOfMemoryError: Java heap space

        List<byte[]> hoarder = new ArrayList<>();
        while (true) {
            hoarder.add(new byte[1024 * 1024]); // 1MB per allocation
            System.out.println("Allocated: " + hoarder.size() + " MB");
        }
    }
}
```

```
JVM flags for Heap:
  -Xms256m    → initial Heap size 256 MB
  -Xmx1g      → maximum Heap size 1 GB
  -Xmn256m    → Young Generation size 256 MB

  Set -Xms == -Xmx in production:
    prevents JVM from resizing heap dynamically
    avoids GC pauses caused by heap expansion
```

---

## Part 3 — The Method Area (Metaspace)

### What It Is

```
The Method Area is a special shared memory region.

Java 7 and before:  called "PermGen" (Permanent Generation)
                    was INSIDE the heap, fixed size
                    famous for "OutOfMemoryError: PermGen space"

Java 8 and after:   called "Metaspace"
                    moved OUTSIDE the heap
                    uses native memory (off-heap)
                    grows dynamically (no fixed size by default)
                    much less likely to run out of space

Shared by ALL threads — just like the Heap.
But unlike Heap, it stores CLASS-LEVEL data, not instance data.
```

### What Lives in the Method Area

```java
public class MethodAreaDemo {

    // ── STATIC VARIABLES → stored in Method Area ──────────────────
    static int    instanceCount = 0;          // primitive value in Method Area
    static String appName = "MyApp";          // reference in Method Area
                                              // String object is on the Heap
                                              // (Method Area holds the reference)

    // ── CONSTANTS → stored in Method Area (Runtime Constant Pool) ─
    static final int MAX_SIZE = 100;          // compile-time constant
    static final String GREETING = "Hello";  // interned string in constant pool

    // ── INSTANCE VARIABLES → stored on HEAP (part of object) ──────
    private int   id;      // NOT in Method Area — on Heap with the object
    private String name;   // NOT in Method Area — on Heap with the object

    // ── METHOD CODE (bytecode) → stored in Method Area ────────────
    public void someMethod() {
        // The bytecode of this method lives in the Method Area
        // ALL instances of MethodAreaDemo SHARE the same bytecode
        // The bytecode is NOT per-object — it is per-class
    }
}
```

```
METHOD AREA CONTENTS — detailed breakdown:

┌──────────────────────────────────────────────────────────────────┐
│                    METHOD AREA (Metaspace)                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  CLASS: MethodAreaDemo                                     │  │
│  │                                                            │  │
│  │  1. CLASS METADATA                                         │  │
│  │     - Class name: "MethodAreaDemo"                         │  │
│  │     - Superclass: "java.lang.Object"                       │  │
│  │     - Interfaces: []                                       │  │
│  │     - Access modifiers: public                             │  │
│  │     - Class version: Java 17                               │  │
│  │                                                            │  │
│  │  2. STATIC VARIABLES                                       │  │
│  │     - instanceCount = 0     ← integer VALUE stored here   │  │
│  │     - appName = ref→[Heap]  ← reference stored here       │  │
│  │       (actual String "MyApp" object lives on Heap)        │  │
│  │                                                            │  │
│  │  3. METHOD BYTECODE                                        │  │
│  │     - someMethod() bytecode: [getstatic, invokevirtual...] │  │
│  │     - constructor bytecode: [aload_0, invokespecial...]   │  │
│  │     - static initializer bytecode (if any)                │  │
│  │                                                            │  │
│  │  4. RUNTIME CONSTANT POOL                                  │  │
│  │     - MAX_SIZE = 100                                       │  │
│  │     - GREETING = ref→String "Hello" (interned on Heap)    │  │
│  │     - Symbolic references to other classes/methods        │  │
│  │     - Numeric literals, string literals                   │  │
│  │                                                            │  │
│  │  5. FIELD DESCRIPTORS                                      │  │
│  │     - id:   type=int,    access=private                   │  │
│  │     - name: type=String, access=private                   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  CLASS: Person  (another loaded class)                     │  │
│  │  ...                                                       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  CLASS: java.lang.String  (JDK class)                      │  │
│  │  ...                                                       │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Static Variables — Shared and Dangerous

```java
// ═══════════════════════════════════════════════════════════════
//  STATIC VARIABLES — shared via Method Area — concurrency danger
// ═══════════════════════════════════════════════════════════════
public class StaticVariableDanger {

    // static variable — lives in Method Area
    // shared by ALL threads AND all instances
    static int requestCount = 0;

    // Each incoming request calls this — from multiple threads
    public void handleRequest(String request) {

        requestCount++;
        // requestCount is in Method Area — SHARED
        // requestCount++ is: READ, ADD, WRITE — not atomic
        // Multiple threads doing this simultaneously → race condition
        // requestCount will be LESS than actual requests processed
    }

    // ─── COMPARING static vs instance fields ──────────────────────
    static class Counter {
        static int  staticCount  = 0;  // Method Area — ONE copy, ALL objects share
        int         instanceCount = 0; // Heap — ONE copy PER OBJECT

        void increment() {
            staticCount++;   // touches Method Area → shared → unsafe
            instanceCount++; // touches this object's Heap field
                             // → still unsafe if same object shared
                             // → safe if each thread has its OWN Counter object
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Counter c1 = new Counter(); // object 1 on Heap
        Counter c2 = new Counter(); // object 2 on Heap

        // c1.instanceCount  and  c2.instanceCount  are DIFFERENT fields
        // on DIFFERENT heap objects — no sharing

        // c1.staticCount    and  c2.staticCount    are THE SAME field
        // both refer to Counter.staticCount in Method Area — SHARED!
        c1.staticCount = 5;
        System.out.println(c2.staticCount); // prints 5 — same variable!
    }
}
```

```
STATIC vs INSTANCE FIELDS — where they live:

  class Counter {
    static int  staticCount  = 0;   ← in Method Area (ONE copy)
    int         instanceCount = 0;  ← in each object on Heap
  }

  METHOD AREA:
  ┌──────────────────────────────────┐
  │  class Counter                   │
  │    staticCount = 0  ◄────────────┼── ALL instances share this
  └──────────────────────────────────┘

  HEAP:
  ┌──────────────────────────────────┐
  │  Counter object 1 (c1)           │
  │    instanceCount = 0             │ ← c1's own copy
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │  Counter object 2 (c2)           │
  │    instanceCount = 0             │ ← c2's own copy
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │  Counter object 3 (c3)           │
  │    instanceCount = 0             │ ← c3's own copy
  └──────────────────────────────────┘
```

### Class Loading — How Classes Get Into Method Area

```java
// ═══════════════════════════════════════════════════════════════
//  CLASS LOADING — how class data gets into Method Area
// ═══════════════════════════════════════════════════════════════
public class ClassLoadingDemo {

    // Static initializer — runs ONCE when class is first loaded
    // Runs before any instance is created or static method called
    static {
        System.out.println("Class loaded into Method Area!");
        // This code runs exactly ONCE — when ClassLoader loads this class
        // Subsequent object creations do NOT re-run this
    }

    static int count = 0; // initialized to 0 when class loads

    public ClassLoadingDemo() {
        count++; // increments Method Area value each time object created
    }

    public static void main(String[] args) {
        System.out.println("Before creating objects: count=" + count); // 0

        ClassLoadingDemo obj1 = new ClassLoadingDemo();
        ClassLoadingDemo obj2 = new ClassLoadingDemo();
        ClassLoadingDemo obj3 = new ClassLoadingDemo();

        System.out.println("After 3 objects: count=" + count); // 3

        // All three objects share the SAME 'count' in Method Area
        // The three objects themselves are DIFFERENT on the Heap
    }
}
```

```
CLASS LOADING PHASES:

  1. LOADING
     ClassLoader reads .class file (bytecode)
     Creates Class object on Heap
     Stores class metadata in Method Area

  2. LINKING
     a. Verification  → bytecode is valid and safe
     b. Preparation   → static variables set to DEFAULT values
                        (int → 0, boolean → false, ref → null)
     c. Resolution    → symbolic references resolved to direct references

  3. INITIALIZATION
     Static initializer blocks run
     Static variables assigned their declared values
     Runs exactly ONCE — thread-safe (ClassLoader synchronizes it)

  ┌────────────────────────────────────────────────────────────┐
  │  static int count = 10;                                    │
  │                                                            │
  │  Preparation:    count = 0    ← default value first       │
  │  Initialization: count = 10   ← then declared value       │
  └────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Everything Working Together

```java
// ═══════════════════════════════════════════════════════════════
//  COMPLETE EXAMPLE — trace every variable to its memory area
// ═══════════════════════════════════════════════════════════════
public class CompleteMemoryTrace {

    // ── WHERE DOES EACH PIECE OF DATA LIVE? ───────────────────────

    static int         appVersion   = 2;         // Method Area
    static String      appName      = "MyApp";   // ref: Method Area
                                                 // "MyApp" String: Heap
    static final int   MAX          = 100;       // Method Area (constant pool)

    private int        instanceId;               // Heap (per object)
    private String     name;                     // ref: Heap, String: Heap

    public CompleteMemoryTrace(int id, String name) {
        this.instanceId = id;    // writing to Heap
        this.name = name;        // writing reference to Heap
    }

    public int process(int input) {              // bytecode: Method Area
        int multiplier = 3;                      // local: Stack
        String label   = "result";               // ref: Stack, String: Heap
        int    result  = input * multiplier;     // local: Stack
        return result;                           // value returned via Stack
    }

    public static void main(String[] args) throws InterruptedException {

        // obj1 and obj2 — REFERENCES on Stack (main thread's stack)
        // Actual objects — on Heap
        CompleteMemoryTrace obj1 = new CompleteMemoryTrace(1, "First");
        CompleteMemoryTrace obj2 = new CompleteMemoryTrace(2, "Second");

        // Both obj1 and obj2 share appVersion (Method Area)
        // But they each have their own instanceId and name (Heap)

        Thread t1 = new Thread(() -> {
            // t1's own Stack Frame for this lambda
            int localVal = obj1.process(10); // localVal on t1's Stack
                                              // obj1 is on Heap — shared
            System.out.println("t1: " + localVal);
        });

        Thread t2 = new Thread(() -> {
            // t2's own Stack Frame — completely separate from t1's
            int localVal = obj2.process(20); // localVal on t2's Stack
                                              // obj2 is on Heap — shared
            System.out.println("t2: " + localVal);
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

**Complete memory picture while both threads run:**

```
METHOD AREA (Metaspace) — shared
══════════════════════════════════════════════════════════════════

  class CompleteMemoryTrace:
    appVersion = 2
    appName    = ref→[Heap: 0x1000]
    MAX        = 100
    bytecode: process(), main(), constructor


HEAP — shared
══════════════════════════════════════════════════════════════════

  [0x1000] String "MyApp"
  [0x2000] String "First"
  [0x3000] String "Second"
  [0x4000] String "result"    ← label in process() — on Heap
                                 (even though reference 'label' is on Stack)

  [0x5000] CompleteMemoryTrace obj1
             instanceId = 1
             name = ref→[0x2000]

  [0x6000] CompleteMemoryTrace obj2
             instanceId = 2
             name = ref→[0x3000]

  [0x7000] Thread object t1
  [0x8000] Thread object t2

  [0x9000] String[] args (main's parameter)


STACK of main thread — private to main thread
══════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────┐  ← TOP (main is at bottom now)
  │  Frame: main()                   │
  │    args  = ref→[0x9000]          │
  │    obj1  = ref→[0x5000]          │
  │    obj2  = ref→[0x6000]          │
  │    t1    = ref→[0x7000]          │
  │    t2    = ref→[0x8000]          │
  └──────────────────────────────────┘


STACK of Thread t1 — private to t1
══════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────┐  ← TOP
  │  Frame: process()  (called by t1)│
  │    this        = ref→[0x5000]    │  ← obj1
  │    input       = 10              │
  │    multiplier  = 3               │
  │    label       = ref→[0x4000]    │  ← ref on stack, String on Heap
  │    result      = 30              │
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │  Frame: lambda (t1's runnable)   │
  │    localVal    = (waiting)       │
  └──────────────────────────────────┘


STACK of Thread t2 — private to t2
══════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────┐  ← TOP
  │  Frame: process()  (called by t2)│
  │    this        = ref→[0x6000]    │  ← obj2
  │    input       = 20              │
  │    multiplier  = 3               │
  │    label       = ref→[0x4000]    │  ← different ref slot, same String
  │    result      = 60              │
  └──────────────────────────────────┘
  ┌──────────────────────────────────┐
  │  Frame: lambda (t2's runnable)   │
  │    localVal    = (waiting)       │
  └──────────────────────────────────┘

OBSERVATIONS:
  1. t1 and t2 have COMPLETELY SEPARATE stacks → no sharing of local vars
  2. multiplier=3 exists TWICE — once on t1's stack, once on t2's stack
     (no sharing even though same method — different frames)
  3. obj1 and obj2 are on Heap — if both threads accessed SAME object,
     instanceId could be corrupted without synchronization
  4. appVersion is in Method Area — shared, could be corrupted if
     any thread writes to it without synchronization
  5. 'label' reference is on Stack (private) but
     the String "result" it points to is on Heap (shared)
     However: String is immutable → shared is safe here
```

---

## The Golden Rules — Concurrency Implications

```
╔══════════════════════════════════════════════════════════════════════╗
║              MEMORY AREA → CONCURRENCY RULE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  JVM STACK                                                           ║
║  ─────────                                                           ║
║  ✅ 100% private to each thread                                      ║
║  ✅ Local variables and method parameters ALWAYS thread-safe         ║
║  ✅ No synchronization EVER needed for stack data                    ║
║  ✅ No visibility issues — other threads literally cannot see it     ║
║  ✅ primitives on stack are safe                                     ║
║  ⚠  Object REFERENCES are on stack but objects they point to        ║
║     are on Heap — the objects themselves may be shared               ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  HEAP                                                                ║
║  ────                                                                ║
║  ⚠  Shared by ALL threads                                           ║
║  ⚠  ALL instance fields of shared objects require care              ║
║  ⚠  Race conditions happen HERE                                      ║
║  ⚠  Visibility bugs happen HERE                                      ║
║  ✅ If object is NOT shared (only one thread has reference)          ║
║     → no concurrency concern even on Heap                            ║
║  ✅ If object is immutable (no state changes after construction)     ║
║     → safe to share without synchronization                          ║
║                                                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  METHOD AREA                                                         ║
║  ───────────                                                         ║
║  ⚠  Shared by ALL threads                                           ║
║  ⚠  Static MUTABLE variables require synchronization                ║
║  ✅  Static FINAL (constants) are safe — never change                ║
║  ✅  Bytecode / class metadata is read-only — safe                   ║
║  ⚠  Static variables holding object references                      ║
║     — the object on Heap may also be concurrently accessed          ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝


THE THREE QUESTIONS TO ASK ABOUT ANY VARIABLE:
══════════════════════════════════════════════

  Q1: Where does this variable LIVE?
      → Stack?        → safe, no sync needed
      → Heap?         → check Q2
      → Method Area?  → check Q2

  Q2: Is it accessed by more than ONE thread?
      → No  → safe (even on Heap — no sharing)
      → Yes → check Q3

  Q3: Is at least one of those accesses a WRITE?
      → No  (all reads, value never changes) → safe
      → Yes → UNSAFE — needs synchronization

  Only if answer to ALL THREE leads to "unsafe"
  do you need to add synchronization.
```

---

## Summary

```
┌────────────────┬───────────────┬──────────────────┬──────────────────────┐
│ Memory Area    │ Shared?       │ What Lives Here  │ Concurrency Risk?    │
├────────────────┼───────────────┼──────────────────┼──────────────────────┤
│ JVM Stack      │ ✗ Private     │ Local variables  │ Zero — completely    │
│ (per thread)   │ (per thread)  │ Method params    │ isolated             │
│                │               │ Return addresses │                      │
│                │               │ Partial results  │                      │
├────────────────┼───────────────┼──────────────────┼──────────────────────┤
│ Heap           │ ✓ All threads │ All objects      │ HIGH — all shared    │
│                │               │ All arrays       │ object mutations     │
│                │               │ Instance fields  │ need protection      │
├────────────────┼───────────────┼──────────────────┼──────────────────────┤
│ Method Area    │ ✓ All threads │ Class metadata   │ MEDIUM — static      │
│ (Metaspace)    │               │ Static variables │ mutable vars need    │
│                │               │ Bytecode         │ protection           │
│                │               │ Constant pool    │ constants are safe   │
├────────────────┼───────────────┼──────────────────┼──────────────────────┤
│ PC Register    │ ✗ Private     │ Current          │ Zero — private       │
│ (per thread)   │ (per thread)  │ instruction ptr  │                      │
└────────────────┴───────────────┴──────────────────┴──────────────────────┘

ONE LINE TO REMEMBER:

  Stack  = your thread's private notebook — nobody else can read it
  Heap   = the shared whiteboard — everyone can read and write
  Method Area = the shared reference manual — everyone reads it,
                static fields can be written (carefully)
```