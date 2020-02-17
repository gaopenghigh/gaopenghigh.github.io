---
layout: post
title: Linux 进程编程
date: 2013-09-30 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 进程编程

## 获取各种ID

在“[一步步理解Linux进程（1）--基础知识](understanding_linux_step_by_step_process_1_basic.html)”
中我们讨论了几种ID:PID, PPID, PGID, SID。另外Linux中还有用户ID(UID),
有效用户ID(EUID), 用户组ID(GID), 有效用户组ID(EGID)等，
下面这些函数正是用来获取这些ID：

    #include <unistd.h>
    pid_t getpid(void);
    pid_t getppid(void);
    pid_t getpgid(pid_t pid); /* pid为0时则返回调用进程的进程组ID */
    pid_t getsid(pid_t pid);  /* pid为0时则返回调用进程的会话首进程的ID */
    uid_t getuid(void);
    uid_t geteuid(void);
    gid_t getgid(void);
    gid_t getegid(void);


## fork函数

`fork`函数可能是Linux中最出名的函数了。
一个现有进程可以调用`fork`函数创建一个新进程。

    #include <unistd.h>
    pid_t fork(void);

很有意思的是，调用`fork`函数时，产生了子进程，而fork函数分别在父进程和子进程中
都有返回，其中在父进程中返回子进程的PID，子进程中则返回0。子进程中的一切都是从
父进程copy过去的（事实上，Linux用了“**写时复制**”的技术，也就是指直到真正对数据
产生写操作时，这些资源才真的被copy）。fork的一个使用示例如下：

    #include <unistd.h>
    #include <stdio.h>

    int main(void) {
        int var = 10;
        pid_t pid;
        if ((pid = fork()) < 0) {
            printf("fork error\n");
        } else if (pid == 0) {                 /* child */
            printf("in child, ");
            var++;
        } else {                              /* parent */
            printf("in parent, ");
            var--;
        }
        printf("pid=%d, ppid=%d, var=%d\n", getpid(), getppid(), var);
        return 0;
    }

编译并运行这个程序：

    $ gcc -Wall fork_test.c
    $./a.out
    in parent, pid=21319, ppid=18586, var=9
    in child, pid=21320, ppid=21319, var=11

正如我们预想的那样，变量`var`分别在父进程和子进程中被-1和+1。


## exit函数和atexit函数

### `exit`

有3个函数用于正常终止一个程序：

    #include <stdlib.h>
    void exit(int status);
    void _Exit(int status);

    #include <unistd.h>
    void _exit(int status);

其中，`_exit`和`_Exit`直接进入内核，`exit`则是先调用执行各终止处理程序
（exit handler），然后关闭所有标准IO流，最后才进入内核。三个函数中，参数
`status`都表示终止状态。

子进程终止时，内核并没有完全抹去这个进程的一切，而是保留以一定的信息，比如PID，
终止状态，使用的CPU时间总量等等。父进程可以通过wait系列函数获取到这些信息，然后
再根据实际需要做更多的处理。在子进程已结束但父进程还没有去获取这些信息这段时间
内，子进程叫做**僵尸进程**（zombie process）。

子进程和父进程独立运行，但当子进程终止时，系统会给父进程发送一个`SIGCHLD`信号，
父进程可以捕获这个信号并做响应的处理，但Linux中默认对次信号忽略。

### `atexit`

前面说过，`exit`函数会先调用执行各终止处理程序（exit handler）。这些终止处理程
序，正是通过`atexit`函数注册的：

    #include <stdlib.h>
    int atexit(void (*func)(void));

其中，`atexit`的参数是一个函数地址，调用此函数时无需传送任何参数，也不期望它返
回一个值。一个进程可以注册32个终止处理程序，exit调用这些函数的顺序与它们注册的
顺序相反。


## wait系列函数

### wait和waitpid函数

子进程终止后，内核仍然保留该进程的一些信息，这些信息可以通过`wait`和`waitpid`函
数来获取。

    #include <sys/wait.h>
    pid_t wait(int *statloc);
    pid_t waitpid(pid_t pid, int *statloc, int options);

这两个函数若执行成功则返回进程PID或者0，出错则返回-1。

执行这两个函数时

1. 如果其所有子进程都还在运行，则阻塞，直到有子进程终止；
2. 如果没有子进程，则出错返回-1；
3. 如果一个子进程已经终止并等待父进程获取其终止状态，则取得其PID立即返回；

参数`statloc`是整形指针，当该参数不为NULL时，子进程的终止状态就存入该指针指向的
单元内。

waitpid和wait的的区别是：

1. waitpid可以等待一个指定的进程终止，所等待的进程由参数pid提供：

        pid == -1   等待任一子进程
        pid > 0     等待进程PID与pid相等的进程，若不存在，则出错返回
        pid == 0    等待PGID等于调用进程PGID的任一子进程
        pid < 0     等待其PGID等于pid绝对值的任一子进程

2. waitpid中的第三个参数options，options可以是0，也可以是以下三个值按位“或”的结
果：

        WCONTINUED  由pid指定的进程在暂停后继续，且尚未报告其状态，则返回其状态
        WNOHANG     waitpid不阻塞，返回0
        WUNTRACED   由pid指定的进程处于暂停状态，且该状态为报告过，则返回其状态

### waitid函数

可以看到，`waitpid`函数中参数`pid`的作用不够纯粹，增加了代码的复杂度，降低了可
读性。`waitid`函数与`waitpid`类似，但`waitid`用单独的参数表示要等待的子进程的类
型。

    #include <unistd.h>
    int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);

具体使用方法可参考`waitid`的man手册。

### wait4函数

与`wait`, `waitpid`, `waitid`相比，`wait4`函数多了一个功能：要求内核返回进程使
用的资源汇总。具体可参考`wait4`的man手册。


## `exec`系列函数

当进程调用一种`exec`函数时，该进程执行的程序完全替换为新程序，新程序由`exec`函
数的参数指定。

    #include <unistd.h>
    extern char **environ;
    int execl(const char *path, const char *arg, ...);
    int execlp(const char *file, const char *arg, ...);
    int execle(const char *path, const char *arg, ..., char * const envp[]);
    int execv(const char *path, char *const argv[]);
    int execvp(const char *file, char *const argv[]);
    int execvpe(const char *file, char *const argv[], char *const envp[]);

可以用exec函数中的字母l, v, p, e对这6个函数做区别:

    l    表示list，要求将新程序的每个命令行参数都表示为一个独立的参数
    v    表示vector，与list对应，先构造一个指向所有参数的指针数组，然后把该数组地址作为参数传给exec
    p    表示使用file作为参数，在PATH中寻找对应的文件，file中含有"/"则视作路径
    e    表示可以传递一个指向环境字符串指针数组的指针，指向的字符串作为环境变量

----

参考资料：

* [《UNIX环境高级编程》](http://book.douban.com/subject/1788421/)
* [“UNIX进程揭秘”](http://www.ibm.com/developerworks/cn/aix/library/au-unixprocess.html)

