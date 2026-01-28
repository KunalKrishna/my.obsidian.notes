
![[J_Data types.png]]


![[J_class_abstract_interface.png]]

| **Relationship**                       | **Keyword**  |
| -------------------------------------- | ------------ |
| Class $\rightarrow$ Interface          | `implements` |
| Abstract Class $\rightarrow$ Interface | `implements` |
| Interface $\rightarrow$ Interface      | `extends`    |
| Class $\rightarrow$ Class              | `extends`    |
| Class $\rightarrow$ Abstract Class     | `extends`    |
Trick : `non-interface` always ***implements*** `interface(s)` 
x always extends y except when x = non-I and y=I

----
# interfaces : 
- fields (no state)
- methods
- no constructors 
- types
	- FI
	- Marker
- design principles : abstraction, loose-coupling

## fields/ variables : 
* **Mandatory Initialization** : `int MAX_PLAYERS`; // error
* **only constants** (no state) : `public static final` $\rightarrow$ global constants
	* `int MAX_PLAYERS = 4`; $\Rightarrow$ compiles as ==`public static final`== `int MAX_PLAYERS = 4;`
* Reference vs. Content (The "Final" Trap)
	* While the reference is final, if the object itself is mutable, its internals can still be changed. 
```Java
interface Config {
    // The array REFERENCE is constant (cannot point to a new array)
    int[] SETTINGS = { 1, 2, 3 };  // implicitly public static final
    
    void hello(); // implicitly public abstract
}

class Game implements Config {
    void hack() {
        // VALID: Changing the content inside the object
        SETTINGS[0] = 99; 

        // INVALID: Trying to change the reference
        // SETTINGS = new int[] { 5, 6 }; // Error
    }
    @Override
    void hello(){
	    System.out.println("Game implementation hello!");
    }
}
```

cannot have : 
- private field
- protected fields
- instance variables (non-static fields)
- transient/volatile fields

## methods  : 
implicitly `public` 
implicitly `abstract`

- Java 7 : **Implicit modifiers**: `public` and `abstract` | **(`pubic abstract`**) int foo()
- Java 8 : `(public) default`  int foo_def()
	-     `(public)  static`  int foo_def()
- Java 9+ : `private` int foo_pvt () {...//with implementation}

| version                        | Permissible Methods                                               | Declaration                                                  | Remarks                                                            |
| ------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------ |
| Java 1.0 – Java 7 (Strict Era) | `public abstract` only.                                           | `int foo()`                                                  | interfaces were pure "contracts." - no code                        |
| Java 8 (Modern Turn)           | New Method 1: `default` methods<br>New Method 2: `static` methods | `default void log() `<br>`static boolean isNull`             | To support the new **Streams API**  it allowed code in interfaces. |
| Java 9 (Refinement)            | `private` methods                                                 | `// HIDDEN helper method`<br>private void validate() {} <br> | Code reusability & encapsulation _within_ the interface.           |

| **Feature**           | **Java 1-7** | **Java 8** | **Java 9+** |
| --------------------- | ------------ | ---------- | ----------- |
| **`public abstract`** | ✅ Yes        | ✅ Yes      | ✅ Yes       |
| **`public default`**  | ❌ No         | ✅ Yes      | ✅ Yes       |
| **`public static`**   | ❌ No         | ✅ Yes      | ✅ Yes       |
| **`private`**         | ❌ No         | ❌ No       | ✅ Yes       |
| **`private static`**  | ❌ No         | ❌ No       | ✅ Yes       |
**Note on "Protected":** `protected` methods are **still not allowed** in interfaces. Interfaces are meant for public contracts (API) or internal helpers (private); `protected` implies a subclassing relationship that doesn't fit the interface model cleanly.
## Imp Types of Interfaces
- Marker Interfaces (Tagging Interface) $\rightarrow$ Modern Way @Annotations 
- Functional Interface (SAM)
- Sealed Interface (Java 17+)
	- `// Only Admin and Customer are allowed to be "AppUser"s `
	- `public sealed interface AppUser permits Admin, Customer { }`
- Mixin Interface (Pattern)
	- This isn't a formal keyword, but a **design pattern** enabled by Java 8 **Default Methods**. A Mixin allows you to "mix in" capabilities to a class without forcing it to inherit from a specific parent class.
- Constant Interface (The "Anti-Pattern") : pre-Java 5
	- _Better Approach:_ Use a `final class` with a private constructor and `import static`.

|**Type**|**Abstract Methods**|**Keywords/Features**|**Key Use Case**|
|---|---|---|---|
|**Marker**|0|(Empty body)|Metadata / Permission (e.g., `Serializable`)|
|**Functional**|1|`@FunctionalInterface`|Lambda expressions (e.g., `Runnable`)|
|**Normal**|2+|Standard|Defining a full contract (e.g., `List`, `Map`)|
|**Sealed**|Any|`sealed`, `permits`|Restricted hierarchy (e.g., `Shape` -> `Circle`, `Square`)|
|**Mixin**|0 (usually)|`default` methods|Adding behavior to unrelated classes|
### Marker Interfaces : 
Examples : ***Serializable*, *Cloneable*,  java.util.*RandomAccess***, java.util.**EventListener**, java.rmi.**Remote**
*Why is Marker interface necessary*?
- It acts like a "sticker" or a "tag" that you put on a class. It communicates metadata to the **JVM** or a **Compiler**. It tells the system to **treat** the object of this class in **special way**.
- The JVM is very strict; it refuses to perform certain "dangerous" or "complex" operations unless you explicitly opt-in by wearing the correct "tag.
- Skipping this might throw **Runtime Exception** that crashes your application. 

We need them to provide **Run-time Type Information (RTTI)**. They act as a **Gatekeeper**.
Without marker interfaces, the JVM would either:
1. Have to assume **everything** is serializable (Security nightmare).
2. Have to assume **nothing** is serializable (Feature unusable).

In modern Java (Spring Boot, Hibernate, etc.), **Annotations** (like `@Entity`, `@Service`) have largely replaced _custom_ marker interfaces.
- **Old Way:** `implements Serializable`
- **New Way:** `@JsonSerializable` (Conceptually)
However, the built-in ones (`Serializable`, `Cloneable`) are hard-coded into the JVM core, so they aren't going away anytime soon.

| Operation                                                                              | Risk                                                                                                                                                                                                                                                                                        | Checks                                                                        | Throws                                                                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| save an object to a file or send it over a network using Java's built-in Serialization | Converting an object (including its private data) into a stream of bytes is a **security risk**. The JVM defaults to "Secure/Closed" and requires you to explicitly stick the `implements Serializable` label on your class to grant permission.                                            | `object instanceof java.io.Serializable`                                      | `NotSerializableException`                                                                                                                                                                                                                                                                                                          |
| call myObject.clone()                                                                  | Cloning (**memory** copying) is tricky. Java forces you to acknowledge you want this behavior.                                                                                                                                                                                              | checks it inside the **JVM's native C++ code**.   <br><br>java.lang.Cloneable | CloneNotSupportedException                                                                                                                                                                                                                                                                                                          |
| perform a binary search or indexed loop on a List.                                     | it **slows down**. The algorithm switches from a fast index-based access ($O(1)$) to a slow iterator-based access ($O(n)$). If you pass a `LinkedList` (which lacks this marker) to an algorithm expecting `RandomAccess`, performance can degrade significantly without any error message. | `Collections.binarySearch` checks `list instanceof RandomAccess`              | It does **not throw an error**.<br><br>If you pass a collection that does not implement the `RandomAccess` marker interface (like `LinkedList`) to `Collections.binarySearch`, the method **automatically switches strategies**. It will still work and give you the correct result, but it **degrades performance significantly**. |
| java.util.**EventListener**                                                            |                                                                                                                                                                                                                                                                                             |                                                                               |                                                                                                                                                                                                                                                                                                                                     |
| java.rmi.**Remote**                                                                    |                                                                                                                                                                                                                                                                                             |                                                                               |                                                                                                                                                                                                                                                                                                                                     |

### Functional Interfaces
Look another slide : Streams

**Does Interface extend Object class?** 

No, an interface does not extend the `Object` class in Java.
Here's why:
- **Interfaces cannot extend classes:** 
    Interfaces are a separate construct from classes and are designed to define a contract for behavior without providing implementation details. They can only extend other interfaces.
- `Object` is a class:  
    `java.lang.Object` is a class, and all classes in Java implicitly or explicitly extend it, forming the root of the class hierarchy.
- Implicit declaration of `Object` methods: 
    While interfaces do not extend `Object`, they implicitly declare abstract methods corresponding to the public instance methods of `Object` (like `equals()`, `hashCode()`, `toString()`). This allows any class implementing the interface to provide an implementation for these methods, which are ultimately inherited from `Object` by the implementing class.

---------------
# abstract class :



----------
# class
- implements one or more interface(s)
- extends abstract class
- extends only one other class (no diamond problem)

## inner class

In Java, there are **four** distinct ways to define a class inside another class. They differ primarily in how they relate to the **Outer Class instance** and their **Scope** (visibility).
### 1. Member Inner Class (Non-Static)
A class defined at the root level of the Outer class, without `static`.
- **Relationship:** It is tied to a specific **instance** of the Outer class. It cannot exist without the Outer object.
- **Scope:** It has access to **all** members of the Outer class (private, static, everything).
- **Instantiation:** `Outer outer = new Outer(); Outer.Inner inner = outer.new Inner();`
```Java
class Outer {
    private String msg = "Hello";
    
    class Inner {
        void print() {
            // Can access private field of Outer directly
            System.out.println(msg); 
        }
    }
}
```
### 2. Static Nested Class
A class defined at the root level, **with** `static`.
- **Relationship:** It is tied to the **Class** itself, not an instance. It behaves exactly like a normal top-level class, just packaged inside another for logical grouping.
- **Scope:** It can **only** access `static` members of the Outer class. It cannot access instance variables (unless you pass an object instance into it).
- **Instantiation:** `Outer.Nested nested = new Outer.Nested();` (No `outer` instance needed).
```Java
class Outer {
    static String staticMsg = "Hi";
    String instanceMsg = "Hello";

    static class Nested {
        void print() {
            System.out.println(staticMsg); // OK
            // System.out.println(instanceMsg); // COMPILE ERROR
        }
    }
}
```
### 3. Local Inner Class (Method-Local)
A class defined **inside a method** (or an `if` block, `for` loop, etc.).
- **Relationship:** It is temporary. It exists only while that method is executing.
- **Scope:** * Can access members of the Outer class.
    - Can access local variables of the method **only if** they are `final` or **effectively final** (never changed).
- **Instantiation:** Can only be instantiated _inside_ that specific method.
```Java
class Outer {
    void myMethod() {
        int x = 10; // "Effectively final"

        class Local {
            void print() {
                System.out.println(x); // OK
            }
        }
        new Local().print(); // Must use here
    }
}
```
### 4. Anonymous Inner Class
A class without a name, defined and instantiated in a single expression.
- **Relationship:** Usually used to implement an `interface` or extend a class on the fly (e.g., event listeners).
- **Scope:** Same as Local Inner Class (Access to Outer members + effectively final locals).
- **Instantiation:** Happens at the definition site.
```Java
// Implementing an interface on the fly
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Anonymous run");
    }
};
```

### Summary Comparison Table

| **Type**          | **Keyword** | **Needs Outer Instance?** | **Access to Outer Private Fields?** | **Scope Limit**            |
| ----------------- | ----------- | ------------------------- | ----------------------------------- | -------------------------- |
| **Static Nested** | `static`    | **No**                    | **No** (Static only)                | Public/Private rules apply |
| **Member Inner**  | None        | **Yes**                   | **Yes** (All)                       | Inside Outer Class         |
| **Local Inner**   | None        | **Yes**                   | **Yes** (All)                       | **Inside Method Block**    |
| **Anonymous**     | `new`       | **Yes**                   | **Yes** (All)                       | **Definition Point**       |

|**Type**|**Keyword**|**Defined Location**|**Needs Outer Instance?**|**Access to Outer Members?**|
|---|---|---|---|---|
|**Static Nested**|`static`|Inside Class|**No**|Static only|
|**Member Inner**|(none)|Inside Class|**Yes**|All (Static + Instance)|
|**Local Inner**|(none)|Inside Method|**Yes**|All + Final Local Vars|
|**Anonymous**|(none)|Inside Expression|**Yes**|All + Final Local Vars|

In Java, there are **four** distinct ways to define a class inside another class. They differ primarily in how they relate to the **Outer Class instance** and their **Scope** (visibility).

### 1. Member Inner Class (Non-Static)

A class defined at the root level of the Outer class, without `static`.

- **Relationship:** It is tied to a specific **instance** of the Outer class. It cannot exist without the Outer object.
    
- **Scope:** It has access to **all** members of the Outer class (private, static, everything).
    
- **Instantiation:** `Outer outer = new Outer(); Outer.Inner inner = outer.new Inner();`
    

Java

```
class Outer {
    private String msg = "Hello";
    
    class Inner {
        void print() {
            // Can access private field of Outer directly
            System.out.println(msg); 
        }
    }
}
```

### 2. Static Nested Class

A class defined at the root level, **with** `static`.

- **Relationship:** It is tied to the **Class** itself, not an instance. It behaves exactly like a normal top-level class, just packaged inside another for logical grouping.
    
- **Scope:** It can **only** access `static` members of the Outer class. It cannot access instance variables (unless you pass an object instance into it).
    
- **Instantiation:** `Outer.Nested nested = new Outer.Nested();` (No `outer` instance needed).
    

Java

```
class Outer {
    static String staticMsg = "Hi";
    String instanceMsg = "Hello";

    static class Nested {
        void print() {
            System.out.println(staticMsg); // OK
            // System.out.println(instanceMsg); // COMPILE ERROR
        }
    }
}
```

### 3. Local Inner Class (Method-Local)

A class defined **inside a method** (or an `if` block, `for` loop, etc.).

- **Relationship:** It is temporary. It exists only while that method is executing.
    
- **Scope:** * Can access members of the Outer class.
    
    - Can access local variables of the method **only if** they are `final` or **effectively final** (never changed).
        
- **Instantiation:** Can only be instantiated _inside_ that specific method.
    

Java

```
class Outer {
    void myMethod() {
        int x = 10; // "Effectively final"

        class Local {
            void print() {
                System.out.println(x); // OK
            }
        }
        new Local().print(); // Must use here
    }
}
```

### 4. Anonymous Inner Class

A class without a name, defined and instantiated in a single expression.

- **Relationship:** Usually used to implement an `interface` or extend a class on the fly (e.g., event listeners).
    
- **Scope:** Same as Local Inner Class (Access to Outer members + effectively final locals).
    
- **Instantiation:** Happens at the definition site.
    

Java

```
// Implementing an interface on the fly
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Anonymous run");
    }
};
```


---
# Enum


-----
# Record 


