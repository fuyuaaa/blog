---
title: 【Dubbo】服务字典
date: 2020-09-18 10:08:00
tags: [dubbo]
categories: dubbo

---

<!-- toc -->

# 概述

由前面【Dubbo】集群容错机制 文章可知，整个容错过程中，会先调用`Directory#list`方法来获取所有可用的`Invoker`列表。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/Directory类图.png" alt="Directory类图" style="zoom:50%;" />

首先，来看一下`Directory`的类图，`Directory`是顶层的接口。`AbstractDirectory`封装了通用的实现逻辑。抽象类包含`RegistryDirectory`和`StaticDirectory`两个子类。

- `AbstractDirectory`：封装了通用逻辑，
- `RegistryDirectory`：`Directory`的动态列表实现，会自动从注册中心更新`Invoker`列表、配置信息、路由列表。
- `StaticDirectory`：`Directory`的静态列表实现，即将传入的`Invoker`列表封装成静态的`Directory`对象，里面的列表不会改变。

# 源码解析

## AbstractDirectory#list方法

`AbstractDirectory#list`方法主要包含以下逻辑：

1. 调用子类的`doList`方法获取所有`Invoker`列表
2. 遍历所有路由列表，过滤`Invoker`，返回过滤之后的`Invoker`

代码如下所示：

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed .url: " + getUrl());
    }
    //调用子类实现的doList方法获取可用的Invoker列表
    List<Invoker<T>> invokers = doList(invocation);

    //获取路由列表
    List<Router> localRouters = this.routers; // local reference
    //如果路由列表不为空，则遍历路由列表，进行路由
    if (localRouters != null && !localRouters.isEmpty()) {
        for (Router router : localRouters) {
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
    }
    return invokers;
}
//模板方法
protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;
```

## StaticDirectory

`StaticDirectory#doList`方法逻辑比较简单，因为其`Invoker`列表是不变的，所以该方法会直接返回传入的`Invoker`列表。代码如下所示：

```java
protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
    return invokers;
}
```

## RegistryDirectory

> `RegistryDirectory`是一种动态服务目录，实现了`NotifyListener`接口。当注册中心服务配置发生变化后，`RegistryDirectory`可收到与当前服务相关的变化。收到变更通知后，`RegistryDirectory`可根据配置变更信息刷新`Invoker`列表。

`RegistryDirectory`中有两条比较重要的逻辑线，第一条，子类实现父类的`doList`方法；第二条，框架与注册中心的订阅，并动态更新本地`Invoker`列表、路由列表、配置信息的逻辑。

### doList方法

首先我们来看一下`doList`方法，该方法主要用来从本地缓存里获取`Invokers`列表，代码如下所示：

```java
public List<Invoker<T>> doList(Invocation invocation) {
  	//当没有invoker时，该值会被置为true
    if (forbidden) {
        //省略noProvider异常抛出
    }
    List<Invoker<T>> invokers = null;
    //获取本地缓存，方法对应的Invoker列表
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        //方法名
        String methodName = RpcUtils.getMethodName(invocation);
        //参数
        Object[] args = RpcUtils.getArguments(invocation);
        //第一个参数是否为String或者enum类型，通过方法名加第一个参数查询Invokers列表
        if (args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // The routing can be enumerated according to the first parameter
        }
        //通过方法名查询Invokers列表
        if (invokers == null) {
            invokers = localMethodInvokerMap.get(methodName);
        }
        //通过*查询Invokers列表
        if (invokers == null) {
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        //遍历map，取第一个
        if (invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

可以看到，`doList`方法主要是从本地缓存`methodInvokerMap`里获取`Invoker`列表，那么本地缓存是何时更新的呢？

### notify方法

`RegistryDirectory`实现了`NotifyListener`接口，通过`notify`这个方法获取注册中心变更通知。该方法主要包含以下逻辑：

1. 根据`url`的`category`和`protocol`对`url`进行分类
2. 将对应的列表转成`Router`和`Configurator`列表
3. 刷新`Invoker`列表

代码如下所示：

```java
public synchronized void notify(List<URL> urls) {
    //provider url, 路由url, 配置器url
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    //遍历urls列表，通过protocol和category将url放入不同的列表
    for (URL url : urls) {
        String protocol = url.getProtocol();
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        if (Constants.ROUTERS_CATEGORY.equals(category)
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            invokerUrls.add(url);
        } //省略日志输出代码
    }
    //将configuratorUrls里的url转成configurators
    if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
        this.configurators = toConfigurators(configuratorUrls);
    }
    //将routerUrls里的url转成routers
    if (routerUrls != null && !routerUrls.isEmpty()) {
        List<Router> routers = toRouters(routerUrls);
        if (routers != null) { // null - do nothing
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // merge override parameters
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && !localConfigurators.isEmpty()) {
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    //刷新Invoker列表
    refreshInvoker(invokerUrls);
}
```

### refreshInvoker方法

接下来看一下`refreshInvoker`方法，该方法主要包含以下逻辑：

1. 判断是否禁用所有服务
2. 将`url`转成`Invoker`，再转成方法名对应的`Invoker`列表
3. 缓存第2步结果到`methodInvokerMap`中
4. 销毁无用的`Invoker`

代码如下所示：

```java
private void refreshInvoker(List<URL> invokerUrls) {
    //如果invokerUrls列表只有一条记录且协议头=empty，则表示禁用所有服务
    if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
            && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        //这个标志在doList方法的开头就用到了，如果为true，doList直接抛NoProvider异常
        this.forbidden = true; // Forbid to access
        //将缓存设置为空，并销毁所有Invokers
        this.methodInvokerMap = null; // Set the method invoker map to null
        destroyAllInvokers(); // Close all invokers
    }
    //正常的情况
    else {
        this.forbidden = false; // Allow to access
        Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
        //缓存invokerUrls
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            this.cachedInvokerUrls = new HashSet<URL>();
            this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
        }
        if (invokerUrls.isEmpty()) {
            return;
        }
        //将url转成Invoker
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map
        //转成方法名->Invoker列表
        Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // Change method name to map Invoker Map
        //转换出错了
        if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
            //省略日志代码
            return;
        }
        //合并多个组的Invoker
        this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
        this.urlInvokerMap = newUrlInvokerMap;
        try {
            //销毁无用的Invoker
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
        } //省略异常抛出代码
    }
}
```

关于`refreshInvoker`方法中所调用的方法此处不做分析。

# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)