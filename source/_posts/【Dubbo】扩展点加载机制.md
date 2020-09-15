---
title: 【Dubbo】扩展点加载机制
date: 2020-09-10 10:00:00
tags: [dubbo]
categories: dubbo


---

<!-- toc -->

# SPI

## 简介

`SPI` 的全称是`Service Provider Interface`，是一种服务提供发现机制。

## Java SPI

`Java SPI`使用了策略模式，一个接口多种实现。我们只声明接口，具体的实现并不在程序中直接确定，而是由程序之外的配置掌控，用于具体实现的装配。具体步骤如下：

1. 定义一个接口及对应的方法。
2. 编写该接口的一个实现类。
3. 在`META-INF/services/`目录下，创建一个以接口全路径命名的文件，如`com.test.spi.PrintService0`
4. 文件内容为具体实现类的全路径名，如果有多个，则用分行符分隔。
5. 在代码中通过`java.util.ServiceLoader`来加载具体的实现类。 

### Java SPI 示例
```java
service
public interface PrintService {
    void print();
}

实现类1
public class ChinesePrintServiceImpl implements PrintService {
    @Override
    public void print() {
        System.out.println("中文");
    }
}
实现类2
public class EnglishPrintServiceImpl implements PrintService {
    @Override
    public void print() {
        System.out.println("English");
    }
}

文件
META-INF/services/top.fuyuaaa.spidemo.PrintService
文件内容
top.fuyuaaa.spidemo.javaspi.ChinesePrintServiceImpl
top.fuyuaaa.spidemo.javaspi.EnglishPrintServiceImpl

测试类
public class JavaSPIDemo {
    public static void main(String[] args) {
        ServiceLoader<PrintService> printServices = ServiceLoader.load(PrintService.class);
        printServices.forEach(PrintService::print);
    }
}

结果
中文
English


```

## Dubbo SPI

与`Java SPI`相比，`Dubbo SPI`做了一定的改进和优化。`Dubbo SPI` 改进了`Java SPI`以下几个问题。

> 1. JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> 2. 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
> 3. Dubbo增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。

### Dubbo SPI 示例
```java
@SPI("chinese")
public interface PrintService {
    void print();
}

文件
META-INF/dubbo/top.fuyuaaa.spidemo.PrintService
文件内容
chinese=top.fuyuaaa.spidemo.javaspi.ChinesePrintServiceImpl
english=top.fuyuaaa.spidemo.javaspi.EnglishPrintServiceImpl

测试类
public class DubboSPIDemo {
    public static void main(String[] args) {
        ExtensionLoader<PrintService> extensionLoader = ExtensionLoader.getExtensionLoader(PrintService.class);
        PrintService defaultPrintService = extensionLoader.getDefaultExtension();
        defaultPrintService.print();

    }
}
结果
中文
```

### Dubbo SPI 特性

#### 扩展点自动包装

`ExtensionLoader`在加载扩展时，如果扩展类存在 一个参数为扩展点构造函数，则该类会被识别成`Wrapper`（包装）类。

🌰如下：`PrintServiceWrapper`实现了扩展点`PrintService`，也与其他扩展类进行了一样的配置，但是`PrintServiceWrapper`有一个参数为`PrintService`的构造函数，此时`PrintServiceWrapper`会被识别称为一个`Wrapper`类，包装在真正的扩展点之外，有点类似`AOP`。当通过`ExtensionLoader.getExtensionLoader(PrintService.class)`获取扩展类时，得到的其实是包装类`PrintServiceWrapper`，并且包装类可以存在多个，如【包装类[包装类(扩展类)]】。

```java
public class PrintServiceWrapper implements PrintService {
    private final PrintService printService;
    public PrintServiceWrapper(PrintService printService) {
        this.printService = printService;
    }
    @Override
    public void print() {
        System.out.println("做一下前置工作");
        printService.print();
        System.out.println("做一下后置工作");
    }
}

在文件
META-INF/dubbo/top.fuyuaaa.spidemo.PrintService
添加内容
wrapperTest=top.fuyuaaa.spidemo.PrintServiceWrapper
```

在加载扩展类时，会判断是否为包装类，如果是包装类，则会将其加入到`cachedWrapperClasses`。

`ExtensionLoader#loadClass`代码如下所示：

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
  //省略无关代码
  //判断是否包装类
  else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    }
  //省略无关代码
}

private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
```

在创建扩展类时，会判断`cachedWrapperClasses`是否为空，如果不为空，会在原扩展类上进行包装，如【包装类[包装类(扩展类)]】。

`ExrensionLoader#createExtension`代码如下所示：

```java
private T createExtension(String name) {
    //省略无关代码
  			//依赖注入
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        //循环装饰，如果有多个装饰者，在生成一个装饰类之后，又会用这个装饰类生成下一个装饰类
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
              	//先生成装饰类实例，再进行装饰类的依赖注入
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    //省略无关代码
}
```



#### 扩展点自动装配

加载扩展点时，自动注入依赖的扩展点。加载扩展点时，扩展点实现类的成员如果为其它扩展点类型，`ExtensionLoader`在会自动注入依赖的扩展点。`ExtensionLoader`通过扫描扩展点实现类的所有`setter`方法来判定其成员。

自动注入依赖主要通过`ExtensionLoader#injectExtension`方法，`injectExtension`主要作用是对扩展类进行依赖注入，也就是`Dubbo IOC`。实现原理比较简单，首先通过反射获取类的所有方法，然后遍历以字符串`set`开头的方法，得到set方法的参数类型，再通过`ExtensionFactory`寻找参数类型相同的扩展类实例，如果找到，就设值进去。

代码如下所示：

```java
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
              	//遍历扩展点的方法，如果是public的 有参的 set方法，会通过set后的名字 以及 参数类型 去查询扩展点。
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        //..省略注解判断
                      	//参数类型
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                          	//参数名
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
	                          //获取依赖的对象，AdaptiveExtensionFactory.getExtension
                            Object object = objectFactory.getExtension(pt, property);
                          	//反射调用set方法设置依赖
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } //...省略catch
                    }
                }
            }
        } //...省略catch
        return instance;
    }
```



#### 扩展点自适应

扩展点自适应机制，主要做的事情就是通过`@Adaptive`注解在调用时通过`URL`参数决定调用哪一个扩展类的方法。

🌰如下：`PrintService#print(URL url)`注解了@Adaptive，并且指定参数名为`language`，然后在`DubboSPIDemo#main`方法中构建`URL`时传入了`language=english`，在调用`print(URL url)`方法时，根据参数`language`，选择了扩展类`EnglishPrintServiceImpl#print(URL url)`，因为在配置文件里指定了`english=top.fuyuaaa.spidemo.javaspi.EnglishPrintServiceImpl`

```java
@SPI("chinese")
public interface PrintService {
    @Adaptive("language")
    void print(URL url);
}

public class ChinesePrintServiceImpl implements PrintService {
    @Override
    public void print(URL url) {
        System.out.println("中文URL");
    }
}

public class EnglishPrintServiceImpl implements PrintService {
    @Override
    public void print(URL url) {
        System.out.println("English URL");
    }
}

public class DubboSPIDemo {
    public static void main(String[] args) {
      	//构建URL，指定language参数值为english
        URL url = new URL("test", "test", 8080, "test", Collections.singletonMap("language", "english"));
        PrintService adaptiveExtension = extensionLoader.getAdaptiveExtension();
        adaptiveExtension.print(url);
    }
}

结果
English URL
```

在加载自适应扩展类时，`Dubbo`会为拓展接口生成具有代理功能的代码。然后通过`javassist`或`jdk`编译这段代码，得到 `Class` 类，最后再通过反射创建代理类。如下图所示，`debug`获取到的`PrintService`其实是`PrintService$Adaptive`。

![PrintService-AdaptiveExtension](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200909145256398.png)

`arthas jad` 反编译出来的代码：在`PrintService$Adaptive#print(URL url)`方法中，会先获取参数`language`的值，然后根据该值去加载对应的扩展类，并调用其`print`方法，从而达到在调用时决定扩展类的功能。

```java
import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import top.fuyuaaa.spidemo.PrintService;

public class PrintService$Adpative implements PrintService {
    public void print(URL uRL) {
        //省略url空校验
        URL uRL2 = uRL;
      	//这里有个chinese是因为在@SPI注解里默认指定了chinese，是一个缺省值。
        String string = uRL2.getParameter("language", "chinese");
        //省略string空校验
        PrintService printService = ExtensionLoader.getExtensionLoader(PrintService.class).getExtension(string);
        printService.print(uRL);
    }
}
```



#### 扩展点自动激活

使用`@Activate`注解，可以标记对应的扩展点默认被激活启用。该注解还可以通过传入不同的参数，设置扩展点在不同的条件下被自动激活，主要的用途是`Filter`和`Listener`等。

🌰：`ProtocolFilterWrapper#buildInvokerChain`会根据指定的条件获取所有激活的扩展类，然后遍历扩展类列表，形成`Filter`的调用链。（`ProtocolFilterWrapper`是`Protocol`的包装类。）

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
  	//根据key和group条件，获取所有已激活的扩展类。
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {
              	//省略
            };
        }
    }
    return last;
}
```



# ExtensionLoader 源码解析

`ExtensionLoader`是整个扩展机制的主要逻辑类，在这个类里面实现了配置的加载、扩展类缓存、自适应对象生成等所有工作。

## getExtensionLoader 方法

`getExtensionLoader`方法的用处就是生产一个`ExtensionLoader`实例。

代码如下所示：`ExtensionLoader`类里维护了一个`static Map`用于缓存`ExtensionLoader`。`getExtensionLoader`方法先从`map`里取，取不到则创建。再来看下构造方法，主要是指定当前`ExtensionLoader`的类型，以及初始化`objectFactory`，`objectFactory`主要用处是在依赖注入时获取依赖的扩展点实例。

```
//Map<类, 类对应的ExtensionLoader>
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    //省略校验逻辑
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}

private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

## getExtension 方法

`getExtension(String name)`方法是`ExtensionLoader`里最核心的方法，实现了一个普通扩展类的加载过程。

主要流程如下所示：

![获取普通扩展类](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/获取普通扩展类.png)



1. 参数校验，处理默认情况
2. 先根据`name`从缓存里获取扩展类`holder`，获取不到则新建
3. 如果扩展类实例不存在，则创建

`getExtension(String name) `代码如下所示：

```java
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

public T getExtension(String name) {
    //name参数校验
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    //如果是true，则返回默认的扩展类
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //获取扩展类实例，如果不存在则新建。cachedInstances缓存了name对应的扩展类实例
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //创建扩展类实例，并设置到holder中
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

### createExtension方法

`createExtension(String name)`代码如下所示，

1. 从配置文件中加载所有扩展类，通过name查询扩展类
2. 根据clazz从缓存里取实例，取不到则创建
3. 依赖注入
4. 包装

```java
private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();
private T createExtension(String name) {
  	//加载所有扩展类（不进行初始化），通过name获取扩展类
    Class<?> clazz = getExtensionClasses().get(name);
   	//省略clazz空校验
    try {
      	//缓存里取实例，不存在则创建实例。
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
      	//依赖注入
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        //循环装饰，如果有多个装饰者，在生成一个装饰类之后，又会用这个装饰类生成下一个装饰类
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                //先生成装饰类实例，再进行装饰类的依赖注入
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } //省略异常处理
}
```

### getExtensionClasses方法

`getExtensionClasses()`代码如下所示，该方法用处是获取所有的的扩展类。

1. 缓存里取，取不到则通过`loadExtensionClasses`加载，并设置到缓存
2. 加载扩展类，处理默认扩展类名，从指定配置文件加载扩展类

```
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
    //获取SPI注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                //省略异常抛出代码
            }
            //设置默认扩展类的name
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    //加载指定文件夹下的配置文件 META-INF/dubbo/internal/,  META-INF/dubbo/,  META-INF/services/
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

关于`loadDirectory`方法和其内部调用的`loadResource`方法这里不贴代码了，主要功能就是将读取配置文件将`chinese=top.fuyuaaa.spidemo.javaspi.ChinesePrintServiceImpl`解析成`chinese`和对应的`className`并通过反射加载类，比较重要的是`loadResource`方法里调用的`loadClass`方法。

### loadClass方法

1. 处理自适应扩展类
2. 处理包装扩展类
3. 处理普通扩展类

代码如下所示：

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
  	//检测clazz是否是type的实现类
    if (!type.isAssignableFrom(clazz)) {
        //省略异常抛出代码
    }
    //如果扩展类被@Adaptive注解，设置cachedAdaptiveClass缓存
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            //省略异常抛出代码
        }
    }
    //如果是包装类，缓存到包装类map
    else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    }
    //普通扩展类
    else {
        //检测是否有默认的构造方法
        clazz.getConstructor();
        //如果扩展类没有指定名字，则尝试从注解@Extension中获取
        if (name == null || name.length() == 0) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                //省略异常抛出代码
            }
        }
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            //如果是自动激活的扩展类，缓存到cachedActivates
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                //缓存class和name的映射关系
                if (!cachedNames.containsKey(clazz)) {
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                //设置name和class的映射关系，extensionClasses是从loadExtensionClasses方法一直传过来的。
                if (c == null) {
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    //省略异常抛出代码
                }
            }
        }
    }
}
```

依赖注入，包装类逻辑在`Dubbo SPI`特性已经描述过了，不再解析。

## getAdaptiveExtension方法

`getAdaptiveExtension`方法是获取自适应扩展的入口方法。

1. 从缓存中获取
2. 缓存获取不到则创建

代码如下所示：

```java
//缓存自适应扩展实例
private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
public T getAdaptiveExtension() {
    //从缓存中获取自适应扩展实例，如获取不到则创建
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //创建自适应扩展实例，设置缓存
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } //省略异常抛出代码
                }
            }
        } //省略异常抛出代码
    }

    return (T) instance;
}
```

### createAdaptiveExtension方法

1. 获取自适应扩展类并实例化
2. 依赖注入

代码如下所示：

```java
private T createAdaptiveExtension() {
    try {
        //获取自适应扩展类，实例化，依赖注入
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } //省略异常抛出代码
}
```

### getAdaptiveExtensionClass方法

1. 获取所有扩展类
2. 从缓存中获取，如果存在则直接返回（有两种情况下会存在缓存）
   1. 手动编码：在`loadClass`方法里有个逻辑，如果扩展类被`@Adaptive`注解，则会被认为是自适应扩展类，并缓存到`cachedAdaptiveClass`。
   2. 自动生成：如果缓存中不存在，会自动创建自适应扩展类（自动生成代码，编译并加载），并缓存。
3. 不存在则自动创建自适应扩展类

代码如下所示：

```
private Class<?> getAdaptiveExtensionClass() {
    //获取所有扩展类
    getExtensionClasses();
    //如果缓存存在，则直接返回
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //缓存不存在，创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

### createAdaptiveExtensionClass方法

1. 创建自适应扩展类的代码
2. 获取类加载器，编译器
3. 编译

代码如下所示：

```java
private Class<?> createAdaptiveExtensionClass() {
  	//创建自适应扩展类的代码
    String code = createAdaptiveExtensionClassCode();
  	//获取类加载器
    ClassLoader classLoader = findClassLoader();
  	//获取编译器
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
  	//编译
    return compiler.compile(code, classLoader);
}
```

`createAdaptiveExtensionClassCode`方法主要就是将代码拼接起来，具体逻辑不再解析，生成的代码可见扩展点自适应小节通过`arthas`反编译出来的`$Adaptive`类。

# 参考

[Dubbo官网](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html)

[《深入理解Apache Dubbo与实战》](https://book.douban.com/subject/34455777/)