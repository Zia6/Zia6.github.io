---
title: raft实现
date: 2025-07-18 19:46:03
tags:
categories: 6.824分布式系统
---
## Lab3 Raft实现
### 3A
3A 主要是不考虑状态机的情况下，实现 Raft Leader选举和心跳机制，确保选举出单一 Leader，如果没有故障就保持 Leader，如果 Leader 故障或者包丢失，则选举新的 Leader
#### ticker实现
我们需要实现超时机制，在一定时间后没有心跳，则我们需要进行选举
#### 选举实现
![原论文中状态转移](状态转移.png)
*状态转移*
![alt text](image.png)
*API 参考*
具体实现参照原论文中状态转移和 API 定义即可，需要注意的点如下
- 在向其他服务器发送 RequestVote 时，需要将锁释放，不然有可能出现多个 Candidater相互要票却无法投票的情况，因为投票也需要加锁，所以会产生死锁
- 在一台服务器决定是否投票时，如果收到的 Term 大于自身 CurrentTerm，需要更新自身 Term 并更新 VoteFor(我没更新 VoteFor 卡了好久)，因为没更新 VoteFor 会导致它进入下一个 Term 但是不能投票的情况出现，进而导致选举不出 Leader
- 发送心跳时也尽量不要锁整个函数，如果出现特殊情况也可能导致死锁，出现两个 Leader 虽然不是同一 Term 但有可能相互要锁导致死锁。
