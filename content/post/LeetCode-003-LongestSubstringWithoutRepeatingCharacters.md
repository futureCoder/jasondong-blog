---
title: "LeetCode-003-LongestSubstringWithoutRepeatingCharacters解题报告"
date: 2017-08-31T13:09:14+08:00
clearReading: false
categories:
- 分类
- 算法
tags:
- LeetCode
- LinkedList
---

#### 题目描述
Given a string, find the length of the **longest substring** without repeating characters.

Examples:

Given {{< hl-text red >}}"abcabcbb"{{< /hl-text >}}, the answer is {{< hl-text red >}}"abc"{{< /hl-text >}}, which the length is 3.

Given {{< hl-text red >}}"bbbbb"{{< /hl-text >}}, the answer is {{< hl-text red >}}"b"{{< /hl-text >}}, with the length of 1.

Given {{< hl-text red >}}"pwwkew"{{< /hl-text >}}, the answer is {{< hl-text red >}}"wke"{{< /hl-text >}}, with the length of 3. Note that the answer must be a **substring**, {{< hl-text red >}}"pwke"{{< /hl-text >}} is a {{< hl-text cyan >}}subsequence{{< /hl-text >}} and not a {{< hl-text cyan >}}substring{{< /hl-text >}}.<br>


#### 思路
如果子串含有重复字符，那么父串一定含有重复字符。单个子问题就可以决定父问题，可以使用贪心法。和动态规划法不同，动态规划中，单个子问题只能影响父问题，不足以决定父问题。
从左到右遍历，遍历时将当前字符及所在位置存入一个map中，若当前字符已出现过，那么接下来从上次出现该字符的下一个位置开始遍历。每趟遍历之后根据当前字符串长度```len```更新最大长度```max(ret)```。<p>
{{< alert info >}}复杂度：最好情况下，无重复字符，只需一趟遍历，复杂度为O(n)；最坏情况下，整个字符串只有一种字符，遍历过程中需要不断拉回，但这样每个元素也只访问2次，复杂度还是O(n)；插入map的复杂度为O(log<sub>n</sub>)，最终，时间复杂度为O(log<sub>n</sub>) {{< /alert >}}
###### AC代码 Version.1
{{< codeblock "AddTwoNumbers.cpp" >}}
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        int ret = 0, len = 0;
        std::map<char, int> appearMap;
        for(int idx = 0; idx < s.length(); ++idx)
        {
            auto iter = appearMap.find(s[idx]);
            if(iter == appearMap.end())
            {
                //has not appear
                ++len;
                appearMap.insert(std::make_pair(s[idx], idx));
            }
            else
            {
                idx = iter->second;
                appearMap.clear();
                len = 0;
            }
            ret = std::max(ret, len);
        }
        return ret;
    }
};
{{< /codeblock >}}

#### 更进一步
若我们在初始时就分配一个可以穷举所有字符的数组作为hash，hash中存的是字符上一次出现的位置，初始为-1。设置一个当前搜索起始位置```index = -1```，在遍历时，根据当前搜索到的位置及当前搜索起始位置index更新子串的最大长度，若当前字符已出现过，则以其上一次出现的位置作为当前搜索的起始位置，更新index而不用拉回重新搜索。<p>
{{< alert info >}}复杂度：由于没有map操作，时间复杂度为O(n)，空间复杂度O(1)，只开了一个固定大小的数组，一般情况下影响不大。{{< /alert >}}
###### AC代码 Version.2
{{< codeblock "AddTwoNumbers.cpp" >}}
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        int alphabet[256];
        memset(alphabet, -1, sizeof(alphabet));
        int index = -1, ret = 0;
        for(int i = 0; i < s.size(); ++i)
        {
        	if(alphabet[s[i]] > index)
        	{
        		index = alphabet[s[i]];
        	}
        	if(ret < i - index)
        	{
        		ret = i - index;
        	}
        	alphabet[s[i]] = i;
        }
        return ret;
    }
};
{{< /codeblock >}}