Class loading trips people up because it's invisible at runtime, so understand the _why_ behind each step. 
## ClassLoader Types (BPA)

**One line:** A three-level hierarchy of loaders, each responsible for a different set of classes.

When the JVM needs a class, it doesn't have one loader — it has a chain of them, each trusted with a different territory:
- **Bootstrap ClassLoader** — the root. Loads the core Java classes: `java.lang.*`, `java.util.*` — everything in the core runtime( `java.base`<--Java 9--`rt.jar`).
	- absolute basics (`java.base` = `java.lang`, `java.util` ,`java.io` , `java.nio`, `java.net` ,`java.math` , `java.concurrent` etc.)
	- written in native code (C/C++), part of the JVM itself. Because it's native, in Java code it shows up as `null`. 
- **Platform ClassLoader** (pre Java 9 **Extension CL** ) — loads additional standard modules that sit above the core but ship with the JDK.
	- secondary standard APIs ( `java.sql`, `java.xml` , `java.compiler` , `java.rmi` , `java.scripting` etc.)
- **Application ClassLoader** (**System CL**) — loads _your_ classes from the application classpath (`-cp` / `CLASSPATH`).  loads the code you write.
Read more at [Run-Time Built-in Class Loaders](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html)

```
[lib/modules]
 │
 ├──> java.base module ──────────────> Loaded by: Bootstrap ClassLoader
 │
 ├──> java.sql module ┐
 ├──> java.xml module ├──────────────> Loaded by: Platform ClassLoader
 .     ...            .
 └──> java.rmi module ┘
 
```

>[!note] Every Class object contains a reference to the ClassLoader that defined it.

You can see the chain yourself:
```java
public class LoaderDemo {
    public static void main(String[] args) {
	    // 1. Core class (java.base module)
        // A core class -> Bootstrap, which prints as null
        System.out.println("ArrayList Loader: " + ArrayList.class.getClassLoader());// String/ArrayList/Map (java.util)
        // null
        
		// 2. Secondary platform class (java.sql module)
		System.out.println("SQL Connection Loader: " + Connection.class.getClassLoader());
		// OR,
        // Its parent -> Platform ClassLoader
        System.out.println(LoaderDemo.class.getClassLoader().getParent());
        
        // e.g. jdk.internal.loader.ClassLoaders$PlatformClassLoader@...
		
		// 3. Your custom code
		// Your own class -> Application ClassLoader
        System.out.println("Custom Class Loader: " + LoaderDemo.class.getClassLoader());
        // e.g. jdk.internal.loader.ClassLoaders$AppClassLoader@...
		
    }
    /*
    ArrayList Loader: null 
    SQL Connection Loader: jdk.internal.loader.ClassLoaders$PlatformClassLoader@... 
    Custom Class Loader: jdk.internal.loader.ClassLoaders$AppClassLoader@...
    */
}
```

The key takeaway: `String` prints `null` because Bootstrap (native) loaded it, while your class prints `AppClassLoader`.

_Interview Q: "Which classloader loads `String` vs your own class?"_ → Bootstrap loads `String` (shows as `null`); Application loads yours.

---
## Class Loading Phases (VPR)

**One line:** Loading → Linking (Verify, Prepare, Resolve) → Initialization. A class walks through these once, lazily, the first time it's actively used.
![[ClassLoaderSubsystem.png]]
This is the heart of the module. The JVM doesn't just "read a file" — it runs a class through a defined lifecycle:

**1. Loading** — the ClassLoader finds the `.class` file, reads the bytecode, and creates a `Class` object in memory to represent the type. At this point the JVM knows the class exists but hasn't checked it or run anything.

**2. Linking** — three sub-steps:
- **Verification** — the bytecode verifier checks the file is valid and safe: correct format, no illegal bytecode, no stack overflows/underflows, type-safe operations. This is a security cornerstone — it's why the JVM can safely run bytecode from untrusted sources. Bad bytecode throws `VerifyError` here.
- **Preparation** — the JVM allocates memory for **static** fields and sets them to their **default** values (e.g., `0`, `false`, or `null`) , not your assigned values yet. So `static int count = 42;` becomes `count = 0` during this phase.
- **Resolution** — symbolic references (names like `"java/lang/String"`) are replaced with direct references (actual memory pointers). This can be lazy.

**3. Initialization** — _now_ the JVM runs static initializers: static blocks and static field assignments, top to bottom. This is when `static int count = 42;` actually becomes `42`.

>[!question]- Loading vs Linking : Essential Difference?
>In the **Loading** phase, the JVM simply grabs raw binary data (`.class` bytes) from your disk and drops it into memory. It is just an unverified heap of bytes. 
>The **Linking (VPR)** phase takes those raw bytes and safely hooks them into the living, breathing JVM execution engine. Essentially, it links **your isolated class bytes to the JVM's memory structures, type safety rules, and other dependent classes**.

Example that makes the Preparation-vs-Initialization split visible:

```java
public class Phases {
    static int count = 42;      // Prepare: count=0, Initialize: count=42

    static {                    // runs during Initialization
        System.out.println("Static block running, count = " + count);
    }

    public static void main(String[] args) {
        System.out.println("main: count = " + count);
    }
}
```

Output:
```
Static block running, count = 42
main: count = 42
```

The static block sees `42` (not `0`) because Initialization runs _after_ Preparation has already been assigned.

**Lazy loading:** a class is only fully loaded/initialized on **first active use** — creating an instance, calling a static method, or accessing a static (non-constant) field. Merely declaring a reference variable does _not_ trigger it.

_Interview Q: "What are the class loading phases?"_ → Loading, Linking (Verification/Preparation/Resolution), Initialization — and know that statics default in Preparation but get real values in Initialization.

---
>[!question]- 4 major principles followed by the 3 types of Class Loaders
>The 3 types of class loaders (connected with inheritance property) and they follow 4 major principles.
> **1. Visibility Principle** : A Child Class Loader can see the class loaded by Parent Class Loader, but a Parent Class Loader cannot find the class loaded by Child Class Loader. 
> **2. Uniqueness Principle** : A class loaded by parent should not be loaded by Child Class Loader again and ensure that duplicate class loading does not occur.
> **3. Delegation Hierarchy Principle** : In order to satisfy above 2 principles, JVM follows a hierarchy of delegation to choose the class loader for each class loading request. Here, starting from the lowest child level, Application Class Loader delegates the received class loading request to Platform Class Loader and then Platform Class Loader delegates the request to Bootstrap Class Loader. If the requested class found in Bootstrap path, the class is loaded. Otherwise the request again transfers back to Platform Class Loader level to find the class from Platform path or custom-specified path. If it also fails, the request comes back to Application Class Loader to find the class from System class path and if Application Class Loader also fails to load the requested class, then we get the run time exception — `java.lang.ClassNotFoundException` .
> **4. No Unloading Principle** : Even though a Class Loader can load a class, it cannot unload a loaded class. Instead, the current class loader can be deleted, and a new class loader can be created.
## Parent Delegation Model

**One line:** Before a loader loads a class itself, it asks its parent first — all the way up to Bootstrap. It only loads the class if no ancestor could.

This is the single most-asked class-loading question, so understand the mechanism _and_ the motivation.

**How it works** — when the Application ClassLoader is asked to load a class, it doesn't immediately search the classpath. It delegates _up_:

```
Application  ──asks──▶  Platform  ──asks──▶  Bootstrap
                                                  │
                       (Bootstrap tries first)    │
   ◀──────────── found? return it ◀───────────────┘
   
If Bootstrap can't find it, Platform tries.
If Platform can't, Application tries (searches classpath).
If nobody can, ClassNotFoundException.
```

So the search goes **up first, then back down**. The topmost loader that _can_ load the class wins.

**Why it exists — two reasons:**

1. **Security.** Suppose an attacker ships a class named `java.lang.String`. Because of delegation, the request goes up to Bootstrap, which loads the _real_ `String` from the core library first. The malicious version is never reached. Core classes can't be spoofed.
2. **Uniqueness / no duplication.** A class's identity in the JVM is `(fully-qualified name + defining classloader)`. Delegation ensures core classes are loaded once by one loader, so you don't get two incompatible `java.lang.Object` types floating around.

You can observe delegation indirectly — a core class always ends up on Bootstrap (`null`) no matter who was originally asked:

```java
// Even though we ask via the app classpath context,
// Integer is delegated up to Bootstrap:
System.out.println(Integer.class.getClassLoader()); // null
```

_Interview Q: "Why parent delegation?"_ → Security (core classes can't be overridden by user code) and uniqueness (each core class loaded once). Mechanism: delegate up before loading yourself.

![[Pasted image 20260725001526.png]]
		Java Class Loaders — Delegation Hierarchy Principle

![[Pasted image 20260725005644.png]]		

![[Parent Delegation Model2.drawio 1.png]]

---

Java uses a **Parents Delegation Model**. When the Application ClassLoader needs to load a class (for example, your `LoaderDemo` class), it does not look for it immediately. Instead, it delegates the request upward:

```
[Application ClassLoader] ──(Delegates Up)──> [Platform ClassLoader] ──(Delegates Up)──> [Bootstrap ClassLoader]
                                                                                                 │
[Application ClassLoader] <──(Loads Custom)── [Platform ClassLoader] <──(Fails to find)──────────┘
```

1. The **Application ClassLoader** passes the request up to the **Platform ClassLoader**.
2. The **Platform ClassLoader** passes it up to the **Bootstrap ClassLoader**.
3. The **Bootstrap ClassLoader** checks `java.base`. It cannot find your custom class, so it passes the request back down.
4. The **Platform ClassLoader** checks `java.sql`, `java.xml`, etc. It cannot find it, so it passes the request down.
5. The **Application ClassLoader** finally searches your application's classpath, finds the file, and loads it into memory.

---

## `Class.forName()` vs `ClassLoader.loadClass()`

**One line:** Both load a class, but `Class.forName()` **initializes** it (runs static blocks) by default; `loadClass()` does **not**.
**Whether or not the class is initialized immediately after being loaded**. : 
 *  `Class.forName()` : YES  
 * `ClassLoader.loadClass()` : NO  
This is a subtle but favorite "gotcha" question. Both bring a class into the JVM, but they stop at different phases of the lifecycle above: 

- **`Class.forName("com.example.Foo")`** — loads _and_ initializes: static blocks and static assignments run immediately. 
- **`ClassLoader.loadClass("com.example.Foo")`** — loads and links, but stops **before** Initialization. Static blocks do _not_ run until the class is later actively used. 

````tabs
tab: Normal Comparision
```java
package com.example;  

class Widget {  
    static { System.out.println("Widgets initialized from 💥static block!"); }  
}  
  
public class ClassLoaderDemo {  
    public static void main(String[] args) throws Exception {  
        String className = Widget.class.getName(); //fully qualified name: "com.example.Widget"   
        ClassLoader classLoader = ClassLoaderDemo.class.getClassLoader();  
  
		System.out.println("--- 1. Testing loadClass ---");  
		classLoader.loadClass(className);  
		System.out.println("ClassLoader.loadClass() finished! (Notice: static block hasn't run yet)");  
		// prints nothing — static block hasn't run yet  
		  
		System.out.println("\n--- 2. Testing forName ---" );  
		Class.forName(className);  
		System.out.println("Class.forName finished()!");  
		// prints "Widget initialized from 💥static block!" right here  
    }  
}
```

tab: Forced Initialization
```java
package com.example;  
  
class Widget {  
    static { System.out.println("Widgets initialized from 💥static block!"); }  
}  
  
public class ClassLoaderDemo {  
    public static void main(String[] args) throws Exception {  
        String className = Widget.class.getName(); //fully qualified name: "com.example.Widget"   
        ClassLoader classLoader = ClassLoaderDemo.class.getClassLoader();  
  
		System.out.println("--- 1. Testing loadClass ---");  
		Class<?> clazz1 = classLoader.loadClass(className);  
		System.out.println("ClassLoader.loadClass() finished! (Notice: static block hasn't run yet)");  
		// prints nothing — static block hasn't run yet  
		
		System.out.println("\n--- 2. Forced initializatin by instantiating the class ---" );  
		System.out.println("\nNow creating an instance of the class loaded by loadClass()...");  
		clazz1.getDeclaredConstructor().newInstance(); // This finally forces initialization
		// prints "Widget initialized from 💥static block!" right here 
		  
		System.out.println("\n--- 3. Testing forName ---" );  
		Class.forName(className);  
		System.out.println("Class.forName finished()!");  
		// prints nothing — static block already run in step 2 (only once)
    }  
}
```
````

**Why it matters in the real world:** JDBC's classic `Class.forName("com.mysql.cj.jdbc.Driver")` relied on the driver's static block running to register itself. If it had used `loadClass()`, the registration static block wouldn't fire and the driver wouldn't work.

There's also an overload if you want `forName` _without_ initialization:

```java
Class.forName(String name, boolean initialize, ClassLoader loader)
Class.forName("Widget", false, classLoader); // initialize = false
```

_Interview Q: "Difference between `Class.forName()` and `loadClass()`?"_ → `forName` initializes (runs static blocks); `loadClass` only loads/links, deferring initialization.

> [!question]- Difference between `Class.forName()` and `loadClass()` ?
> 
|Feature|`Class.forName()`|`ClassLoader.loadClass()`|
|---|---|---|
|**Class Initialization**|**Yes** (Runs static blocks)|**No** (Defers execution)|
|**Execution Phase**|Loads + Links + Initializes|Loads only|
|**Failure Exception**|`ClassNotFoundException`|`ClassNotFoundException`|
|**Customization**|Has an overload to toggle initialization|Behavior depends on the target loader instance|

---
## Custom ClassLoaders

**Context** : Normally, the Java virtual machine loads classes from the **local file system** in a platform-dependent manner. 
However, some classes may not originate from a file; they may originate from **other sources**, such as the network, or they could be constructed by an application. 
	
**One line:** You can subclass `abstract class ClassLoader` to load bytecode from anywhere — a network, a database, an encrypted file — and to isolate classes from each other.

Mostly conceptual for an interview, but knowing _why_ they exist signals maturity:
- **Loading from non-standard sources** — pull `.class` bytes from a URL, DB, or decrypt them on the fly.
- **Isolation** — two web apps in the same Tomcat can each use a _different_ version of the same library, because each app gets its own classloader, and class identity includes the loader. Same class name, different loader = different type, kept apart.
- **Hot reloading / plugins** — discard a classloader and create a new one to reload changed classes without restarting the JVM (how app servers and plugin systems achieve reload).

Minimal shape (you override `findClass`, not `loadClass`, so you still inherit parent delegation):

```java
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytecode = loadBytecodeSomehow(name); // from file/network/db
        // defineClass turns raw bytes into a Class object
        return defineClass(name, bytecode, 0, bytecode.length);
    }

    private byte[] loadBytecodeSomehow(String name) {
        // your custom source logic here 
        // e.g. load the class data from the connection 
        throw new UnsupportedOperationException("demo");
    }
}
```

Note: overriding `findClass` (not `loadClass`) is the recommended pattern precisely because it preserves parent delegation — your custom logic only runs _after_ the parents have declined.

_Interview Q: "When would you write a custom classloader?"_ → App-server app isolation, hot reload/plugins, or loading bytecode from unusual/encrypted sources.

![[Custom ClassLoader.drawio.png]]

---

That's Module 2. The chain runs naturally: **Types** (who the loaders are) → **Phases** (what loading actually does) → **Delegation** (how the loaders cooperate) → the two "gotcha" notes that test whether you understood phases and delegation. For your MOC, link **Phases** ↔ **`forName` vs `loadClass`** (both hinge on Initialization) and **Delegation** ↔ **Custom ClassLoaders** (both hinge on `findClass` preserving delegation).


Class Loaders Acronym: **BPA**
- **B** – **B**ootstrap ClassLoader
- **P** – **P**latform ClassLoader
- **A** – **A**pplication ClassLoader

Linking Phases Acronym: **VPR**
- **V** – **V**erification (Checks bytecode validity)
- **P** – **P**reparation (Allocates memory for static variables)
- **R** – **R**esolution (Transforms symbolic references to direct references)