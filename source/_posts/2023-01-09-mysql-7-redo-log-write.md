---
title: MySQL redo日志写入的过程
categories: MySQL
tags: [MySQL, redo log write]
date: 2023-01-09
---

## redo log block

为了更好管理，把通过MTR生成的redo日志都放在一个大小为512字节的页中，为了区分，这个页就称为block

| 划分 | 字节数 | 作用 |
| :- | -: | :- |
| log block header | 12字节 |  |
| log block body | 496字节 | 存redo数据 |
| log block trailer | 4字节 |  |


### log block header 12个字节
- LOG_BLOCK_HDR_NO，block编号，4字节；
- LOG_BLOCK_HDR_DATA_LEN，block中空余空间的开始位置，2字节，也是block已写数据的长度，初始为12即header的长度；
- LOG_BLOCK_FIRST_REC_GROUP，该block中第一个MTR生成的第一条redo日志的偏移量，即快速找到从哪里开始是新的一组redo的标记，如果一个MTR产生的redo跨越多个block，则最后一个block的LOG_BLOCK_FIRST_REC_GROUP也表示这个MTR的redo结束的位置，也是下个MTR redo开始的位置；
- LOG_BLOCK_CHECKPOINT_NO，checkpoint的序号

### log block trailer 4个字节

- LOG_BLOCK_CHECKSUM，block校验和的值，4字节

## redo 日志缓冲区

redo日志也不是直接写磁盘，也是先写内存缓冲区，然后再从缓冲区落盘

### 写log buffer

redo log buffer 简称 `log buffer`，`按照log block进行划分`，每次log buffer 是按照一个MTR产生的log进行写入，每个MTR先将自己的redo保存在自己的一块buffer上，MTR完成时，再将本次MTR的redo复制到log buffer上，MTR是写redo log的最小粒度。一个事务存在多个MTR的可能，多个事务的多个MTR的redo日志可能存在MTR为粒度的交叉写。

`buf_free全局变量`，用来标记下一次写redo的位置。

新版MySQL 8.0 的redo log，写buffer前先计算需要写buffer的空间大小，然后申请从log buffer上按需申请一块空间区间，多个MTR就可以做到并行写log buffer了

### log buffer落盘

落盘的时机
- log buffer空间不足时，设计认为写满log buffer的50%左右就落盘；
- 事务提交时，数据页面可以不用立即落盘，但是为了事务的持久性，事务提交时，需要等待本事务对应的redo日志落盘；
- 后台有一个线程，大约以每秒1次的频率，将log buffer上的redo日志刷新到磁盘上；
- 服务正常退出时；
- 做checkpoint时。

## redo日志文件组

```sql
SHOW VARIABLES LIKE "%datadir%";
```
可以看到有`ib_logfile0`和`ib_logfile1`的文件，即redo日志文件。

相关配置参数

| 配置项 | 说明 |
| :- | :- |
| innodb_log_group_home_dir | redo日志文件的保存目录，默认数据目录 |
| innodb_log_file_size | 每个ib_logfile的大小上限值 |
| innodb_log_file_in_group | ib_logfile的个数，默认2个，最后一个ib_logfile写满后，回到ib_logfile0覆盖写 |

## redo日志文件格式

- redo日志文件也是由512字节的block组成，log buffer落盘本质就是把redo block写到ib_logfile文件中；
- ib_logfile前4个block（2048字节）为管理信息block，之后的block才是redo日志的block；

redo日志block的结构，前面已述，此处ib_logfile中前4个特殊block的结构如下

- log file header
	| 字段 | 说明 |
	| :- | :- |
	| LOG_HEADER_FORMAT | 4字节，redo日志的版本，5.7.22中永远为1 |
	| LOG_HEADER_PAD1 | 4字节，字节填充，没有意义 |
	| LOG_HEADER_START_LSN | 8字节，本ib_logfile 2048字节后真正的redo日志的开始的lsn值 |
	| LOG_HEADER_CREATOR | 32字节，标记创建本日志文件的创建者，实际内容为MySQL的版本号例如"SQL 5.7.22"，或者备份工具mysqlbackup的"ibbackup"和创建时间 |
	| LOG_BLOCK_CHECKSUM | 4字节，本block的检验和值，redo日志的block也有该值，所有block都有该值 |
- checkpoint1
	| 字段 | 说明 |
	| :- | :- |
	| LOG_CHECKPOINT_NO | 8字节，服务器执行checkpoint的编号，每执行一次加1 |
	| LOG_CHECKPOINT_LSN | 8字节，服务器执行checkpoint结束时的lsn号，系统崩溃后恢复从该lsn开始 |
	| LOG_CHECKPOINT_OFFSET | 8字节，LOG_CHECKPOINT_LSN的lsn值对应的redo日志在日志文件组中的偏移量 |
	| LOG_CHECKPOINT_LOG_BUF_SIZE | 8字节，服务器在执行checkpoint时对应的log buffer的大小 |
	| LOG_BLOCK_CHECKSUM | 4字节，本block的校验和 |
- 未使用
- checkpoint2，结构与checkpoint1一样，`为什么有checkpoint2？`

系统checkpoint的信息只保存在日志文件组的第一个日志文件中

