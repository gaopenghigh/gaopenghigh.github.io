---
layout: post
title: Linux 文件的属性
date: 2013-03-06 10:00:00 +0800
categories: Tech
toc: true
---

# Linux 文件的属性

本文主要介绍了文件的隐藏属性和特殊权限，一个让脚本具有类似SUID位的技巧，还有一个解决mp3文件乱码的方法。
从最开始接触Linux到现在已经有5年了，和所有人一样，少不了折腾。折腾后偶尔我会把方法记录下来，现在简单总结一下。
所以的命令功能通过man都能找到具体用法，我只把自己觉得常用的列举出来。

### 文件的隐藏属性

* `lsattr` : 列出文件的隐藏属性
* `chattr` : 修改文件的隐藏属性

        [root@www ~]# chattr [+-=][ASacdistu] FileName
        + : Add one attribute
        - : Remove one attribute
        = : Set to be the only attributes that the files have

重要选项

* "a"：只能追加文件的内容，但不能修改或删除内容
* "i"：文件不能被删除、改名、不能创建指向它的链接，不能向文件写内容

### 文件的特殊权限 SUID/SGID/Sticky Bit

如果对一个可执行文件设置了SUID或者SGID位，则文件执行时，将会拥有文件所有者（设置了SUID）或者所在组（设置了SGID）的权限。

例子：普通用户不能开启httpd服务，因为httpd服务需要用到80端口，而1024以下的端口只有root用户才能使用。如果我们把httpd可执行文件的所有者设置为root，同时设置SUID位，则普通用户也可以开启httpd服务了。

对一个目录设置了Sticky位，则只有文件的所有者能删除这个文件。在Linux系统中，/tmp目录默认设置了这个位：

    drwxrwxrwt 12 root root 16384 Mar 6 09:04 tmp/

主要使用方法如下：

**SUID**

* 对于文件：以文件所有者的权限运行
* 对于目录：不能对目录设置SUID

设置SUID：

    chmod u+s FILE chmod 4755 FILE

**SGID**

* 对于文件：以文件所属组的权限运行
* 对于目录：目录里面的文件会继承目录的属性

设置SGID：

    chmod g+s FILE/DIR chmod 2771 FILE/DIR

**Sticky**

* 对于文件：不能对文件设置Sticky位
* 对于目录：对于该目录下的文件，只有它们的所有者才能删除它们。

设置Sticky：

    chmod o+t DIR chmod 1777 DIR

用字母设置特殊权限：

    u+s g+s o+t
用数字表示特殊权限，则是：

    4 for SUID
    2 for SGID
    1 for Sticky

### 小花招

需要注意的是，shell、python、perl等脚本文件不能设置SUID位，因为它们事实上是由bash、python、perl解释器解释运行的。要让脚本文件也有类似于SUID这样的功能，需要做一点小花招。

简单地说，我们需要一层壳，这层壳可以设置SUID/SGID位，壳里面真正工作的还是脚本。

比如说我们有一个脚本`/home/jh/bin/myscript.sh`，所有者是普通用户，但脚本里面的操作需要root权限，现在我们用C语言来写这层壳，名称叫做`transeuid.c`：

    /*
    * author: JH Gao <gaopenghigh@gmail.com>
    * Create Date: 2012-06-05
    * Function: transmit euid and egid to other scripts
    * since shell/python/... scripts can't get suid permission in Linux
    */
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #define BUFFSIZE 1024
    /*
    * usually euid is the uid who run the program
    * but when stick is setted to the program
    * euid is the uid or the program's owner
    */
    int main(int argc, char *argv[]) {
        char *cmd = "/home/jh/bin/myscript.sh";
        char *pars[] = {"/home/jh/bin/myscript.sh", "par1", "par2"};
        /* set uid and gid to euid and egid */
        setuid(geteuid());
        setgid(getegid());
        if (execvp(cmd, pars)) {
            printf("error");
            free(cmd);
            exit(1);
        }
        free(cmd);
    }


编译这个程序，在给这个程序设置希望取得的用户，再设置suid，然后就可以用这个用户的权限执行脚本或命令了:

    $ gcc -t transeuid transeuid.c
    $ sudo chown root transeuid
    $ sudo chmod +s transeuid
    $ ./transeuid ......DO SOMETHING

当然具体要执行的脚本和参数可以从外部获取，具体看我以前写的：
[《传递euid和egid给脚本，使脚本具有特殊用户的权限》](http://gaopenghigh.iteye.com/blog/1553535)。

但是需要注意的是，这种花招有很大的安全隐患。


### 最后Linux中解决mp3文件乱码的命令是：

    find . iname "*.mp3" -execdir mid3iconv -e gbk --remove-v1 {} \;

