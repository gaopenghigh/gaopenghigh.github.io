---
layout: post
title: 最简单的 socket 套接字编程
date: 2013-10-23 14:30:00 +0800
categories: Tech
toc: true
---

# 最简单的 socket 套接字编程

本文主要介绍了UNIX环境下socket网络编程的主要概念和步骤，并且附带了一个简单的 服
务器和客户端代码实例，实现了一个网络服务，该服务接受一个字符串的命令，执行该命
令，并且把结 果返回给客户端。分别使用了C语言的多进程、多线程模式，以及Python的
多线程模式实现 。

# 主要概念

## socket

建立socket:

    int socket(int domain, int type, int protocol);
    /*
     * @domain : AF_INET(TCP/IP)
     * @type : SOCK_STREAM | SOCK_DGRAM
     * @protocol : Default 0

socket类型有两种：

* 流式Socket（SOCK_STREAM），一般对应TCP
* 数据报式Socket（SOCK_DGRAM），对于UDP



## bind

`bind`函数将socket与本机上的一个端口相关联，随后你就可以在该端口监听服务请求:

    int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
    /* 成功返回0，出错返回-1并设置errno */

其中`sockaddr`结构用来保存socket信息：

    struct sockaddr {
        unsigned short sa_family;  /* 地址族， AF_xxx */
        char sa_data[14];          /* 14 字节的协议地址 */
    };

另外还可以用一个更方便的结构`sockaddr_in`结构来保存信息：

    struct sockaddr_in {
        short int sin_family;        /* 地址族 */
        unsigned short int sin_port; /* 端口号，设为0表示系统随机选择端口号 */
        struct in_addr sin_addr;     /* IP地址，设为INADDR_ANY表示本机地址 */
        unsigned char sin_zero[8];   /* 填充0 以保持与struct sockaddr同样大小 */
    };

由于大小相等，`sockaddr_in`可以转换为`sockaddr`。`sockaddr_in`中，


## connect

面向连接的客户程序使用Connect函数来配置socket并与远端服务器建立一个TCP连接：

    int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
    /*
     * @sockaddr : 远端服务的地址
     * @addrlen : 远端地址服务的结构
     * 成功返回， 出错返回-1
     */


## listen

Listen函数使socket处于被动的监听模式，并为该socket建立一个输入数据队列，将到达
的服务请求保存在此队列中，直到程序处理它们:

    int listen(int sockfd, int backlog);
    /* 成功返回0，出错返回-1 */

`backlog`表示进程所要入队的连接请求数量，实际值由系统决定，但不能超过
`<sys/socket.h>`中的`SOMAXCONN`，该值默认为128。


## accept

`accept()`函数让服务器接收客户的连接请求:

    int accept(int sockfd, void *addr, int *addrlen);
    /*
     * @sockfd : 被监听的socket描述符
     * @addr : 通常是一个指向sockaddr_in变量的指针，存放client端的信息
     * @addrlen : 常为一个指向值为sizeof(struct sockaddr_in)的整型指针变量
     * 成功时返回0， 出错返回-1 并设置errno
     */


## 数据传输

可以直接用`read()`和`write()`，也可以使用`send()`，`recv()`以及`sendto()`和
`recvfrom()`:

    int send(int sockfd, const void *msg, int len, int flags);
    /*
     * @sockfd :想用来传输数据的socket描述符
     * @msg : 指向要发送数据的指针
     * @len : 以字节为单位的数据的长度
     * @flags : 一般情况下置为0（关于该参数的用法可参照man手册）
     * 成功返回实际上发送的字节数
     */

     int recv(int sockfd,void *buf,int len,unsigned int flags);
     /*
      * 参数定义和 send() 类似
      * 成功返回接受到的字节数
      */

`sendto()`和`recvfrom()`用于在无连接的数据报socket方式下进行数据传输。由于本地
socket并没有与远端机器建立连接，所以在发送数据时应指明目的地址:

     int sendto(int sockfd, const void *msg, int len, unsigned int flags,
                const struct sockaddr *to, int tolen);
     int recvfrom(int sockfd,void *buf, int len,unsigned int flags,
                  struct sockaddr *from, int *fromlen);


## 结束传输

可以调用`close(sockfd)`来释放socket。

还可以使用`shutdown()`函数来关闭socket，该函数允许你只停止在某个方向上的数据传
输，而一个方向上的数据传输继续进行。如你可以关闭某socket的写操作而允许继续在该
socket上接受数据，直至读入所有数据。


# 实例

下面的程序实现了一个网络服务，该服务接受一个字符串的命令，执行该命令，并且把结
果返回给客户端。分别使用了C语言的多进程、多线程模式，以及Python的多线程模式实现
。

## C语言实现

### 客户端

    /* cmd_client.c */
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <arpa/inet.h>
    #include <netdb.h>

    #define BUFLEN 256

    void error(const char *msg)
    {
        perror(msg);
        exit(0);
    }

    int main(int argc, char *argv[])
    {
        int sockfd, portno, n;
        struct sockaddr_in serv_addr;
        struct hostent *server;

        char buffer[BUFLEN];
        if (argc < 3) {
           fprintf(stderr,"usage %s hostname port\n", argv[0]);
           exit(0);
        }
        portno = atoi(argv[2]);

        if ((server = gethostbyname(argv[1])) == NULL) {
            fprintf(stderr,"ERROR, no such host\n");
            exit(0);
        }
        bzero((char *) &serv_addr, sizeof(serv_addr));
        serv_addr.sin_family = AF_INET;
        bcopy((char *)server->h_addr,
              (char *)&serv_addr.sin_addr.s_addr,
              server->h_length);
        serv_addr.sin_port = htons(portno);

        while (1) {
            if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
                error("ERROR opening socket");

            if (connect(sockfd,(struct sockaddr *) &serv_addr,sizeof(serv_addr)) < 0)
                error("ERROR connecting");
            printf(">>> ");
            bzero(buffer, BUFLEN);
            fgets(buffer, BUFLEN, stdin);
            if((n = send(sockfd, buffer, strlen(buffer), 0)) <= 0)
                 error("ERROR writing to socket");
            bzero(buffer, BUFLEN);
            while ((n = recv(sockfd, buffer, BUFLEN, 0)) > 0) {
                printf("%s",buffer);
            }
        }
        return 0;
    }


### 多进程服务器端

    /* A simple server to run system commands use multi process */
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <signal.h>
    #include <unistd.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <netinet/in.h>

    #define PORTNO 4444
    #define BACKLOG 10
    #define BUFLEN 256

    void error(const char *msg) {
        perror(msg);
        exit(1);
    }

    /*
     * function really do the service
     * run cmd geted from client and return the output to client
     */
    void serve(int sockfd) {
        int n;
        char buffer[BUFLEN];
        FILE *fp;

        bzero(buffer, BUFLEN);
        if ((n = read(sockfd,buffer, BUFLEN)) < 0)
            error("ERROR reading from socket");
        printf("CMD : %s\n",buffer);

        if ((fp = popen(buffer, "r")) == NULL)
            error("ERROR when popen");
        while (fgets(buffer, BUFLEN, fp) != NULL) {
            if (send(sockfd, buffer, BUFLEN, 0) == -1)
                error("send ERROR");
        }
        pclose(fp);
        close(sockfd);
        exit(0);
    }

    /*
     * Init listen socket and bind it to addr
     */
    int init_server() {
        int sockfd;
        struct sockaddr_in serv_addr;

        if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
            error("ERROR opening socket");

        bzero((char *) &serv_addr, sizeof(serv_addr));
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = INADDR_ANY;
        serv_addr.sin_port = htons(PORTNO);

        if (bind(sockfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) < 0)
            error("ERROR on binding");

        return sockfd;
    }

    /*
     * Use a child process to serve for every connection
     */
    int main(int argc, char *argv[]) {
        signal(SIGCHLD,SIG_IGN);       /* do not care about child process */

        pid_t pid;
        int sockfd = init_server();
        int newsockfd;
        struct sockaddr_in cli_addr;
        socklen_t clilen = sizeof(cli_addr);

        listen(sockfd, BACKLOG);

        while (1) {
            if ((newsockfd = accept(sockfd, (struct sockaddr *)&cli_addr, &clilen)) < 0)
                error("ERROR on accept");
            if ((pid = fork()) < 0) {
                error("Fork Error");
            } else if (pid == 0) {          /* child */
                serve(newsockfd);
            } else {                        /* parent */
                close(newsockfd);
            }
        }
    }


### 多线程服务器端

    /* A simple server to run system commands use multi threads */
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <signal.h>
    #include <unistd.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <pthread.h>

    #define PORTNO 4444
    #define BACKLOG 10
    #define BUFLEN 256

    void error(const char *msg) {
        perror(msg);
        exit(1);
    }

    /*
     * function really do the service in sub thread
     * run cmd geted from client and return the output to client
     */
    void *serve(void *arg) {
        int sockfd = *(unsigned int *)arg;
        int n;
        char buffer[BUFLEN];
        FILE *fp;

        printf("start serve...\n");
        bzero(buffer, BUFLEN);
        if ((n = read(sockfd, buffer, BUFLEN)) < 0)
            error("ERROR reading from socket");
        printf("3start serve...\n");
        printf("CMD : %s\n",buffer);

        if ((fp = popen(buffer, "r")) == NULL)
            error("ERROR when popen");
        while (fgets(buffer, BUFLEN, fp) != NULL) {
            if (send(sockfd, buffer, BUFLEN, 0) == -1)
                error("send ERROR");
        }
        pclose(fp);
        close(sockfd);

        printf("end serve...\n");
        return(0);
    }

    /*
     * Init listen socket and bind it to addr
     */
    int init_server() {
        int sockfd;
        struct sockaddr_in serv_addr;

        if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
            error("ERROR opening socket");

        bzero((char *) &serv_addr, sizeof(serv_addr));
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = INADDR_ANY;
        serv_addr.sin_port = htons(PORTNO);

        if (bind(sockfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) < 0)
            error("ERROR on binding");

        return sockfd;
    }
    /*
     * Use a sub threads to serve for every connection
     */
    int main(int argc, char *argv[]) {
        signal(SIGCHLD,SIG_IGN);

        int sockfd = init_server();
        int newsockfd;
        struct sockaddr_in cli_addr;
        socklen_t clilen = sizeof(cli_addr);

        listen(sockfd, BACKLOG);

        while (1) {
            if ((newsockfd = accept(sockfd, (struct sockaddr *)&cli_addr, &clilen)) < 0)
                error("ERROR on accept");
            printf("newsockfd=%d\n", newsockfd);
            pthread_t t;
            int fd_in_thread = newsockfd;
            if(pthread_create(&t, NULL, serve, (void *)&fd_in_thread) != 0)
                error("ERROR on pthread_create");
        }
    }


## Python实现

### 客户端

    #!/usr/bin/evn python
    # -*- coding:utf-8 -*-

    from socket import *
    import commands

    HOST = "localhost"
    PORT = 4444
    ADDR = (HOST, PORT)
    BUFLEN = 256
    BAKLOG = 10

    def client():
        while True:
            clisock = socket(AF_INET, SOCK_STREAM)
            clisock.connect(ADDR)
            cmd = raw_input("\n>>> ")
            clisock.send(cmd)
            while True:
                data = clisock.recv(BUFLEN)
                print data
                if not data:
                    break

    if __name__ == '__main__':
        client()


### 多线程服务器端

    #!/usr/bin/evn python
    # -*- coding:utf-8 -*-

    import threading
    import signal
    from socket import *
    import commands
    import sys

    HOST = ""
    PORT = 4444
    ADDR = (HOST, PORT)
    BUFLEN = 256
    BAKLOG = 10


    def server():
        listensock = socket(AF_INET, SOCK_STREAM)
        listensock.bind(ADDR)
        listensock.listen(BAKLOG)

        while True:
            clisock, addr = listensock.accept()
            t = ServeThread(clisock)
            t.start()

        listensock.close()

    class ServeThread(threading.Thread):
        def __init__(self, sock):
            threading.Thread.__init__(self)
            self.sock = sock

        def run(self):
            cmd = self.sock.recv(BUFLEN)
            print "CMD:%s" % cmd
            output = commands.getstatusoutput(cmd)[1]
            self.sock.send(output)
            self.sock.close()
            sys.exit(0)


    if __name__ == '__main__':
        server()


----

参考资料：

* 《UNIX环境高级编程》
* [Sockets Tutorial](http://www.linuxhowtos.org/C_C++/socket.htm)
* [Unix环境下的Socket编程](http://bbs.chinaunix.net/thread-92997-1-1.html)

----

<div id="disqus_thread"></div>
<script type="text/javascript">
/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
    var disqus_shortname = 'gaopenghigh'; // required: replace example with your forum shortname

    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-40539766-1', 'github.com');
  ga('send', 'pageview');

</script>

