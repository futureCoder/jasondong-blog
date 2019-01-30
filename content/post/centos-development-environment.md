---
title: "CentOS7开发环境配置"
date: 2018-10-07T19:32:27+08:00
draft: true
categories:
- 分类
- 技术文章
tags:
- CentOS
---

* 安装Perl支持  
```yum -y groupinstall Perl* ```  
* 安装gcc  
```yum -y install gcc```
* 安装g++  
```yum -y install gcc-c++```
```yum -y install libstdc++-devel```
* 安装CMake  
```yum -y install cmake3```
* 安装m4  
```wget http://mirrors.kernel.org/gnu/m4/m4-1.4.13.tar.gz ```  
```tar -xzvf m4-1.4.13.tar.gz && cd m4-1.4.13 && ./configure -prefix=/usr/local && make && make install```  
* 安装autoconf-依赖m4  
```wget http://mirrors.kernel.org/gnu/autoconf/autoconf-2.65.tar.gz ```  
```tar -xzvf autoconf-2.65.tar.gz && cd autoconf-2.65 && ./configure -prefix=/usr/local && make && make install```
* 安装automake  
```wget http://mirrors.kernel.org/gnu/automake/automake-1.11.tar.gz ```  
```tar xzvf automake-1.11.tar.gz && cd automake-1.11 && ./configure -prefix=/usr/local && make && make install```  
* 安装libtool  
```wget http://mirrors.kernel.org/gnu/libtool/libtool-2.2.6b.tar.gz ```  
```tar xzvf libtool-2.2.6b.tar.gz && cd libtool-2.2.6b && ./configure -prefix=/usr/local && make && make install```  
* 安装Python2开发包  
```yum install -y python-devel```  
```wget https://www.python.org/ftp/python/3.6.0/Python-3.6.0a1.tar.xz```  
```tar xvf Python-3.6.0a1.tar.xz && cd Python-3.6.0a1 && ./configure -prefix=/usr/local && make && make install```  
* 安装Git  
```yum -y install git```