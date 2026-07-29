# What is a Thread vs Process — Complete Tutorial

---

## The Story Before the Definitions

```
Imagine a large office building.

The building has multiple companies renting floors.
Each company is completely isolated from the others.
Company A on floor 3 cannot walk into Company B on floor 7.
Company A cannot read Company B's files.
Company A cannot use Company B's printers.
They are completely separate tenants.

Each COMPANY is a PROCESS.

Now inside Company A on floor 3 —
there are multiple EMPLOYEES.
They all share the same office space.
They share the same printer, the same whiteboard,
the same coffee machine, the same filing cabinets.
They can read each other's sticky notes.
They can overwrite each other's whiteboard.

Each EMPLOYEE is a THREAD.

That is the entire mental model.
Process = Company (isolated, independent)
Thread  = Employee (shares everything within the company)
```

---

## The Big Picture — Side by Side

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    PROCESS vs THREAD                                  ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐  ║
║  │                    OPERATING SYSTEM                             │  ║
║  └─────────────────────────────────────────────────────────────────┘  ║
║           │                    │                    │                 ║
║           ▼                    ▼                    ▼                 ║
║  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        ║
║  │   PROCESS 1     │  │   PROCESS 2     │  │   PROCESS 3     │        ║
║  │   (Chrome)      │  │   (IntelliJ)    │  │   (Your JVM)    │        ║
║  │                 │  │                 │  │                 │        ║
║  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │        ║
║  │ │Own Memory   │ │  │ │Own Memory   │ │  │ │Own Memory   │ │        ║
║  │ │(Heap/Stack/ │ │  │ │(Heap/Stack/ │ │  │ │(Heap/Stack/ │ │        ║
║  │ │ Code/Data)  │ │  │ │ Code/Data)  │ │  │ │ Code/Data)  │ │        ║
║  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │        ║
║  │                 │  │                 │  │                 │        ║
║  │ ┌──┐ ┌──┐ ┌──┐  │  │ ┌──┐ ┌──┐      │  │ ┌──┐ ┌──┐ ┌──┐  │         ║
║  │ │T1│ │T2│ │T3│  │  │ │T1│ │T2│      │  │ │T1│ │T2│ │T3│  │         ║
║  │ └──┘ └──┘ └──┘  │  │ └──┘ └──┘      │  │ └──┘ └──┘ └──┘  │         ║
║  │ Threads share   │  │ Threads share   │  │ Threads share   │        ║
║  │ process memory  │  │ process memory  │  │ process memory  │        ║
║  └─────────────────┘  └─────────────────┘  └─────────────────┘        ║
║           ↕                    ↕                    ↕                 ║
║   ISOLATED from each other — cannot access each other's memory        ║
║   Must use IPC (pipes, sockets, files) to communicate                 ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---
## Part 1 — The Process

### What a Process Is

```
A PROCESS is a program in execution.

When you double-click IntelliJ IDEA:
  - OS loads the program from disk into RAM
  - OS creates a Process for it
  - OS allocates a private chunk of memory for it
  - OS gives it at least one thread to start running
  - OS assigns it a unique Process ID (PID)

The process is now a LIVING, RUNNING program.
```
### What a Process Owns

```
Every process gets its OWN PRIVATE copy of:

┌─────────────────────────────────────────────────────────────────────┐
│                         PROCESS MEMORY SPACE                        │
│                       (virtual address space)                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CODE SEGMENT (Text)                                         │   │
│  │  The compiled machine code / bytecode of the program         │   │
│  │  Read-only — cannot be modified at runtime                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  DATA SEGMENT                                                │   │
│  │  Global variables, static variables                          │   │
│  │  Initialized data (.data) and uninitialized (.bss)           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  HEAP                                                        │   │
│  │  Dynamically allocated memory (objects, arrays)              │   │
│  │  Grows upward as program allocates more objects              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              ▲  ▼  (heap grows up, stack grows down) │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  STACK                                                       │   │
│  │  Function call frames, local variables, return addresses     │   │
│  │  Grows downward as functions are called                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  OS RESOURCES                                                │   │
│  │  Open file handles (file descriptors)                        │   │
│  │  Network sockets                                             │   │
│  │  Pipes, semaphores                                           │   │
│  │  Process ID (PID)                                            │   │
│  │  Environment variables                                       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘

This ENTIRE memory space is INVISIBLE to all other processes.
Process A cannot read even a single byte of Process B's memory.
(Unless explicitly shared via OS mechanisms — rare, complex)
```

### Process Isolation — Why It Matters

```
PROCESS ISOLATION is the OS's #1 safety guarantee.

Chrome crashes  → your IntelliJ keeps running
IntelliJ hangs  → Chrome keeps running
Game freezes    → Terminal still responds

Each process failure is CONTAINED.
One bad process cannot corrupt another process's memory.
This is why your entire computer doesn't crash
when one app dies.

On the JVM:
  Your JVM process crashes (OutOfMemoryError, JVM bug)
  → only YOUR program dies
  → every other program on the machine is unaffected
```

### Creating a Process in Java

```java
import java.io.*;

public class ProcessDemo {

    public static void main(String[] args) throws Exception {

        // ── WAY 1: ProcessBuilder (modern, preferred) ──────────────
        ProcessBuilder pb = new ProcessBuilder("ls", "-la");
        // ↑ Creates a NEW process that runs the OS command "ls -la"
        // This is a COMPLETELY SEPARATE process from our JVM
        // It has its own PID, its own memory, its own stack, its own heap

        pb.redirectErrorStream(true);
        Process process = pb.start();
        // ↑ process is now RUNNING — a separate OS process
        //   Our JVM process and this new process are siblings
        //   They share NO memory

        // Read the output of the child process
        // Communication via PIPE (OS mechanism for inter-process communication)
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(process.getInputStream())
        );

        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }

        int exitCode = process.waitFor();
        // ↑ Our JVM thread BLOCKS here until the child process EXITS
        //   Similar in spirit to Thread.join() but for PROCESSES

        System.out.println("Child process exited with code: " + exitCode);
        // exit code 0 = success, non-zero = some error

        // ── WAY 2: Runtime.exec() (legacy) ────────────────────────
        Process legacyProcess = Runtime.getRuntime().exec("echo hello");
        // Same idea — creates separate OS process
        // Less flexible than ProcessBuilder

        // ── GETTING CURRENT PROCESS INFO ──────────────────────────
        ProcessHandle currentProcess = ProcessHandle.current();
        System.out.println("My PID: " + currentProcess.pid());
        System.out.println("My info: " + currentProcess.info());
    }
}
```

```
PROCESS LIFECYCLE:

  Program on Disk (.class / .jar)
          │
          │  java MyProgram  (OS creates process)
          ▼
  ┌───────────────────┐
  │  NEW / CREATED    │  OS allocates memory, assigns PID
  └────────┬──────────┘
           │  OS schedules it
           ▼
  ┌───────────────────┐
  │     RUNNING       │  CPU executes instructions
  └────────┬──────────┘
           │  may oscillate between RUNNING and WAITING
           │  (waiting for I/O, waiting for CPU time)
           ▼
  ┌───────────────────┐
  │   WAITING/BLOCKED │  waiting for I/O, user input, etc.
  └────────┬──────────┘
           │  I/O complete, resume
           ▼
  ┌───────────────────┐
  │    TERMINATED     │  main() returned, System.exit() called,
  └───────────────────┘  crash, or OS killed it
                         Memory freed, PID released
```

---

## Part 2 — The Thread

### What a Thread Is

```
A THREAD is the smallest unit of execution
WITHIN a process.

Every process has at LEAST one thread.
(The main thread — the one that runs main())

A process can create MORE threads.
All threads in a process SHARE the process's memory.
Each thread gets its OWN private stack.
Everything else (heap, code, data, files) is shared.

Thread = lightweight process
       = process minus the private memory space
       = "employee within a company"
```

### What a Thread Owns vs Shares

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ONE PROCESS                                  │
│                  (e.g., Your JVM process)                           │
│                                                                     │
│  ╔══════════════════════════════════════════════════════════════╗   │
│  ║        SHARED BY ALL THREADS IN THIS PROCESS                 ║   │
│  ║                                                              ║   │
│  ║   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  ║   │
│  ║   │    HEAP     │  │ METHOD AREA │  │  OPEN FILES         │  ║   │
│  ║   │  (objects)  │  │  (classes,  │  │  NETWORK SOCKETS    │  ║   │
│  ║   │  (arrays)   │  │   statics,  │  │  ENVIRONMENT VARS   │  ║   │
│  ║   │             │  │   bytecode) │  │  PROCESS ID (PID)   │  ║   │
│  ║   └─────────────┘  └─────────────┘  └─────────────────────┘  ║   │
│  ╚══════════════════════════════════════════════════════════════╝   │
│                                                                     │
│  ╔══════════╗   ╔══════════╗   ╔══════════╗   ╔══════════╗          │
│  ║ THREAD 1 ║   ║ THREAD 2 ║   ║ THREAD 3 ║   ║ THREAD 4 ║          │
│  ║          ║   ║          ║   ║          ║   ║          ║          │
│  ║ PRIVATE: ║   ║ PRIVATE: ║   ║ PRIVATE: ║   ║ PRIVATE: ║          │
│  ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║          │
│  ║ │Stack │ ║   ║ │Stack │ ║   ║ │Stack │ ║   ║ │Stack │ ║          │
│  ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║          │
│  ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║          │
│  ║ │  PC  │ ║   ║ │  PC  │ ║   ║ │  PC  │ ║   ║ │  PC  │ ║          │
│  ║ │  Reg │ ║   ║ │  Reg │ ║   ║ │  Reg │ ║   ║ │  Reg │ ║          │
│  ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║          │
│  ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║   ║ ┌──────┐ ║          │
│  ║ │Thread│ ║   ║ │Thread│ ║   ║ │Thread│ ║   ║ │Thread│ ║          │
│  ║ │  ID  │ ║   ║ │  ID  │ ║   ║ │  ID  │ ║   ║ │  ID  │ ║          │
│  ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║   ║ └──────┘ ║          │
│  ╚══════════╝   ╚══════════╝   ╚══════════╝   ╚══════════╝          │
└─────────────────────────────────────────────────────────────────────┘

Each thread OWNS:           Each thread SHARES (with all threads in process):
  ✅ Its own Stack             ⚠ Heap (all objects)
  ✅ Its own PC Register       ⚠ Method Area (statics, bytecode)
  ✅ Its own Thread ID         ⚠ Open file handles
  ✅ Its own priority          ⚠ Network connections
  ✅ Its own interrupt flag     ⚠ The process's PID
```

### Creating Threads in Java — Four Ways

```java
import java.util.concurrent.*;

public class ThreadCreationWays {

    // ── WAY 1: Extend Thread ───────────────────────────────────────
    // Old school — avoid in modern code
    // Inflexible (can't extend another class)
    static class MyThread extends Thread {
        private String taskName;

        MyThread(String taskName) {
            this.taskName = taskName;
            // Thread is in NEW state — not yet started
        }

        @Override
        public void run() {
            // This code runs IN THE NEW THREAD
            // A new Stack is created for this thread
            // Local variables here are on THIS thread's private Stack
            System.out.println(taskName + " running on: "
                    + Thread.currentThread().getName());
        }
    }

    // ── WAY 2: Implement Runnable ──────────────────────────────────
    // Better — separates task from thread
    // Can implement other interfaces too
    static class MyTask implements Runnable {
        @Override
        public void run() {
            System.out.println("Runnable running on: "
                    + Thread.currentThread().getName());
        }
    }

    // ── WAY 3: Lambda (Java 8+) ────────────────────────────────────
    // Most concise for simple tasks

    // ── WAY 4: ExecutorService (modern, preferred) ─────────────────
    // Don't create threads directly — submit tasks to a pool

    public static void main(String[] args) throws InterruptedException {

        // Way 1: Extend Thread
        MyThread t1 = new MyThread("Task-1");
        t1.setName("thread-way-1");
        t1.start();
        // ↑ start() does TWO things:
        //   1. Creates a NEW thread in the OS (OS allocates new Stack, PC, Thread ID)
        //   2. Calls run() on that new thread
        // NEVER call run() directly — that just executes on current thread, no new thread created

        // Way 2: Runnable + Thread
        Thread t2 = new Thread(new MyTask(), "thread-way-2");
        t2.start();

        // Way 3: Lambda — most common today
        Thread t3 = new Thread(() -> {
            System.out.println("Lambda running on: "
                    + Thread.currentThread().getName());
        }, "thread-way-3");
        t3.start();

        // Way 4: ExecutorService — preferred in real applications
        ExecutorService pool = Executors.newFixedThreadPool(3);
        pool.submit(() -> System.out.println("Pool task running on: "
                + Thread.currentThread().getName()));
        pool.shutdown();

        t1.join(); t2.join(); t3.join();
    }
}
```

### Thread Identity and Introspection

```java
public class ThreadIdentity {
    public static void main(String[] args) throws InterruptedException {

        // ── EVERY thread has these properties ─────────────────────
        Thread mainThread = Thread.currentThread();

        System.out.println("Name    : " + mainThread.getName());
        // "main" — the default name of the main thread

        System.out.println("ID      : " + mainThread.getId());
        // unique long ID assigned by JVM (not OS thread ID)

        System.out.println("Priority: " + mainThread.getPriority());
        // 5 (Thread.NORM_PRIORITY) — range is 1 (MIN) to 10 (MAX)

        System.out.println("Daemon  : " + mainThread.isDaemon());
        // false — main thread is NOT a daemon thread

        System.out.println("State   : " + mainThread.getState());
        // RUNNABLE — it's currently running (this code!)

        System.out.println("Alive   : " + mainThread.isAlive());
        // true — it's running

        System.out.println("Group   : " + mainThread.getThreadGroup().getName());
        // "main" — the thread group

        // ── Daemon vs Non-Daemon threads ───────────────────────────
        Thread daemon = new Thread(() -> {
            while (true) {
                System.out.println("Daemon: still running...");
                try { Thread.sleep(500); } catch (InterruptedException e) { break; }
            }
        });
        daemon.setDaemon(true);
        // ↑ Daemon thread:
        //   JVM DOES NOT WAIT for daemon threads to finish before exiting
        //   When all non-daemon threads finish → JVM exits
        //   → daemon threads are killed abruptly
        //   Use for: background cleanup, monitoring, GC-like tasks
        //   MUST call setDaemon(true) BEFORE start() — else IllegalThreadStateException

        Thread nonDaemon = new Thread(() -> {
            System.out.println("NonDaemon: doing important work");
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            System.out.println("NonDaemon: finished");
        });
        // ↑ Non-Daemon (user) thread:
        //   JVM WAITS for ALL non-daemon threads to finish before exiting

        daemon.start();
        nonDaemon.start();
        nonDaemon.join();
        // When nonDaemon finishes → main finishes → JVM exits
        // daemon is killed abruptly — even if still in while(true)

        // ── Priority ───────────────────────────────────────────────
        Thread highPriority = new Thread(() -> System.out.println("High"));
        Thread lowPriority  = new Thread(() -> System.out.println("Low"));

        highPriority.setPriority(Thread.MAX_PRIORITY); // 10
        lowPriority.setPriority(Thread.MIN_PRIORITY);  // 1

        // Priority is a HINT to the OS scheduler — NOT a guarantee
        // High priority thread is MORE LIKELY to run first
        // but OS can schedule low priority thread first if it wants
        // Never write code that DEPENDS on thread priority for correctness
    }
}
```

---

## Part 3 — Process vs Thread — Deep Comparison

### Memory Isolation

```java
// ═══════════════════════════════════════════════════════════════
//  MEMORY: Processes are ISOLATED, Threads SHARE
// ═══════════════════════════════════════════════════════════════
public class MemoryIsolationDemo {

    static int sharedValue = 0; // lives in Method Area — shared by ALL threads

    public static void main(String[] args) throws InterruptedException {

        // ── THREADS SHARE memory ──────────────────────────────────
        Thread writer = new Thread(() -> {
            sharedValue = 42;
            System.out.println("Writer set sharedValue = " + sharedValue);
        });

        Thread reader = new Thread(() -> {
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            // Reader can directly see sharedValue
            // No copying, no serialization, no IPC needed
            System.out.println("Reader sees sharedValue = " + sharedValue);
            // prints 42 — same memory, direct access
        });

        writer.start();
        reader.start();
        writer.join();
        reader.join();
        // Threads can communicate by simply reading/writing shared variables
        // FAST — just a memory read/write
        // DANGEROUS — no automatic protection (race conditions)

        // ── PROCESSES are ISOLATED ────────────────────────────────
        // Two separate JVM processes CANNOT do this:
        //
        // Process 1:                    Process 2:
        //   sharedValue = 42;             System.out.println(sharedValue);
        //   // Process 2 CANNOT SEE THIS  // This doesn't compile — different process
        //
        // Processes must use IPC (Inter-Process Communication):
        //   - Files (write to file, other process reads)
        //   - Sockets (network communication, even on same machine)
        //   - Pipes (OS-level byte streams between processes)
        //   - Shared Memory (OS-level explicit shared memory region — complex)
        //   - Message Queues
        //
        // IPC is SLOW (relative to thread memory sharing)
        // and COMPLEX to code correctly
    }
}
```

```
COMMUNICATION SPEED COMPARISON:

  Thread-to-Thread (shared memory):
    write sharedValue = 42     →  read sharedValue
    ~nanoseconds (one memory write + one memory read)
    Direct — no copying, no serialization

  Process-to-Process (via socket on same machine):
    Process A: serialize data → write to socket buffer
    OS: copy buffer across process boundary
    Process B: read from socket → deserialize data
    ~microseconds to milliseconds
    10x to 1000x SLOWER than thread communication
```

### Creation Cost

```
CREATING A PROCESS:
═══════════════════
  OS must:
  1. Allocate new virtual address space      (~megabytes of setup)
  2. Copy parent process's page table
  3. Allocate memory for code, data, heap, stack
  4. Load program from disk (if new program)
  5. Initialize OS data structures (file descriptors, PID, etc.)
  6. Set up memory mapping

  Cost: HEAVY — thousands of microseconds (milliseconds)
  Memory: LARGE — megabytes minimum for address space setup

CREATING A THREAD:
══════════════════
  OS must:
  1. Allocate a new Stack for the thread      (~1MB default in JVM)
  2. Create a Thread Control Block (TCB)       (small kernel struct)
  3. Register the thread with the scheduler

  Cost: LIGHTER — microseconds (much cheaper than process)
  Memory: Stack size only (~1MB per thread by default, tunable)
  BUT: 1000 threads = ~1GB RAM just for stacks — still significant!

VIRTUAL THREADS (Java 21):
══════════════════════════
  JVM manages them — no OS thread per virtual thread
  1. Allocate a tiny initial stack (~few KB, grows as needed)
  2. Register with JVM scheduler

  Cost: VERY CHEAP — hundreds of nanoseconds
  Memory: starts at ~few KB (not 1MB)
  Can create MILLIONS of virtual threads
```

### Context Switching

```
WHAT IS CONTEXT SWITCHING?
════════════════════════════════════════════════════════════════

  When CPU switches from running Thread/Process A
  to running Thread/Process B:

  1. SAVE A's state:
     - CPU registers (instruction pointer, stack pointer, etc.)
     - For process: also save entire memory mapping (page tables)

  2. LOAD B's state:
     - CPU registers
     - For process: also reload entire memory mapping

  For THREADS (same process):
  ┌──────────────────────────────────────────────────────────┐
  │  Switch Thread A → Thread B (same process)               │
  │                                                          │
  │  Save: A's CPU registers, A's Stack pointer, A's PC      │
  │  Load: B's CPU registers, B's Stack pointer, B's PC      │
  │                                                          │
  │  Memory mapping? SAME — no change needed (same process!) │
  │                                                          │
  │  Cost: ~1-10 microseconds (lightweight)                  │
  └──────────────────────────────────────────────────────────┘

  For PROCESSES:
  ┌──────────────────────────────────────────────────────────┐
  │  Switch Process A → Process B                            │
  │                                                          │
  │  Save: A's CPU registers                                 │
  │  Save: A's ENTIRE memory map (page tables) ← EXPENSIVE   │
  │  Flush: CPU's TLB (Translation Lookaside Buffer)          │
  │  Load: B's CPU registers                                 │
  │  Load: B's ENTIRE memory map ← EXPENSIVE                 │
  │  Warm up: CPU caches for B (cold start) ← EXPENSIVE      │
  │                                                          │
  │  Cost: ~100 microseconds (10-100x heavier)               │
  └──────────────────────────────────────────────────────────┘

  Thread context switch ≈ 1-10 μs
  Process context switch ≈ 100-1000 μs
  (10x to 100x more expensive for processes)
```

---

## Part 4 — The JVM as a Process

```java
// ═══════════════════════════════════════════════════════════════
//  THE JVM IS ITSELF A PROCESS
//  Your Java program runs INSIDE the JVM process
// ═══════════════════════════════════════════════════════════════
public class JVMAsProcess {

    public static void main(String[] args) throws InterruptedException {

        // When you run: java MyProgram
        //
        // OS creates ONE JVM process (PID: e.g., 12345)
        //   Inside that ONE process:
        //     JVM creates MULTIPLE threads automatically:
        //       - main thread          (runs your main())
        //       - GC threads           (garbage collector)
        //       - JIT compiler threads (compile hot bytecode to native)
        //       - Signal dispatcher    (handles OS signals)
        //       - Finalizer thread     (runs finalizers)
        //       - Reference handler    (handles weak/soft/phantom refs)
        //       + any threads YOU create

        // Let's see all threads currently running in OUR JVM process:
        ThreadGroup rootGroup = Thread.currentThread().getThreadGroup();
        ThreadGroup parentGroup;
        while ((parentGroup = rootGroup.getParent()) != null) {
            rootGroup = parentGroup;
        }

        // Get count and list of all active threads
        int count = rootGroup.activeCount();
        Thread[] threads = new Thread[count + 10];
        rootGroup.enumerate(threads, true);

        System.out.println("=== All Threads in This JVM Process ===");
        for (Thread t : threads) {
            if (t != null) {
                System.out.printf(
                    "  Name: %-30s  Daemon: %-5s  State: %s%n",
                    t.getName(),
                    t.isDaemon(),
                    t.getState()
                );
            }
        }
    }
}
```

**Typical output:**
```
=== All Threads in This JVM Process ===
  Name: main                            Daemon: false  State: RUNNABLE
  Name: Reference Handler               Daemon: true   State: WAITING
  Name: Finalizer                       Daemon: true   State: WAITING
  Name: Signal Dispatcher               Daemon: true   State: RUNNABLE
  Name: Attach Listener                 Daemon: true   State: RUNNABLE
  Name: Common-Cleaner                  Daemon: true   State: TIMED_WAITING
  Name: Monitor Ctrl-Break              Daemon: true   State: RUNNABLE
  Name: GC Thread#0                     Daemon: true   State: RUNNABLE
  Name: GC Thread#1                     Daemon: true   State: RUNNABLE
  Name: C2 CompilerThread0              Daemon: true   State: RUNNABLE
  Name: C2 CompilerThread1              Daemon: true   State: RUNNABLE
```

```
THE JVM PROCESS PICTURE:
════════════════════════════════════════════════════════════════

  Operating System
  └── JVM Process (PID: 12345)
       │
       ├── SHARED MEMORY:
       │    ├── Heap         (your objects, GC manages this)
       │    ├── Metaspace    (class definitions, statics)
       │    └── Code Cache   (JIT compiled native code)
       │
       ├── Thread: main                 ← runs your main()
       ├── Thread: GC Thread#0          ← garbage collection
       ├── Thread: GC Thread#1          ← garbage collection (parallel)
       ├── Thread: C2 CompilerThread0   ← JIT compiles hot methods
       ├── Thread: C2 CompilerThread1   ← JIT compiles hot methods
       ├── Thread: Finalizer            ← runs Object.finalize()
       ├── Thread: Reference Handler    ← weak/soft/phantom refs
       ├── Thread: Signal Dispatcher    ← handles SIGINT, SIGTERM etc.
       │
       └── Your threads:
            ├── Thread: worker-1        ← your code
            ├── Thread: worker-2        ← your code
            └── Thread: worker-3        ← your code

  ALL threads share the Heap and Metaspace.
  EACH thread has its own Stack and PC Register.
```

---

## Part 5 — The 6 Thread States in Java

```java
public class ThreadStatesComplete {

    public static void main(String[] args) throws InterruptedException {

        final Object lock = new Object();

        // ── STATE 1: NEW ─────────────────────────────────────────────
        Thread thread = new Thread(() -> {});
        System.out.println(thread.getState()); // NEW
        // Thread object created but start() NOT called yet
        // OS thread does NOT exist yet — just a Java object

        // ── STATE 2: RUNNABLE ────────────────────────────────────────
        Thread runner = new Thread(() -> {
            // This thread is RUNNABLE
            // May actually be executing on CPU (RUNNING)
            // or waiting for CPU time slot (READY)
            // Java JVM combines both into RUNNABLE
            long sum = 0;
            for (long i = 0; i < 1_000_000_000L; i++) sum += i;
            System.out.println("Sum: " + sum);
        });
        runner.start();
        Thread.sleep(10); // give runner a head start
        System.out.println("runner: " + runner.getState()); // RUNNABLE

        // ── STATE 3: BLOCKED ─────────────────────────────────────────
        // Thread waiting to acquire a synchronized lock
        Thread blocker = new Thread(() -> {
            synchronized (lock) { // tries to acquire lock — may BLOCK here
                System.out.println("blocker: got lock");
            }
        });
        synchronized (lock) {          // main holds lock
            blocker.start();
            Thread.sleep(50);           // give blocker time to start and block
            System.out.println("blocker: " + blocker.getState()); // BLOCKED
        }
        // main releases lock → blocker unblocks

        // ── STATE 4: WAITING ─────────────────────────────────────────
        // Thread waiting INDEFINITELY for notification
        Thread waiter = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait();      // ← WAITING state (indefinitely)
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        waiter.start();
        Thread.sleep(50);
        System.out.println("waiter: " + waiter.getState()); // WAITING
        synchronized (lock) { lock.notify(); } // wake it up

        // ── STATE 5: TIMED_WAITING ───────────────────────────────────
        // Thread waiting for SPECIFIED TIME
        Thread sleeper = new Thread(() -> {
            try {
                Thread.sleep(5000); // ← TIMED_WAITING for 5 seconds
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        sleeper.start();
        Thread.sleep(50);
        System.out.println("sleeper: " + sleeper.getState()); // TIMED_WAITING
        sleeper.interrupt(); // wake it early

        // ── STATE 6: TERMINATED ──────────────────────────────────────
        Thread finisher = new Thread(() -> System.out.println("done"));
        finisher.start();
        finisher.join();
        System.out.println("finisher: " + finisher.getState()); // TERMINATED
    }
}
```

```
THREAD STATE MACHINE — ALL 6 STATES AND TRANSITIONS:

                    new Thread()
                         │
                         ▼
                    ┌─────────┐
                    │   NEW   │
                    └────┬────┘
                         │  start()
                         ▼
                    ┌──────────┐
              ┌────►│RUNNABLE  │◄────────────────────────┐
              │     └────┬─────┘                         │
              │          │                               │
              │    ┌─────┴───────────────────┐           │
              │    │                         │           │
              │    │ synchronized lock not   │ wait(ms)  │
              │    │ available               │ sleep(ms) │
              │    ▼                         ▼ join(ms)  │
              │  ┌──────────┐          ┌──────────────┐  │
              │  │ BLOCKED  │          │TIMED_WAITING │  │
              │  └────┬─────┘          └──────┬───────┘  │
              │       │ lock               timeout /     │
              │       │ acquired           interrupt /   │
              └───────┘                   notify         │
              │                               │          │
              │         wait()                │          │
              │    ┌──────────────┐           │          │
              │    │   WAITING    │           │          │
              │    └──────┬───────┘           │          │
              │           │ notify() /        │          │
              │           │ notifyAll() /     │          │
              │           │ interrupt         │          │
              └───────────┴───────────────────┘          │
                                                         │
                    run() completes                      │
                    or exception thrown                  │
                         │                               │
                         ▼                               │
                    ┌────────────┐                       │
                    │ TERMINATED │                       │
                    └────────────┘                       │
                    (cannot be restarted)                │
                                                         │

TRANSITIONS SUMMARY:
  NEW          → RUNNABLE:       start()
  RUNNABLE     → BLOCKED:        trying to enter synchronized (lock not available)
  RUNNABLE     → WAITING:        wait(), join() (no timeout), LockSupport.park()
  RUNNABLE     → TIMED_WAITING:  sleep(n), wait(n), join(n), LockSupport.parkNanos()
  BLOCKED      → RUNNABLE:       lock becomes available
  WAITING      → RUNNABLE:       notify(), notifyAll(), interrupt()
  TIMED_WAITING→ RUNNABLE:       timeout expires, notify(), interrupt()
  RUNNABLE     → TERMINATED:     run() returns or throws uncaught exception
```

---

## Part 6 — The Fundamental Trade-Off

```
CHOOSING BETWEEN PROCESSES AND THREADS:

╔═══════════════════════╦══════════════════════════╦═════════════════════════╗
║  PROPERTY             ║  PROCESS                 ║  THREAD                 ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Memory                ║ Own private memory space ║ Shares process memory   ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Isolation             ║ Complete isolation       ║ No isolation            ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Creation cost         ║ Heavy (ms)               ║ Light (μs)              ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Context switch cost   ║ Heavy (100-1000μs)        ║ Light (1-10μs)         ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Communication         ║ IPC (slow, complex)      ║ Shared memory (fast)    ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Crash impact          ║ Contained to process     ║ Can crash entire process║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Data sharing          ║ Hard (explicit IPC)      ║ Easy (direct access)    ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Safety                ║ Safe (OS enforces)       ║ Dangerous (race conds)  ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Scalability           ║ Limited (heavy)          ║ Better (lighter)        ║
╠═══════════════════════╬══════════════════════════╬═════════════════════════╣
║ Use when              ║ True isolation needed    ║ Shared work, fast comm  ║
║                       ║ Different programs       ║ Same program, same data ║
║                       ║ Fault tolerance          ║ Low overhead needed     ║
╚═══════════════════════╩══════════════════════════╩═════════════════════════╝


REAL-WORLD EXAMPLES:

  Uses MULTIPLE PROCESSES:
    Chrome browser:
      One process per tab
      Tab crash → only that tab dies, browser survives
      Security: malicious website can't access other tabs' memory

    Microservices:
      Each service = separate JVM process
      Service A crash → doesn't kill Service B
      Deploy/scale each independently

    Nginx + Worker processes:
      Master process + N worker processes
      Worker crash → master spawns new worker

  Uses MULTIPLE THREADS (within one process):
    Web server handling requests:
      Thread per request (or thread pool)
      All threads share connection pools, caches, config
      Fast communication via shared objects

    Your Spring Boot application:
      Main thread (startup)
      Tomcat request handling threads (dozens to hundreds)
      @Async threads
      @Scheduled threads
      All share Spring beans, DB connections, caches


THE JAVA/SPRING BOOT CONTEXT:
════════════════════════════════

  Your Spring Boot app = ONE JVM process
    Inside = MANY threads:
      - Tomcat/Netty threads (HTTP request handling)
      - HikariCP connection pool threads
      - @Async thread pool threads
      - @Scheduled threads
      - Spring context management threads
      - Your custom threads

  All these threads SHARE:
    → Spring beans (singleton scope = one instance, all threads use it)
    → HikariCP connection pool
    → In-memory caches
    → Static variables
    → Configuration properties

  This is WHY you need to know concurrency.
  This is WHY race conditions happen in Spring Boot apps.
  This is WHY we need synchronized, volatile, thread-safe collections.

  The entire concurrency problem in Spring Boot =
    "Multiple threads sharing the same objects on the Heap"
```

---

## The One Mental Model to Remember Everything

```
╔═════════════════════════════════════════════════════════════════════╗
║                  THE OFFICE BUILDING ANALOGY                        ║
╠═════════════════════════════════════════════════════════════════════╣
║                                                                     ║
║  BUILDING     =  Computer (RAM, CPU, OS)                            ║
║                                                                     ║
║  COMPANY      =  Process                                            ║
║    Each company has its own locked floor                            ║
║    Cannot enter other companies' floors                             ║
║    Must use phone/email to communicate (IPC)                        ║
║    If company goes bankrupt — other companies unaffected            ║
║                                                                     ║
║  EMPLOYEE     =  Thread                                             ║
║    Shares office with all colleagues (shared heap)                  ║
║    Has own desk/notepad (private stack)                             ║
║    Can read/write the shared whiteboard (shared heap objects)       ║
║    If employee goes crazy and erases whiteboard — everyone suffers  ║
║                                                                     ║
║  WHITEBOARD   =  Heap (shared mutable state)                        ║
║    Anyone can read it                                               ║
║    Anyone can write on it                                           ║
║    Without rules (synchronization) → chaos                          ║
║                                                                     ║
║  NOTEPAD      =  Stack (private, per thread)                        ║
║    Only YOU can see your own notepad                                ║
║    Nobody else can read or write it                                 ║
║    Safe by design — no rules needed                                 ║
║                                                                     ║
║  COMPANY MANUAL = Method Area (class definitions, static vars)      ║
║    Everyone in the company reads the same manual                    ║
║    If someone writes in it — everyone sees the change               ║
║    Needs rules if people can write to it                            ║
║                                                                     ║
║  CONCURRENCY = Multiple employees working in the same office        ║
║    Accessing and modifying the SAME whiteboard simultaneously       ║
║    Without coordination = race conditions, corrupted data           ║
║    With coordination (locks, synchronized) = correct but slower     ║
║                                                                     ║
╚═════════════════════════════════════════════════════════════════════╝


BOTTOM LINE:
  Process = isolated box — safe but slow to communicate
  Thread  = shared workspace — fast but needs careful coordination

  Java concurrency = managing multiple employees
                     sharing the same whiteboard (Heap)
                     so they don't step on each other's work
```