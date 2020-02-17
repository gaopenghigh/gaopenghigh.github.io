---
layout: post
title: Linux 内存管理(4)：以字节为单位的内存管理
date: 2013-11-09 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 内存管理(4): 以字节为单位的内存管理

本节讨论以字节为单位的内存管理。

## kmalloc()和vmalloc()

### kmalloc()

`kmalloc()`和用户空间中的`malloc()`类似，只不过多了一个`flag`参数。它可以获得以
字节为单位的一块内核内存。大多数内核分配都使用`kmalloc()`。

声明在`<linux/slab.h>`中：

    void * kmalloc(size_t size, gfp_t flags)

`kmalloc()`的另一端就是`kfree`：

    void kfree(const void *ptr)

### vmalloc()

`vmalloc()`类似于`kmalloc()`，只不过vmalloc()分配的线性地址是连续的，而物理地址
无须连续。这也是用户空间分配函数的工作方式，由malloc()返回的页在进程的线性地址
空间内是连续的，但并不保证它们在物理RAM中也是连续的。vmalloc()只确保页在线性地
址中是连续的，它通过分配非连续的物理内存块，再“修正”页表，把物理内存映射到线性
地址空间的连续区域中。这会有一些性能损耗，所以vmalloc()常用于分配大块的内存。

相对应的释放函数为：

    void vfree(const void *addr)


## slab层

分配和释放数据结构是内核中最普遍的操作之一，为了便于数据的频繁分配和回收，程序
员常常用到空闲链表，空闲链表提供可用的，已经分配好的数据结构块，需要是从链表中
取一个，用完后再放回链表。slab层就是类似这个空闲链表的角色。

slab分配器扮演了通用数据结构缓存层的角色。它把不同的对象划分为所谓高速缓存组，
其中每个高速缓存组都存放不同类型的对象。比如一个高速缓存组用于存放进程描述符（
task_struct），另一个高速缓存存放索引节点对象（struct inode）。

然后这些高速缓存又被划分为slab，slab由一个或多个物理上连续的页组成，每个高速
缓存可以由多个slab组成。slab中包含一些对象成员，而slab的状态可以为“满”，“部分满
”或者“空”。当内核的某一部分需要一个新的对象时，先从部分满的slab中进行分配，如果
没有部分满的slab，就从空的slab中分配。

### 结构和使用

每个高速缓存都使用`kmem_cache`结构体表示，该结构体包含三个链表：`slabs_full`，
`slabs_partial`和`slabs_empty`。

slab描述符`struct slab`用来描述每个slab:

    struct slab {
        struct list_head    list;      /* 满、部分满或空的链表 */
        unsigned long       colouroff; /* slab 着色的偏移量 */
        void                *s_mem;    /* 在slab中的第一个对象 */
        unsigned int        inuse;     /* slab中已分配的对象数 */
        kmem_bufctl_t       free;      /* 第一个空闲对象(如果有的话) */
    };

slab分配器使用`__get_free_pages()`来创建新的slab。

创建新的高速缓存：

    struct kmem_cache * kmem_cache_create(const char *name,
                                          size_t, size,
                                          size_t align,
                                          unsigned long flags,
                                          void (*ctor)(void *));
    /*
     * name 表示高速缓存的名字
     * size 指每个元素的大小
     * align 指slab内第一个元素的偏移，一般为0
     * flags 用来控制高速缓存的行为
     * ctor 是高速缓存的构造函数，但Linux没有用到它
     */

从缓存中分配：

    void * kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)

释放一个对象，返还给原先的slab：

    void kmem_cache_free(struct kmem_cache *cachep, void *objp)

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

