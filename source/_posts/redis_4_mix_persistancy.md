---
title: 'Redis 4.0 新功能：混合持久化'
categories: 开源项目
tags: [Redis]
date: 2017-07-31
---

Redis 的读写都是在内存中进行的，持久化数据只是作为磁盘备份，当实例重启或者机器断电的时候可以从磁盘加载数据到内存中。
目前 Redis 支持的持久化方式有两种：RDB 和 AOF。

#### RDB
RDB 是将某一时刻 Redis 中保存的数据全部写到磁盘中，在这个时刻之后写入的数据就会丢失掉。

#### AOF
AOF 是将 Redis 的写入命令持久化到磁盘中，根据策略的不同，持久化的时机也不同：
- appendfsync = always 每条写入都会刷盘, 最多只会丢失当前正在写入的命令
- appendfsync = everysec 每秒刷一次盘, 最多丢失一秒的数据
- appendfsync = no 不显式刷盘，由操作系统来决定何时刷盘 (linux 貌似大部分默认是 30s)，可能会丢失刷盘之前的写入数据

#### 两者对比
进行 RDB 持久化以后，由于只保存数据，所以持久化过程比较快，持久化文件比较小，重新载入速度快，缺点是会丢失备份以后写入的数据；而 AOF 需要保存所有的写入命令，丢失数据量比较小，缺点是当频繁写入，或开机时间很长以后，持久化文件比较大，重新载入速度慢。

#### 混合持久化
Redis 4.0 开始支持混合持久化，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头，其中 RDB 格式的内容用于记录已有的数据， 而 AOF 格式的内存则用于记录最近发生了变化的数据， 这样 Redis 就可以同时兼有 RDB 持久化和 AOF 持久化的优点 —— 既能够快速地生成重写文件，也能够在出现问题时， 快速地载入数据。
AOF 重写机制：由于 AOF 是通过不断追加写命令来记录数据库状态，所以服务器执行比较久之后，AOF 文件中的内容会越来越多，磁盘占有量越来越大，同时也是使通过过aof文件还原数据库的需要的时间也变得很久。所以就需要通过读取服务器当前的数据库状态来重写新的 AOF 文件。而混合持久化就是在重写的文件中将 RDB 内容写入。
具体实现如下：
```c
// aof_use_rdb_preamble = 1 表示打开混合存储模式
if (server.aof_use_rdb_preamble) {
    int error;
    // aof 文件前面部分就是直接写入 rdb 文件
    if (rdbSaveRio(&aof,&error,RDB_SAVE_AOF_PREAMBLE,NULL) == C_ERR) {
        errno = error;
        goto werr;
    }
} else {
    // 如果是关闭混合存储和之前一样，保持 aof 格式
    if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
}
// 后续的逻辑与之前一样，处理父进程中写入缓冲区的数据
```

重写期间，主进程继续处理命令，对数据库状态进行修改，这样使得当前的数据库状态与重写的 AOF 文件所保存的数据库状态不一致。因此，Redis 设置了 AOF 重写缓冲区，在创建子进程后，主进程每执行一个写命令都会写到重写缓冲区。在子进程完成重写后，主进程会将 AOF 重写缓冲区的数据写入到重写的 AOF 文件，保证数据状态的一致。
因此，可以理解为 RDB 格式的数据是重写开始那个时刻的数据库状态，而 AOF 格式的数据是在重写期间，主进程中的 AOF 重写缓冲区中的数据。

#### 混合加载
开启混合存储模式后 AOF 文件加载的流程如下（判断开头是不是 RDB 格式是看初始的五个字符是不是 REDIS）：
- AOF 文件开头是 RDB 的格式, 先加载 RDB 内容再加载剩余的 AOF
- AOF 文件开头不是 RDB 的格式，直接以 AOF 格式加载整个文件

```c
char sig[5]; /* "REDIS" */
if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
    // 前部分内容不是 rdb 格式，不是混合持久化的方式
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
} else {
    rio rdb;

    serverLog(LL_NOTICE,"Reading RDB preamble from AOF file...");
    if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    rioInitWithFile(&rdb,fp);
    // 前面部分是 rdb 格式说明是混合持久化，先加载 rdb 后面逻辑再加载 aof
    if (rdbLoadRio(&rdb,NULL) != C_OK) {
        serverLog(LL_WARNING,"Error reading the RDB preamble of the AOF file, AOF loading aborted");
        goto readerr;
    } else {
        serverLog(LL_NOTICE,"Reading the remaining AOF tail...");
    }
}
...
// 加载 aof 格式的数据
```
