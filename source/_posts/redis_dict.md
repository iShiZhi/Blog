---
title: 'Redis数据结构: 字典'
categories: 开源项目
tags: [Redis]
date: 2016-10-21
---

字典是一种常用的数据结构，在很多其它的高级编程语言里都有内置的实现方式，但是 C 语言没有，所以 Redis 自己实现了一个。Redis 使用哈希表作为字典的实现方式，代码主要在 dict.h 和 dict.c 中。

首先来看看哈希表的节点结构，这里面包含键值对和一个指向下个节点的指针：

```C
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```
通过这个结构能知道，Redis 的哈希解决冲突的方式是链表，将相同哈希值的键值对保存在一个单项列表中，而且为了性能的考虑，Redis 总是将最后插入链表的节点作为这个链表的表头，这样使得插入操作的时间复杂度为 O(1)，例如 dictRehash 函数中：

```C
de->next = d->ht[1].table[h];
d->ht[1].table[h] = de;
```
其次是哈希表的结构，里面包含了哈希表内容、长度、掩码值和已使用的空间，其中掩码值用于将哈希函数计算出的哈希值对应到表中的索引值，对于哈希函数的使用可以看后面的介绍：

```C
typedef struct dictht {
    dictEntry **table;       // 哈希表内容
    unsigned long size;      // 哈希表的长度
    unsigned long sizemask;  // 掩码值，永远等于 size - 1，用于将哈希值对应到索引值
    unsigned long used;      // 已使用的空间大小
} dictht;
```
接下来是字典的结构，在字典中包含了两个哈希表，自定义类型相关的函数和私有数据，以及 rehash 过程需要用到的索引和安全迭代器的数量，具体的 Rehash 过程在后面进行介绍：

```C
typedef struct dict {
    dictType *type;           // 类型特定函数
    void *privdata;           // 私有数据
    dictht ht[2];             // 哈希表
    int rehashidx;            // rehash 索引，当 rehash 不在进行时，值为 -1
    int iterators;            // 目前正在运行的安全迭代器的数量
} dict;
```
通过 dictType 的定义，可以看到 privdata 保存了类型特定函数的一些可选参数，在调用函数时传入。Redis 给不同的字典实现不同的类型函数以及定义不同的私有数据，从而实现字典的多态：

```C
typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```
在了解完基本的结构以后，我们再来看哈希表和字典的具体实现细节进行。

## 哈希算法

从上述的介绍中，我们可以知道 Redis 利用 dictType 定义的类型特定函数来实现多态，在这些类型函数中，hashFunction 负责将 key 进行转换，得到对应的哈希值。根据不同的字典，使用的哈希算法也不相同。得到哈希值以后，再通过与哈希表的掩码值进行与操作，就获得 key 对应的索引值，具体可以看 dictRehash 函数中计算索引值的操作：

```C
#define dictHashKey(d, key) (d)->type->hashFunction(key)
h = dictHashKey(d, de->key) & d->ht[1].sizemask;
```
Redis 提供了几种种哈希函数:

1. Thomas Wang 发明的 32 bits / 64 bits 的整数哈希函数
   在之前的版本中，使用这个哈希函数来对整数进行哈希操作，这个哈希函数通过加法和位移操作进行哈希值的计算，避免了乘法操作带来的性能损失，同时使结果具有均匀的分布，以及雪崩效应。在之前的版本中单独编写函数，对于整数进行哈希，而在最新的版本中，只作为 figureprint 生成的函数。对于这个哈希函数的细节，可以参考我之前翻译的 Thomas Wang 写的整数哈希函数。
2. Austin Appleby 发明的 MurmurHash2 算法
   murmur 是 multiply and rotate 的意思，因为算法的核心就是不断的乘和移位。MurmurHash 算法具有高运算性能，低碰撞率的特点，由 Austin Appleby 创建于2008年，最新的版本是 MurmurHash3，现已应用到Hadoop、libstdc++、nginx、libmemcached等开源系统。
   通过 murmur 函数获得的哈希值分布均匀，比如：murmur计算”abc”是1118836419，”abd”是413429783。而使用 Horner 算法，”abc”是96354， “abd”就比它多1（96355）。
3. DJB 算法的修改版
   DJB 算法同样是用于字符串类型的哈希函数，原理也非常简单，就是不断乘以 33 再加上对应的字符。该算法的执行效率和随机性都不错。在 Redis 中对该算法进行了简单的修改，使得它对大小写不敏感。

对于 MurmurHash2 和 DJB 两种字符串哈希函数的比较：

1. 有人做过一个[性能对比的实验](http://blog.csdn.net/wwwsq/article/details/4254123)，得出结论：从计算速度上来看，MurmurHash只适用于已知长度的、长度比较长的字符串。长度未知或者长度不超过10字节，都应该使用DJB。而在前面介绍 SDS 的时候，我们知道 Redis 的字符串的 strlen 操作的时间复杂度是 O(1)，因此在长度大于 10 字节时，使用 MurmurHash2 可以获得一个随机性更好的哈希值，性能也更优；
2. 另外，在代码的实现中，也将 DJB 和 MurmurHash2 算法进行了区别，DJB 的实现对字母的大小写不敏感，而 MurmurHash2 则是敏感的。

## Rehash

随着各类操作的不断执行，字典中的键值对的数量会不断地变化。当键值对的数量增加到一定数量以后，会使得碰撞冲突变得剧烈，影响哈希表的性能；而当键值对减少到一定程度以后，则会空闲大量的存储空间。因此，为了让哈希表的负载因子（键值对数量 / 哈希表大小）维持在一个合理的范围内，我们需要对哈希表进行适当的收缩和扩张。这个操作的基本步骤为：

1. 根据不同的操作（不同的大小）创建一个新的哈希表 ht[1]
2. 渐进式地进行哈希表节点的迁移
3. 完成迁移以后，将 ht[0] 的空间释放掉，将 ht[1] 替换 ht[0]

### 进行 Rehash 操作的条件

Redis 基于哈希表的性能和整体服务的性能考虑，将进行 Rehash 的情况分为两种：

1. 正常使用 Redis 时，当负载因子超过 1 时，就考虑进行 Rehash 操作；
2. Redis 利用后台线程进行持久化，这个时候为了最大程度利用 Copy On Write 机制带来的性能提升，会暂时停止 Rehash 操作，除非负载因子超过 dict_force_resize_ratio 的值

```C
static int _dictExpandIfNeeded(dict *d)
{
    if (dictIsRehashing(d)) return DICT_OK;
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

### 创建新的哈希表

扩张哈希表的操作是在 _dictExpandIfNeeded 函数里进行判断和扩张的，而收缩哈希表的操作则是通过调用 dictResize 函数进行。而这两个函数同时调用了 dictExpand 函数。具体如下：

```C
int dictExpand(dict *d, unsigned long size)
{
    dictht n; // 新哈希表
    unsigned long realsize = _dictNextPower(size);  // 根据 size 参数，计算哈希表的大小
    // 不能在字典正在 rehash 时进行
    // size 的值也不能小于 0 号哈希表的当前已使用节点
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;
    // 为哈希表分配空间，并将所有指针指向 NULL
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;
    // 如果此时 ht[0] 如果为空，则表示当前的操作是初始化哈希表
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }
    // 设置新的哈希表大小，开始进行迁移
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```
在这个函数中，根据传入 size 的大小，查找到第一个大于等于 size 的 2 的次幂，作为新的哈希表的大小。在 dictResize 函数中，传入的是 ht[0].used，而在 _dictExpandIfNeeded 中传入的是 ht[0].used * 2。

### 渐进式迁移

如果保存的哈希表节点特别多，将整个迁移过程一次性完成的话，将很长时间无法进行正常的插入操作，所以 Redis 使用渐进式的方式来完成哈希节点的迁移。具体的过程在 dictRehash 中实现，这个函数完成 n 个哈希表节点的迁移，并同时检测 rehash 操作是否已经完成：当整个 ht[0] 中的节点都迁移到 ht[1] 以后，释放 ht[0] 的空间，将 ht[1] 换入 ht[0]，同时重置所有的 rehash 相关的参数，就完成 rehash 的过程：

```C
int dictRehash(dict *d, int n) {
    // 只可以在 rehash 进行中时执行
    if (!dictIsRehashing(d)) return 0;
    // 进行 N 步迁移
    while(n--) {
        dictEntry *de, *nextde;
        // 如果 0 号哈希表为空，那么表示 rehash 执行完毕
        if (d->ht[0].used == 0) {
            zfree(d->ht[0].table);
            d->ht[0] = d->ht[1];
            _dictReset(&d->ht[1]);
            d->rehashidx = -1;
            return 0;
        }
        // 确保 rehashidx 没有越界
        assert(d->ht[0].size > (unsigned)d->rehashidx);
        // 略过数组中为空的索引，找到下一个非空索引
        while(d->ht[0].table[d->rehashidx] == NULL) d->rehashidx++;
        // 指向该索引的链表表头节点
        de = d->ht[0].table[d->rehashidx];
        // 将链表中的所有节点迁移到新哈希表
        while(de) {
            unsigned int h;
            // 保存下个节点的指针
            nextde = de->next;
            // 计算新哈希表的哈希值，以及节点插入的索引位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            // 插入节点到新哈希表
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            // 更新计数器
            d->ht[0].used--;
            d->ht[1].used++;

            de = nextde;
        }
        // 将刚迁移完的哈希表索引的指针设为空
        d->ht[0].table[d->rehashidx] = NULL;
        // 更新 rehash 索引
        d->rehashidx++;
    }
    return 1;
}
```

那么什么时候执行 Rehash 的操作呢？Redis 定义了两个进行渐进式操作的时机：

- 在哈希表被使用的同时，例如对哈希表进行查询或修改，执行 n = 1 的 dictRehash 操作，即调用 _dictRehashStep

```C
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```

- 如果哈希表长时间没有被使用，同样也需要将 rehash 操作进行下去，dictRehashMilliseconds 函数在指定时间内（10ms）以 100 次为单位进行 dictRehash 操作

```C
int dictRehashMilliseconds(dict *d, int ms) {
    // 记录开始时间
    long long start = timeInMilliseconds();
    int rehashes = 0;
    while(dictRehash(d,100)) {
        rehashes += 100;
        // 如果时间已过，跳出
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```
### Rehash 执行期间的哈希表使用

在字典进行 Rehash 的过程中，两个哈希表 ht[0] 和 ht[1] 同时存在并且被使用。因此，所有的查找、更新和删除都需要在两个哈希表上同时进行（先在 ht[0] 上进行操作，如果没有对应的目标，则在 ht[1] 上继续进行）。而增加操作只在 ht[1] 上进行，随着渐进式的迁移操作，ht[0] 表中的节点会越来越少，直到所有的节点都进入 ht[1]，rehash就完成了。

## 哈希表的迭代器

迭代器的结构定义如下：

```C
typedef struct dictIterator {

    // 被迭代的字典
    dict *d;
    // table ：正在被迭代的哈希表号码，值可以是 0 或 1 。
    // index ：迭代器当前所指向的哈希表索引位置。
    // safe ：标识这个迭代器是否安全
    int table, index, safe;
    // entry ：当前迭代到的节点的指针
    // nextEntry ：当前迭代节点的下一个节点
    //             因为在安全迭代器运作时， entry 所指向的节点可能会被修改，
    //             所以需要一个额外的指针来保存下一节点的位置，
    //             从而防止指针丢失
    dictEntry *entry, *nextEntry;
    long long fingerprint;
} dictIterator;
```
Redis 的哈希表的迭代器按照是否 Rehash 安全分为两种，由 safe 字段进行标注。当 Rehash 安全的迭代器存在的时候，将不会进行单步的 Rehash，即在 _dictRehashStep 函数中先判断字典是否存在使用中的安全迭代器，不存在时才会进行 dictRehash。但是对于定时执行的情况，我没有找到在代码中保证在有安全迭代器时不进行 dictRehash 的判断，或许需要阅读更多的代码才能进行判断。
此外，由于迭代器返回以后可能会将迭代器当前指针对应的节点删除，所以还需要保存后续节点指针。而fingerprint 是使用 Thomas Wang 64bits 哈希算法在创建不安全迭代器时生成的指纹，用于在迭代器释放的时候判断在该迭代器在执行的过程中哈希表是否发生了变化。
为什么需要设置这两类迭代器呢？主要针对进行迭代的哈希表是否是可变的。由于在安全迭代器运行的期间不能执行 Rehash 操作，因此会影响性能，在对不可变的哈希表进行迭代操作时（例如 Redis 利用子线程进行 dump 操作），就可以使用不安全的迭代器。

## Scan 操作

由于整体的数据设计，Redis 没有办法提供特别准确的 Scan 操作，它的特点是：
- 提供键空间的遍历操作，支持游标，遍历一遍值需要 O(N) 的时间复杂度
- 无法提供完整的快照遍历，也就是中间如果有数据修改，可能有些涉及改动的数据遍历不到
- 每次返回的数据条数不一定，极度依赖内部实现
- 返回的数据可能有重复，应用层必须能够处理重入逻辑

整个 Scan 操作的算法比较让人费解，我找到了一个写得比较好的博客，直接在这里转载一下，有兴趣的可以直接看，这里就不做过多的解释了：http://chenzhenianqing.cn/articles/1101.html

## 参考资料
- [Redis 设计与实现 - 字典](http://redisbook.readthedocs.io/en/latest/internal-datastruct/dict.html)
- [Redis的字典(dict)rehash过程源码解析](http://www.tuicool.com/articles/3eeIJrR)
- [Redis Scan迭代器遍历操作原理（一）– 基础](http://chenzhenianqing.cn/articles/1090.html)
- [Redis Scan迭代器遍历操作原理（二）– dictScan反向二进制迭代器](http://chenzhenianqing.cn/articles/1101.html)
- [what's the difference of safe/non-safe dict itereator in redis' dict implementation?](http://stackoverflow.com/questions/9223397/whats-the-difference-of-safe-non-safe-dict-itereator-in-redis-dict-implementat)
- [Redis设计与实现（一）内部数据结构](http://github.thinkingbar.com/redisbook_chapter01/)
