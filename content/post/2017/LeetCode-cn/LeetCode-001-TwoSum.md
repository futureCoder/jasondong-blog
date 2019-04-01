---
title: "LeetCode-001-Two Sum解题报告"
date: 2017-08-28T16:33:57+08:00
clearReading: false
draft: true
categories:
- 分类
- 算法
tags:
- C++
- LeetCode
- Map
---

#### 题目描述
> Given an array of integers, return indices of the two numbers such that they add up to a specific target.
You may assume that {{< hl-text cyan >}}each input would have exactly one solution {{< /hl-text >}} , and you may not use the same element twice.<br>
**Example:**
Given nums = [2, 7, 11, 15], target = 9, <br>
Because nums[0] + nums[1] = 2 + 7 = 9, <br>
return [0, 1].

#### 思路
暂存目前已经遍历过的元素，在遍历剩余元素时判断 target-当前元素 是否已存在，如存在则返回。由于返回值是元素下标，同时需要暂存元素值，那么则需要一个map，键为元素值，值为元素下标。<p>
{{< alert info >}}复杂度：遍历一次的复杂度为O(n)，插入map的复杂度为O(log<sub>n</sub>)，总体的时间复杂度为O(log<sub>n</sub>) {{< /alert >}}

###### AC代码
{{< codeblock "TwoSum.cpp" >}}
vector<int> twoSum(vector<int>& nums, int target) {
    std::map<int, int> neededNumMap;
    std::vector<int> ret;
    for(int i = 0; i < nums.size(); ++i)
    {
        auto iter = neededNumMap.find(target - nums[i]);
        if(iter == neededNumMap.end())
        {
            neededNumMap.insert(std::make_pair(nums[i], i));
        }
        else
        {
            ret.push_back(iter->second);
            ret.push_back(i);
            return ret;
        }
    }
    return ret;
}
{{< /codeblock >}}

#### 其他
在提交时居然给了个编译错误: {{< hl-text red >}}no matching function for call to make_pair(__gnu_cxx::__alloc_traits >::value_type&, int&) {{< /hl-text >}}, 原因是之前的函数用错了： ```std::make_pair<int, int>(nums[i], i)``` ， 下面是C++标准库的用法：
##### Convenience Function make_pair() 
The ```make_pair()``` function template enables you to create a value pair without writing the types
explicitly. For example, instead of

```C++
std::pair<int,char>(42,’@’)
```

you can write the following:

```C++
std::make_pair(42,’@’)
```

Before C++11, the function was simply declared and defined as follows:

```C++
namespace std {
    // create value pair only by providing the values
    template <template T1, template T2>
    pair<T1,T2> make_pair (const T1& x, const T2& y) {
        return pair<T1,T2>(x,y);
    }
}
```

However, since C++11, things have become more complicated because this class also deals with {{< hl-text cyan >}} move semantics {{< /hl-text >}} in a useful way. So, since C++11, the C++ standard library states that ```make_pair()``` is declared as:

```C++
namespace std {
    // create value pair only by providing the values
    template <template T1, template T2>
    pair<V1,V2> make_pair (T1&& x, T2&& y);
}
```

where the details of the returned values and their types V1 and V2 depend on the types of x and y.
Without going into details, the standard now specifies that ```make_pair()``` uses {{< hl-text cyan >}} move semantics {{< /hl-text >}} if possible and copy semantics otherwise. In addition, it “decays” the arguments so that the expression ```make_pair("a","xy")``` yields a pair<const char*,const char*> instead of a ```pair<const char[2],const char[3]>```.
The make_pair() function makes it convenient to pass two values of a pair directly to a function that requires a pair as its argument. Consider the following example:

```C++
void f(std::pair<int,const char*>);
void g(std::pair<const int,std::string>);
```

As the example shows, make_pair() works even when the types do not match exactly, because the template constructor provides implicit type conversion. When you program by using maps or multimaps, you often need this ability. Note that since C++11, you can, alternatively, use initializer lists:

```C++
f({42,"empty"}); // pass two values as pair
g({42,"chair"}); // pass two values as pair with type conversions
```

However, an expression that has the explicit type description has an advantage because the resulting type of the pair is not derived from the values. For example, the expression ```std::pair<int,float>(42,7.77)``` does not yield the same as ```std::make_pair(42,7.77)```
The latter creates a pair that has double as the type for the second value (unqualified floating literals have type double). The exact type may be important when overloaded functions or templates are used. These functions or templates might, for example, provide versions for both float and double to improve efficiency.
With the new semantic of C++11, you can influence the type make_pair() yields by forcing either {{< hl-text cyan >}} move or reference semantics {{< /hl-text >}}. For {{< hl-text cyan >}} move semantics {{< /hl-text >}}, you simply use ```std::move()``` to declare
that the passed argument is no longer used:

```C++
std::string s, t;
...
auto p = std::make_pair(std::move(s),std::move(t));
... // s and t are no longer used
```

To force reference semantics, you have to use ref(), which forces a reference type, or ```cref()```, which forces a constant reference type (both provided by <functional>
For example, in the following statements, a pair refers to an int twice so that, finally, i has the value 2:

```C++
#include <utility>
#include <functional>
#include <iostream>
int i = 0;
auto p = std::make_pair(std::ref(i),std::ref(i)); // creates pair<int&,int&>
++p.first; // increments i
++p.second; // increments i again
std::cout << "i: " << i << std::endl; // prints i: 2
```
Since C++11, you can also use the tie() interface, defined in <tuple>, to extract values out of a pair:
```C++
#include <utility>
#include <tuple>
#include <iostream>
std::pair<char,char> p=std::make_pair(’x’,’y’); // pair of two chars
char c;
std::tie(std::ignore,c) = p; // extract second value into c (ignore first one)
```

In fact, here the pair p is assigned to a tuple, where the second value is a reference to c.

