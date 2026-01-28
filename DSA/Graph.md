

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

# Graph Representation

### Matrix representation
limited use

### Adjacency List
(industry -standard)
Many ways but which one is good for which algorithm?
- 
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