---
title: 'Redis 4.0 新功能：非阻塞删除'
categories: 开源项目
tags: [Redis]
date: 2017-07-30
---

### 现存的问题
Redis 基于单线程模型，因此一些耗时比较严重（通常是遍历整张表）的操作会阻塞其它命令的执行。对于某些命令而言（例如 BGSAVE），Redis 会通过新开线程，或者后台操作来避免这种阻塞。而类似 KEYS / FLUSHALL / FLUSHDB 等命令，通常在生产环境下我们也会直接禁用。但是像 DEL / LRANGE / HGETALL 这些可能导致阻塞的命令经常被忽视，而这些命令在 value 比较大的时候跟 KEYS 这些并没有本质区别（例如 value 是链表，集合或者字典，同样要遍历删除）。

### 优化方式
在 Redis 4.0 中，对 DEL / FLUSHALL / FLUSHDB 这些命令进行了优化：FLUSHALL / FLUSHDB 加了一个 ASYNC 参数，同时新增 UNLINK 来表示异步化的删除命令（DEL 命令是支持不定参数，如果加个 ASYNC 参数没办法判断到底这个是 key 还是异步删除的选项，所以索性增加了一个新命令）。

### UNLINK 具体实现
我们可看到 UNLINK 命令会调用 dbAsyncDelete 来实现异步调用。在 dbAsyncDelete 中，并不是所有的键值对的删除都是异步的，而是对于 value 进行评估，如果超过阈值，则放入异步队列中进行处理。
```C
void unlinkCommand(client *c) {
    // lazy 参数设置 1，表示异步删除
    delGenericCommand(c,1);
}

void delGenericCommand(client *c, int lazy) {
    int numdel = 0, j;

    for (j = 1; j < c->argc; j++) {
        expireIfNeeded(c->db,c->argv[j]);
        // 如果是异步删除调用 dbAsyncDelete
        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) :
                              dbSyncDelete(c->db,c->argv[j]);
        ...
    }
    addReplyLongLong(c,numdel);
}

#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    // 先把 key 从过期时间字典里面删除
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    // 把 kv 从字典里面摘除但不是删除 value，后续命令就查询不到
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        // 不是所有的 key 都会走异步化删除，如果 value 比较小会直接删除
        // 如果 value 是字典/链表/集合且不能是压缩的返回对应的元素数目，其他都返回 1
        size_t free_effort = lazyfreeGetFreeEffort(val);

        // 只有计算出来的 free_effort 大于 LAZYFREE_THRESHOLD(64) 才会进入异步处理
        if (free_effort > LAZYFREE_THRESHOLD) {
            atomicIncr(lazyfree_objects,1,lazyfree_objects_mutex);
            // 创建 BIO_LAZY_FREE 任务，放到异步队列
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }

    if (de) {// 如果 key 存在，释放字典里面结构
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```

### FLUSHALL / FLUSHDB 具体实现
Redis 会先检查这两个命令是否有带 ASYNC ，接着在 emptyDb 判断是异步清数据，如果是异步清除则会调用 emptyDbAsync。
```c
int getFlushCommandFlags(client *c, int *flags) {
    if (c->argc > 1) {
        // 判断第二个参数是否为 async
        if (c->argc > 2 || strcasecmp(c->argv[1]->ptr,"async")) {
            addReply(c,shared.syntaxerr);
            return C_ERR;
        }
        *flags = EMPTYDB_ASYNC;
    } else {
        *flags = EMPTYDB_NO_FLAGS;
    }
    return C_OK;
}

long long emptyDb(int dbnum, int flags, void(callback)(void*) ) {
 if (async) {
 emptyDbAsnyc(&server.db[j]);
 } else {
 dictEmpty(server.db[j].dict, callback);
 dictEmpty(server.db[j].expires, callback);
   }
}

void emptyDbAsync(redisDb *db) {
    // 保留老的数据库指针并重新创建新的数据库
    dict *oldht1 = db->dict, *oldht2 = db->expires;
    db->dict = dictCreate(&dbDictType,NULL);
    db->expires = dictCreate(&keyptrDictType,NULL);
    atomicIncr(lazyfree_objects,dictSize(oldht1),
        lazyfree_objects_mutex);
    // 把要清空的 db 作为一个 job 添加到后台的处理队列
    bioCreateBackgroundJob(BIO_LAZY_FREE,NULL,oldht1,oldht2);
}
```
