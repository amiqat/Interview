## Reverse Linked List

**Difficulty:** Easy

**Topic:** Linked Lists

### Problem Statement

Given the head of a singly linked list, reverse the list and return the reversed list.

### Example

```
Input: head = [1,2,3,4,5]
Output: [5,4,3,2,1]
```

```
Input: head = [1,2]
Output: [2,1]
```

```
Input: head = []
Output: []
```

### Solution Approach

**Approach 1: Iterative**
- Use three pointers: prev, current, next
- Iterate through the list, reversing the direction of each pointer
- Time complexity: O(n), Space complexity: O(1)

**Approach 2: Recursive**
- Recursively reverse the rest of the list
- Fix the pointers at current level
- Time complexity: O(n), Space complexity: O(n) due to call stack

### Code Solution

**Python - Iterative:**
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def reverse_list(head):
    """
    Reverse a singly linked list iteratively.
    
    Args:
        head: Head node of the linked list
        
    Returns:
        Head of the reversed linked list
    """
    prev = None
    current = head
    
    while current:
        next_temp = current.next  # Store next node
        current.next = prev       # Reverse the link
        prev = current            # Move prev forward
        current = next_temp       # Move current forward
    
    return prev
```

**Python - Recursive:**
```python
def reverse_list_recursive(head):
    """
    Reverse a singly linked list recursively.
    
    Args:
        head: Head node of the linked list
        
    Returns:
        Head of the reversed linked list
    """
    # Base case: empty list or single node
    if not head or not head.next:
        return head
    
    # Recursively reverse the rest of the list
    new_head = reverse_list_recursive(head.next)
    
    # Fix the pointers
    head.next.next = head
    head.next = None
    
    return new_head
```

**JavaScript - Iterative:**
```javascript
function reverseList(head) {
    let prev = null;
    let current = head;
    
    while (current !== null) {
        const nextTemp = current.next;
        current.next = prev;
        prev = current;
        current = nextTemp;
    }
    
    return prev;
}
```

**Java - Iterative:**
```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode current = head;
    
    while (current != null) {
        ListNode nextTemp = current.next;
        current.next = prev;
        prev = current;
        current = nextTemp;
    }
    
    return prev;
}
```

### Time & Space Complexity

**Iterative Approach:**
- **Time Complexity:** O(n) - Visit each node once
- **Space Complexity:** O(1) - Only use constant extra space

**Recursive Approach:**
- **Time Complexity:** O(n) - Visit each node once
- **Space Complexity:** O(n) - Recursion call stack

### Follow-up Questions

1. Can you reverse the list in-place without using extra space?
2. How would you reverse a doubly linked list?
3. Can you reverse only a portion of the list (between positions m and n)?
4. How would you reverse a linked list in groups of k?

### Tags

`linked-list` `recursion` `iteration` `easy` `popular`
