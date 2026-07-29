# Race Condition — Complete Tutorial
---
## The Story Before the Definition

```sql
It is Saturday morning.
You and your roommate share ONE bank account.
Account balance: ₹1000.

You both decide to withdraw ₹700 at the exact same moment —
you at the ATM on Street A,
your roommate at the ATM on Street B.

ATM on Street A (You):          ATM on Street B (Roommate):
  1. Check balance → ₹1000        1. Check balance → ₹1000
  2. ₹1000 >= ₹700? YES           2. ₹1000 >= ₹700? YES
  3. Dispense ₹700                 3. Dispense ₹700
  4. Update balance: 1000-700=300  4. Update balance: 1000-700=300

Both ATMs dispensed ₹700.
Total dispensed: ₹1400.
But account only had ₹1000.
Final balance: ₹300 (should be -₹400 or second withdrawal should have failed).

The bank just LOST ₹400.

This is a Race Condition.
Two operations RACED to read-check-write the same data.
Both read the SAME value (1000).
Both made decisions based on that SAME value.
Both wrote back — one silently overwrote the other.
The second write ERASED the first write.

Nobody did anything "wrong."
The sequence of events just happened to interleave
in a way that produced an incorrect result.
```

---

## Formal Definition

```vbnet
A RACE CONDITION occurs when:
  1. Two or more threads ACCESS shared data concurrently
  2. At least ONE of those accesses is a WRITE
  3. The threads do NOT use synchronization
  4. The final result DEPENDS on the exact timing /
     order of thread execution

  "The program's correctness depends on the RACE between threads —
   whoever gets there first wins — and the result changes based on who wins."

The key insight:
  Race conditions are NON-DETERMINISTIC.
  The same program may give CORRECT results most of the time
  and WRONG results occasionally.
  They are timing-dependent, hard to reproduce,
  and almost impossible to catch with basic testing.
  This makes them among the MOST DANGEROUS bugs in software.
```

---
## The Root Cause — Non-Atomic Operations

```vbnet
Most operations that LOOK like one step in Java
are actually MULTIPLE steps at the CPU level.

The window between those steps is where another thread
can sneak in and corrupt the result.

counter++   looks like ONE operation.
In reality it is THREE:

  Step 1: READ  counter from memory into CPU register
  Step 2: ADD   1 to the value in register
  Step 3: WRITE the new value back to memory

  ┌────────────────────────────────────────────────────────────┐
  │                counter++ = 3 CPU steps                     │
  │                                                            │
  │   Memory: [counter=5]                                      │
  │                  │                                         │
  │   Step 1: READ   └──────────────► CPU Register: 5          │
  │   Step 2: ADD                     CPU Register: 5+1=6      │
  │   Step 3: WRITE  ◄──────────────  CPU Register: 6          │
  │                  │                                         │
  │   Memory: [counter=6]  ✓ (if no interruption)              │
  └────────────────────────────────────────────────────────────┘

  Another thread can run BETWEEN any two of these steps.
  That gap is the race window.
```

---
## The Anatomy of a Race Condition — Step by Step

```sql
SCENARIO:
  counter = 0
  Thread A and Thread B both do counter++ simultaneously
  Expected final value: 2
  Actual final value:   1  (one increment is LOST)

DETAILED INTERLEAVING:
════════════════════════════════════════════════════════════════

  Time   Thread A (CPU Core 1)           Thread B (CPU Core 2)
  ────   ───────────────────────────     ───────────────────────────

   1     READ  counter → gets 0
         (A has 0 in its register)

   2                                     READ  counter → gets 0
                                         (B has 0 in its register)
                                         ← B reads BEFORE A writes back!
                                           Both threads now hold value 0

   3     ADD   0 + 1 = 1
         (A's register now has 1)

   4                                     ADD   0 + 1 = 1
                                         (B's register now has 1)

   5     WRITE 1 → counter = 1
         (A writes 1 to memory)

   6                                     WRITE 1 → counter = 1
                                         (B writes 1 to memory)
                                         ← B OVERWRITES A's write!
                                           A's increment is LOST!

  FINAL: counter = 1   ← WRONG, should be 2
         One increment silently lost.

  ┌───────────────────────────────────────────────────────────┐
  │  THE RACE WINDOW                                          │
  │                                                           │
  │  Thread A: [READ]────────────────────[ADD]────[WRITE]     │
  │                  ↑                                        │
  │                  └── Thread B sneaks in HERE ──┐          │
  │  Thread B:           [READ]────[ADD]────[WRITE]│          │
  │                                                │          │
  │  The gap between A's READ and A's WRITE        │          │
  │  is the RACE WINDOW.                           │          │
  │  If B enters this window → race condition.     │          │
  └───────────────────────────────────────────────────────────┘
```

---
## The Simplest Race Condition — In Code
```java
// ═══════════════════════════════════════════════════════════════
//  THE CLASSIC RACE CONDITION — lost counter increments
// ═══════════════════════════════════════════════════════════════
public class ClassicRaceCondition {

    // Shared mutable state — lives on the Heap
    // ALL threads can read and write this
    private static int counter = 0;

    public static void main(String[] args) throws InterruptedException {

        int iterations = 10_000;

        Runnable incrementTask = () -> {
            for (int i = 0; i < iterations; i++) {
                counter++;
                // ↑ This is NOT atomic.
                //   READ counter
                //   ADD  1
                //   WRITE counter
                //   Three separate operations.
                //   Any other thread can run between these.
            }
        };

        Thread t1 = new Thread(incrementTask, "Thread-1");
        Thread t2 = new Thread(incrementTask, "Thread-2");

        t1.start();
        t2.start();

        t1.join(); // wait for t1 to finish
        t2.join(); // wait for t2 to finish

        System.out.println("Expected : " + (2 * iterations));
        System.out.println("Actual   : " + counter);
        System.out.println("Lost     : " + (2 * iterations - counter));
    }
}
```

**Output (run multiple times — different result each time):**
```
Run 1:
Expected : 20000
Actual   : 14823   ← 5177 increments LOST
Lost     : 5177

Run 2:
Expected : 20000
Actual   : 17392   ← 2608 increments LOST
Lost     : 2608

Run 3:
Expected : 20000
Actual   : 20000   ← appears correct! (got lucky this run)
Lost     : 0

Run 4:
Expected : 20000
Actual   : 11445   ← 8555 increments LOST
Lost     : 8555
```

```
WHY RUN 3 APPEARS CORRECT:
  By chance, threads did not interleave at the critical moment.
  This is the most dangerous aspect of race conditions:
  They may work correctly 99% of the time.
  They fail silently and unpredictably.
  They often only appear under high load in production.
  They are nearly impossible to reproduce on demand.
```

---
## The Three Classic Patterns of Race Conditions

### Pattern 1: Read-Modify-Write
```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 1: Read-Modify-Write
//  Most common race condition pattern.
//  Read a value, compute something, write back.
// ═══════════════════════════════════════════════════════════════
public class ReadModifyWrite {

    private static int balance = 1000; // shared on Heap

    // UNSAFE: read-modify-write without synchronization
    public static void withdraw(int amount) {
        int current = balance;     // READ  ← race window starts
        int newBalance = current - amount; // MODIFY
        balance = newBalance;      // WRITE ← race window ends
        // Three separate steps — interleaving is possible
    }

    // Same pattern in even simpler forms:
    private static int count = 0;

    public static void unsafeIncrement() { count++;     } // read-modify-write
    public static void unsafeDecrement() { count--;     } // read-modify-write
    public static void unsafeAdd(int n)  { count += n;  } // read-modify-write
    public static void unsafeDouble()    { count *= 2;  } // read-modify-write

    // ── VISUALIZE THE RACE WINDOW ─────────────────────────────────
    //
    //  Thread A: withdraw(700)       Thread B: withdraw(700)
    //
    //  READ  balance → 1000          (waiting)
    //  MODIFY: 1000-700 = 300        (waiting)
    //  (Thread A is paused here by OS)
    //                                READ  balance → 1000  ← B reads stale value!
    //                                MODIFY: 1000-700 = 300
    //                                WRITE balance = 300
    //  WRITE balance = 300           (done, B got money)
    //  (A resumes and overwrites B's write)
    //
    //  Final balance: 300            ← should be -400 (or B should have been blocked)
    //  Both got ₹700 from an account with only ₹1000!
}
```
### Pattern 2: Check-Then-Act
```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 2: Check-Then-Act
//  Check a condition, then act on it.
//  Condition may change between the check and the act.
// ═══════════════════════════════════════════════════════════════
public class CheckThenAct {

    // ── EXAMPLE 1: Lazy initialization ────────────────────────────
    private static ExpensiveObject instance = null;

    // UNSAFE: classic broken singleton
    public static ExpensiveObject getInstance() {
        if (instance == null) {              // CHECK  ← race window starts
            instance = new ExpensiveObject();// ACT    ← race window ends
        }
        return instance;
        // Thread A checks: null → true → about to create
        // Thread B checks: null → true → creates ExpensiveObject (B)
        // Thread A creates ExpensiveObject (A)
        // Now TWO instances exist — singleton is broken!
        // If ExpensiveObject has state → two separate states → bugs
    }

    // ── EXAMPLE 2: Queue size check ───────────────────────────────
    private static List<String> queue = new ArrayList<>();

    public static void processFirst() {
        if (!queue.isEmpty()) {              // CHECK
            String item = queue.remove(0);   // ACT
            System.out.println("Processing: " + item);
        }
        // Thread A checks: not empty → true
        // Thread B checks: not empty → true
        // Thread B removes the last item → queue now empty
        // Thread A calls remove(0) → IndexOutOfBoundsException!
        // (or in other cases: processes same item twice)
    }

    // ── EXAMPLE 3: File creation ──────────────────────────────────
    public static void createFileIfNotExists(String path) throws Exception {
        File file = new File(path);
        if (!file.exists()) {          // CHECK
            file.createNewFile();      // ACT
        }
        // Thread A checks: file doesn't exist → true
        // Thread B checks: file doesn't exist → true
        // Thread A creates file
        // Thread B tries to create → may fail or create duplicate
        // The check result was STALE by the time act executed
    }
}
```

```sql
CHECK-THEN-ACT — Why it fails:

  The world can CHANGE between the CHECK and the ACT.
  The check becomes STALE.

  Time ──────────────────────────────────────────────────────►

  Thread A:  [CHECK: null? yes]                    [ACT: create]
                              ↑                    ↑
                              └── Thread B: ───────┘
                              [CHECK: null? yes] [ACT: create]

  Both checked BEFORE either acted.
  Both saw null.
  Both created.
  Two objects now exist.

  The CHECK and ACT must happen as ONE atomic operation.
  That is what synchronized gives you.
```
### Pattern 3: Stale Read (Visibility Race)

```java
// ═══════════════════════════════════════════════════════════════
//  PATTERN 3: Stale Read
//  Thread reads a value that another thread already updated.
//  But the write hasn't been flushed to main memory yet.
// ═══════════════════════════════════════════════════════════════
public class StaleRead {

    // NOT volatile — no visibility guarantee
    private static boolean stopRequested = false;
    private static int     value         = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            // Worker thread CACHES stopRequested in its CPU register
            // It may NEVER see the updated value from main thread
            while (!stopRequested) {
                // keep running...
                // This loop may run FOREVER even after main sets stopRequested=true
                // because worker is reading from its LOCAL CPU cache
                // not from main memory
            }
            System.out.println("Worker: stop signal received");
        });

        Thread updater = new Thread(() -> {
            value = 42;
            stopRequested = true;
            // These writes go to UPDATER's CPU cache first
            // May not reach main memory immediately
            // Worker may never see them
            System.out.println("Updater: sent stop signal");
        });

        worker.start();
        Thread.sleep(100); // let worker start spinning
        updater.start();

        // Possible outcomes:
        // 1. Worker sees stopRequested=true quickly → stops (lucky)
        // 2. Worker NEVER sees stopRequested=true → loops forever (bug)
        // 3. Worker sees stopRequested=true but value is still 0 (reordering bug)
    }
}
```

```sql
STALE READ — MEMORY VISIBILITY PICTURE:

  CPU Core 1 (worker thread)         CPU Core 2 (updater thread)
  ──────────────────────────         ───────────────────────────
  L1 Cache:                          L1 Cache:
    stopRequested = false  ← STALE     stopRequested = true  ← NEW VALUE
    value         = 0      ← STALE     value         = 42    ← NEW VALUE

                            MAIN MEMORY:
                            stopRequested = false  ← not yet flushed!
                            value         = 0      ← not yet flushed!

  Worker reads its L1 cache → sees old values → loops forever
  Updater wrote to its L1 cache → not yet propagated to main memory

  Without volatile or synchronization:
  JMM does NOT guarantee when (or even IF)
  the write will reach other threads.
```

---
## A More Realistic Race Condition — Bank Transfer
```java
// ═══════════════════════════════════════════════════════════════
//  REALISTIC EXAMPLE: Concurrent Bank Transfers
//  Shows race condition in a real-world scenario
// ═══════════════════════════════════════════════════════════════
public class BankRaceCondition {

    static class BankAccount {
        private String owner;
        private double balance;

        BankAccount(String owner, double balance) {
            this.owner   = owner;
            this.balance = balance;
        }

        // UNSAFE withdraw — no synchronization
        public boolean withdraw(double amount) {
            if (balance >= amount) {     // CHECK
                // ← race window: another thread can modify balance here
                balance -= amount;       // ACT (read-modify-write)
                return true;
            }
            return false;
        }

        // UNSAFE deposit
        public void deposit(double amount) {
            balance += amount;           // read-modify-write
        }

        public double getBalance() { return balance; }
    }

    // UNSAFE transfer — no synchronization on either account
    public static void transfer(BankAccount from,
                                BankAccount to,
                                double amount) {
        if (from.withdraw(amount)) {
            // ← race window between withdraw and deposit
            // another thread can read 'to' balance here
            to.deposit(amount);
        }
    }

    public static void main(String[] args) throws InterruptedException {

        BankAccount alice = new BankAccount("Alice", 1000.0);
        BankAccount bob   = new BankAccount("Bob",   1000.0);

        // 100 threads transferring between Alice and Bob simultaneously
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            final int threadId = i;
            threads[i] = new Thread(() -> {
                if (threadId % 2 == 0) {
                    transfer(alice, bob,   10.0); // Alice → Bob
                } else {
                    transfer(bob,   alice, 10.0); // Bob → Alice
                }
            });
        }

        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        double total = alice.getBalance() + bob.getBalance();
        System.out.println("Alice  : " + alice.getBalance());
        System.out.println("Bob    : " + bob.getBalance());
        System.out.println("Total  : " + total);
        System.out.println("Expected total: 2000.0");
        System.out.println("Money lost/created: " + (2000.0 - total));
        // Total should ALWAYS be 2000 (conservation of money)
        // But it won't be — race conditions create or destroy money!
    }
}
```

**Output:**
```yaml
Run 1:
Alice  : 980.0
Bob    : 1010.0
Total  : 1990.0       ← ₹10 VANISHED from the system!
Money lost/created: 10.0

Run 2:
Alice  : 1020.0
Bob    : 990.0
Total  : 2010.0       ← ₹10 CREATED from thin air!
Money lost/created: -10.0

Run 3:
Alice  : 1000.0
Bob    : 1000.0
Total  : 2000.0       ← looks correct (lucky run)
Money lost/created: 0.0
```

```yaml
HOW MONEY VANISHES — thread interleaving trace:

  alice.balance = 1000, bob.balance = 1000

  Thread A: transfer(alice → bob, 10)
  Thread B: transfer(alice → bob, 10)

  Thread A: alice.withdraw(10)
    READ  alice.balance → 1000
    CHECK 1000 >= 10?  YES
    ← Thread B sneaks in HERE

  Thread B: alice.withdraw(10)
    READ  alice.balance → 1000  ← reads SAME stale 1000!
    CHECK 1000 >= 10?  YES
    WRITE alice.balance = 1000-10 = 990

  Thread A resumes:
    WRITE alice.balance = 1000-10 = 990  ← OVERWRITES B's write!
    bob.deposit(10) → bob = 1010

  Thread B: bob.deposit(10) → bob = 1020 ← but alice only lost 10 once!

  Result: alice=990, bob=1020, total=2010
  ₹10 was CREATED from nothing.

  Or the reverse interleaving causes ₹10 to vanish.
```
---
## Race Conditions are Non-Deterministic — The Heisenbug
```java
// ═══════════════════════════════════════════════════════════════
//  THE HEISENBUG — race conditions that DISAPPEAR when observed
// ═══════════════════════════════════════════════════════════════
public class Heisenbug {

    private static int counter = 0;

    public static void main(String[] args) throws InterruptedException {

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
                // Without println: threads interleave freely → race happens
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
                // Without println: race condition regularly occurs
            }
        });

        t1.start(); t2.start();
        t1.join();  t2.join();
        System.out.println("Without debug: " + counter); // 14823 (wrong)

        // NOW ADD DEBUG PRINTLN:
        counter = 0;

        Thread t3 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
                System.out.println("T3: " + counter); // adding this line!
                // println acquires a lock internally (PrintStream is synchronized)
                // This SLOWS DOWN the thread significantly
                // This changes the TIMING — race window rarely aligns now
                // Race condition appears to DISAPPEAR
            }
        });

        Thread t4 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) {
                counter++;
                System.out.println("T4: " + counter); // adding this line!
            }
        });

        t3.start(); t4.start();
        t3.join();  t4.join();
        System.out.println("With debug: " + counter); // 20000 (appears correct!)
        // But the bug is STILL THERE — we just changed the timing
        // Remove the printlns → bug comes back
    }
}
```

```sql
THE HEISENBUG PRINCIPLE:

  Race conditions are timing-dependent.
  Anything that changes timing can make them
  appear or disappear:

  Makes race condition LESS likely to appear:
    Adding print statements (slow down threads)
    Running in debug mode (stepping through code)
    Adding Thread.sleep() for debugging
    Running on slow hardware / single core
    Lower load / fewer concurrent users

  Makes race condition MORE likely to appear:
    High load (many concurrent threads)
    Fast multi-core hardware
    Removing debug statements
    Production environment
    Stress testing

  The bug EXISTS in all cases.
  Only the probability of observing it changes.
  This is why race conditions are so dangerous —
  they hide in development and strike in production.
```
---
## How to DETECT a Race Condition

```vbnet
DETECTION METHODS (in order of usefulness):

1. CODE REVIEW — look for these patterns:
   ─────────────────────────────────────────
   ✗ Shared mutable field accessed without synchronization
   ✗ Multiple operations on shared state not in synchronized block
   ✗ check-then-act without synchronization
   ✗ read-modify-write (++, --, +=) on shared fields
   ✗ Multiple fields that must be consistent (balance + timestamp)
     updated separately without synchronization

2. THREAD SANITIZER / RACE DETECTORS:
   ─────────────────────────────────────────
   Java Thread Sanitizer tools:
     - RacerD (Facebook/Meta's static analyzer)
     - FindBugs / SpotBugs (looks for concurrency antipatterns)
     - ThreadSafe (commercial static analyzer)
     - jcstress (JVM Concurrency Stress Tests — Oracle)

3. STRESS TESTING:
   ─────────────────────────────────────────
   Run the suspicious code with:
     - Many threads (100, 1000, 10000)
     - Many iterations (millions)
     - Multiple CPU cores
   Race conditions become MUCH more likely to manifest

4. THREAD DUMPS (jstack):
   ─────────────────────────────────────────
   jstack <pid>  → shows all thread states
   Look for: multiple threads in same method on same object
   Deadlocks are shown explicitly

5. JVM FLAGS for extra detection:
   ─────────────────────────────────────────
   -XX:+PrintConcurrentLocks   (print lock info)
   -ea (enable assertions)
   Run with HeapDump on exceptions

6. TESTING WITH CONTROLLED TIMING (advanced):
   ─────────────────────────────────────────
   Insert deliberate Thread.sleep() or Thread.yield()
   at suspected race window to force interleaving
   (Only in test code — never production)
```

```java
// ═══════════════════════════════════════════════════════════════
//  DETECTING RACE CONDITIONS — stress test example
// ═══════════════════════════════════════════════════════════════
public class RaceConditionDetector {

    private static int counter = 0;

    // Method under test — suspected race condition
    public static void increment() {
        counter++;
    }

    // Stress test — exposes the race condition
    public static void main(String[] args) throws InterruptedException {

        int TRIALS     = 100;   // run the test 100 times
        int THREADS    = 50;    // 50 concurrent threads per trial
        int ITERATIONS = 1000;  // each thread increments 1000 times
        int EXPECTED   = THREADS * ITERATIONS; // 50,000

        int racesDetected = 0;

        for (int trial = 0; trial < TRIALS; trial++) {
            counter = 0; // reset

            Thread[] threads = new Thread[THREADS];
            for (int i = 0; i < THREADS; i++) {
                threads[i] = new Thread(() -> {
                    for (int j = 0; j < ITERATIONS; j++) {
                        increment();
                    }
                });
            }

            for (Thread t : threads) t.start();
            for (Thread t : threads) t.join();

            if (counter != EXPECTED) {
                racesDetected++;
                System.out.printf("Trial %3d: Expected=%d  Got=%d  Lost=%d%n",
                        trial, EXPECTED, counter, EXPECTED - counter);
            }
        }

        System.out.printf("%nRace detected in %d / %d trials (%.0f%%)%n",
                racesDetected, TRIALS,
                (racesDetected * 100.0 / TRIALS));
    }
}
```

```yaml
Sample output:
Trial   2: Expected=50000  Got=47823  Lost=2177
Trial   5: Expected=50000  Got=49102  Lost=898
Trial   8: Expected=50000  Got=48756  Lost=1244
Trial  11: Expected=50000  Got=46391  Lost=3609
...

Race detected in 73 / 100 trials (73%)
```
---
## Types of Race Conditions — Named and Classified
```
╔══════════════════════════════════════════════════════════════════════╗
║           TYPES OF RACE CONDITIONS                                   ║
╠══════════════╦═══════════════════════════════════════════════════════╣
║ TYPE         ║ DESCRIPTION + EXAMPLE                                 ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Read-Modify  ║ Read value, compute, write back.                      ║
║ -Write       ║ Another thread reads same value before first writes.  ║
║              ║ counter++, balance -= amount                          ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Check-Then   ║ Check condition, then act on it.                      ║
║ -Act         ║ Condition changes between check and act.              ║
║              ║ if(null) create, if(!empty) remove                    ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Stale Read   ║ Thread reads a variable that another thread wrote     ║
║ (Visibility) ║ but the write is not yet visible (CPU cache issue).   ║
║              ║ while(!stop) { } — stop may never be seen             ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Write-Write  ║ Two threads write to same variable.                   ║
║              ║ One write overwrites the other silently.              ║
║              ║ config = new Config() from two threads                ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ TOCTOU       ║ Time-Of-Check to Time-Of-Use.                         ║
║              ║ Security variant of check-then-act.                   ║
║              ║ Check file permission → use file (OS-level race)      ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Lost Update  ║ Two threads read same value, both compute update,     ║
║              ║ both write. Second write erases first.                ║
║              ║ Both ATMs approve withdrawal of ₹700 from ₹1000       ║
╠══════════════╬═══════════════════════════════════════════════════════╣
║ Dirty Read   ║ Thread reads partially updated state.                 ║
║              ║ Object being updated by another thread                ║
║              ║ halfway through — reader sees inconsistent state.     ║
╚══════════════╩═══════════════════════════════════════════════════════╝
```
---
## The Dirty Read — Seeing Partially Updated State
```java
// ═══════════════════════════════════════════════════════════════
//  DIRTY READ — reading partially updated state
// ═══════════════════════════════════════════════════════════════
public class DirtyRead {

    // A user profile — two fields that must be CONSISTENT with each other
    static class UserProfile {
        String firstName = "John";
        String lastName  = "Smith";
        // Invariant: firstName and lastName belong to the SAME person
        // If they get out of sync → dirty read

        // UNSAFE update — two separate writes
        void update(String first, String last) {
            firstName = first;   // WRITE 1
            // ← another thread can read HERE
            //   it sees new firstName + OLD lastName = inconsistent state!
            lastName  = last;    // WRITE 2
        }

        String getFullName() {
            return firstName + " " + lastName;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        UserProfile profile = new UserProfile();

        // Writer thread: updates profile from John Smith to Alice Johnson
        Thread writer = new Thread(() -> {
            while (true) {
                profile.update("Alice", "Johnson");
                profile.update("John",  "Smith");
            }
        });

        // Reader thread: reads full name
        Thread reader = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                String name = profile.getFullName();
                // Valid names: "John Smith" or "Alice Johnson"
                // INVALID (dirty read):
                //   "Alice Smith"   ← new first + old last
                //   "John Johnson"  ← old first + new last
                if (!name.equals("John Smith") && !name.equals("Alice Johnson")) {
                    System.out.println("DIRTY READ: " + name);
                    // This WILL print — inconsistent state observed
                }
            }
        });

        writer.start();
        reader.start();
        Thread.sleep(1000);
        writer.interrupt();
        reader.interrupt();
    }
}
```

```vbnet
DIRTY READ — timeline:

  UserProfile state: firstName="John", lastName="Smith"

  Writer: firstName = "Alice"     ← WRITE 1 done
          (about to do WRITE 2)
                                  ← Reader runs HERE
                                  Reader: firstName → "Alice"   ← new
                                  Reader: lastName  → "Smith"   ← old
                                  Reader sees: "Alice Smith" ← INCONSISTENT!
  Writer: lastName  = "Johnson"   ← WRITE 2 done

  The reader saw the object in a PARTIALLY UPDATED state.
  firstName was updated but lastName was not yet.
  The invariant (firstName and lastName belong together) was violated.
  This is a dirty read.
```

---
## The Four Necessary Conditions for a Race Condition
```sql
For a race condition to be POSSIBLE, ALL four must be true:

╔════╦═══════════════════════════╦══════════════════════════════════════╗
║ #  ║  CONDITION                ║  EXAMPLE                             ║
╠════╬═══════════════════════════╬══════════════════════════════════════╣
║ 1  ║  SHARED DATA              ║  static field, instance field of     ║
║    ║  Data accessible from     ║  shared object, array element        ║
║    ║  multiple threads         ║                                      ║
╠════╬═══════════════════════════╬══════════════════════════════════════╣
║ 2  ║  MUTABLE DATA             ║  non-final field that gets written   ║
║    ║  Data can be modified     ║  to after initialization             ║
╠════╬═══════════════════════════╬══════════════════════════════════════╣
║ 3  ║  CONCURRENT ACCESS        ║  two or more threads accessing       ║
║    ║  Multiple threads access  ║  the data at overlapping times       ║
║    ║  simultaneously           ║                                      ║
╠════╬═══════════════════════════╬══════════════════════════════════════╣
║ 4  ║  NO SYNCHRONIZATION       ║  no synchronized, no volatile,       ║
║    ║  No coordination          ║  no atomic, no locks                 ║
║    ║  mechanism in place       ║                                      ║
╚════╩═══════════════════════════╩══════════════════════════════════════╝

REMOVE ANY ONE CONDITION → race condition is IMPOSSIBLE:

  Remove 1 (make data non-shared):
    Use ThreadLocal — each thread has its own copy
    Use local variables instead of fields
    → No sharing → no race

  Remove 2 (make data immutable):
    Use final fields
    Use immutable objects (String, records, Integer, etc.)
    → Nobody can modify it → no race

  Remove 3 (make access sequential):
    Use a single thread for all access to this data
    → Impossible to run concurrently → no race

  Remove 4 (add synchronization):
    Use synchronized, volatile, AtomicInteger, ReentrantLock
    → Accesses are coordinated → no race

In practice, you usually fix option 4 (synchronization)
or redesign toward option 1 (eliminate sharing)
or option 2 (immutability).
```

---

## Race Conditions in Real Spring Boot Applications

```java
// ═══════════════════════════════════════════════════════════════
//  RACE CONDITIONS IN SPRING BOOT
//  Spring beans are singletons — shared by ALL request threads
// ═══════════════════════════════════════════════════════════════

// ── EXAMPLE 1: Race condition in Spring Service ────────────────
@Service
public class OrderService {

    // DANGER: instance field in a singleton Spring bean
    // Tomcat creates 200 request-handling threads
    // ALL 200 threads share this SAME OrderService instance
    // ALL 200 threads can read/write requestCount simultaneously
    private int requestCount = 0;   // ← RACE CONDITION

    public Order processOrder(OrderRequest request) {
        requestCount++;  // ← NOT thread-safe!
        // Read-modify-write on shared field
        // 200 threads doing this → lost increments
        return createOrder(request);
    }
}

// ── EXAMPLE 2: Stateful field used across methods ──────────────
@Service
public class ReportService {

    // DANGER: mutable state shared across method calls
    private List<String> reportBuffer = new ArrayList<>(); // NOT thread-safe!

    public void addToReport(String line) {
        reportBuffer.add(line);  // ArrayList.add() is NOT thread-safe
        // Multiple threads calling add() simultaneously
        // → ArrayList internal array can get corrupted
        // → ArrayIndexOutOfBoundsException
        // → duplicate entries
        // → lost entries
    }

    public String generateReport() {
        String report = String.join("\n", reportBuffer);
        reportBuffer.clear(); // ← race: another thread adding while clearing!
        return report;
    }
}

// ── EXAMPLE 3: Cache without synchronization ──────────────────
@Service
public class PriceService {

    // Cache: productId → price
    private Map<String, Double> cache = new HashMap<>(); // NOT thread-safe!

    public double getPrice(String productId) {
        if (!cache.containsKey(productId)) {     // CHECK
            double price = fetchFromDatabase(productId);
            cache.put(productId, price);         // ACT
            // HashMap.put() during concurrent reads → CRASH
            // (HashMap can enter infinite loop or throw exception
            //  under concurrent modification)
        }
        return cache.get(productId);
    }
}
```

```less
WHY SPRING BOOT IS A RACE CONDITION HOTSPOT:

  Spring Boot uses Tomcat by default.
  Tomcat maintains a thread pool (default: 200 threads).
  Each incoming HTTP request runs on one of these 200 threads.
  All 200 threads share the same Spring bean instances
  (singleton scope = one instance for the entire application).

  ANY mutable field in a singleton Spring bean
  is accessed concurrently by up to 200 threads.
  This is a race condition waiting to happen.

  SAFE PATTERNS in Spring Boot:
    1. Stateless beans (no instance fields that change)
       → Most common and recommended approach
    2. Immutable fields (set once in constructor, never changed)
    3. Thread-safe types (AtomicInteger, ConcurrentHashMap)
    4. Synchronized access to mutable fields
    5. Request-scoped beans (@Scope("request"))
       → New bean instance per request → no sharing
```

---

## How to FIX Race Conditions (Overview — Detailed in Later Weeks)

```java
// ═══════════════════════════════════════════════════════════════
//  FIXING RACE CONDITIONS — four approaches (preview)
// ═══════════════════════════════════════════════════════════════
public class RaceConditionFixes {

    // ── THE PROBLEM ───────────────────────────────────────────────
    private static int counter = 0; // ← race condition

    // ── FIX 1: synchronized ───────────────────────────────────────
    // Simplest fix — mutual exclusion
    // Only one thread can execute this method at a time
    public static synchronized void fix1_increment() {
        counter++; // now atomic — thread must hold lock to execute
    }
    // Downside: only one thread at a time → reduces parallelism

    // ── FIX 2: AtomicInteger ──────────────────────────────────────
    // Lock-free — uses CPU-level CAS (Compare-And-Swap)
    // Faster than synchronized for single variable operations
    private static AtomicInteger atomicCounter = new AtomicInteger(0);

    public static void fix2_increment() {
        atomicCounter.incrementAndGet(); // single atomic CPU instruction
    }
    // Best for: counters, flags, single-variable operations

    // ── FIX 3: volatile (for visibility-only races) ───────────────
    // Fixes stale reads but NOT read-modify-write races
    private static volatile boolean stopFlag = false; // visible to all threads

    public static void fix3_stop() {
        stopFlag = true; // visible to all threads immediately
    }
    // Only use when: one thread writes, others read
    // Does NOT fix counter++ type races

    // ── FIX 4: Immutability (eliminate mutability) ─────────────────
    // Best fix: if data never changes, no synchronization needed
    static final class ImmutableConfig {
        final String  host;   // final — set once, never changes
        final int     port;   // final — set once, never changes

        ImmutableConfig(String host, int port) {
            this.host = host;
            this.port = port;
        }
        // No setters. No mutable fields.
        // 100% thread-safe — no synchronization needed.
    }
}
```

---

## The Mental Checklist — Spotting Race Conditions in Code

```
WHEN READING CODE — ask these questions:

  ┌─────────────────────────────────────────────────────────────┐
  │  1. Is this a SHARED field?                                 │
  │     (instance field of object used by multiple threads,     │
  │      static field, parameter passed to multiple threads)    │
  │      NO → safe. YES → continue...                          │
  ├─────────────────────────────────────────────────────────────┤
  │  2. Is it MUTABLE?                                          │
  │     (is it written to anywhere after construction?)         │
  │     NO (final, or effectively final) → safe. YES → continue │
  ├─────────────────────────────────────────────────────────────┤
  │  3. Is it accessed from MULTIPLE THREADS?                   │
  │     (multiple threads could call this code path)            │
  │     NO (single thread) → safe. YES → continue...           │
  ├─────────────────────────────────────────────────────────────┤
  │  4. Is the access SYNCHRONIZED?                             │
  │     (synchronized, volatile, atomic, lock, concurrent coll) │
  │     YES → safe. NO → RACE CONDITION POSSIBLE ⚠             │
  └─────────────────────────────────────────────────────────────┘

QUICK CODE SMELL LIST — these are almost always race conditions
if found in multi-threaded code without synchronization:

  ✗  field++   or   field--   or   field += x   (read-modify-write)
  ✗  if (field == null) { field = new X(); }    (check-then-act)
  ✗  if (!collection.isEmpty()) { collection.remove(0); }
  ✗  field = computeSomething(field)            (read-modify-write)
  ✗  Using HashMap, ArrayList, LinkedList as shared mutable state
  ✗  Multiple related fields updated separately (must be consistent)
  ✗  static mutable fields in singleton beans
  ✗  Instance fields in @Service, @Component, @Repository beans
     that change after construction
```

---

## Summary — Everything in One Picture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    RACE CONDITION — COMPLETE PICTURE                 ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  WHAT IS IT:                                                         ║
║    Two+ threads access shared mutable data concurrently              ║
║    without synchronization, producing incorrect results              ║
║    that depend on thread scheduling order.                           ║
║                                                                      ║
║  WHY IT HAPPENS:                                                     ║
║    Operations that look atomic (counter++) are actually              ║
║    multiple CPU steps with a race window between them.               ║
║    Any thread can run in that window.                                ║
║                                                                      ║
║  THREE ROOT CAUSES:                                                  ║
║    1. Visibility   — one thread's write not seen by other            ║
║    2. Atomicity    — multi-step operation interrupted                ║
║    3. Ordering     — instructions reordered by CPU/compiler          ║
║                                                                      ║
║  THREE CLASSIC PATTERNS:                                             ║
║    1. Read-Modify-Write  → counter++                                 ║
║    2. Check-Then-Act     → if(null) create                           ║
║    3. Stale Read         → while(!stop) {}                           ║
║                                                                      ║
║  WHY DANGEROUS:                                                      ║
║    Non-deterministic — appears/disappears based on timing            ║
║    Hard to reproduce — may work 99.9% of the time                   ║
║    Heisenbug — disappears when you add debug code                    ║
║    Silent corruption — no exception, just wrong results              ║
║                                                                      ║
║  HOW TO FIX:                                                         ║
║    synchronized   → mutual exclusion (one thread at a time)          ║
║    volatile       → visibility only (no atomicity)                   ║
║    AtomicInteger  → lock-free atomic operations                      ║
║    Immutability   → eliminate mutability                             ║
║    ThreadLocal    → eliminate sharing                                ║
║                                                                      ║
║  FOUR CONDITIONS THAT MUST ALL BE TRUE:                              ║
║    Shared + Mutable + Concurrent + Unsynchronized                    ║
║    Remove ANY ONE → race condition impossible                        ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝

THE ONE SENTENCE TO REMEMBER:

  "A race condition is what happens when multiple threads
   share mutable data and at least one writes to it —
   without agreeing on who goes first."
```