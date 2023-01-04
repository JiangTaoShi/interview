#### 1、介绍一下mybatis，说一下它的优点和缺点是什么？

Mybatis是一个半ORM（对象关系映射）的持久层框架,它内部封装了JDBC，开发时只需要关注SQL语句本身，不需要花费精力去处理加载驱动、创建连接、创建statement等繁杂的过程,使用时直接编写原生态sql。

优点：

1：基于SQL语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL写在XML         里，解除sql与程序代码的耦合，便于统一管理，提供XML标签，支持编写动态SQL语句，并可重用；

2：很好的与各种数据库兼容；

3：提供映射标签，支持对象与数据库的ORM字段关系映射，提供对象关系映射标签，支持对象关系组件维护。

4：与JDBC相比，消除了JDBC大量冗余的代码，不需要手动开关连接，能够与Spring很好的集成

缺点：

1：SQL语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写SQL语句的功底有一定要求；

2：SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库；

#### 2、**MyBatis与Hibernate有哪些不同**？

首先Hibernate是一个完全面向对象的持久层框架，mybatis是一个半自动化的持久层框架。

开发方面： hibernate开发中，sql语句已经被封装，直接可以使用，加快系统开发， Mybatis 属于半自动化，sql需要手工完成，稍微繁琐，但是如果对于庞大复杂的系统项目来说，复杂的sql语句较多，选择hibernate 就不是一个好方案。

 sql优化方面：Hibernate 自动生成sql,有些语句较为繁琐，会多消耗一些性能，  Mybatis 手动编写sql，可以避免不需要的查询，提高系统性能；

对象管理方面：Hibernate 是完整的对象-关系映射的框架，开发工程中，无需过多关注底层实现，只要去管理对象即可；Mybatis 需要自行管理 映射关系。

缓存方面：Hibernate的二级缓存配置在SessionFactory生成的配置文件中进行详细配置，然后再在具体的表-对象映射中配置是那种缓存，MyBatis的二级缓存配置都是在每个具体的表-对象映射中进行详细配置，这样针对不同的表可以自定义不同的缓存机制。

总之：Mybatis 小巧、方便、高效、简单、直接、半自动化；Hibernate 强大、方便、高效、复杂、间接、全自动化

#### 3、#{}和${}的区别是什么？

`#{}`是预编译处理，`${}`是字符串替换。

Mybatis在处理`#{}`时，会将sql中的`#{}`替换为?号，调用`PreparedStatement`的set方法来赋值；

Mybatis在处理`${}`时，就是把`${}`替换成变量的值。

使用`#{}`可以有效的防止SQL注入，提高系统安全性。

#### 4、当实体类中的属性名和表中的字段名不一样 ，怎么办 ？

第一种方法：通过在查询的sql语句中定义字段名的别名，让字段名的别名和实体类的属性名一致；

第二种方法：通过`<resultMap>`来映射字段名和实体类属性名的一一对应的关系；

第三种方法：在实体类通过@Column注解也可以实现；

#### 5、通常一个Xml映射文件，都会写一个Dao接口与之对应，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗？

Dao接口即Mapper接口。接口的全限名，就是映射文件中的namespace的值；接口的方法名，就是映射文件中Mapper的Statement的id值；接口方法内的参数，就是传递给sql的参数。

Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为key值，可唯一定位一个MapperStatement。在Mybatis中，每一个`<select>、<insert>、<update>、<delete>`标签，都会被解析为一个MapperStatement对象。

Mapper接口里的方法，是不能重载的，因为是使用 全限名+方法名 的保存和寻找策略。Mapper 接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理对象proxy，代理对象会拦截接口方法，转而执行MapperStatement所代表的sql，然后将sql执行结果返回。

#### 6、mybatis 如何执行批量插入?

有两种方式，第一种就是普通的xml中insert语句可以写成单条插入，在调用方循环N次;第二种是xml中insert语句写成一次性插入一个N条的list，举例下面的list标签

~~~xml
<insert id="insertBatch" >
    insert into person ( <include refid="Base_Column_List" /> ) 
    values 
    <foreach collection="list" item="item" index="index" separator=",">
        (null,#{item.name},#{item.sex},#{item.address})
    </foreach>
</insert>
~~~

#### 7、mybatis 如何获取自动生成的(主)键值?

insert 方法总是返回一个int值 ，这个值代表的是插入的行数，不是插入返回的主键id值；如果采用自增长策略，自动生成的键值在 insert 方法执行完后可以被设置到传入的参数对象中，需要增加两个属性，usegeneratedkeys=”true” keyproperty=”id”

~~~xml
<insert id=”insertname” usegeneratedkeys=”true” keyproperty=”id”>
     insert into names (name) values (#{name})
</insert>
~~~

#### 8、在mapper中如何传递多个参数?

~~~xml
（1）第一种：
//DAO层的函数
Public UserselectUser(String name,String area);  
//对应的xml,#{0}代表接收的是dao层中的第一个参数，#{1}代表dao层中第二参数，更多参数一致往后加即可。
<select id="selectUser"resultMap="BaseResultMap">  
    select *  fromuser_user_t   whereuser_name = #{0} anduser_area=#{1}  
</select>  
 
（2）第二种： 使用 @param 注解:
public interface usermapper {
   user selectuser(@param(“username”) string username,@param(“hashedpassword”) string hashedpassword);
}
然后,就可以在xml像下面这样使用(推荐封装为一个map,作为单个参数传递给mapper):
<select id=”selectuser” resulttype=”user”>
         select id, username, hashedpassword
         from some_table
         where username = #{username}
         and hashedpassword = #{hashedpassword}

~~~

#### 9、Mybatis有哪些动态sql？

Mybatis动态sql可以在Xml映射文件内，以标签的形式编写动态sql，执行原理是根据表达式的值 完成逻辑判断并动态拼接sql的功能。Mybatis提供了9种动态sql标签：`trim | where | set | foreach | if | choose | when | otherwise | bind`

#### 10、Xml映射文件中，除了常见的select|insert|updae|delete标签之外，还有哪些标签？

`<resultMap>、<parameterMap>、<sql>、<include>、<selectKey>`，加上动态sql的9个标签，其中`<sql>`为sql片段标签，通过`<include>`标签引入sql片段，`<selectKey>`为不支持自增的主键生成策略标签。

#### 11、Mybatis的Xml映射文件中，不同的Xml映射文件，id是否可以重复？

不同的Xml映射文件，如果配置了namespace，那么id可以重复；如果没有配置namespace，那么id不能重复；

原因就是namespace+id是作为Map<String, MapperStatement>的key使用的，如果没有namespace，就剩下id，那么，id重复会导致数据互相覆盖。有了namespace，自然id就可以重复，namespace不同，namespace+id自然也就不同。

但是，在以前的Mybatis版本的namespace是可选的，不过新版本的namespace已经是必须的了。

#### 12、Mybatis的一级、二级缓存？

一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空，默认打开一级缓存。

二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储，不同在于其存储作用域为 Mapper(Namespace)，并且可自定义存储源，如 Ehcache。默认不打开二级缓存，要开启二级缓存，使用二级缓存属性类需要实现Serializable序列化接口(可用来保存对象的状态),可在它的映射文件中配置`<cache/>` 

对于缓存数据更新机制，当某一个作用域(一级缓存 Session/二级缓存Namespaces)的进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear 掉并重新更新，如果开启了二级缓存，则只根据配置判断是否刷新。



#### 13、使用MyBatis的mapper接口调用时有哪些要求？

1：Mapper接口方法名和mapper.xml中定义的每个sql的id相同；
2：Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql 的parameterType的类型相同；
3：Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同；
4：Mapper.xml文件中的namespace即是mapper接口的类路径。

#### 14、mybatis plus 了解过么？和mybatis有啥区别？

Mybatis-Plus是一个Mybatis的增强工具，它在Mybatis的基础上只做增强，却不做改变。我们在使用Mybatis-Plus之后既可以使用Mybatis-Plus的特有功能，又能够正常使用Mybatis的原生功能。Mybatis-Plus(简称MP)是为简化开发、提高开发效率而生，自带通用mapper的单表操作。

#### 15、MyBatis框架及原理？

MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架，其主要就完成2件事情：

1. 封装JDBC操作
2. 利用反射打通Java类与SQL语句之间的相互转换

MyBatis的主要设计目的就是让我们对执行SQL语句时对输入输出的数据管理更加方便，所以方便地写出SQL和方便地获取SQL的执行结果才是MyBatis的核心竞争力；

他的执行流程包括

1.读取配置文件，配置文件包含数据库连接信息和Mapper映射文件或者Mapper包路径。

2.有了这些信息就能创建SqlSessionFactory，SqlSessionFactory的生命周期是程序级,程序运行的时候建立起来,程序结束的时候消亡

3.SqlSessionFactory建立SqlSession,目的执行sql语句，SqlSession是过程级,一个方法中建立,方法结束应该关闭

4.当用户使用mapper.xml文件中配置的的方法时，mybatis首先会解析sql动态标签为对应数据库sql语句的形式，并将其封装进MapperStatement对象，然后通过executor将sql注入数据库执行，并返回结果。

5.将返回的结果通过映射，包装成java对象。



