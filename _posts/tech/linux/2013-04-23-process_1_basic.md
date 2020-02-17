---
layout: post
title: Linux 进程基础知识
date: 2013-04-23 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 进程基础知识

## 基本概念

### 进程

进程是处于执行期的程序以及相关资源的总称，注意包含了两个方面：

* 程序代码。
* 资源。比如说打开的文件，挂起的信号，内核内部数据，处理器状态，一个或多个具有
内存映射的内存地址空间，一个或多个执行线程，存放全局变量的数据段等。

### PID

任意一个进程通过PID作为标识，PID一般是一个正整数。

### 线程

对于Linux，内核并没有线程这个概念。Linux把所有的线程都当做进程来实现，线程仅仅
被视为一个与其他进程共享某些资源的进程。

### 进程的状态

进程可以有5种状态：

* `TASK_RUNNING`（运行） -- 进程正在执行，或者在运行队列中等待执行。这是进程在
用户空间中唯一可能的状态。*TODO*:运行队列的概念。
* `TASK_INTERRUPTIBLE`（可中断） -- 进程正在睡眠（阻塞），等待某些条件的达成。
一个硬件中断的产生、释放进程正在等待的系统资源、传递一个信号等都可以作为条件唤
醒进程。
* `TASK_UNINTERRUPTIBLE`（不可中断） -- 与可中断状态类似，除了就算是 收到信号也
不会被唤醒或准备投入运行，对信号不做响应。这个状态通常在进程必须在等待时不受干
扰或等待事件很快就会发生时出现。例如，当进程打开一个设备文件，相应的设备驱动程
序需要探测某个硬件设备，此时进程就会处于这个状态，确保在探测完成前，设备驱动程
序不会被中断。
* `__TASK_TRACED` -- 被其它进程跟踪的进程，比如ptrace对程序进行跟踪。
* `__TASK_STOPPED` -- 进程停止执行。通常这种状态发生在接收到SIGSTOP, SIGTSTP,
SIGTTIN, SIGTTOU等信号的时候。此外，在调试期间接收到任何信号，都会使进程进入这
种状态。*TODO*:这几种信号的意思。


## ps的使用

### ps的参数

Linux中查看进程的最重要的命令是ps，学会ps的使用，就了解了进程的大部分内容。具体
的参数都在man手册，常用的参数是：

    a    显示所有进程（包括其它用户的进程）
    -e   列出所有进程
    x    显示无控制终端的进程
    u    输出用户名
    e    列出程序时，显示每个程序所使用的环境变量
    f    显示树状图
    -L   显示线程
    -o   按照自定义的格式输出信息
    -ww  在终端下不截断命令

命令`ps aux的`输出的各列如下：

    $ps aux
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND

需要解释的是：

* VSZ    Virtual Memory Size, 表示如果一个程序完全驻留在内存的话需要占用多少内存空间;
* RSS    physically resident memory, 指明了当前实际占用了多少内存;
* STAT   显示了进程当前的状态

RSS之所以比VSZ小，是由于进程间经常共享一些库，这些库文件在物理内存中只需要存储
一份，而VSZ在计算的时候是把这一份也计算进去了。使用`pmap -d [PID]`命令查看一个
进程的内存使用情况，可以看到一个进程使用了很多的共享库，`r-x--`表示代码段，
`rw---`表示数据段。在输出的最后一行`writeable/private: 3900K`这里的3900K表示的
是进程“自己”的内存使用量。


### ps输出的进程状态

进程的状态和其表示符号如下：

    D    uninterruptible sleep (usually IO)
    R    running or runnable (on run queue)
    S    interruptible sleep (waiting for an event to complete)
    T    stopped, either by a job control signal or because it is being traced
    W    paging (not valid since the 2.6.xx kernel)
    X    dead (should never be seen)
    Z    defunct ("zombie") process, terminated but not reaped by its parent

    For BSD formats and when the stat keyword is used, additional characters may be displayed:

    <    high-priority (not nice to other users)
    N    low-priority (nice to other users)
    L    has pages locked into memory (for real-time and custom IO)
    s    is a session leader
    l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
    +    is in the foreground process group


## 进程关系

### 进程的创建

Linux非常依赖进程创建来满足用户的需求，比如在shell中输入一条命令，shell进程就生成一个新进程，新进程执行shell的另一个拷贝，这个拷贝的过程由`fork()`函数完成。示例如下：

_示例1_

    $  ps -u jh -eo user,pid,ppid,stat,cmd
    jh       18586  2840 Ss   /bin/bash
    jh       19110 18586 R+   ps -u jh -eo user,pid,ppid,stat,cmd

可以看到用户的终端shell的pid是18586，执行ps程序时，进程18586生产了新的子进程19110来运行ps。

_示例2_

    $ ps -u jh -eo user,pid,ppid,stat,cmd | cat
    USER       PID  PPID STAT CMD
    jh       18586  2840 Ss   /bin/bash
    jh       19130 18586 R+   ps -u jh -eo user,pid,ppid,stat,cmd
    jh       19131 18586 S+   cat

示例中ps的输出通过管道传送给cat，可以看到，运行ps的进程（19130）和运行cat的进程（19131）都是bash进程（18586）的子进程。

由此可见，出了初始的内核进程，所有进程都由其父进程`fork()`产生。
如果一个进程的父进程终止而自己继续运行，则这种进程叫做孤儿进程（orphan process），内核会把init进程（即PID为1）的进程安排为孤儿进程的父进程。

### 作业

顾名思义，作业就是完成某个任务的进程的集合。作业可以在前台或者后台运行。比如

_示例1_

    $ cat main.c

在前台启动了一个作业，这个作业只包含了一个进程，即cat。

_示例2_

    $ ps aux | grep -v root | cat

在前台启动了一个作业，这个作业包含了三个进程，即ps, grep和cat

_示例3_

    $ ps aux | grep -v root | cat &

则是在后台启动了一个包含3个进程的作业

### 进程组

每个进程除了有一个唯一的PID外，还属于一个进程组。

进程组是一个或多个进程的集合。通常，他们与统一作业相关联，可以接受来自同一终端
的各种信号。

每个进程组由一个进程组ID（PGID）标识。当一个进程的PID等于其所在进程组的PGID时，
我们称这个进程是该进程组的组长。

    $ ps -u jh -eo user,pid,ppid,pgid,stat,cmd | cat | cat
    USER       PID  PPID  PGID STAT CMD
    jh       18586  2840 18586 Ss   /bin/bash
    jh       19299 18586 19299 R+   ps -u jh -eo user,pid,ppid,pgid,stat,cmd
    jh       19300 18586 19299 S+   cat
    jh       19301 18586 19299 S+   cat

可以看到，19299, 19300, 19301三个进程具有同样的PGID(19299)，说明它们属于同一进
程组，组长进程为19299.

### 会话（session）

会话是一个或多个进程组的集合。
同样，会话由会话ID（SID）标识。如果一个进程的PID等于其所在会话的SID，我们称这个
进程是该会话的会话首进程。

    $ ps aux | less &
    [1] 19322
    $ ps -u jh -eo user,pid,ppid,pgid,sid,stat,cmd | cat | cat
    USER       PID  PPID  PGID   SID STAT CMD
    jh       18586  2840 18586 18586 Ss   /bin/bash
    jh       19321 18586 19321 18586 T    ps aux
    jh       19322 18586 19321 18586 T    less
    jh       19323 18586 19323 18586 R+   ps -u jh -eo user,pid,ppid,pgid,sid,stat,cmd
    jh       19324 18586 19323 18586 S+   cat
    jh       19325 18586 19323 18586 S+   cat

可以看到，上面两条命令执行下来，产生了两个进程组（PGID分别为19321和19323），它
们都属于同一个会话（18586）。事实上，由当前控制终端产生的所有进程组都会属于同一
个会话。

一个会话中的几个进程组可以被分成一个前台进程组（foreground process group）和一
个或多个后台进程组（background process group）。拥有控制终端的进程所在的进程组
就是前台进程组，在终端中键入中断键（Control + C），就会把中断信号发给前台进程组
中的所有进程。上面的例子中，PGID=19321的进程组就是后台进程组，PGID=19323的进程
组就是前台进程组。
