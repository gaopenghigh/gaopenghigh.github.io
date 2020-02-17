---
layout: post
title: Linux 系统调用的原理
date: 2013-12-17 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 系统调用的原理

原始的系统调用是通过中断向量80所代表的的中断来实现：把系统调用号存到一个寄存器
中，然后发出int 0x80。已经注册号的中断处理程序会检测寄存器的内容，根据不同的系
统调用号， 提供具体的服务。中断的处理过程分三步：a)保持寄存器的内容；b)调用中断
处理程序进行处理；c)重新载入之前寄存器的内容，这个过程的开销并不小，所以
int 0x80的方式，对系统的开销有较大的影响。

后来，Intel推出了两个指令sysenter/sysexit（对于64位系统则是syscall/sysret指令）
，提供 “快速系统调用”的功能，几年后，Linux开始使用这两个指令实现系统调用，并且
引入了vDSO的机制。具体是怎么使用的，可以参考
[Sysenter Based System Call Mechanism in Linux 2.6](http://articles.manugarg.com/systemcallinlinux2_6.html)。


vDSO的大概过程：

1.  内核使用sysenter等指令实现系统调用，这部分代码中主要过程叫做
    `__kernel_vsyscall`（封装在一个so中）;
2.  使用vDSO机制时，内核在初始化过程中，会拷贝这个so到一个物理页;
3.  内核在加载ELF执行文件时，会把这个物理页映射到用户空间，并且会将里面的函数根
    据类型设置到ELF auxiliary vectors中，`__kernel_vsyscall`的地址就会设置到
    `AT_SYSINFO`类型中；使用ldd查看的话，可以看到ELF文件包含一个叫做
    linux-gate.so.1或者叫linux-vdso.so.1的共享库。
4.  glibc中系统调用的核心命令是： `ENTER_KERNEL call *%gs:SYSINFO_OFFSET`
5.  经过复杂的定义声明，最终调用的是ELF auxiliary vectors中AT_SYSINFO类型的地址
    ，也就是调用了__kernel_vsyscall

一般情况下，应用程序不直接使用系统调用，而是使用一些glibc封装好的函数，不过内核
也提供了syscall()方法，让应用程序可以直接发起一个系统调用。

有一些系统调用，比如gettimeofday()，可能会被非常频繁地调用，并且这些系统调用的
具体过程都类似，读取一个有内核维护的变量的值（比如时间），然后把该值返回到用户
空间。对于这些系统调用，内核提供了一种“捷径”，使这些系统调用事实上不进入内核空
间，而是在用户空间执行。大概的原理就是在vDSO的虚拟动态共享库文件中维护了一个变
量，该变量是由内核更新的，vDSO中只是简单地读取这个变量并返回给调用方，而无需再
进行sysenter等指令。

----

参考资料：

* [linux下系统调用的实现](http://www.pagefault.info/?p=99)
* [On vsyscalls and the vDSO](https://lwn.net/Articles/446528/)
* [linux下系统调用的实现](http://www.pagefault.info/?p=99)
* [About ELF Auxiliary Vectors](http://articles.manugarg.com/aboutelfauxiliaryvectors.html)
* [Sysenter Based System Call Mechanism in Linux 2.6](http://articles.manugarg.com/systemcallinlinux2_6.html)
* [What is linux-gate.so.1?](http://www.trilithium.com/johan/2005/08/linux-gate/)
* [Creating a vDSO: the Colonel's Other Chicken](http://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken?page=0,0)

