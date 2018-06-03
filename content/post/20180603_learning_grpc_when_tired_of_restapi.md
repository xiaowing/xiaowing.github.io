+++
title = "[译文]厌倦了REST API设计时的gRPC入门"
author = "xiaowing"
date = 2018-06-03T23:41:52+08:00
draft = false
tags =  [
    "gRPC",
	"微服务",
    "译作",
    ]
topics = [
    "技术",
]
+++

{{< fluid_imgs
  "pure-u-1-1|/img/post/grpc_logo.png|gRPC Logo"
>}}

我对[微服务架构](https://blog.csdn.net/qq_16681169/article/details/73442330)一直都饶有兴趣, 不过由于我目前从事的开发跟这一切半毛钱关系也没有, 所以一直也就只能站在局外人的角度看个热闹. 前端时间在[Qiita](https://qiita.com)上看到一篇关于使用[gRPC](https://grpc.io/)来开发微服务的文章觉得很有意思, 正好gRPC也是我今年想学的技术, 所以就把这篇文章翻译成了中文......

<!--more-->

## 厌倦了REST API设计时的gRPC入门

原题: REST APIの設計で消耗している感じたときのgRPC入門

作者: disc99(disc99.g@gmail.com)

链接: [https://qiita.com/disc99/items/cfca50a32240284578bb](https://qiita.com/disc99/items/cfca50a32240284578bb)

### 面向REST API的设计

最近的系统由于比较重视对于各种各样的终端设备的适配性以及可扩展性, 因此普遍开始采用微服务架构.有了微服务架构,就可以将各个系统彼此分割开并使用轻量的API进行对接. 而在这里所使用的API往往就是所谓的**REST API**

但是, 为了设计这样的API, 要慎重考虑并选择的要素有很多:

* URL, 参数, 异常的设计
* 对应不同编程语言的函数库的设计, 以及对应不同服务器, 客户端的选择与设计
* 认证, 鉴权设计
* API文档管理的方法
* 单元测试, 集成测试, mock, Consumer-Driven Contracts等等, 测试方法的选定
* 开发工具的选择

在缺乏绝对标准规范的状况下, 这些问题将随着系统规模的增大以及团队成员的增加而变得越发复杂, 时间大量地消耗在了设计/管理等方面, 使得原本想要投入功能开发地时间反而越来越少. 

### gRPC是个啥？

gRPC是在2015年2月由Google公开的一款在Google内部也一直被使用的开源RPC框架. 它主要有以下一些特征:

* 默认的数据序列化协议格式是Protocol Buffers(gRPC也支持其他协议格式,不过谈及gRPC通常都是用Protocol Buffers来做例子, 因此本文以使用Protocol Buffers为前提进行说明)
    * 通过[IDL](https://zh.wikipedia.org/wiki/%E6%8E%A5%E5%8F%A3%E6%8F%8F%E8%BF%B0%E8%AF%AD%E8%A8%80)来保证不同语言之间的通信使用一致的数据类型
    * 对应了多种编程语言(C++, Java, Go, Python, Ruby, Node.js, Android Java, C#, Objective-C, PHP等)
    * 保持了数据序列化的兼容性
* 客户端/服务端之间通信用代码可自动生成
* 基于HTTP/2进行通信
    * 并非仅仅是传统的**请求/响应**的通信形式, 也可以使用基于Streaming的双方向通信

### 应用案例

以Google为首, 国内外有大量的应用案例: 

#### 海外

* Google
* Square
* Netflix
* CoreOS
* Docker
* Cockroachdb
* Cisco
* Juniper Networks

#### 国内(*译注: 指日本国内*)

* [Mercari](https://www.mercari.com/)(*译注: 日本的一家C2C的二手交易在线平台*)
* [AbemaTV](https://abema.tv/)(*译注: 日本的一家互联网电视台*)
* [ドワンゴ](http://dwango.co.jp/)(*译注: 日本的一家综合网络娱乐服务提供商, 提供包括网络直播, 网络视频, 网络游戏等一系列娱乐服务*)

### 使用方法

1. `.proto`的定义
2. 生成源代码
3. 服务端, 客户端逻辑的实现
4. 执行

所谓的基本使用方法就是编辑后缀名为`.proto`的定义文件, 将这些定义文件作为输入, 运用命令行工具生成所需编程语言的服务端/客户端代码. 

生成的代码中, 实际上是包含了为了进行C/S间通信的gRPC Server, gRPC Stub, 对应的编程语言接口等一系列代码. 真正在实现服务端/客户端逻辑的时候, 实际上是结合业务逻辑调用所生成的接口代码并在实现中对服务端的环境, 通信设定等进行实现. 

最后, 只需要启动了嵌入了gRPC Server的服务端, 并在客户端经由gRPC stub进行实际的C/S间通信.  相关的概念图如下:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180603/grpc_concept.png|gRPC Concept"
>}}

关于使用的更详细内容, 由于随着所用语言的不通, 在细节上多少会存在一些差异, 而且有很多人也都介绍过了相应的内容, 因此关于这些细节可以去查阅gRPC的官方网站或其他人所写的文章即可. 

由于实际通信时受送信的消息(**数据结构**)以及受送信的服务(**调用逻辑**)都是基于.proto定义文件的记述而自动生成代码, 因此微服务的实现中只需要去调用这些生成的代码即可, 就不需要对原本REST API所需的通信形式以及通信方法进行详细设计了. 

### 适配REST API

gRPC通信时使用的数据格式默认是[Protocol Buffers](https://en.wikipedia.org/wiki/Protocol_Buffers)格式, 而外部工具或者已实现的Web前端中的JavaScript代码仍然期望调用的是REST API. 在这里场景下, 使用[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)对一个基于gRPC开发的微服务进行REST API适配是最方便的途径. 

它实质就是一个gRPC服务的反向代理, 专门用来提供相应的REST API. 
下图很好地说明了它的实质:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180603/grpc_replacing_rest.png|gRPC's reverse-proxy"
>}}

即, API客户端以REST API的形式调用上图的反向代理, 该反向代理又是基于.proto定义文件生成的, 因此将这个REST形式的请求转换成一个基于gRPC的请求, 从而和基于gRPC开发的微服务建立起通信.

使用适配`grpc-gateway`进行适配的方法如下所示:

1. 在.proto文件中追加面向REST API的专有选项
2. 运用命令行工具生成反向代理的代码(Go语言)
3. 在生成的代码中追加基于gRPC开发的微服务的服务器信息等环境信息
4. 执行

使用了`grpc-gateway`就可以让基于gRPC开发的微服务照样可以通过REST API的方式来调用. 而且, 由于生成的反向代理的代码是Go语言的, 生成的代码经过编译后就只是一个可执行文件, 因此也不需要复杂的反向代理服务器配置值类的, 使用起来非常方便. 

另外, 生成的代码仅仅是对微服务本身的反向代理而已, 基于gRPC开发微服务仍然可用任何编程语言.

更为便利的是, `grpc-gateway`还提供了面向Swagger的JSON定义生成功能,这样一来, 经过适配的gRPC微服务的REST API文档也可以直接使用 Swagger UI来进行输出了. 

{{< fluid_imgs
  "pure-u-1-1|/img/post/20180603/adjusted_service_swagger_output.png|适配后的微服务在Swagger上的输出例"
>}}

### 残留的课题

尽管使用gRPC构筑微服务有很多好处, 现时点仍然残留了不少课题:

* 面向HTTP/2的基础环境构建
* 序列化后的通信内容如何监控
* 前端不再能直接使用JavaScript来调用微服务(需要经过`grpc-gateway`中转适配)
* 开发团队对于新技术的学习成本

### 总结

尽管 REST API已经被广泛普及了, 但是它其实并非拥有一个固定下来的绝对的指导原则. 由于自由度很高, 因此交由开发者进行选择的要素有很多. 而这有时就相当于是一把悬在头顶的达摩克利斯之剑——因为一旦选择出现了失误, 它将极可能立刻转变成团队的技术债.

而gRPC则与之不同, 它有相当多确定的(Fixed)特性规格, 因此可以把 各种语言的服务端/客户端代码的生成, 性能问题, 以及必要时的REST API适配和Swagger UI对应等问题统统交给它. 当然, gRPC**不是银弹**,  它并不能解决所有问题, 但是使用它的话由于可以尽快地让通信功能动起来, 因此在某种程度上可以让我们把精力和资源更加集中在本就应该优先展开的业务功能开发上. 

因此, 如果你觉得你的团队在REST API的设计方面, 或者在REST API开发的规划方面已经花费了大量时间的情况下, 我认为是值得探讨一下使用gRPC的可行性的. 

