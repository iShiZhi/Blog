---
title: 'Redis数据结构: 跳跃表'
categories: 开源项目
tags: [Redis]
date: 2016-11-12
---

对于普通的链表而言，即使包含的元素是有序的，包括查找在内的各项操作的时间复杂度也是 O(N)，并没有使用有序的特性。而跳跃表则是对于链表的改良，通过增加额外的空间，使得各项操作的时间复杂度降低。

与双端链表和哈希表相比，跳跃表这个概念大部分人都会觉得陌生一些，之前学数据结构的时候学过，但从来没有使用过，现在看到 Redis 里面实现的跳跃表，就把这个数据结构温习一下，在这边也做个介绍。跳跃表通过对有序链表的每个节点增加一些前进链接，获得快速查找访问节点的能力。下图是一个跳跃表的示例（来自百度百科）：

<img src="http://ocx5ae9jo.bkt.clouddn.com/skip-list-example.jpg" width="500px;" height="200px;">

跳跃表是按层建造的。底层是一个普通的有序链表。每个更高层的构建都是为了跳跃若干个节点更快地定位到需要查找的节点。在构建跳跃表的时候，设定每层会向上增长的概率为 1 / p，则第 m 层向上增长的概率为 1 / p^m；假设链表中的元素个数为 n，则在 m 层元素数目的期望是 n / p^m；当这个数量为 1 时，m = log p(n) 即为层数。而将所有的层上的节点数量相加，即 n + n / p + …… + 1 < 2 n，因此跳跃表的空间复杂度是 O(N)。对于一个节点而言，它恰好层数等于 1 的概率为 1 - p，层数等于 2 的概率为 p (1 - p)，层数等于 3 的概率为 p^2 (1 - p)，因此，一个节点的平均层数为 1 (1 - p) + 2 p (1 - p) + 3 p^2 (1 - p) + … = 1 / (1 - p)，这就是一个节点对应的指针数量。
跳跃列表的时间复杂度的分析比较复杂，这边只给结论，它的平均时间复杂度为 O(log n)。具体的推导过程可以查看 William Pugh 的论文 [Skip Lists: A Probabilistic Alternative to Balanced Trees](ftp://ftp.cs.umd.edu/pub/skipLists/skiplists.pdf)。

可能也有人会问，各类的平衡树也可以用类似的时间复杂度来实现同样的功能，那么为什么 Redis 不用平衡树呢？具体的比较如下：

- 在做范围查找的时候，平衡树比跳跃表操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
- 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。
- 从内存占用上来说，跳跃表比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而跳跃表每个节点包含的指针数目平均为 1 / (1 - p)，具体取决于参数 p 的大小。在 Redis 实现时，p = 1 / 4，那么平均每个节点包含 1.33 个指针，所需要的存储空间比平衡树更小。
- 跳跃表的实现比平衡树更简单。

基于上述的考虑，Redis 采用了性能相似，但在存储空间更节省，时间复杂度更低的跳跃表来进行有序元素的存储。

### 基本结构

在 server.h 中定义了跳跃表的节点结构：

```C
typedef struct zskiplistNode {
    sds ele;                                // 字符串指针
    double score;                           // 分值
    struct zskiplistNode *backward;         // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;      // 前进指针
        unsigned int span;                  // 跨度
    } level[];                              // 该结点出现在的不同层
} zskiplistNode;
```
在这个结构中，定义了跳跃表需要的一些基本属性：成员对象、分值和不同层的前进指针和跨度。另外，多定义了一个后退指针，主要是为了能够从表尾开始遍历整个链表。另外，由于在 Redis 中跳跃表的作用比较有限，所以在最新的版本中，把跳跃表的成员对象的类型改为 sds。

在经典的跳跃表中，只需要使用上述定义的节点结构就可以实现跳跃表。但是为了方便诸如节点数量查询，从表尾遍历等操作，定义了一个结构存储链表的一些相关信息，具体如下：

```C
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;    // 表头节点和表尾节点
    unsigned long length;                   // 表中节点的数量
    int level;                              // 表中层数最大的节点的层数
} zskiplist;
```

### 各类操作

在上述定义的结构中，我们可以进行链表相关的各类操作，由于添加操作中包含查询的相关逻辑，因此，我们就来看看添加操作的代码，查询元素，或者按照排位查询元素的逻辑也基本类似。
添加操作的代码在 tzset.c 中，这个操作的平均时间复杂度为 O(log N)，最差的时间复杂度为 O(N)，具体的细节参见代码的注释：

```C
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;
    redisAssert(!isnan(score));
    // 在各个层查找节点的插入位置
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 如果 i 不是 zsl->level-1 层，那么 i 层的起始 rank 值为 i+1 层的 rank 值
        // 最终 rank[0] 的值加一就是新节点的前置节点的排位
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 沿着前进指针遍历跳跃表
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员， T = O(N)
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            // 记录沿途跨越了多少个节点
            rank[i] += x->level[i].span;
            // 移动至下一指针
            x = x->level[i].forward;
        }
        // 记录将要和新节点相连接的节点
        update[i] = x;
    }
    /*
     * zslInsert() 的调用者会确保同分值且同成员的元素不会出现，
     * 所以这里不需要进一步进行检查，可以直接创建新元素。
     */
    // 获取一个随机值作为新节点的层数，
    level = zslRandomLevel();
    // 如果新节点的层数比表中其他节点的层数都要大
    // 那么初始化表头节点中未使用的层，并将它们记录到 update 数组中
    // 将来也指向新节点
    if (level > zsl->level) {
        // 初始化未使用层
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        // 更新表中节点最大层数
        zsl->level = level;
    }
    // 创建新节点
    x = zslCreateNode(level,score,obj);
    // 将前面记录的指针指向新节点，并做相应的设置
    // T = O(1)
    for (i = 0; i < level; i++) {

        // 设置新节点的 forward 指针
        x->level[i].forward = update[i]->level[i].forward;

        // 将沿途记录的各个节点的 forward 指针指向新节点
        update[i]->level[i].forward = x;
        /* update span covered by update[i] as x is inserted here */
        // 计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        // 更新新节点插入之后，沿途节点的 span 值
        // 其中的 +1 计算的是新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    /* increment span for untouched levels */
    // 未接触的节点的 span 值也需要增一，这些节点直接从表头指向新节点
    // T = O(1)
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
    // 设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    // 跳跃表的节点计数增一
    zsl->length++;
    return x;
}
```
此外，Redis 实现的跳跃表除了常规的链表操作以外，还增加了一些与范围相关的操作，具体的实现方式与上述的添加操作也类似。Redis 定义的范围有两种，一种是按照数值的大小定义的区间，另一种是按照字典序的先后定义的区间，跳跃表对于这两种范围都定义了操作。主要的操作有：判断给定的节点是否在范围中，范围中的第一个节点或最后一个节点，以及删除范围中的所有节点。
