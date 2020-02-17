---
layout: post
title: Linux 内存管理(3)：页框的管理
date: 2013-11-07 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 内存管理(3): 页框的管理

### 页描述符

内核必须记录每个页框当前的状态。
每个页框用一个`struct page`结构表示，通过该结构，内核知道当前这部分内存是什么
状态，在用于什么。因为内核需要知道一个页是否空闲，如果页已经被分配，内核还需要
知道谁拥有这个页。拥有者可能是用户空间进程、动态分配的内核数据、静态内核代码或
者页高速缓存等。

该结构又叫做“页描述符”，位于`<linux/mm_types.h>`中。
一个简化版的`struct page`如下：

    struct page {
        unsigned long           flags;
        atomic_t                _count;
        atomic_t                _mapcount;
        unsigned long           private;
        struct address_space    *mapping;
        pgoff_t                 index;
        struct list_head        lru;
        void                    *virtual;
    };

其中，

*   `flag`存放页的状态，包括是否为脏，是否被锁定在内存中等。
*   `_count`存放页的引用计数，-1表示这一页空闲，当前内核没有引用这一页，在新的
    分配中就可以使用它。
*   `virtual`表示页的线性地址。
*   `mapping`和`index`两个字段用于在页高速缓存中找到这个页。

通过很多的`struct page`结构，内核就可以知道整个物理内存是什么状态。

### 内存管理区

基于硬件方面的限制，内核把每个内存节点的物理内存划分为3个区（zone）:

*   `ZONE_DMA`，包含低于16MB的内存页框，区中的页用来执行DMA，即直接内存访问操作
*   `ZONE_NORMAL`，包含高于16MB且低于896MB的内存页框，此区内都是能正常映射的页
*   `ZONE_HGMEM`，包含从896MB开始的内存页框，“高端内存”，其中的页不能被永久地映
    射到内核地址空间。

对于64位Intel体系，可以映射和处理64位的内存空间，所以没有ZONE_HIGHMEM区，所有内
存都在ZONE_DMA和ZONE_NORMAL区。

当内核调用一个内存分配函数时，必须指明请求页框所在的区。

### 获得和释放页框

获得页框的最核心的函数是：

    struct page * alloc_pages(gfp_t gfp_mask, unsigned int order)

该函数分配2^order个连续的物理页，并返回一个指针，指向第一个页的page结构体。

下面的函数把给定的页框转换为线性地址：

    void * page_address(struct page *page)

如果不需要使用struct page，可以使用下面的方法直接得到线性地址：

    unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)

`gfp_mask`标志指定了内核在分配内存时，需要遵守一定行为约束，并且从需要的区中分
配内存（比如ZONE_DMA区还是ZONE_HIHGMEM区）。“行为约束”包括分配器是否可以睡眠、
是否可以启动磁盘IO、是否应该使用高速缓存中快要淘汰出去的页等等。这些规则可以组
合成不同的类型。最主要的两个`gfp_mask`类型是:

*   `GFP_KERNEL`：常规分配方式，可能会阻塞。这个标志在睡眠安全时用在进程上下文
    代码中。
*   `GFP_ATOMIC`：用在中断处理程序、下半部、持有自选锁以及其他不能睡眠的地方。

使用下面的函数释放页：

    void __free_pages(struct page *page, unsigned int order)
    void free_pages(unsigned long addr, unsigned int order)
    void free_page(unsigned long addr)

由于内核经常请求和释放单个页框，所以为了提高性能，每个内存管理区域（zone）定义
了一个“每CPU页框高速缓存”，其中包含一些预先分配的页框，它们被用于满足本地CPU发
出的单一内存请求。

### 伙伴算法

内核使用**“伙伴关系”**来管理物理内存页框，这样有利于分配出连续的内存页。参考
[Buddy memory allocation](http://en.wikipedia.org/wiki/Buddy_memory_allocation)
。

**原理**

Linux的伙伴算法把所有的空闲页面分为10个块组，每组中块的大小是2的幂次方个页面，
例如，第0组中块的大小都为2^0 （1个页面），第1组中块的大小为都为2^1（2个页面），
第9组中块的大小都为2^9（512个页面）。也就是说，每一组中块的大小是相同的，且这同
样大小的块形成一个链表。

**例子**

假设要求分配的块其大小为128个页面（由多个页面组成的块我们就叫做页面块）。该算
法先在块大小为128个页面的链表中查找，看是否有这样一个空闲块。如果有，就直接分配
；如果没有，该算法会查找下一个更大的块，具体地说，就是在块大小为256个页面的链
表中查找一个空闲块。如果存在这样的空闲块，内核就把这256个页面分为两等份，一份分
配出去，另一份插入到块大小为128个页面的链表中。如果在块大小为256个页面的链表中
也没有找到空闲页块，就继续找更大的块，即512个页面的块。如果存在这样的块，内核就
从512个页面的块中分出128个页面满足请求，然后从384个页面中取出256个页面插入到块
大小为256个页面的链表中。然后把剩余的128个页面插入到块大小为128个页面的链表中。
如果512个页面的链表中还没有空闲块，该算法就放弃分配，并发出出错信号。


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
