# 第3章　Java内存模型

Java线程之间的通信对程序员完全透明，内存可见性问题很容易困扰Java程序员，本章将揭开Java内存模型神秘的面纱。本章大致分4部分：Java内存模型的基础，主要介绍内存模型相关的基本概念；Java内存模型中的顺序一致性，主要介绍重排序与顺序一致性内存模型；同步原语，主要介绍3个同步原语（synchronized、volatile和final）的内存语义及重排序规则在处理器中的实现；Java内存模型的设计，主要介绍Java内存模型的设计原理，及其与处理器内存模型和顺序一致性内存模型的关系。

## 3.1　Java内存模型的基础

### 3.1.1　并发编程模型的两个关键问题

在并发编程中，需要处理两个关键问题：线程之间如何通信及线程之间如何同步（这里的线程是指并发执行的活动实体）。通信是指线程之间以何种机制来交换信息。在命令式编程中，线程之间的通信机制有两种：共享内存和消息传递。

在共享内存的并发模型里，线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信。在消息传递的并发模型里，线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。

同步是指程序中用于控制不同线程间操作发生相对顺序的机制。在共享内存并发模型里，同步是显式进行的。程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。

在消息传递的并发模型里，由于消息的发送必须在消息的接收之前，因此同步是隐式进行的。

Java的并发采用的是共享内存模型，Java线程之间的通信总是隐式进行，整个通信过程对程序员完全透明。如果编写多线程程序的Java程序员不理解隐式进行的线程之间通信的工作机制，很可能会遇到各种奇怪的内存可见性问题。

### 3.1.2　Java内存模型的抽象结构

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享（本章用“共享变量”这个术语代指实例域，静态域和数组元素）。局部变量（Local Variables），方法定义参数（Java语言规范称之为Formal Method Parameters）和异常处理器参数（Exception Handler Parameters）不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。

Java线程之间的通信由Java内存模型（本文简称为JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。Java内存模型的抽象示意如图3-1所示。![Java内存模型的抽象结构示意图](https://ws3.sinaimg.cn/large/006tNc79ly1g3bd9o6cbrj30mr0j3goa.jpg)

图3-1　Java内存模型的抽象结构示意图

从图3-1来看，如果线程A与线程B之间要通信的话，必须要经历下面2个步骤。

1）线程A把本地内存A中更新过的共享变量刷新到主内存中去。

2）线程B到主内存中去读取线程A之前已更新过的共享变量。

下面通过示意图（见图3-2）来说明这两个步骤。![图3-2　线程之间的通信图](https://ws3.sinaimg.cn/large/006tNc79ly1g3bdavpc6fj30ny0irjt8.jpg)

图3-2　线程之间的通信图

如图3-2所示，本地内存A和本地内存B由主内存中共享变量x的副本。假设初始时，这3个内存中的x值都为0。线程A在执行时，把更新后的x值（假设值为1）临时存放在自己的本地内存A中。当线程A和线程B需要通信时，线程A首先会把自己本地内存中修改后的x值刷新到主内存中，此时主内存中的x值变为了1。随后，线程B到主内存中去读取线程A更新后的x值，此时线程B的本地内存的x值也变为了1。

从整体来看，这两个步骤实质上是线程A在向线程B发送消息，而且这个通信过程必须要经过主内存。JMM通过控制主内存与每个线程的本地内存之间的交互，来为Java程序员提供内存可见性保证。

### 3.1.3　从源代码到指令序列的重排序

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3种类型。

1）编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

2）指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level Parallelism，ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

3）内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

从Java源代码到最终实际执行的指令序列，会分别经历下面3种重排序，如图3-3所示。![图3-3　从源码到最终执行的指令序列的示意图](https://ws4.sinaimg.cn/large/006tNc79ly1g3bddprs87j30s3037q3w.jpg)

图3-3　从源码到最终执行的指令序列的示意图

上述的1属于编译器重排序，2和3属于处理器重排序。这些重排序可能会导致多线程程序出现内存可见性问题。对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序（不是所有的编译器重排序都要禁止）。对于处理器重排序，JMM的处理器重排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障（Memory Barriers，Intel称之为Memory Fence）指令，通过内存屏障指令来禁止特定类型的处理器重排序。

JMM属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

### 3.1.4　并发编程模型的分类

现代的处理器使用写缓冲区临时保存向内存写入的数据。写缓冲区可以保证指令流水线持续运行，它可以避免由于处理器停顿下来等待向内存写入数据而产生的延迟。同时，通过以批处理的方式刷新写缓冲区，以及合并写缓冲区中对同一内存地址的多次写，减少对内存总线的占用。虽然写缓冲区有这么多好处，但每个处理器上的写缓冲区，仅仅对它所在的处理器可见。这个特性会对内存操作的执行顺序产生重要的影响：处理器对内存的读/写操作的执行顺序，不一定与内存实际发生的读/写操作顺序一致！为了具体说明，请看下面的表3-1。![表3-1　处理器操作内存的执行结果](https://ws3.sinaimg.cn/large/006tNc79ly1g3bdgbfxgdj30z507cdhf.jpg)

表3-1　处理器操作内存的执行结果

假设处理器A和处理器B按程序的顺序并行执行内存访问，最终可能得到x=y=0的结果。具

体的原因如图3-4所示。![图3-4　处理器和内存的交互](https://ws3.sinaimg.cn/large/006tNc79ly1g3bdhh9rwlj30lf0fywgr.jpg)

图3-4　处理器和内存的交互

这里处理器A和处理器B可以同时把共享变量写入自己的写缓冲区（A1，B1），然后从内存中读取另一个共享变量（A2，B2），最后才把自己写缓存区中保存的脏数据刷新到内存中（A3，B3）。当以这种时序执行时，程序就可以得到x=y=0的结果。

从内存操作实际发生的顺序来看，直到处理器A执行A3来刷新自己的写缓存区，写操作A1才算真正执行了。虽然处理器A执行内存操作的顺序为：A1→A2，但内存操作实际发生的顺序却是A2→A1。此时，处理器A的内存操作顺序被重排序了（处理器B的情况和处理器A一样，这里就不赘述了）。

这里的关键是，由于写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。由于现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写-读操作进行重排序。

表3-2是常见处理器允许的重排序类型的列表。![表3-2　处理器的重排序规则](https://ws2.sinaimg.cn/large/006tNc79ly1g3bdl75xo3j30z006pjt0.jpg)

表3-2　处理器的重排序规则

![052315445292075](https://ws3.sinaimg.cn/large/006tNc79gy1g3c76hhzpnj301901amwy.jpg)**注意**，表3-2单元格中的“N”表示处理器不允许两个操作重排序，“Y”表示允许重排序。

从表3-2我们可以看出：常见的处理器都允许Store-Load重排序；常见的处理器都不允许对存在数据依赖的操作做重排序。sparc-TSO和X86拥有相对较强的处理器内存模型，它们仅允许对写-读操作做重排序（因为它们都使用了写缓冲区）。

**![052315445292075](https://ws1.sinaimg.cn/large/006tNc79gy1g3c76ll4p7j301901amwy.jpg)注意**

·sparc-TSO是指以TSO（Total Store Order）内存模型运行时sparc处理器的特性。

·表3-2中的X86包括X64及AMD64。

·由于ARM处理器的内存模型与PowerPC处理器的内存模型非常类似，本文将忽略它。

·数据依赖性后文会专门说明。

为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为4类，如表3-3所示。

表3-3　内存屏障类型表![表3-3　内存屏障类型表](https://ws3.sinaimg.cn/large/006tNc79ly1g3bdnrry4gj30z30cbaga.jpg)

StoreLoad Barriers是一个“全能型”的屏障，它同时具有其他3个屏障的效果。现代的多处理器大多支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（Buffer Fully Flush）。

### 3.1.5　happens-before简介

从JDK 5开始，Java使用新的JSR-133内存模型（除非特别说明，本文针对的都是JSR-133内存模型）。JSR-133使用happens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。

与程序员密切相关的happens-before规则如下。

·程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

·监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

·volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

·传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

![052315445292075](https://ws1.sinaimg.cn/large/006tNc79gy1g3c76rq0i3j301901amwy.jpg)**注意**　两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前（the first is visible to and ordered before the second）。happens-before的定义很微妙，后文会具体说明happens-before为什么要这么定义。

happens-before与JMM的关系如图3-5所示。![图3-5　happens-before与JMM的关系](https://ws4.sinaimg.cn/large/006tNc79ly1g3bdrfy001j30qd0kc0xp.jpg)

图3-5　happens-before与JMM的关系

如图3-5所示，一个happens-before规则对应于一个或多个编译器和处理器重排序规则。对于Java程序员来说，happens-before规则简单易懂，它避免Java程序员为了理解JMM提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现方法。

## 3.2　重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。

### 3.2.1　数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。数据依赖分为下列3种类型，如表3-4所示。

表3-4　数据依赖类型表![表3-4　数据依赖类型表](https://ws1.sinaimg.cn/large/006tNc79ly1g3bdtprpr2j30z208m75z.jpg)

上面3种情况，只要重排序两个操作的执行顺序，程序的执行结果就会被改变。

前面提到过，编译器和处理器可能会对操作做重排序。编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。

这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑。

### 3.2.2　as-if-serial语义

as-if-serial语义的意思是：不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义。

为了遵守as-if-serial语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被编译器和处理器重排序。为了具体说明，请看下面计算圆面积的代码示例。

```java
double pi = 3.14; 				// A
double r = 1.0; 					// B
double area = pi * r * r; // C
```

上面3个操作的数据依赖关系如图3-6所示。![图3-6　3个操作之间的依赖关系](https://ws2.sinaimg.cn/large/006tNc79ly1g3bdwcd3xvj308z078q38.jpg)

图3-6　3个操作之间的依赖关系

如图3-6所示，A和C之间存在数据依赖关系，同时B和C之间也存在数据依赖关系。因此在最终执行的指令序列中，C不能被重排序到A和B的前面（C排到A和B的前面，程序的结果将会被改变）。但A和B之间没有数据依赖关系，编译器和处理器可以重排序A和B之间的执行顺序。

图3-7是该程序的两种执行顺序。![图3-7　程序的两种执行顺序](https://ws1.sinaimg.cn/large/006tNc79ly1g3be0po3w6j30mr073abg.jpg)

图3-7　程序的两种执行顺序

as-if-serial语义把单线程程序保护了起来，遵守as-if-serial语义的编译器、runtime和处理器共同为编写单线程程序的程序员创建了一个幻觉：单线程程序是按程序的顺序来执行的。asif-serial语义使单线程程序员无需担心重排序会干扰他们，也无需担心内存可见性问题。

### 3.2.3　程序顺序规则

根据happens-before的程序顺序规则，上面计算圆的面积的示例代码存在3个happensbefore

关系。

1）A　happens-before B。

2）B　happens-before C。

3）A　happens-before C。

这里的第3个happens-before关系，是根据happens-before的传递性推导出来的。

这里A happens-before B，但实际执行时B却可以排在A之前执行（看上面的重排序后的执行顺序）。如果A happens-before B，JMM并不要求A一定要在B之前执行。JMM仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。这里操作A的执行结果不需要对操作B可见；而且重排序操作A和操作B后的执行结果，与操作A和操作B按happens-before顺序执行的结果一致。在这种情况下，JMM会认为这种重排序并不非法（not illegal），JMM允许这种重排序。

在计算机中，软件技术和硬件技术有一个共同的目标：在不改变程序执行结果的前提下，尽可能提高并行度。编译器和处理器遵从这一目标，从happens-before的定义我们可以看出，JMM同样遵从这一目标。

### 3.2.4　重排序对多线程的影响

现在让我们来看看，重排序是否会改变多线程程序的执行结果。请看下面的示例代码。

```java
    class ReorderExample {
        int a = 0;
        boolean flag = false;
        public void writer() {
            a = 1; 							// 1
            flag = true;				// 2
        }
        Public void reader() {
            if (flag) { 				// 3
                int i = a * a; 	// 4
                //……
            }
        }
    }
```

flag变量是个标记，用来标识变量a是否已被写入。这里假设有两个线程A和B，A首先执行writer()方法，随后B线程接着执行reader()方法。线程B在执行操作4时，能否看到线程A在操作1对共享变量a的写入呢？

**答案是：不一定能看到。**

由于操作1和操作2没有数据依赖关系，编译器和处理器可以对这两个操作重排序；同样，操作3和操作4没有数据依赖关系，编译器和处理器也可以对这两个操作重排序。让我们先来看看，当操作1和操作2重排序时，可能会产生什么效果？请看下面的程序执行时序图，如图3-8所示。![图3-8　程序执行时序图](https://ws2.sinaimg.cn/large/006tNc79ly1g3bea3a7eaj30j10eqq3k.jpg)

图3-8　程序执行时序图

如图3-8所示，操作1和操作2做了重排序。程序执行时，线程A首先写标记变量flag，随后线程B读这个变量。由于条件判断为真，线程B将读取变量a。此时，变量a还没有被线程A写入，在这里多线程程序的语义被重排序破坏了！

**![052315445292075](https://ws3.sinaimg.cn/large/006tNc79gy1g3c76wvoy3j301901amwy.jpg)注意**　本文统一用虚箭线标识错误的读操作，用实箭线标识正确的读操作。下面再让我们看看，当操作3和操作4重排序时会产生什么效果（借助这个重排序，可以顺便说明控制依赖性）。下面是操作3和操作4重排序后，程序执行的时序图，如图3-9所示。

![图3-9　程序的执行时序图](https://ws4.sinaimg.cn/large/006tNc79ly1g3be9nwg9nj30q70gtabq.jpg)

图3-9　程序的执行时序图

在程序中，操作3和操作4存在控制依赖关系。当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器和处理器会采用猜测（Speculation）执行来克服控制相关性对并行度的影响。以处理器的猜测执行为例，执行线程B的处理器可以提前读取并计算a*a，然后把计算结果临时保存到一个名为重排序缓冲（Reorder Buffer，ROB）的硬件缓存中。当操作3的条件判断为真时，就把该计算结果写入变量i中。

从图3-9中我们可以看出，猜测执行实质上对操作3和4做了重排序。重排序在这里破坏了多线程程序的语义！

在单线程程序中，对存在控制依赖的操作重排序，不会改变执行结果（这也是as-if-serial语义允许对存在控制依赖的操作做重排序的原因）；但在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

## 3.3　顺序一致性

顺序一致性内存模型是一个理论参考模型，在设计的时候，处理器的内存模型和编程语言的内存模型都会以顺序一致性内存模型作为参照。

### 3.3.1　数据竞争与顺序一致性

当程序未正确同步时，就可能会存在数据竞争。Java内存模型规范对数据竞争的定义如下。

在一个线程中写一个变量，

在另一个线程读同一个变量，

而且写和读没有通过同步来排序。

当代码中包含数据竞争时，程序的执行往往产生违反直觉的结果（前一章的示例正是如此）。如果一个多线程程序能正确同步，这个程序将是一个没有数据竞争的程序。

JMM对正确同步的多线程程序的内存一致性做了如下保证。

如果程序是正确同步的，程序的执行将具有顺序一致性（Sequentially Consistent）——即程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同。马上我们就会看到，这对于程序员来说是一个极强的保证。这里的同步是指广义上的同步，包括对常用同步原语（synchronized、volatile和final）的正确使用。

### 3.3.2　顺序一致性内存模型

顺序一致性内存模型是一个被计算机科学家理想化了的理论参考模型，它为程序员提供了极强的内存可见性保证。顺序一致性内存模型有两大特性。

1）一个线程中的所有操作必须按照程序的顺序来执行。

2）（不管程序是否同步）所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立刻对所有线程可见。

顺序一致性内存模型为程序员提供的视图如图3-10所示。![图3-10　顺序一致性内存模型的视图](https://ws1.sinaimg.cn/large/006tNc79ly1g3befw1ifpj30iz0fxgms.jpg)

图3-10　顺序一致性内存模型的视图

在概念上，顺序一致性模型有一个单一的全局内存，这个内存通过一个左右摆动的开关可以连接到任意一个线程，同时每一个线程必须按照程序的顺序来执行内存读/写操作。从上面的示意图可以看出，在任意时间点最多只能有一个线程可以连接到内存。当多个线程并发执行时，图中的开关装置能把所有线程的所有内存读/写操作串行化（即在顺序一致性模型中，所有操作之间具有全序关系）。

为了更好进行理解，下面通过两个示意图来对顺序一致性模型的特性做进一步的说明。

假设有两个线程A和B并发执行。其中A线程有3个操作，它们在程序中的顺序是：

A1→A2→A3。B线程也有3个操作，它们在程序中的顺序是：B1→B2→B3。

假设这两个线程使用监视器锁来正确同步：A线程的3个操作执行后释放监视器锁，随后B

线程获取同一个监视器锁。那么程序在顺序一致性模型中的执行效果将如图3-11所示。![图3-11　顺序一致性模型的一种执行效果](https://ws2.sinaimg.cn/large/006tNc79ly1g3beivrrufj30se0eugqo.jpg)

图3-11　顺序一致性模型的一种执行效果

现在我们再假设这两个线程没有做同步，下面是这个未同步程序在顺序一致性模型中的执行示意图，如图3-12所示。![图3-12　顺序一致性模型中的另一种执行效果](https://ws3.sinaimg.cn/large/006tNc79ly1g3bek088duj30ry0f679c.jpg)

图3-12　顺序一致性模型中的另一种执行效果

未同步程序在顺序一致性模型中虽然整体执行顺序是无序的，但所有线程都只能看到一个一致的整体执行顺序。以上图为例，线程A和B看到的执行顺序都是：

B1→A1→A2→B2→A3→B3。之所以能得到这个保证是因为顺序一致性内存模型中的每个操作必须立即对任意线程可见。

但是，在JMM中就没有这个保证。未同步程序在JMM中不但整体的执行顺序是无序的，而且所有线程看到的操作执行顺序也可能不一致。比如，在当前线程把写过的数据缓存在本地

内存中，在没有刷新到主内存之前，这个写操作仅对当前线程可见；从其他线程的角度来观察，会认为这个写操作根本没有被当前线程执行。只有当前线程把本地内存中写过的数据刷新到主内存之后，这个写操作才能对其他线程可见。在这种情况下，当前线程和其他线程看到的操作执行顺序将不一致。

3.3.3　同步程序的顺序一致性效果

下面，对前面的示例程序ReorderExample用锁来同步，看看正确同步的程序如何具有顺序一致性。

请看下面的示例代码。

```java
    class SynchronizedExample {
        int a = 0;
        boolean flag = false;
        public synchronized void writer() { // 获取锁
            a = 1;
            flag = true;
        } 																	// 释放锁
        public synchronized void reader() { // 获取锁
            if (flag) {
                int i = a;
                ……
            } 															// 释放锁
        }
    }
```

在上面示例代码中，假设A线程执行writer()方法后，B线程执行reader()方法。这是一个正确同步的多线程程序。根据JMM规范，该程序的执行结果将与该程序在顺序一致性模型中的执行结果相同。下面是该程序在两个内存模型中的执行时序对比图，如图3-13所示。

顺序一致性模型中，所有操作完全按程序的顺序串行执行。而在JMM中，临界区内的代码可以重排序（但JMM不允许临界区内的代码“逸出”到临界区之外，那样会破坏监视器的语义）。JMM会在退出临界区和进入临界区这两个关键时间点做一些特别处理，使得线程在这两个时间点具有与顺序一致性模型相同的内存视图（具体细节后文会说明）。虽然线程A在临界区内做了重排序，但由于监视器互斥执行的特性，这里的线程B根本无法“观察”到线程A在临界区内的重排序。这种重排序既提高了执行效率，又没有改变程序的执行结果。![图3-13　两个内存模型中的执行时序对比图](https://ws1.sinaimg.cn/large/006tNc79ly1g3beogtyl8j30pe0k6juu.jpg)

图3-13　两个内存模型中的执行时序对比图

从这里我们可以看到，JMM在具体实现上的基本方针为：在不改变（正确同步的）程序执

行结果的前提下，尽可能地为编译器和处理器的优化打开方便之门。

### 3.3.4　未同步程序的执行特性

对于未同步或未正确同步的多线程程序，JMM只提供最小安全性：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0，Null，False），JMM保证线程读操作读取到的值不会无中生有（Out Of Thin Air）的冒出来。为了实现最小安全性，JVM在堆上分配对象时，首先会对内存空间进行清零，然后才会在上面分配对象（JVM内部会同步这两个操作）。因此，在已清零的内存空间（Pre-zeroed Memory）分配对象时，域的默认初始化已经完成了。

JMM不保证未同步程序的执行结果与该程序在顺序一致性模型中的执行结果一致。因为如果想要保证执行结果一致，JMM需要禁止大量的处理器和编译器的优化，这对程序的执行性能会产生很大的影响。而且未同步程序在顺序一致性模型中执行时，整体是无序的，其执行结果往往无法预知。而且，保证未同步程序在这两个模型中的执行结果一致没什么意义。未同步程序在JMM中的执行时，整体上是无序的，其执行结果无法预知。未同步程序在两个模型中的执行特性有如下几个差异。

1）顺序一致性模型保证单线程内的操作会按程序的顺序执行，而JMM不保证单线程内的操作会按程序的顺序执行（比如上面正确同步的多线程程序在临界区内的重排序）。这一点前面已经讲过了，这里就不再赘述。

2）顺序一致性模型保证所有线程只能看到一致的操作执行顺序，而JMM不保证所有线程能看到一致的操作执行顺序。这一点前面也已经讲过，这里就不再赘述。

3）JMM不保证对64位的long型和double型变量的写操作具有原子性，而顺序一致性模型保证对所有的内存读/写操作都具有原子性。

第3个差异与处理器总线的工作机制密切相关。在计算机中，数据通过总线在处理器和内存之间传递。每次处理器和内存之间的数据传递都是通过一系列步骤来完成的，这一系列步骤称之为总线事务（Bus Transaction）。总线事务包括读事务（Read Transaction）和写事务（Write Transaction）。读事务从内存传送数据到处理器，写事务从处理器传送数据到内存，每个事务会读/写内存中一个或多个物理上连续的字。这里的关键是，总线会同步试图并发使用总线的事务。在一个处理器执行总线事务期间，总线会禁止其他的处理器和I/O设备执行内存的读/写。

下面，让我们通过一个示意图来说明总线的工作机制，如图3-14所示。![图3-14　总线的工作机制](https://ws3.sinaimg.cn/large/006tNc79gy1g3bevvqc3gj30k70k7gnd.jpg)

图3-14　总线的工作机制

由图可知，假设处理器A，B和C同时向总线发起总线事务，这时总线仲裁（Bus Arbitration）会对竞争做出裁决，这里假设总线在仲裁后判定处理器A在竞争中获胜（总线仲裁会确保所有处理器都能公平的访问内存）。此时处理器A继续它的总线事务，而其他两个处理器则要等待处理器A的总线事务完成后才能再次执行内存访问。假设在处理器A执行总线事务期间（不管这个总线事务是读事务还是写事务），处理器D向总线发起了总线事务，此时处理器D的请求会被总线禁止。

总线的这些工作机制可以把所有处理器对内存的访问以串行化的方式来执行。在任意时间点，最多只能有一个处理器可以访问内存。这个特性确保了单个总线事务之中的内存读/写操作具有原子性。

在一些32位的处理器上，如果要求对64位数据的写操作具有原子性，会有比较大的开销。为了照顾这种处理器，Java语言规范鼓励但不强求JVM对64位的long型变量和double型变量的写操作具有原子性。当JVM在这种处理器上运行时，可能会把一个64位long/double型变量的写操作拆分为两个32位的写操作来执行。这两个32位的写操作可能会被分配到不同的总线事务中执行，此时对这个64位变量的写操作将不具有原子性。

当单个内存操作不具有原子性时，可能会产生意想不到后果。请看示意图，如图3-15所示。![图3-15　总线事务执行的时序图](https://ws4.sinaimg.cn/large/006tNc79ly1g3bexp5j8ej30ph0eqq4d.jpg)

图3-15　总线事务执行的时序图

如上图所示，假设处理器A写一个long型变量，同时处理器B要读这个long型变量。处理器A中64位的写操作被拆分为两个32位的写操作，且这两个32位的写操作被分配到不同的写事务中执行。同时，处理器B中64位的读操作被分配到单个的读事务中执行。当处理器A和B按上图的时序来执行时，处理器B将看到仅仅被处理器A“写了一半”的无效值。

![052315445292075](https://ws2.sinaimg.cn/large/006tNc79ly1g3c77n95orj301901amwy.jpg)**注意**，在JSR-133之前的旧内存模型中，一个64位long/double型变量的读/写操作可以被拆分为两个32位的读/写操作来执行。从JSR-133内存模型开始（即从JDK5开始），仅仅只允许把一个64位long/double型变量的写操作拆分为两个32位的写操作来执行，任意的读操作在JSR-133中都必须具有原子性（即任意读操作必须要在单个读事务中执行）。

## 3.4　volatile的内存语义

当声明共享变量为volatile后，对这个变量的读/写将会很特别。为了揭开volatile的神秘面纱，下面将介绍volatile的内存语义及volatile内存语义的实现。

### 3.4.1　volatile的特性

理解volatile特性的一个好方法是把对volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步。下面通过具体的示例来说明，示例代码如下。

```java
class VolatileFeaturesExample {
    volatile long vl = 0L; 								// 使用volatile声明64位的long型变量
    public void set(long l) {
        vl = l; 													// 单个volatile变量的写
    }
    public void getAndIncrement () {
        vl++; 														// 复合（多个）volatile变量的读/写
    }
    public long get() {
        return vl; 												// 单个volatile变量的读
    }
}
```

假设有多个线程分别调用上面程序的3个方法，这个程序在语义上和下面程序等价。

```java
class VolatileFeaturesExample {
    long vl = 0L; 													// 64位的long型普通变量
    public synchronized void set(long l) { 	// 对单个的普通变量的写用同一个锁同步
        vl = l;
    }
    public void getAndIncrement () { 				// 普通方法调用
        long temp = get(); 									// 调用已同步的读方法
        temp += 1L; 												// 普通写操作
        set(temp); 													// 调用已同步的写方法
    }
    public synchronized long get() { 				// 对单个的普通变量的读用同一个锁同步
        return vl;
    }
}
```

如上面示例程序所示，一个volatile变量的单个读/写操作，与一个普通变量的读/写操作都是使用同一个锁来同步，它们之间的执行效果相同。

锁的happens-before规则保证释放锁和获取锁的两个线程之间的内存可见性，这意味着对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。

锁的语义决定了临界区代码的执行具有原子性。这意味着，即使是64位的long型和double型变量，只要它是volatile变量，对该变量的读/写就具有原子性。如果是多个volatile操作或类似于volatile++这种复合操作，这些操作整体上不具有原子性。

简而言之，volatile变量自身具有下列特性。

·可见性。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。

·原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性。

### 3.4.2　volatile写-读建立的happens-before关系

上面讲的是volatile变量自身的特性，对程序员来说，volatile对线程的内存可见性的影响比volatile自身的特性更为重要，也更需要我们去关注。

从JSR-133开始（即从JDK5开始），volatile变量的写-读可以实现线程之间的通信。

从内存语义的角度来说，volatile的写-读与锁的释放-获取有相同的内存效果：volatile写和锁的释放有相同的内存语义；volatile读与锁的获取有相同的内存语义。

请看下面使用volatile变量的示例代码。

```java
    class VolatileExample {
        int a = 0;
        volatile boolean flag = false;
        public void writer() {
            a = 1;									// 1
            flag = true;　　　 			// 2
        }
        public void reader() {
            if (flag) {							// 3
                int i = a;					// 4
								……
            }
        }
    }
```

假设线程A执行writer()方法之后，线程B执行reader()方法。根据happens-before规则，这个

过程建立的happens-before关系可以分为3类：

1）根据程序次序规则，1 happens-before 2;3 happens-before 4。

2）根据volatile规则，2 happens-before 3。

3）根据happens-before的传递性规则，1 happens-before 4。

上述happens-before关系的图形化表现形式如下。![图3-16　happens-before关系](https://ws2.sinaimg.cn/large/006tNc79ly1g3bfkuo3hjj30mm0ksgn4.jpg)

图3-16　happens-before关系

在上图中，每一个箭头链接的两个节点，代表了一个happens-before关系。黑色箭头表示程序顺序规则；橙色箭头表示volatile规则；蓝色箭头表示组合这些规则后提供的happens-before保证。

这里A线程写一个volatile变量后，B线程读同一个volatile变量。A线程在写volatile变量之前所有可见的共享变量，在B线程读同一个volatile变量后，将立即变得对B线程可见。

![052315445292075](https://ws3.sinaimg.cn/large/006tNc79ly1g3c77xyqkrj301901amwy.jpg)**注意**　本文统一用粗实线标识组合后产生的happens-before关系。

### 3.4.3　volatile写-读的内存语义

volatile写的内存语义如下。

当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。

以上面示例程序VolatileExample为例，假设线程A首先执行writer()方法，随后线程B执行

reader()方法，初始时两个线程的本地内存中的flag和a都是初始状态。图3-17是线程A执行volatile写后，共享变量的状态示意图。![图3-17　共享变量的状态示意图](https://ws3.sinaimg.cn/large/006tNc79ly1g3bfpi5rh6j30md0ivmyb.jpg)

图3-17　共享变量的状态示意图

如图3-17所示，线程A在写flag变量后，本地内存A中被线程A更新过的两个共享变量的值被刷新到主内存中。此时，本地内存A和主内存中的共享变量的值是一致的。

volatile读的内存语义如下。

当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

图3-18为线程B读同一个volatile变量后，共享变量的状态示意图。

如图所示，在读flag变量后，本地内存B包含的值已经被置为无效。此时，线程B必须从主内存中读取共享变量。线程B的读取操作将导致本地内存B与主内存中的共享变量的值变成一致。

如果我们把volatile写和volatile读两个步骤综合起来看的话，在读线程B读一个volatile变量后，写线程A在写这个volatile变量之前所有可见的共享变量的值都将立即变得对读线程B可见。

下面对volatile写和volatile读的内存语义做个总结。

·线程A写一个volatile变量，实质上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所做修改的）消息。

·线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。

·线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息。![图3-18　共享变量的状态示意图](https://ws1.sinaimg.cn/large/006tNc79ly1g3bfr57i6ij30m20k540j.jpg)

图3-18　共享变量的状态示意图

### 3.4.4　volatile内存语义的实现

下面来看看JMM如何实现volatile写/读的内存语义。

前文提到过重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM会分别限制这两种类型的重排序类型。表3-5是JMM针对编译器制定的volatile重排序规则表。

表3-5　volatile重排序规则表![表3-5　volatile重排序规则表](https://ws4.sinaimg.cn/large/006tNc79gy1g3bk77lgifj30z1066abb.jpg)

举例来说，第三行最后一个单元格的意思是：在程序中，当第一个操作为普通变量的读或写时，如果第二个操作为volatile写，则编译器不能重排序这两个操作。

从表3-5我们可以看出。

·当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。

·当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。

·当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略。

·在每个volatile写操作的前面插入一个StoreStore屏障。

·在每个volatile写操作的后面插入一个StoreLoad屏障。

·在每个volatile读操作的后面插入一个LoadLoad屏障。

·在每个volatile读操作的后面插入一个LoadStore屏障。

上述内存屏障插入策略非常保守，但它可以保证在任意处理器平台，任意的程序中都能得到正确的volatile内存语义。

下面是保守策略下，volatile写插入内存屏障后生成的指令序列示意图，如图3-19所示。![图3-19　指令序列示意图](https://ws4.sinaimg.cn/large/006tNc79gy1g3bkffs577j30p80egq5d.jpg)

图3-19　指令序列示意图

图3-19中的StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前刷新到主内存。

这里比较有意思的是，volatile写后面的StoreLoad屏障。此屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序。因为编译器常常无法准确判断在一个volatile写的后面是否需要插入一个StoreLoad屏障（比如，一个volatile写之后方法立即return）。为了保证能正确实现volatile的内存语义，JMM在采取了保守策略：在每个volatile写的后面，或者在每个volatile读的前面插入一个StoreLoad屏障。从整体执行效率的角度考虑，JMM最终选择了在每个volatile写的后面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是：一个写线程写volatile变量，多个读线程读同一个volatile变量。当读线程的数量大大超过写线程时，选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。从这里可以看到JMM在实现上的一个特点：首先确保正确性，然后再去追求执行效率。

下面是在保守策略下，volatile读插入内存屏障后生成的指令序列示意图，如图3-20所示。![图3-20　指令序列示意图](https://ws2.sinaimg.cn/large/006tNc79gy1g3bkixh94fj30pk0edack.jpg)

图3-20　指令序列示意图

图3-20中的LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

上述volatile写和volatile读的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。下面通过具体的示例代码进行说明。

```java
class VolatileBarrierExample {
    int a;
    volatile int v1 = 1;
    volatile int v2 = 2;
    void readAndWrite() {
        int i = v1;             // 第一个volatile读
        int j = v2;             // 第二个volatile读
        a = i + j;              // 普通写
        v1 = i + 1;             // 第一个volatile写
        v2 = j * 2;             // 第二个 volatile写
    }
    //...                       // 其他方法
}
```

针对readAndWrite()方法，编译器在生成字节码时可以做如下的优化。![图3-21　指令序列示意图](https://ws4.sinaimg.cn/large/006tNc79gy1g3bknglekbj30nt0kzjy5.jpg)

图3-21　指令序列示意图

注意，最后的StoreLoad屏障不能省略。因为第二个volatile写之后，方法立即return。此时编译器可能无法准确断定后面是否会有volatile读或写，为了安全起见，编译器通常会在这里插入一个StoreLoad屏障。

上面的优化针对任意处理器平台，由于不同的处理器有不同“松紧度”的处理器内存模型，内存屏障的插入还可以根据具体的处理器内存模型继续优化。以X86处理器为例，图3-21中除最后的StoreLoad屏障外，其他的屏障都会被省略。

前面保守策略下的volatile读和写，在X86处理器平台可以优化成如图3-22所示。

前文提到过，X86处理器仅会对写-读操作做重排序。X86不会对读-读、读-写和写-写操作做重排序，因此在X86处理器中会省略掉这3种操作类型对应的内存屏障。在X86中，JMM仅需在volatile写后面插入一个StoreLoad屏障即可正确实现volatile写-读的内存语义。这意味着在X86处理器中，volatile写的开销比volatile读的开销会大很多（因为执行StoreLoad屏障开销会比较大）。![图3-22　指令序列示意图](https://ws2.sinaimg.cn/large/006tNc79gy1g3bl2qbhlmj30qg0drdhw.jpg)

图3-22　指令序列示意图

### 3.4.5　JSR-133为什么要增强volatile的内存语义

在JSR-133之前的旧Java内存模型中，虽然不允许volatile变量之间重排序，但旧的Java内存模型允许volatile变量与普通变量重排序。在旧的内存模型中，VolatileExample示例程序可能被重排序成下列时序来执行，如图3-23所示。![图3-23　线程执行时序图](https://ws4.sinaimg.cn/large/006tNc79gy1g3bl4qo2tyj30mz0i6wfy.jpg)

图3-23　线程执行时序图

在旧的内存模型中，当1和2之间没有数据依赖关系时，1和2之间就可能被重排序（3和4类似）。其结果就是：读线程B执行4时，不一定能看到写线程A在执行1时对共享变量的修改。

因此，在旧的内存模型中，volatile的写-读没有锁的释放-获所具有的内存语义。为了提供一种比锁更轻量级的线程之间通信的机制，JSR-133专家组决定增强volatile的内存语义：严格限制编译器和处理器对volatile变量与普通变量的重排序，确保volatile的写-读和锁的释放-获取具有相同的内存语义。从编译器重排序规则和处理器内存屏障插入策略来看，只要volatile变量与普通变量之间的重排序可能会破坏volatile的内存语义，这种重排序就会被编译器重排序规则和处理器内存屏障插入策略禁止。

由于volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥执行的特性可以确保对整个临界区代码的执行具有原子性。在功能上，锁比volatile更强大；在可伸缩性和执行性能上，volatile更有优势。如果读者想在程序中用volatile代替锁，请一定谨慎，具体详情请参阅Brian Goetz的文章[《Java理论与实践：正确使用Volatile变量》](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html#2.0)。

## 3.5　锁的内存语义

众所周知，锁可以让临界区互斥执行。这里将介绍锁的另一个同样重要，但常常被忽视的功能：锁的内存语义。

### 3.5.1　锁的释放-获取建立的happens-before关系

锁是Java并发编程中最重要的同步机制。锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息。

下面是锁释放-获取的示例代码。

```java
    class MonitorExample {
        int a = 0;
        public synchronized void writer() {         // 1
            a++;                                    // 2
        }                                           // 3
        public synchronized void reader() {         // 4
            int i = a;                              // 5
						……
        }                                           // 6
    }
```

假设线程A执行writer()方法，随后线程B执行reader()方法。根据happens-before规则，这个过程包含的happens-before关系可以分为3类。

1）根据程序次序规则，1 happens-before 2,2 happens-before 3;4 happens-before 5,5 happensbefore6。

2）根据监视器锁规则，3 happens-before 4。

3）根据happens-before的传递性，2 happens-before 5。

上述happens-before关系的图形化表现形式如图3-24所示。![图3-24　happens-before关系图](https://ws3.sinaimg.cn/large/006tNc79gy1g3bliwnl8mj30my0lb75t.jpg)

图3-24　happens-before关系图

在图3-24中，每一个箭头链接的两个节点，代表了一个happens-before关系。黑色箭头表示程序顺序规则；橙色箭头表示监视器锁规则；蓝色箭头表示组合这些规则后提供的happens-before保证。

图3-24表示在线程A释放了锁之后，随后线程B获取同一个锁。在上图中，2 happens-before 5。因此，线程A在释放锁之前所有可见的共享变量，在线程B获取同一个锁之后，将立刻变得对B线程可见。

### 3.5.2　锁的释放-获取的内存语义

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。以上面的MonitorExample程序为例，A线程释放锁后，共享数据的状态示意图如图3-25所示。![图3-25　共享数据的状态示意图](https://ws3.sinaimg.cn/large/006tNc79gy1g3blkzg9wqj30nt0jc3zg.jpg)

图3-25　共享数据的状态示意图

当线程获取锁时，JMM会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。图3-26是锁获取的状态示意图。![图3-26　锁获取的状态示意图](https://ws4.sinaimg.cn/large/006tNc79gy1g3bln1mz0tj30oq0j6myw.jpg)

图3-26　锁获取的状态示意图

对比锁释放-获取的内存语义与volatile写-读的内存语义可以看出：锁释放与volatile写有相同的内存语义；锁获取与volatile读有相同的内存语义。

下面对锁释放和锁获取的内存语义做个总结。

·线程A释放一个锁，实质上是线程A向接下来将要获取这个锁的某个线程发出了（线程A对共享变量所做修改的）消息。

·线程B获取一个锁，实质上是线程B接收了之前某个线程发出的（在释放这个锁之前对共享变量所做修改的）消息。

·线程A释放锁，随后线程B获取这个锁，这个过程实质上是线程A通过主内存向线程B发送消息。

### 3.5.3　锁内存语义的实现

本文将借助ReentrantLock的源代码，来分析锁内存语义的具体实现机制。

请看下面的示例代码。

```java
    class ReentrantLockExample {
        int a = 0;
        ReentrantLock lock = new ReentrantLock();
        public void writer() {
            lock.lock();        // 获取锁
            try {
                a++;
            } finally {
                lock.unlock();  // 释放锁
            }
        }
        public void reader () {
            lock.lock();        // 获取锁
            try {
                int i = a;
								……
            } finally {
                lock.unlock();  // 释放锁
            }
        }
    }
```

在ReentrantLock中，调用lock()方法获取锁；调用unlock()方法释放锁。

ReentrantLock的实现依赖于Java同步器框架AbstractQueuedSynchronizer（本文简称之为AQS）。AQS使用一个整型的volatile变量（命名为state）来维护同步状态，马上我们会看到，这个volatile变量是ReentrantLock内存语义实现的关键。

图3-27是ReentrantLock的类图（仅画出与本文相关的部分）。![图3-27　ReentrantLock的类图](https://ws2.sinaimg.cn/large/006tNc79gy1g3bm2hi52aj30pg0l8gp2.jpg)

图3-27　ReentrantLock的类图

ReentrantLock分为公平锁和非公平锁，我们首先分析公平锁。

使用公平锁时，加锁方法lock()调用轨迹如下。

1）ReentrantLock:lock()。

2）FairSync:lock()。

3）AbstractQueuedSynchronizer:acquire(int arg)。

4）ReentrantLock:tryAcquire(int acquires)。

在第4步真正开始加锁，下面是该方法的源代码。

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();　　　　// 获取锁的开始，首先读volatile变量state
    if (c == 0) {
        if (isFirst(current) &&
                compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)　　
        throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

从上面源代码中我们可以看出，加锁方法首先读volatile变量state。

在使用公平锁时，解锁方法unlock()调用轨迹如下。

1）ReentrantLock:unlock()。

2）AbstractQueuedSynchronizer:release(int arg)。

3）Sync:tryRelease(int releases)。

在第3步真正开始释放锁，下面是该方法的源代码。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);　　　　　// 释放锁的最后，写volatile变量state
    return free;
}
```

从上面的源代码可以看出，在释放锁的最后写volatile变量state。

公平锁在释放锁的最后写volatile变量state，在获取锁时首先读这个volatile变量。根据volatile的happens-before规则，释放锁的线程在写volatile变量之前可见的共享变量，在获取锁的线程读取同一个volatile变量后将立即变得对获取锁的线程可见。

现在我们来分析非公平锁的内存语义的实现。非公平锁的释放和公平锁完全一样，所以

这里仅仅分析非公平锁的获取。使用非公平锁时，加锁方法lock()调用轨迹如下。

1）ReentrantLock:lock()。

2）NonfairSync:lock()。

3）AbstractQueuedSynchronizer:compareAndSetState(int expect,int update)。

在第3步真正开始加锁，下面是该方法的源代码。

```java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

该方法以原子操作的方式更新state变量，本文把Java的compareAndSet()方法调用简称为CAS。JDK文档对该方法的说明如下：如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值。此操作具有volatile读和写的内存语义。

这里我们分别从编译器和处理器的角度来分析，CAS如何同时具有volatile读和volatile写的内存语义。

前文我们提到过，编译器不会对volatile读与volatile读后面的任意内存操作重排序；编译器不会对volatile写与volatile写前面的任意内存操作重排序。组合这两个条件，意味着为了同时实现volatile读和volatile写的内存语义，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

下面我们来分析在常见的intel X86处理器中，CAS是如何同时具有volatile读和volatile写的内存语义的。

下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码。

```java
public final native boolean compareAndSwapInt(Object o, long offset, int expect, int update);
```

可以看到，这是一个本地方法调用。这个本地方法在openjdk中依次调用的c++代码为：

unsafe.cpp，atomic.cpp和atomic_windows_x86.inline.hpp。这个本地方法的最终实现在openjdk的如下位置：openjdk-7-fcs-src-b147-27_jun_2011\openjdk\hotspot\src\os_cpu\windows_x86\vm\atomic_windows_x86.inline.hpp（对应于Windows操作系统，X86处理器）。下面是对应于intel X86处理器的源代码的片段。

```c++
    inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest,
                                 jint compare_value) {
// alternative for InterlockedCompareExchange
        int mp = os::is_MP();
        __asm {
            mov edx, dest
            mov ecx, exchange_value
            mov eax, compare_value
            LOCK_IF_MP(mp)
            cmpxchg dword ptr [edx], ecx
        }
    }
```

如上面源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（Lock Cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

intel的手册对lock前缀的说明如下。

1）确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从Pentium 4、Intel Xeon及P6处理器开始，Intel使用缓存锁定（Cache Locking）来保证指令执行的原子性。缓存锁定将大大降低lock前缀指令的执行开销。

2）禁止该指令，与之前和之后的读和写指令重排序。

3）把写缓冲区中的所有数据刷新到内存中。

上面的第2点和第3点所具有的内存屏障效果，足以同时实现volatile读和volatile写的内存语义。

经过上面的分析，现在我们终于能明白为什么JDK文档说CAS同时具有volatile读和volatile写的内存语义了。

现在对公平锁和非公平锁的内存语义做个总结。

·公平锁和非公平锁释放时，最后都要写一个volatile变量state。

·公平锁获取时，首先会去读volatile变量。

·非公平锁获取时，首先会用CAS更新volatile变量，这个操作同时具有volatile读和volatile写的内存语义。

从本文对ReentrantLock的分析可以看出，锁释放-获取的内存语义的实现至少有下面两种方式。

1）利用volatile变量的写-读所具有的内存语义。

2）利用CAS所附带的volatile读和volatile写的内存语义。

### 3.5.4　concurrent包的实现

由于Java的CAS同时具有volatile读和volatile写的内存语义，因此Java线程之间的通信现在有了下面4种方式。

1）A线程写volatile变量，随后B线程读这个volatile变量。

2）A线程写volatile变量，随后B线程用CAS更新这个volatile变量。

3）A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。

4）A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

Java的CAS会使用现代处理器上提供的高效机器级别的原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。如果我们仔细分析concurrent包的源代码实现，会发现一个通用化的实现模式。

首先，声明共享变量为volatile。

然后，使用CAS的原子条件更新来实现线程之间的同步。

同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。从整体来看，concurrent包的实现示意图如3-28所示。![052315445292049](https://ws4.sinaimg.cn/large/006tNc79gy1g3bmstig92j30nh0h1jsz.jpg)

图3-28　concurrent包的实现示意图

## 3.6　final域的内存语义

与前面介绍的锁和volatile相比，对final域的读和写更像是普通的变量访问。下面将介绍final域的内存语义。

### 3.6.1　final域的重排序规则

对于final域，编译器和处理器要遵守两个重排序规则。

1）在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

2）初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

下面通过一些示例性的代码来分别说明这两个规则。

```java
public class FinalExample {
    int i;           // 普通变量
    final int j;         // final变量
    static FinalExample obj;
    public FinalExample () {   // 构造函数
        i = 1;         // 写普通域
        j = 2;         // 写final域
    }
    public static void writer () {  // 写线程A执行
        obj = new FinalExample ();
    }
    public static void reader () {  // 读线程B执行
        FinalExample object = obj; // 读对象引用
        int a = object.i;      // 读普通域
        int b = object.j;      // 读final域
    }
}
```

这里假设一个线程A执行writer()方法，随后另一个线程B执行reader()方法。下面我们通过这两个线程的交互来说明这两个规则。

### 3.6.2　写final域的重排序规则

写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面。

1）JMM禁止编译器把final域的写重排序到构造函数之外。

2）编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

现在让我们分析writer()方法。writer()方法只包含一行代码：finalExample=newFinalExample()。这行代码包含两个步骤，如下。

1）构造一个FinalExample类型的对象。

2）把这个对象的引用赋值给引用变量obj。

假设线程B读对象引用与读对象的成员域之间没有重排序（马上会说明为什么需要这个假设），图3-29是一种可能的执行时序。

在图3-29中，写普通域的操作被编译器重排序到了构造函数之外，读线程B错误地读取了普通变量i初始化之前的值。而写final域的操作，被写final域的重排序规则“限定”在了构造函数之内，读线程B正确地读取了final变量初始化之后的值。

写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。以上图为例，在读线程B“看到”对象引用obj时，很可能obj对象还没有构造完成（对普通域i的写操作被重排序到构造函数外，此时初始值1还没有写入普通域i）。![052315445292050](https://ws3.sinaimg.cn/large/006tNc79gy1g3bmxzuob8j30m70lt411.jpg)

图3-29　线程执行时序图

### 3.6.3　读final域的重排序规则

读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读final域操作的前面插入一个LoadLoad屏障。

初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也会遵守间接依赖，也不会重排序这两个操作。但有少数处理器允许对存在间接依赖关系的操作做重排序

（比如alpha处理器），这个规则就是专门用来针对这种处理器的。

reader()方法包含3个操作。

·初次读引用变量obj。

·初次读引用变量obj指向对象的普通域j。

·初次读引用变量obj指向对象的final域i。

现在假设写线程A没有发生任何重排序，同时程序在不遵守间接依赖的处理器上执行，图

3-30所示是一种可能的执行时序。![052315445292051](https://ws2.sinaimg.cn/large/006tNc79gy1g3bn30m1lxj30mk0mi76v.jpg)

图3-30　线程执行时序图

在图3-30中，读对象的普通域的操作被处理器重排序到读对象引用之前。读普通域时，该域还没有被写线程A写入，这是一个错误的读取操作。而读final域的重排序规则会把读对象final域的操作“限定”在读对象引用之后，此时该final域已经被A线程初始化过了，这是一个正确的读取操作。

读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。在这个示例程序中，如果该引用不为null，那么引用对象的final域一定已经被A线程初始化过了。

3.6.4　final域为引用类型

上面我们看到的final域是基础数据类型，如果final域是引用类型，将会有什么效果？请看下列示例代码。

```java
public class FinalReferenceExample {
    final int[] intArray;                   // final是引用类型
    static FinalReferenceExample obj;
    public FinalReferenceExample () {       // 构造函数
        intArray = new int[1];              // 1
        intArray[0] = 1;                    // 2
    }
    public static void writerOne () {       // 写线程A执行
        obj = new FinalReferenceExample (); // 3
    }
    public static void writerTwo () {       // 写线程B执行
        obj.intArray[0] = 2;                // 4
    }
    public static void reader () {          // 读线程C执行
        if (obj != null) {                  // 5
            int temp1 = obj.intArray[0];    // 6
        }
    }
}
```

本例final域为一个引用类型，它引用一个int型的数组对象。对于引用类型，写final域的重排序规则对编译器和处理器增加了如下约束：在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

对上面的示例程序，假设首先线程A执行writerOne()方法，执行完后线程B执行writerTwo()方法，执行完后线程C执行reader()方法。图3-31是一种可能的线程执行时序。![052315445292052](https://ws1.sinaimg.cn/large/006tNc79gy1g3bn8nax71j30nz0okdis.jpg)

图3-31　引用型final的执行时序图

在图3-31中，1是对final域的写入，2是对这个final域引用的对象的成员域的写入，3是把被构造的对象的引用赋值给某个引用变量。这里除了前面提到的1不能和3重排序外，2和3也不能重排序。

JMM可以确保读线程C至少能看到写线程A在构造函数中对final引用对象的成员域的写入。即C至少能看到数组下标0的值为1。而写线程B对数组元素的写入，读线程C可能看得到，也可能看不到。JMM不保证线程B的写入对读线程C可见，因为写线程B和读线程C之间存在数据竞争，此时的执行结果不可预知。

如果想要确保读线程C看到写线程B对数组元素的写入，写线程B和读线程C之间需要使用同步原语（lock或volatile）来确保内存可见性。

### 3.6.5　为什么final引用不能从构造函数内“溢出”

前面我们提到过，写final域的重排序规则可以确保：在引用变量为任意线程可见之前，该引用变量指向的对象的final域已经在构造函数中被正确初始化过了。其实，要得到这个效果，

还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程所见，也就是对象引用不能在构造函数中“逸出”。为了说明问题，让我们来看下面的示例代码。

```java
public class FinalReferenceEscapeExample {
    final int i;
    static FinalReferenceEscapeExample obj;
    public FinalReferenceEscapeExample () {
        i = 1; 								// 1写final域
        obj = this; 					// 2 this引用在此"逸出"
    }
    public static void writer() {
        new FinalReferenceEscapeExample ();
    }
    public static void reader() {
        if (obj != null) { 		// 3
            int temp = obj.i; // 4
        }
    }
}
```

假设一个线程A执行writer()方法，另一个线程B执行reader()方法。这里的操作2使得对象还未完成构造前就为线程B可见。即使这里的操作2是构造函数的最后一步，且在程序中操作2排在操作1后面，执行read()方法的线程仍然可能无法看到final域被初始化后的值，因为这里的操作1和操作2之间可能被重排序。实际的执行时序可能如图3-32所示。![052315445292053](https://ws4.sinaimg.cn/large/006tNc79gy1g3bndq0hbkj30pe0ktjtt.jpg)

图3-32　多线程执行时序图

从图3-32可以看出：在构造函数返回前，被构造对象的引用不能为其他线程所见，因为此时的final域可能还没有被初始化。在构造函数返回后，任意线程都将保证能看到final域正确初始化之后的值。

### 3.6.6　final语义在处理器中的实现

现在我们以X86处理器为例，说明final语义在处理器中的具体实现。

上面我们提到，写final域的重排序规则会要求编译器在final域的写之后，构造函数return之前插入一个StoreStore障屏。读final域的重排序规则要求编译器在读final域的操作前面插入一个LoadLoad屏障。

由于X86处理器不会对写-写操作做重排序，所以在X86处理器中，写final域需要的StoreStore障屏会被省略掉。同样，由于X86处理器不会对存在间接依赖关系的操作做重排序，所以在X86处理器中，读final域需要的LoadLoad屏障也会被省略掉。也就是说，在X86处理器中，final域的读/写不会插入任何内存屏障！

### 3.6.7　JSR-133为什么要增强final的语义

在旧的Java内存模型中，一个最严重的缺陷就是线程可能看到final域的值会改变。比如，

一个线程当前看到一个整型final域的值为0（还未初始化之前的默认值），过一段时间之后这个线程再去读这个final域的值时，却发现值变为1（被某个线程初始化之后的值）。最常见的例子就是在旧的Java内存模型中，String的值可能会改变。

为了修补这个漏洞，JSR-133专家组增强了final的语义。通过为final域增加写和读重排序规则，可以为Java程序员提供初始化安全保证：只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指lock和volatile的使用）就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。

## 3.7　happens-before

happens-before是JMM最核心的概念。对应Java程序员来说，理解happens-before是理解JMM的关键。

3.7.1　JMM的设计

首先，让我们来看JMM的设计意图。从JMM设计者的角度，在设计JMM时，需要考虑两个关键因素。

·程序员对内存模型的使用。程序员希望内存模型易于理解、易于编程。程序员希望基于一个强内存模型来编写代码。

·编译器和处理器对内存模型的实现。编译器和处理器希望内存模型对它们的束缚越少越好，这样它们就可以做尽可能多的优化来提高性能。编译器和处理器希望实现一个弱内存模型。

由于这两个因素互相矛盾，所以JSR-133专家组在设计JMM时的核心目标就是找到一个好的平衡点：一方面，要为程序员提供足够强的内存可见性保证；另一方面，对编译器和处理器的限制要尽可能地放松。下面让我们来看JSR-133是如何实现这一目标的。

```java
double pi = 3.14;　　 // A
double r = 1.0;　　　　 // B
double area = pi * r * r;　 // C
```

上面计算圆的面积的示例代码存在3个happens-before关系，如下。

·A happens-before B。

·B happens-before C。

·A happens-before C。

在3个happens-before关系中，B和C是必需的，但A是不必要的。因此，JMM把happens-before要求禁止的重排序分为了下面两类。

·会改变程序执行结果的重排序。

·不会改变程序执行结果的重排序。

JMM对这两种不同性质的重排序，采取了不同的策略，如下。

·对于会改变程序执行结果的重排序，JMM要求编译器和处理器必须禁止这种重排序。

·对于不会改变程序执行结果的重排序，JMM对编译器和处理器不做要求（JMM允许这种重排序）。

图3-33是JMM的设计示意图。![052315445292054](https://ws3.sinaimg.cn/large/006tNc79gy1g3bnkcxj1tj30mt0ngaek.jpg)

图3-33　JMM的设计示意图

从图3-33可以看出两点，如下。

·JMM向程序员提供的happens-before规则能满足程序员的需求。JMM的happens-before规则不但简单易懂，而且也向程序员提供了足够强的内存可见性保证（有些内存可见性保证其实并不一定真实存在，比如上面的A happens-before B）。

·JMM对编译器和处理器的束缚已经尽可能少。从上面的分析可以看出，JMM其实是在遵循一个基本原则：只要不改变程序的执行结果（指的是单线程程序和正确同步的多线程程序），编译器和处理器怎么优化都行。例如，如果编译器经过细致的分析后，认定一个锁只会被单个线程访问，那么这个锁可以被消除。再如，如果编译器经过细致的分析后，认定一个volatile变量只会被单个线程访问，那么编译器可以把这个volatile变量当作一个普通变量来对待。这些优化既不会改变程序的执行结果，又能提高程序的执行效率。

### 3.7.2　happens-before的定义

happens-before的概念最初由Leslie Lamport在其一篇影响深远的论文（《Time，Clocks and the Ordering of Events in a Distributed System》）中提出。Leslie Lamport使用happens-before来定义分布式系统中事件之间的偏序关系（partial ordering）。Leslie Lamport在这篇论文中给出了一个分布式算法，该算法可以将该偏序关系扩展为某种全序关系。

JSR-133使用happens-before的概念来指定两个操作之间的执行顺序。由于这两个操作可以在一个线程之内，也可以是在不同线程之间。因此，JMM可以通过happens-before关系向程序员提供跨线程的内存可见性保证（如果A线程的写操作a与B线程的读操作b之间存在happens-before关系，尽管a操作和b操作在不同的线程中执行，但JMM向程序员保证a操作将对b操作可见）。

《JSR-133:Java Memory Model and Thread Specification》对happens-before关系的定义如下。

1）如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。

2）两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。

上面的1）是JMM对程序员的承诺。从程序员的角度来说，可以这样理解happens-before关系：如果A happens-before B，那么Java内存模型将向程序员保证——A操作的结果将对B可见，且A的执行顺序排在B之前。注意，这只是Java内存模型向程序员做出的保证！

上面的2）是JMM对编译器和处理器重排序的约束原则。正如前面所言，JMM其实是在遵循一个基本原则：只要不改变程序的执行结果（指的是单线程程序和正确同步的多线程程序），编译器和处理器怎么优化都行。JMM这么做的原因是：程序员对于这两个操作是否真的被重排序并不关心，程序员关心的是程序执行时的语义不能被改变（即执行结果不能被改变）。因此，happens-before关系本质上和as-if-serial语义是一回事。

·as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变。

·as-if-serial语义给编写单线程程序的程序员创造了一个幻境：单线程程序是按程序的顺序来执行的。happens-before关系给编写正确同步的多线程程序的程序员创造了一个幻境：正确同步的多线程程序是按happens-before指定的顺序来执行的。

as-if-serial语义和happens-before这么做的目的，都是为了在不改变程序执行结果的前提下，尽可能地提高程序执行的并行度。

### 3.7.3　happens-before规则

《JSR-133:Java Memory Model and Thread Specification》定义了如下happens-before规则。

1）程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

2）监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

3）volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

4）传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

5）start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。

6）join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作

happens-before于线程A从ThreadB.join()操作成功返回。

这里的规则1）、2）、3）和4）前面都讲到过，这里再做个总结。由于2）和3）情况类似，这里只以1）、3）和4）为例来说明。图3-34是volatile写-读建立的happens-before关系图。

图3-34　happens-before关系的示意图![052315445292055](https://ws1.sinaimg.cn/large/006tNc79gy1g3bnt721r1j30kw0jdwh6.jpg)

结合图3-34，我们做以下分析。

·1 happens-before 2和3 happens-before 4由程序顺序规则产生。由于编译器和处理器都要遵守as-if-serial语义，也就是说，as-if-serial语义保证了程序顺序规则。因此，可以把程序顺序规则看成是对as-if-serial语义的“封装”。

·2 happens-before 3是由volatile规则产生。前面提到过，对一个volatile变量的读，总是能看到（任意线程）之前对这个volatile变量最后的写入。因此，volatile的这个特性可以保证实现volatile规则。

·1 happens-before 4是由传递性规则产生的。这里的传递性是由volatile的内存屏障插入策略和volatile的编译器重排序规则共同来保证的。

下面我们来看start()规则。假设线程A在执行的过程中，通过执行ThreadB.start()来启动线程B；同时，假设线程A在执行ThreadB.start()之前修改了一些共享变量，线程B在开始执行后会读这些共享变量。图3-35是该程序对应的happens-before关系图。![052315445292056](https://ws2.sinaimg.cn/large/006tNc79gy1g3bnv9hwfwj30l40j60vk.jpg)

图3-35　happens-before关系的示意图

在图3-35中，1 happens-before 2由程序顺序规则产生。2 happens-before 4由start()规则产生。根据传递性，将有1 happens-before 4。这实意味着，线程A在执行ThreadB.start()之前对共享变量所做的修改，接下来在线程B开始执行后都将确保对线程B可见。

下面我们来看join()规则。假设线程A在执行的过程中，通过执行ThreadB.join()来等待线程B终止；同时，假设线程B在终止之前修改了一些共享变量，线程A从ThreadB.join()返回后会读这些共享变量。图3-36是该程序对应的happens-before关系图。![052315445292057](https://ws1.sinaimg.cn/large/006tNc79gy1g3bnwzd9cjj30m60jgjuh.jpg)

图3-36　happens-before关系的示意图

在图3-36中，2 happens-before 4由join()规则产生；4 happens-before 5由程序顺序规则产生。根据传递性规则，将有2 happens-before 5。这意味着，线程A执行操作ThreadB.join()并成功返回后，线程B中的任意操作都将对线程A可见。

## 3.8　双重检查锁定与延迟初始化

在Java多线程程序中，有时候需要采用延迟初始化来降低初始化类和创建对象的开销。双重检查锁定是常见的延迟初始化技术，但它是一个错误的用法。本文将分析双重检查锁定的错误根源，以及两种线程安全的延迟初始化方案。

### 3.8.1　双重检查锁定的由来

在Java程序中，有时候可能需要推迟一些高开销的对象初始化操作，并且只有在使用这些对象时才进行初始化。此时，程序员可能会采用延迟初始化。但要正确实现线程安全的延迟初始化需要一些技巧，否则很容易出现问题。比如，下面是非线程安全的延迟初始化对象的示例代码。

```java
public class UnsafeLazyInitialization {
    private static Instance instance;
    public static Instance getInstance() {
        if (instance == null) // 1：A线程执行
            instance = new Instance(); // 2：B线程执行
        return instance;
    }
}
```

在UnsafeLazyInitialization类中，假设A线程执行代码1的同时，B线程执行代码2。此时，线程A可能会看到instance引用的对象还没有完成初始化（出现这种情况的原因见3.8.2节）。

对于UnsafeLazyInitialization类，我们可以对getInstance()方法做同步处理来实现线程安全的延迟初始化。示例代码如下。

```java
public class SafeLazyInitialization {
    private static Instance instance;
    public synchronized static Instance getInstance() {
        if (instance == null)
            instance = new Instance();
        return instance;
    }
}
```

由于对getInstance()方法做了同步处理，synchronized将导致性能开销。如果getInstance()方法被多个线程频繁的调用，将会导致程序执行性能的下降。反之，如果getInstance()方法不会被多个线程频繁的调用，那么这个延迟初始化方案将能提供令人满意的性能。

在早期的JVM中，synchronized（甚至是无竞争的synchronized）存在巨大的性能开销。因此，人们想出了一个“聪明”的技巧：双重检查锁定（Double-Checked Locking）。人们想通过双重检查锁定来降低同步的开销。下面是使用双重检查锁定来实现延迟初始化的示例代码。

```java
public class DoubleCheckedLocking {               // 1
  private static Instance instance;               // 2
  public static Instance getInstance() {          // 3
    if (instance == null) {                       // 4:第一次检查
      synchronized (DoubleCheckedLocking.class) { // 5:加锁
        if (instance == null)                   	// 6:第二次检查
          instance = new Instance();          		// 7:问题的根源出在这里
      }                                           // 8
    }                                             // 9
    return instance;                              // 10
  }                                               // 11
}
```

如上面代码所示，如果第一次检查instance不为null，那么就不需要执行下面的加锁和初始化操作。因此，可以大幅降低synchronized带来的性能开销。上面代码表面上看起来，似乎两全其美。

·多个线程试图在同一时间创建对象时，会通过加锁来保证只有一个线程能创建对象。

·在对象创建好之后，执行getInstance()方法将不需要获取锁，直接返回已创建好的对象。

双重检查锁定看起来似乎很完美，但这是一个错误的优化！在线程执行到第4行，代码读取到instance不为null时，instance引用的对象有可能还没有完成初始化。

### 3.8.2　问题的根源

前面的双重检查锁定示例代码的第7行（instance=new Singleton();）创建了一个对象。这一行代码可以分解为如下的3行伪代码。

```java
memory = allocate();		// 1：分配对象的内存空间
ctorInstance(memory);		// 2：初始化对象
instance = memory;			// 3：设置instance指向刚分配的内存地址
```

上面3行伪代码中的2和3之间，可能会被重排序（在一些JIT编译器上，这种重排序是真实发生的，详情见参考文献1的“Out-of-order writes”部分）。2和3之间重排序之后的执行时序如下。

```java
memory = allocate();		// 1：分配对象的内存空间
instance = memory;			// 3：设置instance指向刚分配的内存地址
												// 注意，此时对象还没有被初始化！
ctorInstance(memory);		// 2：初始化对象
```

根据《The Java Language Specification,Java SE 7 Edition》（后文简称为Java语言规范），所有线程在执行Java程序时必须要遵守intra-thread semantics。intra-thread semantics保证重排序不会改变单线程内的程序执行结果。换句话说，intra-thread semantics允许那些在单线程内，不会改变单线程程序执行结果的重排序。上面3行伪代码的2和3之间虽然被重排序了，但这个重排序并不会违反intra-thread semantics。这个重排序在没有改变单线程程序执行结果的前提下，可以提高程序的执行性能。

为了更好地理解intra-thread semantics，请看如图3-37所示的示意图（假设一个线程A在构造对象后，立即访问这个对象）。

如图3-37所示，只要保证2排在4的前面，即使2和3之间重排序了，也不会违反intra-threadsemantics。

下面，再让我们查看多线程并发执行的情况。如图3-38所示。![052315445292058](https://ws4.sinaimg.cn/large/006tNc79ly1g3c6d8q9y7j30nm0cy40x.jpg)

图3-37　线程执行时序图![052315445292059](https://ws3.sinaimg.cn/large/006tNc79ly1g3c6eddzktj30mm0ikdho.jpg)

图3-38　多线程执行时序图

由于单线程内要遵守intra-thread semantics，从而能保证A线程的执行结果不会被改变。但是，当线程A和B按图3-38的时序执行时，B线程将看到一个还没有被初始化的对象。

回到本文的主题，DoubleCheckedLocking示例代码的第7行（instance=new Singleton();）如果发生重排序，另一个并发执行的线程B就有可能在第4行判断instance不为null。线程B接下来将访问instance所引用的对象，但此时这个对象可能还没有被A线程初始化！表3-6是这个场景的具体执行时序。![052315445292060](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6flz6j9j30xh08sq53.jpg)

表3-6　多线程执行时序表

这里A2和A3虽然重排序了，但Java内存模型的intra-thread semantics将确保A2一定会排在A4前面执行。因此，线程A的intra-thread semantics没有改变，但A2和A3的重排序，将导致线程B在B1处判断出instance不为空，线程B接下来将访问instance引用的对象。此时，线程B将会访问到一个还未初始化的对象。

在知晓了问题发生的根源之后，我们可以想出两个办法来实现线程安全的延迟初始化。

1）不允许2和3重排序。

2）允许2和3重排序，但不允许其他线程“看到”这个重排序。

后文介绍的两个解决方案，分别对应于上面这两点。

### 3.8.3　基于volatile的解决方案

对于前面的基于双重检查锁定来实现延迟初始化的方案（指DoubleCheckedLocking示例代码），只需要做一点小的修改（把instance声明为volatile型），就可以实现线程安全的延迟初始化。请看下面的示例代码。

```java
public class SafeDoubleCheckedLocking {
    private volatile static Instance instance;
    public static Instance getInstance() {
        if (instance == null) {
            synchronized (SafeDoubleCheckedLocking.class) {
                if (instance == null){
                  instance = new Instance();//instance为volatile，现在没问题了
                } 
            }
        }
        return instance;
    }
}
```

![052315445292075](https://ws1.sinaimg.cn/large/006tNc79ly1g3c78o44jvj301901amwy.jpg)**注意**　这个解决方案需要JDK 5或更高版本（因为从JDK 5开始使用新的JSR-133内存模型规范，这个规范增强了volatile的语义）。

当声明对象的引用为volatile后，3.8.2节中的3行伪代码中的2和3之间的重排序，在多线程环境中将会被禁止。上面示例代码将按如下的时序执行，如图3-39所示。![052315445292062](https://ws3.sinaimg.cn/large/006tNc79ly1g3c6jbs1tkj30n90jnq4w.jpg)

图3-39　多线程执行时序图

这个方案本质上是通过禁止图3-39中的2和3之间的重排序，来保证线程安全的延迟初始化。

### 3.8.4　基于类初始化的解决方案

JVM在类的初始化阶段（即在Class被加载后，且被线程使用之前），会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。

基于这个特性，可以实现另一种线程安全的延迟初始化方案（这个方案被称之为Initialization On Demand Holder idiom）。

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    public static Instance getInstance() {
        return InstanceHolder.instance ;// 这里将导致InstanceHolder类被初始化
    }
}
```

假设两个线程并发执行getInstance()方法，下面是执行的示意图，如图3-40所示。![052315445292063](https://ws1.sinaimg.cn/large/006tNc79gy1g3c6mt9hxnj30r10gugpw.jpg)

图3-40　两个线程并发执行的示意图

这个方案的实质是：允许3.8.2节中的3行伪代码中的2和3重排序，但不允许非构造线程（这里指线程B）“看到”这个重排序。

初始化一个类，包括执行这个类的静态初始化和初始化在这个类中声明的静态字段。根据Java语言规范，在首次发生下列任意一种情况时，一个类或接口类型T将被立即初始化。

1）T是一个类，而且一个T类型的实例被创建。

2）T是一个类，且T中声明的一个静态方法被调用。

3）T中声明的一个静态字段被赋值。

4）T中声明的一个静态字段被使用，而且这个字段不是一个常量字段。

5）T是一个顶级类（Top Level Class，见Java语言规范的§7.6），而且一个断言语句嵌套在T内部被执行。

在InstanceFactory示例代码中，首次执行getInstance()方法的线程将导致InstanceHolder类被初始化（符合情况4）。

由于Java语言是多线程的，多个线程可能在同一时间尝试去初始化同一个类或接口（比如这里多个线程可能在同一时刻调用getInstance()方法来初始化InstanceHolder类）。因此，在Java中初始化一个类或者接口时，需要做细致的同步处理。

Java语言规范规定，对于每一个类或接口C，都有一个唯一的初始化锁LC与之对应。从C到LC的映射，由JVM的具体实现去自由实现。JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类已经被初始化过了（事实上，Java语言规范允许JVM的具体实现在这里做一些优化，见后文的说明）。

对于类或接口的初始化，Java语言规范制定了精巧而复杂的类初始化处理过程。Java初始化一个类或接口的处理过程如下（这里对类初始化处理过程的说明，省略了与本文无关的部分；同时为了更好的说明类初始化过程中的同步处理机制，笔者人为的把类初始化的处理过程分为了5个阶段）。

**第1阶段：通过在Class对象上同步（即获取Class对象的初始化锁），来控制类或接口的初始化。这个获取锁的线程会一直等待，直到当前线程能够获取到这个初始化锁。**

假设Class对象当前还没有被初始化（初始化状态state，此时被标记为state=noInitialization），且有两个线程A和B试图同时初始化这个Class对象。图3-41是对应的示意图。![052315445292064](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6sfzcbaj30m70e1dhf.jpg)

图3-41　类初始化——第1阶段

表3-7是这个示意图的说明。

表3-7　类初始化——第1阶段的执行时序表![052315445292065](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6ss8waxj30xj07dq5s.jpg)

**第2阶段：线程A执行类的初始化，同时线程B在初始化锁对应的condition上等待。**

表3-8是这个示意图的说明。

表3-8　类初始化——第2阶段的执行时序表![052315445292066](https://ws2.sinaimg.cn/large/006tNc79ly1g3c6tf5vxqj30xl05fabr.jpg)![052315445292067](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6ujelq4j30pk0ku0xd.jpg)

图3-42　类初始化——第2阶段

**第3阶段：线程A设置state=initialized，然后唤醒在condition中等待的所有线程。**![052315445292068](https://ws4.sinaimg.cn/large/006tNc79ly1g3c6wl5vdbj30ld0g0wgn.jpg)

图3-43　类初始化——第3阶段

表3-9是这个示意图的说明。![052315445292069](https://ws3.sinaimg.cn/large/006tNc79ly1g3c6wt5bmmj30w206aabq.jpg)

表3-9　类初始化——第3阶段的执行时序表

**第4阶段：线程B结束类的初始化处理。**![052315445292070](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6y89kr9j30k809fjsj.jpg)

图3-44　类初始化——第4阶段

表3-10是这个示意图的说明。

表3-10　类初始化——第4阶段的执行时序表![052315445292071](https://ws1.sinaimg.cn/large/006tNc79ly1g3c6ye5wvej30w1040t9s.jpg)![052315445292072](https://ws4.sinaimg.cn/large/006tNc79ly1g3c6zas43aj30md0dn762.jpg)图3-45　多线程执行时序图

线程A在第2阶段的A1执行类的初始化，并在第3阶段的A4释放初始化锁；线程B在第4阶段的B1获取同一个初始化锁，并在第4阶段的B4之后才开始访问这个类。根据Java内存模型规范的锁规则，这里将存在如下的happens-before关系。

这个happens-before关系将保证：线程A执行类的初始化时的写入操作（执行类的静态初始化和初始化类中声明的静态字段），线程B一定能看到。

**第5阶段：线程C执行类的初始化的处理。**

图3-46　类初始化——第5阶段![052315445292073](https://ws1.sinaimg.cn/large/006tNc79gy1g3c744zmuzj30kj0990ty.jpg)

表3-11是这个示意图的说明。

表3-11　类初始化——第5阶段的执行时序表![052315445292074](https://ws4.sinaimg.cn/large/006tNc79gy1g3c74drzqdj30w006jjsg.jpg)

在第3阶段之后，类已经完成了初始化。因此线程C在第5阶段的类初始化处理过程相对简单一些（前面的线程A和B的类初始化处理过程都经历了两次锁获取-锁释放，而线程C的类初始化处理只需要经历一次锁获取-锁释放）。

线程A在第2阶段的A1执行类的初始化，并在第3阶段的A4释放锁；线程C在第5阶段的C1获取同一个锁，并在在第5阶段的C4之后才开始访问这个类。根据Java内存模型规范的锁规则，将存在如下的happens-before关系。

这个happens-before关系将保证：线程A执行类的初始化时的写入操作，线程C一定能看到。

![052315445292075](https://ws3.sinaimg.cn/large/006tNc79ly1g3c78w07wrj301901amwy.jpg)**注意**　这里的condition和state标记是本文虚构出来的。Java语言规范并没有硬性规定一定要使用condition和state标记。JVM的具体实现只要实现类似功能即可。

**![052315445292075](https://ws4.sinaimg.cn/large/006tNc79gy1g3c75jvci3j301901amwy.jpg)注意**　Java语言规范允许Java的具体实现，优化类的初始化处理过程（对这里的第5阶段做优化），具体细节参见Java语言规范的12.4.2节。![052315445292076](https://ws1.sinaimg.cn/large/006tNc79gy1g3c75fcxxvj30mi0fgdhs.jpg)

图3-47　多线程执行时序图

通过对比基于volatile的双重检查锁定的方案和基于类初始化的方案，我们会发现基于类初始化的方案的实现代码更简洁。但基于volatile的双重检查锁定的方案有一个额外的优势：除了可以对静态字段实现延迟初始化外，还可以对实例字段实现延迟初始化。

字段延迟初始化降低了初始化类或创建实例的开销，但增加了访问被延迟初始化的字段的开销。在大多数时候，正常的初始化要优于延迟初始化。如果确实需要对实例字段使用线程安全的延迟初始化，请使用上面介绍的基于volatile的延迟初始化的方案；如果确实需要对静态字段使用线程安全的延迟初始化，请使用上面介绍的基于类初始化的方案。

## 3.9　Java内存模型综述

前面对Java内存模型的基础知识和内存模型的具体实现进行了说明。下面对Java内存模型的相关知识做一个总结。

### 3.9.1　处理器的内存模型

顺序一致性内存模型是一个理论参考模型，JMM和处理器内存模型在设计时通常会以顺序一致性内存模型为参照。在设计时，JMM和处理器内存模型会对顺序一致性模型做一些放松，因为如果完全按照顺序一致性模型来实现处理器和JMM，那么很多的处理器和编译器优化都要被禁止，这对执行性能将会有很大的影响。

根据对不同类型的读/写操作组合的执行顺序的放松，可以把常见处理器的内存模型划分为如下几种类型。

·放松程序中写-读操作的顺序，由此产生了Total Store Ordering内存模型（简称为TSO）。

·在上面的基础上，继续放松程序中写-写操作的顺序，由此产生了Partial Store Order内存模型（简称为PSO）。

·在前面两条的基础上，继续放松程序中读-写和读-读操作的顺序，由此产生了Relaxed Memory Order内存模型（简称为RMO）和PowerPC内存模型。

注意，这里处理器对读/写操作的放松，是以两个操作之间不存在数据依赖性为前提的（因为处理器要遵守as-if-serial语义，处理器不会对存在数据依赖性的两个内存操作做重排序）。

表3-12展示了常见处理器内存模型的细节特征如下。

表3-12　处理器内存模型的特征表![052315445292077](https://ws2.sinaimg.cn/large/006tNc79ly1g3c7es9536j30xh07uach.jpg)

从表3-12中可以看到，所有处理器内存模型都允许写-读重排序，原因在第1章已经说明过：它们都使用了写缓存区。写缓存区可能导致写-读操作重排序。同时，我们可以看到这些处理器内存模型都允许更早读到当前处理器的写，原因同样是因为写缓存区。由于写缓存区仅对当前处理器可见，这个特性导致当前处理器可以比其他处理器先看到临时保存在自己写缓存区中的写。

表3-12中的各种处理器内存模型，从上到下，模型由强变弱。越是追求性能的处理器，内存模型设计得会越弱。因为这些处理器希望内存模型对它们的束缚越少越好，这样它们就可以做尽可能多的优化来提高性能。

由于常见的处理器内存模型比JMM要弱，Java编译器在生成字节码时，会在执行指令序列的适当位置插入内存屏障来限制处理器的重排序。同时，由于各种处理器内存模型的强弱不同，为了在不同的处理器平台向程序员展示一个一致的内存模型，JMM在不同的处理器中需要插入的内存屏障的数量和种类也不相同。图3-48展示了JMM在不同处理器内存模型中需要插入的内存屏障的示意图。

JMM屏蔽了不同处理器内存模型的差异，它在不同的处理器平台之上为Java程序员呈现了一个一致的内存模型。![052315445292078](https://ws4.sinaimg.cn/large/006tNc79ly1g3c7hoom9wj30p10l6423.jpg)

图3-48　JMM插入内存屏障的示意图

### 3.9.2　各种内存模型之间的关系

JMM是一个语言级的内存模型，处理器内存模型是硬件级的内存模型，顺序一致性内存模型是一个理论参考模型。下面是语言内存模型、处理器内存模型和顺序一致性内存模型的强弱对比示意图，如图3-49所示。![052315445292079](https://ws2.sinaimg.cn/large/006tNc79ly1g3c7js41p2j30n30i7q67.jpg)

图3-49　各种CPU内存模型的强弱对比示意图

从图中可以看出：常见的4种处理器内存模型比常用的3中语言内存模型要弱，处理器内存模型和语言内存模型都比顺序一致性内存模型要弱。同处理器内存模型一样，越是追求执行性能的语言，内存模型设计得会越弱。

### 3.9.3　JMM的内存可见性保证

按程序类型，Java程序的内存可见性保证可以分为下列3类。

·单线程程序。单线程程序不会出现内存可见性问题。编译器、runtime和处理器会共同确保单线程程序的执行结果与该程序在顺序一致性模型中的执行结果相同。

·正确同步的多线程程序。正确同步的多线程程序的执行将具有顺序一致性（程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同）。这是JMM关注的重点，JMM通过限制编译器和处理器的重排序来为程序员提供内存可见性保证。

·未同步/未正确同步的多线程程序。JMM为它们提供了最小安全性保障：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0、null、false）。

注意，最小安全性保障与64位数据的非原子性写并不矛盾。它们是两个不同的概念，它们“发生”的时间点也不同。最小安全性保证对象默认初始化之后（设置成员域为0、null或false），才会被任意线程使用。最小安全性“发生”在对象被任意线程使用之前。64位数据的非原子性写“发生”在对象被多个线程使用的过程中（写共享变量）。当发生问题时（处理器B看到仅仅被处理器A“写了一半”的无效值），这里虽然处理器B读取到一个被写了一半的无效值，但这个值仍然是处理器A写入的，只不过是处理器A还没有写完而已。最小安全性保证线程读取到的值，要么是之前某个线程写入的值，要么是默认值（0、null、false）。但最小安全性并不保证线程读取到的值，一定是某个线程写完后的值。最小安全性保证线程读取到的值不会无中生有的冒出来，但并不保证线程读取到的值一定是正确的。

图3-50展示了这3类程序在JMM中与在顺序一致性内存模型中的执行结果的异同。![052315445292080](https://ws4.sinaimg.cn/large/006tNc79ly1g3c7mhzyepj30oj0j7tbm.jpg)

图3-50　3类程序的执行结果的对比图

只要多线程程序是正确同步的，JMM保证该程序在任意的处理器平台上的执行结果，与该程序在顺序一致性内存模型中的执行结果一致。

### 3.9.4　JSR-133对旧内存模型的修补

JSR-133对JDK 5之前的旧内存模型的修补主要有两个。

·增强volatile的内存语义。旧内存模型允许volatile变量与普通变量重排序。JSR-133严格限制volatile变量与普通变量的重排序，使volatile的写-读和锁的释放-获取具有相同的内存语义。

·增强final的内存语义。在旧内存模型中，多次读取同一个final变量的值可能会不相同。为此，JSR-133为final增加了两个重排序规则。在保证final引用不会从构造函数内逸出的情况下，final具有了初始化安全性。

## 3.10　本章小结

本章对Java内存模型做了比较全面的解读。希望读者阅读本章之后，对Java内存模型能够有一个比较深入的了解；同时，也希望本章可帮助读者解决在Java并发编程中经常遇到的各种内存可见性问题。