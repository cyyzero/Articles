---
title: 当我们谈论系统调用的时候我们在谈论什么
date: 2018-1-29 17:59:56
tags:
- Linux
---

ps:句式“当我们谈论XX时我们在谈论什么”引用自美国作家卡佛(Ramond Carver)的作品《当我们谈论爱情时我们谈论什么》(What We Talk About When We Talk About Love)。当然，我又标题党了，应该换成**浅析IA32 Linux 系统调用过程**比较合适。

当说到Linux下的系统调用(**System Call**)的时候，程序员都不会陌生。但一般都会联想到fork()，execve()等这些函数。这就是很多人(包括以前我自己)的误区所在了。

诸如fork() 这些，是c语言函数库(linux下常见如glibc)实现的**外壳(wrapper)函数**，由它们来引发真正的系统调用。从C语言编程的角度来看，调用C语言函数库的外壳函数等同于发起系统调用。也就是说，当我们在代码中使用诸如fork()这类函数时，意味着调用外壳函数，然后由外壳函数去调用系统调用(可能有点绕)。

区分了狭义上的外壳(wrapper)函数和系统调用后，就能进入本文的主题了:**IA32上Linux系统调用的过程**。(因为其他的不懂...

---

首先描述两个概念，**中断(interruption)**和**异常(execption)**。从[Intel的手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)上找的简单介绍。

>The processor provides two mechanisms for interrupting program execution, interrupts and exceptions: 

>• An interrupt  is an asynchronous event that is typically triggered by an I/O device.

>• An exception is a synchronous event that is generated when the processor detects one or more predefined conditions while executing an instruction. The IA-32 architecture specifies three classes of exceptions: faults, traps, and aborts. 

简单地说，中断和异常是处理器提供的中断程序执行的机制，能引起X86挂起当前指令流的执行并相应事件。在这两种情况下处理器都会保存当前进程的上下文，并将转至一个预先定义的子程序来执行特殊的服务。

虽说中断和异常概念上有所区别，但在IA32上处理它们却是用了相同的方式。系统为每种类型的中断和异常分配了唯一的一个无符号整型作为一个向量(vector)，即可以理解为索引。然后内核维护的一张叫做**中断描述符表**(interrupt descriptor table， 一般简称为**IDT**)的结构，表中每一项都是某个中断或者异常对应的处理程序的入口地址。然后每次发生了某个中断和异常，就能通过它的向量，就在IDT中找到相应的处理程序并调用。

具体到汇编语言，就是通过INT指令来实现。INT(interrupt的缩写)指令的作用是引发中断和异常的处理程序，格式形如INT n。

从[Intel的手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)上找的INT指令的简单介绍。
>The INT n instruction generates a call to the interrupt or exception handler specified with the destination operand The destination operand specifies a vector from 0 to 255, encoded as an 8-bit unsigned intermediate value. Each vector provides an index to a gate descriptor in the IDT. The first 32 vectors are reserved by Intel for system use. Some of these vectors are used for internally generated exceptions.

大意就是INT n用来调用特定的中断或者异常的处理程序。n取值从0x00-0xFF(8位无符号整型)。N就作为上述提到的向量。举一些简单的例子，比如说0x1表示除法上溢或者被零除，0x12表示栈故障(越界或者栈段不存在)，0x14表示缺页...

当调用处理程序时，系统栈保存处理器的状态。下列事件将发生:
* 若转移涉及特权指令改变，则当前栈段寄存器和当前拓展的栈指针(esp)寄存器的内容被压入栈。
* EFLAGS寄存器的当前值压入栈。
* 中断(IF)和自陷(TF)两个标志被清除。这就禁止了INTR中断，自陷或单步中断。
* 当前代码段(CS)寄存器和当前指令指针寄存器(IP或者EIP)寄存器被压入栈。
* 若中断向量伴随错误代码，则错误代码也入栈。
* 读取中断向量表对应项的内容，将其装入CS和IP(或EIP)寄存器。控制转移到终端服务子程序继续执行。

为从中断返回，中断服务程序执行一条IRET指令。这使得所有保存在栈上的值被取回，并由中断点恢复执行。

---

铺垫了这么久，又回到了系统调用。没猜错，IA32的Linux上系统调用就是通过INT 0x80调用的。

下面讲一下具体过程：

响应中断0x80，内核调用system\_call()。可以再展开说一下，这里通过各个寄存器用于传递参数，EAX寄存器用于表示系统调用的接口号，比如EAX=1用于退出进程(exit)；EAX=2表示创建进程(fork)；EAX=3表示读取文件或者IO(read)等，每个系统调用都对应内核源代码中的一个函数，他们都是以“sys_”开头的，比如说exit调用对应于内核中的sys\_exit函数。你没猜错，和IDT一样，系统也维护着一张系统调用表，通过类似*sys\_call\_table(0,%eax,4)，的跳转，可以通过eax所记录的系统调用号调用对应的系统调用函数。如果系统调用带有函数参数(一般也是通过寄存器传递)，那么还会检查参数的有效性。随后，该函数会执行必要的任务。接着，将状态返回sys\_call()函数。从内核栈中恢复各寄存器值，并将系统调用返回结果置于栈中。最后返回外壳函数，中断结束。

至此，系统调用算是结束了，但是我们调用的外壳(wrapper)函数还没算完。它还有个扫尾工作。如果返回值表明调用有错误，外壳(wrapper)函数就会设置全局变量errno。最后，外壳(wrapper)函数会返回到调用它的地方，并返回一个整型，以表明系统调用是否成功。

终于调用结束，大功告成了...

(取好几本书内容的子集东拼西凑写出来的...真是艰难
