---
title: Running Test Cases
categories: MySQL
tags: [MySQL, mtr]
date: 2021-06-15
---

## 参考

[https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_RUNNING_TESTCASES.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_RUNNING_TESTCASES.html)

## 运行test cases

```bash
cd mysql-test
./mysql-test-run.pl                             # 最简单默认执行用例
./mysql-test-run.pl --force --suite=binlog      # 用例不通过时继续其他用例执行
./mysql-test-run.pl test_name                   # 运行指定名称的用例，用例名称不带.test后缀
./mysql-test-run.pl --do-test=test_name_prefix  # 运行指定前缀的名称的一系列用例
./mysql-test-run.pl --suite=suite_name          # 运行suite目录下指定suite名的目录下的所有用例
```

mysql-test-run.pl所有可用参数见 [mysql-test-run.pl — Run MySQL Test Suite](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN_PL.html)

mysql-test-run.pl程序运行MySQL server，为调用mysqltest初始化环境，然后调用mysqltest执行test case。对于每个要运行的测试用例，mysqltest处理诸如```从测试用例文件读取输入```、```创建服务器连接```以及```将 SQL 语句发送到服务器```等操作。

test case文件中使用的语言是```mysqltest可识别的命令和SQL语句的混合```，对于mysqltest```不能识别的行，都当做SQL语句处理```。

运行测试时不需要自己启动MySQL服务程序，mysql-test-run.pl会自动运行所需的一个或多个MySQL server服务，这里所有被服务启动的服务端口都默认在13000以上范围

## 并行方式运行Test cases

### 同时运行多个实例

在同一台机器上可以运行多个mysql-test-run.pl。多个mysql-test-run.pl都会从13000开始使用端口，为避免端口冲突，多个运行之间会协调使用和检查可用性。

从同一个mysql-test目录同时运行多个测试实例是可能的，但存在问题。首先var日志目录会冲突，需要用```--vardir```为每个实例设置不同的日志目录。即便如此，还有```.reject会写入相同目录（r目录）的问题```（如何解决？）

### 一个实例多线程运行

也可以一个mysql-test-run.pl实例多线程并行执行多个test cases，这需要使用```--parallel=线程数```参数，参数值为线程数，特殊的参数值```auto```表示按照系统（cpu）情况自动取一个可能最优的线程数值。也可以使用```MTR_PARALLEL```环境变量作用为parallel选项(```?```)。

包含了```not_parallel.inc```的test cases将在测试的最后（parallel被覆盖为1）逐个被运行。

