---
layout: post
title: Linux 中的管道和 FIFO
date: 2013-10-24 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 中的管道和 FIFO


进程间通信叫做IPC：InerProcess Communication

Unix系统的进程间通信主要有下面几种机制：

* 管道和FIFO
* 信号量
* 消息（Linux内核提供“System V IPC消息”和“POSIX消息队列”）
* 共享内存区
* 套接字

本文讨论管道和FIFO，以及它们在Linux内核中的实现方式。

## 管道和FIFO

管道和FIFO最适合在进程之间实现生产者/消费者的交互。

### Shell中的管道

让我们考察一个Shell命令`ls | more`执行时到底发生了哪些事：

1. 调用`pipe()`系统调用，产生了两个文件描述符3（读）和4（写）。
2. 两次`fork()`系统调用。
    - 第一次`fork()`，需要执行`more`命令：
        * 调用`dup2(3, 0)`把文件描述符3拷贝到标准输入0.
        * 两次调用`close()`系统调用关闭文件描述符3和4.
        * 调用`execve()`系统调用执行`more`程序，从标准输入（之前已经被接到了管
          道）读入输入，输出到标准输出。
    - 第二次`fork()`，需要执行`ls`命令：
        * 调用`dup2(4, 1)`把文件描述符4拷贝到标准输入1.
        * 两次调用`close()`系统调用关闭文件描述符3和4.
        * 调用`execve()`系统调用执行`ls`程序，输出到标准输出（之前已经被接到了
          管道）。

### pipe()

半双工的管道是最常见的IPC形式。管道通过`pipe()`函数创建：

    #include <unistd.h>
    int pipe(int filedes[2]);
    /* 成功返回0，失败返回-1 */

参数`filedes`是int型的长度为2的数组，经过`pipe()`函数调用后:

* `filedes[0]`可读
* `filedes[1]`可写
* `filedes[0]`的输入是`filedes[1]`的输出

父进程和子进程通过管道通信的例子是：

    #include <unistd.h>
    #include <stdio.h>
    #include <stdlib.h>

    #define MAXLINE 100

    void err(char *s) {
        printf("%s\n", s);
        exit(1);
    }

    void test_pipe() {
        int n;
        int fd[2];
        int pid;
        char line[MAXLINE];

        if (pipe(fd) < 0)
            err("pipe failed");
        if ((pid = fork()) < 0) {
            err("fork failed");
        } else if (pid == 0) {        /* parent */
            close(fd[0]);             /* close read side of PIPE */
            write(fd[1], "hello son\n", 10);
        } else {                      /* child */
            close(fd[1]);             /* close write side of PIPE */
            n = read(fd[0], line, MAXLINE);
            write(STDOUT_FILENO, line, n);
        }
        exit(0);
    }

在APUE中用管道实现了父子进程之间的通信函数TELL_WAIT，实现如下：


    /* tell_wait_pipe.c
     * Use PIPE to sync between child process and parent process
     * From APUE
     * Create 2 PIPEs, fd1[2], fd2[2]
     * fd1 : parent --> child, write 'p' if parent is OK
     * fd2 : child --> parent, write 'c' if child is OK
     */
    #include <stdlib.h>
    #include <stdio.h>
    #include <unistd.h>

    #define ERRORCODE 1

    static int fd1[2], fd2[2];

    void err(char *s) {
        printf("ERROR: %s\n", s);
        exit(ERRORCODE);
    }

    void TELL_WAIT(void) {
        if (pipe(fd1) < 0 || pipe(fd2) < 0)
            err("pipe failed");
    }

    void TELL_PARENT(pid_t pid) {
        if (write(fd2[1], "c", 1) != 1)
            err("write error");
    }

    void WAIT_PARENT(pid_t pid) {
        char c;
        if (read(fd1[0], &c, 1) != 1)
            err("read error");
        if (c != 'p')
            err("WAIT_PARENT: incorrect data");
    }

    void TELL_CHILD(pid_t pid) {
        if (write(fd1[1], "p", 1) != 1)
            err("write error");
    }

    void WAIT_CHILD(pid_t pid) {
        char c;
        if (read(fd2[0], &c, 1) != 1)
            err("read error");
        if (c != 'c')
            err("WAIT_CHILD: incorrect data");
    }


### `popen()`和`pclose()`

`popen()`函数创建一个管道，然后调用fork产生一个子进程，关闭管道的不使用端，执行
一个shell以运行命令，然后等待命令终止。

    #include <stdio.h>
    FILE *popen(const char *cmdstring, const char *type);
    /* type 可以是"r"或者"w" */
    /* 成功返回文件指针，出错返回NULL */
    int pclose(FILE *fp);
    /* 返回cmdstring的终止状态，出错返回-1 */


## FIFO

FIFO又叫做命名管道。

    #include <sys.stat.h>
    int mkfifo(const char *pathname, mode_t mode);
    /* mode的值和open()函数一样，常用O_RDONLY, O_WRONLY, O_RDWR三者之一 */
    /* 返回值：成功返回0，出错返回-1 */

通过prog1产生的一份数据，要分别经过prog2和prog3的处理，使用FIFO可以这样做：

    mkfifo fifo1
    prog3 < fifo1 &
    prog1 < infile | tee fifo1 | prog2

FIFO的一个重要应用是在客户进程和服务器进程间传递数据。


## 管道的内核实现

### 管道数据结构

管道被创建时，进程就可以使用两个VFS系统调用--（即`read()`和`write()`来访问管道
。所以，对于每个管道，内核都要创建一个索引节点和两个文件对象，一个文件对象用于
读。

在索引节点结构`struct inode`中，有一个表示索引节点类型的union：

    union {
        struct pipe_inode_info  *i_pipe;
        struct block_device *i_bdev;
        struct cdev     *i_cdev;
    };

`i_pipe`字段就指向一个代表管道的结构体`struct pipe_inode_info`:

    /**
     *	struct pipe_buffer - a linux kernel pipe buffer
     *	@page: the page containing the data for the pipe buffer
     *	@offset: offset of data inside the @page
     *	@len: length of data inside the @page
     *	@ops: operations associated with this buffer. See @pipe_buf_operations.
     *	@flags: pipe buffer flags. See above.
     *	@private: private data owned by the ops.
     **/
    struct pipe_buffer {
    	struct page *page;
    	unsigned int offset, len;
    	const struct pipe_buf_operations *ops;
    	unsigned int flags;
    	unsigned long private;
    };

    /**
     *	struct pipe_inode_info - a linux kernel pipe
     *	@wait: reader/writer wait point in case of empty/full pipe
     *	@nrbufs: the number of non-empty pipe buffers in this pipe
     *	@buffers: total number of buffers (should be a power of 2)
     *	@curbuf: the current pipe buffer entry
     *	@tmp_page: cached released page
     *	@readers: number of current readers of this pipe
     *	@writers: number of current writers of this pipe
     *	@waiting_writers: number of writers blocked waiting for room
     *	@r_counter: reader counter
     *	@w_counter: writer counter
     *	@fasync_readers: reader side fasync
     *	@fasync_writers: writer side fasync
     *	@inode: inode this pipe is attached to
     *	@bufs: the circular array of pipe buffers
     **/
    struct pipe_inode_info {
    	wait_queue_head_t wait;
    	unsigned int nrbufs, curbuf, buffers;
    	unsigned int readers;
    	unsigned int writers;
    	unsigned int waiting_writers;
    	unsigned int r_counter;
    	unsigned int w_counter;
    	struct page *tmp_page;
    	struct fasync_struct *fasync_readers;
    	struct fasync_struct *fasync_writers;
    	struct inode *inode;
    	struct pipe_buffer *bufs;
    };

每个管道都有自己的管道缓冲区（pipe buffer），事实上，每个管道都有一个管道缓冲区
的数组，即`bufs`字段表示的数组，这个数组可以看作一个环形缓冲区：写进程不断向这
个大缓冲区追加数据，而读进程则不断移出数据。当前使用的那个缓冲区就是`curbuf`。

每个管道缓冲区由`struct pipe_buffer`表示，结构体中指明了缓冲区页框的描述符地址
，页框内有效数据的offset以及数据的长度，`ops`字段表示的是操作管道缓冲区的方法表
的地址。

### 管道的创建和撤销

#### 创建

`pipe()`系统调用由`sys_pipe()`函数处理，后者又会调用`do_pipe()`函数。该函数的主
要操作如下：

1. 创建一个索引节点对象并初始化。
2. 分配可读的文件对象和文件描述符，该文件对象的`f_op`字段初始化成
   `read_pipe_fops`表的地址。
3. 分配可写的文件对象和文件描述符，该文件对象的`f_op`字段初始化成
   `write_pipe_fops`表的地址。
4. 分配一个目录项对象，并使用它把两个文件对象和索引节点连接在一起。然后把新的索
   引节点插入pipefs图书文件系统中。
5. 把两个文件描述符返回给用户态进程。

#### 撤销

只要进程对与管道相关的一个文件描述符调用`close()`系统调用，内核就对相应的文件对
象执行`fput()`函数，这会减少它的引用计数器的值。如果这个计数器变成0，那么该函数
就会调用这个文件操作的`release`方法。`realeas`方法根据文件是与读通道还是写通道
关联，对这个管道相关的资源进行释放。

### 管道的读取和写入

#### 从管道中读取

希望从管道中读取数据的进程发出一个`read()`系统调用，内核最终调用与这个文件描述
符相关的文件操作表中所找到的read方法，对于管道，read方法在`read_pipe_fops`表中
指向`pipe_read()`函数。

当出现下面的情况时，读操作会阻塞：

* 系统调用开始时管道缓冲区为空。
* 管道缓冲区没有包含所有请求的字节，写进程在等待缓冲区的空间时曾被置为睡眠。

当然，通过`fcntl()`系统调用也可以把对管道的读操作设置为非阻塞的。

#### 向管道中写入

希望向管道中写入数据的进程发出一个`write()`系统调用，和读取类似，内核最终会调用
`pipe_write()`函数。

需要注意的是，如果有两个或者多个进程并发地在写入一个管道，那么任何少于一个管道
缓冲区大小的写操作都必须单独（原子地）完成，而不能与其它进程交叉进行。

### FIFO

在Linux中，FIFO和管道几乎是相同的，除了：

* FIFO索引节点出现在系统目录树上而不是pipefs特殊文件系统中。
* FIFO是一种双向通信管道；也就是说，可以以读/写模式打开一个FIFO。

打开一个FIFO时，VFS首先判断到这个文件是特殊的FIFO类型文件，然后找到针对这种文件
的操作函数表，在使用表中对应的函数对FIFO进行操作。


----

参考资料：

* Man pages
* [UNIX环境高级编程](http://book.douban.com/subject/1788421/)
* [Linux内核设计与实现](http://book.douban.com/subject/6097773/)

