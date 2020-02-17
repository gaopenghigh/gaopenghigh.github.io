---
layout: post
title: ip_conntrack 错误处理及原理
date: 2014-01-14 14:30:00 +0800
categories: Tech
toc: true
---

# ip_conntrack 错误处理及原理

## ip_conntrack: table full, dropping packet

服务器上出现了这样的错误(/var/log/messages):

    ip_conntrack: table full, dropping packet.

查了一些资料，原因是使用了iptables，服务器的连接数太大，内核的
**Connection Tracking System(conntrack)**没有足够的空间来存放连接的信息，解决
方法就是增大这个空间。

1. 查看当前大小：

        $ sysctl net.ipv4.netfilter.ip_conntrack_max
        net.ipv4.netfilter.ip_conntrack_max = 65535

2. 增大空间，/etc/sysctl.conf修改或新增下面内容：

        net.ipv4.netfilter.ip_conntrack_max = 655350

3. 生效：

        $ sysctl -p

那么，什么是**Connection Tracking System(conntrack)**，它是如何工作的，和
iptables的关系是什么，为了弄清这些问题，我又查了一些资料，整理如下。

## Netfilter

> Netfilter is a set of hooks inside the Linux kernel that allows kernel
> modules to register callback functions with the network stack. A registered
> callback function is then called back for every packet that traverses the
> respective hook within the network stack.[1](http://www.netfilter.org/)

简单地说，**Netfilter Framework**通过在 Linux network protocol stack 上的一系列
hooks，提供了一种机制，使得内核模块可以在 network stack 中注册一些回调函数，每
个网络包的传输都会经过这些回调函数。

而iptables是基于Netfilter Framework上的一套工具，运行于用户态，用于配制网络包的
过滤规则。由于iptables的chains和hooks和Netfilter Framework有同样的名字，但
iptables只是在Netfilter Framework上的一个工具而已。

### The Hooks and Callback Function

Netfilter在Linux network stack中插入了5个hooks，实现了在不同的阶段对包进行处理
。

*   **PREROUTING**: 所有的包都会进过这个hook，在路由之前进行。DNAT等
    就是在这一层实现。
*   **LOCAL INPUT**: 所有要进入本机的包都经过这个hook.
*   **FORWARD**: 不进入本机的包经过这个hook.
*   **LOCAL OUTPUT**: 离开本机的包经过这个hook.
*   **POSTROUTING**: 经过路由之后的包会经过这个hook，SNAT就在这一层实现。所有由
    本机发出的包都要经过这个hook.

```
NF_IP_PRE_ROUTING              NF_IP_FORWARD       NF_IP_POST_ROUTING
      [1]      ====> ROUTER ====>    [3]    =============> [4]
                       ||                        /\
                       ||                        ||
                       ||                      ROUTER
                       \/                        ||
                      [2] ===> LOCAL PROCESS ===>[5]
                NF_IP_LOCAL_IN              NF_IP_LOCAL_OUT
```

可以在一个hook上注册callback函数，callback的返回下面的某个值：

*   ACCEPT
*   DROP
*   QUEUE: 通过`nf_queue`把包传到用户空间；
*   STOLEN: Silently holds the packet until something happens, so that it
    temporarily does not continue to travel through the stack. This is usually
    used to collect **defragmented IP packets**.也就是说暂停包的传输直到某个条
    件发生；
*   REPEAT: 强制这个包重新走一遍这个hook；

总之就是，Netfilter Framework提供了一个框架，可以在包传输的不同阶段，通过回调
函数的方式对包进行过滤。

上面提到了"defragmented IP packets"，
Wikipedia[解释](http://en.wikipedia.org/wiki/IP_fragmentation)如下；

> The Internet Protocol (IP) implements datagram fragmentation, breaking it
> into smaller pieces, so that packets may be formed that can pass through a
> link with a smaller maximum transmission unit (MTU) than the original
> datagram size.

简单地说就是数据包的长度大于了MTU的大小，就会把数据包分片，装在多个较小的包里面
传输出去。

### The Connection Tracking System and the Stateful inspection

Connection Tracking System, which is the module that provides stateful packet
inspection for iptables.

Basically, the connection tracking system stores information about the state
of a connection in a memory structure that contains the source and destination
IP addresses, port number pairs, protocol types, state, and timeout.
With this extra information, we can define more intelligent filtering policies.
Connection tracking system just tracks packets; it does not filter.
([Netfilter’s connection tracking system](http://people.netfilter.org/pablo/docs/login.pdf))

#### 连接的状态

在conntrack系统中，一个连接可能有如下的状态：

*   NEW: 连接正在建立，比如对于TCP连接，收到了一个SYN包；
*   ESTABLISHED: 连接已经建立，可以看到“来往”的包；
*   RELATED: 关联的连接；
*   INVALID: 不合法的；

所以，即使像是UDP这样无状态的协议，对于Connection Tracking System也是有状态的。


#### 实现

conntrack系统主要使用一个hash表来检索查询。表中的每一项，都是一个双链表。（
Each bucket has a double-linked list of hash tuples.）一个连接有两个hash tuples
，一个是“来”（包来自于建立连接的那一方）方向，一个是“回”方向。每个tuple都存了这
个连接的相关信息，两个tuple的又组织在`nf_conn`结构中，该结构就代表了一个连接的
状态。

![](/assets/pictures/conntrack_structure.png)

Hash表中hash值的计算是基于3层和4层的一些协议信息，同时引入了一个随机量防止攻击
。conntrack表有一个最大容量，表充满时，就会选择一个最近使用时间最早的conntrack
丢弃。

回调函数`nf_conntrack_in`注册在PREROUTING hook上，它会检查包的合法性，并且在表
中查询这个包是否属于哪个conntrack，如果没找到的话，一个新的conntrack就会被创建
，并且其中`confirmed`标志没有被设置。在LOCAL INPUT和POSTROUTING上注册的
`nf_conntrack_cofirm`函数，会把一个conntrack的`confirmed`标志设置上。对于进入本
机或者forward出去的包，这两个hook是包最后的经过的hook，如果这时包还没有被丢弃的
话，就设置`confirmed`位并且把新建的conntrack加入到hash表中。


##### Helpers and Expectation

一些应用层的协议不容易被追踪，比如FTP的passive mode，使用21端口做控制，另外又用
一个随机端口获取数据。对于用户来说 这两个连接是联系在一起的(related)。

Conntrack系统提供了一种叫做helper的机制，使得系统能够判断一个连接是否和已经存在
的某个连接有关系。改机制定义了**expectation**的概念，一个expectation指在一个
预期的时间段内出现的连接。那FTP来说，helper在返回的包中寻找数据传输端口的相关信
息，找到的话，找到的话，一个expectation就被创建并被插入到expectation的链表中。

当一个conntrack创建时，conntrack系统首先会寻找是否有匹配的expectation，没有的话
就会对这个连接使用helper。如果找到匹配的expectation，新的conntrack就会和创建那
个expectation的conntrack关联起来。


JH, 2014-01-14

----

参考资料：

1. [http://www.netfilter.org/](http://www.netfilter.org/)
2. [Netfilter’s connection tracking system](http://people.netfilter.org/pablo/docs/login.pdf)
