## 什么是微服务架构

微服务架构就是将单体的应用程序分成多个应用程序，这多个应用程序就成为微服务，每个微服务运行在自己的进程中，并使用轻量级的机制通信。

这些服务围绕业务能力来划分，并通过自动化部署机制来独立部署。这些服务可以使用不同的编程语言，不同数据库，以保证最低限度的集中式管理。

## 为什么需要学习Spring Cloud

- 首先springcloud基于spingboot的优雅简洁，可还记得我们被无数xml支配的恐惧？可还记得springmvc，mybatis错综复杂的配置，有了spingboot，这些东西都不需要了，spingboot好处不再赘诉，springcloud就基于SpringBoot把市场上优秀的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理

- 什么叫做开箱即用？即使是当年的黄金搭档dubbo+zookeeper下载配置起来也是颇费心神的！而springcloud完成这些只需要一个jar的依赖就可以了！

- springcloud大多数子模块都是直击痛点，像zuul解决的跨域，fegin解决的负载均衡，hystrix的熔断机制等等等等

## Spring Cloud 是什么

- Spring Cloud是一系列框架的有序集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、智能路由、消息总线、负载均衡、断路器、数据监控等，都可以用Spring Boot的开发风格做到一键启动和部署。

- Spring Cloud并没有重复制造轮子，它只是将各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过Spring Boot风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包。

## SpringCloud的优缺点

**优点：**

- 1.耦合度比较低。不会影响其他模块的开发。

- 2.减轻团队的成本，可以并行开发，不用关注其他人怎么开发，先关注自己的开发。

- 3.配置比较简单，基本用注解就能实现，不用使用过多的配置文件。

- 4.微服务跨平台的，可以用任何一种语言开发。

- 5.每个微服务可以有自己的独立的数据库也有用公共的数据库。

- 6.直接写后端的代码，不用关注前端怎么开发，直接写自己的后端代码即可，然后暴露接口，通过组件进行服务通信。

**缺点：**

- 1.部署比较麻烦，给运维工程师带来一定的麻烦。

- 2.针对数据的管理比麻烦，因为微服务可以每个微服务使用一个数据库。

- 3.系统集成测试比较麻烦

- 4.性能的监控比较麻烦。【最好开发一个大屏监控系统】

总的来说优点大过于缺点，目前看来Spring Cloud是一套非常完善的分布式框架，目前很多企业开始用微服务、Spring Cloud的优势是显而易见的。因此对于想研究微服务架构的同学来说，学习Spring Cloud是一个不错的选择。

## SpringBoot和SpringCloud的区别？

- SpringBoot专注于快速方便的开发单个个体微服务。

- SpringCloud是关注全局的微服务协调整理治理框架，它将SpringBoot开发的一个个单体微服务整合并管理起来，

- 为各个微服务之间提供，配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁、决策竞选、分布式会话等等集成服务

- SpringBoot可以离开SpringCloud独立使用开发项目， 但是SpringCloud离不开SpringBoot ，属于依赖的关系

- SpringBoot专注于快速、方便的开发单个微服务个体，SpringCloud关注全局的服务治理框架。

## Spring Cloud和SpringBoot版本对应关系

> | Spring Cloud Version | SpringBoot Version |
> | --- | --- |
> | Hoxton | 2.2.x |
> | Greenwich | 2.1.x |
> | Finchley | 2.0.x |
> | Edgware | 1.5.x |
> | Dalston | 1.5.x |

## SpringCloud由什么组成

这就有很多了，我讲几个开发中最重要的
- Spring Cloud Eureka（现在闭源了）：服务注册与发现

- Spring Cloud Zuul（gateway）：服务网关
- Spring Cloud Ribbon：客户端负载均衡
- Spring Cloud Feign：声明性的Web服务客户端
- Spring Cloud Hystrix：断路器
- Spring Cloud Config：分布式统一配置管理
- 等20几个框架，开源一直在更新

## spring cloud架构图

![](https://img-blog.csdnimg.cn/2021012815435240.png)

## 使用 Spring Boot 开发分布式微服务时，我们面临什么问题

- （1）与分布式系统相关的复杂性-这种开销包括网络问题，延迟开销，带宽问题，安全问题。

- （2）服务发现-服务发现工具管理群集中的流程和服务如何查找和互相交谈。它涉及一个服务目录，在该目录中注册服务，然后能够查找并连接到该目录中的服务。

- （3）冗余-分布式系统中的冗余问题。

- （4）负载平衡 \--负载平衡改善跨多个计算资源的工作负荷，诸如计算机，计算机集群，网络链路，中央处理单元，或磁盘驱动器的分布。

- （5）性能-问题 由于各种运营开销导致的性能问题。

## 服务注册和发现是什么意思？Spring Cloud 如何实现？

当我们开始一个项目时，我们通常在属性文件中进行所有的配置。随着越来越多的服务开发和部署，添加和修改这些属性变得更加复杂。

有些服务可能会下降，而某些位置可能会发生变化。手动更改属性可能会产生问题。 Eureka 服务注册和发现可以在这种情况下提供帮助。由于所有服务都在 Eureka 服务器上注册并通过调用 Eureka 服务器完成查找，因此无需处理服务地点的任何更改和处理。

## 什么是Eureka

Eureka作为SpringCloud的服务注册功能服务器，他是服务注册中心，系统中的其他服务使用Eureka的客户端将其连接到Eureka Service中，并且保持心跳，这样工作人员可以通过Eureka Service来监控各个微服务是否运行正常。

## eureka 原理图

![](https://img-blog.csdnimg.cn/20210128154502803.png)

## Eureka怎么实现高可用

Eureka 的集群搭建方法很简单：每一台 Eureka 只需要在配置中指定另外多个 Eureka 的地址就可以实现一个集群的搭建了。

## 什么是Eureka的自我保护模式，

默认情况下，如果Eureka Service在一定时间内没有接收到某个微服务的心跳，Eureka Service会进入自我保护模式，在该模式下Eureka Service会保护服务注册表中的信息，不在删除注册表中的数据，当网络故障恢复后，Eureka Servic 节点会自动退出自我保护模式

在测试环境中不建议开启这个参数，生产环境中建议开启

## DiscoveryClient的作用

- 可以从注册中心中根据服务别名获取注册的服务器信息。

## Eureka和ZooKeeper都可以提供服务注册与发现的功能,请说说两个的区别

Dubbo作为服务框架的，一般注册中心会选择zk
Spring Cloud作为服务框架的，一般服务注册中心会选择Eureka

**（1）服务注册发现的原理**

![](https://img-blog.csdnimg.cn/20210128153000177.png)

Eureka，peer-to-peer，部署一个集群，但是集群里每个机器的地位是对等的，各个服务可以向任何一个Eureka实例服务注册和服务发现，集群里任何一个Euerka实例接收到写请求之后，会自动同步给其他所有的Eureka实例 

![](https://img-blog.csdnimg.cn/20210128153045165.png)

ZooKeeper，服务注册和发现的原理，Leader + Follower两种角色，只有Leader可以负责写也就是服务注册，他可以把数据同步给Follower，读的时候leader/follower都可以读

1.  ZooKeeper中的节点服务挂了就要选举  
    在选举期间注册服务瘫痪，虽然服务最终会恢复，但是选举期间不可用的，  
    选举就是改微服务做了集群，必须有一台主其他的都是从

2.  Eureka各个节点是平等关系,服务器挂了没关系，只要有一台Eureka就可以保证服务可用，数据都是最新的。 
    如果查询到的数据并不是最新的，就是因为Eureka的自我保护模式导致的

3.  Eureka本质上是一个工程，而ZooKeeper只是一个进程

4.  Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像ZooKeeper 一样使得整个注册系统瘫痪

5.  ZooKeeper保证的是CP，Eureka保证的是AP

**2）一致性保障：**  

C：一致性，Consistency;  
取舍：\(强一致性、单调一致性、会话一致性、最终一致性、弱一致性\)  

A：可用性，Availability;  

P：分区容错性，Partition tolerance;

ZooKeeper是有一个leader节点会接收数据， 然后同步写其他节点，一旦leader挂了，要重新选举leader，这个过程里为了保证C，就牺牲了A，不可用一段时间，但是一个leader选举好了，那么就可以继续写数据了，保证一致性

Eureka是peer模式，可能还没同步数据过去，结果自己就死了，此时还是可以继续从别的机器上拉取注册表，但是看到的就不是最新的数据了，但是保证了可用性，强一致，最终一致性

**（3）服务注册发现的时效性**

zk，时效性更好，注册或者是挂了，一般秒级就能感知到

eureka，默认配置非常糟糕，服务发现感知要到几十秒，甚至分钟级别，上线一个新的服务实例，到其他人可以发现他，极端情况下，可能要1分钟的时间，ribbon去获取每个服务上缓存的eureka的注册表进行负载均衡

服务故障，隔60秒才去检查心跳，发现这个服务上一次心跳是在60秒之前，隔60秒去检查心跳，超过90秒没有心跳，才会认为他死了，2分钟都过去
30秒，才会更新缓存，30秒，其他服务才会来拉取最新的注册表

三分钟都过去了，如果你的服务实例挂掉了，此时别人感知到，可能要两三分钟的时间，一两分钟的时间，很漫长

**(4)容量**

zk，不适合大规模的服务实例，因为服务上下线的时候，需要瞬间推送数据通知到所有的其他服务实例，所以一旦服务规模太大，到了几千个服务实例的时候，会导致网络带宽被大量占用

eureka，也很难支撑大规模的服务实例，因为每个eureka实例都要接受所有的请求，实例多了压力太大，扛不住，也很难到几千服务实例

之前dubbo技术体系都是用zk当注册中心，spring cloud技术体系都是用eureka当注册中心这两种是运用最广泛的，但是现在很多中小型公司以spring cloud居多，所以后面基于eureka说一下服务注册中心的生产优化

## 你们系统遇到过服务发现过慢的问题吗？怎么优化和解决的？

zk，一般来说还好，服务注册和发现，都是很快的

eureka，必须优化参数
- 服务器到注册中心心跳时间设置
- 注册中心定时检测心跳时间设置
- 心跳失效时间设置
- readWrite缓存定更新到readOnly时间设置
- 客户端定时拉取readWrite缓存时间设置

## 什么是网关?

服务网关是统一管理API的一个网络关口、通道，是整个微服务平台所有请求的唯一入口，所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有的非业务功能。

1、路由转发：接收一切外界请求，转发到后端的微服务上去；

2、过滤器：在服务网关中可以完成一系列的横切功能，例如权限校验、限流以及监控等，

这些都可以通过过滤器完成（其实路由转发也是通过过滤器实现的）。

## 网关的作用是什么

- 统一管理微服务请求，权限控制、负载均衡、路由转发、监控、限流、安全控制黑名单和白名单等

## 什么是Spring Cloud Zuul（服务网关）

- Zuul是对SpringCloud提供的成熟对的路由方案，他会根据请求的路径不同，网关会定位到指定的微服务，并代理请求到不同的微服务接口，他对外隐蔽了微服务的真正接口地址。  
  三个重要概念：动态路由表，路由定位，反向代理：

  - 动态路由表：Zuul支持Eureka路由，手动配置路由，这俩种都支持自动更新
  - 路由定位：根据请求路径，Zuul有自己的一套定位服务规则以及路由表达式匹配
  - 反向代理：客户端请求到路由网关，网关受理之后，在对目标发送请求，拿到响应之后在 给客户端

- 它可以和Eureka、Ribbon、Hystrix等组件配合使用，

- Zuul的应用场景：

  - 对外暴露，权限校验，限流等

## 网关与过滤器有什么区别

- 网关是对所有服务的请求进行分析过滤，过滤器是对单个服务而言。

## 常用网关框架有那些？

- Nginx、Zuul、Gateway

## Zuul与Nginx有什么区别？

1) 首先 , Nginx是C语言开发,而 Zuul 是Java语言开发

2) 其次，Nginx负载均衡实现,采用服务器实现负载均衡,而Zuul负载均衡的实现是采用 Ribbon  + Eureka 来实现本地负载均衡.

3) Nginx适合于服务器端负载均衡,Zuul适合微服务中实现网关

4) Nginx相比Zuul功能会更加强大,因为Nginx整合一些脚本语言( Nginx + lua )

5) Nginc 是一个高性能的HTTP 和反向代理服务器, 也是一个 IMAP / POP3 /SMIP 服务器. Zuul是 Spring Cloud  Netflix 中的开源的一个API Gateway 服务器,本质上是一个web servlet 应用, 提供动态路由,监控,弹性,安全等边缘服务的框架. Zuul 相当于是设备和Netflix 流应用的Web 网站后端所有请求的前门

## 如何设计一套API接口

考虑到API接口的分类可以将API接口分为开发API接口和内网API接口，内网API接口用于局域网，为内部服务器提供服务。开放API接口用于对外部合作单位提供接口调用，需要遵循Oauth2.0权限认证协议。同时还需要考虑安全性、幂等性等问题。

## ZuulFilter常用有那些方法

- Run\(\)：过滤器的具体业务逻辑

- shouldFilter\(\)：判断过滤器是否有效

- filterOrder\(\)：过滤器执行顺序

- filterType\(\)：过滤器拦截位置

## 如何实现动态Zuul网关路由转发

通过path配置拦截请求，通过ServiceId到配置中心获取转发的服务列表，Zuul内部使用Ribbon实现本地负载均衡和转发。

## Zuul网关如何搭建集群

使用Nginx的upstream设置Zuul服务集群，通过location拦截请求并转发到upstream，默认使用轮询机制对Zuul集群发送请求。

## Zuul一版本和Zuul二版本的区别

目前spring cloud只集成了zuul1。zuul2是Netflix在2018年5月推出，它最大的特点就是支持异步调用 (zuul1仅支持同步) ，可惜springcloud暂时没有计划集成zuul2，而且还推出spring cloud gateway来替代zuul1。

**zuul一版本（同步阻塞）：**

本质上就是一个同步 Servlet，每来一个请求，zuul会专门分配一个线程去处理，然后转发到后端服务，后端再启线程处理请求，后端处理时网关的线程会阻塞，当请求数量比较大时，很容易造成线程池被沾满而无法接受新的请求，Netflix 为此还专门研发了Hystrix熔断组件来解决慢服务耗尽资源问题。

**zuul2的编程模型（异步非阻塞）：**

zuul2是基于Netty实现的异步非阻塞编程模型，一般异步模式的本质都是使用队列 Queue(或称总线 Bus)。网关中会有一个队列专门处理用户请求，一个队列专门负责后端服务调用，中间有个事件环线程 (Event Loop Thread)同时监听两个队列，它的主要作用是将请求转发给后端，并将后端服务的处理结果返回给客户端，用队列的形式减轻了前端请求数量的压力。

![](https://img-blog.csdnimg.cn/20210128143433786.png)

Zuul1 同步编程模型简单，门槛低，开发运维方便，容易调试定位问题。Zuul2 门槛高，调试不方便。 

Zuul1 监控埋点容易，比如和调用链监控工具 CAT 集成，如果你用 Zuul2 的话，CAT 不好埋点是个问题。 

Zuul1 已经开源超过 6 年，稳定成熟，坑已经被踩平。Zuul2 刚开源很新，实际落地案例不多，难说有 bug 需要踩坑。 大部分公司达不到 Netflix 那个量级，Netflix 是要应对每日千亿级流量，它们才挖空心思搞异步，一般公司亿级可能都不到，Zuul1 绰绰有余。 

Zuul1 可以集成 Hystrix 熔断组件，可以部分解决后台服务慢阻塞网关线程的问题。 Zuul1 可以使用 Servlet 3.0 规范支持的 AsyncServlet 进行优化，可以实现前端异步，支持更多的连接数，达到和 Zuul2 一样的效果，但是不用引入太多异步复杂性。

## 如果网关需要抗每秒10万的高并发访问，你应该怎么对网关进行生产优化？

![](https://img-blog.csdnimg.cn/20210128153713345.png)

Zuul网关部署的是什么配置的机器，部署32核64G，对网关路由转发的请求，每秒抗个小几万请求是不成问题的，几台Zuul网关机器

每秒是1万请求，8核16G的机器部署Zuul网关，5台机器就够了

## 如果需要部署上万服务实例，现有的服务注册中心能否抗住？如何优化？

Eureka 和 ZK都是扛不住了，（可以主动说出注册中心的缺点）

eureka：peer-to-peer，每台机器都是高并发请求，有瓶颈

zookeeper：服务上下线，全量通知其他服务，网络带宽被打满，有瓶颈

1、可以加一个数据库层（或者是 redis缓存层），每个服务定时通过数据库（redis缓存层）来更新服务注册表，然后数据库（redis缓存层）定时拉取注册中心来更新注册表。

2、可以自研，类似于 redis 集群 加主备架构，将压力分散开。按需拉取局部的注册表。比如说服务A在，注册中心1，那么只用拉取注册中心1的注册表。而不用将注册中心1,2,3,4等等其他注册拉取过来。缓解压力。

![](https://img-blog.csdnimg.cn/20210128154142558.png)

## 说说生产环境下，你们是怎么实现网关对服务的动态路由的？

• 通过数据库+网关定时拉取数据库 服务注册中心配置。
• 首先开发注册中心配置系统，通过页面可以动态的将增加新老服务。写入到数据库。
• 同时也可以通过拉取eureka来最新的服务注册中心配置。写入到数据库。
• 网关定时10秒拉取数据库的最新配置。
这样好处减少了eureka的压力，同时当注册中心服务宕机，也不影响当前网关的路由。



## 负载平衡的意义什么？

- 简单来说： 先将集群，集群就是把一个的事情交给多个人去做，假如要做1000个产品给一个人做要10天，我叫10个人做就是一天，这就是集群，负载均衡的话就是用来控制集群，他把做的最多的人让他慢慢做休息会，把做的最少的人让他加量让他做多点。

- 在计算中，负载平衡可以改善跨计算机，计算机集群，网络链接，中央处理单元或磁盘驱动器等多种计算资源的工作负载分布。负载平衡旨在优化资源使用，最大化吞吐量，最小化响应时间并避免任何单一资源的过载。使用多个组件进行负载平衡而不是单个组件可能会通过冗余来提高可靠性和可用性。负载平衡通常涉及专用软件或硬件，例如多层交换机或域名系统服务器进程。

## Ribbon是什么？

- Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法

- Ribbon客户端组件提供一系列完善的配置项，如连接超时，重试等。简单的说，就是在配置文件中列出后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随即连接等）去连接这些机器。我们也很容易使用Ribbon实现自定义的负载均衡算法。（有点类似Nginx）

## Nginx与Ribbon的区别

Nginx是反向代理同时可以实现负载均衡，nginx拦截客户端请求采用负载均衡策略根据upstream配置进行转发，相当于请求通过nginx服务器进行转发。

Ribbon是客户端负载均衡，从注册中心读取目标服务器信息，然后客户端采用轮询策略对服务直接访问，全程在客户端操作。

## Ribbon底层实现原理

底层的话，使用HTTP通信的框架组件，HttpClient，先得使用Ribbon去本地的Eureka注册表的缓存里获取出来对方机器的列表，对同一接口请求进行计数，使用\%取余算法获取目标服务集群索引，进行负载均衡，选出一台机器，接着针对那台机器发送 Http请求过去即可

## @LoadBalanced注解的作用

开启客户端负载均衡。

## 什么是断路器

当一个服务调用另一个服务由于网络原因或自身原因出现问题，调用者就会等待被调用者的响应 当更多的服务请求到这些资源导致更多的请求等待，发生连锁效应（雪崩效应）

**断路器有三种状态**

  - 打开状态：一段时间内 达到一定的次数无法调用 并且多次监测没有恢复的迹象 断路器完全打开 那么下次请求就不会请求到该服务
  
  - 半开状态：短时间内 有恢复迹象 断路器会将部分请求发给该服务，正常调用时 断路器关闭
  
  - 关闭状态：当服务一直处于正常状态 能正常调用

## 什么是 Hystrix？

在分布式系统，我们一定会依赖各种服务，那么这些个服务一定会出现失败的情况，就会导致雪崩，Hystrix就是这样的一个工具，防雪崩利器，它具有服务降级，服务熔断，服务隔离，监控等一些防止雪崩的技术。

- Hystrix有四种防雪崩方式:

  - 服务降级：接口调用失败就调用本地的方法返回一个空
  
  - 服务熔断：接口调用失败就会进入调用接口提前定义好的一个熔断的方法，返回错误信息
  
  - 服务隔离：隔离服务之间相互影响
  
  - 服务监控：在服务发生调用时，会将每秒请求数、成功请求数等运行指标记录下来。

## 谈谈服务雪崩效应

雪崩效应是在大型互联网项目中，当某个服务发生宕机时，调用这个服务的其他服务也会发生宕机，大型项目的微服务之间的调用是互通的，这样就会将服务的不可用逐步扩大到各个其他服务中，从而使整个项目的服务宕机崩溃，发生雪崩效应的原因有以下几点

- 单个服务的代码存在bug.

- 请求访问量激增导致服务发生崩溃\(如大型商城的枪红包，秒杀功能\).

- 服务器的硬件故障也会导致部分服务不可用.

## 在微服务中，如何保护服务\?

一般使用使用Hystrix框架，实现服务隔离来避免出现服务的雪崩效应，从而达到保护服务的效果。当微服务中，高并发的数据库访问量导致服务线程阻塞，使单个服务宕机，服务的不可用会蔓延到其他服务，引起整体服务灾难性后果，使用服务降级能有效为不同的服务分配资源，一旦服务不可用则返回友好提示，不占用其他服务资源，从而避免单个服务崩溃引发整体服务的不可用.

## 服务雪崩效应产生的原因

因为Tomcat默认情况下只有一个线程池来维护客户端发送的所有的请求，这时候某一接口在某一时刻被大量访问就会占据tomcat线程池中的所有线程，其他请求处于等待状态，无法连接到服务接口。

## 谈谈服务降级、熔断、服务隔离

- 服务降级：当客户端请求服务器端的时候，防止客户端一直等待，不会处理业务逻辑代码，直接返回一个友好的提示给客户端。

- 服务熔断是在服务降级的基础上更直接的一种保护方式，当在一个统计时间范围内的请求失败数量达到设定值（requestVolumeThreshold）或当前的请求错误率达到设定的错误率阈值（errorThresholdPercentage）时开启断路，之后的请求直接走fallback方法，在设定时间（sleepWindowInMilliseconds）后尝试恢复。

- 服务隔离就是Hystrix为隔离的服务开启一个独立的线程池，这样在高并发的情况下不会影响其他服务。服务隔离有线程池和信号量两种实现方式，一般使用线程池方式。

## 服务降级底层是如何实现的？

Hystrix实现服务降级的功能是通过重写HystrixCommand中的getFallback\(\)方法，当Hystrix的run方法或construct执行发生错误时转而执行getFallback\(\)方法。

## 什么是Feign？

Feign 是一个声明web服务客户端，这使得编写web服务客户端更容易

他将我们需要调用的服务方法定义成抽象方法保存在本地就可以了，不需要自己构建Http请求了，直接调用接口就行了，不过要注意，调用方法要和本地抽象方法的签名完全一致。

## Feign 原理

在配置类上，加上@EnableFeginClients，那么该注解是基于@Import注解，注册有关Fegin的解析注册类，这个类是实现 ImportBeanDefinitionRegistrar 这个接口，重写registryBeanDefinition 方法。

他会扫描所有加了@FeginClient 的接口，然后针对这个注解的接口生成动态代理，然后你针对fegin的动态代理去调用他方法的时候，此时会在底层生成http协议格式的请求。

## SpringCloud有几种调用接口方式

- Feign

- RestTemplate

## Ribbon和Feign调用服务的区别

- 调用方式同：Ribbon需要我们自己构建Http请求，模拟Http请求然后通过RestTemplate发给其他服务，步骤相当繁琐

- 而Feign则是在Ribbon的基础上进行了一次改进，采用接口的形式，将我们需要调用的服务方法定义成抽象方法保存在本地就可以了，不需要自己构建Http请求了，直接调用接口就行了，不过要注意，调用方法要和本地抽象方法的签名完全一致。

## 什么是 Spring Cloud Bus？

- Spring Cloud Bus就像一个分布式执行器，用于扩展的Spring Boot应用程序的配置文件，但也可以用作应用程序之间的通信通道。

- Spring Cloud Bus 不能单独完成通信，需要配合MQ支持

- Spring Cloud Bus一般是配合Spring Cloud Config做配置中心的

- Springcloud config实时刷新也必须采用SpringCloud Bus消息总线

## 什么是Spring Cloud Config?

Spring Cloud Config为分布式系统中的外部配置提供服务器和客户端支持，可以方便的对微服务各个环境下的配置进行集中式管理。

Spring Cloud Config分为Config Server和Config Client两部分。Config Server负责读取配置文件，并且暴露Http API接口，Config Client通过调用Config Server的接口来读取配置文件。

## 分布式配置中心有那些框架？

- Apollo、zookeeper、springcloud config。

## 分布式配置中心的作用？

- 动态变更项目配置信息而不必重新部署项目。

- 统一修改多个项目中的配置，避免错改漏改的一些情况

## SpringCloud Config 可以实现实时刷新吗？

- springcloud config实时刷新采用SpringCloud Bus消息总线。

## 什么是Spring Cloud Gateway?

Spring Cloud Gateway构建于 Spring 5+，基于 Spring Boot 2.x 响应式的、非阻塞式的 API。同时，它支持 websockets，和 Spring 框架紧密集成，开发体验相对来说十分不错。由于zuul2没有被spring cloud所集成，所以拿zuul1与spring cloud gateway做一些简单的比较。

网关提供API全托管服务，丰富的API管理功能，辅助企业管理大规模的API，以降低管理成本和安全风险，包括协议适配、协议转发、安全策略、防刷、流量、监控日志等贡呢。一般来说网关对外暴露的URL或者接口信息，我们统称为路由信息。如果研发过网关中间件或者使用过Zuul的人，会知道网关的核心是Filter以及Filter Chain（Filter责任链）。Sprig Cloud Gateway也具有路由和Filter的概念。下面介绍一下Spring Cloud Gateway中几个重要的概念。


- 路由。路由是网关最基础的部分，路由信息有一个ID、一个目的URL、一组断言和一组Filter组成。如果断言路由为真，则说明请求的URL和配置匹配

- 断言。Java8中的断言函数。Spring Cloud Gateway中的断言函数输入类型是Spring5.0框架中的ServerWebExchange。Spring Cloud Gateway中的断言函数允许开发者去定义匹配来自于http request中的任何信息，比如请求头和参数等。

- 过滤器。一个标准的Spring webFilter。Spring cloud gateway中的filter分为两种类型的Filter，分别是Gateway Filter和Global Filter。过滤器Filter将会对请求和响应进行修改处理

## Zuul和Gateway的区别

### Zuul1.x：

- 使用的是阻塞式的 API，不支持长连接，比如 websockets。

- 底层是servlet，Zuul处理的是http请求

- 没有提供异步支持，流控等均由hystrix支持。

- 依赖包spring-cloud-starter-netflix-zuul。

### Gateway：

- Spring Boot和Spring Webflux提供的Netty底层环境，不能和传统的Servlet容器一起使用，也不能打包成一个WAR包。

- 依赖spring-boot-starter-webflux和/ spring-cloud-starter-gateway

- 提供了异步支持，提供了抽象负载均衡，提供了抽象流控，并默认实现了RedisRateLimiter限流。

![](https://img-blog.csdnimg.cn/20210128144012403.png)

### Spring Cloud Config

- Config能够管理所有微服务的配置文件

- 集中配置管理工具，分布式系统中统一的外部配置管理，默认使用Git来存储配置，可以支持客户端配置的刷新及加密、解密操作。

### Spring Cloud Netflix\(重点，这些组件用的最多\)

Netflix OSS 开源组件集成，包括Eureka、Hystrix、Ribbon、Feign、Zuul等核心组件。

- Eureka：服务治理组件，包括服务端的注册中心和客户端的服务发现机制；

- Ribbon：负载均衡的服务调用组件，具有多种负载均衡调用策略；
- Hystrix：服务容错组件，实现了断路器模式，为依赖服务的出错和延迟提供了容错能力；
- Feign：基于Ribbon和Hystrix的声明式服务调用组件；
- Zuul：API网关组件，对请求提供路由及过滤功能。

`我觉得SpringCloud的福音是Netflix，他把人家的组件都搬来进行封装了，使开发者能快速简单安全的使用`

### Spring Cloud Bus

- 用于传播集群状态变化的消息总线，使用轻量级消息代理链接分布式系统中的节点，可以用来动态刷新集群中的服务配置信息。

- 简单来说就是修改了配置文件，发送一次请求，所有客户端便会重新读取配置文件。需要利用中间插件MQ

### Spring Cloud Consul

Consul 是 HashiCorp 公司推出的开源工具，用于实现分布式系统的服务发现与配置。与其它分布式服务注册与发现的方案，Consul 的方案更“一站式”，内置了服务注册与发现框架、分布一致性协议实现、健康检查、Key/Value 存储、多数据中心方案，不再需要依赖其它工具（比如 ZooKeeper 等）。使用起来也较为简单。

Consul 使用 Go 语言编写，因此具有天然可移植性\(支持Linux、windows和Mac OS X\)；安装包仅包含一个可执行文件，方便部署，与 Docker 等轻量级容器可无缝配合。

### Spring Cloud Security

- 安全工具包，他可以对

  - 对Zuul代理中的负载均衡从前端到后端服务中获取SSO令牌
  - 资源服务器之间的中继令牌
  - 使Feign客户端表现得像`OAuth2RestTemplate`（获取令牌等）的拦截器
  - 在Zuul代理中配置下游身份验证

- Spring Cloud Security提供了一组原语，用于构建安全的应用程序和服务，而且操作简便。可以在外部（或集中）进行大量配置的声明性模型有助于实现大型协作的远程组件系统，通常具有中央身份管理服务。它也非常易于在Cloud Foundry等服务平台中使用。在Spring Boot和Spring Security OAuth2的基础上，可以快速创建实现常见模式的系统，如单点登录，令牌中继和令牌交换。

### Spring Cloud Sleuth

在微服务中，通常根据业务模块分服务，项目中前端发起一个请求，后端可能跨几个服务调用才能完成这个请求（如下图）。

如果系统越来越庞大，服务之间的调用与被调用关系就会变得很复杂，假如一个请求中需要跨几个服务调用，其中一个服务由于网络延迟等原因挂掉了，那么这时候我们需要分析具体哪一个服务出问题了就会显得很困难。

Spring Cloud Sleuth服务链路跟踪功能就可以帮助我们快速的发现错误根源以及监控分析每条请求链路上的性能等等。  

![](https://img-blog.csdnimg.cn/20200411194744815.jpg)

### Spring Cloud Stream

- 轻量级事件驱动微服务框架，可以使用简单的声明式模型来发送及接收消息，主要实现为Apache Kafka及RabbitMQ。

### Spring Cloud Task

Spring Cloud Task的目标是为Spring Boot应用程序提供创建短运行期微服务的功能。

在Spring Cloud Task中，我们可以灵活地动态运行任何任务，按需分配资源并在任务完成后检索结果。

Tasks是Spring Cloud Data Flow中的一个基础项目，允许用户将几乎任何Spring Boot应用程序作为一个短期任务执行。

### Spring Cloud Zookeeper

- SpringCloud支持三种注册方式Eureka， Consul\(go语言编写\)，zookeeper

- Spring Cloud Zookeeper是基于Apache Zookeeper的服务治理组件。

### Spring Cloud OpenFeign

Feign是一个声明性的Web服务客户端。它使编写Web服务客户端变得更容易。要使用Feign，我们可以将调用的服务方法定义成抽象方法保存在本地添加一点点注解就可以了，不需要自己构建Http请求了，直接调用接口就行了，不过要注意，调用方法要和本地抽象方法的签名完全一致。

## Spring Cloud的版本关系

Spring Cloud是一个由许多子项目组成的综合项目，各子项目有不同的发布节奏。 为了管理Spring Cloud与各子项目的版本依赖关系，发布了一个清单，其中包括了某个Spring Cloud版本对应的子项目版本。 

为了避免Spring Cloud版本号与子项目版本号混淆，Spring Cloud版本采用了名称而非版本号的命名，这些版本的名字采用了伦敦地铁站的名字，根据字母表的顺序来对应版本时间顺序，例如Angel是第一个版本，Brixton是第二个版本。 

当Spring Cloud的发布内容积累到临界点或者一个重大BUG被解决后，会发布一个"service releases"版本，简称SRX版本，比如Greenwich.SR2就是Spring Cloud发布的Greenwich版本的第2个SRX版本。目前Spring Cloud的最新版本是Hoxton。

### Spring Cloud和SpringBoot版本对应关系

> | Spring Cloud Version | SpringBoot Version |
> | --- | --- |
> | Hoxton | 2.2.x |
> | Greenwich | 2.1.x |
> | Finchley | 2.0.x |
> | Edgware | 1.5.x |
> | Dalston | 1.5.x |

### Spring Cloud和各子项目版本对应关系

- Edgware.SR6：我理解为最低版本号

- Greenwich.SR2 :我理解为最高版本号

- Greenwich.BUILD-SNAPSHOT（快照）：是一种特殊的版本，指定了某个当前的开发进度的副本。不同于常规的版本，几乎每天都要提交更新的版本，如果每次提交都申明一个版本号那不是版本号都不够用？

> | Component | Edgware.SR6 | Greenwich.SR2 | Greenwich.BUILD-SNAPSHOT |
> | --- | --- | --- | --- |
> | spring-cloud-aws | 1.2.4.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-bus | 1.3.4.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-cli | 1.4.1.RELEASE | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT |
> | spring-cloud-commons | 1.3.6.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-contract | 1.2.7.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-config | 1.4.7.RELEASE | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT |
> | spring-cloud-netflix | 1.4.7.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-security | 1.2.4.RELEASE | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT |
> | spring-cloud-cloudfoundry | 1.1.3.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-consul | 1.3.6.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-sleuth | 1.3.6.RELEASE | 2.1.1.RELEASE | 2.1.2.BUILD-SNAPSHOT |
> | spring-cloud-stream | Ditmars.SR5 | Fishtown.SR3 | Fishtown.BUILD-SNAPSHOT |
> | spring-cloud-zookeeper | 1.2.3.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-boot | 1.5.21.RELEASE | 2.1.5.RELEASE | 2.1.8.BUILD-SNAPSHOT |
> | spring-cloud-task | 1.2.4.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-vault | 1.1.3.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-gateway | 1.0.3.RELEASE | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-openfeign |  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT |
> | spring-cloud-function | 1.0.2.RELEASE | 2.0.2.RELEASE | 2.0.3.BUILD-SNAPSHOT |