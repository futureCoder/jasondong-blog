---
title: "Lua和C++交互详细总结"
date: 2017-10-31T16:42:45+08:00
draft: true
---

http://blog.csdn.net/hua_jie_zhang/article/details/40557317

# 一、Lua堆栈
要理解Lua和C++交互，首先要理解Lua堆栈。
简单来说，Lua和C/C++语言通信的主要方法是一个无处不在的虚拟栈。栈的特点是先进后出。
在Lua中，Lua堆栈就是一个struct，堆栈索引的方式可是是正数也可以是负数，区别是：正数索引1永远表示栈底，负数索引-1永远表示栈顶。

# 二、堆栈的操作
因为Lua与C/C++是通过栈来通信，Lua提供了C API对栈进行操作。
我们先来看一个最简单的例子：
```
#include <iostream>
#include <string.h>
using namespace std;
extern "C"
{
    #include "lua.h"
    #include "lauxlib.h"
    #include "lualib.h"
}
void main()
{
    //1.创建一个state
    lua_State *L = luaL_newstate();
    //2.入栈操作
    lua_pushstring(L, "I am so cool~");
    lua_pushnumber(L,20);
    //3.取值操作
    if( lua_isstring(L,1)){             //判断是否可以转为string
        cout<<lua_tostring(L,1)<<endl;  //转为string并返回
    }
    if( lua_isnumber(L,2)){
        cout<<lua_tonumber(L,2)<<endl;
    }
    //4.关闭state
    lua_close(L);
    return ;
}
```
可以简单理解为```luaL_newstate```返回一个指向堆栈的指针，其它看注释应该能懂了吧。
其他一些栈操作：
```
int   lua_gettop (lua_State *L);            //返回栈顶索引（即栈长度）
void  lua_settop (lua_State *L, int idx);   //
void  lua_pushvalue (lua_State *L, int idx);//将idx索引上的值的副本压入栈顶
void  lua_remove (lua_State *L, int idx);   //移除idx索引上的值
void  lua_insert (lua_State *L, int idx);   //弹出栈顶元素，并插入索引idx位置
void  lua_replace (lua_State *L, int idx);  //弹出栈顶元素，并替换索引idx位置的值
```
```lua_settop```将栈顶设置为一个指定的位置，即修改栈中元素的数量。如果值比原栈顶高，则高的部分nil补足，如果值比原栈低，则原栈高出的部分舍弃。所以可以用lua_settop(0)来清空栈。
三、C++调用Lua
我们经常可以使用Lua文件来作配置文件。类似ini，xml等文件配置信息。现在我们来使用C++来读取Lua文件中的变量、table、函数。
现在有这样一个hello.lua 文件：
```
str = "I am so cool"
tbl = {name = "dong", id = 012193}
function add(a,b)
    return a + b
end
```
我们写一个test.cpp来读取它：
```
#include <iostream>
#include <string.h>
using namespace std;
extern "C"
{
#include "lua.h"
#include "lauxlib.h"
#include "lualib.h"
}

void main()
{
//1.创建Lua状态
lua_State *L = luaL_newstate();
if (L == NULL)
{
   return ;
}

//2.加载Lua文件
int bRet = luaL_loadfile(L,"hello.lua");
if(bRet)
{
    cout<<"load file error"<<endl;
    return ;
}

//3.运行Lua文件
bRet = lua_pcall(L, 0, 0, 0);
if(bRet)
{
    cout<<"pcall error"<<endl;
    return ;
}

//4.读取变量
lua_getglobal(L, "str");
string str = lua_tostring(L, -1);
cout<< "str = " << str.c_str() << endl;     //str = I am so cool~

//5.读取table
lua_getglobal(L, "tbl"); 
lua_getfield(L, -1, "name");
str = lua_tostring(L, -1);
cout << "tbl:name = " << str.c_str() << endl;     //tbl:name = dong

//6.读取函数
lua_getglobal(L, "add");                    // 获取函数，压入栈中
lua_pushnumber(L, 10);                      // 压入第一个参数
lua_pushnumber(L, 20);                      // 压入第二个参数
int iRet= lua_pcall(L, 2, 1, 0);            // 调用函数，调用完成以后，会将返回值压入栈中，2表示参数个数，1表示返回结果个数。
if (iRet) // 调用出错
{
    const char *pErrorMsg = lua_tostring(L, -1);
    cout << pErrorMsg << endl;
    lua_close(L);
    return ;
}
if (lua_isnumber(L, -1))                    //取值输出
{
    double fValue = lua_tonumber(L, -1);
    cout << "Result is " << fValue << endl;
}

//至此，栈中的情况是：
//=================== 栈顶 =================== 
//索引类型值
// 4 int：30 
// 3 string： dong 
// 2 table: tbl
// 1 string:I am so cool~
//=================== 栈底 =================== 

//7.关闭state
lua_close(L);
return ;
}
```

知道怎么读取后，我们来看下如何修改上面代码中table的值：

```
// 将需要设置的值设置到栈中
lua_pushstring(L, "我是dong～");
// 将这个值设置到table中（此时tbl在栈的位置为2）
lua_setfield(L, 2, "name");
```
我们还可以新建一个table：
```
// 创建一个新的table，并压入栈
lua_newtable(L);
// 往table中设置值
lua_pushstring(L, "Give me a girl friend !");     //将值压入栈
lua_setfield(L, -2, "str");                       //将值设置到table中，并将Give me a girl friend 出栈
```
需要注意的是：堆栈操作是基于栈顶的，就是说它只会去操作栈顶的值。

举个比较简单的例子，函数调用流程是先将函数入栈，参数入栈，然后用lua_pcall调用函数，此时栈顶为参数，栈底为函数，所以栈过程大致会是：参数出栈->保存参数->参数出栈->保存参数->函数出栈->调用函数->返回结果入栈。
类似的还有lua_setfield，设置一个表的值，肯定要先将值出栈，保存，再去找表的位置。
再不理解可看如下例子：
```
lua_getglobal(L, "add");                // 获取函数，压入栈中
lua_pushnumber(L, 10);                  // 压入第一个参数
lua_pushnumber(L, 20);                  // 压入第二个参数
int iRet= lua_pcall(L, 2, 1, 0);        // 将2个参数出栈，函数出栈，压入函数返回结果
lua_pushstring(L, "我是dzx");           //
lua_setfield(L, 2, "name");             // 会将"我是dzx"出栈
```

另外补充一下：
lua_getglobal(L, "var")会执行两步操作：1.将var放入栈中，2.由Lua去寻找变量var的值，并将变量var的值返回栈顶（替换var）。
lua_getfield(L, -1, "name")的作用等价于 lua_pushstring(L, "name") + lua_gettable(L, -2)

