---
title: Redisson源码
date: 2019-01-28 20:44:51
tags: [redis, 分布式锁]
categories: redis
---
#### Redisson源码

##### 加锁

这里主要看一下`tryLock(long waitTime, long leaseTime, TimeUnit unit)`这个方法

```java
/**
 * @param waitTime 等待获取锁的时间
 * @param leaseTime 锁释放的时间
 * @param unit 时间单位
 * @return 如果返回true 锁被成功获取
 */
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    final long threadId = Thread.currentThread().getId();
    //请求获取锁，返回null（获取锁成功） or 锁的剩余时间
    Long ttl = tryAcquire(leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return true;
    }

    time -= (System.currentTimeMillis() - current);
    //超时，获取锁失败
    if (time <= 0) {
        acquireFailed(threadId);
        return false;
    }

    current = System.currentTimeMillis();
    //订阅监听redis消息
    final RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    //阻塞等待subscribe的future的结果对象，如果subscribe调用时间超过了time，则获取锁失败，返回false，并且取消订阅
    if (!await(subscribeFuture, time, TimeUnit.MILLISECONDS)) {
        if (!subscribeFuture.cancel(false)) {
            subscribeFuture.addListener(new FutureListener<RedissonLockEntry>() {
                @Override
                public void operationComplete(Future<RedissonLockEntry> future) throws Exception {
                    if (subscribeFuture.isSuccess()) {
                        unsubscribe(subscribeFuture, threadId);
                    }
                }
            });
        }
        acquireFailed(threadId);
        return false;
    }

    try {
        time -= (System.currentTimeMillis() - current);
        if (time <= 0) {
            acquireFailed(threadId);
            return false;
        }
        //循环获取锁，直至超时OR获取锁成功
        while (true) {
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                return true;
            }

            time -= (System.currentTimeMillis() - currentTime);
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }

            // waiting for message
            currentTime = System.currentTimeMillis();
            if (ttl >= 0 && ttl < time) {
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }
            //更新等待的时间
            time -= (System.currentTimeMillis() - currentTime);
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }
        }
    } finally {
        //取消订阅
        unsubscribe(subscribeFuture, threadId);
    }
//        return get(tryLockAsync(waitTime, leaseTime, unit));
}
```

上述过程中获取锁都是通过`tryAcquire(long leaseTime, TimeUnit unit, long threadId)`这个方法来的
```java
    private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(leaseTime, unit, threadId));
    }
```
接着进入
```java
//异步获取
private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, final long threadId) {
    if (leaseTime != -1) {
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    //tryLockInnerAsync方法中异步执行Lua脚本
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    ttlRemainingFuture.addListener(new FutureListener<Long>() {
        @Override
        public void operationComplete(Future<Long> future) throws Exception {
            if (!future.isSuccess()) {
                return;
            }

            Long ttlRemaining = future.getNow();
            // lock acquired
            if (ttlRemaining == null) {
                //异步调度延长锁的过期时间
                scheduleExpirationRenewal(threadId);
            }
        }
    });
    return ttlRemainingFuture;
}
```

在`scheduleExpirationRenewal(threadId)`中又有重复调用这个方法，这个方法会推迟`leaseTime/3`执行(使用Timeout)。

##### 解锁

```java
    @Override
    public void unlock() {
        Boolean opStatus = get(unlockInnerAsync(Thread.currentThread().getId()));
        //...省略
    }
```
在`unlock()`方法中调用异步方法`unlockInnerAsync`执行Lua脚本。

##### Lua脚本

###### 加锁

```java
//检查锁是否存在，不存在则添加一个hash，field的默认值value设为1
//filed即hash中的key
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hset', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
//如果锁重入，判断key和field是否一致，一致则对应的value+1
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
	//重新设置超时时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;
//返回剩余的过期时间
return redis.call('pttl', KEYS[1]);,
```
```java
KEYS[1] -> lock的key，对应redis中hash的key
ARGV[1] -> 过期时间
ARGV[2] -> lock的value，一般为UUID+ThreadId，对应hash的field
hash的field对应的值 -> 加锁次数
redis.call('hincrby', KEYS[1], ARGV[2], 1) -> hash中field对应的值加1,即实现了可重入锁

```

###### 解锁

```java
//如果不存在，直接发布redis消息
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1;
end;
//hash的key和field不完全匹配，说明并没有持有该锁，无法解锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;
end;
//value值减1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
//如果 counter>0 说明存在重入，不能直接删除key；若 <0 ，删除并发布redis消息说明已解锁。
if (counter > 0) then
    redis.call('pexpire', KEYS[1], ARGV[2]);
    return 0;
else
    redis.call('del', KEYS[1]);
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1;
end;
return nil;
```

###### 刷新过期时间

```java
//如果key和field都一致，刷新过期时间
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
end;
return 0;
```

[fuyuaaa](https://github.com/fuyuaaa/study-java/tree/master/java-basic/src/main/java/top/fuyuaaa/study/thread/redislock)