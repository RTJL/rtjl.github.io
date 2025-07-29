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

## Smallest Subarrays With Maximum Bitwise OR

Store the minimum index where a bit(1) is set.

Use it to determine the smallest subarray length where it will meet the max bitwise OR value.

### Time

`O(n x logB)`

`logB` is the number of bits for one of the num.

### Space

`O(logB)`

`logB` is the number of bits, the `bit_position_lst` variable.

```python
class Solution:
    def smallestSubarrays(self, nums: List[int]) -> List[int]:
        if len(nums) == 1:
            return [1]

        # 10^9 value need 31bits
        # use -1 to indicate that never see this bit at position before
        bit_position_lst = [-1] * 31

        # result list
        res = [1] * len(nums)

        # max or
        max_or = 0

        for i in range(len(nums) - 1, -1, -1):
            # convert current number into binary
            # update the bit_position_lst
            curr_num = nums[i]
            max_or = max_or | curr_num

            # handle special case
            if max_or == 0:
                res[i] = 1
                continue

            index = 0
            while curr_num > 0:
                bit = curr_num % 2
                if bit == 1:
                    if bit_position_lst[index] == -1:
                        bit_position_lst[index] = i
                    else:
                        bit_position_lst[index] = min(bit_position_lst[index], i)

                curr_num = curr_num // 2
                index += 1

            res[i] = max(bit_position_lst) - i + 1
        return res
```

Brute force, double for loop.

### Time

`O(n^2)`

### Space

`O(n)`

Need to store the `max_or_lst`.

```python
class Solution:
    def smallestSubarrays(self, nums: List[int]) -> List[int]:
        # get the max OR value start from current
        max_or_lst = [0] * len(nums)
        max_or_lst[len(nums) - 1] = nums[len(nums) - 1]
        for i in range(len(nums) - 2, -1, -1):
            max_or_lst[i] = max_or_lst[i + 1] | nums[i]

        # result
        res = []

        # double for loop
        for i in range(len(nums)):
            curr = nums[i]

            # the number itself is the max_or value
            # so shortest will be 1
            if curr == max_or_lst[i]:
                res.append(1)
                continue

            for j in range(i + 1, len(nums)):
                curr = curr | nums[j]

                if curr == max_or_lst[i]:
                    res.append(j - i + 1)
                    break

        return res
```
