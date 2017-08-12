---
title: "C++操作Redis实现异步订阅和发布"
date: 2017-08-11T13:05:21+08:00
draft: false
thumbnailImagePosition: left
thumbnailImage: //otnjj3xqi.bkt.clouddn.com/image/blog/C++%E6%93%8D%E4%BD%9CRedis%E5%AE%9E%E7%8E%B0%E5%BC%82%E6%AD%A5%E8%AE%A2%E9%98%85%E5%92%8C%E5%8F%91%E5%B8%83city-750.jpg
coverImage: //otnjj3xqi.bkt.clouddn.com/image/blog/C++%E6%93%8D%E4%BD%9CRedis%E5%AE%9E%E7%8E%B0%E5%BC%82%E6%AD%A5%E8%AE%A2%E9%98%85%E5%92%8C%E5%8F%91%E5%B8%83city.jpg
metaAlignment: center
clearReading: true
categories:
- categories
- techArticle
tags:
- C++
- Redis
- Linux
---

# 安装redis，hiredis库

## 一、实验简介

Redis（Remote Dictionary Server） 是一个由 Salvatore Sanfilippo 写的 key-value 存储系统。
Redis 通常被称为数据结构服务器，因为值（value）可以是 字符串( String ), 哈希( Map ), 列表( list ), 集合( sets ) 和 有序集合( sorted sets )等类型。
它的性能极高，读写速度很快，有很丰富的数据类型，同时还支持对几个操作合并后的原子性执行，
还支持 publish / subscribe , 通知 key 过期等等特性。

### 1.1 知识点

- 介绍 Redis 的基础知识
- 安装使用 Redis
- 安装 hiredis 库，并编写测试程序

### 1.2 效果截图

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2135timestamp1474357807858.png/wm)

## 二、Redis 安装和运行

安装 Redis 可以选择源码安装，或者使用 apt-get 安装。
在环境里安装 Redis 可以使用以下命令：

```
sudo apt-get update
sudo apt-get install redis-server
```

启动 Redis

```
redis-server 	
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2135timestamp1474358245924.png/wm)

查看 Redis 是否启动

```
redis-cli   #重新开一个 shell
redis 127.0.0.1:6379> ping #在客户端里输入ping，检测 Redis 是否装好
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2135timestamp1474357817599.png/wm)

## 三、安装运用 hiredis 库

hiredis 是官方发布的 c/c++ Redis Client Library,源码在 github 上可以找到。
先来了解 hiredis 库的安装。

### 3.1 安装 hiredis 库

运行下面的命令就可以安装好了：

```
wget http://labfile.oss.aliyuncs.com/courses/664/hiredis-master.zip
unzip hiredis-master.zip
cd hiredis-master  
make  
sudo make install
sudo ldconfig /usr/local/lib  #ldconfig是一个动态链接库管理命令 将生成的库复制到 /usr/local/lib 目录下
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2135timestamp1474357824370.png/wm)

### 3.2 同步相关的API

{{< alert info >}}
介绍同步的 API 有几个函数很重要：
{{< /alert >}}

- redisContext \* redisConnect(const char \* ip, int port);
- void \* redisCommand(redisContext \* c, const char \* format, ...);
- void freeReplyObject(void \* reply);

#### 3.2.1 连接相关的函数

这个 `redisConnect` 函数是用来创建 `redisContext` 对象的，context 需要 Hiredis 保持连接状态。当连接出现问题时
`redisContext` 结构有一个 `err` 非0整数来标记错误，`errstr` 将描述错误信息。
Error section 包含了更多错误信息。因此在尝试用 `redisConnect` 连接服务器的时候，应该在 err 数据段
看看连接是否成功。需要了解的是，`redisContext` 不是线程安全的。

使用范例：

```
redisContext *c = redisConnect("127.0.0.1", 6379);
if (c == NULL || c->err) {
    if (c) {
        printf("Error: %s\n", c->errstr);
        // handle error
    } else {
        printf("Can't allocate redis context\n");
    }
}
```

#### 3.2.2 发送相关的命令

有几种方法可以发送命令给 Redis 服务器，`redisCommand` 这个函数很像 `printf`：

```
reply = redisCommand(context, "SET foo bar");
```

占位符 %s 指代一个 string ，可以使用 strlen 来获取 string 的长度

```
reply = redisCommand(context, "SET foo %s", value);
```

占位符 %b 是指代二进制的命令，除了需要指向 string 的指针外，还需要一个 size_t 的数据来指出 string 的长度

```
reply = redisCommand(context, "SET foo %b", value, (size_t) valuelen);
```

除了上面的方式外，还有更灵活的方式来操作 Redis:

```
reply = redisCommand(context, "SET key:%s %s", myid, value);
```

#### 3.2.3 清理

断开连接并且释放 context 可以使用下面的函数：

```
void redisFree(redisContext *c);
```

#### 3.2.4 常见的错误

`REDIS_ERR_IO`:当读或写连接的时候，出现了 I/O 错误。如果你在程序里包含了 errno.h 头文件，可以使用全局变量 errno
来找出哪里出错了。

`REDIS_ERR_EOF`:当断开与服务器连接的时候，读取就是空的。

`REDIS_ERR_PROTOCOL`:解析协议的时候出现了错误。

`REDIS_ERR_OTHER`: 其他的错误，这时可以通过 errstr 来获取错误的信息

### 3.3 异步相关的API

Hiredis 通过其他的 event 库很容易实现异步 API，下面的例程都是用[libev](http://software.schmorp.de/pkg/libev.html) 和[libevent](http://monkey.org/~provos/libevent/)。

#### 3.3.1 连接相关的函数

`redisAsyncConnect` 可以用来创建非阻塞的连接 Redis ，它的返回值是一个指针  `redisAsyncContext`  结构体。 `err`  变量用于检验连接服务端后是否出现了问题。因为创建的连接是非阻塞的，kernel 无法立即返回这个连接是否成功。`\*redisAsyncContext`    也不是线程安全的。



```
redisAsyncContext *c = redisAsyncConnect("127.0.0.1", 6379);
if (c->err) {
    printf("Error: %s\n", c->errstr);
    // handle error
}
```



异步的 `context ` 有一个 用于断开断开的回调函数 ，用于在断开连接获取相关的信息，它的原型如下：



```
void(const redisAsyncContext *c, int status);
```



#### 3.3.2 发送命令和它们的回调

不同于同步 API，异步的模式下发送命令只有一种方式，由于它是异步的，所以在发送命令后需要一个回调函数来检验命令是否发送成功。 Reply 回调函数的原型如下：



```
void(redisAsyncContext *c, void *reply, void *privdata);
```



下面的函数用来在异步的方式下发送命令：



```
int redisAsyncCommand(
  redisAsyncContext *ac, redisCallbackFn *fn, void *privdata,
  const char *format, ...);
int redisAsyncCommandArgv(
  redisAsyncContext *ac, redisCallbackFn *fn, void *privdata,
  int argc, const char **argv, const size_t *argvlen);
```



#### 3.3.3 断开连接

异步的连接将通过如下的方式断开：

```
void redisAsyncDisconnect(redisAsyncContext *ac);
```

当调用这个函数的时候，连接不会立刻终结，并且新的命令也不会接收。连接将会在之前所有的命令都写到 socket 后，并且返回相应的回复后才会终止。这时 disconnection  回调函数会返回 `REDIS_OK`  ， 并且将对象释放掉。

### 3.4 编写测试代码

```
vim test.cpp
```



```
#include <hiredis/hiredis.h>
#include <iostream>
#include <string>
int main(int argc, char **argv)
{
    struct timeval timeout = {2, 0};    //2s的超时时间

    redisContext *pRedisContext = (redisContext*)redisConnectWithTimeout("127.0.0.1", 6379, timeout);	//redisContext是Redis操作对象
    if ( (NULL == pRedisContext) || (pRedisContext->err) )
    {
        if (pRedisContext)
        {
            std::cout << "connect error:" << pRedisContext->errstr << std::endl;
        }
        else
        {
            std::cout << "connect error: can't allocate redis context." << std::endl;
        }
        return -1;
    }
    //redisReply是Redis命令回复对象 redis返回的信息保存在redisReply对象中
    redisReply *pRedisReply = (redisReply*)redisCommand(pRedisContext, "INFO");  //执行INFO 命令，可以看到 redis 的一些详细情况
    std::cout << pRedisReply->str << std::endl;
    //当多条Redis命令使用同一个redisReply对象时
    //每一次执行完Redis命令后需要清空redisReply 以免对下一次的Redis操作造成影响
    freeReplyObject(pRedisReply);   

    return 0;
}
```

运行：

```
sudo redis-server	# 单独开一个 shell 执行命令
g++ test.cpp -o test -lhiredis
./test				
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2135timestamp1474357843053.png/wm)


##  四、实验总结

本节课学习了 redis 的安装和运用，并使用 hiredis 库编写了一个测试程序，执行 INFO 命令，获取 redis 服务器相关的信息。redis 有很丰富的功能，除了nosql 特性之外，有很好的容灾能力。RDB 持久化可以在指定的时间间隔内生成数据集的时间点快照（point-in-time snapshot），AOF 持久化记录服务器执行的所有写操作命令。下节课将编写实现一个 异步的订阅和发布程序。



## 五、参考资料

[Linux下C++操作Redis](http://hahaya.github.io/operator-redis-under-linux/)

[redis/hiredis](https://github.com/redis/hiredis#synchronous-api)



# 异步实现订阅和发布

## 一、实验简介

本节实验将实现消息的订阅和发布

### 1.1 知识点

- 发布订阅模式者模式介绍
- 编写发布者相关模块
- 编写订阅者相关模块
- 编写 makefile

### 1.2 效果截图

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454566716.png/wm)

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454573848.png/wm)


## 二、介绍发布订阅模式

订阅，取消订阅和发布实现了发布/订阅消息范式(引自wikipedia)，发送者（发布者）不是计划发送消息给特定的接收者（订阅者）。而是发布的消息分到不同的频道，不需要知道什么样的订阅者订阅。


![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454591074.png/wm)

订阅者对一个或多个频道感兴趣，只需接收感兴趣的消息，不需要知道什么样的发布者发布的。


![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454585210.png/wm)

这种发布者和订阅者的解耦合可以带来更大的扩展性和更加动态的网络拓扑。

## 二、安装相关的库

安装 libevent

```
sudo apt-get install libevent-dev
```

安装 boost库

```
sudo apt-get install libboost-dev
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454599073.png/wm)


![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454603867.png/wm)



## 三、编写发布者相关的模块

首先创建运行的文件夹：

```
mkdir Test
cd Test
```



### 3.1 编写 redis_publisher.h



redis_publisher.h 封装了 hiredis ， 实现消息发布给 redis的功能。



```
#ifndef REDIS_PUBLISHER_H
#define REDIS_PUBLISHER_H

#include <stdlib.h>
#include <hiredis/async.h>
#include <hiredis/adapters/libevent.h>
#include <string>
#include <vector>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <boost/tr1/functional.hpp>

class CRedisPublisher
{
public:    
    CRedisPublisher();
    ~CRedisPublisher();

    bool init();
    bool uninit();
    bool connect();
    bool disconnect();

    bool publish(const std::string &channel_name,
        const std::string &message);

private:
     // 下面三个回调函数供redis服务调用
    // 连接回调
    static void connect_callback(const redisAsyncContext *redis_context,
        int status);

	// 断开连接的回调
    static void disconnect_callback(const redisAsyncContext *redis_context,
        int status);

	// 执行命令回调
    static void command_callback(redisAsyncContext *redis_context,
        void *reply, void *privdata);

    // 事件分发线程函数
    static void *event_thread(void *data);
    void *event_proc();

private:
     // libevent事件对象
    event_base *_event_base;
	// 事件线程ID
    pthread_t _event_thread;
	// 事件线程的信号量
    sem_t _event_sem;
	// hiredis异步对象
    redisAsyncContext *_redis_context;
};

#endif
```



### 3.2 编写 redis_publisher.cpp

```
#include <stddef.h>
#include <assert.h>
#include <string.h>
#include "redis_publisher.h"

CRedisPublisher::CRedisPublisher():_event_base(0), _event_thread(0),
_redis_context(0)
{
}

CRedisPublisher::~CRedisPublisher()
{
}

bool CRedisPublisher::init()
{
    // initialize the event
    _event_base = event_base_new();    // 创建libevent对象
    if (NULL == _event_base)
    {
        printf(": Create redis event failed.\n");
        return false;
    }

    memset(&_event_sem, 0, sizeof(_event_sem));
    int ret = sem_init(&_event_sem, 0, 0);
    if (ret != 0)
    {
        printf(": Init sem failed.\n");
        return false;
    }

    return true;
}

bool CRedisPublisher::uninit()
{
    _event_base = NULL;

    sem_destroy(&_event_sem);   
    return true;
}

bool CRedisPublisher::connect()
{
    // connect redis
    _redis_context = redisAsyncConnect("127.0.0.1", 6379);    // 异步连接到redis服务器上，使用默认端口
    if (NULL == _redis_context)
    {
        printf(": Connect redis failed.\n");
        return false;
    }

    if (_redis_context->err)
    {
        printf(": Connect redis error: %d, %s\n",
            _redis_context->err, _redis_context->errstr);    // 输出错误信息
        return false;
    }

    // attach the event
    redisLibeventAttach(_redis_context, _event_base);    // 将事件绑定到redis context上，使设置给redis的回调跟事件关联

    // 创建事件处理线程
    int ret = pthread_create(&_event_thread, 0, &CRedisPublisher::event_thread, this);
    if (ret != 0)
    {
        printf(": create event thread failed.\n");
        disconnect();
        return false;
    }

	// 设置连接回调，当异步调用连接后，服务器处理连接请求结束后调用，通知调用者连接的状态
    redisAsyncSetConnectCallback(_redis_context,
        &CRedisPublisher::connect_callback);

	// 设置断开连接回调，当服务器断开连接后，通知调用者连接断开，调用者可以利用这个函数实现重连
    redisAsyncSetDisconnectCallback(_redis_context,
        &CRedisPublisher::disconnect_callback);

	// 启动事件线程
    sem_post(&_event_sem);
    return true;
}

bool CRedisPublisher::disconnect()
{
    if (_redis_context)
    {
        redisAsyncDisconnect(_redis_context);
        redisAsyncFree(_redis_context);
        _redis_context = NULL;
    }

    return true;
}

bool CRedisPublisher::publish(const std::string &channel_name,
    const std::string &message)
{
    int ret = redisAsyncCommand(_redis_context,
        &CRedisPublisher::command_callback, this, "PUBLISH %s %s",
        channel_name.c_str(), message.c_str());
    if (REDIS_ERR == ret)
    {
        printf("Publish command failed: %d\n", ret);
        return false;
    }

    return true;
}

void CRedisPublisher::connect_callback(const redisAsyncContext *redis_context,
    int status)
{
    if (status != REDIS_OK)
    {
        printf(": Error: %s\n", redis_context->errstr);
    }
    else
    {
        printf(": Redis connected!\n");
    }
}

void CRedisPublisher::disconnect_callback(
    const redisAsyncContext *redis_context, int status)
{
    if (status != REDIS_OK)
    {
		// 这里异常退出，可以尝试重连
        printf(": Error: %s\n", redis_context->errstr);
    }
}

// 消息接收回调函数
void CRedisPublisher::command_callback(redisAsyncContext *redis_context,
    void *reply, void *privdata)
{
    printf("command callback.\n");
	// 这里不执行任何操作
}

void *CRedisPublisher::event_thread(void *data)
{
    if (NULL == data)
    {
        printf(": Error!\n");
        assert(false);
        return NULL;
    }

    CRedisPublisher *self_this = reinterpret_cast<CRedisPublisher *>(data);
    return self_this->event_proc();
}

void *CRedisPublisher::event_proc()
{
    sem_wait(&_event_sem);

	// 开启事件分发，event_base_dispatch会阻塞
    event_base_dispatch(_event_base);

    return NULL;
}
```



### 四、编写订阅者相关的模块

封装 hiredis， 实现消息订阅 redis 功能



### 4.1 编写 redis_subscriber.h

```
#ifndef REDIS_SUBSCRIBER_H
#define REDIS_SUBSCRIBER_H

#include <stdlib.h>
#include <hiredis/async.h>
#include <hiredis/adapters/libevent.h>
#include <string>
#include <vector>
#include <unistd.h>
#include <pthread.h>
#include <semaphore.h>
#include <boost/tr1/functional.hpp>

class CRedisSubscriber
{
public:
    typedef std::tr1::function<void (const char *, const char *, int)>         NotifyMessageFn;	// 回调函数对象类型，当接收到消息后调用回调把消息发送出去

    CRedisSubscriber();
    ~CRedisSubscriber();

	bool init(const NotifyMessageFn &fn);	// 传入回调对象
    bool uninit();
    bool connect();
    bool disconnect();

    // 可以多次调用，订阅多个频道
    bool subscribe(const std::string &channel_name);

private:
    // 下面三个回调函数供redis服务调用
    // 连接回调
    static void connect_callback(const redisAsyncContext *redis_context,
        int status);

	// 断开连接的回调
    static void disconnect_callback(const redisAsyncContext *redis_context,
        int status);

	// 执行命令回调
    static void command_callback(redisAsyncContext *redis_context,
        void *reply, void *privdata);

    // 事件分发线程函数
    static void *event_thread(void *data);
    void *event_proc();

private:
    // libevent事件对象
    event_base *_event_base;
	// 事件线程ID
    pthread_t _event_thread;
	// 事件线程的信号量
    sem_t _event_sem;
	// hiredis异步对象
    redisAsyncContext *_redis_context;

	// 通知外层的回调函数对象
    NotifyMessageFn _notify_message_fn;
};

#endif
```

### 4.2 编写 redis_subscriber.cpp

```
#include <stddef.h>
#include <assert.h>
#include <string.h>
#include "redis_subscriber.h"

CRedisSubscriber::CRedisSubscriber():_event_base(0), _event_thread(0),
_redis_context(0)
{
}

CRedisSubscriber::~CRedisSubscriber()
{
}

bool CRedisSubscriber::init(const NotifyMessageFn &fn)
{
    // initialize the event
    _notify_message_fn = fn;
    _event_base = event_base_new();    // 创建libevent对象
    if (NULL == _event_base)
    {
        printf(": Create redis event failed.\n");
        return false;
    }

    memset(&_event_sem, 0, sizeof(_event_sem));
    int ret = sem_init(&_event_sem, 0, 0);
    if (ret != 0)
    {
        printf(": Init sem failed.\n");
        return false;
    }

    return true;
}

bool CRedisSubscriber::uninit()
{
    _event_base = NULL;

    sem_destroy(&_event_sem);   
    return true;
}

bool CRedisSubscriber::connect()
{
    // connect redis
    _redis_context = redisAsyncConnect("127.0.0.1", 6379);    // 异步连接到redis服务器上，使用默认端口
    if (NULL == _redis_context)
    {
        printf(": Connect redis failed.\n");
        return false;
    }

    if (_redis_context->err)
    {
        printf(": Connect redis error: %d, %s\n",
            _redis_context->err, _redis_context->errstr);    // 输出错误信息
        return false;
    }

    // attach the event
    redisLibeventAttach(_redis_context, _event_base);    // 将事件绑定到redis context上，使设置给redis的回调跟事件关联

    // 创建事件处理线程
    int ret = pthread_create(&_event_thread, 0, &CRedisSubscriber::event_thread, this);
    if (ret != 0)
    {
        printf(": create event thread failed.\n");
        disconnect();
        return false;
    }

	// 设置连接回调，当异步调用连接后，服务器处理连接请求结束后调用，通知调用者连接的状态
    redisAsyncSetConnectCallback(_redis_context,
        &CRedisSubscriber::connect_callback);

	// 设置断开连接回调，当服务器断开连接后，通知调用者连接断开，调用者可以利用这个函数实现重连
    redisAsyncSetDisconnectCallback(_redis_context,
        &CRedisSubscriber::disconnect_callback);

	// 启动事件线程
    sem_post(&_event_sem);
    return true;
}

bool CRedisSubscriber::disconnect()
{
    if (_redis_context)
    {
        redisAsyncDisconnect(_redis_context);
        redisAsyncFree(_redis_context);
        _redis_context = NULL;
    }

    return true;
}

bool CRedisSubscriber::subscribe(const std::string &channel_name)
{
    int ret = redisAsyncCommand(_redis_context,
        &CRedisSubscriber::command_callback, this, "SUBSCRIBE %s",
        channel_name.c_str());
    if (REDIS_ERR == ret)
    {
        printf("Subscribe command failed: %d\n", ret);
        return false;
    }

    printf(": Subscribe success: %s\n", channel_name.c_str());
    return true;
}

void CRedisSubscriber::connect_callback(const redisAsyncContext *redis_context,
    int status)
{
    if (status != REDIS_OK)
    {
        printf(": Error: %s\n", redis_context->errstr);
    }
    else
    {
        printf(": Redis connected!");
    }
}

void CRedisSubscriber::disconnect_callback(
    const redisAsyncContext *redis_context, int status)
{
    if (status != REDIS_OK)
    {
		// 这里异常退出，可以尝试重连
        printf(": Error: %s\n", redis_context->errstr);
    }
}

// 消息接收回调函数
void CRedisSubscriber::command_callback(redisAsyncContext *redis_context,
    void *reply, void *privdata)
{
    if (NULL == reply || NULL == privdata) {
        return ;
    }

	// 静态函数中，要使用类的成员变量，把当前的this指针传进来，用this指针间接访问
    CRedisSubscriber *self_this = reinterpret_cast<CRedisSubscriber *>(privdata);
    redisReply *redis_reply = reinterpret_cast<redisReply *>(reply);

	// 订阅接收到的消息是一个带三元素的数组
    if (redis_reply->type == REDIS_REPLY_ARRAY &&
    redis_reply->elements == 3)
    {
        printf(": Recieve message:%s:%d:%s:%d:%s:%d\n",
        redis_reply->element[0]->str, redis_reply->element[0]->len,
        redis_reply->element[1]->str, redis_reply->element[1]->len,
        redis_reply->element[2]->str, redis_reply->element[2]->len);

		// 调用函数对象把消息通知给外层
        self_this->_notify_message_fn(redis_reply->element[1]->str,
            redis_reply->element[2]->str, redis_reply->element[2]->len);
    }
}

void *CRedisSubscriber::event_thread(void *data)
{
    if (NULL == data)
    {
        printf(": Error!\n");
        assert(false);
        return NULL;
    }

    CRedisSubscriber *self_this = reinterpret_cast<CRedisSubscriber *>(data);
    return self_this->event_proc();
}

void *CRedisSubscriber::event_proc()
{
    sem_wait(&_event_sem);

	// 开启事件分发，event_base_dispatch会阻塞
    event_base_dispatch(_event_base);

    return NULL;
}
```

## 五、初始化过程

`CRedisPublisher`和`CRedisSubscriber`的初始化过程：

初始化事件处理，并获得事件处理的实例：

```
_event_base = event_base_new();
```

在获得redisAsyncContext *之后，调用

```
redisLibeventAttach(_redis_context, _event_base);
```

这样就将事件处理和redis关联起来，最后在另一个线程调用

```
event_base_dispatch(_event_base);
```

启动事件的分发，这是一个阻塞函数，因此，创建了一个新的线程处理事件分发，值得注意的是，这里用信号灯_event_sem控制线程的启动，意在程序调用

```
redisAsyncSetConnectCallback(_redis_context,
		&CRedisSubscriber::connect_callback);
redisAsyncSetDisconnectCallback(_redis_context,
        &CRedisSubscriber::disconnect_callback);
```



## 六、编写 publisher.cpp ，实现发布消息

```
#include "redis_publisher.h"

int main(int argc, char *argv[])
{
    CRedisPublisher publisher;

    bool ret = publisher.init();
    if (!ret)
    {
        printf("Init failed.\n");
        return 0;
    }

    ret = publisher.connect();
    if (!ret)
    {
        printf("connect failed.");
        return 0;
    }

    while (true)
    {
        publisher.publish("test-channel", "Hello shiyanlou!");
        sleep(1);
    }

    publisher.disconnect();
    publisher.uninit();
    return 0;
}
```

## 七、编写 subscriber.cpp 实现消息的订阅

```
#include "redis_subscriber.h"

void recieve_message(const char *channel_name,
    const char *message, int len)
{
    printf("Recieve message:\n    channel name: %s\n    message: %s\n",
        channel_name, message);
}

int main(int argc, char *argv[])
{
    CRedisSubscriber subscriber;
    CRedisSubscriber::NotifyMessageFn fn =
        bind(recieve_message, std::tr1::placeholders::_1,
        std::tr1::placeholders::_2, std::tr1::placeholders::_3);

    bool ret = subscriber.init(fn);
    if (!ret)
    {
        printf("Init failed.\n");
        return 0;
    }

    ret = subscriber.connect();
    if (!ret)
    {
        printf("Connect failed.\n");
        return 0;
    }

    subscriber.subscribe("test-channel");

    while (true)
    {
        sleep(1);
    }

    subscriber.disconnect();
    subscriber.uninit();

    return 0;
}
```

## 八 关于编译

编写 makefile ,请注意不能使用空格，只能使用 Tab 键，空格会报错，
直接复制本段 makefile 也会报错，需要将空格转为 Tab 才行,
所以在写的时候请注意这一点。

```
vim makefile
```



```
EXE=server_main client_main
CC=g++
FLAG=-lhiredis -levent -lpthread -Wformat
OBJ=redis_publisher.o publisher.o redis_subscriber.o subscriber.o

all:$(EXE)

$(EXE):$(OBJ)
    $(CC) -o publisher redis_publisher.o publisher.o $(FLAG)
    $(CC) -o subscriber redis_subscriber.o subscriber.o $(FLAG)

redis_publisher.o:redis_publisher.h
redis_subscriber.o:redis_subscriber.h

publisher.o:publisher.cpp
    $(CC) -c publisher.cpp

subscriber.o:subscriber.cpp
    $(CC) -c subscriber.cpp
clean:
    rm publisher subscriber *.o
```



## 九、编译运行


编译程序：
```
make
sudo make install
```

这里也准备了所有代码的压缩包：

```
wget http://labfile.oss.aliyuncs.com/courses/664/Source.zip

```

```
unzip Source.zip
cd Source
```

打开终端：

```
sudo redis-server
```

再打开一个终端，进入Test：

```
./publisher
```

打开第三个终端，进入Test：

```
./subscriber
```

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454641238.png/wm)

![此处输入图片的描述](https://dn-anything-about-doc.qbox.me/document-uid59274labid2136timestamp1474454652615.png/wm)

## 十、实验总结

经过这两节课的学习，我们实现了基于 hiredis 库 的订阅和发布，并且将消息从发布者发送给订阅者，因为是基于异步的发送和接收，所以需要回调函数来返回确认相关的信息，整体的架构很清晰，相信大家在学习完本课后能对 redis 异步的发布和订阅有一个很好的认识。

参考资料

[Redis 发布订阅](http://www.runoob.com/redis/redis-pub-sub.html)

[linux下使用hiredis异步API实现sub/pub消息订阅和发布的功能](http://www.voidcn.com/blog/xumaojun/article/p-5995876.html)
