#### 1、介绍一下spring

关于Spring的话，我们平时做项目一直都在用，不管是使用ssh还是使用ssm，都可以整合。Spring里面主要的就三点，也就是核心思想，IOC控制反转，DI依赖注入，AOP切面编程

我先来说说IOC吧，IOC就是spring里的控制反转，把类的控制权呢交给spring来管理，我们在使用的时候，在spring的配置文件中，配置好bean标签，以及类的全路径，如果有参数，然后在配置上相应的参数。这样的话，spring就会给我们通过反射的机制实例化这个类，同时放到spring容器当中去。

我们在使用的时候，需要结合DI依赖注入使用，把我们想使用的类注入到需要的地方就可以，依赖注入的方式有构造器注入、getset注入还有注解注入。我们现在都使用`@autowired`或者`@Resource`注解的方式注入。

然后就是AOP切面编程，他可以在不改变源代码的情况下对代码功能的一个增强。我们在配置文件中配置好切点，然后去实现切面的逻辑就可以实现代码增强，这个代码增强，包括在切点的执行前，执行中，执行后都可以进行增强逻辑处理，不用改变源代码，这块我们项目中一般用于权限认证、日志、事务处理这几个地方。



#### 2、AOP的实现原理

这块呢，我看过spring的源码，底层就是动态代理来实现的，所谓的动态代理就是说 AOP 框架不会去修改字节码，而是每次运行时在内存中临时为方法生成一个 AOP 对象，这个 AOP 对象包含了 ，目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

Spring AOP 中的动态代理主要有两种方式，JDK 动态代理和 CGLIB 动态代理： 

- JDK 动态代理只提供接口的代理，不支持类的代理。核心InvocationHandler 接口和 Proxy 类，InvocationHandler 通过 invoke()方法反射来调用目标类中的代码，动态地将横切逻辑和业务编织在一起；接着，Proxy 利用InvocationHandler 动态创建一个符合接口的的实例，生成目标类的代理对象。

- 如果代理类没有实现 InvocationHandler 接口，那么 Spring AOP 会选择使用 CGLIB 来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成指定类的一个子类对象，并覆盖其中特定方法并添加增强代码，从而实现 AOP。CGLIB 是通过继承的方式做的动态代理，因此如果某个类被标记为 final，那么它是无法使用 CGLIB 做动态代理的。不过在我们的业务场景中没有代理过final的类，基本上都代理的controller层实现权限以及日志，还有就是service层实现事务统一管理

#### 3、详细介绍下IOC容器

Spring 提供了两种 IoC 容器，分别为 BeanFactory 和 ApplicationContext

BeanFactory 是基础类型的 IoC 容器，提供了完整的 IoC 服务支持。简单来说，BeanFactory 就是一个管理 Bean 的工厂，它主要负责初始化各种 Bean，并调用它们的生命周期方法。

ApplicationContext 是 BeanFactory 的子接口，也被称为应用上下文。它不仅提供了 BeanFactory 的所有功能，还添加了对 i18n（国际化）、资源访问、事件传播等方面的良好支持。

他俩的主要区别在于，如果 Bean 的某一个属性没有注入，则使用 BeanFacotry 加载后，在第一次调用 getBean() 方法时会抛出异常，但是呢ApplicationContext 会在初始化时自检，这样有利于检查所依赖的属性是否注入。

因此，在实际开发中，通常都选择使用 ApplicationContext



#### 4、`@Autowired` 和 `@Resource`的区别

`@Autowired` 默认是按照类型注入的，如果这个类型没找到，会根据名称去注入，如果在用的时候需要指定名称，可以加注解`@Qualifier("指定名称的类")`

`@Resource`注解也可以从容器中注入bean，默认是按照名称注入的，如果这个名称的没找到，就会按照类型去找，也可以在注解里直接指定名称`@Resource(name="类的名称")`



#### 5、springbean的生命周期

生命周期这块无非就是从创建到销毁的过程

spring容器可以管理 singleton 作用域 Bean 的生命周期，在此作用域下，Spring 能够精确地知道该 Bean 何时被创建，何时初始化完成，以及何时被销毁。

而对于 prototype 作用域的 Bean，Spring 只负责创建，当容器创建了 Bean 的实例后，Bean 的实例就交给客户端代码管理，Spring 容器将不再跟踪其生命周期。每次客户端请求 prototype 作用域的 Bean 时，Spring 容器都会创建一个新的实例，并且不会管那些被配置成 prototype 作用域的 Bean 的生命周期。

整体来说就4个步骤：实例化bean，属性赋值，初始化bean，销毁bean

- 首先就是实例化bean，容器通过获取BeanDefinition对象中的信息进行实例化

- 然后呢就是属性赋值，利用依赖注入完成 Bean 中所有属性值的配置注入
- 接着就是初始化bean，如果在配置文件中通过 init-method 属性指定了初始化方法，则调用该初始化方法。
- 最后就是销毁bean，和init-method一样，通过给destroy-method指定函数，就可以在bean销毁前执行指定的逻辑

#### 6、springbean的作用域

Spring 容器中的 bean 可以分为 5 个范围： 

（1）singleton：单例模式，使用 singleton 定义的 Bean 在 Spring 容器中只有一个实例，这也是 Bean 默认的作用域。 controller、service、dao层基本都是singleton的

（2）prototype：原型模式，每次通过 Spring 容器获取 prototype 定义的 Bean 时，容器都将创建一个新的 Bean 实例。 

（3）request：在一次 HTTP 请求中，容器会返回该 Bean 的同一个实例。而对不同的 HTTP 请求，会返回不同的实例，该作用域仅在当前 HTTP Request 内有效。 

（4）session：在一次 HTTP Session 中，容器会返回该 Bean 的同一个实例。而对不同的 HTTP 请求，会返回不同的实例，该作用域仅在当前 HTTP Session 内有效。

（5）global-session：全局作用域，在一个全局的 HTTP Session 中，容器会返回该 Bean 的同一个实例。



#### 7、事务的传播特性

> 解读：事务的传播特性发生在事务方法与非事物方法之间相互调用的时候，在事务管理过程中，传播行为可以控制是否需要创建事务以及如何创建事务

| 属性名称                  | 值            | 描 述                                                        |
| ------------------------- | ------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | required      | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将创建新事务，默认就是这个 |
| PROPAGATION_SUPPORTS      | supports      | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将以非事务状态执行 |
| PROPAGATION_MANDATORY     | mandatory     | 支持当前事务。如果 A 方法没有事务，则抛出异常                |
| PROPAGATION_REQUIRES_NEW  | requires_new  | 将创建新的事务，如果 A 方法已经在事务中，则将 A 事务挂起     |
| PROPAGATION_NOT_SUPPORTED | not_supported | 不支持当前事务，总是以非事务状态执行。如果 A 方法已经在事务中，则将其挂起 |
| PROPAGATION_NEVER         | never         | 不支持当前事务，如果 A 方法在事务中，则抛出异常              |
| PROPAGATION.NESTED        | nested        | 嵌套事务，底层将使用 Savepoint 形成嵌套事务                  |



#### 8、事务的隔离级别

Spring 事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。真正的数据库层的事务提交和回滚是通过 binlog实现的。隔离级别有四种

- Read uncommitted (读未提交)：读未提交，允许另外一个事务可以看到这个事务未提交的数据，最低级别，任何情况都无法保证。

- Read committed (读已提交)：保证一个事务修改的数据提交后才能被另一事务读取，而且能看到该事务对已有记录的更新，可避免脏读的发生。
- Repeatable read (可重复读)：保证一个事务修改的数据提交后才能被另一事务读取，但是不能看到该事务对已有记录的更新，可避免脏读、不可重复读的发生。

- Serializable (串行化)：一个事务在执行的过程中完全看不到其他事务对数据库所做的更新，可避免脏读、不可重复读、幻读的发生。



#### 9、spring中都用了哪些设计模式

（1）工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；

（2）单例模式：Bean默认为单例模式。

（3）代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；

（4）模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。

（5）观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知被制动更新，如Spring中listener的实现–ApplicationListener。



#### 10、spring中如何处理bean在线程并发时线程安全问题

在一般情况下，只有无状态的 Bean 才可以在多线程环境下共享，在 Spring 中，绝大部分 Bean 都可以声明为 singleton 作用域，因为 Spring 对一些 Bean 中非线程安全状态采用 ThreadLocal 进行处理，解决线程安全问题。 

ThreadLocal 和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。同步机制采用了“时间换空间”的方式，仅提供一份变量，不同的线程在访问前需要获取锁，没获得锁的线程则需要排队。而 ThreadLocal 采用了“空间换时间”的方式。 

ThreadLocal 会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal 提供了线程安全的共享对象，在编写多线程代码时，可以把不安全的变量封装进 ThreadLocal。

我们项目中的拦截器里就有这样的逻辑，在我们微服务中，网关进行登录以及鉴权操作，具体的微服务中需要用到token去解析用户信息，我们就在拦截器的preHandler里定义了threadlocal，通过token解析出user的信息，后续controller以及service使用的时候，直接从threadlocal中取出用户信息的，在拦截器的afterCompletion方法中清理threadlocal中的变量，避免变量堆积消耗内存





