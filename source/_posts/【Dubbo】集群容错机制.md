---
title: 【Dubbo】集群容错机制
date: 2020-09-15 17:31:00
tags: [dubbo]
categories: dubbo

---

<!-- toc -->

> 在微服务环境中，为了保证服务的高可用，很少会有单点服务出现，服务通常都是以集群 的形式出现的。当某个服务调用出现异常时，如网络抖动、服务短暂不可用需要自动容错、服务降级，就需要使用到集群容错机制。

# Cluster层

`Cluster`可以看作是一个集群容错层，该层中包含`Cluster、Directory、Router、LoadBalance`几大核心接口。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/Cluster层整体流程.png" alt="Cluster层整体流程" style="zoom: 50%;" />

`Cluster`层的整体工作流程如上图所示，其中第1、2、3步都是在`AbstractClusterInvoker`类中实现，第4步会调用`doInvoke`方法，该方法为一个模板方法，交由子类去实现。

# 容错机制扩展

`Dubbo`提供了`Failover、Failfast、Failsafe、Failback、Forking、Broadcast、AvailableClusterInvoker`等容错机制，默认选择`Failover`机制。下面简单介绍一下这几种容错机制：

## Failover

当出现失败时，会**重试**其他服务器。用户可以通过`retries="2"`，设置重试次数（**不包含第一次**）。这是`Dubbo`的**默认**容错机制，会对请求做负载均衡。通常使用在**读操作或幂等写操作**上， 但重试会导致接口的延迟增大，在下游机器负载已经达到极限时，重试容易加重下游服务的负载。

## Failfast

**快速失败**，当请求失败后，快速返回异常结果，**不做任何重试**。该容错机制会对请求做负载均衡，通常使用在**非幂等**接口的调用上，比如新增记录。

## Failsafe

当出现异常时，**直接忽略异常**。会对请求做负载均衡。通常使用在“佛系”调用场景， 即不关心调用是否成功，并且不想抛异常影响外层调用，如某些不重要的日志同步，即使出现异常也无所谓。

## Failback

请求失败后，会**自动记录在失败队列中，并由一个定时线程池定时重试**，适用于一些**异步或最终一致性**的请求。请求会做负载均。

## Forking

同时调用多个相同的服务，只要其中一个返回，则立即返回结果。用户可以配置`forks="2"`来设置最大并行调用的服务数量。通常使用在对接口实时性要求极高的调用上，但也会浪费更多的资源。

## Broadcast

**广播**调用所有可用的服务，**任意一个节点报错则报错**。由于是广播，因此请求**不需要**做负载均衡。通常用于服务状态更新后的广播。

## Available

请求**不会做**负载均衡，**遍历**所有服务列表，找到**第一个可用**的节点， 直接请求并返回结果。如果没有可用的节点，则直接抛出异常。

## Mock

提供调用失败时，返回伪造的响应结果。或直接强制返回伪造的结果，不会发起远程调用。

## Mergeable

可以自动把多个节点请求得到的结果进行合并。

# 容错接口

容错接口主要分为两大类，`Cluester`以及`ClusterInvoker`。先来看一下`Cluester`与`ClusterInvoker`接口的类图，如下图所示：

![Cluster类图](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/Cluster类图.png)

![AbstractClusterInvoker类图](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/AbstractClusterInvoker类图.png)

两者的关系如以下代码所示，`Cluster`接口下面有多种不同的实现，每种实现中都需要实现接口的`join`方法，在方法中会“`new`”一个对应的`ClusterInvoker`。

```
@SPI(FailoverCluster.NAME)
public interface Cluster {
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}

public class FailoverCluster implements Cluster {
    public final static String NAME = "failover";
    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }
}
```

# 源码解析

## AbstractClusterInvoker#invoke方法

`invoke`方法主要包含以下逻辑：

1. 设置`attachments`
2. 获取可用的`Invoker`列表
3. 获取负载均衡策略（默认`random`）
4. 调用子类实现的`doInvoke`方法

```java
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();
    LoadBalance loadbalance = null;

    // binding attachments into invocation.
    //设置attachments，attachments用来在服务消费方和提供方之间进行参数的隐式传递
    //可以看官方文档http://dubbo.apache.org/zh-cn/docs/user/demos/attachment.html
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }

    //获取可调用的Invoker列表
    List<Invoker<T>> invokers = list(invocation);
    //获取负载均衡策略，默认random
    if (invokers != null && !invokers.isEmpty()) {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl().getMethodParameter(RpcUtils.getMethodName(invocation), Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    }
    //幂等操作，如果是异步调用，则在attachments里添加invocationId，每次异步调用id都会+1。
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    //调用子类实现的doInvoke方法
    return doInvoke(invocation, invokers, loadbalance);
}
```

## AbstractClusterInvoker#list方法

`list`方法的逻辑比较简单，直接调用`Directory#list`方法获取可用的Invoker列（关于`Directory#list`方法后续再分析），代码如下所示：

```java
protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
    List<Invoker<T>> invokers = directory.list(invocation);
    return invokers;
}
```

## FailoverClusterInvoker#doInvoke

`FailoverClusterInvoker#doInvoke`方法主要包含以下逻辑：

1. 校验`invoker`列表是否为空
2. 获取重试次数
3. 循环调用
   1. 重新获取`invokers`（如果是重试阶段），并再次校验`invoker`列表是否为空
   2. 负载均衡选择一个`invoker`
   3. 远程调用，成功则返回
   4. 失败则记录异常和`providers`
4. 重试完了还没成功，抛出异常

代码如下所示：

```
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    //检测invokers是否为空
    checkInvokers(copyinvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    //获取重试次数+1，因为设置的值是不包括第一次的
    int len = getUrl().getMethodParameter(methodName, Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    //已经调用过的invoker
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    //循环重试
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        if (i > 0) {
            checkWhetherDestroyed();
            //在重试之前重新获取invokers列表。
            copyinvokers = list(invocation);
            // check again
            //再次检测invokers是否为空
            checkInvokers(copyinvokers, invocation);
        }
        //负载均衡选择Invoker
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        //将invoker添加到已经调用过的invoked列表
        invoked.add(invoker);
        //设置invoker到rpc上下文里
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            //调用
            Result result = invoker.invoke(invocation);
            //省略日志输出代码
            }
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    //重试完了，还没成功
    throw new RpcException("省略异常抛出内容代码");
}
```

## FailfastClusterInvoker#doInvoke方法

由于前面的介绍可以知道，`failfast`策略遇到异常会直接抛出。所以该`doInvoker`方法主要包含以下逻辑：

1. 检测`invokers`是否为空
2. 负载均衡选择`Invoker`
3. 调用，成功就返回，遇到异常直接抛出

代码如下所示：

```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    //检测invokers是否为空
    checkInvokers(invokers, invocation);
    //负载均衡选择Invoker
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        //调用，遇到异常直接抛出
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
            throw (RpcException) e;
        }
        throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName() + " select from all providers " + invokers + " for service " + getInterface().getName() + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
    }
}
```



由以上两个`doInvoke`方法的分析可知，`ClusterInvoker`的大致流程是：检测 -> 负载均衡 -> 调用 -> 异常处理。其他`ClusterInvoker`的逻辑都与其类似，故此处暂且不做分析。

## AbstractClusterInvoker#select方法

我们可以注意到，如果需要负载均衡，在`doInvoker`方法内部都会调用`select`方法。`AbstractClusterInvoker#select`主要是针对粘滞连接做了特定的处理，若粘滞连接为空或不可用则调用`doSelect`方法重新选取`Invoker`，具体可看代码注释。这里再讲一下几个参数的含义：

1. `invokers`：可用的服务列表
2. `invoked`：已经调用过的服务列表（没调成功的）
3. [粘滞连接](http://dubbo.apache.org/zh-cn/docs/user/demos/stickiness.html)：用于有状态服务，尽可能让客户端总是向同一提供者发起调用，除非该提供者挂了，再连另一台。

代码如下所示：

```java
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (invokers == null || invokers.isEmpty())
        return null;
    //获取方法名
    String methodName = invocation == null ? "" : invocation.getMethodName();

    //获取sticky，sticky表示粘滞连接。
    boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY);
    {
        //ignore overloaded method
        //如果invokers列表不包括stickyInvoker，则说明stickyInvoker挂了，这里将其置空
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        //ignore concurrency problem
        //selected是已经调用过的Invoker列表。如果selected包含stickyInvoker，则说明stickyInvoker没调成功。但是如果invokers还是包含stickyInvoker话，说明stickyInvoker没挂。
        //判断的含义 ： （支持粘滞连接 && 粘滞连接的Invoker不为空 && （粘滞连接未被调用过））
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            //如果打开了可用性检查，则检查stickyInvoker是否可用，可用则返回
            if (availablecheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }
    }
    //重新选一个
    Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

    //如果支持粘滞连接
    if (sticky) {
        stickyInvoker = invoker;
    }
    return invoker;
}
```

## AbstractClusterInvoker#doSelect方法

`AbstractClusterInvoker#doSelect`方法主要包含以下逻辑：

1. 通过负载均衡策略选择`Invoker`
2. 如果`Invoker`已经被调用过了，或者未进行或未通过可用性检查，则进行重选
3. 重选成功则返回，失败则选（第一步选出来的`Invoker`所在列表）下一个

```
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
    if (invokers == null || invokers.isEmpty())
        return null;
    //如果就一个，选个🔨，直接返回
    if (invokers.size() == 1)
        return invokers.get(0);
    //如果loadbalance，则加载默认的负载均衡策略
    if (loadbalance == null) {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

    //selected包含该invoker || （invoker未进行或未通过可用性检查） -> 重选
    if ((selected != null && selected.contains(invoker))
            || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
        try {
            Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            //如果重选的rinvoker不为空，则赋值给invoker
            if (rinvoker != null) {
                invoker = rinvoker;
            } 
            //如果重选的rinvoker为空，则选择 (刚刚选出来的那个invoker所在列表) 的下一个，如果是最后一个则选第一个
            else {
                int index = invokers.indexOf(invoker);
                try {
                    invoker = index < invokers.size() - 1 ? invokers.get(index + 1) : invokers.get(0);
                } //省略异常日志代码
            }
        } //省略异常日志代码
    }
    return invoker;
}
```

## AbstractClusterInvoker#reSelect方法

`AbstractClusterInvoker#reSelect`方法主要进行重选操作，逻辑如下所示：

1. 找到可用的`Invoker`，加入到`reselectInvokers`
2. 如果`reselectInvokers`不为空，则通过负载均衡策略再次选择

代码如下所示：

```java
private Invoker<T> reselect(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected, boolean availablecheck)
        throws RpcException {

    //Allocating one in advance, this list is certain to be used.
    List<Invoker<T>> reselectInvokers = new ArrayList<Invoker<T>>(invokers.size() > 1 ? (invokers.size() - 1) : invokers.size());

    //First, try picking a invoker not in `selected`.
    //允许可用性检查，遍历invokers列表，找到可用的，且未被调用过的，丢到reselectInvokers列表中，再用reselectInvokers进行负载均衡选择并返回
    if (availablecheck) { // invoker.isAvailable() should be checked
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                if (selected == null || !selected.contains(invoker)) {
                    reselectInvokers.add(invoker);
                }
            }
        }
        if (!reselectInvokers.isEmpty()) {
            return loadbalance.select(reselectInvokers, getUrl(), invocation);
        }
    }
    //不允许可用性检查，遍历invokers列表，找到未被调用过的，丢到reselectInvokers列表中，再用reselectInvokers进行负载均衡选择并返回
    else { // do not check invoker.isAvailable()
        for (Invoker<T> invoker : invokers) {
            if (selected == null || !selected.contains(invoker)) {
                reselectInvokers.add(invoker);
            }
        }
        if (!reselectInvokers.isEmpty()) {
            return loadbalance.select(reselectInvokers, getUrl(), invocation);
        }
    }
    // Just pick an available invoker using loadbalance policy
    //如果执行到这了，则说明reselectInvokers为空，直接在selected里找可用，丢到reselectInvokers列表中，再用reselectInvokers进行负载均衡选择并返回
    {
        if (selected != null) {
            for (Invoker<T> invoker : selected) {
                if ((invoker.isAvailable()) // available first
                        && !reselectInvokers.contains(invoker)) {
                    reselectInvokers.add(invoker);
                }
            }
        }
        if (!reselectInvokers.isEmpty()) {
            return loadbalance.select(reselectInvokers, getUrl(), invocation);
        }
    }
    return null;
}
```

关于`LoadBalance#select`方法的代码后续再分析。

# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)