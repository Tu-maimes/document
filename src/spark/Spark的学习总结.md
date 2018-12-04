---
title: Spark的学习总结
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]


# RPC调用的性能模型分析

## 传统RPC调用性能的三宗罪

 1. 兼容性
 2. 可靠性
 3. 序列化后的码流太大

## 高性能的三个主题

 1. 传输
 2. 协议
 3. 线程

![RPC调用性能三要素](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543844794461.png)

# Netty高性能之道
## 异步非阻塞通信

 1. IO多路复用技术。
 2. 低负载、低并发的应用使用同步阻塞IO以降低编程复杂度。
 3. 对于高并发、高负载的应用使用NIO的非阻塞模式进行开发。

![NIO的多路复用模型图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543845950532.png)

 1. Netty架构按照Reactor模式设计和实现的架构图

![NIO服务端通信系列图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543846800281.png)

 2. 客服端通信序列图

![客服端通信序列图](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543847054405.png)

Netty的IO线程NioEventLoop由于聚合了多路复用器Selector,可以同时并发处理上千个客服端Channel,由于读写都是非阻塞的,这就可以提升IO线程运行效率，避免频繁阻塞导致的线程挂起。由于Netty采用异步通信模式，一个IO可以并发处理多个客服端的请求操作。传统同步IO——连接——线程模型。架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

## 零拷贝

Netty的零拷贝主要体现在三个方面

 1. 直接使用堆外内存,减少步骤
 2. 提供Buffer组合对象,避免合并开销
 3. 直接写Channel,避免循环write导致内存的拷贝问题

## 内存池

 - JVM虚拟机和JIT即时编译技术的发展，对象的分配和回收时候是个非常轻量级的工作。
 - 对于堆外直接内存的分配和回收，是一件耗时的操作。为了尽量重用缓冲区，Netty提供基于内存池的缓冲重用机制。

Netty的多种内存管理策略，在启动辅助类中配置相关参数，可以实现个性化定制。

 1. 使用内存池分配器创建直接内存缓冲区
 2. 使用非堆内存分配器创建的直接内存缓冲区

> 通过测试发现：采用内存池的ByteBuf比朝生夕灭的ByteBuf性能更优异。

## 高效的Reactor线程模型

常用的Reactor线程模型有三种：

 1. Reactor单线程模型
 2. Reactor多线程模型
 3. 主从Reactor多线程模型

### 单线程模型

Reactor单线程模型,指的是所有的IO操作都在一个NIO线程上面完成,NIO线程的职责如下:

- 作为NIO服务端,接受客服端的TCP连接
- 作为NIO客服端,向服务端发起TCP连接
- 读取通信对端发送消息或者应答消息

![Reactor单线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543851236858.png)


Reactor模式使用的是异步非阻塞IO,所有的IO操作都不会导致阻塞,理论上一个线程可以独立处理所有IO相关的操作。


#### 单线程的不足有一下三点

 1. NIO线程高负载性能不足，无法处理海量的消息编码、解码、读取和发送。
 2. 高负载导致的消息积压处理超时，达到NIO线程的性能瓶颈。
 3. 可靠性太低，单点故障。

### 多线程模型

多线程与单线程最大的区别是有一组NIO线程处理IO操作。

![Reactor 多线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543929465555.png)

Reactor多线程模型的特点：

 1. 有专门的NIO线程Acceptor线程用于监听服务端，接收客户端的TCP连接请求。
 2. 网络IO操作读、写等由一个NIO线程池负责，线程池也可以采用标准的JDK线程池实现，它包含一个任务队列和N个可用的线程，由这些NIO线程负责消息的读取、解码和发送。
 3. 1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，防止发生并发操作问题。

### Reactor主从多线程模型

利用主从NIO线程模型，可以解决1个服务端监听高负载的问题。

![Reactor主从多线程模型](https://www.github.com/Tu-maimes/document/raw/master/小书匠/1543931538038.png)

## 无锁化的串行设计理念

为了避免锁竞争导致的性能下降,通过实现串行化设计,即消息的处理尽可能的在一个线程里完成,避免线程的切换,导致的线程的竞争和同步锁。

