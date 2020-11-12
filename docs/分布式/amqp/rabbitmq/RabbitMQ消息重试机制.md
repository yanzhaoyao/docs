# RabbitMQ消息重试机制

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkjcx0rb04j30ot0bo40a.jpg" alt="011o950626a1392e76619e04ed4a967bfc8f" style="zoom:67%;" />

# POM依赖设置

在具体的项目的depandencies中添加

```
<dependency>
	
</dependency>
```



# 参数设置（消费端和生产端）



```xml
<rabbit:connection-factory id="connectionFactory"
                           channel-cache-size="100"
                           host="#{rabbitmq['rabbitmq.host']}"
                           username="#{rabbitmq['rabbitmq.username']}"
                           password="#{rabbitmq['rabbitmq.password']}"
                           port="#{rabbitmq['rabbitmq.port']}"
                           factory-timeout="1000"
                           connection-timeout="2000"/>
```

1. 参数说明

|                    |                                                              |
| ------------------ | ------------------------------------------------------------ |
| 参数名称           | 说明                                                         |
| channel-cache-size | 能够缓存的channel数量，一个线程会占用掉一个channel，为降低brocker压力不宜设置过大 |
| factory-timeout    | 该值大于0时，当channel-cache-size到达设定值，那么会报异常，无法创建channel |
| connection-timeout | 创建到broker的连接超时时间，较老的版本，比如1.4.6，是永远不超时的 |
|                    |                                                              |



# 参数设置（消费端）



1. TTLMessageQueueConfiguration核心参数设置如下：

```xml
<propertyname="retryInitialInterval" value="1"/>
<propertyname="retryMaxInterval" value="20"/>
<property name="retryMaxAttempts"value="2"/>
<property name="retryQueueTTL"value="10000"/>
<property name="retryBeforeToDead"value="30"/>


<!-- 注意：下述参数为性能调优参数，需要各系统根据自身实际情况填写，不能直接复制粘贴 -->
<property name="prefetchCount" value="20"/>
<property name="concurrentConsumers" value="10"/>
<property name="maxConcurrentConsumers" value="20"/>
```

1. 参数说明



|                        |                                                              |
| ---------------------- | ------------------------------------------------------------ |
| retryInitialInterval   | 重试的初始时间间隔，单位毫秒，重试策略：第一次1*2，第二次在基础上再乘以2，直到retryMaxInterval，建议值：1 |
| retryMaxInterval       | 参见retryInitialInterval                                     |
| retryMaxAttempts       | 重试次数，如果配置为1是补充是，2是重试一次，总共处理两次；上述这三个参数是线程级别重试，采用sleep线程实现，所以不宜设置过大，一般用于解决数据库乐观锁之类的问题，如果失败，可以有队列级重试保证； |
| retryQueueTTL          | 消息从retry队列转入交易队列超时时间                          |
| retryBeforeToDead      | 消息进死信队列前的队列级重试次数                             |
| prefetchCount          | 单消费线程从Blocker一次获取的消息条数                        |
| concurrentConsumers    | 消费者线程数                                                 |
| maxConcurrentConsumers | 消费者最大线程数                                             |





`TTLMessageQueueConfiguration.java`

```java
public class TTLMessageQueueConfiguration extends SimpleMessageListenerContainer {
    
    private static final String printPrefix = "@TTLMQ=============> ";
    private final static Logger logger      = LoggerFactory.getLogger(TTLMessageQueueConfiguration.class);
    
    // 可配置字段
    private String  businessName             = "unknow";
    private String  suffix                   = "";
    private Integer retryInitialInterval     = 1;
    private Integer retryMaxInterval         = 2000;
    private Integer retryMaxAttempts         = 1;
    private Integer retryQueueTTL            = 2 * 60 * 1000;
    private Integer retryBeforeToDead        = 15;
    private String  routingKey               = "";
    private String  bindToExchange           = "";
    private String  bindToExchangeRoutingKey = "";
    private boolean registToZk               = false;
    
    // 自动生成字段
    private String normalQueueName;
    private String retryQueueName;
    private String deadQueueName;
    private String normalExchangeName;
    private String retryExchangeName;
    private String deadExchangeName;
    
    private RabbitTemplate normalRabbitTemplate;

    @Override
    protected void initializeProxy(Object delegate) {
        setAdviceChain(getAdvice());
        super.initializeProxy(delegate);
    }

    @Override
    protected void doInitialize() {
        if (normalRabbitTemplate == null) {
            setDefault();
            buildExchangeAndQueue();
            super.doInitialize();
        }
    }
    
    private void checkSuffix() {
        if (suffix != null && !suffix.isEmpty()) {
            if (suffix.charAt(0) != '.') {
                suffix = "." + suffix;
            }
        }
    }
    
    @SneakyThrows
    private void buildExchangeAndQueue() {
        checkSuffix();
        
        normalQueueName = businessName + ".ttl.queue" + suffix;
        retryQueueName  = businessName + ".ttl.retry.queue" + suffix;
        deadQueueName   = businessName + ".ttl.dead.letter.queue" + suffix;
        
        normalExchangeName = businessName + ".ttl.direct.exchange" + suffix;
        retryExchangeName  = businessName + ".ttl.fanout.retry.exchange" + suffix;
        deadExchangeName   = businessName + ".ttl.fanout.dead.letter.exchange" + suffix;
        
        setQueueNames(normalQueueName);
        
        Channel channel = getConnectionFactory().createConnection().createChannel(false);
        
        // Normal Queue And Exchange
        Map<String, Object> normalQueueArgs = new HashMap<String, Object>();
        normalQueueArgs.put("x-dead-letter-exchange", retryExchangeName);
        // 参数说明: exchangeName, exchangeType, durable, auto-delete, arguments
        logger.info(printPrefix + "DECLARE Exchange: {}", normalExchangeName);
        channel.exchangeDeclare(normalExchangeName, "direct", true, false, null);
        // 参数说明: queueName, durable, exclusive, auto-delete, arguments
        logger.info(printPrefix + "DECLARE Queue: {}", normalQueueName);
        channel.queueDeclare(normalQueueName, true, false, false, normalQueueArgs);
        // 参数说明: queueName, exchangeName, routingKey
        if (routingKey != null && routingKey.contains(",")) {
            String[] bindKeys = routingKey.split(",");
            for (int i = 0; i < bindKeys.length; i++) {
                channel.queueBind(normalQueueName, normalExchangeName, bindKeys[i]);
            }
        } else {
            channel.queueBind(normalQueueName, normalExchangeName, routingKey);
        }
        
        // Retry Queue And Exchange
        Map<String, Object> retryQueueArgs = new HashMap<String, Object>();
        retryQueueArgs.put("x-dead-letter-exchange", normalExchangeName);
        retryQueueArgs.put("x-message-ttl", retryQueueTTL);
        logger.info(printPrefix + "DECLARE Exchange: {}", retryExchangeName);
        channel.exchangeDeclare(retryExchangeName, "fanout", true, false, null);
        logger.info(printPrefix + "DECLARE Queue: {}", retryQueueName);
        channel.queueDeclare(retryQueueName, true, false, false, retryQueueArgs);
        channel.queueBind(retryQueueName, retryExchangeName, "");
        
        // Dead Queue And Exchange
        logger.info(printPrefix + "DECLARE Exchange: {}", deadExchangeName);
        channel.exchangeDeclare(deadExchangeName, "fanout", true, false, null);
        logger.info(printPrefix + "DECLARE Queue: {}", deadQueueName);
        channel.queueDeclare(deadQueueName, true, false, false, null);
        channel.queueBind(deadQueueName, deadExchangeName, "");
        
        if (bindToExchange != null && !bindToExchange.isEmpty()) {
            if (bindToExchangeRoutingKey != null && bindToExchangeRoutingKey.contains(",")) {
                String[] bindKeys = bindToExchangeRoutingKey.split(",");
                for (int i = 0; i < bindKeys.length; i++) {
                    channel.exchangeBind(normalExchangeName, bindToExchange, bindKeys[i]);
                }
            } else {
                channel.exchangeBind(normalExchangeName, bindToExchange, bindToExchangeRoutingKey);
            }
        }
        
        channel.close();
        
        normalRabbitTemplate = new RabbitTemplate(getConnectionFactory());
        normalRabbitTemplate.setExchange(normalExchangeName);
        normalRabbitTemplate.setRoutingKey(routingKey);
        
        logger.info("create TTL Queue for {}", businessName);
        if (bindToExchange != null && !bindToExchange.isEmpty()) {
            logger.info("Bind EXCHANGE:[{}] ==BindExchange=> EXCHANGE:[{}] with routing key:{}",
                        bindToExchange,
                        normalExchangeName,
                        bindToExchangeRoutingKey);
        }
        logger.info("Normal EXCHANGE:[{}] ==BindQueue=> QUEUE:[{}] with routing key:{}", normalExchangeName, normalQueueName, routingKey);
        logger.info("Retry  EXCHANGE:[{}] ==BindQueue=> QUEUE:[{}] ", retryExchangeName, retryQueueName);
        logger.info("Dead   EXCHANGE:[{}] ==BindQueue=> QUEUE:[{}] ", deadExchangeName, deadQueueName);
        
        if (registToZk) {
            TTLMessageQueueRegister.Instance().regist(deadQueueName);
        }
    }
    
    private MessageRecoverer getMessageRecover() {
        RabbitTemplate deadRabbitTemplate = new RabbitTemplate(getConnectionFactory());
        deadRabbitTemplate.setExchange(deadExchangeName);
        
        RabbitTemplate retryTemplate = new RabbitTemplate(getConnectionFactory());
        retryTemplate.setExchange(retryExchangeName);
        
        MessageRecover messageRecover = new MessageRecover();
        messageRecover.setRetryLimit(retryBeforeToDead);
        messageRecover.setDeadLetterTemplate(deadRabbitTemplate);
        messageRecover.setRetryRabbitTemplate(retryTemplate);
        return messageRecover;
    }
    
    private RetryTemplate getRetryTemplate() {
        if (retryMaxAttempts <= 0) {
            retryMaxAttempts = 1;
        }
        
        RetryTemplate retryTemplateOnce = new RetryTemplate();
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(retryInitialInterval);
        backOffPolicy.setMaxInterval(retryMaxInterval);
        retryTemplateOnce.setBackOffPolicy(backOffPolicy);
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(retryMaxAttempts);
        retryTemplateOnce.setRetryPolicy(retryPolicy);
        return retryTemplateOnce;
    }
    
    private Advice[] getAdvice() {
        StatelessRetryOperationsInterceptorFactoryBean retryInterceptor = new StatelessRetryOperationsInterceptorFactoryBean();
        retryInterceptor.setMessageRecoverer(getMessageRecover());
        retryInterceptor.setRetryOperations(getRetryTemplate());
        return new Advice[]{retryInterceptor.getObject()};
    }
    
    private void setDefault() {
        setDefaultRequeueRejected(false);
        setAcknowledgeMode(AcknowledgeMode.AUTO);
    }
    
    public String getDeadQueueName() {
        return deadQueueName;
    }
    
    /* getter setter */
    public String getBusinessName() {
        return businessName;
    }
    
    public void setBusinessName(String businessName) {
        this.businessName = businessName;
    }
    
    public Integer getRetryQueueTTL() {
        return retryQueueTTL;
    }
    
    public void setRetryQueueTTL(Integer retryQueueTTL) {
        this.retryQueueTTL = retryQueueTTL;
    }
    
    public Integer getRetryInitialInterval() {
        return retryInitialInterval;
    }
    
    public void setRetryInitialInterval(Integer retryInitialInterval) {
        this.retryInitialInterval = retryInitialInterval;
    }
    
    public Integer getRetryMaxInterval() {
        return retryMaxInterval;
    }
    
    public void setRetryMaxInterval(Integer retryMaxInterval) {
        this.retryMaxInterval = retryMaxInterval;
    }
    
    public Integer getRetryMaxAttempts() {
        return retryMaxAttempts;
    }
    
    public void setRetryMaxAttempts(Integer retryMaxAttempts) {
        this.retryMaxAttempts = retryMaxAttempts;
    }
    
    public Integer getRetryBeforeToDead() {
        return retryBeforeToDead;
    }
    
    public void setRetryBeforeToDead(Integer retryBeforeToDead) {
        this.retryBeforeToDead = retryBeforeToDead;
    }
    
    public String getRoutingKey() {
        return routingKey;
    }
    
    public void setRoutingKey(String routingKey) {
        this.routingKey = routingKey;
    }
    
    public RabbitTemplate getRabbitTemplate() {
        return normalRabbitTemplate;
    }
    
    public String getBindToExchange() {
        return bindToExchange;
    }
    
    public void setBindToExchange(String bindToExchange) {
        this.bindToExchange = bindToExchange;
    }
    
    public String getBindToExchangeRoutingKey() {
        return bindToExchangeRoutingKey;
    }
    
    public void setBindToExchangeRoutingKey(String bindToExchangeRoutingKey) {
        this.bindToExchangeRoutingKey = bindToExchangeRoutingKey;
    }
    
    public String getSuffix() {
        return suffix;
    }
    
    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }

    public boolean isRegistToZk() {
        return registToZk;
    }

    public void setRegistToZk(boolean registToZk) {
        this.registToZk = registToZk;
    }
}

```

`MessageRecover.java`

```java
public class MessageRecover implements MessageRecoverer {
    private static final Logger logger = LoggerFactory.getLogger(MessageRecover.class);
    
    private static final String HEADER_X_DEATH = "x-death";
    
    private static final String HEADER_CUSTOM_DEATH = "custom-death-count";
    
    private RabbitTemplate deadLetterTemplate;
    
    private RabbitTemplate retryRabbitTemplate;
    
    private Integer retryLimit;
    
    // x-death头只保留最新2条数据
    private void truncateDeathHeader(Map<String, Object> headers) {
        Object xDeath = headers.get(HEADER_X_DEATH);
        if (xDeath != null) {
            List deathList = (List) xDeath;
            if (deathList.size() > 2) {
                headers.put(HEADER_X_DEATH, deathList.subList(0, 2));
            }
        }
    }
    
    @Override
    public void recover(Message message, Throwable cause) {
        
        MessageProperties messageProperties = message.getMessageProperties();
        
        Map<String, Object> headers = messageProperties.getHeaders();
        
        String messageBody = new String(message.getBody());
        
        // 死亡次数计数
        Object xDeathCount = headers.get(HEADER_CUSTOM_DEATH);
        
        // 移除x-death头
        truncateDeathHeader(headers);
        
        if (cause != null) {
            if (cause.getCause() != null) {
                logger.error("cause by:", cause.getCause());
            } else {
                logger.error("cause by:", cause);
            }
        }
        
        // 初次死亡计数
        if (xDeathCount == null) {
            logger.info("{}: add custom-death-count to message: {}", this, messageBody);
            headers.put(HEADER_CUSTOM_DEATH, Integer.valueOf(0));
            retryRabbitTemplate.send(messageProperties.getReceivedRoutingKey(), message);
            return;
        }
        
        // deathCount类型错误
        if (!(xDeathCount instanceof Integer)) {
            logger.error("{}:custom-death-count type is not Integer,send to DLE for messageBody:{}, with properties:{}",
                         this, messageBody, messageProperties);
            
            deadLetterTemplate.send(messageProperties.getReceivedRoutingKey(), message);
            return;
        }
        
        // 重试计数增加
        Integer deathCount = (Integer) xDeathCount;
        headers.put(HEADER_CUSTOM_DEATH, deathCount + 1);
        
        if (deathCount.intValue() <= retryLimit) {
            logger.info("{}:retry for messageBody:{}, count: {} with properties:{}",
                        this, messageBody, deathCount, messageProperties);
            retryRabbitTemplate.send(messageProperties.getReceivedRoutingKey(), message);
            return;
        }
        
        logger.error("{}: retry times more than {} ,send to DLE for messageBody:{}, with properties:{}",
                     this, retryLimit, messageBody, messageProperties);
        
        deadLetterTemplate.send(messageProperties.getReceivedRoutingKey(), message);
    }
    
    public void setDeadLetterTemplate(RabbitTemplate deadLetterTemplate) {
        this.deadLetterTemplate = deadLetterTemplate;
    }
    
    public void setRetryLimit(Integer retryLimit) {
        this.retryLimit = retryLimit;
    }
    
    public void setRetryRabbitTemplate(RabbitTemplate template) {
        this.retryRabbitTemplate = template;
    }
    
}
```



参考文献

[RabbitMQ / Sneakers 消息重试机制及源码简析](https://ruby-china.org/topics/34022#)

