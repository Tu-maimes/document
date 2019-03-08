---
title: Spark Shuffle参数调优的原理与建议
tags: 作者:汪帅
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---

[toc!?direction=lr]

## Shuffle对性能消耗的原理详解

### Spark Shuffle过程中影响性能的操作：

 1. 磁盘I/O
 2. 网络I/O
 3. 压缩
 4. 解压缩
 5. 序列化
 6. 反序列化

调优是一个动态的过程，需要根据业务数据的特性和硬件设备来综合调优。

