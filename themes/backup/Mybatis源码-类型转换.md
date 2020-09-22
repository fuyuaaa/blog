title: Mybatis源码-类型转换
date: 2020-07-08 12:00:00
tags: [mybatis源码]
categories: mybatis源码

-------------

JDBC 数据类型与 Java 语言中的数据类型井不是完全对应的 ，所以在 `PreparedStatement` 为 SQL 语句绑定参数时，需要从 Java 类型转换成 JDBC 类型，而从结果集中获取数据时，则需要 从 JDBC类型转换成 Java 类型。 

MyBatis使用类型处理器完成上述两种转换，如下图所示。

<img src="https://fuyuaaa-bucket.oss-cn-hangzhou.aliyuncs.com/blog_pics/image-20200708162111317.png" alt="image-20200708162111317" style="zoom: 33%; margin-left: 10px;" />

# TypeHandler

Mybatis中所有的类型转换器都继承了 TypeHandler接口，在 TypeHandler接口中定义了如 下四个方法 ， 这四个方法分为两类 : setParameter()方法负责将数据 由 Java 类型转换成 JDBC 类型: getResult()方法及其重载负责将数据由 JDBC类型转换成 Java类型。

```java
public interface TypeHandler<T> {

  /**
   * 设置 PreparedStatement 的指定参数
   *
   * Java Type => JDBC Type
   */
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

  /**
   * 获得 ResultSet 的指定字段的值
   *
   * JDBC Type => Java Type
   */
  T getResult(ResultSet rs, String columnName) throws SQLException;
  T getResult(ResultSet rs, int columnIndex) throws SQLException;
  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```

`BaseTypeHandler` ，实现 TypeHandler 接口，继承 TypeReference 抽象类，TypeHandler 基础抽象类。`BaseTypeHandler` 实现了`setParameter`和`getResult`，但是对于这俩方法里针对非null参数的处理都是抽象方法，交于子类实现。

```java
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
  if (parameter == null) {
    if (jdbcType == null) {
      throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
    }
    try {
      //绑定参数为null的情况处理
      ps.setNull(i, jdbcType.TYPE_CODE);
    } catch (SQLException e) {
      //...
    }
  } else {
    try {
      setNonNullParameter(ps, i, parameter, jdbcType);
    } catch (Exception e) {
      //绑定非空参数，该方法抽象方法，由子类实现
      setNonNullParameter(ps, i, parameter, jdbcType);
    }
  }
}

@Override
public T getResult(ResultSet rs, String columnName) throws SQLException {
  try {
    //获取非空参数，该方法抽象方法，由子类实现
    return getNullableResult(rs, columnName);
  } catch (Exception e) {
    //...
  }
}
```

`BaseTypeHandler`的实现类比较多 ， 但大多是直接调用`PreparedStatement`和`ResultSet`或`CallableStatement`的对应方法，实现比较简单。

# TypeHandlerRegistry

在 MyBatis 初始化过程中，会为所有己知的`TypeHandler`创建对象，并实现注册到`TypeHandlerRegistry`中，由`TypeHandlerRegistry` 负责管理这些`TypeHandler`对象。

```java
//记录 JdbcType 与 TypeHandler 之间的对应关系，其中 JdbcType 是一个枚举类型，它定义对应的 JDBC 类型
//该集合主要用于从结果集读取数据时，将数据从 Jdbc 类型转换成 Java 类型
private final Map<JdbcType, TypeHandler<?>> jdbcTypeHandlerMap = new EnumMap<>(JdbcType.class);

//记录了 Java 类型向指定 JdbcType 转换时，需妥使用的 TypeHandler 对象。
//例如: Java 类型中的 String 可能转换成数据库 的 char、 varchar 等多种类 型，所以存在一对多关系
private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>();
private final TypeHandler<Object> unknownTypeHandler;

//记录了全部 TypeHandler 的类型以及该类型相应的 TypeHandler 对象
private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>();

//空 TypeHandler 集合的标识
private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = Collections.emptyMap();

private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;
```

`TypeHandler`的注册通过`TypeHandlerRegistry.registe()`进行，`registe()`有很多重载方法，这里略。

`TypeHandler`的查找通过`TypeHandlerRegistry.getTypeHandler()`进行，`getTypeHandler()`有很多重载方法，这里略。

除了 MyBatis 本身提供的`TypeHandler`实现，我们也可以添加自定义的`TypeHandler`接口实现，添加方式是在`mybatis-config.xml` 配置文件中的`<typeHandlers>`节点下， 添加相应的`<typeHandler>`节点配置，并指定自定义的 `TypeHandler`接口实现类。在 MyBatis 初始化时会解 析该节点，并将该`TypeHandler`类型的对象注册到`TypeHandlerRegistry`中，供MyBatis后续使用。

# TypeAliasRegistry

MyBatis 可以为一个类添加一个别名，之后就可以通过别名引用该类。

MyBatis通过`TypeAliasRegisry`类完成别名注册和管理的功能，`TypeAliasRegistry`的结构比较简单，它通过`TYPE_ALIASES`字段 （Map<String， Class<?>>类型）管理别名与Java类型之间的对应关系，通过`TypeAliasRegistry.registerAlias()`方法完成注册别名，该方法的具体实现如下 :

```java
public void registerAlias(String alias, Class<?> value) {
  if (alias == null) {
    throw new TypeException("The parameter alias cannot be null");
  }
  //将别名转换为小写
  String key = alias.toLowerCase(Locale.ENGLISH);
  //检测别名是否已经存在
  if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
    throw new TypeException("...");
  }
  //注册
  typeAliases.put(key, value);
}
```

在`TypeAliasRegisry`的构造方法中，默认为 Java 的基本类型及其数组类型、基本类型的封装类及其`数组类型 、 Date、 BigDecimal、 BigInteger、 Map、 HashMap、 List、 ArrayList、 Collection、 Iterator、 ResultSet`等类型添加了别名。

`TypeAliasRegisry`还可通过扫描指定包下的类来添加别名，或者通过Alias注解

# 参考

[《Mybatis技术内幕》](https://book.douban.com/subject/27087564/)