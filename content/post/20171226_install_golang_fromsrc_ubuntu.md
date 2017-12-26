+++
title = "在Ubuntu 16.04上从源码编译安装go1.9.2"
author = "xiaowing"
date = 2017-12-22T11:37:20+08:00
draft = false
tags =  [
    "golang",
    "ubuntu"
    ]
topics = [
    "技术",
]
+++

这两天在虚拟机上新装了一个 [Ubuntu Server 16.04LTS](http://releases.ubuntu.com/16.04/),于是很自然地想安装一个Go语言环境。以往无论在Windows上还是在Linux上都是用的现成的二进制distribute包来安装Go, 但是这次打算尝试用源码来直接编译安装。于是把本次编译安装的一些手顺和注意事项分享在本文中:

{{< fluid_imgs
  "pure-u-1-1|/img/post/20171226/ubuntu-go.png|Go on Ubuntu"
>}}



<!--more-->

## 编译安装过程

得益于Go语言团队的强大技术，Go语言的编译安装非常智能化，整个编译安装过程非常惬意。不过，由于Go语言从1.5版本开始就实现了[自举](http://www.cnblogs.com/lidyan/p/6727184.html), 因此对于一个只有gcc编译环境的几乎等同于一张白纸的系统，必须先编译安装一个1.5以前版本的Go语言，之后再用这个低版本去编译1.9.2的Go语言。

以下安装过程是在以下环境中实施的

* 操作系统: Ubuntu Server 16.04
* CPU架构:  x86-64
* gcc版本:  5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.5)

### 使用gcc编译Go 1.4

1. Go 1.4的编译依赖于glibc，因此首先需要在Ubuntu上确保安装了新版的glibc

    ````shell
    $sudo apt-get install libc-dev
	````


2. 下载解压 go1.4源码, 并进行编译。由于编译脚本写得非常好，这一步没有什么特别的。

	````shell
	$wget -c -t 3  https://storage.googleapis.com/golang/go1.4-bootstrap-20170531.tar.gz
	$tar zxvf go1.4-bootstrap-20170531.tar.gz
	$cd ~/go/src
	$./all.bash
	````

	all.bash脚本执行编译后，会对1.4的各个pkg进行回归测试。这里可能会有一点问题，具体会在后文描述。总之，当编译完成后，会在`~/go`目录下生成一个`bin`目录，里面有生成的可执行文件`go`; 且所有go语言的标准库会生成在`~/go/pkg/linux_amd64`目录下(我是x86-64的虚拟机，因此目录名是`linux_amd64`。该目录名会因操作系统以及体系架构不同而变化)


3. 使用Go语言自举时，需要创建GOROOT_BOOTSTRAP环境变量，指向低版本的Go语言环境。

	````shell
	$cd ~/go/
	$export GOROOT_BOOTSTRAP=`pwd`
	````

	使用`echo`命令查看该环境变量`$echo ${GOROOT_BOOTSTRAP}`，确认它指向的是Go1.4的路径即可.

	> /home/wing/go

### 使用Go 1.4编译 Go 1.9.2

这样一来，用于自举的低版本的Go语言便已编译安装完毕，接下来就用它编译安装 Go 1.9.2


1. 下载并编译 1.9.2 , 整个过程仍然无比惬意

	````shell
	$wget -c -t 3 https://github.com/golang/go/archive/go1.9.2.tar.gz
	$cd ~
	$tar zxvf go-go1.9.2.tar.gz
	$cd ~/go-go1.9.2/src
	$./all.bash
	````

	如果编译过程没有问题的话，那么标准输出中就会输出类似以下的信息

	> ALL TESTS PASSED
	> ---
	> Installed Go for linux/amd64 in /home/wing/go-go1.9.2
	> Installed commands in /home/wing/go-go1.9.2/bin

    由于我是让这个Go语言语言环境在整个系统内生效，而不是装在我的个人home目录下，因此我还要做以下事项。
	
2. 移至/opt目录下并使安装全局生效

	````shell
	$cd ~
	$sudo mv go-go1.9.2 /opt/.
	$sudo vi /etc/profile
	````

	在`/etc/profile`文件底部增加下述变量导出

	````shell
	export GOROOT=/opt/go-go1.9.2
	export PATH=$GOROOT/bin:$PATH
	````

3. 此时，切换shell会话测试上述安装设置。

	$ go version

	这时理应输出以下信息:
	
	> go version go1.9.2 linux/amd64

至此，整个Go 1.9.2的编译安装就圆满达成

## 一点注意事项

在编译Go 1.4时，`all.bash`最后的回归测试中，有可能会在下述测试中出错并进而打印出出错消息, 从而没有打印出预期的**ALL TESTS PASSED**

    > dial_test.go  all connections connected; expected some to time out

不过，即使出现上述现象也不是意味着编译出了问题。真的要论“锅”属于谁的话，其实它属于 Go语言团队。 根据[这个issue](https://github.com/golang/go/issues/3307)的相关讨论，该现象有大概率会在 64位Linux的 6g环境(即 x84-64 体系的gcc环境)中编译测试稍低版本的Go代码时出现。为了避免这个现象，Go团队也在后续更新中对`dial_test.go`这个测试代码进行了改善(按: 感觉就是把超时时长拉长了一点...)

-以上-