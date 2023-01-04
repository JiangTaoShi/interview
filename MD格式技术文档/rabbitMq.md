## RabbitMQ 的使用场景有哪些？

抢购活动，削峰填谷，防止系统崩塌。

延迟信息处理，比如 10 分钟之后给下单未付款的用户发送邮件提醒。

解耦系统，对于新增的功能可以单独写模块扩展，比如用户确认评价之后，新增了给用户返积分的功能，这个时候不用在业务代码里添加新增积分的功能，只需要把新增积分的接口订阅确认评价的消息队列即可，后面再添加任何功能只需要订阅对应的消息队列即可。

## RabbitMQ 有哪些重要的角色？

RabbitMQ 中重要的角色有：生产者、消费者和代理：

- 生产者：消息的创建者，负责创建和推送数据到消息服务器；

- 消费者：消息的接收方，用于处理数据和确认消息；

- 代理：就是 RabbitMQ 本身，用于扮演“快递”的角色，本身不生产消息，只是扮演“快递”的角色。

## RabbitMQ 有哪些重要的组件？

- ConnectionFactory（连接管理器）：应用程序与Rabbit之间建立连接的管理器，程序代码中使用。

- Channel（信道）：消息推送使用的通道。

- Exchange（交换器）：用于接受、分配消息。

- Queue（队列）：用于存储生产者的消息。

- RoutingKey（路由键）：用于把生成者的数据分配到交换器上。

- BindingKey（绑定键）：用于把交换器的消息绑定到队列上。

## RabbitMQ 中 vhost 的作用是什么？

vhost：每个 RabbitMQ 都能创建很多 vhost，我们称之为虚拟主机，每个虚拟主机其实都是 mini 版的RabbitMQ，它拥有自己的队列，交换器和绑定，拥有自己的权限机制。

## RabbitMQ 的消息是怎么发送的？

首先客户端必须连接到 RabbitMQ 服务器才能发布和消费消息，客户端和 rabbit server 之间会创建一个 tcp 连接。

一旦 tcp 打开并通过了认证（认证就是你发送给 rabbit 服务器的用户名和密码），你的客户端和 RabbitMQ 就创建了一条 amqp 信道（channel）。

信道是创建在“真实” tcp 上的虚拟连接，amqp 命令都是通过信道发送出去的，每个信道都会有一个唯一的 id，不论是发布消息，订阅队列都是通过这个信道完成的。

## RabbitMQ 怎么保证消息的稳定性？

提供了事务的功能。 通过将 channel 设置为 confirm（确认）模式。

## RabbitMQ 怎么避免消息丢失？

消息持久化，当然前提是队列必须持久化
RabbitMQ确保持久性消息能从服务器重启中恢复的方式是，将它们写入磁盘上的一个持久化日志文件，当发布一条持久性消息到持久交换器上时，Rabbit会在消息提交到日志文件后才发送响应。

一旦消费者从持久队列中消费了一条持久化消息，RabbitMQ会在持久化日志中把这条消息标记为等待垃圾收集。如果持久化消息在被消费之前RabbitMQ重启，那么Rabbit会自动重建交换器和队列（以及绑定），并重新发布持久化日志文件中的消息到合适的队列。

## 要保证消息持久化成功的条件有哪些？

### 声明队列必须设置持久化 durable 设置为 true.

- 消息推送投递模式必须设置持久化，deliveryMode 设置为 2（持久）。

- 消息已经到达持久化交换器。

- 消息已经到达持久化队列。

- 以上四个条件都满足才能保证消息持久化成功。

## RabbitMQ 持久化有什么缺点？

持久化的缺地就是降低了服务器的吞吐量，因为使用的是磁盘而非内存存储，从而降低了吞吐量。可尽量使用 ssd 硬盘来缓解吞吐量的问题。

## RabbitMQ 有几种广播类型？

- direct（默认方式）：最基础最简单的模式，发送方把消息发送给订阅方，如果有多个订阅者，默认采取轮询的方式进行消息发送。

- headers：与 direct 类似，只是性能很差，此类型几乎用不到。

- fanout：分发模式，把消费分发给所有订阅者。

- topic：匹配订阅模式，使用正则匹配到消息队列，能匹配到的都能接收到。

## RabbitMQ 怎么实现延迟消息队列？

### 延迟队列的实现有两种方式：

- 通过消息过期后进入死信交换器，再由交换器转发到延迟消费队列，实现延迟功能；

- 使用 RabbitMQ-delayed-message-exchange 插件实现延迟功能。

## RabbitMQ 集群有什么用？

### 集群主要有以下两个用途：

- 高可用：某个服务器出现问题，整个 RabbitMQ 还可以继续使用；

- 高容量：集群可以承载更多的消息量。

## RabbitMQ 节点的类型有哪些？

- 磁盘节点：消息会存储到磁盘。

- 内存节点：消息都存储在内存中，重启服务器消息丢失，性能高于磁盘类型。
  
## RabbitMQ 集群搭建需要注意哪些问题？

- 各节点之间使用“–link”连接，此属性不能忽略。

- 各节点使用的 erlang cookie 值必须相同，此值相当于“秘钥”的功能，用于各节点的认证。

- 整个集群中必须包含一个磁盘节点。

## RabbitMQ 每个节点是其他节点的完整拷贝吗？为什么？

### 不是，原因有以下两个：

- 存储空间的考虑：如果每个节点都拥有所有队列的完全拷贝，这样新增节点不但没有新增存储空间，反而增加了更多的冗余数据；

- 性能的考虑：如果每条消息都需要完整拷贝到每一个集群节点，那新增节点并没有提升处理消息的能力，最多是保持和单节点相同的性能甚至是更糟。

## RabbitMQ 集群中唯一一个磁盘节点崩溃了会发生什么情况？

如果唯一磁盘的磁盘节点崩溃了，

不能进行以下操作：

1.不能创建队列

2.不能创建交换器

3.不能创建绑定

4.不能添加用户

5.不能更改权限

6.不能添加和删除集群节点

唯一磁盘节点崩溃了，集群是可以保持运行的，但你不能更改任何东西。

## RabbitMQ 对集群节点停止顺序有要求吗？

RabbitMQ 对集群的停止的顺序是有要求的，应该先关闭内存节点，最后再关闭磁盘节点。如果顺序恰好相反的话，可能会造成消息的丢失

## 消息怎么路由？

从概念上来说，消息路由必须有三部分：交换器、路由、绑定。生产者把消息发布到交换器上；绑定决定了消息如何从路由器路由到特定的队列；消息最终到达队列，并被消费者接收。

消息发布到交换器时，消息将拥有一个路由键（routing key），在消息创建时设定。

通过队列路由键，可以把队列绑定到交换器上。

消息到达交换器后，RabbitMQ会将消息的路由键与队列的路由键进行匹配（针对不同的交换器有不同的路由规则）。如果能够匹配到队列，则消息会投递到相应队列中；如果不能匹配到任何队列，消息将进入 “黑洞”。

常用的交换器主要分为一下三种：

- direct：如果路由键完全匹配，消息就被投递到相应的队列

- fanout：如果交换器收到消息，将会广播到所有绑定的队列上

- topic：可以使来自不同源头的消息能够到达同一个队列。使用topic交换器时，可以使用通配符。
比如：“*” 匹配特定位置的任意文本， “.” 把路由键分为了几部分，“#” 匹配所有规则等。

特别注意：发往topic交换器的消息不能随意的设置选择键（routing_key），必须是由"."隔开的一系列的标识符组成。

## 如何确保消息正确地发送至RabbitMQ？

RabbitMQ使用发送方确认模式，确保消息正确地发送到RabbitMQ。

**发送方确认模式**：将信道设置成confirm模式（发送方确认模式），则所有在信道上发布的消息都会被指派一个唯一的ID。

一旦消息被投递到目的队列后，或者消息被写入磁盘后（可持久化的消息），信道会发送一个确认给生产者（包含消息唯一ID）。

如果RabbitMQ发生内部错误从而导致消息丢失，会发送一条nack（not acknowledged，未确认）消息。

发送方确认模式是异步的，生产者应用程序在等待确认的同时，可以继续发送消息。当确认消息到达生产者应用程序，生产者应用程序的回调方法就会被触发来处理确认消息。

## 如何确保消息接收方消费了消息？

**接收方消息确认机制**：消费者接收每一条消息后都必须进行确认（消息接收和消息确认是两个不同操作）。

只有消费者确认了消息，RabbitMQ才能安全地把消息从队列中删除。这里并没有用到超时机制，RabbitMQ仅通过Consumer的连接中断来确认是否需要重新发送消息。

也就是说，只要连接不中断，RabbitMQ给了Consumer足够长的时间来处理消息。

下面罗列几种特殊情况：

- 如果消费者接收到消息，在确认之前断开了连接或取消订阅，RabbitMQ会认为消息没有被分发，然后重新分发给下一个订阅的消费者。（可能存在消息重复消费的隐患，需要根据bizId去重）

- 如果消费者接收到消息却没有确认消息，连接也未断开，则RabbitMQ认为该消费者繁忙，将不会给该消费者分发更多的消息。

## 如何避免消息重复投递或重复消费？

在消息生产时，MQ内部针对每条生产者发送的消息生成一个inner-msg-id，作为去重和幂等的依据（消息投递失败并重传），避免重复的消息进入队列；

在消息消费时，要求消息体中必须要有一个bizId（对于同一业务全局唯一，如支付ID、订单ID、帖子ID等）作为去重和幂等的依据，避免同一条消息被重复消费。

这个问题针对业务场景来答分以下几点：

- 1.比如，你拿到这个消息做数据库的insert操作。那就容易了，给这个消息做一个唯一主键，那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。

- 2.再比如，你拿到这个消息做redis的set的操作，那就容易了，不用解决，因为你无论set几次结果都是一样的，set操作本来就算幂等操作。

- 3.如果上面两种情况还不行，上大招。准备一个第三方介质,来做消费记录。以redis为例，给消息分配一个全局id，只要消费过该消息，将<id,message>以K-V形式写入redis。那消费者开始消费前，先去redis中查询有没消费记录即可。

## 如何解决丢数据的问题?

### 1.生产者丢数据

生产者的消息没有投递到MQ中怎么办？从生产者弄丢数据这个角度来看，RabbitMQ提供transaction和confirm模式来确保生产者不丢消息。

transaction机制就是说，发送消息前，开启事物(channel.txSelect())，然后发送消息，如果发送过程中出现什么异常，事物就会回滚(channel.txRollback())，如果发送成功则提交事物(channel.txCommit())。

然而缺点就是吞吐量下降了。因此，按照博主的经验，生产上用confirm模式的居多。一旦channel进入confirm模式，所有在该信道上面发布的消息都将会被指派一个唯一的ID(从1开始)，一旦消息被投递到所有匹配的队列之后，rabbitMQ就会发送一个Ack给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了，如果rabiitMQ没能处理该消息，则会发送一个Nack消息给你，你可以进行重试操作。

### 2.消息队列丢数据

处理消息队列丢数据的情况，一般是开启持久化磁盘的配置。这个持久化配置可以和confirm机制配合使用，你可以在消息持久化磁盘后，再给生产者发送一个Ack信号。这样，如果消息持久化磁盘之前，rabbitMQ阵亡了，那么生产者收不到Ack信号，生产者会自动重发。

那么如何持久化呢，这里顺便说一下吧，其实也很容易，就下面两步

①、将queue的持久化标识durable设置为true,则代表是一个持久的队列

②、发送消息的时候将deliveryMode=2

这样设置以后，rabbitMQ就算挂了，重启后也能恢复数据。在消息还没有持久化到硬盘时，可能服务已经死掉，这种情况可以通过引入mirrored-queue即镜像队列，但也不能保证消息百分百不丢失（整个集群都挂掉）

### 3.消费者丢数据

启用手动确认模式可以解决这个问题

①自动确认模式，消费者挂掉，待ack的消息回归到队列中。消费者抛出异常，消息会不断的被重发，直到处理成功。不会丢失消息，即便服务挂掉，没有处理完成的消息会重回队列，但是异常会让消息不断重试。

②手动确认模式，如果消费者来不及处理就死掉时，没有响应ack时会重复发送一条信息给其他消费者；如果监听程序处理异常了，且未对异常进行捕获，会一直重复接收消息，然后一直抛异常；如果对异常进行了捕获，但是没有在finally里ack，也会一直重复发送消息(重试机制)。

③不确认模式，acknowledge="none" 不使用确认机制，只要消息发送完成会立即在队列移除，无论客户端异常还是断开，只要发送完就移除，不会重发。

## 死信队列和延迟队列的使用?

**死信消息：**

- 消息被拒绝（Basic.Reject或Basic.Nack）并且设置 requeue 参数的值为 false

- 消息过期了

- 队列达到最大的长度

**过期消息：**

在 rabbitmq 中存在2种方可设置消息的过期时间，

第一种通过对队列进行设置，这种设置后，该队列中所有的消息都存在相同的过期时间，

第二种通过对消息本身进行设置，那么每条消息的过期时间都不一样。

如果同时使用这2种方法，那么以过期时间小的那个数值为准。当消息达到过期时间还没有被消费，那么那个消息就成为了一个 死信 消息。

- 队列设置：在队列申明的时候使用 x-message-ttl 参数，单位为 毫秒

- 单个消息设置：是设置消息属性的 expiration 参数的值，单位为 毫秒

**延时队列：**

在rabbitmq中不存在延时队列，但是我们可以通过设置消息的过期时间和死信队列来模拟出延时队列。

消费者监听死信交换器绑定的队列，而不要监听消息发送的队列。

有了以上的基础知识，我们完成以下需求：

需求：用户在系统中创建一个订单，如果超过时间用户没有进行支付，那么自动取消订单。

分析：

1、上面这个情况，我们就适合使用延时队列来实现，那么延时队列如何创建

2、延时队列可以由 过期消息+死信队列 来时间


3、过期消息通过队列中设置 x-message-ttl 参数实现

4、死信队列通过在队列申明时，给队列设置 x-dead-letter-exchange 参数，然后另外申明一个队列绑定x-dead-letter-exchange对应的交换器。

```java
ConnectionFactory factory = new ConnectionFactory(); 
factory.setHost("127.0.0.1"); 
factory.setPort(AMQP.PROTOCOL.PORT); 
factory.setUsername("guest"); 
factory.setPassword("guest"); 
Connection connection = factory.newConnection(); 
Channel channel = connection.createChannel();

// 声明一个接收被删除的消息的交换机和队列 
String EXCHANGE_DEAD_NAME = "exchange.dead"; 
String QUEUE_DEAD_NAME = "queue_dead"; 
channel.exchangeDeclare(EXCHANGE_DEAD_NAME, BuiltinExchangeType.DIRECT); 
channel.queueDeclare(QUEUE_DEAD_NAME, false, false, false, null); 
channel.queueBind(QUEUE_DEAD_NAME, EXCHANGE_DEAD_NAME, "routingkey.dead"); 

String EXCHANGE_NAME = "exchange.fanout"; 
String QUEUE_NAME = "queue_name"; 
channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT); 
Map<String, Object> arguments = new HashMap<String, Object>(); 
// 统一设置队列中的所有消息的过期时间 
arguments.put("x-message-ttl", 30000); 
// 设置超过多少毫秒没有消费者来访问队列，就删除队列的时间 
arguments.put("x-expires", 20000); 
// 设置队列的最新的N条消息，如果超过N条，前面的消息将从队列中移除掉 
arguments.put("x-max-length", 4); 
// 设置队列的内容的最大空间，超过该阈值就删除之前的消息
arguments.put("x-max-length-bytes", 1024); 
// 将删除的消息推送到指定的交换机，一般x-dead-letter-exchange和x-dead-letter-routing-key需要同时设置
arguments.put("x-dead-letter-exchange", "exchange.dead"); 
// 将删除的消息推送到指定的交换机对应的路由键 
arguments.put("x-dead-letter-routing-key", "routingkey.dead"); 
// 设置消息的优先级，优先级大的优先被消费 
arguments.put("x-max-priority", 10); 
channel.queueDeclare(QUEUE_NAME, false, false, false, arguments); 
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ""); 
String message = "Hello RabbitMQ: "; 

for(int i = 1; i <= 5; i++) { 
  // expiration: 设置单条消息的过期时间 
  AMQP.BasicProperties.Builder properties = new AMQP.BasicProperties().builder()
      .priority(i).expiration( i * 1000 + ""); 
  channel.basicPublish(EXCHANGE_NAME, "", properties.build(), (message + i).getBytes("UTF-8")); 
} 
channel.close(); 
connection.close();
```

## 无法被路由的消息去了哪里?

无设置的情况下，无法路由（Routing key错误）的消息会被直接丢弃

**解决方案：**

将mandatory设置为true，并配合ReturnListener，实现消息的回发

声明交换机时，指定备份的交换机

```java
Map<String,Object> arguments = new HashMap<String,Object>();
arguments.put("alternate-exchange","备份交换机名");
```

## 如何保证消息的顺序性

一个队列只有一个消费者的情况下才能保证顺序，否则只能通过全局ID实现（每条消息都一个msgId，关联的消息拥有一个parentMsgId。可以在消费端实现前一条消息未消费，不处理下一条消息；也可以在生产端实现前一条消息未处理完毕，不发布下一条消息）
