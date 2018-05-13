+++
title = "双剑合璧——当PG的FDW遇上GO(之一)"
author = "xiaowing"
date = 2018-05-13T21:02:33+08:00
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

由于对Golang以及PostgreSQL(下文简称**PG**)的FDW(*Foreign Data Wrapper*)两个技术的双重喜爱, 因此我利用假期用Golang实现了一个[访问douban API的FDW](https://github.com/xiaowing/douban_fdw). 同时也借此机会总结了一下 PG的FDW技术并分享一下使用Golang实现FDW的一些经验。

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/fdw_metaphor.jpg|FDW Metaphor"
>}}

全文索引如下: 

* **第一部分: FDW的前世今生**

* [第二部分: 揭秘FDW](/post/20180513_write_pgfdw_in_golang_part02/)

* [第三部分: 如何用Go来实现一个FDW](/post/20180513_write_pgfdw_in_golang_part03/)

<!--more-->

## 1. FDW的前世今生

### 1.1 起源——SQL/MED

通常,随着一个企业/组织的IT体系的日益增大,往往会不可避免的在多个应用之间需要进行数据共享与交换。

通常我们希望这些应用的DAC(Data Access Code)能够简单直观一些。然而不幸的是,即使是在一个企业/组织的内部,不同应用所涉及到的数据往往会存储在不同的数据源中——可能是不同厂商的DBMS产品,甚至是根本不支持SQL的异构数据存储中。因此,往往多个应用的数据访问关系就会变得像下图一样复杂:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/dac_disaster.png|DAC Disaster"
>}}

随着社会的信息化推进,上述问题逐渐称为业界的共性课题。数据库行业在2001年对于该课题给出了一个积极的响应,这就是SQL/MED扩展标准(该标准的第一个版本为 ISO/IEC 9075-9:2001, 目前最新版本为 ISO/IEC 9075-9:2016)

SQL/MED标准旨在建立一个解决此类课题的技术规范: 即应用程序可以通过统一且标准的方式(即,SQL)去访问存储在不同数据源中的数据, 且数据源本身对应用透明。在实际应用中, SQL/MED标准希望达成的效果如下图所示: 

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/med_vision.png|SQL/MED vision"
>}}

需要注意的是SQL/MED标准对于解决上述问题实际上定义了两套技术规范,一个是**Foreign Data Wrapper**, 另一个则是**datalink 类型**(它与PG中的[dblink](https://www.postgresql.org/docs/10/static/dblink.html)以及Oracle中的[Database Link](https://docs.oracle.com/cd/B28359_01/server.111/b28310/ds_concepts002.htm)不是一回事,尽管其目的有相似之处).

本文的剩余部分将主要围绕FDW来展开说明.

### 1.2 PG中FDW的现状

PG对于FDW的支持早在10年前就已经开始了.

社区从2009年推出8.4版本中首先就已经提供了对FDW的创建语句以及Foreign Server的创建语句的语法支持(但未实现实际的功能).到了2011年社区推出9.1版本时正式对外公开了支持FDW功能的内部接口,从而让扩展的编写者可以利用这些接口为不同的数据源编写FDW的实现。并且,社区在接下来的7年内, 每一次PG的版本升级都会带来FDW的功能提升——尽管这些功能增强不一定总是体现在语法层面。比如,在最初的版本中,FDW仅仅只能将远端数据源的数据原封不动地拉至PG中; 但到了最近的两三个版本中,借助FDW已经可以实现将更多地运算(如JOIN, 聚合等)下推至远端数据源,并能够对远端数据源的数据进行更新 (注1)。

截止到目前为止, 全世界已有了成百上千种数据源有了相应的FDW实现,从传统的文件系统到各种新型的Nosql数据库,甚至还包括互联网上的Web Service。在[PG官方社区的wiki页](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)中罗列了一部分较常见的数据源的FDW

另外值得一说的是,尽管基于SQL/MED标准的FDW技术的初衷是为了统一异构数据源的访问方式。但是,随着这些年PG的FDW内置功能(core functionality)的逐渐增强,支持将越来越多的运算下推到远端执行。而且,随着FDW功能成长的,还有一个获得社区官方支持的用于访问远端PG服务器的FDW扩展[postgres_fdw](https://www.postgresql.org/docs/10/static/postgres-fdw.html)所支持的功能也越来越强大。有鉴于此,PG社区核心团队的大佬 **Bruce Momjian** [在2016年给社区写了一封邮件](https://www.postgresql.org/message-id/20160223164335.GA11285%40momjian.us)提议了一个基于FDW技术的分布式分片(术语: Sharding)方案,与此同时,老爷子还[写了一份PPT](http://momjian.us/main/writings/pgsql/sharding.pdf)专门阐述这个想法。基于"对现有PG代码改动最小"这一原则,目前社区已经基本上认可了基于FDW的Sharding方案作为[PG源生的分布式实现方案](https://wiki.postgresql.org/wiki/Built-in_Sharding)

事实上,Bruce老爷提出的这个方案在实现和落地方面,日本技术者显然走得很远(事实上,实现FDW核心功能的patch最早就是[由日本的花田茂提出的](https://www.postgresql.org/message-id/20101125163436.96F6.6989961C%40metrosystems.co.jp))。2017年, NTT DATA分享了他们在基于PG 9.6做的Sharding方案——简而言之就是PG的 **表继承 + postgres_fdw**。其概念和架构如下所示：

* 概念图

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/pg_fdw_sharding.png|基于postgres_fdw插件的分布式概念图"
>}}

* 架构图

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180513/pg_fdw_sharding_internal.png|基于postgres_fdw插件的分布式架构图"
>}}

FDW技术应用到了Sharding上,这恐怕也是SQL/MED标准制定者最初没有设想到的吧。但既然社区已经选取FDW这条技术路线作为PG的内置Sharding方案,那么在可预见的未来,社区必然会继续完善FDW的核心功能并积极地增强postgres_fdw扩展的功能.

*注1: 前提是远端的数据源(特别是异构数据源)本身要能够具备这些被下推的能力(JOIN, 聚合, 数据更新等等)*

(*to be continued...*)
