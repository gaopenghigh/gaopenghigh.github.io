---
layout: post
title: 内核中 hash 函数的实现
date: 2013-04-21 10:00:00 +0800
categories: Tech
toc: true
---

# 内核中 hash 函数的实现

Linux内核中通过PID查找进程描述符`task_struct`时，用到了hash表。
下面介绍一下这一部分内核中hash函数的实现。

内核用`pid_hashfn`宏把PID转换为表索引(kernel/pid.c):

```
#define pid_hashfn(nr, ns)  \
    hash_long((unsigned long)nr + (unsigned long)ns, pidhash_shift)
```

其中`hash_long`在`<linux/hash.h>`中定义如下:

    /* 2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1 */
    #define GOLDEN_RATIO_PRIME_32 0x9e370001UL
    /*  2^63 + 2^61 - 2^57 + 2^54 - 2^51 - 2^18 + 1 */
    #define GOLDEN_RATIO_PRIME_64 0x9e37fffffffc0001UL

    #if BITS_PER_LONG == 32
    #define GOLDEN_RATIO_PRIME GOLDEN_RATIO_PRIME_32
    #define hash_long(val, bits) hash_32(val, bits)
    #elif BITS_PER_LONG == 64
    #define hash_long(val, bits) hash_64(val, bits)
    #define GOLDEN_RATIO_PRIME GOLDEN_RATIO_PRIME_64
    #else
    #error Wordsize not 32 or 64
    #endif

    static inline u64 hash_64(u64 val, unsigned int bits)
    {
        u64 hash = val;

        /*  Sigh, gcc can't optimise this alone like it does for 32 bits. */
        u64 n = hash;
        n <<= 18;
        hash -= n;
        n <<= 33;
        hash -= n;
        n <<= 3;
        hash += n;
        n <<= 3;
        hash -= n;
        n <<= 4;
        hash += n;
        n <<= 2;
        hash += n;

        /* High bits are more random, so use them. */
        return hash >> (64 - bits);
    }

    static inline u32 hash_32(u32 val, unsigned int bits)
    {
        /* On some cpus multiply is faster, on others gcc will do shifts */
        u32 hash = val * GOLDEN_RATIO_PRIME_32;

        /* High bits are more random, so use them. */
        return hash >> (32 - bits);
    }

    static inline unsigned long hash_ptr(const void *ptr, unsigned int bits)
    {
        return hash_long((unsigned long)ptr, bits);
    }
    #endif /* _LINUX_HASH_H */

上面的函数很有趣，我们来仔细看一下。

首先，hash的方式是，让key乘以一个大数，于是结果溢出，就把留在32/64位变量中的值作为hash值，又由于散列表的索引长度有限，我们就取这hash值的高几为作为索引值，之所以取高几位，是因为高位的数更具有随机性，能够减少所谓“**冲突**”。什么是冲突呢？从上面的算法来看，key和hash值并不是一一对应的。有可能两个key算出来得到同一个hash值，这就称为“冲突”。

那么，乘以的这个大数应该是多少呢？从上面的代码来看，32位系统中这个数是`0x9e370001UL`，64位系统中这个数是`0x9e37fffffffc0001UL`。这个数是怎么得到的呢？

> “Knuth建议，要得到满意的结果，对于32位机器，2^32做黄金分割，这个大树是最接近黄金分割点的素数，`0x9e370001UL`就是接近 `2^32*(sqrt(5)-1)/2` 的一个素数，且这个数可以很方便地通过加运算和位移运算得到，因为它等于`2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1`。对于64位系统，这个数是`0x9e37fffffffc0001UL`，同样有`2^63 + 2^61 - 2^57 + 2^54 - 2^51 - 2^18 + 1`。”

从程序中可以看到，对于32位系统计算hash值是直接用的乘法，因为gcc在编译时会自动优化算法。而对于64位系统，gcc似乎没有类似的优化，所以用的是位移运算和加运算来计算。
首先`n=hash`,
然后n左移18位，`hash-=n`，
这样`hash = hash * (1 - 2^18)`，
下一项是`-2^51`，而n之前已经左移过18位了，所以只需要再左移33位，
于是有
    n <<= 33

依次类推，最终算出了hash值。

JH, 2013-04-21

----

参考资料：

* 《深入理解Linux内核》
* 《Linux内核设计与实现》
