---
layout: mybatis源码解读
title: mybatis问题总结
date: 2020-06-24 10:45:10
tags: mybatis应用
categories: mybatis源码解读
---

# mybatis问题总结

## **作业汇总**

### 1、resultType 和 resultMap 的区别?

resultType 是\<select>标签的一个属性，适合简单对象(POJO、JDK 自带类型: Integer、String、Map 等)，只能自动映射，适合单表简单查询。

```xml
<select id="selectAuthor" parameterType="int" resultType="com.gupaoedu.domain.Author"> 
  select author_id authorId, author_name authorName from author where author_id = #{authorId} 
</select>
```

resultMap 是一个可以被引用的标签，适合复杂对象，可指定映射关系，适合关联 复合查询。

```xml
<resultMap id="BlogWithAuthorResultMap" type="com.gupaoedu.domain.associate.BlogAndAuthor"> 
  <id column="bid" property="bid" jdbcType="INTEGER"/> 
  <result column="name" property="name" jdbcType="VARCHAR"/> 
  <!-- 联合查询，将 author 的属性映射到 ResultMap --> 
  <association property="author" javaType="com.gupaoedu.domain.Author"> 
    <id column="author_id" property="authorId"/> 
    <result column="author_name" property="authorName"/> 
  </association> 
</resultMap>
```

### **2、collection 和 association 的区别？** 

association：一对一

```xml
<!-- 另一种联合查询(一对一)的实现，但是这种方式有“N+1”的问题 --> 
<resultMap id="BlogWithAuthorQueryMap" type="com.gupaoedu.domain.associate.BlogAndAuthor"> 
  <id column="bid" property="bid" jdbcType="INTEGER"/> 
  <result column="name" property="name" jdbcType="VARCHAR"/> 
  <association property="author" javaType="com.gupaoedu.domain.Author" column="author_id" select="selectAuthor"/> 
  <!-- selectAuthor 定义在下面 -->
</resultMap>
```

collection：一对多、多对多 

```xml
<!-- 查询文章带评论的结果（一对多） --> 
<resultMap id="BlogWithCommentMap" type="com.gupaoedu.domain.associate.BlogAndComment" extends="BaseResultMap" > 
  <collection property="comment" ofType="com.gupaoedu.domain.Comment"> 
    <id column="comment_id" property="commentId" /> 
    <result column="content" property="content" /> 
  </collection> 
</resultMap>
```

```xml
<!-- 按作者查询文章评论的结果（多对多） --> 
<resultMap id="AuthorWithBlogMap" type="com.gupaoedu.domain.associate.AuthorAndBlog" > 
  <id column="author_id" property="authorId" jdbcType="INTEGER"/> 
  <result column="author_name" property="authorName" jdbcType="VARCHAR"/> 
  <collection property="blog" ofType="com.gupaoedu.domain.associate.BlogAndComment"> 
    <id column="bid" property="bid" /> 
    <result column="name" property="name" /> 
    <result column="author_id" property="authorId" /> 
    <collection property="comment" ofType="com.gupaoedu.domain.Comment"> 
      <id column="comment_id" property="commentId" /> 
      <result column="content" property="content" /> 
    </collection> 
  </collection> 
</resultMap>
```

### **3、PrepareStatement 和 Statement 的区别？** 

两个都是接口，PrepareStatement 是继承自 Statement 的； 

Statement 处理静态 SQL，PreparedStatement 主要用于执行带参数的语句； 

PreparedStatement 的 addBatch()方法一次性发送多个查询给数据库； 

PS 相似 SQL 只编译一次（对语句进行了缓存，相当于一个函数），减少编译次数；

PS 可以防止 SQL 注入； 

MyBatis 默认值：PREPARED 

### **4、跟踪 update()流程，绘制每一步的时序图（4 个）**

自行绘制。 

### **5、总结：MyBatis 里面用到了哪些设计模式？（已讲解）**

第三次课已讲解，笔记中有。 

| **设计模式** | **类**                                                       |
| ------------ | ------------------------------------------------------------ |
| 工厂         | SqlSessionFactory、ObjectFactory、MapperProxyFactory         |
| 建造者       | XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuidler      |
| 单例模式     | SqlSessionFactory、Configuration、ErrorContext               |
| 代理模式     | 绑定：MapperProxy 延迟加载：ProxyFactory（CGLIB、JAVASSIT） 插件：Plugin Spring 集成 MyBaits：SqlSessionTemplate 的内部类 SqlSessionInterceptor MyBatis 自带连接池：PooledDataSource 管理的 PooledConnection 日志打印：ConnectionLogger、StatementLogger |
| 适配器模式   | logging 模块，对于 Log4j、JDK logging 这些没有直接实现 slf4j 接口的日志组件，需要适配器 |
| 模板方法     | BaseExecutor 与子类 SimpleExecutor、BatchExecutor、ReuseExecutor |
| 装饰器模式   | LoggingCache、LruCache 等对 PerpectualCache 的装饰 CachingExecutor 对其他 Executor 的装饰 |
| 责任链模式   | InterceptorChain                                             |

### **6、当我们传入 RowBounds 做翻页查询的时候，使用 limit 物理分页，代替原来的逻辑分页**

基于 mybatis-standalone，MyBatisTest.java —— testSelectByRowBounds() \>代码在 interceptor 包中 

### **7、在未启用日志组件的情况下，输出执行的 SQL，并且统计 SQL 的执行时间（先实现查询** **的拦截）**

\>代码在 interceptor 包中 

## **面试题总结**

### **1、MyBatis 解决了什么问题？** 

或：为什么要用 MyBatis？ 

或：MyBatis 的核心特性？ 

1）资源管理（底层对象封装和支持数据源） 

2）结果集自动映射 

3）SQL 与代码分离，集中管理 

4）参数映射和动态 SQL 

5）其他：缓存、插件等 

### **2、MyBatis 编程式开发中的核心对象及其作用？**

SqlSessionFactoryBuilder 创建工厂类 

SqlSessionFactory 创建会话 

SqlSession 提供操作接口 

MapperProxy 代理 Mapper 接口后，用于找到 SQL 执行 

### **3、Java 类型和数据库类型怎么实现相互映射？**

通过 TypeHandler，例如 Java 类型中的 String 要保存成 varchar，就会自动调用相应的 Handler。如果没有系统自带的 TypeHandler，也可以自定义。

### **4、SIMPLE/REUSE/BATCH 三种执行器的区别？** 

SimpleExecutor 使用后直接关闭 Statement：closeStatement(stmt); 

```java
// SimpleExecutor.java 
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException { 
  Statement stmt = null; 
  try{
    //中间省略…… 
  } finally {
  closeStatement(stmt); 
  } 
}
```

ReuseExecutor 放在缓存中，可复用：PrepareStatement——getStatement()

```java
// ReuseExecutor.Java 
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException { 
  //中间省略…… 
  Statement stmt = prepareStatement(handler, ms.getStatementLog()); 
  //中间省略…… 
}
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException { 
  Statement stmt; 
  //中间省略…… 
  if (hasStatementFor(sql)) { 
    stmt = getStatement(sql); 
    //中间省略…… 
  }
  private Statement getStatement(String s) { 
    return statementMap.get(s); 
  }
}
```

BatchExecutor 支持复用且可以批量执行 update()，通过 ps.addBatch()实现handler.batch(stmt); 

```java
// BatchExecutor.Java 
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException { 
  //中间省略…… 
  final Statement stmt; 
  //中间省略…… 
  stmt = statementList.get(last); 
  //中间省略…… 
  statementList.add(stmt); 
  batchResultList.add(new BatchResult(ms, sql, parameterObject)); 
}
handler.batch(stmt); 
}
```

### **7、MyBatis 一级缓存与二级缓存的区别？** 

一级缓存：在同一个会话（SqlSession）中共享，默认开启，维护在 BaseExecutor中。

二级缓存：在同一个 namespace 共享，需要在 Mapper.xml 中开启，维护在CachingExecutor 中。 

### **8、MyBaits 支持哪些数据源类型？**

UNPOOLED：不带连接池的数据源。

POOLED ： 带 连 接 池 的 数 据 源 ， 在 PooledDataSource 中 维 护PooledConnection。

PooledDataSource 的 getConnection()方法流程图：

![image-20200627161344005](https://tva1.sinaimg.cn/large/007S8ZIlly1gg6x0qv58fj318g0u0k6i.jpg)

JNDI：使用容器的数据源，比如 Tomcat 配置了 C3P0。 

自定义数据源：实现 DataSourceFactory 接口，返回一个 DataSource。 

当 MyBatis 集成到 Spring 中的时候，使用 Spring 的数据源。

### **9、关联查询的延迟加载是怎么实现的？** 

动态代理（JAVASSIST、CGLIB），在创建实体类对象时进行代理，在调用代理对象的相关方法时触发二次查询。 

### **10、MyBatis 翻页的几种方式和区别？**

逻辑翻页：通过 RowBounds 对象。 

物理翻页：通过改写 SQL 语句，可用插件拦截 Executor 实现。 

### **11、怎么解决表字段变化引起的 MBG 文件变化的问题？**

Mapper 继 承 ： 自 动 生 成 的 部 分 不 变 ， 创 建 接 口 继 承 原 接 口 ， 创 建 

MapperExt.xml。在继承接口和 MapperExt.xml 中修改。 

通用 Mapper：提供支持泛型的通用 Mapper 接口，传入对象类型。

### **13、解析全局配置文件的时候，做了什么？**

创建 Configuration，设置 Configuration 

解析 Mapper.xml，设置 MappedStatement 

### **14、没有实现类，MyBatis 的方法是怎么执行的？**

MapperProxy 代理，代理类的 invoke()方法中调用了 SqlSession.selectOne()

### **15、接口方法和映射器的 statement id 是怎么绑定起来的？** 

（怎么根据接口方法拿到 SQL 语句的？） 

MappedStatement 对象中存储了 statement 和 SQL 的映射关系 

### **16、四大对象是什么时候创建的？**

Executor：openSession() 

StatementHandler、ResultsetHandler、ParameterHandler： 

执行 SQL 时，在 SimpleExecutor 的 doQuery()中创建 

### **17、ObjectFactory 的 create()方法什么时候被调用？** 

第一次被调用，创建 DefaultResultHandler 的时候： DefaultResultSetHandler 类中： handleResultSet new DefaultResultHandler() 

第二次被调用，处理结果集的时候： 

DefaultResultSetHandler 

handleResultSets—— 

handleRowValues—— 

handleRowValuesForSimpleResultMap—— 

getRowValue—— 

createResultObject—— 

createResultObject——

### **18、MyBatis 哪些地方用到了代理模式？**

接口查找 SQL：MapperProxy 

日志输出：ConnectionLogger、StatementLogger 

连接池：PooledDataSource 管理的 PooledConnection 

延迟加载：ProxyFactory（JAVASSIST、CGLIB） 

插件：Plugin 

Spring 集成：SqlSessionTemplate 的内部类 SqlSessionInterceptor 

### **19、MyBatis 主要的执行流程？涉及到哪些对象？** 

![image-20200627161702377](https://tva1.sinaimg.cn/large/007S8ZIlly1gg6x44g99kj30wp0u0dmo.jpg)

### **20、MyBatis 插件怎么编写和使用？原理是什么？（画图）** 

使用：继承 Interceptor 接口，加上注解，在 mybatis-config.xml 中配置 

原理：动态代理，责任链模式，使用 Plugin 创建代理对象 

在被拦截对象的方法调用的时候，先走到 Plugin 的 invoke()方法，再走到 Interceptor 实现类的 intercept()方法，最后通过 Invocation.proceed()方法调用被 拦截对象的原方法 

### **21、JDK 动态代理，代理能不能被代理？**

能

### **22、MyBatis 集成到 Spring 的原理是什么？** 

SqlSessionTemplate 中有内部类SqlSessionInterceptor对DefaultSqlSession 进行代理； 

MapperFactoryBean 继 承 了 SqlSessionDaoSupport 获 取 SqlSessionTemplate； 

接口注册到 IOC 容器中的 beanClass 是 MapperFactoryBean。 

### **23、DefaulSqlSession 和 SqlSessionTemplate 的区别是什么？** 

1）为什么 SqlSessionTemplate 是线程安全的？

其内部类 SqlSessionInterceptor 的 invoke()方法中的 getSqlSession()方法： 

如果当前线程已经有存在的 SqlSession 对象，会在 ThreadLocal 的容器中拿到 SqlSessionHolder，获取 DefaultSqlSession。 

如果没有，则会 new 一个 SqlSession，并且绑定到 SqlSessionHolder，放到 ThreadLocal 中。 

SqlSessionTemplate 中在同一个事务中使用同一个 SqlSession。 

调用 closeSqlSession()关闭会话时，如果存在事务，减少 holder 的引用计数。否 则直接关闭 SqlSession。 

2）在编程式的开发中，有什么方法保证 SqlSession 的线程安全？

SqlSessionManager 同时实现了 SqlSessionFactory、SqlSession 接口，通过ThreadLocal 容器维护 SqlSession。 

## **常见问题** 

### **用注解还是用 xml 配置？** 

常用注解：@Insert、@Select、@Update、@Delete、@Param、@Results、 @Result 

在 MyBatis 的工程中，我们有两种配置 SQL 的方式。一种是在 Mapper.xml 中集中管理，一种是在 Mapper 接口上，用注解方式配置 SQL。很多同学在工作中可能两种方式都用过。那到底什么时候用 XML 的方式，什么时候用注解的方式呢？ 

注解的缺点是 SQL 无法集中管理，复杂的 SQL 很难配置。所以建议在业务复杂的项目中只使用 XML 配置的形式，业务简单的项目中可以使用注解和 XML 混用的形式。 

### **Mapper 接口无法注入或 Invalid bound statement (not found)**

我们在使用 MyBatis 的时候可能会遇到 Mapper 接口无法注入，或者 mapper statement id 跟 Mapper 接口方法无法绑定的情况。基于绑定的要求或者说规范，我们可以从这些地方去检查一下： 

1、扫描配置，xml 文件和 Mapper 接口有没有被扫描到 

2、namespace 的值是否和接口全类名一致 

3、检查对应的 sql 语句 ID 是否存在 

### **怎么获取插入的最新自动生成的 ID** 

在 MySQL 的插入数据使用自增 ID 这种场景，有的时候我们需要获得最新的自增 ID， 比如获取最新的用户 ID。常见的做法是执行一次查询，max 或者 order by 倒序获取最大的 ID（低效、存在并发问题）。在 MyBatis 里面还有一种更简单的方式： 

insert 成功之后，mybatis 会将插入的值自动绑定到插入的对象的 Id 属性中，我们用 getId 就能取到最新的 ID。 

```xml
<insert id="insert" parameterType="com.gupaoedu.domain.Blog"> 
  insert into blog (bid, name, author_id) values (#{bid,jdbcType=INTEGER}, #{name,jdbcType=VARCHAR}, #{author,jdbcType=CHAR}) 
</insert>
```

### **如何实现模糊查询 LIKE**

1、字符串拼接在 Java 代码中拼接%%（比如 name *=* **"%"** *+* name *+* **"%"**; ），直接 LIKE。因为没 有预编译，存在 SQL 注入的风险，不推荐使用。 

2、CONCAT（推荐）

```xml
<when test="empName != null and empName != ''"> 
  AND e.emp_name LIKE CONCAT(CONCAT('%', #{emp_name, jdbcType=VARCHAR}),'%') 
</when>
```

3、bind 标签

```xml
<select id="getEmpList_bind" resultType="empResultMap" parameterType="Employee"> 
  <bind name="pattern1" value="'%' + empName + '%'" /> 
  <bind name="pattern2" value="'%' + email + '%'" /> 
  SELECT * FROM tbl_emp 
  <where> 
    <if test="empId != null"> 
      emp_id = #{empId,jdbcType=INTEGER}, 
    </if> 
    <if test="empName != null and empName != ''"> 
      AND emp_name LIKE #{pattern1} 
    </if> 
    <if test="email != null and email != ''"> 
      AND email LIKE #{pattern2} 
    </if> 
  </where> 
  ORDER BY emp_id 
</select>
```

### **什么时候用#{}，什么时候用${}？**

在 Mapper.xml 里面配置传入参数，有两种写法：#{} 、${}。作为 OGNL 表达式，都可以实现参数的替换。这两种方式的区别在哪里？什么时候应该用哪一种？ 

要搞清楚这个问题，我们要先来说一下 PrepareStatement 和 Statement 的区别。 

1、两个都是接口，PrepareStatement 是继承自 Statement 的； 

2、Statement 处理静态 SQL，PreparedStatement 主要用于执行带参数的语句； 

3、PreparedStatement 的 addBatch()方法一次性发送多个查询给数据库； 

4、PS 相似 SQL 只编译一次（对语句进行了缓存，相当于一个函数）,比如语句相 

同参数不同，可以减少编译次数； 

5、PS 可以防止 SQL 注入。 

MyBatis 任意语句的默认值：PREPARED 

这两个符号的解析方式是不一样的： 

\#会解析为 Prepared Statement 的参数标记符，参数部分用？代替。传入的参数会经过类型检查和安全检查。 

（mybatis-standalone - MyBatisTest - testSelect()）

$只会做字符串替换，比如参数是咕泡学院，结果如下： 

（mybatis-standalone - MyBatisTest - selectBlogByBean ()） 

\#和$的区别： 

1、 是否能防止 SQL 注入：$方式不会对符号转义，不能防止 SQL 注入 

2、 性能：$方式没有预编译，不会缓存 

结论：

1、 能用#的地方都用# 

2、 常量的替换，比如排序条件中的字段名称，不用加单引号，可以使用$ 

### **对象属性是基本类型 int double，数据库返回 null 是报错** 

使用包装类型。如 Integer，不要使用基本类型如 int。 

### **If test !=null 失效了？** 

在实体类中使用包装类型。 

### **XML 中怎么使用特殊符号，比如小于 &** 

1、转义< < （大于可以直接写） 

2、使用\<![CDATA[ ]]>——当 XML 遇到这种格式就会把[]里面的内容原样输出，不 进行解析 

