---
title: "LeetCode-004-Median of Two Sorted Arrays解题报告"
date: 2017-09-04T20:43:06+08:00
clearReading: false
draft: true
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
首先划重点，{{< hl-text cyan >}} 两数组有序 {{< /hl-text >}}、{{< hl-text orange >}} 复杂度须为O(log (m+n)) {{< /hl-text >}}。刚看到题时，第一反应是将两数组合并为一个，然后排序取中位数，这样方法由于有排序存在，最低复杂度在(m+n)O(log (m+n))级别；或是不用合并排序，遍历两个数组同时数数，也可以取到中位数，复杂度是O(m+n)级别。但此题要求基本O(log (m+n))，也就是限制了这两种解法。而这时如果注意到{{< hl-text cyan >}} 数组有序 {{< /hl-text >}}的条件，再加上{{< hl-text orange >}} log {{< /hl-text >}}级别的时间复杂度，很容易想到**二分法（二分查找）**。常规的**二分法**是在单个有序数组上实现的，但思想是通用的，这题最终也是用**二分法**的思想AC的，下面在详细介绍思路之前，首先复习一下**二分法及其思想**。
{{< blockquote "维基百科" "//zh.wikipedia.org/wiki/%E4%BA%8C%E5%88%86%E6%90%9C%E7%B4%A2%E7%AE%97%E6%B3%95" "二分搜索算法" >}}
在计算机科学中，二分搜索（英语：binary search），也称折半搜索（英语：half-interval search）、对数搜索（英语：logarithmic search），是一种在有序数组中查找某一特定元素的搜索算法。搜索过程从数组的中间元素开始，如果中间元素正好是要查找的元素，则搜索过程结束；如果某一特定元素大于或者小于中间元素，则在数组大于或小于中间元素的那一半中查找，而且跟开始一样从中间元素开始比较。如果在某一步骤数组为空，则代表找不到。
{{< /blockquote >}}

这种搜索算法每一次比较都使搜索范围缩小一半，以这个数组建立一个平衡二叉树，则在数组中的查找过程其实是和平衡二叉树中的查找过程是等价的。最坏情况下，查找的复杂度就是平衡二叉树的树高O(log n)。

| 属性           | 说明         |
| :------------: | ------------ |
| 数据结构       | 数组（有序） |
| 最坏时间复杂度 | O(log n)     |
| 最优时间复杂度 | O(1)         |
| 平均时间复杂度 | O(log n)     |
| 迭代空间复杂度 | O(1)         |
| 递归空间复杂度 | O(log n)     |
{{< wide-image src="http://otnjj3xqi.bkt.clouddn.com/image/blog/LeetCode-004-MedianOfTwoSortedArraysBinary_search_into_array.png" title="二分查找" >}}

回来说这题，这题其实可以转化为查找第k大的数，中位数只是一个特定的k。下面说如何在双数组中查找第k大的数。
二分法是一次删除掉当前搜索范围的一半，这一半是一定不会包含目标元素的。利用这个思想来看如何在双数组中查找第k大的数，首先我们假设给定数组A和B的元素个数都大于k/2，于是我们将A的第k/2个元素和B的第k/2个元素进行比较，会有以下三种情况：<br><br>
```A[k/2 - 1] == B[k/2 - 1]```<br>
```A[k/2 - 1] < B[k/2 - 1]```<br>
```A[k/2 - 1] > B[k/2 - 1]```<br><br>
当```A[k/2-1] == B[k/2-1]```时，表示已经找到了```A ∪ B```中第K大的数，直接返回；<br><br>
当```A[k/2-1] <  B[k/2-1]```时，表示A[k/2-1]不可能大于```A ∪ B```中第K大的数，那么由于数组有序，A[0]到A[k/2-1]这个范围的数都不可能大于```A ∪ B```中第K大的数，删除这个搜索范围，继续在剩余的搜索范围A[k/2]到A[end]和B中进行搜索。<br><br>
```A[k/2-1]  > B[k/2-1]```同理。<br>
###### AC代码 Version.1
{{< codeblock "findMedianSortedArrays.cpp" >}}
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

#### 顺便
查资料时，看到这么一段话

> 然《编程珠玑》的作者Jon Bentley曾在贝尔实验室做过一个实验，即给一些专业的程序员几个小时的时间，用任何一种语言编写二分查找程序（写出高级伪代码也可以），结果参与编写的一百多人中：90%的程序员写的程序中有bug（我并不认为没有bug的代码就正确）。
也就是说：在足够的时间内，只有大约10%的专业程序员可以把这个小程序写对。但写不对这个小程序的还不止这些人：而且高德纳在《计算机程序设计的艺术 第3卷 排序和查找》第6.2.1节的“历史与参考文献”部分指出，虽然早在1946年就有人将二分查找的方法公诸于世，但直到1962年才有人写出没有bug的二分查找程序。
你能正确无误的写出二分查找代码么？不妨一试，关闭所有网页，窗口，打开记事本，或者编辑器，或者直接在本文评论下，不参考上面我写的或其他任何人的程序，给自己十分钟到N个小时不等的时间，立即编写一个二分查找程序。

Excuse Me？ 90%程序员写不出无BUG的二分查找程序？我也试了试，照着10分钟来的，没测试，不知道对否，也不知是否属于90%

{{< codeblock "binarySearch.cpp" >}}
//找到返回数组中下标，找不到则返回-1
int binarySearch(const vector<int>& nums, int start, int end/*real end idx 's next*/, int target)
{
    if (start >= end)
        return -1;
    int mid = start + (end - start - 1) / 2;			/*real mid idx*/
    if (nums[mid] == target)
        return mid;
    else if (target < nums[mid])
        return binarySearch(nums, start, mid, target);
    else
        return binarySearch(nums, mid + 1, end, target);
}
int targetSearch(const vector<int>& nums, int target) {
    return binarySearch(nums, 0, nums.size(), target);
}
{{< /codeblock >}}