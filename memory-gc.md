# 内存的由来

1. 如果CPU直接访问硬盘，CPU能充分利用吗？
2. 如果CPU直接访问内存，CPU能充分利用吗？

CPU速度很快，但硬盘等持久存储很慢，如果CPU直接访问磁盘，磁盘可以拉低CPU的速度，机器整体性能就会低下，为了弥补这2个硬件之间的速率差异，所以在CPU和磁盘之间增加了比磁盘快很多的内存。

![image-20201026215910514](memory-gc.assets/image-20201026215910514.png)

然而，CPU跟内存的速率也不是相同的，从上图可以看到，CPU的速率提高的很快（摩尔定律），然而内存速率增长的很慢，**虽然CPU的速率现在增加的很慢了，但是内存的速率也没增加多少，速率差距很大**，从1980年开始CPU和内存速率差距在不断拉大，为了弥补这2个硬件之间的速率差异，所以在CPU跟内存之间增加了比内存更快的Cache，Cache是内存数据的缓存，可以降低CPU访问内存的时间.

![image-20201026220157384](memory-gc.assets/image-20201026220157384.png)

三级Cache分别是L1、L2、L3，它们的速率是三个不同的层级，L1速率最快，与CPU速率最接近，是RAM速率的100倍，L2速率就降到了RAM的25倍，L3的速率更靠近RAM的速率。



# 虚拟内存

![image-20201026220316478](memory-gc.assets/image-20201026220316478.png)



上图展示了某进程访问数据，当Cache没有命中的时候，访问虚拟内存获取数据的过程。

访问内存，实际访问的是虚拟内存，虚拟内存通过页表查看，当前要访问的虚拟内存地址，是否已经加载到了物理内存，如果已经在物理内存，则取物理内存数据，如果没有对应的物理内存，则从磁盘加载数据到物理内存，并把物理内存地址和虚拟内存地址更新到页表。

**物理内存就是磁盘存储缓存层**。

另外，在没有虚拟内存的时代，物理内存对所有进程是共享的，多进程同时访问同一个物理内存存在并发访问问题。**引入虚拟内存后，每个进程都要各自的虚拟内存，内存的并发访问问题的粒度从多进程级别，可以降低到多线程级别**。

# 堆和栈

![image-20201026220613034](memory-gc.assets/image-20201026220613034.png)

上图展示了一个进程的虚拟内存划分，代码中使用的内存地址都是虚拟内存地址，而不是实际的物理内存地址。栈和堆只是虚拟内存上2块不同功能的内存区域：

- **栈在高地址，从高地址向低地址增长。**
- **堆在低地址，从低地址向高地址增长。**

**栈和堆相比有这么几个好处**：

1. <font color=red size=5x>栈的内存管理简单，分配比堆上快。</font>
2. <font color=blue size=5x>栈的内存不需要回收，而堆需要，无论是主动free，还是被动的垃圾回收，这都需要花费额外的CPU。</font>
3. <font color=red size=5x>栈上的内存有更好的局部性，堆上内存访问就不那么友好了，CPU访问的2块数据可能在不同的页上，CPU访问数据的时间可能就上去了。</font>



# 堆内存管理

![image-20201026220850546](memory-gc.assets/image-20201026220850546.png)

当我们说内存管理的时候，主要是指堆内存的管理，因为栈的内存管理不需要程序去操心。这小节看下堆内存管理干的是啥，如上图所示主要是3部分：**分配内存块，回收内存块和组织内存块**。

在一个最简单的内存管理中，堆内存最初会是一个完整的大块，即未分配内存，当来申请的时候，就会从未分配内存，分割出一个小内存块(block)，然后用链表把所有内存块连接起来。需要一些信息描述每个内存块的基本信息，比如大小(size)、是否使用中(used)和下一个内存块的地址(next)，内存块实际数据存储在data中。

![image-20201026220948366](memory-gc.assets/image-20201026220948366.png)



一个内存块包含了3类信息，如下图所示，元数据、用户数据和对齐字段，内存对齐是为了提高访问效率。下图申请5Byte内存的时候，就需要进行内存对齐。

go中map的实现也是应用到了内存对齐

![image-20201026221027195](memory-gc.assets/image-20201026221027195.png)

==释放内存实质是把使用的内存块从链表中取出来，然后标记为未使用，当分配内存块的时候，可以从未使用内存块中有先查找大小相近的内存块，如果找不到，再从未分配的内存中分配内存。==

上面这个简单的设计中还没考虑内存碎片的问题，因为随着内存不断的申请和释放，内存上会存在大量的碎片，降低内存的使用率。为了解决内存碎片，可以将2个连续的未使用的内存块合并，减少碎片。



#  ==---------**TCMalloc**---------==

**TCMalloc是Thread Cache Malloc的简称，是Go内存管理的起源**，<font color=red size=5x>Go的内存管理是借鉴了TCMalloc，并不是完全使用</font>随着Go的迭代，Go的内存管理与TCMalloc不一致地方在不断扩大，但**其主要思想、原理和概念都是和TCMalloc一致的**，如果跳过TCMalloc直接去看Go的内存管理，也许你会似懂非懂。



在Linux里，其实有不少的内存管理库，比如glibc的ptmalloc，FreeBSD的jemalloc，Google的tcmalloc等等，为何会出现这么多的内存管理库？本质都是**在多线程编程下，追求更高内存管理效率**：更快的分配是主要目的。

## TCMalloc的做法是什么呢？

**为每个线程预分配一块缓存，线程申请小内存时，可以从缓存分配内存**，这样有2个好处：

1. 为线程预分配缓存需要进行1次系统调用，后续线程申请小内存时，从缓存分配，都是在用户态执行，没有系统调用，**缩短了内存总体的分配和释放时间，这是快速分配内存的第二个层次**。
2. 多个线程同时申请小内存时，从各自的缓存分配，访问的是不同的地址空间，无需加锁，**把内存并发访问的粒度进一步降低了，这是快速分配内存的第三个层次**。



![image-20201026221702017](memory-gc.assets/image-20201026221702017.png)



<font color=red size=5x>==Page==</font>：操作系统对内存管理以页为单位，TCMalloc也是这样，只不过TCMalloc里的Page大小与操作系统里的大小并不一定相等，而是倍数关系。《TCMalloc解密》里称x64下Page大小是8KB。

<font color=red size=5x>==Span==</font>：一组连续的Page被称为Span，比如可以有2个页大小的Span，也可以有16页大小的Span，Span比Page高一个层级，是为了方便管理一定大小的内存区域，Span是TCMalloc中内存管理的基本单位。

<font color=red size=5x>==ThreadCache==</font>**：每个线程各自的Cache，一个Cache包含多个空闲内存块链表，每个链表连接的都是内存块，同一个链表上内存块的大小是相同的，也可以说按内存块大小，给内存块分了个类，这样可以根据申请的内存大小，快速从合适的链表选择空闲内存块。由于每个线程有自己的ThreadCache，所以ThreadCache访问是无锁的。**

<font color=red size=5x>==CentralCache==</font>：是所有线程共享的缓存，也是保存的空闲内存块链表，链表的数量与ThreadCache中链表数量相同，当ThreadCache内存块不足时，可以从CentralCache取，当ThreadCache内存块多时，可以放回CentralCache。由于CentralCache是共享的，**所以它的访问是要加锁的。**

<font color=red size=5x>==PageHeap==</font>：PageHeap是堆内存的抽象，PageHeap存的也是若干链表，链表保存的是Span，当CentralCache没有内存的时，会从PageHeap取，把1个Span拆成若干内存块，添加到对应大小的链表中，当CentralCache内存多的时候，会放回PageHeap。如下图，分别是1页Page的Span链表，2页Page的Span链表等，最后是large span set，这个是用来保存中大对象的。毫无疑问，**PageHeap也是要加锁的**。



![image-20201026222431313](memory-gc.assets/image-20201026222431313.png)

上文提到了小、中、大对象，Go内存管理中也有类似的概念，我们瞄一眼TCMalloc的定义：

1. 小对象大小：0~256KB
2. 中对象大小：257~1MB
3. 大对象大小：>1MB

小对象的分配流程：ThreadCache -> CentralCache -> HeapPage，大部分时候，ThreadCache缓存都是足够的，不需要去访问CentralCache和HeapPage，无锁分配加无系统调用，分配效率是非常高的。

中对象分配流程：直接在PageHeap中选择适当的大小即可，128 Page的Span所保存的最大内存就是1MB。

大对象分配流程：从large span set选择合适数量的页面组成span，用来存储数据。



# ==---------- Go内存管理-------==

![image-20201026222620286](memory-gc.assets/image-20201026222620286.png)

**Page**

与TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

**Span**

与TCMalloc中的Span相同，**Span是内存管理的基本单位**，代码中为mspan，**一组连续的Page组成1个Span**，所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

**mcache**

mcache与TCMalloc中的ThreadCache类似，**mcache保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以无锁访问**。

但mcache与ThreadCache也有不同点，TCMalloc中是每个线程1个ThreadCache，<font color=red size=6x>Go中是**每个P拥有1个mcache**，因为在Go程序中，当前最多有GOMAXPROCS个线程在用户态运行，所以最多需要GOMAXPROCS个mcache就可以保证各线程对mcache的无锁访问，线程的运行又是与P绑定的，把mcache交给P刚刚好。</font>

**mcentral**

mcentral与TCMalloc中的CentralCache类似，**是所有线程共享的缓存，需要加锁访问**，它按Span class对Span分类，串联成链表，当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。

但mcentral与CentralCache也有不同点，CentralCache是每个级别的Span有1个链表，mcache是每个级别的Span有2个链表，这和mcache申请内存有关，稍后我们再解释。

**mheap**

mheap与TCMalloc中的PageHeap类似，**它是堆内存的抽象，把从OS申请出的内存页组织成Span，并保存起来**。当mcentral的Span不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，同样也是需要加锁访问的。

但mheap与PageHeap也有不同点：mheap把Span组织成了树结构，而不是链表，并且还是2棵树，然后把Span分配到heapArena进行管理，它包含地址映射和span是否包含指针等位图，这样做的主要原因是为了更高效的利用内存：分配、回收和再利用。

## **大小转换**

![image-20201026223256131](memory-gc.assets/image-20201026223256131.png)



1. **object size**：代码里简称size，指申请内存的对象大小。
2. **size class**：代码里简称class，它是size的级别，相当于把size归类到一定大小的区间段，比如size[1,8]属于size class 1，size(8,16]属于size class 2。
3. **span class**：指span的级别，但span class的大小与span的大小并没有正比关系。span class主要用来和size class做对应，1个size class对应2个span class，2个span class的span大小相同，只是功能不同，1个用来存放包含指针的对象，一个用来存放不包含指针的对象，不包含指针对象的Span就无需GC扫描了。
4. **num of page**：代码里简称npage，代表Page的数量，其实就是Span包含的页数，用来分配内存。

![image-20201026223546362](memory-gc.assets/image-20201026223546362.png)

前3列分别是size class，object size和span size，根据这3列做size、size class和num of page之间的转换。



# 一般内存分配

![image-20201027213940333](memory-gc.assets/image-20201027213940333.png)



![image-20201027214026037](memory-gc.assets/image-20201027214026037.png)



# ==------go的内存分配核心思想-----==

Go的内存分配的核心思想可以分为以下几点：

- 每次从操作系统申请一大块儿的内存，由Go来对这块儿内存做分配，减少系统调用
- 内存分配算法采用Google的TCMalloc算法。算法比较复杂，究其原理可自行查阅。其核心思想就是把内存切分的非常的细小，分为多级管理，以降低锁的粒度。
- 回收对象内存时，并没有将其真正释放掉，只是放回预先分配的大块内存中，以便复用。只有内存闲置过多的时候，才会尝试归还部分内存给操作系统，降低整体开销

## 基础概念

为了方便自主管理内存，做法便是先向系统申请一块内存，然后将内存切割成小块，通过一定的内存分配算法管理内存。以64位系统为例，Golang程序启动时会向系统申请的内存如下图所示：

![img](memory-gc.assets/memory-01-init.png)

预申请的内存划分为spans、bitmap、arena三部分。其中arena即为所谓的堆区，应用中需要的内存从这里分配。其中spans和bitmap是为了管理arena区而存在的。

arena的大小为512G，为了方便管理把arena区域划分成一个个的page，每个page为8KB,一共有512GB/8KB个页；

spans区域存放span的指针，每个指针对应一个page，所以span区域的大小为(512GB/8KB)*指针大小8byte = 512M

bitmap区域大小也是通过arena计算出来，不过主要用于GC。

## <font color=red size=5x>span</font>

span是用于管理arena页的关键数据结构，每个span中包含1个或多个连续页，为了满足小对象分配，span中的一页会划分更小的粒度，而对于大对象比如超过页大小，则通过多页实现。



spans区域，可以认为是用于上面所说的管理分配arena(即heap)的区域。

此区域存放了mspan的指针，mspan是啥后面会讲。

spans区域用于表示arena区中的某一页(page)属于哪个mspan。

![image-20201027214618983](memory-gc.assets/image-20201027214618983.png)

mspan可以说是go内存管理的最基本单元，但是内存的使用最终还是要落脚到“对象”上。mspan和对象是什么关系呢？

其实“对象”肯定也放到page中，毕竟page是内存存储的基本单元。

我们抛开问题不看，先看看一般情况下的对象和内存的分配是如何的：如下图

## 为什么存在mspan

![image-20201027214649488](memory-gc.assets/image-20201027214649488.png)



假如再分配“p4”的时候，是不是内存不足没法分配了？是不是有很多碎片？

这种一般的分配情况会出现内存碎片的情况，go是如何解决的呢？

可以归结为四个字：按需分配。go将内存块分为大小不同的67种，然后再把这67种大内存块，逐个分为小块(可以近似理解为大小不同的相当于page)称之为span(连续的page)，在go语言中就是上文提及的mspan。

![image-20201027214838353](memory-gc.assets/image-20201027214838353.png)



对象分配的时候，根据对象的大小选择大小相近的span，这样，碎片问题就解决了。67中不同大小的span代码注释如下(目前版本1.11)：

```
// class  bytes/obj  bytes/span  objects  waste bytes
//     1          8        8192     1024            0
//     2         16        8192      512            0
//     3         32        8192      256            0
//     4         48        8192      170           32
//     5         64        8192      128            0
//     6         80        8192      102           32
//     7         96        8192       85           32
//     8        112        8192       73           16
//     9        128        8192       64            0
//    10        144        8192       56          128
//    11        160        8192       51           32
//    12        176        8192       46           96
//    13        192        8192       42          128
//    14        208        8192       39           80
//    15        224        8192       36          128
//    16        240        8192       34           32
//    17        256        8192       32            0
//    18        288        8192       28          128
//    19        320        8192       25          192
//    20        352        8192       23           96
//    21        384        8192       21          128
//    22        416        8192       19          288
//    23        448        8192       18          128
//    24        480        8192       17           32
//    25        512        8192       16            0
//    26        576        8192       14          128
//    27        640        8192       12          512
//    28        704        8192       11          448
//    29        768        8192       10          512
//    30        896        8192        9          128
//    31       1024        8192        8            0
//    32       1152        8192        7          128
//    33       1280        8192        6          512
//    34       1408       16384       11          896
//    35       1536        8192        5          512
//    36       1792       16384        9          256
//    37       2048        8192        4            0
//    38       2304       16384        7          256
//    39       2688        8192        3          128
//    40       3072       24576        8            0
//    41       3200       16384        5          384
//    42       3456       24576        7          384
//    43       4096        8192        2            0
//    44       4864       24576        5          256
//    45       5376       16384        3          256
//    46       6144       24576        4            0
//    47       6528       32768        5          128
//    48       6784       40960        6          256
//    49       6912       49152        7          768
//    50       8192        8192        1            0
//    51       9472       57344        6          512
//    52       9728       49152        5          512
//    53      10240       40960        4            0
//    54      10880       32768        3          128
//    55      12288       24576        2            0
//    56      13568       40960        3          256
//    57      14336       57344        4            0
//    58      16384       16384        1            0
//    59      18432       73728        4            0
//    60      19072       57344        3          128
//    61      20480       40960        2            0
//    62      21760       65536        3          256
//    63      24576       24576        1            0
//    64      27264       81920        3          128
//    65      28672       57344        2            0
//    66      32768       32768        1            0
```

- class： class ID，每个span结构中都有一个class ID, 表示该span可处理的对象类型
- bytes/obj：该class代表对象的字节数
- bytes/span：每个span占用堆的字节数，也即页数*页大小
- objects: 每个span可分配的对象个数，也即（bytes/spans）/（bytes/obj）
- waste bytes: 每个span产生的内存碎片，也即（bytes/spans）%（bytes/obj）

以类型(class)为1的span为例,span中的元素大小是8 byte, span本身占1页也就是8K, 一共可以保存1024个对象。

细心的同学可能会发现代码中一共有66种，还有一种特殊的span：

==即对于大于32k的对象出现时，会直接从heap分配一个特殊的span，这个特殊的span的类型(class)是0, 只包含了一个大对象, span的大小由对象的大小决定。==

## bitmap

bitmap 有好几种:Stack, data, and bss bitmaps，再就是这次要说的heap bitmaps。

在此bitmap的做作用是标记标记arena(即heap)中的对象。<font color=red size=5x>一是的标记对应地址中是否存在对象，另外是标记此对象是否被gc标记过。一个功能一个bit位，所以，heap bitmaps用两个bit位。</font>

bitmap区域中的一个byte对应arena区域的四个指针大小的内存的结构如下：

![image-20201027215324229](memory-gc.assets/image-20201027215324229.png)



bitmap的地址是由高地址向低地址增长的。

宏观图:

![image-20201027215430750](memory-gc.assets/image-20201027215430750.png)

bitmap 主要的作用还是服务于GC。

arena中包含基本的管理单元和程序运行时候生成的对象或实体，这两部分分别被spans和bitmap这两块非heap区域的内存所对应着。

## 三者关系图

![image-20201027215525280](memory-gc.assets/image-20201027215525280.png)



spans和bitmap都会根据arena的动态变化而动态调整大小。



# go内存管理组件

- mspan为内存管理的基础单元，直接存储数据的地方。
- mcache：每个运行期的goroutine都会绑定的一个mcache(具体来讲是绑定的GMP并发模型中的P，所以可以无锁分配mspan，后续还会说到)，mcache会分配goroutine运行中所需要的内存空间(即mspan)。
- mcentral为所有mcache切分好后备的mspan
- mheap代表Go程序持有的所有堆空间。还会管理闲置的span，需要时向操作系统申请新内存。

![image-20201027220123112](memory-gc.assets/image-20201027220123112.png)

**Page**

与TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

**Span**

与TCMalloc中的Span相同，**Span是内存管理的基本单位**，代码中为mspan，**一组连续的Page组成1个Span**，所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

**mcache**

mcache与TCMalloc中的ThreadCache类似，**mcache保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以无锁访问**。

但mcache与ThreadCache也有不同点，TCMalloc中是每个线程1个ThreadCache，Go中是**每个P拥有1个mcache**，因为在Go程序中，当前最多有GOMAXPROCS个线程在用户态运行，所以最多需要GOMAXPROCS个mcache就可以保证各线程对mcache的无锁访问，线程的运行又是与P绑定的，把mcache交给P刚刚好。

**mcentral**

mcentral与TCMalloc中的CentralCache类似，**是所有线程共享的缓存，需要加锁访问**，它按Span class对Span分类，串联成链表，当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。

但mcentral与CentralCache也有不同点，CentralCache是每个级别的Span有1个链表，mcache是每个级别的Span有2个链表，这和mcache申请内存有关，稍后我们再解释。

**mheap**

mheap与TCMalloc中的PageHeap类似，**它是堆内存的抽象，把从OS申请出的内存页组织成Span，并保存起来**。当mcentral的Span不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，同样也是需要加锁访问的。

但mheap与PageHeap也有不同点：mheap把Span组织成了树结构，而不是链表，并且还是2棵树，然后把Span分配到heapArena进行管理，它包含地址映射和span是否包含指针等位图，这样做的主要原因是为了更高效的利用内存：分配、回收和再利用。



## ==---mspan---==

- <font color=red size=5x>mspan有134个,class是直接从heap分配的,>32kb的数据</font>

**Span**

与TCMalloc中的Span相同，**Span是内存管理的基本单位**，代码中为mspan，**一组连续的Page组成1个Span**

![image-20201027215935985](memory-gc.assets/image-20201027215935985.png)

## span数据结构

span是内存管理的基本单位,每个span用于管理特定的class对象, 跟据对象大小，span将一个或多个页拆分成多个块进行管理。

`src/runtime/mheap.go:mspan`定义了其数据结构：

```go
type mspan struct {
    next *mspan            //链表前向指针，用于将span链接起来
    prev *mspan            //链表前向指针，用于将span链接起来
    startAddr uintptr // 起始地址，也即所管理页的地址
    npages    uintptr // 管理的页数
    nelems uintptr // 块个数，也即有多少个块可供分配
    allocBits  *gcBits //分配位图，每一位代表一个块是否已分配
    allocCount  uint16     // 已分配块的个数
    spanclass   spanClass  // class表中的class ID
    elemsize    uintptr    // class表中的对象大小，也即块大小
}
```

以class 10为例，span和管理的内存如下图所示：



![4.1 内存分配原理 - 图2](memory-gc.assets/memory-02-mspan.png)

spanclass为10，参照class表可得出npages=1,nelems=56,elemsize为144。其中startAddr是在span初始化时就指定了某个页的地址。allocBits指向一个位图，每位代表一个块是否被分配，本例中有两个块已经被分配，其allocCount也为2。

next和prev用于将多个span链接起来，这有利于管理多个span。



mspan是双向链表，其中主要的属性如下图所示：

![image-20201027220542096](memory-gc.assets/image-20201027220542096.png)







## ==---mcache---==

为了避免多线程申请内存时不断的加锁，goroutine为每个线程分配了span内存块的缓存，这个缓存即是mcache，每个goroutine都会绑定的一个mcache，各个goroutine申请内存时不存在锁竞争的情况。

mcache与TCMalloc中的ThreadCache类似，**mcache保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以无锁访问**。

但mcache与ThreadCache也有不同点，TCMalloc中是每个线程1个ThreadCache，Go中是**每个P拥有1个mcache**，因为在Go程序中，当前最多有GOMAXPROCS个线程在用户态运行，所以最多需要GOMAXPROCS个mcache就可以保证各线程对mcache的无锁访问，线程的运行又是与P绑定的，把mcache交给P刚刚好。

<font color=red size=5x>因为运行期间一个goroutine只能和一个P关联，而mcache就在P上，所以，不可能有锁的竞争。</font>

## mcache数据结构

有了管理内存的基本单位span，还要有个数据结构来管理span，这个数据结构叫mcentral，各线程需要内存时从mcentral管理的span中申请内存，为了避免多线程申请内存时不断的加锁，Golang为每个线程分配了span的缓存，这个缓存即是cache。

`src/runtime/mcache.go:mcache`定义了cache的数据结构：

```
type mcache struct {
    alloc [67*2]*mspan // 按class分组的mspan列表
}
```

==alloc为mspan的指针数组，数组大小为class总数的2倍。==数组中每个元素代表了一种class类型的span列表，<font color=red size=5x>每种class类型都有两组span列表，`第一组列表中所表示的对象中包含了指针`，`第二组列表中所表示的对象不含有指针`，这么做是为了提高GC扫描性能，对于不包含指针的span列表，没必要去扫描。</font>



根据对象是否包含指针，将对象分为noscan和scan两类，其中noscan代表没有指针，而scan则代表有指针，需要GC进行扫描。

mcache和span的对应关系如下图所示：

![4.1 内存分配原理 - 图3](memory-gc.assets/memory-03-mcache.png)

mchache在初始化时是没有任何span的，在使用过程中会动态的从central中获取并缓存下来，跟据使用情况，每种class的span个数也不相同。上图所示，class 0的span数比class1的要多，说明本线程中分配的小对象要多一些。

![image-20201027221720862](memory-gc.assets/image-20201027221720862.png)

## mcache为什么分为有指针和无指针

mcache中的span链表分为两组，一组是包含指针类型的对象，另一组是不包含指针类型的对象。为什么分开呢？

主要是方便GC，在进行垃圾回收的时候，对于不包含指针的对象列表无需进一步扫描是否引用其他活跃的对

<font color=red size=5x>对于 <=32k的对象，将直接通过mcache分配。</font>

go中对象按照的大小维度的分类。

分为三类：

- tinny allocations (size < 16 bytes，no pointers)
- small allocations (16 bytes < size <= 32k)
- large allocations (size > 32k)

前两类：tiny allocations和small allocations是直接通过mcache来分配的。

对于tiny allocations的分配，有一个微型分配器tiny allocator来分配，分配的对象都是不包含指针的，例如一些小的字符串和不包含指针的独立的逃逸变量等。

small allocations的分配，就是mcache根据对象的大小来找自身存在的大小相匹配mspan来分配。

当mcach没有可用空间时，会从mcentral的 mspans 列表获取一个新的所需大小规格的mspan。

## ==---mcentral---==

为所有mcache提供切分好的mspan。

每个mcentral保存一种特定类型的全局mspan列表，包括已分配出去的和未分配出去的。

![image-20201027222524447](memory-gc.assets/image-20201027222524447.png)

有多少种类型的mspan就有多少个mcentral。

每个mcentral都会包含两个mspan的列表：

- 没有空闲对象或mspan已经被mcache缓存的mspan列表(empty mspanList)
- 有空闲对象的mspan列表(empty mspanList)

==由于mspan是全局的，会被所有的mcache访问，所以会出现并发性问题，因而mcentral会存在一个锁。==

单个的mcentral结构如下：

![image-20201027222605459](memory-gc.assets/image-20201027222605459.png)

假如需要分配内存时，mcentral没有空闲的mspan列表了，此时需要向mheap去获取。

## mcentral数据结构

cache作为线程的私有资源为单个线程服务，而central则是全局资源，为多个线程服务，当某个线程内存不足时会向central申请，当某个线程释放内存时又会回收进central。

`src/runtime/mcentral.go:mcentral`定义了central数据结构：

```
type mcentral struct {
    lock      mutex     //互斥锁
    spanclass spanClass // span class ID
    nonempty  mSpanList // non-empty 指还有空闲块的span列表
    empty     mSpanList // 指没有空闲块的span列表
    nmalloc uint64      // 已累计分配的对象个数
}
```

- lock: 线程间互斥锁，防止多线程读写冲突
- spanclass : 每个mcentral管理着一组有相同class的span列表
- nonempty: 指还有内存可用的span列表
- empty: 指没有内存可用的span列表
- nmalloc: 指累计分配的对象个数

## 获取流程

线程从central获取span步骤如下：

1. 加锁
2. 从nonempty列表获取一个可用span，并将其从链表中删除
3. 将取出的span放入empty链表
4. 将span返回给线程
5. 解锁
6. 线程将该span缓存进cache

## 释放流程

线程将span归还步骤如下：

1. 加锁
2. 将span从empty列表删除
3. 将span加入noneempty列表
4. 解锁



## ==---mheap---==

mheap可以认为是Go程序持有的整个堆空间，mheap全局唯一，可以认为是个全局变量。

其结构如下：

![image-20201027222920788](memory-gc.assets/image-20201027222920788.png)

mheap包含了除了上文中讲的mcache之外的一切，mcache是存在于Go的GMP调度模型的P中的

## mheap的lock

我们知道，大于32K的对象被定义为大对象，直接通过mheap 分配。这些大对象的申请是由mcache发出的，而mcache在P上，程序运行的时候往往会存在多个P，因此，这个内存申请是并发的；所以为了保证线程安全，必须有一个全局锁。

假如需要分配的内存时，mheap中也没有了，则向操作系统申请一系列新的页（最小 1MB）。

## mheap数据结构

从mcentral数据结构可见，每个mcentral对象只管理特定的class规格的span。事实上每种class都会对应一个mcentral,这个mcentral的集合存放于mheap数据结构中。

`src/runtime/mheap.go:mheap`定义了heap的数据结构：

```
type mheap struct {
    lock      mutex
    spans []*mspan
    bitmap        uintptr     //指向bitmap首地址，bitmap是从高地址向低地址增长的
    arena_start uintptr        //指示arena区首地址
    arena_used  uintptr        //指示arena区已使用地址位置
    central [67*2]struct {
        mcentral mcentral
        pad      [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
    }
}
```

- lock： 互斥锁
- spans: 指向spans区域，用于映射span和page的关系
- bitmap：bitmap的起始地址
- arena_start: arena区域首地址
- arena_used: 当前arena已使用区域的最大地址
- central: 每种class对应的两个mcentral

从数据结构可见，mheap管理着全部的内存，事实上Golang就是通过一个mheap类型的全局变量进行内存管理的。

mheap内存管理示意图如下：

![4.1 内存分配原理 - 图4](memory-gc.assets/memory-04-mheap.png)

# go 内存分配流程

针对待分配对象的大小不同有不同的分配逻辑：

- (0, 16B) 且不包含指针的对象： Tiny分配
- (0, 16B) 包含指针的对象：正常分配
- [16B, 32KB] : 正常分配
- (32KB, -) : 大对象分配其中Tiny分配和大对象分配都属于内存管理的优化范畴，这里暂时仅关注一般的分配方法。

以申请size为n的内存为例，分配步骤如下：

1. 获取当前线程的私有缓存mcache
2. 跟据size计算出适合的class的ID
3. 从mcache的alloc[class]链表中查询可用的span
4. 如果mcache没有可用的span则从mcentral申请一个新的span加入mcache中
5. 如果mcentral中也没有可用的span则从mheap中申请一个新的span加入mcentral
6. 从该span中获取到空闲对象地址并返回



# 总结

Golang内存分配是个相当复杂的过程，其中还掺杂了GC的处理，这里仅仅对其关键数据结构进行了说明，了解其原理而又不至于深陷实现细节。

1. Golang程序启动时申请一大块内存，并划分成spans、bitmap、arena区域
2. arena区域按页划分成一个个小块
3. span管理一个或多个页
4. mcentral管理多个span供线程申请使用
5. mcache作为线程私有资源，资源来源于mcentral





















































