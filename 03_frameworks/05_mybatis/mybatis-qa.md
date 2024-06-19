### 1.MyBatis是什么？
MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生类型、接口和 Java 的 POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录.

### 2.ORM是什么
ORM（Object Relational Mapping），对象关系映射，是一种为了解决关系型数据库数据与简单Java对象（POJO）的映射关系的技术。简单的说，ORM是通过使用描述对象和数据库之间映射的元数据，将程序中的对象自动持久化到关系型数据库中。

### 3.为什么说Mybatis是半自动ORM映射工具？它与全自动的区别在哪里？
Hibernate属于全自动ORM映射工具，使用Hibernate查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的。
而Mybatis在查询关联对象或关联集合对象时，需要手动编写sql来完成，所以，称之为半自动ORM映射工具。

### 4.传统JDBC开发存在的问题
JDBC是Java连接关系数据库的底层API，利用JDBC开发持久层存在诸多问题，而MyBatis正是为了解决JDBC的以下问题而生的。

* 数据库连接创建、释放频繁造成系统资源浪费从而影响系统性能，如果使用数据库链接池可解决此问题。
* Sql语句在代码中硬编码，造成代码不易维护，实际应用sql变化的可能较大，sql变动需要改变java代码。
* 使用PreparedStatement向占有位符号传参数存在硬编码，因为sql语句的where条件不一定，可能多也可能少，修改sql还要修改代码，系统不易维护。
* 对结果集解析存在硬编码（查询列名），sql变化导致解析代码变化，系统不易维护，如果能将数据库记录封装成pojo对象解析比较方便。

### 5.JDBC编程有哪些不足之处，MyBatis是如何解决这些问题的？
1、数据库连接创建、释放频繁造成系统资源浪费从而影响系统性能，如果使用数据库链接池可解决此问题。

解决：在mybatis-config.xml中配置数据链接池，使用连接池管理数据库连接。

2、Sql语句在代码中硬编码，造成代码不易维护，实际应用sql变化的可能较大，sql变动需要改变java代码。

解决：将Sql语句配置在XXXXmapper.xml文件中与java代码分离。

3、使用PreparedStatement向占有位符号传参数存在硬编码，因为sql语句的where条件不一定，可能多也可能少，修改sql还要修改代码，系统不易维护。

解决： Mybatis自动将java对象映射至sql语句。

4、对结果集解析存在硬编码（查询列名），sql变化导致解析代码变化，系统不易维护，如果能将数据库记录封装成pojo对象解析比较方便。

解决：Mybatis自动将sql执行结果映射至java对象。

### 6.MyBatis与Hibernate

MyBatis和Hibernate不同，它不完全是一个ORM框架，因为MyBatis需要程序员自己编写Sql语句，不过Mybatis可以通过XML或注解方式灵活配置要运行的sql语句，并将java对象和sql语句映射生成最终执行的sql，最后将sql执行的结果再映射生成java对象。

MyBatis学习门槛低，简单易学，程序员直接编写原生态sql，可严格控制sql执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，例如互联网软件、企业运营类软件等，因为这类软件需求变化频繁，一但需求变化要求成果输出迅速。但是灵活的前提是mybatis无法做到数据库无关性，如果需要实现支持多种数据库的软件则需要自定义多套sql映射文件，工作量大。

Hibernate对象/关系映射能力强，对于关系模型要求高的软件（例如需求固定的定制化软件）如果用Hibernate开发可以节省很多代码，提高效率。但是Hibernate的学习门槛高，要精通门槛更高，而且怎么设计O/R映射，在性能和对象模型之间如何权衡，以及怎样用好Hibernate需要具有很强的经验和能力才行。

### 7.Mybatis优缺点
**优点**

与传统的数据库访问技术相比，ORM有以下优点：

* 基于SQL语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL写在XML里，解除sql与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态SQL语句，并可重用
* 与JDBC相比，减少了50%以上的代码量，消除了JDBC大量冗余的代码，不需要手动开关连接
* 很好的与各种数据库兼容（因为MyBatis使用JDBC来连接数据库，所以只要JDBC支持的数据库MyBatis都支持）
* 提供映射标签，支持对象与数据库的ORM字段关系映射；提供对象关系映射标签，支持对象关系组件维护
* 能够与Spring很好的集成

**缺点**
* SQL语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写SQL语句的功底有一定要求
* SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库

### 8.MyBatis框架适用场景
MyBatis专注于SQL本身，是一个足够灵活的DAO层解决方案。
对性能的要求很高，或者需求变化较多的项目，如互联网项目，MyBatis将是不错的选择。

### 9.MyBatis编程步骤是什么样的？
（1）获取SqlSessionFactory对象
解析文件的每一个信息保存在Configuration中，返回包含Configuration的DefaultSqlSession；
注意：【MappedStatement】：代表一个增删改查的详细信息

（2）获取SqlSession对象
返回一个DefaultSqlSession对象，包含Executor和Configuration；
这一步会创建Executor；

（3）获取接口的代理对象（MapperProxy）
getMapper，使用MapperProxyFactory创建一个MapperProxy的代理对象，代理对象里包含了DefaultSqlSession（Executor）

（4）执行增删改查方法

（5）关闭会话

### 10.请说说MyBatis的工作原理
在学习 MyBatis 程序之前，需要了解一下 MyBatis 工作原理，以便于理解程序。MyBatis 的工作原理如下图



1. SqlMapConfig.xml，此文件作为MyBatis的全局配置文件，配置了mybatis的运行环境等信息。Mapper.xml文件即SQL映射文件，文件中配置了操作数据库的sql语句。此文件需要在SqlMapConfig.xml中加载。
2. 通过MyBatis环境等配置信息构造SqlSessionFactory即会话工厂
3. 由会话工厂创建SqlSession即会话，操作数据库需要通过SqlSession进行。
4. Mybatis底层自定义了Executor执行器接口操作数据库，Executor接口有两个实现，一个是基本执行器、一个是缓存执行器。
5. Mapped Statement也是MyBatis一个底层封装对象，它包装了MyBatis配置信息及sql映射信息等。mapper.xml文件中一个sql对应一个Mapped Statement对象，sql的id即是Mapped statement的id。
6. Mapped Statement对sql执行输入参数进行定义，包括HashMap、基本类型、pojo，Executor通过Mapped Statement在执行sql前将输入的java对象映射至sql中，输入参数映射就是jdbc编程中对PreparedStatement设置参数。
7. Mapped Statement对sql执行输出结果进行定义，包括HashMap、基本类型、Pojo，Executor通过Mapped Statement在执行sql后将输出结果映射至Java对象中，输出结果映射过程相当于Jdbc编程中对结果的解析处理过程.

### 为什么需要预编译
1.定义：
SQL 预编译指的是数据库驱动在发送 SQL 语句和参数给 DBMS 之前对 SQL 语句进行编译, 编译成一个函数，这样 DBMS 执行 SQL 时，就不需要重新编译。

2.为什么需要预编译
JDBC 中使用对象 PreparedStatement 来抽象预编译语句，使用预编译。预编译阶段可以优化 SQL 的执行。预编译之后的 SQL 多数情况下可以直接执行，DBMS 不需要再次编译，越复杂的SQL，编译的复杂度将越大，预编译阶段可以合并多次操作为一个操作。同时预编译语句对象可以重复利用。把一个 SQL 预编译后产生的 PreparedStatement 对象缓存下来，下次对于同一个SQL，可以直接使用这个缓存的 PreparedState 对象。Mybatis默认情况下，将对所有的 SQL 进行预编译。


### Mybatis都有哪些Executor执行器？它们之间的区别是什么？
* SimpleExecutor是最简单的执行器，根据对应的sql直接执行即可，不会做一些额外的操作；
* BatchExecutor执行器，顾名思义，通过批量操作来优化性能。通常需要注意的是批量更新操作，由于内部有缓存的实现，使用完成后记得调用flushStatements来清除缓存。
* ReuseExecutor 可重用的执行器，重用的对象是Statement，也就是说该执行器会缓存同一个sql的Statement，省去Statement的重新创建，优化性能。内部的实现是通过一个HashMap来维护Statement对象的。由于当前Map只在该session中有效，所以使用完成后记得调用flushStatements来清除Map。


### Mybatis 延迟加载
多表联合查询的时候，某些表的数据在使用的时候再加载，而不是查询时立刻加载。

xml
```
<resultMap id="userAccountMap" type="Account">
        <id property="id" column="id"></id>
        <result property="uid" column="uid"></result>
        <result property="money" column="money"></result>
        <!-- 配置封装 User 的内容
            select：查询用户的唯一标识
            column：用户根据id查询的时候，需要的参数值
        -->
        <association property="user" column="uid" javaType="User" select="cn.ideal.mapper.UserMapper.findById"></association>
</resultMap>

 <!-- 根据查询所有账户 -->
    <select id="findAll" resultMap="userAccountMap">
        SELECT * FROM account
    </select>

```

配置
```
<settings>
    <setting name="lazyLoadingEnabled" value="true"/>
     <setting name="aggressiveLazyLoading" value="false"></setting>
</settings>
```

### #{}和${}的区别
#{} 是预编译处理，像传进来的数据会加个双引号（#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号）

${}就是字符串替换 。 直接替换掉占位符. 方式一般用于传入数据库对象，例如传入表名.

使用 ${} 的话会导致 sql 注入。什么是 SQL 注入呢？比如 select * from user where id = ${value}

value 应该是一个数值吧。然后如果对方传过来的是 001 and name = tom。这样不就相当于多加了一个条件嘛？把SQL语句直接写进来了。如果是攻击性的语句呢？001；drop table user，直接把表给删了

所以为了防止 SQL 注入，能用 #{} 的不要去用 ${}

如果非要用 ${} 的话，那要注意防止 SQL 注入问题，可以手动判定传入的变量，进行过滤，一般 SQL 注入会输入很长的一条 SQL 语句。

### 在mapper中如何传递多个参数
* 使用map接口传递参数
* 使用注解传递多个参数　
* 通过Java Bean传递多个参数
* 混合使用以上方式传递多个参数

### Mybatis如何执行批量操作
xml中使用 foreach 标签来实现批量操作，foreach 标签可以遍历集合，并将集合中的每一个元素作为一个参数传递给sql语句。


### 使用MyBatis的mapper接口调用时有哪些要求？

* Mapper接口方法名和mapper.xml中定义的每个sql的id相同。

* Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql 的parameterType的类型相同。

* Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同。

* Mapper.xml文件中的namespace即是mapper接口的类路径


### Mybatis动态sql是做什么的？都有哪些动态sql？能简述一下动态sql的执行原理不？
Mybatis动态sql可以让我们在Xml映射文件内，以标签的形式编写动态sql，完成逻辑判断和动态拼接sql的功能，Mybatis提供了9种动态sql标签trim|where|set|foreach|if|choose|when|otherwise|bind。

其执行原理为，使用OGNL从sql参数对象中计算表达式的值，根据表达式的值动态拼接sql，以此来完成动态sql的功能。

### Mybatis是如何进行分页的？
* 原始方法，使用 limit，需要自己处理分页逻辑：
* 自己实现拦截StatementHandler，其实质还是在最后生成limit语句
* 使用分页插件，mybatis-pagehelper，可以自动完成分页功能，不需要自己处理分页逻辑。

### 简述Mybatis的插件运行原理，以及如何编写一个插件。
Mybatis仅可以编写针对ParameterHandler、ResultSetHandler、StatementHandler、Executor这4种接口的插件，Mybatis使用JDK的动态代理，为需要拦截的接口生成代理对象以实现接口方法拦截功能，每当执行这4种接口对象的方法时，就会进入拦截方法，具体就是InvocationHandler的invoke()方法，当然，只会拦截那些你指定需要拦截的方法。

实现Mybatis的Interceptor接口并复写intercept()方法，然后在给插件编写注解，指定要拦截哪一个接口的哪些方法即可，记住，别忘了在配置文件中配置你编写的插件。

### Mybatis的缓存机制有哪些？  
mybatis的缓存分为两级：一级缓存、二级缓存

（1）一级缓存：
一级缓存为 ​sqlsesson​ 缓存，缓存的数据只在 SqlSession 内有效。在操作数据库的时候需要先创建 SqlSession 会话对象，在对象中有一个 HashMap 用于存储缓存数据，此 HashMap 是当前会话对象私有的，别的 SqlSession 会话对象无法访问。

具体流程：

第一次执行 select 完毕会将查到的数据写入 SqlSession 内的 HashMap 中缓存起来

第二次执行 select 会从缓存中查数据，如果 select 同传参数一样，那么就能从缓存中返回数据，不用去数据库了，从而提高了效率

注意：

1、如果 SqlSession 执行了 DML 操作（insert、update、delete），并 commit 了，那么 mybatis 就会清空当前 SqlSession 缓存中的所有缓存数据，这样可以保证缓存中的存的数据永远和数据库中一致，避免出现差异

2、当一个 SqlSession 结束后那么他里面的一级缓存也就不存在了， mybatis 默认是开启一级缓存，不需要配置

3、 mybatis 的缓存是基于 [namespace:sql语句:参数] 来进行缓存的，意思就是， SqlSession 的 HashMap 存储缓存数据时，是使用 [namespace:sql:参数] 作为 key ，查询返回的语句作为 value 保存的

（2）二级缓存：
二级缓存是​ mapper​ 级别的缓存，也就是同一个 namespace 的 mapper.xml ，当多个 SqlSession 使用同一个 Mapper 操作数据库的时候，得到的数据会缓存在同一个二级缓存区域

二级缓存默认是没有开启的。需要在 setting 全局参数中配置开启二级缓存

