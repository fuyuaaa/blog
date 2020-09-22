title: Mybatis源码-日志模块
date: 2020-07-09 12:00:00
tags: [mybatis源码]
categories: mybatis源码

-------------

在 Java 开发中常用的日志框架有` Log4j、 Log4j2、 Apache Commons Log、 java.util.logging、 slf4j`等，这些工具对外的接口不尽相同。为了统一这些工具的接口，MyBatis 定义了一套统一的日志接口供上层使用 ，并为上述常用的日志框架提供了相应的适配器（适配器模式）。

MyBatis 的日志模块位于`org.apache.ibatis.logging`包中，该模块中通过 Log 接口定义了日志模块的功能 ，当然日志适配器也会实现此接口。Log接口的实现类（适配器）如下图。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200709112706437.png" alt="日志适配器" style="margin-left: 10px" />

`LogFactory`工厂类负责创建对应的日志组件适配器，其逻辑是按序加载并实例化对应日志组件的适配器，然后使用`LogFactory. logConstructor`这个静态宇段，记录当前使用的第三方日志组件的适配器。

```java
//记录当前使用的第三方日志组件所对应的适配器的构造方法
private static Constructor<? extends Log> logConstructor;
static {
  //下面会针对每种日志组件调用 tryimplementation()方法进行尝试加载
  tryImplementation(LogFactory::useSlf4jLogging);
  tryImplementation(LogFactory::useCommonsLogging);
  tryImplementation(LogFactory::useLog4J2Logging);
  tryImplementation(LogFactory::useLog4JLogging);
  tryImplementation(LogFactory::useJdkLogging);
  tryImplementation(LogFactory::useNoLogging);
}
private static void tryImplementation(Runnable runnable) {
  if (logConstructor == null) {
    try {
      runnable.run();
    } catch (Throwable t) {
      // ignore
    }
  }
}
```

`trylmplementation()`方法首先会检测`logConstructor`字段 ，若为空则调用`Runnable.run()`方法。

比如`LogFactory::useJdkLogging`

```java
public static synchronized void useJdkLogging() {
  setImplementation(org.apache.ibatis.logging.jdk14.Jdk14LoggingImpl.class);
}
private static void setImplementation(Class<? extends Log> implClass) {
  try {
    //获取指定适配器的构造方法
    Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
    //实例化适配器
    Log log = candidate.newInstance(LogFactory.class.getName());
    //初始化 logConstructor 字段
    logConstructor = candidate;
  } catch ...
}

```

`Jdkl4Loggingimpl`使用`java.util.logging.Logger`实现了Log接口的方法。代码如下，实现了`org.apache.ibatis.logging.Log`接口的其他适配器也与`Jdkl4Logginglmpl`类似。

```java
public class Jdk14LoggingImpl implements Log {
  private final Logger log;
  public Jdk14LoggingImpl(String clazz) {
    log = Logger.getLogger(clazz);
  }
	//...省略isDebugEnabled，isTraceEnabled
  @Override
  public void error(String s) {
    log.log(Level.SEVERE, s);
  }

  @Override
  public void debug(String s) {
    log.log(Level.FINE, s);
  }

  @Override
  public void trace(String s) {
    log.log(Level.FINER, s);
  }

  @Override
  public void warn(String s) {
    log.log(Level.WARNING, s);
  }

}
```

在 MyBatis的日志模块中有一个Jdbc包，它并不是将日志信息通过 JDBC 保存到数据库中， 而是通过 JDK 动态代理的方式，将 JDBC 操作通过指定的日志框架打印出来 。这个功能通常在 开发阶段使用 ，它可以输出 SQL 语句、用户传入的绑定参数、 SQL 语句影响行数等等信息，对调试程序来说是非常重要的。

`BaseJdbcLogger`是一个抽象类，它是 Jdbc 包下其他 Logger 类的父类，继承关系如下图。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200709131549791.png" alt="BaseJdbcLogger继承关系" style="margin-left:10px" />

