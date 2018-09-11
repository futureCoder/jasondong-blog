---
title: "2D/3D下重力计/陀螺仪系统的搭建"
date: 2018-04-24T16:46:35+08:00
draft: true
clearReading: false
categories:
- 分类
- 技术文章
tags:
- unity
- 游戏编程
- 陀螺仪
- 重力计
---

## 一、简介 
项目需要在2D/3D场景下搭建一套通用的重力计/陀螺仪系统。在这里记录一下实现方法，顺便整理一下思路。
## 二、基础数据结构
### 2.1 参数设置
每个需要用陀螺仪功能的物体都要单独有一套参数，才能为每个物体设置不同的效果。比如在摇手机的时候，背景物体向左，前景物体向右，通过调节每个物体身上的参数就可以实现。
#### 2.1.1 公用基本参数
{{< codeblock "基本参数设置">}}
public class CSetUp
{
    [Tooltip("敏感度")]
    public float sensitivity = 15f;
    [Tooltip("最大水平移动速度")]
    public float maxTurnSpeed = 35f;
    [Tooltip("最大垂直倾斜角移动速度")]
    public float maxTilt = 35f;
    [Tooltip("位移加成速率")]
    public float posRate = 1.5f;
}
{{< /codeblock >}}
{{< codeblock "轴和姿态选择">}}
public enum EnumMotionAxial
{
    None = 0,
    X = 1 << 0,
    Y = 1 << 1,
    Z = 1 << 2,
    XY = X | Y,
    XZ = X | Z,
    YZ = Y | Z,
    All = X | Y | Z,
}
public enum EnumMotionMode
{
    Position = 1,   //只位置变化
    Rotation = 2,
    All = 3,
}
{{< /codeblock >}}
#### 2.1.2 2D特有
用于限制2DUI的最大位移量，负值表示不限制。
{{< codeblock "位移限制">}}
public class CPositionLimit
{
    [Tooltip("左")]
    public int Left = -1;
    [Tooltip("右")]
    public int Right = -1;
    [Tooltip("下")]
    public int Bottom = -1;
    [Tooltip("上")]
    public int Top = -1;
}
{{< /codeblock >}}

### 2.2 重力计/陀螺仪管理类

#### 2.2.1 DataProvider
在此类中提供初始坐标、当前坐标、上一帧坐标，而且有些UI需要特殊效果(比如当摇动手机时，花瓣飞起，而当手机稳定时花瓣再缓慢落下)，这个的实现思路是当检测到手机基本稳定的时候，将初始坐标逐渐插值到当前坐标来，而初始坐标一直被其他UI所使用，不能改变，所以需要加额外的数据，即稳定坐标(其实就是暂时的初始坐标，用来与当前坐标计算出相对坐标)和过渡坐标(当设备稳定时，其逐渐插值到当前坐标，若在插值过程中，手机再次失去稳定时，稳定坐标将设置为当前的过渡坐标)。