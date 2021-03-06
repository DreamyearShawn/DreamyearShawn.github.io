---
layout: post
title: 性能调优－cpu消耗分析
date: 2016-08-09 09:40:30
category: "java"
---

通常性能瓶颈的表现是资源消耗过多、外部处理系统的性能不足，或者资源消耗不多，但程序的响应速度却达不到要求。

资源主要消耗在cpu，io（又分文件io和网络io），内存方面，机器的资源是有限的，当某资源消耗过多时，通常会造成系统的响应速度变慢。

对于java应用而言，寻找性能瓶颈的方法通常为首先分析资源的消耗，然后结合java的一些工具来查找程序中造成资源消耗过多的代码。

今天先谈一谈cpu消耗如何分析，系统为linux，jdk为sun jdk。

在linux中，cpu主要用于中断、内核和用户进程的任务处理，优先级为中断>内核>用户进程，下面先讲述三个重要的概念。

### 上下文切换

每个cpu（多核cpu中的每个cpu）在同一时间只能执行一个线程，linux采用的是抢占式调度。为每个线程分配一定的执行时间，
当到达执行时间、线程中有io阻塞或高优先级的线程要执行时，linux将切换执行的线程，在切换时要存储目前的线程的执行状态，
并恢复要执行的线程的状态，这个过程就是上下文切换。对于java应用而言，典型的是在进行文件io操作、网络io操作、锁等待或者线程sleep时，
当前线程会进入阻塞或休眠状态，从而触发上下文切换，上下文切换过多会造成内核占据较多的cpu使用，从而使应用响应速度变慢。

### 运行队列

每个cpu核都会维护一个可运行的线程队列，例如一个4核的cpu，java应用里启动了8个线程，且这8个线程都处于可运行状态，
那么在分配平均的情况下每个cpu中的运行队列就会有2个线程。通常而言，系统的load主要由cpu运行队列来决定。

### 利用率

cpu利用率为cpu在用户进程、内核、中断处理、io等待以及空闲5个部分使用的百分比，这5个值是用来分析cpu消耗的关键指标。
在linux中，可通过top或pidstat方式来查看进程中线程的cpu的消耗状况。

- top

输入top命令后既可查看cpu的消耗情况，cpu的信息在top视图的上面几行中，如图

![cpu-top](/images/posts/cpu-top.png)


在这里需要关注第三行信息，下面来逐个介绍。

- us 表示用户进程处理所占的百分比

- sy 表示为内核线程处理所占的百分比

- ni 表示被nice命令改变优先级的任务所占的百分比

- id 表示cpu的空闲时间所占的百分比

- wa 表示为在执行过程中等待io所占的百分比

- hi 表示为硬件中断所占的百分比

- si 表示为软件中断所占的百分比

- st 表示虚拟cpu等待实际cpu的时间的百分比

对于多个或多核cpu，上面的显示则会是多个cpu所占用的百分比总合。如需查看每个核的消耗情况，可在进入top视图后按1，就会按核来显示消耗情况。

![cpu-top](/images/posts/cpu-top2.png)

默认情况下，top视图中显示的为进程的cpu消耗状况，在top视图中按shift + h后，可按线程查看cpu的消耗状况，此时的pid既为线程id。

- pidstat

pidstat是sysstat中的工具，如需使用pidstat，要先安装sysstat，在这里就不说明了。



#### us过高

当us值过高时，表示运行的应用消耗了大部分的cpu。在这种情况下，对于java应用而言，最重要的是找到具体消耗cpu的线程所执行的代码，可以采用如下方法。

- 首先通过linux命令top命令查看us过高的pid值

- 通过top -Hp pid查看该pid进程下的线程的cpu消耗状况，得到具体pid值

- 将pid值转化为16进制，这个转化后的值对应nid值的线程

- 通过jstack pid grep -C 20  "16进制的值" 命令查看运行程序的线程信息

该线程就是消耗cpu的线程，在采样时须多执行几次上述的过程，以确保找到真实的消耗cpu的线程。

java应用造成us过高的原因主要是线程一直处于可运行的状态Runnable，通常是这些线程在执行无阻塞、循环、正则或纯粹的计算等动作造成。
另外一个可能会造成us过高的原因是频繁的gc。如每次请求都需要分配较多内存，当访问量高时就导致不断的进行gc，系统响应速度下降，
进而造成堆积的请求更多，消耗的内存严重不足，最严重的时候会导致系统不断进行FullGC，对于频繁的gc需要通过分析jvm内存的消耗来查找原因。

#### sy过高

当sy值过高时，表示linux花费了更多的时间在进行线程切换。java应用造成这种现象的主要原因是启动的线程比较多，
且这些线程多处于不断的阻塞（例如锁等待，io等待）和执行状态的变化过程中，这就导致了操作系统要不断的切换执行的线程，
产生大量的上下文切换。在这种情况下，对java应用而言，最重要的是找出不断切换状态的原因，
可采用的方法为通过kill -3 pid 或jstack -l pid的方法dump出java应用程序的线程信息，查看线程的状态信息以及锁信息，
找出等待状态或锁竞争过多的线程。

原创文章转载请注明出处：[性能调优－cpu消耗分析](http://9leg.com/java/2016/08/09/cpu-consumption-analysis.html)










