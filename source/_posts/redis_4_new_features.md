---
title: 'Redis 4.0 新功能'
categories: 开源项目
tags: [Redis]
date: 2017-07-29
---

### 模块系统
Redis 4.0 发生的最大的变化就是加入模块系统，这个系统可以让用户通过自己编写的代码来拓展和实现 Redis 本身并不具备的功能。
关于加入模块系统的动机，具体可以参考 antirez 写的博客：[Redis Loadable Modules System](http://antirez.com/news/106)
从使用的角度来讲，主要需要知道如何加载模块，以及有哪一些模块可用：
- 静态加载方式：loadmodule /path/to/mymodule.so
- 动态加载方式：MODULE LOAD /path/to/mymodule.so
- 已经编写好的一些模块：https://redis.io/modules
至于如何编写模块等更多细节的内容具体可以参考：https://redis.io/topics/modules-intro

### PSYNC 优化
新版本的 PSYNC 命令解决了如下两个场景需要全量复制的问题：
- 如果一个从服务器在 FAILOVER 之后成为了新的主节点， 那么其他从节点在复制这个新主的时候就必须进行全量复制
- 一个从服务器如果重启了， 那么它就必须与主服务器重新进行全量复制
在 Redis 4.0 中，在其它条件（主要是复制偏移量满足条件）具备的情况下，允许进行增量复制，而不需要进行全量复制。

### 非阻塞删除
在 Redis 4.0 之前， 用户在使用 DEL 命令删除体积较大的键， 又或者在使用 FLUSHDB 和 FLUSHALL 删除包含大量键的数据库时， 都可能会造成服务器阻塞。
为了解决以上问题， Redis 4.0 新添加了 UNLINK 命令， 这个命令是 DEL 命令的异步版本， 它可以将删除指定键的操作放在后台线程里面执行， 从而尽可能地避免服务器阻塞：
```bash
redis> UNLINK fruits
(integer) 1
```

此外， Redis 4.0 中的 FLUSHDB 和 FLUSHALL 这两个命令都新添加了 ASYNC 选项， 带有这个选项的数据库删除操作将在后台线程进行：
```bash
redis> FLUSHDB ASYNC
OK

redis> FLUSHALL ASYNC
OK
```

### 混合持久化
Redis 4.0 新增了 RDB-AOF 混合持久化格式， 这是一个可选的功能， 在开启了这个功能之后， AOF 重写产生的文件将同时包含 RDB 格式的内容和 AOF 格式的内容， 其中 RDB 格式的内容用于记录已有的数据， 而 AOF 格式的内存则用于记录最近发生了变化的数据， 这样 Redis 就可以同时兼有 RDB 持久化和 AOF 持久化的优点 —— 既能够快速地生成重写文件， 也能够在出现问题时， 快速地载入数据。这个功能可以通过 aof-use-rdb-preamble 选项进行开启。

### LFU 数据淘汰策略
Redis 4.0 对数据淘汰策略进行了两项调整：
- 增加了 LFU（Last Frequently Used）淘汰策略
- 在使用多个 Redis 作为缓存的场景下，使用一个采样池进行淘汰，而非一个数据库一个采样池，避免了某些数据库中都是新数据，而另外一些数据库中都是老数据的情况

### 交换数据库
Redis 4.0 对数据库命令的另外一个修改是新增了 SWAPDB 命令， 这个命令可以对指定的两个数据库进行互换： 比如说， 通过执行命令 SWAPDB 0 1 ， 我们可以将原来的数据库 0 变成数据库 1 ， 而原来的数据库 1 则变成数据库 0 。
内存命令
新添加了一个 MEMORY 命令， 这个命令可以用于视察内存使用情况， 并进行相应的内存管理操作：
```bash
redis> MEMORY HELP
1) "MEMORY USAGE <key> [SAMPLES <count>] - Estimate memory usage of key"
2) "MEMORY STATS                         - Show memory usage details"
3) "MEMORY PURGE                         - Ask the allocator to release memory"
4) "MEMORY MALLOC-STATS                  - Show allocator internal stats"
```
- MEMORYUSAGE 命令可以估算储存给定键所需的内存
- MEMORYSTATS 命令可以查看 Redis 当前的内存使用情况
- MEMORYPURGE 命令可以要求分配器释放更多内存
- MEMORYMALLOC-STATS 命令可以展示分配器内部状态

### 兼容 NAT 和 Docker
Redis 4.0 将兼容 NAT 和 Docker ， 在这种模式下，需要显式指定 Redis 的公网 ip 和 端口。配置例如：
```bash
# cluster-announce-ip 10.1.1.5
# cluster-announce-port 6379
# cluster-announce-bus-port 6380
```
