---
title: c++ switch内部的变量定义
date: 2017-10-30 23:59:56
tags:
- c++
---

c++中的switch语句大家肯定都很熟悉。switch执行流程是匹配相应的case。很显然，可能会跳过某些case标签，忽略某些代码。那么就带来了一个有意思的话题：**如果被忽略的代码中有变量的定义该怎么办**？

先来看一个简单的例子：
```cpp
switch(1)
{
case 1:
	int a = 10;
	break;
case 0:
	break;
}
```
然后编译就错了：
* `error: jump to case label [-fpermissive]`
* `error: crosses initialization of ‘int a’`

编译器的意思是控制流跳过了变量的初始化。乍看之下似乎不太好理解为什么出错。其实如果在case 0标签中要使用的变量a的时候，问题就来了。试图在未初始化的时候使用对象，这显然是行不通的。可能由于举的例子中变量是内置类型，所以不太好理解到问题的严重性，再来看一个例子：
```cpp
switch(1)
{
case 1:
	std::string s1;
	std::string s2("hello");
	break;
case 0:
	//可能会使用s1或者s2
	break;
}
```
这样就跳过了类的隐式或者显式初始化。

因此c++规定，不允许跨过变量的初始化语句直接跳转到变量作用域的另一个位置。


然而，对于内置类型来说，定义的时候并非一定要初始化。
比如：
```cpp
switch(1)
{
case 1:
	int a;
	break;
case 0:
	a = 20;
	break;
}
```
这段代码就不会报错，因为变量并没有初始化。可以从底层来理解。

首先，switch下的大括号是一个完整的作用域，不要被case标签的缩进所欺骗，认为每个case有一个作用域（除非显示地加大括号）。

其次，定义变量并不存在“执行动作”，有的只是改变栈指针从而在栈上分配空间。编译器会保证在相应的作用域之中这个变量的空间是被分配了。大部分编译器实现会选择在函数开始把所有局部变量的空间都分配好。但是变量的初始化是有”执行动作“的，比如说`int a = 10`，就会在栈上某4个字节存入10（假设机器上int占4字节）。再比如某个类初始化，也会调用相应的构造函数。所以控制流程的跳转可能造成某个对象没有正常的初始化就使用。

所以，就可以回答一开始的问题了。如果被忽略的代码中的定义包含了初始化的过程，就会编译失败。如果是内置类型的定义（不含初始化），那么就能通过编译。

当然，为了不出现上述的问题，可以直接用大括号将case标签下构成一个单独的作用域。
```cpp
switch(0)
{
case 1:
	{
		std::string s;
		int a = 10;
	}
	break;
case 0:
	//显然，这里也不可能再使用对象s或者a
	break;
}
```