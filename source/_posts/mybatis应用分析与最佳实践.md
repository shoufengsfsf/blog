---
layout: mybatis源码解读
title: mybatis应用分析与最佳实践
date: 2020-06-22 18:20:22
tags: mybatis应用
categories: mybatis源码解读
---

# mybatis应用分析与最佳实践

## 1.为什么要用 MyBatis

### JDBC 连接数据库

在 Java 程序里面去连接数据库，最原始的办法是使用 JDBC 的 API。我们先来回顾 一下使用 JDBC 的方式，我们是怎么操作数据库的。

```java
// 注册 JDBC 驱动
Class.forName("com.mysql.jdbc.Driver"); // 打开连接
conn = DriverManager.getConnection(DB_URL, USER, PASSWORD);
// 执行查询
stmt = conn.createStatement();
String sql= "SELECT bid, name, author_id FROM blog"; ResultSet rs = stmt.executeQuery(sql);
// 获取结果集
while(rs.next()){
int bid = rs.getInt("bid");
String name = rs.getString("name");
String authorId = rs.getString("author_id");
}
```

1. 我们在 maven 中引入 MySQL 驱动的依赖(JDBC 的包在 java.sql 中)。

2. 注册驱动，第二步，通过 DriverManager 获取一个 Connection，参数里 面填数据库地址，用户名和密码。

3. 我们通过 Connection 创建一个 Statement 对象。

4. 通过 Statement 的 execute()方法执行 SQL。当然 Statement 上面定义了 非常多的方法。execute()方法返回一个 ResultSet 对象，我们把它叫做结果集。

5. 我们通过 ResultSet 获取数据。转换成一个 POJO 对象。

6. 我们要关闭数据库相关的资源，包括 ResultSet、Statement、Connection， 它们的关闭顺序和打开的顺序正好是相反的。

这个就是我们通过 JDBC 的 API 去操作数据库的方法，这个仅仅是一个查询。如果 我们项目当中的业务比较复杂，表非常多，各种操作数据库的增删改查的方法也比较多 的话，那么这样代码会重复出现很多次。

在每一段这样的代码里面，我们都需要自己去管理数据库的连接资源，如果忘记写 close()了，就可能会造成数据库服务连接耗尽。

另外还有一个问题就是处理业务逻辑和处理数据的代码是耦合在一起的。如果业务 流程复杂，跟数据库的交互次数多，耦合在代码里面的 SQL 语句就会非常多。如果要修改业务逻辑，或者修改数据库环境(因为不同的数据库 SQL 语法略有不同)，这个工作 量是也是难以估计的。

还有就是对于结果集的处理，我们要把 ResultSet 转换成 POJO 的时候，必须根据 字段属性的类型一个个地去处理，写这样的代码是非常枯燥的:

```java
int bid = rs.getInt("bid");
String name = rs.getString("name");
String authorId = rs.getString("author_id"); blog.setAuthorId(authorId); blog.setBid(bid);
blog.setName(name);
```

也正是因为这样，我们在实际工作中是比较少直接使用 JDBC 的。那么我们在 Java 程序里面有哪些更加简单的操作数据库的方式呢?

### Apache DbUtils

https://commons.apache.org/proper/commons-dbutils/

DbUtils 解决的最核心的问题就是结果集的映射，可以把 ResultSet 封装成 JavaBean。它是怎么做的呢?

首先 DbUtils 提供了一个 QueryRunner 类，它对数据库的增删改查的方法进行了封 装，那么我们操作数据库就可以直接使用它提供的方法。

在 QueryRunner 的构造函数里面，我们又可以传入一个数据源，比如在这里我们 Hikari，这样我们就不需要再去写各种创建和释放连接的代码了。

```java
queryRunner = new QueryRunner(dataSource);
```

那我们怎么把结果集转换成对象呢?比如实体类 Bean 或者 List 或者 Map?在DbUtils 里面提供了一系列的支持泛型的 ResultSetHandler。

![image-20200622221548876](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1fdwe62gj30dk0jgn63.jpg)

我们只要在 DAO 层调用 QueryRunner 的查询方法，传入这个 Handler，它就可以 自动把结果集转换成实体类 Bean 或者 List 或者 Map。

```java
String sql = "select * from blog";
List<BlogDto> list = queryRunner.query(sql, new BeanListHandler<>(BlogDto.class));
```

没有用过 DbUtils 的同学，可以思考一下通过结果集到实体类的映射是怎么实现的? 也就是说，我只传了一个实体类的类型，它怎么知道这个类型有哪些属性，每个属性是 什么类型?然后创建这个对象并且给这些字段赋值的?答案正是反射。

大家也可以去看一下源码映证一下是不是这样。

问题:输出的结果中，authorId 为什么是空的?DbUtils 要求数据库的字段跟对象 的属性名称完全一致，才可以实现自动映射。

```java
BlogDto{bid=3, name='MyBatis 源码分析', authorId='null'}
```

### Spring JDBC

除了 DbUtils 之外，Spring 也对原生的 JDBC 进行了封装，并且给我们提供了一个 模板方法 JdbcTemplate，来简化我们对数据库的操作。

第一个，我们不再需要去关心资源管理的问题。
第二个，对于结果集的处理，Spring JDBC 也提供了一个 RowMapper 接口，可以把结果集转换成 Java 对象。
看代码:比如我们要把结果集转换成 Employee 对象，就可以针对一个 Employee

创建一个 RowMapper 对象，实现 RowMapper 接口，并且重写 mapRow()方法。我们 在 mapRow()方法里面完成对结果集的处理。

```java
public class EmployeeRowMapper implements RowMapper { 
  @Override
  public Object mapRow(ResultSet resultSet, int i) throws SQLException { 
    Employee employee = new Employee(); 
    employee.setEmpId(resultSet.getInt("emp_id"));
    employee.setEmpName(resultSet.getString("emp_name")); 
    employee.setEmail(resultSet.getString("emial"));
    return employee;
  }
}
```

在 DAO 层调用的时候就可以传入自定义的 RowMapper 类，最终返回我们需要的 类型。结果集和实体类类型的映射也是自动完成的。

```java
public List<Employee> query(String sql){
  new JdbcTemplate( new DruidDataSource());
  return jdbcTemplate.query(sql,new EmployeeRowMapper());
}
```

通过这种方式，我们对于结果集的处理只需要写一次代码，然后在每一个需要映射 的地方传入这个 RowMapper 就可以了，减少了很多的重复代码。

但是还是有问题:每一个实体类对象，都需要定义一个 Mapper，然后要编写每个 字段映射的 getString()、getInt 这样的代码，还增加了类的数量。

所以有没有办法让一行数据的字段，跟实体类的属性自动对应起来，实现自动映射 呢?当然，我们肯定要解决两个问题，一个就是名称对应的问题，从下划线到驼峰命名;

第二个是类型对应的问题，数据库的 JDBC 类型和 Java 对象的类型要匹配起来。 我们可以创建一个 BaseRowMapper<T>，通过反射的方式自动获取所有属性，把

表字段全部赋值到属性。 上面的方法就可以改成:

```java
return jdbcTemplate.query(sql,new BaseRowMapper(Employee.class));
```

这样，我们在使用的时候只要传入我们需要转换的类型就可以了，不用再单独创建一个 RowMapper。

我们来总结一下，DbUtils 和 Spring JDBC，这两个对 JDBC 做了轻量级封装的框架， 或者说工具类里面，都帮助我们解决了一些问题:

1. 无论是 QueryRunner 还是 JdbcTemplate，都可以传入一个数据源进行初始 化，也就是资源管理这一部分的事情，可以交给专门的数据源组件去做，不用 我们手动创建和关闭;

2. 对操作数据的增删改查的方法进行了封装;
3.  可以帮助我们映射结果集，无论是映射成 List、Map 还是实体类。 

但是还是存在一些缺点:

1. SQL 语句都是写死在代码里面的，依旧存在硬编码的问题;

2. 参数只能按固定位置的顺序传入(数组)，它是通过占位符去替换的，

   不能自动映射;

3.  在方法里面，可以把结果集映射成实体类，但是不能直接把实体类映射

   成数据库的记录(没有自动生成 SQL 的功能);

4.  查询没有缓存的功能。

### Hibernate

要解决这些问题，使用这些工具类还是不够的，要用到我们今天讲的 ORM 框架。 那什么是 ORM?为什么叫 ORM?
 ORM 的全拼是 Object Relational Mapping，也就是对象与关系的映射，对象是程

序里面的对象，关系是它与数据库里面的数据的关系。也就是说，ORM 框架帮助我们解 决的问题是程序对象和关系型数据库的相互映射的问题。

O:对象——M:映射——R:关系型数据库

![image-20200622222135430](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1fjxgwguj31180gon33.jpg)

今天听课的同学应该有很多同学是用过 Hibernate 或者现在还在用的。Hibernate 是一个很流行的 ORM 框架，2001 年的时候就出了第一个版本。在使用 Hibernate 的时 候，我们需要为实体类建立一些 hbm 的 xml 映射文件(或者类似于@Table 的这样的注 解)。例如:

```xml
<hibernate-mapping>
  <class name="cn.gupaoedu.vo.User" table="user">
    <id name="id">
      <generator class="native"/>
    </id>
    <property name="password"/> <property name="cellphone"/>
    <property name="username"/> </class>
</hibernate-mapping>
```

然后通过 Hibernate 提供(session)的增删改查的方法来操作对象。

```java
//创建对象
User user = new User(); 
user.setPassword("123456"); 
user.setCellphone("18166669999"); 
user.setUsername("qingshan");
//获取加载配置管理类
Configuration configuration = new Configuration(); 
//不给参数就默认加载 hibernate.cfg.xml 文件， 
configuration.configure();
//创建 Session 工厂对象
SessionFactory factory = configuration.buildSessionFactory(); 
//得到 Session 对象
Session session = factory.openSession();
//使用 Hibernate 操作数据库，都要开启事务,得到事务对象
Transaction transaction = session.getTransaction();
//开启事务
transaction.begin();
//把对象添加到数据库中
session.save(user);
//提交事务 
transaction.commit();
//关闭 
Session session.close();
```

我们操作对象就跟操作数据库的数据一样。Hibernate 的框架会自动帮我们生成 SQL 语句(可以屏蔽数据库的差异)，自动进行映射。这样我们的代码变得简洁了，程序的 可读性也提高了。

但是 Hibernate 在业务复杂的项目中使用也存在一些问题:

1. 比如使用 get()、save() 、update()对象的这种方式，实际操作的是所有字段，没有办法指定部分字段，换句话说就是不够灵活。

2. 这种自动生成 SQL 的方式，如果我们要去做一些优化的话，是非常困难的，也就是说可能会出现性能比较差的问题。

3. 不支持动态 SQL(比如分表中的表名变化，以及条件、参数)。

### MyBatis

“半自动化”的 ORM 框架 MyBatis 就解决了这几个问题。“半自动化”是相对于 Hibernate 的全自动化来说的，也就是说它的封装程度没有 Hibernate 那么高，不会自 动生成全部的 SQL 语句，主要解决的是 SQL 和对象的映射问题。

在 MyBatis 里面，SQL 和代码是分离的，所以会写 SQL 基本上就会用 MyBatis，没有额外的学习成本。

我们来总结一下，MyBatis 的核心特性，或者说它解决的主要问题是什么:

1. 使用连接池对连接进行管理
2. SQL 和代码分离，集中管理
3. 结果集映射
4. 参数映射和动态 SQL
5. 重复 SQL 的提取
6. 缓存管理
7. 插件机制

当然，需要明白的是，Hibernate 和 MyBatis 跟 DbUtils、Spring JDBC 一样，都是对 JDBC 的一个封装，我们去看源码，最后一定会看到 Statement 和 ResultSet 这些 对象。

问题来了，我们有这么多的工具和不同的框架，在实际的项目里面应该怎么选择? 在一些业务比较简单的项目中，我们可以使用 Hibernate;
如果需要更加灵活的 SQL，可以使用 MyBatis，对于底层的编码，或者性能要求非常高的场合，可以用 JDBC。
实际上在我们的项目中，MyBatis 和 Spring JDBC 是可以混合使用的。 当然，我们也根据项目的需求自己写 ORM 框架，就像之前 Tom 老师跟大家讲的手写 ORM 框架一样。

## 2.MyBatis 实际使用案例

### 编程式使用

大部分时候，我们都是在 Spring 里面去集成 MyBatis。因为 Spring 对 MyBatis 的 一些操作进行的封装，我们不能直接看到它的本质，所以先看下不使用容器的时候，也 就是编程的方式，MyBatis 怎么使用。

先引入 mybatis jar 包。

首先我们要创建一个全局配置文件，这里面是对 MyBatis 的核心行为的控制，比如 mybatis-config.xml。

第二个就是我们的映射器文件，Mapper.xml，通常来说一张表对应一个，我们会在 这个里面配置我们增删改查的 SQL 语句，以及参数和返回的结果集的映射关系。

跟 JDBC 的代码一样，我们要执行对数据库的操作，必须创建一个会话，这个在 MyBatis 里面就是 SqlSession。SqlSession 又是工厂类根据全局配置文件创建的。所以 整个的流程就是这样的(如下代码)。最后我们通过 SqlSession 接口上的方法，传入我 们的 Statement ID 来执行 SQL。这是第一种方式。

这种方式有一个明显的缺点，就是会对 Statement ID 硬编码，而且不能在编译时进 行类型检查，所以通常我们会使用第二种方式，就是定义一个 Mapper 接口的方式。这 个接口全路径必须跟 Mapper.xml 里面的 namespace 对应起来，方法也要跟 Statement ID 一一对应。

```java
public void testMapper() throws IOException {
  String resource = "mybatis-config.xml";
  InputStream inputStream = Resources.getResourceAsStream(resource); 
  SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
  SqlSession session = sqlSessionFactory.openSession(); 
  try {
    BlogMapper mapper = session.getMapper(BlogMapper.class); 
    Blog blog = mapper.selectBlogById(1); 
    System.out.println(blog);
  } finally { 
    session.close();
  } 
}
```

这个就是我们单独使用 MyBatis 的全部流程。 这个案例非常重要，后面我们讲源码还是基于它。

### 核心对象的生命周期

在编程式使用的这个 demo 里面，我们看到了 MyBatis 里面的几个核心对象: SqlSessionFactoryBuiler、SqlSessionFactory、SqlSession 和 Mapper 对象。这几个 核心对象在 MyBatis 的整个工作流程里面的不同环节发挥作用。如果说我们不用容器，自己去管理这些对象的话，我们必须思考一个问题:什么时候创建和销毁这些对象? 在一些分布式的应用里面，多线程高并发的场景中，如果要写出高效的代码，必须

了解这四个对象的生命周期。这四个对象的声明周期的描述在官网上面也可以找到。 http://www.mybatis.org/mybatis-3/zh/getting-started.html 我们从每个对象的作用的角度来理解一下，只有理解了它们是干什么的，才知道什么时候应该创建，什么时候应该销毁。

**1)SqlSessionFactoryBuiler**

首先是 SqlSessionFactoryBuiler。它是用来构建 SqlSessionFactory 的，而 SqlSessionFactory 只需要一个，所以只要构建了这一个 SqlSessionFactory，它的使命 就完成了，也就没有存在的意义了。所以它的生命周期只存在于方法的局部。

**2)SqlSessionFactory**

SqlSessionFactory 是用来创建 SqlSession 的，每次应用程序访问数据库，都需要 创建一个会话。因为我们一直有创建会话的需要，所以 SqlSessionFactory 应该存在于 应用的整个生命周期中(作用域是应用作用域)。创建 SqlSession 只需要一个实例来做 这件事就行了，否则会产生很多的混乱，和浪费资源。所以我们要采用单例模式。

**3)SqlSession**

SqlSession 是一个会话，因为它不是线程安全的，不能在线程间共享。所以我们在 请求开始的时候创建一个 SqlSession 对象，在请求结束或者说方法执行完毕的时候要及时关闭它(一次请求或者操作中)。

**4)Mapper**

Mapper(实际上是一个代理对象)是从 SqlSession 中获取的。

```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
```

它的作用是发送 SQL 来操作数据库的数据。它应该在一个 SqlSession 事务方法之内。

最后总结如下:

| 对象                    | 生命周期                   |
| ----------------------- | -------------------------- |
| SqlSessionFactoryBuiler | 方法局部(method)           |
| SqlSessionFactory(单例) | 应用级别(application)      |
| SqlSession              | 请求和操作(request/method) |
| Mapper                  | 方法(method)               |

这个就是我们在编程式的使用里面看到的四个对象的生命周期的总结。

### 核心配置解读

第一个是 config 文件。大部分时候我们只需要很少的配置就可以让 MyBatis 运行起 来。其实 MyBatis 里面提供的配置项非常多，我们没有配置的时候使用的是系统的默认值。

#### 一级标签

##### configuration

configuration 是整个配置文件的根标签，实际上也对应着 MyBatis 里面最重要的 配置类 Configuration。它贯穿 MyBatis 执行流程的每一个环节。我们打开这个类看一 下，这里面有很多的属性，跟其他的子标签也能对应上。

注意:MyBatis 全局配置文件顺序是固定的，否则启动的时候会报错。 (一级标签要求全部掌握)

##### properties

第一个是 properties 标签，用来配置参数信息，比如最常见的数据库连接信息。

为了避免直接把参数写死在 xml 配置文件中，我们可以把这些参数单独放在 properties 文件中，用 properties 标签引入进来，然后在 xml 配置文件中用${}引用就 可以了。

可以用 resource 引用应用里面的相对路径，也可以用 url 指定本地服务器或者网络 的绝对路径。

我们为什么要把这些配置独立出来?有什么好处?或者说，公司的项目在打包的时 候，有没有把 properties 文件打包进去?

1. 提取，利于多处引用，维护简单;
2. 把配置文件放在外部，避免修改后重新编译打包，只需要重启应用;
3.  程序和配置分离，提升数据的安全性，比如生产环境的密码只有运维人员掌握。

##### setttings 

setttings 里面是 MyBatis 的一些核心配置，我们最后再看，先看下其他的以及标签。

##### typeAliases

TypeAlias 是类型的别名，跟 Linux 系统里面的 alias 一样，主要用来简化全路径类 名的拼写。比如我们的参数类型和返回值类型都可能会用到我们的 Bean，如果每个地方 都配置全路径的话，那么内容就比较多，还可能会写错。

我们可以为自己的 Bean 创建别名，既可以指定单个类，也可以指定一个 package， 自动转换。配置了别名以后，只需要写别名就可以了，比如 com.gupaoedu.domain.Blog 都只要写 blog 就可以了。

MyBatis 里面有系统预先定义好的类型别名，在 TypeAliasRegistry 中。

##### typeHandlers【重点】

由于 Java 类型和数据库的 JDBC 类型不是一一对应的(比如 String 与 varchar)， 所以我们把 Java 对象转换为数据库的值，和把数据库的值转换成 Java 对象，需要经过 一定的转换，这两个方向的转换就要用到 TypeHandler。

有的同学可能会有疑问，我没有做任何的配置，为什么实体类对象里面的一个 String 属性，可以保存成数据库里面的 varchar 字段，或者保存成 char 字段?

这是因为 MyBatis 已经内置了很多 TypeHandler(在 type 包下)，它们全部全部 注册在 TypeHandlerRegistry 中，他们都继承了抽象类 BaseTypeHandler，泛型就是要处理的 Java 数据类型。

![image-20200622223731401](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1g0h43jkj30eu0d4jza.jpg)

当我们做数据类型转换的时候，就会自动调用对应的 TypeHandler 的方法。

如果我们需要自定义一些类型转换规则，或者要在处理类型的时候做一些特殊的动 作，就可以编写自己的 TypeHandler，跟系统自定义的 TypeHandler 一样，继承抽象类 BaseTypeHandler\<T>。有 4 个抽象方法必须实现，我们把它分成两类:

set 方法从 Java 类型转换成 JDBC 类型的，get 方法是从 JDBC 类型转换成 Java 类 型的。

| 从 Java 类型到 JDBC 类型         | 从 JDBC 类型到 Java 类型                                     |
| -------------------------------- | ------------------------------------------------------------ |
| setNonNullParameter:设置非空参数 | getNullableResult:获取空结果集(根据列名)，一般都是调用这个 getNullableResult:获取空结果集(根据下标值) getNullableResult:存储过程用的 |

比如我们想要在获取或者设置 String 类型的时候做一些特殊处理，我们可以写一个 String 类型的 TypeHandler(mybatis-standalone 工程)。

```java
public class MyTypeHandler extends BaseTypeHandler<String> {
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
    // 设置 String 类型的参数的时候调用，Java 类型到 JDBC 类型
    System.out.println("---------------setNonNullParameter1:"+parameter);
    ps.setString(i, parameter); 
  }
  public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 根据列名获取 String 类型的参数的时候调用，JDBC 类型到 java 类型
    System.out.println("---------------getNullableResult1:"+columnName);
    return rs.getString(columnName); 
  }
}
```

第二步，在 mybatis-config.xml 文件中注册:

```xml
<typeHandlers>
  <typeHandler handler="com.gupaoedu.type.MyTypeHandler"></typeHandler>
</typeHandlers>
```

第三步，在我们需要使用的字段上指定，比如:插入值的时候，从 Java 类型到 JDBC 类型，在字段属性中指定 typehandler:

```xml
<insert id = "insertBlog" parameterType = "com.gupaoedu.domain.Blog"> 
  insert into blog (bid, name, author_id)
  values (#{bid,jdbcType=INTEGER},
  #{name,jdbcType=VARCHAR,typeHandler=com.gupaoedu.type.MyTypeHandler}, #{authorId,jdbcType=INTEGER})
</insert>
```

返回值的时候，从 JDBC 类型到 Java 类型，在 resultMap 的列上指定 typehandler:

```xml
<result column="name" property="name" jdbcType="VARCHAR" typeHandler="com.gupaoedu.type.MyTypeHandler"/>
```

【思考，不强制要求完成】 

如果我们的对象里面有复杂对象，比如 Blog 里面包括了一个 Comment 对象，这个时候 Comment 对象的全部属性不能直接映射到数据库的一个字段。 

要求：创建一个 TypeHandler，可以将任意的对象转换为 json 字符串，保存到数据库的 VARCHAR 类型中。在从数据库查询的时候，再转换为原来的 Java 对象。 

1. 在数据库表添加一个 VARCHAR 字段； 

2. 在 Blog 对象中添加一个 Comment 属性，字段 Integer id;String content； 

3. JSON 工具没有要求，jackson 或者 fastjson、gson 都可以。 

4. 在查询和插入的 statement 上使用这个 TypeHandler。 

##### **objectFactory【重点】**

当我们把数据库返回的结果集转换为实体类的时候，需要创建对象的实例，由于我们不知道需要处理的类型是什么，有哪些属性，所以不能用 new 的方式去创建。在MyBatis 里面，它提供了一个工厂类的接口，叫做 ObjectFactory，专门用来创建对象的 实例，里面定义了 4 个方法。 

![image-20200622225125122](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1geydofhj30s0078dmk.jpg)

| **方法**                                                     | **作用**                       |
| ------------------------------------------------------------ | ------------------------------ |
| **void** setProperties(Properties properties);               | 设置参数时调用                 |
| \<T> T create(Class\<T> type);                               | 创建对象（调用无参构造函数）   |
| \<T> T create(Class\<T> type, List<Class<?>> constructorArgTypes, List\<Object> constructorArgs); | 创建对象（调用带参数构造函数） |
| \<T> **boolean** isCollection(Class\<T> type)                | 判断是否集合                   |

ObjectFactory 有一个默认的实现类 DefaultObjectFactory，创建对象的方法最终都调用了 instantiateClass()，是通过反射来实现的。 

如果想要修改对象工厂在初始化实体类的时候的行为，就可以通过创建自己的对象 工厂，继承 DefaultObjectFactory 来实现（不需要再实现 ObjectFactory 接口）。 

例如：

```java
public class GPObjectFactory extends DefaultObjectFactory {
  @Override public Object create(Class type) { 
    if (type.equals(Blog.class)) { 
      Blog blog = (Blog) super.create(type); 
      blog.setName("by object factory"); 
      blog.setBid(1111); 
      blog.setAuthorId(2222); 
      return blog; 
    }
    Object result = super.create(type); 
    return result; 
  } 
}
```

我们可以直接用自定义的工厂类来创建对象：

```java
public class ObjectFactoryTest { 
  public static void main(String[] args) { 
    GPObjectFactory factory = new GPObjectFactory(); 
    Blog myBlog = (Blog) factory.create(Blog.class); 
    System.out.println(myBlog); } 
}
```

这样我们就直接拿到了一个对象。 

如果在 config 文件里面注册，在创建对象的时候会被自动调用：

```xml
<objectFactory type="org.mybatis.example.GPObjectFactory">
  <!-- 对象工厂注入的参数 --> 
  <property name="gupao" value="666"/> 
</objectFactory>
```

这样，就可以让 MyBatis 的创建实体类的时候使用我们自己的对象工厂。 

应用场景举例： 

比如有一个新锐手机品牌在一个电商平台上面卖货，为了让预约数量好看一点，只要有人预约，预约数量就自动乘以 3。这个时候就可以创建一个 ObjectFactory，只要是查询销量，就把它的预约数乘以 3 返回这个实体类。 被发现后，平台：是程序员干的。

附：

1. 什么时候调用了 objectFactory.create()？ 

   创建 DefaultResultSetHandler 的时候，和创建对象的时候。 

2. 创建对象后，已有的属性为什么被覆盖了？ 

   在 DefaultResultSetHandler 类的 395 行 getRowValue()方法里面里面调用了applyPropertyMappings()。 

3. 返回结果的时候，ObjectFactory 和 TypeHandler 哪个先工作？ 

   先是 ObjectFactory，再是 TypeHandler。肯定是先创建对象。 

PS：step out 可以看到一步步调用的层级。 

##### **plugins**

插件是 MyBatis 的一个很强大的机制，跟很多其他的框架一样，MyBatis 预留了插件的接口，让 MyBatis 更容易扩展。

根据官方的定义，插件可以拦截这四个对象的这些方法，我们把这四个对象称作MyBatis 的四大对象。我们会在带大家阅读源码，知道了这 4 大对象的作用之后，再来分析自定义插件的开发和插件运行的原理。 

http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins

| **类（或接口）** | **方法**                                                     |
| ---------------- | ------------------------------------------------------------ |
| Executor         | update, query, flushStatements, commit, rollback, getTransaction, close, isClosed |
| ParameterHandler | getParameterObject, setParameters                            |
| ResultSetHandler | handleResultSets, handleOutputParameters                     |
| StatementHandler | prepare, parameterize, batch, update, query                  |

##### **environments、environment**

environments 标签用来管理数据库的环境，比如我们可以有开发环境、测试环境、生产环境的数据库。可以在不同的环境中使用不同的数据库地址或者类型。

```xml
<environments default="development"> 
  <environment id="development"> 
    <transactionManager type="JDBC"/> 
    <dataSource type="POOLED"> 
      <property name="driver" value="com.mysql.jdbc.Driver"/> 
      <property name="url" value="jdbc:mysql://127.0.0.1:3306/gp-mybatis?useUnicode=true"/> 
      <property name="username" value="root"/> <property name="password" value="123456"/> 
    </dataSource> 
  </environment> 
</environments>
```

一个 environment 标签就是一个数据源，代表一个数据库。这里面有两个关键的标 签，一个是事务管理器，一个是数据源。 

##### **transactionManager**

如果配置的是 JDBC，则会使用 Connection 对象的 commit()、rollback()、close() 管理事务。

如果配置成 MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因为我们跑的是本地程序，如果配置成 MANAGE 不会有任何事务。 

如 果 是 Spring + MyBatis ， 则 没 有 必 要 配 置 ， 因 为 我 们 会 直 接 在applicationContext.xml 里面配置数据源，覆盖 MyBatis 的配置。

##### **dataSource**

将在下一节（settings）详细分析。在跟 Spring 集成的时候，事务和数据源都会交给 Spring 来管理。

##### **mappers** 

\<mappers>标签配置的是我们的映射器，也就是 Mapper.xml 的路径。这里配置的 

目的是让 MyBatis 在启动的时候去扫描这些映射器，创建映射关系。 

我们有四种指定 Mapper 文件的方式： 

http://www.mybatis.org/mybatis-3/zh/configuration.html#mappers 

1. 使用相对于类路径的资源引用（resource） 

2. 使用完全限定资源定位符（绝对路径）（URL） 

3. 使用映射器接口实现类的完全限定类名 

4. 将包内的映射器接口实现全部注册为映射器（最常用） 

思考：

接口跟 statement 是怎么绑定起来的？——method 有方法全限定名，比如： com.gupaoedu.mapper.BlogMapper.selectBlogById ， 跟 namespace 里 面 的 statement ID 是相同的。 

在哪一步拿到 SQL 的？——ms 里面有 SQL。

```java
// DefaultSqlSession. selectList() 
MappedStatement ms = configuration.getMappedStatement(statement);
```

##### **settings**

最后 settings 我们来单独说一下，因为 MyBatis 的一些最关键的配置都在这个标签里面（只讲解一些主要的）。

| **属性名**              | **含义**                                                     | **简介**                                                     | **有效值**                                                   | **默认值**                         |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------- |
| cacheEnabled            | 是否使用缓存                                                 | 是整个工程中所有映射器配置缓存的 开关，即是一个全局缓存开关  | true/false                                                   | true                               |
| lazyLoadingEnabled      | 是否开启延迟加载                                             | 控制全局是否使用延迟加载 （association、collection）。当有特殊关联关系需要单独配置时，可以使用 fetchType 属性来覆盖此配置 | true/false                                                   | false                              |
| aggressiveLazyLoading   | 是否需要侵入式延迟加载                                       | 开启时，无论调用什么方法加载某个对 象，都会加载该对象的所有属性，关闭之后只会按需加载 | true/false                                                   | false                              |
| defaultExecutorType     | 设置默认的执行器                                             | 有三种执行器：SIMPLE 为普通执行器； REUSE 执行器会重用与处理语句；BATCH 执行器将重用语句并执行批量更新 | SIMPLE/REUSE/BATCH                                           | SIMPLE                             |
| lazyLoadTriggerMeth ods | 指定哪个对象的方 法触发一次延迟加载                          | 配置需要触发延迟加载的方法的名字， 该方法就会触发一次延迟加载 | 一个逗号分隔的方 法名称列表                                  | equals， clone，hashCode，toString |
| localCacheScope         | MyBatis 利用本地 缓存机制（Local Cache）防止循环引 用（circular references）和加 速重复嵌套查询 | 默认值为 SESSION，这种情况下会缓存 一个会话中执行的所有查询。若设置值 为 STATEMENT，本地会话仅用在语句执 行上，对相同 SqlSession 的不同调用 将不会共享数据 | SESSION/STATEMENT                                            | SESSION                            |
| logImpl                 | 日志实现                                                     | 指定 MyBatis 所用日志的具体实现，未 指定时将自动查找         | SLF4J、LOG4J、 LOG4J2、 JDK_LOGGING、 COMMONS_LOGGING、 STDOUT_LOGGING、 NO_LOGGING | 无                                 |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |
|                         |                                                              |                                                              |                                                              |                                    |

##### **Mapper.xml 映射配置文件【重点】**

http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html 

映射器里面最主要的是配置了 SQL 语句，也解决了我们的参数映射和结果集映射的 问题。

一共有 8 个标签： 

cache – 给定命名空间的缓存配置（是否开启二级缓存）。 

cache-ref – 其他命名空间缓存配置的引用。这两个标签我们在讲解缓存的时候会详细讲到。

resultMap – 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。

```java
<resultMap id="BaseResultMap" type="Employee"> 
  <id column="emp_id" jdbcType="INTEGER" property="empId"/> 
  <result column="emp_name" jdbcType="VARCHAR" property="empName"/> 
  <result column="gender" jdbcType="CHAR" property="gender"/> 
  <result column="email" jdbcType="VARCHAR" property="email"/> 
  <result column="d_id" jdbcType="INTEGER" property="dId"/> 
</resultMap>
```

sql – 可被其他语句引用的可重用语句块。

```java
<sql id="Base_Column_List"> emp_id, emp_name, gender, email, d_id </sql>
```

增删改查标签： 

insert – 映射插入语句 

update – 映射更新语句 

delete – 映射删除语句 

select – 映射查询语句

##### **总结**

最后我们来总结一下：

![image-20200622232033369](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1h99a6lqj31f00lwn5c.jpg)

![image-20200622232045602](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1h9gflm6j31eg0ci43q.jpg)

## **3.MyBatis 最佳实践**

以下是一些 MyBatis 的高级用法或者扩展方式，帮助我们更好地使用 MyBatis。 

### **动态 SQL** 

#### **为什么需要动态 SQL？** 

由于前台传入的查询参数不同，所以写了很多的 if else，还需要非常注意 SQL 语句里面的 and、空格、逗号和转移的单引号这些，拼接和调试 SQL 就是一件非常耗时的工作。

MyBaits 的动态 SQL 就帮助我们解决了这个问题，它是基于 OGNL 表达式的。

#### **动态标签有哪些？** 

按照官网的分类，MyBatis 的动态标签主要有四类：if，choose (when, otherwise)， trim (where, set)，foreach。

（案例在 spring-mybatis 工程中） 

if —— 需要判断的时候，条件写在 test 中 

以下语句可以用\<where>改写 

```xml
<select id="selectDept" parameterType="int" resultType="com.gupaoedu.crud.bean.Department"> 
  select * from tbl_dept where 1=1 
  <if test="deptId != null"> 
    and dept_id = #{deptId,jdbcType=INTEGER} 
  </if> 
</select>
```

choose (when, otherwise) —— 需要选择一个条件的时候

```xml
<select id="getEmpList_choose" resultMap="empResultMap" parameterType="com.gupaoedu.crud.bean.Employee"> 
  SELECT * FROM tbl_emp e 
  <where> 
    <choose> 
      <when test="empId !=null"> 
        e.emp_id = #{emp_id, jdbcType=INTEGER} 
      </when> 
      <when test="empName != null and empName != ''"> 
        AND e.emp_name LIKE CONCAT(CONCAT('%', #{emp_name, jdbcType=VARCHAR}),'%') 
      </when> 
      <when test="email != null "> AND e.email = #{email, jdbcType=VARCHAR} 
      </when> 
      <otherwise> 
      </otherwise> 
    </choose> 
  </where> 
</select>
```

trim (where, set)——需要去掉 where、and、逗号之类的符号的时候。 

注意最后一个条件 dId 多了一个逗号，就是用 trim 去掉的：

```xml
<update id="updateByPrimaryKeySelective" parameterType="com.gupaoedu.crud.bean.Employee"> 
  update tbl_emp
  <set>
    <if test="empName != null"> 
      emp_name = #{empName,jdbcType=VARCHAR}, 
    </if> 
    <if test="gender != null"> 
      gender = #{gender,jdbcType=CHAR}, 
    </if> 
    <if test="email != null"> 
      email = #{email,jdbcType=VARCHAR},
    </if> 
    <if test="dId != null"> 
      d_id = #{dId,jdbcType=INTEGER}, 
    </if> 
  </set> 
  where emp_id = #{empId,jdbcType=INTEGER} 
</update>
```

trim 用来指定或者去掉前缀或者后缀：

```xml
<insert id="insertSelective" parameterType="com.gupaoedu.crud.bean.Employee"> 
  insert into tbl_emp 
  <trim prefix="(" suffix=")" suffixOverrides=","> 
    <if test="empId != null"> 
      emp_id, 
    </if> 
    <if test="empName != null"> 
      emp_name, 
    </if> 
    <if test="dId != null"> 
      d_id, 
    </if> 
  </trim> 
  <trim prefix="values (" suffix=")" suffixOverrides=","> 
    <if test="empId != null"> 
      #{empId,jdbcType=INTEGER}, 
    </if> 
    <if test="empName != null"> 
      #{empName,jdbcType=VARCHAR}, 
    </if> 
    <if test="dId != null"> 
      #{dId,jdbcType=INTEGER}, 
    </if> 
  </trim> 
</insert>
```

foreach —— 需要遍历集合的时候：

```xml
<delete id="deleteByList" parameterType="java.util.List"> 
  delete from tbl_emp where emp_id in 
  <foreach collection="list" item="item" open="(" separator="," close=")"> 
    #{item.empId,jdbcType=VARCHAR} 
  </foreach> 
</delete>
```

动态 SQL 主要是用来解决 SQL 语句生成的问题。

### **批量操作**

（spring-mybatis 工程单元测试目录，MapperTest 类） 

我们在生产的项目中会有一些批量操作的场景，比如导入文件批量处理数据的情况 （批量新增商户、批量修改商户信息），当数据量非常大，比如超过几万条的时候，在 Java 代码中循环发送 SQL 到数据库执行肯定是不现实的，因为这个意味着要跟数据库创建几万次会话，即使我们使用了数据库连接池技术，对于数据库服务器来说也是不堪重 负的。

在 MyBatis 里面是支持批量的操作的，包括批量的插入、更新、删除。我们可以直接传入一个 List、Set、Map 或者数组，配合动态 SQL 的标签，MyBatis 会自动帮我们 生成语法正确的 SQL 语句。 

比如我们来看两个例子，批量插入和批量更新。 

#### **批量插入**

批量插入的语法是这样的，只要在 values 后面增加插入的值就可以了。

```sql
insert into tbl_emp (emp_id, emp_name, gender,email, d_id) values ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? )
```

在 Mapper 文件里面，我们使用 foreach 标签拼接 values 部分的语句：

```xml
<!-- 批量插入 --> 
<insert id="batchInsert" parameterType="java.util.List" useGeneratedKeys="true"> 
  <selectKey resultType="long" keyProperty="id" order="AFTER"> 
    SELECT LAST_INSERT_ID() 
  </selectKey> 
  insert into tbl_emp (emp_id, emp_name, gender,email, d_id) values 
  <foreach collection="list" item="emps" index="index" separator=","> 
    ( #{emps.empId},#{emps.empName},#{emps.gender},#{emps.email},#{emps.dId} ) 
  </foreach> 
</insert>
```

Java 代码里面，直接传入一个 List 类型的参数。 

我们来测试一下。效率要比循环发送 SQL 执行要高得多。最关键的地方就在于减少了跟数据库交互的次数，并且避免了开启和结束事务的时间消耗。 

#### **批量更新**

批量更新的语法是这样的，通过 case when，来匹配 id 相关的字段值。

```sql
update tbl_emp 
set 
emp_name = case emp_id 
when ? then ? 
when ? then ? 
when ? then ? end , 
gender =case emp_id 
when ? then ? 
when ? then ? 
when ? then ? end , 
email = case emp_id 
when ? then ? 
when ? then ? 
when ? then ? end 
where emp_id in ( ? , ? , ? )
```

所以在 Mapper 文件里面最关键的就是 case when 和 where 的配置。

需要注意一下 open 属性和 separator 属性。

```xml
<update id="updateBatch"> 
  update tbl_emp set emp_name = 
  <foreach collection="list" item="emps" index="index" separator=" " open="case emp_id" close="end"> 
    when #{emps.empId} then #{emps.empName} 
  </foreach> 
  ,gender = 
  <foreach collection="list" item="emps" index="index" separator=" " open="case emp_id" close="end"> 
    when #{emps.empId} then #{emps.gender} 
  </foreach> 
  ,email = 
  <foreach collection="list" item="emps" index="index" separator=" " open="case emp_id" close="end"> 
    when #{emps.empId} then #{emps.email} 
  </foreach> 
  where emp_id in 
  <foreach collection="list" item="emps" index="index" separator="," open="(" close=")">
    #{emps.empId} 
  </foreach> 
</update>
```

批量删除也是类似的。 

#### **Batch Executor**

当然 MyBatis 的动态标签的批量操作也是存在一定的缺点的，比如数据量特别大的时候，拼接出来的 SQL 语句过大。 

MySQL 的服务端对于接收的数据包有大小限制，max_allowed_packet 默认是4M，需要修改默认配置才可以解决这个问题。 

```java
Caused by: com.mysql.jdbc.PacketTooBigException: Packet for query is too large (7188967 > 4194304). You can change this value on the server by setting the max_allowed_packet' variable.
```

在我们的全局配置文件中，可以配置默认的 Executor 的类型。其中有一种BatchExecutor。

```xml
<setting name="defaultExecutorType" value="BATCH" />
```

也可以在创建会话的时候指定执行器类型： 

```java
SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);
```

BatchExecutor 底层是对 JDBC ps.addBatch()的封装，原理是攒一批 SQL 以后再发 送（参考 standalone - 单元测试目录 JdbcTest.java – testJdbcBatch()）。 

问题：三种执行器的区别是什么？Simple、Reuse、Batch 

#### **嵌套（关联）查询/ N+1 / 延迟加载** 

我们在查询业务数据的时候经常会遇到跨表关联查询的情况，比如查询员工就会关 联部门（一对一），查询成绩就会关联课程（一对一），查询订单就会关联商品（一对 多），等等。 

![image-20200622234226756](https://tva1.sinaimg.cn/large/007S8ZIlly1gg1hw2a9v7j31b60qegv7.jpg)

我们映射结果有两个标签，一个是 resultType，一个是 resultMap。 

resultType 是 select 标签的一个属性，适用于返回 JDK 类型（比如 Integer、String 等等）和实体类。这种情况下结果集的列和实体类的属性可以直接映射。如果返回的字段无法直接映射，就要用 resultMap 来建立映射关系。 

对于关联查询的这种情况，通常不能用 resultType 来映射。用 resultMap 映射，要么就是修改 dto（Data Transfer Object），在里面增加字段，这个会导致增加很多无关 的字段。要么就是引用关联的对象，比如 Blog 里面包含了一个 Author 对象，这种情况下就要用到关联查询（association，或者嵌套查询），MyBatis 可以帮我们自动做结果 的映射。

一对一的关联查询有两种配置方式： 

1、嵌套结果：

（mybatis-standalone - MyBatisTest - testSelectBlogWithAuthorResult ()）

```xml
<!-- 根据文章查询作者，一对一查询的结果，嵌套查询 --> 
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

2、嵌套查询： 

（mybatis-standalone - MyBatisTest - testSelectBlogWithAuthorQuery ()）

```xml
<!-- 另一种联合查询 (一对一)的实现，但是这种方式有“N+1”的问题 --> 
<resultMap id="BlogWithAuthorQueryMap" type="com.gupaoedu.domain.associate.BlogAndAuthor"> 
  <id column="bid" property="bid" jdbcType="INTEGER"/> 
  <result column="name" property="name" jdbcType="VARCHAR"/> 
  <association property="author" javaType="com.gupaoedu.domain.Author" column="author_id" select="selectAuthor"/> 
  <!-- selectAuthor 定义在下面--> 
</resultMap> 
<!-- 嵌套查询 -->
<select id="selectAuthor" parameterType="int" resultType="com.gupaoedu.domain.Author"> 
  select author_id authorId, author_name authorName from author where author_id = #{authorId} 
</select>
```

其中第二种方式：嵌套查询，由于是分两次查询，当我们查询了员工信息之后，会再发送一条 SQL 到数据库查询部门信息。 

我们只执行了一次查询员工信息的 SQL（所谓的 1），如果返回了 N 条记录，就会再发送 N 条到数据库查询部门信息（所谓的 N），这个就是我们所说的 N+1 的问题。 

这样会白白地浪费我们的应用和数据库的性能。 

如果我们用了嵌套查询的方式，怎么解决这个问题？能不能等到使用部门信息的时 候再去查询？这个就是我们所说的延迟加载，或者叫懒加载。 

在 MyBatis 里面可以通过开启延迟加载的开关来解决这个问题。 

在 settings 标签里面可以配置： 

```xml
<!--延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。默认 false --> 
<setting name="lazyLoadingEnabled" value="true"/> 
<!--当开启时，任何方法的调用都会加载该对象的所有属性。默认 false，可通过 select 标签的 fetchType 来覆盖--> 
<setting name="aggressiveLazyLoading" value="false"/> 
<!-- Mybatis 创建具有延迟加载能力的对象所用到的代理工具，默认 JAVASSIST --> 
<setting name="proxyFactory" value="CGLIB" />
```

lazyLoadingEnabled 决定了是否延迟加载。 

aggressiveLazyLoading 决定了是不是对象的所有方法都会触发查询。 

先来测试一下（也可以改成查询列表）： 

1. 没有开启延迟加载的开关，会连续发送两次查询； 

2. 开 启 了 延 迟 加 载 的 开 关 ， 调 用 blog.getAuthor() 以 及 默 认 的（equals,clone,hashCode,toString）时才会发起第二次查询，其他方法并不会触发查询，比如 blog.getName()；

3. 如果开启了 aggressiveLazyLoading=true，其他方法也会触发查询，比如blog.getName()。 

问题：为什么可以做到延迟加载？blog.getAuthor()，只是一个获取属性的方法，里面并没有连接数据库的代码，为什么会触发对数据库的查询呢？ 

我怀疑：blog 根本不是 Blog 对象，而是被人动过了手脚！ 

把这个对象打印出来看看： 

```java
System.out.println(blog.getClass());
```

果然不对： 

```java
class com.gupaoedu.domain.associate.BlogAndAuthor_$$_jvst70_0
```

这个类的名字后面有 jvst，是 JAVASSIST 的缩写。原来到这里带延迟加载功能的对象 blog 已经变成了一个代理对象，那到底什么时候变成代理对象的？我们后面在看源码的时候再去分析，这个也先留一个作业给大家。 

【问题】当开启了延迟加载的开关，对象是怎么变成代理对象的？ 

DefaultResultSetHandler.createResultObject() 

既然是代理对象，那么必须要有一种创建代理对象的方法。我们有哪些实现动态代 理的方式？ 

这个就是为什么 settings 里面提供了一个 ProxyFactory 属性。MyBatis 默认使用 JAVASSIST 创建代理对象。也可以改为 CGLIB，这时需要引入 CGLIB 的包。 

【问题】CGLIB 和 JAVASSIST 区别是什么？

测试一下，我们把默认的 JAVASSIST 修改为 CGLIB，再打印这个对象。 

【问题】 

1、resultType 和 resultMap 的区别？ 

2、collection 和 association 的区别？