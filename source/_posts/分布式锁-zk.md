---
title: 分布式锁-zk
date: 2019-01-30 18:39:14
tags: [分布式锁, zk]
categories: 分布式锁
---

#### 分布式锁-zk

##### 原理

临时顺序节点  Watch

- 创建父节点，即锁
- 在父节点下创建临时顺序节点
- 通过lock获取其所有子节点列表
- 对上述列表进行排序，从小到大
- 判断本节点是否为第一个节点
    - 是：获取锁
    - 否：监听比本节点小的那个节点
- 监听事件生效，回到第二步进行判断，直至获取锁

##### 加锁

使用zkcli，org.I0Itec.zkclient

```java
//加锁
public void lock() {
    //判断父节点是否存在，不存在就创建
    if (!zkClient.exists(PARENT_LOCK_PATH)) {
        try {
            //创建唯一父节点
            zkClient.createPersistent(PARENT_LOCK_PATH);
        } catch (Exception ignored) {
        }
    }
    //创建父节点下的临时顺序节点
    currentLockPath = zkClient.createEphemeralSequential(PARENT_LOCK_PATH + "/", System.currentTimeMillis());
    //校验是否最小节点
    checkMinNode(currentLockPath);
}
```

`checkMinNode`方法

```java
private boolean checkMinNode(String lockPath) {
    //获取父节点下所有子节点
    List<String> children = zkClient.getChildren(PARENT_LOCK_PATH);
    Collections.sort(children);
    //当前节点的排序下标，index=0则最小，获取锁
    int index = children.indexOf(lockPath.substring(PARENT_LOCK_PATH.length() + 1));
    if (index == 0) {
        log.info(name + "：success");
        if (countDownLatch != null) {
            countDownLatch.countDown();
        }
        return true;
    } else {
        String waitPath = PARENT_LOCK_PATH + "/" + children.get(index - 1);
        //给前一个节点加监听（zk中的watch机制），等待前一个节点释放锁
        waitForLock(waitPath);
        return false;
    }
}
```
`waitForLock`方法

```java
private void waitForLock(String prevNodePath) {
    log.info(name + " current path :" + currentLockPath + "：fail add listener" + " wait path :" + prevNodePath);
    countDownLatch = new CountDownLatch(1);
    //监听前一节点的状态变化
    zkClient.subscribeDataChanges(prevNodePath, new IZkDataListener() {
        @Override
        public void handleDataChange(String s, Object o) throws Exception {
        }

        @Override
        public void handleDataDeleted(String s) throws Exception {
            //之前节点失效后，重新检测本节点是否为最小的节点
            log.info("前一个节点失效");
            checkMinNode(currentLockPath);
        }
    });
    //再次判断节点是否失效
    if (!zkClient.exists(prevNodePath)) {
        return;
    }
    try {
        countDownLatch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    countDownLatch = null;
}
```

##### 解锁

由于zk的监听机制，以及当前节点是唯一的，所以无需做别的操作，只需直接删除即可
```java
//解锁
public void unlock() {
    System.out.println("delete : " + currentLockPath);
    zkClient.delete(currentLockPath);
}
```

[fuyuaaa](https://github.com/fuyuaaa/study-java/tree/master/java-basic/src/main/java/top/fuyuaaa/study/thread/redislock)



