---
title: "LeetCode-018-4Sum解题报告"
date: 2017-09-27T10:19:38+08:00
clearReading: false
categories:
- 分类
- 算法
tags:
- LeetCode
---

#### 题目描述
> Given an array **S** of *n* **integers**, are there elements a, b, c, and d in **S** such that a + b + c + d = target? Find all unique quadruplets in the array which gives the sum of target.<br>
<p>**Note:** The solution set must not contain duplicate quadruplets.</p>
For example, given array S = [1, 0, -1, 0, -2, 2], and target = 0.<br>
A solution set is:
```
[
  [-1,  0, 0, 1],
  [-2, -1, 1, 2],
  [-2,  0, 0, 2]
]
```

#### 思路
同 3Sum 。

###### AC代码
{{< codeblock "4Sum.cpp" >}}
    vector<vector<int>> fourSum(vector<int>& nums, int target) {
        vector<vector<int>> ret;
        if (nums.size() < 4)
            return ret;
        sort(nums.begin(), nums.end());
        auto end = nums.end();
        for(auto i = nums.begin(); i < end - 3; ++i)
        {
            while(i != nums.begin() && i < end - 3 && *i == *(i - 1)) ++i;
            for(auto j = i + 1; j < end - 2; ++j)
            {
                while(j != i + 1 && j < end - 2 && *j == *(j - 1)) ++j;
                auto k = j + 1, l = end - 1;
                while(k < l)
                {
                    if(*i + *j + *k + *l == target)
                    {
                        ret.push_back({*i, *j, *k, *l});
                        ++k,--l;
                        while(k < l && *k == *(k - 1)) ++k;
                        while(k < l && *l == *(l + 1)) --l;
                    }
                    else if(*i + *j + *k + *l < target)
                    {
                        ++k;
                        while(k < l && *k == *(k - 1)) ++k;
                    }
                    else
                    {
                        --l;
                        while(k < l && *l == *(l + 1)) --l;
                    }
                }

            }
        }
        return ret;
    }
{{< /codeblock >}}
