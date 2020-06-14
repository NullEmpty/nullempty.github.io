---
title: Lua与C/C++交互小结
date: 2020-05-22 22:39:17
categories: 
    - 开发
tags: 
    - Lua
toc: true # 是否启用内容索引
---

[Lua](https://www.lua.org/manual/5.3/)是一种强大、高效、轻量级、可嵌入的脚本语言。它支持过程式编程、面向对象编程、函数式编程、数据驱动编程和数据描述。

[Lua C Api](https://www.lua.org/manual/5.3/manual.html#4.8)提供了一组宿主程序与Lua通信的C函数，所有API函数和相关类型和常量都在头文件lua.h中声明。

## I. lua栈介绍

Lua使用一个虚拟堆栈与C相互传递值。这个堆栈中的每个元素表示一个Lua值(nil, number, string等)。API中的函数可以通过它们接收的LuaState参数来访问这个堆栈。

每当Lua调用C时，被调用的函数就会得到一个新的堆栈，它独立于以前的堆栈和仍然处于活动状态的C函数的堆栈。这个堆栈最初包含C函数的所有参数，C函数可以在这里存储临时Lua值，并且必须将结果push到堆栈来返回给Lua调用者。

可以使用索引来引用堆栈中的任何元素。正数索引表示从栈底开始向栈顶索引(从1开始)，负数索引表示从栈顶开始向栈底索引（从-1开始）。举个例子：一个有n个元素的堆栈，-1和n均是栈顶元素的索引，1和-n均是栈底元素的索引。

![img][stack]

## II. 常用c api

+ lua_gettop
```C
//返回栈顶元素的索引。 因为索引是从1开始的，所以这个结果等于栈上的元素个数；0表示栈为空。
int lua_gettop (lua_State *L);
```

+ lua_settop
```C
//index参数允许传入任何索引以及0。它将把堆栈的栈顶设为这个索引。如果新的栈顶比原来的大，超出部分的新元素将被填为nil。如果index为0，将把栈上所有元素移除。
void lua_settop (lua_State *L, int index);
```

+ lua_getglobal
```C
//把全局变量name里的值压栈，返回该值的类型。
int lua_getglobal (lua_State *L, const char *name);
```

+ lua_setglobal
```C
//从堆栈上弹出一个值，并将其设为全局变量name的新值。
void lua_setglobal (lua_State *L, const char *name);
```

+ lua_pushinteger
```C
//把值为n的整数压栈。其他lua_pushxxx类似。
void lua_pushinteger (lua_State *L, lua_Integer n);
```

+ lua_call
```C
//调用一个Lua函数。nargs是C函数压入栈的参数个数, nresults是Lua函数执行后的返回值个数。
void lua_call (lua_State *L, int nargs, int nresults);
```

+ luaL_checknumber
```C
//检查函数的第arg个参数是否是一个数字，并返回这个数字
//其他luaL_checkxxx同理
lua_Number luaL_checknumber (lua_State *L, int arg);
```

+ lua_gettable
```C
//把t[k]的值压栈，这里的t是指索引index指向的table，而k则是栈顶放的值。
int lua_gettable (lua_State *L, int index);
```

+ lua_settable
```C
//做一个等价于t[k] = v的操作，这里t是给出的索引index处的table，v是栈顶的那个值，k是栈顶之下的值。这个函数会将键和值都弹出栈。跟在Lua中一样，这个函数可能触发元方法
void lua_settable (lua_State *L, int index);
```

[这里](https://www.lua.org/manual/5.3/manual.html#4.8)可以查看全部的api。


## III. C调用Lua

### 3.1 在C中调用Lua的方法核心是lua_call、lua_pcall等方法

```C
//调用一个函数。
void lua_call (lua_State *L, int nargs, int nresults);
//以保护模式调用一个函数。
int lua_pcall (lua_State *L, int nargs, int nresults, int msgh);
```

>调用**lua_call**须遵循以下协议：
首先，要调用的函数应该被压入栈；
接着，把需要传递给这个函数的参数按正序压栈；
最后调用一下lua_call；
当函数调用完毕后，所有的参数以及函数本身都会出栈。而函数的返回值这时则被压栈。返回值的个数为nresults个，除非nresults被设置成LUA_MULTRET。
在这种情况下，所有的返回值都被压入堆栈中。 Lua会保证返回值都放入栈空间中。函数返回值将按正序压栈。

>**lua_pcall**的参数nargs和nresults的含义与lua_call中的相同。如果在调用过程中没有发生错误，lua_pcall的行为和lua_call完全一致。 但是，如果有错误发生的话，lua_pcall会捕获它，然后把唯一的值（错误消息）压栈，然后返回错误码。 同lua_call一样，lua_pcall总是把函数本身和它的参数从栈上移除。

### 3.2 下面例子在C中调用Lua中的函数及修改Lua中的变量

include头文件
```C
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
```

test.lua代码
```Lua
function add(a, b)
    return a + b
end

food = "rice"
```

C代码
```C
//调用lua中的add方法
int callLuaFuncAdd(lua_State* L, int a, int b) {
    lua_getglobal(L, "add");
    lua_pushinteger(L, a);
    lua_pushinteger(L, b);
    lua_call(L, 2, 1);
    int result = (int)lua_tointeger(L, -1);
    printf("call lua func add(%d,%d)=%d \n", a, b, result);
    return result;
}

//修改lua中的变量food
void modifyLuaVariable(lua_State* L) {
    lua_getglobal(L, "food");
    printf("food=%s \n", lua_tostring(L, -1));
    
    lua_pushstring(L, "apple");
    lua_setglobal(L, "food");
    lua_getglobal(L, "food");
    printf("after modify, food=%s \n", lua_tostring(L, -1));
}

int main(int argc, const char * argv[]) {
    lua_State* L = luaL_newstate();
    //打开指定状态机中的所有Lua标准库
    luaL_openlibs(L);
    //加载并运行指定的lua文件
    luaL_dofile(L, "test.lua");
    callLuaFuncAdd(L, 1, 2);
    modifyLuaVariable(L);
    //销毁指定Lua状态机中的所有对象
    lua_close(L);
    return 0;
}
```

运行结果:
>call lua func add(1,2)=3 
food=rice 
after modify, food=apple 

## IV. Lua调用C

在Lua中调用C函数，需要定义C函数并将C函数注册到Lua中。

### 4.1 C函数的定义

```C
typedef int (*lua_CFunction) (lua_State *L);
```
为了正确的和Lua通讯，C函数必须使用下列协议。这个协议定义了参数以及返回值传递方法：C函数通过 Lua中的栈来接受参数，参数以正序入栈。因此，当函数开始的时候，lua_gettop(L)可以返回函数收到的参数个数。第一个参数（如果有的话）在索引1的地方，而最后一个参数在索引lua_gettop(L)处。 当需要向Lua返回值的时候，C函数只需要把给Lua的返回值以正序压到堆栈上，然后return这些返回值的个数。和Lua函数一样，从Lua中调用C函数也可以有很多返回值。

### 4.2 注册C函数

```C
//把C函数f设到Lua全局变量name中。
void lua_register (lua_State *L, const char *name, lua_CFunction f);
//它通过下面的宏来定义：
#define lua_register(L,n,f) \
    (lua_pushcfunction(L, f), lua_setglobal(L, n))
```

### 4.3 批量注册C函数

```C
//创建一张新的表，并把列表l中的函数注册进去。
void luaL_newlib (lua_State *L, const luaL_Reg l[]);

//它是用下列宏实现的：
#define luaL_newlib(L,l)  \
  (luaL_checkversion(L), luaL_newlibtable(L,l), luaL_setfuncs(L,l,0))
```

### 4.4 下面例子在Lua中调用C函数

C代码
```C
//给lua调用的func, 计算参数的平均值及总和
//avg, sum = average(1, 2, 3, ...)
int average(lua_State* L) {
    //获取参数个数
    int size = lua_gettop(L);
    double num = 0;
    for (int i = 1; i <= size; i++) {
        if(!lua_isnumber(L, i)) {
            lua_pushstring(L, "error, args number only");
            lua_error(L);
        }
        num += lua_tonumber(L, i);
    }
    //将第一个返回值avg压入栈
    lua_pushnumber(L, num / size);
    //将第二个返回值sum压入栈
    lua_pushnumber(L, num);
    //return返回值的个数
    return 2;
}

//给lua调用的func, 计算2个参数之和
//calculator.add(a, b)
int add(lua_State* L) {
    int n1 = luaL_checknumber(L, -1);
    int n2 = luaL_checknumber(L, -2);
    lua_pushnumber(L, n1 + n2);
    return 1;
}

//给lua调用的func, 计算2个参数之差
//calculator.sub(a, b)
int sub(lua_State* L) {
    int n1 = luaL_checknumber(L, -1);
    int n2 = luaL_checknumber(L, -2);
    lua_pushnumber(L, n2 - n1);
    return 1;
}

//用于luaL_setfuncs注册函数的数组类型。name指函数名，func是函数指针。任何luaL_Reg数组必须以一对name与func皆为NULL结束。
static const struct luaL_Reg calculatorLib [] = {
    {"add", add},
    {"sub", sub},
    {NULL, NULL}
};

//将calculatorLib的add、sub等方法赋给calculator
int luaopen_calculator(lua_State* L) {
    luaL_newlib(L, calculatorLib);
    lua_setglobal(L, "calculator");
    return 1;
}

int main(int argc, const char * argv[]) {
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);
    //1 给lua注册名为name的c函数f，此时lua调用函数name。
    lua_register(L, "average", average);
    //2 注册多个函数
    luaopen_calculator(L);
    luaL_dofile(L, "test.lua");
    lua_close(L);
    return 0;
}
```

test.lua代码
```Lua
avg, sum = average(10, 20, 33, 40, 50)
print("avg=", avg, ", sum=", sum)

local r = calculator.add(10, 24)
print("calculator.add(10, 24)=", r)
r = calculator.sub(10, 24)
print("calculator.sub(10, 24)=", r)
```

运行结果：
>avg=	30.6	, sum=	153.0
calculator.add(10, 24)=	34.0
calculator.sub(10, 24)=	-14.0

## V. Lua调用C++的处理

### 5.1 大概原理

前面都是纯C函数调用，没有面向对象，为了实现Lua调用C++对象的方法一般使用userdata配合元表来实现。

userdata是Lua提供了的一个基本类型。userdata提供了一个Lua中的原始内存区域可以用来保存内存中的对象，userdata自身没有定义任何操作，可以给userdata设置元表[metatable](https://www.lua.org/manual/5.3/manual.html#2.4)从而通过__index元方法来实现对象相应的函数。

Lua API提供了以下函数来创建userdata:
```C
//分配给定大小的内存块，将相应的用户数据压入堆栈，并返回内存地址
void *lua_newuserdata (lua_State *L, size_t size);
```

### 5.2 具体可以按下面步骤
* 1. 通过lua_register注册C++类的构造函数
* 2. 在构造函数中lua_newuserdata新建userdata并保存C++对象的指针
* 3. 给userdata设置一个元表，让元表的元方法__index指向元表自身
* 4. 将C++对象的方法注册到元表中

### 5.3 下面例子在Lua中调用C++对象的方法

include头文件:
```C++
extern "C" {
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}
```
或者
```C++
#include <lua.hpp>
```

Book.cpp
```C++
class Book {
private:
    const char* name;
    double price;
public:
    Book(const char* name) {
        this->name = name;
    }

    const char* getName(){
        return name;
    }
    
    void setPrice(double price) {
        this->price = price;
    }
    
    double getPrice() {
        return price;
    }
};
```

C++代码
```C++
// local book = Book(name)
static int newBook(lua_State* L) {
    const char* name = lua_tostring(L, -1);
    Book** book = (Book**)lua_newuserdata(L, sizeof(Book));
    *book = new Book(name);
    luaL_setmetatable(L, "Book");
    return 1;
}

static Book* checkBook(lua_State* L){
    void* ud = luaL_checkudata(L, 1, "Book");
    if (ud == NULL) {
        luaL_argerror(L, 1, "userdata check error!");
    }
    return *(Book**)ud;
}

//local name = book:getName()
static int getName(lua_State* L) {
    Book* book = checkBook(L);
    const char* name = book->getName();
    lua_pushstring(L, name);
    return 1;
}

//book:setPrice(100)
static int setPrice(lua_State* L) {
    Book* book = checkBook(L);
    double price = lua_tonumber(L, -1);
    book->setPrice(price);
    return 0;
}

//local price = book:getPrice()
static int getPrice(lua_State* L) {
    Book* book = checkBook(L);
    lua_pushnumber(L, book->getPrice());
    return 1;
}

static const struct luaL_Reg libs[] = {
    {"getName", getName},
    {"setPrice", setPrice},
    {"getPrice", getPrice},
    {NULL, NULL}
};

int luaopen_book(lua_State* L){
    //metatable.__index = metatable
    luaL_newmetatable(L, "Book");
    lua_pushstring(L, "__index");
    lua_pushvalue(L, -2);
    lua_settable(L, -3);
    
    luaL_setfuncs(L, libs, 0);
    
    lua_register(L, "Book", newBook);
    return 1;
}

int main(int argc, const char * argv[]) {
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);
    luaopen_book(L);
    luaL_dofile(L, "test.lua");
    lua_close(L);
    return 0;
}
```

test.lua代码
```Lua
local b = Book("Geometries")
b:setPrice(56.5)
print("book.getName()=",b:getName(b))
print("book.getPrice()=",b:getPrice())
```

运行结果：
>book.getName()=	Geometries
book.getPrice()=	56.5


[stack]:data:image/png;base64,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

