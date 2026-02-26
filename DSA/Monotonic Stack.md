
![[mono stack cover.png]]
In this article I intend to provide greater insight on how to Construct a MS. Many articles exist explaining its applicability. But constructing MS might be confusing. However, after reading this article reader will be able to intuitively design different variations of Monotonic Stack depending on requirements. 

**TL;DR**: Before pushing new element ensure insertion doesn't violate monotonic property.

## The Structure : 

1. Define Monotonic Property (analyze the requirement depending on the problem)
2. Initialize empty stack
3. for each `newElt` from input stream
4.     continue popping stack **until pushing `newElt` ==violates Property**==  
5.     stack.push(newElt)

```
1. Be ready wih Monotonic Property to be enforced
2. Initialize empty stack
3. for each newElt from input stream
4.     continue popping stack until pushing newElt violates Property  
5.     stack.push(newElt)
```
Only line #1 and #4 need explanation. 
### 1. Defining Monotonic Property
While coding step 1 is not explicitly defined . Rather it is enforced via line 4 implementation. But one needs to be clear with what **property** s/he wants to enforce via the stack.  This brings us to discussing the types of Monotonic Stack.

Depending on problem, we can have four different types which we can be ground as two : 
1. Monotonic **Increasing** Stack
	1. **Strictly Increasing** e.g. 1,2,3,4,5 (non-repeating elements)
	2. Not Strictly Increasing (or **Non-Decreasing** ) e.g. 1,2,3,3,4,4,5,5 (repeating elements)
2. Monotonic **Decreasing** Stack 
	1. **Strictly Decreasing** e.g. 5,4,3,2,1 (non-repeating elements)
	2. Not Strictly Decreasing (or **Non-Increasing** ) e.g. 5,5,4,4,3,3,2,1 (repeating elements)

For the sake of generalizability lets stick to the Non-Decreasing Increasing version. 
#### Monotonic Increasing Stack : Definition
Like beauty, definition lies in the eyes of the beholder. Three perspectives, three definitions but same characteristic : 
- **Popping perspective** : When popping, the popped should be smaller/equal than previously popped element(if exists).
- **Pushing perspective** : When pushing, `newElt` should be greater/equal than  previously pushed element(if exists).
- **External observation (snapshot)** : Although you cannot see all elements of stack but imagine for once you can, then at any time starting from bottom it should grow towards top.

Formally, 
* add new element to stack only *iff* :  `newElt >= st.top()`
* Put negatively add element when `newElt < st.top()` : It is this form of formalization that we shall use in code. 

![[Monotonic Stack 01.png]]
![[Mono Stack Correct.png]]
*Trick to memorize* : The reference point is always **Base** of the stack 
Monotonic Increasing Stack =  
- Monotonic Increasing (from **bottom**) Stack
- Monotonic Increasing (from **last pushed element**) Stack

### 4. newElt violates #1
how does this look like for MIS ? 

```python
1. # MIS - to be enforeced using line #4
2. stack = []
3. for each newElt = a[i]
4.     while newElt < st.top() if st has a top elt
		   st.pop()  
5.     stack.push(newElt)
```

For Monotonic Strictly Increasing Stack we have minor change in #4 : `newElt <= st.top()`

##### Monotonic Increasing Stack Simulation
**Example Array:** `[4, 2, 8, 5, 3]` 
1. **Input 4:** Stack empty -> Push 4. Stack: `[4]`
2. **Input 2:**   2< 4 (top). Pop 4.    2< empty. Push 2. Stack: `[2]`
3. **Input 8:**  8>2(top). Push 8. Stack: `[2, 8]`
4. **Input 5**: 5<8 (top). Pop 8. 5>2. Push 5. Stack: `[2, 5]`
5. **Input 3**: 3<5 (top). Pop 5. 3>2 . Push 3. Stack: `[2, 3]`

_Final Monotonic Increasing Stack:_ `[2, 3]` (elements are in increasing order)
Used to find the next smaller element (nearest element to the right that is smaller)


![[Mono Stack Simul 0.png]]

![[Mono Stack Simul.png]]
##### Monotonic Decreasing Stack Simulation

**Example Array:** `[4, 2, 8, 5, 3]` 
1. **Input 4:** Stack empty -> Push 4. Stack: `[4]`
2. **Input 2:** 2<4 (top). Push 2. Stack: `[4, 2]`
3. **Input 8:** 8>2 (top). Pop 2. 8>4. Pop 4. Push 8. Stack: `[8]`
4. **Input 5:** 5<8 (top). Push 5. Stack: `[8, 5]`
5. **Input 3**: 3<5 (top). Push 3. Stack: `[8, 5, 3]` 

_Final Monotonic Decreasing Stack:_ `[8, 5, 3]` (elements are in decreasing order).
Used to find the next greater element (nearest element to the right that is larger)

![[Mono Stack Simul 2.png]]

[Monotonic Stack/Queue Intro](https://algo.monster/problems/mono_stack_intro) 
[Introduction to Monotonic Stack](https://www.designgurus.io/course-play/grokking-the-coding-interview/doc/introduction-to-monotonic-stack)
[Introduction to Monotonic Stack - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/introduction-to-monotonic-stack-2/) 