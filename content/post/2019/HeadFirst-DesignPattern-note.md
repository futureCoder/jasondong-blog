---
title: "HeadFirst设计模式-读书笔记"
date: 2018-01-06T16:15:48+08:00
hierarchicalCategories: true
categories: 
- 分类
- 读书笔记
---
## 第一章 策略模式
> 定义算法簇，分别封装起来，让它们之间可以相互替换，此模式让算法的**变化**独立于使用算法的客户.  

最初有一个Duck基类，每个鸭子子类继承Duck并添加自己的实现。Duck大概是这样
```
public class Duck
{
    quack();
    swim();
    abstract display();
}
```
如果现在我们想让鸭子能飞，那么一个自然的思路就是给Duck基类中加上fly()方法，成为下边这样
```
public class Duck
{
    quack();
    swim();
    abstract display();
    fly();      //新加的
}
```
但这样带来的问题是，**如果在基类中实现了fly()，将导致所有的鸭子子类都具备fly功能(包括橡皮鸭子)，**
## 第二章 有意义的命名
1. 名副其实。
   * 选个好名字要花时间，但省下来的时间比花掉的多。
   * 如果名称需要注释来补充，那就不算是**名副其实**。好代码 > 坏代码+好注释。
2. 避免误导。应避免使用与本意相悖的词。
   * 不要使用不同之处较小的名称。如 InformationToSet, InformationToSend ，这样会增加区分它们的时间成本。此外，在使用IDE的快捷补全功能时，可能会使用相似但错误的变量名。如果有两个类型为String 的变量，一个为 XYZControlllerForEffcientHandlingOfStrings 另一个为XYZControllerForEffectientStorageOfStrings ，在使用补全功能时，一不小心就可能使用错误的变量名而导致错误。
   * 以同样的方式拼写同样的概念。也就是说，同一个概念如果前后使用不一样的名字，就会产生误导的反效果。
   * 最应该避免的是使用小写字母l和大写字母O作为变量名，它们与数字1和0极难区分。
3. 做有意义的区分。

