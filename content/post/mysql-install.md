---
title: "MySql配置"
date: 2018-10-07T19:32:27+08:00
draft: true
categories:
- 分类
- 技术文章
tags:
- CentOS
---

安装完今天以zip模式在windows10 64位环境下安装mysql8.0，到最后一步提示mysql服务无法启动。

安装步骤如下：

1.配置环境变量

我的电脑->属性->高级->环境变量->path

如:C:\Program Files\MySQL\MySQL Server 8.0\bin 

注意是追加，不要覆盖

2.修改my-default.ini

在其中修改或添加配置： 

[mysqld] 

basedir=C:\Program Files\MySQL\MySQL Server 8.0（mysql所在目录） 

datadir=C:\Program Files\MySQL\MySQL Server 8.0\data （mysql所在目录\data）

3.以管理员身份运行cmd（win10右键左下角开始按钮选择以管理员身份运行cmd即可）

以管理员身份运行cmd（一定要用管理员身份运行，不然权限不够），

输入：cd C:\Program Files\MySQL\MySQL Server 8.0\bin 进入mysql的bin文件夹(不管有没有配置过环境变量，也要进入bin文件夹，否则之后启动服务仍然会报错误2)

输入mysqld -install(如果不用管理员身份运行，将会因为权限不够而出现错误：Install/Remove of the Service Denied!) 

安装成功

4.运行mysqld  --initialize（标题问题所在，若没有init则不存在data目录，自然无法启动成功）

5.安装成功后就要启动服务了，继续在cmd中输入:net start mysql,服务启动成功！

服务启动成功之后，就可以登录了，输入mysql -u root -p（第一次登录没有密码，直接按回车过）,登录成功！

 

追加内容：

在安装mysql5.7版本时，经常会遇到mysql -u root -p直接回车登陆不上的情况，原因在于5.7版本在安装时自动给了一个随机密码，坑爹的是在init步骤的时候不像linux系统会给出命令行提示，需要手动在mysql目录下搜索*.err，以文本形式打开才能看到如下内容：

016-02-25T15:09:43.033062Z 1 [Note] A temporary password is generated for root@localhost: >mso<k70mrWe

红色字母即为第一次的登陆密码，记得加双引号。

使用Navicat Premium 12连接MySql时报错"authentication plugin 'caching_sha2_password'", 解决办法:
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123321';