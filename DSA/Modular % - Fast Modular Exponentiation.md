Evaluating 𝑥𝑛 (mod𝑝) means finding the remainder when xnx to the n-th power 𝑥𝑛 is divided by pp 𝑝 . If pp 𝑝 is a prime number and xx 𝑥 is not a multiple of 𝑝 , you can dramatically speed up the calculation using **Fermat’s Little Theorem** or the **Binary Exponentiation** algorithm

---
# Method 1: Using Fermat's Little Theorem (FLT) 

Fermat's Little Theorem states that if 𝑝 is prime and 𝑥 is not divisible by 𝑝 , then:   𝑥𝑝−1 ≡1 (mod𝑝)


==

Method 1: Using Fermat's Little Theorem (FLT) 

Fermat's Little Theorem states that if 𝑝 is prime and 𝑥 is not divisible by 𝑝 
, then:   𝑥𝑝−1 ≡1 (mod𝑝)  

This allows you to simplify the exponent 𝑛 : 

1. Divide 𝑛 by 𝑝−1 to find the remainder: 𝑟 =𝑛 (mod𝑝−1).
2. Simplify the problem to solving 𝑥𝑟 (mod𝑝). 

**Example:** Calculate $2^{100} (mod\ 11)$ . 
1. Here, 𝑝 =11, which is prime.
2. Find the remainder of the exponent:  100 (mod10) =0.
3. Simplify: 20(mod11)  =1.

# Method 2: Binary Exponentiation (Fast Modular Exponentiation) 

If you need to compute 𝑥𝑛 (mod𝑝) when 𝑛 is very large and you cannot use FLT (or if 𝑝 is composite), you can calculate it efficiently using the binary method. It reduces the number of multiplications from 𝑂(𝑛) to 𝑂(log 𝑛). 

Steps for hand-calculating: 

1. Convert the exponent 𝑛 to its binary form. 
2. Start with a result of 11 - 1 . 
3. For each bit in the binary representation (reading from right to left):
    - If the current bit is 11    1, multiply your result by  𝑥  (and mod by 𝑝). 
    - Square the base 𝑥 (and mod by 𝑝) for the next step. 

**Example:** Calculate 35 (mod7). 

1. Convert exponent 5 to binary:  5 =(101)2    (which is 1
     )⋅4
     )+0
     )⋅2
     )+1
     )⋅1
    ). 
1. Evaluate from right to left:
    - First bit is  1: Multiply 1 by 31 (mod7) =3  . Result = 3.
    - Second bit is 0 : Square the base 3^2  (mod7)  =2.
    - Third bit is 1  : Multiply previous result by base 3⋅2 (mod7) =6. 
2. Final Answer: 6. 

# Computing 𝑥 (mod 𝑝) First 

To keep your numbers small and manageable in any calculation involving
xn to the n-th power 𝑥𝑛 , you can immediately reduce 𝑥 by 𝑝. Replace 𝑥 with 𝑥 (mod 𝑝) at the very beginning of your problem without changing the final result.


-----

---
tags:
  - math
  - cryptography
  - algorithms
---

# Modular Exponentiation $(x^n \bmod p)$

The mathematical expression $(x^n \bmod p)$ represents **modular exponentiation**, which finds the remainder when a base $x$ is raised to the power of $n$ and divided by a modulus $p$. 

To compute this value efficiently, the standard method is the **Binary Exponentiation (Square-and-Multiply) algorithm**, which runs in $(O(log\ n) )$ time complexity. If $p$ is a prime number, you can simplify the calculation using **Fermat's Little Theorem**.

---

### 1. Simplify with Fermat's Little Theorem
When \(p\) is a **prime number** and \(x\) is not divisible by $p$, Fermat's Little Theorem states that:
\[x^{p-1} \equiv 1 \pmod p\]

This allows you to reduce a large exponent \(n\) by taking it modulo \((p-1)\) before computing the power:
\[x^n \bmod p = x^{n \bmod (p-1)} \bmod p\]

* **Example:** Compute \(3^{100} \bmod 7\)
* Since \(7\) is prime, \(p-1 = 6\).
* Reduce the exponent: \(100 \bmod 6 = 4\).
* Therefore: \(3^{100} \bmod 7 = 3^4 \bmod 7 = 81 \bmod 7 = 4\).

---

### 2. Fast Computation (Square-and-Multiply)
For large values of \(n\) where manual multiplication is impossible, the binary exponentiation technique processes the exponent bit by bit. 

#### Procedural Algorithm (Iterative)
1. **Initialize** the result variable: `result = 1`.
2. **Update the base** by keeping it within the modulus: `x = x % p`.
3. **Loop** while the exponent \(n > 0\):
   * If \(n\) is odd, multiply the current base to the result: `result = (result * x) % p`.
   * Square the base for the next iteration: `x = (x * x) % p`.
   * Divide the exponent by 2 (bit-shift right): `n = n / 2`.
4. **Return** the final `result`.

---

### 3. Code Implementations

#### C++
```cpp
long long powerMod(long long x, long long n, long long p) {
    long long result = 1;
    x = x % p;
    while (n > 0) {
        if (n % 2 == 1) {
            result = (result * x) % p;
        }
        x = (x * x) % p;
        n = n / 2;
    }
    return result;
}
```
#### Java 

```java 
import java.math.BigInteger; 
public class ModularExponentiation { 
	// Standard iterative method for primitive types (prevents overflow using long) 
	public static long powerMod(long x, long n, long p) { 
		long result = 1; 
		x = x % p; 
		while (n > 0) { 
			if ((n & 1) == 1) {// Checks if n is odd 
				result = (result * x) % p; 
			} 
			x = (x * x) % p; 
			n >>= 1; // Divides n by 2 using right shift 
		} 
		return result; 
	} 
	
	// Built-in BigInteger method for arbitrarily large numbers
	public static BigInteger powerModBig(String base, String exp, String mod) { 
		BigInteger b = new BigInteger(base); 
		BigInteger e = new BigInteger(exp); 
		BigInteger m = new BigInteger(mod); 
		return b.modPow(e, m); 
		} 
}
```