---
title: CMU 15-213 Part10 Dynamic Memory Allocation
date: "2020/2/12"
categories:
- CMU 15-213
---

### 分配器的测量指标

分配器有**分配(Allocate)**和**释放(Free)**两种操作。给定一系列操作R<sub>0</sub>, R<sub>1</sub>, ..., R<sub>n-1</sub>，我们使用**吞吐量()**和**内存利用率()**来评价分配器。我们的目标是尽可能提高吞吐量和内存利用率，但这两个指标是相互矛盾，无法兼顾的。我们将吞吐量定义单位时间能够处理操作的数目。比如某分配器在10s处理了5000个malloc和5000个free，则吞吐量为10000操作/s。而内存利用率则有很多定义方法，其中一个是峰值利用率。峰值利用率是一系列操作后，内存利用率(有效负载/可用内存)达到的最大值。

下面介绍3种类型的分配器。注意分配器必须返回对齐的地址，下面的分配器以返回8字节对齐的地址为例。

### 隐式列表分配器(Implict List Allocator)

分配器将可用内存划分为一个个内存块，有些已经分配给用户程序，有些是空闲的。内存块在可用内存中紧挨着排列。每个内存块返回有效负载的首地址给用户程序，但是除了有效负载，内存块还需要空间保存块的信息，比如块大小，分配状态等。由于分配的内存块大小是以8字节对齐，所以表示大小的字节最低3位总是全0，那么使用这3位来存储分配状态。块信息一般放在块头和块尾，中间是有效负载，必要时还需要填充额外字节来达到对齐的要求。”隐式列表“是指将块大小作为偏移计算下一个内存块的位置，从而能够遍历所有内存块。放在块尾的块信息是为了下一级块能够找到前一个内存头部。列表尾部是一个标记为已分配的空块，作为搜索停止的哨兵。

![隐式列表分配器的内存块](https://i.loli.net/2020/02/10/YAHCro9Zkapn7fw.png)

![内存块组成隐式列表](https://i.loli.net/2020/02/10/kcRwmLQZKWTaiD2.png)

* 分配操作：在内存块中搜索符合条件的闲置内存块。可以采用不同的匹配策略，比如首次匹配，返回第一个符合条件的闲置块(>=请求大小)，或者最佳匹配，返回最符合条件的闲置块(大于请求大小的闲置块中，返回最小的)。如果闲置块大于请求大小，返回前需要分割闲置块。所以首次匹配存在内存利用率低的问题，因为会在内存中留下不连续的内存碎片。最佳匹配则需要遍历整个内存块列表，导致吞吐率低。

* 分割操作(Splitting)：分配操作找到合适的闲置块后，如果闲置块太大，返回前需要分割闲置块，将剩余部分作为一个新的闲置块放到列表中。但注意内存块有最小大小，比如在这个例子中，内存块至少要能够容纳头尾两个块信息。

* 释放操作：释放内存时，更改块信息中的分配状态。同时，检查释放块前后有无闲置块，如果有则进行合并操作。但是合并操作的时机是可选的，可以在释放时合并，也可以选择在分配搜索时合并。分配器需要维护的一个不变式是：内存中不存在两个连续的闲置块。该不变式保证了在请求一片大内存时，内存中不会出现尽管存在符合要求的连续闲置空间，但找不到符合要求的闲置块这种情况。

* 合并操作(Coalesce)：合并操作将两个及以上的连续闲置块合并成一个。此时内存块尾部保存的块信息起到了作用，访问被释放块前一个字节，就可以检查前一个块的分配情况。


### 显式列表分配器(Explicit List Allocator)

隐式列表分配器的问题在于搜索闲置内存块时会经过许多已分配的内存块，浪费时间。显式列表分配器基于隐式列表分配器进行改进。虽然分配器不允许更改已分配内存块的有效负载，但是闲置内存块的负载部分不受此限制。显式列表分配器维护一个闲置内存块的双向链表，链表指针放在闲置内存块的负载部分。这样改进会影响分配器允许的最小内存块大小，除了头尾两个块信息外，现在还需要容纳两个指针的空间。

![显式列表分配器](https://i.loli.net/2020/02/10/FZ1YuSoLs8cDAJd.png)

* 分配操作：除了上面描述的分配步骤外，还需要将分配块从闲置链表中删除。

![显式列表分配内存块](https://i.loli.net/2020/02/10/pGwfnlkWgHVuvDh.png)

* 释放操作：把释放的内存块插入闲置链表的头部。

![显式列表释放内存块](https://i.loli.net/2020/02/10/tNOmhAioPEyWg4Q.png)

### 多列表分配器(Segregated List Allocator)

如果实现显式列表分配器时为了性能采取首次匹配，那么内存利用率将会降低。多列表分配器维护多个链表，每个链表保存某个尺寸范围的闲置内存块。每次分配时，到对应的链表中搜索合适的内存块，可以使找到的内存块更加匹配，提高内存利用率。

![多列表分配器](https://i.loli.net/2020/02/10/TOvik3UNWJaVrjl.png)


### 附：Malloc Lab

课上给Malloc Lab的建议

* 使用宏/内联函数，比如用宏定义对内存块头尾部的操作

* 指针运算，`p+4`的值为地址p偏移`4*sizeof(p)`

* 不变式检查器，只在错误时输出信息(verbose选项)

* 使用valgrind检查内存泄漏，使用gprof测量性能

```cpp
#ifdef DEBUG
#define CHECKHEAP(lineno) printf("%s\n", __func__); mm_checkheap(__LINE__);
#endif
```

* 检查链表环。