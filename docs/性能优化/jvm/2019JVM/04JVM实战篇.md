# 01 JVM实战篇

## 1.1  JVM参数

### 1.1.1   标准参数

```
-version
-help
-server

-cp
```

![101223221421002](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjte9bb5jlj30pd03fjre.jpg)

### 1.1.2   -X参数

> 非标准参数，也就是在JDK各个版本中可能会变动

```
-Xint     解释执行
-Xcomp   第一次使用就编译成本地代码
-xmixed   混合模式，JVM自己来决定
```

![101223221421001](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjte94y0caj30rx0cydgr.jpg)

### 1.1.3   -XX 参数

**使用得最多的参数类型**

> 非标准化参数，相对不稳定，主要用于JVM调优和Debug

```
a.   Boolean 类型
    格式：-XX：[+-]<name>          +或-表示启用或者禁用name属性
    比如：-xx：+useconcMarksweepGC  表示启用CMS类型的垃圾回收器
				 -xx：+useGlGC           表示启用G1类型的垃圾回收器
b.   非Boolean类型
    格式：-XX<name>=<val ue>表示name属性的值是value
    比如：-XX：MaxGCPauseMillis=500
```

### 1.1.4   其他参数

```
-XmslOOO 等价于 -xx:InitialHeapsize=1000
-Xmx1OOO 等价于 -XX:MaxHeapSi ze=1000 
-Xss100  等价于 -XX:ThreadStackSize=100
```

> 所以这块也相当于是-XX类型的参数

### 1.1.5   查看参数

![101223221421003](/Users/xmly/Documents/gp/2019/jvm/101223221421/101223221421003.Png)

![101223221421004](/Users/xmly/Documents/gp/2019/jvm/101223221421/101223221421004.Png)

> 值得注意的是"="表示默认值表示被用户或VM修改后的值
>
> 要想查看某个进程具体参数的值，可以使用jinfo,这块后面聊 一般要设置参数，可以先查看一下当前参数是什么，然后进行修改
>

### 1.1.6   设置参数的方式

- 开发工具中设置比如IDEA，eclipse


- 运行ja r 包的时候:java -XX:+UseG1GC xxx.ja r


- web容器比如tomcat，可以在脚本中的进行设置


- 通过jinfo实时调整某个java进程的参数（参数只有被标记为manageable的flags可以被实时修改）


### 1.1.7   实践和单位换算 

```
1Byte(字节)=8bit(位) 
1KB=1024Byte(字节) 
1MB=1024KB 
1GB=1024MB 
1TB=1024GB
```

(1)   设置堆内存大小和参数打印

```
-Xmx100M -Xms100M -XX:+PrintFlagsFinal
```

(2)   查询+Pri ntFlagsFi nal 的值

```
:=true
```

(3)   查询堆内存大小MaxHeapSize

```
:= 104857600
```

(4)   换算

```
104857600(Byte)/1024=102400(KB)

102400(KB)/1024=100(MB)
```

(5)   结论

104857600是字节单位

### 1.1.8   常用参数含义

| **参数**                                                     | **含义**                                                     | **说明**                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| -XX:CICompilerCount=3                                        | 最大并行编译数                                               | 如果设置大于1 ，虽然编译速度会提高，但是同样影响系   统稳定性，会增加JVM崩溃的可能 |
| -XX:lni tialHeapSize=100M                                    | 初始化堆大小                                                 | 简写-XmslOOM                                                 |
| -XX:MaxHeapSize=100M                                         | 最大堆大小                                                   | 简写-XmxIOOM                                                 |
| -XX:NewSize=20M                                              | 设置年轻代的大小                                             |                                                              |
| -XX:MaxNewSize=50M                                           | 年轻代最大大小                                               |                                                              |
| -XX:OldSize=50M                                              | 设置老年代大小                                               |                                                              |
| -XX:MetaspaceSize=50M                                        | 设置方法区大小                                               |                                                              |
| -XX:MaxMetaspaceSize=50M                                     | 方法区最大大小                                               |                                                              |
| -XX:+UseParallelGC                                           | 使用 UseParallelGC                                           | 新生代，吞吐量优先                                           |
| -XX:+UseParallelOldGC                                        | 使用 UseParallelOldGC                                        | 老年代，吞吐量优先                                           |
| -XX:+UseC on cMarkSweepGC                                    | 使用CMS                                                      | 老年代，停顿时间优先                                         |
| -XX:+UseG1GC                                                 | 使用G1GC                                                     | 新生代，老年代，停顿时间优先                                 |
| -XX:NewRatio                                                 | 新老生代的比值                                               | 比如-XX:Ratio=4，贝U表示新生代:老年代=1:4，也就是新   生代占整个堆内存的1/5 |
| -XX:SurvivorRatio                                            | 两个S区和Ede n区的比值                                       | 比如-XX:SurvivorRatio=8,也就是(S0+S1):Eden=2:8,  也就是一个S占整个新生代的1/10 |
| -XX:+HeapDumpOnOutOfMemoryError                              | 启动堆内存溢出打印                                           | 当JVM堆内存发生溢出时，也就OOM,自动生成dump文件              |
| -XX:HeapDumpPath=heap.hprof                                  | 指定堆内存溢出打印目录                                       | 表示在当前目录生成一  heap.hprof文件                         |
| XX:+PrintGCDetails  -  XX:+PrintGCTimeStamps  -  XX:+PrintGCDateStamps  Xloggc:$CATALlNA_HOME/logs/gc.log | 打印出GC日志                                                 | 可以使用不同的垃圾收集器，对比查看GC情况                     |
| -Xss128k                                                     | 设置每个线程的堆栈大小                                       | 经验值是3000-5000最佳                                        |
| -XX:MaxTe nuri ngThreshold=6                                 | 提升年老代的最大临界值                                       | 默认值为  15                                                 |
| -XX:lnitiatingHeapOccupancyPercent                           | 启动并发GC周期时堆内存使用占比                               | G1之类的垃圾收集器用它来触发并发GC周期，基于整个堆   的使用率,而不只是某一代内存的使用比. 值为 0 贝表   示”一直执行GC循环".默认值为45. |
| -XX:G1HeapWastePercent                                       | 允许的浪费堆空间的占比                                       | 默认是10%，如果并发标记可回收的空间小于10%,贝不  会触发MixedGC。 |
| -XX:MaxGCPauseMillis=200ms                                   | G1 最大停顿时间                                              | 暂停时间不能太小，太小的话就会导致出现G1跟不上垃   圾产生的速度。最终退化成Full GC。所以对这个参数的  调优是一个持续的过程，逐步调整到最佳状态。 |
| -XX:C on cGCThreads=n                                        | 并发垃圾收集器使用的线程数量                                 | 默认值随JVM运行的平台不同而不同                              |
| -XX:G1MixedGCLiveThresholdPercent=65                         | 混合垃圾回收周期中要包括的旧区域设置 占用率阈值              | 默认占用率为 65%                                             |
| -XX:G1MixedGCCountTarget=8  r                                | 设置标记周期完成后,对存活数据上限为  G1MixedGCLlveThresholdPercent  的旧 区域执行混合垃圾回收的目标次数 | 默认8次混合垃圾回收，混合回收的目标是要控制在此目 标次数以内 |
| XX:G1OldCSetRegionThresholdPercent=1                         | 描述Mixed GC时，Old Region被加入到 CSet 中                   | 默认情况下，G1只把10%的Old Region加入到CSet中                |
|                                                              |                                                              |                                                              |



## 1.2 常用命令    

###   1.2.1 jps  查看java进程  



### 1.2.2   jinfo

(1)     实时查看和调整JVM配置参数

![101223221421006](/Users/xmly/Documents/gp/2019/jvm/101223221421/101223221421006.Png)

(2)     查看

> jinfo -flag name PID 查看某个java进程的name属性的值
>

```
jinfo -flag MaxHeapSize PID
jinfo -flag UseG1GC PID
```

![101223221421007](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjteq0ulihj30k5036q2w.jpg)

(3)     修改

> 参数只有被标记为manageable的flags可以被实时修改
>

```
jinfo -flag [+|-] PID
jinfo -flag = PID
```

(4)     查看曾经赋过值的一些参数

```
jinfo -flags PID
```

![101223221421008](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjter7qt54j31b40c2jsv.jpg)

### 1.2.3   jstat

(1)查看虚拟机性能统计信息



(2)查看类装载信息

```
jstat -class PID 1000 10          查看某个java进程的类装载信息，每1000毫秒输出一次，共输出10次
```

![101223221421009](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjterxnx26j30ir09q0sx.jpg)

(3)查看垃圾收集信息

```
jstat -gc PID 1000 10
```

![101223221421010](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtes4szx5j316u04jaac.jpg)

### 1.2.4   jstack

(1)查看线程堆栈信息

The jstack command prints Java stack traces of Java threads for a specified Java process, core file, or remote debug server.

(2)用法

```
jstack PID
```

![101223221421011](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtet1fyigj31410m2mz6.jpg)

(4)排查死锁案例

• DeadLockDemo

```java
//运行主类
public class DeadLockDemo {
    public static void main(String[] args)
    {
      DeadLock d1=new DeadLock(true); DeadLock d2=new DeadLock(false); Thread t1=new Thread(d1); Thread t2=new Thread(d2); t1.start();
      t2.start(); }
      }
      //定义锁对象 class MyLock{
          public static Object obj1=new Object();
          public static Object obj2=new Object();
      }
      //死锁代码
      class DeadLock implements Runnable{
          private boolean flag;
          DeadLock(boolean flag){
      this.flag=flag; }
          public void run() {
              if(flag) {
      获得obj1锁");
      -if获得obj2锁");
      }
      while(true) { synchronized(MyLock.obj1) {
      System.out.println(Thread.currentThread().getName()+"----if synchronized(MyLock.obj2) {
      System.out.println(Thread.currentThread().getName()+"--- }
      } }
      else {
          while(true){
      -否则获得obj1锁");
      }
      } }
      } }
}
```

•运行结果

Run:DeadLockDemo

![101223221421012](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjteuun5o7j30kq05jt8q.jpg) 

• jstack分析

![101223221421013](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtev6g9hmj311s0fl3zu.jpg)

> 把打印信息拉到最后可以发现
>

![101223221421014](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtevgoxo2j30v50fn3zl.jpg)

### 1.2.5   jmap

⑴生成堆转储快照

The jmap command prints shared object memory maps or heap memory details of a specified process, core file, or remote debug server.

⑵打印出堆内存相关信息

```
-XX：+PrintFlagsFinal -Xms300M -Xmx300M
jmap -heap PID
```

![101223221421015](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtew5dunfj30pf0k6759.jpg)

⑶dump出堆内存相关信息

```
jmap -dump:format=b,file=heap.hprof PID
```

jmap -dump:format=b,file=heap.hprof 44808 ![101223221421016](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjtewo82mej30q504t0ss.jpg)

⑷要是在发生堆内存溢出的时候，能自动dump出该文件就好了

一般在开发中，JVM参数可以加上下面两句，这样内存溢出时，会自动dump出该文件` -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.hprof`

> 设置堆内存大小：`-Xms20M -Xmx20M`
>
> 启动，然后访问localhost: 9090/heap，使得堆内存溢出
>

 (5)关于dump下来的文件

一般dump下来的文件可以结合工具来分析，这块后面再说。

## 1.3常用工具

参数也了解了，命令也知道了，关键是用起来不是很方便，要是有图形化的界面就好了。 一定会有好事之者来做这件事情。

### 1.3.1   jconsole

JConsole工具是JDK自带的可视化监控工具。查看java应用程序的运行概况、监控堆信息、永久区使用 情况、类加载情况等。

> 命令行中输入：jconsole
>

### 1.3.2   jvisualvm

#### 1.3.2.1监控本地Java进程

可以监控本地的java进程的CPU，类，线程等

#### 1.3.2.2监控远端Java进程

比如监控远端tomcat，演示部署在阿里云服务器上的tomcat

(1)在visualvm中选中"远程”，右击"添加"

⑵主机名上写服务器的ip地址，比如31.100.39.63,然后点击"确定"

⑶右击该主机"31.100.39.63"，添加"JMX"［也就是通过JMX技术具体监控远端服务器哪个Java进程］ ⑷要想让服务器上的tomcat被连接，需要改一下bi n/catal ina.sh这个文件

**注意下面的****8998****不要和服务器上其他端口冲突**

```
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote - Djava.rmi.server.hostname=31.100.39.63 -Dcom.sun.management.jmxremote.port=8998 -Dcom.sun.management.jmxremote.ssl=false - Dcom.sun.management.jmxremote.authenticate=true - Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access - Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password"
```

⑸在../conf 文件中添加两个文件jmxremote.access和jmxremote.password

jmxremote.access 文件

```
guest readonly 
manager readwrite

```

jmxremote.password 文件

```
guest guest 
manager manager
```

> 授予权限：chmod 600 *jmxremot*
>

(6)   将连接服务器地址改为公网ip地址

```
hostname -i 查看输出情况
172.26.225.240 172.17.0.1 

vim /etc/hosts
172.26.255.240 31.100.39.63
```

(7)  设置上述端口对应的阿里云安全策略和防火墙策略

(8)  启动tomcat，来到bin目录

./startup.sh

⑼查看tomcat启动日志以及端口监听

tail -f ../logs/catalina.out lsof -i tcp:8080

(10)查看8998监听情况，可以发现多开了几个端口

lsof -i:8998      得到 PID

netstat -antup | grep PID

(11)在刚才的JMX中输入8998端口，并且输入用户名和密码则登录成功

```
端口:8998 

用户名:manager 

密码：manager
```

### 1.3.3   Arthas

github :[ https://github.eom/alibaba/arthas](https://github.com/alibaba/arthas)

Arthas allows developers to troubleshoot production issues for Java applications without modifying code or restarting servers.

Ar thas是Alibaba开源的Java诊断工具，采用**命令行交互模式**，是排查jvm相关问题的利器。

Arthas是Alibaba开源的Java诊断工具，深受开发者喜爰。

当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

0.这个类从哪个jar包加载的？为什么会报各种类相关的Exception?

1. 我改的代码为什么没有执行到？难道是我没commit?分支搞错了？

2. 遇到问题无法在线上debug,难道只能通过加日志再重新发布吗？

3. 线上遇到某个用户的数据处理有问题，但线上同样无法debug,线下无法重现！

4. 是否有一个全局视角来查看系统的运行状况？

5. 有什么办法可以监控到JVM的实时运行状态？

6. 怎么快速定位应用的热点，生成火焰图？

Arthas支持JDK 6+,支扌寺Linux/Mac/Windows,采用命令行交互模式，同时提供丰富的Tab自动补全功能，进一步方便 进行问题的定位和诊断。

#### 1.3.3.1   下载安装

```
curl -O https：//alibaba.github.io/arthas/arthas-boot.jar java -jar arthas-boot.jar

#然后可以选择一个Java进程
```

**Print usage**

```
java -jar arthas-boot.jar -h
```

#### 1.3.3.2   常用命令

具体每个命令怎么使用，大家可以自己查阅资料

```
version:查看arthas版本号 
help:查看命名帮助信息 
cls:清空屏幕 
session:查看当前会话信息 
quit:退出arthas客户端
--- 
dashboard:当前进程的实时数据面板 
thread:当前JVM的线程堆栈信息 
jvm:查看当前JVM的信息 
sysprop:查看JVM的系统属性
---
sc:查看JVM已经加载的类信息
dump:dump已经加载类的byte code到特定目录 
jad:反编译指定已加载类的源码
---
monitor:方法执行监控
watch:方法执行数据观测 
trace:方法内部调用路径，并输出方法路径上的每个节点上耗时 
stack:输出当前方法被调用的调用路径
......
```



### 1.3.4   MAT

Java堆分析器，用于查找内存泄漏

Heap Dump，称为堆转储文件，是Java进程在某个时间内的快照

下载地址：[https://www.eelipse.org/mat/downloads.php](https://www.eclipse.org/mat/downloads.php)

#### 1.3.4.1 Dump信息包含的内容

•   All Objects

Class, fields, primitive values and references

•   All Classes

Classloader, name, super class, static fields

•   Gar bage Collection Roots

Objects defined to be reachable by the JVM

•   Thr ead Stacks and Local Var iables

The call-stacks of threads at the moment of the snapshot, and per-frame information about local objects

#### 1.3.4.2获取Dump文件

-手动

jmap -dump:format=b,file=heap.hprof 44808

•自动

-XX：+HeapDumponoutofMemoryError -XX：HeapDumpPath=heap.hprof

#### 1.3.4.3   使用

- Histog ram

> Histog ram可以列出内存中的对象，对象的个数及其大小
>

Class Name:类名称，java类名

Objects:类的对象的数量，这个对象被创建了多少个

Shallow Heap: 一个对象内存的消耗大小，不包含对其他对象的引用

Retained Heap:是shallow Heap的总和，即该对象被GC之后所能回收到内存的总和

右击类名——>List Objects——>with incoming references——>列出该类的实例

右击Java对象名——>Merge Shortest Paths to GC Roots——>exclude all ...——>找至Ugc Root以及原因

- Leak Suspects


> 查找并分析内存泄漏的可能原因
>

Reports——>Leak Suspects——>Details

- Top Consumers


列出大对象

### 1.3.4   GC日志分析工具

要想分析日志的信息，得先拿到GC日志文件才行，所以得先配置一下

根据前面参数的学习，下面的配置很容易看懂

```
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps

-Xloggc:gc.log
```

- 在线


[http://gceasy.io](http://gceasy.io/)

- GCViewer