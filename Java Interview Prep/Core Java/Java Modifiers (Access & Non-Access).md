Java Modifiers : 
- ==**access modifier**== : { public, private, *default*, protected } controls the access level (visibility). operates at two levels :  
	- **Class level** : 
		- *default*
		- public
	- **Member level (attributes, methods and constructors)** :
		- private, 
		- *default*, 
		- protected
		- public
- ==**non-access modifier**== : do not control access level, but provides other functionality  to classes, methods, and attributes.
	- `final`, `abstract`, `synchronized`, `transient`, `volatile`, and `native`. 
	- can be categorized based on functionality as
		- Inheritance related : `final` 
		- Abstraction related : `abstract`, "`default`"
		- thread related : `synchronized`, `transient`, `volatile`
		- other lang related : `native`

Can use access modifiers and non-access modifiers together, for example, `public static final`

package = named groups of related classes.
class ~ file :: package ~ directory
![[Access Modifier Java.png]]
>Access level modifiers determine whether other classes can use a particular **field** or invoke a particular **method**. There are two levels of access control:
- At the top level—`public`, or _package-private_ (no explicit modifier).
- At the member level—`public`, `private`, `protected`, or _package-private_ (no explicit modifier).

>A **class** may be declared with the modifier `public`, in which case that class is visible to all classes everywhere. If a class has no modifier (the default, also known as _package-private_), it is visible only within its own package. 

>At the **member level**, you can also use the `public` modifier or no modifier (_package-private_) just as with top-level classes, and with the same meaning. For members, there are **two additional** access modifiers: `private` and `protected`. The `private` modifier specifies that the member can only be accessed in its own class. The `protected` modifier specifies that the member can only be accessed within its own package (as with _package-private_) and, in addition, by a subclass of its class in another package. 

${\Rightarrow}$ A class is always visible within the same packages. If you want to make the class visible outside its package, only then add `public` access modifier.
	${\Rightarrow}$ If you want to make class 
${\Rightarrow}$ default behavior for both class & class attributes is to be visible everywhere in the package which contains these class & attributes/methods.
${\Rightarrow}$ for a (default) class you can only increase its visibility scope (by adding public) but for attributes you can decrease its access (private limits within the class) or increase its visibility outside the package (protected, public) 
	${\Rightarrow}$ **class** ${\rightarrow}$ public class 
	${\Rightarrow}$ private ${\leftarrow}$ **attributes** ${\rightarrow}$ protected ${\rightarrow}$ public 


![[access modifier _ venn diag.svg]]

![[Access Modifier 0.svg]]
>

| Modifier          | Class | Package | Subclass | World | Levels             |
| ----------------- | ----- | ------- | -------- | ----- | ------------------ |
| `public`          | Y     | Y       | Y        | ==Y== | top + member level |
| `protected`       | Y     | Y       | ==Y==    | ==Y== | member level only  |
| *package-private* | Y     | ==Y==   | N        | N     | top + member level |
| `private`         | ==Y== | N       | N        | N     | member level only  |
If you need a field to be used  : 
- only in this class : use private 
- only in current package (package-private): use no modifier 
- only by all its sub-classes : use protected
- by any class : use public {only need to import the class}

| If you need a field to be used              | Use Access Modifier | Remarks                                                                                         |
| ------------------------------------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| only in this class                          | `private`           | Static Inner Class, Non-Static Inner, Method-Local Inner, Anonymous Inner class all have access |
| only in current package (*package-private*) | `---`               |                                                                                                 |
| only by all its sub-classes                 | `protected`         | extend `Parent` class; import needed if Child outside Parent's package                          |
| by any class                                | `public`            | only need to import the class                                                                   |


![[Java access modifier 2.png]]

> 

```Java
[ ][public] class Class_name {
	[private][protected][ ][public] data_type field;
	...
	...
}

// this class visible everywhere this package is imported
public class A {
	public    int a;  
	protected int b;  
	          int c;  
	private   int d;
	// constructor, getter, setter, methods()

}

// this class not visible outside the package where it is defined 
class B {
	...
}

[package-private]/[public] class Class_name {
	[private][protected]/[package-private]/[public] field;
}
```

Best practice : 
* Use `private` unless you gave a good reason not to. 
* Avoid `public` fields except for constants. 
* Class must always be p
* 

https://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html



Primary Categories of Class Members : 
- fields, 
- methods, 
- constructors, 
- instance initializers (code blocks), 
- static initializers (code blocks), 
- member classes (inner classes), 
- member interfaces. 


- Variables/Fields
- Constructor()
- Method()
- code blocks {}
- Nested Inner Class and Nested Interfaces

**Primary Categories of Class Members**
1. Variables(fields) 
2. Code Blocks (Arbitrary/Nested Blocks) 
	1. **Static Initialization Blocks (SIB)** 
	2. **Instance Initialization Blocks (IIB)** 
	3. **Synchronized Blocks** 
3. Methods
	1. Constructors() 
	2. Methods()
		1. Class Methods
		2. Instance Methods
4. Nested Class
	1. Non-static : called *Inner classes*.
	2. Static : called *static nested classes*.

**Primary Categories of Class Members**
1. Variables(fields) 
2. Code Blocks (Arbitrary/Nested Blocks) : [Different Types of Blocks in Java: With Examples](https://www.wscubetech.com/resources/java/blocks-types)
	1. **Static Initialization Blocks (SIB)**
		1. a static block itself cannot be explicitly declared with the `synchronized` in its declaration
		2. it is inherently thread-safe. (but not static methods)
	2. **Instance Initialization Blocks (IIB)** : appear directly within a class (without the `static` keyword) and are executed every time an object of the class is created, right before the constructor runs. They are often used to share common initialization code among multiple constructors.
	3. **Synchronized Blocks** : used in multithreaded programming to restrict access to a block of code so that only one thread at a time can execute it. This ensures thread safety when multiple threads try to access shared resources.
3. Methods
	1. Constructors() : same name, no return type, all 4 access modifier allowed. 
	2. Methods
		1. Class Methods
		2. Instance Methods
4. Nested Class
	1. *Non-static nested classes (inner classes)*: have access to other members of the enclosing class, even if they are declared private. 
		1. local classes
		2. anonymous classes
	2. *Static nested classes*: do not have access to other members of the enclosing class

Execution Flow of Blocks in Java : Static block --> IIB --> Constructor --> method blocks

### Code Blocks 
```Java
public class BlockExample {
    // Static variable
    static int staticVar;
    // Non-static (instance) variable
    int instanceVar;

    // Static code block (runs once)
    static {
        staticVar = 10;
        System.out.println("Static block executed (once). Static var: " + staticVar);
    }

    // Non-static code block (runs on each new instance)
    {
        instanceVar = 5;
        System.out.println("Non-static block executed. Instance var: " + instanceVar);
    }

    // Constructor (runs after non-static block)
    public BlockExample() {
        System.out.println("Constructor executed.");
    }

    public static void main(String[] args) {
        System.out.println("Inside Main Method.");
        new BlockExample(); // Creates the first object
        new BlockExample(); // Creates the second object
    }
}
/**
Output : 
Static block executed (once). Static var: 10
Inside Main Method.
Non-static block executed. Instance var: 5
Constructor executed.
Non-static block executed. Instance var: 5
Constructor executed.
**/
```

### Nested Classes


```Java
class OuterClass {
    ...
    class InnerClass {
        ...
    }
    static class StaticNestedClass {
        ...
    }
}
```

class can have 
- access modifier
	- none (default)
	- public
- non-access modifier
	- final
	- abstract
field can have 
- access modifier 
- non-access modifier
	- 


|                    | can use   | cannot use                                                                              | return type          |
| ------------------ | --------- | --------------------------------------------------------------------------------------- | -------------------- |
| Class              |           |                                                                                         |                      |
|                    | public    |                                                                                         |                      |
|                    | final     |                                                                                         |                      |
| Variables/Field    |           |                                                                                         | void                 |
|                    | public    |                                                                                         | primitive            |
|                    | protected |                                                                                         | object               |
|                    | private   |                                                                                         |                      |
| Constructor        |           |                                                                                         | none (not even void) |
|                    | public    | non-access modifiers such as `static`, `final`, `abstract`, `native`, or `synchronized` |                      |
|                    | protected | - Cannot be abstract, static, or final                                                  |                      |
|                    | private   |                                                                                         |                      |
| Method             |           |                                                                                         | void                 |
|                    | public    |                                                                                         | primitive            |
|                    | protected |                                                                                         | object               |
|                    | private   |                                                                                         |                      |
| Code blocks {}     |           |                                                                                         |                      |
|                    |           |                                                                                         |                      |
|                    |           |                                                                                         |                      |
| Nested Class       |           |                                                                                         |                      |
|                    | public    |                                                                                         |                      |
|                    | protected |                                                                                         |                      |
|                    | private   |                                                                                         |                      |
| Static Initializer |           |                                                                                         |                      |
