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


#### AC代码 Version.1
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


#### AC代码 Version.2
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