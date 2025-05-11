Ref：[Raft算法详解](https://zhuanlan.zhihu.com/p/32052223)
## 简介

### 节点类型

Raft将系统中的角色分为`领导者(Leader)`、`跟从者(Follower)`和`候选人(Candidate)`:

- `Leader`: 接受客户端请求，并向Follower同步请求日志，当日志同步到大多数节点上后告诉Follower提交日志。
- `Follower`: 接受并持久化Leader同步的日志，在Leader告之日志可以提交之后，提交日志。
- `Candidate`: Leader选举过程中的临时角色。

![raft-status](https://oss.javaguide.cn/github/javaguide/paxos-server-state.png)

Follower只响应其他服务器的请求。如果Follower超时没有收到Leader的消息，它会成为一个Candidate并且开始一次Leader选举。收到大多数服务器投票的Candidate会成为新的Leader。Leader在宕机之前会一直保持Leader的状态。

### 任期

![term](https://oss.javaguide.cn/github/javaguide/paxos-term.png)

Raft算法将时间分为一个个的任期(term)，每一个term的开始都是Leader选举。在成功选举Leader之后，Leader会在整个term内管理整个集群。如果Leader选举失败，该term就会因为没有Leader而结束。

### Leader选举

通过心跳机制触发选举Leader

正常服务启动时，状态为Follower，接收Leader发来的心跳；超时未接收到heartbeat，则Follower在等待一段时间之后，触发一次新的Leader选举：
- Follower将term+1，状态置为Candidate
- vote itself and 向其他节点发送投票请求：
	1. 超过半数投票成功，成为Leader
	2. 收到Leader信息，已存在Leader
	3. 没有Candidate超过半数投票成功，等待超时下一次选举
- Leader 继续发送心跳，维持统治，否则触发Leader选举

### 日志同步

Raft中，只有唯一的Leader可以接受命令，并同步给其他Follower

日志同步过程：
- client 提交数据到Leader，Leader 封装数据形成Log Entries`<index,term,cmd>`，并发送给Follower
- 过半Follower 接收该entry后向Leader返回同意
- Leader 向 Client 确定此数据的提交
- Leader 向所有Follower发送提交数据请求

![log-replica](https://pic1.zhimg.com/v2-7cdaa12c6f34b1e92ef86b99c3bdcf32_1440w.jpg)

Raft对日志的保证：
- 两个日志的entry的index和term相同，则cmd相同
- 两个日志的entry的index和term相同，则之前的条目也相同

Leader通过强制Followers复制它的日志来处理日志的不一致，Followers上的不一致的日志会被Leader的日志覆盖：
- Leader需要找到Followers同它的日志一致的地方，然后覆盖Followers在该位置之后的条目。
- Leader会从后往前试，每次AppendEntries失败后尝试前一个日志条目，直到成功找到每个Follower的日志一致位点，然后向后逐条覆盖Followers在该位置之后的条目。
### 安全性

1. 拥有最新提交日志的Follower才能成为Leader
其他节点收到投票请求会对比请求中的`<term,index,cmd>`与自身日志中最新的log entry，如果自己更新则拒绝投票

