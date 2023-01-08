---
title: MySQL基于规则的优化
categories: MySQL
tags: [MySQL, optimization]
date: 2023-01-09
---

## 条件简化

### 去除没用的括号

例如

```sql
select * from t1,(t2, t3) ...
select * from t1,t2,t3
```

### 常量传递

例如

```
a = 5 and b > a
```

可优化为

```
a = 5 and b > 5
```


### 移除没用的条件

例如

```
(a < 1 and b = b) or (a = 6 or 5 != 5)
```

优化为

```
a < 1 or a = 6
```

### 表达式计算

执行查询前，如果表达式中只包含常量的话，会被先计算，例如

```
a = 5 + 1
```

优化为

```
a = 6
```

而例如
 
`ABS(a) > 5`、`-a < -8` 带上函数后，item无法直接被优化，所以索引列的条件应该尽量以单独的形式出现在表达式中


### having子句和where子句合并

having 中没有出现sum或者max之类的聚合函数时，就会把having子句和where子句合并起来

### 常量表检测

常量表的定义

- 查询的表中只有1条或0条记录
- 使用主键或者唯一二级索引进行等值匹配作为搜索条件的表

例如 t1是常量表
```sql
select t1.*,t2.* from t1,t2 on t1.c1 = t2.c2 where t1.pkey = 1;
```

t1表示常量表，在分析t2表查询的代价之前，会先查询t1在查询条件上相应的值,优化查询为

```
select t1上的所有列常量值, t2* from t1,t2 on t1.c1在pkey1上的常量值 = t2.c2
```


## 外连接消除


内连接可以通过调整表连接的顺序（驱动表扇出记录数应该尽量少），达到降低成本的目的。
外连接的结果是，驱动表符合条件的记录都要显示，显示驱动表记录若无被驱动表的记录与之匹配，则对应显示记录的被驱动表字段被填充为null，若在where条件中出现被驱动表显示的某一列不能为null，则可将外连接优化为内连接。

```sql
select t1.*,t2.* from t1 left join t2 on t1.col1 = t2.col1 where t2.col2 = 2;
```

可优化为

```sql
select t1.*,t2.* from t1 inner join t2 on t1.col1 = t2.col1 where t2.col2 = 2;
```

## 子查询优化

### 语法

- select子句中的子查询

	```sql
	select (select m1 from t1 limit 1);
	```

- from子句中的子查询

	```sql
	select m1, m2 from (select a+1 as m1, b+1 as m2 from t1 where b > 2) as t2;
	```

	放在from子句中的子查询称为`派生表`

- where或on中的子句中的子查询
	
	```sql
	select m1, m2 from t1 where m1 in (select m1 from t2);
	```

- order by子句中的子查询
	略
- group by子句中的子查询
	略


### 子查询返回结果

- 标量子查询，返回一行一列
- 行子查询，返回一行多列
- 列子查询，返回一列多行
- 表子查询，返回多行多列


### 与外层关系

- 不相关子查询，子查询可以独立运行

	略

- 相关子查询，子查询执行依赖外层查询

	```sql
	select * from t1 where m1 in (select m2 from where n1 = n2);
	```

### 子查询在布尔表达式中的使用

- 比较操作 >、 <、 =、 >=、<=、 <>、 !=、<=>  带子查询 (子查询结果单行的情况)

	```sql
	select * from t1 where m1 < (select min(m2) from t2);    	-- 子查询必须时标量子查询
	select * from t1 where (m1, n1) = (select m2, n2 from t2);	-- 子查询必须时行子查询
	```

- [NOT] IN / ANY / SOME / ALL 带子查询（子查询结果可能多行的情况）

	```sql
	select * from t1 where m1 > any (select m2 from t2); 	-- m1 大于m2的最小值则为ture
	-- some等同于any
	select * from t1 where m1 > all (select m2 from t2); 		-- m1 大于m2的最大值则为ture
	```


- EXISTS 带子查询

	```sql
	select * from t1 where exists (select 1 from t2); 		-- exists 只需要知道是否存在记录，不需要知道记录的具体值
	```

### 子查询语法注意事项

- 子查询必须用括号括起来
- select 中的子查询必须时标量子查询
- 想要得到标量子查询或者行子查询时，又不能保证结果只有1条记录时，就使用limit语句
- [NOT] IN / ANY / SOME / ALL 子查询不允许带LIMIT
- 在子查询中使用 order by 、distinct、没有聚集函数和hanving子句的group by毫无意义。
- 不允许对表的增删改的同时还进行子查询，即增删改的语句中不能存在子查询。


## 子查询的优化

### IN子查询优化

#### 物化表

- IN的子查询的结果集若很大
- IN子查询的外层查询如果使用全表扫描，IN的执行也将非常耗时
- 将IN子查询的结果存到临时表（该过程称为`物化`），该临时表会被去重，一般使用MEMORY作为存储引擎，且会为该表建立`哈希索引`。
- 如果子查询的结果集非常大，超过了tmp_table_size或者max_heap_table_size的值，临时表会转为使用基于磁盘的存储引擎，索引类型页转为使用`B+树索引`。


#### 将物化表转化为连接，计算连接顺序的成本

例如

```sql
select * from s1 where key1 in (select common_field from s2 where key3 = 'a');
```


```sql
-- select common_field from s2 where key3 = 'a 物化为 materialized_table
select s1.* from s1 inner jion materialized_table on key1 = m_val;
```

#### 将子查询转化为半连接

即不将子查询先转为物化表再转连接，直接将子查询转为连接，例如

```sql
select * from s1 where key1 in (select common_field from s2 where key3 = 'a');
```

```sql
select s1.* from s1
	semi join s2 on s1.key1 = s2.common_field
	where s2.key3 = 'a';
```

##### 半连接的执行过程

```sql
select s1.* from s1
	semi join s2 on s1.key1 = s2.common_field
	where s2.key3 = 'a';
```

1. Table pollout，若IN的列是子查询的主键或唯一二级索引，则不存在一条匹配多条的情况，可以将子查询直接拉到外层查询进行合并。
2. Duplicate Weedout，消除重复，建立临时表保存临时记录，利用主键保证不会出现重复记录，利用加入连接的记录到临时表成功或失败，达到去重的目的（加入临时表失败的记录直接丢弃不显示到结果中）。
	这种使用临时表去重的方式即Duplicate Weedout
3. LooseScan，当IN的列是s2的非唯一二级索引时，将s2作驱动表，扫IN的s2的非唯一二级索引，所有重复的key值中做第一次匹配，之后的重复key值直接跳过。
4. semi-join materialization 半连接物化，如果IN中的子查询是不相关子查询，可以将子查询先物化，并去重，使物化表中的记录不重复，然后外层表和物化表连接。
5. First Match，最原始通用的方法，外层表取一条记录，到子查询表中匹配记录，匹配到第一条记录则立即停止匹配并将外层表的记录保存到显示结果中，然后开始下一条外层表的记录到子查询表的匹配；若没有匹配到记录则丢弃该外层表记录。

##### 半连接使用的条件

1. 子查询必须是与IN操作符组成的布尔表达式，并且出现在外层查询的where或on子句中；
2. 该子查询必须是单一的查询，不能是uinon起来的查询；
3. 该子查询不能包含 group by 或者 having语句或者聚集函数；
4. 外层可以有其他的搜索条件，只不过必须使用AND操作符与IN子查询的搜索条件连接起来。

##### 不适用于半连接的情况

1. 子查询所在的IN子句被使用OR与其他条件连接；
2. 使用NOT IN 而不是IN；
3. select子句中的IN子查询；
4. 子查询中包含groupby、having或者聚合函数的情况（要定外查询执行使才能计算，子查询无法物化）；
5. 子查询中包含unoin的情况；

##### 不能优化为半连接的IN子查询的执行

1. 对于不相关子查询，先物化再连接
2. 对于where 或者on 中相关或不相关的IN子查询都可以尝试转化为EXISTS子查询。例如

	```sql
	outer_expr in (select inner_expr from ... where subquery_where)
	```
	转化为
	```sql
	exists (select inner_expr from ... where subquery_where and inner_expr = outer_expr)
	```

### ANY / ALL 子查询优化

若子查询是不相关子查询，则可以优化有

- a < ANY (select a ....) 可以转为 a < (select max(a) ...)
- a > ANY (select a ....) 可以转为 a > (select min(a) ...)
- a < ALL (select a ....) 可以转为 a < (select min(a) ...)
- a > ALL (select a ....) 可以转为 a > (select max(a) ...)


### [NOT] EXISTS 子查询优化

- 如果是不相关子查询，则先执行子查询，得到结果TRUE或者FALSE，并将结果代入外层查询；
- 如果是相关子查询，则只能从外层查询逐条拿记录与子查询表进行匹配；

### 对于派生表（from中的子查询）

先考虑合并，合并不行再物化

- 把派生表和外层查询合并；

	```sql
	select * from (select * from t ...) as drv;
	select * from t ...;

	select * from (select * from s1 where key1 = 'a') as d_s1 inner join s2 on d_s1.key1 = s2.key1 where s2.key2 = 1;
	select * from s1 inner join s2 on s1.key1 = s2.key1 where s2.key2 = 1 and s1.key1 = 'a';
	```

- 派生表不可和外层查询合并的情况

	- 派生表中聚集函数，例如MAX、MIN、SUM、AVG等
	- DISTINCT
	- GROUP BY
	- HAVING
	- LIMIT
	- UNION 或者 UNION ALL
	- 派生表查询select子句中有另一个子查询

- 如果派生表不可被合并，再尝试将派生表物化，但是多出了物化表的创建和访问的成本（by the way 延迟物化，真正用到的时候才执行物化，避免无必要的物化）；
