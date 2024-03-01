## 接雨水

给定 n 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

 

示例 1：

![接雨水](picture/rain.png)

输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
输出：6
解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 
示例 2：

输入：height = [4,2,0,3,2,5]
输出：9
 

提示：

n == height.length
1 <= n <= 2 * 104
0 <= height[i] <= 105

### 解题思路
接雨水不是取决于最高的高度，而是取决于矮的高度，使用双指针的方式，左右两边去比较，用最小的高度计算节水量


### 代码

```
class Solution {
    public int trap(int[] height) {
    	int left = 0;
    	int right = height.length - 1;
    	int leftHight = 0;
    	int rightHight = 0;
    	int res = 0;
    	while(left < right) {
    		leftHight = Math.max(leftHight, height[left]);
    		rightHight = Math.max(rightHight, height[right]);
    		if (leftHight > rightHight) {
    				res += rightHight - height[right];
    				right --;
    			} else {
    				res += leftHight - height[left];
    				left ++;
    			}
    	}
    	return res;
    }
}
```