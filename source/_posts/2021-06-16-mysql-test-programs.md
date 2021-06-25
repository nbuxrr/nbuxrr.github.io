---
title: MySQL Test Programs
categories: MySQL
tags: [MySQL, mtr]
date: 2021-06-15
---

## 参考

- [https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST_LANGUAGE_REFERENCE.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST_LANGUAGE_REFERENCE.html)
- [https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST_PROGRAMS.html](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST_PROGRAMS.html)

|  |  |
| :- | :- |
| mysql-test-run.pl | perl脚本文件，mtr测试的主程序，负责server环境的初始化、server的启动、调用mysqltest程序进行test case的测试 |
| mysqltest | 二进制可执行文件，被mysql-test-run.pl调用，用于test case的执行和输出结果比较 |
| mysql_client_test | 二进制可执行文件，用于无法被mysqltest测试的MySQL client API的测试 |
| mysql-stress-test.pl | perl脚本文件，用于MySQL server压力测试 |

## mysqltest功能

- 可以发送SQL语句到MySQL server被执行
- 可以执行外部shell命令(```?```)
- 可以测试SQL语句或shell命令的结果是否符合预期
- 可以连接到一个或多个独立的mysqld服务器，并在连接之间切换

[一堆参数，略](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_MYSQLTEST.html)

## mysql-test-run.pl

### test cases指定

```bash
cd mysql-test

## 每个test_name参数命名一个test case。 测试名称对应的test case文件是t/test_name.test。 没有指定test_name参数时，执行t目录下所有.test文件。
mysql-test-run.pl [options] [test_name] ...

## 如果没有给定后缀名，则假定后缀名为.test,如果带前导目录，则忽略任何前导目录（总之别乱用，用test_name就可以了）
mysql-test-run.pl mytest
mysql-test-run.pl mytest.test
mysql-test-run.pl t/mytest.test

## suite名称作为test_name的一部分,对应test case文件为suite/suite_name/t/test_name.test，mysql-test/t目录下的test cases的suite_name为隐式的"main".
mysql-test-run.pl [suite_name.]test_name[.suffix]

## 如果默认suite列表中包含多个同名的test case，则不指定suite_name会执行默认suite列表下的所有test_name的test case （默认suite列表在哪里配置？）
mysql-test-run.pl test_name

```

### 测试环境初始化

运行测试前为了初始化设置，mysql-test-run.pl需要调用mysqld带上--initialize参数和--skip-grant-tables（不启用授权表，使用后可任意用户名密码登录）选项。如果编译MySQL时用了-DDISABLE_GRANT_OPTIONS,则 --initialize、 --skip-grant-tables、--init-file参数将不可用。则用```MYSQLD_BOOTSTRAP```环境变量来指定可用这些选项的mysqld的完整路径。这个只用来做初始化，不在测试运行时使用。（主要就是用来初始化数据库文件？）

如果 --init-file 被禁用，init_file 测试将失败。 在这种情况下，这是预期的失败。

### windows平台

windows平台上运行mysql-test-run.pl，需要安装Cygwin和Perl运行库，并需要安装脚本依赖的模块。运行脚本如下

```bash
cd mysql-test
export MTR_VS_CONFIG=debug  ## 或者使用--vs-config选项
./mysqltest-run.pl --force --timer
./mysqltest-run.pl --force --timer --ps-protocol
```

### 环境变量

mysql-test-run.pl中使用的环境变量，一些是mysql-test-run.pl外部设置，用于mysql-test-run.pl，一些是由mysql-test-run.pl内部设置，在测试运行时使用。

| 环境变量名 | 作用 |
| :- | :- |
| MTR_BUILD_THREAD | 如果设置，则用于定义server的端口号（需要间接计算得到端口号） |
| MTR_MAX_PARALLEL | 当--parallel=auto时，定义可并发的最大线程数 |
| MTR_MEM | 如果设置（不管设置为什么内容），将在内存上用tmpfs或ramdisk方式运行测试，提升mtr测试的速度(```测试失败信息能否保留出来?```) |
| MTR_**NAME**_TIMEOUT | 对应于命令行选项--name-timeout，**NAME**可替换为**TESTCASE**、**SUITE**（单位为分钟），还可替换为**START**、**SHUTDOWN**、**CTEST**(单位为妙)。 **MTR_CTEST_TIMEOUT**用于ctest单元测试 |
| MTR_PARALLEL | 如果定义了，则定义并行执行的线程数，同--parallel选项 |
| MTR_PORT_BASE | 如果定义了，直接定义用于server的端口号范围 |
| MTR_RECORD | 如果MTR是带--record选项运行的，则该环境变量值为1，否则为0，用于用例执行时判断是否需要更新.test文件 |
| MTR_UNIQUE_IDS_DIR | 获取空闲端口的方法是通过从一个唯一ID列表中获取，当使用chroot运行多个mtr实例时，每个mtr使用自己的唯一ID目录，将导致可能为多个mtr分配相同端口，导致端口冲突，使用该环境变量，使所有mtr共用一个唯一ID目录，避免端口冲突 |
| MYSQL_BIN_PATH | mysqld等文件所在路径 |
| MYSQL_CLIENT_BIN_PATH | mysql client程序所在路径 |
| MYSQL_CONFIG_EDITOR | mysql_config_editor程序所在目录 |
| MYSQL_TEST | mysqltest程序所在路径 |
| MYSQL_TEST_DIR | mysql-test所在路径的全路径名 |
| MYSQL_TEST_LOGIN_FILE | mysql_config_editor使用的login文件的路径名，若没有配置，则默认使用```$HOME/.mylogin.cnf```或者```%APPDATA%\MySQL\.mylogin.cnf```(windows) |
| MYSQL_TMP_DIR | 运行测试时的temp目录路径 |
| MYSQLD | mysqld程序文件的完整路径名 |
| MYSQLD_BOOTSTRAP | 具有所有选项可用的mysql的程序文件的完整路径名，该mysqld只用于环境初始化 |
| MYSQLD_BOOTSTRAP_CMD | 用于本次测试初始化数据库设置的完整命令行 |
| MYSQLD_CMD | 用于启动测试中使用的服务器的命令行，具有最少的必需参数集。 |
| MYSQLTEST_VARDIR | 测试中var目录的路径 |
| NUMBER_OF_CPUS | 定义cpu处理器的数量 |
| TSAN_OPTIONS | 包含 ThreadSanitizer 抑制的文件的路径名。 |

- MTR_PORT_BASE是MTR_BUILD_THREAD更逻辑直接的替代者，MTR_PORT_BASE配置端口的逻辑是对该配置值```向下取整为10的倍数```即目标端口号，MTR_BUILD_THREAD的逻辑是```配置值 * 10 + 10000```为目标端口号
- 测试有时依赖于定义的某些环境变量。例如某些测试假定MYSQL_TEST已定义，以便mysqltest可以通过```exec $MYSQL_TEST```调用自身(如果假定定义了环境变量，实际没有定义，是否执行报错```?```)
- 其他测试可能会引用其他一些环境变量，例如用来定位读和写的文件位置，如需要创建文件的test通常将文件创建为$MYSQL_TMP_DIR/file_name。
- $MYSQLD_CMD包含的是--mysqld选项带来的对所有测试的server都有效的配置项，但不包含为当前测试特定配置的server选项（这个环境变量类似一个状态变量而不是一个配置变量```?```）

### mysql-test-run.pl命令行参数

mysql-test-run.pl支持如下命令行参数，"``` - ```"参数用来告知mysql-test-run.pl不再将后续的参数当做配置选项来处理。

|  |  |
| :- | :- |
| ```--skip-test=prefix or regex``` | 跳过指定前缀或(perl)正则匹配的test_name的test cases |
| ```--skip-test-list=file``` |  |
| ```--do-test=prefix or regex``` | 只执行指定前缀或(perl)正则匹配的test_name的test cases,例如```--do-test=main.testa```匹配到的是main suite下的所有testa开头的test cases，而```--do-test=main.*testa```匹配到的是所有test_name包含main后0个或n个字符后接testa结尾格式的test cases，不要求main开头，例如xmainytesta，```.*```表示0个或n个任意字符 |
| ```--do-test-list=file``` | 用文件指定要测试的case，文件中每行一个test_name，注释使用#开头 |
| ```--do-suite=prefix or regex``` | 类似--do-test |
| ```--big-test``` | 用于允许标记了"big"的test cases执行。所谓标记了"big"即包含了```--source include/big_test.inc```的test case，这些test case只有在--big_test给出和BIG_TEST设置为1时才允许执行。 big test通常用于需要很长时间运行或使用大量资源的测试，因此它们不适合作为正常测试套件运行的一部分运行。 --big-test和--only-big-tests同时配置时，--only-big-tests将被忽略 |
| --boot-dbx | 通过dbx调试器运行用于引导（bootstrap）数据库的mysqld |
| --boot-ddd | 通过ddd调试器运行用于引导（bootstrap）数据库的mysqld |
| --boot-gdb | 通过gdb调试器运行用于引导（bootstrap）数据库的mysqld |
| --manual-boot-gdb | 和--boot-gdb类似，还允许使用远程的调试器 |
| --manual-dbx | 使用已在dbx调试器中启动的mysql server做test case测试 |
| --manual-ddd | 使用已在ddd调试器中启动的mysql server做test case测试 |
| --manual-debug | 使用已在调试器（不管啥调试器```?```）中启动的mysql server做test case测试 |
| ```--manual-gdb``` | ```使用已在gdb调试器中启动的mysql server做test case测试，可以用来调试testcase的失败原因``` |
| ```--build-thread=number``` | 指定一个数字来计算端口号。 公式为10 * build_thread + 10000，可以设置为```auto```，而不是数字，也是默认值，这样mysql-test-run.pl就会分配一个本机唯一的数字,该配置值（数字或auto）也可以使用 MTR_BUILD_THREAD 环境变量设置。```保留此选项是为了向后兼容。 推荐使用更合乎逻辑的 --port-base。``` |
| --callgrind | 指示命令格式使用callgrind。(```?```) |
| --charset-for-testdb=charset_name | 指定数据库的默认字符集，默认值为latin1 |
| --check-testcases | 检查test cases是否有副作用，通过每个test case测试前后检查系统状态，如果系统状态不一致，则将test case标记为失败 |
| --clean-vardir | Clean up the var directory with logs and test results etc. after the test run, but only if there were no test failures. This option only has effect if also running with option --mem. The intent is to alleviate the problem of using up memory for test results, in cases where many different test runs are being done on the same host.(```?未理解```) |
| --client-bindir=path | client(mysqltest?) 二进制文件所在路径 |
| --client-dbx | 在dbx调试器中启动mysqltest |
| --client-ddd | 在ddd调试器中启动mysqltest |
| --client-gdb | 在gdb调试器中启动mysqltest |
| --client-debugger=debugger_name | 在指定名称debugger_name的调试器中启动mysqltest |
| --client-libdir=path | client lib库所在目录(```?```) |
| ```--colored-diff``` | ```输出diff不同处着色，mysqltest会调用diff命令的--color='always'参数``` |
| --combination=value |  |
| --comment=str |  |
| --compress |  |
| --cursor-protocol |  |
| --ddd |  |
| --debug |  |
| --debugger=debugger |  |
| ```--debug-server``` | 运行 mysqld.debug（如果可用）而不是 mysqld 作为服务器。 如果确实找到了mysqld.debug，它会在它通常所在目录下的子目录debug 中搜索插件库。 此选项不会打开跟踪输出，并且与调试选项无关。 |
| --debug-sync-timeout=seconds |  |
| --default-myisam |  |
| --defaults-file=file_name |  |
| --enable-disabled |  |
| --explain-protocol |  |
| --extern option=value |  |
| --fast |  |
| ```--force``` | 当测试用例失败，若加了--force，将执行继续，否则mysql-test-run.pl会退出而测试结束 |
| ```--force-restart``` | 加了该参数，每次测试用例运行前总是重启mysql服务。对于一些测试用例很有用，例如测试内存泄漏，泄漏范围保证只在一个用例范围内 |
| --gcov |  |
| --gdb | 在 gdb 调试器中启动 mysqld(```?```) |
| --gprof |  |
| --include-ndbcluster, --include-ndb |  |
| --initialize=value |  |
| --json-explain-protocol |  |
| --mark-progress |  |
| --max-connections=num |  |
| --max-save-core=N |  |
| --max-save-datadir=N |  |
| ```--max-test-fail=N``` |  |
| --mem |  |
| --mysqld=value |  |
| --mysqld-env=variable=value |  |
| --mysqltest=options |  |
| --ndb-connectstring=str |  |
| --no-skip |  |
| --nocheck-testcases |  |
| --noreorder |  |
| --notimer |  |
| --nounit-tests |  |
| --nowarnings |  |
| --only-big-tests |  |
| ```--parallel={N/auto}``` |  |
| --port-base=P |  |
| --print-testcases |  |
| --ps-protocol |  |
| --quiet |  |
| --record |  |
| --reorder |  |
| --repeat=N |  |
| --report-features |  |
| --report-times |  |
| ```--report-unstable-tests``` |  |
| ```--retry=N``` |  |
| --retry-failure=N |  |
| --sanitize |  |
| --shutdown-timeout=seconds |  |
| --skip-combinations |  |
| --skip-ndbcluster, --skip-ndb |  |
| --skip-ndbcluster-slave, --skip-ndb-slave |  |
| --skip-rpl |  |
| --skip-* |  |
| --sp-protocol |  |
| --start |  |
| --start-and-exit |  |
| --start-dirty |  |
| --start-from=test_name |  |
| --strace-client |  |
| --strace-server |  |
| --stress=stress options |  |
| --suite(s)={suite_name/suite_list/suite_set} |  |
| ```--suite-timeout=minutes``` |  |
| --summary-report=file_name |  |
| --test-progress[={0/1}] |  |
| ```--testcase-timeout=minutes``` |  |
| --timediff |  |
| ```--timer``` |  |
| ```--timestamp``` |  |
| --tmpdir=path |  |
| --unit-tests |  |
| --unit-tests-report |  |
| --user=user_name |  |
| --user-args |  |
| --valgrind |  |
| --valgrind-clients |  |
| --valgrind-mysqld |  |
| --valgrind-mysqltest |  |
| --valgrind-option=str |  |
| --valgrind-path=path |  |
| --vardir=path |  |
| --verbose |  |
| --verbose-restart |  |
| --view-protocol |  |
| --vs-config=config_val |  |
| --wait-all |  |
| --warnings |  |
| --with-ndbcluster-only, --with-ndb-only |  |
| ```--xml-report=file_name``` |  |


### 测试举例

```bash
cd mysqldir
cd build_release
cd install

./mtr \
--report-unstable-tests \
--force \
--timestamp \
--timer \
--max-test-fail=0 \
--suite-timeout=9000 \
--testcase-timeout=300 \
--parallel=8 \
--retry=0 \
--xml-report=./mtr-report_r.xml \
--big-test
```
