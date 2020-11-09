# RabbitMQ面试热点

[![黑马程序员](https://pic2.zhimg.com/v2-771faa18788d6b61f7c1b526d4b8a6f7_xs.jpg?source=172ae18b)](https://www.zhihu.com/org/hei-ma-cheng-xu-yuan)

[黑马程序员](https://www.zhihu.com/org/hei-ma-cheng-xu-yuan)[](https://zhuanlan.zhihu.com/p/265669117)

已认证的官方帐号

## AMQP工作模型

![image-20201107211025494](https://tva1.sinaimg.cn/large/0081Kckwgy1gkgx0en5l1j31hi0haaud.jpg)

### **消息发送的几种模式**

```text
4种交换器类型
    Direct Exchange: 直连交换器
    小结：直连交换器发送消息会根据路由和交换机的绑定关系发送到队列
         
         如果交换器名称为"",将使用默认交换器
         默认交换器不会绑定任何队列，mq会直接把route_key当做queue名称去查找

    Fanout Exchange: 分发交换器（扇出交换器）
    小结: 分发交换器发送消息会分发至所有和其有绑定的队列中，这样消息会被多个消费者         处理

    Topic Exchange: 主题交换器
    小结：主题交换器可以让每个队列只接收它关注的信息

    Headers Exchange: 头信息交换器（了解）
        头信息交换器可以实现更为复杂的匹配但性能不好，
        不推荐使用，了解即可
```



```text
5种工作模式
    简单模式：
        消息只有一个消费者
        使用默认交换器即可(direct)
    工作队列模式：
        消息有多个消费者，消息只可以被消费一次
        使用默认交换器即可(direct)
    发布订阅模式:
        消息有多个消费者，而且消息会被多个消费者同时消费
        使用分发交换器即可(fanout)
    路由模式:
        根据路由的key，将消息发送到指定的队列
        使用直接交换器即可(direct)
    通配符（主题）模式：
        根据路由的key,进行通配符匹配,发送到指定的队列(topic)
        使用主题交换器即可
```

### **在项目中MQ的应用**

```text
解耦场景:
    1. 商品搜索功能 数据库及索引库的一致性
    2. 商品详情静态化，静态页面的生成
延迟队列:
    3. 利用rabbitmq的死信队列功能实现延时处理
异步采集:
    4. 监控数据的采集可以使用rabbitmq异步采集
```

### **如何保证消费的可靠性传输？**

分析: 消息队列增加了系统架构的复杂性，中间的每一个环节都要保证 99.999%的可用,设想下如果公司中队列的消息丢失了，重复消费了，大量消息堆积造成的问题 都可能带来公司大量财产的损失，所以在面试时可靠性的问题是面试官特别爱问的点,我们的MQ是如何保障消息的可靠性呢?主要从三个角度来分析：

- 生产者发消息的可靠性
- 消息队列数据的可靠性
- 消费者消费数据的可靠性

### **01生产者发消息的可靠性**

从生产者弄丢数据这个角度来看，RabbitMQ提供transaction和confirm模式来确保生产者不丢消息。

**transaction事务机制（了解）**

```text
transaction机制就是说，发送消息前，开启事务（channel.txSelect()）,然后发送消息，如果发送过程中出现什么异常，事务就会回滚（channel.txRollback()）,如果发送成功则提交事务（channel.txCommit()）。然而，这种方式有个缺点：吞吐量下降。
```

**confirm确认机制**

```text
一旦channel进入confirm模式，所有在该信道上发布的消息都将会被指派一个唯一的ID（从1开始），一旦消息被投递到所有匹配的队列之后，rabbitMQ就会发送一个ACK给生产者（包含消息的唯一ID），这就使得生产者知道消息已经正确到达目的队列了。如果rabbitMQ没能处理该消息，则会发送一个Nack消息给你，你可以进行重试操作。处理Ack和Nack的代码如下所示
```

更改发送者配置

```yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    publisher-confirm-type: correlated  # 开启确认机制回调 必须配置这个才会确认回调
    publisher-returns: true # 开启return机制回调
rabbit:
  queues:
    red: red_queue
    green: green_queue
    yellow: yellow_queue
  exchange: my_topic_exchange
```

更改发送者测试代码

```java
@SpringBootTest
class RabbitmqProducerApplicationTests {
    @Autowired
    RabbitTemplate rabbitTemplate;
    @Test
    void sendMsg(){
        //发送消息确认机制监听
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             * @param correlationData 回调数据
             * @param ack  true: 确认  false: 未确认
             * @param cause 原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                System.out.println("回调数据==>" + correlationData);
                System.out.println("是否确认==>" + ack);
                System.out.println("原因==>" + cause);
            }
        });
        //消费者在消息没有被路由到合适队列情况下会被return监听，而不会自动删除
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                System.out.println("返还消息==>"+message);
                System.out.println("回复编码==>"+replyCode);
                System.out.println("回复消息==>"+replyText);
                System.out.println("交换器信息==>"+exchange);
                System.out.println("路由key==>"+routingKey);
            }
        });
//        rabbitTemplate.convertAndSend("","xx.red.green.xxx","wahaha");
        rabbitTemplate.convertAndSend("my_topic_exchange","xx.red.xx","111");
    }
}
```

### **02消息队列数据的可靠性**

处理消息队列丢数据的情况，一般是开启持久化磁盘的配置。

1. 将queue的持久化标识durable设置为true,则代表是一个持久的队列
2. 发送消息的时候将deliveryMode=2

这样设置以后，即使rabbitMQ挂了，重启后也能恢复数据

我们可以查看下,之前配置中创建队列的源码,如果你什么都不设置，实际上默认值都是持久化的。

```java
    /**
     * 默认队列是持久化的，非自动删除的
     * The queue is durable, non-exclusive and non auto-delete.
     *
     * @param name the name of the queue.
     */
    public Queue(String name) {
        this(name, true, false, false);
    }
```

默认的消息也是持久化的:



### **03消费者消费数据的可靠性**

消费者丢数据一般是因为采用了自动确认消息模式。这种模式下，消费者会自动确认收到信息。这时rabbitMQ会立即将消息删除，这种情况下，如果消费者出现异常而未能处理消息，就会丢失该消息。至于解决方案，采用手动确认消息即可。

```text
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        # acknowledge-mode: manual #手动确认
        acknowledge-mode: auto #自动确认    manual #手动确认
```

不过springboot的项目中,在spring的层面使用了aop实现了消息处理失败的自动重试功能

```java
// 在监听消息的方法中 加入抛异常的逻辑

if(null!=msg || "1".equals(msg)){
     throw new RuntimeException("出错了");
}
```

发送消息时: 指定消息内容为1





因为自动重试功能，所以监听方法出现问题了 会不断的重试，并且在消息队列中该消息属于未消费的状态，而不是未确认的状态。这是spring帮我们提供的消息自动补偿机制， 不过持续的重试对系统带来的压力非常大，我们可以对重试的相关参数进行设置来改善。

### **消息重试机制(自动补偿)及幂等性**

底层使用Aop拦截，如果程序(消费者)没有抛出异常，自动提交事务 如果Aop使用异常通知拦截获取到异常后，自动实现补偿机制

### **01重试机制的设置**

```yml
 RabbitMQ自动补偿机制触发:(多用于调用第三方接口)
    1. 当我们的消费者在处理我们的消息的时候,程序抛出异常情况下触发自动补偿(默认无限次数重试)
    2. 应该对我们的消息重试设置间隔重试时间,比如消费失败最多只能重试5次,间隔3秒(防止重复消费,幂等问题)
    3. 如果重试后还未消费默认丢弃，如果配置了死信队列，会发送到死信队列中
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual #手动确认
        # acknowledge-mode: auto #自动确认    manual #手动确认
        # 重试策略相关配置
        retry:
          enabled: true # 是否开启重试功能
          max-attempts: 5 # 最大重试次数
          # 时间策略乘数因子   0  1  2  4  8
          # 时间策略乘数因子
          multiplier: 2.0
          initial-interval: 1000ms # 第一次调用后的等待时间
          max-interval: 20000ms # 最大等待的时间值
        default-requeue-rejected: false # 重试后还未消费 是否丢弃 发送到死信对垒
rabbit:
  queues:
    red: red_queue
    green: green_queue
    yellow: yellow_queue
```

更改消费者代码:

```java
public void greenMsg(String msg){
    System.out.println("green队列收到消息==>" + msg + "当前时间:"+ LocalDateTime.now());
    if(null!=msg || "1".equals(msg)){
        throw new RuntimeException("出错了");
    }
}
```

测试结果



注意: 虽然重试机制可以对称对消息消费方法进行重试，不过重试结束后仍未消费的消息还可能造成消息丢失，这里可以通过配置死信队列来存放暂时无法消费的消息，或者过期失效未处理的消息。

### **02死信队列的设置**

用于存放过期未消费的或重试后还未消费的消息:

```java
    /**
     * 死信队列交换器
     * @return
     */
    @Bean
    public Exchange deadExchange(){
        return new DirectExchange("dead_exchange");
    }
    /**
     * 死信队列
     * @return
     */
    @Bean
    public Queue deadQueue(){
        return new Queue("dead_queue");
    }
    /**
     * 将死信交换器 与 死信队列绑定
     * @return
     */
    @Bean
    public Binding deadBindings(){
        return new Binding("dead_queue", Binding.DestinationType.QUEUE,"dead_exchange","dead",null);
    }
```

修改green队列

```java
/**
     * 为green队列设置死信队列的交换器和路由
     *
     * 这样 重试失败的消息 或 失效的消息 会被发送到对应死信队列中
     *
     * @return
     */
    @Bean
    public Queue greenQueue(){
        Map<String,Object> args=new HashMap<>();
        // 设置该Queue的死信的信箱
        args.put("x-dead-letter-exchange", "dead_exchange");
        // 设置死信routingKey
        args.put("x-dead-letter-routing-key", "dead");
        return new Queue(GREEN_QUEUE,true,false,false,args);
    }
```

变更后再次发消息到green中



green队列的消费者接收到消息后 处理报错 重试后仍然未消费



该消息被转发到死信队列 dead_queue中





### **03幂等性的处理:**

消息重试可能造成消费方法的多次调用，所以在消费方法中一定要处理消息的重复消费(幂等性)

**(1). 使用全局MessageID判断消费方是否消费**

在消息生产时，我们可以生成一个全局的消息ID

**(2).使用业务ID+逻辑保证唯一**

在消息消费时，要求消息体中必须要有一个bizId（对于同一业务全局唯一，如支付ID、订单ID、帖子ID等）作为去重的依据，避免同一条消息被重复消费。

演示伪代码:

发送者测试方法:

```java
@Test
    void sendMsg2(){
        // 自己构建消息
        Message message = MessageBuilder.withBody("我是消息内容".getBytes()).build();
        // 设置消息的全局ID
        message.getMessageProperties().setMessageId(UUID.randomUUID().toString()); // 设置自定义的消息ID
        // 发送消息
        rabbitTemplate.send("my_topic_exchange","xx.yellow.xx",message);
    }
```

消费者代码:

```java
Map cacheMap = new HashMap(); // 用map模拟redis中的key:value结构
    @RabbitListener(
            queues = "yellow_queue"
    )
    public void yellowMsg(Message message){
        // 商品数据   商品ID
        byte[] body = message.getBody();
        // 获取消息ID (mq生成的)
        String messageId = message.getMessageProperties().getMessageId();
        System.out.println("消息ID messageId==>"+messageId);
        String msg = new String(body);
        // 查看记录
        Object o = cacheMap.get(messageId);
        if(o!=null){
            System.out.println("已处理");
        }else {
            System.out.println("yellow队列收到消息==>" + msg);
        }
        // 处理该消息将处理记录保存
        cacheMap.put(messageId,msg);
        // 模拟出现异常
        throw new RuntimeException();
    }
```

像yellow队列发送消息 测试结果





在第一次消费时，将消费记录存入到redis中，后面的消费通过redis判断是否消费 来防止重复消费。

### **Rabbitmq的高可用方案(集群)**

在使用RabbitMQ的过程中，如果只有一个节点，但是一旦单机版宕机，服务不可用，影响比较严重，通过集群就能避免单点故障的问题。 RabbitMQ 集群分为两种 普通集群 和 镜像集群

**普通集群**

```text
以两个节点（rabbit01、rabbit02）为例来进行说明。
rabbit01和rabbit02两个节点仅有相同的元数据，即队列的结构，但消息实体只存在于其中一个节点rabbit01（或者rabbit02）中。 当消息进入rabbit01节点的Queue后，consumer从rabbit02节点消费时，RabbitMQ会临时在rabbit01、rabbit02间进行消息传输，把A中的消息实体取出并经过B发送给consumer。
所以consumer应尽量连接每一个节点，从中取消息，即对于同一个逻辑队列，要在多个节点建立物理Queue；否则无论consumer连rabbit01或rabbit02，出口总在rabbit01，会产生瓶颈。
当rabbit01节点故障后，rabbit02节点无法取到rabbit01节点中还未消费的消息实体。如果做了消息持久化，那么得等rabbit01节点恢复，然后才可被消费；如果没有持久化的话，就会产生消息丢失的现象。
```



**镜像集群**

```text
在普通集群的基础上，把需要的队列做成镜像队列，消息实体会主动在镜像节点间同步，而不是在客户端取数据时临时拉取，也就是说多少节点消息就会备份多少份。该模式带来的副作用也很明显，除了降低系统性能外，如果镜像队列数量过多，加之大量的消息进入，集群内部的网络带宽将会被这种同步通讯大大消耗掉。所以在对可靠性要求较高的场合中适用由于镜像队列之间消息自动同步，且内部有选举master机制，即使master节点宕机也不会影响整个集群的使用，达到去中心化的目的，从而有效的防止消息丢失及服务不可用等问题。
```



详见资源中 镜像集群的搭建流程



**Haproxy + keepalived高可用集群**

非常经典的 mirror 镜像模式，保证 100% 数据不丢失。在实际工作中也是用得最多的，并且实现非常的简单，一般互联网大厂都会构建这种镜像集群模式。

  mirror 镜像队列，目的是为了保证 rabbitMQ 数据的高可靠性解决方案，主要就是实现数据的同步，一般来讲是 2 - 3 个节点实现数据同步。对于 100% 数据可靠性解决方案，一般是采用 3 个节点。

  集群架构如下

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkgxwh3wn1j310q0jedxp.jpg" alt="image-20201107212315602" style="zoom:50%;" />

 mirror 镜像队列

  如上图所示，用 KeepAlived 做了 HA-Proxy 的高可用，然后有 3 个节点的 MQ 服务，消息发送到主节点上，主节点通过 mirror 队列把数据同步到其他的 MQ 节点，这样来实现其高可靠。



作者：HmilyMing
链接：https://www.jianshu.com/p/b7cc32b94d2a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



**如何解决消息队列的延时以及过期失效问题？消息队列满了以后该怎么处理？有几百万消息持续积压几小时，说说怎么解决？**

**问题分析** 你看这问法，其实本质针对的场景，都是说，可能你的消费端出了问题，不消费了；或者消费的速度极其慢。接着就坑爹了，可能你的消息队列集群的磁盘都快写满了，都没人消费，这个时候怎么办？或者是这整个就积压了几个小时，你这个时候怎么办？或者是你积压的时间太长了，导致比如 RabbitMQ 设置了消息过期时间后就没了怎么办？

所以就这事儿，其实线上挺常见的，一般不出，一出就是大 case。一般常见于，举个例子，消费端每次消费之后要写 mysql，结果 mysql 挂了，消费端 hang 那儿了，不动了；或者是消费端出了个什么岔子，导致消费速度极其慢。

关于这个事儿，我们一个一个来梳理吧，先假设一个场景，我们现在消费端出故障了，然后大量消息在 mq 里积压，现在出事故了，慌了。 大量消息在 mq 里积压了几个小时了还没解决W

几千万条数据在 MQ 里积压了七八个小时，从下午 4 点多，积压到了晚上 11 点多。这个是我们真实遇到过的一个场景，确实是线上故障了，这个时候要不然就是修复 consumer 的问题，让它恢复消费速度，然后傻傻的等待几个小时消费完毕。这个肯定不能在面试的时候说吧。

一个消费者一秒是 1000 条，一秒 3 个消费者是 3000 条，一分钟就是 18 万条。所以如果你积压了几百万到上千万的数据，即使消费者恢复了，也需要大概 1 小时的时间才能恢复过来。

一般这个时候，只能临时紧急扩容了，具体操作步骤和思路如下：

先修复 consumer 的问题，确保其恢复消费速度，然后将现有 consumer 都停掉。 新建一个 topic，partition 是原来的 10 倍，临时建立好原先 10 倍的 queue 数量。 然后写一个临时的分发数据的 consumer 程序，这个程序部署上去消费积压的数据，消费之后不做耗时的处理，直接均匀轮询写入临时建立好的 10 倍数量的 queue。 接着临时征用 10 倍的机器来部署 consumer，每一批 consumer 消费一个临时 queue 的数据。这种做法相当于是临时将 queue 资源和 consumer 资源扩大 10 倍，以正常的 10 倍速度来消费数据。 等快速消费完积压数据之后，得恢复原先部署的架构，重新用原先的 consumer 机器来消费消息。

**正常情况**



**紧急扩容**



**mq 中的消息过期失效了** 假设你用的是 RabbitMQ，RabbtiMQ 是可以设置过期时间的，也就是 TTL。如果消息在 queue 中积压超过一定的时间就会被 RabbitMQ 给清理掉，这个数据就没了。那这就是第二个坑了。这就不是说数据会大量积压在 mq 里，而是大量的数据会直接搞丢。

这个情况下，就不是说要增加 consumer 消费积压的消息，因为实际上没啥积压，而是丢了大量的消息。我们可以采取一个方案，就是批量重导，这个我们之前线上也有类似的场景干过。就是大量积压的时候，我们当时就直接丢弃数据了，然后等过了高峰期以后，比如大家一起喝咖啡熬夜到晚上12点以后，用户都睡觉了。这个时候我们就开始写程序，将丢失的那批数据，写个临时程序，一点一点的查出来，然后重新灌入 mq 里面去，把白天丢的数据给他补回来。也只能是这样了。

假设 1 万个订单积压在 mq 里面，没有处理，其中 1000 个订单都丢了，你只能手动写程序把那 1000 个订单给查出来，手动发到 mq 里去再补一次。

```text
消息追踪之rabbitmq_tracing插件
作用: 将发消息 消费消息记录到日志中
https://www.cnblogs.com/Harriet/p/10149144.html
```



**mq 都快写满了** 如果消息积压在 mq 里，你很长时间都没有处理掉，此时导致 mq 都快写满了，咋办？这个还有别的办法吗？没有，谁让你第一个方案执行的太慢了，你临时写程序，接入数据来消费，消费一个丢弃一个，都不要了，快速消费掉所有的消息。然后走第二个方案，到了晚上再补数据吧。

```text
spring.rabbitmq.address 
客户端连接的地址，有多个的时候使用逗号分隔，该地址可以是IP与Port的结合   

spring.rabbitmq.cache.channel.checkout-timeout
当缓存已满时，获取Channel的等待时间，单位为毫秒  

spring.rabbitmq.cache.channel.size  
缓存中保持的Channel数量  

spring.rabbitmq.cache.connection.mode   
连接缓存的模式 CHANNEL

spring.rabbitmq.cache.connection.size   
缓存的连接数   

spring.rabbitmq.connnection-timeout 
连接超时参数单位为毫秒：设置为“0”代表无穷大  

spring.rabbitmq.dynamic 
默认创建一个AmqpAdmin的Bean    true

spring.rabbitmq.host    
RabbitMQ的主机地址   localhost

spring.rabbitmq.listener.acknowledge-mode   
容器的acknowledge模式     

spring.rabbitmq.listener.auto-startup   
启动时自动启动容器   true

spring.rabbitmq.listener.concurrency    
消费者的最小数量     

spring.rabbitmq.listener.default-requeue-rejected   
投递失败时是否重新排队 true

spring.rabbitmq.listener.max-concurrency    
消费者的最大数量     

spring.rabbitmq.listener.prefetch   
在单个请求中处理的消息个数，他应该大于等于事务数量    

spring.rabbitmq.listener.retry.enabled  
不论是不是重试的发布  false

spring.rabbitmq.listener.retry.initial-interval 
第一次与第二次投递尝试的时间间隔    1000

spring.rabbitmq.listener.retry.max-attempts 
尝试投递消息的最大数量 3

spring.rabbitmq.listener.retry.max-interval 
两次尝试的最大时间间隔 10000

spring.rabbitmq.listener.retry.multiplier   
上一次尝试时间间隔的乘数    1.0

spring.rabbitmq.listener.retry.stateless    
不论重试是有状态的还是无状态的 true

spring.rabbitmq.listener.transaction-size   
在一个事务中处理的消息数量。为了获得最佳效果，该值应设置为小于等于每个请求中处理的消息个数，即spring.rabbitmq.listener.prefetch的值   

spring.rabbitmq.password    
登录到RabbitMQ的密码   

spring.rabbitmq.port    
RabbitMQ的端口号    5672

spring.rabbitmq.publisher-confirms  
开启Publisher Confirm机制   false

spring.rabbitmq.publisher-returns   
开启publisher Return机制    false

spring.rabbitmq.requested-heartbeat 
请求心跳超时时间，单位为秒    

spring.rabbitmq.ssl.enabled 
启用SSL支持 false

spring.rabbitmq.ssl.key-store   
保存SSL证书的地址   

spring.rabbitmq.ssl.key-store-password  
访问SSL证书的地址使用的密码  

spring.rabbitmq.ssl.trust-store 
SSL的可信地址     

spring.rabbitmq.ssl.trust-store-password    
访问SSL的可信地址的密码    

spring.rabbitmq.ssl.algorithm   
SSL算法，默认使用Rabbit的客户端算法库  

spring.rabbitmq.template.mandatory  
启用强制信息  false

spring.rabbitmq.template.receive-timeout    
receive()方法的超时时间    0

spring.rabbitmq.template.reply-timeout  
sendAndReceive()方法的超时时间 5000

spring.rabbitmq.template.retry.enabled  
设置为true的时候RabbitTemplate能够实现重试  false

spring.rabbitmq.template.retry.initial-interval 
第一次与第二次发布消息的时间间隔    1000

spring.rabbitmq.template.retry.max-attempts 
尝试发布消息的最大数量 3

spring.rabbitmq.template.retry.max-interval 
尝试发布消息的最大时间间隔   10000

spring.rabbitmq.template.retry.multiplier   
上一次尝试时间间隔的乘数    1.0

spring.rabbitmq.username    
登录到RabbitMQ的用户名  

spring.rabbitmq.virtual-host    
连接到RabbitMQ的虚拟主机     
```





发布于 10-14

[消息队列](https://www.zhihu.com/topic/19708788)

[RabbitMQ](https://www.zhihu.com/topic/19629661)

[topic](https://www.zhihu.com/topic/20668432)