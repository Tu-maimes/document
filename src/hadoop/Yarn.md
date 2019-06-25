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