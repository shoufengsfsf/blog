---
layout: mybatis源码解读
title: mybatis源码解读
date: 2020-06-23 23:17:58
tags: mybatis源码解读
categories: mybatis源码解读
---

# MyBatis源码解读

## 带着问题去看源码

分析源码，我们还是从编程式的 demo 入手。Spring 的集成我们会在后面讲到。

```java
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream); 
SqlSession session = sqlSessionFactory.openSession();
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlogById(1);
```

把文件读取成流的这一步我们就省略了。所以下面我们分成四步来分析。

第一步，我们通过建造者模式创建一个工厂类，配置文件的解析就是在这一步完成 的，包括 mybatis-config.xml 和 Mapper 适配器文件。

问题:解析的时候怎么解析的，做了什么，产生了什么对象，结果存放到了哪里。 解析的结果决定着我们后面有什么对象可以使用，和到哪里去取。

第二步，通过 SqlSessionFactory 创建一个 SqlSession。

问题:SqlSession 是用来操作数据库的，返回了什么实现类，除了 SqlSession，还 创建了什么对象，创建了什么环境?

第三步，获得一个 Mapper 对象。

问题:Mapper 是一个接口，没有实现类，是不能被实例化的，那获取到的这个 Mapper 对象是什么对象?为什么要从 SqlSession 里面去获取?为什么传进去一个接 口，然后还要用接口类型来接收?

第四步，调用接口方法。

问题:我们的接口没有创建实现类，为什么可以调用它的方法?那它调用的是什么 方法?它又是根据什么找到我们要执行的 SQL 的?也就是接口方法怎么和 XML 映射器 里面的 StatementID 关联起来的?

此外，我们的方法参数是怎么转换成 SQL 参数的?获取到的结果集是怎么转换成对 象的?

接下来我们就会详细分析每一步的流程，包括里面有哪些核心的对象和关键的方法。

## 一、配置解析过程

首先我们要清楚的是配置解析的过程全部只解析了两种文件。一个是 mybatis-config.xml 全局配置文件。另外就是可能有很多个的 Mapper.xml 文件，也包 括在 Mapper 接口类上面定义的注解。

我们从 mybatis-config.xml 开始。在第一节课的时候我们已经分析了核心配置了， 大概明白了 MyBatis 有哪些配置项，和这些配置项的大致含义。这里我们再具体看一下 这里面的标签都是怎么解析的，解析的时候做了什么。

```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

首先我们 new 了一个 SqlSessionFactoryBuilder，非常明显的建造者模式，它里面 定义了很多个 build 方法的重载，最终返回的是一个 SqlSessionFactory 对象(单例模 式)。我们点进去 build 方法。

这里面创建了一个 XMLConfigBuilder 对象(Configuration 对象也是这个时候创 建的)。

### XMLConfigBuilder

XMLConfigBuilder 是抽象类 BaseBuilder 的一个子类，专门用来解析全局配置文 件，针对不同的构建目标还有其他的一些子类，比如:

XMLMapperBuilder:解析 Mapper 映射器 

XMLStatementBuilder:解析增删改查标签

![image-20200623232353388](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2mz0ztjcj31oi06k7ac.jpg)

根据我们解析的文件流，这里后面两个参数都是空的，创建了一个 parser。

这里有两步，第一步是调用 parser 的 parse()方法，它会返回一个 Configuration 类。

之前我们说过，也就是配置文件里面所有的信息都会放在 Configuration 里面。 Configuration 类里面有很多的属性，有很多是跟 config 里面的标签直接对应的。

我们先看一下 parse()方法:

首先会检查是不是已经解析过，也就是说在应用的生命周期里面，config 配置文件

只需要解析一次，生成的 Configuration 对象也会存在应用的整个生命周期中。接下来就是 parseConfiguration 方法:

```java
parseConfiguration(parser.evalNode("/configuration"));
```

这下面有十几个方法，对应着 config 文件里面的所有一级标签。

问题:MyBatis 全局配置文件的顺序可以颠倒吗?

### propertiesElement()

第一个是解析\<properties>标签，读取我们引入的外部配置文件。这里面又有两种 类型，一种是放在 resource 目录下的，是相对路径，一种是写的绝对路径的。解析的最 终结果就是我们会把所有的配置信息放到名为 defaults 的 Properties 对象里面，最后把 XPathParser 和 Configuration 的 Properties 属性都设置成我们填充后的 Properties 对象。

### settingsAsProperties()

第二个，我们把\<settings>标签也解析成了一个 Properties 对象，对于\<settings> 标签的子标签的处理在后面。

在早期的版本里面解析和设置都是在后面一起的，这里先解析成 Properties 对象是 因为下面的两个方法要用到。

### loadCustomVfs(settings)

loadCustomVfs 是获取 Vitual File System 的自定义实现类，比如我们要读取本地 文件，或者 FTP 远程文件的时候，就可以用到自定义的 VFS 类。我们根据\<settings>标 签里面的\<vfsImpl>标签，生成了一个抽象类 VFS 的子类，并且赋值到 Configuration 中。

### loadCustomLogImpl(settings)

loadCustomLogImpl 是根据\<logImpl>标签获取日志的实现类，我们可以用到很 多的日志的方案，包括 LOG4J，LOG4J2，SLF4J 等等。这里生成了一个 Log 接口的实 现类，并且赋值到 Configuration 中。

### typeAliasesElement()

接下来，我们解析\<typeAliases>标签，我们在讲配置的时候也讲过，它有两种定义 方式，一种是直接定义一个类的别名，一种就是指定一个包，那么这个 package 下面所 有的类的名字就会成为这个类全路径的别名。

类的别名和类的关系，我们放在一个 TypeAliasRegistry 对象里面。

### pluginElement()

接下来就是解析\<plugins>标签，比如 Pagehelper 的翻页插件，或者我们自定义的 插件。\<plugins>标签里面只有\<plugin>标签，\<plugin>标签里面只有\<property>标 签。

标签解析完以后，会生成一个 Interceptor 对象，并且添加到 Configuration 的 InterceptorChain 属性里面，它是一个 List。

### objectFactoryElement()、objectWrapperFactoryElement()

接 下 来 的 两 个 标 签 是 用 来 实 例 化 对 象 用 的 ， \<objectFactory> 和 \<objectWrapperFactory> 这 两 个 标 签 ， 分 别 生 成 ObjectFactory 、 ObjectWrapperFactory 对象，同样设置到 Configuration 的属性里面。

### reflectorFactoryElement()

解析 reflectorFactory 标签，生成 ReflectorFactory 对象(在官方 3.5.1 的 pdf 文 档里面没有找到这个配置)。

### settingsElement(settings)

这里就是对\<settings>标签里面所有子标签的处理了，前面我们已经把子标签全部 转换成了 Properties 对象，所以在这里处理 Properties 对象就可以了。

二级标签里面有很多的配置，比如二级缓存，延迟加载，自动生成主键这些。需要 注意的是，我们之前提到的所有的默认值，都是在这里赋值的。如果说后面我们不知道这个属性的值是什么，也可以到这一步来确认一下。

所有的值，都会赋值到 Configuration 的属性里面去。

### environmentsElement()

这一步是解析\<environments>标签。

我们前面讲过，一个 environment 就是对应一个数据源，所以在这里我们会根据配 置的\<transactionManager>创建一个事务工厂，根据\<dataSource>标签创建一个数据 源，最后把这两个对象设置成 Environment 对象的属性，放到 Configuration 里面。

回答了前面的问题:数据源工厂和数据源在哪里创建。 先记下这个问题:数据源和事务工厂在哪里会用到?

### databaseIdProviderElement()

解析 databaseIdProvider 标签，生成 DatabaseIdProvider 对象(用来支持不同厂 商的数据库)。

### typeHandlerElement()

跟 TypeAlias 一样，TypeHandler 有两种配置方式，一种是单独配置一个类，一种 是指定一个 package。最后我们得到的是 JavaType 和 JdbcType，以及用来做相互映射 的 TypeHandler 之间的映射关系。

最后存放在 TypeHandlerRegistry 对象里面。

问题:这种三个对象(Java 类型，JDBC 类型，Handler)的关系怎么映射?(Map 里面再放一个 Map)

### mapperElement()

http://www.mybatis.org/mybatis-3/zh/configuration.html#mappers

**1)判断**

最后就是\<mappers>标签的解析。

| 扫描类型 | 含义     |
| -------- | -------- |
| resource | 相对路径 |
| url      | 绝对路径 |
| package  | 包       |
| class    | 单个接口 |

首先会判断是不是接口，只有接口才解析;然后判断是不是已经注册了，单个 Mapper 重复注册会抛出异常。

**2)注册**

XMLMapperBuilder.parse()方法，是对 Mapper 映射器的解析。里面有两个方法:

configurationElement()—— 解 析 所 有 的 子 标 签 ， 其 中 buildStatementFromContext()最终获得 MappedStatement 对象。

bindMapperForNamespace()——把 namespace(接口类型)和工厂类绑定起来。

无论是按 package 扫描，还是按接口扫描，最后都会调用到 MapperRegistry 的 addMapper()方法。

MapperRegistry 里面维护的其实是一个 Map 容器，存储接口和代理工厂的映射关 系。

问题:为什么要放一个代理工厂呢?代理工厂用来干什么?

**3)处理注解**

除了映射器文件，在这里也会去解析 Mapper 接口方法上的注解。在 addMapper() 方法里面创建了一个 MapperAnnotationBuilder，我们点进去看一下 parse()方法。

parseCache() 和 parseCacheRef() 方 法 其 实 是 对 @CacheNamespace 和 @CacheNamespaceRef 这两个注解的处理。

parseStatement()方法里面的各种 getAnnotation()，都是对注解的解析，比如 @Options，@SelectKey，@ResultMap 等等。

最后同样会解析成 MappedStatement 对象，也就是说在 XML 中配置，和使用注 解配置，最后起到一样的效果。

**4)收尾**

如果注册没有完成，还要从 Map 里面 remove 掉。

```java
// MapperRegistry.java
finally {
  if (!loadCompleted) {
    knownMappers.remove(type); 
  }
```

最后，MapperRegistry 也会放到 Configuration 里面去。

第二步是调用另一个 build()方法，返回 DefaultSqlSessionFactory。

### **总结**

在这一步，我们主要完成了 config 配置文件、Mapper 文件、Mapper 接口上的注 解的解析。

我们得到了一个最重要的对象 Configuration，这里面存放了全部的配置信息，它在属性里面还有各种各样的容器。

最后，返回了一个 DefaultSqlSessionFactory，里面持有了 Configuration 的实例。

## 二、会话创建过程

这是第二步，我们跟数据库的每一次连接，都需要创建一个会话，我们用 openSession()方法来创建。

DefaultSqlSessionFactory —— openSessionFromDataSource()

这个会话里面，需要包含一个 Executor 用来执行 SQL。Executor 又要指定事务类 型和执行器的类型。

所以我们会先从 Configuration 里面拿到 Enviroment，Enviroment 里面就有事务 工厂。

### 1、创建 Transaction

| 属性    | 产生工厂类                | 产生事务           |
| ------- | ------------------------- | ------------------ |
| JDBC    | JdbcTransactionFactory    | JdbcTransaction    |
| MANAGED | ManagedTransactionFactory | ManagedTransaction |

如果配置的是 JDBC，则会使用 Connection 对象的 commit()、rollback()、close() 管理事务。

如果配置成 MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因 为我们跑的是本地程序，如果配置成 MANAGE 不会有任何事务。

如果是 Spring + MyBatis，则没有必要配置，因为我们会直接在 applicationContext.xml 里面配置数据源和事务管理器，覆盖 MyBatis 的配置。

### 2、创建 Executor

我们知道，Executor 的基本类型有三种:SIMPLE、BATCH、REUSE，默认是 SIMPLE (settingsElement()读取默认值)，他们都继承了抽象类 BaseExecutor。

为什么要让抽象类实现接口，然后让具体实现类继承抽象类?(模板方法模式)

```java
“定义一个算法的骨架，并允许子类为一个或者多个步骤提供实现。 模板方法使得子类可以在不改变算法结构的情况下，重新定义算法的某些步骤。”
```

![image-20200623234043164](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2ngkhjv1j31fc0ga7fi.jpg)

问题:三种类型的区别(通过 update()方法对比)?

SimpleExecutor:每执行一次 update 或 select，就开启一个 Statement 对象，用 完立刻关闭 Statement 对象。

ReuseExecutor:执行 update 或 select，以 sql 作为 key 查找 Statement 对象， 存在就使用，不存在就创建，用完后，不关闭 Statement 对象，而是放置于 Map 内， 供下一次使用。简言之，就是重复使用 Statement 对象。

BatchExecutor:执行 update(没有 select，JDBC 批处理不支持 select)，将所 有 sql 都添加到批处理中(addBatch())，等待统一执行(executeBatch())，它缓存 了多个 Statement 对象，每个 Statement 对象都是 addBatch()完毕后，等待逐一执行 executeBatch()批处理。与 JDBC 批处理相同。

如果配置了 cacheEnabled=ture，会用装饰器模式对 executor 进行包装:new CachingExecutor(executor)。

包装完毕后，会执行:

```java
executor = (Executor) interceptorChain.pluginAll(executor);
```

此处会对 executor 进行包装。

回答了前面的问题:数据源和事务工厂在哪里会用到——创建执行器的时候。

最终返回 DefaultSqlSession，属性包括 Configuration、Executor 对象。

总结:创建会话的过程，我们获得了一个 DefaultSqlSession，里面包含了一个 Executor，它是 SQL 的执行者。

## 三、获得 Mapper 对象

现在我们已经有一个 DefaultSqlSession 了，必须找到 Mapper.xml 里面定义的 Statement ID，才能执行对应的 SQL 语句。

找到 Statement ID 有两种方式:一种是直接调用 session 的方法，在参数里面传入 Statement ID，这种方式属于硬编码，我们没办法知道有多少处调用，修改起来也很麻 烦。

另一个问题是如果参数传入错误，在编译阶段也是不会报错的，不利于预先发现问 题。

```java
Blog blog = (Blog) session.selectOne("com.gupaoedu.mapper.BlogMapper.selectBlogById ", 1);
```

所以在 MyBatis 后期的版本提供了第二种方式，就是定义一个接口，然后再调用 Mapper 接口的方法。

由于我们的接口名称跟 Mapper.xml 的 namespace 是对应的，接口的方法跟 statement ID 也都是对应的，所以根据方法就能找到对应的要执行的 SQL。

```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
```

在这里我们主要研究一下 Mapper 对象是怎么获得的，它的本质是什么。

DefaultSqlSession 的 getMapper()方法，调用了 Configuration 的 getMapper() 方法。

```java
configuration.<T>getMapper()
```

Configuration 的 getMapper()方法，又调用了 MapperRegistry 的 getMapper() 方法。

```java
mapperRegistry.getMapper()
```

我们知道，在解析 mapper 标签和 Mapper.xml 的时候已经把接口类型和类型对应的 MapperProxyFactory 放到了一个 Map 中。获取 Mapper 代理对象，实际上是从Map 中获取对应的工厂类后，调用以下方法创建对象:

```java
MapperProxyFactory.newInstance()
```

最终通过代理模式返回代理对象:

```java
return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
```

回答了前面的问题:为什么要保存一个工厂类，它是用来创建代理对象的。

JDK 动态代理和 MyBatis 用到的 JDK 动态代理有什么区别?

![image-20200624101405063](https://tva1.sinaimg.cn/large/007S8ZIlly1gg35rjp601j30gq048dgn.jpg)

JDK 动态代理:

![image-20200624101423155](https://tva1.sinaimg.cn/large/007S8ZIlly1gg35rv0bd4j30nq0g2go8.jpg)

JDK 动态代理代理，在实现了 InvocationHandler 的代理类里面，需要传入一个被 代理对象的实现类。

MyBatis 的动态代理:

![image-20200624101459654](https://tva1.sinaimg.cn/large/007S8ZIlly1gg35spwtenj30ne0f476h.jpg)

不需要实现类的原因:我们只需要根据接口类型+方法的名称，就可以找到 Statement ID 了，而唯一要做的一件事情也是这件，所以不需要实现类。在 MapperProxy 里面直接执行逻辑(也就是执行 SQL)就可以。

总结:

获得 Mapper 对象的过程，实质上是获取了一个 MapperProxy 的代理对象。 MapperProxy 中有 sqlSession、mapperInterface、methodCache。

![image-20200624101547188](https://tva1.sinaimg.cn/large/007S8ZIlly1gg35tbjh37j30r004m78r.jpg)

先记下这个问题:在代理类中为什么要持有一个 SqlSession?

## 四、执行 SQL

```java
Blog blog = mapper.selectBlog(1);
```

由于所有的 Mapper 都是 MapperProxy 代理对象，所以任意的方法都是执行 MapperProxy 的 invoke()方法。

问题 1:我们引入 MapperProxy 为了解决什么问题?硬编码和编译时检查问题。它 需要做的事情是:根据方法查找 Statement ID 的问题。

问题 2:这里没有实现类，进入到 invoke 方法的时候做了什么事情?它是怎么找到 我们要执行的 SQL 的?

我们看一下 invoke()方法:

1、MapperProxy.invoke()