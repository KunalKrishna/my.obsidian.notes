#graph #java #leetcode #algorithms #cheatsheet

# Graph Representation — Quick Reference

> [!tip] Purpose of this note Given an algorithm you already know you need, jump straight to the representation it wants, copy the declare/initialize/populate block, and stop reinventing this every time.

---

## 📑 Index

- [[#🎯 TL;DR — Algorithm → Representation Lookup Table]]
- [[#🧭 Two Independent Properties]]
- [[#1️⃣ 2D Matrix]]
    - [[#Undirected + Unweighted (Matrix)]]
    - [[#Undirected + Weighted (Matrix)]]
    - [[#Directed + Unweighted (Matrix)]]
    - [[#Directed + Weighted (Matrix)]]
- [[#2️⃣ Edge List (M×N array)]]
    - [[#Unweighted Edge List (M×2)]]
    - [[#Weighted Edge List (M×3)]]
- [[#3️⃣ Adjacency List]]
    - [[#Array of Lists — Unweighted]]
    - [[#Array of Lists — Weighted]]
    - [[#List of Lists — Unweighted]]
    - [[#List of Lists — Weighted]]
    - [[#Map of Lists — Unweighted]]
    - [[#Map of Lists — Weighted]]
- [[#4️⃣ DSU / Union-Find]]
- [[#⚠️ Java Declaration Gotchas]]
- [[#🔄 Conversion Cheatsheet]]
- [[#📚 Algorithm Deep Dives]]
    - [[#BFS]]
    - [[#DFS]]
    - [[#Dijkstra (SSSP — non-negative weights)]]
    - [[#Bellman-Ford (SSSP — negative weights allowed)]]
    - [[#Floyd-Warshall (APSP)]]
    - [[#Kruskal's MST]]
    - [[#Prim's MST]]
    - [[#Topological Sort]]
    - [[#Cycle Detection]]
    - [[#Union-Find Applications]]
    - [[#SCC — Tarjan's / Kosaraju's]]

---

## 🎯 TL;DR — Algorithm → Representation Lookup Table

| Algorithm                            | Best format                                                             | Why                                                           | Complexity        |
| ------------------------------------ | ----------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------- |
| BFS / DFS                            | [[#Array of Lists — Unweighted]]                                        | Need O(deg) neighbour iteration                               | O(V+E)            |
| Dijkstra (SSSP)                      | [[#Array of Lists — Weighted]] (+ min-heap)                             | Weighted, sparse-friendly with heap                           | O((V+E) log V)    |
| Bellman-Ford (SSSP, neg. weights)    | [[#Weighted Edge List (M×3)]]                                           | Just relax every edge V−1 times, linear scan                  | O(V·E)            |
| Floyd-Warshall (APSP)                | [[#Undirected + Weighted (Matrix)]] / [[#Directed + Weighted (Matrix)]] | Algorithm _is_ a matrix DP                                    | O(V³)             |
| Kruskal's MST                        | [[#Weighted Edge List (M×3)]] + [[#4️⃣ DSU / Union-Find]]               | Sort edges, union-find for cycle check                        | O(E log E)        |
| Prim's MST                           | [[#Array of Lists — Weighted]] (+ min-heap), or Matrix if dense         | Grow a tree from adjacency, heap picks cheapest frontier edge | O(E log V) sparse |
| Topological Sort (Kahn's)            | [[#Array of Lists — Unweighted]] + `int[] indegree`                     | BFS-based, needs indegree array                               | O(V+E)            |
| Topological Sort (DFS)               | [[#Array of Lists — Unweighted]]                                        | Postorder + reverse                                           | O(V+E)            |
| Cycle detection (undirected)         | [[#Weighted Edge List (M×3)]]/[[#Unweighted Edge List (M×2)]] + DSU     | `union()` returns false ⇒ cycle                               | O(E·α(V))         |
| Cycle detection (directed)           | [[#Array of Lists — Unweighted]]                                        | DFS + 3-color (white/gray/black)                              | O(V+E)            |
| Number of connected components       | [[#4️⃣ DSU / Union-Find]]                                               | Count distinct roots after unions                             | O(E·α(V))         |
| SCC — Tarjan's                       | [[#Array of Lists — Unweighted]]                                        | Single DFS pass, low-link values                              | O(V+E)            |
| SCC — Kosaraju's                     | [[#Array of Lists — Unweighted]] + reversed adj list                    | Two DFS passes, needs transpose graph                         | O(V+E)            |
| Max flow (Edmonds-Karp)              | [[#Array of Lists — Weighted]] (capacity + residual) or Matrix          | Needs reverse/residual edges                                  | O(V·E²)           |
| Grid graphs (0/1 BFS, islands, etc.) | Implicit — no structure, compute neighbours inline                      | Saves O(V+E) allocation entirely                              | O(rows·cols)      |

> [!tip] One-line mental model **Iterating neighbours of one vertex → Adjacency List. Iterating all edges at once → Edge List. Need O(1) weight lookup between any two vertices, or all-pairs DP → Matrix. Just connectivity, no paths → DSU.**

---

## 🧭 Two Independent Properties

Every graph has two orthogonal properties. Every representation below has (up to) 4 variants from crossing them:

|                | Unweighted                     | Weighted                        |
| -------------- | ------------------------------ | ------------------------------- |
| **Undirected** | edge has no direction, no cost | edge has no direction, has cost |
| **Directed**   | edge u→v only, no cost         | edge u→v only, has cost         |

- **Undirected** ⇒ matrix is always symmetric (`m[u][v] == m[v][u]`); adjacency lists add the edge on **both** sides; edge list conceptually stores the pair once (order doesn't matter).
- **Directed** ⇒ matrix need not be symmetric (it _can_ be, if for every u→v a v→u also exists — that's just a coincidence of the input, not a rule); adjacency lists add the edge **once**, on the source side only.
- **DSU** and **edge list** don't really have "declaration" variants for direction — direction only changes how you _populate_ them (add edge once vs. twice), not how you _declare_ them. Weight just adds a 3rd column/field.

---

## 1️⃣ 2D Matrix

Use `n` = number of vertices (assumed 0-indexed contiguous ints `0..n-1`).
### Undirected + Unweighted (Matrix)
```java
// Declare
int[][] matrix;      // or: boolean[][] matrix;

// Initialize
int n = ...;
matrix = new int[n][n];        // all 0 by default = "no edge"
// boolean version:
// matrix = new boolean[n][n]; // all false by default

// Populate — add edge (u, v)
matrix[u][v] = 1;
matrix[v][u] = 1;   // symmetric because undirected

// boolean version
// matrix[u][v] = true;
// matrix[v][u] = true;
```

### Undirected + Weighted (Matrix)
```java
// Declare
int[][] matrix;

// Initialize — non-edges must be INF, not 0 (0 would look like a real zero-weight edge)
int n = ...;
int INF = Integer.MAX_VALUE / 2;  // avoid overflow when summing distances
matrix = new int[n][n];
for (int[] row : matrix) Arrays.fill(row, INF);
for (int i = 0; i < n; i++) matrix[i][i] = 0;   // distance to self = 0 (needed for Floyd-Warshall)

// Populate — add edge (u, v) with weight w
matrix[u][v] = w;
matrix[v][u] = w;   // symmetric
```
### Directed + Unweighted (Matrix)
```java
// Declare + Initialize — identical to undirected unweighted
int[][] matrix = new int[n][n];

// Populate — add edge u → v only
matrix[u][v] = 1;
// do NOT set matrix[v][u] unless a reverse edge is explicitly given
```
### Directed + Weighted (Matrix)
```java
// Declare + Initialize — identical to undirected weighted (INF + diagonal 0)
int[][] matrix = new int[n][n];
int INF = Integer.MAX_VALUE / 2;
for (int[] row : matrix) Arrays.fill(row, INF);
for (int i = 0; i < n; i++) matrix[i][i] = 0;

// Populate — one direction only
matrix[u][v] = w;
```

> [!warning] Gotcha Forgetting to seed non-edges with `INF` (instead of leaving default `0`) is the #1 matrix bug — it silently turns "no path" into "path of cost 0" in Floyd-Warshall/Dijkstra-on-matrix.

---

## 2️⃣ Edge List (M×N array)

Good for sparse graphs, and the default shape LeetCode gives you (`edges` parameter). `m` = number of edges.

### Unweighted Edge List (M×2)

```java
// fixed size, known upfront
int[][] edges = new int[m][2]; // Declare & Initialize (fixed-size version)
edges[i] = new int[]{u, v};  // Populate

// or, if building dynamically:
List<int[]> edges = new ArrayList<>();
edges.add(new int[]{u, v});// Populate
```

### Weighted Edge List (M×3)

```java
// {u, v, w} triples
int[][] edges = new int[m][3]; // Declare & Initialize
edges[i] = new int[]{u, v, w}; // Populate

List<int[]> edges = new ArrayList<>();// dynamic version
edges.add(new int[]{u, v, w});       // Populate
```

> [!tip] Direction in an edge list The array shape doesn't change — `{u, v}` looks identical either way. For a **directed** graph, `{u, v}` strictly means u→v. For an **undirected** graph, `{u, v}` is just an unordered pair — store it once; if you later convert to an adjacency list you add both directions there (see [[#🔄 Conversion Cheatsheet]]).

---

## 3️⃣ Adjacency List

### Array of Lists — Unweighted

```java
// Declare
List<Integer>[] adjList;

// Initialize
int n = ...;
adjList = new List[n];              // raw type — see [[#⚠️ Java Declaration Gotchas]]
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();

// Populate — directed edge u → v
adjList[u].add(v);

// Populate — undirected edge (u, v)
adjList[u].add(v);
adjList[v].add(u);
```

### Array of Lists — Weighted

```java
// Declare
List<int[]>[] adjList;              // each int[] = {neighbour, weight}

// Initialize
int n = ...;
adjList = new List[n];
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();

// Populate — directed edge u → v, weight w
adjList[u].add(new int[]{v, w});

// Populate — undirected edge (u, v), weight w
adjList[u].add(new int[]{v, w});
adjList[v].add(new int[]{u, w});
```

Cleaner alternative with a named type instead of `int[]`:

```java
record Edge(int to, int weight) {}
List<Edge>[] adjList = new List[n];
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();
adjList[u].add(new Edge(v, w));
```

### List of Lists — Unweighted

```java
// Declare
List<List<Integer>> adjList;

// Initialize
int n = ...;
adjList = new ArrayList<>();
for (int i = 0; i < n; i++) adjList.add(new ArrayList<>());

// Populate — directed edge u → v
adjList.get(u).add(v);

// Populate — undirected edge (u, v)
adjList.get(u).add(v);
adjList.get(v).add(u);
```

### List of Lists — Weighted

```java
// Declare
List<List<int[]>> adjList;          // each int[] = {neighbour, weight}

// Initialize
int n = ...;
adjList = new ArrayList<>();
for (int i = 0; i < n; i++) adjList.add(new ArrayList<>());

// Populate — directed edge u → v, weight w
adjList.get(u).add(new int[]{v, w});

// Populate — undirected edge (u, v), weight w
adjList.get(u).add(new int[]{v, w});
adjList.get(v).add(new int[]{u, w});
```

### Map of Lists — Unweighted

Use when vertex IDs are **not** contiguous `0..n-1` ints (strings, sparse IDs, coordinates encoded as a key, etc.).

```java
// Declare
Map<Integer, List<Integer>> adjMap;

// Initialize
adjMap = new HashMap<>();

// Populate — directed edge u → v
adjMap.computeIfAbsent(u, k -> new ArrayList<>()).add(v);

// Populate — undirected edge (u, v)
adjMap.computeIfAbsent(u, k -> new ArrayList<>()).add(v);
adjMap.computeIfAbsent(v, k -> new ArrayList<>()).add(u);
```

### Map of Lists — Weighted

```java
// Declare
Map<Integer, List<int[]>> adjMap;    // each int[] = {neighbour, weight}

// Initialize
adjMap = new HashMap<>();

// Populate — directed edge u → v, weight w
adjMap.computeIfAbsent(u, k -> new ArrayList<>()).add(new int[]{v, w});

// Populate — undirected edge (u, v), weight w
adjMap.computeIfAbsent(u, k -> new ArrayList<>()).add(new int[]{v, w});
adjMap.computeIfAbsent(v, k -> new ArrayList<>()).add(new int[]{u, w});
```

---

## 4️⃣ DSU / Union-Find

Not a graph representation per se — a companion structure for **connectivity** questions. Direction is meaningless to DSU (it only ever models undirected "same component" relationships), and it carries no weight/path information — only "are u and v connected?".

```java
// Declare
int[] parent, rank;

// Initialize
int n = ...;
parent = new int[n];
rank = new int[n];                  // defaults to 0 — no explicit fill needed
for (int i = 0; i < n; i++) parent[i] = i;   // each node starts as its own root

// find — with path compression
int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}

// union — by rank; returns false if u and v were ALREADY connected (i.e. this edge closes a cycle)
boolean union(int a, int b) {
    int ra = find(a), rb = find(b);
    if (ra == rb) return false;
    if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
    parent[rb] = ra;
    if (rank[ra] == rank[rb]) rank[ra]++;
    return true;
}
```

> [!tip] DSU's one job `union()` returning `false` **is** cycle detection for undirected graphs — that single boolean is why DSU pairs so naturally with [[#Kruskal's MST]] and [[#Cycle Detection]].

---

## ⚠️ Java Declaration Gotchas

> [!warning] `List<T>[]` can't be created directly Java forbids generic array creation: `new List<Integer>[n]` won't compile. You must either:
> 
> 1. Use the **raw type**: `List<Integer>[] adjList = new List[n];` (unchecked warning, suppress with `@SuppressWarnings("unchecked")` if it bothers you), or
> 2. Sidestep the whole issue with `List<List<Integer>>` instead of `List<Integer>[]` — no array-of-generics involved at all.
> 
> Both are used throughout this note; array-of-lists is marginally faster (no `ArrayList` wrapper indirection for the outer container), list-of-lists is the "safe default" with no raw-type warnings.

> [!warning] Integer overflow in distance sums `Integer.MAX_VALUE` used as "infinity" will silently overflow when you add two of them (or add a weight to it) — e.g. in Floyd-Warshall's `dist[i][k] + dist[k][j]`. Always use a **safe infinity** like `Integer.MAX_VALUE / 2` or a concrete large constant such as `1_000_000_000`.

> [!warning] Autoboxing cost `List<Integer>` / `List<int[]>` box every value. For very large `V`/`E` on performance-sensitive problems, a CSR-style flat `int[]` encoding (head/next/to/weight arrays) avoids boxing entirely — worth knowing exists, rarely needed for LeetCode-scale inputs.

> [!warning] `Arrays.fill` on a 2D array `Arrays.fill(matrix, INF)` does **not** fill a 2D array — it just puts the same `int[]` reference (nonsensical here) into each slot / throws a type error for `int[][]`. You must loop over rows: `for (int[] row : matrix) Arrays.fill(row, INF);`

---

## 🔄 Conversion Cheatsheet

The actual fix for "LeetCode gives me format A, the algorithm I want needs format B."

```java
// Edge list (unweighted, {u, v}) → Array of Lists (directed)
int n = ...;                        // node count, usually given separately
List<Integer>[] adjList = new List[n];
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();
for (int[] e : edges) {
    adjList[e[0]].add(e[1]);
    // undirected? also: adjList[e[1]].add(e[0]);
}
```

```java
// Edge list (weighted, {u, v, w}) → Array of Lists, weighted
List<int[]>[] adjList = new List[n];
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();
for (int[] e : edges) {
    adjList[e[0]].add(new int[]{e[1], e[2]});
    // undirected? also: adjList[e[1]].add(new int[]{e[0], e[2]});
}
```

```java
// Edge list (weighted) → Matrix, directed
int INF = Integer.MAX_VALUE / 2;
int[][] matrix = new int[n][n];
for (int[] row : matrix) Arrays.fill(row, INF);
for (int i = 0; i < n; i++) matrix[i][i] = 0;
for (int[] e : edges) {
    matrix[e[0]][e[1]] = e[2];
    // undirected? also: matrix[e[1]][e[0]] = e[2];
}
```

```java
// Matrix → Array of Lists (e.g. dense input but you want sparse traversal)
List<Integer>[] adjList = new List[n];
for (int i = 0; i < n; i++) adjList[i] = new ArrayList<>();
for (int u = 0; u < n; u++)
    for (int v = 0; v < n; v++)
        if (matrix[u][v] != 0) adjList[u].add(v);
```

```java
// Array of Lists → reversed/transpose Array of Lists (needed for Kosaraju's SCC)
List<Integer>[] rev = new List[n];
for (int i = 0; i < n; i++) rev[i] = new ArrayList<>();
for (int u = 0; u < n; u++)
    for (int v : adjList[u])
        rev[v].add(u);
```

---

## 📚 Algorithm Deep Dives

### BFS

- **Answers:** shortest path in an unweighted graph, level-order distance, connectivity, bipartiteness.
- **Best format:** [[#Array of Lists — Unweighted]] (or [[#List of Lists — Unweighted]]) — a queue-driven traversal only ever needs "give me the neighbours of this vertex," which is exactly O(deg) on an adjacency list.
- **Needs:** `Queue<Integer>`, `boolean[] visited`.

### DFS

- **Answers:** connectivity, cycle detection, topological order (via finish time), building blocks for SCC.
- **Best format:** any adjacency list variant. Recursive or explicit-stack.
- **Directed cycle detection** needs **3 states** per node — white (unvisited) / gray (in current recursion stack) / black (fully done) — a gray node revisited means a back-edge, i.e. a cycle.

### Dijkstra (SSSP — non-negative weights)

- **Best format:** [[#Array of Lists — Weighted]] + `PriorityQueue<int[]>` (min-heap on distance).
- **Why not matrix:** matrix works fine too (O(V²) without a heap — actually simpler code for dense graphs), but for sparse graphs adjacency list + heap gives O((V+E) log V), much better when E ≪ V².
- **Fails on negative weights** — use Bellman-Ford instead.

### Bellman-Ford (SSSP — negative weights allowed)

- **Best format:** [[#Weighted Edge List (M×3)]] — the algorithm's core loop is "relax every edge," so a flat list of edges is the most natural fit; no adjacency structure needed at all.
- **Also detects negative cycles**: run one extra relaxation pass — if anything still improves, a negative cycle exists.
- **Complexity:** O(V·E) — much slower than Dijkstra, only reach for it when negative edges are possible.

### Floyd-Warshall (APSP)

- **Best format:** [[#Undirected + Weighted (Matrix)]] / [[#Directed + Weighted (Matrix)]] — the algorithm _is_ a matrix DP: `dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])` for every `k`. Adjacency list gives no benefit here.
- **Handles negative weights** (not negative cycles — those are detectable via `dist[i][i] < 0` after running).
- **Complexity:** O(V³) — only reasonable for small/medium V.

### Kruskal's MST

- **Best format:** [[#Weighted Edge List (M×3)]] sorted by weight, ascending, + [[#4️⃣ DSU / Union-Find]] for O(α(V)) cycle checks.
- **Only for undirected weighted graphs.**
- **Complexity:** O(E log E) — dominated by the sort.

### Prim's MST

- **Best format:** [[#Array of Lists — Weighted]] + `PriorityQueue` for sparse graphs; [[#Undirected + Weighted (Matrix)]] + linear scan (no heap) is simpler code for dense graphs, same O(V²) either way.
- **Only for undirected weighted graphs.**
- Grows a single tree from an arbitrary root, always picking the cheapest edge leaving the current tree — this is why it wants "neighbours of current tree frontier," an adjacency-list-shaped question.

### Topological Sort

- **Kahn's algorithm (BFS-based):** [[#Array of Lists — Unweighted]] + `int[] indegree` + queue of zero-indegree nodes. If you can't process all `n` nodes, the graph has a cycle (not a DAG).
- **DFS-based:** [[#Array of Lists — Unweighted]] + a stack of finish order, reversed at the end.
- **Only defined for directed acyclic graphs (DAGs).**

### Cycle Detection

- **Undirected graph:** [[#Weighted Edge List (M×3)]] / [[#Unweighted Edge List (M×2)]] + DSU — `union()` returning `false` on an edge means its endpoints were already connected, i.e. a cycle. (Alternative: DFS tracking parent, if you reach a visited node that isn't your immediate parent.)
- **Directed graph:** [[#Array of Lists — Unweighted]] + DFS 3-color marking (see [[#DFS]]), or Kahn's topological sort failing to process all nodes.

### Union-Find Applications

- Number of connected components, redundant connection / redundant edge, accounts-merge style "group these IDs" problems.
- **Best format:** [[#Weighted Edge List (M×3)]] / [[#Unweighted Edge List (M×2)]] as input, paired with [[#4️⃣ DSU / Union-Find]] — no adjacency structure is ever built; you just stream edges through `union()`.

### SCC — Tarjan's / Kosaraju's

- **Tarjan's:** single DFS pass over [[#Array of Lists — Unweighted]] (directed), tracking `disc[]`/`low[]` index arrays and an explicit `onStack[]` boolean array.
- **Kosaraju's:** two DFS passes — first pass on the normal [[#Array of Lists — Unweighted]] to get finish order, second pass on the **reversed/transpose** adjacency list in that finish order (see [[#🔄 Conversion Cheatsheet]] for building the transpose).
- Both only make sense on **directed** graphs — SCC is a meaningless concept for undirected graphs (every connected component is trivially "strongly connected").

---

> [!note] Related This note intentionally leaves out max-flow (Edmonds-Karp/Dinic's) representation details and A* — flag if you want those added as their own sections with the same declare/init/populate treatment.