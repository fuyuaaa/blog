---
title: AQS
date: 2019-02-13 14:22:25
tags: [并发]
categories: 并发
---

# AbstractQueuedSynchronizer

## 1. 简介

AbstractQueuedSynchronizer，抽象队列同步器，位于java.util.concurrent.locks包下，主要用来实现锁或者其他同步框架。JUC包中的大多数同步器以及锁都是使用该同步器来实现的。

- 提供了int state来代表同步状态，并提供了getState、setState、compareAndSetState这三个方法对state进行操作。
- 提供了一个FIFO队列，作为同步队列，来管理多线程的排队以及等待通知工作(即，未获取到同步状态的线程会被加入队列排队，阻塞并等待前驱节点唤醒)。



## 2. 使用

AQS是一个抽象类，主要是以继承的方式使用(建议内部类)。自定义的同步器在实现时只需实现state的获取和释放(即实现模板方法)即可，而不用去关心线程等待队列，AQS已经实现了这部分功能。
AQS的使用只需要使用getState、setState、compareAndSetState这三个方法来实现定义的模板方法即可。模板方法如下所示：

```java
 * tryAcquire			独占方式-获取同步资源
 * tryRelease			独占方式-释放同步资源
 * tryAcquireShared	    共享方式-获取同步资源
 * tryReleaseShared	    共享方式-释放同步资源
 * isHeldExclusively 	当前线程是否在独占资源
```


### 2.1 ReenTrantLock使用AQS

接下来，看一下ReenTrantLock是如何使用AQS的。可以看到，ReenTrantLock使用Sync内部类继承了AQS并重写了tryRelease方法，Sync的子类NonfairSync(非公平)和FairSync(公平)分别重写了tryAcquire方法。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();

    //非公平模式
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //c==0表示当前同步状态未被获取
        if (c == 0) {
            //cas获取同步状态
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    //释放同步状态(不一定完全释放,可能部分,因为存在重入)，不区分是否公平
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        //校验是否当前线程获取独占的
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //释放完了
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
	//省略非关键代码
}
```


如NonfairSync和FairSync中的lock方法，在重写完模板方法之后，就可以直接使用AQS的acquire方法来进行同步状态的获取，即加锁。

```java
static final class NonfairSync extends Sync {

    /**
     * 非公平锁
     * 在lock的时候尝试cas一下，如果成功就获取锁。
     * ReentrantLock的非公平就体现在这。
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    /**
     * 重写AQS的tryAcquire模板方法
     */
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * 重写AQS的tryAcquire模板方法
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //比非公平锁多了这个判断，用来判断队列里是否存在线程，不存在则尝试cas
            //!hasQueuedPredecessors()
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```


### 2.2 acquire方法

由上可知，调用acquire方法就可以获取锁，我们来看一下acquire方法做了什么：
与上文描述的逻辑类似，会先尝试获取同步状态，获取失败则会将当前线程加入同步队列，挂起并等待唤醒。

```java
public final void acquire(int arg) {
    //调用子类实现的tryAcquire方法
    //tryAcquire返回false -> 构建Node节点加入队列 -> 挂起
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

## 3. 详解

上文介绍了AQS及其使用，接下来具体看下AQS是怎么实现的。

### 3.1 数据结构及成员变量

#### 3.1.2 Node类

同步等待队列的节点类，当线程获取同步状态失败时，会将线程构建成一个Node类，并添加到同步队列的尾部。
先来看一下Node类中的几个成员变量：

```java
//当前节点在同步队列中的状态，初始值为0
volatile int waitStatus;
//前驱节点
volatile Node prev;
//后继节点
volatile Node next;
//当前线程
volatile Thread thread;
//指向下一个处于CONDITION状态的节点
Node nextWaiter;
```

waitStatus的枚举值：

```java
//同步状态的请求已被取消
static final int CANCELLED =  1;
//后继节点需要唤醒, 在Node入队时, 会将前一个节点的状态置为SIGNAL
static final int SIGNAL    = -1;
//当前节点在Condition队列里
static final int CONDITION = -2;
//传播 TODO
static final int PROPAGATE = -3;
```


#### 3.1.3 成员变量

```java
//等待队列的头
private transient volatile Node head;
//等待队列的队尾
private transient volatile Node tail;
//同步状态
private volatile int state;
```

AQS中FIFO队列是有Node类型的head和tail这两个成员变量来实现的，数据结构如下所示。
<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/aqs.png" alt="aqs" style="zoom:25%;" />


### 3.2 独占锁实现

独占锁的实现主要涉及到以下方法：

```java
acquire 独占模式获取锁
addWaiter 将当前线程加入等待队列
enq 入队的具体方法
acquireQueued 在队列中的节点, 通过该方法获取锁
shouldParkAfterFailedAcquire 判断在获取锁失败时, 线程需不需要被挂起
cancelAcquire 取消获取锁
```

#### 3.2.1 acquire方法

acquire方法主要做的事情：

1. 调用子类实现的tryAcquire方法
1. 如果获取同步状态失败则将当前线程构建成Node节点并加入等待队列
1. 在等待队列中，会检测当前节点是否为Head节点的后继节点，并尝试获取锁；如果获取失败则会挂起。

```java
/**
 * 独占模式-获取锁
 */
public final void acquire(int arg) {
    //调用子类实现的tryAcquire方法
    //tryAcquire返回false -> 构建Node节点加入队列 -> 挂起
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```


#### 3.2.2 addWaiter方法&enq方法

在tryAcquire方法返回false之后，会调用addWaiter方法往等待队列队尾添加一个节点。

```java
/**
 * 将当前线程构建成Node并加入等待队列，当前状态waitStatus=0
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 快速尝试, 即尝试直接对队列尾进行CAS操作
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //如果队列为空或者快速尝试失败, 则会进入enq方法
    enq(node);
    return node;
}

/**
 * 入队
 * 循环CAS操作, 将当前节点入队
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //头结点不存在，则进行初始化
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //将当前入队节点的前驱节点设置为原先的tail节点, 并更新尾结点为当前节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                //当原先tail节点的后继节点设置为当前入队节点
                t.next = node;
                return t;
            }
        }
    }
}
```

在addWaiter和enq方法中，有一个代码需要注意，即第10~14行和第34~39行。
添加尾结点的步骤分为三步：

1. 设置当前节点的prev为之前的tail节点
1. 将tail指针指向当前节点
1. 将之前的tail节点的next指向当前节点

如下图所示：
<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/aqs-enq.png" alt="aqs-enq" style="zoom:25%;" />
试想一下，如果调换一下执行顺序会咋样：

1. 如果先cas tail节点，再设置prev和next指针，则有可能出现某一时刻，tail节点的的prev和next都为null，队列就不完整了。
1. 如果先设置next指针，再cas tail节点，再设置prev，不合理，cas tail都没成功，不应先操作尾结点的next指针。

综上，将设置prev指针这一步放在if之前较为合理。


#### 3.2.3 acquireQueued方法&shouldParkAfterFailedAcquire方法

在节点被加入到等待队列之后，会通过acquireQueued方法获取锁，主要有以下步骤：

1. 校验当前节点的前驱节点是否为head，如果是就尝试获取锁，获取成功则返回。
1. 如果当前节点的前驱节点不是head，或者获取锁失败了，则会进入shouldParkAfterFailedAcquire方法。

shouldParkAfterFailedAcquire方法的主要逻辑是：将当前节点的前驱节点的状态设置为SIGNAL，如果前驱节点为CANCELLED状态则跳过。当该方法返回true时，则会进入parkAndCheckInterrupt方法并挂起当前线程等待唤醒(此时当前节点的前驱节点状态为SIGNAL)。

```java
/**
 * 在队列中的节点, 通过该方法获取锁
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        //中断标志
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //检测当前节点的前驱节点是否为head(标明当前节点有资格获取锁), 如果为head则尝试获取锁
            //获取成功, 则将head置为当前节点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //获取锁失败, 判断是否应该挂起
            //shouldParkAfterFailedAcquire判断当前线程是否该挂起
            //parkAndCheckInterrupt挂起线程, 唤醒之后会返回是否中断
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果前驱节点为SIGNAL状态, 则前驱节点释放锁的时候会唤醒后继节点, 所以当前节点可以阻塞
    if (ws == Node.SIGNAL)
        return true;
    // ws>0 means CANCELLED -> 前驱节点为CANCELLED
    if (ws > 0) {
        // 从当前节点的前驱节点往前找, 找到第一个状态不为CANCELLED的, 并挂上去
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    }
    // 前驱节点状态不为CANCELLED, 则把它设置成SIGNAL
    else {
        
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/aqs-acquireQueued.png" alt="aqs-acquireQueued" style="zoom:25%;" />

#### 3.2.4 cancelAcquire方法

cancelAcquire方法用来取消某个节点获取锁。该方法为private方法，目前只在获取锁发生异常的finally代码中看到调用。

```java
/**
 * 取消获取锁
 */
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;

    // 往前遍历, 跳过已经取消的节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // 记录下来, 后续CAS使用
    Node predNext = pred.next;

    // 将当前节点的等待状态设置为已取消
    node.waitStatus = Node.CANCELLED;

    // 第一种情况: 如果当前节点是tail, 并且设置tail为当前节点的前驱结点成功
    if (node == tail && compareAndSetTail(node, pred)) {
        // 将前驱节点的next设置为null
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
		// 第二种情况:       
        // 三个条件
        // 1. 前驱节点不为head节点
        // 2. 前驱节点状态为SIGNAL 或者 CAS其状态为SIGNAL成功
        // 3. 前驱节点的线程不为null
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            // 如果当前节点的后续节点不为空, 并且其等待状态不为CANCELLED, 将前驱节点和后继节点连起来
            // ps: 这里只操作了next指针, 并没有把next节点的prev指针指向pred节点,
            // 这一步是是在哪做的呢? -> 别的线程在调用cancelAcquire或者shouldParkAfterFailedAcquire时会通过prev指针跳过CANCELLED状态的节点
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        }
        // 第三种情况: 如果node是head的后继节点, 直接唤醒后继节点
        else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

需要注意的是cancelAcquire对于一下三种情况的不同处理：

1. 当前节点是tail
1. 当前节点不是tail，并且也不是head的后继节点
1. 当前节点是head的后继节点



第一种情况：当前节点是tail，并且设置tail为当前节点的前驱结点成功(如果设置失败，说明别的线程改了tail了)
该情况直接将当前节点的前驱节点设置为tail，并且将该前驱结点的next设置为null。


第二种情况：当前节点不是tail，也不是head的后继节点
该情况会将当前节点的前驱节点和后继节点连起来。但是上面第二种情况的代码，只将前驱节点的next指向了后继节点，并没有对后继节点的prev进行设置。（这里的前驱节点是已经跳过了CANCELLED状态的前驱节点）
针对后继节点的prev设置，主要是别的线程在调用cancelAcquire或者shouldParkAfterFailedAcquire时会通过prev指针跳过CANCELLED状态的节点。


第三种情况：当前节点是head的后继节点
该情况会直接调用unparkSuccessor方法唤醒后继节点。


#### 3.2.5 unparkSuccessor方法

unparkSuccessor方法唤醒当前节点的后续节点。如果当前节点的后续节点为null或者CANCELLED状态，则会从tail节点往前遍历，找到最靠前的状态不为CANCELLED的节点。
这里为什么要从后往前遍历？
从3.2.2可知，addWaiter与enq方法中的入队操作并不是一个原子操作。所以可能存在这样一种情况：
**当前节点是tail节点，但是此刻有新节点入队，并且刚好执行到了3.2.2中描述的第二步(即cas tail完成，此刻当前节点的next指针还是为null)，所以如果从前往后遍历，就可能会漏掉最后这个新入队的节点。**
<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/aqs-unparkSuccessor-从后往前.png" alt="aqs-unparkSuccessor-从后往前" style="zoom:25%;" />

```java
/**
 * 唤醒当前节点的后继节点
 */
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    // 将当前节点的状态设置为0, ?
    if (ws < 0) {
        compareAndSetWaitStatus(node, ws, 0);
    }

    // 获取当前节点的后继节点, 准备唤醒
    Node s = node.next;
    // 如果后继节点为空, 或者是CANCELLED状态
    // 这里s==null并不代表s就是tail, 此时可能有新节点入队, 并处在 完成了cas但没有更新next指针 这一时刻
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从tail开始向前遍历, 找到状态不为CANCELLED的节点
        // 为啥从后往前 -> 在addWaiter与enq方法中, 入队并不是一个原子操作(分为三步1设置prev指针,2设置tail指针,3设置next指针)。
        //              所以从前往后遍历, 可能会漏掉节点, 因为此时next指针可能为空。
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

#### 3.2.6 线程被唤醒之后

线程被唤醒之后，会从parkAndCheckInterrupt方法被唤醒，并返回是否中断，然后会回到acquireQueued方法的循环中。
被唤醒的节点会校验自己的前驱节点是否为head，如果为head则会尝试获取锁，即调用tryAcquire。如果不是head，则会进入shouldParkAfterFailedAcquire方法，跳过状态为CANCELLED的节点，然后继续进入循环，此时前驱节点就是head了。


#### 3.2.7 独占锁释放


```java
    /**
     * 释放同步状态
     */
    public final boolean release(int arg) {
        // 调用子类实现的模板方法释放同步状态
        if (tryRelease(arg)) {
            
            /**
             * 此时head可能的情况
             * 1. null, 此时无竞争, head没有初始化
             * 2. head是当前线程的节点
             * 3. 在tryRelease之后, 别的线程的节点获取到了锁, 通过setHead方法设置(acquireQueued方法里)
             */
            Node h = head;
            // 唤醒后继节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```


### 3.3 共享锁实现

共享锁的实现主要涉及到以下方法：

```java
acquireShared 共享模式-获取锁
doAcquireShared 
addWaiter 将当前线程加入等待队列
setHeadAndPropagate
shouldParkAfterFailedAcquire 判断在获取锁失败时, 线程需不需要被挂起
addWaiter方法和shouldParkAfterFailedAcquire方法上文已经有详细的注释了，不做过多分析。
```

#### 3.3.1 acquireShared 方法

acquireShared方法主要做的事情：

1. 调用子类实现的tryAcquireShared方法，如果返回结果>=0说明获取共享锁成功
1. 获取失败则会调用doAcquireShared方法进入等待队列

这里解释一下tryAcquireShared方法返回值的意义：

- 返回负数：获取共享锁失败
- 返回零：获取成功，但是没有空闲资源了
- 返回正数：获取成功，还有空闲资源，可以唤醒

```java
/**
 * 共享模式-获取锁
 */
public final void acquireShared(int arg) {
    // 调用子类实现的模板方法, 返回<0表示获取锁失败, 则需要调用doAcquireShared方法进行等待队列的处理
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

#### 3.3.2 doAcquireShared方法

doAcquireShared方法主要做的事情：

1. 将当前节点加入等待队列
1. 尝试获取锁，判断是否应该挂起（该逻辑与独占锁的acquireQueued方法类似）

与独占锁不同的地方主要在于，获取共享锁成功之后会调用setHeadAndPropagate唤醒后继节点。

```java
private void doAcquireShared(int arg) {
    // 将当前线程加入等待队列, SHARED类型的节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (; ; ) {
            final Node p = node.predecessor();
            // 前驱节点为head, 尝试获取锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 获取成功
                if (r >= 0) {
                    // 设置头结点, 并唤醒后继节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 同独占锁
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            // 同独占锁
            cancelAcquire(node);
    }
}
```

#### 3.3.3 setHeadAndPropagate方法和doReleaseShared方法

setHeadAndPropagate方法主要做的事情：

1. 将当前线程节点设置为head
1. 判断是否需要唤醒后继节点并调用doReleaseShared方法



如果需要唤醒后继节点，则会调用doReleaseShared方法，该方法主要在两个地方调用：

1. 有线程释放共享锁成功，通过releaseShared方法调用doReleaseShared方法
1. head的后继线程获取共享锁成功（阻塞前获取到，或阻塞后被唤醒获取成功），通过setHeadAndPropagate方法调用doReleaseShared方法。

方法逻辑：如果此刻head的状态为SIGNAL，则唤醒后继节点。如果此刻head的状态为0，表示后继节点暂时不需要唤醒，将其设置为PROPAGATE，保证以后的传递。

```java
/**
 * @param node      成功获取共享锁的节点
 * @param propagate 该值>0说明tryAcquireShared成功并且资源还有剩余, 即后续线程也可获取共享锁; =0说明成功但是后续线程成功不了
 */
private void setHeadAndPropagate(Node node, int propagate) {
    // 旧的head
    Node h = head; // Record old head for check below
    setHead(node);
    /**
     * propagate > 0 -> 说明后继线程也可获取共享锁
     * h.waitStatus < 0 -> 主要是为了检测 waitStatus=PROPAGATE 或者 waitStatus=SIGNAL
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

/**
 * 共享模式下的传播唤醒
 */
private void doReleaseShared() {
    for (; ; ) {

        // 如果是setHeadAndPropagate方法的调用, 该头结点为刚获取共享锁的节点
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果head的状态为SIGNAL(表示后继节点需要唤醒), 尝试将其状态设置为0, 设置成功则唤醒后继节点
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 如果head的状态为0(表示后继节点暂时不需要唤醒), 尝试将其状态设置为PROPAGATE, 保证以后的传递
            // 这里应该对应setHeadAndPropagate方法中的h.waitStatus < 0
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 如果头结点没有发生变化，表示设置完成，退出循环
        // 如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试
        if (h == head)                   // loop if head changed
            break;
    }
}
```

**setHeadAndPropagate方法主要需要注意一下方法里的if判断逻辑：**

- **propagate > 0 **：说明还有共享资源可以获取，所以需要唤醒后继节点
- **h == null || h.waitStatus < 0** ：此时的head为老的head，一般不为null（除非被gc了），此时head的状态可能为0或者-3
  - 状态为0的情况：当前节点被别的线程释放锁然后唤醒，此时会将head的状态从-1设置为0；或者当前线程是未阻塞之前获取到锁，那么head的状态可能还没被从0改为-1
  - 状态为-3的情况：当前节点被唤醒，但还没将自己设置为head，此时老的head状态为0，此时有另一线程释放资源，调用doReleaseShared方法，读到老的head状态为0，将0改为-3
- **（h = head） == null || h.waitStatus < 0** ：此时的head为新的head，可能的取值为0，-1，-3
  - 状态为0的情况：没有后继节点，或者后继节点刚入队，还没阻塞（即还没将当前线程的状态修改为-1）
  - 状态为-1的情况：当前节点有后继节点，并且将当前节点的状态改为-1了
  - 状态为-3的情况：有别的线程释放锁，调用doReleaseShared，读新的head状态为0，将0改为-3



**为什么此处propagate没有大于0，还是要根据waitStatus<0去判断是否唤醒后继节点呢？**
PROPAGATE状态一开始出现是为了解决信号量并发释放可能出现的被hang住的问题，见[BUG](http://bugs.java.com/view_bug.do?bug_id=6801020)，复现代码如下所示。

```java
public class TestSemaphore {

    private static Semaphore sem = new Semaphore(0);

    private static class Thread1 extends Thread {
        @Override
        public void run() {
            sem.acquireUninterruptibly();
        }
    }

    private static class Thread2 extends Thread {
        @Override
        public void run() {
            sem.release();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10000000; i++) {
            Thread t1 = new Thread1();
            Thread t2 = new Thread1();
            Thread t3 = new Thread2();
            Thread t4 = new Thread2();
            t1.start();
            t2.start();
            t3.start();
            t4.start();
            t1.join();
            t2.join();
            t3.join();
            t4.join();
            System.out.println(i);
        }
    }
}
```

AQS一开始的代码是没有PROPAGATE的，如下所示：

```java
private void setHeadAndPropagate(Node node, int propagate) {
    setHead(node);
    if (propagate > 0 && node.waitStatus != 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            unparkSuccessor(node);
    }
}
```

那么考虑这样一种情况：

1. 某一时刻t1，t2在排队，t3释放资源，唤醒head的后继t1，并将head的状态由-1 -> 0，并且t1获取资源成功，但是此时tryAcquireShared方法返回0
1. 此刻t4释放资源，读到head为0（此刻还是老的head），不会去唤醒后继节点
1. t1执行到setHeadAndPropagate方法，发现没有资源了(Propagate=0)，不进行唤醒后继节点
1. 然后没人去唤醒t2，就卡住了



引入PROPAGATE状态之后，该问题如何解决的呢：

1. 某一时刻t1，t2在排队，t3释放资源唤醒head的后继t1，并将head的状态由-1 -> 0，并且t1获取资源成功，但是此时tryAcquireShared方法返回0
1. 此刻t4释放资源，读到head为0（此刻还是老的head），将head的状态设置为PROPAGATE，即-3，改代码封装在doReleaseShared方法里
1. t1执行到setHeadAndPropagate方法，发现没有资源了(Propagate=0)，但是此刻head.waitStatus=-3<0，还是会进入doReleaseShared方法，doReleaseShared方法中获取新的head（即t1设置的），head的状态为-1，唤醒后继节点t2



#### 3.3.4 共享锁释放

releaseShared方法主要做的事情：

1. 调用子类实现的tryReleaseShared释放同步状态
1. 调用doReleaseShared方法唤醒head的后继节点

需要注意的是：
ReentrantReadWriteLock 和 Semaphore 的释放逻辑不一样， ReentrantReadWriteLock 全部释放才会返回true，Semaphore 只要释放就会返回true。

```java
    /**
     * 共享锁释放
     */
    public final boolean releaseShared(int arg) {
        // 调用子类实现的tryReleaseShared释放同步状态
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```


## Other

[代码](https://github.com/fuyuaaa/juc-notes/tree/master/src/main/java/top/fuyuaaa/jucnotes/locks)
参考
[https://www.cnblogs.com/lfls/p/7599863.html](https://www.cnblogs.com/lfls/p/7599863.html)
[https://blog.csdn.net/weixin_36586120/article/details/108642253](https://blog.csdn.net/weixin_36586120/article/details/108642253)

