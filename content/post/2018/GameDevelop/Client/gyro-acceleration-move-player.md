---
title: "使用重力计/陀螺仪作为输入驱动角色移动"
date: 2018-04-25T15:49:48+08:00
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
- 角色控制
---

## 介绍
传统的MMORPG使用JoyStick摇杆作为角色控制器，来控制玩家角色的移动。摇杆符合大多数游戏玩家的操作认知，但是其存在两个问题，一是移动操作全都由左手来承受，负担较重；而是摇杆区域占用了很大一块空间，而且左手操纵摇杆也会影响一部分视野。所以考虑用手机姿态作为输入来驱动角色移动。

## 如何移动

## 操控优化 
玩家在实际游戏中，是不大可能是以手机屏幕水平向上的状态进行游戏的，一般情况是手机屏幕正对面部，这样的话，手机与水平面(比如桌面)的夹角就是45°-90°间。这时需要用这个状态作为初始状态，以这个状态下重力计的值作为基准值，当手机姿态发生改变时，求重力计的真实数据相对于基准值的偏移，将偏移值作为输入来计算物体需要进行的移动。这一过程用代码体现如下：
```
//校准基准值的接口
public void CalibrateAcceleration()
{
    //将当前重力计状态作为初始状态
    InitVal = Input.acceleration;
    //计算出 手机平放姿态到当前姿态 的旋转
    Quaternion rotateQuat = Quaternion.FromToRotation(Vector3.back, InitVal);
    //定义一个移动旋转缩放矩阵
    Matrix4x4 matrix = Matrix4x4.TRS(Vector3.zero, rotateQuat, Vector3.one);
    //求出这个矩阵的逆
    calibrationMatrix = matrix.inverse;
}
//获得偏移值的接口
public Vector3 CaclAccelerationRelativePos()
{
    Vector3 accel = calibrationMatrix.MultiplyVector(CurVal);
    return accel;
}
```
但是实际体验过程中，发现感受并不是太好，因为想要变换方向时，手机的姿态变化必须体现在这个固定基准值之上，这会导致玩家操作手机的动作幅度过大，为了解决这个问题，就需要自动的动态调整基准值。这个过程大概如下：
* 当角色向左移动时，当前手机相对于基准值一定是向左发生偏转，此时将基准值调整