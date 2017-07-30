+++
title = "C语言程序通过函数指针调用Go函数的方法"
author = "xiaowing"
date = 2017-07-30T16:16:03+08:00
draft = false
tags =  [
    "cgo",
    "golang",
    "development",
    ]
topics = [
    "技术",
]
+++

在github上关于cgo的wiki中，有一[章节](https://github.com/golang/go/wiki/cgo#function-pointer-callbacks)专门介绍了如何利用cgo技术通过函数指针调用Golang的函数实现. 不过，仔细观察这个章节的代码示例可以发现，它所要解决的其实是以下的场景:

> 在Golang中想要调用一个已有的C语言函数，但是该C语言函数要求一个函数指针作为参数时应该怎么办？

如果将这个场景稍微改变一下，改成以下场景，对应的解法又该是什么？

> 在一个C语言实现的已有系统中，对于一个要求函数指针的函数，如何传入一个Golang实现的回调函数以实现“用Golang扩展C语言系统”的目的。

我基于wiki中已有的代码简单探索了一下方法，结果分享如下：

<!--more-->

## 试验代码的准备

1. 首先，需要一个声明了函数指针类型的头文件(也就是C语言和Golang的接口)。这里流用了上述wiki中的示例：

    ````c
    /* clib.h */

    #ifndef CLIBRARY_H
    #define CLIBRARY_H
    typedef int (*callback_fcn)(int);
    #endif
    ````

2. 接下来是C语言程序中调用上述函数指针的入口函数.这个文件也是从wiki中流用的.

    ````c
    /* clib.c */
    #include <stdio.h>
    #include "clib.h"

    void some_c_func(callback_fcn callback)
    {
        int arg = 2;
        printf("C.some_c_func(): calling callback with arg = %d\n", arg);
        int response = callback(2);
        printf("C.some_c_func(): callback responded with %d\n", response);
    }
    ````

    在这个程序中，没有定义callback_fcn这个函数指针的具体实现。这个实现将交给下面的Golang进行

3. 在Golang中实现回调函数

    ````
    /* goprog.go */
    package main        /* 包名必须是main */

    /*
    #cgo CFLAGS: -I {clib.h的路径(目录)}

    #include "clib.h"

    int callOnMeGo_cgo(int in); // Forward declaration.
    */
    import "C"

    import "fmt"

    //export callOnMeGo
    func callOnMeGo(in int) int {
        fmt.Printf("Go.callOnMeGo(): called with arg = %d\n", in)
        return in + 1
    }

    func main() {}        /* 必须定义一个空的main函数 */
    ````

    这个文件基于Wiki中的示例稍微改了一点，把main函数的实现给去掉了，但保留了一个空的main函数。 此外，不论这个文件在哪里创建，它的package被定义为main。相关的理由如下：

	* cgo在将go源代码编译成shared-library的过程中，只会将package声明为main的源代码纳入编译，其余的文件都会被忽略
	* 由于参与编译的源代码的package都为main，根据Golang编译器的规则，则必须有一个main()函数，否则编译不过
	* 根据第2点，如果参与编译的源码中有超过一个main函数，编译器也会报错。
    
    注意: 这里有一个坑：如果将带main函数的.c文件和这些go文件放在一起，然后启动golang编译器编译器编译，也会报错，说main函数数量过多。不知golang编译器为什么要去识别C语言的main函数...

4. 这样一来，回调函数的实现本体就已经完成了。但是如果仅仅如此，是无法实现C语言调用这个函数的，这是因为两种语言的类型不一致，因此实际上上述回调函数的接口与函数指针的声明仍然不一样，所以需要一个**Adapter**。在cgo中，这样的**Adapter**被称为"`Gateway Function`". 直接搬用Wiki中的代码即可：

    ````
    /* cfuncs.go */
    package main

    /*

    #include <stdio.h>

    // The gateway function
    int callOnMeGo_cgo(int in)
    {
        printf("C.callOnMeGo_cgo(): called with arg = %d\n", in);
        int callOnMeGo(int);
        return callOnMeGo(in);
    }
    */
    import "C"
    ````

    有意思的是，这个Gateway Function实际上是一个实现在go源码注释中的C语言函数，它的声明与函数指针一致。但是它实际封装的却又是一个golang函数.

## 构建过程

到这时为止，所需的代码就算是写完了，接下来需要把程序构建并运行起来：

1. 用golang编译器构建共享库:

    ````shell
    $go build -buildmode=c-shared -o libgoprog.so {所有参与编译的go源码}
    ````

    值得注意的是，`-buildmode=c-shared` 是直到**golang1.5**开始才有的选项，且该选项到目前为止(golang1.8)只支持linux平台，**不支持windows和Mac OS**.

    编译成功后，会生成两个文件： 一个是库文件(`libgoprog.so`), 另一个是该库文件对应的头文件(`libgoprog.h`).这个头文件的片段如下：

    ````c
    /* libgoprog.h */

    #include "clib.h"
    int callOnMeGo_cgo(int in);  /* <- 在go源码中定义的Gateway Function */

    ...(中略)...

    extern GoInt callOnMeGo(GoInt p0);   
    ...(下略)...
    ````

    这时如果用`file`命令看一下生成的.so文件，应该是类似以下的结果：

    > libgoprog.so: ELF 64-bit LSB  shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=f49bbe5d2d38c184574b65ed11f55e84c1ad19e3, not stripped

    同时，如果用`nm`查看这个.so文件的导出符号，就可以看到callOnMeGo和callOnMeGo_cgo这两个符号了

2. 由于上一步骤生成了这个头文件，所以此处还需要修改一下之前的 clib.c 文件，把这个头文件给 #include 进去，从而就可在clib.c中看见那个Gateway Function的声明了，而且此时就可以为clib.c文件补上 main() 函数的实现了：

    ````c
    /* clib.c 完整版 */

    #include <stdio.h>
    #include "clib.h"
    #include "libgoprog.h"    /* 追加头文件引用 */

    void some_c_func(callback_fcn callback)
    {
        int arg = 2;
        printf("C.some_c_func(): calling callback with arg = %d\n", arg);
        int response = callback(2);
        printf("C.some_c_func(): callback responded with %d\n", response);
    }

    int main(void) {        /* 追加main()函数实现 */
        some_c_func(callOnMeGo_cgo);
    }
    ````

3. 编译这个C程序

    ````c
    $gcc clib.c -I{clib.h的目录路径} -I{生成的libgoprog.h的目录路径} -L{libgoprog.so的目录路径} -lgoprog -o clibmain
    ````

4. 一切正常的话，就可以正常生成可执行文件clibmain了。之后再将先前生成的libgoprog.so 放置到链接器可找到的路径下，执行该程序就可得到下述结果了:

{{< fluid_imgs
  "pure-u-1-1|/img/post/c-calls-go-output.jpg|通过函数指针调用golang函数的输出"
>}}

## 总结

综上，使用golang自带的cgo技术，可以方便地打通C语言和Golang语言。但目前，Go语言编译动态库还只能在Linux平台上实现，需要注意。

另外，考虑到两种语言在数据类型上还是存在较多差异(事实上，编译生成共享库时附带生成的头文件中就定义了大量golang类型到C语言的映射)，因此，如果真的要编写程序在C语言中调用Go，其实有相当一部分工作量应该会花在数据类型转换上。







