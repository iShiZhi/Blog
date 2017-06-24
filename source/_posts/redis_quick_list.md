---
title: 'Redis数据结构: 快速列表'
categories: 开源项目
tags: [Redis]
date: 2016-11-22
---

在前面对于压缩列表的介绍中，我们可以知道压缩列表的一些缺点，如查找操作需要 O(N) 的时间复杂度（即使有对应节点的索引值），数据变动的时候可能会引发连锁的更新，这些缺点决定了压缩列表无法用于大规模的数据。而快速列表的设计，则一定程度上克服了这些缺点，在空间利用率和时间复杂度上取得一个平衡。

快速列表的本质是一个双端链表，它的每一个节点都是压缩列表。这样的设计好处是：

- 可以通过限制压缩列表的节点数和长度，使得双端链表的每个节点对应的压缩列表不会太大，避免更新数据产生 realloc 时需要大量的数据拷贝；
- 使用压缩列表作为节点，可以使得连续的节点的存储在同一段内存空间中，减少内存碎片，同时提高存储效率；
- 由于压缩列表中记录了节点数，所以在使用索引进行查找时，可以比单纯使用双端链表或压缩列表更快。

此外，由于 Redis 频繁地在快速列表的两端进行操作，而中间的节点比较少被使用到，因此，使用 LZF 压缩算法对中间的压缩列表进行压缩能够进一步地节省空间。

### 数据结构

quicklist.h 中定义了快速列表的节点结构，包含了：

- prev、next：前驱节点和后继节点的指针
- zl：指向压缩列表的指针
- sz：压缩列表中的字节数，包括了 zlbytes、zltail、zllen、zlend 和各个数据项，此外，即使压缩列表被压缩了，这边表示的依旧是压缩前的字节数
各类标记，这里使用了一个 C 语言的语法，unsigned int count : 16 表示使用 16 个 bit 来存储 count 变量，因此使用了 32 bit（即 2 个字节）来存储这些标记，包括：
  - count（16 bits）：压缩列表包含的数据项
  - encoding（2 bits）：表示是否压缩过，未压缩则是 1，压缩过的则是 2
  - container（2 bits）：预留字段，当前是固定值 2
  - recompress（1 bit）：置为 1 的时候表示需要将压缩列表重新压缩，由于在遍历的时候，需要把压缩的节点解压后使用，所以解压后设置标志位，以便后面再进行压缩
  - attempted_compress（1 bit）：测试时用的字段，表示不需要进行压缩
  - extra（10 bits）：还未用上的预留空间

```C
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```
quicklistLZF 表示了一个被压缩的节点结构，其中：

- sz：表示压缩以后压缩列表的字节数
- compressed：存放压缩后的压缩列表的字节数组，与 SDS 的 buf 一样，这是一个柔性数组。

```C
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```

快速列表的结构则包括：

- head、tail：列表表头和表尾指针
- count：所有的压缩列表的节点数总和
- len：快速列表的节点数
- 32 位的标记：
  - fill（16 bits）：ziplist 大小设置，存放 list-max-ziplist-size 参数的值
    若为负数，则表示ziplist的最大字节数，取值为 -1（4 Kb），-2（8 Kb），-3（16 Kb），-4（32 Kb），-5（64 Kb）
    若为正数，则表示ziplist的最大节点数，最大为 8192（2^13）
  - compress（16 bits）：节点压缩深度设置，存放 list-compress-depth 参数的值，表示前后各有几个节点不压缩，0 是个特殊值，表示所有节点都不压缩，这是 Redis 的默认值

```C
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes */
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```
快速列表中 fill 属性的设置是为了使存储效率和时间复杂度达到一个平衡：

- 如果压缩列表的节点数太多，则会导致分配连续内存空间的难度加大，同时更新节点数据操作造成大量数据拷贝的风险加大；
- 如果压缩列表的节点数太少，则会使快速列表退化成双端链表，存储效率低，内存空间随便多。

根据使用的场景不同，可以对应地设置限定压缩列表的字节数（每个节点需要的存储空间较大时）或节点数（每个节点需要的存储空间较小时），同时这个参数的设置，也影响了 quicklistNode 结构中的 count 的位数：

- 若这个参数取正值时，压缩列表的最大个数被限定在 2^15（实际上限制为 8192，即 2^13），可以使用 16 bits 进行存储；
- 若这个参数取负值时，压缩列表的最大字节数是 64 K，由于每一个节点至少占用 2 个字节，因此最大的数量为 32 K，也可以使用 16 bits 进行存储。

而 compress 属性的设置主要是由于快速列表的使用场景是针对列表两端的节点进行频繁操作，而对于列表中间的节点进行操作的频率很低，这样压缩中间的节点，可以节省空间，而两端的节点不压缩则方便操作。

### 压缩列表大小的判定

由于快速列表设计的关键是压缩列表的大小限定，通过列表结构中的 fill 属性进行约束，因此在进行压缩列表新节点的插入，或者两个压缩列表的合并时，需要判定是否满足 fill 定义的限制条件，具体代码如下：

```C
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {
    if (unlikely(!node))
        return 0;
    int ziplist_overhead;
    /* size of previous offset
     *
     * 判断 header 占用的字节数
     */
    if (sz < 254)
        ziplist_overhead = 1;
    else
        ziplist_overhead = 5;
    /* size of forward offset
     *
     * 判断 encoding 占用的字节数
     */
    if (sz < 64)
        ziplist_overhead += 1;
    else if (likely(sz < 16384))
        ziplist_overhead += 2;
    else
        ziplist_overhead += 5;
    /* new_sz overestimates if 'sz' encodes to an integer type
     *
     * 当新节点存入的是整型数时，这边会过高估计使用的字节数
     */
    unsigned int new_sz = node->sz + sz + ziplist_overhead;
    // 首先判断字节数是否满足要求，若 fill > 0，则表示对字节数没有限制，只对数量进行限制
    if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(new_sz, fill)))
        return 1;
    else if (!sizeMeetsSafetyLimit(new_sz))
        return 0;
    // 若到这里进行判断，则要么是字节数不满足限制（这种情况下 fill 为负数，这边也肯定不满足要求）
    // 要么是字节数没有限制，而节点数有限制，此时 fill 为正数
    else if ((int)node->count < fill)
        return 1;
    else
        return 0;
}
REDIS_STATIC int _quicklistNodeAllowMerge(const quicklistNode *a,
                                          const quicklistNode *b,
                                          const int fill) {
    if (!a || !b)
        return 0;
    /* approximate merged ziplist size (- 11 to remove one ziplist
     * header/trailer)
     *
     * 11 是 header 和 tail 的大小，合并以后只需要一个 header 和一个 tail
     */
    unsigned int merge_sz = a->sz + b->sz - 11;
    // 首先判断字节数是否满足条件
    if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(merge_sz, fill)))
        return 1;
    // 其次判断是否满足最大字节数的条件
    else if (!sizeMeetsSafetyLimit(merge_sz))
        return 0;
    // 判断节点数是否满足条件
    else if ((int)(a->count + b->count) <= fill)
        return 1;
    else
        return 0;
}
```

### Push 和 Pop 操作

如前所述，快速列表的操作经常在两端进行，因此定义了 Push 和 Pop 两个操作。
在进行 Push 操作的时候，需要判断表头节点（或表尾节点）是否有足够的空间（满足 fill 属性）可以存储新的节点。若可以存储，则直接存入，否则需要新建一个快速列表节点（即压缩列表）进行存储。具体代码如下：

```C
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_tail = quicklist->tail;
    if (likely(
            _quicklistNodeAllowInsert(quicklist->tail, quicklist->fill, sz))) {
        quicklist->tail->zl =
            ziplistPush(quicklist->tail->zl, value, sz, ZIPLIST_TAIL);
        quicklistNodeUpdateSz(quicklist->tail);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_TAIL);
        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeAfter(quicklist, quicklist->tail, node);
    }
    quicklist->count++;
    quicklist->tail->count++;
    return (orig_tail != quicklist->tail);
}
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {
        quicklistPushTail(quicklist, value, sz);
    }
}
```

在进行 Pop 操作的时候，需要根据传入的 saver 函数将取出的节点内容保存在对应的指针中：

```C
int quicklistPopCustom(quicklist *quicklist, int where, unsigned char **data,
                       unsigned int *sz, long long *sval,
                       void *(*saver)(unsigned char *data, unsigned int sz)) {
    unsigned char *p;
    unsigned char *vstr;
    unsigned int vlen;
    long long vlong;
    int pos = (where == QUICKLIST_HEAD) ? 0 : -1;
    if (quicklist->count == 0)
        return 0;
    if (data)
        *data = NULL;
    if (sz)
        *sz = 0;
    if (sval)
        *sval = -123456789;
    quicklistNode *node;
    if (where == QUICKLIST_HEAD && quicklist->head) {
        node = quicklist->head;
    } else if (where == QUICKLIST_TAIL && quicklist->tail) {
        node = quicklist->tail;
    } else {
        return 0;
    }
    // 取出对应的节点
    p = ziplistIndex(node->zl, pos);
    if (ziplistGet(p, &vstr, &vlen, &vlong)) {
        if (vstr) {
            if (data)
                *data = saver(vstr, vlen);
            if (sz)
                *sz = vlen;
        } else {
            if (data)
                *data = NULL;
            if (sval)
                *sval = vlong;
        }
        quicklistDelIndex(quicklist, node, &p);
        return 1;
    }
    return 0;
}
REDIS_STATIC void *_quicklistSaver(unsigned char *data, unsigned int sz) {
    unsigned char *vstr;
    if (data) {
        vstr = zmalloc(sz);
        memcpy(vstr, data, sz);
        return vstr;
    }
    return NULL;
}
int quicklistPop(quicklist *quicklist, int where, unsigned char **data,
                 unsigned int *sz, long long *slong) {
    unsigned char *vstr;
    unsigned int vlen;
    long long vlong;
    if (quicklist->count == 0)
        return 0;
    int ret = quicklistPopCustom(quicklist, where, &vstr, &vlen, &vlong,
                                 _quicklistSaver);
    if (data)
        *data = vstr;
    if (slong)
        *slong = vlong;
    if (sz)
        *sz = vlen;
    return ret;
}
```

### 任意位置的插入

除了从头部或尾部插入，快速列表还实现了从任意指定的位置插入，这时候由于需要满足 fill 设定的压缩列表大小，会将逻辑分为如下几种情况：

- 若当前的压缩列表有空间可以存入新的节点，则直接存入；
- 若当前的压缩列表没有空间，且插入位置在表头或表尾，则考虑前一个或后一个压缩列表是否有空间存储：
  - 若有空间，则直接存入；
  - 否则，新建一个快速列表的节点，将其插入对应的位置，用于存储新节点；
- 若当前的压缩列表没有空间，且插入位置不在表头或表尾，则需要将当前的压缩列表拆分以后，插入合适的压缩列表。

具体代码如下：

```C
REDIS_STATIC void _quicklistInsert(quicklist *quicklist, quicklistEntry *entry,
                                   void *value, const size_t sz, int after) {
    int full = 0, at_tail = 0, at_head = 0, full_next = 0, full_prev = 0;
    int fill = quicklist->fill;
    quicklistNode *node = entry->node;
    quicklistNode *new_node = NULL;
    if (!node) {
        /* we have no reference node, so let's create only node in the list
         *
         * 若没有指定快速列表的节点，则新建一个节点用于存储压缩列表节点
         */
        D("No node given!");
        new_node = quicklistCreateNode();
        new_node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        __quicklistInsertNode(quicklist, NULL, new_node, after);
        new_node->count++;
        quicklist->count++;
        return;
    }
    /* Populate accounting flags for easier boolean checks later
     *
     * 判断压缩列表是否还能存入新节点，插入的位置在表头或者表尾
     */
    if (!_quicklistNodeAllowInsert(node, fill, sz)) {
        D("Current node is full with count %d with requested fill %lu",
          node->count, fill);
        full = 1;
    }
    if (after && (entry->offset == node->count)) {
        D("At Tail of current ziplist");
        at_tail = 1;
        if (!_quicklistNodeAllowInsert(node->next, fill, sz)) {
            D("Next node is full too.");
            full_next = 1;
        }
    }
    if (!after && (entry->offset == 0)) {
        D("At Head");
        at_head = 1;
        if (!_quicklistNodeAllowInsert(node->prev, fill, sz)) {
            D("Prev node is full too.");
            full_prev = 1;
        }
    }
    /* Now determine where and how to insert the new element
     *
     * 判断该如何插入节点
     */
    if (!full && after) {
        D("Not full, inserting after current position.");
        quicklistDecompressNodeForUse(node);
        unsigned char *next = ziplistNext(node->zl, entry->zi);
        // 判断是否在压缩列表的表尾插入
        if (next == NULL) {
            node->zl = ziplistPush(node->zl, value, sz, ZIPLIST_TAIL);
        } else {
            node->zl = ziplistInsert(node->zl, next, value, sz);
        }
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(quicklist, node);
    } else if (!full && !after) {
        D("Not full, inserting before current position.");
        quicklistDecompressNodeForUse(node);
        node->zl = ziplistInsert(node->zl, entry->zi, value, sz);
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(quicklist, node);
    } else if (full && at_tail && node->next && !full_next && after) {
        /* If we are: at tail, next has free space, and inserting after:
         *   - insert entry at head of next node.
         *
         * 如果当前压缩列表已满，而后一个节点未满，且插入在当前压缩列表的表尾，则插入到后一个节点表头
         */
        D("Full and tail, but next isn't full; inserting next node head");
        new_node = node->next;
        quicklistDecompressNodeForUse(new_node);
        new_node->zl = ziplistPush(new_node->zl, value, sz, ZIPLIST_HEAD);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(quicklist, new_node);
    } else if (full && at_head && node->prev && !full_prev && !after) {
        /* If we are: at head, previous has free space, and inserting before:
         *   - insert entry at tail of previous node.
         *
         * 如果当前压缩列表已满，而前一个节点未满，且插入在当前压缩列表的表头，则插入到前一个节点表尾
         */
        D("Full and head, but prev isn't full, inserting prev node tail");
        new_node = node->prev;
        quicklistDecompressNodeForUse(new_node);
        new_node->zl = ziplistPush(new_node->zl, value, sz, ZIPLIST_TAIL);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(quicklist, new_node);
    } else if (full && ((at_tail && node->next && full_next && after) ||
                        (at_head && node->prev && full_prev && !after))) {
        /* If we are: full, and our prev/next is full, then:
         *   - create new node and attach to quicklist
         * 如果插入在压缩列表的表头或表尾，且前后节点也已满，则创建新快速列表节点
         */
        D("\tprovisioning new node...");
        new_node = quicklistCreateNode();
        new_node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
    } else if (full) {
        /* else, node is full we need to split it.
         * covers both after and !after cases
         *
         * 如果不是在两端，则将当前的节点进行拆分后再进行插入
         */
        D("\tsplitting node...");
        quicklistDecompressNodeForUse(node);
        new_node = _quicklistSplitNode(node, entry->offset, after);
        new_node->zl = ziplistPush(new_node->zl, value, sz,
                                   after ? ZIPLIST_HEAD : ZIPLIST_TAIL);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
        _quicklistMergeNodes(quicklist, node);
    }
    quicklist->count++;
}
void quicklistInsertBefore(quicklist *quicklist, quicklistEntry *entry,
                           void *value, const size_t sz) {
    _quicklistInsert(quicklist, entry, value, sz, 0);
}
void quicklistInsertAfter(quicklist *quicklist, quicklistEntry *entry,
                          void *value, const size_t sz) {
    _quicklistInsert(quicklist, entry, value, sz, 1);
}
```

### 迭代器

快速列表定义了迭代器和压缩列表节点结构，并且可以从表头和表尾两个方向对所有的压缩列表中的节点进行遍历，具体的数据结构定义如下：

```C
// 快速列表的迭代器，用于遍历快速列表中所有压缩列表的所有节点，按照每个压缩列表节点进行遍历
// 可以选择遍历压缩列表的方向，从头遍历 AL_START_HEAD 或者从尾遍历 AL_START_TAIL
typedef struct quicklistIter {
    const quicklist *quicklist;    // 遍历的快速列表指针
    quicklistNode *current;        // 遍历的快速列表节点指针
    unsigned char *zi;             // 遍历到的压缩列表节点的指针
    long offset;                   // 遍历到的压缩列表节点的偏移量
    int direction;                 // 遍历方向 AL_START_HEAD 或 AL_START_TAIL
} quicklistIter;
// 快速列表迭代过程中取出的压缩列表节点的结构
typedef struct quicklistEntry {
    const quicklist *quicklist;    // 快速列表指针
    quicklistNode *node;           // 快速列表的节点指针
    unsigned char *zi;             // 压缩列表节点的指针
    unsigned char *value;          // 压缩列表节点的字符串指针（如果内容无法保存到 longval 的话）
    long long longval;             // 压缩列表节点的数值
    unsigned int sz;               // 压缩列表节点的字符串长度
    int offset;                    // 压缩列表节点的索引值，如果从后向前遍历时，为负数
} quicklistEntry;
```

以下是具体的遍历代码，在进行遍历的过程中，会逐一取出压缩列表的节点，若节点被压缩，则进行解压，同时标注 recompress 为 1，以待该节点被使用完以后重新压缩。

```C
int quicklistNext(quicklistIter *iter, quicklistEntry *entry) {
    initEntry(entry);
    if (!iter) {
        D("Returning because no iter!");
        return 0;
    }
    // 从迭代器里获得当前压缩节点对应的快速列表信息
    entry->quicklist = iter->quicklist;
    entry->node = iter->current;
    // 如果当前的快速列表的节点为空，则返回
    if (!iter->current) {
        D("Returning because current node is NULL")
        return 0;
    }
    // 定义一个函数指针，后面将根据遍历的方向决定具体的实现
    unsigned char *(*nextFn)(unsigned char *, unsigned char *) = NULL;
    int offset_update = 0;
    // 如果已经有对应的指针，则说明在遍历一个压缩列表没有结束，这时候可以直接使用，否则需要解压一个快速列表的节点，然后查找对应的节点
    if (!iter->zi) {
        /* If !zi, use current index.
         *
         * 上一个压缩列表已经遍历完了，现在要遍历一个新的快速列表节点对应的压缩列表
         */
        quicklistDecompressNodeForUse(iter->current);
        iter->zi = ziplistIndex(iter->current->zl, iter->offset);
    } else {
        /* else, use existing iterator offset and get prev/next as necessary.
         *
         * 获取下一个节点的指针和偏移量
         */
        if (iter->direction == AL_START_HEAD) {
            nextFn = ziplistNext;
            offset_update = 1;
        } else if (iter->direction == AL_START_TAIL) {
            nextFn = ziplistPrev;
            offset_update = -1;
        }
        iter->zi = nextFn(iter->current->zl, iter->zi);
        iter->offset += offset_update;
    }
    entry->zi = iter->zi;
    entry->offset = iter->offset;
    if (iter->zi) {
        /* Populate value from existing ziplist position
         *
         * 如果当前的压缩列表没有遍历完成，则取出对应的节点的值
         */
        ziplistGet(entry->zi, &entry->value, &entry->sz, &entry->longval);
        return 1;
    } else {
        /* We ran out of ziplist entries.
         * Pick next node, update offset, then re-run retrieval.
         *
         * 将遍历完的节点压缩，然后按照遍历方向进行下一个快速列表节点对应的压缩列表的遍历
         */
        quicklistCompress(iter->quicklist, iter->current);
        if (iter->direction == AL_START_HEAD) {
            /* Forward traversal */
            D("Jumping to start of next node");
            iter->current = iter->current->next;
            iter->offset = 0;
        } else if (iter->direction == AL_START_TAIL) {
            /* Reverse traversal */
            D("Jumping to end of previous node");
            iter->current = iter->current->prev;
            iter->offset = -1;
        }
        // 获取新的压缩列表的第一个遍历节点（根据遍历方向，有可能是头节点，也可能是尾节点）
        iter->zi = NULL;
        return quicklistNext(iter, entry);
    }
}
```
快速列表的主要操作都在上面进行了叙述，其它的操作可以查看相应的源代码注释。
