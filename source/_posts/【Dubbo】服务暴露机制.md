---
title: 【Dubbo】服务暴露机制
date: 2020-09-10 18:00:00
tags: [dubbo]
categories: dubbo



---

<!-- toc -->

# 概述

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200910135718657.png" alt="服务暴露整体流程" style="zoom: 33%;" />

首先，看一下服务暴露的整体流程，如上图所示。`Dubbo`框架将服务暴露分为两大部分，第一步将持有的服务实例通过代理转换成`Invoker`，第二步会把`Invoker`通过具体的协议（比如`Dubbo` ）转换成`Exporter`。接下来会具体分析一下每个步骤的流程以及源码。

# 源码解析

## ServiceConfig类

### ServiceBean类

在看`ServiceConfig`之前，先来看一下其子类`ServiceBean`的`onApplicationEvent`方法，该方法监听了`Spring`容器的上下文刷新事件，当收到该事件时会触发服务的暴露工作，具体代码如下所示：

```java
@Override
//添加了上下文刷新监听，用于暴露服务使用。
public void onApplicationEvent(ContextRefreshedEvent event) {
    //是否延迟导出 && 是否已经导出 && 是不是已经取消导出
    if (isDelay() && !isExported() && !isUnexported()) {
        //省略日志输出代码
        export();
    }
}
```

至于`ServiceBean`的话，是通过配置或者注解解析出来的Bean，本文不做解析。

### export方法

接下来看一下`ServiceConfig#export`方法，该方法主要对`export`和`delay`配置进行检查，如果`export=false`就不做暴露，如果`delay=true`则进行延迟暴露。

```java
public synchronized void export() {
    if (provider != null) {
        //获取export和delay配置。
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    if (export != null && !export) {
        return;
    }
    //延迟暴露
    if (delay != null && delay > 0) {
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
    } else {
        //暴露服务
        doExport();
    }
}
```

接下来看一下`ServiceConfig#doExport`方法，该方法主要是进行配置检查，设置缺省值或者抛出异常等。配置检查部分本文不做解析，主要看一下该方法中调用的`ServiceConfig#doExportUrls`方法。

### doExportUrls方法

`ServiceConfig#doExportUrls`方法会加载注册中心的`URL`，然后遍历协议列表，在每个协议下面都导出服务。代码如下所示：

```java
//多协议多注册中心导出。
private void doExportUrls() {
    //加载注册中心URL
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        // 遍历 protocols，并在每个协议下导出服务
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

### loadRegistries方法

`loadRegistries`方法主要包含以下逻辑：

1. 检测是否存在注册中心配置类，不存在则抛出异常
2. 遍历注册中心配置类，构建注册中心`URL`列表
3. 判断是否加入到`registryList`

代码如下所示：

```
protected List<URL> loadRegistries(boolean provider) {
    //检查是否存在注册中心配置类，不存在则抛出异常。
    checkRegistry();
    List<URL> registryList = new ArrayList<URL>();
    if (registries != null && !registries.isEmpty()) {
        for (RegistryConfig config : registries) {
            String address = config.getAddress();
            //默认值
            if (address == null || address.length() == 0) {
                address = Constants.ANYHOST_VALUE;
            }
            //配置值
            String sysaddress = System.getProperty("dubbo.registry.address");
            if (sysaddress != null && sysaddress.length() > 0) {
                address = sysaddress;
            }
            if (address.length() > 0 && !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
            		//将构建URL所需的参数加入到map里。
                Map<String, String> map = new HashMap<String, String>();
                appendParameters(map, application);
                appendParameters(map, config);
                map.put("path", RegistryService.class.getName());
                map.put("dubbo", Version.getProtocolVersion());
                map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
                if (ConfigUtils.getPid() > 0) {
                    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
                }
                if (!map.containsKey("protocol")) {
                    if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
                        map.put("protocol", "remote");
                    } else {
                        map.put("protocol", "dubbo");
                    }
                }

                //解析得到URL列表，address可能包含多个注册中心ip
                List<URL> urls = UrlUtils.parseURLs(address, map);
                for (URL url : urls) {
                    url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
                    //将URL协议头设置为registry
                    url = url.setProtocol(Constants.REGISTRY_PROTOCOL);

                    //(服务提供者 && 注册) || (非服务提供者 && 订阅)
                    if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
                            || (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
                        registryList.add(url);
                    }
                }
            }
        }
    }
    return registryList;
}
```

### doExportUrlsFor1Protocol方法

`doExportUrlsFor1Protocol`方法主要包含以下逻辑：

1. 组装`URL`
2. 导出服务

这里主要分析一下导出服务的代码，导出服务部分代码主要包含以下逻辑：

1. 如果`scope!=remote`，则导出服务到本地
2. 如果`scope!=local`，则导出服务到远程
   1. 遍历注册中心`URL`，依次导出
3. 其实一般来说`scope`是为空的，上面两个都会执行

代码如下所示：

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    //省略无关代码
    // don't export when none is configured
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        // scope != remote，导出到本地
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // scope != local，导出到远程
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            //省略日志输出代码
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));

                    //加载监视器链接，将监视器链接作为参数添加到 url 中
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    //省略日志输出代码

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }

                    //生成invoker
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    //DelegateProviderMetaDataInvoker用于持有Invoker和ServiceConfig
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    //导出服务，将exporter加入到exporters
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            }
            // 不存在注册中心，仅导出服务
            else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```

### exportLocal方法

`exportLocal`方法用于导出本地服务，主要包含以下逻辑：

1. 判断当前协议是否已经是`injvm`，如果是不作处理
2. 设置`URL`的协议、`host`、`port`为本地
3. 创建`invoker`，导出服务

```java
private void exportLocal(URL url) {
    //判断当前协议是否为injvm，如果是就无需再次导出了。
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                //将协议设置成injvm，host设置为本地，port=0
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(LOCALHOST)
                .setPort(0);
        StaticContext.getContext(Constants.SERVICE_IMPL_CLASS).put(url.getServiceKey(), getServiceClass(ref));
        //创建invoker，导出服务
        //这里的protocol是一个自适应扩展类，会根据url的协议去调用对应扩展类的export方法
      	//此处调用的是InjvmProtocol#export方法
        Exporter<?> exporter = protocol.export(proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
    }
}
```

### getInvoker方法

`getInvoker`是`ProxyFactor`接口提供的，默认的实现类是`JavassistProxyFactory`，`JavassistProxyFactory#getInvoker`方法主要包含以下逻辑：

1. 创建包装类
2. 创建`invoker`，对`invoker`的方法调用最终会调用到包装类的`invokeMethod`

代码如下所示：

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    //创建包装类
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    //创建匿名invoker类，实现doInvoke方法，doInvoke方法是一个抽象法法
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

`getWrapper`方法主要逻辑是对传入的类进行解析，生成`invokeMethod`代码以及一些其他代码，然后通过`Javassist`生成`class`对象并通过反射创建实例，具体代码不做解析。

## Protocol

### RegistryProtocol#export

`RegistryProtocol#export`方法主要包含以下逻辑：

1. 委托具体协议(`Dubbo`)进行服务暴露，创建`NettyServer`监听端口和保存服务实例。 
2. 创建注册中心对象，与注册中心创建`TCP`连接。 
3. 注册服务元数据到注册中心。 
4. 订阅`configurators`节点，监听服务动态属性变更事件。 
5. 服务销毁收尾工作，比如关闭端口、反注册服务信息等

代码如下所示:

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //本地暴露服务，这个方法会取出export参数的值作为url，并根据url的协议调用对应的protocol(比如dubbo)进行export
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

    //获取注册中心URL，以zookeeper注册中心为例，得到的示例 URL 如下：
    //zookeeper://localhost:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F172.18.39.113%3A20880%2Ftop.fuyuaaa.api.TestService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bean.name%3Dtop.fuyuaaa.api.TestService%26bind.ip%3D172.18.39.113%26bind.port%3D20880%26dubbo%3D2.0.2%26generic%3Dfalse%26interface%3Dtop.fuyuaaa.api.TestService%26methods%3Ddemo%26pid%3D72080%26qos.port%3D22222%26revision%3D1.0%26side%3Dprovider%26timestamp%3D1599729148679%26transporter%3Dnetty%26version%3D1.0&pid=72080&qos.port=22222&timestamp=1599729148633
    URL registryUrl = getRegistryUrl(originInvoker);

    //根据URL加载Registry实现类，比如 ZookeeperRegistry
    final Registry registry = getRegistry(originInvoker);

    //获取已注册的服务提供者URL，比如：
    //dubbo://172.18.39.113:20880/top.fuyuaaa.api.TestService?anyhost=true&application=demo-provider&bean.name=top.fuyuaaa.api.TestService&dubbo=2.0.2&generic=false&interface=top.fuyuaaa.api.TestService&methods=demo&pid=72080&revision=1.0&side=provider&timestamp=1599729148679&transporter=netty&version=1.0
    final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

    //to judge to delay publish whether or not
    //获取 register 参数
    boolean register = registeredProviderUrl.getParameter("register", true);

    //向服务提供者与消费者注册表中注册服务提供者
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

    if (register) {
        //向注册中心注册服务
        register(registryUrl, registeredProviderUrl);
        ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
    }

    //获取订阅 URL，比如：
    //provider://172.18.39.113:20880/top.fuyuaaa.api.TestService?anyhost=true&application=demo-provider&bean.name=top.fuyuaaa.api.TestService&category=configurators&check=false&dubbo=2.0.2&generic=false&interface=top.fuyuaaa.api.TestService&methods=demo&pid=72080&revision=1.0&side=provider&timestamp=1599729148679&transporter=netty&version=1.0
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
    //创建监听器，监听服务接口下configurators节点，用于处理动态配置，比如dubbo-admin对集群的服务治理。
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

    //向注册中心进行订阅override数据
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //创建并返回 DestroyableExporter
    return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
}
```

### DubboProtocol#export

`DubboProtocol#export`方法主要包含以下逻辑：

1. 缓存`service`的`key`和`exporter`
2. 创建服务器

代码如下所示：

```
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    //缓存key和对应的export
    //🌰：top.fuyuaaa.api.TestService:1.0:20880
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //省略本地存根相关代码
    
    //创建服务器实例
    openServer(url);
    //优化序列化
    optimizeSerialization(url);
    return exporter;
}

private void openServer(URL url) {
    // find server.
    // 获取 host:port，并将其作为服务器实例的 key，用于标识当前的服务器实例
    String key = url.getAddress();
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            serverMap.put(key, createServer(url));
        } else {
            // 服务器已创建，则根据 url 中的配置重置服务器
            server.reset(url);
        }
    }
}
```

关于服务注册以及创建`netty`服务器具体流程不做解析。

# 流程图



![dubbo-服务暴露](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/dubbo-服务暴露.png)



# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)

