## Hibernate是如何延迟加载(懒加载)?

通过设置属性`lazy`进行设置是否需要懒加载

当Hibernate在查询数据的时候，数据并没有存在与内存中，**当程序真正对数据的操作时，对象才存在与内存中，就实现了延迟加载**，他节省了服务器的内存开销，从而提高了服务器的性能。

## hibernate的三种状态之间如何转换

Hibernate中对象的状态：

- **临时/瞬时状态**
- **持久化状态**
- **游离状态**

### 临时/瞬时状态

当我们**直接new出来的对象就是临时/瞬时状态的**..

- **该对象还没有被持久化【没有保存在数据库中】**
- **不受Session的管理**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203001942107.png)

### 持久化状态

**当保存在数据库中的对象就是持久化状态了**

- **当调用session的save/saveOrUpdate/get/load/list等方法的时候，对象就是持久化状态**
- **在数据库有对应的数据**
- **受Session的管理**
- **当对对象属性进行更改的时候，会反映到数据库中!**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203002029387.png)

我们来测试一下：**当对对象属性进行更改的时候，会反映到数据库中!**

```
session.save(idCard);
idCard.setIdCardName("我是测试持久化对象");
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203002101522.png)

### 游离状态

**当Session关闭了以后，持久化的对象就变成了游离状态了...**

- **不处于session的管理**
- **数据库中有对应的记录**

![](https://img-blog.csdnimg.cn/20210203002142521.png)

有了上面的基础，我们就很容易说出它们之间的转换了

- **new出来的对象是瞬时状态->保存到数据库中(受Session管理)就是持久化状态->将session close掉就是游离状态**

## 比较hibernate的三种检索策略优缺点

> 比较hibernate的三种检索策略优缺点

**立即检索：**

- 优点： 对应用程序完全透明，不管对象处于持久化状态，还是游离状态，应用程序都可以方便的从一个对象导航到与它关联的对象；
- 缺点： 1.select语句太多；2.可能会加载应用程序不需要访问的对象白白浪费许多内存空间；
- 立即检索:`lazy=false`；

**延迟检索：**

- 优点： 由应用程序决定需要加载哪些对象，可以避免可执行多余的select语句，以及避免加载应用程序不需要访问的对象。因此能提高检索性能，并且能节省内存空间；
- 缺点： 应用程序如果希望访问游离状态代理类实例，必须保证他在持久化状态时已经被初始化；
- 延迟加载：`lazy=true`；

**迫切左外连接检索：**

- 优点： 1对应用程序完全透明，不管对象处于持久化状态，还是游离状态，应用程序都可以方便地冲一个对象导航到与它关联的对象。2使用了外连接，select语句数目少；
- 缺点： 1 可能会加载应用程序不需要访问的对象，白白浪费许多内存空间；2复杂的数据库表连接也会影响检索性能；
- 预先抓取： `fetch=“join”`；

## hibernate都支持哪些缓存策略

> hibernate都支持哪些缓存策略

usage的属性有4种：

- 放入二级缓存的对象，只读(Read-only);
- 非严格的读写(Nonstrict read/write)
- 读写； 放入二级缓存的对象可以读、写(Read/write)；
- 基于事务的策略(Transactional)

## hibernate里面的sorted collection 和ordered collection有什么区别

> hibernate里面的sorted collection 和ordered collection有什么区别

sorted collection

- 是在**内存中通过Java比较器**进行排序的

ordered collection

- 是在**数据库中通过order by**进行排序的

对于比较大的数据集，**为了避免在内存中对它们进行排序而出现 Java中的OutOfMemoryError，最好使用ordered collection。**

## 说下Hibernate的缓存机制

**一级缓存：**

- Hibenate中一级缓存，也叫做session的缓存，**它可以在session范围内减少数据库的访问次数！ 只在session范围有效！ Session关闭，一级缓存失效！**
- **只要是持久化对象状态的，都受Session管理，也就是说，都会在Session缓存中！**
- Session的缓存由hibernate维护，**用户不能操作缓存内容； 如果想操作缓存内容，必须通过hibernate提供的evit/clear方法操作**。

**二级缓存：**

- **二级缓存是基于应用程序的缓存，所有的Session都可以使用**
- Hibernate提供的二级缓存有默认的实现，且是一种**可插配的缓存框架**！如果用户想用二级缓存，**只需要在hibernate.cfg.xml中配置即可**； 不想用，直接移除，不影响代码。
- 如果用户觉得hibernate提供的框架框架不好用，**自己可以换其他的缓存框架或自己实现缓存框架都可以**。
- Hibernate二级缓存：**存储的是常用的类**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210203001756365.png)

## 如何优化Hibernate？

> 如何优化Hibernate？

- Ø 数据库设计调整
- Ø HQL优化
- Ø API的正确使用(如根据不同的业务类型选用不同的集合及查询API)
- Ø 主配置参数(日志，查询缓存，fetch_size, batch_size等)
- Ø 映射文件优化(ID生成策略，二级缓存，延迟加载，关联优化)
- Ø 一级缓存的管理
- Ø 针对二级缓存，还有许多特有的策略

## 谈谈Hibernate中inverse的作用

**inverse属性默认是false**，就是说关系的两端都来维护关系。

- 比如Student和Teacher是多对多关系，用一个中间表TeacherStudent维护。Gp)
- 如果Student这边inverse=”true”, 那么关系由另一端Teacher维护，就是说当插入Student时，不会操作TeacherStudent表（中间表）。只有Teacher插入或删除时才会触发对中间表的操作。所以两边都inverse=”true”是不对的，会导致任何操作都不触发对中间表的影响；当两边都inverse=”false”或默认时，会导致在中间表中插入两次关系。

如果表之间的关联关系是“一对多”的话，那么inverse只能在“一”的一方来配置！

## JDBC hibernate 和 ibatis 的区别

### jdbc:手动

- 手动写sql
- delete、insert、update要将对象的值一个一个取出传到sql中,不能直接传入一个对象。
- select:返回的是一个resultset，要从ResultSet中一行一行、一个字段一个字段的取出，然后封装到一个对象中，不直接返回一个对象。

### ibatis的特点:半自动化

- sql要手动写
- delete、insert、update:直接传入一个对象
- select:直接返回一个对象

### hibernate:全自动

- 不写sql,自动封装
- delete、insert、update:直接传入一个对象
- select:直接返回一个对象

## 在数据库中条件查询速度很慢的时候,如何优化?

1.  建索引
2.  减少表之间的关联
3.  优化sql，尽量让sql很快定位数据，不要让sql做全表查询，应该走索引,把数据量大的表排在前面
4.  简化查询字段，没用的字段不要，已经对返回结果的控制，尽量返回少量数据

## 什么是SessionFactory,她是线程安全么

**SessionFactory 是Hibrenate单例数据存储和线程安全的，以至于可以多线程同时访问**。一个SessionFactory 在启动的时候只能建立一次。SessionFactory应该包装各种单例以至于它能很简单的在一个应用代码中储存.

## get和load区别

- **get()立即查询**

- **load()懒加载**

- 1）get如果没有找到会返回null， load如果没有找到会抛出异常。
- 2）get会先查一级缓存， 再查二级缓存，然后查数据库；load会先查一级缓存，如果没有找到，就创建代理对象， 等需要的时候去查询二级缓存和数据库。

## merge的含义：

> merge的含义：

- 如果session中**存在相同持久化标识(identifier)的实例**，用用户给出的对象的状态**覆盖旧有的持久实例**
- 如果session**没有相应的持久实例**，则尝试从数据库中加载，或创建新的持久化实例,最后返回该持久实例
- **用户给出的这个对象没有被关联到session上，它依旧是脱管的**

## persist和save的区别

> persist和save的区别

- **persist不保证立即执行，可能要等到flush；**
- **persist不更新缓存；**
- save, 把一个瞬态的实例持久化标识符，及时的产生,它要返回标识符，所以它会**立即执行Sql insert**
- 使用 save() 方法保存持久化对象时，**该方法返回该持久化对象的标识属性值(即对应记录的主键值)；**
- 使用 persist() 方法来保存持久化对象时，**该方法没有任何返回值**。

## 主键生成 策略有哪些

> 主键生成 策略有哪些

**主键的自动生成策略**

- identity 自增长(mysql,db2)
- sequence 自增长(序列)， oracle中自增长是以序列方法实现**
- **native 自增长【会根据底层数据库自增长的方式选择identity或sequence】**
  - **如果是mysql数据库, 采用的自增长方式是identity**
  - **如果是oracle数据库， 使用sequence序列的方式实现自增长**
- increment 自增长(会有并发访问的问题，一般在服务器集群环境使用会存在问题。)

指定主键生成策略为**手动指定主键的值**

- **assigned**

指定主键生成策略为**UUID生成的值**

- **uuid**

**foreign(外键的方式)**

## 简述hibernate中getCurrentSession和openSession区别

- 1、getCurrentSession会绑定当前线程，而openSession不会，因为我们把hibernate交给我们的spring来管理之后，我们是有事务配置，这个有事务的线程就会绑定当前的工厂里面的每一个session，而openSession是创建一个新session。
- 2、**getCurrentSession事务是有spring来控制的**，而openSession需要我们手动开启和手动提交事务，
- 3、**getCurrentSession是不需要我们手动关闭的**，因为工厂会自己管理，而openSession需要我们手动关闭。
- 4、而**getCurrentSession需要我们手动设置绑定事务的机制**，有三种设置方式，jdbc本地的Thread、JTA、第三种是spring提供的事务管理机制org.springframework.orm.hibernate4.SpringSessionContext，而且srping默认使用该种事务管理机制

  ## Hibernate中的命名SQL查询指的是什么?

- 命名查询指的是用`<sql-query>`标签在影射文档中定义的SQL查询，可以通过使用Session.getNamedQuery()方法对它进行调用。命名查询使你可以使用你所指定的一个名字拿到某个特定的查询。
- Hibernate中的命名查询可以使用注解来定义，也可以使用我前面提到的xml影射问句来定义。在Hibernate中，@NameQuery用来定义单个的命名查询，@NameQueries用来定义多个命名查询。

## 为什么在Hibernate的实体类中要提供一个无参数的构造器这一点非常重要？

每个Hibernate实体类必须包含一个 无参数的构造器, 这是因为**Hibernate框架要使用Reflection API，通过调用Class.newInstance()来创建这些实体类的实例**。如果在实体类中找不到无参数的构造器，这个方法就会抛出一个InstantiationException异常。

## 可不可以将Hibernate的实体类定义为final类?

你**可以**将Hibernate的实体类定义为final类，但这种做法并不好。因为Hibernate会使用代理模式在延迟关联的情况下提高性能，如果你把实体类定义成final类之后，**因为 Java不允许对final类进行扩展，所以Hibernate就无法再使用代理了，** 如此一来就限制了使用可以提升性能的手段。

### Hibernate 的检索方式有哪些 ?  

① 导航对象图检索 （根据已经加载的对象，导航到其他对象。） 
② OID检索  （按照对象的OID来检索对象。）
③ HQL检索  （使用面向对象的HQL查询语言。）
④ QBC检索  （使用QBC(Qurey By Criteria) API来检索对象。）
⑤ 本地SQL检索  （使用本地数据库的SQL查询语句。）

### Session的清理和清空有什么区别？  

清理缓存调用的是 session.flush() 方法. 而清空调用的是 session.clear() 方法.  
Session 清理缓存是指按照缓存中对象的状态的变化来同步更新数据库，但不清空缓存；清空是把Session 的缓存置空, 但不同步更新数据库；  

### hibernate 优缺点  

①. 优点:  
>对 JDBC 访问数据库的代码做了封装，简化了数据访问层繁琐的重复性代码  
>映射的灵活性, 它支持各种关系数据库, 从一对一到多对多的各种复杂关系.  
>非侵入性、移植性会好  
>缓存机制: 提供一级缓存和二级缓存  

**②. 缺点:**  

>无法对 SQL 进行优化  
>框架中使用 ORM原则, 导致配置过于复杂  
>执行效率和原生的JDBC 相比偏差: 特别是在批量数据处理的时候  
>不支持批量修改、删除  

### Hibernate 的OpenSessionView 问题  

①. 用于解决懒加载异常, 主要功能就是把 Hibernate Session 和一个请求的线程绑定在一起, 直到页面完整输出, 这样就可以保证页面读取数据的时候 Session 一直是开启的状态, 如果去获取延迟加载对象也不会报错。  
②. 问题: 如果在业务处理阶段大批量处理数据, 有可能导致一级缓存里的对象占用内存过多导致内存溢出, 另外一个是连接问题: Session 和数据库 Connection 是绑定在一起的, 如果业务处理缓慢也会导致数据库连接得不到及时的释放, 造成连接池连接不够. 所以在并发量较大的项目中不建议使用此种方式, 可以考虑使用迫切左外连接 (LEFT OUTER JOIN FETCH) 或手工对关联的对象进行初始化.  
③. 配置 Filter 的时候要放在 Struts2 过滤器的前面, 因为它要页面完全显示完后再退出.  

8.Hibernate 中getCurrentSession() 和 openSession() 的区别 ?  
①. getCurrentSession() 它会先查看当前线程中是否绑定了 Session, 如果有则直接返回, 如果没有再创建. 而openSession() 则是直接 new 一个新的 Session 并返回。  
②. 使用ThreadLocal 来实现线程 Session 的隔离。  
③. getCurrentSession() 在事务提交的时候会自动关闭 Session, 而 openSession() 需要手动关闭.  

### 说说 Hibernate 的缓存：  

Hibernate缓存包括两大类：Hibernate一级缓存和Hibernate二级缓存：  

1）. Hibernate一级缓存又称为“Session的缓存”，它是内置的，不能被卸载。由于Session对象的生命周期通常对应一个数据库事务或者一个应用事务，因此它的缓存是事务范围的缓存。在第一级缓存中，持久化类的每个实例都具有唯一的OID。  

2）.Hibernate二级缓存又称为“SessionFactory的缓存”，由于SessionFactory对象的生命周期和应用程序的整个过程对应，因此Hibernate二级缓存是进程范围或者集群范围的缓存，有可能出现并发问题，因此需要采用适当的并发访问策略，该策略为被缓存的数据提供了事务隔离级别。第二级缓存是可选的，是一个可配置的插件，在默认情况下，SessionFactory不会启用这个插件。  
当Hibernate根据ID访问数据对象的时候，首先从Session一级缓存中查；查不到，如果配置了二级缓存，那么从二级缓存中查；如果都查不到，再查询数据库，把结果按照ID放入到缓存删除、更新、增加数据的时候，同时更新缓存。

## **hibernate工作原理：**

1.通过Configuration config = new Configuration().configure();//读取并解析hibernate.cfg.xml配置文件  
2.由hibernate.cfg.xml中的<mapping resource="com/xx/User.hbm.xml"/>读取并解析映射信息  
3.通过SessionFactory sf = config.buildSessionFactory();//创建SessionFactory  
4.Session session = sf.openSession();//打开Sesssion  
5.Transaction tx = session.beginTransaction();//创建并启动事务Transation  
6.persistent operate操作数据，持久化操作  
7.tx.commit();//提交事务  
8.关闭Session  
9.关闭SesstionFactory

## 为什么要用hibernate：

1. 对JDBC访问数据库的代码做了封装，大大简化了数据访问层繁琐的重复性代码。  
2. Hibernate是一个基于JDBC的主流持久化框架，是一个优秀的ORM实现。他很大程度的简化DAO层的编码工作  
3. hibernate使用Java反射机制，而不是字节码增强程序来实现透明性。  
4. hibernate的性能非常好，因为它是个轻量级框架。映射的灵活性很出色。它支持各种关系数据库，从一对一到多对多的各种复杂关系。

## Hibernate是如何延迟加载?get与load的区别

1. 对于Hibernate get方法，Hibernate会确认一下该id对应的数据是否存在，首先在session缓存中查找，然后在二级缓存中查找，还没有就查询数据库，数据 库中没有就返回null。这个相对比较简单，也没有太大的争议。主要要说明的一点就是在这个版本(bibernate3.2以上)中get方法也会查找二级缓存！

2. Hibernate load方法加载实体对象的时候，根据映射文件上类级别的lazy属性的配置(默认为true)，分情况讨论：

(1)若为true,则首先在Session缓存中查找，看看该id对应的对象是否存在，不存在则使用延迟加载，返回实体的代理类对象(该代理类为实体类的子类，由CGLIB动态生成)。等到具体使用该对象(除获取OID以外)的时候，再查询二级缓存和数据库，若仍没发现符合条件的记录，则会抛出一个ObjectNotFoundException。

(2)若为false,就跟Hibernateget方法查找顺序一样，只是最终若没发现符合条件的记录，则会抛出一个ObjectNotFoundException。

这里get和load有两个重要区别:

如果未能发现符合条件的记录，Hibernate get方法返回null，而load方法会抛出一个ObjectNotFoundException。

load方法可返回没有加载实体数据的代 理类实例，而get方法永远返回有实体数据的对象。

总之对于get和load的根本区别，hibernate对于 load方法认为该数据在数据库中一定存在，可以放心的使用代理来延迟加载，如果在使用过程中发现了问题，只能抛异常；而对于get方 法，hibernate一定要获取到真实的数据，否则返回null。

## Hibernate中怎样实现类之间的关系?(如：一对多、多对多的关系)

类与类之间的关系主要体现在表与表之间的关系进行操作，它们都市对对象进行操作，我们程序中把所有的表与类都映射在一起，它们通过配置文件中的many-to-one、one-to-many、many-to-many。

## Hibernate的缓存机制：

**Hibernate缓存的作用：**
    Hibernate是一个持久层框架，经常访问物理数据库，为了降低应用程序对物理数据源访问的频次，从而提高应用程序的运行性能。缓存内的数据是对物理数据源中的数据的复制，应用程序在运行时从缓存读写数据，在特定的时刻或事件会同步缓存和物理数据源的数据

**Hibernate缓存分类：**

  Hibernate缓存包括两大类：Hibernate一级缓存和Hibernate二级缓存
Hibernate一级缓存又称为“Session的缓存”，它是内置的，意思就是说，只要你使用hibernate就必须使用session缓存。由于Session对象的生命周期通常对应一个数据库事务或者一个应用事务，因此它的缓存是事务范围的缓存。在第一级缓存中，持久化类的每个实例都具有唯一的OID。 

Hibernate二级缓存又称为“SessionFactory的缓存”，由于SessionFactory对象的生命周期和应用程序的整个过程对应，因此Hibernate二级缓存是进程范围或者集群范围的缓存，有可能出现并发问题，因此需要采用适当的并发访问策略，该策略为被缓存的数据提供了事务隔离级别。第二级缓存是可选的，是一个可配置的插件，在默认情况下，SessionFactory不会启用这个插件。

**什么样的数据适合存放到第二级缓存中？** 　

1 很少被修改的数据 　　
2 不是很重要的数据，允许出现偶尔并发的数据 　　
3 不会被并发访问的数据 　　
4 常量数据 　　

**不适合存放到第二级缓存的数据？** 　　
1经常被修改的数据 　　
2 .绝对不允许出现并发访问的数据，如财务数据，绝对不允许出现并发 　　
3 与其他应用共享的数据。 

**Hibernate查找对象如何应用缓存？**
当Hibernate根据ID访问数据对象的时候，首先从Session一级缓存中查；查不到，如果配置了二级缓存，那么从二级缓存中查；如果都查不到，再查询数据库，把结果按照ID放入到缓存
删除、更新、增加数据的时候，同时更新缓存

Hibernate 的检索方式有哪些 
无论何时，我们在管理Hibernate缓存（Managing the caches）时，当你给save()、update()或saveOrUpdate()方法传递一个对象时，或使用load()、 get()、list()、iterate() 或scroll()方法获得一个对象时, 该对象都将被加入到Session的内部缓存中。 

当随后flush()方法被调用时，对象的状态会和数据库取得同步。 如果你不希望此同步操作发生，或者你正处理大量对象、需要对有效管理内存时，你可以调用evict() 方法，从一级缓存中去掉这些对象及其集合。 

## 如何优化Hibernate？

1.使用双向一对多关联，不使用单向一对多  
2.灵活使用单向一对多关联  
3.不用一对一，用多对一取代  
4.配置对象缓存，不使用集合缓存  
5.一对多集合使用Bag,多对多集合使用Set  

6. 继承类使用显式多态

7. 表字段要少，表关联不要怕多，有二级缓存撑腰

### 什么是Hibernate的并发机制？怎么去处理并发问题？

Hibernate并发机制：

a、Hibernate的Session对象是非线程安全的,对于单个请求,单个会话,单个的工作单元(即单个事务,单个线程),它通常只使用一次, 然后就丢弃。

如果一个Session 实例允许共享的话，那些支持并发运行的,例如Http request,session beans将会导致出现资源争用。

如果在Http Session中有hibernate的Session的话,就可能会出现同步访问Http Session。只要用户足够快的点击浏览器的“刷新”, 就会导致两个并发运行的线程使用同一个Session。

b、多个事务并发访问同一块资源,可能会引发第一类丢失更新，脏读，幻读，不可重复读，第二类丢失更新一系列的问题。

**解决方案：设置事务隔离级别。**  
Serializable：串行化。隔离级别最高  
Repeatable Read：可重复读  
Read Committed：已提交数据读  
Read Uncommitted：未提交数据读。隔离级别最差  

**设置锁：乐观锁和悲观锁**。  

**乐观锁**：使用版本号或时间戳来检测更新丢失,在的映射中设置 optimistic-lock=”all”可以在没有版本或者时间戳属性映射的情况下实现 版本检查，此时Hibernate将比较一行记录的每个字段的状态 行级悲观锁：Hibernate总是使用数据库的锁定机制，从不在内存中锁定对象！只要为JDBC连接指定一下隔 离级别，然后让数据库去搞定一切就够了。类LockMode 定义了Hibernate所需的不同的锁定级别：LockMode.UPGRADE,LockMode.UPGRADE_NOWAIT,LockMode.READ;

### update和saveOrUpdate的区别？

update()和saveOrUpdate()是用来对跨Session的PO进行状态管理的。  
update()方法操作的对象必须是持久化了的对象。也就是说，如果此对象在数据库中不存在的话，就不能使用update()方法。  
saveOrUpdate()方法操作的对象既可以使持久化了的，也可以使没有持久化的对象。如果是持久化了的对象调用saveOrUpdate()则会 更新数据库中的对象；如果是未持久化的对象使用此方法,则save到数据库中。

### 比较hibernate的三种检索策略优缺点

**1立即检索；**  

优点： 对应用程序完全透明，不管对象处于持久化状态，还是游离状态，应用程序都可以方便的从一个对象导航到与它关联的对象；  

缺点： 1.select语句太多；2.可能会加载应用程序不需要访问的对象白白浪费许多内存空间；  

**2延迟检索**：  

优点： 由应用程序决定需要加载哪些对象，可以避免可执行多余的select语句，以及避免加载应用程序不需要访问的对象。因此能提高检索性能，并且能节省内存空间；  

缺点： 应用程序如果希望访问游离状态代理类实例，必须保证他在持久化状态时已经被初始化；  

**3 迫切左外连接检索**  

优点： 1对应用程序完全透明，不管对象处于持久化状态，还是游离状态，应用程序都可以方便地冲一个对象导航到与它关联的对象。2使用了外连接，select语句数目少；  

缺点： 1 可能会加载应用程序不需要访问的对象，白白浪费许多内存空间；2复杂的数据库表连接也会影响检索性能；

### 如何在控制台看到hibernate生成并执行的sql

在定义数据库和数据库属性的文件applicationConfig.xml里面，把hibernate.show_sql 设置为true  
这样生成的SQL就会在控制台出现了  
注意：这样做会加重系统的负担，不利于性能调优

### hibernate都支持哪些缓存策略

**Read-only:** 这种策略适用于那些频繁读取却不会更新的数据，这是目前为止最简单和最有效的缓存策略  
**Read/write:**这种策略适用于需要被更新的数据，比read-only更耗费资源，在非JTA环境下，每个事务需要在session.close和session.disconnect()被调用  
**Nonstrict read/write:** 这种策略不保障两个同时进行的事务会修改同一块数据，这种策略适用于那些经常读取但是极少更新的数据  
 **Transactional:** 这种策略是完全事务化得缓存策略，可以用在JTA环境下

### 什么是SessionFactory，是线程安全么？

SessionFactory 是Hibrenate单例数据存储和线程安全的，以至于可以多线程同时访问。一个SessionFactory 在启动的时候只能建立一次。SessionFactory应该包装各种单例以至于它能很简单的在一个应用代码中储存.
