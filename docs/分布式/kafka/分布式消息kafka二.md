# 关于Topic和Partition

## Topic

在kafka中，topic是一个存储消息的逻辑概念，可以认为是一个消息集合。每条消息发送到kafka集群的消息都有一个类别。物理上来说，不同的topic的消息是分开存储的，

每个topic可以有多个生产者向它发送消息，也可以有多个消费者去消费其中的消息。

![img80](http://ww2.sinaimg.cn/large/006tNc79gy1g631giiugmj30ip06nq3a.jpg)

## Partition

每个topic可以划分多个分区（每个Topic至少有一个分区），同一topic下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个offset（称之为偏移量），它是消息在此分区中的唯一编号，kafka通过offset保证消息在分区内的顺序，offset的顺序不跨分区，即kafka只保证在同一个分区内的消息是有序的。

下图中，对于名字为test的topic，做了3个分区，分别是p0、p1、p2.

- 每一条消息发送到broker时，会根据partition的规则选择存储到哪一个partition。如果partition规则设置合理，那么所有的消息会均匀的分布在不同的partition中，这样就有点类似数据库的分库分表的概念，把数据做了分片处理。

![img95](http://ww2.sinaimg.cn/large/006tNc79gy1g631jg1cs6j30jd07r0tf.jpg)

## Topic&Partition的存储

Partition是以文件的形式存储在文件系统中，比如创建一个名为firstTopic的topic，其中有3个partition，那么在kafka的数据目录（/tmp/kafka-log）中就有3个目录，firstTopic-0~3，命名规则是`<topic_name>-<partition_id>`

```shell
./kafka-topics.sh --create --zookeeper 192.168.11.156:2181 --replication-factor 1 --partitions 3--topic firstTopic
```

# 关于消息分发

## kafka消息分发策略

消息是kafka中最基本的数据单元，在kafka中，一条消息由key、value两部分构成，在发送一条消息时，我们可以指定这个key，那么producer会根据key和partition机制来判断当前这条消息应该发送并存储到哪个partition中。我们可以根据需要进行扩展producer的partition机制。

## 消息默认的分发机制

默认情况下，kafka采用的是hash取模的分区算法。如果Key为null，则会随机分配一个分区。这个随机是在这个参数”metadata.max.age.ms”的时间范围内随机选择一个。对于这个时间段内，如果key为null，则只会发送到唯一的分区。这个值值哦默认情况下是10分钟更新一次。

关于Metadata，这个之前没讲过，简单理解就是Topic/Partition和broker的映射关系，每一个topic的每一个partition，需要知道对应的broker列表是什么，leader是谁、follower是谁。这些信息都是存储在Metadata这个类里面。

## 消费端如何消费指定的分区

通过下面的代码，就可以消费指定该topic下的0号分区。其他分区的数据就无法接收

```java
//消费指定分区的时候，不需要再订阅 
//kafkaConsumer.subscribe(Collections.singletonList(topic)); 

//消费指定的分区 
TopicPartition topicPartition=**new** TopicPartition(topic,0); kafkaConsumer.assign(Arrays.asList(topicPartition)); 
```

# 消息的消费原理

## kafka消息消费原理演示

在实际生产过程中，每个topic都会有多个partitions，多个partitions的好处在于，一方面能够对broker上的数据进行分片有效减少了消息的容量从而提升io性能。另外一方面，为了提高消费端的消费能力，一般会通过多个consumer去消费同一个topic，也就是消费端的负载均衡机制，也就是我们接下来要了解的，在多个partition以及多个consumer的情况下，消费者是如何消费消息的。

同时，在上一节课，我们讲了，kafka存在consumer group的概念，也就是group.id一样的consumer，这些consumer属于一个consumer group，组内的所有消费者协调在一起来消费订阅主题的所有分区。当然每一个分区只能由同一个消费组内的consumer来消费，那么同一个consumer group里面的consumer是怎么去分配该消费哪个分区里的数据的呢？如下图所示，3个分区，3个消费者，那么哪个消费者消分哪个分区？

![img113](http://ww3.sinaimg.cn/large/006tNc79gy1g631ti5mj4j30k208h0tk.jpg)

对于上面这个图来说，这3个消费者会分别消费test这个topic的3个分区，也就是每个consumer消费一个partition。

## 什么是分区分配策略

通过前面的案例演示，我们应该能猜到，同一个group中的消费者对于一个topic中的多个partition，存在一定的分区分配策略。

在kafka中，存在两种分区分配策略，一种是Range(默认)、另一种还是RoundRobin（轮询）。通过`partition.assignment.strategy`这个参数来设置。

### Rangestrategy（范围分区）

Range策略是对每个主题而言的，首先对同一个主题里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。假设我们有10个分区，3个消费者，排完序的分区将会是0, 1, 2, 3, 4, 5, 6, 7, 8, 9；消费者线程排完序将会是C1-0, C2-0, C3-0。然后将partitions的个数除于消费者线程的总数来决定每个消费者线程消费几个分区。如果除不尽，那么前面几个消费者线程将会多消费一个分区。在我们的例子里面，我们有10个分区，3个消费者线程，10 / 3 = 3，而且除不尽，那么消费者线程C1-0 将会多消费一个分区，所以最后分区分配的结果看起来是这样的：

> C1-0 将消费0, 1, 2, 3 分区
>
> C2-0 将消费4, 5, 6 分区
>
> C3-0 将消费7, 8, 9 分区

假如我们有11个分区，那么最后分区分配的结果看起来是这样的：

> C1-0 将消费0, 1, 2, 3 分区
>
> C2-0 将消费4, 5, 6, 7 分区
>
> C3-0将消费8, 9, 10 分区

假如我们有2个主题(T1和T2)，分别有10个分区，那么最后分区分配的结果看起来是这样的：

> C1-0 将消费T1主题的0, 1, 2, 3 分区以及T2主题的0, 1, 2, 3分区
>
> C2-0 将消费T1主题的4, 5, 6 分区以及T2主题的4, 5, 6分区
>
> C3-0将消费T1主题的7, 8, 9 分区以及T2主题的7, 8, 9分区

可以看出，C1-0 消费者线程比其他消费者线程多消费了2个分区，这就是Range strategy的一个很明显的弊端

### RoundRobinstrategy（轮询分区）

轮询分区策略是把所有partition和所有consumer线程都列出来，然后按照hashcode进行排序。最后通过轮询算法分配partition给消费线程。如果所有consumer实例的订阅是相同的，那么partition会均匀分布。

在我们的例子里面，假如按照hashCode 排序完的topic-partitions组依次为T1-5, T1-3, T1-0, T1-8, T1-2, T1-1, T1-4, T1-7, T1-6, T1-9，我们的消费者线程排序为C1-0, C1-1, C2-0, C2-1，最后分区分配的结果为：

> C1-0 将消费T1-5, T1-2, T1-6 分区；
>
> C1-1 将消费T1-3, T1-1, T1-9 分区；
>
> C2-0 将消费T1-0, T1-4 分区；
>
> C2-1 将消费T1-8, T1-7 分区；

使用轮询分区策略必须满足两个条件

1. 每个主题的消费者实例具有相同数量的流

2. 每个消费者订阅的主题必须是相同的

什么时候会触发这个策略呢？

当出现以下几种情况时，kafka会进行一次分区分配操作，也就是kafkaconsumer的rebalance

1. 同一个consumergroup内新增了消费者

2. 消费者离开当前所属的consumergroup，比如主动停机或者宕机

3. topic新增了分区（也就是分区数量发生了变化）

kafkaconsuemr的rebalance机制规定了一个consumergroup下的所有consumer如何达成一致来分配订阅topic的每个分区。而具体如何执行分区策略，就是前面提到过的两种内置的分区策略。而kafka对于分配策略这块，提供了可插拔的实现方式，也就是说，除了这两种之外，我们还可以创建自己的分配机制。

## 谁来执行Rebalance以及管理consumer的group呢？

Kafka提供了一个角色：coordinator来执行对于consumer group的管理，当consumer group的第一个consumer启动的时候，它会去和kafka server确定谁是它们组的coordinator。之后该group内的所有成员都会和该coordinator进行协调通信

## 如何确定coordinator

consumer group如何确定自己的coordinator是谁呢,消费者向kafka集群中的任意一个broker发送一个GroupCoordinatorRequest请求，服务端会返回一个负载最小的broker节点的id，并将该broker设置为coordinator

### JoinGroup的过程

在rebalance之前，需要保证coordinator是已经确定好了的，整个rebalance的过程分为两个步骤，Join和Sync

`join`: 表示加入到consumergroup中，在这一步中，所有的成员都会向coordinator发送joinGroup的请求。一旦所有成员都发送了joinGroup请求，那么coordinator会选择一个consumer担任leader角色，并把组成员信息和订阅信息发送消费者

![img124](http://ww1.sinaimg.cn/large/006tNc79gy1g632429hbrj30fp07174q.jpg)

`protocol_metadata`: 序列化后的消费者的订阅信息

`leader_id`：消费组中的消费者，coordinator会选择一个座位leader，对应的就是member_id

`member_metadata`：对应消费者的订阅信息

`members`：consumergroup中全部的消费者的订阅信息

`generation_id`：年代信息，类似于之前讲解zookeeper的时候的epoch是一样的，对于每一轮rebalance，generation_id都会递增。主要用来保护consumer group。隔离无效的offset提交。也就是上一轮的consumer成员无法提交offset到新的consumergroup中。

### SynchronizingGroupState阶段

完成分区分配之后，就进入了SynchronizingGroupState阶段，主要逻辑是向GroupCoordinator发送SyncGroupRequest请求，并且处理SyncGroupResponse响应，简单来说，就是leader将消费者对应的partition分配方案同步给consumergroup 中的所有consumer

![img127](http://ww4.sinaimg.cn/large/006tNc79gy1g632682uwyj30nb08l3z9.jpg)

每个消费者都会向coordinator发送syncgroup请求，不过只有leader节点会发送分配方案，其他消费者只是打打酱油而已。当leader把方案发给coordinator以后，coordinator会把结果设置到SyncGroupResponse中。这样所有成员都知道自己应该消费哪个分区。

-  consumer group的分区分配方案是在客户端执行的！Kafka将这个权利下放给客户端主要是因为这样做可以有更好的灵活性

## 如何保存消费端的消费位置

### 什么是offset

前面在讲解partition的时候，提到过offset，每个topic可以划分多个分区（每个Topic至少有一个分区），同一topic下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个offset（称之为偏移量），它是消息在此分区中的唯一编号，kafka通过offset保证消息在分区内的顺序，offset的顺序不跨分区，即kafka只保证在同一个分区内的消息是有序的；对于应用层的消费来说，每次消费一个消息并且提交以后，会保存当前消费到的最近的一个offset。那么offset保存在哪里？

![img130](http://ww1.sinaimg.cn/large/006tNc79gy1g6329uzugbj30ec079aad.jpg)

## offset在哪里维护？

在kafka中，提供了一个__consumer_offsets_*的一个topic，把offset信息写入到这个topic中。__consumer_offsets——按保存了每个consumergroup某一时刻提交的offset信息。__consumer_offsets 默认有50个分区。

根据前面我们演示的案例，我们设置了一个KafkaConsumerDemo的groupid。首先我们需要找到这个consumer_group保存在哪个分区中

properties.put(ConsumerConfig.**GROUP_ID_CONFIG**,**"KafkaConsumerDemo"**); 

##### 计算公式

- Math.abs(“groupid”.hashCode())%groupMetadataTopicPartitionCount ; 由于默认情况下groupMetadataTopicPartitionCount有50个分区，计算得到的结果为:35, 意味着当前的consumer_group的位移信息保存在__consumer_offsets的第35个分区
- 执行如下命令，可以查看当前consumer_goup中的offset位移信息

```shell
sh kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 35 --broker-list 192.168.11.153:9092,192.168.11.154:9092,192.168.11.157:9092 --formatter 
```

"kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter"

从输出结果中，我们就可以看到test这个topic的offset的位移日志

# 消息的存储

## 消息的保存路径

消息发送端发送消息到broker上以后，消息是如何持久化的呢？那么接下来去分析下消息的存储

首先我们需要了解的是，kafka是使用日志文件的方式来保存生产者和发送者的消息，每条消息都有一个offset值来表示它在分区中的偏移量。Kafka中存储的一般都是海量的消息数据，为了避免日志文件过大，Log并不是直接对应在一个磁盘上的日志文件，而是对应磁盘上的一个目录，这个目录的明明规则是<topic_name>_<partition_id>

比如创建一个名为firstTopic的topic，其中有3个partition，那么在kafka的数据目录（/tmp/kafka-log）中就有3个目录，firstTopic-0~3

多个分区在集群中的分配

如果我们对于一个topic，在集群中创建多个partition，那么partition是如何分布的呢？

1.将所有N Broker和待分配的i个Partition排序

2.将第i个Partition分配到第(i mod n)个Broker上

![img139](http://ww2.sinaimg.cn/large/006tNc79gy1g632f8bhz1j30k509q751.jpg)

了解到这里的时候，大家再结合前面讲的消息分发策略，就应该能明白消息发送到broker上，消息会保存到哪个分区中，并且消费端应该消费哪些分区的数据了。

## 消息写入的性能

我们现在大部分企业仍然用的是机械结构的磁盘，如果把消息以随机的方式写入到磁盘，那么磁盘首先要做的就是寻址，也就是定位到数据所在的物理地址，在磁盘上就要找到对应的柱面、磁头以及对应的扇区；这个过程相对内存来说会消耗大量时间，为了规避随机读写带来的时间消耗，kafka采用顺序写的方式存储数据。即使是这样，但是频繁的I/O操作仍然会造成磁盘的性能瓶颈，所以kafka还有一个性能策略

**零拷贝**

消息从发送到落地保存，broker维护的消息日志本身就是文件目录，每个文件都是二进制保存，生产者和消费者使用相同的格式来处理。在消费者获取消息时，服务器先从硬盘读取数据到内存，然后把内存中的数据原封不动的通过socket发送给消费者。虽然这个操作描述起来很简单，但实际上经历了很多步骤。

![img142](http://ww2.sinaimg.cn/large/006tNc79gy1g632grmk5mj30d80a2mxk.jpg)

操作系统将数据从磁盘读入到内核空间的页缓存

- 应用程序将数据从内核空间读入到用户空间缓存中
- 应用程序将数据写回到内核空间到socket缓存中
- 操作系统将数据从socket缓冲区复制到网卡缓冲区，以便将数据经网络发出

这个过程涉及到4次上下文切换以及4次数据复制，并且有两次复制操作是由CPU完成。但是这个过程中，数据完全没有进行变化，仅仅是从磁盘复制到网卡缓冲区。

通过“零拷贝”技术，可以去掉这些没必要的数据复制操作，同时也会减少上下文切换次数。现代的unix操作系统提供一个优化的代码路径，用于将数据从页缓存传输到socket；在Linux中，是通过sendfile系统调用来完成的。Java提供了访问这个系统调用的方法：FileChannel.transferTo API

![img145](http://ww2.sinaimg.cn/large/006tNc79gy1g632hn0losj30d20a4q3b.jpg)

使用sendfile，只需要一次拷贝就行，允许操作系统将数据直接从页缓存发送到网络上。所以在这个优化的路径中，只有最后一步将数据拷贝到网卡缓存中是需要的