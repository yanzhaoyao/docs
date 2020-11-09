# Redis内存回收策略

# 1.概念

> Redis的内存回收机制主要体现在以下两个方面: 
>
> - 删除到达过期时间的键对象
> - 内存使用达到maxmemory上限时触发内存溢出控制策略

# 2.删除过期的键

Redis所有的键都可以设置过期属性，内部保存在过期字典中。由于进程内保存大量的键，维护每个键精准的过期删除机制会导致消耗大量的 CPU，对于单线程的Redis来说成本过高，因此Redis采用`惰性删除`和`定时任务删除`机制实现过期键的内存回收。

## 2.1惰性删除

惰性删除用于当客户端读取带有超时属性的键时，如果已经超过键设置的过期时间，会执行删除操作并返回空，这种策略是出于节省 CPU成本考虑，不需要单独维护TTL链表来处理过期键的删除。但是单独用这种方式存在内存泄露的问题，当过期键一直没有访问将无法得到及时删除，从而导致内存不能及时释放。

正因为如此，Redis还提供另一种定时任 务删除机制作为惰性删除的补充。

## 2.2定时任务删除

Redis内部维护一个定时任务，默认每秒运行10次(通过配置hz控制)。定时任务中删除过期键逻辑采用了自适应算法，根据键的过期比例、使用快慢两种速率模式回收键

流程说明:

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkfx3d1armj30u00w949g.jpg" alt="定时任务删除过期键逻辑" style="zoom:50%;" />

1. 定时任务在每个数据库空间随机检查20个键，当发现过期时删除对应的键。
2. 如果超过检查数25%的键过期，循环执行回收逻辑直到不足25%或运行超时为止，慢模式下超时时间为25毫秒。
3. 如果之前回收键逻辑超时，则在Redis触发内部事件之前再次以快模式运行回收过期键任务，快模式下超时时间为1毫秒且2秒内只能运行1次。
4. 快慢两种模式内部删除逻辑相同，只是执行的超时时间不同。

# 3.内存溢出控制策略

当Redis所用内存达到maxmemory上限时会触发相应的溢出控制策略。 具体策略受maxmemory-policy参数控制，Redis支持6种策略，如下所示:

1. `noeviction`:默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息(error)OOM command not allowed when used memory，此时Redis只响应读操作。
2. `volatile-lru`:根据LRU算法删除设置了超时属性(expire)的键，直到腾出足够空间为止。如果没有可删除的键对象，回退到noeviction策略。
3. `volatile-random`:随机删除过期键，直到腾出足够空间为止。
4. `volatile-ttl`:根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到`noeviction`策略。
5. `allkeys-lru`:根据LRU算法删除键，不管数据有没有设置超时属性， 直到腾出足够空间为止。
6. `allkeys-random`:随机删除所有键，直到腾出足够空间为止。

# 4.内存优化

Redis所有的数据都在内存中，而内存又是非常宝贵的资源。如何优化内存的使用一直是Redis用户非常关注的问题。本节深入到Redis细节中，探 索内存优化的技巧。

## 4.1redisObject对象

Redis存储的所有值对象在内部定义为redisObject结构体，内部结构如图

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkfxc1y64yj30uc0matgh.jpg" alt="redisObject内部结构" style="zoom:50%;" />

Redis存储的数据都使用redisObject来封装，包括string、hash、list、 set、zset在内的所有数据类型。理解redisObject对内存优化非常有帮助，下 面针对每个字段做详细说明:

- `type字段`:表示当前对象使用的数据类型，Redis主要支持5种数据类型:string、hash、list、set、zset。可以使用type{key}命令查看对象所属类 型，type命令返回的是值对象类型，键都是string类型。

- `encoding字段`:表示Redis内部编码类型，encoding在Redis内部使用， 代表当前对象内部采用哪种数据结构实现。理解Redis内部编码方式对于优化内存非常重要，同一个对象采用不同的编码实现内存占用存在明显差异。

- `lru字段`:记录对象最后一次被访问的时间，当配置了maxmemory和 maxmemory-policy=volatile-lru或者allkeys-lru时，用于辅助LRU算法删除键数据。可以使用object idletime{key}命令在不更新lru字段情况下查看当前键的空闲时间。
- `refcount字段`:记录当前对象被引用的次数，用于通过引用次数回收内 存，当refcount=0时，可以安全回收当前对象空间。使用object refcount{key} 获取当前对象引用。当对象为整数且范围在[0-9999]时，Redis可以使用共享对象的方式来节省内存。具体细节见《Redis开发与运维（完整篇））》8.3.3节“共享对象池”部分。
- `ptr字段`:与对象的数据内容相关，如果是整数，直接存储数据;否则表示指向数据的指针。Redis在3.0之后对值对象是字符串且长度<=39字节的数据，内部编码为embstr类型，字符串sds和redisObject一起分配，从而只要一次内存操作即可。

> Tips:
>
> 高并发写入场景中，在条件允许的情况下，建议字符串长度控制在39字 节以内，减少创建redisObject内存分配次数，从而提高性能。

## 4.2缩减键值对象

降低Redis内存使用最直接的方式就是缩减键(key)和值(value)的长度。

- key长度:如在设计键时，在完整描述业务情况下，键值越短越好。如

  user:{uid}:friends:notify:{fid}可以简化为u:{uid}:fs:nt:{fid}。

- value长度:值对象缩减比较复杂，常见需求是把业务对象序列化成二 进制数组放入Redis。首先应该在业务上精简业务对象，去掉不必要的属性 避免存储无效数据。其次在序列化工具选择上，应该选择更高效的序列化工具来降低字节数组大小。以Java为例，内置的序列化方式无论从速度还是压缩比都不尽如人意，这时可以选择更高效的序列化工具，如:protostuff、 kryo等

  ![Java常见序列化工具空间压缩对比](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfxj6s0c8j30xe0ds0v6.jpg)

其中java-built-in-serializer表示Java内置序列化方式，更多数据见jvm-serializers项目:https://github.com/eishay/jvm-serializers/wiki，其他语言也有各自对应的高效序列化工具。

值对象除了存储二进制数据之外，通常还会使用通用格式存储数据比 如:json、xml等作为字符串存储在Redis中。这种方式优点是方便调试和跨 语言，但是同样的数据相比字节数组所需的空间更大，在内存紧张的情况 下，可以使用通用压缩算法压缩json、xml后再存入Redis，从而降低内存占 用，例如使用GZIP压缩后的json可降低约60%的空间。

## 4.3共享对象池

共享对象池是指Redis内部维护[0-9999]的整数对象池。创建大量的整数 类型redisObject存在内存开销，每个redisObject内部结构至少占16字节，甚至超过了整数自身空间消耗。所以Redis内存维护一个[0-9999]的整数对象 池，用于节约内存。除了整数值对象，其他类型如list、hash、set、zset内部元素也可以使用整数对象池。

因此开发中在满足需求的前提下，尽量使用整数对象以节省内存。

整数对象池在Redis中通过变量REDIS_SHARED_INTEGERS定义，不能 通过配置修改。可以通过object refcount命令查看对象引用数验证是否启用整 数对象池技术，如下:

```shell
redis> set foo 100
 OK
redis> object refcount foo (integer) 2
redis> set bar 100
 OK
redis> object refcount bar (integer) 3
```

设置键foo等于100时，直接使用共享池内整数对象，因此引用数是2， 再设置键bar等于100时，引用数又变为3。

![整数对象池共享机制](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfxm9db3kj30x00gwgqd.jpg)

**使用整数对象池究竟能降低多少内存?**

让我们通过测试来对比对象池的内存优化效果，

![image-20201107004743023](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfxojxpmtj30x204kdhb.jpg)

> 注意
>
> 本章所有测试环境都保持一致，信息如下:
>
> 服务器信息:cpu=Intel-Xeon E5606@2.13GHz memory=32GB
>
> Redis版本:Redis server v=3.0.7sha=00000000:0malloc=jemalloc- 3.6.0bits=64

使用共享对象池后，相同的数据内存使用降低30%以上。可见当数据大量使用[0-9999]的整数时，共享对象池可以节约大量内存。需要注意的是对象池并不是只要存储[0-9999]的整数就可以工作。当设置maxmemory并启用 LRU相关淘汰策略如:volatile-lru，allkeys-lru时，Redis禁止使用共享对象池，测试命令如下:

```shell
redis> set key:1 99
OK // 设置key:1=99
redis> object refcount key:1
(integer) 2 // 使用了对象共享,引用数为2
redis> config set maxmemory-policy volatile-lru OK // 开启LRU淘汰策略
redis> set key:2 99
OK // 设置key:2=99
redis> object refcount key:2
(integer) 3 // 使用了对象共享,引用数变为3
redis> config set maxmemory 1GB
OK // 设置最大可用内存
redis> set key:3 99
OK // 设置key:3=99
redis> object refcount key:3
(integer) 1 // 未使用对象共享,引用数为1
redis> config set maxmemory-policy volatile-ttl OK // 设置非LRU淘汰策略
redis> set key:4 99
OK // 设置key:4=99
redis> object refcount key:4
(integer) 4 // 又可以使用对象共享,引用数变为4
```

**为什么开启maxmemory和LRU淘汰策略后对象池无效?**

LRU算法需要获取对象最后被访问时间，以便淘汰最长未访问数据，每 个对象最后访问时间存储在redisObject对象的lru字段。对象共享意味着多个引用共享同一个redisObject，这时lru字段也会被共享，导致无法获取每个对象的最后访问时间。如果没有设置maxmemory，直到内存被用尽Redis也不会触发内存回收，所以共享对象池可以正常工作。

综上所述，共享对象池与maxmemory+LRU策略冲突，使用时需要注意。对于ziplist编码的值对象，即使内部数据为整数也无法使用共享对象池，因为ziplist使用压缩且内存连续的结构，对象共享判断成本过高，ziplist 编码细节后面内容详细说明。

**为什么只有整数对象池?**

首先整数对象池复用的几率最大，其次对象共享的一个关键操作就是判 断相等性，Redis之所以只有整数对象池，是因为整数比较算法时间复杂度 为O(1)，只保留一万个整数为了防止对象池浪费。如果是字符串判断相 等性，时间复杂度变为O(n)，特别是长字符串更消耗性能(浮点数在 Redis内部使用字符串存储)。对于更复杂的数据结构如hash、list等，相等 性判断需要O(n2)。对于单线程的Redis来说，这样的开销显然不合理，因此Redis只保留整数共享对象池。

## 4.4字符串优化

字符串对象是Redis内部最常用的数据类型。所有的键都是字符串类型，值对象数据除了整数之外都使用字符串存储。比如执行命令:lpush cache:type"redis""memcache""tair""levelDB"，Redis首先创建"cache:type"键 字符串，然后创建链表对象，链表对象内再包含四个字符串对象，排除 Redis内部用到的字符串对象之外至少创建5个字符串对象。可见字符串对象在Redis内部使用非常广泛，因此深刻理解Redis字符串对于内存优化非常有帮助。

## 4.5编码优化

### 4.5.1了解编码

Redis对外提供了string、list、hash、set、zet等类型，但是Redis内部针对 不同类型存在编码的概念，所谓编码就是具体使用哪种底层数据结构来实 现。编码不同将直接影响数据的内存占用和读写效率。使用object encoding{key}命令获取编码类型。如下所示:

```shell
redis> set str:1 hello
OK
redis> object encoding str:1
"embstr" // embstr编码字符串 redis> lpush list:1 1 2 3
(integer) 3
redis> object encoding list:1
"ziplist" // ziplist编码列表
```

Redis针对每种数据类型(type)可以采用至少两种编码方式来实现，如下对应关系。

![type和encoding对应关系表](https://tva1.sinaimg.cn/large/0081Kckwgy1gkfxtuct39j318e0i6afm.jpg)

> 了解编码和类型对应关系之后，我们不禁疑惑Redis为什么对一种数据 结构实现多种编码方式?
>
> 主要原因是Redis作者想通过不同编码实现效率和空间的平衡。比如当我们的存储只有10个元素的列表，当使用双向链表数据结构时，必然需要维护大量的内部字段如每个元素需要:前置指针，后置指针，数据指针等，造成空间浪费，如果采用连续内存结构的压缩列表(ziplist)，将会节省大量内存，而由于数据长度较小，存取操作时间复杂度即使为O(n2)性能也可满足需求。

### 4.5.2控制编码类型

编码类型转换在Redis写入数据时自动完成，这个转换过程是不可逆的，转换规则只能从小内存编码向大内存编码转换。例如:

```shell
redis> lpush list:1 a b c d
(integer) 4 // 存储4个元素
redis> object encoding list:1
"ziplist" // 采用ziplist压缩列表编码 
redis> config set list-max-ziplist-entries 4 OK // 设置列表类型ziplist编码最大允许4个元素 
redis> lpush list:1 e
(integer) 5 // 写入第5个元素e 
redis> object encoding list:1 
"linkedlist" // 编码类型转换为链表 
redis> rpop list:1
"a" //弹出a元素
redis> llen list:1
(integer) 4 // 列表此时有4个元素
redis> object encoding list:1 
"linkedlist" // 编码类型依然为链表，未做编码回退

```

以上命令体现了list类型编码的转换过程，其中Redis之所以不支持编码回退，主要是数据增删频繁时，数据向压缩编码转换非常消耗CPU，得不偿 失。以上示例用到了list-max-ziplist-entries参数，这个参数用来决定列表长度在多少范围内使用ziplist编码。当然还有其他参数控制各种数据类型的编码，如下所示。

<img src="/Users/xmly/Library/Application Support/typora-user-images/image-20201107005928063.png" alt="image-20201107005928063" style="zoom: 50%;" />

<img src="/Users/xmly/Library/Application Support/typora-user-images/image-20201107010000984.png" alt="image-20201107010000984" style="zoom:50%;" />

## 4.6控制键的数量

当使用Redis存储大量数据时，通常会存在大量键，过多的键同样会消 耗大量内存。Redis本质是一个数据结构服务器，它为我们提供多种数据结 构，如hash、list、set、zset等。使用Redis时不要进入一个误区，大量使用 get/set这样的API，把Redis当成Memcached使用。对于存储相同的数据内容利用Redis的数据结构降低外层键的数量，也可以节省大量内存。

如下所示，通过在客户端预估键规模，把大量键分组映射到多个hash结构中降低键的数量。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkfy2eswogj318e0j6q8i.jpg" alt="image-20201107010125794" style="zoom:50%;" />

hash结构降低键数量分析:

- 根据键规模在客户端通过分组映射到一组hash对象中，如存在100万个 键，可以映射到1000个hash中，每个hash保存1000个元素。
- hash的field可用于记录原始key字符串，方便哈希查找。
- hash的value保存原始值对象，确保不要超过hash-max-ziplist-value限制。

# 	5.总结

本节主要讲解Redis内存优化技巧，Redis的数据特性是“all in memory”， 优化内存将变得非常重要。对于内存优化建议读者先要掌握Redis内存存储 的特性比如字符串、压缩编码、整数集合等，再根据数据规模和所用命令需 求去调整，从而达到空间和效率的最佳平衡。

建议使用Redis存储大量数据 时，把内存优化环节加入到前期设计阶段，否则数据大幅增长后，开发人员 需要面对重新优化内存所带来开发和数据迁移的双重成本。当Redis内存不 足时，首先考虑的问题不是加机器做水平扩展，应该先尝试做内存优化，当 遇到瓶颈时，再去考虑水平扩展。即使对于集群化方案，垂直层面优化也同 样重要，避免不必要的资源浪费和集群化后的管理成本。

> 内存是相对宝贵的资源，通过合理的优化可以有效地降低内存的使 用量，内存优化的思路包括:
>
> - 精简键值对大小，键值字面量精简，使用高效二进制序列化工具。 
> - 使用对象共享池优化小整数对象。 
> - 数据优先使用整数，比字符串类型更节省空间。 
> - 优化字符串使用，避免预分配造成的内存浪费。 
> - 使用ziplist压缩编码优化hash、list等结构，注重效率和空间的平衡。 
> - 使用intset编码优化整数集合。 
> - 使用ziplist编码的hash结构降低小对象链规模。