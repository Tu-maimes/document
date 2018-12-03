---
title: Spark的学习总结
tags: 作者:汪帅
grammar_cjkRuby: true
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


