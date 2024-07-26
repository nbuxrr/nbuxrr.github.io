---
title: PostgreSQL 13
categories: PostgreSQL
tags: [PostgreSQL]
date: 2024-06-27
---

## 起源

PostgreSQL 是基于加州大学伯克利分校计算机科学系开发的 POSTGRES 4.2版本的对象关系数据库管理系统 （ORDBMS），是原始 Berkeley 代码的开源分支。

- 1986年 POSTGRES开始
- 1987年 第一个版本投入使用
- 1988年 ACM-SIGMOD会议展出
- 1989年6月 发布给外部用户使用
- 1990年6月 第2个版本发布
- 1991年 第3个版本
- 最后Illustra Information Technologies（后并入Informix，现在归IBM）将之商业化
- 1993年 Berkeley POSTGRES 项目以 4.2 版正式结束

## 支持情况

支持的SQL标准

- 复杂查询
- 外键
- 触发器
- 可更新视图
- 事务完整性
- 多版本控制（MVCC）

可自定义扩展

- data types 数据类型
- functions 函数
- operators 操作符/算子
- aggregate functions 聚合函数
- index methods 索引方法 
- procedural languages

## 版本信息

```sql
SELECT version();
```

```bash
postgres --version
psql --version
```

## 平台信息

- 内核名称版本
- C 库
- 处理器
- 内存信息

## 安装

- 安装不需要root权限

## 架构基础

- 服务器进程，用于管理数据库文件，接受从客户端应用程序到数据库的连接，并代表客户端执行数据库操作。数据库服务器程序称为 postgres 。
- 要执行数据库操作的用户客户端（前端）应用程序。客户端应用程序在性质上可以非常多样化
- 与客户端/服务器应用程序一样，客户端和服务器可以位于不同的主机上。在这种情况下，它们通过 TCP/IP 网络连接进行通信。
- PostgreSQL 服务器可以处理来自客户端的多个并发连接。为了实现这一点，它为每个连接启动（“分叉”）一个新进程。从那时起，客户端和新服务器进程进行通信，而无需原始 postgres 进程的干预。因此，主服务器进程始终处于运行状态，等待客户端连接，而客户端和关联的服务器进程来来去去。（当然，所有这些对用户来说都是不可见的。我们在这里只是为了完整起见而提及它。

## 基本操作

### 创建、删除数据库

```bash
createdb mydb
/usr/local/pgsql/bin/createdb mydb
dropdb mydb
```

### 访问数据库

```bash
psql mydb
```

```bash
psql (13.15)
Type "help" for help.

mydb=>
或者
mydb=#
```

### 查看帮助

```bash
mydb=> \h

-- 更多命令
mydb=> \?
```

### 退出

```bash
mydb=> \q
```

### 执行sql文件

`\i` 表示从文件中读取sql，`-s`表示单步模式，表示在将每个sql发送给服务器之前暂停。

```bash
$ psql -s mydb
...
mydb=> \i basics.sql
```

### 编译

```bash
cd .../src/tutorial
make
```

### 创建表

```sql
CREATE TABLE weather (
    city            varchar(80),
    temp_lo         int,           -- low temperature
    temp_hi         int,           -- high temperature
    prcp            real,          -- precipitation
    date            date
);

CREATE TABLE cities (
    name            varchar(80),
    location        point
);

```

### 插入数据

```sql
INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');
COPY weather FROM '/home/user/weather.txt';
INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);
```

### 从文件中加载大量数据

```sql
COPY weather FROM '/home/user/weather.txt';
```

### select

```sql
SELECT * FROM weather;
SELECT city, temp_lo, temp_hi, prcp, date FROM weather;
SELECT city, (temp_hi+temp_lo)/2 AS temp_avg, date FROM weather;
SELECT * FROM weather
    WHERE city = 'San Francisco' AND prcp > 0.0;
SELECT * FROM weather
    ORDER BY city;
SELECT * FROM weather
    ORDER BY city, temp_lo;
SELECT DISTINCT city
    FROM weather;
SELECT DISTINCT city
    FROM weather
    ORDER BY city;

-- 带连接
SELECT *
    FROM weather, cities
    WHERE city = name;
SELECT city, temp_lo, temp_hi, prcp, date, location
    FROM weather, cities
    WHERE city = name;
SELECT weather.city, weather.temp_lo, weather.temp_hi,
       weather.prcp, weather.date, cities.location
    FROM weather, cities
    WHERE cities.name = weather.city;
SELECT *
    FROM weather INNER JOIN cities ON (weather.city = cities.name);
SELECT W1.city, W1.temp_lo AS low, W1.temp_hi AS high,
    W2.city, W2.temp_lo AS low, W2.temp_hi AS high
    FROM weather W1, weather W2
    WHERE W1.temp_lo < W2.temp_lo
    AND W1.temp_hi > W2.temp_hi;
SELECT *
    FROM weather w, cities c
    WHERE w.city = c.name;
-- 聚合函数
SELECT max(temp_lo) FROM weather;
-- SELECT city FROM weather WHERE temp_lo = max(temp_lo);     WRONG
SELECT city FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city;
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40;
SELECT city, count(*), max(temp_lo)
    FROM weather
    WHERE city LIKE 'S%'            -- (1)
    GROUP BY city;
SELECT city, count(*) FILTER (WHERE temp_lo < 45), max(temp_lo)
    FROM weather
    GROUP BY city;
```

### update和delete

```sql
UPDATE weather
    SET temp_hi = temp_hi - 2,  temp_lo = temp_lo - 2
    WHERE date > '1994-11-28';
DELETE FROM weather WHERE city = 'Hayward';
```

## 高级功能

### 视图

用于封装查询

```sql
CREATE VIEW myview AS
    SELECT name, temp_lo, temp_hi, prcp, date, location
        FROM weather, cities
        WHERE city = name;

SELECT * FROM myview;
```

### 外键

确保引用存在

```sql
CREATE TABLE cities (
        name     varchar(80) primary key,
        location point
);

CREATE TABLE weather (
        city      varchar(80) references cities(name),
        temp_lo   int,
        temp_hi   int,
        prcp      real,
        date      date
);

INSERT INTO weather VALUES ('Berkeley', 45, 53, 0.0, '1994-11-28');
-- ERROR:  insert or update on table "weather" violates foreign key constraint "weather_city_fkey"
-- DETAIL:  Key (city)=(Berkeley) is not present in table "cities".
```

### 事务

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
UPDATE branches SET balance = balance - 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Alice');
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
UPDATE branches SET balance = balance + 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Bob');
COMMIT;
```

### 窗口函数

OVER子句，window 函数调用始终包含一个子 OVER 句，该子句紧跟在窗口函数的名称和参数之后。这就是它在语法上与普通函数或非窗口聚合的区别。利用窗口函数可以实现一些复杂好用的计算（和平均数对比、排名、和总数对比占百分比、累计数列）

```sql
-- 将每个员工的工资与其部门的平均工资进行比较
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;
SELECT depname, empno, salary,
       rank() OVER (PARTITION BY depname ORDER BY salary DESC)
FROM empsalary;
SELECT salary, sum(salary) OVER () FROM empsalary;
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
SELECT depname, empno, salary, enroll_date
FROM
  (SELECT depname, empno, salary, enroll_date,
          rank() OVER (PARTITION BY depname ORDER BY salary DESC, empno) AS pos
     FROM empsalary
  ) AS ss
WHERE pos < 3;
-- 当查询涉及多个窗口函数时，可以使用单独的 OVER 子句写出每个窗口函数，但如果需要对多个函数使用相同的窗口行为，则会重复且容易出错。相反，可以在 WINDOW 子句中命名每个窗口行为，然后在 OVER 中引用。
SELECT sum(salary) OVER w, avg(salary) OVER w
  FROM empsalary
  WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
```

### 继承

表定义的时候使用继承，可以包含被继承表的所有列（可以同时继承多个表）

```sql
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```

## SQL语句

### 语法

#### 字符常量

- 引号字符串常量

    ```sql
    -- 等效于 SELECT 'foobar';
    -- 但无效 SELECT 'foo'      'bar';
    SELECT 'foo'
    'bar';
    ```

- 类似C语言的反斜杠转义字符常量

    ```sql
    \b
    \f
    \n
    \r
    \t
    \o、\oo、\ooo
    \xh、\xhh
    \uxxxx、\Uxxxxxxxx
    ```

- Unicode转义的字符串常量

    ```sql
    -- 标识符 FOO ， foo 和 "foo" 被 PostgreSQL 认为是相同的，但 "Foo" 和 "FOO" 与这三个标识符和彼此不同
    U&"d\0061t\+000061"     -- 等效于"data"
    U&'d\0061t\+000061'     -- 等效于'data'
    U&"\0441\043B\043E\043D" -- 等效于"slon" in Cyrillic letters
    --  UESCAPE 子句指定转义字符
    U&'d!0061t!+000061' UESCAPE '!'
    ...
    ```

- $符引用的字符串常量

    ```sql
    $$Dianne's horse$$
    $SomeTag$Dianne's horse$SomeTag$

    $function$
    BEGIN
        RETURN ($1 ~ $q$[\t\r\n\v\\]$q$);
    END;
    $function$
    ```

- 位字符串常亮

    ```sql
    B'1001'
    ```

- 数值常量

    ```sql
    digits
    digits.[digits][e[+-]digits]
    [digits].digits[e[+-]digits]
    digitse[+-]digits

    42
    3.5
    4.
    .001
    5e2 5e2
    1.925e-3 1.925E-3
    ```

### 运算符

```txt
+ - * / < > = ~ ! @ # % ^ & | ` ?
```

### 数据定义

- 系统列，每个表都有几个系统列，这些列由系统隐式定义。

  - tableoid
  - xmin 此行版本的插入事务的标识（事务 ID）
  - cmin 插入事务中的命令标识符（从零开始）。
  - xmax 删除事务的标识（事务 ID），对于未删除的行版本，则为零。在可见行版本中，此列可能为非零。这通常表示删除事务尚未提交，或者尝试删除已回滚。
  - cmax 删除事务中的命令标识符，或零。
  - ctid 行版本在其表中的物理位置。请注意，尽管 ctid 可用于非常快速地查找行版本，但如果 更新或移动行 VACUUM FULL 版本，则行的版本 ctid 将发生变化。因此 ctid ，作为长期行标识符是无用的。应使用主键来标识逻辑行。

- 列操作

    ```sql
    -- 添加列
    ALTER TABLE products ADD COLUMN description text;
    ALTER TABLE products ADD COLUMN description text CHECK (description <> '');
    -- 删除列
    ALTER TABLE products DROP COLUMN description;
    ALTER TABLE products DROP COLUMN description CASCADE;
    -- 添加约束
    ALTER TABLE products ADD CHECK (name <> '');
    ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
    ALTER TABLE products ADD FOREIGN KEY (product_group_id) REFERENCES product_groups;
    ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
    -- 删除约束
    ALTER TABLE products DROP CONSTRAINT some_name;
    ALTER TABLE products ALTER COLUMN product_no DROP NOT NULL;
    -- 更改列默认值
    ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
    ALTER TABLE products ALTER COLUMN price DROP DEFAULT;
    -- 更改列数据类型
    ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
    -- 重命名列
    ALTER TABLE products RENAME COLUMN product_no TO product_number;
    -- 重命名表
    ALTER TABLE products RENAME TO items;
    ```

- 权限
  
    ```sql
    ALTER TABLE table_name OWNER TO new_owner;
    GRANT UPDATE ON accounts TO joe;
    REVOKE ALL ON accounts FROM PUBLIC;

    ```

### 数据操作

#### INSERT

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);
INSERT INTO products VALUES (1, 'Cheese', 9.99);
INSERT INTO products (product_no, name, price) VALUES (1, 'Cheese', 9.99);
INSERT INTO products (name, price, product_no) VALUES ('Cheese', 9.99, 1);
INSERT INTO products (product_no, name) VALUES (1, 'Cheese');
INSERT INTO products VALUES (1, 'Cheese');
INSERT INTO products (product_no, name, price) VALUES (1, 'Cheese', DEFAULT);
INSERT INTO products DEFAULT VALUES;
INSERT INTO products (product_no, name, price) VALUES
    (1, 'Cheese', 9.99),
    (2, 'Bread', 1.99),
    (3, 'Milk', 2.99);
INSERT INTO products (product_no, name, price)
  SELECT product_no, name, price FROM new_products
    WHERE release_date = 'today';

-- 插入带返回值
INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;
```

#### UPDATE

```sql
UPDATE products SET price = 10 WHERE price = 5;
UPDATE products SET price = price * 1.10;
UPDATE mytable SET a = 5, b = 3, c = 1 WHERE a > 0;
-- 更新带返回值
UPDATE products SET price = price * 1.10
  WHERE price <= 99.99
  RETURNING name, price AS new_price;
```

#### DELETE

```sql
DELETE FROM products WHERE price = 10;
DELETE FROM products;
-- 删除带返回值
DELETE FROM products
  WHERE obsoletion_date = 'today'
  RETURNING *;
```

### 查询

```sql
-- 语法[WITH with_queries] SELECT select_list FROM table_expression [sort_specification]
SELECT a, b + c FROM table1;
-- FROM table1 是一种简单的表表达式：它只读取一个表。 可以省略
SELECT 3 * 4;
SELECT random();

-- with，with还可用来求递归计算
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
), top_regions AS (
    SELECT region
    FROM regional_sales
    WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

### 数据类型

### 函数运算符

### 类型转换

### 索引

### 全文索引

### 并发控制

#### 事务隔离

#### 显式锁

#### 应用级别的数据一致性检查

#### 锁定和索引

