**The three templates and their names:**

These are the three broad canonical templates that cover the vast majority of recursive/backtracking problems.

```
1. Linear Recursion (i+1)        — "include/exclude" or "pick/skip"
2. Combinatorial Recursion (for) — "backtracking"  
3. Bitmask Iteration             — "subset enumeration"
```


### 1: T1 - Linear Recursion (include/exclude)

```java
// ─────────────────────────────────────────────
// TEMPLATE 1: Linear Recursion (include/exclude)
// Each element: binary choice — take it or skip it
// Call tree: 2^n leaves always
// Use when: yes/no decision per element, order matters
// ─────────────────────────────────────────────
void dfs(int[] nums, int i, /* state */) {
    if (/* base case */) { /* record/return */ return; }
    if (i == nums.length) return;

    // skip nums[i]
    dfs(nums, i + 1, /* state unchanged */);

    // take nums[i]
    dfs(nums, i + 1, /* state updated with nums[i] */);
}
```

### 2: T2 - Combinatorial Recursion (for loop / backtracking)

```java []
// ─────────────────────────────────────────────
// TEMPLATE 2: Combinatorial Recursion (for loop / backtracking)
// At each level, choose ONE element from remaining
// Call tree: only "take" branches, skip is implicit
// Use when: building subsets/combos/perms, want to avoid duplicates
// ─────────────────────────────────────────────
void dfs(int[] nums, int start, /* state */) {
    if (/* base case */) { /* record/return */ return; }

    for (int i = start; i < nums.length; i++) {
        /* update state with nums[i] */
        dfs(nums, i + 1, /* updated state */);  // i+1 for combos, i for reuse
        /* undo state (backtrack) */
    }
}
```

### 3: T3 - Bitmask Iteration
```java
// ─────────────────────────────────────────────
// TEMPLATE 3: Bitmask Iteration
// Enumerate all 2^n subsets using integers
// No recursion — pure iteration
// Use when: n is small (≤20), need all subsets, want cache/DP over subsets
// ─────────────────────────────────────────────
int full = (1 << n) - 1;
for (int mask = 0; mask <= full; mask++) {
    // mask = 0      → empty subset
    // mask = full   → all elements
    // mask = 1..full-1 → non-empty proper subsets

    for (int i = 0; i < n; i++) {
        if ((mask & (1 << i)) != 0) {
            /* nums[i] is in this subset */
        }
    }
    /* evaluate subset for this mask */
}
```

**When to reach for which:**

| Situation                                                | Template                                       |
| -------------------------------------------------------- | ---------------------------------------------- |
| Simple `yes/no` per element, no pruning needed           | Linear (T1)                                    |
| Building combinations, need to prune branches early      | For loop (T2)                                  |
| `n ≤ 20`, need all subsets, want to memoize over subsets | Bitmask (T3)                                   |
| Permutations (order matters)                             | T2 with a `visited[]` array instead of `start` |
| Subset sum, partition problems                           | T1 or T2 equally valid                         |
| DP on subsets (TSP, min cost cover)                      | T3 (bitmask DP)                                |

# Application - Examples 

### [3566. Partition Array into Two Equal Product Subsets](https://leetcode.com/problems/partition-array-into-two-equal-product-subsets/)

```java
class Solution {
    long totalP  = 1L;
    long target;
    public boolean checkEqualPartitions(int[] nums, long target) {
        this.target = target;
        for(int num : nums) {
            totalP *= (long) num;
            if( ((target & 1) == 1) && ((num & 1) == 0)) { // odd Target & even number
                return false;
            }
        }

        if(totalP != (target*target)) return false;

        return dfs(nums, 1L, 0);
    }

    boolean dfs(int[] nums, long prod, int i)  {
        if(i == nums.length)   return false;

        if( prod == target )    return true;

        // don't take it ||  take it
        return dfs(nums, prod, i+1) || 
                dfs(nums, prod*nums[i], i+1);
    }
}
```



```java
class Solution {
    long totalProd  = 1L;
    long target;
    public boolean checkEqualPartitions(int[] nums, long target) {
		this.target = target;
        for(int num : nums) {
            totalProd *= (long) num;
            if( ((target & 1) == 1) && ((num & 1) == 0)) 
                return false;
        }

        if(totalProd != (target*target)) return false;

        return dfs(nums, 1L, 0);
    }

    boolean dfs(int[] nums, long prod, int start)  {

        if (prod == target ) return true;

        for (int i = start; i < nums.length; i++) {
            long num = (long) nums[i];
            if (dfs(nums, prod * num, i + 1))
                return true;
        }
        return false;
    }
}
```


```java
class Solution {
    public boolean checkEqualPartitions(int[] nums, long target) {
        int n = nums.length;
        long totalP = 1L;
        for (int num : nums) {
            totalP *= num;
            if ((target & 1) == 1 && (num & 1) == 0) return false;
        }
        if (totalP != target * target) return false;

        int full = (1 << n) - 1; // all bits set = both subsets combined

        for (int mask = 1; mask < full; mask++) { // skip 0 (empty A) and full (empty B)
            long productA = 1L;
            for (int i = 0; i < n; i++) {
                if ((mask & (1 << i)) != 0) {
                    productA *= nums[i];
                }
            }
            if (productA == target) return true;
        }
        return false;
    }
}
```


