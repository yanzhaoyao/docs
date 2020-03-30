# 抽象工厂模式

抽象工厂模式（Abstract Factory Pattern）是围绕一个超级工厂创建其他工厂。该超级工厂又称为其他工厂的工厂。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

在抽象工厂模式中，接口是负责创建一个相关对象的工厂，不需要显式指定它们的类。每个生成的工厂都能按照工厂模式提供对象。

## 介绍

**意图：**提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。

**主要解决：**主要解决接口选择的问题。

**何时使用：**系统的产品有多于一个的产品族，而系统只消费其中某一族的产品。

**如何解决：**在一个产品族里面，定义多个产品。

**关键代码：**在一个工厂里聚合多个同类产品。

**应用实例：**工作了，为了参加一些聚会，肯定有两套或多套衣服吧，比如说有商务装（成套，一系列具体产品）、时尚装（成套，一系列具体产品），甚至对于一个家庭来说，可能有商务女装、商务男装、时尚女装、时尚男装，这些也都是成套的，即一系列具体产品。假设一种情况（现实中是不存在的，要不然，没法进入共产主义了，但有利于说明抽象工厂模式），在您的家中，某一个衣柜（具体工厂）只能存放某一种这样的衣服（成套，一系列具体产品），每次拿这种成套的衣服时也自然要从这个衣柜中取出了。用 OO 的思想去理解，所有的衣柜（具体工厂）都是衣柜类的（抽象工厂）某一个，而每一件成套的衣服又包括具体的上衣（某一具体产品），裤子（某一具体产品），这些具体的上衣其实也都是上衣（抽象产品），具体的裤子也都是裤子（另一个抽象产品）。

**优点：**当一个产品族中的多个对象被设计成一起工作时，它能保证客户端始终只使用同一个产品族中的对象。

**缺点：**产品族扩展非常困难，要增加一个系列的某一产品，既要在抽象的 Creator 里加代码，又要在具体的里面加代码。

**使用场景：** 1、QQ 换皮肤，一整套一起换。 2、生成不同操作系统的程序。

**注意事项：**产品族难扩展，产品等级易扩展。

## 实现

我们将创建 *IMobilePhone* 和 *IMobilePhoneShell* 接口和实现这些接口的实体类。下一步是创建抽象工厂类 *AbstractFactory*。接着定义工厂类 *PhoneFactory* 和 *PhoneShellFactory*，这两个工厂类都是扩展了 *AbstractFactory*。然后创建一个工厂创造器/生成器类 *FactoryProducer*。

*AbstractFactoryPatternDemo*，我们的演示类使用 *FactoryProducer* 来获取 *AbstractFactory* 对象。它将向 *AbstractFactory* 传递形状信息 *IMobilePhone*（*HUAWEI / Iphone / Vivo*），以便获取它所需对象的类型。同时它还向 *AbstractFactory* 传递颜色信息 *Color*（*Red / Green / Blue*），以便获取它所需对象的类型。





### 步骤 1

为手机创建一个接口。

**IMobilePhone.java**

```java
/**
 * 手机接口
 */
public interface IMobilePhone {
    /**
     *  发布手机
     */
    void publish();
}
```

### 步骤 2

创建实现接口的实体类。

**HUAWEI.java**

```java
public class HUAWEI implements IMobilePhone {

    @Override
    public void publish() {
        System.out.println("发布：HUAWEI Mate 20");
    }
}
```

**IPhone.java**

```java
public class Iphone implements IMobilePhone {
    @Override
    public void publish() {
        System.out.println("发布：Iphone XS");
    }
}
```

**vivo.java**

```java
public class Vivo implements IMobilePhone {
    @Override
    public void publish() {
        System.out.println("发布：vivo X23");
    }
}
```

### 步骤 3

为手机壳创建一个接口。

**IMobilePhoneShell.java**

```java
/**
 * 手机壳接口
 */
public interface IMobilePhoneShell {
    /**
     *  手机壳颜色
     */
    void color();
}
```

### 步骤4

创建实现接口的实体类。

**Red.java**

```java
public class Red implements IMobilePhoneShell {
    @Override
    public void color() {
        System.out.println("红色手机壳");
    }
}
```

**Green.java**

```java
public class Green implements IMobilePhoneShell {
    @Override
    public void color() {
        System.out.println("绿色手机壳");
    }
}
```

**Blue.java**

```java
public class Blue implements IMobilePhoneShell {
    @Override
    public void color() {
        System.out.println("蓝色手机壳");
    }
}
```

### 步骤 5

为 IMobilePhone和 IMobilePhoneShell 对象创建抽象类来获取工厂。

**AbstractFactory.java**

```java
public abstract class AbstractFactory {
    public abstract IMobilePhone getPhone(String trademark);
    public abstract IMobilePhoneShell getPhoneShell(String color);
}
```

### 步骤 6

创建扩展了 AbstractFactory 的工厂类，基于给定的信息生成实体类的对象。

**PhoneFactory.java**

```java
public class PhoneFactory extends AbstractFactory {
    @Override
    public IMobilePhone getPhone(String trademark) {
        if(trademark == null){
            return null;
        }
        if(trademark.equalsIgnoreCase("iphone")){
            return new Iphone();
        }
        if(trademark.equalsIgnoreCase("huawei")){
            return new HUAWEI();
        }
        if(trademark.equalsIgnoreCase("vivo")){
            return new Vivo();
        }
        return null;
    }

    @Override
    public IMobilePhoneShell getPhoneShell(String color) {
        return null;
    }
}
```

**PhoneShellFactory.java**

```java
public class PhoneShellFactory extends AbstractFactory {
    @Override
    public IMobilePhone getPhone(String trademark) {
        return null;
    }

    @Override
    public IMobilePhoneShell getPhoneShell(String color) {
        if(color == null){
            return null;
        }
        if(color.equalsIgnoreCase("red")){
            return new Red();
        }
        if(color.equalsIgnoreCase("green")){
            return new Green();
        }
        if(color.equalsIgnoreCase("blue")){
            return new Blue();
        }
        return null;
    }
}
```

### 步骤 7

创建一个工厂创造器/生成器类，通过传递手机 或 手机壳信息来获取相应工厂。

## FactoryProducer.java

```java
public class FactoryProducer {
    public static AbstractFactory getFactory(String choice){
        if(choice.equalsIgnoreCase("phone")){
            return new PhoneFactory();
        } else if(choice.equalsIgnoreCase("phoneshell")){
            return new PhoneShellFactory();
        }
        return null;
    }
}
```

### 步骤 8

使用 FactoryProducer 来获取 AbstractFactory，通过传递类型信息来获取实体类的对象。

**AbstractFactoryPatternDemo.java**

```java
public class AbstractFactoryPatternDemo {
    public static void main(String[] args) {
        //获取一个手机工厂
        AbstractFactory phoneFactory = FactoryProducer.getFactory("phone");

        //获取iphone对象，并发布手机publish（）
        IMobilePhone iphone = phoneFactory.getPhone("iphone");
        iphone.publish();

        //获取huawei对象，并发布手机publish（）
        IMobilePhone huawei = phoneFactory.getPhone("huawei");
        huawei.publish();

        //获取huawei对象，并发布手机publish（）
        IMobilePhone vivo = phoneFactory.getPhone("vivo");
        vivo.publish();

        //获取一个手机壳工厂
        AbstractFactory phoneShellFactory = FactoryProducer.getFactory("phoneshell");

        //获取iphone对象，并发布手机publish（）
        IMobilePhoneShell phoneShell = phoneShellFactory.getPhoneShell("red");
        phoneShell.color();

        //获取huawei对象，并发布手机publish（）
        IMobilePhoneShell green = phoneShellFactory.getPhoneShell("green");
        green.color();

        //获取huawei对象，并发布手机publish（）
        IMobilePhoneShell blue = phoneShellFactory.getPhoneShell("blue");
        blue.color();
    }
}
```

### 步骤 9

执行程序，输出结果：

```
发布：Iphone XS
发布：HUAWEI Mate 20
发布：vivo X23
红色手机壳
绿色手机壳
蓝色手机壳
```
