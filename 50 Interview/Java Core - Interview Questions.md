
1. *What is pass-by-value in Java?*  
2. *If a class has two fields (id, amount) and I pass the object to a method and update values, will the field update?*  
3. *If I pass int a = 10 to a method and update to 20 inside, will the original value update?*

Learning : 
* Java always passes arguments by value.
* For objects, the value passed is a reference, so fields can be changed, but the reference itself (which object it points to) cannot be changed in the caller by the method.
#### What is pass-by-value in Java?
- In Java, when you pass a variable to a method, you're passing a _copy_ of its value, not the variable itself.[](https://www.baeldung.com/java-pass-by-value-or-pass-by-reference)​
- For primitive types (like int, double), this copy is a direct copy of the value.
- For objects, you pass a copy _of the reference_ to the object – not the object itself, but a pointer to it.[](https://stackoverflow.com/questions/7893492/is-java-really-passing-objects-by-value)​
- Because only the reference is copied, changes to object fields through that reference inside the method _will_ affect the original object.

| Scenario                      | Original Updated? | Why                                                                                                                                        |
| ----------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Pass int, change inside       | No                | Copy of value passed [](https://www.geeksforgeeks.org/java/g-fact-31-java-is-strictly-pass-by-value/)​​                                    |
| Pass object, change field     | Yes               | Reference copy points to same object [](https://stackoverflow.com/questions/11801554/java-object-state-does-not-change-after-method-call)​ |
| Pass object, change reference | No                | Only the local reference changes [](https://stackoverflow.com/questions/7893492/is-java-really-passing-objects-by-value)​                  |

```Java
// Pass-by-value illustration 

//------ Passing a primitive(int) and updating it inside a method

void changePrimitive(int x) { 
	x = 20; // Only changes the local copy 
} 
int a = 10; 
changePrimitive(a); // a is still 10

//------ Passing an object and updating its fields

void changeObjFields(MyClass obj) {
    obj.value = 5; // This changes the original object's field
}

MyClass o = new MyClass();
o.value = 3;

changeObjFields(o); // o.value is now 5

//------ Pass Object, Change Reference 

class Item {
    int price;
}

void reassign(Item obj) {
    obj = new Item();   // obj now points to a new Item object
    obj.price = 999;    // This modifies the new object, not the original
}

// main method
Item myItem = new Item();
myItem.price = 100;

reassign(myItem);

// What is myItem.price now?
System.out.println(myItem.price); // ?
```


4. What are the use cases of wrapper classes in Java?  
5. Difference between stack and heap memory.  
6. What is an OutOfMemoryError? What are heap memory regions?

### `equals()` & `hashCode()`

The contract between `equals()` and `hashCode()` is the "Constitution" of the Java Collections Framework. If you violate it, `HashMap`, `HashSet`, and `Hashtable` stop working correctly.

1. **The Golden Rule**

> **If `a.equals(b)` is `true`, then `a.hashCode()` MUST be equal to `b.hashCode()`**

(Note: The reverse is _not_ required. Two unequal objects _can_ have the same hash code—that’s just a collision.)

![[equals_hashcode2.png]]


𝗝𝗮𝘃𝗮 𝗕𝗮𝗰𝗸𝗲𝗻𝗱 𝗜𝗻𝘁𝗲𝗿𝘃𝗶𝗲𝘄 𝗤𝘂𝗲𝘀𝘁𝗶𝗼𝗻𝘀 (𝗖𝗵𝗮𝗹𝗹𝗲𝗻𝗴𝗲 𝗥𝗼𝘂𝗻𝗱)  
  
1. What is the difference between JVM, JDK, and JRE?  
2. Explain Java Memory Model (Heap, Stack, GC).  
3. What are the four pillars of OOP?  
4. Difference between String, StringBuilder, StringBuffer.  
5. How does the Stream API work internally?  
6. Difference between HashMap, LinkedHashMap, TreeMap.  
7. Explain concurrency and ExecutorService.  


Preparing for a Backend Engineer role ? Just DSA isn't enough 
Here are 10 topics that you must learn : 
1. Concurrency & Parallelism Threads vs async, race conditions, locks, deadlocks, queues 
2. System Design : Design scalable systems (e.g., Dropbox, URL shortener), talk trade-offs: CAP, consistency, availability, latency. 
3. Databases & Caching : Normalize vs denormalize, secondary indexes, Redis vs Memcached, cache invalidation, eventual consistency. 
4. Distributed Systems Fundamentals : Leader election, replication, partition tolerance, distributed locking, failure recovery. 
5. Reliability Patterns: Retries with backoff, circuit breakers, bulkheads, graceful degradation, chaos testing. 
6. Message Queues & Async Flows : Kafka, RabbitMQ, or SQS : delivery guarantees, deduplication, replay strategies, ordering. 
7. Security : OAuth2, JWT pitfalls, mTLS for internal traffic, securing webhooks & service-to-service calls. 
8. Observability: Structured logs, tracing (OpenTelemetry), metrics, alerting : debug distributed requests across services. 
9. Common Coding Challenges : LRU cache, rate limiter, task scheduler, producer-consumer, flatten nested data structures 
10. Performance Tuning : Memory leaks, CPU bottlenecks, slow DB queries, N+1 problems


|**Approach**|**Memory**|**Speed**|**Backtracking Type**|
|---|---|---|---|
|**String (Concatenation)**|High (creates many Strings)|Slowest|Implicit (Copying)|
|**StringBuilder**|Medium (one object, resizing)|Fast|Explicit (`deleteCharAt`)|
|**char[] Array**|**Lowest** (fixed size)|**Fastest**|Implicit (Overwriting)|

# Performance Tuning

1. Why does a Java application slow down over time without throwing errors?  
2. What causes `OutOfMemoryError` even when heap size looks sufficient?  
3. How does GC tuning impact application latency in production?  
4. Why is `synchronized` sometimes worse than `Lock`?  
5. How do thread leaks happen and how do you detect them?  
6. Why does high CPU usage occur even when traffic is low?  
7. What happens if a thread dies while holding a lock?  
8. Why do applications behave differently under load compared to local runs?  
9. How does JVM handle memory for threads vs objects?  
10. What are real-world pitfalls of using parallel streams?  
11. How can improper `equals`/`hashCode` crash a production system?  
12. Why does caching sometimes reduce performance instead of improving it?  
13. How do you debug deadlocks in a live Java application?  
14. What happens if GC runs too frequently?  
15. Why do microservices fail due to thread pool misconfiguration?  
16. How does JVM warm-up affect benchmark results?  
17. What are common mistakes with `CompletableFuture` in production?  
18. Why does excessive logging slow down Java apps?  
19. How do memory leaks occur without static references?  
20. What JVM flags are most useful during performance issues?