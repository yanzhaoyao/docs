# 原型模式

原型模式（Prototype Pattern）是用于创建重复的对象，同时又能保证性能。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

这种模式是实现了一个原型接口，该接口用于创建当前对象的克隆。当直接创建对象的代价比较大时，则采用这种模式。例如，一个对象需要在一个高代价的数据库操作之后被创建。我们可以缓存该对象，在下一个请求时返回它的克隆，在需要的时候更新数据库，以此来减少数据库调用。

# 实现

java中的克隆需要实现 java.lang.Cloneable 接口

## 浅克隆

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/3 20:40
 */
public class Prototype implements Cloneable {

    String name;

    Parameter parameter;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Parameter getParameter() {
        return parameter;
    }

    public void setParameter(Parameter parameter) {
        this.parameter = parameter;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/3 20:40
 */
public class Parameter{

    String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}
```

测试代码

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/3 21:00
 */
public class CloneTest {

    public static void main(String[] args) {

        Prototype p = new Prototype();
        p.setName("张三");
        Parameter parameter = new Parameter();
        parameter.setName("测试一");
        p.setParameter(parameter);

        System.out.println("原型---------------------------------");
        System.out.println(p.getName() + "|" + p.getParameter().getName() + "|" + p.getParameter());

        try {
            Prototype obj = (Prototype) p.clone();
            System.out.println("浅克隆后--------------------------");
            System.out.println(obj.getName() + "|" + obj.getParameter().getName() + "|" + obj.getParameter());

            obj.setName("李四");
            obj.getParameter().setName("测试二");
            System.out.println("修改浅克隆对象后------------------");
            System.out.println(p.getName() + "|" + p.getParameter().getName() + "|" + p.getParameter());
            System.out.println(obj.getName() + "|" + obj.getParameter().getName() + "|" + obj.getParameter());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出结果

```java
原型---------------------------------
张三|测试一|com.yanzhaoyao.pattern.prototype.simple.Parameter@4554617c
浅克隆后--------------------------
张三|测试一|com.yanzhaoyao.pattern.prototype.simple.Parameter@4554617c
修改浅克隆对象后------------------
张三|测试二|com.yanzhaoyao.pattern.prototype.simple.Parameter@4554617c
李四|测试二|com.yanzhaoyao.pattern.prototype.simple.Parameter@4554617c
```

克隆后的对象，成员变量 parameter，与原对象的成员变量parameter为同一对象（地址引用相同）。 

我们修改克隆后的对象的属性时，原型对象的属性也跟着变了，所谓的**浅克隆**，这不是我们想要的。看看下面的深克隆

## 深克隆

只要修改**Prototype.java**的clone方法即可

```java
/**
 * @description
 * @auther yanzhaoyao
 * @date 2018/12/3 20:40
 */
public class Prototype implements Cloneable, Serializable {

    String name;

    Parameter parameter;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Parameter getParameter() {
        return parameter;
    }

    public void setParameter(Parameter parameter) {
        this.parameter = parameter;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return this.deepClone();
    }

    public Object deepClone() {
        try {

            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(this);

            ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bis);

            Prototype copy = (Prototype) ois.readObject();

            return copy;

        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

当然Parameter也要实现序列化接口

```java
public class Parameter implements Serializable {
```

测试代码不变，输出结果

```
原型---------------------------------
张三|测试一|com.yanzhaoyao.pattern.prototype.deep.Parameter@4554617c
浅克隆后--------------------------
张三|测试一|com.yanzhaoyao.pattern.prototype.deep.Parameter@7b23ec81
修改浅克隆对象后------------------
张三|测试一|com.yanzhaoyao.pattern.prototype.deep.Parameter@4554617c
李四|测试二|com.yanzhaoyao.pattern.prototype.deep.Parameter@7b23ec81
```

可见，我们修改克隆后的对象也不会影响原来的原型对象。这就是**深克隆**

