Traversal 
- In Order (a+b) : 1 Stack + 2 while loop
- Pre Order(+ab) : 
- Post Order(ab+) :
## InOrder (Left → Root → Right) :

### The Standard Implementation
The standard textbook implementation of an iterative InOrder traversal explicitly uses **nested `while` loops** (one outer, one inner) along with a Stack.

This structure perfectly mimics the recursive call stack:
1. **The Stack** replaces the implicit "Call Stack" of recursion.
2. **The Inner `while` loop** replaces the recursive call `findKeyNode(root.left)`. It aggressively pushes nodes until it hits `null` (simulating "going deep" to the left).
3. **The Outer `while` loop** ensures we process the popped node and then attempt to traverse the right subtree (`root.right`), just like the recursive function does after returning from the left.

- **The Stack** replaces the implicit "Call Stack" of recursion.
- **The Inner `while` loop** replaces the recursive call `findKeyNode(root.left)`. It aggressively pushes nodes until it hits `null` (simulating "going deep" to the left).
- **The Outer `while` loop** ensures we process the popped node and then attempt to traverse the right subtree (`root.right`), just like the recursive function does after returning from the left.

```Java
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> res = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;

    // Outer Loop: Keeps the traversal alive as long as we have nodes 
    // to process or the stack isn't empty.
    while (curr != null || !stack.isEmpty()) {
        
        // Inner Loop: "Go Left" 
        // Simulates the recursive call to root.left
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        // Process Node (Simulates the base case return + processing)
        curr = stack.pop();
        res.add(curr.val); 

        // "Go Right"
        // Simulates the recursive call to root.right
        curr = curr.right; 
    }
    return res;
}
```
### Why this distinction matters
Understanding this structure is crucial because **PreOrder** traversal is very similar (just change where you `add` to result), but **PostOrder** iterative traversal is significantly harder and usually requires a different trick (like two stacks or a `lastVisited` pointer) to handle the backtracking logic correctly.

## PreOrder (Root → Left → Right)

To switch from InOrder to PreOrder using the "Two While Loops" pattern, you simply move the **processing line** (e.g., `list.add`) from _after_ the inner loop to _inside_ the inner loop.
### The "Two While Loops" Pattern (PreOrder vs InOrder)
The logic holds:
- **InOrder (Left → Root → Right):** We push nodes to the stack to "save them for later." We process them **after** we pop them (coming back from the left).
- **PreOrder (Root → Left → Right):** We process the node **immediately** when we see it (before going left), then push it to the stack to remember to go right later

```Java
    while (curr != null || !stack.isEmpty()) {
        
        // Inner Loop: "Go Left"
        while (curr != null) {
            res.add(curr.val);   // <--- PreOrder: Process BEFORE pushing (Root first)
            stack.push(curr);
            curr = curr.left;
        }

        // Backtrack
        curr = stack.pop(); 
        // InOrder would process here!
        
        // "Go Right"
        curr = curr.right; 
    }
```


### "Single Loop" implementation for PreOrder traversal
**The Key Trick:** Since a Stack is LIFO (Last-In, First-Out), you must push the **Right child first** and the **Left child second**. This ensures the Left child is popped and processed _before_ the Right child.

```Java
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result; // Handle edge case

    Stack<TreeNode> stack = new Stack<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode curr = stack.pop();
        result.add(curr.val); // 1. Process Root immediately

        // 2. Push RIGHT first (so it stays at the bottom)
        if (curr.right != null) {
            stack.push(curr.right);
        }

        // 3. Push LEFT second (so it ends up on TOP to be popped next)
        if (curr.left != null) {
            stack.push(curr.left);
        }
    }
    return result;
}
```

## PostOrder (Left → Right → Root)

PostOrder can be done with **one stack** and the same **two-while-loop** structure you learned for InOrder, but it requires a clever check to distinguish if you are returning from the Left child or the Right child.

Here is the breakdown of the methods:
### Method 1: The "Two-Stack" Trick (Easiest to Memorize)
This is the "cheat code."
- **Logic:** PostOrder (Left → Right → Root) is just the **reverse** of (Root → Right → Left).
- **Algorithm:** Do a "modified PreOrder" (Root → Right → Left) using **1 while loop**, but push the result into a second stack (or `addFirst` to a LinkedList) so it comes out in reverse order.
- **Code:** Very similar to the single-loop PreOrder I showed you.
### Method 2: One Stack (The "Real" Solution)
- **Does it require two stacks?** No.
- **How many while loops?** It uses the same **Two Loop** pattern (Outer + Inner).
- **The Difference:** You use a `lastVisited` pointer. When you peek at a node from the stack, you check: _Did I just come up from its right child?_
    - **If No:** Move to the right child.
    - **If Yes (or no right child):** Pop and process.
### Summary of Differences

|**Traversal**|**Loops**|**Key Logic**|
|---|---|---|
|**InOrder**|2|Go Left → **Pop & Process** → Go Right|
|**PreOrder**|1 or 2|**Process** → Go Left → Go Right|
|**PostOrder**|2|Go Left → Check Right (Peek) → **Pop & Process**|

### **Method 1 (Two Stacks)**.

### The Logic (Reverse Thinking)
1. **Standard PostOrder:** Left → Right → Root  
2. **Reverse PostOrder:**   Root → Right → Left  { (ab+) | (+ba) | 
3. This reverse order is identical to **PreOrder**, just swapping the order of children.

So, we perform a modified PreOrder traversal using `stack1`, push the results into `stack2` (instead of printing them), and then pop `stack2` at the end to get the correct order.

```Java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Stack<TreeNode> stack1 = new Stack<>(); // For traversal
    Stack<TreeNode> stack2 = new Stack<>(); // For storage (reverses the order)

    stack1.push(root);

    while (!stack1.isEmpty()) {
        TreeNode curr = stack1.pop();
        
        // Push to second stack (this node is processed in reverse order)
        stack2.push(curr);

        // Push LEFT first, then RIGHT
        // So that in stack1, RIGHT is on top and processed next (Root -> Right -> Left)
        if (curr.left != null) {
            stack1.push(curr.left);
        }
        if (curr.right != null) {
            stack1.push(curr.right);
        }
    }

    // stack2 now holds: Root, Right, Left (from bottom to top)
    // Popping gives: Left, Right, Root
    while (!stack2.isEmpty()) {
        result.add(stack2.pop().val);
    }

    return result;
}
```
### Optimization Note
In Java, you can avoid explicitly creating `stack2` by using a `LinkedList` for your result list and calling `.addFirst()`. This achieves the exact same "reversal" effect in $O(1)$ extra space logic.

```Java
// Optimization: "stack2" is just the front of the result list
LinkedList<Integer> result = new LinkedList<>(); 
// ... inside loop ...
result.addFirst(curr.val); 
```

### Method 2 : **One Stack** implementation using **Two Loop** structure

```Java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;
    TreeNode lastVisited = null;

    // Outer Loop
    while (curr != null || !stack.isEmpty()) {
        
        // Inner Loop: Go Left Deep (Same as InOrder)
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        // CRITICAL STEP: Peek at the node (do not pop yet!)
        TreeNode peekNode = stack.peek();

        // Check: Do we have a right child we haven't visited yet?
        if (peekNode.right != null && lastVisited != peekNode.right) {
            // Yes -> Move right
            curr = peekNode.right;
        } else {
            // No -> We are done with both children. Process Root.
            stack.pop();
            result.add(peekNode.val);
            lastVisited = peekNode; // Mark as visited
            // Leave 'curr' as null so the loop forces us to peek the next parent
        }
    }
    return result;
}
```

### Approach 1: The "Two Stacks" Trick (Memory Heavy, Logic Easy)

**The Trick:** PostOrder is `Left -> Right -> Root`. If you reverse this sequence, you get `Root -> Right -> Left`. This looks exactly like **PreOrder**, just visiting the Right child before the Left.

So, instead of struggling with complex backtracking, we cheat:
1. Do a modified PreOrder (`Root -> Right -> Left`).
2. Push everything into a second stack.
3. Pop the second stack to get the result.

```Java
public List<Integer> postorderTwoStacks(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Stack<TreeNode> s1 = new Stack<>();
    Stack<TreeNode> s2 = new Stack<>(); // TRICK: The "Reversal" Stack

    s1.push(root);

    while (!s1.isEmpty()) {
        TreeNode curr = s1.pop();
        s2.push(curr); // Store for later (Reverse order)

        // Push LEFT first so RIGHT is processed next (Root -> Right -> Left)
        if (curr.left != null) s1.push(curr.left);
        if (curr.right != null) s1.push(curr.right);
    }

    // Pop s2 to get "Left -> Right -> Root"
    while (!s2.isEmpty()) {
        result.add(s2.pop().val);
    }
    return result;
}
```

---
### Approach 2: The "One Stack" Trick (Memory Efficient, Logic Hard)

**The Trick:** The `lastVisited` pointer. In a standard iterative loop (like InOrder), when you pop a node, you don't know if you are coming back from its **Left** child (meaning you should now go Right) or its **Right** child (meaning you are done and should process Root).

We use `lastVisited` to track where we just came from.
- **Scenario A:** `root.right` is null. -> Process Root.
- **Scenario B:** `root.right` is `lastVisited`. -> We just finished the right side. Process Root.
- **Scenario C:** Neither. -> We must still visit the right side. Push Root back!

```Java
public List<Integer> postorderOneStack(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;
    TreeNode lastVisited = null; // TRICK: The "History" Tracker

    while (curr != null || !stack.isEmpty()) {
        // 1. Go as Deep Left as possible
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }

        // 2. Peek at the top node (Do NOT pop yet!)
        TreeNode peekNode = stack.peek();

        // 3. The Critical Decision:
        // If there is a Right child AND we haven't visited it yet...
        if (peekNode.right != null && lastVisited != peekNode.right) {
            curr = peekNode.right; // Move Right
        } else {
            // 4. Otherwise, we are done with this node (Left & Right are finished)
            result.add(peekNode.val);
            lastVisited = stack.pop(); // Mark as visited and remove
            // Keep curr as null so the loop forces us to peek the parent next
        }
    }
    return result;
}
```
### Which one should you use in an interview?
- **If they ask for "Iterative PostOrder":** Use **Approach 1 (Two Stacks)**. It's much harder to mess up under pressure. (You can also use a `LinkedList` and `addFirst()` instead of `stack2` to look smarter).
- **If they restrict space complexity:** Use **Approach 2**. It is the "pure" Computer Science solution.