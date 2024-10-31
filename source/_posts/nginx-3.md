---
title: nginx的奇幻冒险：master和worker模式
date: 2024-11-01 00:29
categories: nginx的奇幻冒险
---

上一篇主要介绍了`ngx_init_cycle`这个函数，着重在讲常规的服务器如何启动、TCP怎样建立连接。本文继续跟着nginx启动步伐，把剩下的流程说完，主要内容包括进程分支操作和信号量处理。

### 判断需要执行的启动模式

nginx定义了两种服务器工作模式，一种为master和worker模式，将进程分为单个master和多个worker，master监管程序控制子进程，worker负责处理连接事件；另一个是single模式表示一个哦进程完成所有的工作。
- 默认模式是master worker模式，下文主要介绍的也是这种模式。生产环境一般也会用这个模式，稳定性和性能都比较好。

```c
// nginx.c
ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
    ngx_process = NGX_PROCESS_MASTER;
}
```

### 初始化信号量

非window系统会进入这个部分，这里的信号量主要用来进行进程间通讯，一般是异步传输的，当进程收到信号之后会先中断当前的操作优先执行信号处理函数。信号可以在进程间相互传递，也可以系统内核向进程传递。
比如kill命令，kill的字面意思容易造成误会让人以为就是直接杀死进程，其实是向进程传输信号。系统默认会向对应进程发送一个`SIGTERM`信号，如果进程没有对这个信号注册对应的函数就会关闭进程。
- kill可以选择传输信号类型 `kill -s SIGNAL PID`
- 如果进程正在执行阻塞的系统调用，在没有信号屏蔽的情况下收到信号后，就会直接中断返回一个异常值

```c
ngx_int_t
ngx_init_signals(ngx_log_t *log)
{
    ngx_signal_t      *sig;
    struct sigaction   sa;

    for (sig = signals; sig->signo != 0; sig++) {
        ngx_memzero(&sa, sizeof(struct sigaction));

        if (sig->handler) {
            sa.sa_sigaction = sig->handler;
            sa.sa_flags = SA_SIGINFO;

        } else {
            sa.sa_handler = SIG_IGN;
        }

        sigemptyset(&sa.sa_mask);
        if (sigaction(sig->signo, &sa, NULL) == -1) {
            ......
        }
    }

    return NGX_OK;
}
```

signals是一个固定的数组代表了所有需要进行处理的信号量，遍历signals调用`sigaction()`为每个信号设置处理函数，大多数信号的处理函数会设置成`void ngx_signal_handler`，这个函数会根据信号类设置不同的全局变量，比如收到`NGX_SHUTDOWN_SIGNAL`就会把`ngx_quit`设置1，这些全局变量的作用后续会出现在`master_cycle`中。

- `sigaction()`用于设置信号的处理函数。sig表示对应的信号标识号，act是设置信号的结构体，oldact用于备份旧的信号配置，如果不需要可以输入空。
  ```c
  #include <signal.h>
  int sigaction(int sig, const struct sigaction *act,struct sigaction *oldact);
  ```

- `struct sigaction`主要作为`sigaction()`的参数，该函数具备更加丰富的参数，因而取代了`signal()`。`sa_handler`和`sa_sigaction`都可以用来设置处理函数，不过两者参数不同。而且两者为一个联合体，共用内存只能取其一。`sa_flag`可以设置特殊的属性，比如`SA_SIGINFO`用于记载信号的详细信息。

```c
struct sigaction{
    union
    {
        /* Used if SA_SIGINFO is not set.  */
        __sighandler_t sa_handler;
        /* Used if SA_SIGINFO is set.  */
        void (*sa_sigaction) (int, siginfo_t *, void *);
    }  __sigaction_handler;
    # define sa_handler	__sigaction_handler.sa_handler
    # define sa_sigaction	__sigaction_handler.sa_sigaction
    __sigset_t sa_mask;
    int sa_flags;
    void (*sa_restorer) (void);
}
```

### 设置daemon进程

>  守护进程是生存期长的一种进程。它们常常在系统引导装入时使用，仅在系统关闭时才终止。
>
> ---《UNIX环境高级编程第三版》

nginx作为一个服务器程序，必然要长时间地运行，比较适合作为守护进程启动，来看下nginx是如何将进程设置为守护进程的吧。

```c
ngx_int_t 
ngx_daemon(ngx_log_t *log)
{
    int  fd;

    switch (fork()) {
    case -1:
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "fork() failed");
        return NGX_ERROR;

    case 0:
        break;

    default:
        exit(0);
    }
    
    ngx_parent = ngx_pid;
    ngx_pid = ngx_getpid();

    if (setsid() == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "setsid() failed");
        return NGX_ERROR;
    }
	umask(0);	
    fd = open("/dev/null", O_RDWR);
    if (fd == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
        "open(\"/dev/null\") failed");
        return NGX_ERROR;
    }

    if (dup2(fd, STDIN_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDIN) failed");
        return NGX_ERROR;
    }
    
    if (dup2(fd, STDOUT_FILENO) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "dup2(STDOUT) failed");
        return NGX_ERROR;
    }
    
 if (fd > STDERR_FILENO) {
        if (close(fd) == -1) {
            ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "close() failed");
            return NGX_ERROR;
        }
    }
 return NGX_OK;
```
创建一个子进程，关闭父进程，设置子进程的属性，重定向子进程的标准输入输出为空地址等。

- `fork()`会创建一个子进程。子进程返回0，父进程返回子进程id，错误为-1。子进程会和父进程一样执行后面的代码，执行顺序是未知的由系统决定。。
- unix系统中每一个进程都有一个父进程，当父进程退出而子进程还在运行时，系统就会修改该子进程的父进程，一般都由init进程担任。
- `umask(0)`的作用则是将进程的权限掩码设置为零，这样就充值了该进程创建文件的权限。
- 打开并获取`/dev/null`文件的描述符，使用`dup2()`把标准输入输出描述符设置为null的描述符，这个操作会把定向到标准输入和输出的数据全部忽略。也就是说这个进程无法直接和用户进行操作。    

如此一来守护进程会忽略标准的输入输出，并且运行在后台不会收到用户终端的影响。

## master_process_cycle

1. #### 设置信号屏蔽集	

```c
sigset_t           set;
sigemptyset(&set);
sigaddset(&set, SIGCHLD);
......
if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) {
	......
}
```

设置信号屏蔽集，屏蔽不需要的信号以免干扰。

- `sigset_t`表示信号集合，`sigemptyset()`将信号置空，`sigaddset()`将特定的信号加入set中。 `sigprocmask(SIG_BLOCK, &set, NULL)`基于set对当前进程设置屏蔽的信号，第三个参数类型是`sigset_t`用于保存旧的信号屏蔽集合，置空表示不需要保存。
- 子进程会继承父进程的信号屏蔽集。

2. #### 创建工作进程

调用`ngx_spawn_process()`创建n个子进程，子进程创建数量可以在配置文件中设置。这个函数设计地十分巧妙，`ngx_spawn_proc_pt`是函数指针，data表示函数proc中的参数，name表示开启的进程名称，respawn表示启动的类型，我们来看下它的结构。

```c
typedef void (*ngx_spawn_proc_pt) (ngx_cycle_t *cycle, void *data);

ngx_pid_t
ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data, char *name, ngx_int_t respawn){
    ngx_int_t  s;

    if (respawn >= 0) {
        s = respawn;
    }else{
        ......
    }

    if (respawn != NGX_PROCESS_DETACHED) {

        if (socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel) == -1)
     	......

        if (ngx_nonblocking(ngx_processes[s].channel[0]) == -1) 
        ......
    }
  
    pid = fork();

    switch (pid) {
 	......
        case 0:
            ngx_parent = ngx_pid;
            ngx_pid = ngx_getpid();
            proc(cycle, data);
            break;
    }
    
    ngx_processes[s].pid = pid;
    ngx_processes[s].exited = 0;
    .......
}
```

子进程最终会调`proc()`也就是传入的函数指针，这就给这个函数极大的自由度，除了创建工作进程以外还可以创建其他的进程。这里的函数指针指向`ngx_worker_process_cycle()`，而父进程会更新ngx_processes里的内容然后退出函数。

- `socketpair()`可以设置两个可以进行双向通信的unix域套接字，在此先不纠结这个操作，等到用到的时候再看。
- nginx的代码很严谨，判断上一步没出错才能会执行下一步，这个编程习惯值得学习。

3. #### master_cycle

```c
for ( ;; ) {
    ......
    sigsuspend(&set);

    if (ngx_reap) {
 		......
    }

    if (!live && (ngx_terminate || ngx_quit)) {
    	......
    }

    if (ngx_terminate) {
        ......
    }
    
}
```

`sigsuspend()`会忽略之前屏蔽的信号集并会一直阻塞等待信号。如果收到信号则执行信号处理函数并返回，上文对信号处理函数的注册简单说过。然后下面就依次判断相关变量并执行相关操作。总之，master进程会循环等待并处理信号。

4. #### worker cycle

```c
for ( ;; ) {

    if (ngx_exiting) {
   		......
    }

    ngx_process_events_and_timers(cycle);

    if (ngx_terminate) {
       ......
    }
    
    if (ngx_quit) {
       ......
    }
}
```

worker进程主要在循环中执行`ngx_process_events_and_timers`进行事件处理，这个函数是nginx能够处理大量连接的关键所在。

## 后话
1. 这次分析main函数中剩余的部分，涉及到信号处理、守护进程等，以及把整个启动的流程简单过了一遍。下回会分析nginx的事件驱动。
2. 作为练手的小项目Minix还会同步更新。这次增加了后台进程的代码，GitHub不太稳定最终决定把代码托管在[gitee](https://gitee.com/sjian06/minix)上。我尽量每次写完文章之后更新相关部分的代码。我认为学习编程的出发点一定是能够设计出实用的程序，纯粹看代码而完全不动手实践是严重脱离编程实质的。
