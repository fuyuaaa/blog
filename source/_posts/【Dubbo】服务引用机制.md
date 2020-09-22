---
title: 【Dubbo】服务引用机制
date: 2020-09-14 18:09:00
tags: [dubbo]
categories: dubbo
---

<!-- toc -->

# 概述

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200911172613230.png" alt="服务引用整体流程" style="zoom: 33%;" />

首先，看一下服务引用的整体流程，如上图所示。`Dubbo`框架将服务消费分为两大部分，第一步通过持有远程服务实例生成`Invoker`，这个`Invoker`在客户端是核心的远程代理对象。第二步会把`Invoker`通过动态代理转换成实现用户接口的动态代理引用。

# 源码解析

## ReferenceConfig类

服务引用的逻辑是从`ReferenceBean#getObject`开始的，`ReferenceBean`实现了`FactoryBean`。`FactoryBean`是一个工厂`Bean`，可以生成某一个类型`Bean`实例，它最大的一个作用是：可以让我们自定义`Bean`的创建过程。

如以下代码所示，当调用`getBean`方法时，最终会调用到`ReferenceBean#getObject`，并触发`ReferenceConfig#init`方法，生成`TestService`的代理对象并返回。

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"/dubbo-demo-consumer.xml"});
context.start();
TestService testService = (TestService) context.getBean("testService");
```

## ReferenceConfig#createProxy方法

在`ReferenceBean#getObject`被调用时，会调用`ReferenceConfig#get`方法，此时会检查当前引用对象是否已生成，如果未生成则会触发`init`方法，生成引用对象。init方法会对配置进行检查和处理，保证配置的正确性，然后会调用`createProxy`方法创建代理对象。

`createProxy`方法主要包含以下逻辑：

1. 处理本地调用的情（`injvm=true || scope=local`）
2. 处理远程调用的情况（分以下两种）
   1. 点对点调用
   2. 通过注册中心
3. 可用性检查
4. 创建代理类

`createProxy`方法代码比较多，这里分开解析，首先来看一下处理本地调用的逻辑。代码如下所示：

```java
final boolean isJvmRefer;
//没有配置injvm=true
//ps 本地调用有两种配置方式 injvm=true  以及 scope=local，前者已经被@Deprecated。
if (isInjvm() == null) {
    //url配置被指定，则不做本地引用
    if (url != null && url.length() > 0) { // if a url is specified, don't do local reference
        isJvmRefer = false;
    }
    //如果用户显式配置了scope=local或者injvm=true，此时isInjvmRefer返回true
    else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
        // by default, reference local service if there is
        isJvmRefer = true;
    } else {
        isJvmRefer = false;
    }
} 
//配置了injvm
else {
    //获取injvm配置值
    isJvmRefer = isInjvm().booleanValue();
}

//本地引用
if (isJvmRefer) {
    //生成本地引用URL，协议为injvm
    URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
    //调用refer方法构建InjvmInvoker实例，根据url的协议会去调用InjvmProtocol#refer
    invoker = refprotocol.refer(interfaceClass, url);
    //省略日志输出代码
}
```

本地调用逻辑比较简单，看注释就可。接下来看一下远程调用的具体流程，主要包含以下逻辑：

1. 判断有无指定`url`，指定了`url`说明是想要进行点对点调用
2. 未指定`url`，则加载注册中心的`url`
3. 把`url`搞成`invoker`，多个`url`则通过`cluster`合并

代码如下所示：

```java
//url不为空，说明在配置中指定了url的值，表示想进行点对点调用
//比如<dubbo:reference version="1.0" id="xx" interface="xx" url="xxx"/>
if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
    //当配置多个url时，可用分号进行分割，这里会进行切分
    String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
    if (us != null && us.length > 0) {
        //遍历多个url，加入到urls列表
        for (String u : us) {
            URL url = URL.valueOf(u);
            if (url.getPath() == null || url.getPath().length() == 0) {
                //设置url路径=接口全限定名
                url = url.setPath(interfaceName);
            }
            //检测url协议是否为registry，若是，表明用户想使用指定的注册中心
            if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                //将map转换为查询字符串，并作为refer参数的值添加到url中
                urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
            } else {
                //合并 url，移除服务提供者的一些配置（这些配置来源于用户配置的 url 属性），
                //比如线程池相关配置。并保留服务提供者的部分配置，比如版本，group，时间戳等
                //最后将合并后的配置设置为 url 查询字符串中。
                urls.add(ClusterUtils.mergeUrl(url, map));
            }
        }
    }
}
//url为空，表示通过注册中心获取url
else { // assemble URL from register center's configuration
    // 加载注册中心 url
    List<URL> us = loadRegistries(false);
    if (us != null && !us.isEmpty()) {
        for (URL u : us) {
            URL monitorUrl = loadMonitor(u);
            if (monitorUrl != null) {
                map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
            }
            urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
        }
    }
    //未配置注册中心，抛出异常
    if (urls.isEmpty()) {
        //省略异常抛出代码
    }
}

//单个注册中心或服务提供者
if (urls.size() == 1) {
    invoker = refprotocol.refer(interfaceClass, urls.get(0));
}
//多个注册中心或多个服务提供者，或者两者混合
else {
    List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
    URL registryURL = null;
    for (URL url : urls) {
        //根据url协议头加载指定的Protocol实例，并调用实例的refer方法构建invoker，比如registry就会调用RegistryProtocol#refer
        invokers.add(refprotocol.refer(interfaceClass, url));
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            registryURL = url; // use last registry url
        }
    }
    //通过cluster合并多个invoker
    //如果存在注册中心的url，则会使用AvailableCluster，AvailableCluster会先判断是否可用，再进行invoke调用。
    if (registryURL != null) { // registry url is available
        // use AvailableCluster only when register's cluster is available
        URL u = registryURL.addParameterIfAbsent(Constants.CLUSTER_KEY, AvailableCluster.NAME);
        invoker = cluster.join(new StaticDirectory(u, invokers));
    } else { // not a registry url
        invoker = cluster.join(new StaticDirectory(invokers));
    }
}
```

`createProxy`方法还有最后一份逻辑是关于可用性检查以及使用`invoker`创建代理类，代码如下所示：

```java
//如果为配置check=false（默认true），则会进行可用性检查，未通过则会抛出异常。
Boolean c = check;
if (c == null && consumer != null) {
    c = consumer.isCheck();
}
if (c == null) {
    c = true; // default true
}
//可用性检查
if (c && !invoker.isAvailable()) {
    // make it possible for consumer to retry later if provider is temporarily unavailable
    initialized = false;
    //省略异常抛出代码
}
if (logger.isInfoEnabled()) {
    logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
}
// create service proxy
//调用JavassistProxyFactory#getProxy生成代理类。
return (T) proxyFactory.getProxy(invoker);
```

## Protocol#refer方法

无论是本地调用还是远程调用，都会使用`Protocol#refer`方法来创建`invoker`。本地调用使用的`InjvmProtocol#refer`逻辑比较简单，就是创建了一个`InjvmInvoker`对象并返回。这里主要看一下`RegistryProtocol`以及`DubboProtocol`的`refer`方法。

### RegistryProtocol#refer方法

`RegistryProtocol#refer`方法的逻辑比较简单，先获取`url`的协议参数并设置成协议头，再调用`doRefer`方法生成`invoker`并返回。

代码如下所示：

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    //取registry参数值，并将其设置为协议头
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    //获取注册中心实例，如果协议为zookeeper则获取到ZookeeperRegistry
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }
    //省略group配置处理相关代码
    return doRefer(cluster, registry, type, url);
}
```

接下来看一下`doRefer`方法，该方法主要包含一下逻辑：

1. 创建`RegistryDirectory`实例
2. 注册服务消费者，在`consumers`目录下新建节点
3. 订阅`providers、configurators、routers`等节点数据
4. 通过`cluster`创建`invoker`

代码如下所示：

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    //创建RegistryDirectory实例
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);

    //注册服务消费者，在consumers目录下新节点
    if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        URL registeredConsumerUrl = getRegisteredConsumerUrl(subscribeUrl, url);
        registry.register(registeredConsumerUrl);
        directory.setRegisteredConsumerUrl(registeredConsumerUrl);
    }

    //订阅providers、configurators、routers等节点数据
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
            Constants.PROVIDERS_CATEGORY
                    + "," + Constants.CONFIGURATORS_CATEGORY
                    + "," + Constants.ROUTERS_CATEGORY));

    //一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个
    Invoker invoker = cluster.join(directory);
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

### DubboProtocol#refer方法

`DubboProtocol#refer`方法逻辑比较简单，主要就是创建一个`DubboInvoker`并返回。代码如下所示：

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    //创建DubboInvoker
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

这里主要看一下`getClients`方法，该方法用来获取客户端实例。代码如下所示：

```java
private ExchangeClient[] getClients(URL url) {
    // whether to share connection
    // 是否共享连接
    boolean service_share_connect = false;
    // 获取connections参数，connections表示该服务对每个提供者建立的长连接数。
    // 如果未配置或配置了0，则默认共用一个长连接。否则一个service使用connections个连接。
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // if not configured, connection is shared, otherwise, one connection for one service
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }

    //初始化客户端实例
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect) {
            //获取共享客户端
            clients[i] = getSharedClient(url);
        } else {
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

这里解释一下`connections`参数的含义：`connections`参数表示最大的长连接数量，如果`connections`未配置或者值为0，则共享一个长连接（这里的共享指的是`consumer`与某个`provider`只建立一个长连接）；如果`connection`配置了值且不为0，则表示每个`service`最大可创建`connections`个长连接。

接下来看一下`getSharedClient`方法，该方法用来获取共享客户端，代码如下所示：

```java
private ExchangeClient getSharedClient(URL url) {
  	//key为address，所以是相同地址共享一个连接
    String key = url.getAddress();
    //从缓存里获取客户端实例，如果未获取到，则新建一个ExchangeClient实例
    //ReferenceCountExchangeClient就是个带引用计数功能的ExchangeClient，表示该连接实例被多少个service共享了
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    if (client != null) {
        if (!client.isClosed()) {
            client.incrementAndGetCount();
            return client;
        } else {
            referenceClientMap.remove(key);
        }
    }

    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        if (referenceClientMap.containsKey(key)) {
            return referenceClientMap.get(key);
        }

        //初始化客户端实例，然后丢到map里
        ExchangeClient exchangeClient = initClient(url);
        client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
        referenceClientMap.put(key, client);
        ghostClientMap.remove(key);
        locks.remove(key);
        return client;
    }
}
```

`getSharedClient`方法在未查询到缓存时，会调用`initClient`方法创建新的客户端实例，`initClient`代码如下所示：

```java
private ExchangeClient initClient(URL url) {

    // client type setting.
    // 获取客户端类型，默认为netty
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    // enable heartbeat by default
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

    // BIO is not allowed since it has severe performance issue.
    //判断客户端类型是否支持
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        //省略异常抛出代码
    }

    ExchangeClient client;
    try {
        // connection should be lazy
        //创建客户端实例，懒加载（request调用时才进行创建）&普通
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            client = Exchangers.connect(url, requestHandler);
        }
    } //省略异常抛出代码
    return client;
```



## ProxyFactory#getProxy方法

`ReferenceConfig#createProxy`在创建`invoker`之后会进行可用性检查，然后会调用`ProxyFactory.getProxy`创建代理类。

代码如下所示：

```
public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    Class<?>[] interfaces = null;
    //获取接口列表
    String config = invoker.getUrl().getParameter("interfaces");
    if (config != null && config.length() > 0) {
        String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
        if (types != null && types.length > 0) {
            interfaces = new Class<?>[types.length + 2];
            interfaces[0] = invoker.getInterface();
            interfaces[1] = EchoService.class;
            for (int i = 0; i < types.length; i++) {
                interfaces[i + 2] = ReflectUtils.forName(types[i]);
            }
        }
    }
    //创建接口数组
    if (interfaces == null) {
        interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
    }

    //省略泛化调用代码
    
    //生成代理类
    return getProxy(invoker, interfaces);
}
```

> 回声测试用于检测服务是否可用，回声测试按照正常请求流程执行，能够测试整个调用是否通畅，可用于监控。所有服务自动实现 `EchoService` 接口，只需将任意服务引用强制转型为 `EchoService`，即可使用。
>
> ```java
> // 远程服务引用
> MemberService memberService = ctx.getBean("memberService"); 
> EchoService echoService = (EchoService) memberService; // 强制转型为EchoService
> // 回声测试可用性
> String status = echoService.$echo("OK"); 
> assert(status.equals("OK"));
> ```
>
> 

在获取`interfaces`数组之后，会调用`JavassistProxyFactory#getProxy`方法生成代理类对象，该方法不做解析，这里通过反编译看一下生成的代理类的具体代码。

```java
/*
 * Decompiled with CFR.
 * 
 * Could not load the following classes:
 *  top.fuyuaaa.api.TestService
 */
package com.alibaba.dubbo.common.bytecode;

import com.alibaba.dubbo.common.bytecode.ClassGenerator;
import com.alibaba.dubbo.rpc.service.EchoService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import top.fuyuaaa.api.TestService;

public class proxy0 implements ClassGenerator.DC,TestService,EchoService {
    public static Method[] methods;
    private InvocationHandler handler;

    public String demo(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[0], arrobject);
        return (String)object;
    }

    @Override
    public Object $echo(Object object) {
        Object[] arrobject = new Object[]{object};
        Object object2 = this.handler.invoke(this, methods[1], arrobject);
        return object2;
    }

    public proxy0() {
    }

    public proxy0(InvocationHandler invocationHandler) {
        this.handler = invocationHandler;
    }
}

```

# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)