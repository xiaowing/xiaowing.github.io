+++
title = "双剑合璧——当PG的FDW遇上GO(之二)"
author = "xiaowing"
date = 2018-05-13T22:02:33+08:00
draft = false
tags =  [
    "cgo",
    "golang",
    "postgresql",
    "fdw",
    ]
topics = [
    "技术",
]
+++

由于对Golang以及PostgreSQL(下文简称**PG**)的FDW(*Foreign Data Wrapper*)两个技术的双重喜爱,因此我利用假期用Golang实现了一个[访问douban API的FDW](https://github.com/xiaowing/douban_fdw). 同时也借此机会总结了一下 PG的FDW技术并分享一下使用Golang实现FDW的一些经验。


全文索引如下: 

* [第一部分: FDW的前世今生](/post/20180513_write_pgfdw_in_golang_part01/)

* **第二部分: 揭秘FDW**

* [第三部分: 如何用Go来实现一个FDW](/post/20180513_write_pgfdw_in_golang_part03/)

<!--more-->

## 2. 揭秘FDW

### 2.1 FDW的通用用法

使用FDW的核心就在于使用外部表(**FOREIGN TABLE**)。尽管面向不同数据源的FDW实现各有不同,但是收益于SQL/MED定义的标准, 创建不同数据源的外部表的方法都是一样的,分别需要在PG端依次创建以下几个数据库对象:

0. 向PG安装某个数据源的FDW扩展

1. 使用`CREATE FOREIGN DATA WRAPPER`语句创建该数据源的FDW对象。

2. 使用`CREATE SERVER`语句创建该数据源的服务器对象

3. 使用`CREATE USER MAPPING`语句创建外部数据源用户与PG用户的映射关系(这一步是可选的. 比如外部数据源根本没有权限控制时, 也就无需创建USER MAPPING了)

4. 使用`CREATE FOREIGN TABLE`语句创建外部表

之后就可以使用`SELECT`语句按照访问普通表的方式访问外部表; 如果该数据源支持写操作且它的FDW也已实现支持写操作的相关接口,则也可以使用`INSERT`, `UPDATE`或`DELETE`语句去更新外部表

上述过程可以简要地用下图来概述:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/foreign_table_related_objects.png|Foreign Table关联的数据库对象"
>}}

### 2.2 FDW关联的数据库对象

上一小节介绍了FDW的通用用法,这里将简单说明一下该用法中提及的几个数据库对象的作用:

* FOREIGN DATA WRAPPER对象([对应的DDL语法](https://www.postgresql.org/docs/10/static/sql-createforeigndatawrapper.html))

    是一个纯粹的抽象概念,创建该对象的实质是向PG注册了某个数据源的FDW所实现的两个自定义函数——该FDW所实现的所有接口的注册函数(在`CREATE FOREIGN DATA WRAPPER`语句中称为**HANDLER**)以及该FDW的所支持的选项验证函数(在`CREATE FOREIGN DATA WRAPPER`语句中称为**VALIDATOR**)。

    该对象被创建后, 语句中制定的HANDLER与VALIDATOR会被添加至系统表[pg_proc](https://www.postgresql.org/docs/10/static/catalog-pg-proc.html)(保存所有自定义函数的元数据)中,且两者的OID以及该FOREIGN DATA WRAPPER对象的名称与OID一同被保存至系统表[pg_foreign_data_wrapper](https://www.postgresql.org/docs/10/static/catalog-pg-foreign-data-wrapper.html)中。

    通常FOREIGN DATA WRAPPER对象的创建过程是直接包含在了安装FDW扩展的`CREATE EXTENSION`语句中,从而在安装时被自动执行,无需数据库用户在使用中单独执行。

    需要补充说明的是,HANDLER的作用是将该FDW实现的一系列fdw回调函数的地址打包返回给PG,从而使PG之后访问外部表时可以调用这些访问外部数据的函数。而所谓的fdw回调函数则是指[PG手册所提及的下述接口](https://www.postgresql.org/docs/10/static/fdw-callbacks.html)的实现

    * `GetForeignRelSize`
    * `GetForeignPaths`
    * `GetForeignPlan`
    * `BeginForeignScan`
    * `IterateForeignScan`
    * `EndForeignScan`
    * 等等......

    关于这些回调函数的作用,会在后文介绍,此处暂略。

* FOREIGN SERVER对象([对应的DDL语法](https://www.postgresql.org/docs/10/static/sql-createserver.html))

    表示的是外部数据源的数据库对象(比如可以在CREATE SERVER时通过选项指定数据库所在服务器的IP地址等信息)

    FOREIGN SERVER对象被创建后, 相关的元数据被保存在系统表[pg_foreign_server](https://www.postgresql.org/docs/10/static/catalog-pg-foreign-server.html)中

* FOREIGN TABLE对象([对应的DDL语法](https://www.postgresql.org/docs/10/static/sql-createforeigntable.html))

    将外部数据源的数据组织为表的形式,这样的表就被称作为外部表,它可能对应的时外部异构RDBMS的一张表,也有可能对应了文件系统上的某一个文件或是一个微服务的API。具体如何对应,取决于这个数据源的FDW的实现。

    当外部表对象被创建后,它与PG中的普通表一样,元数据都会被保存在系统表[pg_class](https://www.postgresql.org/docs/10/static/catalog-pg-class.html)中,只是它的**relkind**字段会以"**f**"进行标识;同时,该表在也会在系统表[pg_foreign_table](https://www.postgresql.org/docs/10/static/catalog-pg-foreign-table.html)被保存一条记录,它存储了该表在pg_class的OID与该表所属的FOREIGN SERVER的OID的对应关系。

    从9.5开始,PG提供了一个新的语法`IMPORT FOREIGN SCHEMA`支持用户批量导入外部数据源的外部表,以省却一个一个`CREATE FOREIGN TABLE`的繁琐. 当然,前提时该数据源的FDW实现中需要实现`IMPORT FOREIGN SCHEMA`所对应的回掉函数。

有了上述外部对象相关的元数据的支撑,当一个查询试图访问外部表得以获取外部数据源的数据时, FDW在整个查询的执行过程中就可以发挥下述作用:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/fdw_shikumi.png|FDW在外部表查询中发挥的作用"
>}}

在此图中, FDW得以介入整个执行过程的奥秘就在于**回调函数** 

### 2.3 FDW回调函数与外部表查询

如上文所说,一个FDW的实现的核心就是实现一组回调函数,从而在查询外部表对象的SQL的执行过程中可以将运行逻辑切换至自定义的扩展代码中,进而遵照PG的内部机制实现对外部数据源的访问。

截止到PG 10, PG提供的FDW回调接口已有20余个。FDW的实现者需要根据外部数据源自身的能力(比如是否支持写操作,以及是否支持在外部数据源端执行JOIN操作等等)对这些接口有选择性地予以实现。

这些接口中,最核心的接口有7个。无论外部数据源自身能力如何,这7个接口是实现通过外部表对象访问该数据源的必须接口。这7个回调函数的接口定义如下:

````c
typedef void (*GetForeignRelSize_function) (PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);

typedef void (*GetForeignPaths_function) (PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);

typedef ForeignScan *(*GetForeignPlan_function) (PlannerInfo *root, RelOptInfo *baserel,Oid foreigntableid, ForeignPath *best_path, List *tlist, List *scan_clauses, Plan *outer_plan);

typedef void (*BeginForeignScan_function) (ForeignScanState *node, int eflags);

typedef TupleTableSlot *(*IterateForeignScan_function) (ForeignScanState *node);

typedef void (*ReScanForeignScan_function) (ForeignScanState *node);

typedef void (*EndForeignScan_function) (ForeignScanState *node);
````

我们知道,一条查询语句在PG中会经历三个大的阶段:

* Parser: 包含对SQL的语法解析, 语义校验, 查询重写
* Optimizer: 生成查询计划
* Executor: 按照经典的[火山模型](http://dbms-arch.wikia.com/wiki/Volcano_Model)执行查询计划的算子并向上"吐"数据

上述这七个回调函数主要在**Optimizer**和**Executor**阶段进行"介入"。如下所示:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/fdw_callback_sequence.png|FDW回调的时序"
>}}  

需要注意的是,上图仅仅是显示这些回调函数被调用的时序顺序. 图中的箭头并不表示两个回调之间存在调用关系。事实上这些回调函数都是由PG的**Optimizer**和**Executor**进行调用.

这七个回调函数详细的调用时机以及作用总结如下:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/fdw_callbacks_details.png|主要的FDW回调函数的详细解说"
>}} 

### 2.4 FDW回调函数间的数据传递

如上文所述,回调函数本身是由PG来调用的,各回调函数之间并不会产生彼此的互相调用。因此就产生了一个衍生的问题——如果在FDW的实现想要在这些回调函数之间传递数据怎么办? PG在设计回调函数的接口时也充分考虑到了这一点：

* Optimizer阶段

  在Optimizer阶段执行的三个回调函数都会传入一个`RelOptInfo`结构体作为输入参数,该结构体专门有一个字段可供FDW使用:

  ````c
  typedef struct RelOptInfo {
      ...(上略)...
      void	   *fdw_private;
      ...(下略)...
  } RelOptInfo;
  ````

  因此如果FDW的实现需要在Optimizer阶段的回掉函数间传递数据时,只需要自行申请内存存储临时数据后将指针挂在上述字段即可.

* Executor阶段

  在Optimizer阶段执行的回调函数都会传入`ForeignScanState`结构体. 在PG中, 各种Scan算子都会有一个对应的State结构体,用于存放算子相关的状态数据。考虑到不同数据源的FDW实现可能会需要带一些自定义数据,因此`ForeignScanState`结构体也专门有一个字段可供FDW使用:

  ````c
  typedef struct ForeignScanState
  {
      ScanState	ss;				
      List	   *fdw_recheck_quals;	
    
      struct FdwRoutine *fdwroutine;
      void	   *fdw_state;    /* 存放各FDW实现的私有状态数据 */	
  } ForeignScanState;
  ````

  与`RelOptInfo`的`fdw_private`字段类似,FDW的实现只需申请内存存储临时数据后将指针挂在`fdw_state`字段即可.

* 从Optimizer到Executor

  如果FDW的实现中想把一些数据从Optimizer阶段带到Executor阶段,会比上文的两个途径略显复杂。

  在PG 9.5中,这样的数据传递是通过下述方法实现的:

  * 存入私有数据
  
    在回调函数`GetForeignPlan()`的实现中必须为后续的Executor创建一个`ForeignScan`算子(这也是`GetForeignPlan()`最主要的作用)。由于`ForeignScan`中有一个`fdw_private`字段,因此FDW的私有数据可以通过该字段进行存放。

    不过,与`RelOptInfo`结构体不同的是,`ForeignScan`算子的`fdw_private`字段是一个`List *`类型。这是考虑到了FDW的实现中可能想传多个散列的数据结构给后续的Executor中的回调函数, 那么如果仅仅是一个`void *`势必就要重新做内存分配并且还得做拷贝操作,但时机上FDW的实现只是希望找地方挂一下私有数据即可,因此用`List *`的话,对于传递多个结构体的情况将更为方便。

    只是,`List *`里按什么样的顺序去挂怎样的数据,这就是PG的接口无法定义的了。这必须依赖于FDW实现自己去严格按照挂数据的顺序以及类型去取数据,这样的接口设计其实是耦合度较高的。不过除了这个方法似乎也没有更好的方法,因此直到现在的PG 10.3版本中,`ForeignScan`算子的`fdw_private`字段仍然保持着这样的设计。

  * 获取私有数据

    在Executor执行`BeginForeignScan()`回调函数时,利用传入的`ForeignScanState`结构体的`ss`字段(ScanState类型)访问其中的`ps`字段(PlanState类型)的`plan`字段(ForeignScan类型), 进而获取到ForeignScan类型中挂着的存放私有数据的`List`, 最后解开这个`List`即可获取在Optimizer阶段传递的私有数据.

以上进介绍了执行对外部表查询时所涉及到的回调函数之间的数据传递方式,其他的回调函数之间的数据传递方式暂且略过。

### 2.5 小结

就这样, 对于一个外部数据源而言, 只需要实现了上述的7个回调函数,就可以支持PG对于该外部数据源以通用的SQL语句实现简单的查询功能。当然,如果要真的尝试去实现这些回调函数,还是需要通过PG的一些专门面向FDW提供的接口来与PG进行交互,比如在上一节中提及的`make_foreignscan()`和`create_foreignscan_path()`就属于这些接口; 手册上的[Foreign Data Wrapper Helper Functions章节](https://www.postgresql.org/docs/10/static/fdw-helpers.html)也有一些关于实现FDW时可能会用到辅助接口的说明。

此外,上文所说这些回调函数如果要让PG能够知晓且调用到,是需要有一个注册机制. 这个注册机制就是通过上文提及的`CREATE FOREIGN DATA WRAPPER`语句中用`HANDLER`关键字指定的UDF函数来实现。这个UDF函数的逻辑很简单, 参考一些现成的FDW实现即可,此处不再赘述。

(*to be continued...*)

