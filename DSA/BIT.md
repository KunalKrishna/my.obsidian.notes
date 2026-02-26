
A **Binary Indexed Tree (BIT)**, also known as a **Fenwick Tree**, is a data structure that allows efficient updates and prefix sum queries on an array of numbers. It is particularly useful for problems like the [Range Sum Query - Mutable](https://leetcode.com/problems/range-sum-query-mutable/) you are currently working on.

### 1. What is a BIT?

A BIT provides a way to represent an array that balances the time complexity between updates and queries.

- **Update:** $O(\log N)$ time.
- **Query (Prefix Sum):** $O(\log N)$ time.
- **Space:** $O(N)$.

Compared to a Segment Tree, a BIT is often easier to code and has a smaller memory footprint, though it is primarily used for cumulative (prefix) operations.

BIT is typically **1-indexed**.
### 2. How It Works: The Bitwise Logic

The "magic" of a BIT lies in how each index in the tree array handles a specific range of the original array. This is determined by the **Least Significant Bit (LSB)** of the index.

- **Tree Structure:** Each element at index `i` in the `tree[]` array stores the sum of a range of elements from the original array. The length of this range is equal to the value of the LSB of `i`.
- **Finding the LSB:** In Java, you extract the LSB using the bitwise operation `i & -i`.
- **Navigation:**
    - **Moving Up (Update):** To update an index and all ranges that include it, you repeatedly **add** the LSB: `i += i & -i`.
    - **Moving Down (Query):** To calculate a prefix sum, you collect values by repeatedly **subtracting** the LSB: `i -= i & -i`.

### 3. How to Build It

To build a BIT from an existing array, you initialize a tree array of size $N+1$ (since BIT is typically **1-indexed**) and perform an update for every element in your original array.

As seen in [307. Range Sum Query - Mutable](https://leetcode.com/problems/range-sum-query-mutable/):
1. **Initialize:** Create `tree = new int[n + 1]`.
2. **Populate:** Loop through your original `nums` array and call your `init` (or `update`) function for each index.

Java

```
public NumArray(int[] nums) {
    this.n = nums.length;
    this.tree = new int[n + 1];
    for (int i = 0; i < n; i++) {
        // Build the tree by updating each element
        updateBIT(i, nums[i]);
    }
}
```