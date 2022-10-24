# tppc

TPC-C 于 1992 年 7 月 23 日认可为新的基准测试。TPC(Transaction Processing Performance Council，事务处理性能委员会)-C是衡量联机事务处理（OLTP，Online TransactionProcessing）系统的工业标准，是行业中公认的权威和最为复杂的在线事务处理基准测试。它通过模拟仓库和订单管理系统，测试广泛的数据库功能，包括查询、更新和 mini-batch事务（队列式小批量事务）。TPC-C基准测试针对一种模拟订单录入与销售环境测量每分钟事务（tpmC-transactions per minute）吞吐量。

https://www.tpc.org/default5.asp

# benchmarksql

BenchmarkSQL是一款经典的开源数据库测试工具，内嵌了TPCC测试脚本，可以对PostgreSQL、Oracle等数据库进行测试.。



官网地址：https://sourceforge.net/projects/benchmarksql/

文档地址：https://benchmarksql.readthedocs.io/en/latest/

## 创建用户及数据库

```shell
$ psql postgres
psql (9.5.2)
Type "help" for help.

postgres=# CREATE USER benchmarksql WITH ENCRYPTED PASSWORD 'changeme';
postgres=# CREATE DATABASE benchmarksql OWNER benchmarksql;
postgres=# \q
```



## 编译

```shell
unzip benchmarksql-5.0.zip
ant
```

## 配置参数

进入run目录，配置如下参数

vi props.pg

```shell
db=postgres
driver=org.postgresql.Driver
conn=jdbc:postgresql://localhost:5432/benchmarksql
user=postgres
password=PWbmsql
warehouses=1
loadWorkers=4
terminals=1
//To run specified transactions per terminal- runMins must equal zero
runTxnsPerTerminal=10
//To run for specified minutes- runTxnsPerTerminal must equal zero
runMins=0
//Number of total transactions per minute
limitTxnsPerMin=300
//Set to true to run in 4.x compatible mode. Set to false to use the
//entire configured database evenly.
terminalWarehouseFixed=true
//The following five values must add up to 100
//The default percentages of 45, 43, 4, 4 & 4 match the TPC-C spec
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4
// Directory name to create for collecting detailed result data.
// Comment this out to suppress.
resultDirectory=my_result_%tY-%tm-%td_%tH%tM%tS
osCollectorScript=./misc/os_collector_linux.py
osCollectorInterval=1
//osCollectorSSHAddr=user@dbhost
osCollectorDevices=net_eth0 blk_sda
```

主要参数解释如下

| 参数                   | 值                                        | 说明                                                         |
| ---------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| db                     | postgres                                  | 指定数据库类型（修改后增加的数据库类型）                     |
| driver                 | org.postgresql.Driver                     | 数据库的驱动类名称，不需要更改                               |
| conn                   | jdbc:postgresql://localhost:5432/postgres | 连接字符串，需要更改localhost为对应pg服务IP、5432位对应pg服务端口、postgres为对应测试数据库名 |
| user                   | postgres                                  | 用户名                                                       |
| password               | 123456                                    | 用户密码                                                     |
| warehouses             | 10                                        | 初始化加载数据时，需要创建多少仓库的数据。每仓库约80MB数据,数量根据实际服务器内存配置。 |
| loadWorkers            | 4                                         | 表示加载数据时，开启加载数据的进程数，加载速度会随worker数量的增加而有所提升。 |
| terminals              | 20                                        | 终端数量，指同时有多少终端并发执行，通常设置为CPU线程总数的2～6倍 |
| runTxnsPerTerminal     | 0                                         | 每个终端（terminal）运行的固定事务数量，例如：如果该值设置为10，意味着每个terminal运行10个事务，如果有32个终端，那整体运行320个事务后，测试结束。该参数配置为非0值时，下面的runMins参数必须设置为0 |
| runMins                | 1                                         | 要测试的整体时间，单位为分钟，如果runMins设置为60，那么测试持续1小时后结束。该值设置为非0值时，runTxnsPerTerminal参数必须设置为0。这两个参数不能同时设置为正整数，如果设置其中一个，另一个必须为0，主要区别是runMins定义时间长度来控制测试时间；runTxnsPerTerminal定义事务总数来控制时间。 |
| limitTnxsPermin        | 0                                         | 限制每分钟执行的事务总数。0表示不限制                        |
| terminalWarehouseFixed | true                                      | 用于指定终端和仓库的绑定模式，设置为true时可以运行4.x兼容模式，意思为每个终端都有一个固定的仓库。设置为false时可以均匀的使用数据库整体配置，TPCC规定每个终端都必须有一个绑定的仓库，所以一般使用默认值true。 |

 

## 数据准备

创建数据库表并加载数据

```
./runDatabaseBuild.sh props.pg
```

## 执行测试

执行如下命令开始测试

```
./runBenchmark.sh props.pg
```

测试完后会显示简要的测试结果，包括平均tpmC值（每分钟执行的事务数--每分钟能够处理的订单数量），tpmTOTAL（总订单量）等等。

流量指标(Throughput,简称tpmC)：按照TPC组织的定义，流量指标描述了系统在执行支付操作、订单状态查询、发货和库存状态查询这4种交易的同时，每分钟可以处理多少个新订单交易。所有交易的响应时间必须满 足TPC-C测试规范的要求，且各种交易数量所占的比例也应该满足TPC-C测试规范的要求。在这种情况下，流量指标值越大说明系统的联机事务处理能力越高。

## 生成报告

生成图形的脚本generateReport.sh要求操作系统环境中已经安装了R语言

```
./generateReport.sh my_result_2021-05-23_175724
```

## 重新运行测试

执行runDatabaseDestroy.sh脚本带配置文件可以将所有的数据和表都删除，然后再重新修改配置后运行测试。