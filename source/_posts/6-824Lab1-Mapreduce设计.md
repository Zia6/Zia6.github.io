---
title: 6.824Lab1-Mapreduce设计
date: 2025-05-20 16:33:57
tags:
catagories: 6.824分布式系统
---
 
## 概述
用户传递给我们 n 个 files，map 和 reduce 函数，以及一个 nreduce，我们要做的就是把这 n 个 files 经过 map 和 reduce后输出 nreduce 的 output 还给用户

## 框架以及我们需要做的事情
这个 lab 为我们提供了 coordinator 和 worker，我们通过coordinator 去控制多个 worker 来**并行**处理 map 操作或者 reduce 操作。


## 思路
首先考虑 coordinator 和 worker 之间的交互过程，我这里的实现方案是让 worker 去不断的轮询coordinator，这样的话我们的 coordiantor 不需要知道有多少个 worker，它只需要当有请求来的时候，把任务分给 worker 去操作就可以了，我们想搞几个 worker 搞几个 worker，然后大致流程是这样的
- worker 询问 master 请求分配任务
- master 返回请求的任务，worker 开始运行任务(这里如果没有任务要退出，或者还在等其他 map 任务的话可以考虑睡眠)
- worker 完成任务后告诉 master 任务完成了
- master 更新任务状态
- 当所有任务完成后，worker的请求分配请求收到 master 的-1，关闭 worker。
现在的话还没有想好怎么关闭 master 

## 实现要点

### 实现 RPC 需要注意的地方
方法必须满足 Go 语言的 RPC 规则：方法只能有两个可序列化的参数，其中第二个参数是指针类型，并且返回一个 error 类型，同时必须是公开的方法。


