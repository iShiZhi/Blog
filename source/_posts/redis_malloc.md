---
title: 'Redis内存管理机制'
categories: 开源项目
tags: [Redis]
date: 2016-12-5
---

在阅读各类基本数据结构的时候，看到分配空间或调整空间时，使用的是形如 zmalloc 的函数。由于它的用法与 malloc 无异，所以一直把它等同于 malloc 进行阅读，直到看到 object.c 最后新增加的内存数据统计时，才注意到一些内存统计相关的功能，于是阅读了一下 Redis 内存管理相关的代码。

### Redis 的内存管理策略

Redis 在进行内存空间分配的时候，会在分配的空间前面增加一个额外的空间，用于保存分配的空间的大小。同时，使用一个全局变量 used_memory 来记录 Redis 总共申请分配的空间大小。

### 第三方的 malloc 库

Redis 支持使用第三方的 malloc 库（tcmalloc、 jemalloc 以及苹果平台的 malloc.h）来管理内存：

- tcmalloc：tcmalloc 是 google perftool 的一部分，与一般的内存池不同，它直接与 OS 打交道，内存闲置时 OS 会进行回收 (STL 内存池就不回收)，同时使用 TLS (Thread local storage) 管理内存池，避免一个线程内分配内存都要同步。
- jemalloc：jemalloc 与 tcmalloc相似，作者 Jason Evans 是 Free BSD 开发人员，性能与使用率与 tcmalloc 不相伯仲。tcmalloc 更方便与 google perftool 集成，进行性能评测。

对于这些第三方库而言，有两个比较重要的特点：

- 包括苹果平台的 malloc.h，这些第三方库与 STL 标准库可以通过调用 malloc_size(p) 直接获得指针 p 在分配时获得的空间大小。
- 在分配小空间时，这些第三方库可能会多分配一些内存空间以供使用，例如对于 tcmalloc 而言，收到 833 字节到 1024 字节的内存分配请求，都会分配一个 1024 字节大小的内存块，这样会产生内存空间碎片。

关于这些第三方库的实现原理，可以参考：
http://www.360doc.com/content/13/0915/09/8363527_314549128.shtml。

在使用第三方库的时候，直接用这些库中的函数替换掉预先定义的函数，特别注意到，这些第三方库都实现了获取空间大小的方法，统一重新定义为 zmalloc_size(p)，同时使用 HAVE_MALLOC_SIZE 标记是否由 Redis 自己管理分配的空间大小（为 1 表示不由自己管理）：

```C
#if defined(USE_TCMALLOC)
#define ZMALLOC_LIB ("tcmalloc-" __xstr(TC_VERSION_MAJOR) "." __xstr(TC_VERSION_MINOR))
#include <google/tcmalloc.h>
#if (TC_VERSION_MAJOR == 1 && TC_VERSION_MINOR >= 6) || (TC_VERSION_MAJOR > 1)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) tc_malloc_size(p)
#else
#error "Newer version of tcmalloc required"
#endif
#elif defined(USE_JEMALLOC)
#define ZMALLOC_LIB ("jemalloc-" __xstr(JEMALLOC_VERSION_MAJOR) "." __xstr(JEMALLOC_VERSION_MINOR) "." __xstr(JEMALLOC_VERSION_BUGFIX))
#include <jemalloc/jemalloc.h>
#if (JEMALLOC_VERSION_MAJOR == 2 && JEMALLOC_VERSION_MINOR >= 1) || (JEMALLOC_VERSION_MAJOR > 2)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) je_malloc_usable_size(p)
#else
#error "Newer version of jemalloc required"
#endif
#elif defined(__APPLE__)
#include <malloc/malloc.h>
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) malloc_size(p)
#endif
#if defined(USE_TCMALLOC)
#define malloc(size) tc_malloc(size)
#define calloc(count,size) tc_calloc(count,size)
#define realloc(ptr,size) tc_realloc(ptr,size)
#define free(ptr) tc_free(ptr)
#elif defined(USE_JEMALLOC)
#define malloc(size) je_malloc(size)
#define calloc(count,size) je_calloc(count,size)
#define realloc(ptr,size) je_realloc(ptr,size)
#define free(ptr) je_free(ptr)
#endif
```

### 非第三方的内存管理

在使用标准库提供的内存操作函数时，Redis 会额外多申请 PREFIX_SIZE 定义的大小的空间，用于存储申请空间的大小。同时，每次申请或者更新空间大小时，需要同时更新 used_memory 的值，以 zmalloc 为例：

```C
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        atomicIncr(used_memory,__n,used_memory_mutex); \
    } else { \
        used_memory += _n; \
    } \
} while(0)
void *zmalloc(size_t size) {
    void *ptr = malloc(size+PREFIX_SIZE); // 申请一个指定大小加上 header 的空间
    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr)); // 计算空间大小
    return ptr;
#else
    *((size_t*)ptr) = size; // 自己维护空间的大小
    update_zmalloc_stat_alloc(size+PREFIX_SIZE); // 计算空间的大小
    return (char*)ptr+PREFIX_SIZE; // 返回可以使用的空间的指针
#endif
}
```

### 统计使用的内存数

Redis 在统计使用的内存数的时候，按照如下的步骤进行尝试：

- 如果定义了 PROC_FS，则从 /proc/[getpid()]/stat 中读取
- 如果定义了 TASK_INFO，从该进程 id 对应的 task_info_t 结构中读取 resident_size 变量
- 直接返回 used_memory

由于第三方的库在分配内存空间的时候，存在空间碎片，因此，通过操作系统底层的调用函数统计出的内存占用量的数值更为准确，使用这个数值与 used_memory 的比例可以估计内存碎片的情况，而 Redis 也是这么做的：

```C
float zmalloc_get_fragmentation_ratio(size_t rss) {
    return (float)rss/zmalloc_used_memory();
}
```
