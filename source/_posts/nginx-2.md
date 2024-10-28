---
title: nginx的奇幻冒险：启动服务器
date: 2024-10-27 23:44:49
tags:
  - nginx
  - unix网络编程
---

### 前言
本文的重点在于如何启动一个服务器，简单介绍服务器的编程模式是怎么样的以及TCP协议的连接机制。由于笔者是初次接触网络编程，如果文章中出现明显的错误在此表示抱歉。文中的实例代码会保证能够正常运行，并且随着后续的学习进行改善。接下来让我们踏入unix网络编程的大门吧。

### 简析nginx服务的启动流程
首先寻找nginx的启动入口——`nginx.c`文件当中的main函数。大致浏览发现大多代码基于`cycle`做一些初始化的工作。`ngx_cycle_s`的结构体大致如下，这里截取了一部分。我们并不需要一口气把这个结构体内的所有内容全部搞明白，抓住目前的重心，找到与启动服务器想关的结构即可。

```c
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;
    
    ngx_array_t               listening;
    ngx_array_t               paths;
    ......
}
```

我们可以将ngx_cycle_t简单理解成整个程序的容器，它包含了整个程序运行时需要用到的所有模块，因此里面的结构非常复杂。其中listening和启动并监听套接字有关，下文会关注这个字段。

1. cycle的初始化
main函数会进入`ngx_cycle_t *ngx_init_cycle(ngx_cycle_t *old_cycle)`这个函数，初始化cycle变量。这里面有接近九百行代码，使用`GDB`查看代码执行顺序后会发现里面的大部分代码是不会执行的，主要执行了`ngx_int_t ngx_open_listening_sockets(ngx_cycle_t *cycle)`。

2. listening的初始化
`listening`的类型为`ngx_array_t`，属于是项目中封装的对象数组，内部使用万能指针指向任意类型的对象数组，并存储在pool指向的内存池当中。

```c
typedef struct {
    // 对象数组
    void        *elts;
    // 对象数量
    ngx_uint_t   nelts;
    // 对象大小
    size_t       size;
    // 数组容量
    ngx_uint_t   nalloc;
    // 用于存储的内存池
    ngx_pool_t  *pool;
} ngx_array_t;

// ngx_init_cycle()
n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

if (ngx_array_init(&cycle->listening, pool, n, sizeof(ngx_listening_t))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}

ngx_memzero(cycle->listening.elts, n * sizeof(ngx_listening_t));
```

`ngx_open_listening_sockets()`会根据对`cycle->listening.elts`中的值创建套接字。elt指向`ngx_listening_s`结构的数组，该结构体包含了基本的socket创建信息。

```c
struct ngx_listening_s {
// 套接字标识符
    ngx_socket_t        fd;
// 套接字地址
    struct sockaddr    *sockaddr;
//  套接字大小
	socklen_t           socklen;    /* size of sockaddr */
    size_t              addr_text_max_len;
    ngx_str_t           addr_text;

    int                 type;

    int                 backlog;
    int                 rcvbuf;
    int                 sndbuf;
    ngx_connection_handler_pt   handler;
......
}
```

3. 创建套接字并监听
以下就是最核心的部分，创建套接字并设置套接字选项，最后将本机地址绑定给套接字并把套接字设置为监听状态。

``` c
 s = ngx_socket(ls[i].sockaddr->sa_family, ls[i].type, 0);
if (s == (ngx_socket_t) -1) {
    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                  ngx_socket_n " %V failed", &ls[i].addr_text);
    return NGX_ERROR;
}

if (setsockopt(s, SOL_SOCKET, SO_REUSEADDR,
               (const void *) &reuseaddr, sizeof(int))
    == -1)
{
    ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                  "setsockopt(SO_REUSEADDR) %V failed",
                  &ls[i].addr_text);

    if (ngx_close_socket(s) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                      ngx_close_socket_n " %V failed",
                      &ls[i].addr_text);
    }

    return NGX_ERROR;
}
.....
if (bind(s, ls[i].sockaddr, ls[i].socklen) == -1) {
    ......
}
.....
if (listen(s, ls[i].backlog) == -1) {
    ......
}
ls[i].listen = 1;

ls[i].fd = s;
```

再进入`ngx_configure_listening_sockets` 进行调试发现就是进行一系列的判断，最后重复listen socket，目前不知道有什么意义。

```shell
(gdb) n
728             if (ls[i].rcvbuf != -1) {
(gdb) n
739             if (ls[i].sndbuf != -1) {
(gdb) n
750             if (ls[i].keepalive) {
(gdb) n
765             if (ls[i].keepidle) {
(gdb) n
782             if (ls[i].keepintvl) {
(gdb) n
799             if (ls[i].keepcnt) {
(gdb) n
826             if (ls[i].fastopen != -1) {
......
```

然后是`ngx_init_modules()`，作用就是调用模块对应的初始化回调函数，这边不细看代码。好的，`ngx_init_cycle`的主要流程就先讲到这里，进行了大量的初始化先略过，这个函数的核心内容在于对网络编程相关接口的调用，并启动了一个端口用户网络通讯，接下来看下细节。

### 基本的服务器启动流程
socket() -> bind() -> listen() ->  accept() 

1. 创建套接字，这是进行端到端的通讯基础，无论是服务端还是客户端建立连接之后，都需要通过套接字读取对方传过来的数据，或者是通过套接字去写要发送给对方的数据，可以说套接字是网络编程的基本管道。
   ```c
   #include<sys/socket.h>
   // success return fd which > 0 else return -1
   int socket(int family, int type, int protocol);
   ```
2. 服务端需要将套接字绑定到一个IP地址和端口当中用作连接的地址，当客户端连接这个地址的时候，路由器把客户端的数据发送到对应的主机上。
   ```c
   #include<sys/socket.h>
   //success return 0 else -1
   int bind(int sockfd,const struct sockaddr* addr, socklen_t socklen);
   ```
   sockfd表示要绑定的套接字标识符，也就是调用`socket()`成功后返回的一个非负整数，代表这个套接字。addr需要由`sockaddr_in`强制转换成`sockaddr`，着存在着一个历史因素，当初接口设计的时候不存在万能指针，于是开发者们就规定输入一个通用的结构体。socklen表示套接字地址结构体的大小。
3. 开启套接字的监听模式，默认创建的套接字处于主动模式，即该套接字可以指定对方的地址并尝试进行连接并写入数据并发送。服务器则需要将绑定本机地址的套接字转为监听模式，这样一来当客户端发起连接，主机就可以监听到对应的IP和端口发来的请求，并通过套接字进行读写数据。
   ```c
   #include<sys/socket.h> 
   //success return 0 else -1
   int listen(int sockfd,int backlog);
   ```
   backlog表示内核中对应的套接字排队进行连接的最大个数，这个数值在nginx当中默认设为511，越大自然表示允许处理的socket数越多。这个参数不同的操作系统实际会有不同的实现，也就是说backlog的大小不一定代表实际的最大连接数，在Linux 2.4.7中实际数是该值加三，详见参考书籍p87图4-10。
   内核会给监听的套接字创建两个用于存储连接的队列，分别是已完成和未完成队列。已完成队列存储双方连接建立成功的套接字，未完成队列存储还未连接成功的套接字。一个连接建立成功之后就会转移到已完成队列当中，如果队列已满，系统会忽略客户端发送过来的请求，基于tcp的重传机制会重新传送请求数据。两个队列套接字总和不会超过backlog设定的大小。连接过程由操作系统内核进行处理，下一个部分会细说tcp是如何进行连接的。
4. 处理连接。accept可以获取一个已完成的tcp连接，由于套接字默认情况下是阻塞模式，也就是说如果没有可处理的连接，进程会阻塞直到新的连接到来。成功后会返回一个新的描述符表示与客户端进行的tcp连接，可以通过这个描述符向读取或者写入数据。
   ```c
   #include<sys/socket.h> 
   // // success return fd which > 0 else return -1
   int accept(int sockfd,struct sockaddr* sockaddr,socklen_t socklen);
   ```

### TCP是如何建立连接的
我们主要研究基于tcp的服务器开发，因此必须要知道TCP是如何创建和取消连接的，根据传送的分节数量，可以将这两个流程简称为三次握手和四次挥手。
#### 三次握手
传统的tcp网络连接模型中建立连接往往由客户端发送，发送的数据类型分为ack和syn分节，这两种分节一般会包含ip和tcp的首部以及一些tcp选项，通过互相传送两个类型的分节确认并建立tcp连接。
1. 客户端调用`connect()`发送一个syn分节(设它的序号为k)，并发送给服务端通知它要开始建立连接了，然后等待服务端的确认。客户端状态从**closed**转向为**syn_sent**，如果消息超时或者是连接失败则客户端会回到**closed**状态。
2. 服务端收到客户端发送的syn分节之后会发送ack分节，其序号为k+1，并且也会发送一个syn分节，设序号为j。服务端会从**listen**的状态转换为**syn_rcvd**。
3. 客户端收到两个分节后并确认后会传送一个ack分节其序号为j+1给服务端，此时客户端转换成**established**状态。当服务端收到客户端发来的确认分节之后，服务端也会转为**established**状态，这样两者之间的连接就算建立成功了。
#### 四次挥手
客户端和服务端都可主动关闭连接，以下假设客户端主动关闭，整个流程涉及到的两种分节分别是fin和ack。
1. 客户端调用`close()`主动关闭，发送序列号为j的fin分节，随后进入**fin_wait_1**状态
2. fin分节会排在要传输数据之后，当服务端收到fin分节就知道客户端要准备关闭连接了，就会发送序列号为j+1的ack分节向客户端确认关闭。此时服务端进入**close_wait**状态。
3. 过一段时间后服务端会调用`close()`关闭与客户端通讯的套接字，如果这个时候还在传输数据则称为半关闭，结束后会发送fin分节，设序列号为j。此时服务端进入**last_ack**状态。
4. 客户端收到了fin分节，发送序列j+1的ack分节确认关闭。随后客户端进入**time_wait**状态，而服务端收到了收到ack分节之后会进入**closed**状态。如果在规定时间内未接收到服务端的数据，客户端也会转为**closed**状态，至此双方连接完全结束。

### 示例服务器：MiniX
使用上述基本的几个接口写一个简单的服务器程序和客户端程序练习，服务端会向连接完成的客户端发送固定的数据。完整的示例代码如下，包含一个头文件和两个源文件。
```c
// minix.h
#ifndef MINIX_H__
#define MINIX_H__
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

#define MAX_MSG_LEN 1000
#define MINIX_ERROR -1
#endif
```

```c
// client.c
#include"minix.h"

int main(int argc, char const *argv[]) {
  const int port = 9090;

  // sock connect
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  struct sockaddr_in addr;
  addr.sin_family = AF_INET;
  addr.sin_port = htons(port);
  addr.sin_addr.s_addr = htonl(INADDR_ANY);

  int con_res = connect(sockfd, (struct sockaddr *)&addr, sizeof(addr));
  if (con_res == 0) {
    char msg[MAX_MSG_LEN];
    read(sockfd, msg, MAX_MSG_LEN);
    printf("receive server msg %s\n", msg);
  }
  return 0;
}
```


```c
// server.c
#include"minix.h"

#define flog(f, msg)                                \
  fprintf((f), "%s - %s\n", get_cur_time(), (msg)); \
  fflush((f))

char *get_cur_time() {
  time_t now = time(NULL);
  char *time = ctime(&now);
  time[strlen(time) - 1] = '\0';
  return time;
}

void minix_exit(char *errormsg) {
  perror(errormsg);
  exit(MINIX_ERROR);
}

void start_minix() {
  const int port = 9090;
  int reuse = 1;

  // log
  FILE *f = fopen("server.log", "a+");
  if (f == NULL) {
    minix_exit( "can not open server log!!!");
  }

  // sock
  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

  // sockaddr
  struct sockaddr_in addr;
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = htonl(INADDR_ANY);
  addr.sin_port = htons(port);

  if (bind(sockfd, (struct sockaddr *)&addr, sizeof(addr)) == -1) {
    minix_exit( "can not bind sock!!!");
  }
  if (listen(sockfd, 10) == -1) {
    minix_exit( "can not listen sock!!!");
  }
  flog(f, "start server success!!!");
  struct sockaddr_in clientaddr;
  memset(&clientaddr, 0, sizeof(clientaddr));
  size_t clilen = sizeof(clientaddr);
  char *buf = (char *)malloc(MAX_MSG_LEN * sizeof(char));

  for (;;) {
    int con_fd = accept(sockfd, (struct sockaddr *)&clientaddr, &clilen);
    if (con_fd == -1) {
      continue;
    }
    // convert client port message
    uint8_t port = ntohs(clientaddr.sin_port);
    char *ip_addr =
        inet_ntop(AF_INET, &clientaddr.sin_addr.s_addr, buf, clilen);
    printf("cli port: %d, ipaddr: %s\n", port, ip_addr);
    fflush(stdout);

    // send msg to client
    char *msg = "hello";
    write(con_fd, msg, strlen(msg));
    close(con_fd);
  }
  fclose(f);
  close(sockfd);
}

int main(int argc, const char *argv[]) {
  start_minix();
  return 0;
}
```

分别启动server和client，server成功启动会写入日志文件中，当client成功连接上server会让server的控制台打印client socket的地址信息。服务端会向客户端发送hello。

### 参考书籍
 Richard Stevens , Bill Fenner , AndrewM.Rudoff UNIX网络编程 卷1：套接字联网API (第三版) 北京：人民邮电出版社 2010.7