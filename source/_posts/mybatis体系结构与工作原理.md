---
layout: mybatis源码解读
title: mybatis体系结构与工作原理
date: 2020-06-23 10:33:12
tags: mybatis应用
categories: mybatis源码解读
---

# mybatis体系结构与工作原理

## MyBatis 的工作流程分析

首先在 MyBatis 启动的时候我们要去解析配置文件，包括全局配置文件和映射器配置文件，这里面包含了我们怎么控制 MyBatis 的行为，和我们要对数据库下达的指令，也就是我们的 SQL 信息。我们会把它们解析成一个 Configuration 对象。

接下来就是我们操作数据库的接口，它在应用程序和数据库中间，代表我们跟数据库之间的一次连接:这个就是 SqlSession 对象。

我们要获得一个会话，必须有一个会话工厂 SqlSessionFactory。

SqlSessionFactory 里面又必须包含我们的所有的配置信息，所以我们会通过一个 Builder 来创建工厂类。

我们知道，MyBatis 是对 JDBC 的封装，也就是意味着底层一定会出现 JDBC 的一 些核心对象，比如执行 SQL 的 Statement，结果集 ResultSet。在 Mybatis 里面， SqlSession 只是提供给应用的一个接口，还不是 SQL 的真正的执行对象。

我们上次课提到了，SqlSession 持有了一个 Executor 对象，用来封装对数据库的操作。

在执行器 Executor 执行 query 或者 update 操作的时候我们创建一系列的对象， 来处理参数、执行 SQL、处理结果集，这里我们把它简化成一个对象:StatementHandler， 在阅读源码的时候我们再去了解还有什么其他的对象。

这个就是 MyBatis 主要的工作流程，如图:

![mybatis主要工作流程](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2ixaym3zj30pk0n4js3.jpg)

## **MyBatis** 架构分层与模块划分

在 MyBatis 的主要工作流程里面，不同的功能是由很多不同的类协作完成的，它们 分布在 MyBatis jar 包的不同的 package 里面。

我们来看一下 MyBatis 的 jar 包(基于 3.5.1)，

![image-20200623211022205](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2j43crszj30h40sw0v3.jpg)

大概有 300 多个类，这样看起来不够清楚，不知道什么类在什么环节工作，属于什 么层次。

跟 Spring 一样，MyBatis 按照功能职责的不同，所有的 package 可以分成不同的 工作层次。

我们可以把 MyBatis 的工作流程类比成餐厅的服务流程。

第一个是跟客户打交道的服务员，它是用来接收程序的工作指令的，我们把它叫做 接口层。

第二个是后台的厨师，他们根据客户的点菜单，把原材料加工成成品，然后传到窗 口。这一层是真正去操作数据的，我们把它叫做核心层。

最后就是餐厅也需要有人做后勤(比如清洁、采购、财务)，来支持厨师的工作和 整个餐厅的运营。我们把它叫做基础层。

来看一下这张图，我们根据刚才的分层，和大体的执行流程，做了这么一个总结。 当然，从不同的角度来描述，架构图的划分有所区别，这张图画起来也有很多形式。我们先从总体上建立一个印象。每一层的主要对象和主要的功能我们也给大家分析一下。

![image-20200623211140973](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2j5ihscyj31g40u0tlv.jpg)

### 接口层

首先接口层是我们打交道最多的。核心对象是 SqlSession，它是上层应用和 MyBatis 打交道的桥梁，SqlSession 上定义了非常多的对数据库的操作方法。接口层在接收到调 用请求的时候，会调用核心处理层的相应模块来完成具体的数据库操作。

### 核心处理层

接下来是核心处理层。既然叫核心处理层，也就是跟数据库操作相关的动作都是在 这一层完成的。

核心处理层主要做了这几件事:

1. 把接口中传入的参数解析并且映射成JDBC类型;

2. 解析xml文件中的SQL语句，包括插入参数，和动态SQL的生成; 
3. 执行SQL语句;
4. 处理结果集，并映射成Java对象。

插件也属于核心层，这是由它的工作方式和拦截的对象决定的。

### 基础支持层

最后一个就是基础支持层。基础支持层主要是一些抽取出来的通用的功能(实现复 用)，用来支持核心处理层的功能。比如数据源、缓存、日志、xml 解析、反射、IO、 事务等等这些功能。

这个就是 MyBatis 的主要工作流程和架构分层。接下来我们来学习一下基础层里面 的一个主要模块，缓存。我们一起来了解一下 MyBatis 一级缓存和二级缓存的区别，和 它们的工作方式，以及使用过程里面有什么注意事项。

## **MyBatis** 缓存详解

### cache 缓存

(基于 mybatis-standalone 工程)

缓存是一般的 ORM 框架都会提供的功能，目的就是提升查询的效率和减少数据库的 压力。跟 Hibernate 一样，MyBatis 也有一级缓存和二级缓存，并且预留了集成第三方 缓存的接口。

#### 缓存体系结构

MyBatis 跟缓存相关的类都在 cache 包里面，其中有一个 Cache 接口，只有一个默 认的实现类 PerpetualCache，它是用 HashMap 实现的。

除此之外，还有很多的装饰器，通过这些装饰器可以额外实现很多的功能:回收策略、日志记录、定时刷新等等。

```java
// 煎饼加鸡蛋加香肠
“装饰者模式(Decorator Pattern)是指在不改变原有对象的基础之上，将功能附加到对象上，提供了比继承更有弹性的替代方案(扩展原有对象的功能)。”
```

![image-20200623211938385](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2jdqym38j30vk0bg79s.jpg)

但是无论怎么装饰，经过多少层装饰，最后使用的还是基本的实现类(默认 PerpetualCache)。

![image-20200623212124898](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2jfldpvjj30x60fo4bf.jpg)

所有的缓存实现类总体上可分为三类:基本缓存、淘汰算法缓存、装饰器缓存。

| 缓存实现类          | 描述             | 作用                                                         | 装饰条件                                            |
| ------------------- | ---------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| 基本缓存            | 缓存基本实现类   | 默认是 PerpetualCache，也可以自定义比如 RedisCache、EhCache 等，具备基本功能的缓存类 | 无                                                  |
| LruCache            | LRU 策略的缓存   | 当缓存到达上限时候，删除最近最少使用的缓存 (Least Recently Use) | eviction="LRU"(默 认)                               |
| FifoCache           | FIFO 策略的缓存  | 当缓存到达上限时候，删除最先入队的缓存                       | eviction="FIFO"                                     |
| SoftCache WeakCache | 带清理策略的缓存 | 通过 JVM 的软引用和弱引用来实现缓存，当 JVM 内存不足时，会自动清理掉这些缓存，基于 SoftReference 和 WeakReference | eviction="SOFT" eviction="WEAK"                     |
| LoggingCache        | 带日志功能的缓存 | 比如:输出缓存命中率                                          | 基本                                                |
| SynchronizedCache   | 同步缓存         | 基于 synchronized 关键字实现，解决并发问题                   | 基本                                                |
| BlockingCache       | 阻塞缓存         | 通过在 get/put 方式中加锁，保证只有一个线程操 作缓存，基于 Java 重入锁实现 | blocking=true                                       |
| SerializedCache     | 支持序列化的缓存 | 将对象序列化以后存到缓存中，取出时反序列化                   | readOnly=false(默 认)                               |
| ScheduledCache      | 定时调度的缓存   | 在进行 get/put/remove/getSize 等操作前，判断 缓存时间是否超过了设置的最长缓存时间(默认是 一小时)，如果是则清空缓存--即每隔一段时间清 空一次缓存 | flushInterval 不为 空                               |
| TransactionalCache  | 事务缓存         | 在二级缓存中使用，可一次存入多个缓存，移除多 个缓存          | 在 TransactionalCach eManager 中用 Map 维护对应关系 |

思考:缓存对象在什么时候创建?什么情况下被装饰?

我们要弄清楚这个问题，就必须要知道 MyBatis 的一级缓存和二级缓存的工作位置 和工作方式的区别。

#### 一级缓存

##### 一级缓存(本地缓存)介绍

一级缓存也叫本地缓存，MyBatis 的一级缓存是在会话(SqlSession)层面进行缓 存的。MyBatis 的一级缓存是默认开启的，不需要任何的配置。

首先我们必须去弄清楚一个问题，在 MyBatis 执行的流程里面，涉及到这么多的对 象，那么缓存 PerpetualCache 应该放在哪个对象里面去维护?如果要在同一个会话里面 共享一级缓存，这个对象肯定是在 SqlSession 里面创建的，作为 SqlSession 的一个属 性。

DefaultSqlSession 里面只有两个属性，Configuration 是全局的，所以缓存只可能 放在 Executor 里面维护——SimpleExecutor/ReuseExecutor/BatchExecutor 的父类BaseExecutor 的构造函数中持有了 PerpetualCache。

在同一个会话里面，多次执行相同的 SQL 语句，会直接从内存取到缓存的结果，不会再发送 SQL 到数据库。但是不同的会话里面，即使执行的 SQL 一模一样(通过一个 Mapper 的同一个方法的相同参数调用)，也不能使用到一级缓存。

![image-20200623214412856](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2k3bek4xj31580iywio.jpg)

接下来我们来验证一下，MyBatis 的一级缓存到底是不是只能在一个会话里面共享， 以及跨会话(不同 session)操作相同的数据会产生什么问题。

##### 一级缓存验证

(基于 mybatis-standalone 工程，注意演示一级缓存需要先关闭二级缓存， localCacheScope 设置为 SESSION)

判断是否命中缓存:如果再次发送 SQL 到数据库执行，说明没有命中缓存;如果直 接打印对象，说明是从内存缓存中取到了结果。

1、在同一个 session 中共享

```java
BlogMapper mapper = session.getMapper(BlogMapper.class); 
System.out.println(mapper.selectBlog(1)); 
System.out.println(mapper.selectBlog(1));
```

2、不同 session 不能共享

```java
SqlSession session1 = sqlSessionFactory.openSession(); 
BlogMapper mapper1 = session1.getMapper(BlogMapper.class); 
System.out.println(mapper.selectBlog(1));
```

PS:一级缓存在 BaseExecutor 的 query()——queryFromDatabase()中存入。在 queryFromDatabase()之前会 get()。

3、同一个会话中，update(包括 delete)会导致一级缓存被清空

```java
mapper.updateByPrimaryKey(blog); 
session.commit();
System.out.println(mapper.selectBlogById(1));
```

一级缓存是在 BaseExecutor 中的 update()方法中调用 clearLocalCache()清空的 (无条件)，query 中会判断。

如果跨会话，会出现什么问题?

4、其他会话更新了数据，导致读取到脏数据(一级缓存不能跨会话共享)

```java
// 会话 2 更新了数据，会话 2 的一级缓存更新
BlogMapper mapper2 = session2.getMapper(BlogMapper.class); 
mapper2.updateByPrimaryKey(blog);
session2.commit();
// 会话 1 读取到脏数据，因为一级缓存不能跨会话共享
System.out.println(mapper1.selectBlog(1));
```

##### 一级缓存的不足

使用一级缓存的时候，因为缓存不能跨会话共享，不同的会话之间对于相同的数据 可能有不一样的缓存。在有多个会话或者分布式环境下，会存在脏数据的问题。如果要 解决这个问题，就要用到二级缓存。

【思考】一级缓存怎么命中?CacheKey 怎么构成?

【思考】一级缓存是默认开启的，怎么关闭一级缓存?

#### 二级缓存

##### 二级缓存介绍

二级缓存是用来解决一级缓存不能跨会话共享的问题的，范围是 namespace 级别 的，可以被多个 SqlSession 共享(只要是同一个接口里面的相同方法，都可以共享)， 生命周期和应用同步。

思考一个问题:如果开启了二级缓存，二级缓存应该是工作在一级缓存之前，还是 在一级缓存之后呢?二级缓存是在哪里维护的呢?

作为一个作用范围更广的缓存，它肯定是在 SqlSession 的外层，否则不可能被多个 SqlSession 共享。而一级缓存是在 SqlSession 内部的，所以第一个问题，肯定是工作 在一级缓存之前，也就是只有取不到二级缓存的情况下才到一个会话中去取一级缓存。

第二个问题，二级缓存放在哪个对象中维护呢? 要跨会话共享的话，SqlSession 本 身和它里面的 BaseExecutor 已经满足不了需求了，那我们应该在 BaseExecutor 之外创 建一个对象。

实际上 MyBatis 用了一个装饰器的类来维护，就是 CachingExecutor。如果启用了 二级缓存，MyBatis 在创建 Executor 对象的时候会对 Executor 进行装饰。

CachingExecutor 对于查询请求，会判断二级缓存是否有缓存结果，如果有就直接 返回，如果没有委派交给真正的查询器 Executor 实现类，比如 SimpleExecutor 来执行 查询，再走到一级缓存的流程。最后会把结果缓存起来，并且返回给用户。

![image-20200623221148435](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2kw0qa4rj31eu0qg199.jpg)

一级缓存是默认开启的，那二级缓存怎么开启呢?

##### 开启二级缓存的方法

第一步:在 mybatis-config.xml 中配置了(可以不配置，默认是 true)

```xml
<setting name="cacheEnabled" value="true"/>
```

只要没有显式地设置 cacheEnabled=false，都会用 CachingExecutor 装饰基本的执行器。

第二步:在 Mapper.xml 中配置\<cache/>标签:

```xml
<!-- 声明这个 namespace 使用二级缓存 --> 
<cache type="org.apache.ibatis.cache.impl.PerpetualCache" 
       size="1024" <!—最多缓存对象个数，默认 1024--> 
			 eviction="LRU" <!—回收策略--> 
			 flushInterval="120000" <!—自动刷新时间 ms，未配置时只有调用时刷新--> 
			 readOnly="false"/> <!—默认是 false（安全），改为 true 可读写时，对象必须支持序列 化 -->
```

cache 属性详解:

| 属性          | 含义                                | 取值                                                         |
| ------------- | ----------------------------------- | ------------------------------------------------------------ |
| type          | 缓存实现类                          | 需要实现 Cache 接口，默认是 PerpetualCache                   |
| size          | 最多缓存对象个数                    | 默认 1024                                                    |
| eviction      | 回收策略(缓存淘汰算法)              | LRU – 最近最少使用的:移除最长时间不被使用的对象(默认)。<br/>FIFO – 先进先出:按对象进入缓存的顺序来移除它们。<br/> SOFT – 软引用:移除基于垃圾回收器状态和软引用规则的对象。<br/> WEAK – 弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象。 |
| flushInterval | 定时自动清空缓存间隔                | 自动刷新时间，单位 ms，未配置时只有调用时刷新                |
| readOnly      | 是否只读                            | true:只读缓存;会给所有调用者返回缓存对象的相同实例。因此这些对象 不能被修改。这提供了很重要的性能优势。<br/>false:读写缓存;会返回缓存对象的拷贝(通过序列化)，不会共享。这 会慢一些，但是安全，因此默认是 false。<br/>改为 false 可读写时，对象必须支持序列化。 |
| blocking      | 是否使用可重入锁实现 缓存的并发控制 | true，会使用 BlockingCache 对 Cache 进行装饰 默认 false      |

Mapper.xml 配置了\<cache>之后，select()会被缓存。update()、delete()、insert() 会刷新缓存。

思考:如果 cacheEnabled=true，Mapper.xml 没有配置标签，还有二级缓存吗? 还会出现 CachingExecutor 包装对象吗?

只要 cacheEnabled=true 基本执行器就会被装饰。有没有配置\<cache>，决定了在启动的时候会不会创建这个 mapper 的 Cache 对象，最终会影响到 CachingExecutor

query 方法里面的判断:

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
  Cache cache = ms.getCache();
  if (cache != null) {
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

如果某些查询方法对数据的实时性要求很高，不需要二级缓存，怎么办? 我们可以在单个 Statement ID 上显式关闭二级缓存(默认是 true):

```xml
<select id="selectBlog" resultMap="BaseResultMap" useCache="false">
```

了解了二级缓存的工作位置和开启关闭的方法之后，我们也来验证一下二级缓存。 

##### **二级缓存验证**

（验证二级缓存需要先开启二级缓存）

1、 事务不提交，二级缓存不存在 

```java
BlogMapper mapper1 = session1.getMapper(BlogMapper.class); 
System.out.println(mapper1.selectBlogById(1)); 
// 事务不提交的情况下，二级缓存不会写入 
// session1.commit(); 
BlogMapper mapper2 = session2.getMapper(BlogMapper.class); 
System.out.println(mapper2.selectBlogById(1));
```

思考：为什么事务不提交，二级缓存不生效？

因为二级缓存使用 TransactionalCacheManager（TCM）来管理，最后又调用了 

TransactionalCache 的getObject()、putObject和 commit()方法，TransactionalCache 里面又持有了真正的 Cache 对象，比如是经过层层装饰的 PerpetualCache。 

在 putObject 的时候，只是添加到了 entriesToAddOnCommit 里面，只有它的 commit()方法被调用的时候才会调用 flushPendingEntries()真正写入缓存。它就是在 DefaultSqlSession 调用 commit()的时候被调用的。

2、 使用不同的 session 和 mapper，验证二级缓存可以跨 session 存在取消以上 commit()的注释 

3、 在其他的 session 中执行增删改操作，验证缓存会被刷新

```java
Blog blog = new Blog(); 
blog.setBid(1); 
blog.setName("357"); 
mapper3.updateByPrimaryKey(blog); 
session3.commit(); 
// 执行了更新操作，二级缓存失效，再次发送 SQL 查询 
System.out.println(mapper2.selectBlogById(1));
```

思考：为什么增删改操作会清空缓存？ 

在 CachingExecutor 的 update()方法里面会调用 flushCacheIfRequired(ms)，isFlushCacheRequired 就是从标签里面渠道的 flushCache 的值。而增删改操作的 flushCache 属性默认为 true。 

##### **什么时候开启二级缓存？** 

一级缓存默认是打开的，二级缓存需要配置才可以开启。那么我们必须思考一个问题，在什么情况下才有必要去开启二级缓存？ 

1、因为所有的增删改都会刷新二级缓存，导致二级缓存失效，所以适合在查询为主的应用中使用，比如历史交易、历史订单的查询。否则缓存就失去了意义。 

2、如果多个 namespace 中有针对于同一个表的操作，比如 Blog 表，如果在一个namespace 中刷新了缓存，另一个 namespace 中没有刷新，就会出现读到脏数据的情 况。所以，推荐在一个 Mapper 里面只操作单表的情况使用。 

思考：如果要让多个 namespace 共享一个二级缓存，应该怎么做？ 跨 namespace 的缓存共享的问题，可以使用\<cache-ref>来解决： 

```xml
<cache-ref namespace="com.gupaoedu.crud.dao.DepartmentMapper" />
```

cache-ref 代表引用别的命名空间的 Cache 配置，两个命名空间的操作使用的是同一个 Cache。在关联的表比较少，或者按照业务可以对表进行分组的时候可以使用。 

注意：在这种情况下，多个 Mapper 的操作都会引起缓存刷新，缓存的意义已经不大了。 

##### **第三方缓存做二级缓存**

除了 MyBatis 自带的二级缓存之外，我们也可以通过实现 Cache 接口来自定义二级缓存。

MyBatis 官方提供了一些第三方缓存集成方式，比如 ehcache 和 redis： 

https://github.com/mybatis/redis-cache 

pom 文件引入依赖： 

```xml
<dependency> 
  <groupId>org.mybatis.caches</groupId> 
  <artifactId>mybatis-redis</artifactId> 
  <version>1.0.0-beta2</version> 
</dependency>
```

Mapper.xml 配置，type 使用 RedisCache：

```xml
<cache type="org.mybatis.caches.redis.RedisCache" eviction="FIFO" flushInterval="60000" size="512" readOnly="true"/>
```

redis.properties 配置： 

```properties
host=localhost 
port=6379 
connectionTimeout=5000 
soTimeout=5000 database=0
```

Redis 作为二级缓存的验证： 

![image-20200623230224583](https://tva1.sinaimg.cn/large/007S8ZIlly1gg2mdnianvj313k0i2n74.jpg)

当然，我们也可以使用独立的缓存服务，不使用 MyBatis 自带的二级缓存。