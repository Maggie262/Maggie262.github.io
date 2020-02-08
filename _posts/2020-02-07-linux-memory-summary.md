---
layout:     post
title:      "linux memory management summary"
subtitle:   "  "
date:       2020-02-07
author:     "Maggie"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - linux
    - memory
---

## linux内存管理总结



### 进程的地址空间分布

![](/img/in-post/process-memory-location.jpg)



linux内存管理大图：

![](/img/in-post/2020-02-07-linux-memory-summary.gif)







### RES 不下降到原来的值，原因

一个进程，malloc分配内存然后释放，使用top命令观察该进程的RES，没有下降，原因：

* 进程申请内存路径process -> Glibc -> kernel，Glibc的内存分配策略是ptmalloc

* kernel回收页面： 当页面完全没有被使用，并且是movable pages，才能进行pages迁移和section回收。



> - **page-wise allocation and fragmentation**: Operating systems allocate memory in terms of *pages*, at least on modern systems using [MMU](https://en.wikipedia.org/wiki/Memory_management_unit)s. Your standard library's `malloc()` (or the allocator for `new` in C++) serves arbitrary sizes and manages them internally. A page can only be returned to the operating system when the last allocation occupying it is returned. If only a single allocation in the page is still active, your process must keep this page.
> - **libraries**: A lot of libraries do their own dynamic memory allocations, even the C standard library does.
>
> ---
>
> * 进程释放内存后，这部分内存并没有归还给OS，但是当再调用`malloc`，`remalloc`时，free的这部分内存可以被复用
> * OS分配内存以页为单位，如果页中有一部分为active，那该页也不会归还给OS
> * 库： 很多库有自己的动态内存分配策略。C程序通过Glibc接口向内核申请内存时，路径为process -> glibc -> linux kernel。Glibc有自己的内存分配策略（[Glibc](https://www.cnblogs.com/tibetanmastiff/p/4383061.html)介绍），采用ptmalloc



> To oversimplify things, there are two memory managers at work in dynamic memory allocation: the **OS Memory Manager**, and the **Process Memory Manager** (which there can be more than one of). The OS Memory Manager allocates "large chunks" of memory to individual Process Memory Managers. Each Process Memory Manager keeps track of the allocated segments, as well as the "free'd segments". The Process Memory Manager does not return the free'd segments to the OS Memory Manager because it is more efficient to hold on to it, in case it needs to allocate more memory later.





原因参考： 

https://stackoverflow.com/questions/47508165/will-processs-res-memory-drop-after-memory-freed

https://stackoverflow.com/questions/13232119/memory-usage-doesnt-decrease-when-free-used

https://stackoverflow.com/questions/55294985/does-malloc-reserve-more-space-while-allocating-memory

参考：

Glibc: https://www.cnblogs.com/tibetanmastiff/p/4383061.html

ptmalloc: https://blog.csdn.net/phenics/article/details/777053

malloc: https://linux.die.net/man/3/malloc 

 



#### 管理区分配器

linux中的内存被分为一个个页框，并用页表进行管理。此外，有如下限制，

- DMA处理器，只能对RAM的前16MB寻址。
- 32位机器CPU最大寻址空间，只有4GB，对于大容量超过4GB的RAM，无法访问所有的地址空间。

Linux还将物理内存划分为不同的管理区：ZONE_DMA、ZONE_NORMAL、ZONE_HIGHMEM，每个管理区都有自己的描述符，也有自己的页框分配器，

```
管理区分配器
|
|----ZONE_DMA--------|-每CPU高速缓存
|										 |-Buddy system
|
|----ZONE_NORMAL-----|-每CPU高速缓存
|										 |-Buddy system
|
|----ZONE_HIGHMEM----|-每CPU高速缓存
|										 |-BUddy system
```



* 大块内存分配： 伙伴系统，并能减少外碎片的产生。同时，没存管理区定义了没CPU高速缓存，可以提高性能
* 小块内存的分配： Slab分配器算法，在Buddy系统的基础上进一步细化。可以减少内碎片的产生

* 非连续内存的分配：

  把内存区映射到一组连续的页框是最好的选择，这样会充分利用高速缓存。如果对内存区的请求不是很频繁，那么分配非连续的页框，会是比较好的选择，因为这样会避免外部碎片，缺点是内核的页表比较乱。Linux以下方面使用了非连续内存区：

  - 为活动交换区分配数据结构。
  - 给某些IO驱动程序分配缓冲区等



### linux 内存调试

* 进程的实存和虚存

  * 虚存：包含进程可以访问的所有内存，包含被换出、已经分配但还未使用的内存，以及来自共享库的内存。可以通过 ps -o vsz 查看进程的虚存大小。
  * 实存： 进程分配并加载到主存中的库，包括共享库、堆栈和堆内存。查看命令`ps -o rss`

  ---

  * 一个例子

    如果进程A具有500K二进制文件并且链接到2500K共享库，则具有200K的堆栈/堆分配，其中100K实际上在内存中（其余是交换或未使用），并且它实际上只加载了1000K的共享库然后是400K自己的二进制文件：

    ```text
    RSS: 400K + 1000K + 100K = 1500K
    VSZ: 500K + 2500K + 200K = 3200K
    ```

    **实存和虚存是怎么转换的呢？**当程序尝试访问的地址未处于实存中时，就发生页面错误，操作系统必须以某种方式处理这种错误，从而使应用程序正常运行。这些操作可以是：

    - 找到页面驻留在磁盘上的位置，并加载到主存中。
    - 重新配置MMU，更新线性地址和物理地址的映射关系。
    - 等。

    随着进程页面错误的增长，主存中可用页面越来越少，为了防止内存完全耗尽，操作系统必须尽快释放主存中暂时不用的页面，以释放空间供以后使用，方式如下：

    - 将修改后的页面写入到磁盘的专用区域上（调页空间或者交换区）。
    - 将未修改的页面标记为空闲（没必要写入磁盘，因为没有被修改）。

    调页或者交换是操作系统的正常部分，需要注意的是过度交换，这表示当前主存空间不足，页面换出抖动对系统极为不利，会导致CPU和I/O负载升高，极端情况下，会造成操作系统所有的资源花费在调页层面。

* linux下大块内存的分配（pages为单位）最终都是由伙伴系统分配，slab负责管理小块内存，查看相关信息的指令为

  ```shell
  cat /proc/buddyinfo
  cat /proc/slabinfo
  ```



* page chche缓存机制

  Linux中通过page cache机制来加速对磁盘文件的许多访问，当它首次读取或写入数据介质时，Linux会将数据存储在未使用的内存区中，通过这些区域充当缓存，如果再次读取这些数据时，直接从内存中快速获取该数据。当发生写操作时，Linux不会立刻执行磁盘写操作，而是把page cache中的页面标记为脏页，定期同步到存储设备中。

  可以通过`free -m`来查看page cache情况：

  ```text
  total       used       free     shared    buffers     cached
  Mem:         32013      31288        724          0        241      12000
  -/+ buffers/cache:      19046      12966
  Swap:        32767      23134       9633
  ```

  `cached`这列显示了page cache的情况。







