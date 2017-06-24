---
title: Ruby 垃圾回收机制的演变过程
categories: 开发语言
tags: [Ruby on Rails, Garbage Collection]
date: 2016-09-04
---

垃圾回收（英语：Garbage Collection，缩写为 GC）是一种自动的内存管理机制。当一个电脑上的动态内存不再需要时，就应该予以释放，以让出内存，这种内存资源管理，称为垃圾回收（garbage collection）。垃圾回收最早起源于 LISP 语言。

## 基本算法

### 引用计数器 Reference Counting

对数据存储的物理空间附加多一个计数器空间，当有其他数据与其相关时则加一，反之相关解除时减一，定期检查各储存对象的计数器，为零的话则认为已经被抛弃而将其所占物理空间回收。这是最简单，也是最早在 LISP 实现的垃圾回收方式，其缺点是无法回收循环引用的存储对象。

<img src="http://ocx5ae9jo.bkt.clouddn.com/reference-counting.jpeg" width="300px" height="300px">

### 跟踪收集器 Tracing

注意到，被回收的对象不再被应用程序使用，即从应用程序的根节点出发，沿着引用查找，将不会再查找到这些需要被回收的对象。因此，可以通过定期对若干根储存对象开始遍历，对整个程序所拥有的储存空间查找与之相关的存储对象和没相关的存储对象进行标记，然后将没相关的存储对象所占物理空间回收。根据标记和回收算法实现的不同，可以将这类跟踪收集的垃圾回收算法分为若干类。

### 标记-清除 Mark and Sweep

先暂停整个程序的全部运行线程（Stop the word），让回收线程以单线程进行扫描标记，并进行直接清除回收，然后回收完成，恢复运行线程。这个算法的缺点是会导致大量零碎的空闲空间碎片，使得大容量对象不容易获得连续的内存空间，而造成空间浪费。

<img src="http://ocx5ae9jo.bkt.clouddn.com/mark-and-sweep.jpeg" width="450px" height="300px">

### 标记-复制 Mark and Copy

需要程序将所拥有的内存空间分成两个部分。程序运行所需的存储对象先存储在其中一个分区（定义为“分区0”）。同样暂停整个程序的全部运行线程后，进行标记后，回收期间将将保留的存储对象搬运汇集到另一个分区（定义为“分区1”），完成回收，程序在本次回收后将接下来产生的存储对象会存储到“分区1”。在下一次回收时，两个分区的角色对调。

<img src="http://ocx5ae9jo.bkt.clouddn.com/mark-and-copy1.jpeg" width="400px" height="150px">
<img src="http://ocx5ae9jo.bkt.clouddn.com/mark-and-copy2.jpeg" width="400px" height="150px">

### 标记-压缩 Mark and Compact

和“标记－清除”相似，不同的是，在回收期间同时会将保留的存储对象搬运汇集到连续的内存空间（例如从小端地址开始），从而集成空闲空间。这样既不会像“标记-清除”那样产生内存碎片，也不会像“标记-复制”那样只能使用一半的空间。

### 增量回收 Incremental Collection

使用这种回收机制时，程序会将所拥有的内存空间分成若干分区。程序运行所需的存储对象会分布在这些分区中，每次只对其中一个分区进行回收操作，从而避免程序全部运行线程暂停来进行回收，允许部分线程在不影响回收行为而保持运行，并且降低回收时间，增加程序响应速度。

### 分代回收 Generational Collection

不同的储存对象存活的时间差异很大，而不同存活时间的对象回收的特别也不一样：

- 存活时间长，大容量的储存对象在“复制”算法上需要耗费更多的移动时间
- 存活时间短的储存对象通常在特定的作用域中，且容量小，适合被快速回收

<img src="http://ocx5ae9jo.bkt.clouddn.com/object-lifecycle.jpeg" width="500px" height="250px">

因此，可以将程序拥有的内存空间划分区域，并标记为年轻代空间和年老代空间。程序运行所需的存储对象会先存放在年轻代分区，年轻代分区会较为频密进行较为激进垃圾回收行为，每次回收完成幸存的存储对象内的寿命计数器加一。当年轻代分区存储对象的寿命计数器达到一定阈值或存储对象的占用空间超过一定阈值时，则被移动到年老代空间，年老代空间会较少运行垃圾回收行为。一般情况下，还有永久代的空间，用于涉及程序整个运行生命周期的对象存储，例如运行代码、数据常量等，该空间通常不进行垃圾回收的操作。

## Ruby 1.8 最基本的“标记-清除”

Ruby 1.8里实现的就是1960年提出来的那个最简单的“标记-清除”算法，在整个 gc 的过程中，程序运行的所有线程都停机。

<img src="http://ocx5ae9jo.bkt.clouddn.com/ruby-1-8.jpeg" width="600px" height="150px">

## Ruby 1.9.3 Lazy sweep

Ruby 1.9.3实现了增量回收机制的一半，它的基本过程是：

- 暂停整个程序的全部运行线程，让回收线程以单线程进行扫描标记
- 进行清除回收，直到找到一个可以使用的空间就停止清除回收过程

<img src="http://ocx5ae9jo.bkt.clouddn.com/ruby-1-9-3.jpeg" width="600px" height="100px">

## Ruby 2.0 加入针对 Copy-On-Write 的优化

### Ruby的数据结构和 GC 过程

我们先来看一下 Ruby 存储 String 的数据结构 RString，可以看到，除了一个指向该 String 具体值的指针以外，还包含了一个特殊的结构——RBasic，它保存了大量的标记位：

```C
struct RString {
  struct RBasic basic;
  union {
    struct {
      long len;
      char *ptr;
      union {
        long capa;
        VALUE shared;
      } aux;
    } heap;
    char ary[RSTRING_EMBED_LEN_MAX + 1];
  } as;
};
```

``` C
 290  struct RBasic {
 291      unsigned long flags;
 292      VALUE klass;
 293  };

(ruby.h)
```
Ruby 程序中还有大量类似的结构，例如 RArray、RHash、RFile等等，它们都是由一些数据和一些标记位组成，统称为 RValue（Ruby Value）。在这些标记位中，有一个特殊的标记位 FL_MARK（包含在RBasic），它用于在 GC 过程中表示该结构是否需要被清除。因此，整个 GC 过程可以描述为：

- 检查对象，标记 FL_MARK
- 将所有可用的对象放入一个链表中，以待使用

<img src="http://ocx5ae9jo.bkt.clouddn.com/gc_fl_mark.jpeg" with="600px" height="200px">

### Copy-On-Write 机制以及产生的 GC 问题

众所周知，在基于 Unix 结构的系统中，有一个被称为“Copy-On-Write”的机制：当我们调用 fork 在父进程中创建子进程的时候，此时，子进程共享父进程的所有储存空间，包括数据、变量等等。这样可以使得 fork 命令执行得更快，需要的储存空间更少。当某个子进程需要修改共享的储存空间时，系统才会将这个存储空间进行复制，然后修改对应的数据。而 Ruby 在处理使用的数据结构时，也用了类似的方案，在不同的线程使用相同的数据时，会先使用同一个数据备份，直到其中某个线程需要修改。

<img src="http://ocx5ae9jo.bkt.clouddn.com/ruby-share-memory.jpeg" width="500px" height="500px">

但是，由此产生了一个 GC 问题，在 GC 的标记阶段，会扫描所有的 RValue 结构，给 FL_MARK 进行赋值，此时对于共享的数据标记位进行的修改引发了拷贝。这样会大大降低 GC 的效率和存储空间的使用效率。

<img src="http://ocx5ae9jo.bkt.clouddn.com/ruby-share_memory2.jpeg" width="500px" height="500px">

### Bitmap Marking

在 Ruby2.0 中，FL_MARK 不在存储在 RValue 结构体中，而是存在 bitmap 中。对于每一个堆，都会有一个对应的 bitmap，用来存储堆中结构体的 FL_MARK 标记位。

<img src="http://ocx5ae9jo.bkt.clouddn.com/gc-bitmap1.jpeg" width="700px" height="180px">

通过这样的方式，优化了 Copy-On-Write 机制对于 GC 过程的影响，提高了空间的使用率和 GC 的时间复杂度。

## Ruby 2.1 Generational GC

### 基本实现

Ruby2.1 引入了分代回收的机制，它将 Object 分成 Yong 和 Mature 两类。新建的 Object 都是 Yong Object，当发生若干次垃圾回收（默认3次）以后，留存下来的 Object 都会变成 Mature Object。在每次发生 minior GC 时，只有 Yong Object 会被回收，而 Mature Object 只会在 full GC 中被回收。

<img src="http://ocx5ae9jo.bkt.clouddn.com/generational-gc1.jpeg" width="400px" height="300px">

注意，在 Ruby2.1 中实现的分代回收并没有真的将物理内存分为 Yong Object 和 Mature Object 两个区域，而是在实现的时候通过一些标记位（oldgen bits）和算法来保证被标记成 Mature Object 的结构不会再被 Mark and Swap。

### Write Barriers

在上面的实现中，遗留了如下一个问题：当一个新对象被创建以后，并被关联到一个 Mature Object上，这时候发生了 GC，这个对象是否被回收？

<img src="http://ocx5ae9jo.bkt.clouddn.com/generational-gc2.jpeg" width="400px" height="300px">

为了解决这个问题，Ruby2.1 使用了一种称为 Write Barriers 的技术。它监控所有的 Mature Object，一旦发生从 Mature Object 到 Yong Object 的引用，则会对这些 Mature Object 进行记录，并在下次的 Mark and Sweep 中使用。事实上，为了支持 用 C 编写的扩展，Ruby2.1 对 Write Barriers的实现比较复杂，并非所有堆中的对象都被保护，而这些没有被保护的对象将不会被提升为 Muture Object，例如 Proc、Ruby::Env等，而被 C 编写的扩展访问的对象也是不安全的，也不会被提升。这些对象使用 remembered set 来加速GC。

## Ruby 2.2 Incremental GC & Symbol GC

### Incremental GC

首先，先介绍三个术语：

- white object：没有被标记过的 object
- grey object：被标记过 object，且引用了 white object
- black object：被标记过 object，且没有引用任何 white object

使用上述的术语，Mark and Sweep 的过程可以描述为：

- 将所有的 object 标记为 white object；
- 将程序使用的 object 标记为 gray object；
- 选择一个 gray object，将它引用的所有的 white object 标记为 gray，而将这个 gray object 标记为 black，重复这个过程，直到所有的 object 都不是 gray object；
- 回收所有的 white object。

如果将上述的 Mark 过程变为增量式的，那么会面临一个问题：在增量 Mark 的过程中，随着程序的执行，一些 black object 会产生对 white object 的引用，而这样与 black object 的定义产生冲突。为了解决这个问题，我们还是使用 Write Barriers 技术。当产生从 black object 到 white object 的引用时，这个 black object 将会自动地被变为 gray object。注意到，由于一些 object 没有办法使用 Write Barriers 进行保护，因此需要对这些 object 进行特殊处理。上述的 Mark and Sweep 过程调整为：

- 将所有的 object 标记为 white object；
- 将程序使用的 object 标记为 gray object；
- 选择一个 gray object，将它引用的所有的 white object 标记为 gray，而将这个 gray object 标记为 black，重复这个过程，直到所有的 object 都不是 gray object，这个过程增量式地进行；
- 遍历所有没有被 Write Barriers 保护的 object，判断它们是否需要被回收；
- 回收所有的 white object。

<img src="http://ocx5ae9jo.bkt.clouddn.com/ruby-2-2.jpeg" width="800px" height="150px">

### Symbol GC

在 Ruby2.2 以前，程序中的 symbol 是不会被 GC 的，例如：

```Ruby
before = Symbol.all_symbols.size
100_000.times do |i|
  "sym#{i}".to_sym
end
GC.start
after = Symbol.all_symbols.size
puts after - before
# => 100001
```

而如果我们直接对 url 的参数调用 to_sym 获取 symbol，则会导致”symbol DoS”。

```Ruby
def show
  step = params[:step].to_sym
end
```
为了支持用 C 编写的拓展，在 Ruby2.2 以前，所有的 symbol 对应一个唯一的 objectID，而由于rb_intern的实现方式，这个 objectID 必须在整个程序的生命周期里保持一致。在 C-Ruby 里创建一个方法时，会同时在方法表中添加一个唯一的 ID 来对应这个方法。在随后的调用过程中，会到方法列表中去寻找 ID 对应的静态内存，如果对应到这个方法的 symbol 被 GC 回收掉，则这个方法就再也不会被调用。
在 Ruby2.2 中，将所有的 symbol 分为两类：

在 Ruby 运行过程中动态创建的 symbol（例如通过to_sym创建的），这类的 symbol 将会被 GC
在 Ruby 解释器里生成的 symbol，或程序里静态的 symbol，这些将不会被 GC
但是，需要注意两点：

- 这样做依然存在一些风险，例如在代码中使用 define_method 来定义函数（它将会调用rb_intern），则这个 symbol 将会被识别为第一类 symbol，从而被 GC

```Ruby
define_method(params[:step].to_sym) do
  # ...
end
```
- 将 symbol 转化为 string 会耗费大量的时间

## 参考资料

- [Ruby 1.9.3 Preview 1 Released, Improves GC Pauses With Lazy Sweep GC](http://www.infoq.com/news/2011/08/ruby193-gc)
- [Never create Ruby strings longer than 23 characters](http://patshaughnessy.net/2012/1/4/never-create-ruby-strings-longer-than-23-characters)
- [Ruby Hacking Guide Chapter 2: Objects](https://ruby-hacking-guide.github.io/object.html)
- [Why You Should Be Excited About Garbage Collection in Ruby 2.0](http://patshaughnessy.net/2012/3/23/why-you-should-be-excited-about-garbage-collection-in-ruby-2-0)
- [Ruby 2.1: RGenGC](http://tmm1.net/ruby21-rgengc/)
- [Generational GC in Python and Ruby](http://patshaughnessy.net/2013/10/30/generational-gc-in-python-and-ruby)
- [Incremental Garbage Collection in Ruby 2.2](https://engineering.heroku.com/blogs/2015-02-04-incremental-gc/)
- [Symbol GC in Ruby 2.2](https://www.sitepoint.com/symbol-gc-ruby-2-2/)
