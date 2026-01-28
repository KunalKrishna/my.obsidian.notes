# String
String = an ==*immutable*== ==sequence of characters== . 
Modification methods (like `toUpperCase()`) actually return a **new** String object.
Java 9+ (Compact Strings era)

Look beyond the basic API and understand the 
- **memory model**, 
- **immutability**, and 
- **internal optimizations**

Agenda : 
- Internal Architecture 
- String Literal (SCP) vs String Object
- `intern()`
- Key API methods

***Why String objects are immutable in Java?*** 
- Security 
- Memory saving : SCP
- Thread Safety
- Caching : `hashCode()`

***Explain what is going on under the hood and why (advantages)*** : 
* `String lit1 = "text"` , `String lit2 = "text";` 
* `String s_concat = "A" + "B" + "C";`
* `String s_for_concat = ""; for (int i=0; i<10; i++) s += i;`
* `String s1 = new String("text"); `
* `String s2 = new String("text");`
* `String s3 = s2.intern(); `
* `if (variable.equals("constant")`) vs `if ("constant".equals(variable))` 

***How would would you declare a string and why ?*** 
* `String str = "whatever_string_required_to_use";` vs 
* `String str = new String("whatever_string_required_to_use");`  

### Internal Architecture 
```Java
public final class String  
    implements java.io.Serializable, Comparable<String>, CharSequence...
	{
		private final byte[] value; //payload
		
		private final byte coder; // the encoding used to encode the bytes in value.
			   
		public String(String original) {  
			this.value = original.value;  
			this.coder = original.coder;  
			this.hash = original.hash;  
			this.hashIsZero = original.hashIsZero;  
		}
		               
	}
```

```java
Stack:
   String s --> reference to object

Heap:
   String object
   └── byte[] value   <-- payload
   └── coder
   └── hash
```

### `intern()`

interned = lit. confine (someone) as a prisoner, especially for political or military reasons.

What is it ?

***When is intern() actually useful ?*** 
> `intern()` is **only** useful when you **do not know** the string at compile time. It is used for **Runtime Strings** generated from external sources (Databases, Files, Network Requests) that have high repetition.

```Java
// Simulating reading lines from a file
while ((line = reader.readLine()) != null) {
    String[] parts = line.split(",");
    String name = parts[0];
    
    // BAD: Creates a new "USA" object in heap for every row
    // String country = parts[1]; 

    // GOOD: Checks Pool. If "USA" exists, returns that ONE instance.
    // The "new" heap string from parts[1] is immediately discarded for GC.
    String country = parts[1].intern(); 
    
    // ... store in User object ...
}
```

- **For Literals (`"Java"`):** Never use `intern()`. Just use `String s = "Java"`.
- **For Runtime Data (DB/Files):** Use `intern()` if the data has low cardinality (few unique values repeating many times, like Country Codes, Statuses, Categories).
- **The Pattern:** `String finalized = (incomingData).intern();` is the correct approach to ensure you hold only the memory-efficient reference.


### Key API Methods (Quick Reference)

|**Method**|**Description**|**Note for Interviews**|
|---|---|---|
|**`equals(Object o)`**|Checks content equality.|Always use this, never `==`.|
|**`equalsIgnoreCase()`**|Case-insensitive check.|Efficient; avoids creating new uppercase strings.|
|**`substring(int begin, int end)`**|Returns partial string.|In modern Java, this copies the bytes (no memory leak).|
|**`trim()` vs `strip()`**|Removes whitespace.|`strip()` (Java 11) is "Unicode-aware" and safer than `trim()`.|
|**`join(delimiter, elements)`**|Joins array/list into String.|Very useful for "Comma Separated" output.|
|**`toCharArray()`**|Converts to `char[]`.|Essential for algorithms (Two Pointers, Sliding Window).|


| **Class**              | **Mutable?** | **Thread-Safe?**       | **Speed**            | **Use Case**                   |
| ---------------------- | ------------ | ---------------------- | -------------------- | ------------------------------ |
| **`String`**           | **No**       | Yes                    | Slow (concatenation) | Constants, params, map keys.   |
| ~~**`StringBuffer`**~~ | Yes          | **Yes** (Synchronized) | Slow                 | Legacy code (rarely used now). |
| **`StringBuilder`**    | Yes          | **No**                 | **Fast**             | String manipulation, Loops.    |
# StringBuffer (legacy code)

# StringBuilder*