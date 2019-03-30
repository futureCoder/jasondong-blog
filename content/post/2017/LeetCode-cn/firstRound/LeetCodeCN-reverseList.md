---
title: "反转链表解题报告"
date: 2018-09-12T19:58:55+08:00
clearReading: false
draft: true
categories:
- 分类
- 算法
tags:
- C++
- LeetCodeCN
- LinkedList
---

#### 题目描述
> 反转一个单链表。  
**示例:**
输入: 1->2->3->4->5->NULL  
输出: 5->4->3->2->1->NULL  
**进阶:**
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？

#### 思路
- 迭代方法--使用头插法重新构建单链表
    1. 新建一个指针ret作为新链表的起始结点，初始值为nullptr(默认肯定是null)；
    2. 摘下链表head的首结点作为待插入结点，若head为空则停止并返回ret；
    3. 使用头插法将摘下的结点插入到新链表中；
    4. 重复2,3步骤。
- 递归方法--同样使用头插法思路
    1. 递归终止条件为输入链表为nullptr，；
    2. 在函数体内摘下输入链表的头结点，并将头结点的next域指向上一个结点；

#### AC代码
{{< codeblock "反转链表.cpp" >}}
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        return reverseList_Recusive(head);
    }
    ListNode* reverseList_Recusive(ListNode* head) {
        return reverseList_Recusive(nullptr, head);
    }

    ListNode* reverseList_Recusive(ListNode* pre, ListNode* cur) {
        if(nullptr == cur)
            return pre;
        ListNode* next = cur->next;
        cur->next = pre;
        return reverseList_Recusive(cur, next);
    }

    ListNode* reverseList_Iterative_HeadInsert(ListNode* head) {
        ListNode* ret = nullptr;
        while(nullptr != head)
        {
            ListNode* next = head->next;
            head->next = ret;
            ret = head;
            head = next;
        }
        return ret;
    }
};
{{< /codeblock >}}
