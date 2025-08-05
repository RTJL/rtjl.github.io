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

## Longest Subarray With Maximum Bitwise AND

> let k be the maximum value of the bitwise AND of any subarray of nums

k is just the maximum value in nums, because a number on its own is also a subarray.

Find how many contiguous blocks of k, then choose the maximum contiguous block.

### Time

`O(n)`

### Space

`O(1)`

```python
class Solution:
    def longestSubarray(self, nums: List[int]) -> int:
        max_num = max(nums)

        count = 0
        res = 0
        for n in nums:
            if n == max_num:
                count += 1
            else:
                res = max(res, count)
                count = 0

        return max(res, count)
```

## Bitwise ORs of Subarrays

Optimised brute force

### Time

`O(n x w)`

`w` is the number of bits required to store the maximum number in the array

### Space

`O(n x w)`

`w` is the number of bits required to store the maximum number in the array

```python
class Solution:
    def subarrayBitwiseORs(self, arr: List[int]) -> int:
        res_set = set()

        cache_set = set()
        for i in range(len(arr) - 1, -1, -1):
            new_set = set()
            for n in cache_set:
                new = arr[i] | n
                new_set.add(new)

            new_set.add(arr[i])
            res_set.update(new_set)
            cache_set = new_set

        return len(res_set)
```

Brute force

### Time

`O(n^2)`

### Space

`O(n)`

```python
class Solution:
    def subarrayBitwiseORs(self, arr: List[int]) -> int:
        res = set()

        for i in range(len(arr)):
            res.add(arr[i])
            tmp = arr[i]

            for j in range(i + 1, len(arr)):
                tmp = tmp | arr[j]
                res.add(tmp)

        return len(res)
```

## Pascal's Triangle

Just iterating.

### Time

`O(n^2)`

### Space

`O(n^2)`

```python
class Solution:
    def generate(self, numRows: int) -> List[List[int]]:
        res = [[1]]

        if numRows == 1:
            return res

        for i in range(1, numRows):
            prev_res = res[i - 1]
            curr_res = [1]

            for j in range(1, len(prev_res)):
                new = prev_res[j - 1] + prev_res[j]
                curr_res.append(new)

            curr_res.append(1)
            res.append(curr_res)

        return res
```

## Rearranging Fruits

Trick is also can swap 2x with the minimum number, in addition to just swap from basket1 to basket2.

if need to swap `basket1[i]` <--> `basket2[j]`, compare both `min(basket1[i], basket1[j])` & `2 x min(all numbers in basket1/2)`.

### Time

`O(nlogn)`

### Space

`O(n)`

```python
class Solution:
    def minCost(self, basket1: List[int], basket2: List[int]) -> int:
        count = {}

        for n in basket1:
            if n in count:
                count[n][0] += 1
                count[n][2] += 1
            else:
                # [basket1, basket2, total]
                count[n] = [1, 0, 1]

        for n in basket2:
            if n in count:
                count[n][1] += 1
                count[n][2] += 1
            else:
                # [basket1, basket2, total]
                count[n] = [0, 1, 1]

        trans_b1 = []
        trans_b2 = []

        for n in count.keys():
            total = count[n][2]
            if total % 2 != 0:
                return -1

            per_basket = total//2
            if count[n][0] > per_basket:
                # basket1 too many of this item
                trans_b1 += [n] * (count[n][0] - per_basket)
            elif count[n][1] > per_basket:
                # basket2 too many of this item
                trans_b2 += [n] * (count[n][1] - per_basket)

        trans_b1.sort()
        trans_b2.sort()

        res = 0
        b1_indx = 0
        b2_indx = 0
        min_num = min(list(count.keys())) * 2

        for _ in range(len(trans_b1)):
            if min_num < trans_b1[b1_indx] and min_num < trans_b2[b2_indx]:
                res += min_num
            elif trans_b1[b1_indx] < trans_b2[b2_indx]:
                res += trans_b1[b1_indx]
                b1_indx += 1
            else:
                res += trans_b2[b2_indx]
                b2_indx += 1

        return res
```

## Maximum Fruits Harvested After at Most K Steps

Check move 1 way(left or right only) how much max

Check move left+right or right+left max how much

Optimise using a prefix sum array

### Time

`O(n + k)`

### Space

`O(k)`

```python
class Solution:
    def maxTotalFruits(self, fruits: List[List[int]], startPos: int, k: int) -> int:
        res = 0

        left_lst = [0] * k
        right_lst = [0] * k

        left_limit = startPos - k
        right_limit = startPos + k

        for fruit in fruits:
            curr_pos = fruit[0]
            curr_count = fruit[1]

            if curr_pos == startPos:
                # startPos already has fruits
                res += curr_count
            elif curr_pos >= left_limit and curr_pos < startPos:
                # left side
                left_index = startPos - curr_pos - 1
                left_lst[left_index] = curr_count
            elif curr_pos <= right_limit and curr_pos > startPos:
                # right side
                right_index = curr_pos - startPos - 1
                right_lst[right_index] = curr_count
            elif curr_pos > right_limit:
                break

        # add the prefix sum
        for i in range(1, k):
            left_lst[i] = left_lst[i] + left_lst[i - 1]
            right_lst[i] = right_lst[i] + right_lst[i - 1]

        # with movement
        tmp = 0
        if k > 0:
            max_one_way = max(left_lst[-1], right_lst[-1])
            max_two_way = 0
            for i in range(1, k//2 + 1):
                # get the index positions
                index_one = i - 1
                index_two = k - (2 * i) - 1

                # go left then right
                left_count = left_lst[index_one] if index_one >= 0 else 0
                right_count = right_lst[index_two] if index_two >= 0 else 0
                left_right_count = left_count + right_count

                # go right then left
                left_count = left_lst[index_two] if index_two >= 0 else 0
                right_count = right_lst[index_one] if index_one >= 0 else 0
                right_left_count = left_count + right_count

                max_two_way = max(max_two_way, max(left_right_count, right_left_count))

            tmp = max(max_one_way, max_two_way)

        return res + tmp
```

## Fruit Into Baskets

Sliding window

### Time

`O(n)`

### Space

`O(1)`

Map is fixed size of 2.

```python
class Solution:
    def totalFruit(self, fruits: List[int]) -> int:
        left_index = 0

        # count the left & right items how many
        counter = {
            fruits[left_index]: 1
        }

        res = 1

        for i in range(1, len(fruits)):
            # shrink left if current fruit not inside & already got 2 fruits
            while fruits[i] not in counter and len(counter.keys()) > 1 and left_index < i:
                # reduce count
                counter[fruits[left_index]] -= 1

                # remove if the value is 0
                if counter[fruits[left_index]] == 0:
                    counter.pop(fruits[left_index])

                # move the left_index forward
                left_index += 1

            # add the current fruit
            current_count = counter.get(fruits[i], 0) + 1
            counter[fruits[i]] = current_count

            res = max(res, sum(counter.values()))
        return res
```

## Fruits Into Baskets II

### Time

`O(n^2)`

### Space

`O(n)`

If update input varibales, then can become `O(1)`

```python
class Solution:
    def numOfUnplacedFruits(self, fruits: List[int], baskets: List[int]) -> int:
        used_basket_index = set()

        res = len(fruits)

        for f in fruits:
            for i, b in enumerate(baskets):
                if i in used_basket_index:
                    continue

                if b >= f:
                    used_basket_index.add(i)
                    res -= 1
                    break

        return res
```

##
