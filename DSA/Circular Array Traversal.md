# Circular Array Traversal: One Formula to Rule Them All 🔄

## Introduction

Traversing circular arrays seems simple on the surface, but handling arbitrary shifts (both positive and negative, both small and large) turns out to be surprisingly complex. Join me on a journey through three stages of problem-solving, culminating in a universal formula that works for ANY shift in ANY direction.

This isn't just about code optimization—it's about understanding how constraints evolve and how seemingly different problems can be unified into one elegant solution.

---

## The Problem 🤔

Imagine you have a circular array:
```
Index: 0 1 2 3 4 
Array: [A, B, C, D, E]
```
You're at index `i` and want to move `k` positions. Simple, right?
- Shift right by 2 from index 1? → `(1 + 2) % 5 = 3` ✓
- Shift left by 3 from index 1? → `(1 - 3) % 5 = ?` ❌

Welcome to the circular array traversal rabbit hole.

---

## Stage I: The Limited Formula Era 🏗️

### The Initial Problem
Different formulas for different directions:

```java
// Moving RIGHT (next)
next_idx = (i + k) % n;  // Works for k >= 0

// Moving LEFT (previous)  
prev_idx = (i - k + n) % n;  // Needs manual correction with +n
```

### Code Example
```java
class CircularArray_Stage1 {
    public int shiftRight(int i, int k, int n) {
        // Limited: k must be >= 0
        return (i + k) % n;
    }
    
    public int shiftLeft(int i, int k, int n) {
        // Limited: k must be <= i
        // Adding +n to handle negative modulo
        return (i - k + n) % n;
    }
    
    public static void main(String[] args) {
        int n = 5;
        int i = 1;
        
        System.out.println("Right by 2: " + shiftRight(i, 2, n));      // ✓ 3
        System.out.println("Left by 3: " + shiftLeft(i, 3, n));        // ✓ 3
        
        // But what about...
        System.out.println("Right by -7: " + shiftRight(i, -7, n));    // ❌ Wrong!
        System.out.println("Left by 10: " + shiftLeft(i, 10, n));      // ❌ Wrong!
    }
}
```

### Problems 🚫
- ✅ Works for limited ranges: `-(i) <= k <= (i+n)`
- ❌ No upper bound handling for left shifts
- ❌ No lower bound handling for right shifts  
- ❌ Language-dependent behavior (Java's modulo with negatives is problematic)
- ❌ Need different formulas for different directions

### Limitations Table

| Operation | Formula | Works When | Fails When |
|-----------|---------|-----------|-----------|
| Right shift | `(i + k) % n` | `k >= -i` | `k < -i` (wraps wrong) |
| Left shift | `(i - k + n) % n` | `k <= i+n` | `k > i+n` (wraps wrong) |

---

## Stage II: The Correction Era 🔧

### The Insight
"We need to ensure the dividend is positive before taking modulo!"

### Three Techniques Emerged

#### Technique 1: Double Mod (Manual Correction)
```java
class CircularArray_Stage2_DoubleMod {
    public int traverse(int i, int k, int n) {
        // First mod reduces the range
        // Second mod ensures positive result
        return ((i - k + n) % n + n) % n;
    }
    
    public static void main(String[] args) {
        int n = 5;
        int i = 1;
        
        System.out.println("Shift -3: " + traverse(i, -3, n));   // ✓ Correct!
        System.out.println("Shift -7: " + traverse(i, -7, n));   // ✓ Correct!
        System.out.println("Shift 10: " + traverse(i, 10, n));   // ✓ Correct!
    }
}
```

#### Technique 2: Pre-Reduce Technique
```java
class CircularArray_Stage2_PreReduce {
    public int traverse(int i, int k, int n) {
        // Reduce k to range [0, n) first
        // Then apply modulo
        k = k % n;
        return (i - k + n) % n;
    }
}
```

#### Technique 3: Java's Built-In Helper (Best! ✨)
```java
class CircularArray_Stage2_FloorMod {
    public int traverse(int i, int k, int n) {
        // Math.floorMod handles negative numbers automatically!
        return Math.floorMod(i - k, n);
    }
    
    public static void main(String[] args) {
        int n = 5;
        int i = 1;
        
        System.out.println("Shift -3: " + traverse(i, -3, n));   // 4 ✓
        System.out.println("Shift -7: " + traverse(i, -7, n));   // 3 ✓
        System.out.println("Shift 10: " + traverse(i, 10, n));   // 1 ✓
    }
}
```

### Progress Made ✅
- Handles negative and positive values
- Single formula works for both directions
- But: Still needs separate logic for left vs right

### Remaining Issues 🚫
- Two different formulas (one for left, one for right)
- Need to remember which is which
- Not yet "one formula to rule them all"

---

## Stage III: The Unified Era 👑

### The Breakthrough Insight
**Redefine k as displacement (signed), not just magnitude!**

Instead of:
- "Move left by 3" → separate formula
- "Move right by 2" → separate formula

Think of it as:
- `k = +3` → move right by 3
- `k = -3` → move left by 3

### The Universal Formula

```java
// ============================================
// ONE FORMULA TO RULE THEM ALL 👑
// ============================================

// Old School Way (Compatible with all languages)
nextIdx = (((i + k) % n) + n) % n;

// Modern Java Way (Java 8+) ⭐ RECOMMENDED
nextIdx = Math.floorMod(i + k, n);

// Semantic Expression
new_index = Math.floorMod(i + (direction × steps), n);
// where direction = +1 for Right, -1 for Left
```

### Complete Implementation

```java
class CircularArray_Stage3_Universal {
    
    /**
     * Universal formula: works for ANY k, ANY direction
     * @param currentIndex current position (0 to n-1)
     * @param shift signed displacement (+ve = right, -ve = left)
     * @param size array size
     * @return new index after shift
     */
    public static int traverse(int currentIndex, int shift, int size) {
        return Math.floorMod(currentIndex + shift, size);
    }
    
    /**
     * Semantic alternative: explicit direction and steps
     */
    public static int traverse(int currentIndex, int direction, int steps, int size) {
        // direction: +1 for right (next), -1 for left (prev)
        return Math.floorMod(currentIndex + (direction * steps), size);
    }
    
    public static void main(String[] args) {
        int n = 5;
        int i = 1;
        
        System.out.println("=== Universal Formula Tests ===\n");
        
        // Positive shifts (right)
        System.out.println("Shift +2: " + traverse(i, 2, n));      // 3 ✓
        System.out.println("Shift +7: " + traverse(i, 7, n));      // 3 ✓
        System.out.println("Shift +12: " + traverse(i, 12, n));    // 3 ✓
        
        // Negative shifts (left)
        System.out.println("Shift -1: " + traverse(i, -1, n));     // 0 ✓
        System.out.println("Shift -3: " + traverse(i, -3, n));     // 3 ✓
        System.out.println("Shift -8: " + traverse(i, -8, n));     // 3 ✓
        
        System.out.println("\n=== Semantic Alternative ===\n");
        
        // Right shifts
        System.out.println("Right 2 steps: " + traverse(i, 1, 2, n));    // 3 ✓
        System.out.println("Right 7 steps: " + traverse(i, 1, 7, n));    // 3 ✓
        
        // Left shifts
        System.out.println("Left 1 step: " + traverse(i, -1, 1, n));     // 0 ✓
        System.out.println("Left 3 steps: " + traverse(i, -1, 3, n));    // 3 ✓
    }
}
```

### Key Advantages 🎉

✅ **Size agnostic**: Works with any array size
✅ **Sign agnostic**: Works with any shift direction
✅ **Magnitude agnostic**: Works with any shift magnitude
✅ **Single formula**: One-liner, no conditionals
✅ **Language compatible**: Works in Java, C++, Python, etc.
✅ **Semantically clear**: Easy to understand intent
✅ **Production-ready**: No edge cases

---

## Comparative Analysis 📊

### Formula Comparison

| Stage   | Formula                 | Positive k  | Negative k | Huge k | Single Formula       |
| ------- | ----------------------- | ----------- | ---------- | ------ | -------------------- |
| **I**   | Multiple                | ✅ (limited) | ❌          | ❌      | ❌                    |
| **II**  | `Math.floorMod(i-k, n)` | ✓           | ✓          | ✓      | ❌ (still 2 formulas) |
| **III** | `Math.floorMod(i+k, n)` | ✓           | ✓          | ✓      | ✅                    |
### Time Complexity
All stages: **O(1)** – Single modulo operation
### Code Complexity

Stage I:  Complex (multiple formulas) 
Stage II: Better (but still 2 paths) 
Stage III:  Elegant (one formula)

---

## Why Java's Math.floorMod Matters 🔑

The reason Java's `Math.floorMod` is special:

```java
// Regular modulo (broken for negatives in Java)
-3 % 5 = -3  ❌ (we want 2)

// Math.floorMod (correct!)
Math.floorMod(-3, 5) = 2  ✅

// Double mod workaround (works but verbose)
((-3 % 5) + 5) % 5 = 2  ✓ (but ugly)
```

**This is why Stage III with `Math.floorMod` is the modern solution.**

---

## Real-World Example: LRU Cache Circular Buffer 💾

```java
class CircularBuffer {
    private int[] buffer;
    private int capacity;
    private int readPointer = 0;
    
    public CircularBuffer(int capacity) {
        this.buffer = new int[capacity];
        this.capacity = capacity;
    }
    
    /**
     * Move read pointer by offset (can be +ve or -ve)
     * Using Stage III universal formula!
     */
    public void moveReadPointer(int offset) {
        readPointer = Math.floorMod(readPointer + offset, capacity);
    }
    
    /**
     * Peek ahead or behind without changing pointer
     */
    public int peekAt(int offset) {
        int newPointer = Math.floorMod(readPointer + offset, capacity);
        return buffer[newPointer];
    }
    
    public static void main(String[] args) {
        CircularBuffer buf = new CircularBuffer(5);
        
        // Read position at 1
        buf.readPointer = 1;
        
        // Move forward 2
        buf.moveReadPointer(2);   // Now at 3
        
        // Move backward 5 (wraps around)
        buf.moveReadPointer(-5);  // Now at 3 (wrapped correctly!)
        
        // Peek ahead 10
        int val = buf.peekAt(10); // Doesn't change pointer, correctly wraps
    }
}
```

---

## Journey Summary 📝

| Aspect | Stage I | Stage II | Stage III |
|--------|---------|----------|-----------|
| **Formulas** | 2 | Still 2 | 1 ✅ |
| **Flexibility** | Very Limited | Better | Universal ✅ |
| **Readability** | Needs comments | Better | Obvious ✅ |
| **Production Ready** | ❌ | ⚠️ | ✅ |
| **LOC for solution** | 10+ | 5-7 | 1 |

---
## Key Learnings 🎓

1. **Redefine the problem**: Shifting left by k = shifting right by -k
2. **Use signed displacement**: k encodes both direction and magnitude
3. **Trust language utilities**: `Math.floorMod` exists for a reason
4. **Unification over specialization**: One formula beats many special cases
5. **Progressive problem-solving**: Start simple, identify patterns, generalize

---
## The Final Formula 🏆

```java
// For any circular array traversal:
newIndex = Math.floorMod(currentIndex + displacement, arraySize);

// Where:
// - currentIndex: current position (0 to n-1)
// - displacement: signed integer (+ = right, - = left)
// - arraySize: size of the circular array
// - Result: valid index in range [0, arraySize)
```
**This is the one formula to rule them all.**

---

## Tabular Compilation
[Circular Array Traversal - Google Sheets](https://docs.google.com/spreadsheets/d/e/2PACX-1vRfMbHiJynZImn_vAP-5k3LMkry-50ct9CBUDO54c3Zkq4yo4SIhnWnxH5-ZtlLICdQZesYsaqxmVc7/pubhtml?gid=1311287431&single=true) 

| Stage | Range of the shift (k)                              | Formula                                      | Remark                                                                                     | Limitations (Java) : (-1 % n) = -1 and not n-1                                                                                |
| ----- | --------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| I     | `-(i) <= k <= (i+n)`                                |                                              | limited shift in any direction                                                             | Size & Sign dependent. (works for given k range only)                                                                         |
|       |                                                     |                                              |                                                                                            |                                                                                                                               |
|       | `-(i) <= k <= +inf`                                 | `next_idx = (i+k ) % n`                      | no upper bound on k for +ve right shift                                                    | lower bound on k for -ve right shift `-i <= k` e.g. from `i=1` shift right by k= -2                                           |
|       | `-inf <= k <= (i+n)`                                | `prev_idx = (i-k+n) % n`                     | no lower bound on k for -ve left shift                                                     | upper bound on k for +ve left shift k <= (i+n)                                                                                |
|       |                                                     | Note: shift(k,←) ≡ shift(n-k,→)              |                                                                                            | Reason:% of (-)ve is lang dependent (Unsafe for Java).                                                                        |
|       |                                                     |                                              |                                                                                            |                                                                                                                               |
| II    | `-(i) <= k <= +inf`                                 |                                              | infinite shift in left direction  <br>still limited shift in right direction               | Right Shift still Sign & Size dependent <br><br>Assumes shift to be distance not displacement. Unsuitable for physics/vectors |
|       | `-inf <= k <= +inf`                                 | `prev_idx = [{(i-k+n) % n} + n] % n`         | Double Mod Technique                                                                       | Manual mod correction                                                                                                         |
|       | `-inf <= k <= +inf`                                 | `prev_idx = { i-(k%n) + n} % n`              | Pre-Reduce Technique                                                                       | Manual mod correction                                                                                                         |
|       | `-inf <= k <= +inf`                                 | `prev_idx = Math.floorMod(i-k, n)`           | Pro-Java Way                                                                               | Automatic mod correction (Java 8+ Built-In Helper)                                                                            |
|       | `-(i) <= k <= +inf`                                 | `next_idx = (i+k) % n`                       |                                                                                            | still cannot shift right by large -ve value i.e. fails if k < -i                                                              |
|       |                                                     |                                              |                                                                                            |                                                                                                                               |
| III   | k ∈ Z                                               | `nextIdx = (((i + k) % n) + n) % n`          | Universal "Old School" Way (Compatible with C++/Old Java)                                  | one formula to rule them all - Any Language                                                                                   |
|       | any integer k (positive, negative, huge, or small). | nextIdx = Math.floorMod(i + k, n)            | Modern Java Way (Java 8+)                                                                  | one formula to rule them all - Java recommended                                                                               |
|       | k = displacement (+ve for Right, -ve for Left)      |                                              |                                                                                            |                                                                                                                               |
|       |                                                     | `new_index = Math.floorMod(i+(dir×steps),n)` | Let dir be the direction: +1 for Right/Next, -1 for Left/Prev. Let steps be the magnitude. | Semantic expression                                                                                                           |
|       |                                                     |                                              |                                                                                            |                                                                                                                               |

**Happy circular traversing! 🔄**