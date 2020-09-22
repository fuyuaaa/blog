title: Mybatis源码-反射工具箱
date: 2020-07-08 1:00:00
tags: [mybatis源码]
categories: mybatis源码

-----------

MyBatis 在进行参数处理、结果映射等操作时 ，会涉及大量的反射操作。 Java 中的反射虽然功能强大，但是代码编写起来比较复杂且容易出错，为了简化反射操作的相关代码， MyBatis 提供了专门的反射模块，该模块位于`org.apache.ibatis.reflection`包中，它对常见的反射操作做了进一步封装，提供了更加简洁方便的反射 API。

# Reflector & ReflectorFactory

## Reflector

`Reflector`是 MyBatis 中反射模块的基础，每个`Reflector`对象都对应一个类，在`Reflector`中缓存了反射操作需要使用的类的元信息。

### Reflector的字段信息及构造器

```java
//对应的Class类型
private final Class<?> type;
//可读属性的名称集合，可读属性就是存在相应getter方法的属性，初始值为空数纽
private final String[] readablePropertyNames;
//可写属性的名称集合，可写属性就是存在相应setter方法的属性，初始值为空数纽
private final String[] writablePropertyNames;
//记录了属性相应的setter方法，key是属性名称，value是Invoker对象它是对setter方法对应
private final Map<String, Invoker> setMethods = new HashMap<>();
//属性相应的getter方法集合，key是属性名称，value也是Invoker对象
private final Map<String, Invoker> getMethods = new HashMap<>();
//记录了属性相应的setter方法的参数值类型，key是属性名称，value是setter方法的参数类型
private final Map<String, Class<?>> setTypes = new HashMap<>();
//记录了属性相应的getter方法的返回位类型，key是属性名称，value是getter方法的返回位类型
private final Map<String, Class<?>> getTypes = new HashMap<>();
//默认构造方法
private Constructor<?> defaultConstructor;
//记录了所有属性名称的集合
private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();
```

在`Reflector`的构造方法中会解析指定的Class对象，并填充上述字段。

```java
public Reflector(Class<?> clazz) {
  //初始化type
  type = clazz;
  //查找 clazz 的默认构造方法(元参构造方法)，具体实现是通过反射遍历所有构造方法，找到无参的那个
  addDefaultConstructor(clazz);
  //处理 clazz 中的 getter 方法，填充 getMethods 集合和 getTypes 集合
  addGetMethods(clazz);
  //处理 clazz 中的 setter 方法，填充 setMethods 集合和 setTypes 集合
  addSetMethods(clazz);
  //处理没有 getter/setter 方法的字段
  addFields(clazz);
  //根据 getMethods/setMethods 集合 ，初始化可读/写属性的名称集合
  readablePropertyNames = getMethods.keySet().toArray(new String[0]);
  writablePropertyNames = setMethods.keySet().toArray(new String[0]);
  //初始化 caseInsensitivePropertyMap 集合，其中记录了所有大写格式的属性名称
  for (String propName : readablePropertyNames) {
    caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
  }
  for (String propName : writablePropertyNames) {
    caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
  }
}
```

### Reflector.addGetMethods()方法

```java
private void addGetMethods(Class<?> clazz) {
  //Map<属性名, get方法列表>
  //同一个属性可能有多个get方法（子类覆盖父类的方法）
  Map<String, List<Method>> conflictingGetters = new HashMap<>();
  //获取当前类以及其父类中定义的所有方法的唯一签名以及相应的 Method 对象
  Method[] methods = getClassMethods(clazz);
  //将method中的get方法过滤出来（无参数，且get或者is开头的），加入到 conflictingGetters 中
  Arrays.stream(methods).filter(m -> m.getParameterTypes().length == 0 && PropertyNamer.isGetter(m.getName()))
    .forEach(m -> addMethodConflict(conflictingGetters, PropertyNamer.methodToProperty(m.getName()), m));
  //处理同一个属性多个get方法的情况，并填充getMethods，getTypes
  resolveGetterConflicts(conflictingGetters);
}

private Method[] getClassMethods(Class<?> clazz) {
    //Map<唯一签名, Method 对象>
    //用于记录指定类中定义的全部方法的唯一签名以及对应的 Method 对象
    Map<String, Method> uniqueMethods = new HashMap<>();
    Class<?> currentClass = clazz;
    while (currentClass != null && currentClass != Object.class) {
      //记录 currentClass 这个类中定义的全部方法
      addUniqueMethods(uniqueMethods, currentClass.getDeclaredMethods());
 
      //记录接口中定义的方法
      Class<?>[] interfaces = currentClass.getInterfaces();
      for (Class<?> anInterface : interfaces) {
        addUniqueMethods(uniqueMethods, anInterface.getMethods());
      }
      
      //继续获取父类的方法
      currentClass = currentClass.getSuperclass();
    }

    Collection<Method> methods = uniqueMethods.values();

    return methods.toArray(new Method[0]);
  }

  private void addUniqueMethods(Map<String, Method> uniqueMethods, Method[] methods) {
    for (Method currentMethod : methods) {
      if (!currentMethod.isBridge()) {
        //通过 Reflector.getSignature() 方法得到的方法签名是:返回值类型#方法名称:参数类型列表。
        //例如 Reflector.getSignature(Method)方法的唯一签名是:java.lang.String#getSignature:java.lang.reflect.Method
        //通过 Reflector.getSignature() 方法得到的方法签名是全局唯一的，可以作为该方法的唯一标识
        String signature = getSignature(currentMethod);
        //检测是否在子类中已经添加过该方法，如果在子类中已经添加过，则表示子类覆盖了该方法，无须再向 uniqueMethods 集合中添加该方法了
        if (!uniqueMethods.containsKey(signature)) {
          uniqueMethods.put(signature, currentMethod);
        }
      }
    }
  }

```

### Reflector.addFields()

该方法会处理类中定义的所有字段 ，并且将处理后的字段信息添加到 `setMethods` 集合、 `setTypes` 集合、 `getMethods` 集合以及 `getTypes` 集合中。

```java
  private void addFields(Class<?> clazz) {
    //获取定义的全部字段
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
      if (!setMethods.containsKey(field.getName())) {
        int modifiers = field.getModifiers();
        //过滤掉final和static修饰的
        if (!(Modifier.isFinal(modifiers) && Modifier.isStatic(modifiers))) {
          //填充 setMethods 集合和 setTypes 集合
          addSetField(field);
        }
      }
      if (!getMethods.containsKey(field.getName())) {
        addGetField(field);
      }
    }
    //递归处理父类字段
    if (clazz.getSuperclass() != null) {
      addFields(clazz.getSuperclass());
    }
  }
```

## ReflectorFactory

`ReflectorFactory`接口主要实现了对`Reflector`对象的创建和缓存。

```java
public interface ReflectorFactory {
  //检测该 ReflectorFactory 对象是否会缓存 Reflector 对象 
  boolean isClassCacheEnabled();

  //设置是否缓存 Reflector 对象 
  void setClassCacheEnabled(boolean classCacheEnabled);

  //创建指定 Class 对应的 Reflector 对象
  Reflector findForClass(Class<?> type);
}
```

Mybatis给该接口提供了默认实现，`DefaultReflectorFactory`。

```java

//默认开启对 Reflector 对象的缓存
private boolean classCacheEnabled = true;
//使用 ConcurrentMap 集合实现对 Reflector 对象的缓存
private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<>();

public Reflector findForClass(Class<?> type) {
  //检测是否开启缓存
  if (classCacheEnabled) {
    // synchronized (type) removed see issue #461
    //存在就返回，不存在新建 Reflector 对象并返回
    return reflectorMap.computeIfAbsent(type, Reflector::new);
  } else {
    //未开启缓存，则直接创建并返回 Reflector 对象
    return new Reflector(type);
  }
}
```

除了使用 MyBatis 提供的`DefaultReflectorFactory`实现，我们还可以在`mybatis-config.xml`中配置自定义的`ReflectorFactory`实现类，从而实现功能上的扩展。

# TypeParameterResolver

`TypeParameterResolver`是一个工具类，提供了一系列静态方法来解析指定类中的字段、方法返回值或方法参数的类型。 

## Type 接口

Type接口有四个子接口，如下所示

- Class，原始类型 。
- ParameterizedType，参数化类型，比如List<String>。
- TypeVariable，类型变量，比如List<T>中的T。
- GenericArrayType，表示的是数组类型且组成元素是 ParameterizedType 或 TypeVariable，比如List<String>[]或T[]。
- WildcardType，通配符类型，比如 ? super Integer，? extends Number。

## resolveFieldType()方法

`TypeParameterResolver`中通过` resolveFieldType()`方法、 `resolveReturnType() `方法、` resolveParamTypes()`方法分别解析宇段类型、方法返回值类型和方法参数列表中各个参数的类型 。这三个方法的逻辑基本类似，这里只看`resolveFieldType()`方法。

```java
public static Type resolveFieldType(Field field, Type srcType) {
  //获取字段的声明类型
  Type fieldType = field.getGenericType();
  //获取字段定义所在的类的 Class 对象
  Class<?> declaringClass = field.getDeclaringClass();
  //调用 resolveType ()方法进行后续处理
  return resolveType(fieldType, srcType, declaringClass);
}

//resolveFieldType()方法、resolveReturnType()方法、resolveParamTypes()方法都会调用该方法进行解析
//resolveTypeVar、resolveParameterizedType、resolveGenericArrayType这些具体的方法就不看了
private static Type resolveType(Type type, Type srcType, Class<?> declaringClass) {
  if (type instanceof TypeVariable) { //解析 TypeVariable 类型
    return resolveTypeVar((TypeVariable<?>) type, srcType, declaringClass);
  } else if (type instanceof ParameterizedType) { //解析 ParameterizedType 类型
    return resolveParameterizedType((ParameterizedType) type, srcType, declaringClass);
  } else if (type instanceof GenericArrayType) { //解析 GenericArrayType 类型
    return resolveGenericArrayType((GenericArrayType) type, srcType, declaringClass);
  } else {
    return type;
  }
}
```

# ObjectFactory



```java
public interface ObjectFactory {

  //设置配置信息
  default void setProperties(Properties properties) {}

  //通过无参构造器创建指定类的对象
  <T> T create(Class<T> type);

  //根据参数列表，从指定类型中选择合适的构造器创建对象
  <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

  //检测指定类型是否为集合类型，主妥处理 java.util.Collection 及其子类
  <T> boolean isCollection(Class<T> type);
}
```

`DefaultObjectFactory`是 MyBatis提供的`ObjectFactory`接口的唯一实现，它是一个反射工厂，其`create()`方法通过调用 `instantiateClass()`方法实现。 

```java
//会根据传入的参数列表选择合适的构造函数实例化对象
private  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  try {
    Constructor<T> constructor;
    //通过无参构造函数创建对象
    if (constructorArgTypes == null || constructorArgs == null) {
      constructor = type.getDeclaredConstructor();
      try {
        return constructor.newInstance();
      } catch (IllegalAccessException e) {
        if (Reflector.canControlMemberAccessible()) {
          constructor.setAccessible(true);
          return constructor.newInstance();
        } else {
          throw e;
        }
      }
    }
    //根据指定的参数列表查找构造函数，并实例化对象
    constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Class[0]));
    try {
      return constructor.newInstance(constructorArgs.toArray(new Object[0]));
    } catch (IllegalAccessException e) {
      //...省略
    }
  } catch (Exception e) {
    //...省略
  }
}
```



除了使用 MyBatis 提供的`DefaultObjectFactory`实现，我们还可以在`mybatis-config.xml`配置文件中指定自定义的`ObjectFactory`接口实现类，从而实现功能上的扩展。

# Property 工具集

- PropertyCopier，属性拷贝的工具类。
- PropertyNamer，提供多个静态方法帮助完成方法名到属性名的转换，以及多种检测操作 。
- PropertyTokenizer，解析属性表达式，比如举个例子，在访问 `"order[0].item[0].name"` 时，我们希望拆分成 `"order[0]"`、`"item[0]"`、`"name"` 三段，那么就可以通过 PropertyTokenizer 来实现

# MetaClass

`MetaClass`通过`Reflector`和`PropertyTokenizer`组合使用，实现了对复杂的属性表达式的解析，并实现了获取指定属性描述信息的功能。 

`MetaClass`字段以及构造器

```java
//ReflectorFactory 对象，用于缓存 Reflector 对象
private final ReflectorFactory reflectorFactory;

//在创建 MetaClass 时会指定一个类，该 Reflector 对象会用于记录该类相关 的元信息
private final Reflector reflector;


private MetaClass(Class<?> type, ReflectorFactory reflectorFactory) {
  this.reflectorFactory = reflectorFactory;
  this.reflector = reflectorFactory.findForClass(type);
}

//使用静态方法创建MetaClass
public static MetaClass forClass(Class<?> type, ReflectorFactory reflectorFactory) {
  return new MetaClass(type, reflectorFactory);
}
```

`MetaClass`中比较重要的是`findProperty()`方法，它是通过调用`MetaClass.buildProperty()`方法 实现的 ，而`buildProperty()`方法会通过`PropertyTokenizer`解析复杂的属性表达式，具体实现如下 :

```java
private StringBuilder buildProperty(String name, StringBuilder builder) {
  //解析属性表达式
  PropertyTokenizer prop = new PropertyTokenizer(name);
  //是否还有子表达式
  if (prop.hasNext()) {
    //查找 PropertyTokenizer.name 对应的属性
    String propertyName = reflector.findPropertyName(prop.getName());
    if (propertyName != null) {
      builder.append(propertyName);
      builder.append(".");
      //为该属性创建对应的 MetaClass 对象
      MetaClass metaProp = metaClassForProperty(propertyName);
      //递归解析 PropertyTokenizer.children 字段，并将解析结果添加到 builder 中保存
      metaProp.buildProperty(prop.getChildren(), builder);
    }
  } else {
    String propertyName = reflector.findPropertyName(name);
    if (propertyName != null) {
      builder.append(propertyName);
    }
  }
  return builder;
}

public MetaClass metaClassForProperty(String name) {
  //查找指定属性对应的 Class
  Class<?> propType = reflector.getGetterType(name);
  //为该属性创建对应的 MetaClass 对象
  return MetaClass.forClass(propType, reflectorFactory);
}
```

`MetaClass.hasGetter()、 hasSetter()`方法负责判断属性表达式所表示的属性是否有对应的属性，不多分析。

# ObjectWrapper

`ObjectWrapper`接口是对对象的包装 ，抽象了对象的属性信息 ，它定义了一系列查询对象属性信息的方法，以及更新属性的方法 。

```java
public interface ObjectWrapper {

  //如采 ObjectWrapper 中封装的是普通的 Bean 对象 ，则 调用相应属性的相应 getter 方法
  //如采封装的是集合类，则获取指定 key 或下标对应的 value 值
  Object get(PropertyTokenizer prop);

  //如果 ObjectWrapper 中封装的是普通的 Bean 对象 ， 则调用相应属性的相应 setter 方法
  //如果封装的是集合类，则设置指定 key 或下标对应的 value 值
  void set(PropertyTokenizer prop, Object value);

  //查找属性表达式指定的属性，第二个参数表示是否忽略属性表达式中的下画线
  String findProperty(String name, boolean useCamelCaseMapping);

  //查找可读属性的名称集合
  String[] getGetterNames();

  //查找可写属性的名称集合
  String[] getSetterNames();

  //解析属性表达式指定属性的 setter 方法的参数类型
  Class<?> getSetterType(String name);

  //解析属性表达式指定属性的 getter 方法的返回值类型
  Class<?> getGetterType(String name);

  //判断属性表达式指定属性是否有 getter/setter 方法
  boolean hasSetter(String name);
  boolean hasGetter(String name);

  //为属性表达式指定的属性创建相应的 MetaObject 对象
  MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory);

  boolean isCollection(); // 封装的对象是否为 Collection 类型
  void add(Object element); // 调用 Collection 对象的 add()方法
  <E> void addAll(List<E> element) ; // 调用 Collection 对象的 addAll ()方法
}
```

# MetaObject

`MetaObject` ，对象元数据，提供了对象的属性值的获得和设置等等方法。可以理解成，对 BaseWrapper 操作的进一步**增强**。

构造方法

```java
// MetaObject.java

/**
 * 原始 Object 对象
 */
private final Object originalObject;
/**
 * 封装过的 Object 对象
 */
private final ObjectWrapper objectWrapper;
private final ObjectFactory objectFactory;
private final ObjectWrapperFactory objectWrapperFactory;
private final ReflectorFactory reflectorFactory;

private MetaObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
    this.originalObject = object;
    this.objectFactory = objectFactory;
    this.objectWrapperFactory = objectWrapperFactory;
    this.reflectorFactory = reflectorFactory;
  
    if (object instanceof ObjectWrapper) {
        this.objectWrapper = (ObjectWrapper) object;
    } else if (objectWrapperFactory.hasWrapperFor(object)) { // <2>
        // 创建 ObjectWrapper 对象
        this.objectWrapper = objectWrapperFactory.getWrapperFor(this, object);
    } else if (object instanceof Map) {
        // 创建 MapWrapper 对象
        this.objectWrapper = new MapWrapper(this, (Map) object);
    } else if (object instanceof Collection) {
        // 创建 CollectionWrapper 对象
        this.objectWrapper = new CollectionWrapper(this, (Collection) object);
    } else {
        // 创建 BeanWrapper 对象
        this.objectWrapper = new BeanWrapper(this, object);
    }
}

//创建 MetaObject 对象
//如果 object 为空的情况下，返回 SystemMetaObject.NULL_META_OBJECT
public static MetaObject forObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
    if (object == null) {
        return SystemMetaObject.NULL_META_OBJECT;
    } else {
        return new MetaObject(object, objectFactory, objectWrapperFactory, reflectorFactory);
    }
}
```

`metaObjectForProperty(String name)` 方法，创建指定属性的 MetaObject 对象。代码如下：

```java
public MetaObject metaObjectForProperty(String name) {
    // 获得属性值
    Object value = getValue(name);
    // 创建 MetaObject 对象
    return MetaObject.forObject(value, objectFactory, objectWrapperFactory, reflectorFactory);
}
```

`getValue(String name)` 方法，获得指定属性的值。代码如下：

```java
// MetaObject.java

public Object getValue(String name) {
    // 创建 PropertyTokenizer 对象，对 name 分词
    PropertyTokenizer prop = new PropertyTokenizer(name);
    // 有子表达式
    if (prop.hasNext()) {
        // 创建 MetaObject 对象
        MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
        // 递归判断子表达式 children ，获取值
        if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
            return null;
        } else {
            return metaValue.getValue(prop.getChildren());
        }
    // 无子表达式
    } else {
        // 获取值
        return objectWrapper.get(prop);
    }
}
```

- 大体逻辑上，就是不断对 `name` 分词，递归查找属性

# 参考

[《Mybatis技术内幕》](https://book.douban.com/subject/27087564/)

