---
layout: post
title: Linux 内存管理(5)：进程地址空间的结构
date: 2013-11-09 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 内存管理(5): 进程地址空间的结构

以下内容严重参考了
[Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory)。

## 内核空间和用户空间

每个进程运行在自己的内存沙盒中，这个沙盒就是所谓的进程的地址空间
（address space）。线性内存内存和物理内存主要通过“页表”进行转换，每个进程都有自
己的页表。由于内核必须随时随地都能运行，所以在虚拟地址空间中都有固定的一部分是
属于内核的。叫做内核空间（Kernel Space），剩下的部分叫做用户空间
(User Mode Space)。

对于32位系统来说，整个虚拟内存空间的大小是4G，其中内核空间占用了1G，用户空间
使用了3G。

     __________________
    | Kernel Space(1G) | 0xffffffff
    |__________________| 0xc0000000
    |                  |
    | User Mode Space  |
    |       (3G)       |
    |                  |
    |                  |
    |__________________| 0x00000000


进程切换的过程中，内核空间不变，只不过使用的用户空间是各个进程自己的地址空间。
如图：

![](/assets/virtualMemoryInProcessSwitch.png)

上图中，蓝色部分表示的虚拟内存已经映射到了物理内存中，白色部分没有映射。

## 进程地址空间的结构

上图中，不同的带区代表了不同的段。这些段主要有：
Stack，Heap，Memory Mapping Segment，BSS Segment，Data Segment，Text Segment。

如图：

![](/assets/linuxFlexibleAddressSpaceLayout.png)

简单介绍如下：

**Stack**

进程地址空间的最顶端是栈。
栈存放进程临时创建的局部变量，也就是在'{}'之间的变量（但不包括static
声明的变量，Static声明的变量存放在Data Segment中），
另外，在函数被调用时，其参数也会被压入发起调用的进程栈中，并且待到调用结束后，
函数的返回值也会被存放回栈中。

**Memory Mapping Segment**

这个段在栈的底下，存放文件映射（包括动态库）和匿名映射的数据。
进程可以通过 mmap() 系统调用进行文件内存映射。

有一种映射没有对应到任何文件。叫做匿名映射(TODO：给出参考链接)。
在C标准库中，如果malloc一块大于某个大小的内存，C库就会创建一个匿名映射。

**Heap**

在Memory Mapping Segment底部就是堆(Heap)。
堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减。
当进程调用malloc等函数分配内存时，新分配的内存就通过 brk() 系统调用被动态添加到
堆上（堆被扩张），当利用free等函数释放内存时，被释放的内存从堆中被剔除（堆被缩
减）。

**BSS Segment**

包含了程序中未初始化的全局变量，在内存中bss段全部置零。该段是匿名(anonymous)的
，它没有映射到任何文件。

**Data Segment(数据段)**

该段包含了在代码中初始化的变量。这个区域不是匿名(anonymous)的。它映射到了程序二
进制文件中包含了初始化静态变量的那个部分。

**Text Segment(代码段)**

该段存放可执行文件的操作指令，也就是说这个段是可执行文件在内存中的镜像。

**段之间的随机偏移**

从图中可以看到，段之间会有一个随机的偏移（offset），这是因为几乎所有的进程都拥
有一样的内存结构，如果让进程获取到各个段的真实地址，就会有安全上的风险。

可以参考[Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory)
中所说的：

> When computing was happy and safe and cuddly, the starting virtual addresses
> for the segments shown above were exactly the same for nearly every process in
> a machine. This made it easy to exploit security vulnerabilities remotely. An
> exploit often needs to reference absolute memory locations: an address on the
> stack, the address for a library function, etc. Remote attackers must choose
> this location blindly, counting on the fact that address spaces are all the
> same. When they are, people get pwned. Thus address space randomization has
> become popular. Linux randomizes the stack, memory mapping segment, and heap
> by adding offsets to their starting addresses. Unfortunately the 32-bit
> address space is pretty tight, leaving little room for randomization and
> hampering its effectiveness.


### 一个内存结构的例子

使用 pmap 工具或者查看 '/proc/[pid]/maps' 就可以看到一个进程的内存结构。

示例程序(a.c)如下：

    #include <stdio.h>
    #include <fcntl.h>
    #include <sys/mman.h>

    int main() {
        int i;
        int fd;
        int *map;
        fd = open("a.c", O_RDONLY);
        map = mmap(0, 10, PROT_READ, MAP_SHARED, fd, 0);
        scanf("%d", &i);
    }

程序中打开了一个文件并且mmap了一部分内容到内存中。使用 pmap 查看该进程如下：

    $ pmap -d 12340
    12340:   ./a.out
    Address           Kbytes Mode  Offset           Device    Mapping
    0000000000400000       4 r-x-- 0000000000000000 008:00003 a.out
    0000000000600000       4 r---- 0000000000000000 008:00003 a.out
    0000000000601000       4 rw--- 0000000000001000 008:00003 a.out
    00007f3dd91c5000    1784 r-x-- 0000000000000000 008:00001 libc-2.17.so
    00007f3dd9383000    2044 ----- 00000000001be000 008:00001 libc-2.17.so
    00007f3dd9582000      16 r---- 00000000001bd000 008:00001 libc-2.17.so
    00007f3dd9586000       8 rw--- 00000000001c1000 008:00001 libc-2.17.so
    00007f3dd9588000      20 rw--- 0000000000000000 000:00000   [ anon ]
    00007f3dd958d000     140 r-x-- 0000000000000000 008:00001 ld-2.17.so
    00007f3dd9786000      12 rw--- 0000000000000000 000:00000   [ anon ]
    00007f3dd97ab000       4 rw--- 0000000000000000 000:00000   [ anon ]
    00007f3dd97ac000       4 r--s- 0000000000000000 008:00003 a.c
    00007f3dd97ad000       8 rw--- 0000000000000000 000:00000   [ anon ]
    00007f3dd97af000       4 r---- 0000000000022000 008:00001 ld-2.17.so
    00007f3dd97b0000       8 rw--- 0000000000023000 008:00001 ld-2.17.so
    00007fff7c355000     132 rw--- 0000000000000000 000:00000   [ stack ]
    00007fff7c3fe000       8 r-x-- 0000000000000000 000:00000   [ anon ]
    ffffffffff600000       4 r-x-- 0000000000000000 000:00000   [ anon ]
    mapped: 4208K    writeable/private: 196K    shared: 4K

可以看到，进程使用了4208K的空间，但真正自己消耗的只有196K，其他都是被共享库什么
的占用了。查看'/proc/[pid]/maps'中的内容如下：

    $ cat /proc/12340/maps
    00400000-00401000 r-xp 00000000 08:03 2364279                            /home/gp/tmp/a.out
    00600000-00601000 r--p 00000000 08:03 2364279                            /home/gp/tmp/a.out
    00601000-00602000 rw-p 00001000 08:03 2364279                            /home/gp/tmp/a.out
    7f3dd91c5000-7f3dd9383000 r-xp 00000000 08:01 394355                     /lib/x86_64-linux-gnu/libc-2.17.so
    7f3dd9383000-7f3dd9582000 ---p 001be000 08:01 394355                     /lib/x86_64-linux-gnu/libc-2.17.so
    7f3dd9582000-7f3dd9586000 r--p 001bd000 08:01 394355                     /lib/x86_64-linux-gnu/libc-2.17.so
    7f3dd9586000-7f3dd9588000 rw-p 001c1000 08:01 394355                     /lib/x86_64-linux-gnu/libc-2.17.so
    7f3dd9588000-7f3dd958d000 rw-p 00000000 00:00 0
    7f3dd958d000-7f3dd95b0000 r-xp 00000000 08:01 394329                     /lib/x86_64-linux-gnu/ld-2.17.so
    7f3dd9786000-7f3dd9789000 rw-p 00000000 00:00 0
    7f3dd97ab000-7f3dd97ac000 rw-p 00000000 00:00 0
    7f3dd97ac000-7f3dd97ad000 r--s 00000000 08:03 2364727                    /home/gp/tmp/a.c
    7f3dd97ad000-7f3dd97af000 rw-p 00000000 00:00 0
    7f3dd97af000-7f3dd97b0000 r--p 00022000 08:01 394329                     /lib/x86_64-linux-gnu/ld-2.17.so
    7f3dd97b0000-7f3dd97b2000 rw-p 00023000 08:01 394329                     /lib/x86_64-linux-gnu/ld-2.17.so
    7fff7c355000-7fff7c376000 rw-p 00000000 00:00 0                          [stack]
    7fff7c3fe000-7fff7c400000 r-xp 00000000 00:00 0                          [vdso]
    ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

可以看到在改进程的地址空间中，地址从低到高，一开始分别是"Text Segment"，
"Data Segment" 和 "BSS Segment"，然后是libc，ld以及程序中做的 a.c 这个文件的映
射。接下来是进程的栈，在栈之后是 vdso 和 vsyscall。简单地说，vdso和vsyscall是
内核的一种机制，能让进程使用某些系统调用时（比如获取时间的系统调用）运行得更快
。具体可参考
[On vsyscalls and the vDSO](http://lwn.net/Articles/446528/)，
[VDSO](http://blog.csdn.net/wlp600/article/details/6886162)。

TODO:上面的输出内容中，libc有一个VMA的Mode是`----`不知道是什么意思？

    00007f3dd9383000    2044 ----- 00000000001be000 008:00001 libc-2.17.so

----

参考资料：

* [Linux内核设计与实现](http://book.douban.com/subject/6097773/)
* [深入分析Linux内核源码](http://oss.org.cn/kernel-book/ch06/6.3.1.htm)
* [深入理解Linux内核](http://book.douban.com/subject/2287506/)
* [The Linux Kernel](http://www.win.tue.nl/~aeb/linux/lk/lk.html)
* [Anatomy of a Program in Memory](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory)
* [Memory Translation and Segmentation](http://duartes.org/gustavo/blog/post/memory-translation-and-segmentation)
* [How The Kernel Manages Your Memory](http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory)
* [Page Cache, the Affair Between Memory and Files](http://duartes.org/gustavo/blog/category/linux)
* [The Thing King](http://duartes.org/gustavo/blog/post/the-thing-king)
* [CPU Rings, Privilege, and Protection](http://duartes.org/gustavo/blog/post/cpu-rings-privilege-and-protection)

