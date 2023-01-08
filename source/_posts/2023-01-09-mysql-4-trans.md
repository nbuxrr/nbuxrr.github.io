---
title: MySQL 事务
categories: MySQL
tags: [MySQL, transaction]
date: 2023-01-09
---

## ACID

原子性、一致性、隔离性、持久性

## 事务的状态

- 活动的 active

执行中

- 部分提交的 partially committed

事务最后一个操作已经执行完成，还没有刷到磁盘

- 失败的 failed

- 中止的 aborted

- 提交的 committed

## 事务的语法

### 开启事务

```sql
BEGIN;
-- 或者
BEGIN WORK;  -- WORK可省略
```


```
START TRANSACTION;		-- 默认读写模式
START TRANSACTION READ ONLY;	-- 只读模式（外部事务可见的表等只读），事务自己的临时表可读可写
START TRANSACTION READWRITE;	-- 默认
START TRANSACTION READ ONLY WITH CONSISTENT SNAPSHOT;		-- 一致性只读模式
START TRANSACTION READWRITE WITH CONSISTENT SNAPSHOT;	-- 一致性读写模式
```

### 提交事务

```sql
COMMIT;
-- 或者
COMMIT WORK; -- WORK可省略
```

### 手动中止事务

```sql
ROLLBACK
-- 或者
ROLLBACK WORK  -- WORK 可省略
```

### 支持事务的存储引擎

- InnoDB
- NDB

若在事务中修改不支持事务引擎（MyISAM）的表，事务回滚时，支持事务的表的撤销修改，但是不支持事务的表的修改无法撤销。

### 自动提交

系统变量 autocommit

- 默认为ON，即若没有显式的使用BEGIN或START TRANSACTION，每条语句都是一个独立的事务。
- 为OFF，则写入的多条语句算作同一个事务，直到我们显示的执行COMMIT语句提交事务或者ROLLBACK回滚事务。

### 隐式提交

当使用`BEGIN`或`START TRANSACTION`显式开启事务时或者`autocommit`为OFF时，正常应该使用COMMIT提交事务，但是当有一些情况时会隐式提交

- 定义或修改数据库对象时，DDL语句（CREATE、ALTER、DROP等语句）会隐式提交前面语句所属的事务；
- 隐式使用或修改mysql数据库中的表时，会自动提交前面语句所属的事务；
- 一个事务还没提交或回滚就又执行`BEGIN`或`START TRANSACTION`时，会自动提交前面语句所属的事务；
- 当系统变量autocommit的值由OFF置为ON时，自动提交之前语句所属的事务；
- 使用LOCK TABLES 或者 UNLOCK TABLES ...
- 加载数据的语句，例如LOAD DATA语句向数据库批量导入数据时
- 关于MySQL复制的一些语句，`START SLAVE`、`STOP SLAVE`、`RESET SLAVE`、`CHANGE MASTER TO`等
- 其他语句，ANALYZE TABLE、CACHE INDEX、CHECK TABLE、FLUSH、LOAD INDEX INTO CACHE、OPTIMIZE TABLE、REPAIR TABLE、RESET等

### 保存点

用来在回滚事务的时候不必全部回滚然后结束事务，可以回滚到指定点继续事务

```
-- 创建保存点
SAVEPOINT 保存点名称;

-- 回滚到指定保存点
ROLLBACK TO 保存点名称;
```












