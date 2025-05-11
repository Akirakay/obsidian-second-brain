## zk分布式锁实现原理
![zk分布式锁实现原理](https://s9.51cto.com/oss/202301/12/38d2e2933a9705719f4271ff75c1190e2fe28f.png)

核心要求：
- 多线程之间顺序创建**临时节点**，通过判断该节点是否是最小顺序来获取锁

> [Java 基于 ZooKeeper 实现分布式锁需要注意什么](https://www.lenshood.dev/2020/03/25/zk-distributed-lock/)
