---
title: Tokio源码阅读记录
date: 2024-12-09 19:45:05
tags: 
categories: Tokio
---


## lib.rs
主要是说Tokio的一些feature,其中默认带的是async_read和async_write,其他很多都需要手动打开一些feature,例如net,std_io,runtime等等。
然后就是平台的介绍，目前保证支持的有这些
- Linux
- Windows
- Android (API level 21)
- macOS
- iOS
- FreeBSD

## Runtime

### mod.rs总览
#### 多线程和单线程Runtime的new
- runtime::Runtime::new() 新建一个运行时，默认为多线程，需要开启'rt-multi-thread'
- runtime::Builder::new_current_thread() 新建一个运行在当前线程上的运行时
#### 资源驱动 Resouece drivers
Tokio提供了`Builder::enable_io`和`Builder::enable::enable_time`以及`Builder::enable_all`来开启资源驱动

#### 一些关于调度时的参数
- MAX_TASKS  任何情况下运行时中的任务数量不会超过MAX_TASKS
- MAX_SCHEDULE 任何情况下运行时不会让一个任务运行超过MAX_SCHEDULE个时间单元
- MAX_DELAY 一个任务在被wake后经过MAX_DELAY时间一定会被调度

**Runtime只能尽可能保证任务调度之间的公平，无法保证当多个任务都被Wake时谁先调度**

