---
title: "Skynet 安装"
date: 2017-10-31T16:42:45+08:00
draft: true
---

提示错误 
```
make[1]: Entering directory `/data/skynet'
cd 3rd/jemalloc && ./autogen.sh --with-jemalloc-prefix=je_ --disable-valgrind
autoconf
./autogen.sh: line 5: autoconf: command not found
Error 0 in autoconf
make[1]: *** [3rd/jemalloc/Makefile] Error 1
make[1]: Leaving directory `/data/skynet'
make: *** [linux] Error 2
```

解决：
sudo apt-get install autoconf


提示错误
```


```