## 简介

分布式共识算法：**共识算法**的作用是让分布式系统中的多个节点之间对某个提案（Proposal）达成一致的看法

两个部分：
- Basic Paxos算法：多节点之间如何就某个值(提案 Value)达成共识
- Multi-Paxos思想：多节点之间对一系列值如何达成共识，核心就是多次Basic Paxos

## Basic Paxos
### 角色

- 提议者（Proposers）：提出提案，提供提案号和对应值
- 决策者（Acceptors）：参与决策，回应提案。若提案被多数决策者认同，则批准该提案
- 最终决策学习者（Learner）：不参与决策，从Proposers/Acceptors学习最新达成一致的提案(Value)。

> 简单理解为人大代表(Proposer)在人大向其它代表(Acceptors)提案，通过后让老百姓(Learner)落实。

### 三个阶段

1. prepare阶段
	Proposer向Acceptors发出prepare请求，携带提案编号Proposal ID，
	acceptor对该请求做出promise承诺：
	- 不再接收 <= 该请求的prepare
	- 不再接收 < 当前Proposal ID 的请求
	- 回应accepted的最大Proposal ID 的请求
	
2. accept阶段
	Proposer收到多数Acceptors承诺的Promise后，向Acceptors发出Propose请求，Acceptors针对收到的Propose请求进行Accept处理。
	- `Propose`: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
	- `Accept`: Acceptor收到Propose请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。
    
3. learn阶段
	- 获取一个Proposal ID n，为了保证Proposal ID唯一，可采用时间戳+Server ID生成；
	- Proposer向所有Acceptors广播Prepare(n)请求；
	- Acceptor比较n和minProposal，如果n>minProposal，minProposal=n，并且将 acceptedProposal 和 acceptedValue 返回；
	- Proposer接收到过半数回复后，如果发现有acceptedValue返回，将所有回复中acceptedProposal最大的acceptedValue作为本次提案的value，否则可以任意决定本次提案的value；
	- 到这里可以进入第二阶段，广播Accept (n,value) 到所有节点；
	- Acceptor比较n和minProposal，如果n>=minProposal，则acceptedProposal=minProposal=n，acceptedValue=value，本地持久化后，返回；否则，返回minProposal。
	- 提议者接收到过半数请求后，如果发现有**返回值result >n**，表示有更新的提议，跳转到1；否则value达成一致。

## Multi-Paxos
Multi-Paxos基于Basic Paxos做了两点改进:

- 针对每一个要确定的值，运行一次Paxos算法实例(Instance)，形成决议。每一个Paxos实例使用唯一的Instance ID标识。
- 在所有Proposers中选举一个Leader，由Leader唯一地提交Proposal给Acceptors进行表决。这样没有Proposer竞争，解决了活锁问题。在系统中仅有一个Leader进行Value提交的情况下，Prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

