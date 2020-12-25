## 一、概念

​    JVM 中的垃圾回收是基于 标记-复制、标记-清除和标记-整理三种模式的，那么其中最重要的其实是如何标记，像Serial、Parallel这类的回收器，无论是单线程标记和多线程标记，其本质采用的是暂停用户线程进行全面标记的算法，这种算法的好处就是标记的很干净，而且实现简单，缺点就是标记时间相对很长，导致STW的时间很长。

​    那么后来就有了并发标记，适用于CMS和G1，并发标记的意思就是可以在不暂停用户线程的情况下对其进行标记，那么实现这种并发标记的算法就是三色标记法，三色标记法最大的特点就是可以异步执行，从而可以以中断时间极少的代价或者完全没有中断来进行整个GC。

## 二、基本算法

要找出存活对象，根据可达性分析，从GC Roots开始进行遍历访问，可达的则为存活对象：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy83Nzc5NjA3LTkwZmI1NmQxYjM2MmEwNzQucG5nP2ltYWdlTW9ncjIvYXV0by1vcmllbnQvc3RyaXB8aW1hZ2VWaWV3Mi8yL3cvNTA5?x-oss-process=image/format,png)

*图片链接：https://www.jianshu.com/p/12544c0ad5c1*

最终结果：A/D/E/F/G 可达

我们把遍历对象图**过程**中遇到的对象，按“是否访问过”这个条件标记成以下三种颜色：

- **白色**：尚未被GC访问过的对象，如果全部标记已完成依旧为白色的，称为不可达对象，既垃圾对象。
- **黑色**：本对象已经被GC访问过，且本对象的子引用对象也已经被访问过了。
- **灰色**：本对象已访问过，但是本对象的子引用对象还没有被访问过，全部访问完会变成黑色，属于中间态。

![img](https://img-blog.csdnimg.cn/20200730105400197.gif)

*图片链接：https://www.jianshu.com/p/12544c0ad5c1*

**标记过程：**

1、在GC并发标记刚开始时，所以对象均为白色集合。

2、将所有GCRoots直接引用的对象标记为灰色集合。

3、判断若灰色集合中的对象不存在子引用，则将其放入黑色集合，若存在子引用对象，则将其所有的子引用对象放入灰色集合，当前对象放入黑色集合

4、按照步骤三，以此类推，直至灰色集合中的所有对象变成黑色后，本轮标记完成，且当前白色集合内的对象称为不可达对象，既垃圾对象。

**问题：**由于此过程是在和用户线程并发运行的情况下，对象的引用处于随时可变的情况下，那么就会造成多标和漏标的问题。

**浮动垃圾：本应该被标记为白色的对象，没有被标记，造成该对象可能不会被回收。**

比如E对象在GC扫描D对象时,E还正在被D引用，那么此时E就被标记为灰色，此时业务逻辑的变化，D指向E的引用被置空了，这时候E以及后续子引用本应该被当成垃圾回收，但是此时E已经被标记为灰色，导致E对象以及其子对象没有被及时清理掉，变成了浮动垃圾，还有在并发标记开始后的**新对象**，通常的做法是直接全部**当成黑色**，本轮不会进行清除。这部分对象期间可能会变为垃圾，这也算是浮动垃圾的一部分。

**漏标：灰色对象指向白色对象的引用消失了，然后一个黑色的对象重新引用了白色对象。**

比如：D对象引用E对象，E引用G，此时GC正好处于D已经变成黑色，E处于灰色，G是白色的情况下，此时因为业务逻辑的变化，E不引用G了，D对象引用了G，按照三色标记法看，黑色对象是已完成状态，不可能再去找子引用，所以G就不会变成灰色，这样就会造成白色对象此时正在被线程使用中，但是无法被标记成灰色或者白色，造成一个正在被使用的对象被错误回收。

 

![img](https://img-blog.csdnimg.cn/20200730143812205.png)![img](https://img-blog.csdnimg.cn/20200730144111702.png)

不难分析，漏标只有**同时满足**以下两个条件时才会发生：
**条件一：灰色对象 断开了 白色对象的引用；即灰色对象 原来成员变量的引用 发生了变化。
条件二：黑色对象 重新引用了 该白色对象；即黑色对象 成员变量增加了 新的引用。**

**解决方案：**

**CMS：Incremental Update算法**

当一个白色对象被一个黑色对象引用，将黑色对象重新标记为灰色，让垃圾回收器重新扫描。（破坏条件二）

**G1：SATB（Snapshot At The Beginning）算法**

当**原来成员变量的引用发生变化之前，记录下原来的引用对象，既原始快照**，当B和C之间的引用马上被断掉时，将这个引用记录下来，使GC依旧能够访问到，那样白色就不会漏标。（破坏条件一）

**对比：**

SATB 算法是关注引用的删除。（B->C 的引用）
Incremental Update 算法关注引用的增加。（A->C 的引用）
G1 如果使用Incremental Update 算法，因为变成灰色的成员还要重新扫，重新再来一遍，效率太低了。所以G1 在处理并发标记的过程比CMS 效率要高，这个主要是解决漏标的算法决定的。

 

## 三、跨代引用

堆空间通常被划分为新生代和老年代。由于新生代的垃圾收集通常很频繁，如果老年代对象引用了新生代的对象，那么回收新生代的话，需要跟踪从老年代到新生代的所有引用，所以要避免每次YGC 时扫描整个老年代，减少开销。

**Rset（记忆集）**

RSet记录了其他Region中的对象引用本Region中对象的关系，属于points-into结构（谁引用了我的对象）。RSet的价值在于使得垃圾收集器不需要扫描整个堆找到谁引用了当前分区中的对象，只需要扫描RSet即可。

**CardTable（卡表）**

![img](https://img-blog.csdnimg.cn/20200730154253827.png)

卡表的意思就是将一个Region区分为若干个card，组成的一张card表，可以将其理解为数组card[i]，Rset的存储结构就相当于hashMap，key值为引用当前Region区的另外一个Region区的内存地址，value就是另外一个Region区中引用本Region区的card的索引值，意思就是GC可以通过Rset快速找到是哪个Region的哪个card对本Region进行了引用。

 

**四、安全点与安全区域**

**安全点：**用户线程暂停，GC 线程要开始工作，但是要**确保用户线程暂停的这行字节码指令是不会导致引用关系的变化**。所以JVM 会在字节码指令中，选一些指令，作为“安全点”，**比如方法调用、循环跳转、异常跳转等，一般是这些指令才会产生安全点**。为什么它叫安全点，是这样的，GC 时要暂停业务线程，并不是抢占式中断（立马把业务线程中断）而是主动是中断。主动式中断是设置一个标志，这个标志是中断标志，各业务线程在运行过程中会不停的主动去轮询这个标志，一旦发现中断标志为True,就会在自己最近的“安全点”上主动中**断挂起。**

**安全区域：**为什么需要安全区域？要是业务线程都不执行（业务线程处于Sleep 或者是Blocked 状态），那么**程序就没办法进入安全点，对于这种情况，就必须引入安全区域**。**安全区域是指能够确保在某一段代码片段之中， 引用关系不会发生变化**，因此，在这个区域中任意地方开始垃圾收集都是安全的。我们也可以把安全区城看作被扩展拉伸了的安全点。

![img](https://img-blog.csdnimg.cn/20200730163029816.png)

 

## 五、低延迟的垃圾回收器

**垃圾回收器三项指标**：传统的垃圾回收器一般情况下内存占用、吞吐量、延时只能同时满足两个。但是现在的发展，延迟这项的目标越来越重要。所以就有低延迟的垃圾回收器。

Eplison（了解即可）：这个垃圾回收器不能进行垃圾回收，是一个“不干活”的垃圾回收器，由RedHat 退出，它还要负责堆的管理与布局、对象的分配、与解释器的协作、与编译器的协作、与监控子系统协作等职责，主要用于需要剥离垃圾收集器影响的性能测试和压力测试。

ZGC（了解即可）：有类似于G1 的Region，但是没有分代。
标志性的设计是染色指针ColoredPointers（这个概念了解即可），染色指针有4TB 的内存限制，但是效率极高，它是一种将少量额外的信息存储在指针上的技术。它可以做到几乎整个收集过程全程可并发，短暂的STW 也只与GC Roots 大小相关而与堆空间内存大小无关，因此考科一实现任何堆空间STW 的时间小于十毫秒的目标。

Shenandoah（了解即可）：第一款非Oracle 公司开发的垃圾回收器，有类似于G1 的Region，但是没有分代。也用到了染色指针ColoredPointers。效率没有ZGC 高，大概几十毫秒的目标。

 

## 六、GC参数整理

![img](https://img-blog.csdnimg.cn/20200730163428355.png)

**GC 常用参数**
-Xmn -Xms -Xmx –Xss 年轻代最小堆最大堆栈空间
-XX:+UseTLAB 使用TLAB，默认打开
-XX:+PrintTLAB 打印TLAB 的使用情况
-XX:TLABSize 设置TLAB 大小

-XX:+DisableExplicitGC 启用用于禁用对的调用处理的选项System.gc()
-XX:+PrintGC 查看GC 基本信息
-XX:+PrintGCDetails 查看GC 详细信息
-XX:+PrintHeapAtGC 每次一次GC 后，都打印堆信息
-XX:+PrintGCTimeStamps 启用在每个GC 上打印时间戳的功能
-XX:+PrintGCApplicationConcurrentTime 打印应用程序时间(低)
-XX:+PrintGCApplicationStoppedTime 打印暂停时长（低）
-XX:+PrintReferenceGC 记录回收了多少种不同引用类型的引用（重要性低）
-verbose:class 类加载详细过程
-XX:+PrintVMOptions 可在程序运行时，打印虚拟机接受到的命令行显示参数
-XX:+PrintFlagsFinal -XX:+PrintFlagsInitial 打印所有的JVM 参数、查看所有JVM 参数启动的初始值（必须会用）
-XX:MaxTenuringThreshold 升代年龄，最大值15, 并行（吞吐量）收集器的默认值为15，而CMS 收集器的默认值为6。


Parallel 常用参数
-XX:SurvivorRatio 设置伊甸园空间大小与幸存者空间大小之间的比率。默认情况下，此选项设置为8
-XX:PreTenureSizeThreshold 大对象到底多大，大于这个值的参数直接在老年代分配
-XX:MaxTenuringThreshold 升代年龄，最大值15, 并行（吞吐量）收集器的默认值为15，而CMS 收集器的默认值为6。
-XX:+ParallelGCThreads 并行收集器的线程数，同样适用于CMS，一般设为和CPU 核数相同
-XX:+UseAdaptiveSizePolicy 自动选择各区大小比例


CMS 常用参数
-XX:+UseConcMarkSweepGC 启用CMS 垃圾回收器
-XX:+ParallelGCThreads 并行收集器的线程数，同样适用于CMS，一般设为和CPU 核数相同

-XX:CMSInitiatingOccupancyFraction 使用多少比例的老年代后开始CMS 收集，默认是68%(近似值)，如果频繁发生SerialOld 卡顿，应该调小，（频繁CMS 回收）
-XX:+UseCMSCompactAtFullCollection 在FGC 时进行压缩
-XX:CMSFullGCsBeforeCompaction 多少次FGC 之后进行压缩
-XX:+CMSClassUnloadingEnabled 使用并发标记扫描（CMS）垃圾收集器时，启用类卸载。默认情况下启用此选项。
-XX:CMSInitiatingPermOccupancyFraction 达到什么比例时进行Perm 回收，JDK 8 中不推荐使用此选项，不能替代。
-XX:GCTimeRatio 设置GC 时间占用程序运行时间的百分比（不推荐使用）
-XX:MaxGCPauseMillis 停顿时间，是一个建议时间，GC 会尝试用各种手段达到这个时间，比如减小年轻代


G1 常用参数
-XX:+UseG1GC 启用CMS 垃圾收集器
-XX:MaxGCPauseMillis 设置最大GC 暂停时间的目标（以毫秒为单位）。这是一个软目标，并且JVM 将尽最大的努力（G1 会尝试调整Young 区的块数来）来实现它。默认情况下，没有最大暂停时间值。
-XX:GCPauseIntervalMillis GC 的间隔时间
-XX:+G1HeapRegionSize 分区大小，建议逐渐增大该值，1 2 4 8 16 32。随着size 增加，垃圾的存活时间更长，GC 间隔更长，但每次GC 的时间也会更长
-XX:G1NewSizePercent 新生代最小比例，默认为5%
-XX:G1MaxNewSizePercent 新生代最大比例，默认为60%
-XX:GCTimeRatioGC 时间建议比例，G1 会根据这个值调整堆空间
-XX:ConcGCThreads 线程数量
-XX:InitiatingHeapOccupancyPercent 启动G1 的堆空间占用比例，根据整个堆的占用而触发并发GC 周期

 