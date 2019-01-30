+++
title = "一个小问题: Golang可执行程序的退出码"
author = "xiaowing"
date = 2019-01-30T23:50:00+08:00
draft = false
tags =  [
    "golang",
    "development",
	"程序设计",
    ]
topics = [
    "技术",
]
+++

本周因为工作上的遍历需要, 用Go语言写了一个用于批量文本解析的命令行工具。由于这个工具是要嵌入shell脚本中运行的，所以在写该工具的异常处理时突然意识到一个问题: **Go语言可执行程序的异常退出码该怎么设置？**

于是带着这个非常细节的问题，稍稍调查了一下，结果发现还是有不少东西可说道说道的...

<!--more-->

## 什么是"退出码"

在现代操作系统中, “退出码”指的是子进程退出时返回给父进程的一个**整数值**, 用以标识子进程退出时的状态. 具体而言, 在Linux操作系统中, 子进程退出时通过`exit`系统调用设置退出码; 而父进程通过`wait`系统调用获得子进程的退出码。

当我们在Shell中调用一个可执行文件(最典型的场景: 执行命令行工具)时, 实际上Shell进程会通过`execev`系统调用派生一个子进程来通过加载器加载该工具的可执行文件并执行(通常情况下此时shell进程会阻塞)。当命令执行完毕后，子进程退出，Shell进程继续。这时候通过下述命令即可获得刚刚退出的子进程的退出码

````sh
$echo $?
````

而关于退出码的语义, 事实上并没有一个标准的定义。只是在常年的最佳实践中形成一个共识:

* **零值**表示子进程正常退出
* **非零值**表示子进程异常退出

## C语言可执行程序的场景

在C语言编写的可执行程序中, 通常可以通过以下两种方式来设置**退出码**

1. 在程序的任意位置直接调用`exit`系统调用
2. 在程序的`main`函数中通过它的返回值来设置退出码

对于第一种方式, 这本就是操作系统向父进程传递退出码的最直接方式，所以很容易理解; 但是对于第二种方式中退出码的传递机制, 则需要简单地介绍一下`main`函数的调用原理:

众所周知, C 语言的`main`函数有两种接口定义

````c
/* 无命令行参数 */
int main(void) {
    ...
}

/* 带命令行参数 */
int main(int argc, char *argv[]) {
    ...
}
````

在实际执行可执行程序时, 当**加载器**加载完可执行文件后, `main`函数实际用户代码的[Entry Point](https://en.wikipedia.org/wiki/Entry_point). 而`main`函数返回时, 在当代大部分平台上, 该返回值与其他函数的返回值一样会保存在寄存器`%eax`中。之后加载器便从该寄存器中获取该返回值, 并调用`exit`系统调用从而将退出码传递出来。

因此, 我们可以看出来无论是上述的方式1还是方式2, 本质上都还是通过`exit`系统调用来传递**退出码**。差别只在于是你的可执行代码自己主动调用`exit`; 还是由加载器帮你调用`exit`

*关于可执行程序的main函数调用机制的详细情况，可以参考文章《[Linux X86程序启动 – main函数是如何被执行的](https://luomuxiaoxiao.com/?p=516)》*

## Go语言可执行程序的场景

有了上面的基于C语言可执行程序的背景知识, 接下来让我们再看看Go语言可执行程序的退出码设置方法和其背后的传递机制

### 退出码的设置方法

Go语言的`main`函数的接口是没有返回值一说的, 其接口规范如下:

````c
package main    //必须将main函数定义在名为main的包内

func main() {   // 无参, 无返回值
    ...
}
````

经过实测可以发现, 通过`main`函数正常**return**的情况下, 该程序的退出码永远是**零**。因此，如果要设置一个**非零值**的退出码，就只能通过Go语言标准库中封装好后的`exit`系统调用来设置

````c
os.Exit(code int)
````

比如, 一个用C语言写的小程序，改用Go写, 就会变成如下模样:

* C语言

    ````c
    /* helloc.c */
    int main(int argc, char *argv[]) {
        int opt, upper = 0, lower = 0;
        char *org = "hello, let's see";
        char result[256] = { 0x00 };

        while((opt=getopt(argc, argv,"u:l:"))!=-1){
            switch (opt) {
                case 'u':
                    upper = atoi(optarg);
                    break;
                case 'l':
                    lower = atoi(optarg);
                    break;
                default:
                    ;;
            }
        }

        if (upper <= lower) {
            printf("invalid arguments for upper %d and lower %d", upper, lower);
            return 100;
        }

        strncpy(result, org + lower, (upper - lower));
        printf("%s\n", result);
        return 0;
    }
    ````

* Go语言

    ````c
    //hellogo.go
    func main() {
        str := "hello, let's go"
        upper := flag.Int("u", 0, "the upper bound index")
        lower := flag.Int("l", 0, "the lower bound index")

        flag.Parse()

        if (*upper <= *lower) {
            fmt.Printf("%v", 
                fmt.Errorf("invalid arguments for upper %d and lower %d", *upper, *lower))
            os.Exit(100)    
        }

        fmt.Printf("%s\n", str[*(lower):(*upper)])
    }
    ````

之后可以通过以下shell脚本来测试上述的退出码:

````sh
#!/bin/bash
./helloc -l 3 -u 10
echo $?
./helloc
echo $?

./hellogo -l 3 -u 10
echo $?
./hellogo
echo $?
./hellogo -l 30 -u 40
echo $?
````

可得到执行结果如下:

````sh
$ ./test.sh
lo, let
0
invalid arguments for upper 0 and lower 0
100
lo, let
0
invalid arguments for upper 0 and lower 0
100
panic: runtime error: slice bounds out of range

goroutine 1 [running]:
main.main()
        /hellogo.go:22 +0x19f
2
````

需要注意的是当通过这个go程序触发了一个**panic**时，可观察到其退出码为**2**. 而它并非是在程序代码中所定义的退出码。

接下来我们将研究一下上述Go语言例子中的三个退出码`0`,`100`,`2`具体是在哪里通过什么方式传递出去的。

### Go语言可执行程序的进程启动与退出

我们知道Go语言**可执行程序**编译成本地机器码时，其实是包含了两部分代码:

1. 开发者自己写的代码(包含`main`函数)
2. Go语言自己的运行时代码(包含了GC, goroutine调度器等等)

因此它的执行也是需要由加载器加载起来执行的, 那么必然就衍生出一个问题: Go语言可执行程序对于加载器而言的**Entry Point**在哪里？

查阅了一下 `$GOROOT` 下 `runtime`包的代码, 找到了 `runtime.main()`这个函数:

````c
//$GOROOT/src/runtime/proc.go
func main() {
    g := getg()
    ...(中略)...
    runtime_init() // must be before defer
    ...(中略)...
    gcenable()
    if iscgo {
        ...(中略)...
        cgocall(_cgo_notify_runtime_init_done, nil)
    }
    fn := main_init 
    fn()
    
    ...(中略)...
    if isarchive || islibrary 
    {
        return
    }
    fn = main_main
    fn()  
    ...(中略)...

    if panicking != 0 {
        gopark(nil, nil, "panicwait", traceEvGoStop, 1)
    }

    exit(0)
}
````

从代码中可看见, `main_main`函数有着如下的声明，所以可知它本质上就是用户自己写的`main`函数。

````c
//go:linkname main_main main.main
func main_main()
````

由于`main_main()`的调用只是`runtime.main()`的中间一步. 因此可知, Go语言可执行程序在加载器上的**Entry Point**就是整个这个`runtime.main()`函数. 

*注:上述go:linkname 注释的含义是指定链接时重定向的实际函数名。即main包的main函数*

再次观察`runtime.main()`函数可知，只要用户的`main()`函数能正常结束(即没有**panic**), 那么最终它会走到系统调用`exit(0)`。也就是说，最普通的场景下，可执行程序的退出码必然为**0**。

另一方面, 发生panic的情况下返回2也并非是用户代码中设计的。当程序触发panic时, Go运行时会最终调用`runtime.dopanic_m()`函数，而该函数的最末一行逻辑便是执行系统调用`exit(2)`,从而传递退出码**2**

### os.Exit(int)的陷阱

经过上文可知, 尽管在形式上, Go可执行程序返回退出码有以下三种方式，但他们本质上都是相同的——都是通过系统调用`exit`来传递退出码。由于`os.Exit(int)`就是系统调用`exit`的Go语言封装，那么我们在代码中是不是就可以放心大胆地使用`os.Exit(int)`来退出程序传递退出码呢？

很遗憾, 我们调用`os.Exit(int)`也不能太随意，因为这里有两个坑:

1. 如果我们在`main()`函数里直接传递一个非零值退出码给`os.Exit()`, 那么倘若我们在`main()`函数里事先已经登记了`defer`函数，那么这个`defer`函数将不会被执行。

2. 直接调用`os.Exit`的话, 如果正好同一时刻正好有其他**goroutine**在执行且恰好它触发了panic, 那么这个panic将无法被呈现出来，进而将可能使开发者错过早期发现致命BUG的机会。

    事实上，这个坑就是Go语言开发团队发现的. 在Go语言的语境中，panic应该享有最高优先级, 一旦触发则应当力保它被抛出到最外面。但是由于`os.Exit`是直接退出进程, 反而享有de-facto的最高优先级。因此, Go语言团队为了解决这一问题，专门在`runtime.main()`代码中调用`exit`之前插入了下面这段逻辑用来等待正在触发panic的goroutine执行完.

    ````c
    // runtime/proc.go
    // main()
    if panicking != 0 {
        gopark(nil, nil, "panicwait", traceEvGoStop, 1)
    }

    exit(0)
    ````

    *关于这个坑的详细信息，可以参见Go项目的[Issue 3934](https://github.com/golang/go/issues/3934)*

因此, 在使用`os.Exit`传递自定义退出码时, 上面两个坑需要规避

### 运行时预留的退出码

由于我们希望Go程序能够传递出自定义的退出码的目的实际上还是出于语义上进行状态的区分, 因此我们定义退出码时也自然应当注意语义的唯一性。在Go运行时中预留了以下这些退出码供它自己使用, 开发者在定义自己的退出码时应当避免与它们重叠，以保障程序调用者(如Shell脚本)不至于产生二义性:

| 预留值       | 含义                              |
|:------------|:---------------------------------------------|
| 1           | Go运行时自身与操作系统交互时(申请栈空间，创建新进程等)失败    |
| 2           | 用户代码触发了panic，且未被恢复|
| 3           | 程序panic时也触发了panic (**panic during panic**)  |
| 4           | 程序panic 时遇到了严重错误，导致无法成功打印完traceback信息 |
| 5           | 程序panic时遇到了致命错误, 导致甚至无法开始打印traceback信息 |

## 总结与建议

基于以上总结的关于Go程序的退出码相关Know-how, 如果开发者真的希望自己编写的Go可执行程序能够在退出时传递自定义退出码, 那么在使用`os.Exit()`传递退出码时可以考虑遵循下述最佳实践:

1. 确保`os.Exit()`的调用只出现在最外层的`main`函数中.且这种情况下不要在`main`函数中声明defer调用

2. 调用`os.Exit()`时最好确保已无其他goroutine在并发执行

3. 传递的自定义退出码应避开Go运行时预留的退出码

从代码层面而言, 使用`os.Exit()`向外传递退出码的最佳实践如下所示:

````c
package main

func inner_main() int {
    // 实现原本想在main()函数中实现的逻辑,
    // 通过函数返回值来返回退出码
    ...(中略)...
}

func main() {
    os.Exit(inner_main())
}
````