
### GCD(a, b)

Euclidean Algorithm makes use of the following properties:
- GCD(A,0) = A
- GCD(0,B) = B
- **If A = B⋅Q + R and B≠0 then GCD(A,B) = GCD(B,R)** where Q is an integer, R is an integer between 0 and B-1

Here are the primary approaches to finding the GCD (Greatest Common Divisor), moving from the naive to the optimal.
### 1. The Subtraction Method (Original Euclidean)1
This is the ancient way of finding GCD. The logic is simple: if you have two sticks of different lengths, you keep cutting the longer stick by the length of the shorter stick until they are equal.
- **Logic:** Repeatedly subtract the smaller number from the larger number until both become equal.2

```Java
// Subtraction Approach
int gcdSubtraction(int a, int b) {
    // Handle 0 cases explicitly if needed
    if (a == 0) return b;
    if (b == 0) return a;

    while (a != b) {
        if (a > b)
            a = a - b;
        else
            b = b - a;
    }
    return a;
}
```

### 2. The Remainder Method (Optimized Euclidean)
This is the modern standard. Instead of subtracting repeatedly, we use the modulo operator (`%`) to find the remainder immediately.
- **Logic:** $GCD(a, b) = GCD(b, a \% b)$. The remainder replaces the larger number in one step.

```Java
// Remainder Approach
int gcdRemainder(int a, int b) {
    if (b == 0) return a;
    return gcdRemainder(b, a % b);
}
```

---
### Comparison: Subtraction vs. Remainder
The **Remainder approach is significantly faster**.  Here is why.
#### The "Why" (The Step Difference)
Imagine calculating **GCD(1000, 1)**.
Subtraction Method:
It subtracts 1 from 1000 repeatedly.
- `1000 - 1 = 999`
- `999 - 1 = 998`
- ... (999 steps later) ...
- `1 - 1 = 0`.
- **Total Steps:** **1000** iterations.

Remainder Method:
It performs division immediately.
- `1000 % 1 = 0`.
- Next call: `gcd(1, 0)`.
- **Total Steps:** **1** iteration.
#### recursive : Optimized Euclidean Algorithm

```java
gcd(a, b) : assume b < a
---------------------------------
 a =  b.q1 + r1 
 b = r1.q2 + r2
r1 = r2.q3 + r3
r2 = r3.q4 + r4
..
...
....
r(n-3) = r(n-2).q(n-1) + r(n-1)
r(n-2) = r(n-1).q(n)   + r(n) = 0 // STOP
```



```Java
int gcd(int a, int b) {
	int divisor = Math.min(a,b); // b
	int dividend = (a+b) - divisor;  //a
	int remainder = dividend % divisor;
	while(remainder != 0) {
		 dividend = remainder;
		 divisor = remainder;
		 remainder =  dividend % divisor;
	}
	return remainder;
}

int gcd(int dividend, int divisor) {
	if (divisor == 0)
		return dividend;
	return gcd(divisor, dividend % divisor);
}

int gcd(int a, int b) {
	if (b == 0)           // Check Slot 2
		return a;
	return gcd(b, a % b); // Remainder to Slot 2
}



//Scenario A:
int gcd(int a, int b) {
	if (b == 0)           // Check Slot 2
		return a;
	return gcd(b, a % b); // Remainder to Slot 2
	//"If `b` is zero, return `a`. Else, put the remainder in `b`'s spot."
}

//Scenario B:
int gcd(int a, int b) {
	if (a == 0)
		return b;
	return gcd(b % a, a);
}

int gcd(int d, int r) {
	if (r == 0)
		return d;
	return gcd(r, d % r);
}
```

### The Golden Rule: Match the Slots
**The position where you check for `0` is the SAME position where you must put the `remainder`.**
#### Scenario A: The Standard (Checking 2nd Param)
- **Check:** `if (b == 0)` $\rightarrow$ You are checking **Slot 2**.
- **Recurse:** `gcd(b, a % b)` $\rightarrow$ You pass remainder into **Slot 2**.
- **Return:** `return a` $\rightarrow$ You return **Slot 1** (the survivor).
#### Scenario B: The Swapped (Checking 1st Param)
- **Check:** `if (a == 0)` $\rightarrow$ You are checking **Slot 1**.
- **Recurse:** `gcd(b % a, a)` $\rightarrow$ You pass remainder into **Slot 1**.
- **Return:** `return b` $\rightarrow$ You return **Slot 2** (the survivor)



#### iterative  
##### Brute Force 
go down from min(a,b)
```Java
int gcd(int a, int b)
    {
        int result = Math.min(a, b); // Find Minimum of a and b
        while (result > 0) {
            if (a % result == 0 && b % result == 0) {
                break;
            }
            result--;
        }
        return result;
    }
```

#####  Euclidean Algorithm using Subtraction 
```
gcd(a, b):  
    if a = b:  
        return a  
    if a > b:  
        return gcd(a - b, b)  
    else:  
        return gcd(a, b - a)
```

```Java
int gcd(int a, int b) {
        // If any number is 0 return the other
        if (a == 0 || b == 0)
            return a+b;
        // Base case
        if (a == b)
            return a;
        // a is greater
        if (a > b)
            return gcd(a - b, b);
        return gcd(a, b - a);
    }
    
int getGCD(int a, int b) {
        while (a > 0 && b > 0) {
            if (a > b) {
                a = a % b;
            } else {
                b = b % a;
            }
        }
        return (a == 0) ? b : a;
    }
```
### gcd_Arr(int[] arr)
The GCD of three or more numbers equals the product of the prime factors common to all the numbers, but it can also be calculated by **repeatedly taking the GCDs of pairs of numbers**.
```
gcd(a, b, c) = gcd(a, gcd(b, c))  
			 = gcd(gcd(a, b), c)  
			 = gcd(gcd(a, c), b)
```


```Java
int findArrayGCD(int[] array) {
        int res = array[0];
        for (int i = 1; i < array.length; i++) {
            res = gcd(array[i], res);

            // If res becomes 1 at any iteration then it remains 1
            if (res == 1)
                return 1;
        }
        return res;
    }
```

### LCM (a,b)
https://www.geeksforgeeks.org/maths/lcm-least-common-multiple/
``` 
a x b = LCM(a, b) * GCD (a, b)  
LCM(a, b) = (a x b) / GCD(a, b)
```

![[lcm-gcd.png]]

```Java
    int gcd(int a, int b) {
        return (b == 0) ? a : gcd(b, a % b);
    }

    int lcm(int a, int b) {
        return (a / gcd(a, b)) * b;
    }
```

### lcm_Array(int[] arr)

![[lcm_arr.png]]

```
ans = LCM(ans, arr[i]) = ans x arr[i] / gcd(ans, arr[i])
```

#### Rolling LCM 

```Java
int lcm_of_array(ArrayList<Integer> arr)
    {
        int lcm = arr.get(0);
        for (int i = 1; i < arr.size(); i++) {
            lcm = (lcm * arr.get(i)) / gcd(lcm, arr.get(i));
        }
        return lcm;
    }

```

```Java
int lcm(List<Integer> numbers)
	{
		return numbers.stream().reduce(
			1, (x, y) -> (x * y) / gcd(x, y));
	}
```


