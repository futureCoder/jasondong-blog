---
title: "LeetCode-002-Add Two Numbers解题报告"
date: 2017-08-30T22:27:59+08:00
clearReading: false
categories:
- 分类
- 算法
tags:
- LeetCode
- LinkedList
---

#### 题目描述
> You are given two {{< hl-text cyan >}}non-empty{{< /hl-text >}} linked lists representing two non-negative integers. The digits are stored in reverse order and each of their nodes contain a single digit. Add the two numbers and return it as a linked list.
You may assume the two numbers do not contain any leading zero, except the number 0 itself.<br>
**Input**: (2 -> 4 -> 3) + (5 -> 6 -> 4)<br>
**Output**: 7 -> 0 -> 8<br>

#### 思路
链表没有前导0，所以不需要考虑特殊情况。按位加就好了，纯考代码实现。接口中给出的是两个链表的头结点地址，所以实现上考虑直接在实参对象链表```l1```上进行修改，然后返回```l1```，若```l1```的长度较短,则将```l2```中多出的部分接到```l1```的尾结点上。需要注意，在这么干了之后，```l1```和```l2```会成为带有公共结点的链表，所以代码中额外定义了一个```pre```指针，用来修改这种情况。实际工程可能不推荐这么做或者视情况而定，这里仅是为了减少分配额外空间。
{{< alert info >}}复杂度：O(n) {{< /alert >}}
###### AC代码 Version.1
{{< codeblock "AddTwoNumbers.cpp" >}}
ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    if(nullptr == l1)
        return l2;
    if(nullptr == l2)
        return l1;
    ListNode *ret = l1, *pre1;
    ListNode *pre2;     //防止函数返回后，析构 l2 时出问题
    int cf = 0;
    while(l1 && l2)
    {
        l1->val += l2->val + cf;
        cf = l1->val / 10;
        l1->val %= 10;
        pre1 = l1;
        pre2 = l2;
        l1 = l1->next;
        l2 = l2->next;
    }
    if(nullptr == l1)
    {
        pre1->next = l2;
        pre2->next = nullptr;   //pre2的作用到此为止
        l1 = l2;
    }
    while(l1 && cf)
    {
        l1->val += cf;
        cf = l1->val / 10;
        l1->val %= 10;
        pre1 = l1;
        l1 = l1->next;
    }
    if(cf)
    {
        ListNode* newNode = new ListNode(cf);
        pre1->next = newNode;
    }
    return ret;
}
{{< /codeblock >}}

也可以考虑用二级指针，指向链表节点的地址，可以直接修改指针域，不过也就是少定义了两个变量，但换来的是每次访问链表元素都要“间接寻址”，多一次解引用。<p>
{{< alert info >}}复杂度：O(n) {{< /alert >}}
###### AC代码 Version.2
{{< codeblock "AddTwoNumbers.cpp" >}}
ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    if(nullptr == l1)
        return l2;
    if(nullptr == l2)
        return l1;
    ListNode *ret = l1, **ppL1 = &l1, **ppL2 = &l2;
    int cf = 0;
    while( *ppL1 && *ppL2 )
    {
        (*ppL1)->val += (*ppL2)->val + cf;
        cf = (*ppL1)->val / 10;
        (*ppL1)->val %= 10;
        ppL1 = &((*ppL1)->next);
        ppL2 = &((*ppL2)->next);
    }
    if(nullptr == *ppL1)
    {
        *ppL1 = *ppL2;
        *ppL2 = nullptr;
    }
    while(*ppL1 && cf)
    {
        (*ppL1)->val += cf;
        cf = (*ppL1)->val / 10;
        (*ppL1)->val %= 10;
        ppL1 = &((*ppL1)->next);
    }
    if(cf)
    {
        ListNode* newNode = new ListNode(cf);
        *ppL1 = newNode;
    }
    return ret;
}
{{< /codeblock >}}