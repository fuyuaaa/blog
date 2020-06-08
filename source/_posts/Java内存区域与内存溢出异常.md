---
title: Java内存区域与内存溢出异常
date: 2020-06-02 17:55:00
tags: [JVM]
categories: JVM
---

## 运行时数据区

根据《Java虚拟机规范》的规定，Java虚拟机所管理的内存将会包括以下几个运行时数据区域。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/Java虚拟机运行时数据区.jpg" alt="Java虚拟机运行时数据区" style="zoom:50%;" />

### 程序计数器（PC寄存器）

**程序计数器**是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，它是程序控制流的指示器，分支、循环、跳转、异常处 理、线程恢复等基础功能都需要依赖这个计数器来完成。

- 每一条Java虚拟机线程都有自己的程序计数器，线程私有。
- 此内存区域是唯一一个在《Java虚拟机规范》中没有规定任何OutOfMemoryError情况的区域。

### Java虚拟机栈

**虚拟机栈**描述的是Java方法执行的线程内存模型：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧用于存储局部变量表、操作数栈、动态连接、方法出口等信息。每一个方法被调用直至执行完毕的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

- 线程私有。
- 如果线程请求的栈深度大于虚 拟机所允许的深度，将抛出StackOverflowError异常。
- 如果Java虚拟机栈容量可以动态扩展（HotSpot不支持），当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。

#### 运行时栈帧结构

栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/栈帧结构.png" alt="栈帧结构" style="zoom:50%;" />

### 本地方法栈

与Java虚拟机栈基本类似。区别只是Java虚拟机栈执行的是Java方法，而本地方法栈执行的是native方法。（HotSpot中直接把这俩搞成一个）

- 线程私有

### Java堆

在Java虚拟机中，**堆**是可供各个线程共享的运行时内存区域，也是供所有类实例对象和数组对象分配内存的地方。

- 线程共享，随虚拟机启动创建。
- 如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展（应该指的是从-Xms扩展到-Xmx的过程，不过一般为了避免自动扩展，这两值会搞成一样）时，Java虚拟机将会抛出OutOfMemoryError异常。

### 方法区

**方法区**用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。

- 线程共享。
- 根据《Java虚拟机规范》的规定，如果方法区无法满足新的内存分配需求时，将抛出 OutOfMemoryError异常。

ps:

- 在JDK 8以前，HotSpot虚拟机设计团队说使用永久代来实现方法区。
- 到了JDK 7的HotSpot，已经把原本放在永久代的字符串常量池、静态变量等移出（在堆里）。
- 而到了 JDK 8，终于完全废弃了永久代的概念，改用元空间（Metaspace）来代替，把JDK 7中永久代还剩余的内容（主要是类型信息）全部移到元空间中。

#### 运行时常量池

**运行时常量池**是方法区的一部分。Class文件中除了有类的版本、字 段、方法、接口等描述信息外，还有一项信息是常量池表，用于存放编译期生成的各种**字面量**与**符号引用**，这部分内容将在类加载后存放到方法区的运行时常量池中。

- 关于**符号引用** -> [链接]([http://fuyuaaa.top/2020/06/01/%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/#%E8%A7%A3%E6%9E%90](http://fuyuaaa.top/2020/06/01/虚拟机的类加载机制/#解析))



## HotSpot虚拟机对象探秘

### 对象创建

当Java虚拟机遇到一条字节码**new指令**时：

- 首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，先执行相应的类加载过程。
- 为新生对象分配内存空间。
- 将内存空间初始化为零值。
- 设置对象头。
- 执行init方法。

#### 分配内存空间的两种方式

**指针碰撞**：假设Java堆中内存是**绝对规整**的，所有被使用过的内存都被放在一边，空闲的内存被放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离。

**空闲列表**：但如果Java堆中的内存**并不是规整**的，已被使用的内存和空闲的内存相互交错在一起，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录。

选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的**垃圾收集器是否带有空间压缩整理（Compact）的能力**决定。

#### 分配内存时的并发问题

对象创建在虚拟机中是非常频繁的行为，即使仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的。解决这个问题 有两种可选方案：

1. 对分配内存空间的动作进行**同步处理**——实际上虚拟机是采用**CAS配上失败重试**的方式保证更新操作的原子性；
2. 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为**本地线程分配缓冲（TLAB）**，哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完 了，分配新的缓存区时才需要同步锁定。（虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来 设定）

### 对象的内存布局

在HotSpot虚拟机里，**对象在堆内存中的存储布局**可以划分为三个部分：**对象头、实例数据和对齐填充**。

**对象头**又由两部分构成：

1. 用于存储对象自身的运行时数据，如HashCode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。
2. 类型指针，即对象指向它的类型元数据的指针。

**实例数据部分**是对象真正存储的有效信息。

**对齐填充**，并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。

### 对象的访问定位

使用对象的时候，Java程序会通过栈上的reference数据来操作堆上的具体对象。

主流的访问方式主要有使用**句柄**和**直接指针**两种。HotSpot主要使用直接指针的方式。

使用**句柄**访问，Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息。 

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/通过句柄访问对象.jpg" alt="通过句柄访问对象" style="zoom:50%;" />

使用**直接指针**访问，Java堆中对象的内存布局就必须考虑如何放置访问类型数据的相关信息，reference中存储的直接就是对象地址，如果只是访问对象本身的话，就不需要多一次间接访问的开销。

![通过直接指针访问对象](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/通过直接指针访问对象.jpg)



## OOM异常

### 模拟堆OOM

```java
/**
 * test堆内存OOM
 * 设置堆内存 10m
 * -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
 */
@Test
public void testHeapOOM() {
  List<Object> list = new ArrayList<>();
  while (true) {
    list.add(new Object());
    System.out.println(Runtime.getRuntime().freeMemory() / 1024 / 1024 + "MB");
  }
}

result
======
六月 02, 2020 4:28:21 下午 org.junit.platform.launcher.core.DefaultLauncher handleThrowable
警告: TestEngine with ID 'junit-jupiter' failed to execute tests
java.lang.OutOfMemoryError: Java heap space
```

使用JProfiler查看dump文件发现Object对象非常多。

![HeapOOMDumpResult](https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/HeapOOMDumpResult.png)

### 模拟栈StackOverflowError

```java
/**
 * 栈内存OOM
 * 设置栈大小 128k
 * -Xss128k
 */
@Test
public void testStackOOM() {
  try {
    stackLeak();
  } catch (Throwable e) {
    System.out.println("stack length:" + stackLength);
    throw e;
  }
}

private int stackLength = 1;

public void stackLeak() {
  //记录栈深度
  stackLength++;
  stackLeak();
}

result
======
stack length:17897

java.lang.StackOverflowError
	at top.fuyuaaa.study.jvm.OOMTest.stackLeak(OOMTest.java:48)
```



## 参考

《[深入理解Java虚拟机](https://book.douban.com/subject/34907497/)》《[Java虚拟机规范](https://docs.oracle.com/javase/specs/index.html)》

图片来源《[深入理解Java虚拟机](https://book.douban.com/subject/34907497/)》

