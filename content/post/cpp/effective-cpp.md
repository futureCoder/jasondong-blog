---
title: "Effect C++笔记"
date: 2017-08-11T13:05:21+08:00
draft: true
thumbnailImagePosition: left
thumbnailImage: //otnjj3xqi.bkt.clouddn.com/image/blog/C++%E6%93%8D%E4%BD%9CRedis%E5%AE%9E%E7%8E%B0%E5%BC%82%E6%AD%A5%E8%AE%A2%E9%98%85%E5%92%8C%E5%8F%91%E5%B8%83city-750.jpg
coverImage: //otnjj3xqi.bkt.clouddn.com/image/blog/C++%E6%93%8D%E4%BD%9CRedis%E5%AE%9E%E7%8E%B0%E5%BC%82%E6%AD%A5%E8%AE%A2%E9%98%85%E5%92%8C%E5%8F%91%E5%B8%83city.jpg
metaAlignment: center
clearReading: true
categories:
- 分类
- 技术文章
tags:
- C++
---

# 一、


# 二、

# 三、资源管理
## 条款13 - 以对象管理资源
- 获得资源后立刻放进管理对象，即“资源获得时机便是初始化时机”(Resource Acquisition Is Initialization;**RAII**)。
- 管理对象运用析构函数确保资源被释放。例如，使用```auto_ptr``` 或 ```tr1::shared_ptr``` 来替代*raw pointer*来保存资源。
- ```auto_ptr``` 和 ```tr1::shared_ptr```两者在析构函数内做 ```delete``` 而非 ```delete[]``` 动作。
- 小心使用```auto_ptr```，复制动作会是它指向null。
 
## 条款14 - 在资源管理类中小心 copying 行为
- 禁止复制。
    - 有时候允许RAII对象被复制并不合理。
    - 考虑互斥器```Mutex```，很少能够拥有这类对象的副本。
    - 将```copying```操作声明为```private```。
- 对底层资源使用引用计数法。