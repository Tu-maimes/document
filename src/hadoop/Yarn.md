---
title: Yarn
tags: 
grammar_cjkRuby: true
---
# 简介

## 需要yarn的特性

- 可扩展性
- 可维持性
- 多租户
- 位置感知
- 高集群使用率
- 安全和可审计的操作
- 可靠性和可用性
- 对编程模型多样性的支持
- 灵活的资源模型
- 向后兼容


JobTracker的内存管理
JobHistoryServer是为了缓解JobTracker的内存压力而提出来的。


Capacity调度器
FIFO公平调度器    在调度时最重要的是数据的本地性偏好
JobTracker负责工作服务器节点的资源管理，跟踪资源使用率、可用率，管理作业的生命周期，如调度作业的各个任务，跟踪进度，以及为任务提供容灾服务。
TaskTracker的职责比较简单——根据JobTracker的命令启动、清楚任务，并周期性地向JobTracker提供任务的状态信息。


NodeManager

- 保持与ResourceManager的同步
- 跟踪节点的健康状况
- 管理各个Container的生命周期，监控每个Container的资源使用情况
- 管理分布式缓存（对Container所需的JAR、库等文件的本地文件系统缓存）
- 管理各个Container生成的日志
- 不同的YARN应用可能需要的辅助服务

Container被杀死：
 
 - 任务已完成
 - 调度器做出要为其他应用程序或者用户抢占该Container的决策
 - NodeManager监控到这个Container超出了ContainerToken中指定的资源限制