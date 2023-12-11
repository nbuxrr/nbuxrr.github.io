---
title: MySQL 数据文件目录
categories: MySQL
tags: [MySQL, datadir]
date: 2021-07-02
---

阅读笔记

《MySQL是怎样运行的:从根儿上理解MySQL》

## 数据目录

- basedir是安装目录，保存可执行文件等
- datadir是数据目录，保存用户数据文件

    ```SQL
    show variables like 'datadir';
    ```

## create database

创建数据库时会触发两件事

- 在datadir下创建一个数据库同名的文件夹 ```$datadir/database_name/```
- 创建一个名为```$datadir/database_name/db.opt```的文件，该文件中包含了该数据库的各种属性，例如字符集和比较规则等

> 特例: information_schema数据库没有对应文件夹

## create table

每个表信息分为两种

- 表结构属性  ```$datadir/database_name/table_name.frm``` 二进制格式保存
- 表记录数据
    - 表空间(table space)或者文件空间(file space)，对应文件系统上一个或多个物理文件。
    - 每个表空间被划分成很多很多的页，表数据就保存在某个表空间的某个页上

## 系统表空间 system tablespace

所谓系统表空间可以对应系统上一个或多个实际的文件。
InnoDB默认会在$datadir目录下创建一个名为```ibdata1```的文件，初始大小为12M（为什么是12M？），12M不够用后，文件会增大，增大到限制的大小（例如512MB）后，再增大就会扩展例如```ibdata2```文件。

ibdata文件的名字及目录可以配置

```ini
[server]
innodb_data_file_path=data1:512M;data2:512M:autoextend
innodb_data_home_dir=xxxxxxx
```

系统表空间只有一份，MySQL5.5.7 - MySQL5.6.6 表中的数据都默认存储在这个系统表空间中

## 独立表空间 file-per-table tablespace

MySQL5.6.6之后，InnoDB默认每个表的数据分别存储到自己独立的表空间。使用独立表空间的表，会在表所在数据库目录```$datadir/database_name```下创建与表名对应的```table_name.ibd```文件,

所以默认InnoDB的表有两个文件

```$datadir/database_name/table_name.frm```     // 保存表结构属性
```$datadir/database_name/table_name.ibd```     // 独立表空间文件，保存表索引和数据

当然也可以通过参数```innodb_file_per_table```来控制表数据保存在```系统表空间```还是```独立表空间```,该参数仅对新建表时有效

将表的数据从系统表空间中移到独立表空间的方法

```SQL
ALTER TABLE 表名 TABLESPACE innodb_file_per_table;
```

将表从独立表空间移到系统表空间

```SQL
ALTER TABLE 表名 TABLESPACE innodb_system;
```

## 通用表空间 general tablespace

## undo表空间 undo  tablespace

## 临时表空间 temporary tablespace


## MyISAM的数据存储

table_name.frm  表结构
table_name.MYD  数据
table_name.MYI  索引 全部索引为二级索引（即叶子节点都不包含完整用户记录）

## 视图的存储

视图是一个虚拟的表，本质就是一个sql语句，所以一个frm文件即可。

```
view_name.frm
```

## 文件系统对数据库的影响

- 数据库名、表名受文件系统最大路径长度的影响
- 数据库名或表名中含有特殊字符（除数字、字母、'-' 以外的所有字符会被转为@编码的形式，例如test?的表的frm文件对应为test@003f.frm）
- .ibd文件大小(表空间大小)受文件系统影响

## MySQL系统数据库

|  |  |
| :- | :- |
| mysql | 主要保存用户信息和权限信息 |
| information_schema | 主要保存MySQL中其他数据库的元数据 |
| performance_schema | 主要保存性能统计信息 |
| sys | 通过视图形式将information_schema和performance_schema结合起来，方便查看一些性能信息 |

## 独立表空间结构

系统表空间比独立表空间多了一些包含整个系统空间的信息，所以先从简单的独立表空间开始。

### 区 extent (64 pages)

    对于16k的页，64个页为一个区，这样1个extent就是1M

### 组        (256 * 64 pages)

每个组包含256个区，即256M

一个独立表空间第一个组前3个页是固定的FSP_HDR、IBUF_BITMAP、INODE，之后的组的前2个页是固定的XDES、IBUF_BITMAP
> XDES = extent descriptor

|  |  |
| :- | :- |
| FSP_HDR / XDES (16K) | 用来登记```整个表空间的一些整体属性``` + 本组的所有区的属性(XDES) |
| IBUF_BITMAP (16K) | 用来保存本组所有区的所有页关于INSERT BUFFER的信息 |
| INODE(16K) | 只有表空间的第一个组有这个页，保存段属性信息的结构 |

B+树每次新增节点，每层的节点都是双向链表，如果每次新增节点```分配空间以页为单位```，则很可能双向链表的相邻页的磁盘位置相差很对，就会导致经常的```随机IO```，如果新增节点时```分配空间以区为单位```，则同一个区内的页基本都是磁盘位置连续的，即```顺序IO```。

### 段 segment

段和组 差别？

虽然一个区中是连续的64个page，但范围查询时，B+树的扫描是在（叶子节点或非叶子节点）同一层上的扫描，为了保持顺序IO，需要将叶子节点和非叶子节点分开放到不同的区中。所以就形成了段的概念。
|  |  |
| :- | :- |
| ```叶子节点段``` | 存放叶子节点的区的集合算作一个段 |
| ```非叶子节点段``` | 存放非叶子节点的区的集合也算作一个段 |

一个表至少有一个索引即聚簇索引，一个索引分两个段，如果每个段各分配一个区，则一个表存很少量数据就会占用2M空间，浪费严重，所以加入了```碎片区```的功能。

```碎片区```可以存放各种段的页的数据，不属于某个段，碎片区属于表空间。

表分配页或分配区的策略

1. 在刚开始向表插入数据的时候，段是从某个碎片区以单个页为单位来分配存储空间
1. 当某个段以及占用了碎片区的32个页面后，就会以完整的区为单位进行分配存储空间了

另外的段还有回滚段。

区的分类

| 状态名 | 含义 |
| :- | :- |
| FREE | 空闲的区 还未被使用的区 |
| FREE_FRAG | 有剩余空间的碎片区 还有可用的页 |
| FULL_FRAG | 没有剩余可用页面的区 |
| FESG | 附属于某个段的区 |

### XDES Entry

每个区有个管理对象，成为XDES Entry，包含信息

| 字节 | 名称 | 作用 |
| -: | :- | :- |
| 8B | Segment ID | 该区所属的段的ID，如果属于某个段的话 |
| 4B | ListNode: Prev Node Page Number | 前一个XDES Entry所在页号 |
| 2B | ListNode: Prev Node Offset | 前一个XDES Entry所在页的偏移 |
| 4B | ListNode: Next Node Page Number | 后一个XDES Entry所在页号 |
| 2B | ListNode: Next Node Offset | 后一个XDES Entry所在页的偏移 |
| 4B | State | 该区的状态 FREE、FREE_FRAG、FULL_FRAG、FESG |
| 16B | Page State Bitmap | 16Bytes = 128bits 每2 bits 对应区上的1页，一个区64页，刚好128bits。 2bits的第一个bit标识页是否空闲，第二个bit预留暂时未使用 |

这么多概念的目的归结起来就是既想插入和扫描数据速度快，又想少浪费空间

### 各种链表

#### 属于表空间的三种XDES Entry链表

- FREE链表 所有FREE的XDES Entry组成的链表
- FREE_FRAG链表 所有FREE_FRAG的XDES Entry组成的链表
- FULL_FRAG链表 所有FULL_FRAG的XDES Entry组成的链表

#### 属于段的三种XDES Entry链表

为了避免不同的索引的页分配到同一个区的情况，需要为每个段根据段创建三种链表，不同段不共用链表。

- FREE链表，该段中所有页都空闲的区的XDES Entry链表
- NOT_FULL链表，该段中所有用过且有空闲空间的区的XDES Entry链表
- FULL链表，该段中没有空闲空间的区的XDES Entry链表

例如每个索引都有两个段，每个段都有这三种链表，每个索引就有6个XDES Entry链表

### 链表基节点 List Base Node

|  |  |  |
| -: | :- | :- |
| 4B | List Length | 链表长度 |
| 4B | First Node Page Number | 链表头所在页号 |
| 2B | First Node Offset | 链表头所在页中的偏移 |
| 4B | Last Node Page Number | 链表尾所在页号 |
| 2B | Last Node Offset | 链表尾所在页中的偏移 |

链表基节点在表空间的固定位置，就可以轻松找到各种链表（链表和段怎样关联？）
