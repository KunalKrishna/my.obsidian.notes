
// array of lists of Long
```java
List<Long>[] minList = new List[n];
for (int i = 0; i < n; i++) 
	minList[i] = new ArrayList<>();
```

### Circular Array

 [Transformed Array - LeetCode](https://leetcode.com/problems/transformed-array/solutions/7554297/x-from-left-n-x-from-right-by-thevagabon-2bqr/)


-------

### Circular Array Traversal 
Trick : **Virtual Index Mapping** using % operator
**Note**: standard modulo on negative numbers behaves weirdly in Java. Implementation is language specific.

let : 
```
array size = n
current index = i
shift = k times
target index = j (next or prev)

// Virtual Index
j = next = (i + k) % n; 
j = prev = (i - k + n) % n; 
```

Trick : 
* Forward : Add k, then mod
* Backward: Subtract k, then add n, then mod


| **Goal**                   | **Technique**                         | **Code Snippet**                         |
| -------------------------- | ------------------------------------- | ---------------------------------------- |
| **Simulate Linear Circle** | Iterate to `2*n`, access `i % n`      | `for(int i=0; i<2*n; i++) Use(arr[i%n])` |
| **Move "Next" `k` steps**  | Add `k`, then mod                     | `next = (i + k) % n`                     |
| **Move "Prev" `k` steps**  | Subtract `k`, add `n`, then mod       | `prev = (i - k + n) % n`                 |
| **Fixed Size Window**      | Keep window size `w`, slide `start++` | `winEnd = (start + w - 1) % n`           |



| Stage | shift (k) range                                                                                    | Formula                                                                               | Remark                                                    | Limitations                                                                                                                                           |
| ----- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| I     | **0<k<=n**                                                                                         | next_idx = (i+k)%n                                                                    |                                                           |                                                                                                                                                       |
|       | i.e. smaller +ve shift L or R                                                                      | prev_idx = (i-k+n)%n                                                                  |                                                           | Sign & Size dependent                                                                                                                                 |
|       |                                                                                                    |                                                                                       |                                                           | `prev_idx` breaks if shift k > i+n because % of (-)ve is ill-defined or lang dependent (Unsafe for Java).                                             |
|       |                                                                                                    |                                                                                       |                                                           |                                                                                                                                                       |
| II    | **n < k**                                                                                          | next_idx = (i+k)%n                                                                    | same                                                      |                                                                                                                                                       |
|       |                                                                                                    |                                                                                       | manual mod correction                                     | Still **conditioned on Sign**. i.e. fails if we shift right by k= -2. Assumes shift to be distance not displacement. Unsuitable for physics/vectors . |
|       |                                                                                                    | prev_idx = ((i-k+n)%n + n) % n                                                        | Double Mod Idiom                                          |                                                                                                                                                       |
|       |                                                                                                    | prev_idx = (i-(k%n)+n)%n                                                              | Pre-Reduce                                                |                                                                                                                                                       |
|       |                                                                                                    |                                                                                       | Java 8+ Built-In Helper                                   |                                                                                                                                                       |
|       |                                                                                                    | Math.floorMod(i-k, n)                                                                 | Pro-Java Way                                              |                                                                                                                                                       |
|       |                                                                                                    |                                                                                       |                                                           |                                                                                                                                                       |
| III   | $k\in \mathbb{Z}$                                                                                  | nextIndex = (((i + k) % n) + n) % n;                                                  | Universal "Old School" Way (Compatible with C++/Old Java) | None. Size and Sign agnostic. treats k as a signed displacement.                                                                                      |
|       | k = displacement (+ve for Right, -ve for Left)                                                     | nextIndex = Math.floorMod(i + k, n);                                                  | Modern Java Way (Java 8+)                                 | works for **any** integer `k` (positive, negative, huge, or small).                                                                                   |
|       |                                                                                                    |                                                                                       |                                                           |                                                                                                                                                       |
| IV    | Let `dir` be the direction: `+1` for Right/Next, `-1` for Left/Prev. Let `steps` be the magnitude. | $$\text{new\_index} = \text{Math.floorMod}(i + (\text{dir} \times \text{steps}), n)$$ | **one** formula to rule them all                          |                                                                                                                                                       |
|       |                                                                                                    |                                                                                       |                                                           |                                                                                                                                                       |
One Formula to rule them all : 


----------------
[Minimum Cost Path with Teleportations - LeetCode](https://leetcode.com/problems/minimum-cost-path-with-teleportations/?envType=daily-question&envId=2026-01-29)



-----------
## Map 

How to populate a  map 
- {Integer -> Map}
- {Integer -> PriorityQueue }
```java
// Map: ItemID -> PriorityQueue
    private Map<Integer, PriorityQueue<Bidder>> itemHeaps;
    // Map: ItemID -> UserID -> CurrentBidAmount (Source of Truth)
    private Map<Integer, Map<Integer, Integer>> currentBids;

	currentBids.computeIfAbsent(itemId, k -> new HashMap<>()).put(userId, bidAmount);
	
	// Add to heap (Lazy add: we don't remove the old one)
	itemHeaps.computeIfAbsent(itemId, 
		k -> new PriorityQueue<>((a, b) -> {// PQ Definition
								if (a.bidAmt != b.bidAmt) 
									return Integer.compare(b.bidAmt, a.bidAmt);
								return Integer.compare(b.uId, a.uId);
									}
								)// end Definition
							).offer(new Bidder(userId, bidAmount)
		);  
```

- **`itemHeaps.computeIfAbsent(itemId, k -> new PriorityQueue<>(...))`**: This part ensures a `PriorityQueue` exists for the key. If it doesn't, the lambda creates and returns a new one. The method then returns that queue.
- **`.offer(object)`**: We call this on the _result_ of step 1. This adds the number to the correct queue.
-----

Observance : 
- we need at least 1 egg to determine the breakpoint floor irrespective of floor in the the building ( n >= 1 )
	- In this case, MinMove for n floor building = n : NOTE THIS. 
- how do having more eggs help ?
	- 2 eggs for n=1 is more than sufficient (1 egg for 1 floor building was sufficient)
	- 2 eggs for n=2 do not minimizes #moves than 1 egg for n=2 ? NO, for both case MinMoves=2
	- 2 eggs for n=3 ? Now it begins to help. With 1 egg it would take 3(=n) moves, With 2 eggs we can try binary search - drop from center - 2nd floor (1 move) . If it breaks , then you have 1 more unexplored floor and 1 unbroken egg which requires 1 move. If it doesn't break then also you have 1 more unexplored floor and 2 unbroken eggs which also requires 1 move. MinMoves = 1+1.

| Config (n,k) | Min Moves | Critical Floors                                                 |
| ------------ | --------- | --------------------------------------------------------------- |
| (0,2)        | ?         |                                                                 |
| (1,0)        | ?         |                                                                 |
| (1,1)        | 1         |                                                                 |
| (2,1)(n,1)   | n         | 1,2,3,4,5,6 ...n                                                |
|              |           |                                                                 |
| (1,2)        | 1         | 1                                                               |
| (2,2)        | 2         | 1 --> 2                                                         |
| **(3,2)**    | 2         | **2 {1} --> 3** (binary search works)                           |
| (4,2)        | 3         | 3 {1->2} --> 4 (binary search works)                            |
|              |           | 2 {1} --> 3 --> 4                                               |
| (5,2)        | 3         | 3 {1->2} --> 5 {4} (binary search works)                        |
| **(6,2)**    | **3**     | **3 {1->2} --> 5 {4} --> 6**                                    |
| (7,2)        | 4         | 4 {1->2->3} --> 7 {5-> 6}                                       |
| (8,2)        | 4         | 4 {1->2->3} --> 7 {5-> 6} --> 7                                 |
| (9,2)        | 4         | 4 {1->2->3} --> 7 {5-> 6} --> 9 {8}                             |
| **(10,2)**   | **4**     | **4 {1->2->3} --> 7 {5-> 6} --> 9 {8}--> 10**                   |
|              |           |                                                                 |
| (11,2)       | 5         | 5 {1->2->3->4} --> 9 {6->7->8} --> 11 {10}                      |
| 12           | 5         | 5 {1->2->3->4} --> 9 {6->7->8} --> 12 {10->11}                  |
|              |           | 5 {1->2->3->4} --> 10 {6->7->8->9} --> 12 {11}                  |
|              |           | 5 {1->2->3->4} --> 10 {6->7->8->9} --> 11 --> 12                |
| 13           | 5         |                                                                 |
| 14           | 5         | 5 {1->2->3->4} -->9 { 6->7->8}--> 11 {10}                       |
| **(15,2)**   | 5         | **5 {1->2->3->4} -->9 { 6->7->8}--> 12 {10->11}-> 14{13}-->15** |
| 16           | 6         |                                                                 |
|              |           |                                                                 |


| Floors     | MinMoves for 🥚 🥚 🥚 | MinMoves for 🥚 🥚 |
| ---------- | --------------------- | ------------------ |
| (0,2)      | ?                     |                    |
| (1,0)      | ?                     |                    |
| (1,1)      | 1                     |                    |
| (2,1)(n,1) | n                     |                    |
|            |                       |                    |
| 1          | 1                     | 1                  |
| 2          | 2                     | 2                  |
| 3          | 2                     | 2                  |
| 4          | 3                     | 3                  |
|            |                       |                    |
| 5          | 3                     | 3                  |
| 6          | 3                     | 3                  |
| 7*         | 3                     | 4                  |
| 8          | 1                     |                    |
| 9          | 2                     |                    |
|            | 2                     |                    |
| 10         | 3                     |                    |
|            |                       |                    |
| 11*        | 3                     | 5                  |
| 12*        | 3                     | 5                  |
|            |                       |                    |
|            |                       |                    |
| 13         | 4                     | 5                  |
| 14         | 4                     | 5                  |
| 15         | 5                     | 5                  |
| 16         | 6                     | 5                  |
|            |                       |                    |
[1884. Egg Drop With 2 Eggs and N Floors](https://leetcode.com/problems/egg-drop-with-2-eggs-and-n-floors/)

You are given **two identical** eggs and you have access to a building with `n` floors labeled from `1` to `n`.

You know that there exists a floor `f` where `0 <= f <= n` such that any egg dropped at a floor **higher** than `f` will **break**, and any egg dropped **at or below** floor `f` will **not break**.

In each move, you may take an **unbroken** egg and drop it from any floor `x` (where `1 <= x <= n`). If the egg breaks, you can no longer use it. However, if the egg does not break, you may **reuse** it in future moves.

Return _the **minimum number of moves** that you need to determine **with certainty** what the value of_ `f` is.

--------
What ***strategy*** will you use to minimize the moves required to determine the critical floor(`f`) in a building having h floors using only e=2 eggs ? 

If we have only 1 egg 
	then there is only 1 (trivial) strategy to cover all possible `0<=f <=n` which is to check all starting from 1st floor all the way to n-th floor.

 What if we have more than 1 egg? Where should we begin to throw the first egg and WHY ? 
	 INTUTIVE STRATEGY (might/not be optimal) : Intuition says, use BINARY SEARCH STRATEGY if e=2 i.e 2 partitions each comprising n/2 floors. Once the first egg breaks, we  do exhaustive search of the bottom unexplored part all the way from bottom. 
	 
Each throw partitions floor into 2 parts where we have to apply following strategy : 
 - EXHAUSTIVE/TRIVIAL SEARCH in the bottom part if the egg breaks.
 - RECURSIVE SEARCH in the top part if the egg doesn't break using same strategy applied to parent floor. 

MinMoves in this strategy = 
- 1 (from mid) + (n/2-1)    ; if egg breaks search bottom half         OR     
- 1 (from mid) + (n - n/2) ; if egg doesn't break search upper half

We can observe that if the opponent always choses f = close to the highest floor our minMove might be too high. Because binary search is bottom-biased. if egg breaks it helps but if it doesn't we have to search again in same space.

What if the optimum strategy has the first throw point on some floor other than midpoint ? how to know that floor? Try all using for loop.

int min = inf
for floor=1 to n  :
	int eggBreaks = function
	 int eggDoesntBreak = function

  
 
 Or when e=d, search each n/d blocks from bottom to top. 
 MinMove in this startegy = (n/d-1) + d
 But I say, try every floor
 One way to look at the optimal strategy is that 



------
Let n = 10.

ME : I have picked up the `f`
YOU : Let me think the answer.... 
..... what min number of moves I should say that no matter which f is , the min move to determine is never more than what I answered. 
.... ok the answer is `p`. tell me your `f` now.

ME : f was 5
YOU : see if using

ME : What if I lied and told you f = 8
YOU : I can still detect f using only p moves 

| Floor | Critical floor `f` | Moves |
| ----- | ------------------ | ----- |
| 10    |                    |       |
| 9     |                    |       |
| 8     |                    |       |
| 7     |                    |       |
| 6     |                    |       |
| 5*    |                    |       |
| 4     |                    |       |
| 3     |                    |       |
| 2     |                    |       |
| 1     |                    |       |


Number of Eggs determine the range up to which we can minimize the move such that it is not exhaustive search i.e minMoves < floors