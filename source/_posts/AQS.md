---
title: AQS
date: 2019-02-13 14:22:25
tags: [并发]
categories: 并发
---

### 简介
AQS，即AbstractQueuedSynchronizer（抽象队列同步器）的缩写，是并发编程中实现同步器的一个框架。
AQS是一个抽象类，主是是以继承的方式使用。

它维护了一个`volatile int state`（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）。

AQS主要结构：
- 两种队列
    - `Sync Queue`：并发安全的CLH队列。
    - `Condition Queue`：独占模式可能会使用的，等待队列（普通的list）。
        - 在线程获取锁之后，通过Lock的newCondition()方法获取；
        - 调用Condition.await()方法会添加线程到等待队列；调用Condition.signal() OR signalAll()方法会唤醒。
        - 一个Lock可以获取很多个等待队列。
- `state`：`volatile int`类型，表示同步状态。
    - state = 0，表示可以获取锁，并且在获取锁之后会将state设置为1。
    - state = 1，表示当前锁已经被其他线程获取。
- `ConditionObject`：用于独占模式, 是等待队列的具体实现。
- `Node`节点类：用于存放获取线程的节点，被同步队列和等待队列所共用。

队列中的节点表示被阻塞的线程，队列节点元素有4种类型， 每种类型表示线程被阻塞的原因：
`CANCELLED` : 1, 表示该线程是因为超时或者中断原因而被放到队列中
`SIGNAL` : -1, 表示该节点的后继节点的线程在等待，当前节点释放同步状态或者被取消，将通知后继节点
`CONDITION` : -2, 节点在等待队列中，当其他线程对该Condition调用signal()方法后，该节点会从等待队列转移到同步队列中
`PROPAGATE` ： -3, 表示在共享模式下，当前节点执行释放release操作后，当前结点需要传播通知给后面所有节点
`INITIAL` ：0, 初始状态
由于一个共享资源同一时间只能由一条线程持有，也可以被多个线程持有，因此AQS中存在两种模式，如下：
- 独占模式
独占模式表示共享状态值state每次只能由一条线程持有，其他线程如果需要获取，则需要阻塞，如JUC中的ReentrantLock。
- 共享模式
共享模式表示共享状态值state每次可以由多个线程持有，如JUC中的CountDownLatch。

### 内部类-Node
![avatal](/pics/Node.png)
源码：
```java
/**
 * AQS.Node类 源码
 * @author: fuyuaaa
 * @creat: 2019-02-14 16:36
 */
public class Node {
    /**
     * 共享模式 
     */
    static final Node SHARED = new Node();

    /**
     * 独占模式 
     */
    static final Node EXCLUSIVE = null;

    /**
     * 场景：当该线程等待超时或者被中断，需要从同步队列中取消等待，则该线程被置1，即被取消（这里该线程在取消之前是等待状态）。
     * 节点进入了取消状态则不再变化。
     */
    static final int CANCELLED = 1;

    /**
     * 标识当前节点的后继节点需要唤醒，通常是在 独占模式下使用
     * 场景：后继的节点处于等待状态，当前节点的线程如果释放了同步状态或者被取消（当前节点状态置为-1），
     *      将会通知后继节点，使后继节点的线程得以运行
     */
    static final int SIGNAL = -1;
    
    /**
     * 场景：节点处于等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()方法后，
     *      该节点从等待队列中转移到同步队列中，加入到对同步状态的获取中
     */
    static final int CONDITION = -2;
    
    /**
     * 场景：表示下一次的共享状态会被无条件的传播下去
     */
    static final int PROPAGATE = -3;
    
    /**
     * 该字段只取上面四种值 
     */
    volatile int waitStatus;
    
    /**
     * 节点在 Sync Queue 里面时的前继节点(主要来进行 skip CANCELLED 的节点)
     */
    volatile Node prev;
    
    /**
     * Node 在 Sync Queue 里面的后继节点, 主要是在release lock 时进行后继节点的唤醒
     * 而后继节点在前继节点上打上 SIGNAL 标识, 来提醒他 release lock 时需要唤醒
     */
    volatile Node next;
    
    /**
     * 获取到同步状态的线程 
     */
    volatile Thread thread;

    /**
     * 作用分成两种:
     *  1. 在 Sync Queue 里面, nextWaiter用来判断节点是 共享模式, 还是独占模式
     *  2. 在 Condition queue 里面, 节点主要是链接且后继节点 (Condition queue是一个单向的, 不支持并发的 list)
     */
    Node nextWaiter;

    /**
     * 判断当前节点是否是共享模式 
     */
    final boolean isShared() {
        return this.nextWaiter == SHARED;
    }

    /**
     * 获取前继节点 
     */
    final Node predecessor() throws NullPointerException {
        Node var1 = this.prev;
        if (var1 == null) {
            throw new NullPointerException();
        } else {
            return var1;
        }
    }

    Node() {
    }

    /**
     * 用于Sync队列里
     */
    Node(Thread var1, Node var2) {      // Used by addWaiter
        this.nextWaiter = var2;
        this.thread = var1;
    }

    /**
     * 用于Condition队列里
     */
    Node(Thread var1, int var2) {       // Used by Condition
        this.waitStatus = var2;
        this.thread = var1;
    }
}
```

### 同步队列
![avatar](/pics/同步队列.png)
同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。
该队列也是一个CLH队列，但是使用了变种的CLH队列锁。
1. 原来的CLH使用locked自旋，AQS的CLH在每一个Node中使用了一个状态字段来控制阻塞，而不是自旋。
2. 为了可以处理timeout和cancle，每个node中有一个prev，如果前驱被cancle，node可以向前移动。
3. head使用了傀儡节点。

### 独占锁
#### 获取锁
源码：
```java
    public final void acquire(int arg) {
        /**
         * 1. 尝试获取锁, tryAcquire(arg)。该方法需要使用者去继承实现的。
         * 2. 没有获取到锁，进行入队操作 addWaiter()，在方法中会将当前线程封装成 Node，添加到同步队列末尾。
         * 3. 在acquireQueued()中进行自旋，直到条件满足获取同步状态，从队列退出并结束自旋。
         * PS 如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现
         */
        if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(AbstractQueuedSynchronizer.Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
`addWaiter`方法的具体实现
```java
    private Node addWaiter(Node mode) {
        //封装node，将当前线程封装成Node类型
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        //pred != null说明队列中已经有节点，尝试使用cas将节点添加到末尾。
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果是第一次插入，或者cas失败，则调用该方法入队
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        //自旋
        for (;;) {
            Node t = tail;
            //如果队列为空，即没有头结点，所以需要初始化，设置头结点，并且将tail指向head。
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                //添加节点到队尾
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
注：head和tail为AQS中的变量，head指向头部，但是head是空节点，不储存信息；tail则是队列的队尾。

在添加节点成功之后，会进行`acquireQueued`方法。
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //用来查看在等待时是否被中断过。如果true，acquire()中会调用 selfInterrupt() 中断线程。
            boolean interrupted = false;
            for (;;) {
                //获取当前节点的前继节点
                final Node p = node.predecessor();
                /** 判断前继节点是否是head节点，前继节点是head, 存在两种情况 
                 * (1) 前继节点现在占用 lock ;
                 * (2) 前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquire尝试获取一下。
                 */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                /**
                 * 1. shouldParkAfterFailedAcquire 方法判断是否需要挂起，如果 p（前驱）的状态为 SIGNAL ，则返回 true 。
                 * 2. parkAndCheckInterrupt，挂起并且返回 boolean 值（是否被中断过）。如果 true，将 interrupted 设置为 true
                 */
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //在获取过程中出错，会将node节点清除；清除时先给节点打上CANCELLED标志（node.waitStatus = Node.CANCELLED），再进行清除。
            if (failed)
                cancelAcquire(node);
        }
    }
```
获取锁流程图：
![avatar](/pics/AQS获取锁流程图.png)

#### 释放锁
release(int)
```java
public final boolean release(int arg) {
        //调用子类的实现，释放锁，可能存在重入的情况。存在重入锁时未释放完锁也返回false。
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //唤醒后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
unparkSuccessor(h)
```java
    /**
     * node: 当前节点，即已经执行完的
     * s: 下一个需要唤醒的节点
     */
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        //将当前节点的状态置为0，即清除该节点的标识
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        //如果s为空，或者状态是已取消，则向后查找
        if (s == null || s.waitStatus > 0) {
            s = null;
            //这里的向后查找，是从最后一个节点往前查找。
            for (Node t = tail; t != null && t != node; t = t.prev)
                //跳过取消的节点
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒线程
            LockSupport.unpark(s.thread);
    }
```
AQS使用Demo：[传送门](https://github.com/fuyuaaa/study-java/blob/master/java-basic/src/main/java/top/fuyuaaa/study/thread/aqssources/Mutex.java)
参考：[并发Lock之AQS（AbstractQueuedSynchronizer）详解](https://juejin.im/post/5a4a4530518825697078553e#heading-1)