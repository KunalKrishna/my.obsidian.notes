VM  = hardware level abstraction 
JVM = a VM which loads and executes java programs in memory.
### JDK vs JRE vs JVM

**One line:** JVM runs bytecode, JRE lets you run apps, JDK lets you build them.
- **JVM** (Java Virtual Machine) — the engine that executes `.class` bytecode. Platform-specific (different JVM per OS).
- **JRE** (Java Runtime Environment) — JVM **+** core libraries (`java.lang`, `java.util`, etc.). Enough to _run_ Java, not to compile.
- **JDK** (Java Development Kit) — JRE **+** dev tools (`javac`, `jar`, `javadoc`, `jdb`). Needed to _develop_.

Nesting: `JDK ⊃ JRE ⊃ JVM`

_Interview Q: "Difference between JDK, JRE, JVM?"_ → JDK to build, JRE to run, JVM to execute bytecode.


![[Pasted image 20260724222722.png]]
![[JDK-JRE-JVM.png]]


### Compilation & Execution Flow

**One line:** Source → bytecode (compile once) → machine code (run anywhere).
```
Hello.java  --javac-->  Hello.class  --java-->  JVM  -->  output
 (source)              (bytecode)            (runtime)
```

Example:
```java
// Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hi");
    }
}
```

```bash
javac Hello.java   # produces Hello.class (bytecode)
java Hello         # JVM loads, verifies, runs it
```

Inside the JVM at runtime: **ClassLoader** loads `Hello.class` → bytecode is **verified** → **Execution Engine** runs it (interpreted first, hot code JIT-compiled to native).

_Interview Q: "What happens when you run `java Hello`?"_ → load → link/verify → initialize → execute (interpret + JIT).

---
### JVM Architecture Overview (ClassLoader + Memory + Execution )

**One line:** Three subsystems — load classes, hold data, execute code.
![[Pasted image 20260729173221.png]]

```
        ┌─────────────────────────────┐
        │      1. ClassLoader         │  loads .class files
        └──────────────┬──────────────┘
                       ▼
        ┌─────────────────────────────┐
        │   2. Runtime Data Areas     │  Heap, Stack, Metaspace,
        │        (Memory Area)        │  PC Register, Native Stack
        └──────────────┬──────────────┘
                       ▼
        ┌─────────────────────────────┐
        │   3. Execution Engine       │  Interpreter + JIT + GC
        └─────────────────────────────┘
```

1. **ClassLoader** — finds and loads bytecode → _(detail in Module 2)_
2. **Runtime Data Areas** — memory the JVM manages → _(detail in Module 3)_
3. **Execution Engine** — interpreter, JIT compiler, garbage collector → _(detail in Modules 5 & 6)_
![[JVM Architecture Overview.png]]
![[JVM Arch.png]]
![[JVM Architecture Overview2.png]]

_Interview Q: "Explain JVM architecture."_ → name the 3 subsystems and one job each.

---
### Platform Independence (WORA)

**One line:** "Write Once, Run Anywhere" — bytecode is universal, the JVM is not.

- `javac` compiles to **bytecode**, which is OS-neutral.
- Each OS has its **own JVM** that translates bytecode to that machine's native instructions.
- So _Java is portable; the JVM is platform-specific_.

```
				Windows JVM ──┐
(same .class)	Linux JVM   ──┼── run identical Hello.class
				macOS JVM   ──┘
				

													Windows JVM ──┐
[Hello.java]  --javac-->  [Hello.class]  --java-->  Linux JVM   ──┼── run identical Hello.class
													macOS JVM   ──┘		
(source) 					(bytecode) 				(runtime)
							Platform                 Platform
							Indepedent               Depedent
```

![[Pasted image 20260724233434.png]]
Contrast: C compiles straight to native code, so you recompile per platform. Java's bytecode layer removes that step.

The Architecture Comparison

```
Java: [Source Code] ──> [Java Compiler (Same for all OS)]    ──> [Bytecode (.class)]                ──> [JVM (Different for each OS)] 
C:    [Source Code] ──> [C Compiler (Different for each OS)] ──> [Native Machine Code (.exe, .out, etc.)]
```

_Interview Q: "How does Java achieve platform independence?"_ → bytecode + a per-platform JVM.

---
## The Two Translation Phases (Compiled Twice)

Java uses a **two-phase translation system** to achieve both platform independence and high performance. Here is how your code is processed: 

```
                  [ PHASE 1: BUILD TIME ]
                      Your Source Code (.java)
                                 │
                                 ▼  <── Compiled by `javac` (Static Compiler)
                         Bytecode (.class) 
                 (Stored as Class Metadata in Metaspace)
                                 │
                  [ PHASE 2: RUNTIME ENGINE ]
                                 ├────────────────────────┐
                                 ▼                        ▼
                       [ JIT Compiler (C1/C2) ]     [ Interpreter ]
                                 │                        │
  (Compiles Hot Code)            ▼                        ▼ (Translates Line-by-Line)
                    Native Machine Code Binary ──> [ PHYSICAL CPU CHIP ]
                     (Stored in Code Cache)
```
### Phase 1: Compilation to Bytecode (`javac`)

When you run `javac MyProg.java`, the compiler translates human-readable Java text into an intermediate language called **Bytecode** (`.class`). ]

- **Why it is called "Uncompiled" in the context of the JVM:** To your physical computer processor (Intel, AMD, ARM), bytecode is uncompiled. The CPU chip cannot read or execute a `.class` file directly. Bytecode is written for a _fictional_ processor (the JVM).
- This bytecode is what gets loaded into **Metaspace** as part of the Class Metadata.  

> [!note]- **JVM Languages & Bytecode**
> The JVM is **language-agnostic** — any language that compiles to valid bytecode can run on JVM:
> 
> | Language | Notes                            |
> | -------- | -------------------------------- |
> | Java     | Primary language                 |
> | Kotlin   | Android, modern Java alternative |
> | Scala    | Functional + OOP                 |
> | Groovy   | Dynamic scripting                |
> | Clojure  | Lisp dialect                     |
### Phase 2: Compilation to Native Code (The JIT Compiler)  

When the program runs, the execution engine reads that bytecode out of Metaspace. If a loop or method runs thousands of times, the **JIT Compiler** steps in and compiles that bytecode a _second time_.

- It translates the intermediate bytecode into **Native Machine Code Binary** (zeros and ones tailored precisely for your specific physical CPU chip).
- This high-speed binary is what gets cached inside the **Code Cache**.  

---
### 📊 Comparing the Output of Both Compilers

To prove this difference in an interview, you can compare what both translation steps output:

|Feature|Phase 1: `javac` Output|Phase 2: JIT Compiler Output|
|---|---|---|
|**Input Format**|Plain Text Java Code (`.java`)|Intermediate Bytecode (`.class`)|
|**Output Format**|**Bytecode** (Standardized JVM code)|**Native Machine Code** (Hardware binary)|
|**Stored Where?**|Inside **Metaspace** (as Class Metadata)|Inside the **Code Cache**|
|**Read By**|The JVM Execution Engine|The Physical Hardware CPU|
|**OS Dependent?**|No. Universal for all computers.|Yes. Tailored to the exact physical chip.|

