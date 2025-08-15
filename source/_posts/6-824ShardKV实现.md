---
title: 6.824ShardKV实现
date: 2025-08-15 16:34:22
tags:
categories: 6.824分布式系统
---

## 概述
这也是 6.824 的最后一个 lab，即在之前 raft 和 RSM 的基础上，实现一个multi raft 的分片存储来提高性能，整个集群的性能会随着分片数量的增加而线性提高
### 整体架构
对于这个集群，主要是分为三部分，shardctrler、shardGroup 以及 shardClient，每个 shardGroup 由一个 client 和 raft server集群组成，shardctrler负责对整体集群分片存储的配置，shardClient由用户进行使用，shardctrler和shardClient都只能和 shardGroup 进行交互，如图
![alt text](shardkv.png)
- shardctrler会将配置的信息存入到 kvsrv 中，通过 RPC 与shardGroup进行交互，进行碎片迁移
- shardClient每次操作先获取 kvsrv 中的信息得到每个分片对应的shardGroup，再和shardGroup进行交互

### 实现思路
我们从初始状态开始实现，首先实现shardctrler的 InitConfig 和 Query，这里要考虑我们是否需要让shardGroup知道自己控制的是哪些分片，很显然是需要的，因为我们后续在处理配置更改时希望能处理没有变化的分片请求，还有一个原因是网络或者 RSM 可能会导致请求的乱序导致处理这个请求时它已经不负责这个分片了，如果它不知道这个信息，那么就可能会导致数据的不一致，因此我们需要让shardGroup知道自己控制的是哪些分片，因此我们需要有一个初始的状态，也就是 InitConfig 时我们需要有某种办法从0 加载出这个配置，我们这里是保证 InitConfig的状态一定是所有分片都给 pid1
#### 初始状态的实现
```go
func (sck *ShardCtrler) InitConfig(cfg *shardcfg.ShardConfig) {
	// Your code here
	sck.mu.Lock()
	defer sck.mu.Unlock()
	_, version, err := sck.Get(CURRENT_KEY)
	if err == rpc.OK {
		// 已经初始化过了，不能重复初始化
		return
	}
	configStr := cfg.String()
	configKey := configKey(int(cfg.Num))
	sck.Put(configKey, configStr, version)

	//更新最大版本号
	sck.Put(CURRENT_KEY, configKey, version)
}
```



