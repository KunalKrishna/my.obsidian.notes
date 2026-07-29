
BinarySearch
`findPos (int[] array, int target)` : `int` (the pos of target or -1)
    - array: all unique elements -  [704. Binary Search](https://leetcode.com/problems/binary-search/)
`findPos (int[] array, int target)` : `int []`  (_a list of the target indices_)
    - array: may contain duplicates :  - [2089. Find Target Indices After Sorting Array](https://leetcode.com/problems/find-target-indices-after-sorting-array/submissions/2006147882)
	    - [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)
`findMin (int[] rotated_array)` : int 
    - array: all unique elements : LC 153 (Medium)
    - array: may contain duplicates  : LC 154 (hard)

  

> [!NOTE] Binary Search Applicablity Condition

> At each iteration is there any consistent criteria for partition decision?


## LC 704 :

```java
class Solution {
    public int search(int[] A, int T) {
        int l = 0, r= A.length-1;
        while(l <= r) {
            int mid = (l+r)/2;
            if(A[mid] == T) return mid;
            if(A[mid] >  T) r = mid - 1;  // (A[mid] >  T)
            else            l = mid + 1;  // (A[mid] <  T)
        }
        return -1;
    }
}
```


------
# search()

Two choice for `while` loop condition : 
1. `while(l < r)` : doesn't enter while loop when `l=r` i.e. single element (sub) array
2. `while(l <= r)` : enters while loop when `l=r` i.e. even for a single element (sub) array

Two ways of calculating `mid` : depending on *check* is made before or after updating `mid`
2. **PRE** : check `A[mid]` --> update `mid` for next while loop iteration
3. **POST** :  update `mid` for current while loop iteration --> check `A[mid]` . It offers following benefits : 
	1. `mid` can be initialized with any value in range `[0,n-1]`
	2. in case of single element (sub) array the `mid` can be checked and returned.

Different formulae for `mid` : `[1,2,3,...,n || n+1,n+2,n+3,...,2n]`
1. `mid = (l + r) / 2 `: LEFT-biased (rounds DOWN using floor)
	1. Example: `[1,2,3,4] → l=0,r=3 → mid=(0+3)/2=1` (points LEFT of center)
	2. `mid = l + (r - l) / 2` : prevents overflow 
2. `mid = (l+r+1) / 2` : RIGHT-biased (rounds UP, ceil-like behavior)
	1. Example: `[1,2,3,4] → l=0,r=3 → mid=(0+3+1)/2=2` (points RIGHT of center)
	2. `mid = l + (r - l + 1) / 2` :  prevents overflow 

Java interpreter optimizes div by 2 by automatically translating `numerator/2 `as `numerator >> 1`.

We shall later see following correspondence : 
- PRE - LEFT-biased mid
- POST - RIGHT-biased mid
## Check every element inside `while(l<=r)`
even for single element (sub) array it enters while loop, hence any required pos must be found inside while loop.
### using LEFT-biased `mid`
```java
public int search(int[] A, int T) {
	int l = 0, r= A.length-1;
	
	// do I need to "see" one elt if A.size = 1 ? Yes --> <=
	// in other words is there any default ans  ? No  --> <=  
	while(l <= r) { 
		int mid = (l+r)/2; // NOTE : it has LEFT-bias 

		if(A[mid] == T) return mid;

		if(A[mid] >  T) r = mid - 1;  // (A[mid] > T)
		else            l = mid + 1;  // (A[mid] < T)
	}

	return -1; // NOT FOUND !
}
```
### using right-biased `mid`
```java
    public int search(int[] A, int T) {
        int l = 0, r = A.length-1;
        
        while(l <= r) { 
            int mid = (l+r+1)/2; // NOTE : mid is RIGHT-based pos 

            if(A[mid] == T) return mid;

            if(A[mid] >  T) r = mid - 1;  // (A[mid] > T) --> search left 
            else            l = mid + 1;  // (A[mid] < T) --> search right
        }

        return -1; // NOT FOUND !
    }
```
## Check (also) outside `while(l< r)` 
for single element (sub) array it does not enter the while loop, hence the required pos can also be returned from outside the while loop. 
### using left biased mid : return `l` (for 1 elt)
```java
class Solution {
    public int search(int[] A, int T) {
        int l = 0, r = A.length-1;
        
        // inside while loop : 
        // do I need to "see" one elt if A.size = 1 ?  "No" --> use <
        // or
		// is there any default ans? "Yes" --> use < 
		//      assume by default A.size = 1, hence outside while() checks A[l] == T ?
        while(l < r) { 
            int mid = (l+r)/2; // PRE

            if(A[mid] == T) return mid;

            if(A[mid] >  T) r = mid - 1;  // (A[mid] > T) --> search left 
            else            l = mid + 1;  // (A[mid] < T) --> search right
        }

        // for a single elt sub-array : `while(l < r)` skips (A[mid] == T) check 
        // but `l` contains index of that single elt subarry because mid is LEFT-biased
        // return (A[mid] == T)? mid : -1; doesn't work because `mid` lags, it reflects mid of last iteration
        return (A[l] == T)? l : -1; 
    }
}
```
but we can modify above code to ensure every variant returns through `mid` by increasing the scope of var mid from while to method level. 
#### return `mid` (for 1 elt) : POST `mid` calc
```java
public int search(int[] A, int T) {
	int l = 0, r = A.length-1;
	
	int mid = 0;  // any mid ∈ [l, r]
	while(l < r) { 

		if(A[mid] == T) return mid;

		if(A[mid] >  T) r = mid - 1;  // (A[mid] > T) --> search left 
		else            l = mid + 1;  // (A[mid] < T) --> search right

		mid = (l+r) / 2; // POST : update before next iteration
	}

	return (A[mid] == T)? mid : -1; // WORKs now bcz `mid`updated before while check, it reflects mid of current l & r
	
	// return (A[l] == T)? l : -1; // still works as l==mid (mid LEFT-biased)
}
```

Now, how to scan using right biased mid formula inside while < 
### using right biased mid : return `r` (for 1 elt)
```java
public int search(int[] A, int T) {
	int l = 0, r = A.length-1;
	
	while(l < r) { 

		int mid = (l+r+1) / 2; 

		if(A[mid] == T) return mid;

		if(A[mid] > T) r = mid - 1;  // (A[mid] > T) --> search left 
		else           l = mid + 1;  // (A[mid] < T) --> search right

	}

	return (A[r] == T)? r : -1; // symmetry 
    }
```
#### return `mid` (for 1 elt) : POST `mid` calc
```java
public int search(int[] A, int T) {
	int l = 0, r = A.length-1;
	
	int mid = 0;  // mid ∈ [l, r]
	while(l < r) { 

		if(A[mid] == T) return mid;

		if(A[mid] > T) r = mid - 1;  // (A[mid] > T) --> search left 
		else           l = mid + 1;  // (A[mid] < T) --> search right

		mid = (l+r+1) / 2; // POST
	}
	// The bounds check IS necessary for right-biased, but NOT necessary for left-biased.
	return (mid<A.length && A[mid] == T)? mid : -1; // right biased mid can exceed n-1
}
```

NOTE : unlike left biased `mid`, right based `mid` is POST calculated

##### SUMMARY

```java
public int search(int[] A, int T) {
	int l = 0, r= A.length-1;
	
	// POST
	// mid : scope method level
	while(l INEQ r) { // `<` or `<=`
		// PRE : 
		mid = left/right bias
		// mid : scope while level
		
		check : A[mid] == T 
		decide left or right partition
		
        // POST
        mid = left/right bias 
	}
	/*
	if ineq <= 
		return -1
	if ineq <
		if mid PRE : check A[l] for left-biased mid (or A[r] for right-biased mid)
		if mid POST: check A[mid] 
	*/
}
```

| while() | mid bias | mid scope  | PRE or POST | returns from inside while() if T exists else |
| ------- | -------- | ---------- | ----------- | -------------------------------------------- |
| l <= r  | left     | while only | either      | -1 outside                                   |
| --do--  | right    | while only | either      | -1 outside                                   |
| l < r   | left     | while only | PRE         | `(A[l] == T)? l : -1;`                       |
| --do--  | left     | method()   | POST        | `(A[mid] == T)? mid : -1;`                   |
| --do--  | right    | while only | PRE         | `(A[r] == T)? r : -1;`                       |
| --do--  | right    | method()   | POST        | `(mid<n && A[mid] == T)? mid : -1;`          |
##### MOST GENERIC TEMPLATE 
if we break at check inside while() we can have single point of return at then end of the method.
```java
class Solution {
    public int search(int[] A, int T) {
        int n = A.length;
        int l = 0, r= n-1;
        
        int mid = 0; // Initialize to any valid index ∈ [0, n-1]
        while(l < r) { // or, <=
			// Check current mid (initialized first iteration, calculated from previous iteration)
            if(A[mid] == T) break;// with break always use POST

            if(A[mid] >  T) r = mid - 1; 
            else            l = mid + 1; 
			
			// Recalculate mid for next iteration (or final check after loop exits)
            mid = (l+r)/2;// or (l+r+1)/2 for right-biased
        }
        // After loop: mid is either the found element OR the last calculated mid

        // single point of return
        // RIGHT-BIASED POST: mid CAN exceed n-1 after loop, bounds check REQUIRED
		return (mid < A.length && A[mid] == T) ? mid : -1;
		
		// LEFT-BIASED POST: mid stays within bounds, bounds check NOT required
		// return (A[mid] == T) ? mid : -1;  // Safe without the mid < A.length check
    }
}
`````

NOTE : 
- always update mid using POST when using break
- inequality sign < or <= becomes irrelevant 
- NOTE ON BOUNDS CHECKING:
	- `l <= `r: Always returns inside while() via direct check, no outside bounds check needed
	- `l < r` with POST mid: 
	  * LEFT-biased: mid stays within `[0, n-1]`, bounds check (optional)
	  * RIGHT-biased: mid can exceed `[0, n-1]`, bounds check REQUIRED

-----
# `1a. (l <= r) BASE CASE: ( A[l] <= A[r] )`
```java
public int findMin(int[] A) {
	int l = 0, r = ( A.length - 1);
	
	while( l <= r ) {
		if( A[l] <= A[r] )
			return A[l]; 

		int mid = (l + r) / 2 ;  

		if( A[mid] > A[r] )    // or symmetry : ( A[mid] >= A[l] ) 
			l = mid + 1;
		else // ( A[mid] <= A[r] )   or symmetry ( A[mid] < A[l] ) 
			r = mid ;
	}
	
	return -9999; // unreachable
}
```
# `1a.(symmetry) (l <= r) BASE CASE: ( A[l] <= A[r] )`
```java
public int findMin(int[] A) {
	int l = 0, r = ( A.length - 1);
	
	while( l <= r ) {
		if( A[l] <= A[r] )
			return A[l]; // ALWAYS returns from here incl arr.size = 1

		int mid = (l + r) / 2 ; // = l + (r - l) / 2 ;

		if( A[mid] < A[l] ) 
			r = mid ;
		else // ( A[mid] >= A[l] ) 
			l = mid + 1; 
	}
	
	return -9999; // unreachable
}
```
# `1b.(l < r) BASE CASE: ( A[l] < A[r] )`

```java
public int findMin(int[] A) {
	int l = 0, r = ( A.length - 1);
	int mid = (l + r) / 2 ;
	
	while( l < r ) {
		if( A[l] < A[r] ) // works since all elt unique 
			return A[l]; 

		mid = (l + r) / 2 ; 

		if( A[mid] > A[r] ) 
			l = mid + 1;
		else // A[mid] <= A[r]
			r = mid ;
	}
	
	return A[l]; // arr.size < 2
	// do not return A[mid]
}
```
# `1b.(symmetry) (l < r) BASE CASE: ( A[l] < A[r] )`
```java
public int findMin(int[] A) {
	int l = 0, r = ( A.length - 1);
	int mid = (l + r) / 2 ;
	while( l < r ) {
		if( A[l] < A[r] )
			return A[l];  // for arr.size > 1

		mid = (l + r) / 2 ; // = l + (r - l) / 2 ;

		if( A[mid] > A[r] ) 
			l = mid + 1;
		else
			r = mid ;
	}
	return A[l]; // returns from [arr.size = 1] Can return from here. 
	// don't return A[mid] or 
}
```


```java
    public int countLocalMaximums(int[][] G) {
        int n = G.length, m = G[0].length;
        int cnt = 0; // count of local maximum
        
        for(int row = 0; row < n; row++) {
            for(int col = 0; col < m; col++) {
                int x = G[row][col];
                
                if(x != 0) { // local maximum if it is non-zero
                    
                    // considered cell : every cell within x rows and x columns of (row, col).
                    for(int i= -x; i <= x; i++ ) { // r_dist 
                        for(int j= -x; j <= x; j++) { // c_dist
                            
                            //Ignore the cells where both the row distance and column distance are exactly x.
                            if( (i+j!=0) || (Math.abs(i)+Math.abs(j) != 2*x)  ) { 
                                int row_ = row+i, col_ = col+j;
                                if(row_>=0 && row_<n && col_>=0 && col_<m) {
                                    if(G[row_][col_] > x)
                                        break outer;
                                }
                            }
                        }
                    }
                    cnt++; // if no considered cell has a value greater than x.
                }
            }
        }

        return cnt;
    } 
```