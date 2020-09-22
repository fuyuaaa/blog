

title: Mybatis源码-解析器
date: 2020-07-08 10:00:00
tags: [mybatis源码]
categories: mybatis源码

--------

# 概述

解析器模块：一个功能是对XPath进行封装，为MyBatis初始 化时解析 mybatis-config.xml 配置文件以及映射配置文件提供支持。另一个功能是为处理动态 SQL 语句中的占位符提供支持。

源码路径：`org.apache.ibatis.parsing`

主要的类

```a
org.apache.ibatis.parsing.XPathParser 解析mybatis-config.xml
org.apache.ibatis.parsing.XNode xml文件中的节点
org.apache.ibatis.parsing.GenericTokenParser 解析到占位符的字面值，再交于下面的VariableTokenHandler处理
org.apache.ibatis.parsing.PropertyParser 处理${id}占位符的内容，替换成<properties>里id的值（主要是内部类VariableTokenHandler）
```

# XPathParser

```java
  // Document对象
  private final Document document;
  // 是否开启验证
  private boolean validation;
  // 用于加载本地DTD文件
  private EntityResolver entityResolver;
  // mybatis-config.xml 中<properties>标签定义的键位对集合
  private Properties variables;
  // XPath 对象
  private XPath xpath;
```

`document`  xml使用DOM解析之后的结果（常用的三种xml解析方式：DOM,SAX,StAX）

`validation`是否开启验证，即是否校验xml

`entityResolver`  用于加载DTD文件，mybatis实现了`org.apache.ibatis.builder.xml.XMLMapperEntityResolver`，来加载本地的DTD

```java
private static final String MYBATIS_CONFIG_DTD = "org/apache/ibatis/builder/xml/mybatis-3-config.dtd";
private static final String MYBATIS_MAPPER_DTD = "org/apache/ibatis/builder/xml/mybatis-3-mapper.dtd";
```



`variables `mybatis-config.xml中`<properties>`里的键值对

`xpath` 用来查询xml中指定路径的节点值，比如`xpath.evaluate("/x/x", document, XPathConstants.STRING);`

## XMLMapperEntityResolver

默认情况下， 对 XML 文档进行验证时，会根据 XML 文档开始位置指定的网址加载对应的 DTD 文件或 XSD 文件 。如果解析 `mybatis-config.xml` 配置文件，默认联网加载 http://mybatis.org/dtd/mybatis-3-config.dtd这个 DTD 文档，当网络比较慢时会导致验证过程缓慢。 在实践中往往会提前设置 `EntityResolver `接口对象加载本地的 DTD 文件，从而避免联网加载 DTD 文件。XMLMapperEntityResolver是 MyBatis提供的` EntityResoIver`接口的实现类。

`EntityResolver` 接口的核心是 `resolveEntity()`方法， `XMLMapperEntityResolver `的实现如下

```java
  //指定 mybatis-config.xm 文件和映射文件 对应的 DTD 的 SystemId
  private static final String IBATIS_CONFIG_SYSTEM = "ibatis-3-config.dtd";
  private static final String IBATIS_MAPPER_SYSTEM = "ibatis-3-mapper.dtd";
  private static final String MYBATIS_CONFIG_SYSTEM = "mybatis-3-config.dtd";
  private static final String MYBATIS_MAPPER_SYSTEM = "mybatis-3-mapper.dtd";

  //指定 mybatis-config.xm 文件和映射文件对应的 DTD 文件的具体位置
  private static final String MYBATIS_CONFIG_DTD = "org/apache/ibatis/builder/xml/mybatis-3-config.dtd";
  private static final String MYBATIS_MAPPER_DTD = "org/apache/ibatis/builder/xml/mybatis-3-mapper.dtd";

  @Override
  public InputSource resolveEntity(String publicId, String systemId) throws SAXException {
    try {
      if (systemId != null) {
        String lowerCaseSystemId = systemId.toLowerCase(Locale.ENGLISH);
        //根据systemId加载对应的dtd文件
        if (lowerCaseSystemId.contains(MYBATIS_CONFIG_SYSTEM) || lowerCaseSystemId.contains(IBATIS_CONFIG_SYSTEM)) {
          return getInputSource(MYBATIS_CONFIG_DTD, publicId, systemId);
        } else if (lowerCaseSystemId.contains(MYBATIS_MAPPER_SYSTEM) || lowerCaseSystemId.contains(IBATIS_MAPPER_SYSTEM)) {
          return getInputSource(MYBATIS_MAPPER_DTD, publicId, systemId);
        }
      }
      return null;
    } catch (Exception e) {
      throw new SAXException(e.toString());
    }
  }
```

## 构造方法中创建Document

```java
//构造器
public XPathParser(String xml) {
  commonConstructor(false, null, null);
  this.document = createDocument(new InputSource(new StringReader(xml)));
}

//createDocument必须在commonConstructor之后调用
private Document createDocument(InputSource inputSource) {
  try {
    //创建DocumentBuilderFactory
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    //进行一系列配置（没细看这些配置干啥用）
    //...省略

    //创建DocumentBuilder
    DocumentBuilder builder = factory.newDocumentBuilder();
    builder.setEntityResolver(entityResolver);
    builder.setErrorHandler(new ErrorHandler() {
      //...省略
    });
    //加载xml
    return builder.parse(inputSource);
  } catch (Exception e) {
    throw new BuilderException("Error creating document instance.  Cause: " + e, e);
  }
}

//设置构造器传入的值，创建xpath
private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
  this.validation = validation;
  this.entityResolver = entityResolver;
  this.variables = variables;
  XPathFactory factory = XPathFactory.newInstance();
  this.xpath = factory.newXPath();
}
```



# XNode

`XPathParser` 通过 `XPathParser.evalNode` 获取`XNode`。

```java
public XNode evalNode(Object root, String expression) {
  Node node = (Node) evaluate(expression, root, XPathConstants.NODE);
  if (node == null) {
    return null;
  }
  //XNode对象其实是对org.w3c.dom.Node对象做了封装和解析
  //org.w3c.dom.Node 就是 xml 里对应的节点
  return new XNode(this, node, variables);
}
```

## PropertyParser

`XPathParser` 中提供了一系列的` eval*()`方法用于解析 `boolean、 short、 long、 int、 String、 Node` 等类型的信息，它通过调用前面介绍的` XPath.evaluate()`方法查找指定路径的节点或属性，并进行相应的类型装换 。

这里需要注意

`XPathParser.evalString()`

```java
public String evalString(Object root, String expression) {
  String result = (String) evaluate(expression, root, XPathConstants.STRING);
  //PropertyParser.parse方法会进行占位符的解析以及默认值的处理
  result = PropertyParser.parse(result, variables);
  return result;
}
```

在 `PropertyParser` 中指定了是否开启使用默认值的功能以及默认的分隔符

```java
//在 mybatis-config.xml 中<properties>节点下自己置是否开启默认值功能的对应配置项
public static final String KEY_ENABLE_DEFAULT_VALUE = KEY_PREFIX + "enable-default-value";

//配置占位符与默认值之间的默认分隔符的对应配置项
public static final String KEY_DEFAULT_VALUE_SEPARATOR = KEY_PREFIX + "default-value-separator";

//上述两个配置的默认值
private static final String ENABLE_DEFAULT_VALUE = "false";
private static final String DEFAULT_VALUE_SEPARATOR = ":";
```

在`mybatis-config.xml`中使用如下

```xml
<properties>
  <property name="enable-default-value" value="true"/>
  <property name="default-value-separator" value=":"/>
</properties>
```

`PropertyParser.parse()`方法中会创建 `GenericTokenParser `解析器，井将默认值的处理委托给`GenericTokenParser.parse()`方法，

```java
public static String parse(String string, Properties variables) {
  VariableTokenHandler handler = new VariableTokenHandler(variables);
	//openToken = "${"  ;  closeToken = "}"
  GenericTokenParser parser = new GenericTokenParser("${", "}", handler);
  return parser.parse(string);
}
```

`GenericTokenParser.parse()`会顺序查找 `openToken` 和 `closeToken`， 解析得到占位符的字面值（比如`${id}`会被处理成`id`），并将其交给传入的`TokenHandler`（即`PropertyParser`的内部类`VariableTokenHandler`的`handleToken`方法）处理， 然后将解析结果重新拼装成字符串井返回。

`VariableTokenHandler`的`handleToken`方法如下

```java
public String handleToken(String content) {
  // 检测 variables 集合是否为空
  if (variables != null) {
    String key = content;
    // 如果允许默认值，处理默认值逻辑
    if (enableDefaultValue) {
      // 分隔，一般是:，比如id:1
      final int separatorIndex = content.indexOf(defaultValueSeparator);
      // 解析默认值
      String defaultValue = null;
      if (separatorIndex >= 0) {
        // 获取占位符的名称
        key = content.substring(0, separatorIndex);
        defaultValue = content.substring(separatorIndex + defaultValueSeparator.length());
      }
      // 存在默认值的处理逻辑
      if (defaultValue != null) {
        // 在 variables 集合中查找指定的占位符
        return variables.getProperty(key, defaultValue);
      }
    }
    // 不支持默认值的功能，直接查找 variables 集合
    if (variables.containsKey(key)) {
      return variables.getProperty(key);
    }
  }
  // 查不到，返回原来的格式
  return "${" + content + "}";
}
```



# 参考

[《Mybatis技术内幕》](https://book.douban.com/subject/27087564/)