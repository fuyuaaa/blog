

本文主要是结合前面的文章知识，分析下一次Dubbo调用的具体流程。话不多说，测试代码如下所示：

```java
public interface TestService {
    String demo(String words);
}
public class Consumer {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"/dubbo-demo-consumer.xml"});
        context.start();
        TestService testService = (TestService) context.getBean("testService");
        String hello = testService.demo("world111");
        System.out.println(hello);
        }
    }
}
```

```xml
<dubbo:registry address="zookeeper://localhost:2181" />
<dubbo:reference version="1.0" id="testService" interface="top.fuyuaaa.api.TestService"/>
```

首先对testService.demo("world111")进行debug，stepInto进入，发现其调用了com.alibaba.dubbo.common.bytecode.proxy0#demo，由于这个类是代理类，debug不到里面的过程，这里直接arthas jad一下他的代码，proxy0这个代理类实现了TestService，如下所示：

```
//省略注释和import
public class proxy0 implements ClassGenerator.DC, TestService, EchoService {
    public static Method[] methods;
    private InvocationHandler handler;

    public String demo(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[0], arrobject);
        return (String) object;
    }

    //省略构造方法和echo方法
}
```

继续debug，进入到InvokerInvocationHandler#invoke方法，该方法主要做的事情就是把要调用的方法和参数封装到RpcInvocation对象中，然后调用MockClusterInvoker#invoke方法。

![InvokerInvocationHandler#invoke方法](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200922181425970.png)

MockClusterInvoker#invoke方法主要做一些mock操作以及服务降级操作，如果不需要mock操作，则会直接调用Invoker#invoke方法，此处Invoker默认类型为FailoverClusterInvoker（重试容错机制）。

![MockClusterInvoker#invoke方法](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200922181921854.png)

由之前【Dubbo】集群容错机制文章可知，ClusterInvoker的的invoke方法时交于AbstractClusterInvoker实现的，AbstractClusterInvoker#invoke方法会先调用list方法获取Invoker列表，再调用子类实现的doInvoke，此处则是FailoverClusterInvoker#doInvoke方法。

![AbstractClusterInvoker#invoke方法](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200922182424533.png)

FailoverClusterInvoker#doInvoke方法会先调用AbstractClusterInvoker#select方法并且通过负载均衡策略选取一个可用的Invoker，然后进行调用。

![FailoverClusterInvoker#doInvoke方法](/Users/fuyuaaa/Library/Application Support/typora-user-images/image-20200922182758036.png)

此处RegistryDirectory$InvokerDelegate是RegistryDirectory的内部类，该类继承了InvokerWrapper类，所以最终会使用InvokerWrapper#invoke方法。

![image-20200922185207091](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200922185207091.png)

可以看到该类持有的Invoker实际类型为ProtocolFilterWrapper，这里主要做过滤器的操作，在经过Filter操作之后会调用到DubboInvoker#invoke方法，然后会调用DubboInvoker#doInvoke。







待续...