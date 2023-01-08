---
title: MySQL Buffer Pool
categories: MySQL
tags: [MySQL, buffer pool]
date: 2023-01-09
---

## BufferPool

BufferPool是一块"连续"的内存，左边是地址"连续"的控制块，右边是地址"连续"的缓冲页，中间可能是划分多余的碎片

### 控制块

包含在BufferPool中，与数据页一一对应的控制块，每个控制块包含对应的页的表空间编号、页号、缓冲页在BuffePool中的地址，链表节点等信息。

### 缓冲页

包含在BufferPool中，数据页。

### Free list

不包含在BufferPool中， BufferPool中所有空闲页的链表，用于从磁盘加载页时，快速找到空白页用于缓存数据。

### 缓冲页哈希表

不包含在BufferPool中，`表空间号+页号`作为key，页的`控制块地址`作为value，用于快速找到某页是否已加载到BufferPool中，若不在，则需要从freelist中取一个空闲页来使用。

### Flush list

不包含在BufferPool中，BufferPool中所有脏页的链表，用于方便遍历脏页刷脏。

### LRU list

不包含在BufferPool中，用于在BufferPool空闲缓冲页用完时，找出BufferPool中最近最少使用的页作为淘汰让出缓冲页。
	
1. LRU 元素的来源，读一个页时

	- 如果该页不在BufferPool中，在页加载到缓冲页时，就把该页的控制块加到LRU list的头部；
	- 如果该页已经在BufferPool中，则直接将该页的控制块移动到LRU list头部；

2. LRU在预读时的问题，预读会导致很多页被加载了，但之后可能有些页没有被读取就被淘汰了，这降低了BufferPool的命中率

	- `线性预读` 如果顺序访问某个区的页面数大于 innodb_read_ahead_threshold，则触发异步读取下一个区中的所有页面到BufferPool
	- `随机预读` 如果某个区的一定数量（13个）的连续页面（不论是否顺序读）被加载到BufferPool，则将本区的所有页面加载到BufferPool

3. LRU在全表扫描时的问题，会产生非常多缓冲页（低频缓冲页）（假设表的大小大于BufferPool的大小），可能导致高频缓冲页被误淘汰；
4. 解决预读和全表扫的问题。为了解决高频数据被误淘汰，将LRU list分为两个区域（按比例划分，没有固定节点），一部分存放高频页（热数据）的young区域，另一部分存放低频页（冷数据）的old区（old区域大约占3/8）；预读中加载了但没有被读的页面放到LRU的old区域，全表扫的页面也放到LRU的old区域。
5. LRU进一步优化避免频繁的节点移动操作，只有young区域1/4之后区域的节点前移才会前移，即young前1/4的页面被访问时，不再热度前移。
6. unzip LRU list 管理解压页，zip clean list 管理压缩页，zip free数组，每个元素即一个链表 三个zip链表组成伙伴系统来管理压缩页，为压缩页提供内存控件。

## 刷脏页到磁盘

1. BUF_FLUSH_LRU，就是后台线程定时从LRU list中刷新一部分脏页到磁盘，即修改过的冷的脏页先落盘。LRU尾部扫innodb_lru_scan_depth个元素。
2. BUF_FLUSH_FLUSH，就是后台线程定时从flush list中刷新一部分脏页到磁盘。
3. 用户线程加载磁盘页到BufferPool时，若空闲页不足，就会尝试查看LRU list尾部，看是否有可以直接释放的未修改的页，若没有，则只能将LRU尾部的一个脏页落盘。
4. 最后是在系统特别繁忙的时候，用户线程从flushlist链表中刷脏页到磁盘，见redo和checkpoint时。


## 多个BufferPool实例

并发数比较大时，多个BufferPool减少同个BufferPool上的资源竞争。BufferPool实例并不是越多越好，当innodb_buffer_pool_size小于1GB时，innodb_buffer_pool_instances会自动修改为1.

## innodb_buffer_pool_chunk_size

如果BufferPool一次性申请一块很大的内存，速度会很慢，也会造成系统资源利用率的浪费。所以使用chunk为单位进行申请，每个chunk是连续的内存空间，含有若干个控制块和缓冲页，一个BufferPool实例又是由若干个chunk组成。`innodb_buffer_pool_chunk_size`默认128MB，只能数据库启动前修改，启动后无法修改，运行时只能改innodbBufPool的大小。

innodb_buffer_pool_chunk_size的值并不包含缓冲页对应控制块的大小，所以实际每个chunk的大小比innodb_buffer_pool_chunk_size的值要大一些。

- `innodb_buffer_pool_size`必须是`innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances`的整数倍，主要是想保证每个BufferPool实例中的chunk的个数是一致的.
- 如果配置的不是倍数值，数据库会将该配置值`自动调整为倍数值`。
- 如果配置的`innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances`大于`innodb_buffer_pool_size`，则会将`innodb_buffer_pool_chunk_size`自动调整为`innodb_buffer_pool_size / innodb_buffer_pool_instances`

缓冲页除了保存磁盘上的数据页外，还可以保存自适应哈希等信息。

## 查看BufferPool状态

```sql
show engine innodb status;\G
```

|  |  |
| - | - |
| Total memory allocated | 表示BufferPool向操作系统申请的连续内存控件大小，包括全部控制块、缓冲页以及碎片的大小 |
| Dictionary memory allocated | 为数据字典信息分配的内存控件大小，与BufferPool没有关系，不在Total memory allocated中 |
| Buffer pool size | 表示该BufferPool可以容纳的缓冲`页`个数 |
| Free buffers | BufferPool中空闲`页`个数 |
| Database pages | LRU链表中的`页`个数，包含young区域和old区域 |
| Old database pages | LRU链表old区域中的`页`个数 |
| Modified db pages | 代表脏页的个数，即flush链表中的节点个数 |
| Pending reads | 等待从磁盘加载的页数。加载页时，先分配一个空闲页和控制块，然后把这个控制块加到LRU old区域的头部，但磁盘上的内容还没有加载上来 |
| Pending write LRU | 即将从LRU list刷到磁盘上的页数 |
| Pending write flush list | 即将从flush list刷到磁盘上的页数 |
| Pending write single page | 即将以单个页形式刷到磁盘上的页数 |
| Pages made young | 标记从其他状态变为young的页数，不包含从young到young |
| Pages made not young | 访问old区域的页，由于时间间隔条件不符，不能将页移到young区域头部的页个数 |
| youngs/s | 每秒钟从old区域移到young区域的页的个数 |
| non-youngs/s | 代表每秒钟由于时间限制，不能从old区域移到young区域的页的个数 |
| Page read、created、written | 代表读取、创建、写入的页的数量，后面跟着各自的速度 |
| Buffer pool hit rate | 表示过去某段时间内，平均访问1000次页面时，页面已缓存的命中次数 |
| yong-makng rate | 表示过去某段时间内，平均访问1000次页面时，页面被移到young区的次数，包括从old到young和从young到young区域 |
| not(yong-makng rate) | 表示过去某段时间内，平均访问1000次页面时，页面没有移动（到young）的次数，包括young1/4和时间限制的情况 |
| LRU len | LRU list的节点个数 |
| unzip_LRU | unzip_LRU的节点个数 |
| I/O sum | 最近50s内读取磁盘页的总数 |
| I/O cur | 当前正在读取的磁盘页数量 |
| I/O unzip sum | 最近50s解压的页面数量 |
| I/O unzip cur | 当前正在解压的页面数量 |
