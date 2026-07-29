# Bit Manipulation - DSA Reference Guide

## 🎯 Core Objectives

- Understand difference between logical operators (`&&`, `||`) vs bitwise operators (`&`, `|`, `^`)
- Master 1's and 2's complement representations
- Learn key bit manipulation tricks and patterns
- Solve LeetCode-style bit manipulation problems efficiently

---

## 1. Bitwise Operators Fundamentals

### 1.1 Unary Operators (Single Operand)

|Operator|Name|Description|Example|
|---|---|---|---|
|`~`|Bitwise Complement|Inverts all bits (one's complement)|`~3 = -4`|
|`+`|Unary Plus|Positive value|`+5`|
|`-`|Unary Minus|Negates expression|`-x`|
|`++` / `--`|Increment/Decrement|Increases/decreases by 1|`x++` or `++x`|
|`!`|Logical NOT|Inverts boolean|`!isTrue`|

### 1.2 Binary Operators (Two Operands)

#### Bitwise Operations

| Operator               | Name                 | Notation  | Effect                     |
| ---------------------- | -------------------- | --------- | -------------------------- |
| AND                    | Bitwise AND          | `A & B`   | Both bits must be 1        |
| OR                     | Bitwise OR           | `A \| B`  | At least one bit is 1      |
| XOR                    | Bitwise XOR          | `A ^ B`   | Bits are different         |
| Left Shift             | Left Shift           | `A << B`  | `A × 2^B`                  |
| Right Shift (Signed)   | Right Shift          | `A >> B`  | `A ÷ 2^B` (preserves sign) |
| Right Shift (Unsigned) | Unsigned Right Shift | `A >>> B` | `A ÷ 2^B` (fills with 0s)  |
| Bitwise Difference     | Difference           | `A & ~B`  | Bits in A but not B        |

#### Important Distinctions

⚠️ **Avoid Confusion:**

- `&` and `&&` are different: bitwise AND vs logical AND
- `|` and `||` are different: bitwise OR vs logical OR
- `^` has no logical equivalent
- `~` and `!` are both unary but different operators

### 1.3 Right Shift Operators: `>>` vs `>>>`

**`>>` (Signed Right Shift)** - Preserves sign bit

```java
int a = 10;      // Binary: 0000 1010
a >> 2;          // Result: 2 (0000 0010) - filled with 0s

int c = -10;     // Binary: 1111 0110 (two's complement)
c >> 2;          // Result: -3 (1111 1101) - filled with 1s
```

**`>>>` (Unsigned Right Shift)** - Always fills with 0s

```java
int a = 10;      // Binary: 0000 1010
a >>> 2;         // Result: 2 (same as signed for positive)

int c = -10;     // Binary: 1111 0110
c >>> 2;         // Result: 1073741821 (huge positive number) - all 1s replaced
```

---

## 2. Number Representation Systems

### 2.1 Binary, Octal, and Hexadecimal Literals in Java

```java
// Binary (prefix: 0b or 0B)
int binary = 0b00001010;              // Decimal 10
int binary_readable = 0b1010_0001;    // Underscores for readability

// Octal (prefix: 0)
int octal = 012;                      // Decimal 10
int octal2 = 010;                     // Decimal 8

// Hexadecimal (prefix: 0x or 0X)
int hex = 0xA;                        // Decimal 10
int hex_long = 0xFFL;                 // Long literal
```

### 2.2 Converting Between Number Systems

**Integer to Binary String:**

```java
// Without leading zeros
String binary = Integer.toBinaryString(5);        // "101"

// With leading zeros (32-bit)
String binary = String.format("%32s", Integer.toBinaryString(5))
                       .replace(' ', '0');        // "00000000000000000000000000000101"
```

**Binary String to Integer:**

```java
int num = Integer.parseInt("101", 2);             // 5
```

---

## 3. Negative Number Representations

### 3.1 One's Complement (Historical)

**Definition:** Invert all bits (flip 0s to 1s, 1s to 0s)
**Example:** Represent -5 in 8-bit
```
+5  = 00000101
-5  = 11111010 (flip all bits)
```

**Ways to compute one's complement:**

```java
~x                              // Using tilde operator
x ^ 0xFFFFFFFF                  // XOR with all 1s
-x - 1                          // Algebraic form
-(x + 1)                        // Alternative algebraic form
```

**Characteristics:**
- Two representations of zero: `00000000` and `11111111`
- Range for n-bit: `-(2^(n-1) - 1)` to `+(2^(n-1) - 1)`
- Not used in modern systems

### 3.2 Two's Complement (Modern Standard)

**Definition:** Invert all bits, then add 1
**Example:** Represent -5 in 8-bit

```
+5           = 00000101
Flip bits    = 11111010 (one's complement)
Add 1        = 11111011 (two's complement)
```

**Key Properties:**

- Only one representation of zero
- Range for n-bit: `-(2^(n-1))` to `+(2^(n-1) - 1)`
- Used in all modern computers
- In Java: `~(n - 1)` is effectively `-n`

**Example in Java:**

```java
int n = 5;
int negN = ~n + 1;              // Two's complement of 5 = -5
// OR simply
int negN = -n;                  // -5
```

**Verification:**

```java
int n = -10;                    // Binary: 1111 0110
int pos = ~n + 1;               // Returns: 10
// OR
int pos = ~(n - 1);             // Returns: 10
```

---

## 4. Essential Bit Tricks for DSA

### 4.1 Check if Number is Odd

```java
boolean isOdd = (n & 1) == 1;   // Check LSB (Least Significant Bit)
```

### 4.2 Turn OFF Rightmost 1-Bit

**Formula:** `x & (x - 1)`
**Mental Image:** Removes the rightmost set bit
**Uses:**
- Count number of 1-bits (Kernighan's algorithm)
- Check if power of 2

```java
// Example: 1101 (13)
int x = 13;                     // Binary: 1101
int result = x & (x - 1);       // 1101 & 1100 = 1100 (12)
```

### 4.3 Turn ON Rightmost 0-Bit
**Formula:** `x | (x + 1)`

```java
int x = 10;                     // Binary: 1010
int result = x | (x + 1);       // 1010 | 1011 = 1011 (11)
```

### 4.4 Isolate Rightmost Set Bit (LSB Isolation)

**Formula:** `x & -x` or `x & ~(x - 1)`

**Key insight:** In two's complement, `-x = ~x + 1`

**Uses:**

- Get position of rightmost 1-bit
- Check if power of 2

```java
int x = 12;                     // Binary: 1100
int rightmost = x & -x;         // 1100 & 0100 = 0100 (4)
// This isolates only the rightmost 1
```

### 4.5 Isolate Rightmost Unset Bit (LSZ Isolation)

**Formula:** `~x & (x + 1)` or `(~x) & (-(~x) - 1)`

```java
int x = 10;                     // Binary: 1010
int rightmost0 = ~x & (x + 1);  // 0101 & 1011 = 0001 (position of rightmost 0)
```

### 4.6 Check if Power of 2

**Formula:** `(x & (x - 1)) == 0` AND `x != 0`

```java
boolean isPowerOf2 = (x > 0) && ((x & (x - 1)) == 0);
```

### 4.7 Swap Two Numbers (No Temp Variable)

**Formula:** `x ^= y; y ^= x; x ^= y;`

```java
int a = 5, b = 7;
a ^= b;                         // a = a ^ b
b ^= a;                         // b = b ^ (a ^ b) = a
a ^= b;                         // a = (a ^ b) ^ a = b
// Now a = 7, b = 5
```

**Alternative with arithmetic:** `a = a + b; b = a - b; a = a - b;`

---

## 5. Bit Manipulation Properties

### Property 1: XOR and Set Bit Count

```
If: popcount(A) = X, popcount(B) = Y
Then: popcount(A ^ B) is even if (X + Y) is even
      popcount(A ^ B) is odd if (X + Y) is odd
```

### Property 2: Conditional Swap using XOR

When you need: `if (X == A) X = B; else if (X == B) X = A;`

Use concise version:

```java
X = A ^ B ^ X;                  // Elegant solution
```

### Property 3: Addition Formula

```
A + B = (A ^ B) + 2(A & B)
A + B = (A ^ B) + ((A & B) << 1)
A + B = (A | B) + (A & B)
```

---

## 6. Advanced Bit Algorithms

### 6.1 Count Set Bits (Kernighan's Algorithm)

**Concept:** Each iteration removes the rightmost 1-bit

```java
int countSetBits(int num) {
    int count = 0;
    while (num > 0) {
        num &= (num - 1);       // Remove rightmost 1
        count++;
    }
    return count;
}
```

**Time Complexity:** O(number of 1-bits)

**Java Built-in:**

```java
Integer.bitCount(num);          // Returns count of 1-bits
Integer.bitCount(num);          // For long: Long.bitCount(num)
```

### 6.2 Position of Rightmost Set Bit

**Formula:** Find 0-indexed position of rightmost 1-bit

```java
// Method 1: Using log
public int getPositionRightMostSetBit(int n) {
    int isolated = n & -n;
    return (int) (Math.log(isolated) / Math.log(2));
}

// Method 2: Java built-in (FASTEST)
public int getPositionRightMostSetBit(int n) {
    return Integer.numberOfTrailingZeros(n);
}
```

### 6.3 Position of Rightmost Unset Bit

**Formula:** Find 0-indexed position of rightmost 0-bit

```java
// Method 1: Invert, then isolate
public int getPositionRightMostUnsetBit(int n) {
    if (n == -1) return -1;     // All bits set
    int inverted = ~n;
    return Integer.numberOfTrailingZeros(inverted);
}

// Method 2: Direct formula
public int getPositionRightMostUnsetBit(int n) {
    return Integer.numberOfTrailingZeros(~n);
}
```

### 6.4 Add Two Numbers Without `+` Operator

**Concept:** Use XOR for sum, AND for carry

```java
public int getSum(int a, int b) {
    while (b != 0) {
        int carry = (a & b) << 1;  // Calculate carry
        a = a ^ b;                  // Sum without carry
        b = carry;                  // Update b for next iteration
    }
    return a;
}

// Example: a = 5, b = 3
// Iteration 1: a = 5^3=6, b = (5&3)<<1=4
// Iteration 2: a = 6^4=2, b = (6&4)<<1=4
// ... continues until b = 0
```

---

## 7. Java-Specific Bit Methods

### Integer Class Methods

```java
Integer.bitCount(n)                     // Count 1-bits
Integer.numberOfLeadingZeros(n)         // Count leading 0s
Integer.numberOfTrailingZeros(n)        // Count trailing 0s
Integer.highestOneBit(n)                // Highest 1-bit position
Integer.lowestOneBit(n)                 // Lowest 1-bit (rightmost 1)
Integer.rotateLeft(n, distance)         // Rotate bits left
Integer.rotateRight(n, distance)        // Rotate bits right
Integer.toBinaryString(n)               // Convert to binary string
Integer.valueOf("101", 2)               // Parse binary string
Integer.bitCount(n ^ m)                 // Hamming distance
```

### Long Class Methods

Same as Integer, use `Long.` prefix for long integers.

---

## 8. Common DSA Patterns

### Pattern 1: Check Bit Set at Position

```java
boolean isBitSet(int num, int pos) {
    return (num & (1 << pos)) != 0;
}
```

### Pattern 2: Set Bit at Position

```java
int setBit(int num, int pos) {
    return num | (1 << pos);
}
```

### Pattern 3: Unset Bit at Position

```java
int unsetBit(int num, int pos) {
    return num & ~(1 << pos);
}
```

### Pattern 4: Toggle Bit at Position

```java
int toggleBit(int num, int pos) {
    return num ^ (1 << pos);
}
```

### Pattern 5: Check if Bit Pattern Exists

```java
// Check if b is a subset of a (all set bits in b are set in a)
boolean isSubset(int a, int b) {
    return (a & b) == b;
}
```

### Pattern 6: Find Bit Difference (Hamming Distance)

```java
int hammingDistance(int x, int y) {
    return Integer.bitCount(x ^ y);
}
```

### Pattern 7: Mask First N Bits

```java
int maskFirstNBits(int num, int n) {
    return num & ((1 << n) - 1);
}
```

---

## 9. One-Liners & Quick Facts

```java
-(~n) == n + 1                          // Double negation and complement
~n == -(n + 1)                          // Complement equals negative minus one
x & (x - 1) == 0 && x != 0              // Check power of 2
x ^ (x - 1) == (x << 1) - 1             // Power of 2 property
(x | (x - 1)) + 1                       // Next power of 2
x & -x                                  // Isolate rightmost set bit
~x & (x + 1)                            // Isolate rightmost unset bit
```

---

## 10. Practice Resources

### LeetCode Problem Sets

- [Bit Manipulation Problem List](https://leetcode.com/problem-list/bit-manipulation/)

### Common LeetCode Problems

- Add Two Numbers Without `+` Sign
- Divide Two Integers Without `/` or `*`
- Single Number (XOR trick)
- Missing Number (XOR or math)
- Power of Two
- Hamming Distance
- Number of 1 Bits
- Reverse Bits
- Binary Watch
- Bitwise AND of Number Range

### Advanced Topics

- Bit Islands Counting
- Next Lexicographic Permutation with same set bits
- Bit Scan Forward (BSF)

---

## 11. Quick Reference Cheat Sheet

|Problem|Solution|
|---|---|
|Check if odd|`(n & 1) == 1`|
|Check power of 2|`(n > 0) && (n & (n-1)) == 0`|
|Get rightmost 1|`n & -n`|
|Get rightmost 0|`~n & (n+1)`|
|Count 1-bits|`Integer.bitCount(n)`|
|Swap a, b|`a ^= b; b ^= a; a ^= b;`|
|Remove rightmost 1|`n & (n-1)`|
|Toggle bit at pos|`n ^ (1 << pos)`|
|Check bit at pos|`(n & (1 << pos)) != 0`|
|Add without +|Use XOR for sum, AND for carry|

---

## Bibliography & Resources

- [Unlocking Bit Manipulation Secrets](https://www.youtube.com/watch?v=5kvgF0xN6SM)
- [10 Bitwise Tricks - Competitive Programming](https://www.youtube.com/watch?v=LGrE0siZ-ZA)
- [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html)
- [CP Algorithms - Bit Manipulation](https://cp-algorithms.com/algebra/bit-manipulation.html)
- [Position of Right Most Set Bit](https://aticleworld.com/position-of-right-most-set-bit/)