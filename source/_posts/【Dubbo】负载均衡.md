---
title: 【Dubbo】负载均衡
date: 2020-09-22 14:00:00
tags: [dubbo]
categories: dubbo


---

<!-- toc -->

# 概述

由前面【Dubbo】集群容错机制 文章可知，`AbstractClusterInvoker#doSelect`会调用`LoadBalance#select`方法选取一个`Invoker`并返回。

首先，先来看一下`LoadBalance`接口，`LoadBalance`默认使用的负载均衡实现是随机算法，而且过`url`的`loadbalance`参数进行自适应选择扩展。

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    /**
     * @param invokers   供选择的invoker列表.
     * @param url        refer url
     * @param invocation 消费者的调用会被封装成一个invocaion
     * @return 选择的invoker.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```

接下来看一下`LoadBalance`接口的类图，如下所示：`LoadBalance`是顶层接口，`AbstractLoadBalance`封装了通用的实现逻辑。`AbstractLoadBalance`抽象类包含四个子类，分别对应随机、一致性哈希、轮询、最少活跃调用（慢的`provider`收到更少的请求）。<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/LoadBalance类图.png" alt="LoadBalance类图" style="zoom:50%;" />

# 源码解析

## AbstractLoadBalance

### select方法

代码如下所示，逻辑比较简单，不多说。

```java
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    //如果为空，返回空；如果只有一个，直接返回
    if (invokers == null || invokers.isEmpty())
        return null;
    if (invokers.size() == 1)
        return invokers.get(0);
    //调用子类实现的模板方法
    return doSelect(invokers, url, invocation);
}

protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
```

### getWeight方法

该方法用来获取权重，主要逻辑就是计算服务的运行时间，当运行时间小于预热时间时，进行降权，防止服务一启动就进入高负载状态。代码如下所示。

```
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    //获取URl里weight参数，默认100
    int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
    if (weight > 0) {
        //获取provider启动时间戳
        long timestamp = invoker.getUrl().getParameter(Constants.REMOTE_TIMESTAMP_KEY, 0L);
        if (timestamp > 0L) {
            //计算provider运行时间
            int uptime = (int) (System.currentTimeMillis() - timestamp);
            //获取服务预热时间，默认10分钟
            int warmup = invoker.getUrl().getParameter(Constants.WARMUP_KEY, Constants.DEFAULT_WARMUP);
            //如果运行时间小于预热时间，则重新计算权重
            if (uptime > 0 && uptime < warmup) {
                weight = calculateWarmupWeight(uptime, warmup, weight);
            }
        }
    }
    return weight;
}
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    //(uptime / warmup) * weight
    //（运行时间/预热时间）* 权重
    int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
    return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```

## RandomLoadBalance

`RandomLoadBalance`是加权随机负载均衡算法的具体实现，其算法思想比较简单，下面举俩🌰说明一下。

```java
🌰1：
[10,10,10]
三个Invoker的权重都一样，则直接取小于列表长度3的随机数（0，1，2），然后返回对应index的Invoker

🌰2：
[3,4,5]
总权重=12，则会取小于12的随机数，假设随机数是6。
  index=0,6-3=3
  index=1,3-4=-1，-1小于0，所以index=1
返回列表内index为1的Invoker
```

代码如下所示：

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size(); // Number of invokers
    int totalWeight = 0; // The sum of weights
    boolean sameWeight = true; // Every invoker has the same weight?
    //计算总权重，并记录下是否有不一样的权重值。
    //如果权重值全都一样，则直接取小于列表大小的随机数，可看最后一步。
    for (int i = 0; i < length; i++) {
        int weight = getWeight(invokers.get(i), invocation);
        totalWeight += weight; // Sum
        if (sameWeight && i > 0
                && weight != getWeight(invokers.get(i - 1), invocation)) {
            sameWeight = false;
        }
    }
    //权重值存在不一样的情况。
    //取一个小于总权重的随机数，然后从第一个开始遍历，一个个减掉Invoker对应的权重，当减到负数时，说明落在该Invoker对应的权重区间上。
    if (totalWeight > 0 && !sameWeight) {
        int offset = random.nextInt(totalWeight);
        for (int i = 0; i < length; i++) {
            offset -= getWeight(invokers.get(i), invocation);
            if (offset < 0) {
                return invokers.get(i);
            }
        }
    }
    return invokers.get(random.nextInt(length));
}
```



## RoundRobinLoadBalance

`RoundRobinLoadBalance`是平滑加权轮询负载均衡算法的具体实现，主要包含以下逻辑：

1. 遍历`Invoker`列表，如果`Invoker`对应的`WeightedRoundRobin`不存在，则新建一个并初始化权重
2. 将当前`current = current + weight`，找出当前`current`最大的，并更新`lastUpdate`
3. 删除长时间未更新`lastUpdate的WeightedRoundRobin`
4. `current = current - 总权重`，返回选中的`Invoker`

下面看个例子：

```java
🌰：
[1,2,3] 对应 A B C三个Invoker  
 第一次：[0,0,0] -> [1,2,3] -> [1,2,-3] 选C
 第二次：[1,2,-3] -> [2,4,0] -> [2,-2,0] 选B
 第三次：[2,-2,0] -> [3,0,3] -> [3,0,-3] 选C
 第四次：[3,0,-3] -> [4,2,0] -> [-2,2,0] 选A
 第五次：[-2,2,0] -> [-1,4,3] -> [-1,-2,3] 选B
 第六次：[-1,-2,3] -> [0,0,6] -> [0,0,0] 选C
A B C 分别对应 1 2 3次
  
ps：
 中间一列，每个都加上自身权重。
 最后一列，会用最大的current建议一个totalWeight。
 所以总加的数量和总减的数量是一样的。
 自身权重越大的节点增长越快，那么比其他节点大的几率就越高，被选中的机会就越多。
```

代码如下所示：

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    //服务名+方法名
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
    ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.get(key);
    if (map == null) {
        methodWeightMap.putIfAbsent(key, new ConcurrentHashMap<String, WeightedRoundRobin>());
        map = methodWeightMap.get(key);
    }
    int totalWeight = 0;
    long maxCurrent = Long.MIN_VALUE;
    long now = System.currentTimeMillis();
    Invoker<T> selectedInvoker = null;
    WeightedRoundRobin selectedWRR = null;
    //遍历所有Invoker
    for (Invoker<T> invoker : invokers) {
        String identifyString = invoker.getUrl().toIdentityString();
        WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
        int weight = getWeight(invoker, invocation);
        if (weight < 0) {
            weight = 0;
        }
        //如果map中不存在对应的WeightedRoundRobin，则初始化一个put进去
        if (weightedRoundRobin == null) {
            weightedRoundRobin = new WeightedRoundRobin();
            weightedRoundRobin.setWeight(weight);
            map.putIfAbsent(identifyString, weightedRoundRobin);
            weightedRoundRobin = map.get(identifyString);
        }
        //如果权重与通过getWeight得到的不一样，则设置成getWeight得到的
        //这里不相等的原因可能是还在预热，因为在预热阶段，weight是会随着时间变大的。
        if (weight != weightedRoundRobin.getWeight()) {
            //weight changed
            weightedRoundRobin.setWeight(weight);
        }

        //increaseCurrent = current + weight
        long cur = weightedRoundRobin.increaseCurrent();
        weightedRoundRobin.setLastUpdate(now);
        //这里是为了找出最大的那一个
        if (cur > maxCurrent) {
            maxCurrent = cur;
            selectedInvoker = invoker;
            selectedWRR = weightedRoundRobin;
        }
        //累加权重
        totalWeight += weight;
    }
    
    //上面那个for循环会更新lastUpdate，如果map里某个WeightedRoundRobin的lastupdate长时间未被更新，说明他可能挂了，所以把他移除掉。
    if (!updateLock.get() && invokers.size() != map.size()) {
        if (updateLock.compareAndSet(false, true)) {
            try {
                // copy -> modify -> update reference
                ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<String, WeightedRoundRobin>();
                newMap.putAll(map);
                Iterator<Entry<String, WeightedRoundRobin>> it = newMap.entrySet().iterator();
                while (it.hasNext()) {
                    Entry<String, WeightedRoundRobin> item = it.next();
                    if (now - item.getValue().getLastUpdate() > RECYCLE_PERIOD) {
                        it.remove();
                    }
                }
                methodWeightMap.put(key, newMap);
            } finally {
                updateLock.set(false);
            }
        }
    }
    if (selectedInvoker != null) {
        //current = current-weight
        selectedWRR.sel(totalWeight);
        return selectedInvoker;
    }
    // should not happen here
    return invokers.get(0);
}
```



## LeastActiveLoadBalance

`LeastActiveLoadBalance`是最少活跃调用数负载均衡算法的集体实现，主要包含以下逻辑：

1. 遍历`invoker`列表，找到最小活跃数的`invoker`数组（可能存在多个活跃数=最小活跃数的情况）
2. 如果只有一个直接返回
3. 如果存在多个，则根据权重是否相同来决定使用加权随机还是随机。

代码如下所示：

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size(); // Number of invokers
    int leastActive = -1; // The least active value of all invokers
    int leastCount = 0; // The number of invokers having the same least active value (leastActive)
    int[] leastIndexs = new int[length]; // The index of invokers having the same least active value (leastActive)
    int totalWeight = 0; // The sum of with warmup weights
    int firstWeight = 0; // Initial value, used for comparision
    boolean sameWeight = true; // Every invoker has the same weight value?
    //遍历invokers列表
    for (int i = 0; i < length; i++) {
        Invoker<T> invoker = invokers.get(i);
        //获取当前invoker的活跃适量
        int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); // Active number
        //权重
        int afterWarmup = getWeight(invoker, invocation); // Weight
        //如果活跃数量=-1 (说明是第一个遍历) 或者 当前活跃数量小于最小活跃数
        //将当前活跃数量设置到最小活跃数，并更新其他字段
        if (leastActive == -1 || active < leastActive) { // Restart, when find a invoker having smaller least active value.
            leastActive = active; // Record the current least active value
            leastCount = 1; // Reset leastCount, count again based on current leastCount
            leastIndexs[0] = i; // Reset
            totalWeight = afterWarmup; // Reset
            firstWeight = afterWarmup; // Record the weight the first invoker
            sameWeight = true; // Reset, every invoker has the same weight value?
        }
        //如果当前活跃数量 = 最小活跃数，则加入到leastIndexs数组，并累加权重
        else if (active == leastActive) { // If current invoker's active value equals with leaseActive, then accumulating.
            leastIndexs[leastCount++] = i; // Record index number of this invoker
            totalWeight += afterWarmup; // Add this invoker's weight to totalWeight.
            // If every invoker has the same weight?
            //判断活跃数相等的，权重是否相等
            if (sameWeight && i > 0
                    && afterWarmup != firstWeight) {
                sameWeight = false;
            }
        }
    }
    // assert(leastCount > 0)
    //如果只有一个活跃数=最小活跃数，直接返回
    if (leastCount == 1) {
        // If we got exactly one invoker having the least active value, return this invoker directly.
        return invokers.get(leastIndexs[0]);
    }
    //存在多个活跃数=最小活跃数并且权重不相等的情况，进行加权随机
    if (!sameWeight && totalWeight > 0) {
        // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
        int offsetWeight = random.nextInt(totalWeight) + 1;
        // Return a invoker based on the random value.
        for (int i = 0; i < leastCount; i++) {
            int leastIndex = leastIndexs[i];
            offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
            if (offsetWeight <= 0)
                return invokers.get(leastIndex);
        }
    }
    // If all invokers have the same weight value or totalWeight=0, return evenly.
    //存在多个活跃数=最小活跃数并且权重相等
    return invokers.get(leastIndexs[random.nextInt(leastCount)]);
}
```

## ConsistentHashLoadBalance

一致性`hash`算法的原理此处不做解释，直接看一下`Dubbo`一致性`hash`负载均衡的实现。

### doSelect方法

首先，`ConsistentHashLoadBalance`内部维护了一个`ConcurrentMap`，缓存了方法以及对应的`ConsistentHashSelector`。`doSelect`方法主要做的事情，就是通过`hashCode`来检测`Invoker`列表是否发生了变化，如果发生了变化，则重新初始化`ConsistentHashSelector`，然后调用`ConsistentHashSelector#select`方法选择一个`Invoker`并返回。代码如下所示

```java
private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    String methodName = RpcUtils.getMethodName(invocation);
    String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
    int identityHashCode = System.identityHashCode(invokers);
    ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
    //如果没有初始化过或者invokers列表发生了变化，则需要重新初始化一次。
    if (selector == null || selector.identityHashCode != identityHashCode) {
        selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
        selector = (ConsistentHashSelector<T>) selectors.get(key);
    }
    //选择一个Invoker
    return selector.select(invocation);
}
```

### ConsistentHashSelector

接下来看一下`ConsistentHashSelector`的源码，注释很详细了，代码如下所示：

```java
private static final class ConsistentHashSelector<T> {

    //虚拟节点，用TreeMap来实现，TreeMap的tailMap(K fromKey, boolean inclusive)方法可以获取比fromKey大的子map
    private final TreeMap<Long, Invoker<T>> virtualInvokers;

    //虚拟节点数量，默认会取160
    private final int replicaNumber;

    //这个用来校验Invoker列表是不是发生了变化
    private final int identityHashCode;

    //在配置中可以用hash.arguments指定需要进行hash值计算的参数，比如0,1,2就是用第0,1,2这三个参数，默认取0
    private final int[] argumentIndex;

    //初始化一致性hash选择器
    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
        this.identityHashCode = identityHashCode;
        URL url = invokers.get(0).getUrl();
        this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
        String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) {
            argumentIndex[i] = Integer.parseInt(index[i]);
        }

        //遍历invokers列表，
        for (Invoker<T> invoker : invokers) {
            String address = invoker.getUrl().getAddress();
            for (int i = 0; i < replicaNumber / 4; i++) {
                byte[] digest = md5(address + i);
                for (int h = 0; h < 4; h++) {
                    long m = hash(digest, h);
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }

    public Invoker<T> select(Invocation invocation) {
        //将参数转成key，根据argumentIndex拼接key
        String key = toKey(invocation.getArguments());
        //md5算法，生成16个字节数组
        byte[] digest = md5(key);
        //计算hash值，使用selectForKey方法查询大于该hash值的第一个节点
        return selectForKey(hash(digest, 0));
    }

    //根据argumentIndex数据，拼接key，key是后面用来计算hash的
    //具体逻辑是，根据argumentIndex里的值取参数，比如0,2，就会取args里index=0,2的参数进行拼接
    private String toKey(Object[] args) {
        StringBuilder buf = new StringBuilder();
        for (int i : argumentIndex) {
            if (i >= 0 && i < args.length) {
                buf.append(args[i]);
            }
        }
        return buf.toString();
    }

    //使用TreeMap.tailMap方法获取大于hash的子map
    private Invoker<T> selectForKey(long hash) {
        Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
        if (entry == null) {
            entry = virtualInvokers.firstEntry();
        }
        return entry.getValue();
    }

    private long hash(byte[] digest, int number) {
        return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                | (digest[number * 4] & 0xFF))
                & 0xFFFFFFFFL;
    }

    private byte[] md5(String value) {
        MessageDigest md5;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        md5.reset();
        byte[] bytes;
        try {
            bytes = value.getBytes("UTF-8");
        } catch (UnsupportedEncodingException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        md5.update(bytes);
        return md5.digest();
    }

}
```

# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)

