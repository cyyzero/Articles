---
title: c++引用的本质
date: 2017-07-13 23:59:56
tags:
- c++
---
很多c++初学者对于引用（reference）的的印象不外乎就是“对象的别名”、“同义词”。而引用作为c++对于c语言的重要扩充，与指针有着千丝万缕、剪不断理还乱的联系。甚至，引用在底层就是通过“指针”实现。

**左值引用**

接下来，看一个很简单的小例子：

```c++
void test()
{
	int a = 6;
	int &b = a;
	b = 7;
}
```

汇编后的代码节选：

```assembly
_Z4testv:
.LFB0:
	pushq	%rbp
	.seh_pushreg	%rbp
	movq	%rsp, %rbp
	.seh_setframe	%rbp, 0
	subq	$16, %rsp
	.seh_stackalloc	16
	.seh_endprologue           #以上几行是test函数栈帧的建立过程
	movl	$6, -12(%rbp)      #保存a
	leaq	-12(%rbp), %rax    #将a的地址存入rax寄存器
	movq	%rax, -8(%rbp)     #保存b，存入的是a的地址
	movq	-8(%rbp), %rax     #将b，即a的地址传入rax寄存器
	movl	$7, (%rax)         #通过b中保存的地址对a赋值
	addq	$16, %rsp          #从以下几行是test函数栈帧的销毁过程
	popq	%rbp
	ret
	.seh_endproc
```

可以看到，内存上b存储的其实是a的地址。

在底层，引用是通过指针来实现。引用算的上是c++的一个语法糖。

**右值引用**

以上的讲解都是基于c++传统的、用&表示的引用，也就是左值引用。而c++11标准还有个右值引用的概念。那右值引用也还是用指针的实现的吗？要知道，右值引用可是对于临时变量的引用，万一是存储在寄存器中的临时变量呢？要知道，寄存器内的数据可不能寻址...作为一个c++菜鸟，我对右值引用也没有很深的理解...想到这，我不禁冒冷汗，难道我的猜测错了？上面的都得重写？

吓得我赶紧试验一下...

```c++
void test()
{
	int a = 6;
	int &&b = a+1;
}
```

汇编之后的节选：

```assembly
_Z4testv:
.LFB0:
	pushq	%rbp
	.seh_pushreg	%rbp
	movq	%rsp, %rbp
	.seh_setframe	%rbp, 0
	subq	$32, %rsp
	.seh_stackalloc	32
	.seh_endprologue           #以上是栈帧的建立过程
	movl	$6, -4(%rbp)       #保存a
	movl	-4(%rbp), %eax     #将a存入寄存器eax
	addl	$1, %eax           #eax中数据+1，得到a+1
	movl	%eax, -20(%rbp)    #保存a+1
	leaq	-20(%rbp), %rax    #将保存的a+1的地址传入寄存器eax
	movq	%rax, -16(%rbp)    #保存a+1的地址
	addq	$32, %rsp          #从这以下是这个栈帧的销毁过程
	popq	%rbp
	ret
	.seh_endproc
```

可以看到，程序会先将a+1计算结果保存进栈帧，然后把a+1被保存的地址传给b。b保存的还是一个地址。

当然...我由于c++水品有限，可能这只是我错误的猜想和认识。欢迎指正我的错误所在。 

ps：“怎么指正错误啊喂，你的博客还不能评论啊啊啊啊啊”  （来自官方的吐槽- -。