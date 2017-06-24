---
title: 'Redis数据结构: 对象'
categories: 开源项目
tags: [Redis]
date: 2016-12-1
---

Redis 主要的基本数据结构有 SDS，双端链表，整数集合，哈希表，跳跃表，压缩表等等。Redis 并没有直接使用这些基本的数据结构，而是把它们封装成对象来进行使用，对象的类型有字符串对象（OBJ_STRING）、列表对象（OBJ_LIST）、哈希对象（OBJ_HASH）、集合对象（OBJ_SET）和有序集合对象（OBJ_ZSET）。通过这五种不同的对象类型，Redis 可以在不同的场景下使用不同的基本数据结构，从而优化空间的利用率和操作的时间复杂度。

### Redis 对象结构

在 server.h 中定义了 redisObject 的结构：

```C
typedef struct redisObject {
    unsigned type:4;              // 类型
    unsigned encoding:4;          // 编码
    unsigned lru:REDIS_LRU_BITS;  // 对象最后一次被访问的时间
    int refcount;                 // 引用计数
    void *ptr;                    // 指向实际值的指针
} robj;
```

#### 类型、编码、指向实际值的指针

不同的类型决定了该对象是上述五种对象中的哪一种，而编码则决定了使用了何种底层的基本数据结构，具体的对应关系是：

| 类型 | 编码 | 对象 |
|-----|------|-----|
| OBJ_STRING | OBJ_ENCODING_INT | 使用整数值实现的字符串对象 |
| OBJ_STRING | OBJ_ENCODING_EMBSTR | 使用 embstr 编码的 SDS 实现的字符串对象 |
| OBJ_STRING | OBJ_ENCODING_RAW | 使用 SDS 实现的字符串对象 |
| OBJ_LIST | OBJ_ENCODING_QUICKLIST | 使用快速列表实现的列表对象 |
| OBJ_HASH | OBJ_ENCODING_ZIPLIST | 使用压缩列表实现的哈希对象 |
| OBJ_HASH | OBJ_ENCODING_HT | 使用哈希表实现的哈希对象 |
| OBJ_SET | OBJ_ENCODING_INTSET | 使用整数集合实现的集合对象 |
| OBJ_SET | OBJ_ENCODING_HT | 使用哈希表实现的集合对象 |
| OBJ_ZSET | OBJ_ENCODING_ZIPLIST | 使用压缩列表实现的有序集合对象 |
| OBJ_ZSET | OBJ_ENCODING_SKIPLIST | 使用快速列表实现的有序集合对象 |

具体的组合之间的转换关系和对象的细节后面再进行叙述。

#### 引用计数

因为 C 语言本身没有实现垃圾回收机制，因此 Redis 自己实现了一套基于引用计数器的回收机制。而 refcount 则用于记录当前对象被程序引用的次数：

- 对象创建时，refcount 被设置为 1
- 对象被新的程序引用时，refcount 增加 1
- 对象不再被程序引用时，refcount 减去 1
- refcount = 0 时，销毁对象，回收空间

同时，值得一提的是，Redis 在启动的时候会生成 0 - 10000 对应的字符串对象，在使用这些整数值时，所有的字符串指针会指向这些预先生成的字符串对象。这样能够大量节省这些常用字符串的存储空间。那么为什么只对数值类型的字符串进行共享呢？因为程序判定是创建新的字符串对象，还是使用共享对象时，需要对存储的内容进行比对，整数型字符串对象的比对时间为 O(1)，而普通的字符串则需要耗费 O(N)。而由于这样的比对操作在每次创建字符串对象或修改的时候都需要进行，十分频繁，会严重影响 Redis 的性能。

#### 对象最后一次被访问时间

这个属性记录了对象的空转时长，使用OBJECT IDLETIME可以查看对象的空转时长。注意，除了该命令以外的其它命令都会更新这个属性。
当 Redis 的 maxmemory 选项被设置以后，Redis 占用的空间到达 maxmemory 设置的值时，Redis 就会开始清理空转时间较长的对象来获取可用的空间。

### 字符串对象

embstr 编码的字符串和普通 SDS 字符串的区别是：对象结构和字符串结构（字符串结构仍是 SDS）是否存储在连续的空间中。它的好处是：

- 只需要一次分配空间和释放空间的操作，而普通 SDS 字符串实现的字符串对象需要两次
- 由于字符串对象和字符串内容被分配在连续的空间中，能更好地利用缓存

字符串对象在选择底层的数据结构实现时，遵循如下的原则：

- 可以用 long 类型保存的字符串用整数值保存
- 长度小于 44 字节的字符串用 embstr 编码的字符串对象保存
- 无法用 long 类型保存的，且转化为字符串长度大于 44 字节的数值类型，或长度大于 44 的字符串则用普通的 SDS 对应的字符串类型保存

为什么是 44 字节呢？ 因为 EMBSTR 的对象和字符串都保存在一个连续的内存空间中，包括了

- redisObject 结构 type、encoding 和 lru 共占用 4 个字节，refcount 占用 4 个字节，void* 占用不超过 8 个字节
- sdshdr8 header 占用 3 个字节
- 字符串占用 44 个字节
- 结尾的空字符占用 1 个字节

总计 4 + 4 + 8 + 3 + 44 + 1 = 64，所以可以使用 64 字节的连续空间来进行存储。

```C
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
### 列表对象

在最新的 unstable 分值上，Redis 实现的列表对象的底层实现就是快速列表，而之前的版本中可以是压缩列表或者双端列表。之前用于判定压缩列表和双端链表的两个配置 list-max-ziplist-value 和 list-max-ziplist-entries 不再使用。
在前面对快速列表的介绍中可以知道，快速列表是一个节点为压缩列表的双端链表，因此，双端链表就是节点为单个元素的快速列表，而压缩列表就是只有一个节点的快速列表，使用一个单独的参数 list-max-ziplist-size 就可以进行控制。

### 哈希对象

如果用压缩列表作为哈希对象的底层实现，那么会将键值对保存在压缩列表的相邻两个节点中，哈希键在前，哈希值在后，所有的键值对按照插入的顺序依次放入压缩列表中。
当以下两个条件同时满足时，使用压缩列表作为底层实现：

- 保存的所有的字符串对象的长度都不大于 hash-max-ziplist-value 设定的值（默认为 64 字节）
- 保存的字符串对象的数量不大于 hash-max-ziplist-entries 设定的值（默认为 512）

虽然用压缩列表实现的哈希对象在进行键值查询的时候需要通过遍历列表的方式进行查找，但是由于限制了数量，所以查找操作依旧是常数级的时间复杂度。而通过压缩列表的方式，可以节省存储空间。这里使用了用时间换空间的思想，即在对操作时间影响不大的情况下，尽量压缩存储空间，这个思想就是引入压缩列表最主要的动机，在有序集合对象的实现中也是同样的考虑。

```C
void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {
    int i;
    if (o->encoding != OBJ_ENCODING_ZIPLIST) return;
    for (i = start; i <= end; i++) {
        // 这边只检查字符串的长度，时间复杂度为 O(1)
        if (sdsEncodedObject(argv[i]) &&
            sdslen(argv[i]->ptr) > server.hash_max_ziplist_value)
        {
            hashTypeConvert(o, OBJ_ENCODING_HT);
            break;
        }
    }
}
int hashTypeSet(robj *o, sds field, sds value, int flags) {
    int update = 0;
    if (o->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;
        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            fptr = ziplistFind(fptr, (unsigned char*)field, sdslen(field), 1);
            if (fptr != NULL) {
                /* Grab pointer to the value (fptr points to the field)
                 *
                 * 如果能找到对应的键，就更新对应的值
                 */
                vptr = ziplistNext(zl, fptr);
                serverAssert(vptr != NULL);
                update = 1;
                /* Delete value */
                zl = ziplistDelete(zl, &vptr);
                /* Insert new value */
                zl = ziplistInsert(zl, vptr, (unsigned char*)value,
                        sdslen(value));
            }
        }
        // 如果不是更新，则将对应的键值对插入表尾
        if (!update) {
            /* Push new field/value pair onto the tail of the ziplist */
            zl = ziplistPush(zl, (unsigned char*)field, sdslen(field),
                    ZIPLIST_TAIL);
            zl = ziplistPush(zl, (unsigned char*)value, sdslen(value),
                    ZIPLIST_TAIL);
        }
        o->ptr = zl;
        /* Check if the ziplist needs to be converted to a hash table
         *
         * 如果节点数超过 hash_max_ziplist_entries 设置的值，则需要转换成 hashtable，如果有必要的话
         */
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        // 查找键，如果存在，则进行更新，否则进行创建
        dictEntry *de = dictFind(o->ptr,field);
        if (de) {
            sdsfree(dictGetVal(de));
            if (flags & HASH_SET_TAKE_VALUE) {
                dictGetVal(de) = value;
                value = NULL;
            } else {
                dictGetVal(de) = sdsdup(value);
            }
            update = 1;
        } else {
            sds f,v;
            if (flags & HASH_SET_TAKE_FIELD) {
                f = field;
                field = NULL;
            } else {
                f = sdsdup(field);
            }
            if (flags & HASH_SET_TAKE_VALUE) {
                v = value;
                value = NULL;
            } else {
                v = sdsdup(value);
            }
            dictAdd(o->ptr,f,v);
        }
    } else {
        serverPanic("Unknown hash encoding");
    }
    /* Free SDS strings we did not referenced elsewhere if the flags
     * want this function to be responsible.
     *
     * 根据 flags 的设置选择性释放键值
     */
    if (flags & HASH_SET_TAKE_FIELD && field) sdsfree(field);
    if (flags & HASH_SET_TAKE_VALUE && value) sdsfree(value);
    return update;
}
```

### 集合对象

当以下两个条件同时满足时，使用整数集合作为底层实现：

1. 保存的所有元素都是整数
2. 保存的数量不超过 set-max-intset-entries 设置的值（默认为 512）

在使用哈希表作为底层实现时，所有的集合元素都作为哈希键保存，而对应的哈希值则设置为 NULL。

```C
int setTypeAdd(robj *subject, sds value) {
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) {
        dict *ht = subject->ptr;
        // 如果已经存在，dictAddRaw 返回的是 NULL，跳过处理
        // 否则将 value 作为键存入，值设为 NULL
        dictEntry *de = dictAddRaw(ht,value,NULL);
        if (de) {
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        // 判断是否是整数值，如果是，则插入，否则的话，需要把 intset 换成 hashtable
        if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);
            if (success) {
                /* Convert to regular set when the intset contains
                 * too many entries.
                 *
                 * 如果节点数过多，则将 intset 转成 hashtable
                 */
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)
                    setTypeConvert(subject,OBJ_ENCODING_HT);
                return 1;
            }
        } else {
            /* Failed to get integer from object, convert to regular set. */
            setTypeConvert(subject,OBJ_ENCODING_HT);
            /* The set *was* an intset and this value is not integer
             * encodable, so dictAdd should always work. */
            serverAssert(dictAdd(subject->ptr,sdsdup(value),NULL) == DICT_OK);
            return 1;
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}
```

### 有序集合对象

有序集合对象的底层实现可以是压缩列表，也可以是跳跃表。当以下两个条件同时满足时，使用压缩列表作为底层实现：

- 保存的所有的字符串对象的长度都不大于 zset-max-ziplist-value 设定的值（默认为 64 字节）
- 保存的字符串对象的数量不大于 zset-max-ziplist-entries 设定的值（默认为 512）
```C
void zsetConvertToZiplistIfNeeded(robj *zobj, size_t maxelelen) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) return;
    zset *zset = zobj->ptr;
    if (zset->zsl->length <= server.zset_max_ziplist_entries &&
        maxelelen <= server.zset_max_ziplist_value)
            zsetConvert(zobj,OBJ_ENCODING_ZIPLIST);
}
```
当底层实现是压缩列表时，用连续的两个压缩列表的节点来保存有序集合的元素和分值，元素在前，分值在后。同时，所有的元素在压缩列表中，按照分值的大小从小到大保存在压缩列表中，因此需要找到合适的位置将新的元素插入。这样的插入操作可能会引发链式更新后续节点的操作，但是由于压缩列表中的节点数量是有限的，因此时间复杂度并不会变得特别糟糕。
当底层实现是跳跃表时，Redis 封装了一个新的数据结构 zset 来作为实现，包含了一个哈希表和一个跳跃表：

- 哈希表的键为成员，值为分值，这样能用 O(1) 的时间复杂度取出对应的分值
- 而跳跃表按照分值排序成员，用于支持平均复杂度为 O(log N) 的按分值定位成员操作，以及范围操作

通过空间换时间的方式，可以使得有序集合对象的各项操作的时间复杂度都维持在一个较低的水平上。

```C
typedef struct zset {
    // 字典，键为成员，值为分值，用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;
    // 跳跃表，按分值排序成员，用于支持平均复杂度为 O(log N) 的按分值定位成员操作，以及范围操作
    zskiplist *zsl;
} zset;
```
注意，在给 zset 添加元素的时候，会进行 ziplist 到 skiplist 的转换条件判断，但是删除元素的时候并不会进行反向的判断。Redis 只有在进行 rdb 文件处理时，才会进行反向的判断。

```C
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore) {
    /* Turn options into simple to check vars. */
    int incr = (*flags & ZADD_INCR) != 0;
    int nx = (*flags & ZADD_NX) != 0;
    int xx = (*flags & ZADD_XX) != 0;
    *flags = 0; /* We'll return our response flags. */
    double curscore;
    /* NaN as input is an error regardless of all the other parameters.
     *
     * 如果分值非数值型，直接返回
     */
    if (isnan(score)) {
        *flags = ZADD_NAN;
        return 0;
    }
    /* Update the sorted set according to its encoding. */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *eptr;
        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            /* NX? Return, same element already exists.
             *
             * ZADD_NX 被设置，所以直接不操作，返回
             */
            if (nx) {
                *flags |= ZADD_NOP;
                return 1;
            }
            /* Prepare the score for the increment if needed.
             *
             * 计算新的分值
             */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }
            /* Remove and re-insert when score changed.
             *
             * 执行到这里，说明 ZADD_NX 没有被设置，所以直接替换原有的元素
             */
            if (score != curscore) {
                zobj->ptr = zzlDelete(zobj->ptr,eptr);
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                *flags |= ZADD_UPDATED;
            }
            return 1;
        } else if (!xx) {
            /* Optimize: check if the element is too large or the list
             * becomes too long *before* executing zzlInsert.
             *
             * 在这个分支里，元素原先不存在，这个时候需要插入新元素
             * 这个操作可能导致元素个数过多，或者元素大小过大，使得 zset 的编码由 ziplist 变为 skiplist
             */
            zobj->ptr = zzlInsert(zobj->ptr,ele,score);
            if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries)
                zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
            if (sdslen(ele) > server.zset_max_ziplist_value)
                zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
            if (newscore) *newscore = score;
            *flags |= ZADD_ADDED;
            return 1;
        } else {
            // ZADD_XX 被设置，所以直接不操作，返回
            *flags |= ZADD_NOP;
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            /* NX? Return, same element already exists.
             *
             * ZADD_NX 被设置，所以直接不操作，返回
             */
            if (nx) {
                *flags |= ZADD_NOP;
                return 1;
            }
            curscore = *(double*)dictGetVal(de);
            /* Prepare the score for the increment if needed.
             *
             * 计算新的分值
             */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }
            /* Remove and re-insert when score changes.
             *
             * 执行到这里，说明 ZADD_NX 没有被设置，所以直接替换原有的元素
             */
            if (score != curscore) {
                zskiplistNode *node;
                serverAssert(zslDelete(zs->zsl,curscore,ele,&node));
                znode = zslInsert(zs->zsl,score,node->ele);
                /* We reused the node->ele SDS string, free the node now
                 * since zslInsert created a new one.
                 *
                 * 在 zslInsert 中，为元素新建了一个 SDS，所以这边可以将老的节点释放掉
                 */
                node->ele = NULL;
                zslFreeNode(node);
                /* Note that we did not removed the original element from
                 * the hash table representing the sorted set, so we just
                 * update the score. */
                dictGetVal(de) = &znode->score; /* Update score ptr. */
                *flags |= ZADD_UPDATED;
            }
            return 1;
        } else if (!xx) {
            /*
             * 这里直接添加新的元素，注意，这里不做从 skiplist 到 ziplist 的判断
             * 因为从 ziplist 转换成 skiplist，说明至少一个条件不满足，而这边并不会使不满足的条件重新被满足
             */
            ele = sdsdup(ele);
            znode = zslInsert(zs->zsl,score,ele);
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            *flags |= ZADD_ADDED;
            if (newscore) *newscore = score;
            return 1;
        } else {
            // ZADD_XX 被设置，所以直接不操作，返回
            *flags |= ZADD_NOP;
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}
int zsetDel(robj *zobj, sds ele) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *eptr;
        if ((eptr = zzlFind(zobj->ptr,ele,NULL)) != NULL) {
            zobj->ptr = zzlDelete(zobj->ptr,eptr);
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        dictEntry *de;
        double score;
        // 这边没有从字典中立刻删除是因为还要获取分值
        de = dictUnlink(zs->dict,ele);
        if (de != NULL) {
            /* Get the score in order to delete from the skiplist later. */
            score = *(double*)dictGetVal(de);
            /* Delete from the hash table and later from the skiplist.
             * Note that the order is important: deleting from the skiplist
             * actually releases the SDS string representing the element,
             * which is shared between the skiplist and the hash table, so
             * we need to delete from the skiplist as the final step.
             *
             * 注意：一定要先从 dict 中删除以后，再从 skiplist 中删除
             * 因为在 skiplist 中删除节点，会将 ele 也进行释放
             */
            dictFreeUnlinkedEntry(zs->dict,de);
            /* Delete from skiplist.
             *
             * 从 skiplist 中删除对应的元素
             */
            int retval = zslDelete(zs->zsl,score,ele,NULL);
            serverAssert(retval);
            if (htNeedsResize(zs->dict)) dictResize(zs->dict);
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* No such element found. */
}
```
