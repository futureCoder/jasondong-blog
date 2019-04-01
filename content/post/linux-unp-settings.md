---
title: "Unix网络编程卷1-环境搭建"
date: 2018-01-06T16:15:48+08:00
hierarchicalCategories: true
draft: true
categories: 
- 分类
- 随手一写
---

1. chomod u+x configure  
./configure
2. cd lib  
make
3. cd ../libfree  
make
4. cd ../libgai  
make


若出现如下错误:
```
gcc -I../lib -g -O2 -D_REENTRANT -Wall   -c -o inet_ntop.o inet_ntop.c  
inet_ntop.c: In function ‘inet_ntop’:  
inet_ntop.c:60:9: error: argument ‘size’ doesn’t match prototype  
  size_t size;  
         ^  
In file included from inet_ntop.c:27:0:  
/usr/include/arpa/inet.h:64:20: error: prototype declaration  
 extern const char *inet_ntop (int __af, const void *__restrict __cp,  
                    ^  
make: *** [inet_ntop.o] Error 1  
```

将生成的libnup.a复制到/usr/lib下
sudo cp libunp.a /usr/lib
修改lib/unp.h并将其和config.h拷到/usr/include中,为了include方便
vim lib/unp.h //将#include "../config.h"改成#include "config.h"
sudo cp lib/unp.h /usr/include
sudo cp config.h /usr/include

编译例子
cd intro
gcc daytimetcpcli.c -o cli -lunp  

     _                       ____                    
    | | __ _ ___  ___  _ __ |  _ \  ___  _ __   __ _ 
 _  | |/ _` / __|/ _ \| '_ \| | | |/ _ \| '_ \ / _` |
| |_| | (_| \__ \ (_) | | | | |_| | (_) | | | | (_| |
 \___/ \__,_|___/\___/|_| |_|____/ \___/|_| |_|\__, |
                                               |___/ 