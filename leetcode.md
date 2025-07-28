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

## Count Number of Maximum Bitwise-OR Subsets

Calculate the max bitwise value.

Just recursively generate all possible sets.

### Time

`O(2^n)`

### Space

`O(n)`

Tree up till `n` levels deep

```python
class Solution:
    def countMaxOrSubsets(self, nums: List[int]) -> int:
        # max value
        max_val = 0
        for n in nums:
            max_val = max_val | n

        def calculate(res, index):
            # base case
            if index >= len(nums):
                return res == max_val

            # include current index
            with_curr = calculate(res | nums[index], index + 1)

            # skip current index
            without_curr = calculate(res, index + 1)

            return with_curr + without_curr

        return calculate(0, 0)
```
