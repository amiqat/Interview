## Binary Tree Level Order Traversal

**Difficulty:** Medium

**Topic:** Trees & Breadth-First Search

### Problem Statement

Given the root of a binary tree, return the level order traversal of its nodes' values (i.e., from left to right, level by level).

### Example

```
Input: root = [3,9,20,null,null,15,7]
    3
   / \
  9  20
    /  \
   15   7
Output: [[3],[9,20],[15,7]]
```

```
Input: root = [1]
Output: [[1]]
```

```
Input: root = []
Output: []
```

### Solution Approach

**Approach 1: Breadth-First Search (BFS) with Queue**
- Use a queue to process nodes level by level
- For each level, process all nodes in the queue
- Add children of current level to queue for next iteration
- Time complexity: O(n), Space complexity: O(n)

**Approach 2: Depth-First Search (DFS) with Recursion**
- Use recursion with level tracking
- Append values to corresponding level in result
- Time complexity: O(n), Space complexity: O(h) where h is height

### Code Solution

**Python - BFS:**
```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def level_order(root):
    """
    Perform level order traversal of a binary tree.
    
    Args:
        root: Root node of the binary tree
        
    Returns:
        List of lists containing node values by level
    """
    if not root:
        return []
    
    result = []
    queue = deque([root])
    
    while queue:
        level_size = len(queue)
        current_level = []
        
        for _ in range(level_size):
            node = queue.popleft()
            current_level.append(node.val)
            
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        
        result.append(current_level)
    
    return result
```

**Python - DFS:**
```python
def level_order_dfs(root):
    """
    Perform level order traversal using DFS.
    
    Args:
        root: Root node of the binary tree
        
    Returns:
        List of lists containing node values by level
    """
    result = []
    
    def dfs(node, level):
        if not node:
            return
        
        if len(result) == level:
            result.append([])
        
        result[level].append(node.val)
        dfs(node.left, level + 1)
        dfs(node.right, level + 1)
    
    dfs(root, 0)
    return result
```

**JavaScript - BFS:**
```javascript
function levelOrder(root) {
    if (!root) return [];
    
    const result = [];
    const queue = [root];
    
    while (queue.length > 0) {
        const levelSize = queue.length;
        const currentLevel = [];
        
        for (let i = 0; i < levelSize; i++) {
            const node = queue.shift();
            currentLevel.push(node.val);
            
            if (node.left) queue.push(node.left);
            if (node.right) queue.push(node.right);
        }
        
        result.push(currentLevel);
    }
    
    return result;
}
```

**Java - BFS:**
```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> currentLevel = new ArrayList<>();
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            currentLevel.add(node.val);
            
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        
        result.add(currentLevel);
    }
    
    return result;
}
```

### Time & Space Complexity

**BFS Approach:**
- **Time Complexity:** O(n) - Visit each node once
- **Space Complexity:** O(n) - Queue can hold up to n/2 nodes at the last level

**DFS Approach:**
- **Time Complexity:** O(n) - Visit each node once
- **Space Complexity:** O(h) - Recursion stack where h is the height of the tree

### Follow-up Questions

1. How would you do a zigzag level order traversal (alternating left-to-right and right-to-left)?
2. Can you return the level order traversal from bottom to top?
3. How would you find the rightmost node at each level?
4. What if you need to return the average value of nodes at each level?

### Tags

`tree` `binary-tree` `bfs` `dfs` `queue` `medium` `popular`
