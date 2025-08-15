---
title: 6.824-实现 raft_kv
date: 2025-07-30 10:48:37
tags:
categories: 6.824分布式系统
---


### 整体架构
这里实现的总体架构大致是 server-rsm-raft，server 主要提供对外的 API 服务，rsm 负责将任务进行提交并调用 server 的 do_op()将日志进行应用后返回，raft 负责分布式一致性和分区容错性的保证。
请求处理逻辑如下
- 初始化时 rsm 会开启一个reader后台线程
- client 多线程发送请求
- 一个请求到达server 时，server 调用rsm.submit()对请求进行提交
- rsm 将请求通过 raft.start()提交给 raft，开始睡眠等待
- raft将请求apply，reader 读取到 apply 的请求，执行 do_op()，然后唤醒提交线程
- 调用rsm.submit()的线程将请求返回给 server
- server将结果返回给 client
### RSM
按照上述处理逻辑实现即可，唯一需要注意的点是我们的Submit 和 Reader 都应当是与 op 的格式无关，因此在设计op -> channel的map 时，应该把 key 设计为 Raft Start取得的 Term 和 Index，然后如果 start()后出现分区，我们只要不断轮询检查当前节点的 Term 和 State 是否变化即可
```go
// submit发送后的逻辑
for {
    select {
    case result := <-ch:
        rsm.mu.Lock()
        delete(rsm.waitChans, key)
        rsm.mu.Unlock()
        return result.Err, result.Value
    case <-time.After(10 * time.Millisecond):
        nowTerm, nowIsLeader := rsm.Raft().GetState()
        if nowTerm != term || nowIsLeader != isLeader {
            return rpc.ErrWrongLeader, nil
        }
        continue
    }
}
```

### 实现kv
分别实现RSM 要求实现的 StateMachine 接口也就是 DoOp(),SnapShot(),ReStore()三个接口即可
#### DoOp
在 Raft 成功 Apply 一条日志之后我们的 RSM 会通过 reader()线程调用 server 的 doOp接口。
```go
func (rsm *RSM) reader() {
	lastSnapshotIndex := 0
    //***
	for msg := range rsm.applyCh {
		result := rsm.sm.DoOp(msg.Command)
        //***
	}
}

func (kv *KVServer) DoOp(req any) any {
	// Your code here

	op := req.(rsm.Op)
	switch op.Type {
	case PUT:
		return kv.DoPutOp(op.Args.(rpc.PutArgs))
	case GET:
		return kv.DoGetOp(op.Args.(rpc.GetArgs))
	}
	return nil
}
```
#### SnapShot
在 RSM 检测到当前日志已经足够长的时候，调用 SnapShot 接口获取当前快照并调用 Raft 提供的 SnapShot(),这里主要是进行日志的截断
```go
func (kv *KVServer) Snapshot() []byte {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(kv.data)
	return w.Bytes()
}

```
#### ReStore
当 Raft 启动时，或者 follower 跟不上 leader 时，我们需要安装快照，调用 Restore()函数
```go
func (kv *KVServer) Restore(data []byte) {
	if len(data) == 0 {
		return
	}

	kv.mu.Lock()
	defer kv.mu.Unlock()

	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)

	var restoredData map[string]struct {
		value   string
		version rpc.Tversion
	}

	if d.Decode(&restoredData) == nil {
		kv.data = restoredData
	}
}
```
