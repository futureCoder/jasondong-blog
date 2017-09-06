---
title: "LeetCode-004-Median of Two Sorted Arrays解题报告"
date: 2017-09-04T20:43:06+08:00
clearReading: false
categories:
- 分类
- 算法
tags:
- LeetCode
- Binary Search
---

#### 题目描述
> <p>There are two sorted arrays **nums1** and **nums2** of size m and n respectively.
Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).</p>
**Example 1:**
```
nums1 = [1, 3]
nums2 = [2]
The median is 2.0
```
**Example 2:**
```
nums1 = [1, 2]
nums2 = [3, 4]
The median is (2 + 3)/2 = 2.5
```

#### 思路
首先划重点，{{< hl-text cyan >}} 两数组有序 {{< /hl-text >}}、{{< hl-text orange >}} 复杂度须为O(log (m+n)) {{< /hl-text >}}。刚看到题时，第一反应是将两数组合并为一个，然后排序取中位数，这样方法由于有排序存在，最低复杂度在(m+n)O(log (m+n))级别；或是不用合并排序，遍历两个数组同时数数，也可以取到中位数，复杂度是O(m+n)级别。但此题要求基本O(log (m+n))，也就是限制了这两种解法。而这时如果注意到{{< hl-text cyan >}} 数组有序 {{< /hl-text >}}的条件，再加上{{< hl-text orange >}} log {{< /hl-text >}}级别的时间复杂度，很容易想到**二分法**。常规的**二分法**是在单个有序数组上实现的，但思想是通用的，这题最终也是用**二分法**的思想AC的，下面在详细介绍思路之前，首先复习一下**二分法及其思想**。

###### AC代码 Version.1
{{< codeblock "AddTwoNumbers.cpp" >}}
double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
    int len1 = nums1.size(), len2 = nums2.size();
    int k = (len1 + len2) / 2 + 1;
    if((len1 + len2) & 0x1)
        return find_Kth_InSortedArrays(nums1.begin(), nums2.begin(), len1, len2, k);
    else
        return ( find_Kth_InSortedArrays(nums1.begin(), nums2.begin(), len1, len2, k - 1) 
                + find_Kth_InSortedArrays(nums1.begin(), nums2.begin(), len1, len2, k ) ) / 2.0;
}

int find_Kth_InSortedArrays(vector<int>::const_iterator pLeft, vector<int>::const_iterator pRight, int lLen, int rLen, int k)
{
    if(k < 1 || k > lLen + rLen)
        return -1;
    if(lLen > rLen)
        return find_Kth_InSortedArrays(pRight, pLeft, rLen, lLen, k);
    if(0 == lLen)
        return *(pRight + k - 1);
    if(1 == k)
        return std::min(*pLeft, *pRight);
    int lMid = std::min(k >> 1, lLen);
    int rMid = k - lMid;
    if(*(pLeft + lMid - 1) == *(pRight + rMid - 1))
        return *(pLeft + lMid - 1);
    else if(*(pLeft + lMid - 1) < *(pRight + rMid - 1))
        return find_Kth_InSortedArrays(pLeft + lMid, pRight, lLen - lMid, rLen, k - lMid);
    else 
        return find_Kth_InSortedArrays(pLeft, pRight + rMid, lLen, rLen - rMid, k - rMid);
}
{{< /codeblock >}}
