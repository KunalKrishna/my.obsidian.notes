# Thread, Runnable, Callable — Comprehensive Notes

---

## The Big Picture — Why Three Things Exist

```
Java's concurrency model evolved over time.
Each addition solved a problem the previous one had.

JAVA 1.0 (1995)
  Thread class introduced.
  To run code concurrently → extend Thread, override run().
  Problem: Java allows only single inheritance.
            Extending Thread means you cannot extend anything else.
            Also mixes WHAT to do (task) with HOW to run (thread).

JAVA 1.0 (same release)
  Runnable interface introduced alongside Thread.
  Separates the TASK (what to do) from the THREAD (how to run).
  Thread accepts a Runnable — runs it on a new thread.
  Problem: run() returns void — cannot return a result.
            run() cannot throw checked exceptions.
            Caller has no way to know when task is done or get its result.

JAVA 5 (2004) — java.util.concurrent
  Callable<V> interface introduced.
  Like Runnable but returns a value V and can throw checked exceptions.
  Works with ExecutorService and Future<V>.
  The result can be retrieved later via Future.get().

EVOLUTION STORY:
  Thread    →  I am both the task AND the runner    (too coupled)
  Runnable  →  I am the task. Thread runs me.       (better, but no result)
  Callable  →  I am the task. I return a result.   (complete solution)
```

```
THE THREE IN ONE PICTURE:

  ┌─────────────────────────────────────────────────────────────────┐
  │                    WHAT YOU WANT TO DO                          │
  │                                                                 │
  │  Just run code      Use Runnable   → void run()                 │
  │  concurrently?      or Thread      → no return, no checked ex   │
  │                                                                 │
  │  Run code AND       Use Callable   → V call()                   │
  │  get a result?                     → returns V, throws Exception│
  └─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │                    HOW TO EXECUTE IT                            │
  │                                                                 │
  │  Raw thread         new Thread(runnable).start()               │
  │  (direct)                                                       │
  │                                                                 │
  │  Thread pool        executorService.execute(runnable)           │
  │  (preferred)        executorService.submit(runnable)            │
  │                     executorService.submit(callable)            │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Part 1 — The Thread Class

### What Thread Is

```
Thread is the concrete class that represents a thread of execution.
Package: java.lang (no import needed)
Extends:  Object
Implements: Runnable

When you call thread.start():
  JVM asks the OS to create a REAL OS-level thread.
  OS allocates a new Stack for that thread.
  OS assigns a Thread ID.
  OS registers the thread with the scheduler.
  JVM calls the thread's run() method on that new thread.

The Thread class has TWO roles:
  Role 1: TASK DEFINITION — override run() to define what to do
  Role 2: THREAD CONTROL  — start(), join(), interrupt(), sleep(), etc.

When you extend Thread, you merge both roles into one class.
This is the design problem — see Runnable for the fix.
```

### Thread Class — Complete API

```java
// ═══════════════════════════════════════════════════════════════
//  THREAD CLASS — EVERY METHOD YOU NEED TO KNOW
// ═══════════════════════════════════════════════════════════════
public class ThreadAPIComplete {

    public static void main(String[] args) throws InterruptedException {

        // ── CONSTRUCTION ──────────────────────────────────────────

        Thread t1 = new Thread();
        // Creates thread with empty run() — does nothing
        // Rarely used directly

        Thread t2 = new Thread(/* Runnable */ () -> System.out.println("task"));
        // Creates thread that runs the given Runnable
        // Most common constructor

        Thread t3 = new Thread(/* Runnable */ () -> {}, "my-thread-name");
        // Creates thread with Runnable AND a name
        // Name appears in logs, thread dumps — always name your threads!

        ThreadGroup group = new ThreadGroup("my-group");
        Thread t4 = new Thread(group, () -> {}, "grouped-thread");
        // Creates thread in a specific ThreadGroup
        // Groups allow batch operations (interrupt all, etc.)

        // ── IDENTITY ──────────────────────────────────────────────

        System.out.println(t2.getName());
        // Returns thread name. Default: "Thread-0", "Thread-1", etc.

        t2.setName("worker-thread");
        // Set a meaningful name — critical for debugging and thread dumps

        System.out.println(t2.getId());
        // Returns unique long ID assigned by JVM
        // Not the OS thread ID — JVM's own sequential counter

        System.out.println(t2.getThreadGroup());
        // Returns the ThreadGroup this thread belongs to
        // Default: same group as parent thread

        // ── LIFECYCLE CONTROL ─────────────────────────────────────

        t2.start();
        // Creates OS thread, begins execution of run()
        // Returns IMMEDIATELY — does NOT wait for thread to finish
        // Can only be called ONCE — calling again throws IllegalThreadStateException
        // After start(): thread is in RUNNABLE state

        t2.join();
        // Calling thread BLOCKS until t2 DIES (reaches TERMINATED)
        // Calling thread enters WAITING state
        // When t2 dies: JVM calls notifyAll() on t2 object → caller wakes

        t3.start();
        t3.join(1000);
        // Calling thread waits at most 1000ms for t3 to die
        // If t3 finishes first → join() returns early
        // If timeout expires → join() returns regardless
        // After join(timeout): check t3.isAlive() to know which happened

        t3.join(1000, 500000);
        // Waits at most 1000ms + 500000ns = 1.5 seconds
        // Second param: additional nanoseconds (0 to 999999)

        // ── INTERRUPTION ──────────────────────────────────────────

        Thread longTask = new Thread(() -> {
            try {
                Thread.sleep(10_000);        // sleeping for 10 seconds
            } catch (InterruptedException e) {
                System.out.println("Was interrupted!");
                // When interrupted during sleep/wait/join:
                //   InterruptedException is thrown
                //   Interrupt flag is CLEARED automatically
                Thread.currentThread().interrupt(); // re-set the flag
                // Best practice: re-interrupt after catching
                // So callers can also check interrupt status
            }
        });
        longTask.start();
        Thread.sleep(500);
        longTask.interrupt();
        // Sets the interrupt flag on longTask
        // If longTask is in WAITING/TIMED_WAITING/BLOCKED:
        //   It is woken up, interrupt flag cleared, InterruptedException thrown
        // If longTask is RUNNABLE:
        //   Interrupt flag is SET but nothing immediate happens
        //   Thread must CHECK the flag via isInterrupted()

        System.out.println(longTask.isInterrupted());
        // INSTANCE method — checks interrupt flag of THIS thread
        // Does NOT clear the flag — non-destructive read

        System.out.println(Thread.interrupted());
        // STATIC method — checks interrupt flag of CURRENT thread
        // CLEARS the flag after reading — destructive read!
        // Almost always called as Thread.interrupted() (static call on class)

        // ── SLEEPING ──────────────────────────────────────────────

        Thread.sleep(1000);
        // STATIC — pauses the CURRENT thread for at least 1000ms
        // Does NOT release any locks the thread holds!
        // Puts thread in TIMED_WAITING state
        // Can be interrupted → throws InterruptedException
        // "at least" because OS scheduling may add delays

        Thread.sleep(1000, 500000);
        // Sleep for 1000ms + 500000ns

        // ── YIELDING ──────────────────────────────────────────────

        Thread.yield();
        // STATIC — hint to scheduler: "I am willing to give up CPU"
        // Scheduler MAY or MAY NOT honor this
        // Moves thread from RUNNING back to RUNNABLE (ready queue)
        // Almost never used in production code
        // Useful in tight spin-loops to reduce CPU burn

        // ── PRIORITY ──────────────────────────────────────────────

        t3.setPriority(Thread.MAX_PRIORITY);  // 10
        t3.setPriority(Thread.NORM_PRIORITY); // 5 (default)
        t3.setPriority(Thread.MIN_PRIORITY);  // 1
        // Range: 1 (MIN) to 10 (MAX)
        // Hint to OS scheduler — NOT a guarantee
        // OS may completely ignore it
        // NEVER write correctness-critical code that depends on priority

        System.out.println(t3.getPriority());

        // ── STATE INSPECTION ──────────────────────────────────────

        System.out.println(t3.getState());
        // Returns Thread.State enum:
        // NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED

        System.out.println(t3.isAlive());
        // true if thread has been started AND not yet terminated
        // false if NEW or TERMINATED

        // ── DAEMON ────────────────────────────────────────────────

        Thread daemon = new Thread(() -> {
            while(true) { /* background work */ }
        });
        daemon.setDaemon(true);
        // MUST call before start() — else IllegalThreadStateException
        // JVM exits when all non-daemon threads finish
        // Daemon threads are killed abruptly on JVM exit
        // No finally blocks, no cleanup guaranteed for daemons
        // Use for: background monitoring, heartbeats, cleanup tasks

        System.out.println(daemon.isDaemon()); // true

        // ── CURRENT THREAD ────────────────────────────────────────

        Thread current = Thread.currentThread();
        // STATIC — returns reference to the thread currently executing
        // Works from ANY context — main, lambda, anonymous class
        // Use to get name, ID, state of the executing thread
        System.out.println(current.getName());   // "main"
        System.out.println(current.getState());  // RUNNABLE

        // ── STACK TRACE ───────────────────────────────────────────

        StackTraceElement[] trace = t3.getStackTrace();
        // Returns the stack trace of THIS thread (t3), not current thread
        // Snapshot — may be empty if thread not started or terminated
        // Use: debugging, monitoring, logging
        for (StackTraceElement frame : trace) {
            System.out.println(frame);
        }

        Thread.currentThread().getStackTrace();
        // Gets stack trace of the currently running thread
    }
}
```

### Extending Thread — The Old Way

```java
// ═══════════════════════════════════════════════════════════════
//  WAY 1: EXTENDING THREAD
//  Old approach — understanding it helps, using it: avoid
// ═══════════════════════════════════════════════════════════════
public class ExtendingThread {

    // ── BASIC EXTENSION ───────────────────────────────────────────
    static class DownloadThread extends Thread {

        private final String url;
        private String result; // will hold the result (workaround for no return)

        DownloadThread(String url) {
            super("downloader-" + url); // give thread a meaningful name
            this.url = url;
        }

        @Override
        public void run() {
            // This is the task AND the thread in one class
            // Everything in here runs on the new thread
            System.out.println(Thread.currentThread().getName()
                    + " downloading: " + url);
            result = "content of " + url; // simulate download result
        }

        public String getResult() { return result; }
    }

    public static void main(String[] args) throws InterruptedException {

        DownloadThread d1 = new DownloadThread("http://site1.com");
        DownloadThread d2 = new DownloadThread("http://site2.com");

        d1.start();
        d2.start();

        d1.join(); // wait for d1 to die
        d2.join(); // wait for d2 to die

        // Now safe to read results — threads are dead
        System.out.println("Result 1: " + d1.getResult());
        System.out.println("Result 2: " + d2.getResult());
    }
}
```

### Why Extending Thread Is Problematic

```java
// ═══════════════════════════════════════════════════════════════
//  THE PROBLEMS WITH EXTENDING THREAD
// ═══════════════════════════════════════════════════════════════
public class ExtendingThreadProblems {

    // ── PROBLEM 1: Single Inheritance ─────────────────────────────
    static class Animal {
        void breathe() { System.out.println("breathing"); }
    }

    // I want to run Animal on a thread AND have Animal behavior
    static class AnimalThread extends Thread {   // ← extends Thread
        // Now I CANNOT also extend Animal
        // Java does not allow:
        //   class AnimalThread extends Thread extends Animal ← ILLEGAL
        void breathe() {} // must copy behavior — terrible design
    }

    static class BetterAnimal extends Animal implements Runnable { // ✓
        // This works! Runnable is an interface, not a class
        // Can extend Animal AND implement Runnable
        @Override
        public void run() { breathe(); }
    }

    // ── PROBLEM 2: TASK mixed with THREAD ─────────────────────────
    // Extending Thread: the HOW (thread) and WHAT (task) are merged

    static class ReportGeneratorThread extends Thread {
        // This class is both:
        //   - A Thread (knows how to run on OS thread)
        //   - A ReportGenerator (knows what to do)
        // If I want to run the same task on:
        //   - A cached thread pool   → can't, no constructor for that
        //   - A fixed thread pool    → can't
        //   - A scheduled executor  → can't
        //   - A virtual thread      → can't
        // The task is LOCKED into being a Thread.
        @Override
        public void run() { /* generate report */ }
    }

    static class ReportGeneratorTask implements Runnable {
        // This class is ONLY the task — knows nothing about execution
        // I can run it:
        //   new Thread(new ReportGeneratorTask()).start()       ← raw thread
        //   executor.execute(new ReportGeneratorTask())         ← thread pool
        //   Thread.ofVirtual().start(new ReportGeneratorTask()) ← virtual
        //   scheduledExecutor.schedule(new ReportGeneratorTask(), 1, SECONDS)
        // Complete flexibility!
        @Override
        public void run() { /* generate report */ }
    }

    // ── PROBLEM 3: Cannot reuse Thread objects ─────────────────────
    static class TaskThread extends Thread {
        @Override
        public void run() { System.out.println("working"); }
    }

    public static void main(String[] args) {
        TaskThread t = new TaskThread();
        t.start();

        // t.start(); // ← IllegalThreadStateException!
        // A Thread object can only be started ONCE.
        // Once started, it cannot be restarted even after it terminates.
        // With Runnable: create one task, submit to pool multiple times.
        // With Thread: create new Thread object every time — wasteful.
    }
}
```

---

## Part 2 — The Runnable Interface

### What Runnable Is

```
Runnable is the simplest task abstraction in Java.
Package: java.lang (no import needed)
Type:     @FunctionalInterface (since Java 8)

@FunctionalInterface
public interface Runnable {
    void run();
}

That is the entire interface. One method. Void return. No exceptions.

Runnable separates WHAT to do (the task)
from HOW to run it (Thread, pool, virtual thread, etc.).

The Thread class itself implements Runnable.
ExecutorService.execute() accepts Runnable.
ExecutorService.submit() accepts Runnable.
Every execution mechanism in Java accepts Runnable.
It is the universal task type.
```

### Runnable — All Creation Styles

```java
// ═══════════════════════════════════════════════════════════════
//  RUNNABLE — ALL WAYS TO CREATE AND USE
// ═══════════════════════════════════════════════════════════════
public class RunnableAllStyles {

    // ── STYLE 1: Named class implementing Runnable ─────────────────
    // Use when: task is complex, has many fields, needs to be reused
    static class DataProcessingTask implements Runnable {
        private final String dataFile;
        private final int    batchSize;

        DataProcessingTask(String dataFile, int batchSize) {
            this.dataFile  = dataFile;
            this.batchSize = batchSize;
        }

        @Override
        public void run() {
            System.out.println("Processing " + dataFile
                    + " in batches of " + batchSize
                    + " on " + Thread.currentThread().getName());
            // complex processing logic here
        }
    }

    // ── STYLE 2: Anonymous class ───────────────────────────────────
    // Use when: one-time use, slightly more than a lambda
    // Pre-Java 8 style — lambdas are cleaner now
    static void anonymousStyle() {
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("Anonymous Runnable running");
            }
        };
        new Thread(task).start();
    }

    // ── STYLE 3: Lambda (Java 8+) ──────────────────────────────────
    // Use when: simple, short, one-liner tasks
    // Most common in modern Java code
    static void lambdaStyle() {
        Runnable task = () -> System.out.println("Lambda Runnable running");
        new Thread(task).start();
    }

    // ── STYLE 4: Method Reference ──────────────────────────────────
    // Use when: the logic already exists as a method
    static void doWork() {
        System.out.println("doWork() running on "
                + Thread.currentThread().getName());
    }

    static void methodRefStyle() {
        Runnable task = RunnableAllStyles::doWork; // method reference
        new Thread(task).start();

        // Or instance method reference
        RunnableAllStyles instance = new RunnableAllStyles();
        Runnable task2 = instance::instanceWork;
        new Thread(task2).start();
    }

    void instanceWork() {
        System.out.println("instanceWork() running");
    }

    // ── ALL STYLES TOGETHER ────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        // Style 1: Named class
        Runnable named = new DataProcessingTask("data.csv", 100);
        Thread t1 = new Thread(named, "named-thread");

        // Style 2: Anonymous class
        Runnable anon = new Runnable() {
            @Override
            public void run() {
                System.out.println("Anon: " + Thread.currentThread().getName());
            }
        };
        Thread t2 = new Thread(anon, "anon-thread");

        // Style 3: Lambda — most idiomatic today
        Runnable lambda = () ->
                System.out.println("Lambda: " + Thread.currentThread().getName());
        Thread t3 = new Thread(lambda, "lambda-thread");

        // Style 4: Method reference
        Thread t4 = new Thread(RunnableAllStyles::doWork, "method-ref-thread");

        // Submit same Runnable to multiple execution contexts
        ExecutorService pool     = Executors.newFixedThreadPool(2);
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        // The SAME task, different executors — Runnable enables this
        pool.execute(named);        // run in thread pool
        pool.execute(lambda);       // run in thread pool
        scheduler.schedule(named, 2, TimeUnit.SECONDS); // run after 2s delay

        t1.start(); t2.start(); t3.start(); t4.start();
        t1.join();  t2.join();  t3.join();  t4.join();
        pool.shutdown();
        scheduler.shutdown();
    }
}
```

### Runnable — Accessing Outer State

```java
// ═══════════════════════════════════════════════════════════════
//  RUNNABLE — ACCESSING STATE FROM ENCLOSING CONTEXT
//  Lambda closures — effectively final rule
// ═══════════════════════════════════════════════════════════════
public class RunnableClosures {

    static int sharedCounter = 0; // static — accessible, mutable

    public static void main(String[] args) throws InterruptedException {

        String taskName = "report-generator"; // local variable
        int    batchId  = 42;                 // local variable

        // ── Capturing local variables (must be effectively final) ──
        Runnable task = () -> {
            // taskName and batchId are CAPTURED from enclosing scope
            // They must be EFFECTIVELY FINAL:
            //   never reassigned after being captured
            System.out.println("Task: " + taskName + " batch: " + batchId);
        };

        // taskName = "other"; // ← This would make taskName NOT effectively final
        //                     //   Compiler error: "Variable used in lambda
        //                     //   should be effectively final"

        // ── Why effectively final? ─────────────────────────────────
        // Lambda runs on a DIFFERENT thread.
        // Local variable lives on the CURRENT thread's stack.
        // When current thread returns, its stack frame is GONE.
        // The lambda has no access to the original stack frame.
        // JVM COPIES the value into the lambda at capture time.
        // If you could reassign → which copy is correct? Undefined.
        // So Java enforces: you can capture, but not reassign.

        // ── What you CAN mutate: arrays (reference is final, content is not) ─
        int[] result = new int[1]; // reference is final, but result[0] can change

        Runnable compute = () -> {
            result[0] = 42; // modifying array CONTENT is allowed
            // result = new int[2]; // ← NOT allowed — reassigning reference
        };

        Thread t = new Thread(compute);
        t.start();
        t.join();
        System.out.println("Result: " + result[0]); // 42

        // ── Static fields: accessible and mutable ─────────────────
        Runnable counter = () -> {
            sharedCounter++; // accessing and modifying static field
            // This is a RACE CONDITION if multiple threads run this!
            // Static fields are shared → need synchronization
        };

        Thread t1 = new Thread(counter);
        Thread t2 = new Thread(counter);
        t1.start(); t2.start();
        t1.join(); t2.join();
        // sharedCounter may be 1 or 2 — race condition!
    }
}
```

### Runnable — The Fatal Limitation

```java
// ═══════════════════════════════════════════════════════════════
//  RUNNABLE'S LIMITATIONS — what it CANNOT do
//  These limitations are exactly what Callable solves
// ═══════════════════════════════════════════════════════════════
public class RunnableLimitations {

    // ── LIMITATION 1: Cannot return a value ───────────────────────
    static String computeResult(int input) {
        return "result: " + (input * 2);
    }

    static void limitation1_noReturnValue() throws InterruptedException {
        String[] holder = new String[1]; // workaround: array trick

        Runnable task = () -> {
            holder[0] = computeResult(21); // store result in shared variable
        };

        Thread t = new Thread(task);
        t.start();
        t.join(); // wait for thread to finish

        System.out.println(holder[0]); // awkward workaround
        // Problems with this workaround:
        //   1. Ugly — extra array just to pass a value
        //   2. No type safety
        //   3. Thread safety: if multiple threads write to holder[0]
        //   4. No way to know if task succeeded or failed
        //   → Callable solves this cleanly
    }

    // ── LIMITATION 2: Cannot throw checked exceptions ─────────────
    static void riskyOperation() throws Exception {
        throw new Exception("Something went wrong");
    }

    static void limitation2_noCheckedException() {
        Runnable task = () -> {
            try {
                riskyOperation(); // MUST wrap in try-catch inside run()
            } catch (Exception e) {
                // What do you do here?
                // 1. Log it — caller never knows it failed
                // 2. Wrap in RuntimeException — loses checked exception type
                // 3. Store in a variable — same array trick as above
                Thread.currentThread().interrupt(); // for InterruptedException only
                throw new RuntimeException(e); // wrapping — messy
            }
        };

        // Runnable.run() signature: public void run()
        // It cannot declare "throws Exception"
        // So any checked exception MUST be caught inside run()
        // Caller has NO WAY to know if the task threw an exception
        // → Callable solves this cleanly
    }

    // ── LIMITATION 3: Caller cannot easily know when task is done ─
    static void limitation3_noDoneSignal() throws InterruptedException {
        boolean[] done = new boolean[1]; // workaround

        Runnable task = () -> {
            // do work
            done[0] = true; // signal completion via shared variable
        };

        Thread t = new Thread(task);
        t.start();

        // To wait for completion:
        t.join(); // works for raw threads
        // But if using ExecutorService.execute(runnable):
        //   execute() returns void — no handle to the task!
        //   You can't join the task
        //   Can't know when it's done (unless you implement manually)
        //   → ExecutorService.submit(callable) returns Future → solves this
    }
}
```

---

## Part 3 — The Callable Interface

### What Callable Is

```
Callable<V> is a task that returns a result and can throw an exception.
Package: java.util.concurrent (must import)

@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}

Compare with Runnable:
  Runnable:  void run()            — no return, no checked exception
  Callable:  V    call() throws Exception — has return, has exception

Callable was designed to work with:
  ExecutorService.submit(Callable<T> task) → returns Future<T>
  The result is obtained later via Future<T>.get()

Callable cannot be passed directly to new Thread() constructor.
  new Thread(callable) ← COMPILE ERROR — Thread accepts only Runnable
  Callable is for ExecutorService, not for raw Thread creation.
```

### Callable — Complete Deep Dive

```java
import java.util.concurrent.*;
import java.util.*;

// ═══════════════════════════════════════════════════════════════
//  CALLABLE — COMPLETE DEEP DIVE
// ═══════════════════════════════════════════════════════════════
public class CallableDeepDive {

    // ── DEFINING A CALLABLE — Named Class ─────────────────────────
    static class PrimeCheckerTask implements Callable<Boolean> {
        // Type parameter Boolean means call() will return Boolean

        private final int number;

        PrimeCheckerTask(int number) {
            this.number = number;
        }

        @Override
        public Boolean call() throws Exception {
            // call() CAN throw checked exceptions — no try-catch needed!
            if (number < 2) return false;
            if (number == 2) return true;

            // Simulate a checked exception situation
            if (number < 0) {
                throw new IllegalArgumentException(
                        "Number cannot be negative: " + number);
                // Caller will receive this via ExecutionException
                // when they call future.get()
            }

            for (int i = 2; i <= Math.sqrt(number); i++) {
                if (number % i == 0) return false;
            }
            return true; // Returns a value! Unlike Runnable.
        }
    }

    // ── CALLABLE AS LAMBDA ─────────────────────────────────────────
    static Callable<String> lambdaCallable(String input) {
        return () -> {
            // Lambda for Callable — same as Runnable lambda
            // but return type and exception are inferred
            Thread.sleep(1000); // can throw InterruptedException — it's checked!
                                // no need to catch — call() declares throws Exception
            return "Processed: " + input.toUpperCase();
        };
    }

    // ── CALLABLE WITH RICH RESULT TYPE ────────────────────────────
    static class ReportTask implements Callable<Map<String, Integer>> {
        // Callable<V> — V can be any type: String, List, Map, custom object

        @Override
        public Map<String, Integer> call() {
            Map<String, Integer> report = new HashMap<>();
            report.put("totalRecords", 1000);
            report.put("successCount", 980);
            report.put("failureCount", 20);
            return report; // returns rich Map result
        }
    }

    public static void main(String[] args) throws Exception {

        ExecutorService executor = Executors.newFixedThreadPool(4);

        // ── SUBMITTING CALLABLE → GETTING FUTURE ─────────────────
        Future<Boolean> future1 = executor.submit(new PrimeCheckerTask(97));
        //   ↑ submit() immediately returns a Future<Boolean>
        //   Thread is running the task RIGHT NOW (or queued to run)
        //   future1 is a handle to the PENDING result
        //   We can do other work here while task runs in background

        System.out.println("Task submitted. Doing other work...");

        // Do other things while callable runs in background
        Future<String>              future2 = executor.submit(lambdaCallable("hello"));
        Future<Map<String, Integer>> future3 = executor.submit(new ReportTask());

        // ── GETTING THE RESULT ────────────────────────────────────
        Boolean isPrime = future1.get();
        //   ↑ BLOCKS the calling thread until PrimeCheckerTask.call() returns
        //   Returns the value that call() returned (true or false)
        //   If task threw exception → future1.get() throws ExecutionException
        //   If thread was interrupted → throws InterruptedException

        System.out.println("97 is prime: " + isPrime);

        // ── FUTURE.GET WITH TIMEOUT ───────────────────────────────
        try {
            String result = future2.get(2, TimeUnit.SECONDS);
            //   ↑ Wait at most 2 seconds for the result
            //   If not done in 2 seconds → throws TimeoutException
            //   TimeoutException does NOT cancel the task!
            //   Task keeps running — you just gave up waiting
            System.out.println("Result: " + result);
        } catch (TimeoutException e) {
            System.out.println("Task took too long — cancelling");
            future2.cancel(true); // true = interrupt the running thread
        }

        // ── HANDLING EXCEPTIONS FROM CALLABLE ────────────────────
        Future<Boolean> badFuture = executor.submit(new PrimeCheckerTask(-5));
        try {
            Boolean result = badFuture.get(); // task threw exception
        } catch (ExecutionException e) {
            // ExecutionException WRAPS the original exception
            Throwable cause = e.getCause(); // get the original exception
            System.out.println("Task failed: " + cause.getMessage());
            // cause is IllegalArgumentException: "Number cannot be negative: -5"
        }

        // ── FUTURE METHODS ─────────────────────────────────────────

        Future<Boolean> future4 = executor.submit(new PrimeCheckerTask(101));

        future4.isDone();
        // true if task completed (successfully, with exception, or cancelled)
        // false if still running or pending

        future4.isCancelled();
        // true ONLY if task was cancelled via future.cancel()
        // false otherwise

        future4.cancel(false);
        // Attempt to cancel the task.
        // false = don't interrupt if already running — let it finish
        // true  = interrupt running thread (if it's interruptible)
        // Returns true if cancel was successful
        // Returns false if task already completed or was already cancelled
        // Once cancelled: future.get() throws CancellationException

        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

### Callable — Exception Handling in Detail

```java
// ═══════════════════════════════════════════════════════════════
//  CALLABLE EXCEPTION HANDLING — complete picture
// ═══════════════════════════════════════════════════════════════
public class CallableExceptions {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // ── SCENARIO 1: Normal completion ─────────────────────────
        Future<Integer> normal = executor.submit(() -> 42);
        try {
            System.out.println(normal.get()); // prints 42
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }

        // ── SCENARIO 2: Callable throws checked exception ─────────
        Future<Integer> checked = executor.submit(() -> {
            throw new IOException("File not found");
            // IOException is a checked exception
            // In Runnable: would need try-catch inside run()
            // In Callable: can throw freely — call() declares throws Exception
        });
        try {
            checked.get(); // task threw → ExecutionException wraps IOException
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            System.out.println(cause.getClass().getName()); // java.io.IOException
            System.out.println(cause.getMessage());         // File not found
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // ── SCENARIO 3: Callable throws unchecked exception ────────
        Future<Integer> unchecked = executor.submit(() -> {
            throw new IllegalStateException("Bad state");
        });
        try {
            unchecked.get(); // also wrapped in ExecutionException
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            System.out.println(cause.getClass().getName()); // IllegalStateException
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // ── SCENARIO 4: Calling thread interrupted while waiting ───
        Future<Integer> slow = executor.submit(() -> {
            Thread.sleep(10_000); // takes 10 seconds
            return 99;
        });
        Thread.currentThread().interrupt(); // interrupt current thread first
        try {
            slow.get(); // current thread is interrupted → throws InterruptedException
        } catch (InterruptedException e) {
            System.out.println("Interrupted while waiting for result");
            Thread.currentThread().interrupt(); // re-set interrupt flag
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        executor.shutdown();
    }
}
```

```
EXCEPTION FLOW DIAGRAM:

  Callable.call()                Future.get()
  ───────────────                ────────────

  returns normally ──────────────► get() returns the value

  throws CheckedException ───────► get() throws ExecutionException
  (IOException, etc.)              cause = CheckedException

  throws UncheckedException ─────► get() throws ExecutionException
  (RuntimeException, etc.)         cause = UncheckedException

  throws Error ──────────────────► get() throws ExecutionException
  (OutOfMemoryError, etc.)         cause = Error

  task was cancelled ────────────► get() throws CancellationException

  calling thread interrupted ────► get() throws InterruptedException
  (while waiting in get())         (task may still be running!)
```

---

## Part 4 — Thread vs Runnable vs Callable — Complete Comparison

### Feature Matrix

```
╔═══════════════════════════╦══════════════════╦══════════════════╦══════════════════╗
║  FEATURE                  ║     Thread       ║    Runnable      ║    Callable<V>   ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Type                      ║ Class            ║ Interface        ║ Interface        ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Package                   ║ java.lang        ║ java.lang        ║ java.util        ║
║                           ║ (no import)      ║ (no import)      ║  .concurrent     ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Method to override        ║ run()            ║ run()            ║ call()           ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Return type               ║ void             ║ void             ║ V (generic)      ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Throws checked exception? ║ ✗ No             ║ ✗ No             ║ ✓ Yes (Exception)║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Functional Interface?     ║ ✗ No             ║ ✓ Yes            ║ ✓ Yes            ║
║ (usable as lambda?)       ║                  ║                  ║                  ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Can extend other class?   ║ ✗ No             ║ ✓ Yes            ║ ✓ Yes            ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Pass to new Thread()?     ║ ✓ Yes (IS a      ║ ✓ Yes            ║ ✗ No             ║
║                           ║    Thread)       ║                  ║                  ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Pass to executor.execute? ║ ✓ (implements    ║ ✓ Yes            ║ ✗ No             ║
║                           ║   Runnable)      ║                  ║                  ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Pass to executor.submit?  ║ ✓ (via Runnable) ║ ✓ Yes            ║ ✓ Yes            ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Returns Future?           ║ ✗ (join only)    ║ Future<?> (Void) ║ ✓ Future<V>      ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Java version introduced   ║ Java 1.0         ║ Java 1.0         ║ Java 5           ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Reusable (submit many     ║ ✗ No (start()    ║ ✓ Yes            ║ ✓ Yes            ║
║ times)?                   ║   only once)     ║                  ║                  ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Separation of concerns    ║ ✗ Task + Thread  ║ ✓ Task only      ║ ✓ Task only      ║
║                           ║   merged         ║                  ║                  ║
╠═══════════════════════════╬══════════════════╬══════════════════╬══════════════════╣
║ Best for                  ║ Legacy code,     ║ Fire-and-forget, ║ Tasks needing    ║
║                           ║ understanding    ║ background work, ║ result/exception ║
║                           ║ internals        ║ event handlers   ║ handling         ║
╚═══════════════════════════╩══════════════════╩══════════════════╩══════════════════╝
```

### The Same Task Written Three Ways

```java
// ═══════════════════════════════════════════════════════════════
//  SAME TASK — written as Thread, Runnable, Callable
//  Task: compute sum of 1 to N
// ═══════════════════════════════════════════════════════════════
public class SameTaskThreeWays {

    static final int N = 1_000_000;

    // ── AS THREAD ─────────────────────────────────────────────────
    static class SumThread extends Thread {
        long result; // must store result in field — no return from run()

        @Override
        public void run() {
            long sum = 0;
            for (int i = 1; i <= N; i++) sum += i;
            result = sum;
        }
    }

    // ── AS RUNNABLE ───────────────────────────────────────────────
    static class SumRunnable implements Runnable {
        long result;         // same workaround — store result in field

        @Override
        public void run() {
            long sum = 0;
            for (int i = 1; i <= N; i++) sum += i;
            result = sum;    // no way to return — must use shared variable
        }
    }

    // ── AS CALLABLE ───────────────────────────────────────────────
    static class SumCallable implements Callable<Long> {
        @Override
        public Long call() {
            long sum = 0;
            for (int i = 1; i <= N; i++) sum += i;
            return sum;      // clean return — no shared variable needed
        }
    }

    public static void main(String[] args) throws Exception {

        // ── USING THREAD ──────────────────────────────────────────
        SumThread sumThread = new SumThread();
        sumThread.start();
        sumThread.join();         // must join to safely read result
        System.out.println("Thread  result: " + sumThread.result);
        // Problems:
        //   1. result field exposed publicly (bad OOP)
        //   2. Must remember to join() before reading
        //   3. No exception signaling
        //   4. Can't reuse the SumThread object (start() only once)

        // ── USING RUNNABLE ────────────────────────────────────────
        SumRunnable sumRunnable = new SumRunnable();
        Thread t = new Thread(sumRunnable, "sum-thread");
        t.start();
        t.join();
        System.out.println("Runnable result: " + sumRunnable.result);
        // Same problems as Thread (minus the single-inheritance issue)
        // Runnable IS reusable — can submit to different threads/pools
        // But result retrieval is still awkward

        // ── USING CALLABLE ────────────────────────────────────────
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Callable<Long>  sumCallable = new SumCallable();

        Future<Long> future = executor.submit(sumCallable);
        // submit returns immediately — task running in background

        // Can do other work here...

        Long result = future.get(); // blocks until result is ready
        System.out.println("Callable result: " + result);
        // Clean!
        //   1. Result comes back via return value (natural Java)
        //   2. No public mutable fields
        //   3. Exception handling via ExecutionException
        //   4. Callable is reusable — can submit many times
        //   5. Future provides isDone(), cancel(), get(timeout)

        executor.shutdown();
    }
}
```

### submit(Runnable) vs submit(Callable) — The Subtle Difference

```java
// ═══════════════════════════════════════════════════════════════
//  submit(Runnable) vs submit(Callable) — subtle differences
// ═══════════════════════════════════════════════════════════════
public class SubmitDifference {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // ── submit(Runnable) → Future<?> ──────────────────────────
        Future<?> runnableFuture = executor.submit(() -> {
            System.out.println("Runnable task running");
            // returns void
        });

        Object runnableResult = runnableFuture.get();
        System.out.println("Runnable result: " + runnableResult);
        // Prints: null
        // Future<?> from Runnable always returns null on get()
        // The only useful thing it tells you: task is done (no exception thrown)
        // It's essentially a "done" signal

        // ── submit(Runnable, T result) → Future<T> ─────────────────
        // Overload: you can provide the result value to return
        Future<String> runnableWithResult = executor.submit(
            () -> System.out.println("Running"),
            "hardcoded-result"  // this is returned on get()
        );
        String val = runnableWithResult.get();
        System.out.println(val); // "hardcoded-result"
        // Not commonly used — but available

        // ── submit(Callable<V>) → Future<V> ───────────────────────
        Future<Integer> callableFuture = executor.submit(() -> {
            return 42; // actual computed result
        });

        Integer callableResult = callableFuture.get();
        System.out.println("Callable result: " + callableResult);
        // Prints: 42
        // Future<V> from Callable returns the actual computed value

        // ── execute(Runnable) → void ───────────────────────────────
        executor.execute(() -> System.out.println("execute - no future"));
        // execute() returns NOTHING
        // No way to know when it's done (without external coordination)
        // No way to handle exceptions (they are silently swallowed
        //   unless an UncaughtExceptionHandler is set!)
        // Use only for true fire-and-forget tasks

        executor.shutdown();
    }
}
```

```
SUBMIT vs EXECUTE:

  executor.execute(runnable)
    Returns: void
    Exception handling: swallowed (goes to UncaughtExceptionHandler)
    Done detection: none
    Use: fire-and-forget, logging, metrics, notifications

  executor.submit(runnable)   → Future<?>
    Returns: Future<?> (always null on get())
    Exception handling: wrapped in ExecutionException on get()
    Done detection: future.isDone(), future.get()
    Use: when you need to know if task finished or threw exception

  executor.submit(callable)   → Future<V>
    Returns: Future<V> (actual result on get())
    Exception handling: wrapped in ExecutionException on get()
    Done detection: future.isDone(), future.get()
    Use: when you need the task's result
```

---

## Part 5 — How They Work Together — Internals

### The Thread Execution Chain

```
HOW start() ACTUALLY WORKS:

  Java Code:
    Thread t = new Thread(runnable);
    t.start();

  What happens step by step:

  1. t.start() calls start0() — a NATIVE method
     (implemented in C/C++ in JVM source)

  2. start0() asks the OS to create a new OS thread
     OS allocates:
       - Thread Stack (default ~1MB for JVM threads)
       - OS Thread Control Block
       - OS Thread ID
       - CPU registers snapshot space

  3. OS thread is created and marks it as RUNNABLE
     (ready to run, waiting for CPU time slice)

  4. JVM maps Java Thread object ↔ OS Thread
     (this is the "platform thread" concept in Java 21)

  5. When OS scheduler gives this thread a CPU time slice:
     JVM calls thread.run() on that OS thread

  6. thread.run() — what does it do?

     Looking at Thread.run() source:
     public void run() {
         if (target != null) {    // target = the Runnable passed to constructor
             target.run();        // delegates to Runnable's run()
         }
         // if no target (direct Thread extension): subclass run() is called
     }

  7. When run() returns → thread state becomes TERMINATED
     JVM calls notifyAll() on the Thread object
     (this wakes threads blocked in join())


THE DELEGATION CHAIN:
  Thread.start()
    → OS creates thread
      → Thread.run()
          → Runnable.run()  (if Thread has a Runnable target)
              → your code

  OR for Callable via ExecutorService:
  ExecutorService.submit(callable)
    → FutureTask wraps callable  (FutureTask implements Runnable!)
    → ThreadPoolExecutor.execute(futureTask)
      → Worker Thread's run()
        → FutureTask.run()
          → callable.call()
            → your code
            → result stored in FutureTask
    → future.get() retrieves result from FutureTask
```

### FutureTask — The Bridge Between Callable and Runnable

```java
// ═══════════════════════════════════════════════════════════════
//  FutureTask — the bridge that makes Callable work with Thread
//  FutureTask implements both Runnable AND Future
// ═══════════════════════════════════════════════════════════════
public class FutureTaskDemo {

    public static void main(String[] args) throws Exception {

        // FutureTask wraps Callable, makes it act like Runnable
        // AND provides Future interface for result retrieval
        Callable<String> callable = () -> {
            Thread.sleep(1000);
            return "I am the result";
        };

        FutureTask<String> futureTask = new FutureTask<>(callable);
        //   ↑ FutureTask implements:
        //     Runnable → can be passed to Thread, execute(), submit()
        //     Future   → has get(), isDone(), cancel()

        // Can pass FutureTask to raw Thread (because it's a Runnable)
        Thread thread = new Thread(futureTask, "futureTask-thread");
        thread.start();

        // Can also pass to ExecutorService
        // ExecutorService executor = Executors.newSingleThreadExecutor();
        // executor.submit(futureTask); // works too

        System.out.println("Task submitted — doing other work");

        // Get the result — blocks until callable.call() returns
        String result = futureTask.get();
        System.out.println("Result: " + result);

        // FutureTask state machine:
        // NEW → COMPLETING → NORMAL (normal completion)
        //    → EXCEPTIONAL (exception thrown)
        //    → CANCELLED   (cancel() called before run)
        //    → INTERRUPTING → INTERRUPTED (cancel(true) called while running)

        // This is what ExecutorService.submit(callable) does internally:
        //   FutureTask<V> ftask = new FutureTask<>(callable);
        //   execute(ftask);      // run it as a Runnable
        //   return ftask;        // return it as a Future
    }
}
```

```
FUTUREASK AS THE BRIDGE:

  Callable<V>                    FutureTask<V>             Future<V>
  ───────────                    ─────────────             ────────
  V call()          wrapped by   implements:               
  throws Exception  ──────────►  Runnable    ──────────►   get()
                                   run() {                 isDone()
                                     result=call();        cancel()
                                   }                       isCancelled()

                                 Future
                                   get() → returns result

  FutureTask is BOTH Runnable AND Future.
  It wraps the Callable inside.
  Its run() method calls callable.call() and stores the result.
  Its get() method retrieves the stored result.
  This is how Callable works with any Runnable-accepting API.
```

---

## Part 6 — Real-World Patterns

### Pattern 1 — Parallel Tasks with Results

```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 1: Launch multiple Callables, collect all results
// ═══════════════════════════════════════════════════════════════
public class ParallelCallables {

    static int fetchUserCount(String region)      throws Exception {
        Thread.sleep(500); // simulate DB call
        return switch (region) {
            case "NORTH" -> 1500;
            case "SOUTH" -> 2300;
            case "EAST"  -> 1800;
            case "WEST"  -> 2100;
            default      -> 0;
        };
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        // Submit all 4 tasks at once — run in PARALLEL
        Future<Integer> northFuture = executor.submit(() -> fetchUserCount("NORTH"));
        Future<Integer> southFuture = executor.submit(() -> fetchUserCount("SOUTH"));
        Future<Integer> eastFuture  = executor.submit(() -> fetchUserCount("EAST"));
        Future<Integer> westFuture  = executor.submit(() -> fetchUserCount("WEST"));

        // All 4 tasks are running in parallel NOW
        // Each takes ~500ms
        // Total time: ~500ms (parallel) vs 2000ms (sequential)

        // Collect results — blocks until each is done
        int north = northFuture.get();
        int south = southFuture.get();
        int east  = eastFuture.get();
        int west  = westFuture.get();

        System.out.println("Total users: " + (north + south + east + west));

        executor.shutdown();
    }
}
```

### Pattern 2 — invokeAll — Submit Collection of Callables

```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 2: invokeAll — submit a list of Callables at once
//  Waits for ALL to complete before returning
// ═══════════════════════════════════════════════════════════════
public class InvokeAllPattern {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        // Build a list of tasks
        List<Callable<String>> tasks = new ArrayList<>();
        for (int i = 1; i <= 6; i++) {
            final int taskId = i;
            tasks.add(() -> {
                Thread.sleep(100 * taskId); // each takes different time
                return "Task-" + taskId + " result";
            });
        }

        // invokeAll submits ALL tasks and waits for ALL to complete
        // Returns List<Future<String>> — one per task, in same order
        List<Future<String>> futures = executor.invokeAll(tasks);
        //   ↑ BLOCKS here until every single task is done
        //     All tasks run in parallel internally
        //     Result list is in SAME ORDER as input tasks list

        for (int i = 0; i < futures.size(); i++) {
            System.out.println("Task " + i + ": " + futures.get(i).get());
        }

        // invokeAll with timeout
        List<Future<String>> timedFutures = executor.invokeAll(
                tasks, 300, TimeUnit.MILLISECONDS
        );
        // Tasks NOT done within 300ms are CANCELLED
        // Their future.get() will throw CancellationException
        for (Future<String> f : timedFutures) {
            if (f.isCancelled()) {
                System.out.println("This task timed out");
            } else {
                System.out.println(f.get());
            }
        }

        executor.shutdown();
    }
}
```

### Pattern 3 — invokeAny — First Result Wins

```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 3: invokeAny — return the FIRST successful result
//  All others are cancelled
// ═══════════════════════════════════════════════════════════════
public class InvokeAnyPattern {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // Scenario: call 3 replicated services, use whichever responds first
        List<Callable<String>> services = Arrays.asList(
                () -> { Thread.sleep(300); return "Service-A response"; },
                () -> { Thread.sleep(100); return "Service-B response"; }, // fastest
                () -> { Thread.sleep(500); return "Service-C response"; }
        );

        String fastestResult = executor.invokeAny(services);
        //   ↑ Submits ALL tasks
        //     Waits until ANY ONE completes successfully
        //     Returns that result
        //     Cancels all other running tasks
        //     If ALL tasks fail → ExecutionException thrown

        System.out.println("Fastest response: " + fastestResult);
        // "Service-B response" (took only 100ms)

        // Use case: redundant service calls for reliability/latency
        //   Call primary + 2 backup services
        //   Use whoever responds first
        //   High-availability pattern

        executor.shutdown();
    }
}
```

### Pattern 4 — Wrapping Runnable Tasks in Callable

```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 4: Adapting Runnable to Callable when needed
// ═══════════════════════════════════════════════════════════════
public class RunnableToCallable {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        Runnable task = () -> System.out.println("Original Runnable task");

        // Method 1: Executors.callable(Runnable) → Callable<Object>
        // Returns null when done
        Callable<Object> adapted1 = Executors.callable(task);
        Future<Object>   future1  = executor.submit(adapted1);
        Object result1 = future1.get(); // null — but at least you know it's done

        // Method 2: Executors.callable(Runnable, T result) → Callable<T>
        // Returns specified value when done
        Callable<String> adapted2 = Executors.callable(task, "done");
        Future<String>   future2  = executor.submit(adapted2);
        String result2 = future2.get(); // "done"
        System.out.println("Result: " + result2);

        // Method 3: Wrap manually in lambda
        Callable<Boolean> adapted3 = () -> {
            task.run();       // run the Runnable
            return true;      // indicate success
        };

        executor.shutdown();
    }
}
```

---

## Part 7 — Common Mistakes and Pitfalls

```java
// ═══════════════════════════════════════════════════════════════
//  COMMON MISTAKES — every developer makes these
// ═══════════════════════════════════════════════════════════════
public class CommonMistakes {

    static int counter = 0;

    // ── MISTAKE 1: Calling run() instead of start() ───────────────
    static void mistake1() {
        Thread t = new Thread(() -> {
            System.out.println("Running on: "
                    + Thread.currentThread().getName());
        });

        t.run();
        // ↑ WRONG! run() is just a regular method call.
        //   Executes on the CURRENT thread, not a new thread.
        //   Output: "Running on: main" (not a new thread!)
        //   No new OS thread is created.
        //   No concurrency happens.
        //   This is a very common mistake for beginners.

        t.start();
        // ✓ CORRECT! Creates new thread, calls run() on it.
        //   Output: "Running on: Thread-0" (new thread)
    }

    // ── MISTAKE 2: Starting a thread twice ────────────────────────
    static void mistake2() throws InterruptedException {
        Thread t = new Thread(() -> System.out.println("running"));
        t.start();
        t.join();

        t.start(); // ← IllegalThreadStateException!
        // Thread is in TERMINATED state.
        // Cannot restart a terminated thread.
        // Create a new Thread object instead.
    }

    // ── MISTAKE 3: Not joining and reading result too early ────────
    static void mistake3() {
        int[] result = new int[1];
        Thread t = new Thread(() -> {
            result[0] = 42; // write happens on background thread
        });
        t.start();
        // NO JOIN HERE — reading result immediately
        System.out.println("Result: " + result[0]);
        // May print 0 (race condition — thread may not have run yet)
        // May print 42 (if background thread happened to run first)
        // Non-deterministic!
    }

    static void mistake3_fixed() throws InterruptedException {
        int[] result = new int[1];
        Thread t = new Thread(() -> result[0] = 42);
        t.start();
        t.join(); // ← wait for thread to complete
        System.out.println("Result: " + result[0]); // guaranteed 42
    }

    // ── MISTAKE 4: Forgetting that sleep() holds locks ─────────────
    static final Object lock = new Object();

    static void mistake4() throws InterruptedException {
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                System.out.println("T1: got lock, sleeping");
                try {
                    Thread.sleep(5000); // HOLDS lock while sleeping!
                    // NO OTHER THREAD can acquire 'lock' for 5 seconds!
                    // sleep() does NOT release the lock.
                    // Use wait() if you want to release lock while waiting.
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("T1: done sleeping, releasing lock");
            }
        });

        Thread t2 = new Thread(() -> {
            System.out.println("T2: trying to get lock...");
            synchronized (lock) { // BLOCKED for 5 full seconds!
                System.out.println("T2: finally got lock");
            }
        });

        t1.start();
        Thread.sleep(100); // ensure t1 gets lock first
        t2.start();
        // T2 waits 5 seconds doing nothing — lock held by sleeping T1
    }

    // ── MISTAKE 5: Using Callable result without handling exceptions ─
    static void mistake5() {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> future = executor.submit(() -> {
            if (true) throw new RuntimeException("Task failed!");
            return 42;
        });

        try {
            int result = future.get();         // exception happens here
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            // handle interrupt
        } catch (ExecutionException e) {
            // The actual exception is WRAPPED inside ExecutionException
            Throwable actual = e.getCause();   // unwrap it!
            System.out.println("Actual error: " + actual.getMessage());
            // Many developers forget to unwrap — logs show ExecutionException
            // instead of the real cause
        } finally {
            executor.shutdown();
        }
    }

    // ── MISTAKE 6: execute() silently swallows exceptions ─────────
    static void mistake6() {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        executor.execute(() -> {
            throw new RuntimeException("SILENT FAILURE");
            // This exception goes to UncaughtExceptionHandler
            // Default handler: prints to stderr
            // BUT: your program may not notice it!
            // This task silently failed — no Future to check.
        });

        // BETTER: use submit() and check the Future
        Future<?> future = executor.submit(() -> {
            throw new RuntimeException("VISIBLE FAILURE");
        });
        try {
            future.get(); // exception surfaces here — visible!
        } catch (ExecutionException e) {
            System.out.println("Task failed: " + e.getCause().getMessage());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        executor.shutdown();
    }
}
```

---

## Part 8 — Interview Questions and Answers

```
Q1: What is the difference between Thread and Runnable?
────────────────────────────────────────────────────────
A:  Thread is a class that represents a thread of execution.
    Runnable is an interface that represents a task to be executed.

    Extending Thread: combines task (what) and thread (how) in one class.
      Problem: Java is single inheritance → can't extend anything else.
      Problem: Task is tightly coupled to Thread — can't run on pools.

    Implementing Runnable: separates task from thread.
      Same Runnable can run on raw Thread, thread pool, virtual thread, etc.
      Class can extend something else AND implement Runnable.
      Better design — separation of concerns.

    Prefer Runnable over extending Thread.

────────────────────────────────────────────────────────
Q2: What is the difference between Runnable and Callable?
────────────────────────────────────────────────────────
A:  Three differences:
    1. Return type:
         Runnable.run()    → void    (no result)
         Callable.call()  → V       (returns a result)

    2. Exception handling:
         Runnable.run()   → cannot throw checked exceptions
         Callable.call()  → can throw checked exceptions (throws Exception)

    3. Usage:
         Runnable → Thread, execute(), submit() (returns Future<?>: null)
         Callable → submit() only (returns Future<V>: actual result)

    Use Callable when you need the task's result or
    when the task might throw checked exceptions.

────────────────────────────────────────────────────────
Q3: Can you pass a Callable to new Thread()?
────────────────────────────────────────────────────────
A:  No. Thread constructor accepts only Runnable, not Callable.
    To run a Callable on a raw thread, wrap it in FutureTask:

    Callable<String> callable = () -> "result";
    FutureTask<String> futureTask = new FutureTask<>(callable);
    new Thread(futureTask).start();
    String result = futureTask.get();

    FutureTask implements both Runnable and Future<V>.

────────────────────────────────────────────────────────
Q4: What does start() do vs run()?
────────────────────────────────────────────────────────
A:  start(): Creates a new OS thread. JVM calls run() on that new thread.
             Returns immediately. Non-blocking. Concurrency happens.
    run():   Just a regular method call on the current thread.
             No new thread created. No concurrency. Sequential execution.

    Classic mistake: calling t.run() instead of t.start().
    Everything executes on the current thread — as if no thread was created.

────────────────────────────────────────────────────────
Q5: What happens if you call start() twice on the same Thread?
────────────────────────────────────────────────────────
A:  IllegalThreadStateException is thrown.
    A Thread can only be started once.
    Once started (even after termination), it cannot be restarted.
    Solution: create a new Thread object.
    This is another reason to prefer Runnable/Callable over extending Thread —
    the same Runnable/Callable can be submitted multiple times.

────────────────────────────────────────────────────────
Q6: What does future.get() do if the Callable threw an exception?
────────────────────────────────────────────────────────
A:  future.get() throws ExecutionException.
    The original exception from call() is wrapped as the cause.
    Retrieve it with: e.getCause()
    This is how exceptions cross thread boundaries in Java.

    future.get() can throw:
      ExecutionException    — Callable threw an exception
      InterruptedException  — calling thread was interrupted while waiting
      CancellationException — task was cancelled before completing

────────────────────────────────────────────────────────
Q7: What is FutureTask?
────────────────────────────────────────────────────────
A:  FutureTask<V> is a class that:
      Implements Runnable (so it can be passed to Thread or execute())
      Implements Future<V> (so caller can get() the result)
      Wraps a Callable<V> (runs it in run(), stores result)

    It is the bridge that makes Callable work with any Runnable-accepting API.
    ExecutorService.submit(callable) internally creates a FutureTask.

────────────────────────────────────────────────────────
Q8: What is the difference between execute() and submit()?
────────────────────────────────────────────────────────
A:  execute(Runnable):
      Returns void — no handle to the task.
      Exceptions are silently sent to UncaughtExceptionHandler.
      Fire-and-forget — no way to check completion.

    submit(Runnable/Callable):
      Returns Future<?> or Future<V>.
      Exceptions are wrapped in ExecutionException on get().
      Can check isDone(), cancel(), get().
      Prefer submit() in most cases — better visibility and control.
```

---

## Summary — The Decision Tree

```
WHICH TO USE?

  ┌────────────────────────────────────────────────────────────┐
  │  Do you need a return value from the task?                 │
  └────────────────────────────┬───────────────────────────────┘
                               │
                ┌──────────────▼──────────────┐
                │           YES               │  NO
                │       Use Callable          │
                │       submit() → Future<V>  │
                └─────────────────────────────┘
                                               │
                ┌──────────────────────────────▼──────────────┐
                │  Do you need to catch checked exceptions     │
                │  thrown by the task?                        │
                └──────────────────────────────┬──────────────┘
                                               │
                               ┌───────────────▼─────────────┐
                               │           YES               │  NO
                               │       Use Callable          │
                               │   (call() throws Exception) │
                               └─────────────────────────────┘
                                                              │
                ┌─────────────────────────────────────────────▼──┐
                │  Do you need to extend another class?           │
                └────────────────────────────────┬────────────────┘
                                                 │
                                 ┌───────────────▼─────────────┐
                                 │           YES               │  NO
                                 │       Use Runnable          │
                                 │   (implements interface     │
                                 │    can extend other class)  │
                                 └─────────────────────────────┘
                                                               │
                                 ┌─────────────────────────────▼──┐
                                 │  Simple background task,        │
                                 │  no result, no checked ex?      │
                                 │                                 │
                                 │  Use Runnable — clean,simple    │
                                 │  Or Thread if never using       │
                                 │  a thread pool                  │
                                 └─────────────────────────────────┘

FINAL RULE OF THUMB:

  Raw Thread     → almost never in modern code
                   (except learning/testing)

  Runnable       → fire-and-forget tasks, event listeners,
                   background work with no result needed

  Callable<V>    → tasks that compute and return results,
                   tasks that throw checked exceptions,
                   tasks that need proper error handling
                   (most real-world production tasks)
```