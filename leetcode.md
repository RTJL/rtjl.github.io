## Count Hills and Valleys in an Array

Just iterating through with some conditions.

1. Find the start.

2. Start from the start index found previously.

    In each loop, find the next index.

    Check if is valley/hill & update result.

    If next index out of bounds, then straight away return.

### Time

`O(n)`

Simple loop through the list.

### Space

`O(1)`

Only a few extra constant space variables used to track the progress.

```python
class Solution:
    def countHillValley(self, nums: List[int]) -> int:
        res = 0

        n = len(nums)

        prev_num = nums[0]

        # find the index to start at
        start = 1
        while nums[start] == prev_num and start < n - 1:
            start += 1
        
        # update prev num
        prev_num = nums[start - 1]

        # start to iterate through the numbers
        curr_idx = start
        while curr_idx < n:
            curr_num = nums[curr_idx]

            # find the closest right that is != to current number
            next_idx = curr_idx + 1
            while next_idx < n and nums[next_idx] == curr_num:
                next_idx += 1
            
            # break early if next closest right is out of bounds
            if next_idx == n:
                return res

            # add to the res
            is_valley = (prev_num > curr_num) and (curr_num < nums[next_idx])
            is_hill = (prev_num < curr_num) and (curr_num > nums[next_idx])
            if is_valley or is_hill:
                res += 1
            
            # update curr index to next
            curr_idx = next_idx
            prev_num = curr_num
        
        return res
```

## Number of 1 bits

1. Get the remainder mod by 2, add to result.

2. Keep on shifting the bits to the right by 1 per iteraction.

### Time

`O(1)`

### Space

`O(1)`

```python
class Solution:
    def hammingWeight(self, n: int) -> int:
        res = 0
        while n > 0:
            res += n % 2
            n = n >> 1

        return res
```

## Implement Trie (Prefix Tree)

Tree with multiple children nodes

```python
class Trie:

    def __init__(self):
        self.is_leaf = False
        self.children = {}

    def insert(self, word: str) -> None:
        node = self
        for c in word:
            if c in node.children:
                node = node.children[c]
            else:
                new_node = Trie()
                node.children[c] = new_node
                node = node.children[c]

        node.is_leaf = True

    def search(self, word: str) -> bool:
        node = self

        for c in word:
            if c in node.children:
                node = node.children[c]
            else:
                return False

        return node.is_leaf

    def startsWith(self, prefix: str) -> bool:
        node = self

        for c in prefix:
            if c in node.children:
                node = node.children[c]
            else:
                return False

        return True
```
