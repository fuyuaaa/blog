---
title: Java反射
date: 2019-01-24 21:22:21
tags: Java基础
categories: Java基础
---

Java反射机制在程序运行时，对于任意一个类，都能知道这个类的所有属性和方法，对于任意一个对象，都能调用它的任意一个方法和属性。这种动态获取信息和动态调用对象方法的功能被称为Java的反射机制。

## 获取类信息

### 获取&操作Class
```
@Test
public void testGetClass() throws Exception {
    //获取class
    Class<ReflectTest> reflectTestClass = ReflectTest.class;
    //通过类名获取class
    reflectTestClass = (Class<ReflectTest>) Class.forName("top.fuyuaaa.study.basic.reflect.ReflectTest");
    //通过对象获取class
    reflectTestClass = (Class<ReflectTest>)reflectTestClass.newInstance().getClass();
    //通过class创建对象
    ReflectTest reflectTest = reflectTestClass.newInstance();
}
```
```
@Test
public void testOperateClass() throws Exception {
    //获取class
    Class<ReflectTest> reflectTestClass = ReflectTest.class;

    
    //获取全限定名top.fuyuaaa.study.basic.reflect.ReflectTest
    String className = reflectTestClass.getName();
    //获取类名ReflectTest
    String classSimpleName = reflectTestClass.getSimpleName();
    //获取实现的接口
    Class<?>[] interfaces = reflectTestClass.getInterfaces();
    //获取父类, 没有父类默认是Object.class
    Class<? super ReflectTest> superclass = reflectTestClass.getSuperclass();


    //获取所有构造方法
    Constructor<?>[] declaredConstructors = reflectTestClass.getDeclaredConstructors();
    //获取指定参数的构造方法(不管啥修饰的)
    Constructor<ReflectTest> declaredConstructor = reflectTestClass.getDeclaredConstructor();

    
    //获取所有 公共 构造方法
    Constructor<?>[] constructors = reflectTestClass.getConstructors();
    //获取指定参数的 公共 构造方法(onlt public)
    Constructor<ReflectTest> constructor = reflectTestClass.getConstructor();
    
    
    //通过构造方法创建对象
    ReflectTest reflectTest = constructor.newInstance();
}
```

### 获取类属性

```
//省略get,set
private String privateParam;
public String publicParam;
String defaultParam;
@Test
public void testGetAndOperateField(){
    //获取class
    Class<ReflectTest> reflectTestClass = ReflectTest.class;


    //获取所有属性
    Field[] declaredFields = reflectTestClass.getDeclaredFields();
    //获取public属性
    Field[] field = declaredFields = reflectTestClass.getFields();

}
```
### 获取 方法&参数

```
@Test
public void testGetAndOperateMethodAndParam() throws Exception {
    //获取class
    Class<ReflectTest> reflectTestClass = ReflectTest.class;


    //获取所有方法(子类所有)
    Method[] declaredMethods = reflectTestClass.getDeclaredMethods();
    //获取public方法(包括子类、父类)
    Method[] methods = reflectTestClass.getMethods();
    Method method = methods[0];
    //获取方法返回类型
    Class<?> returnType = method.getReturnType();
    //获取方法抛出异常类型
    Class<?>[] exceptionTypes = method.getExceptionTypes();
    //获取方法的参数类型数组
    Class<?>[] parameterTypes = method.getParameterTypes();


    Method setPrivateParam = reflectTestClass.getDeclaredMethod("setPrivateParam", String.class);
    //获取全部参数
    Parameter[] parameters = setPrivateParam.getParameters();
    Parameter parameter = parameters[0];
    //获取参数名, 如果是arg0, 可在编译参数加上-paramter
    String name = parameter.getName();
    //获取参数类型
    Class<?> type = parameter.getType();
}
```

## 操作私有方法&修改私有属性

### 操作私有方法
```
private String testOperatePrivateMethod(String arg1, String arg2) {
    return arg1 + "&" + arg2;
}

@Test
public void testOperatePrivateMethod() throws Exception {
    ReflectTest reflectTest = new ReflectTest();
    Class<? extends ReflectTest> reflectTestClass = reflectTest.getClass();
    //操作私有方法
    Method testPrivate = reflectTestClass.getDeclaredMethod("testOperatePrivateMethod", String.class, String.class);
    //获取方法访问权限
    testPrivate.setAccessible(true);
    System.out.println(testPrivate.invoke(reflectTest, "code", "dance"));
}
```

### 修改私有属性
```
private String name = "code";

@Test
public void testOperatePrivateAttributes() throws Exception {
    ReflectTest reflectTest = new ReflectTest();
    Class<? extends ReflectTest> reflectTestClass = reflectTest.getClass();

    //修改私有属性的值
    Field name = reflectTestClass.getDeclaredField("name");
    //code
    System.out.println(reflectTest.getName());
    name.set(reflectTest, "code is worse than dance");
    //code is worse than dance
    System.out.println(reflectTest.getName());
}
```

### 修改常量-基本类型

1. 直接初始化-无法修改
```
private final String testFinalOne = "code";
@Test
public void testOperateFinalAttributes() throws Exception {
    ReflectTest reflectTest = new ReflectTest();
    Class<? extends ReflectTest> reflectTestClass = reflectTest.getClass();

    //修改不了
    Field testFinalOne = reflectTestClass.getDeclaredField("testFinalOne");
    testFinalOne.setAccessible(true);
    //输出code
    System.out.println(reflectTest.getTestFinalOne());
    testFinalOne.set(reflectTest, "code is worse than dance");
    //输出code is worse than dance, 反射改了值, 但是常量的值不会被修改
    System.out.println(testFinalOne.get(reflectTest));
    //输出code
    System.out.println(reflectTest.getTestFinalOne());
}
```

2. 构造器初始化-可以修改
```
private final String testFinalTwo;

//构造器初始化
public ReflectTest() {
    testFinalTwo = "code";
}
@Test
public void testOperateFinalAttributes() throws Exception {
    ReflectTest reflectTest = new ReflectTest();
    Class<? extends ReflectTest> reflectTestClass = reflectTest.getClass();

    //修改成功, 在构造器里定义final变量的值
    Field testFinalTwo = reflectTestClass.getDeclaredField("testFinalTwo");
    testFinalTwo.setAccessible(true);
    System.out.println(reflectTest.getTestFinalTwo());
    testFinalTwo.set(reflectTest, "code is worse than dance");
    //输出code
    System.out.println(reflectTest.getTestFinalTwo());
}
```

3. 三目表达式-可以修改
```

//三目表达式
private final String testFinalThree = null == null ? "code" : "";
//这样写不行，基本类型会被计算
//private final String testFinalThree = 1 == 1 ? "code" : "";
@Test
public void testOperateFinalAttributes() throws Exception {
    ReflectTest reflectTest = new ReflectTest();
    Class<? extends ReflectTest> reflectTestClass = reflectTest.getClass();

    //修改成功, 在构造器里定义final变量的值
    Field testFinalTwo = reflectTestClass.getDeclaredField("testFinalTwo");
    testFinalTwo.setAccessible(true);
    System.out.println(reflectTest.getTestFinalTwo());
    testFinalTwo.set(reflectTest, "code is worse than dance");
    //输出code
    System.out.println(reflectTest.getTestFinalTwo());
}
```

[代码地址](https://github.com/fuyuaaa/study-java/blob/master/java-basic/src/main/java/top/fuyuaaa/study/basic/reflect/ReflectTest.java)