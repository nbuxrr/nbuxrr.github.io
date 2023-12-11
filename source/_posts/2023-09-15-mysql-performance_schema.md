---
title: MySQL Performance Schema
categories: MySQL
tags: [MySQL, pfs, performance schema]
date: 2023-09-15
---

## 参考文档

### PFS 表目录

- [ ] [MySQL :: MySQL 8.0 Reference Manual :: 27.12.1 Performance Schema Table Reference](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-table-reference.html)

### PFS 表分类描述

- [ ]  [MySQL :: MySQL 8.0 Reference Manual :: 27.12 Performance Schema Table Descriptions](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-table-descriptions.html)

## Performance Schema （pfs）

pfs的实现的本质为一个存储引擎，默认开启，可通过`performance_schema`配置参数进行明确打开或关闭

```SQL
-- 查看PFS引擎的信息
SELECT * FROM INFORMATION_SCHEMA.ENGINES WHERE ENGINE='PERFORMANCE_SCHEMA'\G

-- 查看pfs数据库中的表信息
USE performance_schema;
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'performance_schema';
SHOW TABLES FROM performance_schema;             -- 或者
```

## 打开PFS

```SQL
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES';
```

## 举例

```SQL
-- 查看每个线程最近一次的事件
SELECT * FROM performance_schema.events_waits_current;
```

### 接口

### 实现

#### pfs Buffers

#### pfs Engine

#### pfs Tables

## Performance_schema_error_log

## Performance_schema_tables
