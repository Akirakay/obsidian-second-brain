## 最简易的分布式锁

命令：`SETNX`  key不存在，则设置成功，否则什么也不做

```Shell
SETNX lockKey uniqueValue
(integer) 1
SETNX lockKey uniqueValue
(integer) 0
```

释放锁：

```bash
DEL lockKey
(integer) 1
```

也可以使用Lua脚本原子执行，释放锁：

```lua
// 释放锁时，先比较锁对应的 value 值是否相等，避免锁的误释放
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

此方式的缺点：如果释放锁过程中出现问题，导致锁没有被正常释放，则其他线程无法再获取到锁
## 锁过期

通过使用`SET`命令：SET key value EX timeout NX 

```bash
127.0.0.1:6379> SET lockKey uniqueValue EX 3 NX
OK
```

注意事项：
- 要保证设置指定 key 的值和过期时间是一个原子操作
- 如果业务执行时间过久，超过timeout时间会导致锁过期释放，丧失锁的意义
- 如果timeout设置的太久，则影响服务质量，降低服务性能
## 锁续期

为解决超时时间设置不合理的问题，`redisson`引入看门狗机制，定时续期：

```java
private void renewExpiration() {
         //......
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                //......
                // 异步续期，基于 Lua 脚本
                CompletionStage<Boolean> future = renewExpirationAsync(threadId);
                future.whenComplete((res, e) -> {
                    if (e != null) {
                        // 无法续期
                        log.error("Can't update lock " + getRawName() + " expiration", e);
                        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                        return;
                    }

                    if (res) {
                        // 递归调用实现续期
                        renewExpiration();
                    } else {
                        // 取消续期
                        cancelExpirationRenewal(null);
                    }
                });
            }
         // 延迟 internalLockLeaseTime/3（默认 10s，也就是 30/3） 再调用
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);

        ee.setTimeout(task);
    }
```

```Java
protected CompletionStage<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            // 判断是否为持锁线程，如果是就执行续期操作，就锁的过期时间设置为 30s（默认）
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return 1; " +
                    "end; " +
                    "return 0;",
            Collections.singletonList(getRawName()),
            internalLockLeaseTime, getLockName(threadId));
}
```
## RedLock

官方分布式锁：
1. 第一步是，客户端获取当前时间（t1）
2. 第二步是，客户端按顺序依次向 N 个 Redis 节点执行加锁操作：
	- 加锁操作使用 SET 命令，带上 NX，EX/PX 选项，以及带上客户端的唯一标识。
	- 如果某个 Redis 节点发生故障了，为了保证在这种情况下，Redlock 算法能够继续运行，我们需要给「加锁操作」设置一个超时时间（不是对「锁」设置超时时间，而是对「加锁操作」设置超时时间），加锁操作的超时时间需要远远地小于锁的过期时间，一般也就是设置为几十毫秒。
3. 第三步是，一旦客户端从超过半数（大于等于 N/2+1）的 Redis 节点上成功获取到了锁，就再次获取当前时间（t2），然后计算计算整个加锁过程的总耗时（t2-t1）。如果 t2-t1 < 锁的过期时间，此时，认为客户端加锁成功，否则认为加锁失败。

核心要求：
- 过半节点上锁成功
- 总耗时不超过锁过期时间
### RedLock分布式问题

分布式面临的三大问题：NPC
N: network delay 网络延迟
P: process pause 进程暂停
C: clock drift 时钟漂移

> [深度剖析：Redis分布式锁到底安全吗？看完这篇文章彻底懂了!](http://kaito-kidd.com/2021/06/08/is-redis-distributed-lock-really-safe/)
