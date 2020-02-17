---
layout: post
title: Linux 内核同步
date: 2013-05-06 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 内核同步

Linux内核同步的方法，主要有这些：原子操作，自旋锁，读写自旋锁，信号量，读写信号
量，互斥体，完成变量，顺序锁，禁止抢占，顺序和屏障。

下面我们来一个个了解一下这些同步的方法。


## 原子操作

内核提供了两组院子操作借口，一组针对整数，另一组针对单独的位。

针对整数的原子操作通过过`atomic_t`或者`atomic64_t`类型的数据进行，后者表示64位
的整数。

可以用`ATOMIC_INIT()`宏对一个`atomic_t`类型的数据进行初始化，然后使用
`atomic_set()`，`atomic_add()`和`atomic_inc()`等宏和函数对其设置和执行各种操作

原子位操作是对普通的指针进行的操作，所以没有专门的类型，而是通过`set_bit`，
`clear_bit()`和`change_bit()`等宏和函数进行。


## 自旋锁

自旋锁，spin lock，在内核中被广泛地使用。如果一个线程试图获得一个已经被持有的自
旋锁，那么该线程就会一直进行忙循环，等待锁重新可用，就像是在“自旋”一样。

使用自旋锁的方式如下：

    DEFINE_SPINLOCK(my_spinlock);
    spin_lock(&my_spinlock);
    /* do something */
    spin_unlock(&my_spinlock);

另外，还可以使用`spin_lock_init()`方法来初始化动态创建的自旋锁，使用
`spin_try_lock()`方法试图获得某个自旋锁，当该锁已经被持有时立刻返回一个非0值。


## 读写自旋锁

读写自旋锁和自旋锁类似，但一个或多个读任务可以并发地持有读锁，而写锁只能被一个
写任务持有。典型的使用方法如下：

    DEFINE_RWLOCK(my_rwlock);

    /* 在读任务的代码分支中： */
    read_lock(&my_rwlock);
    /* do something */
    read_unlock(&my_rwlock);

    /* 在写任务的代码分支中 */
    write_lock(&my_rwlock);
    /* do something */
    write_unlock(&my_rwlock);

需要注意的是，这种锁机制照顾读比照顾写要多一点。当读锁被持有时，写操作只能等待
，而其他读操作可以继续持有，这就有可能造成写操作的饥饿。


## 信号量

自旋锁是“忙锁”，它在等待时瞎转，而信号（semaphone）量是一种“睡眠锁”，获取不到时
，进程就去睡觉了，事实上是放到了一个等待队列中，等锁别其他地方释放后再醒来。

信号量并不太“轻”，所以使用是要考虑睡眠、维护等待队列以及唤醒所花费的开销与锁占
用时间的比较。

信号量允许任意数量的锁持有者，这个数量在声明信号量是指定。当只允许一个持有者时
，成为“互斥信号量”。

信号量有两类操作：down和up。down指对信号量计数减1来请求获得一个信号量，如果结果
大于等于0，那么就获得信号量锁，否则任务就被放入等待队列。up操作释放信号量，它会
增加信号量的计数值。可以很形象地把信号量看作老式磁带播放器的按钮。

up操作使用`up()`函数实现，down操作使用`down()`函数实现，但更常用的是
`down_interruptible()`函数，因为`down()`会让进程在`TASK_UNINTERRUPTIBLE`状态下
睡眠，这样等待信号量的时候就不再响应信号了。而`down_interruptible()`函数正如其
命名，使进程以`TASK_INTERRUPTIBLE`的状态进入睡眠。

信号量的典型的用法是：

    static DECLARE_MUTEX(my_sem);
    /* 试图获取信号量 */
    if (down_interruptible(&my_sem) {
        /* 还没有获取到信号量，但接收到了信号 */
    }

    /* 临界区 */

    up(&my_sem);

`down_trylock()`函数用来以非阻塞的方式获取指定的信号量。


## 读写信号量

“读写信号量”之于“信号量”正如“读写自旋锁”之于“自旋锁”。不赘述。


## 互斥体

互斥体（mutex）的特性相当于只允许一个持有者的信号量，但相比信号量有着更为简单的
接口，它在内核和从对应数据结构`mutex`。典型的用法如下：

    DEFINE_MUTEX(my_mutex);
    mutex_init(&my_mutex);
    mutex_lock(&my_mutex);
    /* do something */
    mutex_unlock(&my_mutex);


## 完成变量

“完成变量（completion variable）”用于一个任务发出信号通知另一个人物说，有个特定
的事件发生了。它用`completion`结构体表示，典型的用法是：

    DECLARE_COMPLETION(my_comp);
    /* 或者动态地：init_completion(&my_comp) */

    /* 需要等待的任务调用： */
    wait_for_completion(&my_comp);

    /* 产生事件的任务调用： */
    complete(&my_comp);


## 顺序锁

前面我们说过读写自旋锁更照顾读操作，而“顺序锁（seqlock）”和读写自旋锁类似，但更
照顾写操作。事实上，即使在读者正在读的适合也允许写者继续运行。这种策略的好处是
写者永远不会等待（除非有另外一个写者正在写），缺点是有些时候读者不得不反复多次
读相同的数据知道它获得有效的副本。

实现这种锁主要依靠一个序列计数器。当有疑义的数据被写入时，会得到一个锁，并且序
列值会增加。在读取数据之前和之后，序列号都被读取，如果读取的序列好值相同，说明
在读操作过程中没有被写操作打断过。

顺序锁的典型用法是：

    seqlock_t my_seqlock = DEFINE_SEQLOCK(my_seqlock);

    write_seqlock(&my_seqlock);
    /* 写操作 */
    write_sequnlock(&my_seqlock);

    /* 读操作 */
    unsigned long seq;
    do {
        seq = read_seqbegin(&my_seqlock);
        /* 读取数据 */
    } while (read_seqretry(&my_seqlock, seq));
    /* 当发现数据被写过时，就一直重复去读 */


## 禁止抢占

内核是抢占性的：内核中的进程时刻都可能停下来以便另一个具有更高优先级权的进程运
行。有时候为了同步性的需要，我们希望能够禁止内核的抢占。通过`preempt_disable()`
和`preempt_enable()`两个调用可以实现禁止和启用内核抢占。

通过`get_cpu()`和`put_cpu()`两个方法可以专门处理在每个CPU上的数据：

    int cpu;
    /* 禁止内核抢占，并将cpu设置为当前处理器 */
    cpu = get_cpu();
    /* 对每个处理器的数据进行操作 */
    /* ... */
    /* 再给予内核抢占性 */
    put_cpu();

## 顺序和屏障

有时候编译器和处理器为了提高效率，可能对读和写重新排序，这有时候和程序员的初衷
相违背。有一些指令可以确保编译器不要给某个点周围的指令序列进行重新排序，这些指
令称为“屏障（barriers）”。

`rmb()`方法提供了一个“读”内存屏障，它确保跨越`rmb()`的载入动作不会发生重排序。
与之类似的，`wmb()`方法提供“写”屏障，`mb()`方法既提供了读屏障也提供了写屏障。


----

参考资料：

* Man pages
* [UNIX环境高级编程](http://book.douban.com/subject/1788421/)
* [Linux内核设计与实现](http://book.douban.com/subject/6097773/)

