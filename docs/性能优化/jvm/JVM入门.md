# JVM体系结构概述

## JVM的位置

![022219094210020](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbq2yww8j30br0b8wf9.jpg)

## JVM体系结构概览

![022219094210036](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbq5bukij31400u0jyx.jpg)

# 类装载器

负责加载class文件，class文件在文件开头有特定的文件标示，并且ClassLoader只负责class文件的加载，至于它是否可以运行，则由Execution Engine决定

![](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbq75kzxj30ek0datap.jpg)

## 虚拟机自带的加载器

> • 启动类加载器（Bootstrap）C++
> • 扩展类加载器（Extension）Java
> • 应用程序类加载器（AppClassLoader）
> Java也叫系统类加载器，加载当前应用的classpath的所有类

## 用户自定义加载器

java.lang.ClassLoader的子类，用户可以定制类的加载方式

![022219094210032](http://ww2.sinaimg.cn/large/006tNc79gy1g5xbq966wgj309a0ba74a.jpg)

# JVM运行时数据区

![1551354032678](http://ww2.sinaimg.cn/large/006tNc79gy1g5xbqaer8kj30pv0asgqg.jpg)

## 程序计数器PC Register

程序计数器是一块较小的内存空间，是当前线程所执行的字节码的行号指示器

• 程序计算器处于线程独占区

• 如果线程执行的是java方法，记录的是正在执行的虚拟机字节码指令的地址，如果是native方法，这个计数器值为undefined

## 方法区Method Area 

方法区是被所有线程共享，所有字段和方法字节码，以及一些特殊方法如构造函数，接口代码也在此定义。简单说，所有定义的方法的信息都保存在该区域，此区属于共享区间。

静态变量+常量+类信息(构造方法/接口定义)+运行时常量池存在方法区

> 方法区
>
> 永久存储区是一个常驻内存区域，用于存放JDK自身所携带的 Class,Interface的元数据，也就是说它存储的是运行环境必须的类信息，被装载进此区域的数据是不会被垃圾回收器回收掉的，关闭 JVM 才会释放此区域所占用的内存。
>
> 如果出现java.lang.OutOfMemoryError: PermGen space，说明是Java虚拟机对永久代Perm内存设置不够。一般出现这种情况，都是程序启动需要加载大量的第三方jar包。例如：在一个Tomcat下部署了太多的应用。或者大量动态反射生成的类不断被加载，最终导致Perm区被占满。
>
> **Jdk1.6及之前： 有永久代, 常量池1.6在方法区**
>
> **Jdk1.7： 有永久代，但已经逐步“去永久代”，常量池1.7在堆**
>
> **Jdk1.8及之后： 无永久代，常量池1.8在元空间**

## Java栈Stack

栈也叫栈内存，主管Java程序的运行，是在线程创建时创建，它的生命期是跟随线程的生命期，线程结束栈内存也就释放，对于栈来说不存在垃圾回收问题，只要线程一结束该栈就Over，生命周期和线程一致，是线程私有的。

8种基本类型的变量+对象的引用变量+实例方法都是在函数的栈内存中分配

**栈存储什么**?

> • 局部变量表:输入参数和输出参数以及方法内的变量类型；局部变量表在编译期间完成分配，当进入一个方法时，这个方法在帧中分配多少内存是固定的
> • 栈操作（Operand Stack）:记录出栈、入栈的操作；
> • 动态链接
> • 方法出口

栈溢出
`StackOverflowError`,`OutOfMemory`

![022219094210094](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbqdbff2j309y0gfdgy.jpg)

![022219094210102](http://ww4.sinaimg.cn/large/006tNc79ly1g61hpd0xcmj302z04fmx6.jpg)

**栈+堆+方法区的交互关系**

![022219094210106](http://ww4.sinaimg.cn/large/006tNc79gy1g5xbqg1ihsj30j308a78z.jpg)

HotSpot是使用指针的方式来访问对象

Java堆中会存放访问类元数据的地址

reference存储的就直接是对象的

## 本地方法栈

它的具体做法是Native Method Stack中登记native方法，在Execution Engine 执行时加载本地方法库

## 堆Heap

一个JVM实例只存在一个堆内存，堆内存的大小是可以调节的。类加载器读取了类文件后，需要把类、方法、
常变量放到堆内存中，保存所有引用类型的真实信息，以方便执行器执行，堆内存分为三部分：
⚫ Permanent Space			永久区		Perm
⚫ Young Generation Space	新生区		Young/New
⚫ Tenure Generation Space	养老区		Old/ Tenure

### Heap堆(Java8)

![022219094210121](http://ww2.sinaimg.cn/large/006tNc79gy1g5xbqikkb5j31560gcgxg.jpg)

> 方法区（Method Area），是各个线程共享的内存区域，它用于存储虚拟机加载的：类信息+普通常量+静态常量+编译器编译后的代码等等，虽然JVM规范将方法区描述为堆的一个逻辑部分，但它却还有一个别名叫做Non-Heap(非堆)，目的就是要和堆分开。
>
> 对于HotSpot虚拟机，很多开发者习惯将方法区称之为“永久代(ParmanentGen)” ，但严格本质上说两者不同，或者说使用永久代来实现方法区而已，永久代是方法区(相当于是一个接口interface)的一个实现，jdk1.7的版本中，已经将原本放在永久代的字符串常量池移走。
>
> 常量池（Constant Pool）是方法区的一部分，Class文件除了有类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池，这部分内容将在类加载后进入方法区的运行时常量池中存放。

### Java7

![022219094210213](http://ww1.sinaimg.cn/large/006tNc79gy1g5xbqknvyxj30uq0bfaaq.jpg)

### Java8

JDK 1.8之后将最初的永久代取消了，由元空间取代。![022219094210217](http://ww3.sinaimg.cn/large/006tNc79gy1g5xbqo318ej30uq0bfjry.jpg)

# 对象

## 给对象分配内存

• 指针碰撞
• 空间列表

## 线程安全问题

• 线程同步
• 本地线程分配缓冲(TLAB)

## 对象的结构

> • Header(对象头)
>
> 自身运行时数据（Mark Word）
>  哈希值
>  GC分代年龄
> 锁状态标志
> 线程持有锁
>  偏向线程ID
> 偏向时间戳
>  类型指针
>  数组长度（只有数组对象才有）
>
> • InstanceData
>  相同宽度的数据分配到一起（long,double）
>
> • Padding（对齐填充）
>  8个字节的整数倍

![](http://ww4.sinaimg.cn/large/006tNc79gy1g5xbqsalz6j30o30bo41j.jpg)

## 对象的访问定位

• 使用句柄
• 直接指针