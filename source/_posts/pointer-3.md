---
title: 指针实战技巧：自定义内存分配(以Nginx内存池为例)
date: 2024-10-07 23:49:50
tags:
    - C
    - 指针
    - nginx
---

## 前言

这篇文章陆陆续续拖了蛮久的，本来打算国庆假期结束前写完的，结果到了国庆后的一周才把文章结束掉，实在是拖延症病入膏肓。
关于C语言的指针前面两篇文章进行了简单的介绍，我本为到此即可，直到看到了Nginx的源代码后为之折服。指针还可以更加灵活，更加强大。nginx中的指针用法看似简单随意，实则别有用心。所以本文尝试以指针的视角对Nginx的内存池主要代码进行分析，在实战当中进一步领略指针的强大之处。

## 内存模型

要想知道指针在内存中做了什么事情，首先要会分析内存是如何分配的，又是如何在计算机当中存储数据的。计算机的内存只会存0和1这两个数，而内存的基本存储单位是字节，一个字节也就是八位二进制数，内存存储的数据以及地址数的模型如下。

![内存模型图](img/pointer/memory_simple.jpg)  

上面的数字是地址号，用来标志数据在内存中的位置，矩形中的就代表了内存实际存储的数据。两个地址号相减可以算出这两个地址之间的容量大小，单位是字节。

## 何为内存池

我们经常能够听到线程池，内存池几乎在别的语言没听说过。那是因为C语言不同于别的GC语言，它需要开发者自己去管理内存分配，而Java、Python等语言就不需要考虑甚至可以无视这个问题。自己管理内存就要意味着需要手动开辟空间以及销毁空间，为了更统一地管理内存，就需要通过内存池来全局管理内存的分配。
线程池和内存池的共同点在于都反映出一个池化的思想——将初始化好的资源放入池子当中。当你需要使用资源时候就可以直接从池子里获取，用不着要的时候放回去，从而实现资源复用。

## Nginx中的内存池

Nginx的大名想必不用我多说吧，在Web领域中几乎无人不知无人不晓，它经常用来进行负载均衡和反向代理，可以轻松面对上万的并发数，因而广泛地用于生产环境当中。
Nginx的模块很多，结构非常复杂。我们重点看下其中的内存池是怎么工作的，并且尽量以指针的视角去看，不过分关注那些细枝末节。进入主题，首先列出Nginx当中的用来表示内存池的相关结构体以及主要函数

```c
typedef struct ngx_pool_large_s  ngx_pool_large_t;
typedef struct ngx_pool_s ngx_pool_t;

struct ngx_pool_large_s {
    ngx_pool_large_t     *next;
    void                 *alloc;
};

typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;


struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};

// 主要的函数
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
void ngx_destroy_pool(ngx_pool_t *pool);
void ngx_reset_pool(ngx_pool_t *pool);
// 分配内存相关
void *ngx_palloc(ngx_pool_t *pool, size_t size);
void *ngx_pnalloc(ngx_pool_t *pool, size_t size);
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);
// 这三个函数不对外进行链接
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align);
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)

// 本文关注的Nginx版本为 release-1.16.0
```
主要的结构体为`ngx_pool_t`和`ngx_pool_data_t`，data结构体嵌套在pool当中，并注意data不是指针类型，结构图如下。
![结构体组成](img/pointer/ngx_pool.jpg)

### ngx_create_pool

```c
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;
    }

    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;

    size = size - sizeof(ngx_pool_t);
    // MAX SIZE is max_integer - 1
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    p->current = p;
    // 这些不用管
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```

`ngx_create_pool`函数的作用就是创建线程池对象并进行必要的初始化，最终返回指向内存池的指针。

```c
 // ngx
 p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
 // 常规写法
 p = (ngx_pool_t *)malloc(sizeof(ngx_pool_t));
```

这里使用了`ngx_memalign`进行空间分配，这个函数很陌生，究竟有什么作用呢？这个函数会先判断，操作系统说服存在内存对齐的函数，存在就直接调用，如果不存在就使用malloc进行分配。
Nginx不愧为全球最受欢迎的web服务器之一，竟然连内存对齐这样的细节都不愿意放过，其实我也不清楚内存对齐具体的意义，不过在这边我们完全不需要在意这点，只当这个函数为`malloc`即可。
这里需要关注这两个赋值的语句，为什么last能够通过p相加得到结果？

> p->d.last = (u_char \*) p + sizeof(ngx_pool_t);
> p->d.end = (u_char \*) p + size;

由于u_char类型的大小是一个字节，在进行加减法时基本单位是一个字节，因此内存可以直接通过大小去计算地址位置。last存储的是未占用的开始地址，end存储的是已分配内存的结尾地址，整个过程如下图所示

![create_pool的内存模型](img/pointer/create_pool.jpg)

### ngx_palloc

```c
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max) {
        return ngx_palloc_small(pool, size, 1);
    }
#endif

    return ngx_palloc_large(pool, size);
}
```

这个函数主要从内存池中获取size大小的内存。如果size比内存池中的max小，就使用`ngx_palloc_small`进行分配，否则使用`ngx_palloc_large`。 `ngx_palloc_large` `ngx_palloc_small` `ngx_palloc_block`这三个函数是分配内存的实际作用函数，并且都使用static修饰，不对外编译链接。接下来分别看下这三个函数的代码。

### ngx_palloc_small

```c
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;

    p = pool->current;

    do {
        m = p->d.last;
        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }

        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;
            return m;
        }

        p = p->d.next;
    } while (p);

    return ngx_palloc_block(pool, size);
}

```

small这个函数负责获相对小的内存，并返回内存池中可用的内存地址。

```c
if ((size_t) (p->d.end - m) >= size) {
    p->d.last = m + size;
    return m;
}
```

这边判断剩余的空间是否比size大，是则更新d.last的值，重新计算可用空间的地址号。然后返回m，指向一个size大小的空间，这个空间和current指向的pool空间处于一个连续的内存空间当中。
如果比size小就说明内存池中的容量不够分了，那就遍历下一个可用的pool节点，继续判断。到最后如果所有的节点都无法分配size大小的空间，就调用`ngx_palloc_block`分配一个新的pool节点。

### ngx_palloc_large

```c
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;

    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    n = 0;

    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

large函数首先会直接分配size内存给p，然后遍历pool中的large链表，如果large中的alloc指针为空，则把p存的地址赋予alloc并返回p。
如果`pool->large`指针为空或者前三个large中alloc都赋值(3就是个随意的值，如果链表前三个节点的alloc已经分配了值，那么后面的节点极有可能一样已经赋值，就不必要再去查找)，那么就通过small函数使用内存池存储一个large变量，并将这个large的节点首部插入到链表`pool->large`中。如下图所示

![large内存模型](img/pointer/ngx_large_t.jpg)

### ngx_palloc_block

```c
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size){
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;

    psize = (size_t) (pool->d.end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;

    for (p = pool->current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            pool->current = p->d.next;
        }
    }

    p->d.next = new;

    return m;
}
```

这个函数的作用是添加新的pool节点，看到这里会对`ngx_pool_t`产生新的认知。先说下函数的流程吧，使用`ngx_memalign`分配psize大小内存，psize是pool指向内存的总大小，pool这个指针一般不会变，所以这个大小一般是固定的，也就是说每个新的pool节点分配的空间都是固定的。在计算psize时注意要把pool指针强转成`u_char*`，否则不同类型的指针相减可能会出现错误的结果。
将m强制转换成`ngx_pool_t`赋给new，使用new对首部的内存空间初始化。这个new只给d的变量进行赋值，别的没有赋值是因为不需要，nginx的策略是完全不在内存中存放不需要的变量，这可以说对内存利用极为"吝啬"了。这就是为什么在计算d.last的时候m加上的是`sizeof(ngx_data_t)`而不是`sizeof(ngx_pool_t)`。
最后对于pool链表上的节点进行遍历并累加d.failed。failed这个值之前没有提及，代表的是从该节点获取可用内存失败的次数。由于block函数只会在所有节点可提供内存不够时调用，如果失败次数超过四次表明该节点所剩余的可用内存较少，需要更新current指针。
初看current可能会认为这个指针是多余的，不是有一个全局的pool指针指向初次分配的pool节点吗，为什么不修改pool节点将其指向d.next呢？这是因为别的节点不存在除了`ngx_pool_data_t`以外的数据，只有通过`ngx_create_pool`创建的主节点存在其余的变量，也就是说主节点pool只有一个，作用就是索引其他参数。current指针指向当前提供内存的pool。

![ngx_pool_t](/img/pointer/ngx_pool_block.jpg)

## 总结

篇幅差不多了先分析到此，本文主要介绍了nginx中内存池的结构以及内存池中主要分配内存的函数，将其概括三点为：

1. 从nginx的内存池代码当中我们学习了指针的底层使用技巧，指针不仅是一个变量的代理工具，而且还是处理一整块内存分配的利器。
2. nginx的内存池整体是一个链表，由全局的pool指向主节点，并通过d.next形成一个单向链表。每个链表中的节点表示一个提供内存的内存池，d.last指向可用内存的起始地址，d.end指向可用内存的结束地址。
3. nginx的内存分配策略分为大内存和小内存，大内存会插入large链表中；小内存会直接获取current中可用内存，可用内存不够则遍历d.next，如果所有的节点剩余内存都不够则会创建新的pool节点。
