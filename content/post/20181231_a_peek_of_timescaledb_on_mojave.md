+++
title = "在MacOS Mojave上管中窥豹TimescaleDB"
author = "xiaowing"
date = 2018-12-31T14:19:05+08:00
draft = false
tags =  [
    "database",
    "time-series",
	"postgresql",
    ]
topics = [
    "技术",
]
+++

{{< fluid_imgs
  "pure-u-1-3|/img/post/timescale_logo.png|Logo of TimescaleDB"
>}}

年底换了工作, 因此未来一段时间的工作方向会集中在[时序数据库](https://en.wikipedia.org/wiki/Time_series_database).然后恰逢今年9月,号称第一款基于PostgreSQL的时序数据库 [TimescaleDB](https://en.wikipedia.org/wiki/Time_series_database)正式推出了可用于生产环境的1.0版本。于是就赶快先试用了一把...

<!--more-->

## 编译安装

由于换了新东家,所以开发机换成了Macbook。因此顺带记录一下TimescaleDB在**Mac OS 10**的编译安装过程。

仔细研究了一下TimescaleDB，发现它本质上是PostgreSQL的一个**Extension**. 因此下文将从PG在Mac OS 10上的编译安装和TimescaleDB插件的编译安装两部分来叙述。

### PostgreSQL在Mac OS 10上的编译安装

考虑到PG11已经发布，所以选择编译安装PG11.1的代码，而且PG11主打的是LLVM，所以编译安装时自然要选装LLVM的支持, 在Mac OS上需要以下的准备过程：

1. 安装XCode command line tools

    这是编译安装的基本，它会安装上类似gcc, make等一系列GNU工具

    ````sh
    $xcode-select --install
    ````

2. 安装llvm工具库

    PG的LLVM支持需要使用`llvm-config`工具，这是Mac下集成了llvm功能的Clang环境无法替代的。因此需要单独下载并安装. 下载并安装直接使用`$brew install llvm`即可，之后llvm会被安装到/usr/local/opt路径。

    安装后需要配置环境变量`PATH`及以下两个编译器相关的环境变量:

    ````sh
    $export LDFLAGS="-L${llvm安装路径}/lib"
    $export CPPFLAGS="-I${llvm安装路径}/include"
    $export PATH=${llvm安装路径}/bin:${PATH}
    ````

3. PG安装三板斧

    接下来就是用三板斧(configure-make-make install)编译安装PG了, 流程上并没有什么特殊，但是由于Mac环境的特殊性，在`configure`阶段有以下注意事项:

    1. 由于Mac OS中已提供了BSD许可证的`libedit`, 鉴于Mac系产品的商用性质，所以应当使用`libedit`代替GPL协议的`libreadline`

    2. 若要编译后的PG支持LLVM，需要在`configure`指定`--with-llvm`并跟上**llvm-config**的路径

    3. 需要在`configure`时通过指定`PG_SYSROOT`显式指定**command line tools sdk**，否则后面的代码编译时会连 `stdio.h`这样的标准库头文件都找不到。而SDK的路径通常在`/Library/Developer/CommandLineTools/SDKs/MacOS.sdk/`

    考虑到以上注意事项. 本例中的三板斧命令展开即为

    ````sh
    $cd postgresql-11.1/
    $./configure --prfix=${PG的目标安装目录} --with-libedit-preferred --with-llvm LLVM_CONFIG='/usr/local/opt/llvm/bin/llvm-config' PG_SYSROOT=/Library/Developer/CommandLineTools/SDKs/MacOS.sdk/
    $make
    $make install
    ````

### Timescaledb插件的编译安装

由于Timescaledb是PG的插件，与大部分PG插件相同，其编译安装需要依赖`pg_config`命令。因此在编译安装前首先确保`pg_config`可以直接使用(如已分别将安装后的PG的bin目录和lib目录导出到了`PATH`变量和`DYLD_LIBRARY_PATH`变量)

1. 执行Timescaledb的bootstrap. 

    在执行bootstrap时，脚本会调用`pg_config`确认PG环境有无对OpenSSL的支持。在本例中, 我在编译PG时未指定`--with-openssl`，因此执行bootstrap时需要额外指定`-DUSE_OPENSSL=0`

    ````sh
    $cd timescaledb-1.1.0
    $./bootstrap
    ````

2. 执行编译和安装

    这一步没有什么特殊的，在步骤1.的基础上执行下述命令即可

    ````sh
    $cd build
    $make
    $make install
    ````

如此，TimescaleDB便已安装完成。与大部分PG插件相同，它的库文件(二进制)会被安装到`${PG的目标安装目录}/lib`目录下，它的SQL脚本会被安装至`${PG的目标安装目录}/share`目录下.

但是安装完毕仍然是不够的，若要正常使用TimescaleDB，还需要以下设置

3. PG实例的设置

    通过PG的[initdb](https://www.postgresql.org/docs/11/app-initdb.html)命令创建实例后，需要在实例目录的配置文件`postgresql.conf`中将timescaledb设置为预加载库，这样timescaledb的库文件才会随实例启动而加载

    ````sh
    #postgresql.conf
    shared_preload_libraries = 'timescaledb', ...
    ````
4. 在目标database中启用timescaledb

    在上述步骤3.完成后，重启PG实例。之后使用psql连接至PG实例上的某个Database，通过psql执行下述语句

    ````sql
    CREATE EXTENSION timescaledb;
    ````

    这便会[在且仅在](https://www.postgresql.org/docs/11/sql-createextension.html)当前Database范围内启用timescaledb.

    到此为止，整个安装过程才算结束


## 试用

根据Timescaledb的手册可知, 使用Timescaledb创建一张时序数据表本身就是创建一张普通表, 加上将该普通表创建Hypertable的步骤. 因此可以按下述方式试用:

1. 创建普通表

    ````sql
    CREATE TABLE IF NOT EXISTS foobar (
        time TIMESTAMP NOT NULL,
        metric VARCHAR(32) DEFAULT 'cpu',
        usage  DOUBLE PRECISION  DEFAULT RANDOM(),
        temperature DOUBLE PRECISION DEFAULT (RANDOM()*(40-10)+10)
    );
    ````

    由于这张表后面需要升级为时序表，因此遵照Timescaledb的建议在表中包含了一个`TIMESTAMP`类型的字段(尽管这不是必须的)

2. 利用Timescaledb的接口`create_hypertable()`(实际上就是一个SQL函数)将上述普通表升级为受Timescaledb管理的时序表

    ````sql
    SELECT create_hypertable('foobar', 'time'); 
    ````

    该接口的第二参数需要接收表中的一个用于存放时间戳数据的列名以作为该时序表默认分区时的分区键, 这也是为什么Timescaledb推荐时序表中包含一个`TIMESTAMP`类型字段的原因

这样一来, 一张时序表就创建成功了。接下来将从以下两个角度来对时序表进行一些简单评测.
* 时序库的常用操作
* 时序表的内部构成

### 常用操作

* 写入 & 数据点查询

    时序库最大的试用场景便是时序数据的写入。在Timescaledb中可以很简单地利用`INSERT`语句进行写入. 以上文创建的时序表`foobar`为例:

    ````sql
    timedb=# INSERT INTO foobar(time) SELECT * FROM generate_series(now(), now() + '14 days', '1 second');
    INSERT 0 1209601
    timedb=#
    timedb=# SELECT * FROM foobar LIMIT 10;
                time            | metric |       usage       |   temperature
    ----------------------------+--------+-------------------+------------------
     2019-01-01 15:00:58.629342 | cpu    | 0.536569389514625 | 15.0133295543492
     2019-01-01 15:00:59.629342 | cpu    | 0.161968716420233 | 27.6340685179457
     2019-01-01 15:01:00.629342 | cpu    | 0.878905369434506 | 31.2941490672529
     2019-01-01 15:01:01.629342 | cpu    |  0.80755941895768 | 14.5778051810339
     2019-01-01 15:01:02.629342 | cpu    | 0.796080893371254 | 32.3037584684789
     2019-01-01 15:01:03.629342 | cpu    | 0.200302240438759 | 32.3949345899746
     2019-01-01 15:01:04.629342 | cpu    |  0.20913993800059 | 31.4002282964066
     2019-01-01 15:01:05.629342 | cpu    |  0.91796684730798 |  19.221549173817
     2019-01-01 15:01:06.629342 | cpu    | 0.849315787199885 | 11.5261871553957
     2019-01-01 15:01:07.629342 | cpu    | 0.468408294487745 | 38.0471089156345
    (10 rows)
    ````

    当然, 如果是通过JDBC等数据库驱动程序的话，更可以利用[批量插入接口](https://viralpatel.net/blogs/batch-insert-in-java-jdbc/)来获得更高的写入效率

* 降精度聚合查询

    利用Timescaledb的特有接口`time_bucket()`, 再加上标准SQL的`GROUP BY`语句就可以很容易的实现降精度聚合查询

    ````sql
    timedb=# SELECT time_bucket('5 minutes', time) AS five_min, avg(usage), max(temperature) FROM foobar GROUP BY five_min ORDER BY five_min DESC LIMIT 10;
          five_min       |        avg        |       max
    ---------------------+-------------------+------------------
     2019-01-15 15:00:00 | 0.511477737916412 |   39.75819949992
     2019-01-15 14:55:00 | 0.471913051234248 | 39.7251598956063
     2019-01-15 14:50:00 | 0.489521298462835 | 39.9647896923125
     2019-01-15 14:45:00 | 0.476991963624023 | 39.9298739805818
     2019-01-15 14:40:00 | 0.509973571114242 | 39.8932682536542
     2019-01-15 14:35:00 | 0.483743709454623 | 39.9439396383241
     2019-01-15 14:30:00 | 0.515407514680798 | 39.9820741685107
     2019-01-15 14:25:00 | 0.498390285711115 |  39.929482084699
     2019-01-15 14:20:00 | 0.504943751936468 | 39.9079268751666
     2019-01-15 14:15:00 | 0.526519591989927 |  39.821355114691
    (10 rows)
    ````

* 数据保存策略(Retention Policies)

    **数据保存策略**是部分时序数据库(如Influxdb)提供的特性. 其着眼点在于时序数据的使用场景通常都是越新的数据越会被频繁使用, 超过了一段时间的时序数据则通常只需要归档保存. 
    
    因此Retention Policies用于在数据写入时指定保存期限，超过时间后数据被自动删除或降精度归档。

    在Influxdb中，尚不支持完整的Retention Policies功能, 目前(截至v1.1)只是提供了一个`drop_chunks()`接口用于执行高性能的数据删除。Timescaledb建议用户配合crontab任务使用`drop_chunks()`来实现类似Retention Policies的功能

综上, 时序库中常用的功能, Timescaledb算是在PG的框架下差强人意地予以了实现 

### 内部构成

接下来研究一下Timescaledb中的时序表内部构成是什么样子的。仍然以上文创建的`foobar`表为例

首先看一下`create_hypertable()`对普通表偷偷地做了什么改动:

````sql
timedb=# \d+ foobar
...(表结构略)...
Indexes:
    "foobar_time_idx" btree ("time" DESC)
Triggers:
    ts_insert_blocker BEFORE INSERT ON foobar FOR EACH ROW EXECUTE PROCEDURE _timescaledb_internal.insert_blocker()
Child tables: _timescaledb_internal._hyper_4_10_chunk,
              _timescaledb_internal._hyper_4_11_chunk,
              _timescaledb_internal._hyper_4_12_chunk
````

可以发现`create_hypertable()`对指定的普通表做了以下更改:

1. 为传入`create_hypertable()`第二参数的字段创建了一个默认的Btree索引
2. 为表`foobar`创建了一个BEFORE INSERT的触发器. 阅读代码后发现该触发器实际上是屏蔽了对原始`foobar`的写操作。
3. 为`foobar`创建了一系列子表(这一点只有当有数据写入时才能够呈现出来)

更深入一点可以发现, 其实通过`create_hypertable()`升级的时序表的信息都写入了timescaledb的内部表`_timescaledb_catalog.hypertable`中，写入的信息如下所示:

````sql
timedb=# SELECT * FROM _timescaledb_catalog.hypertable WHERE table_name='foobar';

 id | schema_name | table_name | associated_schema_name | associated_table_prefix | num_dimensions | chunk_sizing_func_schema |  chunk_sizing_func_name  | chunk_target_size
----+-------------+------------+------------------------+-------------------------+----------------+--------------------------+--------------------------+-------------------
  4 | public      | foobar     | _timescaledb_internal  | _hyper_4                |              1 | _timescaledb_internal    | calculate_chunk_interval |                 0
(1 row)
````

记录了原始表的以下扩展信息:

* 原始的Schema名 & Table名
* 原始表在Timescaledb中所关联子表的前缀名
* 子表的自动分区的Dimensions数
* 自动分区的计算函数(即决定何时开始一个新分区(子表))

这里的原始表和子表的关系实际上就是TimescaleDB实现的一个最核心的特性——时序表的**自动分区**.对于这个核心特性, 我个人的理解如下:

试想一下如果是在PG中的普通表, 为了快速进行数据点查询，那么必须要对时间戳字段上B-tree索引。根据《Database System Implementation》的说法, 理论上B-tree索引的最佳性能通常是确保树高在**3**左右,但树高为3的B-Tree所能索引的数据行数不过是**千万~亿**这个级别。而时序数据库的使用场景下, 一组设备/监控项在一天之内的数据写入恐怕就要达到**千万~亿**级数据。因此随着后续数据不断写入, 树高会增加，这会带来写入时叶节点的不断分裂，从而降低写性能; 同样，树高的增长也会对读性能带来负面影响。这便是普通表不适用于时序数据库的一个最大短板。

Timescaledb瞄准的实际上就是上述短板, 通过自动分区的方式，确保每一个子表(分区)的索引规模都维持在一个较小的范围从而提升写和读的性能。

## 小结

如果说硬要从功能角度给Timescale的优缺点做一个第一印象的小结, 我的第一印象的评价如下:

* 优点
    * 完整的SQL支持, 特别是支持与普通表的JOIN查询
    * 背靠PG好乘凉(稳定性值得信赖)

* 缺点
    * 时序功能使用略不直观, 需要预先设计Scheme
    * 尚不支持自动的Retention Policies, 增加运维成本

以上便是对Timescaledb进行试用后整理的一点内容, 当然Timescaledb自身索要包含的内容还远不仅此。

不过由于TimescaleDB是基于PG做的，所以不可避免的，它的时序数据是有Scheme的. 这样的数据模型和当前大部分时序数据库的数据模型(**metric-tagset**模型)都截然不同. 由于时序数据库发展时间尚不长，目前也没有一个标准，所以这两种模型孰优孰劣, 现在还暂时给不了一个定论, 包括从网上的性能评测报告来看, 即便是时序数据的场景下, 也暂时无法断定哪一种模型更好。后面继续拭目以待吧.

