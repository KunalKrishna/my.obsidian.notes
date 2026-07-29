# Binary Search: The Complete Master Guide

## Demystifying Formulas, Inequality Signs, and the Art of Precise Boundaries

---

## Table of Contents

1. [The Core Problem](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#the-core-problem)
2. [The Pseudocode Framework](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#the-pseudocode-framework)
3. [Part 1: Mid Formula Choices](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#part-1-mid-formula-choices)
4. [Part 2: Inequality Signs & Their Constraints](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#part-2-inequality-signs--their-constraints)
5. [Part 3: Shifting Strategies (l and r adjustment)](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#part-3-shifting-strategies)
6. [Case Study 1: Unique Elements Array](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#case-study-1-unique-elements-array)
7. [Case Study 2: Repeating Elements Array](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#case-study-2-repeating-elements-array)
8. [The Seven Sacred Binary Search Patterns](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#the-seven-sacred-binary-search-patterns)
9. [Decision Tree: Choosing Your Combination](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#decision-tree-choosing-your-combination)
10. [Pitfalls & How to Avoid Them](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#pitfalls--how-to-avoid-them)
11. [Memory Aids & Mnemonics](https://claude.ai/chat/9dd43644-99ac-4465-89b7-8857206e9df3#memory-aids--mnemonics)

---

## The Core Problem

```java
public int find(int[] A, int target) {
    int l = 0, r = (A.length - 1);
    
    while(l [INEQUALITY] r) {                    // ← Choice #1
        int mid = [FORMULA];                     // ← Choice #2
        
        if(A[mid] [INEQUALITY] target)           // ← Choice #3
            return mid;
        
        if(A[mid] [INEQUALITY] target)           // ← Choice #4
            l = mid [+/- ?];                     // ← Choice #5
        else
            r = mid [+/- ?];                     // ← Choice #6
    }
    
    return -1;
}
```

**You have 8+ critical choices to make**, and a wrong choice → infinite loop or wrong answer.

---

## The Pseudocode Framework

This is your blueprint for ALL binary search problems:

```java
public int binarySearch(int[] A, int target) {
    int l = 0, r = A.length - 1;
    
    while (l COND r) {                    // While loop condition
        int mid = l + (r - l) / 2;        // Safe mid calculation
        
        if (A[mid] == target)
            return mid;                    // Found it!
        
        if (A[mid] < target)              // mid is too small
            l = mid + 1;                  // Search right
        else                              // mid is too large
            r = mid - 1;                  // Search left
    }
    
    return -1;                            // Not found
}
```

**Key insight:** This template works because:

- We always eliminate at least one element per iteration
- `l = mid + 1` and `r = mid - 1` guarantee progress
- We never revisit the same state

---

## Part 1: Mid Formula Choices

### Formula 1: `mid = (l + r) / 2` — The Naive Way

```java
mid = (l + r) / 2;  // Integer division truncates (LEFT-BIASED)
```

**Overflow Risk:**

```
If l = 2,000,000,000 and r = 2,147,483,647 (near INT_MAX)
l + r = 4,147,483,647 → OVERFLOW! Becomes negative.
```

**Bias Behavior:**

```
Range [0,1]: mid = 0     (LEFT)
Range [0,3]: mid = 1     (CENTER)
Range [0,5]: mid = 2     (LEFT)
```

**When to use:** Rarely. Only when you're 100% sure l + r won't overflow.

**Verdict:** ❌ **AVOID** — Can cause overflow on large indices

---

### Formula 2: `mid = l + (r - l) / 2` — The Safe Standard

```java
mid = l + (r - l) / 2;  // Safe, left-biased, standard
```

**Why it's safe:**

```
(r - l) is always smaller than (l + r)
Maximum value: (r - l) ≤ length of array
This NEVER overflows for valid array indices
```

**Bias Behavior:**

```
Same left-bias as Formula 1, but safe:
Range [0,1]: mid = 0 + (1-0)/2 = 0     (LEFT)
Range [0,3]: mid = 0 + (3-0)/2 = 1     (CENTER)
Range [0,5]: mid = 0 + (5-0)/2 = 2     (LEFT)
```

**When to use:** **DEFAULT CHOICE** — Always use this unless you have a reason not to.

**Verdict:** ✅ **BEST PRACTICE** — Safe, readable, standard

---

### Formula 3: `mid = l + (r - l + 1) / 2` — Right-Biased

```java
mid = l + (r - l + 1) / 2;  // Right-biased
```

**Bias Behavior:**

```
Range [0,1]: mid = 0 + (1-0+1)/2 = 1   (RIGHT)
Range [0,3]: mid = 0 + (3-0+1)/2 = 2   (RIGHT of center)
Range [0,5]: mid = 0 + (5-0+1)/2 = 3   (RIGHT)
```

**Why it exists:**

- Avoids infinite loops in specific scenarios
- When you use `l = mid` (instead of `l = mid + 1`)

**When to use:**

```java
// Rare pattern where you DON'T advance mid
while (l < r) {
    mid = l + (r - l + 1) / 2;  // RIGHT-biased
    if (arr[mid] <= target) {
        l = mid;                 // mid could equal l, so need right-bias
    } else {
        r = mid - 1;
    }
}
```

**Verdict:** ⚠️ **SPECIALIZED** — Use only when your assignment requires it

---

### Formula 4: `mid = l + (r - l) >> 1` — Bit Shift Optimization

```java
mid = l + (r - l) >> 1;  // Bitwise right shift (same as / 2)
```

**Why:** Bit shift is marginally faster than division.

**Practical difference:** Essentially zero on modern CPUs (JIT compiler optimizes both to the same instruction).

**When to use:** Very rarely; compiler optimization makes this pointless.

**Verdict:** ℹ️ **INFORMATIONAL** — Don't use; let the compiler optimize

---

### Formula Comparison Table

|Formula|Overflow Safe|Bias|Readability|Use Case|
|---|---|---|---|---|
|`(l+r)/2`|❌ NO|LEFT|Medium|Never|
|`l+(r-l)/2`|✅ YES|LEFT|✅ Excellent|**DEFAULT**|
|`l+(r-l+1)/2`|✅ YES|RIGHT|Good|Specialized|
|`l+((r-l)>>1)`|✅ YES|LEFT|Medium|Never|

**KEY OBSERVATION #1:**

> Use `l + (r - l) / 2` as your default. It's safe, readable, and handles 99% of cases.

---

## Part 2: Inequality Signs & Their Constraints

This is where most binary search bugs originate.

### The Fundamental Constraint

When you write:

```
while (l [INEQUALITY] r) {
    mid = l + (r - l) / 2;
    ...
    if (...) l = mid [+/- ?];
    else r = mid [+/- ?];
}
```

The **inequality signs you choose MUST match your shifting strategy**.

### The Four Loop Conditions

#### Condition 1: `while (l < r)` — The Most Common

```java
while (l < r) {
    mid = l + (r - l) / 2;  // Left-biased
    if (arr[mid] < target) {
        l = mid + 1;        // MUST advance (not just l = mid)
    } else {
        r = mid;            // Can set r = mid (mid might equal l, loop exits)
    }
}
// After loop: l == r, points to the answer
```

**Why this works:**

- When `l < r`, mid is always `<= r` (left-biased)
- If we set `r = mid`, we don't lose the potential answer
- When `l = mid` and `r = mid`, loop exits
- No infinite loops possible with `l = mid + 1` and `r = mid`

**Critical constraint:**

```
If you use: while (l < r) and l = mid (instead of l = mid + 1)
Then you MUST use: arr[mid] < target (NOT arr[mid] <= target)
Otherwise: INFINITE LOOP
```

**When to use:** Finding lower/upper bounds with exact boundary positioning.

---

#### Condition 2: `while (l <= r)` — The Safe Standard for Exact Match

```java
public int findExact(int[] arr, int target) {
    int l = 0, r = arr.length - 1;
    
    while (l <= r) {                    // ← Key: <=
        int mid = l + (r - l) / 2;
        
        if (arr[mid] == target)
            return mid;                 // Found!
        
        if (arr[mid] < target) {
            l = mid + 1;                // Safe: always advances
        } else {
            r = mid - 1;                // Safe: always decreases
        }
    }
    
    return -1;                          // Not found
}
```

**Why this works:**

- `l <= r` means "while there's at least one element to check"
- We always remove `mid` from consideration with `l = mid + 1` or `r = mid - 1`
- Eventually `l > r`, loop exits
- No infinite loop possible

**When to use:** When you're searching for an exact element and don't care about boundaries.

---

#### Condition 3: `while (l + 1 < r)` — For Finding Neighbors

```java
while (l + 1 < r) {  // Keep going until l and r are adjacent
    mid = l + (r - l) / 2;
    if (arr[mid] < target) {
        l = mid;
    } else {
        r = mid;
    }
}
// After loop: l and r are adjacent, check both manually
if (arr[l] == target) return l;
if (arr[r] == target) return r;
return -1;
```

**Why this works:**

- Loop stops when `l` and `r` are exactly 1 apart
- You manually check the final two candidates
- Useful for finding closest element, or left/right boundary

**When to use:** Finding the closest element, or when you need to check exactly two candidates.

---

### The Inequality in the Comparison

The most critical choice: **`arr[mid]` compared to `target`**

```java
if (arr[mid] [INEQUALITY] target) {
    l = mid + 1;
} else {
    r = mid [+/- ?];
}
```

#### Pattern A: Using `<` in the Comparison

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] < target) {        // ← Using <
        l = mid + 1;                 // Advance
    } else {
        r = mid;                     // r could become l
    }
}
// After loop: l points to first element >= target
```

**Intuition:**

- "If mid is less than target, move past it"
- "Otherwise, mid could be the answer"

**Type of answer:** First element `>= target` (lower bound)

---

#### Pattern B: Using `<=` in the Comparison

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] <= target) {       // ← Using <=
        l = mid + 1;                 // Always advance
    } else {
        r = mid;
    }
}
// After loop: l points to first element > target
```

**Intuition:**

- "If mid is less than or equal to target, move past it"
- "Otherwise, mid is strictly greater"

**Type of answer:** First element `> target`

---

#### Pattern C: Using `>` in the Comparison

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] > target) {        // ← Using >
        r = mid;                     // r could become l
    } else {
        l = mid + 1;                 // Always advance
    }
}
// After loop: l points to first element > target
```

**Type of answer:** First element `> target`

---

### KEY OBSERVATION #2: The Symmetry Rule

Binary search patterns come in **mirrored pairs**:

```
Finding LOWER BOUND (first >= target):
    if (arr[mid] < target) l = mid + 1; else r = mid;

Finding UPPER BOUND (last <= target):
    if (arr[mid] <= target) l = mid + 1; else r = mid;
```

**The relationship:** Lower bound + upper bound = complete picture of duplicates.

---

## Part 3: Shifting Strategies

After deciding on the comparison, you shift `l` or `r`.

### Strategy 1: `l = mid + 1` (Most Common)

```java
if (arr[mid] < target) {
    l = mid + 1;  // Skip mid entirely; it's too small
}
```

**Why this works:**

- Eliminates `mid` and everything to its left
- Always makes progress (l increases)
- Impossible to get stuck

**When to use:** Almost always for the "search right" branch.

---

### Strategy 2: `l = mid` (Risky)

```java
if (arr[mid] <= target) {
    l = mid;  // Don't skip mid; it might be the answer
}
```

**Danger zone:**

- If `mid == l`, the loop might not make progress
- ONLY safe if mid can never equal l
- With left-biased mid, this can cause infinite loops

**When it's safe:**

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] <= target) {
        l = mid;  // ← This is safe IF l can never stay l
    } else {
        r = mid - 1;  // ← Because r decreases, so eventually l < r becomes false
    }
}
```

Wait, let me reconsider. If `l = mid` and `mid = l`, then `l` doesn't change. But we also have `r = mid - 1`. So:

- Iteration 1: l=0, r=1, mid=0, arr[0]<=5? YES, l=0, r=0 → loop exits ✓

Actually, this is safe because `r` decreases.

**When to use:** Rarely; only in specific patterns where r always decreases on the other branch.

---

### Strategy 3: `r = mid - 1` (Standard)

```java
else {
    r = mid - 1;  // Skip mid entirely; it's too large
}
```

**Why this works:**

- Eliminates `mid` and everything to its right
- Always makes progress (r decreases)
- Complements `l = mid + 1` perfectly

**When to use:** When you're confident the answer is strictly to the left.

---

### Strategy 4: `r = mid` (Most Careful)

```java
else {
    r = mid;  // Don't skip mid; it might be the answer
}
```

**Why it works:**

- With left-biased mid and `while (l < r)`, mid could equal l
- Setting `r = mid` lets mid be re-examined (as potential l next iteration)
- When `l == r`, loop exits

**When to use:** When building boundaries and mid might be the answer.

---

### Shifting Strategy Summary

|Shift|Progress Guarantee|Risk|Use Case|
|---|---|---|---|
|`l = mid + 1`|✅ Always|None|Default for "go right"|
|`l = mid`|⚠️ Conditional|Infinite loop|Rare, specialized|
|`r = mid - 1`|✅ Always|None|Default for "go left"|
|`r = mid`|✅ Always (with `l < r`)|None|Default for "go left, keep mid"|

**KEY OBSERVATION #3:**

> The safest pattern is `l = mid + 1` and `r = mid - 1`. Use this when searching for exact elements. For boundary finding, use `l = mid + 1` and `r = mid`.

---

## Case Study 1: Unique Elements Array

**Problem:** Find target in array with all unique elements.

**Example:** `[1, 3, 5, 7, 9, 11]`, find 7

### Solution Pattern

```java
public int findInUnique(int[] arr, int target) {
    int l = 0, r = arr.length - 1;
    
    while (l <= r) {                           // Condition: <=
        int mid = l + (r - l) / 2;             // Formula: safe mid
        
        if (arr[mid] == target) {
            return mid;                        // Found!
        } else if (arr[mid] < target) {
            l = mid + 1;                       // Go right
        } else {
            r = mid - 1;                       // Go left
        }
    }
    
    return -1;                                 // Not found
}
```

**Why this works:**

- `while (l <= r)`: "While there's at least one element"
- `l = mid + 1` and `r = mid - 1`: Never revisit mid
- `==` check: We stop immediately when found
- Guaranteed termination: Elements always eliminated

**Execution trace for target=7:**

```
Array: [1, 3, 5, 7, 9, 11]
         0  1  2  3  4   5

Iteration 1: l=0, r=5, mid=2, arr[2]=5 < 7 → l=3
Iteration 2: l=3, r=5, mid=4, arr[4]=9 > 7 → r=3
Iteration 3: l=3, r=3, mid=3, arr[3]=7 == 7 → RETURN 3 ✓

Time: O(log n)
```

**Observations for unique elements:**

1. ✓ The exact match case is simple
2. ✓ `==` comparison makes it foolproof
3. ✓ `l <= r` is the clearest condition
4. ✓ `l = mid + 1` and `r = mid - 1` are the safest shifts

---

## Case Study 2: Repeating Elements Array

**Problem:** Find boundaries (leftmost or rightmost occurrence) in array with duplicates.

**Example:** `[1, 3, 3, 3, 3, 5, 7]`, find leftmost 3 and rightmost 3

### Sub-case 2A: Find Leftmost Occurrence

```java
public int findLeftmost(int[] arr, int target) {
    int l = 0, r = arr.length - 1;
    int result = -1;
    
    while (l <= r) {                           // Condition: <=
        int mid = l + (r - l) / 2;             // Formula: safe mid
        
        if (arr[mid] == target) {
            result = mid;                      // Record it
            r = mid - 1;                       // But keep searching LEFT
        } else if (arr[mid] < target) {
            l = mid + 1;                       // Go right
        } else {
            r = mid - 1;                       // Go left
        }
    }
    
    return result;                             // First occurrence
}
```

**Execution trace for target=3 in `[1, 3, 3, 3, 3, 5, 7]`:**

```
Array: [1, 3, 3, 3, 3, 5, 7]
         0  1  2  3  4  5  6

Iteration 1: l=0, r=6, mid=3, arr[3]=3 == 3 → result=3, r=2
Iteration 2: l=0, r=2, mid=1, arr[1]=3 == 3 → result=1, r=0
Iteration 3: l=0, r=0, mid=0, arr[0]=1 < 3 → l=1
Iteration 4: l=1, r=0 → l > r, EXIT

RETURN result=1 ✓
```

**Key insight:** When you find the target, **record it but keep searching** in the direction you want boundaries.

---

### Sub-case 2B: Find Rightmost Occurrence

```java
public int findRightmost(int[] arr, int target) {
    int l = 0, r = arr.length - 1;
    int result = -1;
    
    while (l <= r) {
        int mid = l + (r - l) / 2;
        
        if (arr[mid] == target) {
            result = mid;                      // Record it
            l = mid + 1;                       // But keep searching RIGHT
        } else if (arr[mid] < target) {
            l = mid + 1;                       // Go right
        } else {
            r = mid - 1;                       // Go left
        }
    }
    
    return result;                             // Last occurrence
}
```

**Execution trace for target=3 in `[1, 3, 3, 3, 3, 5, 7]`:**

```
Iteration 1: l=0, r=6, mid=3, arr[3]=3 == 3 → result=3, l=4
Iteration 2: l=4, r=6, mid=5, arr[5]=5 > 3 → r=4
Iteration 3: l=4, r=4, mid=4, arr[4]=3 == 3 → result=4, l=5
Iteration 4: l=5, r=4 → l > r, EXIT

RETURN result=4 ✓
```

---

### Sub-case 2C: Advanced Pattern - Find Lower Bound Without Recording

```java
public int findLowerBound(int[] arr, int target) {
    // Returns: index of first element >= target
    // If not found: returns arr.length (pointing past the end)
    
    int l = 0, r = arr.length;                 // r = length, not length-1!
    
    while (l < r) {                            // Condition: <
        int mid = l + (r - l) / 2;
        
        if (arr[mid] < target) {
            l = mid + 1;                       // mid is too small
        } else {
            r = mid;                           // mid could be the answer
        }
    }
    
    return l;                                  // l == r, both pointing to answer
}
```

**Why this pattern is different:**

- `r = arr.length` (not `arr.length - 1`)
- `while (l < r)` (not `<=`)
- No recording or comparisons; pure boundary finding
- Returns `l` which equals `r` at the end

**Execution trace for target=3 in `[1, 3, 3, 3, 3, 5, 7]`:**

```
l=0, r=7 (array length)

Iteration 1: l=0, r=7, mid=3, arr[3]=3 >= 3? YES → r=3
Iteration 2: l=0, r=3, mid=1, arr[1]=3 >= 3? YES → r=1
Iteration 3: l=0, r=1, mid=0, arr[0]=1 >= 3? NO → l=1
Iteration 4: l=1, r=1 → l == r, EXIT

RETURN l=1 ✓ (first index where arr[i] >= 3)
```

---

### KEY OBSERVATION #4: The Three Paths for Duplicates

|Goal|Pattern|Loop|Comparison|Recording|
|---|---|---|---|---|
|Find leftmost|Search, record, go LEFT|`l<=r`|`==`|Yes, then `r=mid-1`|
|Find rightmost|Search, record, go RIGHT|`l<=r`|`==`|Yes, then `l=mid+1`|
|Find lower bound|Pure boundary (no matches)|`l<r`|`<`|No recording|
|Find upper bound|Pure boundary (no matches)|`l<r`|`<=`|No recording|

---

## The Seven Sacred Binary Search Patterns

Every binary search problem fits into ONE of these patterns:

### Pattern 1: Exact Match in Unique Array

```java
while (l <= r) {
    mid = l + (r - l) / 2;
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) l = mid + 1;
    else r = mid - 1;
}
return -1;
```

**Examples:** Find a number in a sorted unique array.

---

### Pattern 2: First Occurrence (Leftmost)

```java
int result = -1;
while (l <= r) {
    mid = l + (r - l) / 2;
    if (arr[mid] == target) {
        result = mid;
        r = mid - 1;  // Continue searching LEFT
    } else if (arr[mid] < target) {
        l = mid + 1;
    } else {
        r = mid - 1;
    }
}
return result;
```

**Examples:** First occurrence of a duplicate element.

---

### Pattern 3: Last Occurrence (Rightmost)

```java
int result = -1;
while (l <= r) {
    mid = l + (r - l) / 2;
    if (arr[mid] == target) {
        result = mid;
        l = mid + 1;  // Continue searching RIGHT
    } else if (arr[mid] < target) {
        l = mid + 1;
    } else {
        r = mid - 1;
    }
}
return result;
```

**Examples:** Last occurrence, rightmost boundary.

---

### Pattern 4: Lower Bound (First >= Target)

```java
int l = 0, r = arr.length;
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] < target) {
        l = mid + 1;
    } else {
        r = mid;
    }
}
return l;  // l == r, points to first >= target
```

**Examples:** First element not smaller than target.

---

### Pattern 5: Upper Bound (First > Target)

```java
int l = 0, r = arr.length;
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] <= target) {
        l = mid + 1;
    } else {
        r = mid;
    }
}
return l;  // Points to first > target
```

**Examples:** First element strictly greater than target.

---

### Pattern 6: Closest Element

```java
while (l + 1 < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] < target) {
        l = mid;
    } else {
        r = mid;
    }
}
// Check l and r, return the closer one
return Math.abs(arr[l] - target) <= Math.abs(arr[r] - target) ? l : r;
```

**Examples:** Find the element closest in value to target.

---

### Pattern 7: Search in Rotated Array

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] > arr[r]) {
        // Pivot is in the right half
        l = mid + 1;
    } else {
        // Pivot is in the left half
        r = mid;
    }
}
return l;  // Points to the pivot/minimum
```

**Examples:** Find minimum in rotated sorted array.

---

## Decision Tree: Choosing Your Combination

```
START: Binary Search Problem

├─ Question 1: Are elements UNIQUE?
│  │
│  ├─ YES → Are you looking for EXACT MATCH?
│  │  │
│  │  ├─ YES → Pattern 1: Exact Match
│  │  │         Loop: while (l <= r)
│  │  │         Shift: l=mid+1, r=mid-1
│  │  │
│  │  └─ NO → Is it a SPECIAL PROPERTY search (rotated, peak)?
│  │         → Pattern 7 or variant
│  │
│  └─ NO (DUPLICATES) → What are you finding?
│     │
│     ├─ FIRST OCCURRENCE? → Pattern 2: Leftmost
│     │                      Loop: while (l <= r)
│     │                      Record when found, search LEFT
│     │
│     ├─ LAST OCCURRENCE? → Pattern 3: Rightmost
│     │                     Loop: while (l <= r)
│     │                     Record when found, search RIGHT
│     │
│     ├─ FIRST >= TARGET? → Pattern 4: Lower Bound
│     │                     Loop: while (l < r)
│     │                     Shift: l=mid+1 if arr[mid]<target, else r=mid
│     │
│     ├─ FIRST > TARGET? → Pattern 5: Upper Bound
│     │                    Loop: while (l < r)
│     │                    Shift: l=mid+1 if arr[mid]<=target, else r=mid
│     │
│     ├─ CLOSEST ELEMENT? → Pattern 6: Closest
│     │                     Loop: while (l+1 < r)
│     │                     Check l and r manually
│     │
│     └─ CUSTOM PREDICATE? → Use Pattern 4/5 with custom comparator
                              Think: "What's the first thing that satisfies/fails predicate?"

```

---

## Pitfalls & How to Avoid Them

### Pitfall 1: Infinite Loop from Left-Biased Mid

**The trap:**

```java
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] <= target) {
        l = mid;  // ← mid could equal l, no progress!
    } else {
        r = mid - 1;
    }
}
```

**Why it happens:**

- Left-biased mid means when `l < r`, mid is often equal to l
- If `l = mid` and nothing else changes, infinite loop

**How to fix:**

```java
// Option 1: Use l = mid + 1 (advance past mid)
if (arr[mid] <= target) {
    l = mid + 1;  // Always makes progress
} else {
    r = mid - 1;
}

// Option 2: Use right-biased mid with l = mid
mid = l + (r - l + 1) / 2;  // Right-biased
if (arr[mid] <= target) {
    l = mid;  // Now safe, mid is to the right of center
} else {
    r = mid - 1;
}
```

**Prevention:**

- ✓ Always use `l = mid + 1` (safer)
- ✗ Avoid `l = mid` unless you understand the consequences

---

### Pitfall 2: Overflow in Mid Calculation

**The trap:**

```java
mid = (l + r) / 2;  // Overflow if l and r are large
```

**When it happens:** Arrays with millions of elements where indices approach Integer.MAX_VALUE.

**How to fix:**

```java
mid = l + (r - l) / 2;  // Safe, always use this
```

**Prevention:** Make it a habit. Always write `l + (r - l) / 2`.

---

### Pitfall 3: Off-by-One Errors with Array Bounds

**The trap:**

```java
int r = arr.length;  // Should this be arr.length - 1?
```

**When to use each:**

```java
// If you use: while (l <= r)
int r = arr.length - 1;  // r points to last element

// If you use: while (l < r)
int r = arr.length;      // r points past the array (not accessed directly)
```

**Prevention:**

- For `l <= r` loop: r = arr.length - 1
- For `l < r` loop: r = arr.length

---

### Pitfall 4: Wrong Comparison for Duplicates

**The trap:**

```java
// Finding leftmost occurrence
while (l < r) {
    mid = l + (r - l) / 2;
    if (arr[mid] < target) {
        l = mid + 1;
    } else if (arr[mid] > target) {
        r = mid - 1;
    }
    // Missing: what if arr[mid] == target?
}
```

**How to fix:**

```java
// Pattern 2: Find leftmost
int result = -1;
while (l <= r) {  // Use <=
    mid = l + (r - l) / 2;
    if (arr[mid] == target) {
        result = mid;
        r = mid - 1;  // Continue searching LEFT
    } else if (arr[mid] < target) {
        l = mid + 1;
    } else {
        r = mid - 1;
    }
}
return result;
```

**Prevention:**

- Explicitly handle all three cases: <, ==, >
- When found, decide: search left or right?

---

### Pitfall 5: Not Returning the Right Value

**The trap:**

```java
while (l < r) {
    ...
}
return l;  // But sometimes you need r, result, or arr[l]
```

**When to return what:**

```java
// Pattern 1: Exact match
if (arr[mid] == target) return mid;  // Immediately

// Pattern 2/3: Leftmost/rightmost
return result;  // Variable you recorded

// Pattern 4/5: Lower/upper bound
return l;  // Both l and r point to same position

// Pattern 6: Closest
return (closer of l or r);  // Compare distances
```

**Prevention:** Be clear about what you're returning before you write the loop.

---

### Pitfall 6: Confusing Lower Bound and Upper Bound

**Lower bound (first >= target):**

```java
if (arr[mid] < target) l = mid + 1; else r = mid;
// Returns: first element >= target
```

**Upper bound (first > target):**

```java
if (arr[mid] <= target) l = mid + 1; else r = mid;
// Returns: first element > target
```

**The ONE-CHARACTER difference:** `<` vs `<=`

**Prevention:** Memorize:

- Lower bound uses `<`
- Upper bound uses `<=`
- (opposite to intuition!)

---

## Memory Aids & Mnemonics

### Mnemonic 1: "SAFE-S"

**S**afe mid: `l + (r - l) / 2`  
**A**dvance left: `l = mid + 1`  
**F**all back right: `r = mid` (or `r = mid - 1`)  
**E**quality: only when exact match needed  
**S**hift consistently

---

### Mnemonic 2: "The Golden Rule"

```
if (arr[mid] < target)
    l = mid + 1;     // Too small, go RIGHT
else
    r = mid;         // Could be answer, go LEFT but keep mid
```

This ONE pattern solves 70% of binary search problems!

---

### Mnemonic 3: "LowUp"

**Low**er **B**ound: `if (arr[mid] < target) l = mid + 1;`  
(**L**ower uses **`<`**)

**Up**per **B**ound: `if (arr[mid] <= target) l = mid + 1;`  
(**Up**per uses **`<=`**)

---

### Mnemonic 4: "Three Paths for Duplicates"

When you **find** the target in a duplicate array:

|Path|Direction|Continue With|
|---|---|---|
|**Leftmost**|← LEFT|`r = mid - 1`|
|**Rightmost**|→ RIGHT|`l = mid + 1`|
|**Don't record**|N/A|Use Pattern 4/5 (boundaries)|

---

### Mnemonic 5: "Loop Choice Determines Everything"

```
while (l <= r)     →  Searching for EXACT match or first/last
while (l < r)      →  Searching for BOUNDARY (lower/upper bound)
while (l + 1 < r)  →  Finding NEIGHBORS or CLOSEST
```

---

## Quick Reference Chart

### All Combinations at a Glance

```
UNIQUE ELEMENTS:
├─ Loop: while (l <= r)
├─ Mid: l + (r - l) / 2
├─ Compare: arr[mid] == target
└─ Shift: l = mid + 1, r = mid - 1

DUPLICATES (Leftmost):
├─ Loop: while (l <= r)
├─ Mid: l + (r - l) / 2
├─ Find: arr[mid] == target → record, r = mid - 1
└─ Other: else if (arr[mid] < target) l = mid + 1; else r = mid - 1;

DUPLICATES (Rightmost):
├─ Loop: while (l <= r)
├─ Mid: l + (r - l) / 2
├─ Find: arr[mid] == target → record, l = mid + 1
└─ Other: else if (arr[mid] < target) l = mid + 1; else r = mid - 1;

LOWER BOUND (first >= target):
├─ Loop: while (l < r)
├─ r = arr.length (not arr.length - 1!)
├─ Mid: l + (r - l) / 2
├─ If arr[mid] < target: l = mid + 1
└─ Else: r = mid

UPPER BOUND (first > target):
├─ Loop: while (l < r)
├─ r = arr.length
├─ Mid: l + (r - l) / 2
├─ If arr[mid] <= target: l = mid + 1
└─ Else: r = mid

CLOSEST ELEMENT:
├─ Loop: while (l + 1 < r)
├─ Mid: l + (r - l) / 2
├─ If arr[mid] < target: l = mid
├─ Else: r = mid
└─ Return: closer of arr[l] or arr[r]
```

---

## Master Your Binary Search Checklist

Before writing code, answer these questions:

- [ ] Are elements **unique or duplicated**?
- [ ] Am I looking for an **exact match, leftmost, rightmost, or boundary**?
- [ ] Should the loop be `l <= r`, `l < r`, or `l + 1 < r`?
- [ ] What is my **initial r value** (length - 1 or length)?
- [ ] What **comparison operator** does my search need (<, <=, ==, >, >=)?
- [ ] On "go right": should I use `l = mid + 1` or `l = mid`?
- [ ] On "go left": should I use `r = mid - 1` or `r = mid`?
- [ ] What should I **return** (mid, result, l, r, or arr[l])?
- [ ] Can this loop ever **infinite loop**? (Test with l=0, r=1)

---

## Conclusion: The Path Forward

Binary search is not about memorizing combinations. It's about **understanding the constraints**:

1. **Mid formula:** Always use `l + (r - l) / 2`. Safe, readable, standard.
    
2. **Loop condition:**
    
    - `l <= r` for exact/leftmost/rightmost
    - `l < r` for boundaries
3. **Inequality signs:** Determine what you're searching for:
    
    - Use `<` for lower bound
    - Use `<=` for upper bound
    - Use `==` when exact match matters
4. **Shifting strategy:** Make progress:
    
    - `l = mid + 1` always advances (safest)
    - `r = mid` or `r = mid - 1` retreat as needed

**The ultimate skill:** Given a problem description, map it to ONE of the seven patterns. The code writes itself.

Practice these patterns until they're muscle memory. Then, binary search stops being a source of bugs and becomes your superpower.

---

## Practice Problems by Pattern

### Pattern 1: Exact Match

- Binary Search (LeetCode 704)
- Find in Sorted Array (LeetCode 33 - but rotated)

### Pattern 2 & 3: First/Last Occurrence

- First Bad Version (LeetCode 278)
- Find First and Last Position (LeetCode 34)

### Pattern 4 & 5: Boundaries

- Search Insert Position (LeetCode 35)
- Arrange Coins (LeetCode 441)

### Pattern 6: Closest

- Find Closest Number to Zero (LeetCode 2239)
- Valid Perfect Square (LeetCode 367)

### Pattern 7: Special Properties

- Search in Rotated Sorted Array (LeetCode 33)
- Find Peak Element (LeetCode 162)

---

**Happy Searching! 🔍**