---
layout: post
title: Linux 中的线程
date: 2013-05-15 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 中的线程

## 线程和线程ID

在[“Linux 进程基础知识”](2013-04-23-process_1_basic.md)中我们介绍过：

> 内核并没有线程这个概念。Linux把所有的线程都当做进程来实现，线程仅仅被视为一个
> 与其他进程共享某些资源的进程。

下面关于线程的讨论是基于POSIX的标准。

进程的id用`pid_t`结构表示，类似的，线程的id用`pthread_t`结构表示。具体的值可以
通过`pthread_self`函数得到，`pthread_t`之间的比较通过`pthread_equal`函数实现。


    #include <pthread.h>
    pthread_t pthread_self(void);
    int pthread_equal(pthread_t tid1, pthread_t tid2);


## 创建线程

新线程使用`pthread_create`函数创建：

    #include <pthread.h>
    int pthread_create(pthread_t *restirct tidp,
                       const pthread_attr_t *restrict attr,
                       void *(*start_rtn)(void), void *restrict arg);
    /* 成功则返回0，出错返回错误编号 */

参数解释如下：

* `tidp`指向的内存单元被存入新线程的id。
* `attr`表示线程创建的一些属性，之后再介绍。
* `start_rtn`是一个函数的地址，表示新线程创建后要执行的函数。
* `arg`表示`start_rtn`函数的参数的指针。


## 线程终止

进程的终止有3种方式：

1. 从启动时执行的函数中返回，返回值是线程的退出码。
2. 线程调用`pthread_exit`函数。
3. 被同一进程中的其他线程取消。


```C
#include <pthread.h>
void pthread_exit(void *rval_ptr);
```

`rval_ptr`是一个无类型指针，如果线程是从它的启动例程返回，则可以把`rval_ptr`强
制转换为int，表示线程的返回码。进程中的其他线程可以通过`pthread_join`函数访问到
这个地址：

```C
#include <pthread.h>
int pthread_join(pthread_t thread, void **rval_ptr);
```

调用线程将一直阻塞，直到指定的线程终止。

线程可以通过`pthread_cancel`函数请求取消同一进程中的其他线程：

```C
#include <pthread.h>
int pthread_cancel(pthread_t tid);
```

## 线程同步

内核中的同步可以参考[一步步理解Linux之内核同步](http://gaopenghigh.github.io/posts/understanding_linux_step_by_step_synchronization.html)
POSIX也定义了类似的同步方法。


### 互斥量

互斥量用pthread_mutex_t数据类型表示，使用前需要初始化，可以通过把它设置为常量
`PTHREAD_MUTEX_INITIALIZER`（只对静态分配的互斥量）或者通过`pthread_mutex_init`
函数进行初始化。销毁互斥量可以通过`pthread_mutex_destroy`函数。

```C
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

参数`attr`表示互斥量的参数，使用默认参数直接设置为NULL即可。

下面三个函数用于获得互斥量和释放互斥量：

```C
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```


### 读写锁

类似于内核中的“读写自旋锁”，有可能导致写进程的饥饿。主要的操作函数是：

```C
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
```


### 条件变量

条件变量给多个线程提供了一个汇合的场所，条件变量通过`pthread_cond_t`结构表示，
它本身需要一个互斥量进行保护。在等待的条件还没有成立时，线程会被放到一个队列中
，等到这个条件满足时，可以选择唤醒队列中的一个线程或者全部线程。

可以静态地把一个条件变量初始化为`PTHREAD_INITIALIZER`，也可以使用下面的函数初始
化和销毁一个条件变量：

    #include <pthread.h>
    int pthread_cond_init(pthread_cond_t *restrict cond,
                          pthread_cond_attr *restrict attr);
    int pthread_cond_destroy(pthread_cond_t *cond);

下面两个函数让线程等待特定的条件变量，注意使用了一个互斥量进行保护，其中后者有
一个超时时间的参数，但超过这个时间后即使条件没有成立也会返回。

    #include <pthread.h>
    int pthread_cond_wait(pthread_cond_t *restrict cond,
                          pthread_mutex_t *restrict mutex);
    int pthread_cond_timewait(pthread_cond_t *restrict cond,
                              pthread_mutex_t *restrict mutex,
                              const struct timespec *restrict timeout);

下面两个函数用于通知线程某个条件已经满足：

    #include <pthread.h>
    int pthread_cond_signal(pthread_cond_t *cond);
    int pthread_cond_broadcast(pthread_cond_t *cond);

其中`pthread_cond_broadcast`会唤醒这个条件变量等待队列上的所有线程，而
`pthread_cond_signal`只唤醒其中的一个。

----

参考资料：

* 《UNIX环境高级编程》
* 《Linux内核设计与实现》
* 《深入理解Linux内核》
