---
title: MySQL redo 日志格式
categories: MySQL
tags: [MySQL, redo log]
date: 2023-01-09
---

## 为什么要写redo日志

- 事务持久性要求必须将修改落盘
- InnoDB以页为单位落盘数据
	- 落盘一个完整的数据页`太浪费`
	- 一条语句可能修改多个数据页，而这些数据页有可能不相邻，将修改数据页落盘就要`大量随机IO`
- redo日志只记录修改的表空间号、数据页号、修改位置偏移量、修改数据,`没有多余数据的落盘`
- redo日志是`顺序IO`

## redo日志格式

常规格式

| type | spaceID | page number | data |

| 字段 | 内容 |
| :- | :- |
| type | redo日志的类型 |
| spaceID | 表空间号 |
| page number | 数据页号 |
| data | redo日志的具体内容 |

## 简单的redo日志类型

例如修改一个数据值

| type | spaceID | page number | offset | 具体数据 |

| type | 说明 |
| :- | :- |
| MLOG_1BYTE | 在offset位置修改1字节的数据 |
| MLOG_2BYTE | 在offset位置修改2字节的数据 |
| MLOG_4BYTE | 在offset位置修改4字节的数据 |
| MLOG_8BYTE | 在offset位置修改8字节的数据 |
| MLOG_WRITE_STRING | 在offset位置修改不定字节数的数据，offset后接不定长数据的len |

为啥有len，还要用MLOG_1BYTE之类的type？为了节省空间

## 复杂一些的redo日志类型

例如插入一条记录，可能会修改很多个页面，除了向B+树的页面插入数据外，还会更新系统数据Max Row ID

- 对于B+树，表中有多少个索引，一条INSERT就可能更新多少棵B+树
- 对于某一棵B+树，即可能更新叶子节点页面、也可能更新中间节点页面、也可能页分裂创建新的页并在中间节点页上添加记录项（中间节点可能也会页分裂？）

插入一条记录，一个页中可能发生的修改

- 可能更新Page Directory中的槽信息
- 可能更新Page Header中的统计信息，比如PAGE_N_DIR_SLOTS（表示槽数量）、PAGE_HEAP_TOP（本页面还未使用的空间的最小地址）、PAGE_N_HEAP（本页面中的记录数）
- 数据页中的记录按照索引从小到大排序，组成一个单向链表，每条记录的next_record里保存着下条记录的偏移量，插入新记录的前面一条记录需要更新next_record

实际解决策略

| type | 说明 |
| :- | :- |
| MLOG_REC_INSERT | 插入一条非紧凑行格式（REDUNDANT）的记录 |
| MLOG_COMP_REC_INSERT | 插入一条COMPACT、DYNAMIC、COMPRESSED行格式记录 |
| MLOG_COMP_PAGE_CREATE | 创建一个使用紧凑行格式的页面 |
| MLOG_COMP_REC_DELETE | 删除一条使用紧凑行格式的记录 |
| MLOG_COMP_LIST_START_DELETE | 表示删除页面中从某一条使用紧凑行格式的记录开始，直到到某一条记录为止的所有记录（记录之间是单向列表） |
| MLOG_COMP_LIST_END_DELETE | 见MLOG_COMP_LIST_START_DELETE |
| MLOG_ZIP_PAGE_COMPRESS | 表示压缩一个数据页 |

以MLOG_COMP_REC_INSERT为例

| redo | 说明 |
| :- | :- |
| type |  |
| spaceID |  |
| page number |  |
| n_fields | 该条记录中有多少个字段 |
| n_uniques | 决定该记录唯一的字段数量 |
| field1_len | 第1个字段的占用空间长度 |
| field2_len | 第2个字段的占用空间长度 |
| .... | .... |
| fieldn_len | 第n个字段的占用空间长度 |
| offset | `前一条记录`的偏移量 |
| end_seg_len | 从该字段可以计算出该条记录占用的总的空间大小 |
| 一些记录头信息 | 表示记录头的前4个bits的值以及record_type的值 |
| extra_size | 记录的额外信息占用的空间大小 |
| mismatch index | 为节省redolog大小而设立的字段，暂时忽略含义 |
| 记录的真实数据 |  |

很显然，MLOG_COMP_REC_INSERT的日志记录并没有记录需要修改的PAGE_N_DIR_SLOTS（表示槽数量）、PAGE_HEAP_TOP（本页面还未使用的空间的最小地址）、PAGE_N_HEAP（本页面中的记录数）之类的信息，而只是记录了修改记录的必要的信息，这样是因为在用redo日志做恢复的时候，直接调用插入记录的函数，PAGE_N_DIR_SLOTS之类的数据就会自动被恢复为崩溃前的样子了。

