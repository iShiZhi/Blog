---
title: 'Redis数据结构: 压缩列表'
categories: 开源项目
tags: [Redis]
date: 2016-11-3
---

与整数集合类似，压缩列表也是 Redis 为了节约内存而开发的。它是由一系列特殊编码的内存块构成的列表。一个压缩列表可以包含多个节点，每个节点可以保存一个长度受限的字符数组或者整数。通过特殊的编码方式，可以用最少的空间来存储不同长度的字符串和整数，使得使用的存储空间最小。

### 压缩列表的基本结构

一个压缩列表包含如下的字段：

- zlbytes：uint32_t，记录整个压缩列表占用的内存字节数
- zltail：uint32_t，记录压缩列表表尾节点距离压缩列表的起始发位置的字节数
- zllen：uint16_t，记录了压缩列表包含的节点数量，注意，这个节点的数量不一定是真实的数量，当节点的数量小于 65535（uint16_t 所能表达的最大的值）时，这个字段记录了节点的数量，当数量超过这个限制的时候，就需要遍历整个压缩列表才能获得节点的数量
- entryX：在上述 10 个节点之后，是保存在这个压缩列表中的节点，长度由具体的结构决定。每个压缩列表的节点保存了一个长度受限的字节数组，或者一个整数值，具体的结构如下：
  - previous_entry_length：记录前一个节点的长度，如果前一个节点的长度小于 254，即 1 个字节能够存储，则使用 1 个字节来存储这个长度；否则使用 5 个字节来保存，其中第一个字节固定保存 255，后续四个字节保存长度。这个字段与 zltail 联合起来，可以完成从压缩列表的末端开始，向表头遍历的操作：首先，通过表头指针和 zltail 计算获得最后一个节点的指针；然后，通过最后一个节点的 previous_entry_length 计算获得前一个节点的指针，重复上述过程，就可以持续向前直到表头
  - encoding：保存节点的 content 属性对应的数据类型和长度，encoding 的第一个字节的前两位决定了保存的数据类型
    - 00：表示的是长度小于等于 63（2^4 - 1）的字节数组，此时 encoding 的长度是 1 个字节
    - 01：表示的是长度小于等于 16383（2^4 - 1）的字节数组，此时 encoding 的长度是 2 个字节
    - 10：表示的是长度小于等于 4294967295（23^2 - 1）的字节数组，此时 encoding 的长度是 5 个字节，此时第一个字节除了前两位以外全部留空
    - 11：表示的是整数，对应五种整数类型
      - 11000000：int16_t
      - 11010000：int32_t
      - 11100000：int64_t
      - 11110000：24 位有符号整数
      - 11111110：8 位有符号整数
      - 1111xxxx：可以表示 0 - 12 这几个数，由于 0000、1110 和 1111 被占用，因此二进制值为 1 - 13，减 1 表示实际的值
  - content：节点保存的值
- zlend：uint8_t，用特殊值 0xFF（255）来标记压缩列表的末端

### Push 和 Pop 操作

Redis 的压缩列表支持从两端插入（Push）或删除（Pop）节点，以 Push 为例：

```C
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    // 根据 where 参数的值，决定将值插入到表头还是表尾
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    // 返回添加新值后的 ziplist
    return __ziplistInsert(zl,p,s,slen);
}
```
在 __ziplistInsert 函数中实现了新节点的插入，主要的步骤为：

1. 根据前置节点和当前节点计算 entry 的 previous_entry_length、encoding 和 contents 字段占用的字节数
2. 判断当前指针指向的位置是否能够存储当前的节点，如果不能存储，则需用 realloc 函数调整压缩列表的大小
3. 移动后续节点腾出空间，将当前需要插入的节点放入，同时需要更新后续的节点的 previous_entry_length

具体的代码如下：

```C
static unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    // 记录当前 ziplist 的长度
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789;
    zlentry entry, tail;
    if (p[0] != ZIP_END) {
        // 如果 p[0] 不指向列表末端，说明列表非空，并且 p 正指向列表的其中一个节点，那么取出 p 所指向节点的信息，并将它保存到 entry 结构中
        // 然后用 prevlen 变量记录前置节点的长度，当插入新节点之后 p 所指向的节点就成了新节点的前置节点
        entry = zipEntry(p);
        prevlen = entry.prevrawlen;
    } else {
        // 如果 p 指向表尾末端，那么程序需要检查列表是否为：
        // 1)如果 ptail 也指向 ZIP_END ，那么列表为空；
        // 2)如果列表不为空，那么 ptail 将指向列表的最后一个节点。
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            // 表尾节点为新节点的前置节点
            // 取出表尾节点的长度
            // T = O(1)
            prevlen = zipRawEntryLength(ptail);
        }
    }
    // 尝试看能否将输入字符串转换为整数，如果成功的话：
    // 1)value 将保存转换后的整数值
    // 2)encoding 则保存适用于 value 的编码方式
    // 无论使用什么编码， reqlen 都保存节点值的长度
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        reqlen = zipIntSize(encoding);
    } else {
        reqlen = slen;
    }

    // 计算编码前置节点的长度所需的大小
    reqlen += zipPrevEncodeLength(NULL,prevlen);
    // 计算编码当前节点值所需的大小
    reqlen += zipEncodeLength(NULL,encoding,slen);
    // 只要新节点不是被添加到列表末端，
    // 那么程序就需要检查看 p 所指向的节点（的 header）能否编码新节点的长度。
    // nextdiff 保存了新旧编码之间的字节大小差，如果这个值大于 0
    // 那么说明需要对 p 所指向的节点（的 header ）进行扩展
    // T = O(1)
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    // 因为重分配空间可能会改变 zl 的地址
    // 所以在分配之前，需要记录 zl 到 p 的偏移量，然后在分配之后依靠偏移量还原 p
    offset = p-zl;

    // curlen 是 ziplist 原来的长度
    // reqlen 是整个新节点的长度
    // nextdiff 是新节点的后继节点扩展 header 的长度（要么 0 字节，要么 4 个字节）
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;
    if (p[0] != ZIP_END) {
        // 新元素之后还有节点，因为新元素的加入，需要对这些原有节点进行调整
        // 移动现有元素，为新元素的插入空间腾出位置
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);
        // 将新节点的长度编码至后置节点
        // p+reqlen 定位到后置节点
        // reqlen 是新节点的长度
        zipPrevEncodeLength(p+reqlen,reqlen);
        // 更新到达表尾的偏移量，将新节点的长度也算上
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);
        // 如果新节点的后面有多于一个节点
        // 那么程序需要将 nextdiff 记录的字节数也计算到表尾偏移量中
        // 这样才能让表尾偏移量正确对齐表尾节点
        tail = zipEntry(p+reqlen);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        // 新元素是新的表尾节点
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }
    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    // 当 nextdiff != 0 时，新节点的后继节点的（header 部分）长度已经被改变，
    // 所以需要级联地更新后续的节点
    if (nextdiff != 0) {
        offset = p-zl;
        // T  = O(N^2)
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }
    /* Write the entry */
    // 一切搞定，将前置节点的长度写入新节点的 header
    p += zipPrevEncodeLength(p,prevlen);
    // 将节点值的长度写入新节点的 header
    p += zipEncodeLength(p,encoding,slen);
    // 写入节点值
    if (ZIP_IS_STR(encoding)) {
        // T = O(N)
        memcpy(p,s,slen);
    } else {
        // T = O(1)
        zipSaveInteger(p,value,encoding);
    }
    // 更新列表的节点数量计数器
    // T = O(1)
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

在更新后续节点的 previous_entry_length 的过程中，可能会发生如下一种情况：当前的节点原先的长度小于 254，而新插入的节点长度大于 254，那么 encoding 的长度变长了，这样，若后续节点是一系列长度为 250 - 254，那么需要连续地更新后续节点的 header（即计算节点长度、重新分配空间然后后移后续节点插入新的 encoding 字段，直到整个压缩列表满足条件）。这个过程是在函数 __ziplistCascadeUpdate 中完成的：

```C
static unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), rawlen, rawlensize;
    size_t offset, noffset, extra;
    unsigned char *np;
    zlentry cur, next;
    while (p[0] != ZIP_END) {
        // 将 p 所指向的节点的信息保存到 cur 结构中
        cur = zipEntry(p);
        // 当前节点的长度
        rawlen = cur.headersize + cur.len;
        // 计算编码当前节点的长度所需的字节数
        // T = O(1)
        rawlensize = zipPrevEncodeLength(NULL,rawlen);
        /* Abort if there is no next entry. */
        // 如果已经没有后续空间需要更新了，跳出
        if (p[rawlen] == ZIP_END) break;
        // 取出后续节点的信息，保存到 next 结构中
        next = zipEntry(p+rawlen);
        // 后续节点编码当前节点的空间已经足够，无须再进行任何处理，跳出
        // 可以证明，只要遇到一个空间足够的节点，
        // 那么这个节点之后的所有节点的空间都是足够的
        if (next.prevrawlen == rawlen) break;
        if (next.prevrawlensize < rawlensize) {
            // 执行到这里，表示 next 空间的大小不足以编码 cur 的长度
            // 所以程序需要对 next 节点的（header 部分）空间进行扩展
            // 记录 p 的偏移量
            offset = p-zl;
            // 计算需要增加的节点数量
            extra = rawlensize-next.prevrawlensize;
            // 扩展 zl 的大小
            zl = ziplistResize(zl,curlen+extra);
            // 还原指针 p
            p = zl+offset;
            /* Current pointer and offset for next element. */
            // 记录下一节点的偏移量
            np = p+rawlen;
            noffset = np-zl;
            // 当 next 节点不是表尾节点时，更新列表到表尾节点的偏移量
            if ((zl+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))) != np) {
                ZIPLIST_TAIL_OFFSET(zl) =
                    intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
            }
            // 向后移动 cur 节点之后的数据，为 cur 的新 header 腾出空间
            memmove(np+rawlensize,
                np+next.prevrawlensize,
                curlen-noffset-next.prevrawlensize-1);
            // 将新的前一节点长度值编码进新的 next 节点的 header
            zipPrevEncodeLength(np,rawlen);
            // 移动指针，继续处理下个节点
            p += rawlen;
            curlen += extra;
        } else {
            if (next.prevrawlensize > rawlensize) {
                /* This would result in shrinking, which we want to avoid.
                 * So, set "rawlen" in the available bytes. */
                // 执行到这里，说明 next 节点编码前置节点的 header 空间有 5 字节
                // 而编码 rawlen 只需要 1 字节
                // 但是程序不会对 next 进行缩小，
                // 所以这里只将 rawlen 写入 5 字节的 header 中就算了。
                // T = O(1)
                zipPrevEncodeLengthForceLarge(p+rawlen,rawlen);
            } else {
                // 运行到这里，说明 cur 节点的长度正好可以编码到 next 节点的 header 中
                zipPrevEncodeLength(p+rawlen,rawlen);
            }
            break;
        }
    }
    return zl;
}
```
不仅是插入有可能导致这样的连锁更新，当删除或改变一个节点的内容时，也可能发生这样的连锁更新。这样的连锁更新是非常耗费时间的（最差的情况时间复杂度为 O(N)），但是只有碰到连续长度为 250 - 254 的节点串时，才会触发该操作，当遍历到第一个满足条件的节点以后，操作就停止了。

值得注意的是，由于压缩列表的各项操作都需要遍历整个列表才能完成，因此，操作的时间复杂度都为 O(N)，这样使得在使用压缩列表时，表中的节点数不能太多，否则将极度影响操作的效率。在后续的数据结构介绍过程中，我们可以看到 Redis 对于压缩列表的使用是有限制的，使得既能利用压缩列表节省空间的特性，也能通过其它的设计或约束降低操作的时间复杂度。
