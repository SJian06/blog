---
title: nginx的奇幻冒险：事件处理
date: 2024-11-08 23:35:34
categories: nginx的奇幻冒险
---

### 前言  

上篇文章我们分析了启动流程最后的步骤，简单地提了一下master和worker的处理逻辑。

最后worker进程会执行`ngx_process_events_and_timers()`用来处理所有的连接，什么样的技术能使它应对百万的并发量呢？我们往下看。

```c
(void) ngx_process_events(cycle, timer, flags);
```

这是个宏定义实际会替换成`ngx_event_actions.process_events`

```c
typedef struct {
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*notify)(ngx_event_handler_pt handler);

    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                                 ngx_uint_t flags);

    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```

不同平台的事件处理技术都不太一样，`ngx_event_actions_t`这个结构体统一对所有平台的事件处理操作进行封装。

- `ngx_event_actions` 是在`ngx_event.c`中定义的具体全局变量，它会在具体事件模块中重新赋值。
- nginx具备很强的跨平台能力，封装了大部分平台的IO复用技术，会按照平台支持情况编译对应的源文件。我目前用的是`Ubuntu`，则编译`ngx_epoll_module.c`。
- IO复用指的是进程可以把IO事件注册到系统中，系统一旦发现注册好的IO条件就绪就会通知该进程进行处理。IO复用这个技术可以让进程同时处理多个IO事件，等到特定的IO条件就绪之后集中进行处理，这比一个个轮询处理所有事件的效率高得多。

`ngx_event_actions`在event模块初始化的时候赋值，要想弄明白事件处理的流程，首先把模块如何加载进nginx搞清楚。之前简单介绍过启动流程，不过省略了很多的代码，模块处理就是其中之一，现在回去看这部分的代码。

### 事件模块加载

nginx包含众多的模块，每个模块都有着属于它的任务，不同模块之间几乎不会进行交互，这很符合软件工程设计里面的高内聚低耦合思想。用户可以选择要添加哪些模块，因此项目中包含的模块是在运行`configure`脚本后才确立的。auto文件夹下的`modules`脚本会在`ngx_module.c`直接写入`ngx_modules[]`的值，这个方法很巧妙不过在此不对编译脚本进行分析，我们尽量把注意力放到运行的代码当中。

我们顺着一个个函数看模块是如何加载的：

1. `ngx_preinit_modules()`会设置全局变量`ngx_modules[]`的index和name，类型为`ngx_module_s`。`ngx_modules_n`表示总的模块数，`ngx_max_module`表示最大模块数，等于`ngx_module_n + 128`。

   ```c
   struct ngx_module_s {
       ngx_uint_t            ctx_index;
       ngx_uint_t            index;
       void                 *ctx;
       ngx_uint_t            type;
       ngx_int_t           (*init_module)(ngx_cycle_t *cycle);
       ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
   	...... 
   }
   ```

2. 在`ngx_init_cycle()`里面调用`ngx_cycle_modules()`，把`ngx_modules[]`的内容复制到`cycle->modules`中，并复制`ngx_modules_n`的值给`cycle->modules_n`，这样`cycle`就包含所有模块信息了。

   ```c
   struct ngx_cycle_s {
   	......
       ngx_module_t            **modules;
       ngx_uint_t                modules_n;
      	......
   };
   ```

   然后遍历数组并筛选类型是`NGX_CORE_MODULE`的模块，调用`ngx_modules[]->ctx`指向的函数指针`create_conf()` 和`init_conf()`，`ctx`是个万能指针会隐式转换为`ngx_core_module_t`类型的临时变量`module` 。

   ```c
   typedef struct {
       ngx_str_t             name;
       void               *(*create_conf)(ngx_cycle_t *cycle);
       char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
   } ngx_core_module_t;
   ```

3. 在worker进程初始化函数中调用`ngx_module[]->init_process()`初始化每个子进程的模块。初步调试下发现很多的模块这个函数指针为空，事件相关的模块中只有`ngx_event_core_module`会执行初始化进程函数，这是下节的讨论重点。

- context简写为ctx，经常被翻译成上下文，我认为这里翻译成容器更合适。
- nginx的模块化设计使得它的扩展性很好，所有的模块可以在某个的阶段统一调用相同名称的函数指针，代码非常简洁。

### 事件模块初始化

调试之后发现事件模块的初始化工作主要是在`ngx_event_core_module`中的`ngx_event_process_init()`完成的。

```c
for (m = 0; cycle->modules[m]; m++) {
    if (cycle->modules[m]->type != NGX_EVENT_MODULE) {
    	continue;
    }
    if (cycle->modules[m]->ctx_index != ecf->use) {
    	continue;
    }
    module = cycle->modules[m]->ctx;
    if (module->actions.init(cycle, ngx_timer_resolution) != NGX_OK) {
        /* fatal */
        exit(2);
    }
	break;
}
```

这个循环会寻找第一个事件模块，于是找到`ngx_epoll_module`，并调用`module->actions.init()`，实际执行`ngx_epoll_init()`对epoll进行初始化。

- 这个循环里的逻辑和上一节中的第二小节很像，不同的是临时变量`module`是`ngx_event_module_t`类型的。

  ```c
  typedef struct {
      ngx_str_t              *name;
      void                 *(*create_conf)(ngx_cycle_t *cycle);
      char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);
      ngx_event_actions_t     actions;
  } ngx_event_module_t;
  ```

- epoll是Linux系统上的一种IO复用技术。由于多数服务器喜欢用Linux操作系统，本系列文章主要基于Linux分析epoll。
- epoll作为select，poll的替代品，它的强大之处体现在处理大量连接的场景下，select需要对每个连接轮训查找待处理的事件，而epoll直接处理活跃的连接，相比之下效率更高。

### 初始化epoll

```c
if (ep == -1) {
    ep = epoll_create(cycle->connection_n / 2);
    if (ep == -1) {
        ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno, "epoll_create() failed");
        return NGX_ERROR;
    }
}
```

`epoll_create()`是Linux系统提供的系统调用，系统内核会创建epoll然后返回描述符。入参代表epoll能够处理事件最大数，这里会输入512，在新的Linux内核版本中这个参数似乎没有作用。

```c
tc = ngx_get_connection(ls[i].fd, cycle->log);
c->type = ls[i].type;
c->log = &ls[i].log;
c->listening = &ls[i];
ls[i].connection = c;

rev = c->read;
rev->log = c->log;
rev->accept = 1;
rev->handler = (c->type == SOCK_STREAM) ? ngx_event_accept : ngx_event_recvmsg;
```

c表示一个空闲的连接，ls在第二篇中讲socket的时候说过表示正在监听的套接字数组，rev表示一个连接的读事件。

- nginx把一个连接封装成`ngx_connection_t`，通过`ngx_get_connection()`获取一个空闲的连接。`ngx_connection_t`的设计也很巧妙是个单向链表，`data`指向前一个connection，每次获取空闲连接就把指针往前移动。具体的`ngx_get_connection()`代码不贴出来了，简单了解即可。

  ```c
  struct ngx_connection_s {
      void               *data;
      ngx_event_t        *read;
      ngx_event_t        *write;
      ngx_socket_t        fd;
  	......
  };
  ```

```c
#define ngx_add_event        ngx_event_actions.add
#define NGX_READ_EVENT     (EPOLLIN|EPOLLRDHUP)
#define NGX_WRITE_EVENT    EPOLLOUT

if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
    return NGX_ERROR;
}
```

向epoll里面添加监听套接字的读事件。这个函数`ngx_add_event()`也是宏，会替换为`ngx_event_actions.add`。实际执行`ngx_epoll_add_event()`会叠加监听事件，关键代码如下：

```c
if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
    ......
}
```

- `epoll_ctl()`用来注册、修改、删除注册在epoll中的事件。epfd表示要操作的epoll描述符。op的值可选，分别是`EPOLL_CTL_ADD`表示添加新的fd到epoll中；`EPOLL_CTL_MOD`表示修改已经注册的事件；`EPOLL_CTL_DEL`表示删除监听的fd。fd表示要监听的fd，event表示具体监听的事件。

  ```c
  int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
  
  typedef union epoll_data
  {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
  } epoll_data_t;
  
  struct epoll_event
  {
    uint32_t events;	/* Epoll events */
    epoll_data_t data;	/* User data variable */
  } __EPOLL_PACKED;
  ```

- `epoll_event`中的data是一个联合体，用户可以自定义数据，最终会通过`epoll_wait()`返回该数据。

- `epoll_event.events`表示epoll的触发事件类型，`EPOLLIN`表示对应的文件描述符可读，`EPOLLOUT`表示对应的文件描述符可以写，`EPOLLHUP`表示对应的文件描述符挂断，`EPOLLET`可以将epoll的触发方式设置为边缘触发。

- epoll有两种触发事件的方式ET和LT。LT表示水平触发(Level Triggered)，是默认的触发方式，一旦有准备好的事件就触发；ET表示边缘触发(Edge Triggered)仅支持非阻塞状态的描述符。

### 处理事件

在说完事件模块的加载是如何加载之后，兜了一圈又回到了起点——如何处理事件。

`ngx_process_events()`也是宏替换成`ngx_event_actions.process_events`，实际执行`ngx_epoll_process_events()`。关于事件处理的操作都是宏替换实际调用的都是`ngx_event_actions`内的函数，这点在上文介绍过。

```c
events = epoll_wait(ep, event_list, (int) nevents, timer);
```

`epoll_wait()`会让进程阻塞并等待内核的通知，返回events表示监听到的事件个数。

- ep表示等待的epollfd。event_list是`epoll_event`指针类型表示发生的事件列表，系统内核会把发生的事件列表复制到这个地址里面所以确保这个指针指向的空间是分配好的且容量足够。nevents表示最大的事件数，这个值不能大于`epoll_create()`传入的值。timer表示等待的时间。
- 使用gdb调试多进程程序需要先设置`set follow-fork-mode child`才能调试子进程。

worker进程进入阻塞状态时我们可以使用浏览器对80端口发起连接请求，epoll就会检测到对应的事件，然后子进程从`epoll_wait()`返回。

```c
836         for (i = 0; i < events; i++) {
(gdb)
837             c = event_list[i].data.ptr;
(gdb)
839             instance = (uintptr_t) c & 1;
(gdb)
840             c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
(gdb)
842             rev = c->read;
(gdb)
844             if (c->fd == -1 || rev->instance != instance) {
(gdb)
856             revents = event_list[i].events;
(gdb) p revents
$28 = 0
(gdb) n
858             ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
(gdb)
862             if (revents & (EPOLLERR|EPOLLHUP)) {
(gdb)
883             if ((revents & EPOLLIN) && rev->active) {
(gdb)
886                 if (revents & EPOLLRDHUP) {
(gdb)
890                 rev->available = 1;
(gdb)
893                 rev->ready = 1;
(gdb)
895                 if (flags & NGX_POST_EVENTS) {
(gdb)
902                     rev->handler(rev);
```

当子进程收到内核返回的数据后通过read事件的handler处理对应的事件，这是经典的事件驱动编程——当某个事件发生就执行对应事件的处理函数，这里会执行`ngx_event_accept()`。

- `rev->handler`是在**初始化epoll**一节注册的

```c
if (use_accept4) {
    s = accept4(lc->fd, &sa.sockaddr, &socklen, SOCK_NONBLOCK);
} else {
    s = accept(lc->fd, &sa.sockaddr, &socklen);
}
......
c = ngx_get_connection(s, ev->log);
.......
ls->handler(c);
```

`accept()`在第二篇的时候简单提到过这边使用了更新版的`accept4()`，这个函数会完成TCP连接的三次握手返回连接成功的套接字描述符s。最后调用ls中注册的处理函数，执行`ngx_http_init_connection()`，涉及到http协议的处理在此略过。

### 总结

本文主要讲了事件模块的加载以及IO复用部分的处理，基本上把nginx主要的启动流程都从头到尾简单过了一遍，主线部分到此就结束了。但是有很多的细节被忽略掉了，比如主进程和子进程之间是如何发消息的，http模块是如何工作的，配置文件怎么读取等等。之后会以补充的形式更新这个系列文章，另外的练手服务器项目`minix`可能会以开发日志的形式更新文章。

- 拆解庞大的项目就像剥洋葱，外层看起来相当复杂(实际上也确实如此)，一层层地剥下去之后才能看到本质。
- 之前粗略看过nginx的源代码，但没有像现在这样自己从头到尾分析过，对于服务器编程技巧完全没有任何的印象，看来学编程最好的方法就是自己动手。

![主要流程图](img/nginx/ngx_event.jpg)

