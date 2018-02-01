---
title:  nullptr vs NULL
date: 2017-11-04 10:13:15
tags:
- c++
---

大家都知道,在C语言中,NULL被用来表示空指针常量(null pointer constant),用来与任何指向真实对象的指针区分.

c99标准里这么描述:
> 3An integer constant expression with the value 0, or such an expression cast to type void *, is called a null pointer constant.55) If a null pointer constant is converted to a pointer type, the resulting pointer, called a null pointer, is guaranteed to compare unequal to a pointer to any object or function.

而且NULL属于implementation defined,所以编译器一般都把NULL宏定义为常量0(或者0L)或者 (void*)0...

因为c语言允许void* 指针向其他指针类型隐式转化(比如说很常用的malloc函数的返回值就是void* ),所以可以通过NULL给任意类型的指针赋null pointer constant.

但是在c++中就有不同了.c++不支持void* 隐式转化为其他指针类型,NULL宏定义成(void*)0就没什么意义. 于是,c++中NULL一般宏定义为0. 

但是这么以来,问题就出现了
```c++
#include <iostream>
void f(void *)
{
	std::cout << "arg: pointer" << std::endl;
}

void f(int )
{
	std::cout << "arg: int" << std::endl;
}
int main()
{
	int *p  = NULL;
	f(p);
	return 0;
}
```
编译出错:
```
test.cpp:14:8: error: call of overloaded ‘f(NULL)’ is ambiguous
  f(NULL);
        ^
test.cpp:2:6: note: candidate: void f(int*)
 void f(int *)
      ^
test.cpp:7:6: note: candidate: void f(int)
 void f(int )
      ^
```
因为函数调用的时候两个重载版本都会匹配.

还有就是模板推断参数类型的时候,也会由于NULL而把指针类型推断成int(当然,如果宏定义是0L就推断成long long),造成编译错误.

当然,随着c++11版本的出现,引入了一个新的关键字nullptr,c++终于有了正式的null pointer constant.

> The keyword nullptr denotes the pointer literal. It is a prvalue of type std::nullptr_t. There exist implicit conversions from nullptr to null pointer value of any pointer type and any pointer to member type. Similar conversions exist for any null pointer constant, which includes values of type std::nullptr_t as well as the macro NULL. 

大概意思就是nullptr类型是std::nullptr_t(大概可以理解成true和false之于bool类型的关系),是个纯右值(prvalue).nullptr可以隐式转化成任意指针类型的空指针值(null pointer value).

再来看一个例子:
```c++
#include <cstddef>
#include <iostream>
template<class F, class A>
void Fwd(F f, A a)
{
    f(a);
}
 
void g(int* i)
{
    std::cout << "Function g called\n";
}
 
int main()
{
    g(NULL);           // Fine
    g(0);              // Fine
 
    Fwd(g, nullptr);   // Fine
    Fwd(g, NULL);      // ERROR: No function g(int)
}
```
有了nullptr就解决了NULL模板推断的一些问题.

所以给指针赋值尽量用nullptr...

ps: 还要就是std::nullptr_t类型的定义比较奇怪= =.  这样真的大丈夫?
```
typedef decltype(nullptr) nullptr_t;
```
