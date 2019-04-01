---
title: "LeetCode-015-3Sum解题报告"
date: 2017-09-18T13:35:57+08:00
clearReading: false
draft: true
categories:
- 分类
- 算法
tags:
- LeetCode
---

#### 题目描述
> <p>Given an array **S** of **n integers**, are there elements a, b, c in S such that ```a + b + c = 0```? Find all unique triplets in the array which gives the sum of zero.</p>
<p>**Note**: The solution set must not contain duplicate triplets.</p>
For example, given array ```S = [-1, 0, 1, 2, -1, -4]```,
A solution set is:
```[
  [-1, 0, 1],
  [-1, -1, 2]
]```

#### 思路
两侧夹逼，迭代过程中注意去重。

###### AC代码
{{< codeblock "3Sum.cpp" >}}
vector<vector<int>> threeSum(vector<int>& nums) {
    vector<vector<int>> ret;
    if (nums.size() < 3)
        return ret;
    sort(nums.begin(), nums.end());
    const int target = 0;
    auto last = nums.end();
    for (auto i = nums.begin(); i < last - 1; ++i)
    {
        if (i != nums.begin() && *i == *(i - 1))
            continue;
        auto j = i + 1, k = last - 1;
        while( j < k)
        {
            if ( *i + *j + *k == target )
            {
                ret.push_back({ *i, *j, *k });
                ++j;
                while (*j == *(j - 1) && j < k) ++j;
                --k;
                while (*k == *(k + 1) && j < k) --k;
            }
            else if ( *i + *j + *k < target )
            {
                ++j;
                while ( *j == *(j - 1) && j < k ) ++j;
            }
            else
            {
                --k;
                while ( *k == *(k + 1) && j < k ) --k;
            }
        }
    }
    return ret;
}
{{< /codeblock >}}
