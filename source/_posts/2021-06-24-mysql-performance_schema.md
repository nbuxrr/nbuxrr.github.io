---
title: MySQL Performance Schema
categories: MySQL
tags: [MySQL, pfs]
date: 2021-06-25
---

## 配置

```ini
[mysqld]
performance_schema=ON
```

```sql
SHOW VARIABLES LIKE 'performance_schema';
USE performance_schema;
```

默认不会收集所有事件，打开所有事件

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES';
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES';
```

## 表信息分类

- 当前事件、事件历史和统计
- 对象实例（例如文件等）
- 设置

### 事件表

例如

|  |  |
| :- | :- |
| [events_waits_current](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-events-waits-current-table.html) | 包含每个线程当前的waits事件 |
| [events_waits_history](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-events-waits-history-table.html) | 包含每个线程最近的10个waits事件  |
| [events_waits_history_long](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-events-waits-history-long-table.html) | 包含每个线程最近的10000个waits事件 |
| [events_waits_summary_global_by_event_name](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-wait-summary-tables.html) | 记录一段时间内的各种事件的统计信息，通过触发次数和消耗时间可以判断热点 |

### 实例表

例如

|  |  |
| :- | :- |
| [file_instances](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-file-instances-table.html) | 打开的文件的事件及io次数等 |

### 配置表

#### [setup_instruments](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-setup-instruments-table.html)

用于配置和显示监视特性

命名约定

[Section 27.6, “Performance Schema Instrument Naming Conventions”.](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-instrument-naming.html)

控制某一种事件是否被收集

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME = 'wait/synch/mutex/sql/LOCK_mysql_create_db';
```

#### [setup_consumers](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-setup-consumers-table.html)

pfs用收集到的事件更新pfs中的表，所以pfs扮演了consumer的角色, setup_consumers表列出了可用的consumers，及哪些consumers被使能。

consumers里包含了例如

- events_stages 三张表
- events_statements 三张表
- events_transactions 三张表
- events_waits 三张表
- global_instrumentation 全局监视
- thread_instrumentation 线程监视
- statements_digest 语句摘要

### 杂项表

例如 [performance_timers](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-performance-timers-table.html)列出了性能计时器

