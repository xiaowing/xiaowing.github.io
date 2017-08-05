+++
title = "主线程等待子线程结束的各语言实现"
author = "xiaowing"
date = 2017-08-05T22:18:06+08:00
draft = false
tags =  [
    "C",
    "python",
    "golang",
    "development",
    "并发",
    ]
topics = [
    "技术",
]
+++

在涉及到并发编程的情况下，经常性地会碰到一种场景:

> 由一个线程开启了多个线程并发执行多个任务，之后由该线程(so called "主线程")等待多个线程都结束后汇总结果.

这种场景下，主线程在其创建的子线程执行期间内需要阻塞，直到其他子线程都执行完毕。由于这类场景已经在不同语言的开发中遇到多次，所以汇总一下这些语言的常用实现方法，以后查起来也方便~

<!--more-->

1. C语言实现
C语言在操作多线程方便由于缺乏一个统一的标准库，所以在Linux和Windows上各有各的实现方法:

    * Linux版实现
    
        在Linux上的实现，主要是基于POSIX thread库进行实现，实例代码如下:

        ````c
        #include <pthread.h>

        int main(void) {
            int i;
            pthread_t threads[THREAD_NUM];

            pthread_setconcurrency(THREAD_NUM);

            for (i = 0; i < THREAD_NUM; i++){
                /* 将需要执行的job的函数地址func传入新建的子线程 */
                pthread_create(&threads[i], NULL, func, NULL);
            }

            for(i = 0; i < THREAD_NUM; i++){
                pthread_join(handles[i], NULL);
            }

            /* 后续的处理逻辑略... */
        }
        ````

    * Windows版实现

        在Windows版上的实现中，主要是基于Windows API实现。由于Windows本身就和Linux就是风格迥异，因此在阻塞主线程的API设计上，也是略有不同：

        ````c
        #include <Windows.h>

        int main(void) {
            int i;
            HANDLE handles[THREAD_NUM];

            for (i = 0; i < THREAD_NUM; i++){
                /* 将需要执行的job的函数地址func传入新建的子线程 */
                handles[i] = CreateThread(NULL, 0, func, NULL, 0, NULL);
            }

            WaitForMultipleObjects(THREAD_NUM, handles, TRUE, INFINITE);

            /* 后续的处理逻辑略... */
        }
        ````

        需要注意的是，Windows API的`WaitForMultipleObjects()`其实不仅仅是为多线程场景服务的，它可以用与多种内核句柄，如`Event`，`Mutex`，`Process`，`Thread`，`Semaphore`.

2. Python3的实现

    Python3的实现形式与C语言的Linux版类似: 当子线程仍活着的时候，则通过类似Join()方法之类的API来阻塞当前的主线程。代码示例如下:

    ````python
    from threading import Thread

    if __name__ == '__main__':
        thread_list = []

        for x range (0, THREAD_NUM):
            # 将需要执行的job的函数和参数传入新建的子线程
            t = Thread(target=job_func, args=(job_args,))
            t.start()
            thread_list.append(t)
        
        for element in thread_list:
            if element.is_alive():
                element.join()    # 通过join方法阻塞主线程
        else:
            # 后续的处理逻辑略...
    ````

3. Golang的实现

    由于Golang的语言特色，并发通过goroutine来实现。通常情况下，各个goroutine根本不需要知道彼此的存在。因此对于这个场景的实现方式，与之前的那些语言都有所不同.代码示例如下:

    ````
    package main
    
    import (
        "sync"
    )

    func main() {
        var wait sync.WaitGroup
        wait.Add(ROUTINE_NUM)

        for i := 0; i < ROUTINE_NUM; i++ {
            go func() {
                defer wait.Done()

                //Goroutine所要执行的Job逻辑略...
            }()
        }
        wait.Wait()

        // 后续的处理逻辑略...
    }
    ````

    在之前的语言中，主线程在起了多个子线程后，不管用什么API阻塞，主线程或多或少还需要关注一下子线程(句柄等)，但是在Golang中，主协程不需要关注各个携程。主协程等待其他协程的这个场景，完全基于Workgroup就可以简单实现。

    真不愧是一门面向并发的语言:)

最后需要说明的是，虽然本文描述的这个场景叫做"主线程等待子线程"。但实际上，无论是线程模型还是协程模型，线程与线程之间(协程与协程之间)都是平等的。毕竟只有在多进程模型下，被fork出的子进程会继承父进程的大部分数据(如打开的文件描述符)，完全相当于父进程的副本的形式。而这样的关系在线程模型(或协程模型)中并不存在，此处的说法完全只是遵循某种不成文的惯例，算是"**阀值**"之于"**阈值**"这样的错误吧。

