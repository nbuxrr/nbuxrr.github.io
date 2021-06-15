---
title: MySQL Test Framework Components
categories: MySQL
tags: [MySQL, mtr]
date: 2021-06-11
---

## 参考

[https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_TESTING_TOOLS.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_TESTING_TOOLS.html)

## 测试框架程序

### 测试框架程序文件

- [mysql-test-run.pl](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN_PL.html) 测试主程序 调用mysqltest测试单个用例（单个测试文件）
- [mysqltest](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST.html) 测试单个用例，被mysql-test-run.pl调用
- [mysql_client_test](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_CLIENT_TEST.html) 用来测试无法被mysqltest测试的MySQL client API
- [mysql-stress-test.pl](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_STRESS_TEST_PL.html) 用于MySQL压力测试
- unit-testing facility 用于创建测试存储引擎或插件的单独的单元测试

### 测试suite程序文件所在目录

- ```mysqltest``` 源码mysqltest.cc在client目录下，编译结果在bin目录下
- ```mysql_client_test``` 源码mysql_client_test.cc在testclients目录下，编译结果在bin目录下
- 其他测试程序 源码在mysql-test目录下，编译结果在```install```目录下的```mysql-test```目录

## 目录结构

install/mysql-test目录结构

    ```bash
    -rw-r--r--.  asan.supp
    drwxr-xr-x.  collections         # 集成与发布测试时使用，保留在源码仓中以供参考
    drwxr-xr-x.  extra
    drwxr-xr-x.  include             # 主要版本default_xxx.cnf文件及一些将被test文件包含的.inc文件
    drwxr-xr-x.  lib                 # 保存了一些.pm .t .pl文件，将作为mysql-test-run.pl的模块被调用
    -rw-r--r--.  lsan.supp
    lrwxrwxrwx.  mtr -> ./mysql-test-run.pl              # 别名或副本
    -rwxr-xr-x.  mysql-stress-test.pl                    # 压力测试
    lrwxrwxrwx.  mysql-test-run -> ./mysql-test-run.pl   # 别名或副本
    -rwxr-xr-x.  mysql-test-run.pl                       # 用来一次测试
    drwxr-xr-x.  r                   # 存放.result期望结果文件、.reject(与.result不一致的)实际结果文件
    -rw-r--r--.  README
    -rw-r--r--.  README.gcov
    -rw-r--r--.  README.stress
    drwxr-xr-x.  std_data            # 包含一些测试使用的数据文件
    drwxr-xr-x.  suite               # 每个子目录代表一个以文件件命名的test suite
    drwxr-xr-x.  t                   # 存放测试输入文件 .cnf .inc .opt .test等文件
    -rw-r--r--.  valgrind.supp
    drwxr-xr-x.  var                 # 存放各种测试结果信息
    ```

### t目录

t包含了测试case的输入文件，对于一个用例ABC可能有文件

| | |
| :- | :- |
| ABC.cnf | 指定测试case的附加配置信息 |
| ABC-client.opt | 提供客户端的配置 |
| ABC-master.opt | 即使没有涉及主从复制，也加master，如果当前运行的server的配置和-master.opt的不一样，mysql-test-run.pl就会重启server；mysql-test-run.pl也会按照opt文件的配置重启server。每个bootstrap变量必须作为--initialize选项的参数，mysql-test-run.pl才能在服务器初始化的时候识别出需要使用的变量|
| ABC-slave.opt | 有主从复制是才需要 |
| ABC.test |  |
| ABC.result |  |
| ABC.combinations | 为每次测试case运行提供选项段 |
| ABC-master.sh | 在main server启动前将被执行，win不支持，将来可能被其他机制替换 |
| ABC-slave.sh | 在slave server启动前将被执行，win不支持，将来可能被其他机制替换 |
| suite.opt | 为所有该suite内的test case提供配置，如果一个test运行多个server，则suite.opt对所有这些server都有效。该文件中的选项会被-master.opt、-slave.opt中的同选项覆盖 |
| disabled.def | 用来配置将被延期或禁止运行的test case，如果由于server有bug致使一些test会失败，想忽略这些test，不被mysql-test-run.pl执行，可以将这些test列到这个文件中 |


cnf文件可以包含基础或其他配置文件

```ini
!include include/default_my.cnf

[mysqld.1]                      # 可以使用.1/.2等组后缀名区分不同server组，每个server启动的时候都默认带有组后缀名(defaults-group-suffix)
Options for server mysqld.1

[mysqld.2]
Options for server mysqld.2

[mysqltest]                     # 测试客户端的配置选项
ps-protocol

...
...
...

[ENV]                           # 指定测试case的环境变量
SERVER_MYPORT_1= @mysqld.1.port # 定义一个SERVER_MYPORT_1环境变量，值为上段mysqld.1中的port的值
SERVER_MYPORT_2= @mysqld.2.port
```

### r目录

可能包含的文件

|  |  |
| :- | :- |
| ABC.result | ABC.test文件的期望输出内容 |
| ABC.reject | 如果test case是由于输出不一致而失败的（非其他原因失败），则.reject文件中包含test case的实际输出 |

如果--check-testcases选项打开，若test文件没有对应result文件时，mtr将对其标记为失败。

> --check-testcases作用
> 
> 检查测试用例是否有副作用。 这是通过在每个测试用例之前和之后检查系统状态来完成的。 如果有任何差异，则测试用例因此被标记为失败。
> 
> 类似地，当启用 --check-testcases 选项时，MTR 会对丢失的 .result 文件进行额外检查，并且没有相应 .result 文件的测试用例被标记为失败。
> 
> 默认情况下启用此检查。 要禁用它，请使用 --nocheck-testcases 选项。

### var目录

用来存放各种测试运行中生成的结果文件：log文件、temp文件、trace文件、Unix socket文件等。这个目录不能被同时跑的测试所共享。

### suite目录

该目录下每个子文件夹代表一个与文件夹同名的test suite。
每个test suite可能包含如下部分
- t目录
- r目录
- include目录
- 一个combinations格式的文件，为每次测试运行提供配置段([Controlling the Binary Log Format Used for an Entire Test Run](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_CONTROLLING_BINARY_LOG_FORMAT.html))
- 一个my.cnf格式的文件，为本suite中的所有测试提供配置项，同配置项内容会被test_name.cnf文件的所覆盖。

### collections目录

此目录包含我们在集成和发布测试期间运行的测试运行的集合。 

这些文件在此上下文之外没有直接用处，但需要成为源码仓的一部分并包含在内以供参考。 

每个文件包含零行或多行，每行都调用一次 mysql-test-run.pl。

这些调用是这样编写的
- 假设perl在环境搜索路径中
- 原则上任何集合都可以作为shell脚本或批处理文件运行
- mysql-test目录是当前工作目录。

> 每行格式例如
> perl mysql-test-run.pl --force --timer  --big-test --testcase-timeout=90  --parallel=auto --experimental=collections/default.experimental --comment=normal-big  --vardir=var-normal-big  --report-features --skip-test-list=collections/disabled-daily.list --unit-tests-report

### unittest目录

单元测试相关目录，相关于存储引擎和插件的附加文件可能存在于storage或plugin目录的子目录下。

## 执行测试与评估

在顶层Makefile中有多个targets可用于运行测试集。

make test只运行单元测试(```?```),其他测试集见Makefile文件。
一个“test case”是单个文件，case中可能包含多个测试命令，任意一个测试命令没有产生预期的结果都认为整个测试用例失败（预期结果包括测试某种预期的错误，例如语法错误）。

test case的输出内容(test result, 和.result文件进行diff)包括

- 输入的SQL语句及其输出信息
- mysqltest命令(例如echo、exec)的输出结果，而命令本身不输出到结果。

disable_query_log和enable_query_log命令控制是否logging输出SQL语句(.result```?```)
disable_result_log和enable_result_log命令控制是否logging输出SQL语句的结果包括warning、error信息(.result```?```)

## mysqltest文件

mysqltest默认从其标准输入读入test case，也可以使用--test-file或-X选项显式地给定一个test case文件名。

mysqltest默认向其标准输出写入test case的结果，也可以使用--result-file或-R选项来显式地指定result文件的位置。
这个配置项和--record选项共同确定mysqltest如何处理一个test case的实际和预期测试结果。

- 如果一个test没有输出result，mysqltest会带着error信息退出，除非--result-file指定的文件名为空
- 如果--result-file没有给出，mysqltest将发送结果到标准输出
- 如果有--result-file但没有--record选项mysqltest从指定的文件中读取期望的result文件，并且和期望的result结果做比较。如果结果不匹配，mysqltest就将实际结果写到log目录下.reject文件中，并error退出，只要有可用的diff工具，就会再输出实际与预期的diff结果。
- 如果--result-file和--record都给出了，则mysqltest将用实际测试结果更新到给出的文件中，该文件不需要预先存在(最开始的result文件自动生成)。

mysqltest程序本身对t/r目录一无所知，这些目录下的文件，约定由mysql-test-run.pl使用，由该pl文件为每个test case以适当的参数调用mysqltest，告诉mysqltest从哪里读取输入和向哪里输出。

## MySQL测试框架的要求

- 需要C++运行时库

    mysqltest和mysql_client_test程序是用C++编写的，可以在任何可以编译MySQL本身的系统上使用，或者可以使用二进制MySQL发行版。

- 需要perl

    测试框架的其他部分，例如 mysql-test-run.pl 是 Perl 脚本，应该在安装了 Perl 的系统上运行。

- 需要diff

    mysqltest使用diff程序来比较预期和实际测试结果。 如果未找到diff，mysqltest会写入错误消息并转储 .result 和 .reject 文件的全部内容，以便您可以尝试确定测试未成功的原因。 如果您的系统没有diff，您可以从以下站点之一获取它：
    [http://www.gnu.org/software/diffutils/diffutils.html](http://www.gnu.org/software/diffutils/diffutils.html)
    [http://gnuwin32.sourceforge.net/packages/diffutils.htm](http://gnuwin32.sourceforge.net/packages/diffutils.htm)

- 目录不能带空格

    如果从完整路径包含空格字符的目录中启动，mysql-test-run.pl 将无法正常运行，因为这将会使在所有引用这个路径的不同上下文中正确处理它变得很复杂。

## mysql-test-run.pl文件

参考 [https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN_PL.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQL_TEST_RUN_PL.html)


## test case测试不通过处理

- 尽可能多收集错误信息，再上报bug

    [https://dev.mysql.com/doc/refman/8.0/en/bug-reports.html.](https://dev.mysql.com/doc/refman/8.0/en/bug-reports.html.)

- 确保包含了mysql-test-run.pl的输出、var/log中所有的.reject文件以及diff报告
- 检查单独跑这个用例是否失败

    ```
    cd mysql-test
    ./mysql-test-run.pl test_name
    ```
    如果还失败，再继续检查是否自己编译MySQL使用了--with-debug选项且运行mysql-test-run.pl是否使用了--debug选项。如果这样还失败，则连带var/tmp/master.trace文件一起上报（顺带包含系统描述、mysqld版本、如何编译该mysqld文件）。

- 运行mysql-test-run.pl带--force选项，查看是否还有其他test case失败。
- Result length mismatch或者Result content mismatch，就表示可能有bug或该mysql版本在某些情况下产生的结果略有不同。
- 如果一个test case完全失败，应该**检查var/log目录中的logs文件中的错误信息**。
- 如果自己编译的debug版本MySQL，出现test case失败的情况，可以运行**mysql-test-run.pl加--gdb和--debug选项来调试失败原因**。在CMake时可以使用-DWITH_DEBUG来指定编译debug版本的MySQL

