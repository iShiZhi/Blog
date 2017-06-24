---
title: 'Redis数据结构: 双端链表'
categories: 开源项目
tags: [Redis]
date: 2016-10-15
---

链表是一种常用的数据结构，在很多的编程语言，例如 C++ 里都有现成的容器可以使用，除了 Redis 使用的 C 语言。因此，在 Redis 中，作者自己实现了一个双端链表的结构，在这个结构中，保存了头节点，尾节点和长度，这样可以很方便地从两端进行链表的遍历，也可以用 O(1) 的时间复杂度获得链表的长度。

链表的结构在 adlist.h 中：

```C
typedef struct listNode {
    struct listNode *prev;    // 前置节点
    struct listNode *next;    // 后继节点
    void *value;              // 节点的值
} listNode;
typedef struct list {
    listNode *head;           // 表头节点
    listNode *tail;           // 表尾节点
    void *(*dup)(void *ptr);  // 节点值复制函数
    void (*free)(void *ptr);  // 节点值释放函数
    int (*match)(void *ptr, void *key);  // 节点值对比函数
    unsigned long len;        // 链表所包含的节点数量
} list;
```

在上述的结构中，链表的值使用 void* 指针来保存节点的值，而节点的值通过三个属性保存的函数来进行节点值的复制、比较和释放，这样实现了链表的多态。
另外，需要注意的是，表头节点的前置节点和表尾节点的后继节点都指向 NULL，意味着这个链表是无环的。

由于双端链表可以由两个方向进行遍历，所以实现的迭代器中也包含了方向：

```C
// 从表头向表尾进行迭代
#define AL_START_HEAD 0
// 从表尾到表头进行迭代
#define AL_START_TAIL 1
typedef struct listIter {
    listNode *next;           // 当前迭代到的节点
    int direction;            // 迭代的方向
} listIter;
```
由于这个结构比较简单，可以直接阅读 adlist.h 和 adlist.c 中的源代码，不再赘述。
