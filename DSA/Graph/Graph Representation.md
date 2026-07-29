- Adjacency 2D Matrix
- Adjacency Lists 

- Adjacency 2D Matrix
	- Unweighted (`boolean[][]`)
		- Directed
		- Undirected
	- Weighted (`int[][]`)
		- Directed
		- Undirected
- Adjacency List 
	- List of Lists : `List<List<Integer>>`
	- Array of Lists : `List<Integer>[]`
	- Map of Lists : `Map<Integer, List<Integer>>`


Types 
- Undirected (symmetry) 
	- Undirected Weighted  
- Directed  
	- Directed Weighted 

| Basis     | Types             |          |
| --------- | ----------------- | -------- |
| Direction | Undirected        | Directed |
| Weight    | No weight(all eq) | Weighted |
| Cyclic    | Acyclic (DAG)     | Cyclic   |
|           |                   |          |

| Algorithm | Ideal Graph Representation |
| --------- | -------------------------- |
|           |                            |
|           |                            |
| SSSP      |                            |
|           |                            |
|           |                            |
| APSP      |                            |
|           |                            |
### Algorithm --> Ideal Graph Representation

Shortest Path (s, t) 
- 

SSSP 
- 

APSP 
- **Floyd-Warshall Algorithm**-> **Adjacency Matrix**.
- 

# Graph Representation

### Matrix representation
Unweighted Graph : `Boolean/bitset matrix boolean[][]`
```
boolean[][] g = new boolean[V][V];
g[u][v] = true; // directed
// undirected
g[u][v] = g[v][u] = true;
```



### Adjacency List
(industry -standard)
Many ways but which one is good for which algorithm?

**Graph representation (Adjacency List):** Each node in a graph can store a list of adjacent nodes. An array of linked lists is an ideal structure for this.

**`List<Integer>[] adjList;`**
- an array named `adjList` where each index holds a `LinkedList` of `Integer` objects.
- Used to represent graphs, where `adjList[i]` contains all neighbors connected to node `i`

`List<Integer>[] listArray = new LinkedList[3];`

```java
class Graph {
    private int vertices;
    private LinkedList<Integer>[] adjList;
    private int maxDepth;

    @SuppressWarnings("unchecked")
    public Graph(int vertices) {
        this.vertices = vertices;
        adjList = new LinkedList[vertices];
        
        for (int i = 0; i < vertices; i++) {
            adjList[i] = new LinkedList<>();
        }
        this.maxDepth = 0;
    }

    public void addEdge(int src, int dest) {
        adjList[src].add(dest);
        // For an undirected graph, add the reverse edge as well
        // adjList[dest].add(src); 
    }

    // Main method to find max depth
    public int findMaxDepth(int startVertex) {
        boolean[] visited = new boolean[vertices];
        // Start DFS from the chosen startVertex with initial depth 1
        dfsUtil(startVertex, visited, 1);
        return maxDepth;
    }

    // Recursive DFS utility function
    private void dfsUtil(int currentVertex, boolean[] visited, int currentDepth) {
        visited[currentVertex] = true;
        
        // Update the global maximum depth found so far
        if (currentDepth > maxDepth) {
            maxDepth = currentDepth;
        }

        // Recur for all the unvisited neighbors
        for (int neighbor : adjList[currentVertex]) {
            if (!visited[neighbor]) {
                // Pass currentDepth + 1 to the recursive call
                dfsUtil(neighbor, visited, currentDepth + 1);
            }
        }
        
        // Note: For finding the *overall* maximum depth from *any* starting node
        // in a general graph (not a tree), this visited array approach might not be
        // sufficient, as a node could be reachable via a longer path from another
        // component. For a single component/tree, this works fine.
    }
}
```


An **array of lists** (commonly called an ==**adjacency list**==) is one of the most efficient data structures for graph representation in Java.
While you can technically use a raw array of lists (`List<Integer>[] adj = new List[V]`), Java dislikes arrays of generic types and will throw "type safety" compiler warnings. To write modern, clean code, you should use a **nested list** (`ArrayList<ArrayList<Integer>>`) instead.
```java
import java.util.ArrayList;
import java.util.List;

class Graph {
    // A list where each index represents a vertex, containing a list of adjacent neighbors
    private List<List<Integer>> adjList;
    private int numVertices;

    // Constructor to initialize the graph structure
    public Graph(int numVertices) {
        this.numVertices = numVertices;
        this.adjList = new ArrayList<>(numVertices);
        
        // Initialize an empty list for every single vertex to avoid NullPointerException
        for (int i = 0; i < numVertices; i++) {
            this.adjList.add(new ArrayList<>());
        }
    }
    // Method to add an edge to the graph
    public void addEdge(int source, int destination, boolean isDirected) {
        // Add edge from source to destination
        adjList.get(source).add(destination);
        
        // If graph is undirected, add the reverse link too
        if (!isDirected) {
            adjList.get(destination).add(source);
        }
    }
    // Method to print out the adjacency list
    public void printGraph() {
        for (int i = 0; i < numVertices; i++) {
            System.out.print("Vertex " + i + " is connected to: ");
            System.out.println(adjList.get(i));
        }
    }
}
```

#### 1. Undirected Graph
- **Input:** `edges[i] = [u, v]`
- **Key Logic:** Since it is undirected, a connection from `u` to `v` implies a connection from `v` to `u`. You must add the edge to **both** lists.
- **Data Structure:** `List<List<Integer>>`
```Java
// n = number of nodes
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

for (int[] edge : edges) {
    int u = edge[0];
    int v = edge[1];
    
    adj.get(u).add(v);
    adj.get(v).add(u); // Symmetry
}
```
#### 2. Undirected Weighted Graph
- **Input:** `edges[i] = [u, v, w]` (w = weight)
- **Key Logic:** You still add edges symmetrically, but now you need to store the weight.
- **Data Structure:** `List<List<int[]>>` (Each entry is a generic `int[]{neighbor, weight}`).
```Java
List<List<int[]>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

for (int[] edge : edges) {
    int u = edge[0];
    int v = edge[1];
    int w = edge[2];
    
    // Store as {neighbor, weight}
    adj.get(u).add(new int[]{v, w});
    adj.get(v).add(new int[]{u, w}); 
}
```
#### 3. Directed Graph
- **Input:** `edges[i] = [u, v]` (Direction: $u \to v$)
- **Key Logic:** The edge is a "one-way street". You only add `v` to `u`'s list.
- **Data Structure:** `List<List<Integer>>`
```Java
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

for (int[] edge : edges) {
    int u = edge[0];
    int v = edge[1];
    
    adj.get(u).add(v); 
    // Do NOT add u to v's list
}
```
#### 4. Directed Weighted Graph
Variations 
1. `List<List<int[]>>` or  `List<List<Edge>>` (Custom Class)
2. `Map<Integer, Map<Integer, Integer>>`  
3. `List<Map<Integer, Integer>>`

#### 4.1 `List<List<int[]>>` or  `List<List<Edge>>`

- **Input:** `edges[i] = [u, v, w]` (Direction: $u \to v$ with weight $w$)
- **Key Logic:** One-way street with a cost.
- **Data Structure:** `List<List<int[]>>` or `List<List<Edge>>` (Custom Class)
- **Suitable for** : Dijkstra 
```Java
public buildGraph(int[][] edges) {
	List<List<int[]>> adj = new ArrayList<>(); // class level var (Global)
	
	for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
	
	for (int[] edge : edges) {
	    int u = edge[0];
	    int v = edge[1];
	    int w = edge[2];
	    
	    adj.get(u).add(new int[]{v, w}); // Store as {neighbor, weight}
	}
}

public int getWeight(int source, int destination) {
    // 1. Get the list of edges starting from 'source'
    List<int[]> neighbors = adj.get(source);
    
    // 2. Iterate through them to find the specific 'destination'
    for (int[] edge : neighbors) {
        int targetNode = edge[0]; // Index 0 is Destination
        int weight = edge[1];     // Index 1 is Weight
        
        if (targetNode == destination) {
            return weight;
        }
    }
    // 3. Return -1 (or throw exception) if edge not found
    return -1; 
}
```

**Performance Note**
- **Time Complexity:** $O(Degree)$ — linear relative to the number of neighbors the source has.
- **Comparison:** This is slower than the Map approach ($O(1)$) for lookups, but efficient enough for standard graph traversals (BFS/Dijkstra) **where you iterate all neighbors anyway**.

`List<List<Edge>>` (Custom Class)

```Java
// Cleaner for production code
record Edge(int to, int weight) {}
adj.get(u).add(new Edge(v, w));
```

Use a **Map of Maps** to represent a weighted graph, especially when you need $O(1)$ access to check if a specific edge exists or to retrieve its weight.

Instead of a `List` of pairs, you map the **Source Node** to another Map, which maps **Destination Nodes** to **Weights**.
- **Key Logic:** `adj.get(u).get(v)` returns the weight of the edge $u \to v$.
- **Data Structure:** `Map<Integer, Map<Integer, Integer>>`  

```Java
import java.util.*;

class Graph {
    // Map<Source, Map<Destination, Weight>>
    Map<Integer, Map<Integer, Integer>> adj = new HashMap<>();

    public void buildGraph(int[][] edges) {
        for (int[] edge : edges) {
            int u = edge[0]; // Source
            int v = edge[1]; // Destination
            int w = edge[2]; // Weight

            // 1. Ensure the source node 'u' exists in the outer map
            adj.putIfAbsent(u, new HashMap<>());
            // 2. Add the edge and weight to the inner map
            adj.get(u).put(v, w);
            
            // 1+2
            adj.computeIfAbsent(u, k -> new HashMap<>()).put(v, w);
        }
    }
    
    public void iterateGrah() {
	    // adj is Map<Integer, Map<Integer, Integer>>
		adj.forEach((u, neighbors) -> {
		    System.out.println("Source Node: " + u);
		    
		    neighbors.forEach((v, w) -> {
		        System.out.println("  -> Destination: " + v + ", Weight: " + w);
		    });
		});
		
		// Traditional `for` loop (better if you need `break`/`continue` or exception handling):
		for (Map.Entry<Integer, Map<Integer, Integer>> entry : adj.entrySet()) {
		    int u = entry.getKey();
		    Map<Integer, Integer> edges = entry.getValue();
		
		    for (Map.Entry<Integer, Integer> edge : edges.entrySet()) {
		        int v = edge.getKey();
		        int w = edge.getValue();
		        // Process edge u -> v (weight w)
		    }
		}
    }

    // Helper: Get weight of edge u -> v (Returns -1 if no edge exists)
    public int getWeight(int u, int v) {
        if (!adj.containsKey(u) || !adj.get(u).containsKey(v)) {
            return -1; 
        }
        return adj.get(u).get(v);
    }
}
```
### Use Case Advice
- **Use the HashMap approach** if your algorithm frequently asks: _"Is there a direct connection between X and Y?"_ or _"What is the weight of the edge X->Y?"_ (e.g., checking constraints in a path).
- **Use the List approach** for standard BFS/DFS traversals where you just need to iterate _all_ neighbors of a node regardless of who they are.

`List<Map<Integer, Integer>>`
This is the **"Hybrid Adjacency List"**. It is extremely popular in Competitive Programming.
- **Structure:** An Array (or List) where each index holds a `Map`.  
- **Why it's great:**
    - **Node Access:** $O(1)$ array access (faster/lighter than an outer HashMap).
    - **Edge Access:** $O(1)$ lookup for weights.
    - **Ideal for:** Dense node labels `0` to `n-1`.
```Java
// 1. Initialize
List<Map<Integer, Integer>> adj = new ArrayList<>();
for(int i=0; i<n; i++) adj.add(new HashMap<>());

// 2. Build Graph
for(int[] edge : edges) {
    // No need for computeIfAbsent because the Map already exists at index u
    adj.get(edge[0]).put(edge[1], edge[2]);
}

// 3. Query
int weight = adj.get(u).getOrDefault(v, -1);
```



### Dijkstra 

```Java
// 1. Graph Representation: index = source, int[] = {destination, weight}
List<List<int[]>> adj = new ArrayList<>();

public int dijkstra(int n, int[][] edges, int start, int end) {
    // Build Graph
    for(int i=0; i<n; i++) adj.add(new ArrayList<>());
    for(int[] e : edges) adj.get(e[0]).add(new int[]{e[1], e[2]});

    // 2. Priority Queue: Stores {node, current_dist}
    // Sorts by distance (ascending)
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    
    // 3. Distance Array
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    
    // Initialize
    dist[start] = 0;
    pq.offer(new int[]{start, 0});
    
    while(!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int d = curr[1];
        
        // Optimization: Skip stale entries
        if(d > dist[u]) continue;
        
        // Iterate neighbors (Fastest with List<List<int[]>>)
        for(int[] edge : adj.get(u)) {
            int v = edge[0];
            int weight = edge[1];
            
            if(dist[u] + weight < dist[v]) {
                dist[v] = dist[u] + weight;
                pq.offer(new int[]{v, dist[v]});
            }
        }
    }
    
    return dist[end] == Integer.MAX_VALUE ? -1 : dist[end];
}
```
**Why?**
1. **Iteration Speed:** Dijkstra's core operation is _iterating_ through all neighbors of the current node (`u`). Iterating an `ArrayList` is significantly faster (CPU cache-friendly) than iterating a `HashMap`.
2. **Memory:** It uses minimal memory ($O(V + E)$).
3. **No Random Access Needed:** Dijkstra never asks "What is the weight of edge A->B?" in isolation. It only asks "Give me _all_ edges starting from A."
Why NOT the others?
- **Adjacency Matrix (`int[][]`):** Takes $O(V^2)$ space. Iterating neighbors takes $O(V)$ time (scanning 0 to N) even if only 2 neighbors exist. This kills performance on sparse graphs.
- **Adjacency Map (`Map<Map...>`):** While powerful, `HashMap` iteration is slower due to entry set overhead and lack of memory locality. It adds unnecessary overhead since we don't need $O(1)$ edge lookups.



## Floyd-Warshall Algorithm

### 1. Initialization (The Setup)
You must initialize the matrix carefully to handle "Infinity" without causing integer overflow during addition.
**Standard `INF` Value:** Use `1_000_000_000` (1e9) or `Integer.MAX_VALUE / 2`.
- _Why?_ Because `Integer.MAX_VALUE + positive_weight` will wrap around to a negative number, breaking your logic.
```Java
public void floydWarshall(int n, int[][] edges) {
    // 1. Define Safe Infinity
    final int INF = (int) 1e9; 
    
    // 2. Initialize Matrix
    int[][] dist = new int[n][n];
    
    // Fill with INF and set Diagonal to 0
    for (int i = 0; i < n; i++) {
        Arrays.fill(dist[i], INF);
        dist[i][i] = 0;
    }

    // 3. Populate Weights from Input
    // Assuming edges[i] = [u, v, w]
    for (int[] edge : edges) {
        int u = edge[0];
        int v = edge[1];
        int w = edge[2];
        
        dist[u][v] = w;
        
        // If Undirected, add: dist[v][u] = w;
        // If Multiple edges exist between u,v, take min: 
        // dist[u][v] = Math.min(dist[u][v], w);
    }

    // ... Run Algorithm (See Below) ...
}
```

### 2. The Core Algorithm ($O(N^3)$)
The beauty of Floyd-Warshall is its simplicity. It tries to improve the path between every pair `i` and `j` by going through an intermediate node `k`.
```Java
    // 4. The Triple Loop
    // Order matters: 'k' (intermediate) MUST be the outer loop
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                
                // Optimization: Skip if intermediate path is unreachable
                if (dist[i][k] == INF || dist[k][j] == INF) continue;
                
                // Relaxation
                if (dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }
    
    // Result: dist[i][j] now holds the shortest path from i to j
```
### Critical Checklist for Interviews
1. **Outer Loop `k`**: If you put `k` inside, the algorithm fails. `k` represents the "set of allowed intermediate nodes" expanding from $\{0\}$ to $\{0...n-1\}$.
2. **Overflow Safety**: Always check `if (dist[i][k] == INF ...)` before adding, or use a Safe INF value.
3. **Negative Cycles**: Floyd-Warshall can detect negative cycles. If `dist[i][i] < 0` after the algorithm finishes, a negative cycle involving node `i` exists.

----

Beyond the popular `List<Integer>[]` array of lists, here are the most effective alternative representations for competitive programming:
## 1. The Head/Next Array (Forward Star Representation)

This is an old-school competitive programming trick. Instead of objects, arrays of objects, or lists, you use three flat primitive arrays (`head[]`, `to[]`, `next[]`) and an edge counter (`edgeCount`).

```java
int MAX_VERTICES = 100005;
int MAX_EDGES = 200005;

int[] head = new int[MAX_VERTICES];
int[] to = new int[MAX_EDGES];
int[] next = new int[MAX_EDGES];
int edgeCount = 0;

// Initialize head array with -1 at the start of your solution
Arrays.fill(head, -1);

void addEdge(int u, int v) {
    to[edgeCount] = v;
    next[edgeCount] = head[u];
    head[u] = edgeCount++;
}

// How to traverse neighbors of vertex 'u':
void traverse(int u) {
    for (int e = head[u]; e != -1; e = next[e]) {
        int neighbor = to[e];
        // Do processing here
    }
}
```

Use code with caution.
- **Why CP loves it**: It is the **fastest possible representation in Java**. It allocates all memory at compile-time up front. It uses pure primitive arrays, which bypasses all object creation and memory allocation bottlenecks.
- **When to use**: Extremely tight time limits on platforms like Codeforces or when a problem contains up to 10⁵ to 10⁶ edges.
## 2. Flat 1D Array Tracking Indegrees (For Trees or Functional Graphs)

Many LeetCode problems feature trees or graphs where each node has exactly **one** outgoing edge (e.g., Directed Trees, Disjoint Set Union problems, or functional graphs). In these scenarios, you do not need any list structure at all.

```java
// If every node 'i' points to exactly one parent/next node
int[] parent = new int[n]; 

// Example LeetCode problem style input parsing:
for (int i = 0; i < edges.length; i++) {
    parent[edges[i][0]] = edges[i][1];
}
```

Use code with caution.
- **Why CP loves it**: It takes O(1) memory per node and enables trivial, hyper-fast array indexing traversal like `curr = parent[curr]`.
- **When to use**: Tree ancestors, Disjoint Set Union (DSU / Union-Find), or finding cycles in functional graphs (e.g., LeetCode 2127).
## 3. Bitset Adjacency Matrix (The `java.util.BitSet` Array)

While a standard `boolean[][]` matrix uses too much memory for large graphs, an array of `BitSet` objects compresses boolean states down to individual bits. It represents an adjacency list where index i holds a bitmask of all its neighbors.

```java
BitSet[] adj = new BitSet[n];
for (int i = 0; i < n; i++) adj[i] = new BitSet(n);

// To add an edge
adj[u].set(v);

// Check if edge exists in O(1)
if (adj[u].get(v)) { ... }

// Find common neighbors between two vertices in ultra-fast O(N/64) bitwise operations:
BitSet common = (BitSet) adj[u].clone();
common.and(adj[v]); 
int commonNeighborCount = common.cardinality();
```

Use code with caution.
- **Why CP loves it**: It allows you to perform graph intersections, reachability queries, and transitive closures at lightning speeds because the JVM executes the operations using 64-bit word instructions under the hood.
- **When to use**: Dense graphs where N ≤ 5000, or problems requiring you to find intersections of neighbors or reachability (e.g., LeetCode 841, 1466).
## 4. Packaged 64-bit Long Array (For Weighted Graphs)

When a LeetCode problem requires a weighted graph, people usually resort to creating an `Edge` object or storing pairs as an `int[]` of size 2. To avoid object overhead in Dijkstra's or Prim's algorithm, you can pack both the neighbor ID and the edge weight into a single primitive `long` variable.

```java
// Top 32 bits = weight, Bottom 32 bits = destination vertex ID
long pack(int weight, int targetVertex) {
    return ((long) weight << 32) | (targetVertex & 0xFFFFFFFFL);
}

// To read them back out:
int weight = (int) (packedValue >> 32);
int target = (int) packedValue;
```

Use code with caution.
- **Why CP loves it**: Allows you to use standard `List<Long>[]` or `long[]` structures to hold both destination and weight without generating millions of pair array instances (`new int[]{v, w}`) that trigger LeetCode's Time Limit Exceeded (TLE) errors.