---
title: Nginx的奇幻冒险：从编译开始
date: 2024-10-21 21:22:42
tags: 
    - nginx
---

## 前言

之前的文章简单地分析了nginx中的内存池分配方法，于是就想着写一系列关于nginx的文章，最好能讲清楚nginx中最核心的内容——nginx是怎么处理用户连接的，为什么nginx能处理上万的并发量，web服务器的原理等等。。。这个系列的名字就叫nginx的奇幻冒险吧٩(•̤̀ᵕ•̤́๑)

## 下载nginx  

  我用的系统是`Ubuntu 20.04.6 LTS`,下载nginx官方提供的包进行解压([下载地址](https://nginx.org/en/download.html)),我安装的版本是1.16.1。执行目录下的configure脚本，默认情况下需要安装两个依赖，zlib和pcre。前者已经安装完毕，后者需要安装dev版本，直接用apt找会定位不到。

  > sudo apt install libpcre3-dev

## 执行configure  

  > ./configure \--with-debug \--with-cc-opt="-g -O0"  

  我为了调试添加了debug和额外的编译器参数`-g`和`-O0`，这两个参数主要为了让gdb调试起来更为直观，避免编译器优化导致无法追溯源码。以下是configure脚本执行后的结果,都是默认的路径

  ``` shell
  Configuration summary
    + using system PCRE library
    + OpenSSL library is not used
    + using system zlib library

    nginx path prefix: "/usr/local/nginx"
    nginx binary file: "/usr/local/nginx/sbin/nginx"
    nginx modules path: "/usr/local/nginx/modules"
    nginx configuration prefix: "/usr/local/nginx/conf"
    nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
    nginx pid file: "/usr/local/nginx/logs/nginx.pid"
    nginx error log file: "/usr/local/nginx/logs/error.log"
    nginx http access log file: "/usr/local/nginx/logs/access.log"
    nginx http client request body temporary files: "client_body_temp"
    nginx http proxy temporary files: "proxy_temp"
    nginx http fastcgi temporary files: "fastcgi_temp"
    nginx http uwsgi temporary files: "uwsgi_temp"
    nginx http scgi temporary files: "scgi_temp"
  ```

## 生成makefile  

  执行完毕之后会生成一个makefile文件和一个objs文件夹，根目录下的Makefile文件内容如下:

  ``` makefile
  default: build

  clean:
    rm -rf Makefile objs

  build:
    $(MAKE) -f objs/Makefile

  install:
    $(MAKE) -f objs/Makefile install

  modules:
    $(MAKE) -f objs/Makefile modules

  upgrade:
    /usr/local/nginx/sbin/nginx -t

    kill -USR2 `cat /usr/local/nginx/logs/nginx.pid`
    sleep 1
    test -f /usr/local/nginx/logs/nginx.pid.oldbin

    kill -QUIT `cat /usr/local/nginx/logs/nginx.pid.oldbin`
  ```

  基本的Makefile文件格式为

  >名称: 依赖  
  >   指令

  执行 `make && make install`就可以安装自己编译的nginx了。makefile会依次执行objs里makefile中的make和install。  
  objs里的Makefile会执行具体的编译器指令，最后生成二进制的可执行文件。install指令依赖默认的build，会生成默认安装的文件夹，最后把二进制文件拷贝到`/usr/local/nginx/sbin`下。由于linux系统完善的安全机制，普通用户使用`make install`在`/usr`下创建文件会出现` Permission denied`的报错，加上`sudo`即可。  

## 生成二进制文件  

  虽然nginx源码比较多，但是没想到gcc的编译速度还挺快的，可能是因为只编译了部分的模块，那我们就来看下nginx编译的结果是啥吧。
  > tree -d ./objs/src/

  ``` shell
  ./objs/src/
  ├── core
  │   ├── nginx.o
  │   ├── ngx_array.o
  │   ├── ngx_buf.o
  │   ├── ngx_conf_file.o
  │   ├── ngx_connection.o
  │   ├── ngx_cpuinfo.o
  │   ├── ngx_crc32.o
  ....
  ├── event
  │   ├── modules
  │   ├── ngx_event.o
  │   ├── ngx_event_accept.o
  │   ├── ngx_event_connect.o
  │   ├── ngx_event_pipe.o
  │   ├── ngx_event_posted.o
  │   ├── ngx_event_timer.o
  │   └── ngx_event_udp.o
  ├── http
  │   ├── modules
  │   ├── ngx_http.o
  │   ├── ngx_http_copy_filter_module.o
  │   ├── ngx_http_core_module.o
  │   ├── ngx_http_file_cache.o
  │   ├── ngx_http_header_filter_module.o
  │   ├── ngx_http_parse.o
  ....
  ├── os
  │   ├── unix
  ```

  这里把每个c文件编译成.o的对象文件，最后将这些对象文件链接成一个可执行的二进制文件，其中不生成任何的库文件。文件较多这里只列出来一部分且没有文件的文件夹未列出，如`os/win32`nginx会依照不同操作系统编译不同的源文件，linux中的系统调用并不适用于window环境，反过来也一样。最后查看安装目录里的内容：

  > ls /usr/local/nginx/sbin/

  结果为`conf  html  logs  sbin`这四个文件夹，sbin中存放的是二进制可执行文件。如果要执行的话就要到对应目录下，挺麻烦的。在环境变量路径里面设置个软连接可以避免输入路径，软连接可以简单理解为window里的快捷方式。

  > ln -s  /usr/local/nginx/sbin/nginx /usr/sbin/nginx

  测试下是否能够正常运行。

  ``` shell
  nginx -h

  nginx version: nginx/1.16.1
  Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]

  Options:
    -?,-h         : this help
    -v            : show version and exit
    -V            : show version and configure options then exit
    -t            : test configuration and exit
    -T            : test configuration, dump it and exit
    -q            : suppress non-error messages during configuration testing
    -s signal     : send signal to a master process: stop, quit, reopen, reload
    -p prefix     : set prefix path (default: /usr/local/nginx/)
    -c filename   : set configuration file (default: conf/nginx.conf)
    -g directives : set global directives out of configuration file
  ```

  ``` shell
  sudo nginx -t

  nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
  nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
  ```
  
  ```shell
  sudo nginx  
  ps -aux | grep nginx

  root       13762  0.0  0.0   4664   392 ?        Ss   11:01   0:00 nginx: master process nginx
  nobody     13763  0.0  0.0   5356  2504 ?        S    11:01   0:00 nginx: worker process
  ssj        13765  0.0  0.0   8168   724 pts/0    S+   11:01   0:00 grep --color=auto nginx
  ```

  上面两个进程就是nginx运行后开启的，nginx.conf中`worker_processes  1`设置worker子进程的数量，所以这边只有一个worker子进程。

## GDB

使用gdb调试看看
> sudo gdb ./obj/nginx
> b nginx.c:204  
> run  

```shell
Starting program: /home/ssj/project/nginx-1.16.1/objs/nginx-deb
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=1, argv=0x7fffffffe648) at src/core/nginx.c:206
206         if (ngx_strerror_init() != NGX_OK) {
```

咦怎么跳过204行了?但是它依旧停下来了，看下代码对不对。

```shell
list

201         ngx_conf_dump_t  *cd;
202         ngx_core_conf_t  *ccf;
203
204         ngx_debug_init();
205
206         if (ngx_strerror_init() != NGX_OK) {
207             return 1;
208         }
209
210         if (ngx_get_options(argc, argv) != NGX_OK) {
```

代码是对的，我本来想看看`ngx_debug_init`做了什么，查一下发现是个空的宏定义。

``` c
// ngx_linux_config.h
#define ngx_debug_init()
```

好吧，总而言之调试环境也算搭建好了。

## 总结

1. 本文主要介绍了源码编译的方式安装nginx，展示了Makefile的编译指令以及编译后的结果文件，对编译过程进行了简单的介绍。
2. 使用了几个较为常见的shell指令，如ps查询nginx的进程信息、ls列出对应文件夹下的内容等。
3. 编译可使用gdb调试的二进制执行文件，并使用简单的gdb指令run，list，break等追踪代码的执行流程。