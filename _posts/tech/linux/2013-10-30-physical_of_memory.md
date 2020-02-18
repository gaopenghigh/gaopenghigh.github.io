---
layout: post
title: 内存相关的硬件基础知识
date: 2013-10-30 10:00:00 +0800
categories: Tech
toc: true
---

# 内存相关的硬件基础知识

### 主板上的芯片们

TODO

### 内存相关的硬件知识

感谢[Gustavo Duarte](http://duartes.org/gustavo/blog/)，画了下面的双核结构图：

![](/assets/physicalMemoryAccess.png)

CPU通过 前端总线，即Front Side Bus(FSB)连接到北桥芯片（Northbridge），再连接到
内存模块。上图列出了主要的3组针脚：“Address Pins”，“Data Pins”和“Request Pins”
。在FSB中的数据交互的**事务（transaction）**中，主要由这三种数据起作用。

一个FSB的事务分为5个阶段：arbitration, request, snoop, response, data.
(TODO: 每种阶段的意义）。FSB有不同的组件，通常包括处理器和北桥芯片，这些组件被
称为**agents**。在每个不同的阶段，agents扮演了不同的角色。

#### FSB 的 request阶段

我们主要看一下 request 阶段到底发生了哪些事情。在 request 阶段，两个
包（packet）通过agents输出。

第一个包通过 address pins 和 request pins 输出，其内容如下：

![](/assets/fsbRequestPhasePacketA.png)

通过 request pins 输出的内容则代表了不同的request类型。

通过 address pins 输出的 address line ，代表这次事务需要操作的物理内存的起始地
址，但当request类型是I/O read和I/O write时，其输出的内容代表I/O port。

第一个包发送后，第二个包在接下来的时钟周期内发送。第二个包的数据如下：

![](/assets/fsbRequestPhasePacketB.png)

通过部分 address pins 输出的 attribute signals 代表了Intel处理器中5种不同的缓存
方式：

*   UC, uncacheable, 不做缓存
*   WC, Write-combining, 即把需要写入的信息先存在缓存中，积攒到一定程度后在真的
    写入。参考[Write-combining](http://en.wikipedia.org/wiki/Write-combining)
*   WT, Write-through, 表示读操作具有缓存，写操作直接写入。
*   WP, Write-protected, 写保护，不可真正写入
*   WB, Write-back, 回写，类似于Linux的Page Cache，数据的变更先在缓存中完成，
    “回头”在真的写入。

通过 attribute signals, request agend 让其他处理器知道这次事务将会如何影响它们
的缓存，并让内存控制器（北桥）知道该如何行为。

一般的内核都以“回写”模式处理所有内存，一般情况下有最好的性能。在回写模式（WB）
下，内存访问的单位是**cache line**，即使程序只需要一个bit的数据，处理器也会加载
整个cache line大小的数据到缓存。（类似于内核中的page frame和硬盘中的block）。

下图中的例子代表了需要从内存获取一份数据的过程：

![](/assets/memoryRead.png)

在Intel架构中，一部分内存区域被映射到一些设备，比如网卡，于是驱动程序对这些设备
的操作，就是通过对这部分内存区域的读写完成。内核在页表（page tables）中把这部分
区域标注为Uncacheable。通过第二个包中的“Byte Enable”掩码，对Uncacheable内存区域
的访问，可以只单独地只读或写一个字节，一个字等等。

#### 分析

通过对与内存相关的硬件的理解，可以得到一些应用结论：

*   对性能要求很高的程序，应该尽可能地把需要一起访问的数据放到一个cache line大
    小的结构中，这样就可以直接用CPU的缓存而不是使用内存。
*   回写模式下，对一个单独cache line中的内容的操作都是原子（atomic）的。对这些
    内容的访问都发生在处理器的L1缓存中，一次性读出或者写入，不可能中途被其它处
    理器或者线程影响。
*   Front bus被所有的agents共享使用，在一个agents开始一个事务前，必须先经过
    arbitrate（仲裁）来决定由谁来使用。为了保持缓存的一致性，每个agents必须监听
    所有的事务。于是，当越来越多的处理器加入进来时，bus的争用越来越严重。i7内核
    发展出了每个内核对内存直接访问的技术来解决bus争用的问题。


----

参考资料：

*   [Getting Physical With Memory](http://duartes.org/gustavo/blog/post/getting-physical-with-memory)
