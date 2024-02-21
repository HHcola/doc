## 两数之和
给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

 

示例 1：

输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
示例 2：

输入：nums = [3,2,4], target = 6
输出：[1,2]
示例 3：

输入：nums = [3,3], target = 6
输出：[0,1]
 

提示：

2 <= nums.length <= 104
-109 <= nums[i] <= 109
-109 <= target <= 109
只会存在一个有效答案
 

进阶：你可以想出一个时间复杂度小于 O(n2) 的算法吗？


这道题考察了对于列表的求和计算方法，普通方法是2个for循环计算，时间复杂度比较高O（n^2），时间复杂度比较高，可以考虑使用map的方式存储，通过差值做Key，一次遍历寻找对应的序列值；

### 暴力解法

```
class Solution {
    public int[] twoSum(int[] nums, int target) {
        int[] ret = new int[2];
        for(int i = 0; i < nums.length; i ++) {
            int value = nums[i];
            if (target < value) {
                continue;
            }
            
            int index = findIndex(nums, i + 1, target - value);
            if (index > 0) {
                ret[0] = i;
                ret[1] = index;
            }
        }
        return ret;
    }

    private int findIndex(int[] nums, int startIndex, int target) {
        for(int i = startIndex; i < nums.length; i ++) {
            int value = nums[i];
            if (value == target) {
                return i;
            }
        }
        return -1;
    }
}
```

### Map解法

通过差值作为key值，通过key值匹配，寻找对应的序列

```
class Solution {
    public int[] twoSum(int[] nums, int target) {
        HashMap<Integer, Integer> map = new HashMap<>();
        for(int i = 0; i < nums.length; i ++) {
            if (map.containsKey(nums[i])) {
                return new int[]{map.get(nums[i]), i};
            }
            map.put(target - nums[i], i);
        }

        return null;
    }

}
```

这种方式计算的时间复杂度比较低，遍历一次O(n)，使用差值做Key去寻找对应的值是一种思路。
