## DSU / Union-Find

### What it is DSU / UF
- Same thing: DSU = **Disjoint Set Union**, Union-Find = **the two operations**
- Groups n elements into disjoint sets
- Supports `union(x,y) `and `find(x)` in $O(α(n)) ≈ O(1)$ amortized

### When to reach for it
- "Connected components", "provinces", "islands", "clusters"
- "Do `x` and `y` belong to the same group?"
- "Merge groups with transitive relationship"
- "Detect cycle in undirected graph" (redundant edge = same root before union)
- "Minimum spanning tree" (Kruskal: sort edges by weight, skip if same root)
```java
    class DSU_Just_Builder {

        int[] parent;

        public DSU_Just_Builder(int n) {
            parent = new int[ n];
            for (int i = 0; i <n; ++i) {
                parent[i] = i;
            }
        }

        public int find(int x) {
            return x == f[x] ? x : (f[x] = find(f[x]));
        }

        public void merge(int x, int y) {
            f[find(x)] = find(y);
        }
    }
```

more detail

```java
int[] parent, rank;
void init(int n){
    parent=new int[n]; rank=new int[n];
    for(int i=0;i<n;i++) parent[i]=i;
}
int find(int x){
    if(parent[x]!=x) parent[x]=find(parent[x]);
    return parent[x];
}
boolean union(int a, int b){
    a=find(a); b=find(b);
    if(a==b) return false;
    if(rank[a]<rank[b]){int t=a;a=b;b=t;}
    parent[b]=a;
    if(rank[a]==rank[b]) rank[a]++;
    return true;
```
### The internal representation — the parent array

`parent[]` :  Each element points to a **parent**. The **root** (element pointing to itself) is the group's representative. Two elements are in the same group if and only if they share the same root.
```java
int px = find(x), py = find(y); // assume shorter
if (px == py) // implies x & y belong to same (connected) component i.e. have same 
```

### The two function : `union(x,y)` & `find(x)`
#### bool union(x, y) : connect two nodes *iff* they aren't

#### int find(x) : group leader - a self pointing root 
```java
int find(int x) {// find The Ultimate Root: a node that points to itself
	if (parent[x] != x) 
		parent[x] = find(parent[x]); // compresses path: optimizes later queries
	return parent[x];
}
```
- determines which **set** or **group** a specific element belongs to. It does this by returning the **representative** (often called the "root" or "leader") of that element's set.
* Does find(x) finds parent of x ? No. it doesn't necessarily returns `parent of x` . Only after you use **path compression** and call `find(x)` on every node in a set, the tree completely flattens. Every single node in that group will point directly to the ultimate root. At that exact point, `find(x)` will equal `parent[x]` for all elements.
* The `find(x)` function does not just find the _immediate_ parent of `x`. Instead, it finds the **ultimate root** (the top-most ancestor) of the set that `x` belongs to.
```
        5   ← 5 is representative ("root") of this set
      / |  \
     6  4    0      find(0) = parent[0] = 5
             |
             3      find(4) ≠ parent[4]=0
            / \     find(4) = 5 i.e. "root"
           1   2
indexes    0    1   2   3  4  5   6 
parent[] = {5,  3,  3,  0, 5, 5*, 5} // 5 is root

After find(2) :
parent[] = {5,  3, 5*, 5*, 5, 5*, 5} // all nodes above 2 (2,3) are compressed
         parent[1] is stil 3 not 5 !!!

After find(1) : 
parent[] = {5, 5*,  5,  5, 5, 5*, 5} 

only now, find(x) = parent[x]
```

Here is how the search behavior breaks down inside the `parent[]` array:
- **The Immediate Parent:** `parent[x]` only gives you the node directly above `x`. If the tree is deep, this is not the root.
- **The Ultimate Root:** The root is a node that points to itself (`parent[root] == root`).
- **The `find(x)` Process:** The function follows the chain of parents (`parent[x]`, `parent[parent[x]]`, etc.) until it hits the node where `parent[root] == root`. It then returns that ultimate root. 
### Template (Java) — memorize

```java
class DSU {
    int[] parent, 
	      rank,// tracks each root's height 
	           // The rule: always attach the shorter tree under the taller tree
	      size;
	// rank[i] = HEIGHT of the tree rooted at i → used for EFFICIENCY 
	// size[i] = NODE COUNT of the component at i → used to ANSWER QUESTIONS
    int components;
    
    DSU(int N) {
        parent = new int[N]; rank = new int[N]; size = new int[N];
        components = N;
        for (int i = 0; i < N; i++) { parent[i] = i; size[i] = 1; }
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    boolean union(int x, int y) {
	    // ── STEP 1: Find the group representative (root) of each node ────── 
	    // find() chases parent pointers up until it hits a self-loop. 
	    // px = root of x's entire component (not x itself!) 
	    // py = root of y's entire component 
	    // 
	    // Example: if the chain is 3→1→0 then find(3) = 0 
	    // We operate on ROOTS, not on the input nodes directly.
        int px = find(x), // assume TALLER
            py = find(y); // assume shorter
		
		// ── STEP 2: Early exit — already in the same group ───────────────── 
		// If px == py, x and y already share the same root. 
		// No merge needed. Return false = "nothing changed." 
		// 
		// In graph problems this is your CYCLE DETECTION check: 
		//   if union() returns false → this edge connects two already-connected 
		//   nodes → adding it would create a cycle.
        if (px == py) return false;

        // ── STEP 3: Guarantee px holds the TALLER root ───────────────────── 
        // rank[i] = height of the tree rooted at i. 
        // We always want to attach the SHORTER tree under the TALLER tree. 
        // So we need px = the taller root, py = the shorter root. 
        // 
        // If px is currently shorter than py, swap them. 
        // After this block: rank[px] >= rank[py] is ALWAYS true. 
        // 
        // Example: rank[px]=1, rank[py]=2 
        // → px is shorter, py is taller 
        // → swap so that px=taller root, py=shorter root 
        // → we're about to do parent[py]=px, i.e., attach shorter under taller ✓
        if (rank[px] < rank[py]) { int t=px; px=py; py=t; } // swap
        
        // ── STEP 4: The actual merge ──────────────────────────────────────── 
        // py (shorter/equal tree) becomes a child of px (taller/equal tree).
        // Every single node in py's component now has px as its ultimate root. 
        // This ONE line merges potentially millions of nodes.
        parent[py] = px; // parent[short] = tall
        
        // ── STEP 5: Update component size ────────────────────────────────── 
        // px is now root of the COMBINED component.
        // Its size = its old size + everything that came from py's component.
        // 
		// py's size[] entry is now stale — we never touch it again.
		// py is no longer a root, so size[py] is irrelevant.
        // 
        // To query component size later: size[find(anyNode)]
        size[px] += size[py];
        
        // ── STEP 6: Update rank — ONLY when heights were equal ───────────── 
        // Three possible cases after the merge: 
        //
		// CASE A — px taller: rank[px]=2, rank[py]=1 
		// We attached height-1 tree under height-2 tree. 
		// New height = max(2, 1+1) = 2. rank[px] stays 2. No change. 
		// 
		// CASE B — equal heights: rank[px]=1, rank[py]=1 
		// We attached height-1 tree under another height-1 tree. 
		// New height = 1+1 = 2. rank[px]++ → 2. Increment! 
		// 
		// CASE C — two single nodes: rank[px]=0, rank[py]=0 
		// Merging two leaves. 
		// New height = 1. rank[px]++ → 1. Increment! 
		// 
		// Summary: rank ONLY grows when two equal-height trees merge.
        if (rank[px] == rank[py]) rank[px]++;
        
        // ── STEP 7: Track number of components ───────────────────────────── 
        // Two groups became one. Decrement the counter.
        components--;
        
        return true;// merge happened — caller may use this signal
    }

    boolean connected(int x, int y) { return find(x) == find(y); }
    int     sizeOf(int x)           { return size[find(x)];      }
    int     countComponents()       { return components;         }
}
```
### Key patterns
1. Count components → initialize `components=n`, decrement on successful union 
2. Cycle detection → union `returns false` = edge is redundant 
3. Kruskal MST → sort edges by weight, union if not already connected 
4. Virtual node → create node `(n+k)` as proxy for a group 
### Complexity
- Build:  $O(n)$
- Find:   $O(α(n)) ≈ O(1)$
- Union:  $O(α(n)) ≈ O(1)$
- Space:  $O(n)$
### Problems : [Problem list union find](https://leetcode.com/problem-list/union-find/) 
- [LC 547 ](https://leetcode.com/problems/number-of-provinces) Number of Provinces          (warmup)
- [LC 684](https://leetcode.com/problems/redundant-connection)  Redundant Connection         (cycle detection)
- [LC 1319](https://leetcode.com/problems/number-of-operations-to-make-network-connected/) Make Network Connected       (components + edges)
- LC 721  Accounts Merge               (entity grouping)
- LC 1584 Min Cost to Connect Points   (Kruskal MST)

[LC 547 ](https://leetcode.com/problems/number-of-provinces) Number of Provinces  
```java
public int findCircleNum(int[][] isConnected) {
        int n = isConnected.length;
        DSU dsu = new DSU(n);

        for(int i=0; i<n; i++) {
            for(int j=0 ; j<n ; j++ ) {
                if(isConnected[i][j] == 1) {
                    dsu.union(i, j);
                }
            } 
        }

        return dsu.components ; // dsu.countComponents()
    }
```

[LC 684](https://leetcode.com/problems/redundant-connection)  Redundant Connection  
```java
public int[] findRedundantConnection(int[][] edges) {
        int n = edges.length; // The graph is represented as an array edges of length n

        DSU dsu = new DSU(n+1); // 1-based index

        for (int[] edge : edges) {
            int u = edge[0]; // 1-based index
            int v = edge[1];
            if (!dsu.union(u, v)) {
                return edge; // this edge is redundant — cycle found
            }
        }
        return edges[n]; // any dummy array
    }
```

 [LC 1319](https://leetcode.com/problems/number-of-operations-to-make-network-connected/) Make Network Connected  
```java
    public int makeConnected(int n, int[][] connections) {
        if(connections.length < n-1) return -1; 

        DSU dsu = new DSU(n);
        for(int[] edge : connections) {
            dsu.union(edge[0], edge[1]);
        }
        return dsu.countComponents() - 1;
    }
```
### When NOT to use DSU

|Situation|Use instead|
|---|---|
|**Directed** graph connectivity|DFS/BFS, Tarjan's SCC|
|Need the **actual path** between nodes|BFS/DFS|
|Components can **split** (undo union)|Link-Cut Tree (rare in contests)|
|Shortest path / distances|Dijkstra, BFS|
|Need to know **which** component each node is in (stable labels)|BFS/DFS with coloring|

---
### Practice problems (in order)

| #                                                                                        | Problem                        | Why it's here                   |
| ---------------------------------------------------------------------------------------- | ------------------------------ | ------------------------------- |
| [LC 547 ](https://leetcode.com/problems/number-of-provinces)                             | Number of Provinces            | Pure DSU warmup                 |
| [LC 684](https://leetcode.com/problems/redundant-connection)                             | Redundant Connection           | Cycle detection pattern         |
| LC 1971                                                                                  | Find if Path Exists            | Simplest connectivity check     |
| [LC 1319](https://leetcode.com/problems/number-of-operations-to-make-network-connected/) | Make Network Connected         | Components + edge count         |
| LC 200                                                                                   | Number of Islands              | DSU on a grid                   |
| LC 721                                                                                   | Accounts Merge                 | Grouping non-graph entities     |
| LC 1584                                                                                  | Min Cost to Connect All Points | Kruskal's MST                   |
| LC 839                                                                                   | Similar String Groups          | Hard grouping, good contest sim |
## Input transformation for DSU-UF applicability

**Array-based DSU requires integer indices in 0..n-1. But there are several ways to handle input that isn't in that form — and sometimes you skip the transformation entirely with a HashMap DSU.**

```
What are your node identifiers?
│
├─ Already 0..n-1 integers?
│   └─ Use DSU directly. No transformation.
│
├─ 1-indexed integers (1..n)?
│   └─ DSU(n+1), use nodes as-is.
│
├─ 2D grid cells (r, c)?
│   └─ index = r * cols + c. DSU(rows * cols).
│
├─ Strings or arbitrary labels?
│   └─ HashMap<String, Integer> → assign IDs → DSU(id count).
│
├─ Large sparse integers, all known upfront?
│   └─ Coordinate compress → DSU(unique count).
│
└─ Large sparse integers, unknown/streaming?
    └─ HashMap DSU — no transformation needed.
```

## DSU — input transformation patterns

### 1-indexed nodes
`DSU(n+1)`, use indices 1..n directly
### 2D grid  
`index = i * cols + j → DSU(rows * cols)`

```java
// cell (r, c) → index r*cols + c
DSU dsu = new DSU(rows * cols);

// union cell (r1,c1) with (r2,c2):
dsu.union(r1 * cols + c1, r2 * cols + c2);

// check if (0,0) and (2,3) are connected:
dsu.connected(0, 2 * cols + 3);
```
LC 200 (Number of Islands), LC 305, LC 1202.
### String keys
`Map<String,Integer> id → assign counter++ on first touch → DSU(counter)`
### Sparse integers (known upfront)
`Arrays.sort(vals) → Map<val, rank> → DSU(vals.length)`
### Sparse integers (unknown/dynamic) → HashMap DSU
```java 
class HashDSU { 
    Map<Integer,Integer> parent = new HashMap<>(); 
    int find(int x) { 
        parent.putIfAbsent(x, x); 
        if (parent.get(x) != x) parent.put(x, find(parent.get(x)));
        return parent.get(x); 
    } 
    boolean union(int x, int y) {
        int px=find(x),py=find(y);
        if(px==py) return false;
        parent.put(px,py); return true;
    }
}
```
### Virtual nodes
Real nodes: 0..n-1 | Virtual group proxies: n..n+k-1
Union each member to its group's virtual node instead of pairwise
