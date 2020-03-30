# 单例模式

单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

**注意：**

- 1、单例类只能有一个实例。
- 2、单例类必须自己创建自己的唯一实例。
- 3、单例类必须给所有其他对象提供这一实例。

# 实现

通常单例模式在[Java语言](https://baike.baidu.com/item/Java%E8%AF%AD%E8%A8%80)中，有以下构建方式：

- 饿汉方式。指全局的单例实例在类装载时构建。
- 懒汉方式。指全局的单例实例在第一次被使用时构建。
- 静态内部类。利用静态内部类的特性：静态内部类在外部类对象初始化前加载。
- 注册登记式。用一个Map通过key维护单一实例。

## 1、饿汉式

饿的意思就是要吃，类装载时构实例

优点：天生的线程安全，比较简单

缺点：可能浪费资源

```java
/**
 * @description 饿汉式单例
 * @auther yanzhaoyao
 * @date 2018/12/1 11:37
 */
public class Hunger {

    //构造函数私有化，保证外界不能实例化
    private Hunger() {
    }

    //全局的单例实例在类装载时构建
    private static Hunger hunger = new Hunger();

    public static Hunger getInstance() {
        return hunger;
    }
}
```

不测试了，比较简单

## 2、懒汉式，线程不安全

懒的意思，就是需要的时候在实例化，延迟加载。比较懒。

在多线程的环境下，getInstance（）可能会有多个线程同时进入到if语句块中，造成lazy不能保持唯一的实例。

因此破坏了单例模式，所谓的线程不安全

```java
/**
 * @description 懒汉式单例(线程不安全)
 * @auther yanzhaoyao
 * @date 2018/12/1 17:39
 */
public class Lazy {

    //构造函数私有化，确保外界无法通过new 实例化
    private Lazy() {
    }

    public static Lazy lazy;

    public static Lazy getInstance() {
        if (lazy == null) {
            lazy = new Lazy();
        }
        return lazy;
    }
}
```

测试

```java
/**
 * @description 懒汉式，线程不安全
 * @auther yanzhaoyao
 * @date 2018/12/1 17:51
 */
public class LazyTest {

    private static int COUNT = 200;

    public static void main(String[] args) {
        //懒汉式，线程不安全
//        Lazy lazy = new Lazy();//此处不能通过new 来实例化，因为构造函数私有化
        CyclicBarrier cyclicBarrier = new CyclicBarrier(COUNT);

        for (int i = 0; i < COUNT; i++) {
            new Thread(()->{
                try {
                    try {
                        cyclicBarrier.await();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                    Lazy lazy = Lazy.getInstance();
                    System.out.println(lazy);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

输出结果，出现了多个不同的实例

```java
com.yanzhaoyao.patter.singleton.Lazy@7449d4b
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
......
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
com.yanzhaoyao.patter.singleton.Lazy@1aa42512
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
com.yanzhaoyao.patter.singleton.Lazy@3f0e4bd
```

## 3、懒汉式，线程安全

此种方式通过添加关键字**synchronized**实现了线程安全，但这种方式效率极低，因为多线程环境下，绝大部分时间式不需要同步的

```java
/**
 * @description 懒汉式单例(线程安全)
 * @auther yanzhaoyao
 * @date 2018/12/1 17:39
 */
public class Lazy2 {

    //构造函数私有化，确保外界无法通过new 实例化
    private Lazy2() {
    }

    public static Lazy2 lazy;

    public static synchronized Lazy2 getInstance() {
        if (lazy == null) {
            lazy = new Lazy2();
        }
        return lazy;
    }
```

测试

```java
package com.yanzhaoyao.patter.singleton;

import java.util.concurrent.CountDownLatch;

/**
 * @description 懒汉式，线程安全
 * @auther yanzhaoyao
 * @date 2018/12/1 17:51
 */
public class Lazy2Test {

    private static int COUNT = 200;

    public static void main(String[] args) {
        
        CyclicBarrier cyclicBarrier = new CyclicBarrier(COUNT);

        for (int i = 0; i < COUNT; i++) {
            new Thread(() -> {
                try {
                    try {
                        cyclicBarrier.await();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                    Lazy2 lazy = Lazy2.getInstance();
                    System.out.println(lazy);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

输出结果

```
com.yanzhaoyao.patter.singleton.Lazy2@774c0c70
com.yanzhaoyao.patter.singleton.Lazy2@774c0c70
......//省略中间的
com.yanzhaoyao.patter.singleton.Lazy2@774c0c70
com.yanzhaoyao.patter.singleton.Lazy2@774c0c70
```

## 4、懒汉式，线程安全，（DCL）

（Double Check Lock）双重校验锁机制，为处理以上版本非延迟加载方式瓶颈问题，我们需要对 lazy 进行第二次检查，目的是避开过多的同步（因为这里的同步只需在第一次创建实例时才同步，一旦创建成功，以后获取实例时就不需要获取锁了），但在Java中行不通，**因为同步块外面的if (lazy == null)可能看到已存在，但不完整的实例。**

(解释一下

这行代码  **lazy = new LazyDCL();**

`它可以分解成三个操作，`

`【伪代码】3`

`1.memory = allocate()`

`2.createInstance(memory)`

`3.resource = memory`

`其中1,2指令有数据依赖，所以不会被重排序。而2,3指令没有数据依赖，如果没有volatile关键字，可能会被重排序。`

`那么假如2，3执行顺序进行了调换。`

`那么就有可能发生，假设A,B线程都即将执行getResource操作，目前在A线程，`

`首先A线程第一次判断resource是否为null，结果为null，那么加锁进入执行创建对象的这三步。`

`假设执行了上述的1，3后，发生了调度，B开始执行。`

`**此时它面临的状态是，判断resource是否为null，结果不为null.因为刚才A执行了3已经给resource赋值了。**`

`**那么B会认为resource已经初始化完成，它可能要对这个对象进行一些操作，但是事实上A还没有执行2操作，resource对象还没有初始化完成。**`

`这样运行下去可能会产生异常，风险由此产生。`

[参考这里](https://www.cnblogs.com/vete-l/p/7107700.html))

Jdk5.0以后版本若lazy为**volatile**则可行（禁止指令重排）：

```java
/**
 * @description 懒汉式单例(线程安全) DCL双重校验锁，效率较高
 * @auther yanzhaoyao
 * @date 2018/12/1 17:39
 */
public class LazyDCL {

    //构造函数私有化，确保外界无法通过new 实例化
    private LazyDCL() {
    }

    public volatile static LazyDCL lazy;

    public static LazyDCL getInstance() {
        if (lazy == null) {
            synchronized (LazyDCL.class) {
                if (lazy == null) {
                    lazy = new LazyDCL();
                }
            }
        }
        return lazy;
    }
}
```

后面会对所有的单例实现方式进行性能测试

## 5、静态内部类

```java
/**
 * @description 单例模式-静态内部类
 * @auther yanzhaoyao
 * @date 2018/12/1 19:09
 */
public class StaticSingleton {
    private StaticSingleton() {
        //只会初始化一次
        System.out.println("StaticSingleton()");
    }

    public static StaticSingleton getInstance() {
        return LazyHolder.LAZY;
    }

    //默认不加载,在外部类加载时，该静态内部类仅加载到内存中一次
    private static class LazyHolder {
        private static final StaticSingleton LAZY = new StaticSingleton();
    }
}
```

测试

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/1 19:20
 */
public class Test {
    private static int COUNT = 200;

    public static void main(String[] args) {
        for (int i = 0; i < COUNT; i++) {
            StaticSingleton.getInstance();
        }
    }
}
```

输出结果：（一调用了一次构造函数）

```
StaticSingleton()
```

## 6、注册登记式

```java
/**
 * @description 单例模式-注册登记式
 * 登记式单例实际上维护的是一组单例类的实例，将这些实例存储到一个Map(登记簿)
 * 中，对于已经登记过的单例，则从工厂直接返回，对于没有登记的，则先登记，而后
 * 返回
 * @auther yanzhaoyao
 * @date 2018/12/1 19:32
 */
public class RegSingleton {

    /**
     * 登记簿，用来存放所有登记的实例
     * 用ConcurrentHashMap来维护映射关系，这是线程安全的
     */
    private static Map<String, Object> map = new ConcurrentHashMap<>();

    //在类加载时添加一个实例到登记簿
    static {
        RegSingleton singleton = new RegSingleton();
        map.put(singleton.getClass().getName(), singleton);//运用了反射
    }

    /**
     * 受保护的默认构造方法
     */
    protected RegSingleton() {

    }

    /**
     * 静态工厂方法，返回指定登记对象的唯一实例
     * 对于已经登记的直接取出返回，对于还未登记的先登记，然后取出返回
     */
    public static Object getInstance(String className) {
        if (className == null) {
            className = RegSingleton.class.getName();
        }
        //如果没有登记就用反射new一个
        if (!map.containsKey(className)) {
            //没有登记就进入同步块
            synchronized(RegSingleton.class){
                //再次检测是否登记
                if (!map.containsKey(className)) {
                    try {
                        //实例化对象
                        map.put(className,Class.forName(className).newInstance());
                    } catch (InstantiationException e) {
                        e.printStackTrace();
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
        return map.get(className);
    }
}
```

测试

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/1 22:41
 */
public class RegSingletonTest {
    static CyclicBarrier cyclicBarrier=new CyclicBarrier(1000);
    public static void main(String[] args) {
        for (int i = 0; i <1000 ; i++) {
            int n = i;
            new Thread(()->{
                System.out.println("线程"+ n +"准备就绪");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(RegSingleton.getInstance("com.yanzhaoyao.patter.singleton.register.ClassA"));
            }).start();
        }
    }
}
```

结果是线程安全的（ClassA是一个空类，里面什么也没有）

注册登记式单例[参考这里](https://www.cnblogs.com/twoheads/p/9723543.html)

# 测试各种方式的效率

我们来测试一下不同方式的效率：

现在我们用200个线程，每个线程分别 创建一百万个各种实例，所耗费的时间

代码如下：

```java
/**
 * @description 效率测试
 * @auther yanzhaoyao
 * @date 2018/12/1 17:51
 */
public class EfficientTest {

    private static int THREAD_COUNT = 200;
    private static int COUNT = 1000000;

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch count = new CountDownLatch(THREAD_COUNT);

        long start = System.currentTimeMillis();
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        for (int j = 0; j < COUNT; j++) {
                            Hunger.getInstance();
//                            Lazy.getInstance();
//                            Lazy2.getInstance();
//                            LazyDCL.getInstance();
//                            StaticSingleton.getInstance();
                        }
                        count.countDown();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        count.await();
        long end = System.currentTimeMillis();
        System.out.println("总共耗时:" + (end - start) + "毫秒");
    }
}
```

结果比较（供参考）

| 单例模式类型           | 每次耗时                  | 平均耗时 | 说明                       |
| ---------------------- | ------------------------- | -------- | -------------------------- |
| 饿汉式                 | 783ms、767ms、722ms       | 757ms    |                            |
| 懒汉式（线程不安全）   | 1136ms、2760ms、715ms     | 2305ms   | 这个线程不安全，没什么意义 |
| 懒汉式（线程安全）     | 24844ms、30281ms、27018ms | 27381ms  | 这个就比较耗时了           |
| 懒汉式（线程安全 DCL） | 790ms、1861ms、1198ms     | 1283ms   |                            |
| 静态内部类式           | 750ms、386ms、479ms       | 538ms    |                            |

# 反射和序列化都可以破坏单例

## 反射

```java
/**
 * @description 反射测试
 * @auther yanzhaoyao
 * @date 2018/12/2 22:41
 */
public class ReflactTest {
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        // 正常测试
        System.out.println(LazyDCL.getInstance());
        System.out.println(LazyDCL.getInstance());

        //反射测试
        Class<?> lazyClass = LazyDCL.class;

        //通过反射拿到私有的构造方法
        Constructor c = lazyClass.getDeclaredConstructor(null);
        //设置构造函数的访问权限，必须有
        c.setAccessible(true);
        System.out.println(c.newInstance());
        System.out.println(c.newInstance());
    }
}
```

输出结果

```
com.yanzhaoyao.patter.singleton.lazy.Lazy@4554617c
com.yanzhaoyao.patter.singleton.lazy.Lazy@4554617c
com.yanzhaoyao.patter.singleton.lazy.Lazy@74a14482
com.yanzhaoyao.patter.singleton.lazy.Lazy@1540e19d
```

破坏了单例，解决办法

在调用构造方法时，先判断实例是否存在，如果已存在则不允许再创建新的对象

```java
//构造函数私有化，确保外界无法通过new 实例化
private LazyDCL() {
    synchronized (LazyDCL.class){
        if(lazy != null){
            throw new RuntimeException("单例已被侵犯");
        }
    }
}
```

在此测试结果

```java
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.yanzhaoyao.patter.singleton.ReflactTest.main(ReflactTest.java:27)
Caused by: java.lang.RuntimeException: 单例已被侵犯
	at com.yanzhaoyao.patter.singleton.lazy.LazyDCL.<init>(LazyDCL.java:14)
	... 5 more
```

## 反序列化

```java
/**
 * @description 反序列化 破坏单例
 * @auther yanzhaoyao
 * @date 2018/12/2 23:03
 */
public class SeriableTest {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        // 正常测试
        System.out.println(LazyDCL.getInstance());
        System.out.println(LazyDCL.getInstance());

        //反序列化
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("D:\\a.txt"));
        oos.writeObject(LazyDCL.getInstance());
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("D:\\a.txt"));
        LazyDCL lazyDCL = (LazyDCL) ois.readObject();
        ois.close();
        System.out.println(lazyDCL);

    }
}
```

注意LazyDCL要实现序列化接口

```java
public class LazyDCL implements Serializable {
```

输出结果如下

```
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c

com.yanzhaoyao.patter.singleton.lazy.LazyDCL@568db2f2
```

**反序列化的解决方法**

添加一个Object类型的回调方法readResolve()

```java
public class LazyDCL implements Serializable {
    ....
    // 在反序列化时，直接调用返回对象，而不是返回新的对象
    private Object readResolve() {
        return lazy;
    }
    ...
}
```

输出结果

```
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c
com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c

com.yanzhaoyao.patter.singleton.lazy.LazyDCL@4554617c
```

