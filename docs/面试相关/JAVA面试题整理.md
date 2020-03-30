https://blog.csdn.net/u014042066/article/details/77584668

# JAVA基础

1. JAVA中的几种基本数据类型是什么，各自占用多少字节。
2. String类能被继承吗，为什么。
3. String，StringBuffer，StringBuilder的区别。
4. ArrayList和LinkedList有什么区别。
5. 讲讲类的实例化顺序，比如父类静态数据，构造函数，字段，子类静态数据，构造函数，字段，当new的时候，他们的执行顺序。
6. 用过哪些Map类，都有什么区别，HashMap是线程安全的吗,并发下使用的Map是什么，他们内部原理分别是什么，比如存储方式，hashcode，扩容，默认容量等。
7. JAVA8的ConcurrentHashMap为什么放弃了分段锁，有什么问题吗，如果你来设计，你如何设计。
8. 有没有有顺序的Map实现类，如果有，他们是怎么保证有序的。
9. 抽象类和接口的区别，类可以继承多个类么，接口可以继承多个接口么,类可以实现多个接口么。
10. 继承和聚合的区别在哪。
11. IO模型有哪些，讲讲你理解的nio ，他和bio，aio的区别是啥，谈谈reactor模型。
12. 反射的原理，反射创建类实例的三种方式是什么。
13. 反射中，Class.forName和ClassLoader区别 。
14. 描述动态代理的几种实现方式，分别说出相应的优缺点。
15. 动态代理与cglib实现的区别。
16. 为什么CGlib方式可以对接口实现代理。
17. final的用途。
18. 写出三种单例模式实现 。
19. 如何在父类中为子类自动完成所有的hashcode和equals实现？这么做有何优劣。
20. 请结合OO设计理念，谈谈访问修饰符public、private、protected、default在应用设计中的作用。
21. 深拷贝和浅拷贝区别。
22. 数组和链表数据结构描述，各自的时间复杂度。
23. error和exception的区别，CheckedException，RuntimeException的区别。
24. 请列出5个运行时异常。
25. 在自己的代码中，如果创建一个java.lang.String类，这个类是否可以被类加载器加载？为什么。
26. 说一说你对java.lang.Object对象中hashCode和equals方法的理解。在什么场景下需要重新实现这两个方法。
28. 在jdk1.5中，引入了泛型，泛型的存在是用来解决什么问题。
29. 这样的a.hashcode() 有什么用，与a.equals(b)有什么关系。
30. 有没有可能2个不相等的对象有相同的hashcode。
31. Java中的HashSet内部是如何工作的。
32. 什么是序列化，怎么序列化，为什么序列化，反序列化会遇到什么问题，如何解决。
33. java8的新特性。

# JVM知识

1. **什么情况下会发生栈内存溢出（StackOverflowError）。**

   答：

   栈是线程私有的，他的生命周期与线程相同，每个方法在执行的时候都会创建一个栈帧，用来存储局部变量表，操作数栈，动态链接，方法出口灯信息。局部变量表又包含基本数据类型，对象引用类型（局部变量表编译器完成，运行期间不会变化）

   所以我们可以理解为栈溢出就是方法执行是创建的栈帧超过了栈的深度。那么最有可能的就是方法递归调用产生这种结果

2. **什么情况下会发生堆内存溢出（OutOfMemoryError:java heap space）。**

   答：

   heap space表示堆空间，堆中主要存储的是对象。如果不断的new对象则会导致堆中的空间溢出

3. **JVM的内存结构，Eden和Survivor比例。**

   答：

   Java1.7及以下  新生区Young/New、老年区Old/ Tenure、永久区Perm

   Java1.8及以上  新生区Young/New、老年区Old/ Tenure、元空间MetaSpace

   新生区又分为：Eden区、S0、S1，空间比例默认8:1:1

4. **JVM内存为什么要分成新生代，老年代，持久代。新生代中为什么要分为Eden和Survivor。**

   答：

   如果没有Survivor，Eden区每进行一次Minor GC，存活的对象就会被送到老年代。老年代很快被填满，触发Major GC（因为Major GC一般伴随着Minor GC，也可以看做触发了Full GC）。老年代的内存空间远大于新生代，进行一次Full GC消耗的时间比Minor GC长得多。

   那我们来想想在没有Survivor的情况下，有没有什么解决办法，可以避免上述情况：

   | 方案           | 优点                                        | 缺点                                                      |
   | -------------- | ------------------------------------------- | --------------------------------------------------------- |
   | 增加老年代空间 | 更多存活对象才能填满老年代。降低Full GC频率 | 随着老年代空间加大，一旦发生Full GC，执行所需要的时间更长 |
   | 减少老年代空间 | Full GC所需时间减少                         | 老年代很快被存活对象填满，Full GC频率增加                 |

   显而易见，没有Survivor的话，上述两种解决方案都不能从根本上解决问题。

   我们可以得到第一条结论：**Survivor的存在意义，就是减少被送到老年代的对象，进而减少Full GC的发生，Survivor的预筛选保证，只有经历`GC的分代年龄`次Minor GC还能在新生代中存活的对象，才会被送到老年代**

   （为什么要分两个survivor区？https://blog.csdn.net/qq_35181209/article/details/78033329）

5. **JVM中一次完整的GC流程是怎样的，对象如何晋升到老年代。**

   答：对象优先在新生代区中分配，若没有足够空间，Minor GC； 

   大对象（需要大量连续内存空间）直接进入老年态；

   长期存活的对象进入老年态。

   如果对象在新生代出生并经过第一次MGC后仍然存活，年龄+1（GC分代年龄），若年龄超过一定限制（默认是 15 岁，可以通过参数 `-XX:MaxTenuringThreshold` 来设定），则被晋升到老年态。

   ![img](https://upload-images.jianshu.io/upload_images/3347487-b351d2225c627850.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/702/format/webp)

6. **JVM内存分配策略，空间分配担保、动态对象年龄**

   答：

   • 优先分配Eden区

   • 大对象直接分配到老年代

   ​	-XX:PretenureSizeThreshold

   • 长期存活的对象分配老年代（多次Minor GC，熬过了年龄限制）

   ​	-XX:MaxTenuringThreshold=15

   • 空间分配担保

   ​	-XX:+HandlePromotionFailure 

   > 检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小。
   >
   > 如果大于，则正常进行一次YGC，尽管有风险（因为判断的是平均大小，有可能这次的晋升对象比平均值大很多）；
   >
   > 如果小于，或者HandlePromotionFailure设置不允许空间分配担保，这时要进行一次FGC。

   • 动态对象年龄

   > ​	虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄。

7. **你知道哪几种垃圾收集器，各自的优缺点，重点讲下cms和G1，包括原理，流程，优缺点。**

   答：

   一、常见垃圾收集算法
   1、标记-清除算法：首先标记出需要回收的对象，在标记完成后统一回收所有被标记的对象 
   2、复制算法：将可用内存按容量分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉 
   3、标记-整理算法：标记存活的对象都向一端移动，然后直接清理掉端边界以外的内存 
   4、分代收集算法：根据各个年代的特点采用最合适的收集算法

   二、垃圾收集器

   | 垃圾收集器                      | 使用位置          | 使用算法          | 可配合收集器       | 流程                                       | 优点                                                         | 缺点                                                         |
   | ------------------------------- | ----------------- | ----------------- | ------------------ | ------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | Serial（串行）                  | 新生代            | 复制算法          | CMS                |                                            | 单线程、Client模式下默认新生代收集器                         |                                                              |
   | ParNew（并行）                  | 新生代            | 复制算法          | CMS                |                                            | Serial的多线程版本、Server模式下默认收集器、默认线程数=CPU数量 |                                                              |
   | Parallel Scavenge（吞吐量优先） | 新生代            | 复制算法          |                    |                                            | 多线程**（并行回收）**、目标关注吞吐量                       |                                                              |
   | Serial Old                      | 老年代            | 标记-整理         |                    |                                            | Serial的老年代版本、单线程、Client模式下使用                 |                                                              |
   | Parallel Scavenge Old           | 老年代            | 标记-整理         |                    |                                            | Parallel Scavenge的老年代版本、多线程、关注吞吐量            |                                                              |
   | **CMS，Concurrent Mark Sweep**  | **老年代**        | **标记-清除**     | **Serial、ParNew** | **初始标记、并发标记、重新标记、并发清除** | **并发低停顿、关注最短回收停顿时间**                         | **占用CPU资源、无法处理浮动垃圾、出现Concurrent Mode Failure、空间碎片** |
   | **G1**                          | **新生代+老年代** | **复制+标记整理** |                    | **初始标记、并发标记、最终标记、筛选回收** | **并行与并发、分代收集、空间整理（整体基于标记整理，局部（两个Region见）基于复制）、可预测的停顿** |                                                              |

   1）吞吐量=运行用户代码时间／（运行用户代码时间+垃圾收集时间） 

   2）停顿时间短则响应速度好提升用户体验；高吞吐量则CPU利用率高，适合后台运算

   3）可预测的停顿（G1处处理追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经实现Java（RTSJ）的来及收集器的特征）

   博文https://www.cnblogs.com/chengxuyuanzhilu/p/7088316.html

8. **垃圾回收算法的实现原理。**

   答：

   <http://www.importnew.com/13493.html>

9. **当出现了内存溢出，你怎么排错。**

   答：`https://blog.csdn.net/make__It/article/details/86522550`

   确定是哪个内存区域溢出：堆还是栈，是年轻代，还是老年代：通过jstat -gcutil在线看（如果年轻代都内存不断上升，并且100%，minor gc， 但是还是不释放的场景）

   1. **堆溢出**

      堆用于储存对象实例，只要不断的创建对象或者对象无法被回收，达到一定程度必然会产生OOM。产生原因可能有2种情况：

      1. 存在内存泄漏
      2. 堆分配得到的内存过小，异常信息会进一步提示“Java Heap Space”。

      解决方法：2种情况有不同解决方法，因此首先我们需要分清是哪一个原因导致了OOM。

      通过参数`-XX:HeapDumpOnOutOfMemoryError`可以让jvm在OOM时Dump出当前的内存堆快照，利用内存映像分析工具（如Eclipse Memory Analyzer）进行查看。

      如果是内存泄漏，通过工具查看泄漏对象，定位到对象位置，进行修正。如果不存在泄漏，即就是内存中的对象确实还必须活着，那就应当检查堆参数（`-Xmx` 和`-Xms`）是否过小，针对当前机器是否可以调大。

   2. **栈溢出**
      针对栈溢出，java虚拟机规范中描述了2中异常

      StackOverflowError：线程请求的栈深度大于虚拟机所允许的最大深度。

      OutOfMemoryError：虚拟机在扩展栈时无法申请到足够的空间。异常信息会进一步提示“。。。。create thread”

      但其实这2种情况描述的是一个事情，即到底是内存太小，还是已经使用的栈空间太大，无法再申请到栈空间。不同的是，OOM在多线程更容易出现。

      解决：StackOverflowError：通过-Xss参数增加栈内存容量（指单个栈容量） 。注意是否有太多的本地变量被定义，减少定义。

      ```
      OutOfMemoryError：通过减少最大堆（增加总的栈容量）或减少单个栈容量。此处为何说发生了OOM还要减少单个栈容量呢，多线程OOM发生的原因是因为每个线程申请一个栈，减少单个容量来增加可申请栈的总数，以此避免OOM。
      ```

   3. **方法区和运行时常量池溢出**
      大量的类或者大量的字面量被定义时会导致方法区和运行时常量池OOM，但出现的可能性比堆OOM要低。出现时会进一步提示“PermGen space”。（PS：Java8以下，Java8开始元空间取代永久代，不会出现该异常）

      解决方法：通过`-XX:PermSize` 和 `-XX:MaxPermSize`限制方法区大小

   4. **本机直接内存溢出**
      程序中直接或者间接的使用了nio，通过DirectByteBuffer请求内存，内存不足时也有可能产生OOM。明显特征是出现OOM时dump下来的文件很小，没有明显的异常信息。

      解决方法： 通过`-XX：MaxDirectMemorySize`可定义直接内存大小。

      > ByteBuffer有两种:
      >
      > **heap ByteBuffer -> -XX:Xmx**
      >
      > 1.一种是heap ByteBuffer,该类对象分配在JVM的堆内存里面，直接由Java虚拟机负责垃圾回收，
      >
      > **direct ByteBuffer -> -XX:MaxDirectMemorySize**
      >
      > 2.一种是direct ByteBuffer是通过jni在虚拟机外内存中分配的。通过[jmap](https://www.baidu.com/s?wd=jmap&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)无法查看该快内存的使用情况。只能通过top来看它的内存使用情况。
      >
      > **JVM堆内存大小可以通过-Xmx来设置，同样的direct ByteBuffer可以通过-XX:MaxDirectMemorySize来设置，此参数的含义是当Direct ByteBuffer分配的堆外内存到达指定大小后，即触发Full GC。注意该值是有上限的，默认是64M，最大为sun.misc.VM.maxDirectMemory()，在程序中中可以获得-XX:MaxDirectMemorySize的设置的值。**

10. **JVM内存模型的相关知识了解多少，比如重排序，内存屏障，happen-before，主内存，工作内存等。**

    答：

    内存屏障：为了保障执行顺序和可见性的一条cpu指令

    重排序：为了提高性能，编译器和处理器会对执行进行重拍

    happen-before：操作间执行的顺序关系。有些操作先发生

    主内存：共享变量存储的区域即是主内存

    工作内存：每个线程copy的本地内存，存储了该线程以读/写共享变量的副本

    1. [Java 劝退系列：内存模型 volatile 必会必知](http://mp.weixin.qq.com/s?__biz=MzI1NTI3MzEwMg==&mid=2247484581&idx=1&sn=bfae77d50aa21dbf33685af7b8ff1a3c&chksm=ea393544dd4ebc520aac6c9647fb91f49d5592f233c5c721e967b26ea241e4bf5bfd7e284f06&scene=21#wechat_redirect)
    2. [Java 劝退系列：volatile 实现原理必会必知](http://mp.weixin.qq.com/s?__biz=MzI1NTI3MzEwMg==&mid=2247484586&idx=1&sn=dbb3b752c25ca0b2e249451ab629c183&chksm=ea39354bdd4ebc5d7ef147f5ad505f96d9ae7b80e2822f0da5a28a798b6c78fb182e021e06d8&scene=21#wechat_redirect)
    3. [Java 劝退系列：内存模型重排序必会必知](http://mp.weixin.qq.com/s?__biz=MzI1NTI3MzEwMg==&mid=2247484590&idx=1&sn=f9855369deaaa8343ae4f25bdb39f715&chksm=ea39354fdd4ebc59b37654bb88aecf2f070cac4bec35266fe70c923add77dc37a981cf698bdd&scene=21#wechat_redirect)
    4. [Java 劝退系列：内存模型 happens-before 必会必知](http://mp.weixin.qq.com/s?__biz=MzI1NTI3MzEwMg==&mid=2247484606&idx=2&sn=0d08d6a3b15fe694f012f8a1a46553d8&chksm=ea39355fdd4ebc491a42c5fcb01defa8904117fae983ae1ecb93421d18480e3ae647764ad3f1&scene=21#wechat_redirect)
    5. [面试必会必知：Java 内存模型总结](https://mp.weixin.qq.com/s?__biz=MzI1NTI3MzEwMg==&amp;mid=2247484701&amp;idx=1&amp;sn=f11e1f164ba48d6fea5370510d1570dd&amp;chksm=ea3934fcdd4ebdeab0cf0421327147815d6f15a2c5cafabbd1213129ca765e7fe4227004868b&amp;scene=0&amp;key=05ef868c6b7e1181aba49af403c2acd37c89e131e8468095ababe5d75fa9482f6a25a11826882906d2059498a4c869f41f7fba3932ae4a403f4b48e31e3bfee018e49a041a3c0a7ee1b04f7f73cde8b2&amp;ascene=1&amp;uin=Mjg1NjQyOTE2MA%3D%3D&amp;devicetype=Windows+10&amp;version=62060720&amp;lang=zh_CN&amp;pass_ticket=PIuliO4yIh8I%2Bo2A%2BcHhx1goayDfSAz3RyqsJH%2B2aiEwh%2Fr74ZC4hscOBEp%2BkTCS )

11. **简单说说你了解的类加载器，可以打破双亲委派么，怎么打破。**

    答：

    类加载器的分类（bootstrap,extension,app,curstom）

    类加载的流程：装载（Load），链接（Link），初始化(Initialize)

    https://blog.csdn.net/gjanyanlig/article/details/6818655/

    # [【JVM】浅谈双亲委派和破坏双亲委派](https://www.cnblogs.com/joemsu/p/9310226.html)

12. 讲讲JAVA的反射机制。

    答：

    Java程序在运行状态可以动态的获取类的所有属性和方法，并实例化该类，调用方法的功能

    http://www.importnew.com/23560.html

13. 你们线上应用的JVM参数有哪些。

    答：

    > JVM虚拟机参数配置官方文档
    >
    > JDK8 <https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html>
    >
    > JDK7 <https://docs.oracle.com/javase/7/docs/technotes/tools/solaris/java.html>
    >
    > 官方博客 <https://blogs.oracle.com/poonam/>

    ```js
    -server #Server模式启动
    -Xms6000M #堆初始大小
    -Xmx6000M #堆最大大小  The -Xmx option is equivalent to -XX:MaxHeapSize.
    -Xss1024k #每个线程栈空间1024k
    -Xmn500M  
    -XX:PermSize=500M  #永久代初始大小
    -XX:MaxPermSize=500M  #永久代最大大小
    -XX:SurvivorRatio=65536
    -XX:MaxTenuringThreshold=15  #GC分代对象年龄阈值（最大15） parallel 收集器默认15，CMS默认6
    -Xnoclassgc  #禁用垃圾收集器
    -XX:+DisableExplicitGC#启用禁用对System.gc()调用的处理的选项。默认情况下禁用此选项，这意味着将处理对System.gc()的调用。如果禁用了对System.gc()的调用处理，JVM仍然在必要时执行GC。
    -XX:+UseParNewGC #使用ParNew收集器
    -XX:+UseConcMarkSweepGC #使用CMS收集器
    -XX:+CMSClassUnloadingEnabled #在使用并发标记-清除(CMS)垃圾收集器时启用类卸载。默认情况下启用此选项。要禁用CMS垃圾收集器的类卸载，请指定-XX:-CMSClassUnloadingEnabled
    
    -XX:+HeapDumpOnOutOfMemoryError #让jvm在OOM时Dump出当前的内存堆快照
    -XX:HeapDumpPath=path 			#路径
    #-XX:HeapDumpPath=./java_pid%p.hprof
    #-XX:HeapDumpPath=/var/log/java/java_heapdump.hprof
    
    -XX:+PrintClassHistogram #设置此选项相当于运行jmap -histo命令或jcmd pid GC。class_histogram命令，其中pid是当前Java进程标识符。
    -XX:+PrintGCDetails #支持在每次GC上打印详细的消息。
    -XX:+PrintGCTimeStamps #支持在每次GC时打印时间戳。
    
    -XX:TargetSurvivorRatio #设定survivor区的目标使用率。默认50，即survivor区对象目标使用率为50%。	
    
    -Xloggc:log/gc.log
    ```

14. g1和cms区别,吞吐量优先和响应优先的垃圾收集器选择。

    答：

    Cms是以获取最短回收停顿时间为目标的收集器。基于标记-清除算法实现。比较占用cpu资源，切易造成碎片。
    G1是面向服务端的垃圾收集器，是jdk9默认的收集器，并行与并发、分代收集、空间整合（整体基于标记整理，局部（两个Region见）基于复制））、可预测的停顿。

15. 怎么打出线程栈信息。

​	答：[jstack-查看Java进程的线程堆栈信息，锁定高消耗资源代码。](https://www.cnblogs.com/zhuqq/p/5938187.html)

​	jstack主要用来查看某个Java进程内的线程堆栈信息。

# 开源框架知识

1. 简单讲讲tomcat结构，以及其类加载器流程，线程模型等。
2. tomcat如何调优，涉及哪些参数 。
3. 讲讲Spring加载流程。
4. Spring AOP的实现原理。
5. 讲讲Spring事务的传播属性。
6. Spring如何管理事务的。
7. Spring怎么配置事务（具体说出一些关键的xml 元素）。
8. 说说你对Spring的理解，非单例注入的原理？它的生命周期？循环注入的原理，aop的实现原理，说说aop中的几个术语，它们是怎么相互工作的。
10. Springmvc 中DispatcherServlet初始化过程。
11. netty的线程模型，netty如何基于reactor模型上实现的。
12. 为什么选择netty。
13. 什么是TCP粘包，拆包。解决方式是什么。
14. netty的fashwheeltimer的用法，实现原理，是否出现过调用不够准时，怎么解决。
15. netty的心跳处理在弱网下怎么办。
16. netty的通讯协议是什么样的。
17. springmvc用到的注解，作用是什么，原理。
18. springboot启动机制。

# 操作系统

1. Linux系统下你关注过哪些内核参数，说说你知道的。
2. Linux下IO模型有几种，各自的含义是什么。
3. epoll和poll有什么区别。
4. 平时用到哪些Linux命令。
5. 用一行命令查看文件的最后五行。
6. 用一行命令输出正在运行的java进程。
7. 介绍下你理解的操作系统中线程切换过程。
8. 进程和线程的区别。
9. top 命令之后有哪些内容，有什么作用。
10. 线上CPU爆高，请问你如何找到问题所在。

# 多线程

1. 多线程的几种实现方式，什么是线程安全。
2. volatile的原理，作用，能代替锁么。
3. 画一个线程的生命周期状态图。
4. sleep和wait的区别。
5. sleep和sleep(0)的区别。
6. Lock与Synchronized的区别 。
7. synchronized的原理是什么，一般用在什么地方(比如加在静态方法和非静态方法的区别，静
8. 态方法和非静态方法同时执行的时候会有影响吗)，解释以下名词：重排序，自旋锁，偏向锁，轻量级锁，可重入锁，公平锁，非公平锁，乐观锁，悲观锁。
10. 用过哪些原子类，他们的原理是什么。
11. JUC下研究过哪些并发工具，讲讲原理。
12. 用过线程池吗，如果用过，请说明原理，并说说newCache和newFixed有什么区别，构造函数的各个参数的含义是什么，比如coreSize，maxsize等。
14. 线程池的关闭方式有几种，各自的区别是什么。
15. 假如有一个第三方接口，有很多个线程去调用获取数据，现在规定每秒钟最多有10个线程同
16. 时调用它，如何做到。
17. spring的controller是单例还是多例，怎么保证并发的安全。
18. 用三个线程按顺序循环打印abc三个字母，比如abcabcabc。
19. ThreadLocal用过么，用途是什么，原理是什么，用的时候要注意什么。
20. 如果让你实现一个并发安全的链表，你会怎么做。
21. 有哪些无锁数据结构，他们实现的原理是什么。
22. 讲讲java同步机制的wait和notify。
23. CAS机制是什么，如何解决ABA问题。
24. 多线程如果线程挂住了怎么办。
25. countdowlatch和cyclicbarrier的内部原理和用法，以及相互之间的差别(比如countdownlatch的await方法和是怎么实现的)。
27. 对AbstractQueuedSynchronizer了解多少，讲讲加锁和解锁的流程，独占锁和公平所加锁有什么不同。
29. 使用synchronized修饰静态方法和非静态方法有什么区别。
30. 简述ConcurrentLinkedQueue和LinkedBlockingQueue的用处和不同之处。
31. 导致线程死锁的原因？怎么解除线程死锁。
32. 非常多个线程（可能是不同机器），相互之间需要等待协调，才能完成某种工作，问怎么设计这种协调方案。
33. 用过读写锁吗，原理是什么，一般在什么场景下用。
34. 开启多个线程，如果保证顺序执行，有哪几种实现方式，或者如何保证多个线程都执行完再拿到结果。
36. 延迟队列的实现方式，delayQueue和时间轮算法的异同。
37. 点击这里有一套答案版的多线程试题。

# TCP与HTTP

1. http1.0和http1.1有什么区别。
2. TCP三次握手和四次挥手的流程，为什么断开连接要4次,如果握手只有两次，会出现什么。
3. TIME_WAIT和CLOSE_WAIT的区别。
4. 说说你知道的几种HTTP响应码，比如200, 302, 404。
5. 当你用浏览器打开一个链接（如：[http://www.javastack.cn](http://link.zhihu.com/?target=http%3A//www.javastack.cn)）的时候，计算机做了哪些工作步骤。
6. TCP/IP如何保证可靠性，说说TCP头的结构。
7. 如何避免浏览器缓存。
8. 如何理解HTTP协议的无状态性。
9. 简述Http请求get和post的区别以及数据包格式。
10. HTTP有哪些method
11. 简述HTTP请求的报文格式。
12. HTTP的长连接是什么意思。
13. HTTPS的加密方式是什么，讲讲整个加密解密流程。
14. Http和https的三次握手有什么区别。
15. 什么是分块传送。
16. Session和cookie的区别。
17. 点击这里有一套答案版的试题。

# 架构设计与分布式

1. 用java自己实现一个LRU。
2. 分布式集群下如何做到唯一序列号。
3. 设计一个秒杀系统，30分钟没付款就自动关闭交易。
4. 如何使用redis和zookeeper实现分布式锁？有什么区别优缺点，会有什么问题，分别适用什么

   场景。（延伸：如果知道redlock，讲讲他的算法实现，争议在哪里）
6. 如果有人恶意创建非法连接，怎么解决。
7. 分布式事务的原理，优缺点，如何使用分布式事务，2pc 3pc 的区别，解决了哪些问题，还有
8. 哪些问题没解决，如何解决，你自己项目里涉及到分布式事务是怎么处理的。
9. 什么是一致性hash。
10. 什么是restful，讲讲你理解的restful。
11. 如何设计一个良好的API。
12. 如何设计建立和保持100w的长连接。
13. 解释什么是MESI协议(缓存一致性)。
14. 说说你知道的几种HASH算法，简单的也可以。
15. 什么是paxos算法， 什么是zab协议。
16. 一个在线文档系统，文档可以被编辑，如何防止多人同时对同
17. 一份文档进行编辑更新。
18. 线上系统突然变得异常缓慢，你如何查找问题。
19. 说说你平时用到的设计模式。
20. Dubbo的原理，有看过源码么，数据怎么流转的，怎么实现集群，负载均衡，服务注册
21. 和发现，重试转发，快速失败的策略是怎样的 。
22. 一次RPC请求的流程是什么。
23. 自己实现过rpc么，原理可以简单讲讲。Rpc要解决什么问题。
24. 异步模式的用途和意义。
25. 编程中自己都怎么考虑一些设计原则的，比如开闭原则，以及在工作中的应用。
26. 设计一个社交网站中的“私信”功能，要求高并发、可扩展等等。 画一下架构图。
27. MVC模式，即常见的MVC框架。
28. 聊下曾经参与设计的服务器架构并画图，谈谈遇到的问题，怎么解决的。
29. 应用服务器怎么监控性能，各种方式的区别。
30. 如何设计一套高并发支付方案，架构如何设计。
31. 如何实现负载均衡，有哪些算法可以实现。
32. Zookeeper的用途，选举的原理是什么。
33. Zookeeper watch机制原理。
34. Mybatis的底层实现原理。
35. 请思考一个方案，实现分布式环境下的countDownLatch。
36. 后台系统怎么防止请求重复提交。
37. 描述一个服务从发布到被消费的详细过程。
38. 讲讲你理解的服务治理。
39. 如何做到接口的幂等性。
40. 如何做限流策略，令牌桶和漏斗算法的使用场景。
41. 什么叫数据一致性，你怎么理解数据一致性。
42. 分布式服务调用方，不依赖服务提供方的话，怎么处理服务方挂掉后，大量无效资源请求的浪费，如果只是服务提供方吞吐不高的时候该怎么做，如果服务挂了，那么一会重启，该怎么做到最小的资源浪费，流量半开的实现机制是什么。
45. dubbo的泛化调用怎么实现的，如果是你，你会怎么做。
46. 远程调用会有超时现象，如果做到优雅的控制，JDK自带的超时机制有哪些，怎么实现的。

# 算法

1. 10亿个数字里里面找最小的10个。
2. 有1亿个数字，其中有2个是重复的，快速找到它，时间和空间要最优。
3. 2亿个随机生成的无序整数,找出中间大小的值。
4. 给一个不知道长度的（可能很大）输入字符串，设计一种方案，将重复的字符排重。
5. 遍历二叉树。
6. 有3n+1个数字，其中3n个中是重复的，只有1个是不重复的，怎么找出来。
7. 写一个字符串（如：[http://www.javastack.cn](http://link.zhihu.com/?target=http%3A//www.javastack.cn)）反转函数。
8. 常用的排序算法，快排，归并、冒泡。 快排的最优时间复杂度，最差复杂度。冒泡排序的
9. 优化方案。
10. 二分查找的时间复杂度，优势。
11. 一个已经构建好的TreeSet，怎么完成倒排序。
12. 什么是B+树，B-树，列出实际的使用场景。
13. 一个单向链表，删除倒数第N个数据。
14. 200个有序的数组，每个数组里面100个元素，找出top20的元素。
15. 单向链表，查找中间的那个元素。

# 数据库知识

1. 数据库隔离级别有哪些，各自的含义是什么，MYSQL默认的隔离级别是是什么。
2. 什么是幻读。
3. MYSQL有哪些存储引擎，各自优缺点。
4. 高并发下，如何做到安全的修改同一行数据。
5. 乐观锁和悲观锁是什么，INNODB的标准行级锁有哪2种，解释其含义。
6. SQL优化的一般步骤是什么，怎么看执行计划，如何理解其中各个字段的含义。
7. 数据库会死锁吗，举一个死锁的例子，mysql怎么解决死锁。
8. MYsql的索引原理，索引的类型有哪些，如何创建合理的索引，索引如何优化。
9. 聚集索引和非聚集索引的区别。
10. select for update 是什么含义，会锁表还是锁行或是其他。
11. 为什么要用Btree实现，它是怎么分裂的，什么时候分裂，为什么是平衡的。
12. 数据库的ACID是什么。
13. 某个表有近千万数据，CRUD比较慢，如何优化。
14. Mysql怎么优化table scan的。
15. 如何写sql能够有效的使用到复合索引。
16. mysql中in 和exists 区别。
17. 数据库自增主键可能的问题。
18. MVCC的含义，如何实现的。
19. 你做过的项目里遇到分库分表了吗，怎么做的，有用到中间件么，比如sharding jdbc等,他们的原理知道么。
21. MYSQL的主从延迟怎么解决。

# 消息队列

1. 消息队列的使用场景。

2. 消息的重发，补充策略。

3. 如何保证消息的有序性。

4. 用过哪些MQ，和其他mq比较有什么优缺点，MQ的连接是线程安全的吗，你们公司的MQ服务

   架构怎样的。

5. MQ系统的数据如何保证不丢失。

6. rabbitmq如何实现集群高可用。

7. kafka吞吐量高的原因。

8. kafka 和其他消息队列的区别，kafka 主从同步怎么实现。

9. 利用mq怎么实现最终一致性。

10. 使用kafka有没有遇到什么问题，怎么解决的。

11. MQ有可能发生重复消费，如何避免，如何做到幂等。

13. MQ的消息延迟了怎么处理，消息可以设置过期时间么，过期了你们一般怎么处理。

# 缓存

1. 常见的缓存策略有哪些，如何做到缓存(比如redis)与DB里的数据一致性，你们项目中用到了

   什么缓存系统，如何设计的。

2. 如何防止缓存击穿和雪崩。

3. 缓存数据过期后的更新如何设计。

4. redis的list结构相关的操作。

5. Redis的数据结构都有哪些。

6. Redis的使用要注意什么，讲讲持久化方式，内存设置，集群的应用和优劣势，淘汰策略等。

7. redis2和redis3的区别，redis3内部通讯机制。

8. 当前redis集群有哪些玩法，各自优缺点，场景。

9. Memcache的原理，哪些数据适合放在缓存中。

10. redis和memcached 的内存管理的区别。

11. Redis的并发竞争问题如何解决，了解Redis事务的CAS操作吗。

12. Redis的选举算法和流程是怎样的。

13. redis的持久化的机制，aof和rdb的区别。

14. redis的集群怎么同步的数据的。

15. 知道哪些redis的优化操作。

16. Reids的主从复制机制原理。

17. Redis的线程模型是什么。

18. 请思考一个方案，设计一个可以控制缓存总体大小的自动适应的本地缓存。

19. 如何看待缓存的使用（本地缓存，集中式缓存），简述本地缓存和集中式缓存和优缺点。

20. 本地缓存在并发使用时的注意事项。

# 搜索

1. elasticsearch了解多少，说说你们公司es的集群架构，索引数据大小，分片有多少，以及一些
2. 调优手段 。elasticsearch的倒排索引是什么。
3. elasticsearch 索引数据多了怎么办，如何调优，部署。
4. elasticsearch是如何实现master选举的。
5. 详细描述一下Elasticsearch索引文档的过程。
6. 详细描述一下Elasticsearch搜索的过程。
7. Elasticsearch在部署时，对Linux的设置有哪些优化方法？
8. lucence内部结构是什么。