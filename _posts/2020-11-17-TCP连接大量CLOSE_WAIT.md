---
title: TCP连接大量CLOSE_WAIT
key: 20201118
tags: 网络 垃圾回收
---

# 背景
线上一台机器突然出现大量调用其他服务超时异常。过了一遍监控指标，发现网络连接指标存在异常，机器上处于CLOES_WAIT状态的tcp连接数量从某一时刻开始不断上升，并且已经积累了一定的量

![指标1](
https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/%E6%8C%87%E6%A0%871.png)

![指标2](
https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/%E6%8C%87%E6%A0%872.png)

首先回顾一下TCP连接的状态转移，如图：

![TCP1](https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/tcp.jpg
)

CLOSE_WAIT状态处于TCP四次挥手的过程中，当对端主动关闭连接时，会向本端发送FIN，本端回复ACK，此时本端的tcp连接状态就进入CLOES_WAIT状态。接着本端在发送FIN给对端，对端回复ACK，此时tcp连接进入CLOSED状态，完成可tcp连接的关闭。

大量的tcp连接处于CLOES_WAIT状态，说明对端主动关闭tcp连接，本端（被动关闭的一方）没有发出FIN包

![TCP2](
https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/tcp2.jpeg)

# 分析

首先怀疑应用程序代码的问题，调用其他服务的client是框架部门封装的SOA组件，看了一下源码，底层是使用了org.apache.http.nio.client包里的HttpAsyncClient。按理说apache的公共组件出bug的可能性不大，翻看了一下源码也没有发现问题。

只能抓一个dump看看，抓下来后用jprofiler分析，发现存在较多的sun.nio.ch.SocketChannelImpl对象，且数量与CLOSE_WAIT状态的tcp连接数一致。

![socket](https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/socket2.JPG
)

看来是这个对象存在泄漏导致tcp未能正确关闭。通过jprofiler查找其中一个对象的gc root，查看是哪个地方hold住了这些对象，gc root如下

SocketChannelImpl -> SelectionKeyImpl -> .... -> ConcurrentLinkedQueue -> BaseIOReactor

看来这些对象是被hold住在了BaseIOReactor对象中的一个链表成员变量里，名为newChannels，看了一下这个链表在代码中的使用，大概使用的过程是：建立成功的连接会被放入newChannels这个链表里，由一个名为I/O dispatcher的守护线程取出并进行读写等后续操作。

看来这些对象被加入后就没有被取出过，会不会是I/O dispatcher守护线程线程退出了，查看thread dump，果然I/O dispatcher 4 这个守护线程没了，看来是某些原因导致守护线程退出了，这些建立成功的连接被搁置在这，最终对端主动关闭，连接进入CLOSE_WAIT状态

![thread](
https://nothinghappen.oss-cn-shanghai.aliyuncs.com/close_wait/%E7%BA%BF%E7%A8%8B.JPG)

# 解决
至于守护线程为何退出找不到任何线索了，而且这个情况也只发生过这一次，重启应用解决问题
