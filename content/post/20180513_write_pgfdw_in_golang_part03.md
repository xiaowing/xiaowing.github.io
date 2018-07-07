+++
title = "双剑合璧——当PG的FDW遇上GO(之三)"
author = "xiaowing"
date = 2018-05-13T23:02:33+08:00
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

* [第二部分: 揭秘FDW](/post/20180513_write_pgfdw_in_golang_part02/)

* **第三部分: 如何用Go来实现一个FDW**

<!--more-->

## 3. 如何用Go来实现一个FDW

上文介绍了PG中的FDW的历史以及运行机制,接下来聊一个有趣的话题——如何用Go语言来实现一个FDW.

### 3.1 为什么要用Go来写FDW?

由于PG以C语言写成,因此其内部公开的接口(无论是FDW的回调函数接口还是供FDW使用的内部接口)都是面向C语言设计的,因此用C语言实现FDW是最自然的选择。

但是用C语言实现有着一个比较令人头痛的问题——开发效率低下。通常我们需要基于一个Idea来为一个新的数据源编写一个FDW以实现PG与新数据源的对接。但是用C语言编写FDW很容易使得我们在迅速陷入一些实现的细枝末节中从而丧失了开始编写FDW时的趣味,这个问题主要是源于C语言在开发效率上的一些先天性不足: 

  * 需要编码者自行分配/释放内存
  * 字符串设计过于底层缺乏必要的抽象和封装,使得字符串处理必须小心翼翼
  * 标准库函数的功能过于薄弱,许多看上去很基础的功能要么必须由编码者重新造轮子,要么必须依赖来源并不是那么靠谱的第三方库

为了将FDW的开发者从繁琐的底层细节中解放出来,广大PG爱好者也做了不少实践: 比如有一个叫做["Multicorn"](http://multicorn.org/)的FDW框架,它封装了FDW相关的底层细节使得开发者们可以用Python快速地编写FDW。尽管这是一个很棒的开源项目,但遗憾的是,这个框架依赖于PG的**PL/Python**组件,而它往往并非默认安装的(甚至在编译PG源码时,它默认不会参与编译), 而且[它至今仍然不是一个**受信**的存储过程语言组件](https://www.postgresql.org/docs/10/static/plpython.html)。

考虑到Go语言与C语言之间相对友好的交互机制(比起Java的`JNI`机制要友好很多),而且相对于C语言的三个缺点GO语言都可以在语法层面或者内置功能层面可以很好地应对,更何况在语法层面GO语提供了诸多类似`defer`, `go routine`等简化编码复杂度的利器,因此使用Go语言可以帮助我们**相对快速地实现一个FDW**。

需要注意的是,此处提及的使用Go语言编写FDW的概念严格意义上说应该是使用**cgo**来编写FDW。为了简单化概念,因此本文将继续使用"Go语言"来称呼,但需要牢记**Dave Cheney**所撰写 [cgo is not Go](https://dave.cheney.net/2016/01/18/cgo-is-not-go) 

### 3.2 使用Go编写FDW的方法论

#### 回调函数的cgo实现

由于实现FDW的核心就在于实现那一系列回调函数. 按照[cgo的官方介绍](https://github.com/golang/go/wiki/cgo), 我们实现FDW的回调函数时通常只需要做以下几件事情:

1. 在实现回调函数的go文件中声明`import "C"`, 这是使用cgo所必须的
2. 按照回调函数的接口规格予以GO语言的函数定义.PG的回调函数接收的都是PG内部的数据类型,因此在定义函数时参数应依照cgo中映射C语言数据类型的规则进行声明
3. 将刚刚定义的GO语言函数的函数名按cgo的规则在注释中予以导出
4. 使用GO语言**实现**该回调函数的逻辑

比如,在我的个人项目 [douban_fdw](https://github.com/xiaowing/douban_fdw)中,对于回调函数`GetForeignRelSize_function`,用cgo实现的代码示例如下:

````c
//export doubanGetForeignRelSize
func doubanGetForeignRelSize(root *C.PlannerInfo,
	baserel *C.RelOptInfo, foreigntableid C.Oid) {
	var referredAttrs *C.Bitmapset

	// Collect all the attributes needed for joins or final output.
	targetlist := (*C.Node)(unsafe.Pointer(baserel.reltargetlist)) // TODO: member field of 'RelOptInfo' changed in 9.6
	C.pull_varattnos(targetlist, baserel.relid, (**C.Bitmapset)(unsafe.Pointer(&referredAttrs)))

	// Add all the attributes used by restriction clauses.
	restrictNum := int(C.list_length(baserel.baserestrictinfo))
	for i := 0; i < restrictNum; i++ {
		rinfo := (*C.RestrictInfo)(unsafe.Pointer(uintptr(C.list_nth(baserel.baserestrictinfo, C.int(i)))))
		C.pull_varattnos((*C.Node)(unsafe.Pointer(rinfo.clause)), baserel.relid, (**C.Bitmapset)(unsafe.Pointer(&referredAttrs)))
	}
	
	// check if the name of the referred attrs are valid
	attributesRetrieved := referredFieldsValidator(foreigntableid, referredAttrs)
	C.bms_free(referredAttrs)

	baserel.fdw_private = Save(attributesRetrieved)
	baserel.rows = C.double(MovieRankingTop250Num)
	//TODO: width
}
````

不过,虽然从道理上实现FDW的回调函数就这几个步骤,但实际实现起来,其中还是有不少坑的。而上述的示例代码正好体现了这一过程中几个最大的坑:

1. 无法直接使用的PG接口

    通常情况下, 在cgo编程时我们可以用形如`C.xxx`的形式直接使用原本在C语言中定义的类型,函数等等。但是如果碰上了C语言的宏定义,这种方法就失效了。如果尝试用`C.xxx`来使用一个宏,将会引发一个编译错误。

    这个问题会给我们使用PG的接口带来一定麻烦,比如对于PG中的`List`类型, PG的代码中提供了一个宏`foreach`来方便开发者遍历`List`:

    ````c
    #define foreach(cell, l)	\
	    for ((cell) = list_head(l); (cell) != NULL; (cell) = lnext(cell))
    ````

    在原本PG代码中,则可以使用下述方式来方便地遍历`List`

    ````c
    foreach(cell, list)
    {
      .../* ~(对当前遍历地元素进行处理,代码略)~ */
    }
    ````

    但是在cgo实现中,上述宏无法使用,且就算将该宏展开后仍然有宏定义嵌套,因此不得不采用别的迂回方法来遍历`List`。在上文地示例代码中遍历`List`的方式就变成了下述形式——先获取元素个数,再挨个取第n个元素进行处理:

    ````c
    restrictNum := int(C.list_length(baserel.baserestrictinfo))
    for i := 0; i < restrictNum; i++ {
      rinfo := (*C.RestrictInfo)(unsafe.Pointer(uintptr(C.list_nth(baserel.baserestrictinfo, C.int(i)))))
      ...(代码略)...
    }
    ````

    此外,除了极具代表性的`foreach`宏,PG代码中随处可见的`ereport`宏函数, `heap_close`宏函数,以及各种类型的SQLSTATE宏(如`ERRCODE_FDW_ERROR`)等都无法在cgo代码中直接使用,需要迂回解决

2. 指针类型的强制类型转换

    在上文的示例代码中,有类似以下的代码：

    ````c
    targetlist := (*C.Node)(unsafe.Pointer(baserel.reltargetlist))
    ````

    这是由于PG的`pull_varattnos()`函数接收的是一个`Node *`类型,但如果直接用`baserel.reltargetlist`得到的是一个`List *`类型,因此需要做强制类型转换(注: 这种转换根据PG代码,逻辑上是能转,因为PG中大量的类型都是 `Node`的派生类型)

    但是,GO语言中并不允许指针类型之间的强制转换,如果不得不转的话必须用`unsafe.Pointer`类型进行过渡. 这就造成了只要代码中涉及类似的强制类型转换,代码就会变得冗长。

3. 符合cgo规范的GO指针传递

    在上文的2.4章节曾介绍过FDW的回调函数之间如何传递私有数据。由于我们是用Go语言来实现FDW的回调函数,因此很自然会涉及到一个问题: 使用Go语言创建的私有数据(通常指结构体之类)如何进行传递? 

    本来这个问题不该成为一个问题,因为根据2.4章节的描述,把GO语言的结构体挂在相应的PG数据类型中即可。然而很遗憾的是,从GO 1.6开始, cgo的规范对于GO运行时内存中的指针向C语言运行时传递进行了很严格的限制(详讯[Rules for passing pointers between Go and C](https://github.com/golang/proposal/blob/master/design/12416-cgo-pointers.md)). 在此处的FDW回调函数之间传递私有数据这个场景下,我们无法直接将一个存在于GO运行时的指针直接挂在一个PG的结构体中并让其带着往下传递。

    这个问题的解决可以借鉴来自日本的GO语言大咖**mattn**所提供的一个[解决方案](https://github.com/mattn/go-pointer)。简而言之就是需要传递一个GO指针到C运行时中时,先利用cgo从C语言侧申请一个字节的内存,之后在一个全局哈希表中建立这个字节的地址与GO语言结构体的映射。这样在PG的数据结构的字段中保存的仍然是C运行时的地址,但当回调函数被调用,控制流程回到GO语言侧时,就可以通过映射关系顺着保存的C语言地址找到对应的GO语言结构体.

    上文示例中的语句`baserel.fdw_private = Save(attributesRetrieved)`就是在利用**mattn**提供的解决方案做这样的映射传递。

    不过，在mattn的解决方案中，对于全局哈希的访问是通过mutex来确保并发安全的。在应用到PG的场景时，由于一个会话对应于一个进程，因此这样的互斥是没有必要，可以去除。

#### 如何构建

由于FDW的实现本质上就是一个PG的扩展(Extension), 因此FDW的构建首先应当寻求利用PG的[Extension Building Infrastructure](https://www.postgresql.org/docs/10/static/extend-pgxs.html). 那么,当我们使用cgo编写FDW时, 我们就可以为FDW代码编写类似下面这样的**Makefile**:

````
MODULES = {模块名}
EXTENSION = {FDW名}
DATA = {数据库对象的安装sql脚本名}

# {自定义变量声明}

# {cgo模块的构建规则定义}

PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
````

需要注意的是, [pg_config](https://www.postgresql.org/docs/current/static/app-pgconfig.html)是PG提供的用于获取安装环境信息的一个命令。它可以返回一个PG安装环境中的相关信息，如：PG的库文件安装路径, PG的二进制文件的安装路径，该PG的版本号等等。通常，一个利用Extension Building Infrastructure的Makefile最终会需要用到全局的PGXS Makefile, 而该全局Makefile的路径则需要由`pg_config`来提供，因此在类似上文这样的Makefile之前，必须首先确保PG安装环境中的`pg_config`命令的路径(即，PG的安装目录中的`bin`目录)一定要在用户的`PATH`环境变量中.

此外, 在编写FDW时我们的cgo代码会需要调用PG内部的一些函数，因此我们的代码会需要依赖PG的内部头文件以及相关的动态库. cgo本身提供了一些变量让开发者指定编译/链接时的选项:
  
  * `CGO_CFLAGS`   指定编译时选项
  * `CGO_LDFLAGS`  指定链接时选项

利用这些选项，再加上先前所述的`pg_config`命令, 我们就可以在上述Makefile的变量声明部分指定FDW所需依赖的编译/链接时变量了。

而对于Makefile的构建规则定义，只需要定义好cgo代码的编译规则和对应的命令，以及FDW安装/卸载时的一些文件移动/删除的命令即可。这些与大部分Makefile并无区别，故不再详述。

### 3.3 注意事项

以上就是对使用GO语言实现FDW的简单介绍。不过还需要再三强调的是，虽然使用GO语言实现FDW在编写代码的效率上会大大提高，复杂度也会有显著降低。但是必须牢记的是，GO调用C语言代码以及C语言回调GO代码都会导致[栈的切换(stack switch)](https://dave.cheney.net/2016/01/18/cgo-is-not-go), 即GO routine栈和C运行时栈的切换。因此，与`Multicorn`框架所建议的一样——对于一些性能攸关的场景(如作为PG官方sharding方案的postgres_fdw), 最好还是使用纯C语言来编写。

## 4. 总结与展望

综上所述，尽管PG的FDW以及SQL/MED规范就目前而言，还是一个比较小众的技术话题。但是我们可以看见从PostgreSQL 9.2 到 10.0, 每一个PG版本的升级都会带来FDW功能的大幅度增强，因此可以相信随着基于postgres_fdw的Sharding方案日趋成熟，FDW技术将会得到越来越多的关注。毕竟 

Writing a FDW is fun.

(*fin*)

*本文已投稿至 PG中文社区 以及 DBAPlus社区，并获发表*

* *[http://www.postgres.cn/news/viewone/1/337](http://www.postgres.cn/news/viewone/1/337)*
* *[http://dbaplus.cn/news-19-2090-1.html](http://dbaplus.cn/news-19-2090-1.html)*

## 参考资料

* Peter Eisentraut, "SQL/MED and PostgreSQL"[EB/OL], https://wiki.postgresql.org/images/4/4c/SQLMED-FOSDEM2009.pdf, 2009
* Florian Schwendener, "SQL/MED and More: Management of External Data in PostgreSQL and Microsoft SQL Server"[EB/OL], https://wiki.hsr.ch/Datenbanken/files/SQLMED_and_More_Schwendener_Paper.pdf, 2011
* 花田 茂, "外部データラッパによるPostgreSQLの拡張"[EB/OL], https://www.slideshare.net/babystarmonja/postgre-sql-11764943, 2012
* Bernd Helmle, "Writing A Foreign Data Wrapper"[EB/OL], https://wiki.postgresql.org/images/6/67/Pg-fdw.pdf, 2012
* 澤田 雅彦, "FDW-based Sharding Update and Future"[EB/OL], https://www.slideshare.net/masahikosawada98/fdwbased-sharding-update-and-future, 2017
* Bruce Momjian, "The Future of Postgres Sharding"[EB/OL], https://momjian.us/main/writings/pgsql/sharding.pdf, 2018
