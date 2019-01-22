---
title: "四元数与旋转"
date: 2018-04-25T10:27:59+08:00
draft: true
clearReading: false
categories:
- 分类
- 技术文章
tags:
- unity
- 游戏编程
- 图形学
- 四元数
---

## 基本介绍
在U3D中，使用四元数(Quaternion)来表示旋转。比如transform.rotation就是一个四元数，其由四部分组成  
Quaternion = (xi,yj,zk,w) = (x,y,z,w)  
维基百科上关于四元数的定义：[[English]](https://en.wikipedia.org/wiki/Quaternion) [[中文]](https://zh.wikipedia.org/wiki/%E5%9B%9B%E5%85%83%E6%95%B8)  
四元数与空间旋转：[[English]](https://en.wikipedia.org/wiki/Quaternions_and_spatial_rotation) [[中文]](https://zh.wikipedia.org/wiki/%E5%9B%9B%E5%85%83%E6%95%B0%E4%B8%8E%E7%A9%BA%E9%97%B4%E6%97%8B%E8%BD%AC)  
还有一篇详细阐述四元数的文章：[[English]](https://www.3dgep.com/understanding-quaternions/) [[中文]](http://www.qiujiawei.com/understanding-quaternions/#8)  

四元数理解起来很抽象：简单的超复数，是复数的不可交换延伸。  
我们都知道向量，向量是用来表示一个二维空间的轨迹和方向，形象化的表示为带箭头的线段。而四元数则表示一个四维空间的轨迹和方向。四元数的表现形式：  
```Q = [q1, q2, q3, q4]```  
x、y、z轴的偏移量分别为：  
```z = atan2(2(q2q3 - q1q4), 2(q1<sup>2</sup> + q2<sup>2</sup>) - 1)  ```
```y = atan2(2(q3q4 - q1q2), 2(q1<sup>2</sup> + q4<sup>2</sup>) - 1)  ```
```x = -asin(2(q2q4 + q1q3))  ```
于是，只要我们知道一个四元数的值，就可以算出其x、y、z轴的偏移量。
我们从陀螺仪传感器获得的四元数数据，是相对于手机平方在桌面的xyz轴的偏移量，如果我们转换成相对于上一个位置的偏移量，然后根据这个偏移量来做相应的操作。
下面我们看一下如果获取相对的偏移量。


## U3D中的使用
在U3D中，Quaternion 的乘法操作(operator *)有两种操作：  
1. Quaternion * Quaternion，例如 q = t * p，表示将一个点先进行t操作旋转，然后进行p操作旋转；  
2. Quaternion * vector3 ，例如 p:Vector3, t:Quaternion,q:Quaternion，q = t * p表示将点p进行t操作旋转。  
Quaternion的基本数学方程为：  
```Q = cos(θ/2) + (x*sin(θ/2))i + (y*sin(θ/2))j + (z*sin(θ/2))k``` 其中，θ为旋转角度.  
```Quaternion.w = cos(θ/2)```  
```Quaternion.x = axis.x * sin(θ/2)```  
```Quaternion.y = axis.y * sin(θ/2)```  
```Quaternion.z = axis.z * sin(θ/2)```  
我们只要有角度就可以给出四元数的四个部分值，比如我想要让点M=Vector3(o,p,q)绕x轴顺时针旋转90°，那么对应的Quaternion数值就应该为：  
```
Q.x = 1*sin((π/2) / 2) = sin(π/4) = 0.7071  
Q.y = 0  
Q.z = 0  
Q.w = cos((π/2) / 2) = cos(π/4) = 0.7071  
Q = (0.7071, 0, 0, 0.7071)  
m = Q * M
```
Vector3 a = transform.right;    //沿x轴上的方向  
Quaternion rotate = Quaternion.Euler(0, 0, angleOffset); //在z轴上旋转角度angleOffset  
Vector3 ret = rotate* a;        //按rotate旋转a，得到目标ret  
