# Two Sum

## 题目

Given an array of integers, return **indices** of the two numbers such that they add up to a specific target.

You may assume that each input would have ***exactly*** one solution, and you may not use the same element twice.

**Example:**

```text
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

## 1. Brute Force

```kotlin
fun twoSum(nums: IntArray, target: Int): IntArray {

    for (i in 0..nums.lastIndex) {
        for (j in i + 1..nums.lastIndex) {
            if (nums[i] + nums[j] == target) {
                return intArrayOf(i, j)
            }
        }
    }

    return intArrayOf(-1, -1)
}
```

穷举数组的所有组合，找出结果，时间复杂度 $O(N^2)$，空间复杂度 $O(1)$

## 2. HashMap

考虑一种特殊情况，如果给定的数组是有序的，可以使用双指针方法获取到 index，那么这里也可以对数组进行排序后使用双指针来获取 index：

```kotlin
// Wrong Answer
fun twoSum(nums: IntArray, target: Int): IntArray {
    nums.sort()
    var x = 0;
    var y = nums.lastIndex

    while (x < y) {
        when {
            nums[x] + nums[y] < target -> x += 1
            nums[x] + nums[y] > target -> y -= 1
            else -> return intArrayOf(x, y)
        }
    }

    return intArrayOf(-1, -1)
}

Wrong Answer:
Input:
[3,2,4]
6
Output:
[0,2]
Expected:
[1,2]
```

将无序数组排序后可以通过双指针方法获取到 `nums[x] + nums[y] = target`，但经过排序数组已经不是原来的数组，x 与 y 已经不是原始数组中的下标。

是否可以将原始数组的值与对应下表保存起来，最后通过 nums[x] 和 nums[y] 获取到下标：

```kotlin
// Wrong Answer
fun twoSum(nums: IntArray, target: Int): IntArray {
    val indexMap = HashMap<Int, Int>()
    for (i in 0..nums.lastIndex) {
        indexMap[nums[i]] = i
    }

    nums.sort()

    var x = 0;
    var y = nums.lastIndex

    while (x < y) {
        when {
            nums[x] + nums[y] < target -> x += 1
            nums[x] + nums[y] > target -> y -= 1
            else -> return intArrayOf(indexMap[nums[x]]!!, indexMap[nums[y]]!!)
        }
    }

    return intArrayOf(-1, -1)
}

Wrong Answer:
Input:
[3,3]
6
Output:
[1,1]
Expected:
[0,1]
```

当数组中存在相同元素时前一个元素的 index 会被后一个元素的 index 刷新，通过 haspMap 查询到的是两个相同的 index

再次换一个思路，此题目也可以解释成给定两个数 x 和 target，求数组中 target - x 的位置：

```kotlin
fun twoSum(nums: IntArray, target: Int): IntArray {
    val indexMap = HashMap<Int, Int>()
    // 保存 index
    for (i in 0..nums.lastIndex) {
        indexMap[nums[i]] = i
    }

    for (i in 0..nums.lastIndex) {
        // 查询 index
        val other = target - nums[i]
        if (indexMap.containsKey(other) && indexMap[other] != i) {
            return intArrayOf(i, indexMap[other]!!)
        }
    }

    return intArrayOf(-1, -1)
}

// 优化版本
fun twoSum(nums: IntArray, target: Int): IntArray {
    val indexMap = HashMap<Int, Int>()

    nums.forEachIndexed { index, num ->
        val other = target - num
        if (indexMap.contains(other)) {
            return intArrayOf(indexMap[other]!!, index)
        } else {
            indexMap[num] = index
        }
    }

    return intArrayOf(-1, -1)
}
```
