
![[map_colored_border.drawio.png]]

![[map 04.png]]

**Best Practice:** **Keys in a Map should be Immutable.** This is why `String` and `Integer` are the most popular Map keys—they cannot change.

**Map** < I> 
- protected Object clone()
- boolean	equals(Object o)
- int	hashCode()
- int	size()
- boolean containsKey(Object key)
- Set< K>	keySet()   
- boolean containsValue(Object value)
- Collection< V>	values()
- abstract Set<Map.Entry<K,V>> entrySet()
- V	get(Object key)
- V	put(K key, V value)
- void	putAll(Map<  extends K, extends V> m
- DELETION 
	- V	remove(Object key)
	- void	clear()
	- boolean	isEmpty()
- DEFAULT
	- default V	merge(K key, V value, BiFunction< super V, super V, extends V> remappingFunction)
	- default V	putIfAbsent(K key, V value)
	- default boolean	remove(Object key, Object value)
	- default V	replace(K key, V value)
	- default boolean	replace(K key, V oldValue, V newValue)
	- default void	replaceAll(BiFunction< super K, super V, extends V> function)

Sub Interfaces : 
- **SortedMap**<K,V> extends Map<K,V>
	- **NavigableMap**<K,V> extends SortedMap<K,V>
	- **ConcurrentNavigableMap**<K,V> extends ConcurrentMap<K,V>, NavigableMap<K,V>
- ConcurrentMap<K,V> extends Map<K,V> 

Implementing classes : 

Abstract class
- AbstractMap implements Map<K,V>
Class
- **HashMap** extends AbstractMap<K,V>
	- LinkedHashMap extends HashMap<K,V>
- **ConcurrentHashMap** extends AbstractMap<K,V> implements <u>ConcurrentMap</u><K,V>, Serializable
- **TreeMap**  extends AbstractMap<K,V> implements <u>NavigableMap</u><K,V>, Cloneable, Serializable
- WeakHashMap extends AbstractMap<K,V>

`Map` interface organized by functionality. This structure separates basic operations from the modern Java 8+ features.

|      | ==1. Basic Operations (CRUD)==                              |                                                                                                                                                                                                                                       |
| ---- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | `V get(Object key)`                                         | Returns the value mapped to the key, or `null` if not found.                                                                                                                                                                          |
| 2    | `V put(K key, V value)`                                     | Associates the value with the key. If the key existed, it returns the _old_ value.                                                                                                                                                    |
| 3    | `void putAll(Map m)`                                        | Copies all mappings from map `m` to this map.                                                                                                                                                                                         |
| 4    | `V remove(Object key)`                                      | Removes the mapping for the key and returns the value.                                                                                                                                                                                |
| 5    | `void clear()`                                              | Removes all mappings.                                                                                                                                                                                                                 |
|      | ==**2. Inspection & Status**==                              |                                                                                                                                                                                                                                       |
| 6    | `boolean containsKey(Object key)`                           | Fast check ($O(1)$ typically) if a key exists.                                                                                                                                                                                        |
| 7    | `boolean containsValue(Object value)`                       | Slow check ($O(n)$) if a value exists anywhere.                                                                                                                                                                                       |
| 8    | `int size()`                                                | Returns the number of key-value pairs.                                                                                                                                                                                                |
| 9    | `boolean isEmpty()`                                         | Returns `true` if the map contains no key-value mappings.                                                                                                                                                                             |
|      | ==**3. Collection Views (Iterating)**==                     | *==These provide "windows" to view the data as Collections==*                                                                                                                                                                         |
| 10   | `Set< K> keySet()`                                          | Returns a Set of all keys.                                                                                                                                                                                                            |
| 11   | `Collection< V> values()`                                   | Returns a Collection of all values (can contain duplicates).                                                                                                                                                                          |
| 12   | `Set<Map.Entry<K, V>> entrySet()`                           | Returns a Set of **Key-Value objects**. This is the most efficient way to iterate if you need both the key and the value.                                                                                                             |
|      | ==**4. Java 8: Default & Conditional ("Safety" Methods)**== | ==*These reduce `if-else` boilerplate and null checks.*==                                                                                                                                                                             |
| 13   | `V	getOrDefault(Object key, V defaultValue)`                | Returns the value if present, otherwise returns `defaultValue`. (Great for avoiding `null`).                                                                                                                                          |
| 14   | `V	putIfAbsent(K key, V value)`                             | Adds the key _only_ if it is not already associated with a value (or is mapped to null).                                                                                                                                              |
| 15   | `boolean	remove(Object key, Object value)`                  | Removes the entry only if the key is currently mapped to the specific `value`.                                                                                                                                                        |
| 16   | `V	replace(K key, V value)`                                 | `map.put("A", 1)`: If "A" is missing, it **creates** it.<br>`map.replace("A", 1)`: If "A" is missing, it **does nothing**`                                                                                                            |
| 17   | `boolean	replace(K key, V oldValue, V newValue`             | Updates the value only if the key is currently mapped to `oldValue`.                                                                                                                                                                  |
| 18   | `replaceAll(BiFunction<K,V,V> function)`                    | `// (key, val) -> return new value `<br>`map.replaceAll((k, v) -> v.toUpperCase());`                                                                                                                                                  |
|      | ==**5. Java 8: Functional & Compute ("Smart" Methods)**==   | *These methods take Lambda expressions to perform atomic operations. They are crucial for writing concise modern Java.*                                                                                                               |
| 19   | `void forEach(BiConsumer action)`                           | `map.forEach((k, v) -> System.out.println(k + "=" + v));`                                                                                                                                                                             |
| 20*  | `V	compute(K key, BiFunction  remappingFunction) `          | `map.compute("Score", (key, currVal) -> {`<br>`if (currentVal == null) return 10;`<br>`if (currentVal > 100) return null;`<br>`return currentVal + 10;`<br>`});`                                                                      |
| 21*  | `V	computeIfAbsent(K key, Function mappingFunction)`        | "Lazy Loading." If the key is missing, run the function to create the value, put it in the map, and return it.<br>`//Create a new list for the user if one doesn't exist`<br>`map.computeIfAbsent("User1", k -> new ArrayList<>());`  |
| 22*  | `V computeIfPresent(K key, BiFunction remappingFunction)`   | "The Smart Updater" If the key exists, calculate a new value based on the key and old value.                                                                                                                                          |
| 23** | `V merge(K key, V value, BiFunction remappingFunction)`     | "The Combiner." If the key is missing, put `value`. If the key exists, merge the old value and the new `value` using the function.<br>`// Classic Word Count example`<br>`map.merge("word", 1, (oldVal, newVal) -> oldVal + newVal);` |

|**Method**|**Runs Logic When...**|**Ideal For...**|
|---|---|---|
|`computeIfAbsent`|Key is **Missing**|Lazy loading / Initialization.|
|`computeIfPresent`|Key is **Present**|Updating existing data only.|
|**`compute`**|**Always**|Complex logic (Add, Update, or Delete dynamically).|

==`V`==**`compute(K key, BiFunction remappingFunction)`**
- **What it does:** The <u>"Universal" atomic operator</u>. It executes the function ***regardless** of whether the key exists or not.*
- **How it works:**
    1. It checks the key.
    2. It passes the Key and the Current Value (or `null` if missing) to your Lambda.
    3. **If Lambda returns a value:** The map is updated/added.
    4. **If Lambda returns `null`:** The key is **removed** from the map.
- **Use Case:** When you need complex logic that covers both "**creation**" and "**update**" in one atomic step, or when you might want to **delete** the entry *based on a calculation*.

==`V`==**`computeIfAbsent(K key, Function mappingFunction)` : **"The Lazy Loader"**
- **Logic:** "I want the value for this key. If it exists, give it to me. If it is **missing** (or null), calculate it using this function, put it in the map, and _then_ give it to me."
- **The Function:** Takes **1 argument** (the Key) $\rightarrow$ Returns the **New Value**.
- **Key Characteristic:** The function **only runs** if the key is missing.

| **Feature**       | **computeIfAbsent**                    | **computeIfPresent**                        |
| ----------------- | -------------------------------------- | ------------------------------------------- |
| **Trigger**       | Key is **Missing** (or value is null). | Key is **Present** (and value is not null). |
| **Arguments**     | `Function` (Key only).                 | `BiFunction` (Key + Old Value).             |
| **Primary Use**   | Initialization, Caching, Lazy Loading. | Updates, Conditional Removal.               |
| **Return `null`** | key remains missing (nothing added).   | Key is **REMOVED** from map.                |
|                   |                                        |                                             |
|                   |                                        |                                             |

**Scenario A: The "Multi-Map" Problem (Best Use Case) - GROUP BY** 
Imagine you are grouping employees by Department. You need a `Map<String, List<String>>`.

**The Old Way (Verbose):**
```Java
List<String> employees = map.get("Engineering");
if (employees == null) {
    employees = new ArrayList<>();
    map.put("Engineering", employees);
}
employees.add("Alice");
```
**The New Way (Concise):**
```Java
// Logic: If "Engineering" is missing, create new ArrayList. Then return that list.
map.computeIfAbsent("Engineering", k -> new ArrayList<>())
   .add("Alice"); 

// Adding another person to the SAME list (Lambda won't run this time)
map.computeIfAbsent("Engineering", k -> new ArrayList<>())
   .add("Bob");
```

**Scenario B: Expensive Calculation (Caching)**
If calculating the value is expensive (e.g., database call), this method ensures you only pay that cost once.
```Java
// "fibonacciCache" is a Map<Integer, Integer>
int result = fibonacciCache.computeIfAbsent(10, k -> {
    System.out.println("Calculating..."); // Prints only once
    return heavyCalculation(k);
});
```

==`V`==**`computeIfPresent(K key, BiFunction remappingFunction)` : "The Smart Updater"**
- **Logic:** "I want to update the value for this key, but **only if it already exists**. If the key is missing, do nothing."
- **The Function:** Takes **2 arguments** (Key, OldValue) $\rightarrow$ Returns the **New Value**.
- **Key Characteristic:** If your function returns `null`, the entry is **removed** from the map.
**Scenario A: salary Hike**
You want to give a raise to an employee, but only if they are actually in your active directory.
```Java
Map<String, Double> salaries = new HashMap<>();
salaries.put("Alice", 50000.0);

// Bob is not in the map
salaries.computeIfPresent("Bob", (key, oldVal) -> oldVal + 10000); 
// Result: Nothing happens. Bob is not added.

// Alice is in the map
salaries.computeIfPresent("Alice", (key, oldVal) -> oldVal + 10000); 
// Result: Alice is now 60000.0
```
**Scenario B: The Conditional Delete**
You can use this to remove entries based on a calculation.
```Java
Map<String, Integer> stock = new HashMap<>();
stock.put("iPhone", 1);

// Logic: Decrement stock. If stock reaches 0, remove the item completely.
stock.computeIfPresent("iPhone", (item, quantity) -> {
    int newQty = quantity - 1;
    return (newQty == 0) ? null : newQty; // Return null to remove!
});

// Result: "iPhone" key is completely removed from the map.
```

==**`V`**== **`merge(K key, V value, BiFunction remappingFunction)`**
This method is designed to solve the **"Upsert"** problem (Update or Insert) in a single line. It is the most powerful method for aggregating data (like counting, summing, or grouping).

| **Method**                      | **Best Use Case**      | **Logic Summary**                                     |
| ------------------------------- | ---------------------- | ----------------------------------------------------- |
| **`put(k, v)`**                 | Blind Insert/Overwrite | "I don't care what was there. Here is the new value." |
| **`putIfAbsent(k, v)`**         | Safe Insert            | "Only put this if the spot is empty."                 |
| **`computeIfAbsent(k, func)`**  | Lazy Load              | "If empty, calculate the value and put it."           |
| **`computeIfPresent(k, func)`** | Safe Update            | "If it exists, change it."                            |
| **`merge(k, v, func)`**         | **Aggregate / Upsert** | "If empty, put $V$. If exists, combine Old+$V$."      |

1. **The Logic: "Add or Combine"**
`V merge(K key, V value, BiFunction remappingFunction)`
It follows this simple 3-step decision tree:
2. **Is the Key Missing (or null)?**
    - $\rightarrow$ Ignore the function. Just **INSERT** the `value` directly.
3. **Is the Key Present?**
    - $\rightarrow$ Run the `remappingFunction` (Old Value + New Value).
    - $\rightarrow$ **UPDATE** the map with the result.
4. **Did the Function return `null`?**
    - $\rightarrow$ **REMOVE** the key from the map.

---
2. **The "Killer Use Case": Frequency Counting**
This is the most common interview scenario for `merge`. You have a list of words, and you want to count how many times each word appears.
**The Old Way (Pre-Java 8)**
Verbose and error-prone.
```Java
Map<String, Integer> counts = new HashMap<>();
String[] words = {"apple", "banana", "apple", "orange", "banana", "apple"};

for (String word : words) {
    Integer count = counts.get(word);
    if (count == null) {
        counts.put(word, 1); // First time seeing it
    } else {
        counts.put(word, count + 1); // Increment existing
    }
}
```

**The `merge()` Way**
Clean and thread-safe (if using ConcurrentHashMap).
```Java
Map<String, Integer> counts = new HashMap<>();

for (String word : words) {
    // Logic: 
    // 1. If "word" is missing, put 1.
    // 2. If "word" exists, take old count (a) and add 1 (b).
    counts.merge(word, 1, (oldVal, newVal) -> oldVal + newVal);
    
    // Or simpler with Method Reference:
    // counts.merge(word, 1, Integer::sum);
}
```
---
3. **Scenario: Merging Complex Data**
Imagine you have two maps of User scores and you want to combine them.
- **Map A:** `{Alice=10, Bob=20}`
- **Map B:** `{Bob=15, Charlie=30}`
You want to merge B into A. If a user exists in both (Bob), add their scores.
```Java
mapB.forEach((key, scoreFromB) -> {
    mapA.merge(key, scoreFromB, (scoreFromA, newScore) -> scoreFromA + newScore);
});

// Result in Map A:
// Alice = 10 (Left alone, wasn't in B)
// Bob = 35   (Merged: 20 + 15)
// Charlie = 30 (Inserted, wasn't in A)
```
---
4. **Scenario: Conditional Deletion**
Just like `compute`, if your remapping function returns `null`, the entry is removed.
_Example:_ You are tracking a "balance". You add money. If the balance becomes 0 (or negative), close the account (remove the key).
```Java
// Balance: { "User1": -10 }
// We add 10 to User1.
map.merge("User1", 10, (currentBalance, amountToAdd) -> {
    int newBalance = currentBalance + amountToAdd;
    return (newBalance == 0) ? null : newBalance;
});

// Result: "User1" is removed from the map entirely.
```


## HashMap<K,V>
![[hashMap.png]]
A `HashMap` in Java is a part of the `java.util` package and implements the `Map` interface. It stores data in key-value pairs, where each key is unique and maps to a specific value.

Key Characteristics of `HashMap`:
- **Key-Value Pairs:** Stores data as pairs, where a unique key is associated with a value.
- **Unordered:**  (unlike `LinkedHashMap` or `TreeMap`).
- **Non-Synchronized:** Not thread-safe by default. For thread-safe operations, `ConcurrentHashMap` or `Collections.synchronizedMap()` can be used.
- **Allows Nulls:** Permits **one null key** and multiple null values.
- **Performance:** efficient `get` & `put` operations, ~  O(1) avg time(assuming a good hash function and minimal collisions.)

```Java
transient Node<K,V>[] table;

class Node<K,V> implements Map.Entry<K,V> {  
    final int hash;  
    final K key;  
    V value;  
    Node<K,V> next;
    ...
    }
```

Internal Working :
`HashMap` uses a **hash table** for storing key-value pairs. Internally, it relies on an **==array of linked lists==** (or **red-black trees** in Java 8+ for large buckets) called **buckets**. 
- **Hashing:** When a <K,V> pair is added, the `key.hashCode()` generates a hash value.
  ==`return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);`==
	- **Perturbed Hash** ("Smearing") : `XOR (h >>> 16)`): mixes "High Bits" into "Low Bits".
	- Edge Cases : 
		- hash = 0 [[#Edge Case keyObj = null]]
		- hash = k [[#Edge Case k]] effectively degrades into a **Linked List**🔗/**Tree** 🌳
		  (tests your understanding of how resizing works vs how indexing works)
- **Index Calculation:**  This hash value is used to determine the index in the internal array where the key-value pair will be stored.  index formula: ==`(n - 1) & hash`==
- **Collision Handling:** If multiple keys map to the same index (a collision), Java handles it using separate chaining, where elements at that index are stored in a **linked list** or a **red-black tree**. 
	- `p.next = newNode(hash, key, value, null);`
	- `e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);`
- **Retrieval:** When retrieving a value using a key, the same hashing process determines the index, and then the `equals()` method is used to find the exact key within the linked list/tree at that index.
```
Original h:      10101010 11110000 00001111 01010101
h >>> 16:        00000000 00000000 10101010 11110000 (Shift top to bottom)
XOR Result:      10101010 11110000 10100101 10100101
                                   ^
                                   The lower bits now "know" about the top bits.
```


![[hashmap - internal.png]]
#### Edge Case : keyObj = null

**The Problem:** If the key is `null`, you cannot call `null.hashCode()`. Doing so would immediately throw a `NullPointerException` and crash the program.

**The Mechanics: Where does it go?**
Since `null` has no hash code, Java **hardcodes** its hash to **0**.
- **Bucket location:** Since hash = 0, the index calculation `(n - 1) & 0` always results in **Index 0**.
- **Storage:** The `null` key entry will **always** be found in **Bucket 0**.
**Note on Chaining:** Bucket 0 is not reserved _exclusively_ for null. If you have _other_ (non-null) keys that happen to calculate to Index 0, they **will collide** with the `null` key. So, you can have a chain that looks like this: [ Key=null ] $\rightarrow$ [ Key="RareStringThatHashesToZero" ] $\rightarrow$ [ Key=Integer(0) ]
But in this chain, **only one node** has a `null` key.

**Summary**
- **Multiple `null` keys?** **Impossible.** (They overwrite each other).
- **Multiple `null` values?** **Possible.** (You can have `{A=null, B=null}`).
- **Chain in the Null Bucket?** **Possible.** (If other non-null keys collide with index 0).

**The Challenges & "Gotchas"**

**A. The Ambiguity of `get()`**
This is the most common bug introduced by null keys (and values).
If `map.get(k)` returns `null`, it can mean two different things:
1. The key `k` is missing.
2. The key `k` exists, but its value is `null`.
**Solution:** You cannot trust `get()` alone. You must use `containsKey()` to verify existence.

**B. The Compatibility Trap (Interview Favorite)**
Not all Maps allow null keys. If you switch your implementation later, your code will crash.

|**Map Implementation**|**Null Key Allowed?**|**Why?**|
|---|---|---|
|**`HashMap`**|**YES** (1 max)|Handles it as a special 0-hash case.|
|**`LinkedHashMap`**|**YES**|Inherits HashMap behavior.|
|**`Hashtable`**|**NO**|Throws NPE. (Legacy design decision).|
|**`ConcurrentHashMap`**|**NO**|Throws NPE. Ambiguity in concurrent threads is too dangerous.|
|**`TreeMap`**|**NO**|Throws NPE. It cannot sort/compare `null` vs a String.|
**C. The Uniqueness Limit**
You can only have one null key.
If you try to `put(null, "A")` and then `put(null, "B")`, "A" is overwritten. `null` is just like any other unique key in this regard.
```Java
HashMap<String, String> map = new HashMap<>();
// 1. First insert
map.put(null, "First Value"); 
// Bucket 0: [ Key=null, Val="First Value" ]

// 2. Second insert
map.put(null, "Second Value");
// Bucket 0: [ Key=null, Val="Second Value" ]
// The previous node was updated. The list size is still 1.
```
**Best Practices**
1. **Avoid using Null Keys**: While technically allowed, using null as a key is generally considered bad design. It usually implies a "default" or "unknown" state. It is cleaner to use a specific Constant object (Sentinel pattern) or a specific String like "**UNKNOWN**".
    - _Bad:_ `map.put(null, defaultSettings);`
    - _Good:_ `map.put(Configs.DEFAULT, defaultSettings);`
2. **Never iterate assuming non-null**: If you allow null keys, your for-each loops must be null-safe.
```Java
    for (String key : map.keySet()) {
        // If you don't check this, key.length() will throw NPE
        if (key != null) { 
            System.out.println(key.length());
        }
    }
```
3. **Know your Collection type**: If you are designing an API that accepts a Map, do not assume it supports nulls. If the user passes a ConcurrentHashMap and you try to insert a null key/value, your library will crash.
#### Edge Case : k 
If `hashCode()` always returns a constant `k`, the `HashMap` essentially breaks. Here is exactly how the **Resize** and **Growth** are impacted:

**1. The Immediate Impact: "The Linked List"**
Since every key produces the same hash `k`, every single `put` operation results in a collision.
- **Result:** All entries are stacked into **one single bucket** (e.g., Bucket X).
- **The Other Buckets:** The entire rest of the array remains completely empty (null).
- **Structure:** It effectively degrades from a Hash Table into a **Linked List** (or a Tree).

 **2. Impact on Resizing (The "Futile Expansion")**
Does the map stop resizing? No.
The HashMap is "dumb" regarding distribution. It only cares about the count.
```Java
/**  
 * The number of key-value mappings contained in this map. */
    transient int size;
	
	final V putVal() {
		...
		if (++size > threshold)  
		    resize();
		...
```
1. **The Trigger:** As you add elements, the `size` increases.1 Once `size > capacity * 0.75`, the map **WILL** resize (double the array).
2. **The Re-hashing (The Tragedy):**
    - The map creates a new array (size 32, then 64, etc.).
    - It loops through the old bucket.
    - It recalculates the index for every item: `index = k & (newCap - 1)`.
    - **Problem:** Since `k` is constant, the result of the calculation is **identical for every single item**.
**Result:** The map spends time creating a bigger array and copying data, but **all 1,000 items simply move together** from "Old Bucket 5" to "New Bucket 5" (or maybe "New Bucket 21"). They remain clumped together.

**3. Impact on Memory (The Waste)**
Because the map keeps doubling the array size to satisfy the `loadFactor` rule, you end up with a massive array that is **99.9% empty**.
- If you have 10,000 items, you might have an array of size 16,384.
- **16,383 buckets are null.**
- **1 bucket** holds a massive tree of 10,000 nodes.
- This is a massive waste of allocated memory space.

**4. Impact on Performance (The Slowdown)**
This is the worst-case scenario for time complexity.
- **Java 7 (and below):**
    - The bucket is a Linked List.
    - `get()` becomes **$O(n)$** (Linear).
    - If you have 1M items, a lookup takes 1M steps. The application effectively hangs.
- **Java 8+ (Modern):**
    - Once the bucket exceeds 8 items (and capacity $\ge$ 64), it converts to a **Red-Black Tree**.
    - `get()` becomes **$O(\log n)$**.
    - **Verdict:** It is _better_ than Java 7, but still significantly slower than the ideal **$O(1)$**, and the overhead of maintaining the tree structure is high

|**Feature**|**Normal Behavior**|**Constant HashCode Behavior**|
|---|---|---|
|**Storage**|Spread across all buckets.|Crammed into **one** bucket.|
|**Resize Trigger**|Works as expected.|**Still happens**, but uselessly.|
|**Memory**|Efficiently used.|**High waste** (Empty array slots).|
|**Lookup Time**|$O(1)$|$O(\log n)$ (Tree) or $O(n)$ (List).|

### Interview Questions
1. Why is `String`, `Integer`, or other Wrapper classes considered the best candidates for `HashMap` keys?
2. What happens if I use a mutable object as a Key, and I change a field after putting it in the Map?


------
## LinkedHashMap

A `LinkedHashMap` is a data structure that combines the hash table functionality of a `HashMap` with a ==doubly-linked== list to maintain the order of entries, typically insertion order. This allows for efficient key-value lookups while also providing a predictable iteration order, unlike a standard `HashMap`. In Java, it implements the `Map` interface and can be configured for== insertion-order or access-order iteration==, but it is ==not synchronized== by default and requires external synchronization if used concurrently by multiple threads. 

Key features
- **Maintains order**: Unlike a `HashMap`, which does not guarantee any order, a `LinkedHashMap` stores entries in a consistent order.
- **Dual data structure**: It uses a hash table for quick lookups and a doubly-linked list to maintain the order of entries.
- **Ordered iteration**: Iterating through a `LinkedHashMap` will return the entries in the order they were inserted (by default).
- **Access-order**: It can be configured to maintain a different order, where the most recently accessed entries are moved to the end of the list.- **Not synchronized**: This class is not thread-safe. If multiple threads access a `LinkedHashMap` concurrently, and at least one of them modifies the structure, external synchronization must be used. 

Use cases
- **Predictable iteration**: When you need to process the elements of a map in the same order they were added.
- **Caching**: It can be used to implement a cache, where the least recently used items can be easily removed to make space for new ones (using access-order).
- **Maintaining a history**: Useful in situations where the sequence of operations or insertions is important.

------------
## TreeMap

--------------
## ConcurrentHashMap

------------
## Interview Questions

### I. Internals & Architecture (Deep Dive)

**1. Why is `String`, `Integer`, or other Wrapper classes considered the best candidates for `HashMap` keys?**
- **The Trap:** Candidates often say "because they are final."
- **The Real Answer:**
    1. **Immutability:** They cannot be changed after creation, ensuring the `hashCode()` remains constant. You won't "lose" the object in the wrong bucket.
    2. **Cached HashCode:** `String` calculates its hash once and caches it (in a private field `hash`). Subsequent calls are $O(1)$, making retrieval extremely fast.
    3. **Correctness:** They have well-defined `equals()` and `hashCode()` implementations.

**2. What happens if I use a mutable object as a Key, and I change a field _after_ putting it in the Map?**
- **Answer:** The map breaks.
- **Explanation:** The object is stored in a bucket based on the _original_ hash. When you modify the object, its hash likely changes. When you try to `get()` it, Java calculates the _new_ hash, looks in the _new_ bucket (or the same bucket but fails the check), and returns `null`. This creates a "Memory Leak" because the object exists but is irretrievable.

**3. Why does `HashMap` use a Load Factor of 0.75 by default? Why not 0.5 or 1.0?**
- **Answer:** It is a mathematical trade-off (Poisson Distribution).
- **0.5 (Empty):** Reduces collisions significantly but wastes half the memory.
- **1.0 (Full):** Maximizes memory usage but drastically increases collision probability, degrading performance toward $O(n)$ or $O(\log n)$.
- **0.75:** Empirically minimizes collisions (avg bucket size ~0.5) while keeping memory overhead manageable.

**4. Can you use a byte array (`byte[]`) as a key in a `HashMap`?**
- **The Trick:** It compiles, but it **fails** logically.
- **Reason:** Arrays in Java do **not** override `equals()` or `hashCode()`. They use standard Object reference identity. Two arrays `new byte[]{1}` and `new byte[]{1}` will have different hash codes and return `false` for equals. You must use `ByteBuffer` or wrap it in a custom class.

---
### II. Concurrency & Thread Safety

**5. How does `ConcurrentHashMap` achieve thread safety differently in Java 8 compared to Java 7?**
- **Java 7 (Segment Locking):** Divided the map into 16 "Segments" (mini-HashMaps). A thread only locked one segment. Concurrency level = 16.
- **Java 8 (CAS + Synchronized):** Removed segments.
    - **Read:** No locking (volatile nodes).
    - **Write (No Collision):** Uses **CAS** (Compare-And-Swap) for speed.
    - **Write (Collision):** Uses `synchronized` on the **Head Node** of the specific bucket only.
- **Why:** Java 8 approach is more granular; locks are on the specific index, allowing massive concurrency.

**6. What is the difference between `Collections.synchronizedMap(new HashMap())` and `ConcurrentHashMap`?**
- **SynchronizedMap:** Uses a global lock (`mutex`) on the **entire map**. Only one thread can read or write at a time. High contention = poor performance.
- **ConcurrentHashMap:** Never locks the whole map. Allows concurrent reads and concurrent writes (on different buckets).

**7. Why does `ConcurrentHashMap` NOT allow `null` keys or values?**
- **Answer:** Ambiguity in multi-threading.
- **Scenario:** If `map.get(key)` returns `null`, you cannot tell if the key is missing or if the value is null. In a single-threaded map, you check `containsKey()`. In a multi-threaded map, the map might change _between_ the `get` and the `containsKey` check, making the result impossible to trust. To avoid this race condition, nulls are banned.

---
### III. Coding & Design Patterns

**8. How would you implement a "Least Recently Used" (LRU) Cache using Java Collections?**
- **The "Cheat" Answer:** Extend `LinkedHashMap`.
- **Mechanism:** `LinkedHashMap` maintains insertion order. It has a protected method `removeEldestEntry(Map.Entry eldest)`.
- **Implementation:** Override `removeEldestEntry` to return `true` when `size() > capacity`. Also, set the constructor flag `accessOrder` to `true` (so accessing an item moves it to the end/newest position).

**9. Design a Map that automatically cleans up old entries after a specific time (Time-To-Live).**
- **Concept:** "Passive Expiration" vs "Active Cleaning".
- **Approach:**
    - Wrap the Value in a custom object: `Wrapper { Value v, long expiryTime }`.
    - On `get()`: Check `if (System.currentTimeMillis() > expiryTime)`. If yes, remove and return null.
    - Background Thread: Run a `ScheduledExecutorService` to periodically scan and `remove()` expired keys to free memory.

**10. What is `WeakHashMap` and when would you use it?**
- **Concept:** Keys are wrapped in `WeakReference`.
- **Behavior:** If the **Key** object is no longer referenced anywhere else in the application, the Garbage Collector will delete the entry from the map (even if the map still holds it).
- **Use Case:** Storing metadata for objects (e.g., caching a specific property for a `Socket` or `Session`). If the Socket closes and is nulled out, the cache entry automatically disappears.
---
### IV. "Trick" Question

**11. What is the worst-case time complexity of `HashMap.put()` in Java 8, and exactly when does it happen?**
- **Answer:** $O(n)$ (Technically $O(n)$ for resize, $O(\log n)$ for tree insertion).
- **Explanation:**
    1. **Resize:** If the put triggers a resize, we must rehash all $n$ elements. Cost: $O(n)$.
    2. **Tree Insertion:** If no resize, but the bucket is a Tree, insertion is $O(\log k)$.
    3. **DOS Attack:** If someone maliciously sends 10,000 keys with the same hash code (but different values) and prevents treeification (by using a class that doesn't implement `Comparable`), it degrades to a Linked List: $O(n)$.