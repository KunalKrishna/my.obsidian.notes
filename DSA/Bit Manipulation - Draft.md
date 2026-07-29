
Aim : 
- understand diff b/w logical operators & bitwise operators
- understand 1's & 2's complement of a number
- range of results of a|b , a&b, a^b wrt values of a,b
- tricks to bit manipulation
	- odd check (LSB)
	- swap (`a ^ b`)
	- Turn **OFF** rightmost **1** : $x \& (x-1)$
		- count number of 1's
		- 2^k
	- Turn **ON** rightmost **0** : $x \vert (x+1)$ 
	- Extracting the rightmost set bit / Isolate LSB / **BSF**
		- Position of rightmost 1 : `(x & -x)`
		- **Finding the LSB:** In Java, you extract the LSB using the bitwise operation `i & -i`.
	- Extracting the rightmost unset bit / Isolate Lowest 0 / Isolate LSZ(Least Significant Zero )
		- Position of rightmost 0 = pos of rm 1 in (nInverted = ~n)
- More Tricks
	- Masked Copy 
## Binary
In Java, **unary operators** require only one operand, while **binary operators** require two.
### BitWise operators
#### Unary Operators
Unary operators perform operations on a single value, such as incrementing/decrementing a value, negating an expression, or inverting a boolean's value.

| Operator | Name               | Description                                                                               | Example        |
| -------- | ------------------ | ----------------------------------------------------------------------------------------- | -------------- |
| `+`      | Unary plus         | Indicates a positive value (though numbers are positive by default).                      | `+5`           |
| `-`      | Unary minus        | Negates an expression, changing the sign of the value.                                    | `-x`           |
| `++`     | Increment          | Increases the value of an operand by 1. It can be pre- (`++x`) or post-increment (`x++`). | `x++` or `++x` |
| `--`     | Decrement          | Decreases the value of an operand by 1. It can be pre- (`--x`) or post-decrement (`x--`). | `x--` or `--x` |
| `!`      | Logical NOT        | Inverts the value of a boolean.                                                           | `!isTrue`      |
| `~`      | Bitwise complement | Inverts all the bits of an integer value (one's complement). ~3 = -4                      | `~x`           |
#### Binary Operators
operate on two operands, usually one on the left and one on the right. Most of Java's operators fall into this category. 
- **Arithmetic Operators:** Perform mathematical calculations.
    - `+` (Addition)
    - `-` (Subtraction)
    - `*` (Multiplication)
    - `/` (Division)
    - `%` (Modulus/Remainder)
- **Relational (Comparison) Operators:** Compare two values and return a boolean result.
    - `==` (Equal to)
    - `!=` (Not equal to)
    - `>` (Greater than)
    - `<` (Less than)
    - `>=` (Greater than or equal to)
    - `<=` (Less than or equal to)
- **Logical Operators:** Used to perform "Logical AND", "Logical OR", etc., on boolean expressions.
    - `&&` (Logical AND)
    - `||` (Logical OR)
- **Bitwise Operators:** Perform operations on individual bits of data.
    - `&` (Bitwise AND)
    - `|` (Bitwise OR)
    - `^` (Bitwise XOR)
    - `<<` (Left shift)
    - `>>` (Signed right shift)
    - `>>>` (Unsigned right shift)
- **Assignment Operators:** Assign a value to a variable, often in combination with other operations (e.g., `+=`, `-=`).
    - `=` (Simple assignment)
    - `+=` (Add and assign)
    - `-=` (Subtract and assign)
    - `*=` (Multiply and assign)
    - `/=` (Divide and assign)
    - `%=` (Modulus and assign)

<mark style="background-color: #fff88f; color: black">NOTE</mark> : 
- only bitwise op `&` and ` |` has binary logical equivalent `&&` and `||` (scope of confusion)
- `xor` and `shift` operators are exclusively binary (no scope of confusion)
- `~` and `!`  operators are exclusively unary (no scope of confusion)

Bitwise Operators (Topic of our interest)
* Bitwise complement : ~ (or bit flip operator) - the only unary operatory of interest
* AND : `A & B`
* OR    : `A | B` 
* XOR  : `A ^ B`
* Left Shift : `A << B` : $A < <  B = A * 2^B$
* (Signed) Right Shift : `A >> B` : retains sign of original number $A >> B = A / 2^B$
* Unsigned Right Shift : `A >>> B`  : always makes it positive   $A >> B = |A| / 2^B$
* Bitwise Diff : `A & ~B` : 


##### `>>` vs `>>>`  (Signed Right Shift) vs (Unsigned Right Shift)
**`>>` (Signed Right Shift)**
Preserves the sign of the original number. It fills the vacated leftmost bits with the **sign bit** (0 for positive numbers, 1 for negative numbers).
```Java
int a = 10; // Binary: 0000 1010
int b = a >> 2; // Result: 2

// Explanation:
// Original: 0000 1010
// Shift right by 2: 0000 0010 (Decimal 2)
// The two new bits are filled with the sign bit (0).

int c = -10; // Binary (two's complement): 1111 0110
int d = c >> 2; // Result: -3

// Explanation:
// Original: 1111 0110
// Shift right by 2: 1111 1101 (Decimal -3)
// The two new bits are filled with the sign bit (1).
```
**`>>>` (Unsigned Right Shift)**
Does not preserve the sign. It always fills the vacated leftmost bits with **zeros**, regardless of whether the original number was positive or negative.
```Java
int a = 10; // Binary: 0000 1010
int b = a >>> 2; // Result: 2

// Explanation:
// Original: 0000 1010
// Shift right by 2: 0000 0010 (Decimal 2)
// The two new bits are filled with 0s.
// (For positive numbers, >> and >>> produce the same result.)

int c = -10; // Binary (two's complement): 1111 0110
int d = c >>> 2; // Result: 1073741821 (A large positive number)

// Explanation:
// Original: 1111 0110
// Shift right by 2: 0011 1101 ... (lots of 1s in 32-bit int)
// The two new bits are filled with 0s, forcing the number to become positive.

```

### Representing different system in code
**Binary Literals** use the prefix `0b` or `0B` followed by a sequence of `0`s and `1`s.
**Octal Literals** use the prefix `0` followed by a sequence of digits from `0` to `7`.
**Hexadecimal Literals** use the prefix `0x` or `0X` followed by a sequence of digits (`0`-`9`) and letters (`a`-`f` or `A`-`F`).

```Java
int binaryValue = 0b00001010; // Represents the decimal value 10

int octalValue = 012;  // Represents the decimal value 10
int octalValue1 = 010; // Octal 010 is decimal 8

int hexValue = 0xA; // Represents the decimal value 10
```
These literals can be used with integer data types like `byte`, `short`, `int`, and `long`. For `long` literals, you can append an `L` (or `l`) at the end of the number, after the prefix. For example: `0b101L` or `0xFFL`. 

Additionally, Java allows the use of the underscore character (`_`) to separate digits in any numeric literal for better readability (e.g., `0b1010_0001`)

**Convert binary code directly into an integer** in Java : int number = 0b011;
**Convert integer directly into an binary** literal in Java : two ways
1. No Leading Zeros : `String binary = Integer.toBinaryString(3);`
2. With Leading Zeros (Formatted): `String binary = String.format("%32s", Integer.toBinaryString(number)).replace(' ', '0');`
```java
int number = 5; // Binary is 101 
String binary = Integer.toBinaryString(number);
System.out.println(binary); // Output: "101"

int number = 5;
// Format to 32 bits, replace spaces with zeros
String binary = String.format("%32s", Integer.toBinaryString(number)).replace(' ', '0');
System.out.println(binary); 
// Output: 00000000000000000000000000000101
```
### Binary representation of negative numbers in digital systems
* how negative numbers are represented in digital systems (fundamental concept)
* How negative number is stored in 1's complement ? It's drawback?
* How negative number is stored in 2's complement ? 
#### 1's complement : "Invert" 
Negative numbers in 1's complement are stored ==by taking the positive binary representation of a number and **inverting (flipping) all of its bits** (changing 0s to 1s and 1s to 0s)==. The most significant bit (MSB) acts as a sign indicator, where '1' indicates a negative number.

**Steps to Represent a Negative Number in 1's Complement** 
$$\sim x : one's\ complement\ of\ a\ number\ =\ invert\ every\ bit\ of\ the\ number$$
To represent a negative number (e.g., -5) in an 𝑛-bit system: 
1. **Find the positive binary representation:** Determine the binary for the positive value (e.g., +5 in 8-bit is `00000101`).
2. **Invert all bits:** Flip every bit, including the sign bit (e.g., `00000101` becomes `11111010`). 

$$-9_{10} \xrightarrow{ \color{red}{1}\color {black} \color{black} {'s\ complement} } (????)_{2} $$
$$-9 \xrightarrow{abs} +9 \xrightarrow{binary} 0000\_{}1001 \xrightarrow{bitwise\ complement (\sim)} 1111\_{}0110$$
Verify by adding +9(0000 1001) & -9(1111_0110) = 1111 1111 = 0 in 1's Complement  

Ways to invert all bits : 
1. `~x` : use tilde(~) unary operatory
2. `~x = (x ^ 0xffff_ffff)`  : `xor` the number with  `ALL_ONES` 
3. `~x = -x - 1`  : From standard algebra, `~A` is equivalent to `-A - 1`.
4. `~x = -(x+1)` : see 2's complement :  substitute t = x+1 in `-t = ~(t - 1)` 

**Example (-5 in 8-bit):** 
- +5: `00000101`
- -5 (1's complement): `11111010` 
**Key Characteristics** 
- **Two Representations of Zero:** 1's complement allows for both a positive zero (`00000000`) and a negative zero (`11111111`).
- **Range:** For an 𝑛-bit word, the range is $−(2^{n−1}-1)$  to  $+(2^{n−1}-1)$ .
- **End-Around Carry:** When performing addition, if a carry is generated from the MSB, it must be added back to the least significant bit (LSB).
- **Not Used in Modern Computers:** While 1's complement is useful for understanding, most modern systems use 2's complement for integer representation because it avoids "negative zero" and streamlines arithmetic.

The tilde symbol (~) in programming languages like C/C++ is the bitwise NOT operator, which performs the 1's complement operation by flipping all the bits (0s to 1s, 1s to 0s) of a binary number.

`~x : one's complement of a number = invert every bit of the number = bitwise flip of every bit in binary representation`

#### 2's complement : "Invert then Add 1" 
**Steps to Represent a Negative Number in 2's Complement** 
1. `-x = (~x) + 1` : "Invert then Add 1"  
2. `-x = ~(x - 1)` : "Subtract 1 then Invert" 
3. `-x = (x ^ 0xffff_ffff ) + 1` (identical to expression #1)
4. Flip all bits to the left of the rightmost set bit.

NOTE: 
`~(A - B) = ~A + B` : "The complement of a difference is the complement of the minuend plus the subtrahend."

$$(-9)_{10} \xrightarrow{ \color{red}{2}\color {black} \color{black} {'s\ complement} } (????)_{2} $$

**Approach 1** : `-x = (~x) + 1`
$$-9_{10} \xrightarrow{ \color{red}{1}\color {black} \color{black} {'s\ complement} } (1111\_{}0110)_{2} \xrightarrow{+1} 1111\_{}0111 $$
$$-9 \xrightarrow{abs} +9 \xrightarrow{binary} 0000\_{}1001 \xrightarrow{bitwise\ complement (\sim)} 1111\_{}0110 \xrightarrow{+1} 1111\_{}0111 $$
Verify : add +9(0000 1001) & -9(1111 0111) = 0000 0000 = 0 in 2's Complement

**Approach 2** : `-x = ~(x - 1)`
2's complement of n = 1's complement of (n-1)

$$-9 \xrightarrow{abs} +9 \xrightarrow{x-1} +8 \xrightarrow{binary} 0000\_{}1000 \xrightarrow{bitwise\ complement (\sim)} 1111\_{}0111$$
**Approach 3** : `(n ^ 0xffff_ffff )+ 1`

```java
public int twosComplement(final int value) {
        final int ALL_ONES = 0xffff_ffff;
        System.out.println("2s compl of"+value+ "="+ Integer.toBinaryString((value ^ mask) + 1));
        return (value ^ ALL_ONES) + 1;
    }
```

**Approach 4** : Manual Approach : **Flip all bits to the left of the rightmost set bit.**

**The Algorithm (Right-to-Left Scan):**
1. Keep all bits the same **up to and including** the first `1` you encounter from the right.
2. **Flip** all bits to the left of that `1`

 x = 0b0110 **1000** = 104
-x = 0b1001 **1000** = -104
$$x(=5)=0000\_{}010\color{red}1 \color{black} \rightarrow 1111\_{}101\color{red}1$$
$$x(=42)=0010\_{}1\color{red}100 \color{black} \rightarrow 1101\_{}0\color{red}100$$

```
   x   =  5 = 0000 0101
  x-1  =  4 = 0000 0100
~(x-1) = -5 = 1111 1011  // rightmost 1 unchanges

    x  =  5 = 0000 0101
   ~x  = ~5 = 1111 1010
  ~x+1 = -5 = 1111 1011
```

In a 32-bit two's complement system, the number **-2^31** (or -2,147,483,648) is stored as
$-2^{31}$ = `1000 0000 0000 0000 0000 0000 0000 0000`
- **A Special Case:** To see how this representation arises, one might expect to convert the positive value of 2^31 (which is out of range for a positive 32-bit integer) and take its two's complement. Instead, this specific bit pattern `100...0` is _defined_ as the most negative number in the system, allowing for an asymmetric range that includes one more negative number than positive numbers.
- **Unique Representation:** The number -2^31 is a special case; it is the only negative number in the range that does not have a corresponding positive counterpart that can be negated to produce it within the same bit-length (negating it results in an overflow). Its two's complement is itself.

Imp properties :
- -t = ~t +1
- -t = ~(t-1)
- -(~t) = (t + 1)  {substitute t with ~p  and then p with t}
- -(~t) = ~(~t - 1) {substitute t with ~p  and then p with t}
- ~t = -t -1 (same as eq 1)

Ponder :
- If a 2's complement of a (negative) number is given , what is the (positive) number ? `~x = 111...011 then x = ? 
	- Ans : reapply 2's complement on ~x  = ~(111...011) = 000...100 + 1 = 000...101 = 5
	-  
- 
### Range Observations

| **Operation**    | **Notation** | **Range Lower Bound** | **Range Upper Bound** | **Logic / Pattern**                                           |
| ---------------- | ------------ | --------------------- | --------------------- | ------------------------------------------------------------- |
| **AND**          | `A & B`      | $0$                   | $\min(A, B)$          | Intersection of bits.                                         |
| **XOR**          | `A ^ B`      | $0$                   | $A + B$               | Addition without carry.                                       |
| **OR**           | `A\|B`       | $\max(A, B)$          | $A + B$               | Union of bits.                                                |
| **Right Shift**  | `A >> B`     | $0$                   | $A$                   | **Division:** $\lfloor A / 2^B \rfloor$. Shrinks $A$.         |
| **Left Shift**   | `A << B`     | $A$                   | $A \times 2^B$        | **Multiplication:** Grows exponentially. Breaks $A+B$ bounds. |
| **Bitwise Diff** | `A & ~B`     | $0$                   | $A$                   | **Subtraction:** Removes bits of $B$ from $A$.                |
### Odd / Even check : (x&1)
`if (num & 1 == 1 )`
* true  $\Rightarrow$   **odd**  - num=5 (0101 **&** 0001 = **1**)
* false  $\Rightarrow$  even - num=6 (0110 & 0001 = 0)
![[Pasted image 20260119235437.png]]
### If any of two number is negative : (a|b) < 0
`if( (a|b) < 0 )`
```java
int a = 3;
int b = 2;

if((a|b) > 0)	System.out.println("positive");
if((a*b) > 0)	System.out.println("positive");

b = -2;
if((a|b) < 0)	System.out.println("negative");
if((a*b) < 0)	System.out.println("negative");

a = -3;
if((a|b) < 0)	System.out.println("still negative");
if((a*b) > 0)	System.out.println("positive");
```

### If only one of two number is negative : (x ^ y) < 0 
```java
boolean isNegative = (dividend ^ divisor) < 0; 

int quotient_abs = division_funky_fun( Math.abs(dividend), Math.abs(divisor) );

int sign = (isNegative)? -1 : +1;

return quotient_abs*sign ;
```

# XOR

### XOR of first `n` natural numbers
[XOR of 1 to n Numbers - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/calculate-xor-1-n/)


| n   | XOR (1^2^3^...^n-1^n)            | Result | Result as a function of n |
| --- | -------------------------------- | ------ | ------------------------- |
| 1   | 01                               | 1      | 1                         |
| 2   | 01 ^ 10                          | 3      | n+1                       |
| 3   | 01 ^ 10 ^ 11                     | 0      | 0                         |
| 4   | 01 ^ 10 ^ 11 ^ 100               | 4      | n                         |
| 5   | 01 ^ 10 ^ 11 ^ 100^101           | 1      | 1                         |
| 6   | ........................^101^110 | 7      | n+1                       |
| 7   | ........................^110^111 | 0      | 0                         |
| 8   | ...................111 ^ 1000    | 8      | n                         |
**Corollary** : if `xor` of first `n` natural numbers is `P` and `xor` of first `m` (<n) natural numbers is `Q` then `xor` of number `[ (m+1 )^ (m+2) ^ ........^ (n-1) ^ n]` is P ^ Q. **Prefix XOR Property**.

**Prefix XOR Property** : 
If  : 
$$

P = 1 \oplus 2 \oplus \dots \oplus m \oplus (m+1) \oplus \dots \oplus n

$$

$$Q = 1 \oplus 2 \oplus \dots \oplus m$$
Then,
$$P \oplus Q = (m+1) \oplus \dots \oplus n$$

### Swap two nums using XOR (^)
- a = 3, b = 6
- a = a ^ b 
- b = a ^ b (=3)
- a = a ^ b (=6) 

Using Addition(+) & Subtraction(-)
```java
// Assuming a and b are the two numbers to be swapped.
a = a + b;  // a now holds the sum of the original a and b
b = a - b;  // b now holds the value of the original a (a + b - b = a)
a = a - b;  // a now holds the value of the original b (original a - original b + original b = original b)
```
For example, if `a = 10` and `b = 5`:
1. `a = 10 + 5` (15) = “hybrid”
2. `b = 15 - 5` (10)
3. `a = 15 - 10` (5)

Swapping in 1 line : `a = (a+b) - (b=a);`

Using XOR(^)
```java
a = a ^ b;  // a becomes the XOR of the original a and b
b = a ^ b;  // b becomes the original a (a ^ b ^ b = a)
a = a ^ b;  // a becomes the original b (original a ^ original b ^ original a = original b)
```
For example, if `a = 10` (binary `1010`) and `b = 5` (binary `0101`):
1. `a = 10 ^ 5` (binary `1111`, which is 15) = “hybrid”
2. `b = 15 ^ 5` (binary `1111 ^ 0101` = `1010`, which is 10)
3. `a = 15 ^ 10` (binary `1111 ^ 1010` = `0101`, which is 5)

[Swap two variables using XOR – BetterExplained](https://betterexplained.com/articles/swap-two-variables-using-xor/)

### Num Power of 2 ? 
2 = 0010
4 = 0100
8 = 1000

*isPowerOf2(num)* : if binary representation of num has only one 1.

`if (num & (num-1) == 0)`  
- true  $\Rightarrow$   **power of 2**  - num=8 (**1**000 **&** 0111 = **0**000)
* false  $\Rightarrow$  not power of 2 - num=6 (01**1**0 & 0101 = 01**0**0) :  actually sets rightmost $1\rightarrow0$ 
* exception :  `num=0` 
*more on power of $x \& (x-1)$ later*

![[Pasted image 20260120002534.png]]
### Playing with (get/set/unset) k-th bit in a number
1. **Check** if the Kth bit is set / [Get Kth bit of N](https://stackoverflow.com/questions/2249731/how-do-i-get-bit-by-bit-data-from-an-integer-value-in-c)  **(AND with 1)**
	`bool bit_K = num & (1 << K)`
	If you want the Kth bit of n, then do : 
		`(n & ( 1 << k )) >> k`
		`(n >> k) & 1 `is equally valid
2. **Set** Kth bit of N:  **(OR with 0)**
	 `num |= (1 << pos)`
3. **Unset** Kth bit of N: **(AND with 0)**
	`num &= (~(1 << pos))`
4. **Toggling** a bit at Kth position:  **(XOR with 1)**
	`num ^= (1 << pos)`

Of particular interest is the **rightmost** set or unset bit.
### Clear all bits from LSB to the ith bit

```java
mask = ~((1 << i+1 ) – 1);
x &= mask;
```
**Logic:** To clear all bits from LSB to i-th bit, we have to AND x with mask having LSB to i-th bit 0. To obtain such mask, first left shift 1 i times. Now if we minus 1 from that, all the bits from 0 to i-1 become 1 and remaining bits become 0. Now we can simply take complement of mask to get all first i bits to 0 and remaining to 1.
**Example:**
```java
x = 29; //(00011101) and we want to clear LSB to 3rd bit, total 4 bits
mask = 1 << 4 // 16(00010000)
mask = 16 – 1 // 15(00001111)
mask = ~mask // 11110000
x & mask // 16 (00010000)
```
### Clearing all bits from MSB to i-th bit
```java
mask = (1 << i) – 1;
x &= mask;
```

**Logic:** To clear all bits from MSB to i-th bit, we have to AND x with mask having MSB to i-th bit 0. To obtain such mask, first left shift 1 i times. Now if we minus 1 from that, all the bits from 0 to i-1 become 1 and remaining bits become 0.

**Example:**
```java
x = 215 (11010111) and we want to clear MSB to 4th bit, total 4 bits
mask = 1 << 4 // 16(00010000)
mask = 16 – 1 // 15(00001111)
x & mask // 7(00000111)
```
### Upper case English alphabet to lower case : manipulating the 5th bit
`ch |= ' '; // OR with space` 

traditional trick : 
- °F = (9/5)× °C  **+ 32**
- lc = UC **+ 32** (U°C **+ 32**)
new trick
- lc = UC | ' ';

![[ascii lc uc 2.png]]

| char/symbol | DEC    | BIN          | OCT  | HEX  | Description                          |
| ----------- | ------ | ------------ | ---- | ---- | ------------------------------------ |
| '  '        | 32     | 00**1**00000 |      |      | Space (only bit 5 is set!)           |
| 0           | 48     |              | 060  | 0x30 | Zero                                 |
| A           | **65** | 01**0**00001 | 0101 | 0x41 | Uppercase a                          |
| '_'         | 95     | 01011111     | 0137 | 0x5F | Underscore(everything except bit 5!) |
| a           | **97** | 01**1**00001 | 141  | 0x61 | Lowercase a                          |
The Design of ASCII itself gives away easy & efficient way to toggle b/w cases : 

> [!NOTE]
> ASCII chose 65 for 'A' and 97 for 'a' ==to fit within early 7-bit codes, grouping uppercase (65-90) and lowercase (97-122) in sequence, with a key design decision to make case conversion a simple single-bit flip (a 32-bit difference) for efficient sorting and keyboard/printer hardware==. 'A' (01000001) and 'a' (01100001) differ only in the 6th bit(1-indexed), allowing easy case toggling with simple bitwise operations, a core goal for early computing efficiency.

```java
char toLowerCase(char upperCaseChar) { // ='B'
	// If it is uppercase, add 32 to its ASCII value to convert to lowercase
	return upperCaseChar + 32; // 'B' (ASCII 66) + 32 = 'b' (ASCII 98)
	//'B' (0100_0010) | 32 (0010_0000) = 'b' (0110_0010)
	
    return upperCaseChar | 0b0010_0000;
    return upperCaseChar | 040;   // Octal
    return upperCaseChar | 0x20; // Hexadecimal
    return upperCaseChar | ' '; // space char
}

char toUpperCase(char lowerCaseChar) { // ='b'
	// If it is lowercase, subtract 32 to its ASCII value to convert to uppercase
	return lowerCaseChar - 32; // 'b' (ASCII 98) - 32 = 'B' (ASCII 66)
	
    return lowerCaseChar & 0b0101_1111; // NOTE: it is not ~(32)
    return lowerCaseChar & 0137;  // Octal
    return lowerCaseChar & 0x5F; // Hexadecimal
    return lowerCaseChar & '_'; // Underscore char
}
```

$$'A'\xrightleftharpoons[clear\ \color{red}5-th\ \color{black}bit]{set\ \color{red}5-th\ \color{black}bit}\ 'a'$$

$$'A'\xrightharpoonup {set\ \color{red}5-th\ \color{black}bit}\ 'a'$$
**Logic:** The bit representation of upper case and lower case English alphabets are:
A -> 01000001 a -> 01100001
B -> 01000010 b -> 01100010
C -> 01000011 c -> 01100011
. .
. .
Z -> 01011010 z -> 01111010

We can set **5th bit(0-indexed) of upper case character**s, it will be converted into lower case character. Make mask having 5th bit 1 and other 0 (00100000). This mask is a bit representation of the space character (‘ ‘).
  
**Example:**
```java
ch   = 'A' (01000001)
mask = ' ' (00100000)
ch | mask = ‘a’ (01100001)
```
### Lower case English alphabet to upper case: `ch &= '_'
`ch &= '_'; // AND with underscore`

**Logic:** The bit representation of upper case and lower case English alphabets are:
A -> 01000001 a -> 01100001
B -> 01000010 b -> 01100010
C -> 01000011 c -> 01100011
. .
. .
Z -> 01011010 z -> 01111010

$$'A'\xleftharpoondown[clear\ \color{red}5-th\ \color{black}bit]\ 'a'$$

Clear the 5th bit of lower case characters, it will be converted into upper case character. Make a mask having 5th bit 0 and other 1 (10111111). This mask is a bit representation of the underscore character (`_`). AND the mask with the character.
  
**Example:**
```java
ch = ‘a’ (01100001)
mask = ‘_’ (11011111)
ch & mask = ‘A’ (01000001)
```

$$'[A-Z]'\xrightleftharpoons[UC\ =\ lc\ \color{red}\&\ '\_']{lc\ = \ UC\ \color{red}\ |\ '\ ' }\ '[a-z]'$$
### Efficient modulo x % 2^k
$x \bmod 2^k = x \& ((1\ll k) -1)$
$10 \bmod 2^2 =  10 \bmod 4 = 2$
$10 \bmod 2^2 = 10 \& ((1\ll 2) -1) =10 \& 3 = 1010\&0011 = 0010 =2$ 
### Masked Copy 
Goal :  Copy bits from B into A where M = 1
Ans : `A = (B & M) | (A & ~M`)

![[Pasted image 20260120050354.png]]

### Swapping Bits
Goal : given a number x swap the bits positioned at A & B
Ans : use an intermediate var P 
If P = 0, the bits were equal . No swap required. XOR with P will do nothing. 
```
P = (X>>A) ^ (X>>B) & 1 // if P =0 no need to swap
X ^= (P << A)
X ^= (P << B)
```
![[Pasted image 20260120050739.png]]


Before going below let's make & appreciate following observations about components to be used later :  
- `x`: The original number.
- `~x`: The complement (inverts 0s to 1s).
- `x + 1`: The number with the "rightmost zero" flipped to 1 and trailing ones cleared.
- `x - 1`: The number with the "rightmost one" flipped to 1 and trailing zeros cleared.

### Clear rightmost 1 : (x-1) trick
x & (x-1)

![[Pasted image 20260120015802.png]]

![[Pasted image 20260120064135.png]]

This helps in : 
- Finding number of set  bits in a binary representation of a number
- Finding if number is power of 2.

How to find unset bits in a number ? 
```java
int countValues(int n) {
    // unset_bits keeps track of count of un-set
    // bits in binary representation of n
    int unset_bits=0;
    while (n)
    {
        if ((n & 1) == 0)
            unset_bits++;
        n=n>>1;
    }
    return unset_bits;
}
// NOTE: 
// 1 << unset_bits i.e. 2 ^ unset_bits i.e. pow(2,count of zero bits) 
// = 
// Count of numbers (x) smaller than or equal to n such that n+x = n^x:
```

#### Convert Trailing 0's to 1
x | (x-1)

### Set  rightmost 0 : (x+1) trick

`x+1` : (`~`) flips entire half of binary representation after (including) the **first 0** , or
- sets off all trailing 1's and sets on first 0, or
- sets first unset LSB (least significant bit)

**The Rule of Inversion** The operation `x + 1` **inverts** all bits from the Least Significant Bit (LSB) up to and including the **rightmost zero**.

x     =  1 0 0 1 0 <mark style="background-color: #fff88f; color: black">0 1 1 1</mark>
x+1 = 1 0 0 1 0 <mark style="background-color: #fff88f; color: black">1 0 0 0</mark>

x     =  1 0 0 1 0 <mark style="background-color: #fff88f; color: black">0</mark> 1 1 1
x+1 = 1 0 0 1 0 <mark style="background-color: #fff88f; color: black">1</mark> 0 0 0
|
`------------------`
       1 0 0 1 0 <mark style="background-color: #fff88f; color: black">1</mark> **1 1 1**

x     =  1 0 0 1 0 <mark style="background-color: #fff88f; color: black">0</mark> 1 1 1
x+1 = 1 0 0 1 0 <mark style="background-color: #fff88f; color: black">1</mark> 0 0 0
&
`--------------------`
       1 0 0 1 0 <mark style="background-color: #fff88f; color: black">0</mark> **0 0 0**

If the binary representation of $x$ ends in a string of $k$ ones (i.e., $\dots 011\dots1$), then `x + 1`:
1. **Sets** the rightmost `0` to `1`.
2. **Clears** all $k$ trailing `1`s to `0`.
Visual Example 
$$\begin{aligned} x &= \dots \mathbf{0}\color{red}{111}\\ 
x + 1 &= \dots \mathbf{1}\color{red}{000} \end{aligned}$$
![[Pasted image 20260119210717.png]]

`x | (x+1)` = sets the **rightmost zero** in the binary representation of `x` to **1**.
`x & (x+1)` = retains all bits to the left of first 0, every thing on right is reset to 0
`x & (x-1)` = sets the **rightmost one** in the binary representation of x to 0. 
- is given number power of 2?
- count set bits in a binary representation of a number. 

Corollaries 
- **Identify the Change:** The bits that change between `x` and `x+1` are exactly the suffix `011...1`.
- **Clear Trailing Ones:** The expression `x & (x + 1)` clears the continuous block of trailing ones (and leaves the newly set bit as `0`).
    - _Example:_ `0111` (7) & `1000` (8) = `0000`.
    - _Example:_ `1011` (11) & `1100` (12) = `1000`.


### Extract Rightmost Set Bit : (x & -x)

**Finding the LSB:**  You extract the LSB using the bitwise operation `i & -i`.

Application : A **Binary Indexed Tree (BIT)**, also known as a **Fenwick Tree**, is a data structure that allows efficient updates and prefix sum queries on an array of numbers.

![[Pasted image 20260120011253.png]]

### Comparison of Bit Hacks

| **Operation**      | **Logic**                    | **Effect**                                                                    |
| ------------------ | ---------------------------- | ----------------------------------------------------------------------------- |
| $x \& (x-1)$       | Turn **OFF** rightmost **1** | Used to count set bits (Brian Kernighan’s Algorithm) or check for Power of 2. |
| $x \textbar (x+1)$ | Turn **ON** rightmost **0**  | $x\vert(x+1)$                                                                 |
|                    |                              |                                                                               |
### The 4 Pillars of Low-Level Bit Manipulation

| **Expression**                        | **Name / Logic**     | **Effect on Binary String**                                                                | **Typical Use Case**                                                                                               |
| ------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **`x & (x - 1)`**                     | **Clear LSB**        | Sets the **rightmost `1`** to `0`.                                                         | Counting set bits (popcount), checking if $x$ is Power of 2.                                                       |
| **`x & -x`**                          | **Isolate LSB**      | Sets **all bits to `0`** _except_ the rightmost `1`. i.e. Extracting the rightmost set bit | Fenwick Tree (BIT) updates, getting the specific value of the LSB.                                                 |
| $x \textbar (x+1)$                    | **Fill Lowest 0**    | Sets the **rightmost `0`** to `1`.                                                         | [How to get position of right most set Bit - Aticleworld](https://aticleworld.com/position-of-right-most-set-bit/) |
| **`~x & (x + 1)`** = **`~x & -(~x)`** | **Isolate Lowest 0** | Sets **all bits to `0`** _except_ the rightmost `0` (which becomes `1`).                   | Finding the first available "slot" or "zero bit" in a resource mask.                                               |
![[BitManipulation03.png]]

### Five most important formulas : 

x & (x-1) : turns OFF rightmost 1 : rm 1 ->0
x | (x+1)   : turns ON rightmost 0 : rm 0 ->1

x | (x-1)   : sets trailing 0's -> 1
x & (x+1) : clears trailing 1's ->0

x & -x : position of lowest set bit 

### Summary Table

| **Expression** | **Correct Description**       | **Correct Logic for Your Goal** |
| -------------- | ----------------------------- | ------------------------------- |
| `x & (x-1)`    | turns OFF rightmost 1         | rm 1 ->0                        |
| `x \| (x+1)    | turns ON rightmost 0          | rm 0 ->1                        |
| `x \| (x-1)    | sets trailing 0's -> 1        |                                 |
| `x & (x+1)`    | Clears **all** trailing `1`s. | 01010111 -> 01010000            |
### 1. Single Bit Modifications (Precision Targets)

|**Operation**|**Formula**|**Effect**|**Example (x=10101100)**|**Result**|
|---|---|---|---|---|
|**Remove LSB**|`x & (x-1)`|Turns OFF rightmost `1`|$10101\mathbf{1}00 \to 10101\mathbf{0}00$|**Standard** (Kernighan's Algo)|
|**Fill LSZ**|`x|(x+1)`|Turns ON rightmost `0`|$1010110\mathbf{0} \to 1010110\mathbf{1}$|
### 2. Block Modifications (Trailing Sequences)

|**Operation**|**Formula**|**Effect**|**Example**|**Logic**|
|---|---|---|---|---|
|**Fill Trailing Zeros**|`x|(x-1)`|Sets all trailing `0`s to `1`|$1\mathbf{00} \to 1\mathbf{11}$|
|**Clear Trailing Ones**|`x & (x+1)`|Clears all trailing `1`s to `0`|$0\mathbf{11} \to 0\mathbf{00}$|Masks the carry ripple from `011` to `100`.|
### Position of Rightmost set Bit
c program

The bitwise operation `(n & ~(n-1))` isolates the rightmost set bit in “n”. The bitwise AND operation between `n` and `∼(n−1)` return only the rightmost set bit of n by turning off all other bits. Now the last steps to use **log2( n & ~(n-1) )** to get the position of rightmost set bit.

```c
#include <stdio.h>
#include <math.h>
int getPositionRightMostSetBit(unsigned int n)
{
	unsigned int hasOnlyRightSetBit = (n & ~(n-1));
	int posOfRightMostSetBit = (int)log2(hasOnlyRightSetBit);
	return posOfRightMostSetBit;
}
```

Java doesn't have `Math.log2`
```Java
// returns 0-indexed position

public int getPositionRightMostSetBit(int n) {
    if (n == 0) return -1; // Safety check: log(0) is undefined
    
    // In Java, ~ (n - 1) is effectively -n (Two's Complement)
    int hasOnlyRightSetBit = n & -n;
    
    // Java has Math.log (base e) and Math.log10, but no Math.log2
    // We use the change of base formula: log2(x) = log(x) / log(2)
    return (int) (Math.log(hasOnlyRightSetBit) / Math.log(2));
}

OR
// Java's `Integer` class has a specific method for this that maps directly to CPU instructions (like `TZCNT` on x86) where possible.
public int getPositionRightMostSetBit(int n) {
	// Returns the number of zero bits following the lowest-order one-bit
	// Equivalent to log2(n & -n)
    return Integer.numberOfTrailingZeros(n);
}
//-----------

```

To isolate lowest 0
```java
public int getPositionRightMostUnsetBit(int n) {
    if (n == -1) return -1; // Handle case where all bits are set (no 0s)
    
    // Step 1: Invert n so 0s become 1s
    int nInverted = ~n; 
    
    // Step 2: Isolate the rightmost 1 (which was the rightmost 0)
    // Using the logic: x & ~(x-1)
    int isolatedBit = nInverted & ~(nInverted - 1);
    
    // Step 3: Get position using log2
    return (int) (Math.log(isolatedBit) / Math.log(2));
}

OR 
public int getPositionRightMostUnsetBit(int n) {
    if (n == -1) return -1; // All bits are set (no unset bits)
    
    // Your logic: Isolate the rightmost unset bit directly
    int isolatedBit = ~n & (n + 1);
    
    // Get the position
    return (int) (Math.log(isolatedBit) / Math.log(2));
}

OR
public int getPositionRightMostUnsetBit(int n) {
	// Returns the number of zero bits following the lowest-order one-bit
	// Equivalent to log2(n & -n)
    return Integer.numberOfTrailingZeros(~n);
}
```

### Property 
**P1**
*if*
no of set bits in A = X
no of set bits in B = Y
no of set bits in A^B = Z
*then* 
Z is even if X+Y is even
Z is odd if X+Y is odd
**P2**
situation in code
```
if (X == A)
	X = B
else if (X == B )
	X = A
```
in such situation use concise version
```
`X = A ^ B ^ X`
```

**P3**
`A + B = (A^B) + 2(A & B) ` = `A + B = (A^B) + (A & B)<<1`
`A + B = (A|B) +  (A & B)`




### Kernighan's Population Count
Count the number of 1's in a binary number (POPCNT is an x86 instruction)
`x = x & (x-1)`

Menta Image: **1s in a num** = **num of times rightmost 1 needs to be unset** 

```java
int num = 29; // Binary : 11101
int count = 0;
while(num > 0) {
	num &= (num-1)
	count++; 
}
return count;
```

### Counting Bit Islands
 
![[Pasted image 20260120051747.png]]
`I = (x&1) + PopCnt( x^(x>>1) ) / 2`

![[Pasted image 20260120052012.png]]

### BSF Bit Scan Forward

![[Pasted image 20260120052340.png]]

### Next lexicographic permutation
Goal : Given a bit string, generate the next permutation above with the same number of 1's
```
t = x|(x-1)
x = (t+1) | ( (~t & -(~t)) -1) >> ((BSF(x) + 1) )
```

### ONE LINERS
- `-(~n) = n++` : - `~n` results in `-(n+1)` (e.g., if `n=6`, `~6` is `-7`).
	- - Applying negation again, `-(~n)`, results in `-(-(n+1))`, which simplifies to `n+1` (e.g., `-(-7)` is `7`).
- In Java, `~ (n - 1)` is effectively `-n` (Two's Complement)
- 

Practice Questions : 
- [Bit Manipulation Problem List - LeetCode](https://leetcode.com/problem-list/bit-manipulation/)
- 

Standard Questions 
* a +b without using + sign : [Add Two Numbers Without The "+" Sign (Bit Shifting Basics) - YouTube](https://www.youtube.com/watch?v=qq64FrA2UXQ)
* a / b without using / sign [L9. Divide Two Integers without using Multiplication and Division Operators \| Bit Manipulation - YouTube](https://www.youtube.com/watch?v=pBD4B1tzgVc)
* 

```java
public int getSum(int a, int b) {
        while(b != 0 ) { 
            int carry_for_next_bit = (a&b)<<1;
            a = a^b;
            b = carry_for_next_bit; 
        }
        return a;
    }
    
static int add(int a, int b) {
    int carry;
    // Loop until there is no more carry
    while (b != 0) {
        // Calculate the carry bits
        carry = (a & b) << 1;
        // Calculate the sum without carry and store in 'a'
        a = a ^ b;
        // Update 'b' to hold the carry for the next iteration
        b = carry;
    }
    // 'a' holds the final sum when 'b' (carry) is 0
    return a;
}
```


----
BIBLIOGRAPHY

[Unlocking Bit Manipulation Secrets! - YouTube](https://www.youtube.com/watch?v=5kvgF0xN6SM)
[How to get position of right most set Bit - Aticleworld](https://aticleworld.com/position-of-right-most-set-bit/)
[Learn these 10 Bitwise Tricks Or Regret Later \| Competitive Programming Tricks Part 2 - YouTube](https://www.youtube.com/watch?v=LGrE0siZ-ZA)
[L1. Introduction to Bit Manipulation \| 1's 2's Compliment \| Bit Operators - YouTube](https://www.youtube.com/watch?v=qQd-ViW7bfk&list=PLgUwDviBIf0rnqh8QsJaHyIX7KUiaPUv7)
[Bit Hacks from Beginner to Advanced - 11 Amazing Bit Twiddling Techniques - YouTube](https://www.youtube.com/watch?v=ZRNO-ewsNcQ)
[Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html)
[How to Manipulate Bits in C and C++ \| HackerNoon](https://hackernoon.com/bit-manipulation-in-c-and-c-1cs2bux)
[Code 360 by Coding Ninjas](https://www.naukri.com/code360/library/bit-manipulation-for-competitive-programming)
[Bit manipulation - Algorithms for Competitive Programming](https://cp-algorithms.com/algebra/bit-manipulation.html)

| Necessary Condition |
| ------------------- |
# 1's Complement vs 2's Complement Table

**Format:** Binary representation (Decimal value) for 8-bit signed integers

---
(8-bit representation)

| 1's Complement   | Number | 2's Complement   | Notes                                             |
| ---------------- | ------ | ---------------- | ------------------------------------------------- |
| `00001010` (10)  | 10     | `00001010` (10)  |                                                   |
| `00001001` (9)   | 9      | `00001001` (9)   |                                                   |
| `00001000` (8)   | 8      | `00001000` (8)   |                                                   |
| `00000111` (7)   | 7      | `00000111` (7)   |                                                   |
| `00000110` (6)   | 6      | `00000110` (6)   |                                                   |
| `00000101` (5)   | 5      | `00000101` (5)   |                                                   |
| `00000100` (4)   | 4      | `00000100` (4)   |                                                   |
| `00000011` (3)   | 3      | `00000011` (3)   |                                                   |
| `00000010` (2)   | 2      | `00000010` (2)   |                                                   |
| `00000001` (1)   | 1      | `00000001` (1)   | ⚠️ Zero has TWO representations in 1's complement |
| `11111111` (-0)  | 0      | `00000000` (0)   | ⚠️ Positive zero in 2's complement                |
| `00000000` (-1)  | 0      |                  |                                                   |
| `11111110` (-2)  | -1     | `11111111` (-1)  |                                                   |
| `11111101` (-3)  | -2     | `11111110` (-2)  |                                                   |
| `11111100` (-4)  | -3     | `11111101` (-3)  |                                                   |
| `11111011` (-5)  | -4     | `11111100` (-4)  |                                                   |
| `11111010` (-6)  | -5     | `11111011` (-5)  | 1's comp of (-5) = 2's comp of (-6)               |
| `11111001` (-7)  | -6     | `11111010` (-6)  |                                                   |
| `11111000` (-8)  | -7     | `11111001` (-7)  |                                                   |
| `11110111` (-9)  | -8     | `11111000` (-8)  |                                                   |
| `11110110` (-10) | -9     | `11110111` (-9)  |                                                   |
| `11110101` (-11) | -10    | `11110110` (-10) |                                                   |

## Key Observations

### 1. One's Complement (`~n`)
- **Rule:** Flip all bits (0→1, 1→0)
- **Issue:** Zero has two representations (`00000000` and `11111111`)
- **Range (8-bit):** -127 to +127
- **Formula:** `~n = -(n) - 1`

### 2. Two's Complement (Modern Standard, Used in Java)
- **Rule:** Flip all bits, then add 1
- **Advantage:** Only one zero representation
- **Range (8-bit):** -128 to +127
- **Formula:** `~n + 1 = -n`
- **Relationship:** `2's complement = 1's complement + 1`

### 3. Interesting Patterns
**For positive numbers (0 to 10):**
- In 1's complement: They appear as negative values (flip of positive)
- In 2's complement: They map to one less negative (since 2's = 1's + 1)

**For negative numbers (-1 to -10):**
- In 1's complement: They appear as positive values (flip of negative)
- In 2's complement: They represent the actual negative value

**Special case: Zero (-1)**
- 1's complement of -1 is `00000000` (positive zero)
- 2's complement of -1 is `00000001` (which is 1)

---

## How to Read the Table

**Example: For number -5**

1. **1's Complement of -5:**
    
    - Take binary of positive 5: `00000101`
    - Flip all bits: `11111010` = 250 (as unsigned) or -6 (if interpreted as 1's complement)
    - Result: `11111010` (9 in decimal when flipped back)
2. **2's Complement of -5 (as stored in Java):**
    
    - Take binary of positive 5: `00000101`
    - Flip all bits: `11111010`
    - Add 1: `11111011` = 251 (as unsigned) or -5 (if interpreted as 2's complement)
    - Result: `11111011` (actual representation of -5 in Java)

---

## Java Code to Generate This Table

```java
public static void main(String[] args) {
    System.out.println("1's Complement | Number | 2's Complement");
    System.out.println("==========================================");
    
    for (int num = -10; num <= 10; num++) {
        byte b = (byte) num;
        
        // 1's complement: flip all bits
        byte onesComp = (byte) ~b;
        
        // 2's complement: flip and add 1
        byte twosComp = (byte) (~b + 1);
        
        String ones = String.format("%8s", Integer.toBinaryString(onesComp & 0xFF))
                            .replace(' ', '0');
        String twos = String.format("%8s", Integer.toBinaryString(twosComp & 0xFF))
                            .replace(' ', '0');
        
        System.out.printf("%s (%3d) | %3d | %s (%3d)%n", 
            ones, onesComp, num, twos, twosComp);
    }
}
```

---

## Why 2's Complement?

|Aspect|1's Complement|2's Complement|
|---|---|---|
|Zero Representation|Two (00000000 and 11111111)|One (00000000)|
|Range (8-bit)|-127 to +127|-128 to +127|
|Arithmetic|End-around carry needed|Natural addition works|
|Used in Practice|Historical only|All modern systems|
|End-Around Carry|Required in addition|Not needed|

**Verdict:** 2's complement is the standard because it simplifies arithmetic and has no duplicate zero.