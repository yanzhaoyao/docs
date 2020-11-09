#  消息的文件存储机制

 前面我们知道了一个topic的多个partition在物理磁盘上的保存路径，那么我们再来分析日志的存储方式。通过如下命令找到对应partition下的日志内容

```shell
$ ls /tmp/kafka-logs/test-0/
00000000000000000000.index     00000000000000000004.snapshot
00000000000000000000.log       leader-epoch-checkpoint
00000000000000000000.timeindex
```

kafka是通过分段的方式将Log分为多个LogSegment，LogSegment是一个逻辑上的概念，一个LogSegment对应磁盘上的一个日志文件和一个索引文件，其中日志文件是用来记录消息的。索引文件是用来保存消息的索引。那么这个LogSegment是什么呢？

## **LogSegment**

假设kafka以partition为最小存储单位，那么我们可以想象当kafka producer不断发送消息，必然会引起partition文件的无线扩张，这样对于消息文件的维护以及被消费的消息的清理带来非常大的挑战，所以kafka 以segment为单位又把partition进行细分。每个partition相当于一个巨型文件被平均分配到多个大小相等的segment数据文件中（每个segment文件中的消息不一定相等），这种特性方便已经被消费的消息的清理，提高磁盘的利用率。

- log.segment.bytes=107370(设置分段大小),默认是1gb，我们把这个值调小以后，可以看到日志分段的效果

抽取其中3个分段来进行分析

![img94](http://ww4.sinaimg.cn/large/006tNc79gy1g6593raokfj30r50a9abf.jpg)

segment file由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀"`.index`"和“`.log`”分别表示为segment索引文件、数据文件.

segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值进行递增。数值最大为64位long大小，20位数字字符长度，没有数字用0填充

## **查看segment文件命名规则**

- 通过下面这条命令可以看到kafka消息日志的内容

```shell
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.log --print-data-log
```

输出结果为：

```
offset: 5376 position: 102124 CreateTime: 1531477349287 isvalid: true keysize: -1 valuesize: 12 magic: 2 compresscodec: NONE producerId: 1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: message_5376
```

第一个log文件的最后一个offset为:5376,所以下一个segment的文件命名为:00000000000000005376.log。对应的index为00000000000000005376.index

## **segment中index和log的对应关系**

从所有分段中，找一个分段进行分析

为了提高查找消息的性能，为每一个日志文件添加2个索引索引文件：OffsetIndex 和TimeIndex，分别对应*`.index`以及*`.timeindex`,TimeIndex索引文件格式：它是映射时间戳和相对offset

查看索引内容：

```shell
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.index --print-data-log
```

![img99](http://ww1.sinaimg.cn/large/006tNc79gy1g6593jvnj4j30l00e0wg5.jpg)

如图所示，index中存储了索引以及物理偏移量。log存储了消息的内容。索引文件的元数据执行对应数据文件中message的物理偏移地址。举个简单的案例来说，以[4053,80899]为例，在log文件中，对应的是第4053条记录，物理偏移量（position）为80899.  position是ByteBuffer的指针位置

## **在partition中如何通过offset查找message**

查找的算法是

1. 根据offset的值，查找segment段中的index索引文件。由于索引文件命名是以上一个文件的最后一个offset进行命名的，所以，使用二分查找算法能够根据offset快速定位到指定的索引文件。

2. 找到索引文件后，根据offset进行定位，找到索引文件中的符合范围的索引。（kafka采用稀疏索引的方式来提高查找性能）

3. 得到position以后，再到对应的log文件中，从position出开始查找offset对应的消息，将每条消息的offset与目标offset进行比较，直到找到消息

比如说，我们要查找offset=2490这条消息，那么先找到00000000000000000000.index, 然后找到[2487,49111]这个索引，再到log文件中，根据49111这个position开始查找，比较每条消息的offset是否大于等于2490。最后查找到对应的消息以后返回

## **Log文件的消息内容分析**

前面我们通过kafka提供的命令，可以查看二进制的日志文件信息，一条消息，会包含很多的字段。

```
offset: 5371 position: 102124 CreateTime: 1531477349286 isvalid: true keysize: -1 valuesize: 12 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: message_5371
```

offset和position这两个前面已经讲过了、createTime表示创建时间、keysize和valuesize表示key和value的大小、compresscodec表示压缩编码、payload:表示消息的具体内容

日志的清除策略以及压缩策略

## **日志清除策略**

前面提到过，日志的分段存储，一方面能够减少单个文件内容的大小，另一方面，方便kafka进行日志清理。日志的清理策略有两个

1. 根据消息的保留时间，当消息在kafka中保存的时间超过了指定的时间，就会触发清理过程

2. 根据topic存储的数据大小，当topic所占的日志文件大小大于一定的阀值，则可以开始删除最旧的消息。kafka会启动一个后台线程，定期检查是否存在可以删除的消息

通过log.retention.bytes和log.retention.hours这两个参数来设置，当其中任意一个达到要求，都会执行删除。

默认的保留时间是：7天

## **日志压缩策略**

Kafka还提供了“日志压缩（LogCompaction）”功能，通过这个功能可以有效的减少日志文件的大小，缓解磁盘紧张的情况，在很多实际场景中，消息的key和value的值之间的对应关系是不断变化的，就像数据库中的数据会不断被修改一样，消费者只关心key对应的最新的value。因此，我们可以开启kafka的日志压缩功能，服务端会在后台启动启动Cleaner线程池，定期将相同的key进行合并，只保留最新的value值。日志的压缩原理是

![img106](http://ww2.sinaimg.cn/large/006tNc79gy1g6593c93jxj30i1087mxq.jpg)

# partition的高可用副本机制

我们已经知道Kafka的每个topic都可以分为多个Partition，并且多个partition会均匀分布在集群的各个节点下。虽然这种方式能够有效的对数据进行分片，但是对于每个partition来说，都是单点的，当其中一个partition不可用的时候，那么这部分消息就没办法消费。所以kafka为了提高partition的可靠性而提供了副本的概念（Replica）,通过副本机制来实现冗余备份。

每个分区可以有多个副本，并且在副本集合中会存在一个leader的副本，所有的读写请求都是由leader副本来进行处理。剩余的其他副本都做为follower副本，follower副本会从leader副本同步消息日志。这个有点类似zookeeper中leader和follower的概念，但是具体的时间方式还是有比较大的差异。所以我们可以认为，副本集会存在一主多从的关系。

一般情况下，同一个分区的多个副本会被均匀分配到集群中的不同broker上，当leader副本所在的broker出现故障后，可以重新选举新的leader副本继续对外提供服务。通过这样的副本机制来提高kafka集群的可用性。

副本分配算法

将所有N Broker和待分配的i个Partition排序.

将第i个Partition分配到第(i mod n)个Broker上.

将第i个Partition的第j个副本分配到第((i + j) mod n)个Broker上.

创建一个带副本机制的topic

通过下面的命令去创建带2个副本的topic

```shell
./kafka-topics.sh --create --zookeeper 192.168.0.102:2181 --replication-factor 2 --partitions 3 --topic secondTopic
```

然后我们可以在/tmp/kafka-log路径下看到对应topic的副本信息了。我们通过一个图形的方式来表达。

-  针对secondTopic这个topic的3个分区对应的3个副本

  ![img111](http://ww3.sinaimg.cn/large/006tNc79gy1g65931fauoj30kk09fgmb.jpg)

如何知道那个各个分区中对应的leader是谁呢？

在zookeeper服务器上，通过如下命令去获取对应分区的信息,比如下面这个是获取secondTopic第1个分区的状态信息。

get /brokers/topics/secondTopic/partitions/1/state

{"controller_epoch":12,"leader":0,"version":1,"leader_epoch":0,"isr":[0,1]}

leader表示当前分区的leader是那个broker-id。下图中。

绿色线条的表示该分区中的leader节点。其他节点就为follower

![img114](http://ww1.sinaimg.cn/large/006tNc79gy1g6592tleeaj30kb09cdgk.jpg)

Kafka提供了数据复制算法保证，如果leader发生故障或挂掉，一个新leader被选举并被接受客户端的消息成功写入。Kafka确保从同步副本列表中选举一个副本为leader；leader负责维护和跟踪ISR(in-Sync replicas ，副本同步队列)中所有follower滞后的状态。当producer发送一条消息到broker后，leader写入消息并复制到所有follower。消息提交之后才被成功复制到所有的同步副本。

-  既然有副本机制，就一定涉及到数据同步的概念，那接下来分析下数据是如何同步的？

需要注意的是，大家不要把zookeeper的leader和follower的同步机制和kafka副本的同步机制搞混了。虽然从思想层面来说是一样的，但是原理层面的实现是完全不同的。

## kafka副本机制中的几个概念

Kafka分区下有可能有很多个副本(replica)用于实现冗余，从而进一步实现高可用。副本根据角色的不同可分为3类：

leader副本：响应clients端读写请求的副本

follower副本：被动地备份leader副本中的数据，不能响应clients端读写请求。

ISR副本：包含了leader副本和所有与leader副本保持同步的follower副本——如何判定是否与leader同步后面会提到每个Kafka副本对象都有两个重要的属性：LEO和HW。注意是所有的副本，而不只是leader副本。

LEO：即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息！也就是说，如果LEO=10，那么表示该副本保存了10条消息，位移值范围是[0, 9]。另外，leader LEO和follower LEO的更新是有区别的。我们后面会详细说

HW：即上面提到的水位值。对于同一个副本对象而言，其HW值不会大于LEO值。小于等于HW值的所有消息都被认为是“已备份”的（replicated）。同理，leader副本和follower副本的HW更新是有区别的

## 副本协同机制

刚刚提到了，消息的读写操作都只会由leader节点来接收和处理。follower副本只负责同步数据以及当leader副本所在的broker挂了以后，会从follower副本中选取新的leader。

![img119](http://ww4.sinaimg.cn/large/006tNc79gy1g6598f2jadj30fp0e5q3n.jpg)

写请求首先由Leader副本处理，之后follower副本会从leader上拉取写入的消息，这个过程会有一定的延迟，导致follower副本中保存的消息略少于leader副本，但是只要没有超出阈值都可以容忍。但是如果一个follower副本出现异常，比如宕机、网络断开等原因长时间没有同步到消息，那这个时候，leader就会把它踢出去。kafka通过ISR集合来维护一个分区副本信息集合

### ISR(in-Sync replica ，副本同步)

ISR表示目前“可用且消息量与leader相差不多的副本集合，这是整个副本集合的一个子集”。怎么去理解可用和相差不多这两个词呢？具体来说，ISR集合中的副本必须满足两个条件

1. 副本所在节点必须维持着与zookeeper的连接

2. 副本最后一条消息的offset与leader副本的最后一条消息的offset之间的差值不能超过指定的阈值(replica.lag.time.max.ms)

`replica.lag.time.max.ms`：如果该follower在此时间间隔内一直没有追上过leader的所有消息，则该follower就会被剔除isr列表

-  ISR数据保存在Zookeeper的/brokers/topics/\<topic\>/partitions/\<partitionId\>/state节点中

### HW&LEO

关于follower副本同步的过程中，还有两个关键的概念，HW(HighWatermark)和LEO(Log End Offset). 这两个参数跟ISR集合紧密关联。HW标记了一个特殊的offset，当消费者处理消息的时候，只能拉去到HW之前的消息，HW之后的消息对消费者来说是不可见的。也就是说，取partition对应ISR中最小的LEO作为HW，consumer最多只能消费到HW所在的位置。每个replica都有HW，leader和follower各自维护更新自己的HW的状态。一条消息只有被ISR里的所有Follower都从Leader复制过去才会被认为已提交。这样就避免了部分数据被写进了Leader，还没来得及被任何Follower复制就宕机了，而造成数据丢失（Consumer无法消费这些数据）。而对于Producer而言，它可以选择是否等待消息commit，这可以通过acks来设置。这种机制确保了只要ISR有一个或以上的Follower，一条被commit的消息就不会丢失。

数据的同步过程

了解了副本的协同过程以后，还有一个最重要的机制，就是数据的同步过程。它需要解决

1. 怎么传播消息

2. 在向消息发送端返回ack之前需要保证多少个Replica已经接收到这个消息

数据的处理过程是

Producer在发布消息到某个Partition时，先通过ZooKeeper找到该Partition的Leader【get /brokers/topics/\<topic\>/partitions/2/state】，然后无论该Topic的Replication Factor为多少（也即该Partition有多少个Replica），Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW(HighWatermark)并且向Producer发送ACK。

### **初始状态**

初始状态下，leader和follower的HW和LEO都是0，leader副本会保存remoteLEO，表示所有followerLEO，也会被初始化为0，这个时候，producer没有发送消息。follower会不断地向leader发送FETCH请求，但是因为没有数据，这个请求会被leader寄存，当在指定的时间之后会强制完成请求，这个时间配置是(`replica.fetch.wait.max.ms`)，如果在指定时间内producer有消息发送过来，那么kafka会唤醒fetch请求，让leader继续处理

这里会分两种情况，第一种是leader处理完producer请求之后，follower发送一个fetch请求过来、第二种是follower阻塞在leader指定时间之内，leader副本收到producer的请求。这两种情况下处理方式是不一样的。先来看第一种情况

follower的fetch请求是当leader处理消息以后执行的

**生产者发送一条消息**

- leader处理完producer请求之后，follower发送一个fetch请求过来。

leader副本收到请求以后，会做几件事情

1. 把消息追加到log文件，同时更新leader副本的LEO

2. 尝试更新leaderHW值。这个时候由于follower副本还没有发送fetch请求，那么leader的remote LEO仍然是0。leader会比较自己的LEO以及remote LEO的值发现最小值是0，与HW的值相同，所以不会更新HW

### **followerfetch消息**

- follower发送fetch请求，leader副本的处理逻辑是:

1. 读取log数据、更新remoteLEO=0(follower还没有写入这条消息，这个值是根据follower的fetch请求中的offset来确定的)

2. 尝试更新HW，因为这个时候LEO和remoteLEO还是不一致，所以仍然是HW=0

3. 把消息内容和当前分区的HW值发送给follower副本



- follower副本收到response以后

1. 将消息写入到本地log，同时更新follower的LEO

2. 更新followerHW，本地的LEO和leader返回的HW进行比较取小的值，所以仍然是0第一次交互结束以后，HW仍然还是0，这个值会在下一次follower发起fetch请求时被更新

- follower发第二次fetch请求，leader收到请求以后

1. 读取log数据

2. 更新remoteLEO=1，因为这次fetch携带的offset是1.

3. 更新当前分区的HW，这个时候leader LEO和remoteLEO都是1，所以HW的值也更新为1

4. 把数据和当前分区的HW值返回给follower副本，这个时候如果没有数据，则返回为空

- follower副本收到response以后

1. 如果有数据则写本地日志，并且更新LEO

2. 更新follower的HW值



到目前为止，数据的同步就完成了，意味着消费端能够消费offset=0这条消息。

follower的fetch请求是直接从阻塞过程中触发

前面说过，由于leader副本暂时没有数据过来，所以follower的fetch会被阻塞，直到等待超时或者leader接收到新的数据。当leader收到请求以后会唤醒处于阻塞的fetch请求。处理过程基本上和前面说的一直

1. leader将消息写入本地日志，更新Leader的LEO

2. 唤醒follower的fetch请求

3. 更新HW



kafka使用HW和LEO的方式来实现副本数据的同步，本身是一个好的设计，但是在这个地方会存在一个数据丢失的问题，当然这个丢失只出现在特定的背景下。我们回想一下，HW的值是在新的一轮FETCH中才会被更新。我们分析下这个过程为什么会出现数据丢失

数据丢失的问题

前提：`min.insync.replicas=1`的时候。->设定ISR中的最小副本数是多少，默认值为1,当且仅当acks参数设置为-1（表示需要所有副本确认）时，此参数才生效.表达的含义是，至少需要多少个副本同步才能表示消息是提交的

所以，当`min.insync.replicas=1`的时候

一旦消息被写入leader端log即被认为是“已提交”，而延迟一轮FETCH RPC更新HW值的设计使得follower HW值是异步延迟更新的，倘若在这个过程中leader发生变更，那么成为新leader的follower的HW值就有可能是过期的，使得clients端认为是成功提交的消息被删除。

![img140](http://ww4.sinaimg.cn/large/006tNc79gy1g659olwfcnj30m90ebgn5.jpg)

### 数据丢失的解决方案

在kafka0.11.0.0版本以后，提供了一个新的解决方案，使用leader epoch来解决这个问题，leaderepoch实际上是一对之(epoch,offset), epoch表示leader的版本号，从0开始，当leader变更过1次时epoch就会+1，而offset则对应于该epoch版本的leader写入第一条消息的位移。比如说

(0,0) ; (1,50);  表示第一个leader从offset=0开始写消息，一共写了50条，第二个leader版本号是1，从50条处开始写消息。这个信息保存在对应分区的本地磁盘文件中，文件名为：/tml/kafka-log/topic/leader-epoch-checkpoint

leader broker中会保存这样的一个缓存，并定期地写入到一个checkpoint文件中。

当leader写log时它会尝试更新整个缓存——如果这个leader首次写消息，则会在缓存中增加一个条目；否则就不做更新。而每次副本重新成为leader时会查询这部分缓存，获取出对应leader版本的offset

![img145](http://ww2.sinaimg.cn/large/006tNc79gy1g659s4bvwuj30lu0dzta4.jpg)

### 如何处理所有的Replica不工作的情况

在ISR中至少有一个follower时，Kafka可以确保已经commit的数据不丢失，但如果某个Partition的所有Replica都宕机了，就无法保证数据不丢失了

1. 等待ISR中的任一个Replica“活”过来，并且选它作为Leader

2. 选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader



这就需要在可用性和一致性当中作出一个简单的折衷。

如果一定要等待ISR中的Replica“活”过来，那不可用的时间就可能会相对较长。而且如果ISR中的所有Replica都无法“活”过来了，或者数据都丢失了，这个Partition将永远不可用。

选择第一个“活”过来的Replica作为Leader，而这个Replica不是ISR中的Replica，那即使它并不保证已经包含了所有已commit的消息，它也会成为Leader而作为consumer的数据源（前文有说明，所有读写都由Leader完成）。在我们前面讲的版本中，使用的是第一种策略。

## ISR的设计原理

在所有的分布式存储中，冗余备份是一种常见的设计方式，而常用的模式有同步复制和异步复制，按照kafka这个副本模型来说

如果采用同步复制，那么需要要求所有能工作的Follower副本都复制完，这条消息才会被认为提交成功，一旦有一个follower副本出现故障，就会导致HW无法完成递增，消息就无法提交，消费者就获取不到消息。这种情况下，故障的Follower副本会拖慢整个系统的性能，设置导致系统不可用

如果采用异步复制，leader副本收到生产者推送的消息后，就认为次消息提交成功。follower副本则异步从leader副本同步。这种设计虽然避免了同步复制的问题，但是假设所有follower副本的同步速度都比较慢他们保存的消息量远远落后于leader副本。而此时leader副本所在的broker突然宕机，则会重新选举新的leader副本，而新的leader副本中没有原来leader副本的消息。这就出现了消息的丢失。

kafka权衡了同步和异步的两种策略，采用ISR集合，巧妙解决了两种方案的缺陷：当follower副本延迟过高，leader副本则会把该follower副本提出ISR集合，消息依然可以快速提交。当leader副本所在的broker突然宕机，会优先将ISR集合中follower副本选举为leader，新leader副本包含了HW之前的全部消息，这样就避免了消息的丢失。