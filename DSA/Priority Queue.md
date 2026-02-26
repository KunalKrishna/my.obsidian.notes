
## Priority Queue

```java
public class User {
    String name;
    int age;
    double salary;
    double rating;

    public User(String name, int age, double salary, double rating) {
        this.name = name;
        this.age = age;
        this.salary = salary;
        this.rating = rating;
    }

    @Override
    public String toString() {
        return String.format("%s (Rating: %.1f, Salary: %.0f)", name, rating, salary);
    }
}
```
#### A. Min Heap (Default Behavior)

```Java
// 1. Using Lambda (Java 8+) - Sort by Salary Ascending
PriorityQueue<User> minHeap = new PriorityQueue<>(
    (u1, u2) -> Double.compare(u1.salary, u2.salary)
);

minHeap.add(new User("Alice", 25, 90000, 4.5));
minHeap.add(new User("Bob", 30, 70000, 4.0));

// Output: Bob (70k) -> Alice (90k)
System.out.println(minHeap.poll());
```

#### B. Max Heap (Reverse Order)

```Java
// 1. Using Comparator.reversed()
PriorityQueue<User> maxHeap = new PriorityQueue<>(
    (u1, u2) -> Double.compare(u2.rating, u1.rating) // Note arg order
);

// OR 2. Modern Java 8 Comparator approach (Cleaner)
PriorityQueue<User> maxHeapModern = new PriorityQueue<>(
    Comparator.comparingDouble((User u) -> u.rating).reversed()
);

maxHeapModern.add(new User("Alice", 25, 90000, 4.5));
maxHeapModern.add(new User("Bob", 30, 70000, 4.0));

// Output: Alice (4.5) -> Bob (4.0)
System.out.println(maxHeapModern.poll());
```
or 
```Java
List<Integer> list = Arrays.asList(5, 1, 9, 3);
// 1. Create Empty Max-Heap
PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Collections.reverseOrder());
// 2. Add All
maxPQ.addAll(list);
```

#### C. Handling Ties (Chaining Comparators)
If ratings are equal, prioritize by salary.

```Java
PriorityQueue<User> complexHeap = new PriorityQueue<>(
    Comparator.comparingDouble((User u) -> u.rating).reversed()
              .thenComparingDouble(u -> u.salary)
);
```


### Common Use Cases
1. **Top-K Elements Pattern:** Finding the "Top 10 highest paid employees" in a massive stream of data without sorting the whole list. You keep a Min Heap of size K; if a new element is larger than the root, you pop the root and push the new element.
2. **Graph Algorithms:** Dijkstra’s Shortest Path and Prim’s Minimum Spanning Tree (to select the next closest node).
3. **Task Scheduling:** Operating systems use this to execute processes with the highest priority/shortest job time first.
4. **Merging Sorted Streams:** Merging multiple sorted log files into one chronologically sorted view.