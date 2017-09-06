---
title: "LeetCode-005-Longest Palindromic Substring解题报告"
date: 2017-09-05T18:55:22+08:00
clearReading: false
categories:
- 分类
- 算法
tags:
- LeetCode
- Manacher Algorithm
- String 
- Palindromic
---

#### 题目描述
> <p>Given a string **s**, find the longest palindromic substring in **s**. You may assume that the maximum length of **s** is 1000.</p>
**Example:**
```
Input: "babad"
Output: "bab"
Note: "aba" is also a valid answer.
```
**Example:**
```
Input: "cbbd"
Output: "bb"
```

#### 思路
Manacher(马拉车)算法，详细后续更新，先上代码。

###### AC代码 Version.1 非常乱 凑合看
{{< codeblock "AddTwoNumbers.cpp" >}}
string longestPalindrome(string s) {
    if (s.size() < 2)
        return s;
    auto GetAddBounadriesStr = [](const string& str) -> std::string {
        std::string ret;
        for (const auto& i : str)
        {
            ret.push_back('#');
            ret.push_back(i);
        }
        ret.push_back('#');
            return ret;
    };
    auto GetRemoveBounadriesStr = [](const string& str) -> std::string {
        std::string ret;
        for (int i = 0; i < str.size(); ++i) {
            ++i;
            ret.push_back(str[i]);
        }
        return ret;
    };
    std::string str = std::move(GetAddBounadriesStr(s));
    std::vector<int> p(str.size(), 1);
    int id = 0, max = 0;
    for (int i = 0; i < str.size(); ++i)
    {
        if (i < max)
        {
            int j = 2 * id - i;
                if (p[j] + i < max)
                {
                    p[i] = p[j];
                    continue;
                }
        }
        int k = i + 1;
        while (k < str.size() && (2 * i - k) >= 0 && str[k] == str[2 * i - k])
        {
            ++p[i];
            ++k;
        }
        if (p[i] > p[id])
        {
            id = i;
            max = i + p[i];
        }
    }
    return GetRemoveBounadriesStr(str.substr(id - p[id] + 1, 2 * p[id] - 1));
}
{{< /codeblock >}}
