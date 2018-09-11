---
title: "使用UnityRemote进行远程调试"
date: 2018-05-17T17:47:02+08:00  
draft: true  
clearReading: true  
categories:
- 分类
- 技术文章
tags:
- unity
- 游戏编程
- 远程调试
---

## 环境准备
1. PC：下载安卓sdk；安卓机：安装Unity Remote.apk
2. 使用USB连接手机，打开开发者模式（设置->关于手机->版本号 连续点击7次，返回，出现开发者选项），打开USB调试
3. PC上安装安卓设备USB驱动程序，步骤在下面
4. 手机运行 Unity Remote.apk 
5. PC打开Unity，在 Edit->Project Settings->Editor下的Unity Remote下选择Any Android Device 
6. 在Unity中Play即可

## PC安装安卓设备USB驱动程序
1. 设备管理器找adb interface，华为机可能都是hdb interface，右键->更新驱动程序软件
2. 浏览计算机以查找..
3. 从计算机的...选择
4. 默认选项，下一步
5. 从磁盘安装，选 %Android SDK目录%\extras\google\usb_driver目录中的android_winusb.inf
6. 选 Android Composite ADB Interface 确定