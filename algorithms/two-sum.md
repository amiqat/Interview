## Two Sum

**Difficulty:** Easy

**Topic:** Arrays & Hash Tables

### Problem Statement

Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`.

You may assume that each input would have exactly one solution, and you may not use the same element twice.

You can return the answer in any order.

### Example

```
Input: nums = [2,7,11,15], target = 9
Output: [0,1]
Explanation: Because nums[0] + nums[1] == 9, we return [0, 1].
```

```
Input: nums = [3,2,4], target = 6
Output: [1,2]
```

```
Input: nums = [3,3], target = 6
Output: [0,1]
```

### Solution Approach

**Approach 1: Brute Force**
- Use nested loops to check every pair of numbers
- Time complexity: O(n²)

**Approach 2: Hash Map (Optimal)**
- Use a hash map to store numbers and their indices
- For each number, check if (target - number) exists in the hash map
- Time complexity: O(n)

### Code Solution

**Python:**
```python
def two_sum(nums, target):
    """
    Find two numbers that add up to target.
    
    Args:
        nums: List of integers
        target: Target sum
        
    Returns:
        List of two indices
    """
    num_map = {}
    
    for i, num in enumerate(nums):
        complement = target - num
        if complement in num_map:
            return [num_map[complement], i]
        num_map[num] = i
    
    return []
```

**JavaScript:**
```javascript
function twoSum(nums, target) {
    const numMap = new Map();
    
    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];
        if (numMap.has(complement)) {
            return [numMap.get(complement), i];
        }
        numMap.set(nums[i], i);
    }
    
    return [];
}
```

**Java:**
```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> numMap = new HashMap<>();
    
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (numMap.containsKey(complement)) {
            return new int[] { numMap.get(complement), i };
        }
        numMap.put(nums[i], i);
    }
    
    return new int[] {};
}
```

### Time & Space Complexity

- **Time Complexity:** O(n) - Single pass through the array
- **Space Complexity:** O(n) - Hash map stores at most n elements

### Follow-up Questions

1. What if the array is sorted? Could you use a two-pointer approach?
2. How would you handle duplicate numbers?
3. What if you need to find all pairs that sum to target instead of just one?
4. Can you solve it with O(1) space? (Spoiler: Not while maintaining O(n) time)

### Tags

`array` `hash-table` `easy` `popular`
