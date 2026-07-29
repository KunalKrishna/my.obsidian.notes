
## int → long overflow: all cases

### Limits
- `int` max: ~2.1 × 10⁹  
- `long` max: ~9.2 × 10¹⁸

### Multiplication (most common bug)
```java
long r = (long) a * b;      // SAFE — cast FIRST operand
long r = (long)(a * b);     // WRONG — overflows before cast
long r = 1L * a * b;        // also safe
```

### Addition
```java
int mid = lo + (hi - lo) / 2;         // safe midpoint
int mid = (int)(lo + (long)hi) / 2;   // cast to promote
```

### Division
```java
(long) x / -1          // avoids MIN_VALUE / -1 trap
Math.floorDiv(a, b)    // floor division, handles negatives
```

### Modulo
```java
((a % m) + m) % m      // always non-negative
Math.floorMod(a, m)    // Java 8+, equivalent
```

### Sneaky traps
- `Math.abs(Integer.MIN_VALUE)` → still negative! Use `Math.abs((long)x)`
- `(int) someLong` truncates silently, no exception

### When to always use long
- `lo + hi` in binary search
- row * cols index math
- sum of large arrays
- combinatorics / factorial
- comparing `a * b > target`


----

[Just a moment...](https://leetcode.com/problems/reverse-linked-list/solutions/8310140/ascii-animation-wide-screen-by-thevagabo-bfpp/)

ASCII Animation [Wide Screen]

# Code

```java []

class Solution {

    public ListNode reverseList(ListNode head) {

        if(head == null || head.next == null)   return head;

        /** Temporarily save reference of first two nodes which shall be used POST recursion stitching */

        //    [[head]] ------------------------> [[tail nodes -> t2 -> ... -> tx -> NULL]]

        //       ^                                 ^

        //       ¦                                 ¦

        //       ¦                                 ¦

        //       ¦                                 ¦

        ListNode HeadRefBeforeRecursion = head;//  ¦

        //                                         ¦

        //                                         ¦

        //                                         ¦

        //                                         ¦

        //                                         ¦

        //                                         ¦

        ListNode                      HeadNextRefBeforeRecursion = head.next;

  

        /** LEAP OF FAITH : Trust Base case & Recursion */

  

        // (HeadRefBeforeRecursion)              (HeadNextRefBeforeRecursion)

        // [[head]] --------------------------> [[tail nodes <- t2 <- ... <- tx]]

        //                                         |                         ^

        //                                         V                         |

        //                                        NULL                       |

        //                                                                   |

        //                                                                   |

        //                                                                   |

        //                                                                   |

        //                                                                   |

        //                                                                   |

        //                                                                   |

        ListNode                                               headOfRecursedTailNodes = reverseList(head.next);

  

        /** POST Operations stitching (BACKTRACKING): stitch now (i.e. do operation on current node AFTER recurively operating children nodes) */

  

        HeadNextRefBeforeRecursion.next = HeadRefBeforeRecursion; // forms a CYCLE

        //                   [[head]] ----------OldHeadLink------> [[tail nodes                 <- t2 <- ... <- tx]]

        // [[HeadRefBeforeRecursion]] <----JustCreatedInLine38---- [[HeadNextRefBeforeRecursion <- t2 <- ... <- headOfRecursedTailNodes]]

  
  

        /** set OldHeadLink to NULL */

        HeadRefBeforeRecursion.next = null; // this is V.IMP , need to GROUND(null) the "next" of head as you back track as this is the LAST NODE of the reversed LL

        // [[HeadRefBeforeRecursion]] <----------------------- [[HeadNextRefBeforeRecursion <- t2 <- ... <- headOfRecursedTailNodes]]

        //       |                                                                                             ^

        //   OldHeadLink                                                                                       |

        //       |                                                                                             |

        //       V                                                                                             |

        //      NULL                                                                                           |

        return                                                                                      headOfRecursedTailNodes;

    }

}

```

```java []

class Solution {

    public ListNode reverseList(ListNode head) {

        if(head == null || head.next == null)   return head;

        ListNode first = head, secnd = head.next;

  

        ListNode reversedTail = reverseList(head.next);

  

        secnd.next = first;

        first.next = null;

        /* Replace above 2 lines by these 3 to REUSE null of original LL

        ListNode nullNode = secnd.next;

        secnd.next = first;

        first.next = nullNode;

        */

        return reversedTail;

    }

}

```

```java []

class Solution {

    public ListNode reverseList(ListNode head) {

        if(head == null || head.next == null)

            return head;

  

        ListNode tail = reverseList(head.next);

  

        ListNode nullNode = head.next.next;

        head.next.next = head;

        head.next = nullNode;  // reuse

        return tail;

    }

}

```

# Complexity

- Time complexity:  $O(n)$  each node visited(processed) once
- Space complexity:  $O(n)$ implicit stack space

----

```java
class Solution {

    public Node cloneGraph(Node node) {
        Node clone = new Node();
        dfs(node, clone);
        return clone;
    }

    void dfs(Node from, Node to) {
        if(from == null) return;
        Node to = new Node(from.val);
        for(Node neighbour : from.neighbors) {

            // Think : like traditional DFS ques say "Print every node" we still need NOT REVISIT the visited node

            // BUT unlike those ques we need to perform the task of LINKING curr node & ALREADY VISITED NODE.

            // How to do that? ? ?

            // using original graph & Auxillary DS (Visited set etc) we need to somehow ensure that

            /** pseudocode
            if(already not LINKED)
                Link
            else
                do nothing
            */
        }
        return to;
    }
}
```