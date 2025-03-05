---
date : '2024-11-30T20:43:12+08:00'
draft : false
title : 'Leetcode 周赛 424'
tags : ["Leetcode"]
categories: ["Algorithm", "Leetcode"]
---

### 第一题: [Make Array Elements Equal to Zero](https://leetcode.com/problems/make-array-elements-equal-to-zero/)
解析：
首先如果这个数组是一个有效的数组，那么肯定0元素左边的所有元素的和和0元素右边所有元素的和的绝对差<=1  
比如，下面这个有效数组
```bash 
[1,0,2,0,3]
index = 3
leftSum = 3, rightSum = 3
```
所以是有效数组,而下面这个数组
```bash
[2,3,4,0,4,1,0]
无论选中哪个0元素，左右两边的和均不相等，所以肯定不是有效数组
```
答案如下：时间复杂度 O(n)
```Go {linenos=true}
int countValidSelections(vector<int>& nums) {
    std::vector<int> ps(nums.size() + 1);
    // 高级写法：计算向量和
    partial_sum(nums.begin(), nums.end(), ps.begin() + 1);      // 后撤一位,这样i元素之前所有的元素和就是ps[i]
    int res = 0;
    for(int i = 0; i < nums.size(); i++) {
        if(nums[i] == 0) {
            if(ps.back() == 2 * ps[i]) res += 2;                // 相等则左右两个方向移动均可
            else if(abs(ps.back() - 2 * ps[i]) == 1) res += 1;  // 相差一则只能一个方向移动
        }
    }
    return res;
}
```

### 第二题: [Zero Array Transformation I](https://leetcode.com/problems/zero-array-transformation-i/description/)
解析：
反向思考，只要记录下 queries 区间内的元素间隔，叠加这个间隔之间的元素的值，判断是否大于原始数组即可
比如
```bash
nums = [4,3,2,1], queries = [[1,3],[0,2]]
则刚开始是 [0,0,0,0]
query = [1,3] ==>  [0,1,1,1]
query = [0,2] ==>  [1,2,2,1]
明显不相等，所以不能构成 Zero Array
```
答案如下：时间复杂度O(n)
```Go {linenos=true}
bool isZeroArray(vector<int>& nums, vector<vector<int>>& queries) {
    int n = nums.size();
    std::vector<int> freq(n + 1, 0);
    for(auto i : queries) {
        freq[i[0]]++;                       // 只记录区间初始值即可
        freq[i[1]+1]--;                     // 区间之后减1
    }
    
    int op = 0;
    for(int i = 0; i < nums.size(); i++) {
        op += freq[i];                      // 累加区间值
        if(op < nums[i]) return false;
    }
    return true;
}
```

### 第三题: [Zero Array Transformation II](https://leetcode.com/problems/zero-array-transformation-ii/)
上面第二题的升级版本，这里query范围内降低的数值最大不能超过给定的值，要求返回使得数组变成 Zero-Array 所需操作的最小次数
解析：
还是按照上面的思路，这里需要遍历每个query参数来判断依次判断与nums[i]的差是多少
对每个 index:i，我们先计算 sum 和
检查是否 sum >= nums[i]，如果不满足，则继续遍历 query
答案如下：时间复杂度 O(n)
```Go {linenos=true}
int minZeroArray(vector<int>& nums, vector<vector<int>>& queries) {
    int n = nums.size(), sum = 0, k = 0;
    std::vector<int> cnt(n + 1, 0);
    for(int i = 0; i < n; i++) {
        while(sum + cnt[i] < nums[i]) {
            if(k == queries.size()) return -1;         // 遍历所有不满足直接返回-1
            int l = queries[k][0], r = queries[k][1], val = queries[k][2];
            k++;
            if(r < i) continue;                        // 不一定query都是有序的
            cnt[max(l,i)] += val;                      // 更新最大值
            cnt[r+1] -= val;
        }
        sum += cnt[i];
    }
    return k;
}
```