Module 4 is the payoff of Module 3, turning "here are the memory areas" into a precise rule for _any_ variable. I've reused the stack-reference/heap-object framing so it reinforces what you just learned.

---
## Where Things Live

**One line:** A single rule covers almost every interview variant — objects & static fields on the heap, method-local primitives and references on the stack. 

Memorize this table; interviewers love turning it into "where is X stored?" rapid-fire:

| What                                                      | Where it lives                                                                               |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Instance fields (fields inside an object)                 | **Heap** (inside their owning object)                                                        |
| Objects and arrays (`new ...`)                            | **Heap**                                                                                     |
| Local primitives (`int`, `boolean`, etc. inside a method) | **Stack** (in the frame's locals)                                                            |
| Local reference variables (the pointer, not the object)   | **Stack** (value is a reference into the heap)                                               |
| Method parameters                                         | **Stack** (same as locals)                                                                   |
| `static` fields                                           | **Heap** — hosted inside the class's `java.lang.Class` mirror object _(pre-Java 8: PermGen)_ |
| String literals                                           | **String Pool** (a special region of the heap)                                               |
| Class Metadata                                            | **Metaspace** (Native OS Memory - RAM)                                                       |
🗺️ The Master Storage Matrix (Java 8+)

| The Item (X)                            | Where is it stored? (Physical Location) | Core Interview Catch / Detail                         |
| --------------------------------------- | --------------------------------------- | ----------------------------------------------------- |
| **Instance Variables (Primitives)**     | **Main Heap**                           | Embedded directly inside the parent object wrapper.   |
| **Instance Variables (Objects)**        | **Main Heap**                           | The reference variable points to another Heap object. |
| **Static Primitive Variables**          | **Main Heap**                           | Attached to the `java.lang.Class` mirror object.      |
| **Static Object References**            | **Main Heap**                           | The reference _and_ the object live on the Heap.      |
| **Local Variables (Primitives)**        | **JVM Stack Frame**                     | Destroyed instantly when the method returns.          |
| **Local Variables (Object References)** | **JVM Stack Frame**                     | The 4/8-byte pointer is on Stack; object is on Heap.  |
| **Class Metadata**                      | **Metaspace** (Native Memory)           | Moved out of PermGen in Java 8.                       |
| **Method Bytecode Instructions**        | **Metaspace** (Native Memory)           | Lives inside the Class Metadata structure.            |
| **String Literals / Interned Strings**  | **Main Heap** (String Table)            | Moved out of PermGen back in Java 7.                  |
| **Thread Execution Bookmark**           | **PC Register**                         | Ultra-small thread-private native tracking slot.      |

The one example that exercises every row:

```java
public class Storage {
    static int shared = 100;        // static field -> HEAP (inside Storage.class mirror object)
    int instanceField = 5;          // instance field -> HEAP (inside each object)

    public void doWork(int param) { // param -> STACK (this frame's locals)
        int local = 10;             // primitive local -> STACK
        Storage s = new Storage();  // 's' reference -> STACK; the OBJECT -> HEAP
        int[] arr = new int[3];     // 'arr' reference -> STACK; the array -> HEAP
    }
}
```

Two distinctions that catch people out:

**1. The reference vs. the object.** `Storage s = new Storage();` creates _two_ things in _two_ places — the object on the heap, and the reference `s` on the stack pointing to it. "Where is `s`?" → the reference is on the stack; the object is on the heap.

**2. Instance field vs. local primitive.** An `int` declared _inside a method_ is on the stack. The _same_ `int` declared _as a field of a class_ lives on the heap, inside its object. Location depends on **where it's declared**, not on the type.

```
STACK (doWork frame)                    HEAP
   ┌──────────────────────┐              ┌────────────────────┐
   │ param  = 5           │              │ Storage object     │
   │ local  = 10          │      ┌──────▶│  instanceField = 5 │
   │ s      ──────────────┼──────┘       └────────────────────┘
   │ arr    ──────────────┼──────┐       ┌────────────────────┐
   └──────────────────────┘      └──────▶│ int[3] array       │
                                         └────────────────────┘
                                         ┌──────────────────────────┐
                                         │ Storage.class (mirror)   │  <- also on HEAP
                                         │   static shared = 100    │
                                         └──────────────────────────┘

      METASPACE (native memory)
   ┌──────────────────────────────────┐
   │ Storage class *metadata*:        │   method bytecode, constant pool,
   │  field/method descriptors, etc.  │   field descriptors — NOT the static values
   └──────────────────────────────────┘
```

Scenario 1: "Where is an Object stored?"
- **The Baseline:** All objects, without exception, are allocated on the **Main Heap Area**.
- **The Nuance:** The variables that _point_ to that object can be stored in three different locations depending on scope:
    1. If it is a **local method variable**, the reference pointer is on the **JVM Stack Frame**.
    2. If it is a **class field variable**, the reference pointer is inside the parent object on the **Heap**.
    3. If it is a **static variable**, the reference pointer is attached to the `Class` object on the **Heap**.

Scenario 2: "Where are Local Variables stored?"
- **The Rule:** Local variables are thread-private and live inside the **JVM Stack Area** within a temporary structure called a **Stack Frame**.
- **The Type Split:**
    - If it is a primitive type (`int x = 5;`), the actual value `5` is saved directly inside the Local Variable Array of that Stack Frame.
    - If it is an object reference (`User u = new User();`), the variable `u` on the stack frame holds only a 32-bit or 64-bit numerical memory address pointer. The actual `User` object data structure sits out on the shared **Heap**. 

Scenario 3: "Where are Static Fields stored post-Java 8?"
- **The Historical Context Trap:** Interviewers check if your knowledge is outdated. Older textbooks say PermGen.
- **The Modern Truth:** Static variables **do not live in Metaspace**. When Java 8 deleted PermGen, class metadata moved to native Metaspace, but all static variables were moved directly to the **Main Heap**. They are physically hosted as fields inside the `java.lang.Class` instance created by the JVM for that specific type.

_Interview Q: "Where are static fields stored post-Java 8?"_ → On the **heap**, hosted inside the `java.lang.Class` object for that type — **not** in Metaspace. Metaspace holds the class _metadata_ (bytecode, constant pool, descriptors); the static _variable values_ live on the heap. (Pre-Java 8 they were in PermGen.) This is a deliberate "is your knowledge current?" trap — the outdated answer is "Metaspace" or "PermGen."

---
## String Pool & `intern()`

**One line:** String literals are cached and reused in a special heap area (the String Pool); `new String()` bypasses the pool and forces a separate object — which is why `.==` behaves surprisingly.

Strings get special treatment because they're immutable and extremely common, so the JVM reuses identical literals instead of duplicating them.

**The two ways to create a String differ:**

- **Literal** — `String a = "hi";` → the JVM checks the String Pool. If `"hi"` is already there, it reuses that same object. If not, it adds it. So identical literals share one object.
- **`new String("hi")`** → **always** creates a brand-new object on the heap, _outside_ the pool, even if `"hi"` already exists in the pool. The `new` keyword forces a fresh allocation.

This is exactly why `.==`  (reference equality) trips people up:

```java
public class StringPoolDemo {
    public static void main(String[] args) {
        String a = "hi";              // pooled
        String b = "hi";              // reuses the SAME pooled object
        String c = new String("hi");  // NEW object on heap, not pooled

        System.out.println(a == b);        // true  -> same pooled reference
        System.out.println(a == c);        // false -> different objects
        System.out.println(a.equals(c));   // true  -> same CONTENTS
    }
}
```

```
   POOL (in heap)              HEAP (outside pool)
   ┌───────────┐               ┌───────────┐
   │ "hi"  ◀───┼── a           │ "hi"  ◀───┼── c   (separate object)
   │       ◀───┼── b           └───────────┘
   └───────────┘
```

- `a == b` → **true**: both point to the one pooled `"hi"`.
- `a == c` → **false**: `c` is a separate heap object.
- `a.equals(c)` → **true**: `.equals()` compares _contents_, not references.

**The rule that falls out of this:** always compare String contents with `.equals()`, never `.==`.  `.==` compares whether they're the _same object_, which is almost never what you want for Strings.

**`intern()`** — manually forces a String into the pool (or returns the existing pooled copy):

```java
String c = new String("hi");   // heap object, not pooled
String d = c.intern();         // returns the POOLED "hi"
String a = "hi";               // the pooled literal
System.out.println(a == d);    // true -> d is now the pooled reference
```

_Interview Q: "`new String("hi")` vs `"hi"` — how many objects, and why does `.==` differ?"_ → The literal is pooled and shared; `new String()` forces a separate heap object, so `.==` is false while `.equals()` is true. Always use `.equals()` for content comparison; `intern()` pulls a String into the pool.

---

That's Module 4 — deliberately compact. For your MOC, link **Where Things Live** ↔ Module 3's _Heap vs Stack_ (it's the applied version of that comparison) and ↔ Module 2 (statics landing in Metaspace connects to the Loading phase). Link **String Pool** ↔ Module 5's _Reference types_ later, since both deal with special heap behavior.

Module 5 is the big one — Garbage Collection — and it's the heaviest interview area. Want me to do it as one large batch, or split it (say, GC fundamentals first, then collectors + troubleshooting)?