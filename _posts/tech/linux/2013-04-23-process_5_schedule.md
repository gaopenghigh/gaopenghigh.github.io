---
layout: post
title: Linux 进程调度
date: 2013-04-23 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 进程调度

## 什么是进程调度

Linux是多任务系统，系统中需要运行很很多进程，而CPU就那么几个，于是内核就把CPU资
源分配给这些进程，让它们都能运行一小会，然后又让位给其他进程运行。当遇到阻塞，
比如从硬盘读取某个文件时，也会把CPU资源让给其它进程使用。 把CPU资源分配给进程，
就叫做进程调度。


## 调度策略

最开始，Linux的调度算法很简单。所有进程都有一个优先级，这个优先级是时刻变化的：
当一个进程用了较多的CPU时，内核就把它的优先级调低，而当一个进程用了很少的CPU时
，内核就把它的优先级调高，表示它很需要CPU资源了。每次进程切换时，内核扫描可运行
进程的链表，计算进程的优先级，然后选择“最佳”进程来运行。当进程数目较大时，选择
“最佳”进程所消耗的时间会比较长，这种算法开销太大。内核把进程分为实时进程和普通
进程，实时进程的优先级是静态设定的，而且始终大于普通进程的优先级。一共有3中调度
策略：`SCHED_FIFO`, `SCHED_RR`和`SCHED_NORMAL`。其中`SCHED_FIFO`采用先进先出的
策略， 最先进入runqueue的进程会首先运行，除非它主动放弃使用CPU，否则会一直占用
。`SCHED_RR`则在进程间论转分配CPU时间。这两种策略针对实时进程，
`SCHED_NORMAL`策略针对普通进程。

后来，Linux 2.6采用了一种叫做O(1)的调度程序，该算法的名字就透露出它能在恒定的时
间内选出“最佳”进程。这个算法很复杂，事实上很不美观。 调度器为每一个CPU维护了两
个进程队列数组：active(时间片未用完的进程)数组和expire（时间片用完的进程)数组。
数组中的元素保存着某一优先级的进程队列指针。系统一共有140个不同的优先级，因此这
两个数组大小都是140。当需要选择当前最高优先级的进程时，调度器不用遍历整个
runqueue，而是直接从active数组中选择当前最高优先级队列中的第一个进程。
每次时钟tick中断中，进程的时间片(`time_slice`)被减一。当`time_slice`为0时，调度
器判断当前进程的类型，如果是交互式进程或者实时进程，则重置其时间片并重新插入
active数组。如果不是交互式进程则从active数组中移到expired数组。这样实时进程和交
互式进程就总能优先获得CPU。然而这些进程不能始终留在active数组中，否则进入
expire数组的进程就会产生饥饿现象。当进程已经占用CPU时间超过一个固定值后，即使它
是实时进程或者交互式进程也会被移到expire数组中。 当active数组中的所有进程都被移
到 expire数组中后，调度器交换active数组和expire数组。当进程被移入expire数组时，
调度器会重置其时间片，因此新的active数组又恢复了初始情况，而expire数组为空，从
而开始新的一轮调度。

Linux在2.6.23版本以后，使用了**“完全公平调度算法”（CFS）**。


## 完全公平调度算法（CFS）

CFS不依靠实时优先级来调度，进程得到的也不是确定的时间片，而是一个CPU的使用百分
比。如果有2个进程，则它们各自能得到50%的CPU使用时间。CFS的做法是对每一个进程记
录它使用CPU的时间，然后选择使用CPU最少的一个进程作为下一个运行的进程。也就是说
，如果一个可运行的进程（没有被阻塞）得到的CPU时间比其他进程少，那么就认为内核对
它不公平，就把下一次运行的机会让给它。而每个进程的nice值，则作为一个权重，在计
算使用了多少CPU时间时加权进去。比如一个nice值较高（优先级较低）的进程明明跑了
了50ms的时间，由于它的nice值比较高，CFS就给它多算点时间。选择下一个进程时，由于
是要选得到CPU时间最少的进程，那么这个nice值较高的进程就排到后面去了，正好体现出
了它优先级低的属性。

CFS抛弃了active/expire数组，而使用红黑树选取下一个被调度进程。所有状态为RUNABLE
的进程都被插入红黑树。在每个调度点，CFS调度器都会选择红黑树的最左边的叶子节点作
为下一个将获得cpu的进程。简单地说，红黑数是一个时间复杂度为O(log n)的
自平衡二叉搜索树。它提供了一种方式，能够以较小时间对树中的节点位置进行调整，使
key值最小的一个节点就在树的最左边。

下面我们来看看CFS具体是怎么实现的。

### 时间记账

CFS用调度器实体结构`sched_entity`来追踪进程运行记账，该结构嵌入在进程描述符
`task_stuct`内。

    struct sched_entity {
        struct load_weight  load;   /* for load-balancing */
        struct rb_node  run_node;
        struct list_headgroup_node;
        unsigned inton_rq;

        u64 exec_start;
        u64 sum_exec_runtime;
        u64 vruntime;
        u64 prev_sum_exec_runtime;

        u64 nr_migrations;

        #ifdef CONFIG_SCHEDSTATS
        struct sched_statistics statistics;
        #endif

        #ifdef CONFIG_FAIR_GROUP_SCHED
        struct sched_entity *parent;
        /* rq on which this entity is (to be) queued: */
        struct cfs_rq   *cfs_rq;
        /* rq "owned" by this entity/group: */
        struct cfs_rq   *my_q;
        #endif
    };

`sched_entity`中，vruntime变量是进程的虚拟运行时间，CFS使用该时间来记录一个进程到底运行了多长时间以及它还应该运行多久。

`update_curr()`函数实现了该记账功能：

    /* <kernel/sched/fair.c> */
    static void update_curr(struct cfs_rq *cfs_rq)
    {
        struct sched_entity *curr = cfs_rq->curr;
        u64 now = rq_of(cfs_rq)->clock_task;
        unsigned long delta_exec;

        if (unlikely(!curr))
            return;

        /*
        * Get the amount of time the current task was running
        * since the last time we changed load (this cannot
        * overflow on 32 bits):
        */
        delta_exec = (unsigned long)(now - curr->exec_start);
        if (!delta_exec)
            return;

        __update_curr(cfs_rq, curr, delta_exec);
        curr->exec_start = now;

        if (entity_is_task(curr)) {
            struct task_struct *curtask = task_of(curr);

            trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
            cpuacct_charge(curtask, delta_exec);
            account_group_exec_runtime(curtask, delta_exec);
        }

        account_cfs_rq_runtime(cfs_rq, delta_exec);
    }

    static inline void
    __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
     unsigned long delta_exec)
    {
        unsigned long delta_exec_weighted;

        schedstat_set(curr->statistics.exec_max,
        max((u64)delta_exec, curr->statistics.exec_max));

        curr->sum_exec_runtime += delta_exec;
        schedstat_add(cfs_rq, exec_clock, delta_exec);
        delta_exec_weighted = calc_delta_fair(delta_exec, curr);

        curr->vruntime += delta_exec_weighted;
        update_min_vruntime(cfs_rq);

        #if defined CONFIG_SMP && defined CONFIG_FAIR_GROUP_SCHED
        cfs_rq->load_unacc_exec_time += delta_exec;
        #endif
    }

    static inline unsigned long
    calc_delta_fair(unsigned long delta, struct sched_entity *se)
    {
        if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = calc_delta_mine(delta, NICE_0_LOAD, &se->load);

        return delta;
    }

`update_curr()`是由系统的tick中断周期性调用的。`update_curr`函数首先统计当前进
程所获取的CPU时间(当前时间 - 进程开始时的时间)，然后调用`__update_curr`。
`__update_curr`函数首先跟新一些当前进程和`cfs_rq`的统计信息，然后调用
`calc_delta_fair`根据当前的负载重新计算出进程使用的CPU时间，把这个时间加到
`vruntime`里面。

**TODO**: Detail

### 进程选择

#### 挑选下一个任务

显然下一个任务就在红黑树的最左边，事实上这个值已经被缓存起来，直接调用即可。

#### 向树中加入进程

当进程变为可运行状态（被唤醒）或者通过fork调用第一次创建进程时，CFS会把进程加入
到rbtree中，并缓存最左子节点。`enqueue_entity()`函数实现了这一目的：

    static void
    enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
    {
    /*
    * Update the normalized vruntime before updating min_vruntime
    * through callig update_curr().
    */
    if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
    se->vruntime += cfs_rq->min_vruntime;

    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    update_cfs_load(cfs_rq, 0);
    account_entity_enqueue(cfs_rq, se);
    update_cfs_shares(cfs_rq);

    if (flags & ENQUEUE_WAKEUP) {
    place_entity(cfs_rq, se, 0);
    enqueue_sleeper(cfs_rq, se);
    }

    update_stats_enqueue(cfs_rq, se);
    check_spread(cfs_rq, se);
    if (se != cfs_rq->curr)
    __enqueue_entity(cfs_rq, se);
    se->on_rq = 1;

    if (cfs_rq->nr_running == 1) {
    list_add_leaf_cfs_rq(cfs_rq);
    check_enqueue_throttle(cfs_rq);
    }
    }

该函数更新运行时间和其他一些统计数据，然后调用`__enqueue_entity()`进行插入：

    static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_node;
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    int leftmost = 1;

    /*
    * Find the right place in the rbtree:
    */
    while (*link) {
    parent = *link;
    entry = rb_entry(parent, struct sched_entity, run_node);
    /*
    * We dont care about collisions. Nodes with
    * the same key stay together.
    */
    if (entity_before(se, entry)) {
    link = &parent->rb_left;
    } else {
    link = &parent->rb_right;
    leftmost = 0;
    }
    }

    /*
    * Maintain a cache of leftmost tree entries (it is frequently
    * used):
    */
    if (leftmost)
    cfs_rq->rb_leftmost = &se->run_node;

    rb_link_node(&se->run_node, parent, link);
    rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline);
    }

`__enqueue_entity()`函数中，通过while循环在红黑树中找到合适的插入点。当键值小于
当前节点的键值，则转向树的左分支，键值大于当前节点的键值，则转向右分支，同时这
也说明了待插入的这个键代表的进程不是runtime最小的进程，也就不是下一个被执行的进
程。如果插入的节点的键值最小，则跟新rb_leftmost缓存为这个节点。

#### 从树中删除进程

删除动作发生在进程阻塞（变为不可运行态）或终止时（结束运行）：

    static void
    dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
    {
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);

    update_stats_dequeue(cfs_rq, se);
    if (flags & DEQUEUE_SLEEP) {
    #ifdef CONFIG_SCHEDSTATS
    if (entity_is_task(se)) {
    struct task_struct *tsk = task_of(se);

    if (tsk->state & TASK_INTERRUPTIBLE)
    se->statistics.sleep_start = rq_of(cfs_rq)->clock;
    if (tsk->state & TASK_UNINTERRUPTIBLE)
    se->statistics.block_start = rq_of(cfs_rq)->clock;
    }
    #endif
    }

    clear_buddies(cfs_rq, se);

    if (se != cfs_rq->curr)
    __dequeue_entity(cfs_rq, se);
    se->on_rq = 0;
    update_cfs_load(cfs_rq, 0);
    account_entity_dequeue(cfs_rq, se);

    /*
    * Normalize the entity after updating the min_vruntime because the
    * update can refer to the ->curr item and we need to reflect this
    * movement in our normalized position.
    */
    if (!(flags & DEQUEUE_SLEEP))
    se->vruntime -= cfs_rq->min_vruntime;

    /* return excess runtime on last dequeue */
    return_cfs_rq_runtime(cfs_rq);

    update_min_vruntime(cfs_rq);
    update_cfs_shares(cfs_rq);
    }

    static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
    {
    if (cfs_rq->rb_leftmost == &se->run_node) {
    struct rb_node *next_node;

    next_node = rb_next(&se->run_node);
    cfs_rq->rb_leftmost = next_node;
    }

    rb_erase(&se->run_node, &cfs_rq->tasks_timeline);
    }

从红黑树删除的过程和添加的过程类似，不细说。

#### min_vruntime

上面几段代码中，都出现了`cfs_rq->min_vruntime`的操作，`cfs_rq-min_vruntime`
保存了树中所有节点的最小的一个`vruntime`值。如果树是空的，则为当前进程的
runtime。这个值是单调递增的。那么，这个变量的意义是什么呢？

可以简单地这么理解，每个CPU都有一个红黑树，进程加入一棵树时，会把`vruntime`加上
这棵树的"本底"`vruntime`也就是`min_vruntime`，离开这棵树时又会减去这个"本底"值
，"净身出户"。然后过了一会它有要加入到另外一颗树，由于已经净身，原来那颗树的
"本底"不会对进程在现在这颗树的地位有影响。这样就更公平了。

#### 调度器入口

进程的调度主要通过schedule()函数，该函数会选择一个调度类，调度类再去选择下一个
需要执行的进程。对于普通进程，调度类就是CFS调度类，对于实时进程，则有其他的调度
类，且优先级高于CFS调度类，这就保证了实时进程总是比普通进程优先运行。实时进程的
调度策略还是两种：`SCHED_FIFO`和`SCHED_RR`。

### 睡眠和唤醒

遇到阻塞时，进程把自己标记成休眠状态，从可执行红黑树中移出，放入等待队列，然后
调用schedule()选择和执行下一个进程。

唤醒过程则是：进程被设置为可执行状态，然后从等待队列中移到可执行红黑树中。

### 等待队列

等待队列是由等待某些事件发生的进程组成的简单列表，对于不同的事件，可以创建不同
的等待队列。等待队列可以通过`DECLARE_WAITQUEUE()`静态创建，也可以由
`init_waitqueue_head()`动态创建。

下面通过代码简单说一下进程进入休眠状态的过程:

    /* 'q' is the waitqueue we hope to sleep in */

    /* 创建一个等待队列的项 */
    DEFINE_WAIT(wait);

    add_wait_queue(q, &wait);
    while (!condition) { /* 'condition' 是我们等待的事件 */
        /*
        * 调用prepare_to_wait()方法把进程状态改变为TASK_INTERRUPTIBLE
        * 或者TASK_UNINTERRUPTIBLE, 取决于是否需要信号唤醒进程。
        */
        prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE);
        if(signal_pending(current)) {
            /* 处理信号 */
        }
        /*
        * 当进程唤醒时，它会再次检验条件是否为真
        * 若否，则继续schedule，让其它进程运行
        */
        schedule();
    }
    /*
    * 条件满足后，进程将自己设置为TASK_RUNNING并调用finish_wait()方法
    * 从等待队列中移除
    */
    finish_wait(&q, &wait);

### 唤醒

唤醒操作通过函数`wake_up()`进行，它会唤醒指定的等待队列上的所有进程。它调用函数
`try_to_wake_up()`，该函数负责将进程设置为`TASK_RUNNING`，调用`enqueue_task()`
把进程插入红黑树，如果被唤醒的进程优先级比当前正在自行的进程的优先级高，还要设
置`need_resched`标志，这个标准一旦被设置，则会马上运行一次调度。 通常哪段代码
促使等待条件达成，它就要负责随后调用`wake_up()`函数，比如磁盘数据到来时，VFS就
要负责对等待队列调用`wake_up()`。

### 用户抢占和内核抢占

#### 用户抢占

从系统调用返回用户空间和中断返回的时候，内核也会检查`need_resched`标志, 如果需
要，内核就会重新找下一个进程来运行，这就是用户抢占。这个标志之所以在每个
`task_struct`中而不是一个全局变量，是因为后者访问起来比较慢。

#### 内核抢占

Linux支持内核抢占。只要没有持有锁，内核就可以进行抢占。每个进程的`thread_info`
里面有一个`preempt_count`计数器，初始值为0，每当使用锁的时候加1，释放锁的时候减
1。从中断返回内核空间的时候，内核会检查`need_resched`和`preempt_count`值，如果
`need_resched`被设置且`preempt_count`为0，说明有一个更重要的任务需要执行并且可
以安全地抢占，此时，调度程序就会被调用。

### CFS组调度

CFS 另一个有趣的地方是组调度概念（在 2.6.24 内核中引入）。组调度是另一种为调度
带来公平性的方式，尤其是在处理产生很多其他任务的任务时。 假设一个产生了很多任务
的服务器要并行化进入的连接（HTTP 服务器的典型架构）。不是所有任务都会被统一公平
对待， CFS 引入了组来处理这种行为。产生任务的服务器进程在整个组中
（在一个层次结构中）共享它们的虚拟运行时，而单个任务维持其自己独立的虚拟运行时
。这样单个任务会收到与组大致相同的调度时间。您会发现 /proc 接口用于 管理进程
层次结构，让您对组的形成方式有完全的控制。
使用此配置，您可以跨用户、跨进程或其变体分配公平性。

**TODO**:CFS组调度的细节

----

参考资料：

* [《深入理解Linux内核》](http://book.douban.com/subject/2287506/)
* [《Linux内核设计与实现》](http://book.douban.com/subject/6097773/)
* [Linux 2.6 Completely Fair Scheduler 内幕](http://www.ibm.com/developerworks/cn/linux/l-completely-fair-scheduler/)
* [Linux 调度器发展简述](http://www.ibm.com/developerworks/cn/linux/l-cn-scheduler/index.html)
* [CFS调度器从2.6.25到2.6.29关于min_vruntime更新的跃进](http://blog.csdn.net/dog250/article/details/5302869)
* [红黑树](http://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)

