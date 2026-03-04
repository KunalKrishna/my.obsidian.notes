Bible for Concurrency in Java = Rule Book
![[Java Concurrency In Practice.png]]

This book presents a simplified set of *rules* for writing concurrent programs. 
write concurrent prog *without* understanding low-level details of the Java Memory Model. 
practical design rules to create safe and performant concurrent classes.

# Fundamentals. Part I (Chapters 2-5) 
focuses on the basic concepts of concurrency and thread safety, and how to compose thread-safe classes out of the concurrent building blocks provided by the class library


| Theme                               | Part                      | Focus                                                                                                                                                       |
| ----------------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fundamentals                        | Part I (Chapters 2-5)     | basic concepts of concurrency and thread safety, and how to compose thread-safe classes out of the concurrent building blocks provided by the class library |
| Structuring Concurrent Application  | Part II (Ch 6-9)          | exploit threads to improve the throughput (responsiveness) ; task-execution framework                                                                       |
| Liveness, Performance, and Testing. | Part III (Chapters 10-12) |                                                                                                                                                             |

## Chapter 1. Introduction
----
**Concurrency** is about _structure_ (dealing with many things at once), while **Asynchrony** is about _flow_ (not waiting for one thing to finish before starting another).

**Threads** are sometimes called *lightweight processes*, and most modern operating systems treat threads, not processes, as the *basic units of scheduling*.

Since **threads share the memory address space** of their owning process, all threads within a process have access to the same variables and allocate objects from the same heap, which allows finer-grained data sharing than inter-process mechanisms. But without explicit synchronization to coordinate access to shared data, a thread may modify variables that another thread is in the middle of using, with unpredictable results.

Single-threaded programs require no synchronization, because no data is shared across threads.

**Benefits of threads**
Threads make it easier to model how humans work and interact, by turning asynchronous workflows into mostly sequential ones.
- Exploiting multiple processors
- Simplicity of modelling
- Simplified handling of asynchronous events
- More responsive user interfaces

|**Feature**|**Synchronous I/O**|**Non-blocking I/O (NIO)**|
|---|---|---|
|**Code Style**|Straight-line, easy to read.|Event-driven, state machines.|
|**Threads**|Many (One per client).|Few (One thread for many clients).|
|**Blocking**|Yes (Thread stops at `read()`).|No (Thread keeps moving).|
|**Scalability**|Harder for 100,000+ connections (Thread overhead).|Excellent for massive connections.|
|**Analogy**|**A Bank:** Each customer has their own dedicated teller who stays with them until the transaction is done.|**A Fast Food Window:** One person takes orders, gives you a buzzer, and moves to the next person. You only come back when the buzzer rings.|

**Risks of threads**
- **Safety Hazards** "nothing bad ever happens."
	- absence of sufficient synchronization, 
	- ordering of operations in multiple threads is unpredictable
- **Liveness hazards** :  "something good eventually happens."
	- bugs that cause liveness failures can be elusive because they depend on the relative timing of events in different threads, and therefore do not always manifest themselves in development or testing.
- **Performance hazards** : "something good happens _quickly_."
	- Performance issues subsume a broad range of problems, including poor service time, responsiveness, throughput, resource consumption, or scalability

All these these issue exist in single-threaded prog, but multi-threaded prog have additional issues that are introduced by the use of threads. 

Threads share the same memory address space and run concurrently, they can access or modify variables that other threads might be using.

For a multithreaded program’s behavior to be predictable, access to shared variables must be properly coordinated so that threads do not interfere with one another.




| Single-threaded prog                             | Multi-threaded prog                                                                                                      |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Safety                                           | Safety & Correctness                                                                                                     |
| must take care to preserve safety and correctnes | use of threads introduces *additional* safety hazards                                                                    |
| Liveness failure exists                          | use of threads introduces additional forms of liveness failure                                                           |
| e.g. an inadvertent infinite loop                | e.g. if thread A is waiting for a resource that thread B holds exclusively, and B never releases it, A will wait forever |
|                                                  | deadlock , starvation , and livelock                                                                                     |

| Safety                     | Liveliness                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------- |
| “nothing bad ever happens” | “something good eventually happens”                                                            |
|                            | when an activity gets into a state such that it is permanently unable to make forward progress |
|                            | Sequential Prog: an inadvertent infinite loop                                                  |
##### Why are concurrency bugs elusive (hard to detect) ? 
because they depend on the relative timing of events in different threads, and therefore do not always manifest themselves in development or testing.
### Three Risks of Thread : Safety, Liveness, Performance
#### 1. Safety (Correctness)
**Safety** means "nothing bad ever happens." A safe program produces the correct result even under heavy load.
- **Single-Threaded Case:** Usually caused by **logic errors**.
    - _Example:_ An `if` condition that doesn't account for a null value. The program crashes or gives a wrong answer because the code itself is flawed.
- **Multi-Threaded Case:** Caused by **Race Conditions**.
    - _Example:_ Two threads try to increment a shared `counter++` at the exact same time.
    - _The "Why":_ Incrementing is actually three steps (Read -> Modify -> Write). If Thread A reads `5`, then Thread B reads `5`, they both write `6`. One increment is "lost."
#### 2. Liveness

**Liveness** means "something good eventually happens." A program with a liveness failure is "stuck"—it’s not crashing, but it’s not doing work either.

- **Single-Threaded Case:** Usually caused by an **Infinite Loop**.
    
    - _Example:_ `while(true) { ... }` where the exit condition is never met. The thread is "alive" but the application is effectively dead to the user.
        
- **Multi-Threaded Case:** Caused by **Deadlock** or **Starvation**.
    
    - _Example (Deadlock):_ Thread A holds Lock 1 and waits for Lock 2. Thread B holds Lock 2 and waits for Lock 1. They will sit there until the end of time.
        
    - _Example (Starvation):_ A low-priority thread never gets CPU time because high-priority threads are constantly hogging the "dance floor."
        

---

#### 3. Performance
**Performance** means "something good happens _quickly_." This is about throughput, responsiveness, and resource usage.
- **Single-Threaded Case:** Usually caused by **Inefficient Algorithms**.
    - _Example:_ Using $O(n^2)$ bubble sort on a million items when you should have used $O(n \log n)$ quicksort. The CPU is working hard, but it's taking too long.
- **Multi-Threaded Case:** Caused by **Context Switching** and **Lock Contention**.
    - _Example (Context Switching):_ If you create 10,000 threads on a 4-core CPU, the CPU spends more time "swapping" threads in and out of memory than actually running your code.
    - _Example (Contention):_ If every thread is fighting for the same `synchronized` block, they spend all their time **BLOCKED** instead of running in parallel.

##### Summary Table

|**Risk**|**Single-Threaded Root Cause**|**Multi-Threaded "Extra" Risk**|
|---|---|---|
|**Safety**|Bad Logic / Bugs|**Race Conditions** (Data corruption)|
|**Liveness**|Infinite Loops|**Deadlocks** (Threads waiting on each other)|
|**Performance**|Poor Algorithms ($O(n^2)$)|**Overhead** (Context switching & Lock fighting)|

> **Pro-Tip for Java Dev Interviews:** When asked about performance, mention that "Adding more threads doesn't always make a program faster." Because of the **Coherency Traffic** (the cost of keeping CPU caches in sync), there is a point of diminishing returns.


### Invariant : Guardian of the Truth🛡️⚔️
In programming, an **Invariant** is simply a condition or a relationship that must **always be true** for the program to be considered "correct." If an invariant is broken, even for a millisecond, the program is in a "corrupt" state.
#### 1. What is an Invariant? (The Definition)
The word comes from "In-varying." It is a logical rule about your data that **never changes**.
- **Sequential Logic:** In a single thread, you might break an invariant temporarily while updating data, but you fix it before anyone else looks.
- **Concurrent Logic (The Danger):** In multi-threading, another thread might "peek" at your data exactly when you've broken the rule but haven't fixed it yet.
#### 2. Example 1: The "Factorizer" (From the Book)
Goetz uses the example of a service that caches the last number it factored.
- **Variable A:** `lastNumber` (e.g., `10`)
- **Variable B:** `lastFactors` (e.g., `2, 5`)
- **The Invariant:** `lastNumber` **MUST ALWAYS EQUAL** the product of `lastFactors`.

**The Failure Scenario:**
1. Thread A starts updating the cache for the number `12`.
2. Thread A updates `lastNumber` to `12`. (**Invariant is now BROKEN** because the factors are still `2, 5`).
3. **Unlucky Timing:** Thread B arrives and reads the cache. It sees `12` and `2, 5`. It returns a wrong result to the user.
4. Thread A finally updates `lastFactors` to `2, 2, 3`. (**Invariant is FIXED**).

Because Thread B saw the "broken" state, the class is **not thread-safe**.
#### 3. Example 2: The "Bank Transfer"
This is the most intuitive way to understand invariants in your Java/Spring Boot career.
- **Variables:** `AccountA_Balance` and `AccountB_Balance`.
- **The Invariant:** The **Total Sum** of both accounts must remain constant during a transfer.

**The Atomic Fix:**
To keep the invariant safe, you must wrap both updates in the **same lock**:

```Java
synchronized(this) {
    accountA.withdraw(100);
    accountB.deposit(100);
}
// Outside this block, the sum is always correct.
// Inside this block, the invariant is briefly broken, 
// but no other thread can "peek" because of the lock.
```
#### 4. Why Goetz emphasizes "The Same Lock" (P2)
In Paragraph 2, Goetz warns: _"each variable participating in the invariant must be guarded by the same lock."_
If you guarded `AccountA` with `Lock_A` and `AccountB` with `Lock_B`:
1. Thread 1 grabs `Lock_A` and withdraws money.
2. Thread 2 (an auditor) grabs `Lock_B` and reads the balance.
3. Since Thread 2 doesn't need `Lock_A`, it can read the balances while the money is "in flight" (missing from A but not yet in B). **The Invariant is violated.**
##### Summary Table: The Invariant Rules

| **Concept**          | **Meaning**                                                                           |
| -------------------- | ------------------------------------------------------------------------------------- |
| **The Invariant**    | The "Golden Rule" of your data (e.g., $A + B = Total$).                               |
| **Interleaving**     | When Thread B sneaks in while Thread A is halfway through a change.                   |
| **Atomic Operation** | Grouping multiple updates so they appear as one single "jump" to the outside world.   |
| **The Same Lock**    | The "curtain" that prevents anyone from seeing the data until the Invariant is fixed. |

---
### Invariant vs Postcondition
*are terms "invariants" and "postconditions" refer to the same thing? what are they?*

In short: **No, they are not the same.** While both are "rules" for your code, they differ in **when** they must be true.
Think of it this way: An **Invariant** is a permanent health check for the object, while a **Postcondition** is a specific promise made by a single method.
#### 1. Invariants (The "Always True" Rule)

A **Class Invariant** is a condition that defines a "valid state" for an object. It must be true for the entire lifetime of the object, specifically at every "stable" point (after the constructor finishes and before/after any public method runs).
- **When it's checked:** Before a method starts and after it finishes.
- **Analogy:** A "Speedometer" invariant: _Current speed can never be negative._

```java
public class BankAccount {
        private int balance;
        // INVARIANT: balance >= 0 (Overdraft not allowed)
    
        public synchronized void withdraw(int amount) {
            // Method must ensure balance doesn't drop below 0 to preserve invariant
            if (balance >= amount) balance -= amount;
        }
    }
```
#### 2. Postconditions (The "Method Promise")
A **Postcondition** is a guarantee provided by a specific method or operation. It describes the state of the system **only at the moment the method finishes**. It doesn't have to be true before the method starts.
- **When it's checked:** Only at the exit point of a specific method.
- **Analogy:** A "Deposit" postcondition: _After calling deposit(50), the balance must be exactly 50 units higher than it was before._
- **Java Example:**
```Java
   /**
     * @postcondition The returned value is the square of the input.
     */
    public int square(int n) {
        return n * n; 
    }
```
#### 3. Key Differences at a Glance

|Feature|Invariant|Postcondition|
|---|---|---|
|**Scope**|Entire Class / Object.|Single Method / Function.|
|**Timeline**|Always (at stable points).|Only after execution.|
|**Purpose**|To define "Internal Health."|To define a "Result/Effect."|
|**Relationship**|Methods must **preserve** it.|Methods must **achieve** it.|
#### Why Goetz discusses them together
In _Java Concurrency in Practice_, Goetz links them because **Compound Actions** (like `check-then-act`) are usually intended to protect an **Invariant**. If your code isn't thread-safe, a second thread can "interleave" and see the object while the invariant is briefly broken, leading to a violation of the **Postcondition** the user expected.

## Chapter 2. Thread Safety 🧵🛡️🦺
----
What is Thread-Safety ?
Atomicity
- Race Condition
- Compound actions 
Locking (mechanism for enforcing Atomicity)
- Intrinsic Locks
- Reentrancy
Guarding state with locks
Liveness and performance

Mechanism : Means to an End : 
	building bridges requires the correct use of a lot of rivets and I-beams.
	building concurrent programs require the correct use of **threads** 🧵and **locks** 🔒 

Writing thread-safe code is, at its core, about managing access to **state**, and in particular to ***shared***, ***mutable state***


| Means (Mechanism)                             | End                          |
| --------------------------------------------- | ---------------------------- |
| correct use of a lot of rivets and I-beams    | building bridges             |
| correct use of **threads** 🧵and **locks** 🔒 | building concurrent programs |
|                                               |                              |
| synchronized keyword                          |                              |

State? 
An object’s state encompasses any data that can affect its externally visible behavior.
an object’s state is its data, stored in state variables such as instance or static fields.

**Thread-safety** 
- is about code. 
- But actually we trying to protect *data* from uncontrolled concurrent access.
- This is a property of how the object is *used* in a program, not what it *does*.
- Thread safety may be a term that is applied to *code*, but it is about *state*, and it can only be applied to the entire body of code that encapsulates its state, which may be an object or an entire program.
- To ensure thread safety, check-then-act operations (like lazy initialization) and read-modify-write operations (like increment) must always be **atomic**.

compound action(check-then-act / read-modify-write) --> atomic operation (atomic var , locking)

**Thread-safety** : DEFINITION 
- **Correctness** : Correctness means that a class conforms to its specification. A good specification defines invariants constraining an object’s state and postconditions describing the effects of its operations.
- a class is thread-safe when it continues to behave correctly when accessed from multiple threads.
- A class is thread-safe if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization or other coordination on the part of the calling code.
- Stateless objects are always thread-safe.
- The definition of thread safety requires that invariants be preserved regardless of timing or interleaving of operations in multiple threads.


Operations A and B are **atomic** with respect to each other if, from the perspective of a thread executing A, when another thread executes B, either all of B has executed or none of it has. An atomic operation is one that is atomic with respect to all operations, including itself, that operate on the same state. 

**Intrinsic locks** : 
mechanism for enforcing Atomicity -> `synchronized` block has two parts : 
1. a reference object that will serve as the *lock* 
2. a block of code to be guarded by that lock  
A synchronized method is a shorthand for a synchronized block that spans an entire method body, and whose lock is the object on which the method is being invoked

```java
synchronized (lock) {
	// Access or modify shared state guarded by lock
}
```

**intrinsic locks/monitor locks** : 
- Every Java object can implicitly act as a lock for purposes of synchronization; these built-in locks are called intrinsic locks or monitor locks.  
- Intrinsic locks in Java act as ***mutexes*** (or mutual exclusion locks), which means that at most one thread may own the lock.   

**Reentrancy** =   locks are acquired on a per-thread rather than per-invocation basis
- implemented by associating with each lock an **acquisition count** and an **owning thread**. 
- Reentrancy facilitates encapsulation of locking behavior.
- JVM monitors two specific info per object's monitor :
	- Owner thread
	- Acquisition count 

**Non-Reentrant:** "You can't enter this room because you are already inside it." (Impossible/Deadlock)
**Reentrant:** "Since you're already inside the house, you can open any interior door without needing a new key."


For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock.


Acquiring the lock associated with an object does not prevent other threads from accessing that object—the only thing that acquiring a lock prevents any other thread from doing is acquiring that same lock.


| Programme                                           | Synchronization Requirement                                        |
| --------------------------------------------------- | ------------------------------------------------------------------ |
| Single-threaded prog                                | No synchronization                                                 |
| Mutli-threaded prog                                 |                                                                    |
| 1. single variable                                  | the variable guarded by a lock                                     |
| 2. class has invariants involving 1+ state variable | each variable in the invariant must be guarded by the *same* lock. |



| `synchronized` keyword     | term “synchronization”                                                             |
| -------------------------- | ---------------------------------------------------------------------------------- |
| provides exclusive locking | also includes the use of volatile variables, explicit locks, and atomic variables. |



| Rule                                                                                                                                                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Whenever more than one thread accesses a given state variable, and one of them might write to it, they all must coordinate their access to it using synchronization.                                                                                                                                                      |
| If multiple threads access the same mutable state variable without appropriate synchronization, your program is broken. There are three ways to fix it:<br>1. Don’t share the state variable across threads;<br>2. Make the state variable immutable; or<br>3. Use synchronization whenever accessing the state variable. |
| It is far easier to design a class to be thread-safe than to retrofit it for thread safety later.                                                                                                                                                                                                                         |
| When designing thread-safe classes, good object-oriented techniques— encapsulation, immutability, and clear specification of invariants—are your best friends                                                                                                                                                             |
| To preserve state consistency, update related state variables in a single atomic operation.                                                                                                                                                                                                                               |
| For each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock.                                                                                                    |
| Every shared, mutable variable should be guarded by exactly one lock. Make it clear to maintainers which lock that is                                                                                                                                                                                                     |
| For every invariant that involves more than one variable, all the variables involved in that invariant must be guarded by the same lock.                                                                                                                                                                                  |
| There is frequently a tension between simplicity and performance. When implementing a synchronization policy, resist the temptation to prematurely sacrifice simplicity (potentially compromising safety) for the sake of performance                                                                                     |
| Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network or console I/O                                                                                                                                                                                            |



---
### Reentrancy in Depth
"*Reentrancy*
*When a thread requests a lock that is already held by another thread, the requesting thread blocks. But because intrinsic locks are reentrant, if a thread tries to acquire a lock that it already holds, the request succeeds. "*
*Explain the second line w/ example*

To understand **reentrancy**, think of the lock as a **bracelet** or a **pass**.
In most non-reentrant systems, if you are already wearing the "lock bracelet" and you encounter another door that requires the same bracelet, you would get stuck waiting for yourself to take it off—a classic **deadlock**.
Because Java's intrinsic locks (`synchronized`) are **reentrant**, the JVM says: _"Oh, it's you again! You already have the pass, so come right in."_

#### 1. Why do we need this? (The Problem)

The most common case for reentrancy is when a **synchronized method calls another synchronized method** in the same class (or a subclass calling a superclass method).

Without reentrancy, the following code would **lock up forever**:
```Java
public class ReentrantDance {
    
    // Method 1: Grabs the lock for 'this' object
    public synchronized void outerStep() {
        System.out.println("Executing Outer Step...");
        
        // Method 2: ALSO wants the lock for 'this' object
        innerStep(); 
        
        System.out.println("Outer Step Finished.");
    }

    public synchronized void innerStep() {
        System.out.println("Executing Inner Step while holding the same lock!");
    }

    public static void main(String[] args) {
        ReentrantDance dance = new ReentrantDance();
        // This works perfectly because of Reentrancy
        dance.outerStep(); 
    }
}
```
#### 2. How Java tracks this "Behind the Scenes"

The JVM doesn't just check if the lock is "busy." It tracks two specific pieces of information for every object's monitor:
1. **Owner:** Which thread currently holds the lock?
2. **Acquisition Count:** How many times has this specific thread entered a synchronized block for this object?

- **When a thread acquires the lock:** Owner is set to `Thread-A`, and Count becomes `1`.
- **When the same thread calls another synchronized method:** The JVM sees Owner is already `Thread-A`, so it just increments the Count to `2`.
- **When the thread exits a method:** The Count decrements.
- **When Count reaches 0:** The lock is officially released for other threads to grab.
#### 3. Reentrancy with Inheritance
This is a favorite **interview question**. If a subclass overrides a synchronized method and calls `super.doSomething()`, it relies on reentrancy.
```Java
class Parent {
    public synchronized void doWork() {
        System.out.println("Parent working...");
    }
}
class Child extends Parent {
    @Override
    public synchronized void doWork() {
        System.out.println("Child working...");
        super.doWork(); // Reentrancy allows this!
    }
}
```

If locks weren't reentrant, the `Child` thread would block forever trying to call the `super` method because it already "owns" the lock on itself (and in Java, the `Child` instance and the `Parent` part of it are the **same object**).
##### Summary
- **Non-Reentrant:** "You can't enter this room because you are already inside it." (Impossible/Deadlock)
- **Reentrant:** "Since you're already inside the house, you can open any interior door without needing a new key."

--- 
*"(and in Java, the Child instance and the Parent part of it are the same object)." explain this from your answer in more detail*

This is a fundamental concept of Object-Oriented Programming (OOP) in Java that often confuses developers when they start thinking about memory and locks.
To understand why **reentrancy** works with inheritance, you have to look at how Java stores objects in the **Heap**.
#### 1. The "Single Memory Block" Concept
When you write `Child c = new Child();`, the JVM doesn't create two separate objects (one for Parent and one for Child). Instead, it allocates **one single, continuous block of memory** on the Heap.
This memory block contains:
- Fields defined in the `Parent` class.
- Fields defined in the `Child` class.
- **A single Object Header.**
#### 2. The Single Monitor Lock
Every object in Java has exactly **one** monitor lock (intrinsic lock) located in its **Object Header**.
When the `Child` thread enters the `synchronized` method in the `Child` class, it grabs the lock for that specific memory block (the `this` instance). When it then calls `super.doWork()`, it is trying to access a method that belongs to the **same memory block**.

Since there is only **one lock** for that entire block of memory, the thread is essentially asking itself for permission to use a lock it already owns.

#### 3. Why this matters for Reentrancy

If `Parent` and `Child` were separate objects in memory, reentrancy wouldn't even be the issue—you would be dealing with two different locks entirely.
But because they are the **same object**:
1. **Thread A** enters `Child.doWork()` → **Count = 1**, Owner = Thread A.
2. **Thread A** calls `super.doWork()` (which is also `synchronized`).
3. The JVM checks the Object Header of the `Child` instance.
4. It sees **Owner = Thread A**.
5. **Reentrancy kicks in:** It increments the **Count to 2** and lets the thread proceed.
#### 4. Comparison: Composition vs. Inheritance
This is where the distinction becomes clear. If you used **Composition** instead of Inheritance, you would have two different objects and two different locks:

|Structure|Relationship|Number of Locks|Reentrancy Needed?|
|---|---|---|---|
|**Inheritance**|Child **is a** Parent|**1 Lock**|**Yes** (to call super).|
|**Composition**|Teacher **has a** Book|**2 Locks**|**No** (they are different "keys").|

> **Key Takeaway:** In Java, `super` is not a reference to a different object; it is simply a keyword that tells the JVM to look at the method implementation in the parent's class file for the **current** object.

----

## Chapter 3. Sharing Objects

1. **Visibility** 
	1. Stale data
	2. Nonatomic 64-bit operations
	3. Locking and visibility
	4. Volatile variables
2. **Publication and escape** 
	1. Safe construction practices
3. **Thread confinement** 
	1. Ad-hoc thread confinement
	2.  Stack confinement
	3. `ThreadLocal`
4. **Immutability** 
	1. Final fields
	2. Example: Using volatile to publish immutable objects
5. **Safe publication**  
	1. Improper publication: when good objects go bad
	2. Immutable objects and initialization safety
	3. Safe publication idioms
	4. Effectively immutable objects
	5. Mutable objects
	6. Sharing objects safely

---

Immutable objects are always thread-safe.

An object is *immutable* if:
• Its state cannot be modified after construction;
• All its fields are `final`; and
• It is properly constructed (the `this` reference does not escape during construction).

Just as it is a good practice to make all fields **private** unless they need greater **visibility**, it is a good practice to make all fields **final** unless they need to be **mutable**.

Immutable objects can be used safely by any thread without additional synchronization, even when synchronization is not used to publish them.

The publication requirements for an object depend on its mutability:
• **Immutable objects** can be published through any mechanism;
• **Effectively immutable objects** must be safely published;
• **Mutable objects** must be safely published, and must be either thread-safe or guarded by a lock. 



## Chapter 4. Composing Objects

1. Designing a thread-safe class  
	1. Designing a thread-safe class
	2. 
2. Instance confinement 
3. Delegating thread safety  
4. Adding functionality to existing thread-safe classes 
5. Documenting synchronization policies 


Making a class thread-safe means ensuring that its invariants hold under concurrent access; this requires reasoning about its state. 

---
# Terms
- invariants : Class invariants
- postconditions : Method postconditions 
- **race condition** : occurs when the correctness of a computation depends on the relative timing or interleaving of multiple threads by the runtime. The most common type of race condition is *check-then-act*. 
	- Example: race conditions in lazy initialization.
	- Read-modify-write operations
- Context switches : runtime overhead
- **mutable** : An object (state) that may be modified after construction. 
- **compound actions** : sequences of operations that must be executed atomically in order to remain thread-safe.  e.g. check-then-act and read-modify-write sequences. 
- 
- **Effectively Immutable** : Objects not technically immutable but whose state will not be modified after publication. 
- 1


| Mutable                        | Immutable |
| ------------------------------ | --------- |
| Date (*effectively immutable*) | String    |
