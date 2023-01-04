## **1.概念/使用方法向的问题**

### **1.1 什么是Mybatis?**

（1）Mybatis是一个半ORM框架，它内部封装了JDBC，开发时只需要关注SQL语句本身，不需要花费精力去处理加载驱动、创建连接、创建statement等繁杂的过程。

（2）MyBatis 可以使用 XML 或注解来配置和映射原生信息，将 POJO映射成数据库中的记录，避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

（3）通过xml 文件或注解的方式将要执行的各种 statement 配置起来，并通过java对象和statement中sql的动态参数进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射为java对象并返回。

### 1.2 为什么说**Mybatis是半ORM框架?与Hibernate有哪些不同?**

ORM是对象和关系之间的映射，包括对象->关系和关系->对象两方面。Hibernate是个完整的ORM框架，而MyBatis只完成了关系->对象，准确地说MyBatis是SQL映射框架而不是ORM框架，因为其仅有字段映射，对象数据以及对象实际关系仍然需要通过手写SQL来实现和管理。

（1）Hibernate为完整的ORM框架，Mybatis为半ORM框架。

（2）Mybatis程序员直接编写原生sql，可严格控制sql执行性能，灵活度高，适用于对关系数据模型要求不高的软件开发，例如互联网软件、企业运营类软件等；Hibernate只能通过编写hql实现数据库查询（hql好难用哦）。

（3）Hibernate对象/关系映射能力强，数据库无关性好，适用于对关系模型要求高的软件； Mybatis的数据库无关性较差，如果需要实现支持多种数据库的软件则需要自定义多套sql映射文件。

### **1.3 Mybaits的优点?**

（1）基于SQL语句编程，不会对应用程序或者数据库的现有设计造成任何影响，解除sql与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态SQL语句，重用性高。

（2）与JDBC相比，减少了50%以上的代码量，消除了JDBC大量冗余的代码，不需要手动开关连接；

（3）很好的与各种数据库兼容（因为MyBatis使用JDBC来连接数据库，所以只要JDBC支持的数据库MyBatis都支持）。

（4）能够与Spring很好的集成；

（5）提供映射标签，支持对象与数据库的ORM字段关系映射；提供对象关系映射标签，支持对象关系组件维护。

### **1.4 MyBatis框架的缺点?**

（1）SQL语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写SQL语句的功底有一定要求。

（2）SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

### **1.5 #{}和${}的区别?**

（1）${}是properties文件中的变量占位符，它可以用于标签属性值和sql内部，属于静态文本替换。

（2）#{}是sql的参数占位符，Mybatis会将sql中的#{}替换为?号，在sql执行前会使用PreparedStatement的参数设置方法，按序给sql的?号占位符设置参数值。使用#{}可以有效的防止 SQL 注入，提高系统安全性。

```
${param}传递的参数会被当成sql语句中的一部分，举例：
order by ${param}，则解析成的sql为：
order by    id
 
#{parm}传入的数据都当成一个字符串，会对自动传入的数据加一个双引号，举例：
select * from table where name = #{param}，则解析成的sql为：
select * from table where name =   "id"
```

### 1.6 怎么解决**实体类中的属性名和表中的字段名不一样的问题?**

（1）通过在查询的sql语句中定义字段名的别名，使字段名的别名和实体类的属性名一致

```
<select id="selectUserById" parameterType="java.lang.Integer" resultetype="com.en.entity.user">
       select user_id as id, user_no as no from test where user_id = #{id};
</select>
```

（2）Mybatis和hibernate不同，它不完全是一个ORM框架，因为MyBatis需要程序员自己编写Sql语句。

```
   <resultMap type=”me.gacl.domain.order” id=”orderresultmap”>
        <!–用id标签来映射主键字段–>
        <id property="id" column="user_id">
        <!–用result属性来映射非主键字段，property为实体类属性名，column为数据表中的属性–>
        <result property="no" column="user_no"/>
   </reslutMap>
```

### **1.7 如何在mapper中传递多个参数?**

（1）使用 @param 注解：

```
user selectUser(@param("username") string username,@param("password") string password);
```

（2）Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql 的parameterType的类型相同；

```
    Map<String, Object> map = new HashMap();
    map.put("start", start);
    map.put("end", end);
    sqlSession.selectList("student.selectUser", map);
```

### **1.8 MyBatis的接口绑定有哪些实现方式？**

接口绑定有两种实现方式：

（1）一种是通过注解绑定,就是在接口的方法上面加上@Select@Update等注解里面包含Sql语句来绑定

```
@Select("select ID,CODE,NAME from T_SYS_DICT_TYPE ")
@Results(id = "distTypeMap",value ={@Result(id =true,property="id",column="ID")
            ,@Result(property="code",column="CODE")
            ,@Result(property="name",column="NAME")
            ,@Result(property = "dictDtos" ,column = "ID",many = @Many(select="com.santbbd.ams.sysconfig.mapper.SysInitMapper.findByDistTypeId",fetchType = FetchType.EAGER))
    })
List<SysDictTypeDto> getAllDist();
```

（2）另外一种就是通过xml里面写SQL来绑定,在这种情况下,要指定xml映射文件里面的namespace必须为接口的全路径名.

```
<mapper namespace="com.xxx.xxx.modular.batch.mapper.IllegalCollectionMapper">
<select id="queryFileDisposeInfo" parameterType="FileDisposeVo" resultMap="illegalcollection-map">
   SELECT 
        BATCH_NUMBER,
        FINISH_DATE,
        FILE_NAME,
        FILE_SIZE,
        DATA_SIZE,
        FILE_TYPE,
        ORG_CODE
   FROM 
        T_FILE_DISPOSE
</select>
```

### 1.9 **使用MyBatis Mapper接口开发时有哪些要求？**

（1）Mapper接口方法名和mapper.xml中定义的每个sql的id相同；  
（2）Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql 的parameterType的类型相同；  
（3）Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同；  
（4）Mapper.xml文件中的namespace即是mapper接口的类路径;

## **2.源码向的问题**

### 2.1 解释下**MyBatis面向Mapper编程工作原理？**

Mapper接口是没有实现类的，当调用接口方法时，采用了JDK的动态代理，先从Configuration配置类MapperRegistry对象中获取mapper接口和对应的代理对象工厂信息（MapperProxyFactory），然后利用代理对象工厂MapperProxyFactory创建实际代理类（MapperProxy），最后在MapperProxy类中通过MapperMethod类对象内保存的中对应方法的信息，以及对应的sql语句的信息进行分析，最终确定对应的增强方法进行调用。

### 2.2 为什么**MyBatis Mapper接口中的方法不支持重载？**

在MyBatis源码中有这么几行代码，我们可以看到在解析XML文件创建mappe接口对应方法的时候，采用了接口全限名+方法名的方式作为StrictMap(MappedStatement数据存放的Map集合)的key值，而源码对于StrictMap的put方法进行了判断，如果存入的数据key已重复则抛出异常，所以Mapper接口中的方法不支持重载。

```
id = applyCurrentNamespace(id, false);

public String applyCurrentNamespace(String base, boolean isReference) {
   ...
   //返回值为mapper的全限名(xml中namespace的值)+方法名(xml中Statement id的值)
   return currentNamespace + "." + base;
}
```

![](https://img-blog.csdnimg.cn/20200920171405540.png)

### **2.3 Mybatis动态sql执行原理?**

（1）初始化阶段：通过XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuilder解析XML文件中的信息存储到Configuration类中；  
（2）代理阶段：先从Configuration配置类MapperRegistry对象中获取mapper接口和对应的代理对象工厂信息，再利用代理对象工厂MapperProxyFactory创建实际代理类，最后在MapperProxy类中通过MapperMethod类对象内保存的中对应方法的信息，以及对应的sql语句的信息进行分析，最终确定对应的增强方法进行调用。  
（3）数据读写阶段：通过四种Executor调用四种Handler进行查询和封装数据；

### **2.4 Mybatis的一级、二级缓存实现原理?**

（1）一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空，Mybatis默认打开一级缓存，一级缓存存放在BaseExecutor的localCache变量中：

![](https://img-blog.csdnimg.cn/2020092018550771.png)

（2）二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储，不同在于其存储作用域为 Mapper(Namespace)级别。Mybatis默认不打开二级缓存，可以在config文件中xml<settings><setting name="cacheEnabled" value="true"/></settings>开启全局的二级缓存，但并不会为所有的Mapper设置二级缓存，每个mapper.xml文件中使用标签来开启当前mapper的二级缓存，二级缓存存放在MappedStatement类cache变量中：

![](https://img-blog.csdnimg.cn/20200920190114946.png)

（3）对于缓存数据更新机制，当某一个作用域(一级缓存 Session/二级缓存Namespaces)的进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被清除并重新更新，如果开启了二级缓存，则只根据配置判断是否刷新。

### **2.5 Mybatis是如何进行分页的？**

（1）SQL分页(物理分页)：

```
<select id="queryStudentsBySql" parameterType="map" resultMap="studentmapper"> 
           select * from student limit #{start} , #{end}
</select>
```

（2）使用RowBounds实现分页(逻辑分页)：

```
Service:
publicList queryRolesByPage(String roleName,intstart,int limit) {
     returnroleDao.queryRolesByPage(roleName,new RowBounds(start, limit));
}

```

```
Dao:
     public List queryUsersByPage(String userName, RowBounds rowBounds);
```

（3）使用分页插件PageHelper：

```
public Json queryByPage(User userParam,Integer pageNum,Integer pageSize) {
        PageHelper.startPage(pageNum, pageSize);
        List<User> userList = userMapper.queryByPage(userParam);
        Json json = new Json();
        return json;
}
```

### **2.6 Mybatis都有哪些Executor执行器？它们之间的区别是什么？**

![](https://img-blog.csdnimg.cn/20200917203424287.png)

**BaseExecutor：**基础抽象类，实现了executor接口的大部分方法，主要提供了缓存管理和事务管理的能力，使用了模板模式，doUpdate,doQuery,doQueryCursor 等方法的具体实现交给不同的子类进行实现

**CachingExecutor：**直接实现Executor接口，使用装饰器模式提供二级缓存能力。先从二级缓存查，缓存没有命中再从数据库查，最后将结果添加到缓存中。如果在xml文件中配置了cache节点，则会创建CachingExecutor。

**BatchExecutor：**BaseExecutor具体子类实现，在doUpdate方法中，提供批量执行多条SQL语句的能力；

**SimpleExecutor：**BaseExecutor具体子类实现且为默认配置，在doQuery方法中使用PrepareStatement对象访问数据库， 每次访问都要创建新的 PrepareStatement对象；

**ReuseExecutor：**BaseExecutor具体子类实现，与SimpleExecutor不同的是，在doQuery方法中，使用预编译PrepareStatement对象访问数据库，访问时，会重用缓存中的statement对象，而不是每次都创建新的PrepareStatement。

### **2.7 Mybatis中如何指定使用哪一种Executor执行器？**

在Mybatis配置文件中，可以指定默认的ExecutorType执行器类型，也可以手动给DefaultSqlSessionFactory的创建SqlSession的方法传递ExecutorType类型参数。

![](https://img-blog.csdnimg.cn/20200920183713811.png)

### ![](https://img-blog.csdnimg.cn/20200920183657317.png)

### **2.8 Mybatis的Xml映射文件和Mybatis内部数据结构之间的映射关系？**

![](https://img-blog.csdnimg.cn/20200915221915851.png)

![](https://img-blog.csdnimg.cn/2020092018294130.png)

![](https://img-blog.csdnimg.cn/20200920183022252.png)

Mybatis将所有Xml配置信息都封装到All-In-One重量级对象Configuration内部。在Xml映射文件中，<resultMap>标签会被解析为ResultMap对象，其每个子元素会被解析为ResultMapping对象。每一个<select>、<insert>、<update>、<delete>标签均会被解析为MappedStatement对象，标签内的sql会被解析为BoundSql对象。

### **2.9 Mybatis中用到了哪些设计模式？**

日志模块：代理模式、适配器模式

数据源模块：代理模式、工厂模式

缓存模块：装饰器模式

初始化阶段：建造者模式

代理阶段：策略模式

数据读写阶段：模板模式

插件化开发：责任链模式

## **2.MyBatis源码结构**

### 2.1 源码包功能模块图

![](https://img-blog.csdnimg.cn/20200830162428980.png)

### 2.2 各包详细功能解析

**org.apache.ibatis.logging：**包含所有mapper 接口中用到的注解

**org.apache.ibatis.binding：**生成mapper 接口的动态代理并进行管理

**org.apache.ibatis.builder：**

1.  包含Configuration对象所有构建器，主要包括XML、注解2种方式配置解析
2.  BaseBuilder 构建器基类
3.  XMLConfigBuilder 解析configuration.xml配置文件
4.  XMLMapperBuilder 解析Mapper.xml配置文件
5.  XMLStatementBuilder 解析selectupdatedelete 标签
6.  MapperAnnotationBuilder 注解式Mapper

**org.apache.ibatis.cache：**

1.  缓存功能实现、包含各种缓存装饰器
2.  TransactionalCache 二级缓存功能实现

**org.apache.ibatis.cursor：**实现游标的方式查询数据、游标非常适合处理百万级别的数据查询

**org.apache.ibatis.datasource：**数据源 包括jndi数据源、连接池功能

**org.apache.ibatis.executor：**

1.  包含SQL语句执行器，核心功能包
2.  功能包括：主键生成功能、执行参数解析功能、执行结果集解析功能、SQL执行器、缓存执行器

**org.apache.ibatis.exceptions：**框架异常，常见异常：TooManyResultsException

**org.apache.ibatis.io：**资源文件读取

**org.apache.ibatis.jdbc：**

1.  JDBC一些操作
2.  SqlRunner SQL执行
3.  ScriptRunner 脚本执行，可以执行建库语句

**org.apache.ibatis.logging：**

1.  日志功能，实现多种日志框架的对接
2.  org.apache.ibatis.logging.jdbc 代理所有功能JDBC 操作，实现了在debug模式下能够输出SQL

**org.apache.ibatis.mapping：**配置文件与实体对象的映射功能，Mapper映射、参数映射、结果映射等

**org.apache.ibatis.parsing：**

1.  解析工具包
2.  GenericTokenParser：解析#{} ${} 这种占位符
3.  XPathParser：XPath形式解析XML
4.  PropertyParser: properties解析器

**org.apache.ibatis.scripting：**动态SQL语言实现，配置文件中<if> <where> <set> <foreach> <choose> 功能就是在这个包实现，借助OGNL表达式,你也可以扩展自己的语言实现功能

**org.apache.ibatis.session：**

1.  主要实现SqlSession功能，非常核心包
2.  官方注释：SqlSession包含了MyBatis工作的所有的Java接口，通过这些接口你可以 执行SQL命令（insertdeleteupdateselect），获取Mapper，管理实务

**org.apache.ibatis.transaction：**事务功能实现，包装了数据库连接，处理数据库连接生命周期包括：连接创建，预编译，提交回滚和关闭

**org.apache.ibatis.type：**类型处理器，包括所有数据库类型对应Java类型的处理器，如果要实现自己类型处理器就需要实现包下的基础接口

## **啃下MyBatis源码 - MyBatis核心流程三大阶段之初始化阶段**

**1.加载配置文件**

**2.解析配置文件、将配置文件中的信息装载到Configuration中**

**3.根据Configuration创建SqlSessionFactory并返回**

**--------------------------------------------------------------------------------------------------------------------------**

前面几篇分析了MyBatis的日志、数据源和缓存模块的源码，本篇将分析MyBatis核心流程三大阶段的第一阶段：初始化阶段。Mybatis启动初始化的核心就是将所有xml配置文件信息加载到Configuration对象中，Configuration为单例，生命周期为应用级。

MyBatis初始化流程大致有三步：

1.  加载配置文件
2.  解析配置文件、将配置文件中的信息装载到Configuration中。
3.  根据Configuration创建SqlSessionFactory并返回。

### **1.加载配置文件**

下面我们来看一段经典查询操作：

```
String resouce = "config/mybatis/mybatis-config.xml";
InputStream is = Resources.getResourceAsStream(resouce);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
SqlSession session = sqlSessionFactory.openSession();
user = session.selectOne("com.luoxn28.dao.UserDao.getById", 1);
```

以上代码经过了MyBatis初始化、创建sqlSession、执行sql语句3个过程。首先由mybatis-config.xml配置文件创建SqlSessionFactory，然后由session工厂创建SqlSession对象，执行SQL语句。**当然初始化的第一阶段：扫描配置文件所在包路径并加载**。

### **2.解析配置文件、将配置文件中的信息装载到Configuration中**

让我们来看一下梦开始的地方：

```
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
```

跟进build()方法，我们可以看到new了一个XMLConfigBuilder对象并调用了parse()方法：

```
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      //创建XMLConfigBuilder对象解析XML配置
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      //将XML配置解析成Configuration对象，通过Configuration对象创建SqlSessionFactory
      return build(parser.parse());
    } 
    ....
}
```

跟进parse()方法，我们可以看到parser.evalNode("/configuration")，evalNode为xml结点解析器，可以解析指定参数结点的信息，再看这个"/configuration"有点眼熟丫，这不就是mybatis.xml的根节点嘛：

```
public Configuration parse() {
    ...
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
```

看到这，我们就不得不提初始化的三大金刚了，分别是XMLConfigBuilder、XMLMapperBuilder、XMLStatementBuilder。

**XMLConfigBuilder：**主要负责解析mybatis-config.xml

**XMLMapperBuilder：**主要负责解析映射配置文件

**XMLStatementBuilder：**主要负责解析映射配置文件中的sql节点

**三大金刚图解：**

![](https://img-blog.csdnimg.cn/20200915222654319.png)

MyBatis中的xml文件是由三大金刚读取到Configuration类中，那么我们来看下Configuration类的数据结构：

![](https://img-blog.csdnimg.cn/20200915221915851.png)

Configuration类的源码实在太多，童鞋们先对这个类有个大致印象，了解下该类中有哪些成员变量对应存储着些什么数据。下面主要列举几个比较重要的成员变量：

**MapperRegistry：**mapper接口动态代理工厂类的注册中心。通过mapperProxy实现InvocationHandler接口，其中的MapperProxyFactory用于生成动态代理的实例对象；  
**ResultMap：**用于解析mapper.xml文件中的resultMap节点，使用ResultMapping来封装id，result等子元素；  
**MappedStatement：**用于存储mapper.xml文件中select、insert、update和delete节点，同时还包含了这些节点的重要属性；  
**SqlSource：**mapper.xml文件中的sql语句会被解析成SqlSource对象，经过解析SqlSource包含的语句最终仅仅包含?占位符，可以直接提交给数据库执行；

接上面XMLConfigBuilder开始解析"/configuration"节点：

```
parseConfiguration(parser.evalNode("/configuration"));
```

```
private void parseConfiguration(XNode root) {
    try {
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

点进去一看，就是对照着MyBatis官网主配置文件中的元素一个一个的进行解析

![](https://img-blog.csdnimg.cn/20200915223612191.png)

在解析"mappers"节点的时候，就引入了XMLMapperBuilder开始对映射配置文件进行解析

```
XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
mapperParser.parse();
```

```
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      ...
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

一一对应官网提供的节点信息进行解析

![](https://img-blog.csdnimg.cn/20200915224121930.png)

下面大家猜也猜到了，在解析具体select、insert、update、delete的时候，引入了XMLStatementBuilder对节点数据进行解析：

```
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

继三大金刚分别解析自己负责的xml文件之后，Configuration对象的数据被填充完毕，**初始化的第二阶段：解析配置文件，将数据装载进Configuration对象完成。**

### **3.根据Configuration创建SqlSessionFactory并返回**

第三阶段就是根据SqlSessionFactoryBuilder的内部方法直接返回一个DefaultSqlSessionFactory：

```
public class SqlSessionFactoryBuilder {
   ...
   public SqlSessionFactory build(Configuration config) {
      return new DefaultSqlSessionFactory(config);
   }
}
```

此工厂内封装了Configuration对象：

```
public class DefaultSqlSessionFactory implements SqlSessionFactory {
  private final Configuration configuration;
  public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
  ...
}
```

初始化阶段图解：

![](https://img-blog.csdnimg.cn/20200915225840294.png)

至此，MyBatis初始化阶段完成。

## **啃下MyBatis源码 - MyBatis核心流程三大阶段之代理阶段（binding模块分析）**

**1.MyBatis是如何做到面向Mapper接口编程？**

**2.代理阶段流程梳理**

**--------------------------------------------------------------------------------------------------------------------------**

### **1.MyBatis是如何做到面向Mapper接口编程？**

只有接口，没有实现类，那么我们很容易会想到是通过解析xml配置文件+动态代理来实现的。我们先来说下MyBatis动态代理实际做了一些什么事情，我们正常编写的代码：

```
SqlSession sqlSession = sqlSessionFactory.openSession();
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
User uer = userMapper.selectByPrimarKey(1);
```

MyBatis动态代理后执行的为下面这段代码：

```
SqlSession sqlSession = sqlSessionFactory.openSession();
User uer = sqlSession.selectOne("com.en.iot.mapper."+"UserMapper.selectByPrimarKey",1);
```

我们可以看到MyBatis动态代理主要做的是翻译的工作，主要翻译的内容有三点：

**1、找到Session中对应的方法执行**

**2、找到命名空间和方法名**

**3、传递参数**

这三项工作主要是由MapperMethod这个类来实现的，解读这个类之前，我们有必要对binding模块进行一个整体的分析：

![](https://img-blog.csdnimg.cn/20200916211402947.png)

**MapperRegistry：**为MyBatis配置类Configuration类中一个重要的属性，它是mapper接口和对应的代理对象工厂的注册中心；

```
public class MapperRegistry {
   private final Configuration config;
   //mapper接口和对应的代理对象工厂之间的关系
   private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
   ...
}
```

**MapperProxyFactory：**用于生成mapper接口动态代理的实例对象；

```
public class MapperProxyFactory<T> {
  ...
  //key为mapper接口中的某个方法的method对象，value为对应的MapperMethod
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();
  ...
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
}
```

**MapperProxy：**实现InvocationHandler接口，它是增强mapper接口的实现；![](https://img-blog.csdnimg.cn/20200916214700706.png)

接着跟进cachedInvoker(method).invoke(proxy, method, args, sqlSession)方法

![](https://img-blog.csdnimg.cn/20200916213826490.png)

我们可以看到在cachedInvoker中判断了一下是选用DefaultMethodInvoker还是PlainMethodInvoker

![](https://img-blog.csdnimg.cn/20200916214037965.png)

我们可以看到在PlainMethodInvoker类中封装了一个MapperMethod对象，然后在invoke方法中的execute方法中最终通过增删改查的类型来调用增强的方法，当然调用前先用参数解析器过滤一下参数。

![](https://img-blog.csdnimg.cn/20200916214239389.png)

那么我们大胆猜测一下，调用execute的这个MapperMethod类中一定保持着Mapper接口中对应方法以及对应的sql语句的信息。

![](https://img-blog.csdnimg.cn/20200916214959863.png)

![](https://img-blog.csdnimg.cn/20200916215029787.png)  
通过观察这三个对象的构造方法我们可以看到，这三个对象全部是从Configuration类中获取信息，由此证实了我们的猜想，MapperMethod类中通过这三个对象建立mapper接口和配置文件sql语句的联系。

![](https://img-blog.csdnimg.cn/20200916215442588.png)

![](https://img-blog.csdnimg.cn/20200916215556263.png)

![](https://img-blog.csdnimg.cn/20200916215523383.png)

### **2.代理阶段流程梳理**

1、先从Configuration配置类MapperRegistry对象中获取mapper接口和对应的代理对象工厂信息（MapperProxyFactory）

2、利用代理对象工厂MapperProxyFactory创建实际代理类（MapperProxy）

3、在MapperProxy类中通过MapperMethod类对象内保存的中对应方法的信息，以及对应的sql语句的信息进行分析，最终确定对应的增强方法进行调用。

### 啃下MyBatis源码 - MyBatis核心流程三大阶段之数据读写阶段

**1.MyBatis是怎样的封装jdbc操作的**

**2.sqlSession查询流程图和Executor内部调用流程图**

**--------------------------------------------------------------------------------------------------------------------------**

### **1.MyBatis是怎样的封装jdbc操作的**

我们先来回忆一下jdbc代码：

```
//1.加载驱动
Class.forName("com.mysql.jdbc.Driver");     
//2.获取连接conn
Connection con=DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "123");
//3.创建查询接口
Statement sta= con.createStatement();
//4.执行SQL，返回结果集
ResultSet rs= sta.executeQuery("SELECT * FROM `user`");
//5.对结果集数据进行操作
User user = new User();
user.setUserName(String.valueOf(rs.getObject(1)));
```

其中第一步加载驱动在MyBatis的初始化阶段就已经完成了，数据读写阶段就是处理sqlSession.executeQuery的阶段，对应JDBC第二步获取连接开始，到返回结果集封装对象结束。那MyBatis究竟是怎样封装JDBC操作的呢？我们先从sqlSession的默认实现DefaultSqlSession开始入手：

![](https://img-blog.csdnimg.cn/20200917203050838.png)

可以看到该类包含一个核心组件Executor（执行器），查询相关操作最终都借助该组件实现，那么我们来看一下Executor的关系类图：

![](https://img-blog.csdnimg.cn/20200917203424287.png)

**BaseExecutor：**基础抽象类，实现了executor接口的大部分方法，主要提供了缓存管理和事务管理的能力，使用了模板模式，doUpdate,doQuery,doQueryCursor 等方法的具体实现交给不同的子类进行实现

**CachingExecutor：**直接实现Executor接口，使用装饰器模式提供二级缓存能力。先从二级缓存查，缓存没有命中再从数据库查，最后将结果添加到缓存中。如果在xml文件中配置了cache节点，则会创建CachingExecutor。

**BatchExecutor：**BaseExecutor具体子类实现，在doUpdate方法中，提供批量执行多条SQL语句的能力；

**SimpleExecutor：**BaseExecutor具体子类实现且为默认配置，在doQuery方法中使用PrepareStatement对象访问数据库， 每次访问都要创建新的 PrepareStatement对象；

**ReuseExecutor：**BaseExecutor具体子类实现，与SimpleExecutor不同的是，在doQuery方法中，使用预编译PrepareStatement对象访问数据库，访问时，会重用缓存中的statement对象，而不是每次都创建新的PrepareStatement。

一下子丢出来这么多执行器有点蒙，没关系我们跟进一个查询流程走下来就清楚了。首先从DefaultSqlSession开始，我们调用的sqlSession.selectList方法：

![](https://img-blog.csdnimg.cn/20200917205236272.png)

可以看到只有BaseExecutor和CachingExecutor两个类重写了query方法，而CachingExecutor类前面也说过，在Configuration类初始化的时候如果在XML中配置了<cache>节点的话，则会用装饰器模式对基础执行器进行增强，使其拥有二级缓存能力，并且我们也可以看到在初始化Executor时是通过设定的类型来决定初始化哪一个执行器子类。

![](https://img-blog.csdnimg.cn/20200917205454850.png)

好的我们继续跟进BaseExecutor的query()方法:

![](https://img-blog.csdnimg.cn/20200917210415293.png)

![](https://img-blog.csdnimg.cn/20200917210606702.png)

可以看到首先通过MappedStatement拿到对应的SQL信息BoundSql，再封装一级缓存值CacheKey，具体的查询为先从一级缓存拿，如果一级缓存为空，就从数据库加载数据，具体从数据库查询的方法源码：

![](https://img-blog.csdnimg.cn/20200917210831277.png)

我们跟进默认实现SimpleExecutor的doQuery方法：

![](https://img-blog.csdnimg.cn/20200917211035123.png)

这段代码有两点值得我们注意，一个是prepareStatement(handler, ms.getStatementLog());这个方法，我们跟进去会发现：

![](https://img-blog.csdnimg.cn/20200917211124589.png)

**终于找到了我们熟悉的JDBC代码，获取Connection，创建Statement查询接口**；再一个是我们看到了四个新面孔，四种不同的处理器，一起来看下StatementHandler体系结构类图：

![](https://img-blog.csdnimg.cn/20200919104808424.png)

**BaseStatementHandler：** 所有子类的抽象父类，定义了初始化statement的操作顺序，由具体子类实例化不同的statement

**CallableStatementHandler：**调用存储过程

**PreparedStatementHandler：**使用预编译PrepareStatement对象访问数据库

**RoutingStatementHandler：**Excutor组件真正实例化的子类，使用静态代理模式，根据上下文决定创建哪个具体实体类

**SimpleStatementHandler：**直接使用statement对象访问数据库，无须参数化

RoutingStatementHandler类源码，很清晰的静态代理

![](https://img-blog.csdnimg.cn/20200917211725203.png)

接上文调用SimpleStatementHandler的query方法:

![](https://img-blog.csdnimg.cn/20200917211854628.png)

**jdbc的execute()方法也找到了**，最后借助DefaultResultSetHandler对数据库返回的结果集进行封装，返回用户指定的实体类型。handleResultSets()方法部分源码：

![](https://img-blog.csdnimg.cn/20200919110111271.png)

处理结果集的过程略复杂，这里只简单的梳理下MyBaits对于结果集封装的步骤：

1.  创建multipleResults集合，保存最终返回的结果。

2.  取出第一个结果集

3.  获取对应的resultMap

4.  根据resultMap转化结果集，转换成目标对象后添加到multipleResults集合；

5.  resultset.close()关闭结果集，将multipleResults集合返回

### **2.sqlSession查询流程图和Executor内部调用流程图**

**sqlSession查询流程图：**

![](https://img-blog.csdnimg.cn/2020091911381497.png)

**Executor内部调用流程图：**

![](https://img-blog.csdnimg.cn/2020091911204922.png)

至此MyBatis核心流程最后一个阶段：数据读写阶段完成。