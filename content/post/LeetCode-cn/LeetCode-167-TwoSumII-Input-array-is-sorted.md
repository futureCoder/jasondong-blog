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
> Given an array of integers that is already sorted in ascending order, find two numbers such that they add up to a specific target number.<br>
The function twoSum should return indices of the two numbers such that they add up to the target, where index1 must be less than index2. Please note that your returned answers (both index1 and index2) are not zero-based.<br>
You may assume that each input would have exactly one solution and you may not use the same element twice.<br>
**Input:** numbers={2, 7, 11, 15}, target=9
**Output:** index1=1, index2=2

#### 思路
同 3Sum 。

###### AC代码 Version1
{{< codeblock "twoSum.cpp" >}}
    vector<int> twoSum(vector<int>& numbers, int target) {
        vector<int> ret;
        if(numbers.size() < 2)
            return ret;
        std::map<int, int> myMap;
        for(int i = 0; i < numbers.size(); ++i)
        {
            auto iter = myMap.find(target - numbers[i]);
            if( iter != myMap.end())
            {
                ret.push_back(iter->second + 1);
                ret.push_back(i + 1);
                return ret;
            }
            myMap.insert(std::make_pair(numbers[i], i));
        }
        return ret;
    }
{{< /codeblock >}}

###### AC代码 Version2
{{< codeblock "twoSum.cpp" >}}
    vector<int> twoSum(vector<int>& numbers, int target) {
        vector<int> ret;
        if(numbers.size() < 2)
            return ret;
        auto i = numbers.begin(), end = numbers.end();
        auto j = end - 1;
        while( i < j )
        {
            if( *i + *j < target )
            {
                ++i;
                while( i < j && *i == *(i - 1) ) ++i;
            }
            else if( *i + *j > target )
            {
                --j;
                while( i < j && *j == *(j + 1) ) --j;
            }
            else
            {
                ret.push_back(i - numbers.begin() + 1);
                ret.push_back(j - numbers.begin() + 1);
                return ret;
            }
        }
        return ret;
    }
{{< /codeblock >}}