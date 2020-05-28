---
title: Condition等待队列
date: 2019-02-18 14:22:31
tags: [并发, AQS]
categories: 并发
---
#### 简介
任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式。         ————《java并发编程的艺术》

- 前提：获取Lock，通过Lock.newCondition()方法获取Condition对象
- 调用方式：condition.await(),condition.signal()
- 支持多个等待队列
- 支持不响应中断

##### 基本结构

![avatar](/pics/等待队列基础结构.png)

#### Condition使用

通过一个有界队列的示例来深入了解Condition的使用方式。
有界队列是一种特殊的队列，当队列为空时，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”。

```java
public class BoundedQueue<T> {
    private Object[] items;
    /**
     * 添加的下标，删除的下标和数组当前数量
     */
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();
    public BoundedQueue(int size) {
        items = new Object[size];
    }

    /**
     * 添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
     * @param t
     * @throws InterruptedException
     */
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[addIndex] = t;
            if (++addIndex == items.length) {
                addIndex = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
     * @return
     * @throws InterruptedException
     */
    @SuppressWarnings("unchecked")
    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            Object x = items[removeIndex];
            if (++removeIndex == items.length) {
                removeIndex = 0;
            }
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```

#### 实现分析
##### 同步队列和等待队列:
![avatar](/pics/同步队列和等待队列.png)
注：一个Lock可以生产多个Condition对象，即多个等待队列。
##### 等待
![avatar](/pics/condition-await.png)
源码：
```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            //将当前线程封装成一个新Node，添加到等待队列尾
            Node node = addConditionWaiter();
            //释放同步状态
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //判断节点是否在同步队列中，不在则继续阻塞
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //被唤醒后重新加入到同步队列开始锁的竞争
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            //清除等待队列中状态不是 Node.CONDITION 的节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。
当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException。
##### 通知
![avatar](/pics/condition-signal.png)
源码：
```java
        public final void signal() {
            //判断是否是以独占方式获取同步状态的。
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            //由于等待队列也是FIFO，所以signal第一个节点。
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                //将头节点从等待队列中移除
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
        
    final boolean transferForSignal(Node node) {
        //设置node的waitStatus：CONDITION->0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        //加入到AQS的同步队列，让节点继续获取锁,设置前置节点状态为SIGNAL
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
transferForSignal(Node node) 方法会将等待队列中的头节点线程安全地移动到同步队列；当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程，被唤醒的线程会从await()方法中的while循环退出，加入到对同步状态的竞争。在获取同步状态之后，会退出await()方法。