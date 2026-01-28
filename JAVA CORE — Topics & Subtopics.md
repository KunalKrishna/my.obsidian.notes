## 1. JVM Architecture (Fundamentals Only)

-  JVM overview
-  JDK vs JRE vs JVM
-  Java compilation pipeline (`.java → .class → JVM`)
-  Class loaders (Bootstrap, Extension, Application)
-  Runtime Data Areas
    -  Method area
    -  Heap
    -  Java stack
    -  PC register
    -  Native method stack
-  Execution engine basics
    -  Interpreter
    -  JIT compiler (just fundamentals)
-  Bytecode basics (`javap -c`)

---
## 2. Java Syntax & Structure
-  Source file structure
-  Packages
-  Imports
-  Comments and readability rules

---
## 3. Data Types & Variables
-  Primitive types (size, range, default value)
-  Reference types
-  Variable kinds
    -  Local
    -  Instance
    -  Static
-  Type casting :  Implicit vs  Explicit
-  `var` (Java 10+)

---
## 4. Operators

---
## 5. Control Flow Statements

---

## 6. Classes (Complete Fundamentals)

-  Class declaration & structure
-  Access modifiers
    -  `public`
    -  `protected`
    -  default
    -  `private`
-  Non-access modifiers
    -  `final`
    -  `static`
    -  `abstract`
    -  `strictfp`

-  Fields & initialization order
-  Constructors
    -  Default constructor
    -  Parameterized constructor
    -  Constructor overloading
    -  Constructor chaining (`this()`, `super()`)
-  Static vs instance members
-  Blocks (static block, instance block)
-  Nested types
    -  Static nested class
    -  Inner class
    -  Local class
    -  Anonymous class

---

## 7. Methods (Core Concepts)

-  Method declaration syntax
-  Return types
-  Method overloading
-  Pass-by-value semantics (especially objects)
-  Varargs
-  Static methods vs instance methods

---

## 8. OOP Principles (Complete Core)

-  Encapsulation
-  Abstraction
-  Inheritance
    -  `extends`
    -  Multilevel inheritance
    -  Method overriding
    -  `super` keyword
-  Polymorphism
    -  Compile-time polymorphism
    -  Runtime polymorphism
    -  Dynamic dispatch
-  `final` for class/method/variable
-  Composition vs inheritance

---

## 9. Interfaces & Abstract Classes

-  Interface declaration
-  Default methods (Java 8+)
-  Static methods in interfaces
-  Private methods in interfaces (Java 9+)
-  Functional interfaces
-  Marker interfaces
-  Abstract classes vs interfaces (full comparison)

---

## 10. Strings (Fundamentals Only)

-  String immutability
-  String literal pool
-  `new String()` vs literal
-  `intern()`
-  Basic APIs: `length`, `charAt`, `substring`, `equals`, `compareTo`, `concat`
-  `StringBuilder` vs `StringBuffer` (only core difference)
-  Compact strings (high level)

---

## 11. Arrays (Core)

-  Declaration & initialization
-  Single-dimensional arrays
-  Multi-dimensional arrays
-  Array copying (`System.arraycopy`)
-  Passing arrays to methods

---

## 12. Enums

-  Enum basics
-  Enum with fields & methods
-  Enum constructors
-  Enum constants behavior

---

## 13. Exception Handling

-  Checked vs unchecked exceptions
-  `try`, `catch`, `throw`, `throws`, `finally`
-  Multi-catch
-  Exception hierarchy
-  Custom exceptions
-  `try-with-resources` & AutoCloseable

---

## 14. I/O Fundamentals (Only Basic)

-  Streams concept
    -  InputStream
    -  OutputStream
    -  Reader
    -  Writer
-  File basics (java.io.File)
-  Basic reading/writing text
_(Not including NIO or Channels)_

---

## 15. Annotations (Core Basics)
-  Built-in annotations (`@Override`, `@Deprecated`, `@SuppressWarnings`)
-  Custom annotations (simple)
-  Retention & Target basics (no reflection API exploration)

---

## 16. Java Memory Model (Only as needed for Core)

-  Stack vs heap
-  Object layout (fields, references)
-  Reference vs object
-  Garbage collection fundamentals
-  Escape analysis (conceptual)

---

## 17. Java 8+ Core Additions

(Non-collections, non-streams basics)
-  Lambda basics (syntax only)
-  Functional interfaces (`Runnable`, `Supplier`, etc.)
-  Method references
-  Default methods in interfaces
_(Exclude Streams API if you want pure Core only)_

---

## 18. Command-Line & Tools
-  `javac`
-  `java`
-  `jar`
-  `javap` (bytecode basic inspection)