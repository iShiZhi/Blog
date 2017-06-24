---
title: 'Redis数据结构: 简单动态字符串'
categories: 开源项目
tags: [Redis]
date: 2016-10-14
---

Redis 是用 C 语言编写的，而 C 语言用空字符（’\0’）结尾的字符数组表示字符串的方式又备受诟病。因此，Redis 的作者自己设计了一个新的结构，命名为简单动态字符串（Simple Dynamic String, SDS）。在 Redis 内部绝大部分需要使用字符串的地方，使用的都是 SDS，除了一些不需要改变的字面量以外，例如打印日志。

Redis 是一个 key-value 类型的数据库，所有的 key 都是字符串。而在 Redis 的各种操作中，会频繁地使用字符串的长度查询和增加字符两种操作。对于原生的 C 字符串而言，长度查询的时间复杂度是 O(N)，而多次的 append 操作则需要频繁地调用 malloc 和 realloc 操作，这些操作的耗时较多。因此，优化这两个操作势在必行。此外，为了在客户端和服务器之间传输，并且可以保存除了文本数据以外的数据格式，因此字符串必须是二进制安全的。这些就是设计 SDS 要考虑的基本因素。由于在具体实现的过程中，使用了一些关于 C 语言的特性，因此我们对照着源代码进行解释。

### SDS 结构

在 sds.h 中可以看到 SDS 的定义，我们可以看到 sds 对应的是字符串数组（与 C 字符串定义一致）：

```C
typedef char *sds;
```
其后定义了五个 struct（sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64），它们的结构一样（除了 sdshdr5 以外），包含了四个对应的属性，分别是：

- len：SDS 保存的字符串的实际长度，不包含最后的空字符
- alloc：SDS 用于保存字符串的所有空间的大小，不包括空字符以及 header
- flags：低 3 位用于保存字符串对应的 struct，高 5 位只有 sdshdr5 用于保存字符串长度，其它的 struct 没有用到
- buf：指向保存的字符串的指针，上面定义的 sds 实际上就是这里的 buf

上述除了 buf 以外的三个字段组成了 SDS 的 header，保存了除了字符串内容以外的信息，用于快速实现一些操作，提高字符串操作的效率。
注意：sdshdr5 的结构有所不同，它没有 len 和 alloc 两个字段，字符串长度保存在 flags 的高 5 位中，所以最大的值为 31 ，这样的字符串适合于保存较短的不可变字符串。

```C
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

与 flags 相关的常量如下：

```C
/* sds 的类型常量，类型存于 flags 的最低 3 位 */
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
/* SDS_TYPE_MASK 用来与 flags 配合，获得 sds 的类型，注意 7 的二进制表示是 111，与它进行 & 操作以后能获得最低 3 位的值 */
#define SDS_TYPE_MASK 7
/* 与 sdshdr5 配合使用，flags 的最低 3 位存储类型，高位用于长度 */
#define SDS_TYPE_BITS 3
```

### 使用的 C 语言特性

在上述 sdshdr 的定义中，用到了 C 语言的几个特性，从而使 SDS 变得比 C 字符串更加灵活：

- 柔性数组（flexible array member）：sdshdr 的 buf 属性并没有指明数组的尺寸，被称为柔性数组，它只是作为一个标记存在在 struct 中，在 sizeof 操作的时候并不计入
- Struct Hack：Struct 中有且仅有一个变长的字段，且该字段是 Struct 中最后的一个字段，这样在分配空间的时候，可以直接给这个变长字段分配空间，使得该变长字段的内容和 struct 中其它的字段放在连续的空间中，直接通过该变长变量的指针就可以获得其它的字段。在 sdshdr 的定义中，使用了 Struct Hack，使得只要知道 buf 字段的起始位置指针，就可以推导获得 header 字段的起始位置指针
- attribute ((packed))：它的作用就是告诉编译器取消结构在编译过程中的优化对齐，按照实际占用字节数进行对齐，是 GCC 特有的语法。这个功能是跟操作系统没关系，跟编译器有关，GCC 编译器不是紧凑模式的。在 Windows 下，用 VC 的编译器也不是紧凑的，用 TC 的编译器就是紧凑的。例如：
  - 在 TC 下：struct my{ char ch; int a;} sizeof(int) = 2; sizeof(my) = 3;（紧凑模式）
  - 在 GCC 下：struct my{ char ch; int a;} sizeof(int) = 4; sizeof(my) = 8;（非紧凑模式）
  - 在 GCC 下：struct my{ char ch; int a;}attrubte ((packed)) sizeof(int) = 4; sizeof(my) = 5;（紧凑模式）

从上述的解释中，我们可以得知 SDS 设计的主要内容：

- 通过使用 Struct Hack，我们可以将 SDS 的 header 和对应的字符串分配在连续的空间中，然后仅通过字符串指针就可以找到对应的 header
- 在使用的过程中，直接使用 sds 类型，这个与 C 字符串的使用无异，可以方便的使用 str 的各种库函数，需要 header 的信息的时候，使用额外的操作，找到对应的辅助信息

### 获取 header 的操作

由于不同的 sdshdr struct 存储 len 和 alloc 的变量类型不同，所以需要先获取类型的信息，而类型存储在 flags 中的第三位，flags 又固定存储在字符串第一个字符的前一个字节，因此，通过 s[-1] & SDS_TYPE_MASK 就可以获得类型的信息，而后通过指针减去 header 的 size，就可以获得 header 的指针，具体如下：

```C
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```

在获得 header 的指针以后，就得到了对应的 struct，就可以取出对应的字段，以 len 为例：

```C
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```
### 创建与销毁

```C
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
#endif
    return SDS_TYPE_64;
}
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this.
     * 如果是空字符串，则使用 SDS_TYPE_8
     */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    // 使用 Struct Hack，同时分配 header 和 buf 的空间，最后加上的 1 是给空字符留的空间
    sh = s_malloc(hdrlen+initlen+1);
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;

    // 找到 buf 的起始位置，以及 flags 的位置
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    // 设置 header
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }

    // 拷贝存储在 init 中的字符串
    if (initlen && init)
        memcpy(s, init, initlen);

    // 在最后加上 空字符
    s[initlen] = '\0';
    // 返回 buf 的指针
    return s;
}
sds sdsempty(void) {
    return sdsnewlen("",0);
}
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
sds sdsdup(const sds s) {
    return sdsnewlen(s, sdslen(s));
}
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
无论是何种方式进行创建，底层都是通过调用 sdsnewlen 函数来完成的，值得注意的是：

- 通过对于初始字符串的长度判断，可以获得对应的 struct 类型，这个是创建 SDS header 的基础
- 若是空字符串，将使用 sdshdr8 而不是 sdshdr5 来进行创建，因为 sdshdr5 里没有活动的可用空间，不利于后续的拓展
- 与 C 字符串一样，SDS 会在创建的新字符串的末尾自动地添加空字符（’\0’）

由于添加了空字符，因此，SDS 在将字符串进行截短或者是清除的时候，只需要调整 header 中的信息以及空字符的位置，而不用处理原先字符串的内容：

```C
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}
```

### 连接（追加）字符串操作

```C
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    /* Return ASAP if there is enough space left.
     * 如果已经有足够空间了，直接返回
     */
    if (avail >= addlen) return s;
    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    // 设定新的长度，以及对应的类型
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
    type = sdsReqType(newlen);
    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation.
     * SDS_TYPE_5 没有办法保存空闲的空间长度，所以在拓展的时候避免使用它
     */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc
         * 当长度改变以后，没有办法直接使用 realloc，因为 header 的长度也发生变化了
         */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}
static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                /* 直接将长度设置到 flags 字段的高 5 位 */
                unsigned char *fp = ((unsigned char*)s)-1;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len = newlen;
            break;
    }
}
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}
sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}
```
可以看到，sdscat 操作底层是通过 sdscatLen 实现的，它主要分为如下几个步骤：

- 检查剩余的空间是否足够，如果不够的话需要分配新的空间，在 sdsMakeRoomFor 函数里完成
- 将需要追加的字符串拷贝入新的空间中
- 更新字符串长度

值得注意的是，在追加了新的字符串以后，SDS header的长度可能发生变化，例如 sdshdr8 类型的 SDS 追加了字符串以后变成了 sdshdr16 类型，这时候 header 增加了 2 个字节，这时候需要使用 realloc 重新分配空间，SDS 的指针就会随之发生变化。所以，所有改变 SDS 内容的操作都需要注意，操作前的 SDS 指针在操作完成以后不一定可用，需要使用函数返回的指针。

### 回收空闲空间

```C
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```
即使 SDS 发生截短操作，也并不会立刻回收空间，而是留待下一次可能的连接（追加）操作使用，除非主动调用 sdsRemoveFreeSpace 函数。

### 二进制安全

由于 SDS header 中储存着字符串的长度，所以即使在字符串中出现了空字符（’\0’），Redis 定义的函数也可以正确地进行处理。而这些带有空字符的字符串在使用 C 语言的字符串函数的时候需要特别注意，有可能产生错误。因此，需要 SDS 的使用者自己对内容进行分辨和正确地使用。
