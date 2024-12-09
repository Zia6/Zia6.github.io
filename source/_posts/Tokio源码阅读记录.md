---
title: Tokio源码阅读记录
date: 2024-12-09 19:45:05
tags: 
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
