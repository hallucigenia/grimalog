---
title: "Python与Golang协程对比"
date: 2020-03-24T11:48:33+08:00
draft: true

categories: ['Article']
tags: ['Coroutine', 'Python', 'Golang']
author: "Fancy"

resizeImages: false
---

从yield到async到goroutine，究竟改变了什么？
<!--more-->

yield是像return一样的关键字，染灰一个生成器。


yield item用于产出一个值，反馈给next()的调用方。
作出让步，暂停执行生成器，让调用方继续工作，直到需要使用另一个值时再调用next()。

协程是对线程的调度，yield类似惰性求值方式可以视为一种流程控制工具，
实现协作式多任务,在Python3.5正式引入了 Async/Await表达式，使得协程正式在语言层面得到支持和优化，大大简化之前的yield写法。
线程是内核进行抢占式的调度的，这样就确保了每个线程都有执行的机会。

而 coroutine 运行在同一个线程中，由语言的运行时中的 EventLoop（事件循环）来进行调度。
和大多数语言一样，在 Python 中，协程的调度是非抢占式的，也就是说一个协程必须主动让出执行机会，其他协程才有机会运行。



一般来说协程都是n:m的协程，即n个协程在m个线程里跑，有一个runtime协程作为调度，所以可以有效利用多核。
但是python的协程是n：1的协程，即n个协程在1个线程里跑，可以实现异步I/O，但是不能有效利用多核。


Go 语言通过系统的线程来多路派遣这些函数的执行，使得 每个用 go 关键字执行的函数可以运行成为一个单位协程。
当一个协程阻塞的时候，调度器就会自 动把其他协程安排到另外的线程中去执行，从而实现了程序无等待并行化运行。
而且调度的开销非常小，一颗 CPU 调度的规模不下于每秒百万次，这使得我们能够创建大量的 goroutine，
从而可以很轻松地编写高并发程序，达到我们想要的目的


Python 中的协程是严格的 1:N 关系，也就是一个线程对应了多个协程。虽然可以实现异步I/O，但是不能有效利用多核(GIL)。
而 Go 中是 M:N 的关系，也就是 N 个协程会映射分配到 M 个线程上，这样带来了两点好处：
多个线程能分配到不同核心上,CPU 密集的应用使用 goroutine 也会获得加速.
即使有少量阻塞的操作，也只会阻塞某个 worker 线程，而不会把整个程序阻塞。



go的协程本质上还是系统的线程调用，而Python中的协程是eventloop模型实现

goroutine 是 m:n 的线程调度器

goalng没有多进程，它是单进程:m线程:n协程的结构，由于CPU的执行最小单元是线程，所以golang可以有效利用多核。但是golang的基础数据结构比如map就不是线程安全，有时候要考虑加锁。那么在加锁的情况下，golang其实这段代码其实同时只有一个协程可以跑。



两种协程对比:
async是非抢占式的,一旦开始采用 async 函数，那么你整个程序都必须是 async 的，不然总会有阻塞的地方(一遇阻塞对于没有实现异步特性的库就无法主动让调度器调度其他协程了)，也就是说 async 具有传染性。
Python 整个异步编程生态的问题，之前标准库和各种第三方库的阻塞性函数都不能用了，requests 不能用了，redis.py 不能用了，甚至 open 函数都不能用了。所以 Python 协程的最大问题不是不好用，而是生态环境不好。
goroutine 是 go 与生俱来的特性，所以几乎所有库都是可以直接用的，避免了 Python 中需要把所有库重写一遍的问题。
Goroutine 中不需要显式使用 await 交出控制权，但是 Go 也不会严格按照时间片去调度 goroutine，而是会在可能阻塞的地方插入调度。Goroutine 的调度可以看做是半抢占式的。

Channels
Channels are a typed conduit through which you can send and receive values with the channel operator <-. And that's all :D You only need to know that when a main function executes <–c, it will wait for a value to be sent. Similarly, when the goroutined function executes c <– value, it waits for a receiver to be ready. A sender and receiver must both be ready to play their part in the communication. Otherwise we wait until they are: you don't have to deal with semaphores, locks, etc: channels both communicate and synchronize. This is really important to remember and understand, and also one of the biggest difference between GoLang and other languages I know.
