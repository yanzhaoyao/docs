# Kafka的简介

## 什么是Kafka

Kafka是一款分布式消息发布和订阅系统，具有高性能、高吞吐量的特点而被广泛应用与大数据传输场景。它是由LinkedIn公司开发，使用Scala语言编写，之后成为Apache基金会的一个顶级项目。kafka提供了类似JMS的特性，但是在设计和实现上是完全不同的，而且他也不是JMS规范的实现。 

## kafka产生的背景

kafka作为一个消息系统，早起设计的目的是用作LinkedIn的活动流（Activity Stream）和运营数据处理管道（Pipeline）。活动流数据是所有的网站对用户的使用情况做分析的时候要用到的最常规的部分,活动数据包括页面的访问量（PV）、被查看内容方面的信息以及搜索内容。这种数据通常的处理方式是先把各种活动以日志的形式写入某种文件，然后周期性的对这些文件进行统计分析。运营数据指的是服务器的性能数据（CPU、IO使用率、请求时间、服务日志等）。 

## Kafka的应用场景

Kafka的应用场景由于kafka具有更好的吞吐量、内置分区、冗余及容错性的优点(kafka每秒可以处理几十万消息)，让kafka成为了一个很好的大规模消息处理应用的解决方案。所以在企业级应用上，主要会应用于如下几个方面 

- 行为跟踪：kafka可以用于跟踪用户浏览页面、搜索及其他行为。通过发布-订阅模式实时记录到对应的topic中，通过后端大数据平台接入处理分析，并做更进一步的实时处理和监控 
- 日志收集：日志收集方面，有很多比较优秀的产品，比如Apache Flume，很多公司使用kafka代理日志聚合。日志聚合表示从服务器上收集日志文件，然后放到一个集中的平台（文件服务器）进行处理。在实际应用开发中，我们应用程序的log都会输出到本地的磁盘上，排查问题的话通过linux命令来搞定，如果应用程序组成了负载均衡集群，并且集群的机器有几十台以上，那么想通过日志快速定位到问题，就是很麻烦的事情了。所以一般都会做一个日志统一收集平台管理log日志用来快速查询重要应用的问题。所以很多公司的套路都是把应用日志集中到kafka上，然后分别导入到es和hdfs上，用来做实时检索分析和离线统计数据备份等。而另一方面，kafka本身又提供了很好的api来集成日志并且做日志收集
- ![img93](http://ww3.sinaimg.cn/large/006tNc79gy1g61ln6hw2wj30m90fxaat.jpg)

# Kafka本身的架构

一个典型的kafka集群包含若干Producer（可以是应用节点产生的消息，也可以是通过Flume收集日志产生的事件），若干个Broker（kafka支持水平扩展）、若干个Consumer Group，以及一个zookeeper集群。kafka通过zookeeper管理集群配置及服务协同。Producer使用push模式将消息发布到broker，consumer通过监听使用pull模式从broker订阅并消费消息。

多个broker协同工作，producer和consumer部署在各个业务逻辑中。三者通过zookeeper管理协调请求和转发。这样就组成了一个高性能的分布式消息发布和订阅系统。

图上有一个细节是和其他mq中间件不同的点，producer 发送消息到broker的过程是push，而consumer从broker消费消息的过程是pull，主动去拉数据。而不是broker把数据主动发送给consumer 

![img97](http://ww2.sinaimg.cn/large/006tNc79ly1g61low4qtij30ln0bdq3y.jpg)

# kafka的安装部署

## 下载安装包

https://www.apache.org/dyn/closer.cgi?path=/kafka/1.1.0/kafka_2.11-1.1.0.tgz

## 安装过程

`tar -zxvf`解压安装包 

## kafka目录介绍

1. `/bin` 操作kafka的可执行脚本 

2. `/config` 配置文件 

3. `/libs` 依赖库目录 
4. `/logs` 日志数据目录 

## 启动/停止kafka

1. 需要先启动zookeeper，如果没有搭建zookeeper环境，可以直接运行kafka内嵌的zookeeper 

   启动命令： `bin/zookeeper-server-start.sh config/zookeeper.properties &` 

2. 启动kafka，进入kafka目录，运行 `bin/kafka-server-start.sh ｛-daemon 后台启动｝ config/server.properties &` 

3. 停止kafka，进入kafka目录，运行`bin/kafka-server-stop.sh config/server.properties` 

## Kafka的基本操作

### 创建topic

```shell
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

Replication-factor 表示该topic需要在不同的broker中保存几份，这里设置成1，表示在两个broker中保存两份，Partitions 分区数 

### 查看topic

```shell
./kafka-topics.sh --list --zookeeper localhost:2181
```

### 查看topic属性

```shell
./kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
```

### 消费消息

```shell
./kafka-console-consumer.sh –bootstrap-server localhost:9092 --topic test --from-beginning
```

### 发送消息

```shell
./kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

# 安装集群环境

## 修改server.properties配置

1. 修改`server.properties` `broker.id=0 / 1` 

2. 修改`server.properties` 修改成本机IP 

   `advertised.listeners=PLAINTEXT://192.168.11.153:9092` 

   当Kafka broker启动时，它会在ZK上注册自己的IP和端口号，客户端就通过这个IP和端口号来连接 

   Kafka JAVA API

## 配置信息分析

### 发送端的可选配置信息分析

`acks`：acks配置表示producer发送消息到broker上以后的确认值。有三个可选项 

- 0：表示producer不需要等待broker的消息确认。这个选项时延最小但同时风险最大（因为当server宕机时，数据将会丢失）。 
-  1：表示producer只需要获得kafka集群中的leader节点确认即可，这个选择时延较小同时确保了leader节点确认接收成功。 
- all(-1)：需要ISR中所有的Replica给予接收确认，速度最慢，安全性最高，但是由于ISR可能会缩小到仅包含一个Replica，所以设置参数为all并不能一定避免数据丢失， 

`batch.size`：生产者发送多个消息到broker上的同一个分区时，为了减少网络请求带来的性能开销，通过批量的方式来提交消息，可以通过这个参数来控制批量提交的字节数大小，默认大小是16384byte,也就是16kb，意味着当一批消息大小达到指定的batch.size的时候会统一发送 

`linger.ms`：Producer默认会把两次发送时间间隔内收集到的所有Requests进行一次聚合然后再发送，以此提高吞吐量，而linger.ms就是为每次发送到broker的请求增加一些delay，以此来聚合更多的Message请求。 这个有点想TCP里面的Nagle算法，在TCP协议的传输中，为了减少大量小数据包的发送，采用了Nagle算法，也就是基于小包的等-停协议。 

- batch.size和linger.ms这两个参数是kafka性能优化的关键参数，很多同学会发现batch.size和linger.ms这两者的作用是一样的，如果两个都配置了，那么怎么工作的呢？实际上，当二者都配置的时候，只要满足其中一个要求，就会发送请求到broker上

`max.request.size`：设置请求的数据的最大字节数，为了防止发生较大的数据包影响到吞吐量，默认值为1MB。 

### 消费端的可选配置分析

`group.id`：consumer group是kafka提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例(consumer instance)，它们共享一个公共的ID，即group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个consumer来消费.如下图所示，分别有三个消费者，属于两个不同的group，那么对于firstTopic这个topic来说，这两个组的消费者都能同时消费这个topic中的消息，那么此时这个firstTopic就类似于ActiveMQ中的topic概念。如右图所示，如果3个消费者都属于同一个group，那么此时firstTopic就是一个Queue的概念 

![img115](http://ww3.sinaimg.cn/large/006tNc79ly1g61m3ouk2qj30hz0g1q44.jpg)

`enable.auto.commit`：消费者消费消息以后自动提交，只有当消息提交以后，该消息才不会被再次接收到，还可以配合auto.commit.interval.ms控制自动提交的频率。 

当然，我们也可以通过consumer.commitSync()的方式实现手动提交 

`auto.offset.reset`：这个参数是针对新的groupid中的消费者而言的，当有新groupid的消费者来消费指定的topic时，对于该参数的配置，会有不同的语义 

- auto.offset.reset=latest情况下，新的消费者将会从其他消费者最后消费的offset处开始消费Topic下的消息 

- auto.offset.reset= earliest情况下，新的消费者会从该topic最早的消息开始消费 

- auto.offset.reset=none情况下，新的消费者加入以后，由于之前不存在offset，则会直接抛出异常。 

`max.poll.records`：此设置限制每次调用poll返回的消息数，这样可以更容易的预测每次poll间隔要处理的最大值。通过调整此值，可以减少poll间隔 

这个参数是针对新的groupid中的消费者而言的，当有新groupid的消费者来消费指定的topic时，对于该参数的配置，会有不同的语义 