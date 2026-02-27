Imp Timeline : 
- 1.2 Collection API
- 1.5 Generics
- 1.8 Functional Programming (Lambda Expression, Streams API++)

What was the need of FP in Java? Why Java 8 introduced FP?

How Java 8 introduced FP in Java? What architectural change Java introduced to enable FP in Java? 
## Meta Questions 
###  What is the need of Functional Programming(Java 8) in Java ?

THE 5 REAL PROBLEMS Functional Programming Solved in Java : 
#### 1️⃣ **Too Much Boilerplate Code (Anonymous Classes Hell)**
**Before Java 8:**  
To pass behavior (a function) to a method, you had to define **whole anonymous classes** — even for one-line logic
Ex : sorting a list case-insensitively:
```Java
List<String> names = Arrays.asList("Kiran", "rahul", "John");

Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareToIgnoreCase(s2);
    }
});
```
**After Java 8 (with lambdas):**
```Java
names.sort((s1, s2) -> s1.compareToIgnoreCase(s2));
```
#### 2️⃣ **Rigid, Imperative Collection Processing**
**Before Java 8:** You had to manually write loops for everything.
Ex : filter even numbers: 
```Java
List<Integer> evens = new ArrayList<>(); 
for (Integer n : numbers) {  
    if (n % 2 == 0) { 
        evens.add(n); 
    } 
} 
```
**After Java 8 (with Streams + lambdas):**
```Java
List<Integer> evens = numbers.stream()
                             .filter(n -> n % 2 == 0)
                             .toList();
```

#### 3️⃣ **Difficult Parallelization and Multi-Core Utilization**
**Before Java 8:** Parallel programming = threads, synchronization, locks — **complicated and error-prone.**
Ex : summing large list:
```Java
int sum = 0;
for (int n : numbers) {
    sum += n;
}
// To parallelize, you’d need threads, executors, futures, locks...
```
**After Java 8 (with Streams + lambdas):**
```Java
int sum = numbers.parallelStream()
                 .mapToInt(n -> n)
                 .sum();
```

#### 4️⃣ **No Simple Way to Pass Behavior as Data**
**Before Java 8:** you could only pass _objects_ and _data_, not _actions (functions)_.
Ex : you want to run an action on every element:
```Java
import java.util.*;

interface NameProcessor {
    void process(String name);
}

public class BeforeJava8Example {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Kiran", "Rahul", "John");

        // Passing behavior using ANONYMOUS CLASS
        processNames(names, new NameProcessor() {
            @Override
            public void process(String name) {
                System.out.println("Hello " + name);
            }
        });
    }

    // Method expecting some ACTION on each name
    static void processNames(List<String> names, NameProcessor processor) {
        for (String n : names) {
            processor.process(n);
        }
    }
}
```
➡️ Overkill for simple operations.

**After Java 8 :**
```Java
import java.util.*;
import java.util.function.Consumer;

//---------------Already in JAVA LIBRARY------------------
@FunctionalInterface  
public interface Consumer<T> {  
	void accept(T t);  
	  
	default Consumer<T> andThen(Consumer<? super T> after) {  
		Objects.requireNonNull(after);  
		return (T t) -> { accept(t); after.accept(t); };  
	}  
}
//---------------------------------------------------------

public class AfterJava8Example {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Kiran", "Rahul", "John");

        // Passing behavior using LAMBDA
        processNames(names, name -> System.out.println("Hello " + name));

        // Even shorter using METHOD REFERENCE
        processNames(names, System.out::println);
    }

    // Here we reuse Java's built-in functional interface
    static void processNames(List<String> names, Consumer<String> processor) {
        for (String n : names) {
            processor.accept(n);
        }
    }
}
```

##### More on : Ways/Tricks/Design Patterns to pass Function/Behaviour as argument in (high-order) functions 
Before Java 8, Java **did not** have any official functional-programming features, but developers _still needed_ to pass behavior into methods.  
Since Java had **no functions as first-class citizens**, developers used a workaround.

That workaround _did_ have well-known names in the Java community.

Below are the correct historical/technical terms.

---

✅ **Names/Terms Used for “Passing a Function Without FP” Before Java 8**

1) **Callback Pattern** ← Most widely used term. This is the proper industry name.
A “callback” means:
> _You pass an object containing a method that the callee will call back later._

Java 7 did callbacks using interfaces + anonymous inner classes. Example:
```Java
MathOperation addition = new MathOperation() {     
	@Override     
	public int operate(int a, int b) {         
		return a + b;     
	} 
}; 
execute(10, 5, addition);
```

This is a **callback**, even though underneath it's OOP-based.

---

2) **Strategy Pattern**

Your code defines a family of algorithms (addition, subtraction, etc.) as **strategy objects**.
```Java
MathOperation strategy = new MathOperation() { ... }; 
execute(x, y, strategy);
```

This is literally the textbook definition of Strategy Pattern:
- Define an interface representing an operation.
- Pass different implementations to change behavior.

Java 8 lambdas became a “lightweight strategy”.

---
3) **Anonymous Inner Class as Function Object**

Not an official design pattern, but frequently described this way in books and forums.

Since Java couldn't pass methods, developers passed **objects whose only purpose was to hold a method**.

This was called a _function object_ in OO terminology.

---

4) **Command Pattern**
A “command” represents an action packaged as an object.  
Java devs used this for event listeners, button click handlers, etc.

```Java
button.addActionListener(new ActionListener() {     
		public void actionPerformed(ActionEvent e) {         
			...     
		} 
	}
);
```
Before lambdas → GUI code relied heavily on **Command Pattern**.

---

5) **SAM Pattern** (prior to formal “Functional Interface”)

People called this concept: Single Abstract Method (SAM) interfaces

Even before Java 8 standardized it as “Functional Interface”, Java libraries like:
- Runnable
- Callable
- Comparator  
    …were already SAM interfaces.

Developers used SAM interfaces _implicitly_ as poor man’s function types.

---

📌 Summary Table

|Before Java 8 term|Meaning|How it matches your example|
|---|---|---|
|**Callback Pattern**|Pass object so its method is later called|`execute(10, 5, new MathOperation(){})`|
|**Strategy Pattern**|Swap algorithms by passing objects|Addition, subtraction strategies|
|**Command Pattern**|Represent action as an object|GUI listeners, event handlers|
|**Function Object**|OO way to mimic function-as-value|anonymous inner class with single method|
|**SAM Interface**|Interface with one method|Runnable, Comparator, your interface|

All of these existed **to simulate functional programming in Java 7**.

---

🎯 The Key Insight

Before Java 8, Java _emulated_ functional programming using:
1) **Interfaces with one method**
2) **Anonymous inner classes**
3) **Object wrapping around behavior**

This was not called “functional programming”.  It was **OOP-pattern-based function passing**.

When Java 8 arrived, it formalized this workaround into:
- **functional interfaces**
- **lambda expressions**
- **method references**
…making the same concept expressive and concise.

| **Before Java 8                                                                                                                                                                                                                                                                                                                                                                                                                 | **After Java 8 :**                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `interface NameProcessor {`<br>    `void process(String name);`<br>`}`<br>                                                                                                                                                                                                                                                                                                                                                      | `@FunctionalInterface`  <br>`public interface Consumer<T> {`  <br>	`void accept(T t);`  <br>	`...andThen()`  <br>`}`                                                                                                                                                                                                                                                             |
| `public static void main(String[] args) {`<br>        `List<String> names = Arrays.asList("Kiran", "Rahul", "John");`<br><br>        `// Passing behavior using ANONYMOUS CLASS`<br>        `processNames(names, new NameProcessor() {`<br>            `@Override`<br>            `public void process(String name) {`<br>                `System.out.println("Hello " + name);`<br>            `}`<br>        `});`<br>    `}` | `public static void main(String[] args) {`<br>        `List<String> names = Arrays.asList("Kiran", "Rahul", "John");`<br><br>        `// Passing behavior using LAMBDA`<br>        `processNames(names, name -> System.out.println("Hello " + name));`<br><br>        `// Even shorter using METHOD REFERENCE`<br>        `processNames(names, System.out::println);`<br>    `}` |
| `// Method expecting some ACTION on each name`<br>    `static void processNames(List<String> names, NameProcessor processor) {`<br>        `for (String n : names) {`<br>            `processor.process(n);`<br>        `}`<br>    `}`                                                                                                                                                                                          | `// Here we reuse Java's built-in functional interface`<br>    `static void processNames(List<String> names, Consumer<String> processor) {`<br>        `for (String n : names) {`<br>            `processor.accept(n);`<br>        `}`<br>    `}`                                                                                                                                |

#### 5️⃣ **Verbose APIs and Poor Readability in Data Transformation Pipelines**
**Before Java 8:** Chained operations required **manual intermediate collections**.
Ex : uppercase all names starting with “K”:
```Java
List<String> upper = new ArrayList<>();
for (String n : names) {
    if (n.startsWith("K")) {
        upper.add(n.toUpperCase());
    }
}
```
**After Java 8 (with Streams Operations +lambdas + Method References):**
```Java
List<String> upper = names.stream()
                          .filter(n -> n.startsWith("K"))
                          .map(String::toUpperCase)
                          .toList();
```

#### 💡 Summary Table

| Problem (Pre-Java 8)                | Example                      | Solution in Java 8    | Key Feature            |
| ----------------------------------- | ---------------------------- | --------------------- | ---------------------- |
| 1. Boilerplate anonymous classes    | `new Comparator<>() { ... }` | Lambdas simplify      | Lambda expressions     |
| 2. Manual collection loops          | `for (..)`                   | Stream API            | Functional + Streams   |
| 3. Complex parallelization          | Threads, locks               | `parallelStream()`    | Functional concurrency |
| 4. No function-as-parameter         | Passing objects only         | Functional interfaces | SAM + Lambdas          |
| 5. Verbose transformation pipelines | Multiple temporary lists     | Stream chaining       | Map, Filter, Reduce    |
**Functional Programming Style (FP)**
This is an umbrella term for : 
- passing functions as values 
- avoiding mutable state 
- declarative style (focus on _what_, not _how_) 

**First-Class Functions / First-Class Citizens**
If a language allows functions to be:
- passed as **arguments**
- **returned** from functions
- **stored** in variables
…then the language supports **first-class functions**.

**Higher-Order Functions (HOF)**
A function is called a _higher-order function_ if:
- it **takes another function as an argument**, or
- **returns** a function.

| Before Java 8                                                                                                          | After Java 8                                                                                                       |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Callback Pattern : _You pass an object containing a method that the callee will call back later._ or <br>              |                                                                                                                    |
| **Strategy Pattern** : Your code defines a family of algorithms (addition, subtraction, etc.) as **strategy objects**. | Java 8 lambdas became a “lightweight strategy”.                                                                    |
| Purely Object-oriented, functions second class                                                                         | Functional interfaces(SAM) = bridges b/w Java's Object-oriented & functional world: functions became equal citizen |
| writing simple op on Collections required verbose code                                                                 | concise code                                                                                                       |
| **imperative** code (step-by-step commands explicitly telling computer "how" to perform a task)                        | declarative code                                                                                                   |
| focus : how to do                                                                                                      | focus : what to do                                                                                                 |
## TOPICS
- **Default & Static Methods in Interfaces:** foundation. The architectural change that allows interfaces to evolve.
	- It explains how Java added new methods like `.stream()` to existing interfaces like `List` without breaking old code.    
- **Functional Interfaces(SAM):** the rule (Single Abstract Method) for where Lambdas can be used. 
- **Lambda Expressions:** The new syntax (`->`).
- **Built-in Functional Interfaces** : standard toolbox provided by `java.util.function` so you don't have to create your own interfaces for every task
	- **`Predicate<T>`** → boolean test
	- **`Consumer<T>`** → side-effect operation
	- **`Supplier<T>`** → provides a value 
	- **`Function<T, R>`** → transformation
	- **`BiFunction<T, U, R>`** → two-argument function
- **Method References:** A shorthand syntax (`::`) which replaces Lambdas for cleaner code. 
- **Streams API:** The engine for processing data. 
- **Collectors:** The tool to pack Stream results back into Lists, Maps, or Sets.
- **Optional Class:** A container object often returned by Stream operations to handle nulls safely. 
- **CompletableFuture:** For asynchronous programming, which relies heavily on Functional Interfaces.


![[Lambda.png]]



#### Topic 1: Default & Static Methods in Interfaces
- Why `default` was needed ? maintain backward compatibility 
- Why `static` was needed? kill anti-pattern
- Diamond problem ? ambiguity in OOP that occurs w/ multiple inheritance
	- class D inherits from two classes B & C that share a common ancestor A 
	- class D implements `default` method of two interfaces B & C 

| **Feature**      | **Java 7 Interface**  | **Java 8 Interface**      | Java 9+                       |
| ---------------- | --------------------- | ------------------------- | ----------------------------- |
| **Methods**      | Abstract only         | Abstract, Default, Static | private (), private static () |
| **Variables**    | `public static final` | `public static final`     |                               |
| **Constructors** | No                    | No                        |                               |

##### Default Methods in Interfaces

**Problem: Backward Compatibility** - Java 8 needed to add `.stream()` to the `Collection` interface. Doing so would break **every single class** implementing that interface until they implemented the new method.

**Solution : `default` method** - Java 8 introduced the `default` keyword. It allows an **interface** to provide a (default) **method implementation**. 

```Java
public interface Vehicle {
    // 1. Standard Abstract Method (Forces implementation)
    void start();

    // 2. Default Method (Optional override)
    default void turnAlarmOn() {
        System.out.println("Turning the vehicle alarm on.");
    }
    //Implementing Classes:Automatically inherit `turnAlarmOn()`. They _can_ override it, but they don't _have_ to.
}
```

##### Static Methods in Interfaces
The primary reason Java 8 introduced **Static Methods** in interfaces was to **kill the "Utility Class" Anti-Pattern.**

Before Java 8, we had to create side-car utility classes (e.g., `Collections` class for `Collection` interface). Now, we can put static utility methods directly inside the interface.
```Java
public interface Vehicle {
    // Utility method accessible via Vehicle.getHorsePower(2500)
    static int getHorsePower(int rpm, int torque) {
        return (rpm * torque) / 5252;
    }
}
```

| Anti-pattern                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| An anti-pattern is ==a common, poor solution to a recurring problem that, while it may seem effective at first, leads to negative consequences in software development==. These can include code that is hard to maintain, buggy, or inefficient. Common examples include "spaghetti code," which is disorganized and hard to follow, and "God objects," which are classes that take on too much responsibility. e.g. `java.util.Collections` library for utility function like `sort()` |

|**Feature**|**Before Java 8**|**Java 8**|
|---|---|---|
|**Philosophy**|Interfaces are for _contracts_ only.|Interfaces are _contracts_ + _basic tools_.|
|**Code Location**|Separated into `Collections`, `Arrays`, `Paths`.|Moved inside `List`, `Comparator`, `Stream`.|
|**Inheritance**|N/A|**NOT Inherited.** You must call them via the Interface name.|

##### The "Diamond Problem" (Interview Favorite)

| Diamond Problem                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The "Diamond Problem" is an ambiguity in OOP that occurs with multiple inheritance, where a class (D) inherits from two classes (B and C) that share a common ancestor (A).                                                                            |
| How Java Avoids the Problem? by **restricting class inheritance to a single parent** (single inheritance) using the `extends` keyword. A class in Java can only inherit from one other class, which ensures a linear and unambiguous inheritance path. |
![[diamond problem.png]]
![[diamond problem 2.png]]
Since Java allows implementing multiple interfaces, what happens if you implement two interfaces that have the **same default method name**?

**Rule:** The compiler throws an error unless you **override** the method in your class to resolve the ambiguity.
```Java
// Diamond Problem : Resolution Logic
@Override
public void commonMethod() {
    InterfaceA.super.commonMethod(); // Pick A explicitly
}

//--------------------------------------------

@Override
public void commonMethod() {
    // re-define 
    // re-define
    // re-define
}
```

#### Topic 2: Functional Interfaces & SAM
**The Golden Rule:** A Functional Interface ==an interface that contains **exactly one abstract method**==.
This is called the **SAM (Single Abstract Method)** rule.

##### Lambdas & FI
Lambdas _only_ work with Functional Interfaces. Why? **Because a Lambda provides the implementation for _one_ specific behavior**. If an interface had two abstract methods, the compiler wouldn't know which one your Lambda is trying to implement.

**What counts towards the SAM rule?**
- **Abstract Methods:** YES (Must have exactly 1).
- **Default Methods:** NO (Infinite allowed).
- **Static Methods:** NO (Infinite allowed).
- **Object Class Methods:** NO (Methods like `toString()` or `equals()` don't count because every object inherits them anyway)
Some points to keep in mind : 
* Interface can extend @FI
* @FI can extend another @FI *ONLY IF* both child FI has the same method as parent FI.
* @FI can extend an Interface containing `default` and `static` methods
* 

A valid Functional Interface that looks complex but follows the rules perfectly:
```Java
@FunctionalInterface
public interface Converter {
	int MAX_VALUE = 100;// implicitly `public`, `static`, and `final` 
    
    // 1. THE SAM (The only abstract method)
    // This is the ONLY method the Lambda will implement.
    int convert(String input); // implicitly `public`, `abstract` 

    // 2. Default method (Ignored by SAM rule)
    default void log() {
        System.out.println("Logging...");
    }

    // 3. Static method (Ignored by SAM rule)
    static void printInfo() {
        System.out.println("Converter Interface");
    }
    
    // 4. Object class method (Ignored by SAM rule)
    @Override
    String toString(); 
}
```
`@FunctionalInterface` Annotation
- **Purpose:** It acts as a safety check.
- **Behavior:** Adding a second abstract method to an interface tagged with this annotation makes the compiler break with an error.
- **Is it required?** No. Any interface with 1 abstract method is functional _by definition_ (e.g., `Runnable` or `Comparable` in older Java versions). However, adding the annotation is best practice to prevent future developers from accidentally breaking the SAM rule


|              | `Comparable<T>`             | `Comparator<T>`             |
| ------------ | --------------------------- | --------------------------- |
| Speech       | adjective                   | noun                        |
| represents   | State(Identity)             | Strategy(Function)          |
| package      | `java.lang`                 | `java.util`                 |
| SAM          | n/a                         | `@FunctionalInterface`      |
| function     | `public int compareTo(T o)` | `int compare(T o1, T o2)`   |
| relationship | IS-A                        | HAS-A                       |
|              | trait of an object          | tool to compare two objects |
| Purpose      | natural ordering            | custom ordering             |
| Views        | only one view               | one view per Comparator     |



| pre-java8 @FunctionalInterface  | SAM                      |
| ------------------------------- | ------------------------ |
| public interface Comparator< T> | int compare(T o1, T o2); |
*Why  `java.util.Comparator<T>`  is marked @FunctionalInterface but  not `java.lang.Comparable<T>` ?*
short answer: **`Comparable` represents "State" (Identity), while `Comparator` represents "Strategy" (Function).**
Parts of Speech : 
- "Comparable" **adjective** - "able to be compared" or "worthy of comparison"
- "Comparator" **noun** - a person or thing that compares
The Method Signature Difference
- **`Comparator<T>` (`compare(T o1, T o2)`):**
    - Takes **two** external objects.
    - It does not rely on its own internal state (`this`).
    - It is a pure function: $f(x, y) \rightarrow \text{int}$.
    - **Verdict:** This fits the definition of a Lambda function perfectly.
- **`Comparable<T>` (`compareTo(T o)`):**
    - Takes **one** external object.
    - The other object is **`this`** (the instance itself).
    - It fundamentally relies on the internal state of the object it is attached to.
    - **Verdict:** This is not a "function" in the functional programming sense; it is an object method.
The Conceptual "Is-A" vs "Has-A"
- **Comparable:** An object **IS** comparable. A `Student` object _is_ comparable to another Student. It is intrinsic to the class definition. You implement it inside the class.
- **Comparator:** An object **HAS A** comparator. You create a separate "Sorting Machine" (Strategy) to sort Students by grade, or by name. This "Machine" is a function.
Summary
- **Comparator** is a tool/function used to compare two _other_ things. $\rightarrow$ **Functional Interface**.
- **Comparable** is a trait of an object itself. It requires `this` instance to exist. $\rightarrow$ **Regular Interface**.

#### Topic 3: Lambda Expressions
- Syntax Rules
- Scope & "Effectively Final"

**Functional Interface** is a contract with a single requirement (the SAM).
In the past (Java 7), fulfilling this contract required writing a clumsy **Anonymous Inner Class**. It was like writing a formal letter just to say "Hello."

Lambda Expressions = **shorthand notation**. It strips away the boilerplate code (the class definition, the method name, the overrides) and leaves only the logic.
A Lambda Expression = essentially an **anonymous method**. It provides the **implementation for the SAM** of a Functional Interface.
* A Lambda Expression in Java is **context-dependent**. It does not know what it is until you assign it to something.
* A Lambda is just a **recipe**. To get the result, you must **cook** (execute) it by passing data into it.

**Java 7: Anonymous Inner Class (The "noisy" way)** 
```Java
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running old school...");
    }
};
```
**Java 8: Lambda Expression (The "clean" way)**
```Java
// The compiler knows 'run()' is the only method, so we skip the name.
Runnable r2 = () -> System.out.println("Running new school...");
```

Here is a complete, runnable code example using a custom interface named `MathOperation`.
This example demonstrates how we moved from 5 lines of "boilerplate" code to a single line of logic.

**The Custom Functional Interface** 
```Java
@FunctionalInterface interface MathOperation { 
	int operate(int a, int b);
}
```
**Implementation (Before vs. After)**
```Java
public class EvolutionDemo {

    public static void main(String[] args) {
        // ==========================================
        // BEFORE JAVA 8: Anonymous Inner Class
        // ==========================================
        // We have to "speak" the full syntax: new Interface() { @Override ... }
        MathOperation addition7 = new MathOperation() {
            @Override
            public int operate(int a, int b) {
                return a + b;
            }
        };
        // Usage
        System.out.println("Java 7 Result: " + addition7.operate(10, 5));
        // ==========================================
        // AFTER JAVA 8: Lambda Expression
        // ==========================================
        // We focus ONLY on the input (a, b) and the output (a + b)
        MathOperation addition8 = (a, b) -> a + b;
        // Usage
        System.out.println("Java 8 Result: " + addition8.operate(10, 5));
        
        // ==========================================
        // PROOF THEY ARE THE SAME
        // ==========================================
        // We can pass both to a method expecting the interface
        execute(100, 50, addition7); // Works
        execute(100, 50, addition8); // Works
    }

    // A helper method that accepts our Custom Interface
    // Higher-Order Functions (HOF)
    public static void execute(int x, int y, MathOperation op) {
        System.out.println("Execution result: " + op.operate(x, y));
    }
}
```
Key Differences Observed
1. **The Name:** In Java 7, we had to explicitly write `new MathOperation()`. In Java 8, the compiler infers that `(a, b)` belongs to `MathOperation` because we assigned it to a variable of that type.    
2. **The Method Signature:** In Java 7, we had to write `@Override public int operate...`. In Java 8, the compiler knows the interface only has one method (`operate`), so it maps the Lambda `{body}` directly to it.

##### Syntax Rules
The syntax relies heavily on **Type Inference**. Since the interface only has _one_ abstract method, the compiler can guess the types.

**Standard Syntax:** `(parameters) -> { body }`

**Simplification Rules:**

| **Scenario**       | **Code Example**                      | **Rule Applied**                                        |
| ------------------ | ------------------------------------- | ------------------------------------------------------- |
| **Full Syntax**    | `(int x, int y) -> { return x + y; }` | explicit types, braces, return keyword.                 |
| **Type Inference** | `(x, y) -> { return x + y; }`         | Removed `int`. Compiler guesses based on Interface.     |
| **One Liner**      | `(x, y) -> x + y`                     | Removed `{}` & `return`. Value returned automatically.  |
| **Single Param**   | `x -> x * 2`                          | Removed `()` around `x`. Only allowed for single param. |

##### Scope & "Effectively Final"
Lambdas can access variables from the outer scope (like the method they are written in), but there is a catch: the variable must be **effectively final**. Because ==Lambdas capture values, not variables==. Java demands the value remains constant to avoid concurrency issues.
```Java
int factor = 10; // This variable cannot be changed later!
Converter c = (input) -> input * factor; // OK
// factor = 20; // ERROR! If you uncomment this, the Lambda above breaks.
```


*How to implement a lambda with varargs ?* 
Java lambdas themselves do not directly support varargs in their parameter list in the same way regular methods do.

However, you can achieve similar functionality by defining a functional interface whose single abstract method accepts an array as a parameter. Since varargs are essentially syntactic sugar for arrays, you can then pass an array to a lambda that implements this interface.

```Java
import java.util.function.Consumer;

public class LambdaVarargs {

    // Functional interface that accepts an array
    interface ArrayConsumer<T> {
        void accept(T[] array);
    }

    public static void main(String[] args) {
        // Using a lambda with the custom functional interface
        ArrayConsumer<String> stringProcessor = (strings) -> {
            for (String s : strings) {
                System.out.println(s);
            }
        };

        // Calling the lambda with an array
        stringProcessor.accept(new String[]{"apple", "banana", "cherry"});

        // You can also use a pre-existing functional interface like Consumer<T[]>
        Consumer<String[]> anotherStringProcessor = (strings) -> {
            for (String s : strings) {
                System.out.println("Another: " + s);
            }
        };

        anotherStringProcessor.accept(new String[]{"dog", "cat", "bird"});
    }
}
```


#### Topic 4:  Built-in Functional Interfaces 🛠️
In the example above, we had to manually create `MathOperation` just to add two numbers.

If you had to create a new interface for _every_ small action (printing strings, checking booleans, creating objects), your project would be cluttered with hundreds of tiny interface files.

Java 8 (`java.util.function`) provides a set of generic interfaces (`Function`, `Predicate`, `Consumer`, `Supplier`) that cover 90% of use cases, so you almost **never** have to write a custom interface like `MathOperation` again. 
Interfaces define logic, but to **compose** complex logic FIs further provides **Combinators** .

| **Interface** | **I/p** | **O/p**   | **SAM Name** | **Mnemonic**                      |
| ------------- | ------- | --------- | ------------ | --------------------------------- |
| **Predicate** | `T`     | `boolean` | `test`       | **The Bouncer** (Yes/No)          |
| **Function**  | `T`     | `R`       | `apply`      | **The Machine** (Input -> Output) |
| **Consumer**  | `T`     | `void`    | `accept`     | **The Black Hole** (Eats data)    |
| **Supplier**  | `None`  | `T`       | `get`        | **The Factory** (Produces data)   |

| **Interface** | **Primary (SAM)** | **Chaining (Default)** | **Static Utility** |
| ------------- | ----------------- | ---------------------- | ------------------ |
| **Predicate** | `test(T)`         | `and`, `or`, `negate`  | `isEqual`, `not    |
| **Function**  | `apply(T)`        | `andThen`, `compose`   | `identity`         |
| **Consumer**  | `accept(T)`       | `andThen`              | -                  |
| **Supplier**  | `get()`           | -                      | -                  |
##### 1. Predicate `<T>` (The Boolean Logic Gate)
- **Role:** The **Decision Maker** (Boolean Tester).
- **SAM:** `boolean test(T t)`
- **Concept:** Takes an input, returns `true`/`false`.
- **Use Case:** ==Filtering streams==, conditional checks.
```Java
Predicate<String> isLong = str -> str.length() > 5;
System.out.println(isLong.test("Hello")); // false
```
**`default`** methods : 
- **`and(Predicate other)`**: Logical AND (Short-circuiting).
- **`or(Predicate other)`**: Logical OR.
- **`negate()`**: Logical NOT (!).
**`static`** methods : 
- **`isEqual(Object targetRef)`** : returns a predicate that tests if two arguments are equal using `Objects.equals()`.
- **`not(Predicate target)` (Java 11+)**: A static method for negation, useful with method references.
```Java
Predicate<String> startsWithJ = str -> str.startsWith("J");
Predicate<String> isLong = str -> str.length() > 5;

// Chaining: (Starts with J) AND (Length > 5)
Predicate<String> complexCheck = startsWithJ.and(isLong);

System.out.println(complexCheck.test("Javascript")); // true
System.out.println(complexCheck.negate().test("Javascript"));// false (The NOT version)
```
##### 2. Function `<T, R>` (The Transformation Pipeline)
- **Role:** The **Transformer**.
- **SAM:** `R apply(T t)`
- **Concept:** Takes type `T`, transforms it, returns type `R`.
- **Use Case:** ==Mapping data (e.g., String to Integer, Object to JSON)==.
```Java
Function<String, Integer> stringToLength = str -> str.length();
System.out.println(stringToLength.apply("Java")); // 4
```
**`default`** methods : 
- **`.andThen(Function after)`**: Run _this_, then run _after_. (Most common). 
- **`.compose(Function before)`**: - Run _before_, then run _this_. (Reverse order). Returns a composed function that first applies the before function to its input, and then applies this function to the result
**`static`** methods
- **`Function.identity()`** : Returns the input as is (useful in Streams). 
```Java
Function<String, String> trim = str -> str.trim();
Function<String, String> toUpper = str -> str.toUpperCase();

// Pipeline: Trim first -> Then UpperCase
Function<String, String> cleanString = trim.andThen(toUpper);

System.out.println(cleanString.apply("  hello  ")); // "HELLO"
```
##### 3. Consumer `<T>` (The Chain of Events)
- **Role:** The **End User** (Void).
- **SAM:** `void accept(T t)`
- **Concept:** Takes an input, ==does something with it==, returns **nothing**.
- **Use Case:**  executing side effects (==printing==, ==saving== (to DB), logging) , ==sending an email==.
```Java
Consumer<String> printer = str -> System.out.println("Processing: " + str);
printer.accept("Data Payload"); // Prints: Processing: Data Payload
```
**`default`** methods : 
- **`.andThen(Consumer after)`**: Run _this_, then run _after_. (Most common).

```Java
Consumer<String> saveToDb = str -> System.out.println("Saving: " + str);
Consumer<String> logAudit = str -> System.out.println("Logging: " + str);

// Pipeline: Save -> Then Log
Consumer<String> workflow = saveToDb.andThen(logAudit);

workflow.accept("User123"); 
// Output:
// Saving: User123
// Logging: User123
```
##### 4. Supplier `<T>` (The Generator)
- **Role:** The **Factory** (Provider).
- **SAM:** `T get()`
- **Concept:** Takes **no input**, returns a result.
- **Use Case:** Generating random IDs, fetching current time, produce data lazily.
```Java
Supplier<Double> randomizer = () -> Math.random();
System.out.println(randomizer.get()); // Prints random number
```

* **Chaining Methods** : None. (Since it takes no input, it cannot be chained in the standard way).
---------------


#### Topic 5:  Method References (`::`)

MR shows how to replace the Lambda syntax with **direct references** to these *existing methods* for cleaner code.
Method References are **syntactic sugar**. They allow you to refer to a method without executing it.

When a Lambda Expression does nothing but **call an existing method**, you can replace the Lambda with a direct reference to that method by name.

Why use them?
- **Readability:** `String::toUpperCase` is easier to read than `s -> s.toUpperCase()`.
- **Reusability:** It encourages using existing methods rather than writing throwaway logic.

**The Syntax:** `ClassOrObjectName::methodName`

The 4 Types of Method References

| **Type**                | **Lambda Syntax**                    | **Method Ref Syntax** | **Logic**                                            |
| ----------------------- | ------------------------------------ | --------------------- | ---------------------------------------------------- |
| **1. Static Method**    | `(args) -> Class.staticMethod(args)` | `Class::staticMethod` | Pass arguments to static method.                     |
| **2. Constructor**      | `(args) -> new Class(args)`          | `Class::new`          | Pass arguments to constructor.                       |
| **3. Specific Object**  | `(args) -> myObj.method(args)`       | `myObj::method`       | Call method on an **existing** external instance.    |
| **4. Arbitrary Object** | `(arg0, rest) -> arg0.method(rest)`  | `Class::method`       | **Tricky:** The _first_ argument becomes the target. |
**1. Static Method**
Syntax: `ClassName::staticMethodName` 
```Java
Function<String, Integer> lambda = s -> Integer.parseInt(s);// Lambda
Function<String, Integer> ref = Integer::parseInt;// Method Ref
```

**2. Constructor Reference**
Syntax: `ClassName::new`
```Java
Supplier<List<String>> lambda = () -> new ArrayList<>();// Lambda
Supplier<List<String>> ref = ArrayList::new;// Method Ref
```

**3. Instance Method on Specific Object**_The object (`System.out`) already exists outside the lambda._
Syntax: `objectName::instanceMethodName`
```Java
Consumer<String> lambda = str -> System.out.println(str);// Lambda
Consumer<String> ref = System.out.println; // Method Ref
``` 

**4. Instance Method on Arbitrary Object (The Interview Trap)**_This looks like a static call, but it isn't. You are calling the method on the **input** itself._
Syntax: `ClassName::instanceMethodName` 
```Java
Function<String, String> lambda = str -> str.toUpperCase(); // Lambda
Function<String, String> ref = String::toUpperCase; // Method Ref
``` 

> **Note:** The compiler sees `String::toUpperCase`. It knows `toUpperCase` is not static. Therefore, it assumes the **first argument** passed to the function is the object to call the method on.

| **Type**                                  | **Lambda Syntax**            | **Method Reference**  | **Description**                                        |
| ----------------------------------------- | ---------------------------- | --------------------- | ------------------------------------------------------ |
| **1. Static Method**                      | `x -> Math.sqrt(x)`          | `Math::sqrt`          | Refers to a static method of a class.                  |
| **2. Instance Method (Specific Object)**  | `x -> System.out.println(x)` | `System.out::println` | Calls a method on an **existing** object (like `out`). |
| **3. Instance Method (Arbitrary Object)** | `s -> s.toUpperCase()`       | `String::toUpperCase` | Calls a method on the **input argument** itself.       |
| **4. Constructor**                        | `() -> new ArrayList()`      | `ArrayList::new`      | Creates a new object.                                  |
#### 🔍 Deep Dive: Type 3 is Tricky!
Type 2 and Type 3 often confuse learners.
- **Type 2 (Specific Object):** You are calling a method on an external object (like `System.out` or a `dbService`). The lambda argument is passed _into_ that method.
    - _Lambda:_ `x -> myPrinter.print(x)`
    - _Ref:_ `myPrinter::print`
- **Type 3 (Arbitrary Object):** You are calling a method **on the lambda argument itself**.
    - _Lambda:_ `(String s) -> s.toUpperCase()`
    - _Ref:_ `String::toUpperCase`
    - _Logic:_ "For every String `s`, call `toUpperCase()` on `s`."

```Java
import java.util.*;
import java.util.function.*;

public class MethodRefDemo {
    public static void main(String[] args) {

        // 1. Static Method Ref
        // Lambda: (a, b) -> Math.max(a, b)
        BiFunction<Integer, Integer, Integer> maxFinder = Math::max;
        System.out.println(maxFinder.apply(10, 20)); // 20

        // 2. Instance Method (Specific Object)
        // Lambda: str -> System.out.println(str)
        Consumer<String> printer = System.out::println;
        printer.accept("Hello Specific Object!");

        // 3. Instance Method (Arbitrary Object)
        // Lambda: str -> str.toLowerCase()
        Function<String, String> lower = String::toLowerCase;
        System.out.println(lower.apply("JAVA")); // java

        // 4. Constructor Reference
        // Lambda: () -> new ArrayList<>()
        Supplier<List<String>> listMaker = ArrayList::new;
        List<String> myList = listMaker.get(); // Creates new empty list
    }
}
```

#### Topic 6:  Streams API  (Processing, lazily)
We now have the **Syntax** (Lambdas), the **Vocabulary** (Interfaces), and the **Shorthand** (Method References). 

But so far, we have only processed data one item at a time. The real power of Java 8 is taking these tools and applying them to **collections of data** concurrently and efficiently.  

**Concept:** A Stream is a sequence of elements supporting sequential and parallel aggregate operations.  
- **Collections (List, Set):** Focus on **Storage** (Data structures).  
- **Streams:** Focus on **Computation** (Processing).  

Think of a Stream as an **Assembly Line**. The raw materials (**data**) come from the tank (Collection), move through various stations (**processing**), and finally get packaged (**result**). 
##### 1. The 3-Stage Pipeline 
Every Stream has three distinct parts. If you miss the last part, nothing happens.
1. **Source:** Where the data comes from (`List`, `Set`, `Array`).
2. **Intermediate Operations:** Steps that transform the stream (`filter`, `map`, `sorted`).
    - _Key Characteristic:_ These are **Lazy**. They don't run immediately. They just setup instructions.
3. **Terminal Operation:** The trigger (`collect`, `forEach`, `count`).
    - _Key Characteristic:_ These are **Eager**. They start the pipeline, pull data through, and close the stream.

| Stream         |                           | Intermediate Operations | Terminal Operations |
| -------------- | ------------------------- | ----------------------- | ------------------- |
| **Values**     | `Stream.of("A", "B")`     |                         |                     |
| **Collection** | `list.stream()`           | filter()                | .count()            |
| **Array**      | `Arrays.stream(arr)`      | map()                   | .sum()              |
| **String**     | `str.chars()`             | skip(n)                 | .average()          |
| **Primitives** | `IntStream.range(0, 10)`  | limit(n)                | max()               |
| **Generator**  | `Stream.iterate/generate` | distinct()              | min()               |
| **File**       | `Files.lines(path)`       | sorted()                | findFirst()         |
|                |                           |                         | findAny()           |
|                |                           |                         | anyMatch()          |
|                |                           |                         | allMatch()          |
|                |                           |                         | noneMatch()         |
|                |                           |                         | reduce()            |
|                |                           |                         | Collect             |

![[Java streams 00.png]]
![[Java Streams 01.png]]

###### **Creating Stream**
1. From `java.util.Collection` (List, Set, Queue), Not ~~`java.util.Map`~~ 
	1. **Sequential Stream:** `collection.stream()`
	2. **Parallel Stream:** `collection.parallelStream()`
2. **From Arrays** 
	1. Static Method: `Arrays.stream(T[] array)`
	2. Specific Range: `Arrays.stream(array, startInclusive, endExclusive)`
3. **From Static Factory Methods**
	1. **Fixed Values:** `Stream.of(T... values)`
	2. **Empty Stream:** `Stream.empty()` (Useful for avoiding null returns)
	3. **Nullable Single Value:** `Stream.ofNullable(T t)` (Java 9+)
4. **From Files (I/O)** : This is powerful for processing massive log files line-by-line without loading the whole file into memory (Lazy Loading). 
	1. **Lines:** `Files.lines(Path path)` 
		1. `Stream<String> lines = Files.lines(Paths.get("data.txt"));`
	2. **Directory Walk:** `Files.walk(Path path)` (Recursively lists files)
	3. **File Find:** `Files.find(Path, maxDepth, BiPredicate)`
5. **From Strings** : Strings are technically sequences of characters. 
	1. **Chars:** `str.chars()` (Returns `IntStream` of char codes) 
	2. **Regex Split:** `Pattern.compile(",").splitAsStream("a,b,c")` 
6. **Infinite Streams (Generators)** : Used to generate data on the fly. **Warning:** You must use `.limit()` or these will run forever.
	1. **Iterate:** `Stream.iterate(seed, UnaryOperator)` (Like a for-loop: i, i+1, i+2...)
		1. `Stream<Integer> evens = Stream.iterate(0, n -> n + 2).limit(10);`
		2. `// 0, 2, 4, 6...`
	2. **Generate:** `Stream.generate(Supplier)` (Constant or Random values)
		1. `Stream<Double> randoms = Stream.generate(Math::random).limit(5);`
7. **Primitive Streams** (Optimized) : Avoids the overhead of boxing (`Integer` wrapper objects).
	1. **Range:** `IntStream.range(start, endExclusive)`
	2. **Range Closed:** `IntStream.rangeClosed(start, endInclusive)`
	3. **Random:** `new Random().ints()` 
8. **The Builder Pattern** : `Stream.Builder<T>`
	1. **Method:** `Stream.builder()`
	2. `Stream.Builder<String> builder = Stream.builder().add("One").add("Two");`
	3. `if (someCondition) builder.add("Two");`
	4. `Stream<String> stream = builder.build();`

| **Source**     | **Method**                | **Use Case**              |
| -------------- | ------------------------- | ------------------------- |
| **Collection** | `list.stream()`           | Standard data processing. |
| **Array**      | `Arrays.stream(arr)`      | Processing legacy arrays. |
| **Values**     | `Stream.of("A", "B")`     | Quick tests / Fixed data. |
| **File**       | `Files.lines(path)`       | Large text files (Logs).  |
| **String**     | `str.chars()`             | Character manipulation.   |
| **Generator**  | `Stream.iterate/generate` | Sequences / Random data.  |
| **Primitives** | `IntStream.range(0, 10)`  | For loops replacement.    |
###### **Intermediary Operations**
We can group them into **<u>four logical categories</u>** based on **what they do to the data**  
1. Filtering ᯤ (Selecting Elements)
2. Slicing 🔪 (Resizing the Stream) 
3. Mapping➡️ (**Transforming** Elements)
4. Sorting & Peeking 🫣

plus <u>one technical category</u> that is critical for performance interviews
1. Stateless Operations
2. Stateful Operations

**1. Filtering (Selecting Elements)** : 
These operations decide which elements stay and which get thrown out. The type of the stream (`Stream<T>`) remains the same.

| **Operation**           | **Description**                                    | **Type**       |
| ----------------------- | -------------------------------------------------- | -------------- |
| **`filter(Predicate)`** | Keeps elements where the Predicate returns `true`. | Stateless      |
| **`distinct()`**        | Removes duplicates (uses `.equals()`).             | ***Stateful*** |
***Stateful:*** `distinct` is expensive as it remembers _all_ previous elt to know if the current one is a duplicate.

**2. Slicing (Resizing the Stream)**  : 
These operations change the _size_ of the stream based on position or count, rather than content.

| **Operation**       | **Description**                                 | **Type** |
| ------------------- | ----------------------------------------------- | -------- |
| **`limit(long n)`** | Truncates the stream to the first `n` elements. | Stateful |
| **`skip(long n)`**  | Discards the first `n` elements.                | Stateful |

```Java
// Page 2 of a search result (Items 11-20)
stream.skip(10).limit(10).forEach(...);
```

**3. Mapping (Transforming Elements)**
These operations change the **type** or **structure** of the data. `Stream<String>` might become `Stream<Integer>`.

| **Operation**                 | **Description**                                                          | **Type**  |
| ----------------------------- | ------------------------------------------------------------------------ | --------- |
| **`map(Function)`**           | One-to-One transformation. Input `T` -> Output `R`.                      | Stateless |
| **`flatMap(Function)`**       | One-to-Many flattening. Input `T` -> Output `Stream<R>`.                 | Stateless |
| **`mapToInt`, `mapToDouble`** | Special versions that return primitive streams (avoids boxing overhead). | Stateless |
**The `flatMap` Example (Crucial):** Use `flatMap` when each item in your stream contains _another_ list, and you want to process all the inner items as a single flat stream.
```Java
List<List<String>> books = Arrays.asList(
    Arrays.asList("Java", "8"),
    Arrays.asList("Streams", "Option")
);

// .map() would create Stream<List<String>> (Stream of Lists)
// .flatMap() creates Stream<String> (Stream of Words)
books.stream()
     .flatMap(list -> list.stream()) 
     .forEach(System.out::println); 
// Output: Java, 8, Streams, Option (All in one level)
```
**4. Sorting & Peeking**
Miscellaneous operations for ordering and debugging.

| **Operation**            | **Description**                                                                        | **Type**     |
| ------------------------ | -------------------------------------------------------------------------------------- | ------------ |
| **`sorted()`**           | Sorts using natural order (Comparable).                                                | **Stateful** |
| **`sorted(Comparator)`** | Sorts using a custom Comparator.                                                       | **Stateful** |
| **`peek(Consumer)`**     | Performs an action (like printing) without modifying the stream. Purely for debugging. | Stateless    |
**Warning on `sorted`:** This is the most expensive operation. To sort, the stream must buffer **all** elements (it can't output the first element until it has seen the last one). Avoid using `sorted()` on infinite streams!

**Grouping: Stateful vs. Stateless**
If you are asked about Parallel Streams in an interview, this is the most important grouping.
1. **Stateless Operations:** (`filter`, `map`, `peek`)
    - No dependency on other elements.
    - _Example:_ Checking if "Java" starts with "J" doesn't require knowing what the previous word was.
    - **Performance:** Excellent in parallel.
2. **Stateful Operations:** (`distinct`, `sorted`, `limit`, `skip`)
    - Need to know information about other elements.
    - _Example:_ To know if "Java" is `distinct`, I must compare it to every word seen before.
    - **Performance:** Can kill performance in parallel pipelines (bottleneck).

We have covered the **Source** (List, Files) and the **Intermediary** (Filter, Map).
But remember, none of this code actually runs until you hit the **Terminal Operation**.
- If you want to count them: `.count()`
- If you want to return a boolean: `.anyMatch()`
- **Most importantly:** If you want to turn the Stream _back_ into a List or Map: `.collect()`

###### **Terminal Operation 🔥🔑**
Terminal operations are the "**Ignition Key**" of the stream. Until you invoke one of these, your stream does absolutely nothing (lazy evaluation).
- all methods of `java.util.stream.Stream` interface which do not return Stream< T>

We can group them into **5 Categories** based on the **Result Type** they produce.
**1. Matching (Result: `boolean`)**
Used to check if elements satisfy a condition.
- **Key Feature:** These are **Short-Circuiting**. If `anyMatch` finds a `true`, it stops immediately (doesn't check the rest).

| **Operation**              | **Description**                                     |
| -------------------------- | --------------------------------------------------- |
| **`anyMatch(Predicate)`**  | Returns `true` if **at least one** element matches. |
| **`allMatch(Predicate)`**  | Returns `true` if **every single** element matches. |
| **`noneMatch(Predicate)`** | Returns `true` if **zero** elements match.          |

```Java
boolean hasJava = list.stream().anyMatch(s -> s.contains("Java"));
```

**2. Finding (Result: `Optional<T>`)**
Used to retrieve a specific element.
- **Key Feature:** Also **Short-Circuiting**. They return as soon as the element is found.

| **Operation**     | **Description**                                   | **Note**                        |
| ----------------- | ------------------------------------------------- | ------------------------------- |
| **`findFirst()`** | Returns the very first element (Deterministic).   | Used in ordered streams.        |
| **`findAny()`**   | Returns _any_ element found (*Nondeterministic*). | Faster in **Parallel** streams. |
```Java
// Returns Optional<String> because the list might be empty
Optional<String> first = list.stream().findFirst(); 
``` 

**3. Reduction (Result: `Single Value`)**
Used to fold the entire stream into one value (e.g., a sum, a max, or a concatenated string).

| **Operation**                | **Description**                                    |
| ---------------------------- | -------------------------------------------------- |
| **`count()`**                | Returns the number of elements (`long`).           |
| **`min(Comparator)`**        | Returns the smallest element (`Optional`).         |
| **`max(Comparator)`**        | Returns the largest element (`Optional`).          |
| **`reduce(BinaryOperator)`** | The general accumulator (e.g., `(a, b) -> a + b`). |
```Java
int sum = IntStream.of(1, 2, 3).sum(); // Special reduction for primitive streams
```
**4. Collection (Result: `Collection/Map`)**
Used to package the stream elements back into a data structure. This is formally called **Mutable Reduction**.

|**Operation**|**Description**|
|---|---|
|**`collect(Collector)`**|The most powerful operation. Converts stream to List, Set, Map, or String.|
|**`toArray()`**|Converts stream to a raw array `Object[]`.|
```Java
List<String> validNames = stream.collect(Collectors.toList());
```
**5. Iteration (Result: `void`)**
Used for side effects (printing, logging, modifying external state).

| **Operation**                  | **Description**                                                        |
| ------------------------------ | ---------------------------------------------------------------------- |
| **`forEach(Consumer)`**        | Process each element. Order is **not guaranteed** in parallel streams. |
| **`forEachOrdered(Consumer)`** | Process each element. Guarantees order even in parallel (slower).      |
**⚠️ Common Interview Trap: `reduce` vs `collect`**
Both turn a stream into a single result, but they work differently.
- **`reduce` (Immutable):** Creates a _new_ value at every step.
    - _Analogy:_ `String` concatenation (creates new String every time).
    - _Use for:_ Summing numbers, finding max.
- **`collect` (Mutable):** Updates an _existing_ container.
    - _Analogy:_ `StringBuilder` (appends to the same buffer).
    - _Use for:_ Building Lists, Maps, Sets.

------------

##### 2. Code Example: The Pipeline in Action
```Java
import java.util.*;
import java.util.stream.*;

public class StreamDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alex", "Bob", "Chad", "David", "Alan");

        // Goal: Find names starting with 'A', convert to uppercase, & print them.
        names.stream()                          // 1. Source
             .filter(nm -> nm.startsWith("A"))  // 2. Intermediate (Predicate)
             .map(String::toUpperCase)          // 2. Intermediate (Function)
             .sorted()                          // 2. Intermediate
             .forEach(System.out::println);     // 3. Terminal (Consumer)
    }
    // Output : ALEX, ALAN
}
```
##### 3. The "Lazy Evaluation" (Crucial Interview Concept)
Streams are efficient because they are lazy. If you define a massive pipeline but never call a Terminal Operation, Java does **zero work**.

```Java
Stream<String> stream = Stream.of("one", "two", "three")
    .filter(s -> {
        System.out.println("Filtering: " + s); // This will NOT print yet
        return true;
    });

System.out.println("Pipeline defined."); 
// Result so far: Only "Pipeline defined." is printed. "Filtering..." has not run.

stream.forEach(s -> {}); 
// NOW it runs: 
// Filtering: one
// Filtering: two...
```
##### 4. Short-Circuiting
Because of laziness, Streams can optimize. If you search for the "first" match, the stream stops processing as soon as it finds it. It doesn't process the whole list.

```Java
Stream.of("A", "B", "C", "D")
      .map(s -> {
          System.out.println("Mapping: " + s);
          return s;
      })
      .findFirst(); 
// Output: "Mapping: A" -> STOP. 
// It never touches B, C, or D. A loop would have processed them all unless manually broken.
```

**Streams are "One-Time Use"**
Once you call a terminal operation, the stream is **closed**. You cannot reuse it.
```Java
Stream<String> s = Stream.of("A", "B");
s.forEach(System.out::println); // OK

s.forEach(System.out::println); // EXCEPTION! java.lang.IllegalStateExcept
```
![[Java streams 02.png]]
![[streams a.png]]
![[streams b.png]]
![[Java streams 03.png]]

![[Java streams 04.png]]

![[stream d.png]]

![[stream c.png]]



Imp points about Streams : 
* Streams are "**One-Time Use**" : Once you call a terminal operation, the stream is **closed**. You cannot reuse it.
* streams offer ***deferred execution*** or *lazy evaluation* i.e. no pipeline operations are processed until a terminal operation is encountered and evaluated.
* streams employ a bottom-up approach, wherein the evaluation begins with the terminal operation and then traces back to intermediate operations.


From "Processing" to "Safety"
In the Short-Circuit example above, we used `.findFirst()`.
- What if the list is empty?
- What does `.findFirst()` return? It can't return `null` (that leads to crashes), and it can't return a String (because there might not be one).
It returns an **Optional**. 
``

#### Topic 7: The Optional Class (Safety, null safety)
(Optional Class) is the container Java 8 uses to hand you a result that *might not exist*, forcing you to handle the "not found" case safely.

`Optional<T>`  is a container object that may or may not contain a non-null value. It was introduced to fix the "Billion Dollar Mistake": the `NullPointerException` (NPE). 

###### 1. The Core Concept
Instead of returning `null` (which leads to crashes if unchecked), a method returns an `Optional`. This forces the calling code to consciously handle the "empty" case.

- **Before Java 8:**
```Java
    User user = findUser("bob");
    if (user != null) { // Easy to forget this check
        System.out.println(user.getName());
    }
    ```
- **With Optional:**
```Java
    Optional<User> userOpt = findUser("bob");
    // The type system FORCES you to "unwrap" the box to get the data.
    userOpt.ifPresent(u -> System.out.println(u.getName()));
    ```
###### 2. Creating an Optional
You rarely create these yourself (unless writing APIs), but you need to know how.

|**Method**|**Description**|**Usage**|
|---|---|---|
|**`Optional.empty()`**|Creates an empty box.|When you have no result to return.|
|**`Optional.of(val)`**|Wraps a non-null value.|**Throws NPE** if `val` is null. Use only when sure.|
|**`Optional.ofNullable(val)`**|Wraps a value _or_ empty.|Safe. If `val` is null, it returns `empty()`.|
###### 3. Unwrapping the Value (The Important Part)
Once you have an `Optional`, how do you get the data out safely?
**❌ The Bad Way (Anti-Pattern)**

```Java
// Calling .get() without checking is just as bad as a NullPointer
String name = opt.get(); // Throws NoSuchElementException if empty!
```
**✅ The Safe Ways**

|**Method**|**Description**|**Code Example**|
|---|---|---|
|**`orElse(T other)`**|Return value if present, otherwise return `other`.|`opt.orElse("Unknown")`|
|**`orElseGet(Supplier)`**|Return value if present, otherwise run Supplier (Lazy).|`opt.orElseGet(() -> db.fetchDefault())`|
|**`orElseThrow(Supplier)`**|Return value if present, otherwise throw exception.|`opt.orElseThrow(() -> new UserNotFoundException())`|
|**`ifPresent(Consumer)`**|If present, run the lambda. Otherwise do nothing.|`opt.ifPresent(System.out::println)`|
###### 4. Advanced: Transformation (`map`)
You can transform the value _inside_ the Optional without opening it. If the Optional is empty, the transformation is skipped.
```Java
Optional<String> name = Optional.of("java");
// Logic: If name exists -> length -> check if > 2
// If name was empty, "result" simply becomes false. No null checks needed!
boolean isLong = name.map(String::length)
                     .filter(len -> len > 2)
                     .isPresent();
```

**⚠️ Interview Trap: `orElse` vs `orElseGet`**
They look similar, but performance differs.
- **`orElse(new heavyObject())`**: The `heavyObject` is created **ALWAYS**, even if the Optional is not empty. (Wasted memory).
- **`orElseGet(() -> new heavyObject())`**: The `heavyObject` is created **ONLY** if the Optional is empty. (Lazy loading) 

###### From "Safety" to "Packaging"
We have now covered the full lifecycle of a stream _computation_:
1. **Source** (Collection)
2. **Process** (Filter/Map)
3. **Result** (Terminal Operation)
4. **Safety** (Optional)

The final missing piece is the most powerful Terminal Operation: **Collectors**. How do you turn a `Stream<Student>` into a `Map<School, List<Student>>`? Or calculate the average age of all users?

#### Topic 8: Collectors
`java.util.stream.Collectors` is a utility class in Java's Stream API that provides static methods for creating various `Collector` implementations. These `Collector` instances are used as arguments to the `Stream.collect()` terminal operation to perform mutable reduction operations on elements of a stream.

Key functionalities of `Collectors`:

- **Accumulating into Collections:** 
    Methods like `toList()`, `toSet()`, `toMap()`, `toCollection()`, `toConcurrentMap()`, `toUnmodifiableList()`, `toUnmodifiableSet()`, and `toUnmodifiableMap()` enable collecting stream elements into various types of collections.
    
- **Aggregation and Summarization:** 
    `counting()`, `summingInt()`, `summingLong()`, `summingDouble()`, `averagingInt()`, `averagingLong()`, `averagingDouble()`, `minBy()`, `maxBy()`, `summarizingInt()`, `summarizingLong()`, and `summarizingDouble()` provide ways to perform common aggregation tasks and generate summary statistics.
    
- **Grouping and Partitioning:** 
    `groupingBy()` and `partitioningBy()` allow for categorizing stream elements into maps based on a classifier function or a predicate, respectively. These methods can also take **downstream collectors** for further processing within each group/partition.
    
- **String Manipulation:** 
    `joining()` concatenates stream elements (typically strings) into a single string, with optional delimiters, prefixes, and suffixes.

- **Transformations:**    
    `mapping()`, `flatMapping()`, `filtering()`, `reducing()`, and `collectingAndThen()` offer ways to transform elements or apply additional operations during the collection process.

-----
This is the most powerful tool in the Stream API.

While `.forEach()` prints data and `.reduce()` summarizes it, **`Collectors`** allows you to perform complex data transformations similar to SQL (grouping, partitioning, and summarizing) directly in Java.

It is a **utility class** full of <u>static methods</u> typically imported via `import static java.util.stream.Collectors.*;` so you can just write `toList()` instead of `Collectors.toList()`.

###### 1. Basic Accumulation (The Essentials)
Getting data out of the stream and <u>into a standard collection</u>.

| **Method**                   | **Description**                                                  | **Example**                                  |
| ---------------------------- | ---------------------------------------------------------------- | -------------------------------------------- |
| **`toList()`**               | Collects into a `List`.                                          | `.collect(Collectors.toList())`              |
| **`toSet()`**                | Collects into a `Set` (Removes duplicates).                      | `.collect(toSet())`                          |
| **`toCollection(Supplier)`** | Collects into a _specific_ type (e.g., `LinkedList`, `TreeSet`). | `.collect(toCollection(TreeSet::new))`       |
| **`joining(delimiter)`**     | Concatenates Strings.                                            | `.map(User::getName).collect(joining(", "))` |
###### 2. The "Group By" (SQL in Java)
This is the most asked Interview Question regarding streams: _"How do you group a list of objects by a property?"_
- used to group elements from a stream based on a **classification function.**
- `groupingBy()` takes a `Function` as an argument, which acts as the classifier.

At its core, `groupingBy` manages a `Map`. It iterates through your stream one item at a time and decides two things:
1. **Which Bucket (Key)?** Defined by the _Classifier_.
2. **What to put there (Value)?** Defined by the _Downstream Collector_.

**Method:** `groupingBy(Function classifier)` It returns a `Map<K, List<V>>`.
**Scenario:** Group a list of `Person` objects by their `City`.
```Java
List<Person> people = Arrays.asList(
    new Person("Alice", "New York"),
    new Person("Bob", "London"),
    new Person("Charlie", "New York")
);
// Result: Map<String, List<Person>>
// { "New York": [Alice, Charlie], "London": [Bob] }
Map<String, List<Person>> byCity = people.stream()
    .collect(groupingBy(Person::getCity));
    
// What groupingBy(Employee::getDept) effectively does: 
Map<String, List<Employee>> map = new HashMap<>(); 
for (Employee e : streamSource) { 
	String key = e.getDept(); // 1. Run Classifier 
	
	
	// 2. Check if bucket exists 
	if (!map.containsKey(key)) { 
		map.put(key, new ArrayList<>()); // Create new bucket (Downstream Supplier) 
	} 
	// 3. Add to bucket (Downstream Accumulator) 
	map.get(key).add(e); 
	//2+3 in one line
	map.computeIfAbsent(key, k -> new ArrayList<>()).add(e);
}
```
----

| Overloaded Versions                              |                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `groupingBy(classifier )`                        | It groups elements by the `classifier` function and collects the values into a `List`.                                                                                                                  |
| `groupingBy(classifier,  downstream)`            | allows you to specify a `downstream` collector to perform a further reduction on the values associated with each key. This could be collecting into a `Set`, counting elements, averaging, summing, etc |
| `groupingBy(classifier, mapFactory, downstream)` | provides even more control by allowing you to specify the type of `Map` to be used for storing the grouped results (e.g., `TreeMap`, `LinkedHashMap`).                                                  |
There are **three main overloaded** versions of `groupingBy()`:
- `groupingBy(Function<? super T, ? extends K> classifier)`: This is the most basic version. It groups elements by the `classifier` function and collects the values into a `List`.
```Java
List<String> words = Arrays.asList("apple", "banana", "apricot", "berry");
Map<Character, List<String>> groupedByFirstLetter = words.stream()
					.collect(Collectors.groupingBy(s -> s.charAt(0)));    
    // Output: {a=[apple, apricot], b=[banana, berry]}
```
- `groupingBy(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream)`: This version allows you to specify a `downstream` collector to perform a further reduction on the values associated with each key. This could be collecting into a `Set`, counting elements, averaging, summing, etc.
```Java
List<String> words = Arrays.asList("apple", "banana", "apricot", "berry");    Map<Character, Long> countByFirstLetter = words.stream()            .collect(Collectors.groupingBy(s -> s.charAt(0), Collectors.counting()));    // Output: {a=2, b=2}
```
- `groupingBy(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream)`: This version provides even more control by allowing you to specify the type of `Map` to be used for storing the grouped results (e.g., `TreeMap`, `LinkedHashMap`).
```Java
    List<String> words = Arrays.asList("apple", "banana", "apricot", "berry");    Map<Character, List<String>> groupedByFirstLetterSorted = words.stream()            .collect(Collectors.groupingBy(s -> s.charAt(0), TreeMap::new, Collectors.toList()));    // Output: {a=[apple, apricot], b=[banana, berry]} (keys are sorted)
```
more on `groupingBy(classifier, downstream)`
The "Downstream Collector" changes what happens inside the bucket.
If you write `groupingBy(Employee::getDept, counting())`:
1. **Supplier:** Instead of `new ArrayList()`, it creates a mutable counter (like `long[] {0}`).
2. **Accumulator:** Instead of `list.add(e)`, it runs `counter[0]++`.

**The Three Moving Parts**
To fully grasp `groupingBy`, you must recognize its three internal arguments. Even if you only type one, the others are defaulted.
1. **Classifier (Key Mapper):**
    - _Function:_ `emp -> emp.getDept()`
    - _Role:_ Decides which "row" in the map this item belongs to.
2. **Map Factory (Optional):**
    - _Default:_ `HashMap::new`
    - _Role:_ Decides what _kind_ of Map holds the groups. Use this to switch to `TreeMap` (sorted keys) or `LinkedHashMap` (preserve order).
3. **Downstream Collector (Value Processor):**
    - _Default:_ `Collectors.toList()`
    - _Role:_ Decides how to aggregate the data _within_ a single bucket.
    - _Common Options:_ `counting()`, `summingInt()`, `mapping()`, `maxBy()`.
**Key Insight: The "Cascade"**
The most powerful feature of `groupingBy` is that it allows **nesting**. The "Value" of the outer map can be the result of _another_ `groupingBy`.
**Example:** Group by Dept, then inside Dept, group by Gender.
- **Outer Map:** `Key = Dept, Value = Inner Map.`
- **Inner Map:** `Key = Gender, Value = List<Employee>`

###### 3. Partitioning (True vs False)
A special case of grouping where there are only two groups: `true` and `false`. 
**Method:** `partitioningBy(Predicate)`
**Scenario:** Split students into "Pass" and "Fail".
```Java
// Result: Map<Boolean, List<Student>>
// { true: [List of passing], false: [List of failing] }
Map<Boolean, List<Student>> results = students.stream()
    .collect(partitioningBy(s -> s.getScore() >= 50));
```
###### 4. Advanced: Downstream Collectors
What if you don't want the _whole_ object in the list? What if you just want the **count** of people in each city?
`groupingBy` accepts a second argument called a **Downstream Collector**.

```Java
// SQL Equivalent: SELECT city, COUNT(*) FROM people GROUP BY city;
Map<String, Long> countByCity = people.stream()
    .collect(groupingBy(
        Person::getCity,       // 1. Classifier (Key)
        counting()             // 2. Downstream (Value)
    ));
// Output: { "New York": 2, "London": 1 }
```
You can chain these endlessly. You could group by City, then map to Names, then join them.

---
###### 5. Statistics (Summarizing)
Instead of running a stream multiple times to get Max, Min, Average, and Sum, you can get them all at once.
```Java
IntSummaryStatistics stats = numbers.stream()
    .collect(summarizingInt(x -> x));

System.out.println(stats.getMax());
System.out.println(stats.getAverage());
System.out.println(stats.getCount());
```

---
###### 6. Converting to Map (Handling Duplicates)
Using `toMap` is tricky. If your stream has duplicate keys, **it throws an Exception** unless you handle it.
```Java
// Scenario: Convert List<Person> to Map<ID, Name>

// ❌ UNSAFE (Crashes if two people have same ID)
Map<Integer, String> map = list.stream()
    .collect(toMap(Person::getId, Person::getName));

// ✅ SAFE (Merge Function handles collisions)
// (oldValue, newValue) -> newValue  means "Keep the latest one"
Map<Integer, String> map = list.stream()
    .collect(toMap(
        Person::getId, 
        Person::getName,
        (existing, replacement) -> existing // Or keep existing
    ));
```



We have now mastered sequential processing. You can filter, map, and group data that is currently in memory.

But in modern microservices (which you mentioned you are interested in), you often have to call 3 different APIs, wait for them, and combine their results. Doing this one by one is slow.

**Topic 9** (CompletableFuture) applies these functional concepts (Lambda, Supplier, Function) to **Asynchronous Programming**, allowing you to run tasks in parallel without blocking the main thread.

![[streams API features.png]]

----------------------

Reduce : [how to reduce a Stream](https://x.com/java/status/2026683666915557829?s=20)

#### Bibliography : 
[Lambda Expressions in Java - Full Simple Tutorial - YouTube](https://youtu.be/tj5sLSFjVj4)