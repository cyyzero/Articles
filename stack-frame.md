---
title: 浅谈栈帧
date: 2017-07-10 19:50:17
catagories: 底层
tags: 
- 深入理解计算机系统
---
**本文内容是结合《深入理解计算机系统》第三章以及自己对于栈帧、C语言函数调用实现机制的浅薄理解。如有错误，欢迎指正。** （逃~

## 什么是栈帧##

> C语言中，每个栈帧对应着一个未运行完的函数。栈帧中保存了该函数的返回地址和局部变量。


大多数机器用程序栈来支持过程调用。机器用栈来传递过程参数，存储返回信息，保存寄存器用于以后恢复，以及本地存储。为单个过程分配的那部分栈成为栈帧（stack frame）。

![stack frame](http://images0.cnblogs.com/blog2015/688670/201507/271950018913915.png)

此图描绘了栈帧的通用结构。最顶端的栈帧用两个指针界定。在IA32机器上寄存器%ebp为帧指针，寄存器%esp为栈顶指针。在64位机器上则分别用%rbp和%rsp寄存器。**以下提到的汇编代码如不特别说明一般指的是在32位机器上的汇编结果。**

假设过程P（调用者）调用Q（被调用者）时，P会在自己栈帧存储Q的参数，另外会将P的返回地址也压入栈中。返回地址就是程序从Q返回时应该继续执行的地方。当调用返回时，这个返回地址会被pop进%eip（指令指针寄存器，同理，在64位上%rip）。

## 控制转移指令##

控制转移指令主要有call和leave和ret。

- call

  call指令有一个目标，即指令被调用过程起始指令的地址。

  call指令的效果是将返回地址入栈。返回地址也就是当前指令指针寄存器（%eip）中指向的指令的下一个指令的地址。然后指令指针寄存器会存储call的目标，即相当于实现了控制的转移。

  call之后会进入另一个函数。此时会建立新的栈帧。建立过程如下：

  ```assembly
  pushl   %ebp
  movl    %esp, %ebp
  subl    $24, %esp
  ```

  首先，保存调用者的%ebp，然后赋予%ebp栈顶的地址，最后，为当前栈帧分配空间（这里举得例子是分配24字节，视实际情况而定），由于栈增长方向为较小地址，所以减去某个值即可分配完成。BTW，GCC分配的空间一般是16字节的整数倍。

- leave

  leave指令可以使栈做好返回的准备。等价于下面的代码序列：

  ```assembly
  movl	%ebp, %esp
  pop     %ebp
  ```

  将栈顶指针与帧指针同步，然后把当前帧顶，即调用者的帧的最底位置（上一个栈帧的帧底地址）pop给%ebp，实现了调用者帧指针的恢复。

- ret

  ret的作用就是将call指令时保存的下一个指令的地址，即当前栈顶的数据pop给%eip。此时，调用者%esp也恢复成了调用前的状态。并且指令指针寄存器中的数据也已经变成call指令之后的一条指令的地址。此时，调用完美结束。

## 实例##

**以下汇编代码来自64位机器上 gcc编译产生。**

这是一个极其简单的c函数。

```c
void Q()
{
	
} 

void P()
{
	Q(); 
}
```

这是编译结果的节选：

```assembly
Q:
	pushq   %rbp
	movq	%rsp, %rbp
	popq	%rbp
	ret
P:
	pushq   %rbp
	movq	%rsp, %rbp
	subq	$32, %rsp
	call	Q
```

可以看出符合上面的讲解。P首先建立栈帧，然后调用Q，Q保存P的%rbp，建立自己的栈帧，最后返回P的控制。

## 小小进阶##

上面举的实例，以及之前的讲解中都是用的P调用Q的情况。其实P、Q是同一函数也可以啊。蛤蛤，这就拓展到了函数递归了。每次函数调用自身的时候也都是一样的原理。都是先call自身，保存下一条指令地址，接着创建新的栈帧。到达终止条件后，这些栈帧就一个个ret上一个栈帧...毅种循环。蛤蛤，知道为什么没有终止条件的递归容易栈溢出了吧。

以下是一个大家很熟悉的阶乘的递归例子（**64位机器 gcc编译**）

这是c代码：

```c
int fact(int n)
{
	if(n <= 1)
		return 1;
	else
		return n*fact(n-1);
}
```

以下是汇编节选：

```assembly
fact:
	pushq   %rbp
	movq	%rsp, %rbp
	subq	$32, %rsp
	movl	%ecx, 16(%rbp)
	cmpl	$1, 16(%rbp)
	jg	.L2
	movl	$1, %eax
	jmp	.L3
.L2:
	movl	16(%rbp), %eax
	subl	$1, %eax
	movl	%eax, %ecx
	call	fact
	imull   16(%rbp), %eax
.L3:
	addq	$32, %rsp
	popq	%rbp
	ret
```

我们可以看到递归调用一个函数本身的结果和调用其它函数是一样的。这种调用机制也适用于更加复杂的情况，比如相互递归调用（P调用Q，Q再调用P）。

---

更新（2017.7.18）：

以上谈的主要是基于X86-32架构的栈帧，而随着X86-64的出现，过程调用有着很大的不同，同时栈帧也不可避免的有一些变化。

**x86-64上的栈帧布局特点:**

- 参数（最多六个）通过寄存器传递到过程，而不用保存在栈上。
- 很多函数不需要栈帧。只有那些不能将所有局部变量都放在寄存器中的函数才需要在栈上分配空间。
- 过程的存储空间拓展到了地址低于当前栈指针的存储空间（最多低128byte）。这就可以避免push带来的开销，而且保持栈指针不变。即可以直接通过栈指针定位到存储的数据。
- 从上面的描述也可以看出来，帧指针已经没有必要存在了。即**不存在帧指针**。大多数函数在调用开始时分配所需要的整个栈存储，并保持栈指针指向固定的位置。

每个栈帧的部分就是在栈指针到保存的返回地址之间。X86-64对栈帧使用的减少，并充分发挥寄存器。而且不用帧指针，多了一个可用的寄存器，栈指针兼职了之前帧指针的作用。

至于其他不同架构上的变化，我就不是很了解了。

ps：GCC编译的时候提供了一个选项“-fomit-frame-pointer”，用来显式的不使用帧指针。以下是文档说明

> Don't keep the frame pointer in a register for functions that don't need one. This avoids the instructions to save, set up and restore frame pointers; it also makes an extra register available in many functions. It also makes debugging impossible on some machines.
>
> On some machines, such as the VAX, this flag has no effect, because the standard calling sequence automatically handles the frame pointer and nothing is saved by pretending it doesn't exist. The machine-description macro "FRAME_POINTER_REQUIRED" controls whether a target machine supports this flag.

亲测gcc在开启优化的情况下也会省去帧指针。不过缺省情况(不开启优化并且不加“-fomit-frame-pointer”选项)将使用帧指针，建立栈帧的过程与本篇前半段的讲解几乎相同。