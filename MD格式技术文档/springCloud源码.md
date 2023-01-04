## Eureka的一些概念

- Register：服务注册
当Eureka客户端向Eureka Server注册时，它提供自身的元数据，比如IP地址、端口，运行状况指示符URL，主页等。

- Renew：服务续约
Eureka客户会每隔30秒发送一次心跳来续约。 通过续约来告知Eureka Server该Eureka客户仍然存在，没有出现问题。 正常情况下，如果Eureka Server在90秒没有收到Eureka客户的续约，它会将实例从其注册表中删除。 建议不要更改续约间隔。

- Fetch Registries：获取注册列表信息
Eureka客户端从服务器获取注册表信息，并将其缓存在本地。客户端会使用该信息查找其他服务，从而进行远程调用。该注册列表信息定期（每30秒钟）更新一次。每次返回注册列表信息可能与Eureka客户端的缓存信息不同， Eureka客户端自动处理。如果由于某种原因导致注册列表信息不能及时匹配，Eureka客户端则会重新获取整个注册表信息。 Eureka服务器缓存注册列表信息，整个注册表以及每个应用程序的信息进行了压缩，压缩内容和没有压缩的内容完全相同。Eureka客户端和Eureka 服务器可以使用JSON / XML格式进行通讯。在默认的情况下Eureka客户端使用压缩JSON格式来获取注册列表的信息。

- Cancel：服务下线
Eureka客户端在程序关闭时向Eureka服务器发送取消请求。 发送请求后，该客户端实例信息将从服务器的实例注册表中删除。该下线请求不会自动完成，它需要调用以下内容：
DiscoveryManager.getInstance().shutdownComponent()；

- Eviction 服务剔除
在默认的情况下，当Eureka客户端连续90秒没有向Eureka服务器发送服务续约，即心跳，Eureka服务器会将该服务实例从服务注册列表删除，即服务剔除。

## Eureka的高可用架构

![](https://img-blog.csdnimg.cn/20210127002225465.png)

从图可以看出在这个体系中，有2个角色，即Eureka Server和Eureka Client。而Eureka Client又分为Applicaton Service和Application Client，即服务提供者何服务消费者。 

每个区域有一个Eureka集群，并且每个区域至少有一个eureka服务器可以处理区域故障，以防服务器瘫痪。

Eureka Client向Eureka Serve注册，并将自己的一些客户端信息发送Eureka Serve。然后，Eureka Client通过向Eureka Serve发送心跳（每30秒）来续约服务的。 

如果客户端持续不能续约，那么，它将在大约90秒内从服务器注册表中删除。 注册信息和续订被复制到集群中的Eureka Serve所有节点。 

来自任何区域的Eureka Client都可以查找注册表信息（每30秒发生一次）。根据这些注册表信息，Application Client可以远程调用Applicaton Service来消费服务。

## Register服务注册

服务注册，即Eureka Client向Eureka Server提交自己的服务信息，包括IP地址、端口、service ID等信息。如果Eureka Client没有写service ID，则默认为 ${spring.application.name}。

服务注册其实很简单，在Eureka Client启动的时候，将自身的服务的信息发送到Eureka Server。

现在来简单的阅读下源码。在Maven的依赖包下，找到eureka-client-1.6.2.jar包。在com.netflix.discovery包下有个DiscoveryClient类，该类包含了Eureka Client向Eureka Server的相关方法。

其中DiscoveryClient实现了EurekaClient接口，并且它是一个单例模式，而EurekaClient继承了LookupService接口。它们之间的关系如图所示。

![](https://img-blog.csdnimg.cn/20210127002339740.png)

在DiscoveryClient类有一个服务注册的方法register()，该方法是通过Http请求向Eureka Client注册。其代码如下：

```java
boolean register() throws Throwable {
    logger.info(PREFIX + appPathIdentifier + ": registering service...");
    EurekaHttpResponse<Void> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn("{} - registration failed {}", PREFIX + appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info("{} - registration status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == 204;
}
```

在DiscoveryClient类继续追踪register()方法，它被InstanceInfoReplicator 类的run()方法调用，其中InstanceInfoReplicator实现了Runnable接口，run()方法代码如下：

```java
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

而InstanceInfoReplicator类是在DiscoveryClient初始化过程中使用的，其中有一个initScheduledTasks()方法。该方法主要开启了获取服务注册列表的信息，如果需要向Eureka Server注册，则开启注册，同时开启了定时向Eureka Server服务续约的定时任务，具体代码如下：

```java
private void initScheduledTasks() {
   ...//省略了任务调度获取注册列表的代码
    if (clientConfig.shouldRegisterWithEureka()) {
     ...
        // Heartbeat timer
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {

                instanceInfoReplicator.onDemandUpdate();
            }
        };
      ...
}
```

然后在来看Eureka server端的代码，在Maven的eureka-core:1.6.2的jar包下。打开com.netflix.eureka包，很轻松的就发现了又一个EurekaBootStrap的类，BootStrapContext具有最先初始化的权限，所以先看这个类。

```java
protected void initEurekaServerContext() throws Exception {

...//省略代码
 PeerAwareInstanceRegistry registry;
      if (isAws(applicationInfoManager.getInfo())) {
         ...//省略代码，如果是AWS的代码
      } else {
          registry = new PeerAwareInstanceRegistryImpl(
                  eurekaServerConfig,
                  eurekaClient.getEurekaClientConfig(),
                  serverCodecs,
                  eurekaClient
          );
      }

      PeerEurekaNodes peerEurekaNodes = getPeerEurekaNodes(
              registry,
              eurekaServerConfig,
              eurekaClient.getEurekaClientConfig(),
              serverCodecs,
              applicationInfoManager
      );
}
```

其中PeerAwareInstanceRegistryImpl和PeerEurekaNodes两个类看其命名，应该和服务注册以及Eureka Server高可用有关。先追踪PeerAwareInstanceRegistryImpl类，在该类有个register()方法，该方法提供了注册，并且将注册后信息同步到其他的Eureka Server服务。代码如下：

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    super.register(info, leaseDuration, isReplication);
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

其中 super.register(info, leaseDuration, isReplication)方法，点击进去到子类AbstractInstanceRegistry可以发现更多细节，其中注册列表的信息被保存在一个Map中。

replicateToPeers()方法，即同步到其他Eureka Server的其他Peers节点，追踪代码，发现它会遍历循环向所有的Peers节点注册，最终执行类PeerEurekaNodes的register()方法，该方法通过执行一个任务向其他节点同步该注册信息，代码如下：

```java
public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            },
            expiryTime
    );
}
```

经过一系列的源码追踪，可以发现PeerAwareInstanceRegistryImpl的register()方法实现了服务的注册，并且向其他Eureka Server的Peer节点同步了该注册信息，那么register()方法被谁调用了呢？

之前在Eureka Client的分析可以知道，Eureka Client是通过 http来向Eureka Server注册的，那么Eureka Server肯定会提供一个注册的接口给Eureka Client调用，那么PeerAwareInstanceRegistryImpl的register()方法肯定最终会被暴露的Http接口所调用。在Idea开发工具，按住alt+鼠标左键，可以很快定位到ApplicationResource类的addInstance ()方法，即服务注册的接口，其代码如下：

```java
 @POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {

    ...//省略代码                 
    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

## Renew服务续约

服务续约和服务注册非常类似，通过之前的分析可以知道，服务注册在Eureka Client程序启动之后开启，并同时开启服务续约的定时任务。在eureka-client-1.6.2.jar的DiscoveryClient的类下有renew()方法，其代码如下：

```java
/**
 * Renew with the eureka service by making the appropriate REST call
 */
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug("{} - Heartbeat status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == 404) {
            REREGISTER_COUNTER.increment();
            logger.info("{} - Re-registering apps/{}", PREFIX + appPathIdentifier, instanceInfo.getAppName());
            return register();
        }
        return httpResponse.getStatusCode() == 200;
    } catch (Throwable e) {
        logger.error("{} - was unable to send heartbeat!", PREFIX + appPathIdentifier, e);
        return false;
    }
}
```

另外服务端的续约接口在eureka-core:1.6.2.jar的 com.netflix.eureka包下的InstanceResource类下，接口方法为renewLease()，它是REST接口。为了减少类篇幅，省略了大部分代码的展示。其中有个registry.renew()方法，即服务续约，代码如下:

```java
@PUT
public Response renewLease(...参数省略）{
     ...  代码省略
    boolean isSuccess=registry.renew(app.getName(),id, isFromReplicaNode);
       ...  代码省略
 }
```

读者可以跟踪registry.renew的代码一直深入研究。在这里就不再多讲述。另外服务续约有2个参数是可以配置，即Eureka Client发送续约心跳的时间参数，和Eureka Server在多长时间内没有收到心跳将实例剔除的时间参数，在默认的情况下这两个参数分别为30秒和90秒，官方给的建议是不要修改，如果有特殊要求还是可以调整的，只需要分别在Eureka Client和Eureka Server修改以下参数：

```java
eureka.instance.leaseRenewalIntervalInSeconds
eureka.instance.leaseExpirationDurationInSeconds
```

最后，服务注册列表的获取、服务下线和服务剔除就不在这里进行源码跟踪解读，因为和服务注册和续约类似，有兴趣的朋友可以自己看下源码，深入理解。总的来说，通过读源码，可以发现，整体架构与前面小节的eureka 的高可用架构图完全一致。

## Eureka Client注册一个实例为什么这么慢

- Eureka Client一启动（不是启动完成），不是立即向Eureka Server注册，它有一个延迟向服务端注册的时间，通过跟踪源码，可以发现默认的延迟时间为40秒，源码在eureka-client-1.6.2.jar的DefaultEurekaClientConfig类下，代码如下：

```java
public int getInitialInstanceInfoReplicationIntervalSeconds() {
  return configInstance.getIntProperty(
      namespace + INITIAL_REGISTRATION_REPLICATION_DELAY_KEY, 40).get();
}
```

- Eureka Server的响应缓存
Eureka Server维护每30秒更新的响应缓存,可通过更改配置eureka.server.responseCacheUpdateIntervalMs来修改。 所以即使实例刚刚注册，它也不会出现在调用/ eureka / apps REST端点的结果中。

- Eureka Server刷新缓存
Eureka客户端保留注册表信息的缓存。 该缓存每30秒更新一次（如前所述）。 因 此，客户端决定刷新其本地缓存并发现其他新注册的实例可能需要30秒。

- LoadBalancer Refresh
Ribbon的负载平衡器从本地的Eureka Client获取服务注册列表信息。Ribbon本身还维护本地缓存，以避免为每个请求调用本地客户端。 此缓存每30秒刷新一次（可由ribbon.ServerListRefreshInterval配置）。 所以，可能需要30多秒才能使用新注册的实例。

综上几个因素，一个新注册的实例，特别是启动较快的实例（默认延迟40秒注册），不能马上被Eureka Server发现。另外，刚注册的Eureka Client也不能立即被其他服务调用，因为调用方因为各种缓存没有及时的获取到新的注册列表。

## Eureka 的自我保护模式

当一个新的Eureka Server出现时，它尝试从相邻节点获取所有实例注册表信息。如果从Peer节点获取信息时出现问题，Eureka Serve会尝试其他的Peer节点。

如果服务器能够成功获取所有实例，则根据该信息设置应该接收的更新阈值。如果有任何时间，Eureka Serve接收到的续约低于为该值配置的百分比（默认为15分钟内低于85％），则服务器开启自我保护模式，即不再剔除注册列表的信息。

这样做的好处就是，如果是Eureka Server自身的网络问题，导致Eureka Client的续约不上，Eureka Client的注册列表信息不再被删除，也就是Eureka Client还可以被其他服务消费。

## 什么是Hystrix

在分布式系统中，服务与服务之间依赖错综复杂，一种不可避免的情况就是某些服务将会出现失败。

Hystrix是一个库，它提供了服务与服务之间的容错功能，主要体现在延迟容错和容错，从而做到控制分布式系统中的联动故障。

Hystrix通过隔离服务的访问点，阻止联动故障，并提供故障的解决方案，从而提高了这个分布式系统的弹性。

## 使用举例

```java

@Service
@Slf4j
public class HystrixServiceImpl {
 
    @HystrixCommand(
      fallbackMethod = "fallBackTestHystrix",
      threadPoolProperties = {  //10个核心线程池,超过20个的队列外的请求被拒绝; 当一切都是正常的时候，线程池一般仅会有1到2个线程激活来提供服务
          @HystrixProperty(name = "coreSize", value = "10"),
          @HystrixProperty(name = "maxQueueSize", value = "100"),
          @HystrixProperty(name = "queueSizeRejectionThreshold", value = "20")},
      commandProperties = {
          @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500"), //命令执行超时时间
          @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "2"), //若干10s一个窗口内失败三次, 则达到触发熔断的最少请求量
          @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000") //断路30s后尝试执行, 默认为5s
      })
 
    public String fallBackTestHystrix(){
        log.info("走降级策略");
        return "is fallBackTestHystrix";
    }
}
```


## Hystrix解决了什么问题

在复杂的分布式系统中，可能有成百上千个依赖服务，这些服务由于某种故障，比如机房的不可靠性、网络服务商的不可靠性等因素，导致某个服务不可用，如果系统不隔离该不可用的服务，可能会导致整个系统不可用。

例如，对于依赖30个服务的应用程序，每个服务的正常运行时间为99.99％，这是您期望的

> 99.9930 = 99.7％的正常运行时间
10亿次请求中有0.3％= 3,000,000次失败
2小时停机时间/月，即使所有的依赖都有很好的正常运行时间。

实际情况可能比这更糟糕。

如果不设计整个系统的韧性，即使所有依赖关系表现良好，即使0.01％的停机时间对数十个服务中的每一个服务的总体影响等同于每个月停机的潜在时间。

在高并发的情况下，单个服务的延迟，可能导致所有的请求都处于延迟状态，可能在几秒钟就使服务处于负载饱和的状态。

服务的单个点的请求故障，会导致整个服务出现故障，更为糟糕的是该故障服务，会导致其他的服务出现负载饱和，资源耗尽，直到不可用，从而导致这个分布式系统都不可用。这就是“雪崩”。

## Hystrix的设计原则

- 防止单个服务的故障，耗尽整个系统服务的容器（比如tomcat）的线程资源。

- 减少负载并快速失败，而不是排队。

- 在可行的情况下提供回退以保护用户免受故障。

- 使用隔离技术（如隔板，泳道和断路器模式）来限制任何一个依赖的影响。

- 通过近乎实时的指标，监控和警报来优化发现故障的时间。

- 通过配置更改的低延迟传播优化恢复时间，并支持Hystrix大多数方面的动态属性更改，从而允许您使用低延迟反馈循环进行实时操作修改。

- 保护整个依赖客户端执行中的故障，而不仅仅是在网络流量上进行保护降级、限流。

## Hystrix 是怎么实现它的设计目标的？

- 通过HystrixCommand 或者HystrixObservableCommand 将所有的外部系统（或者称为依赖）包装起来，整个包装对象是单独运行在一个线程之中（这是典型的命令模式）。

- 超时请求应该超过你定义的阈值

- 为每个依赖关系维护一个小的线程池（或信号量）; 如果它变满了，那么依赖关系的请求将立即被拒绝，而不是排队等待。

- 统计成功，失败（由客户端抛出的异常），超时和线程拒绝。

- 打开断路器可以在一段时间内停止对特定服务的所有请求，如果服务的错误百分比通过阈值，手动或自动的关闭断路器。

- 当请求被拒绝、连接超时或者断路器打开，直接执行fallback逻辑。

- 近乎实时监控指标和配置变化。

## Hystrix是怎么工作的？

![](https://img-blog.csdnimg.cn/20210127004554465.png)

具体将从以下几个方面进行描述：

**1.构建一个HystrixCommand或者HystrixObservableCommand 对象。**

第一步是构建一个HystrixCommand或HystrixObservableCommand对象来表示你对依赖关系的请求。 其中构造函数需要和请求时的参数一致。

构造HystrixCommand对象，如果依赖关系预期返回单个响应。 可以这样写：

```java
HystrixCommand command = new HystrixCommand(arg1, arg2);
```

同理，可以构建HystrixObservableCommand ：

```java
HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
```

**2.执行Command**

通过使用Hystrix命令对象的以下四种方法之一，可以执行该命令有四种方法（前两种方法仅适用于简单的HystrixCommand对象，并不适用于HystrixObservableCommand）：

- execute()–阻塞，，然后返回从依赖关系接收到的单个响应（或者在发生错误时抛出异常）

- queue()–返回一个可以从依赖关系获得单个响应的future 对象

- observe()–订阅Observable代表依赖关系的响应，并返回一个Observable，该Observable会复制该来源Observable

- toObservable() --返回一个Observable，当您订阅它时，将执行Hystrix命令并发出其响应

```java
K             value   = command.execute();
Future<K>     fValue  = command.queue();
Observable<K> ohValue = command.observe();         
Observable<K> ocValue = command.toObservable();
```

同步调用execute（）调用queue（）.get（）. queue（）依次调用toObservable（）.toBlocking（）.toFuture（）。 这就是说，最终每个HystrixCommand都由一个Observable实现支持，甚至是那些旨在返回单个简单值的命令。

**3.响应是否有缓存？**

如果为该命令启用请求缓存，并且如果缓存中对该请求的响应可用，则此缓存响应将立即以“可观察”的形式返回。

**4.断路器是否打开？**

当您执行该命令时，Hystrix将检查断路器以查看电路是否打开。

如果电路打开（或“跳闸”），则Hystrix将不会执行该命令，但会将流程路由到（8）获取回退。

如果电路关闭，则流程进行到（5）以检查是否有可用于运行命令的容量。

**5.线程池/队列/信号量是否已经满负载？**

如果与命令相关联的线程池和队列（或信号量，如果不在线程中运行）已满，则Hystrix将不会执行该命令，但将立即将流程路由到（8）获取回退。

**6.HystrixObservableCommand.construct() 或者 HystrixCommand.run()**

在这里，Hystrix通过您为此目的编写的方法调用对依赖关系的请求，其中之一是：

- HystrixCommand.run（） - 返回单个响应或者引发异常

- HystrixObservableCommand.construct（） - 返回一个发出响应的Observable或者发送一个onError通知

如果run（）或construct（）方法超出了命令的超时值，则该线程将抛出一个TimeoutException（或者如果命令本身没有在自己的线程中运行，则会产生单独的计时器线程）。 

在这种情况下，Hystrix将响应通过8进行路由。获取Fallback，如果该方法不取消/中断，它会丢弃最终返回值run（）或construct（）方法。

请注意，没有办法强制潜在线程停止工作 - 最好的Hystrix可以在JVM上执行它来抛出一个InterruptedException。 

如果由Hystrix包装的工作不处理InterruptedExceptions，Hystrix线程池中的线程将继续工作，尽管客户端已经收到了TimeoutException。 

这种行为可能使Hystrix线程池饱和，尽管负载“正确地流失”。 大多数Java HTTP客户端库不会解释InterruptedExceptions。 

因此，请确保在HTTP客户端上正确配置连接和读/写超时。

如果该命令没有引发任何异常并返回响应，则Hystrix在执行某些日志记录和度量报告后返回此响应。 

在run（）的情况下，Hystrix返回一个Observable，发出单个响应，然后进行一个onCompleted通知; 在construct（）的情况下，Hystrix返回由construct（）返回的相同的Observable。

**7.计算Circuit 的健康**

Hystrix向断路器报告成功，失败，拒绝和超时，该断路器维护了一系列的计算统计数据组。

它使用这些统计信息来确定电路何时“跳闸”，此时短路任何后续请求直到恢复时间过去，在首次检查某些健康检查之后，它再次关闭电路。

**8.获取Fallback**

当命令执行失败时，Hystrix试图恢复到你的回退：当construct（）或run（）（6.）抛出异常时，当命令由于电路断开而短路时（4.），当 命令的线程池和队列或信号量处于容量（5.），或者当命令超过其超时长度时。

编写Fallback ,它不一依赖于任何的网络依赖，从内存中获取获取通过其他的静态逻辑。如果你非要通过网络去获取Fallback,你可能需要些在获取服务的接口的逻辑上写一个HystrixCommand。

**9.返回成功的响应**

如果 Hystrix command成功，如果Hystrix命令成功，它将以Observable的形式返回对呼叫者的响应或响应。 根据您在上述步骤2中调用命令的方式，此Observable可能会在返回给您之前进行转换：

![](https://img-blog.csdnimg.cn/2021012700452262.png)

- execute（） - 以与.queue（）相同的方式获取Future，然后在此Future上调用get（）来获取Observable发出的单个值

- queue（） - 将Observable转换为BlockingObservable，以便将其转换为Future，然后返回此未来

- observe（） - 立即订阅Observable并启动执行命令的流程; 返回一个Observable，当您订阅它时，重播排放和通知

- toObservable（） - 返回Observable不变; 您必须订阅它才能实际开始导致命令执行的流程

## 断路器（Circuit Breaker）

![](https://img-blog.csdnimg.cn/20210127004451811.png)

下图显示HystrixCommand或HystrixObservableCommand如何与HystrixCircuitBreaker及其逻辑和决策流程进行交互，包括计数器在断路器中的行为。

发生电路开闭的过程如下：

1.假设电路上的音量达到一定阈值（HystrixCommandProperties.circuitBreakerRequestVolumeThreshold（））…

2.并假设错误百分比超过阈值错误百分比（HystrixCommandProperties.circuitBreakerErrorThresholdPercentage（））…

3.然后断路器从CLOSED转换到OPEN。

4.虽然它是开放的，它使所有针对该断路器的请求短路。

5.经过一段时间（HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds（）），下一个单个请求是通过（这是HALF-OPEN状态）。 如果请求失败，断路器将在睡眠窗口持续时间内返回到OPEN状态。 如果请求成功，断路器将转换到CLOSED，逻辑1.重新接管。