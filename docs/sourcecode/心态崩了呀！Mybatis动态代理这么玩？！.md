## 前言

假如有人问你这么几个问题，看能不能答上来

1. Mybatis Mapper 接口没有实现类，怎么实现的 SQL 查询
2. JDK 动态代理为什么不能对类进行代理（充话费送的问题）
3. 抽象类可不可以进行 JDK 动态代理（附加问题）

![](https://images-machen.oss-cn-beijing.aliyuncs.com/你说啥呢-表情包.jpeg?x-oss-process=image/resize,h_200,w_200)

答不上来的铁汁，证明 Proxy、Mybatis 源码还没看到位。不过没有关系，继续往下看就明白了

## 动态代理实战

众所周知哈，Mybatis 底层封装使用的 JDK 动态代理。说 Mybatis 动态代理之前，先来看一下平常我们写的动态代理 Demo，抛砖引玉

一般来说定义 JDK 动态代理分为三个步骤，如下所示

1. 定义代理接口
2. 定义代理接口实现类
3. 定义动态代理调用处理器

三步代码如下所示，玩过动态代理的小伙伴看过就能明白

```java
public interface Subject { // 定义代理接口
    String sayHello();
}

public class SubjectImpl implements Subject {  // 定义代理接口实现类
    @Override
    public String sayHello() {
        System.out.println(" Hello World");
        return "success";
    }
}

public class ProxyInvocationHandler implements InvocationHandler {  // 定义动态代理调用处理器
    private Object target;

    public ProxyInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(" 🧱 🧱 🧱 进入代理调用处理器 ");
        return method.invoke(target, args);
    }
}
```

写个测试程序，运行一下看看效果，同样是分三步

1. 创建被代理接口的实现类
2. 创建动态代理类，说一下三个参数
   - 类加载器
   - 被代理类所实现的接口数组
   - 调用处理器（调用被代理类方法，每次都经过它）
3. 被代理实现类调用方法

```java
public class ProxyTest {
    public static void main(String[] args) {
        Subject subject = new SubjectImpl();
        Subject proxy = (Subject) Proxy
                .newProxyInstance(
                        subject.getClass().getClassLoader(),
                        subject.getClass().getInterfaces(),
                        new ProxyInvocationHandler(subject));

        proxy.sayHello();
        /**
         * 打印输出如下
         * 调用处理器：🧱 🧱 🧱 进入代理调用处理器
         * 被代理实现类：Hello World
         */
    }
}
```

Demo 功能实现了，大致运行流程也清楚了，下面要针对原理实现展开分析

## 动态代理原理分析

从原理的角度上解析一下，上面动态代理测试程序是如何执行的

第一步简单明了，**创建了 Subject 接口的实现类**，也是我们常规的实现

第二步是创建被代理对象的动态代理对象。这里有朋友就问了，怎么证明这是个动态代理对象？如图所示

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210111133425968.png)

JDK 动态代理对象名称是有规则的，凡是经过 Proxy 类生成的动态代理对象，前缀必然是 **\$Proxy**，后面的数字也是名称组成部分

如果有小伙伴想要一探究竟，**关注 Proxy 内部类 ProxyClassFactory**，这里会有想要的答案

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210111134232414.png)

回归正题，继续看一下 ProxyInvocationHandler，**内部保持了被代理接口实现类的引用**，invoke 方法内部使用反射调用被代理接口实现类方法

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112101502099.png)

可以看出生成的动态代理类，继承了 Proxy 类，然后对 Subject 接口进行了实现，而实现方法 sayHello 中实际调用的是 ProxyInvocationHandler 的 invoke 方法

> 一不小心发现了 JDK 动态代理不能对类进行代理的原因 ^ ^

也就是说，当我们调用 `Subject#sayHello` 时，方法调用链是这样的

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210111140427269.png)

但是，Demo 里有被代理接口的实现类，Mybatis Mapper 没有，这要怎么玩

不知道不要紧，知道了估计也看不到这了，一起看下 mybatis 源码是怎么玩的

> mybatis version：3.4.x

## Mybatis 源码实现

不知道大家考没考虑过这么一个问题，**Mapper Mapper 为什么不需要实现类？**

假如说，我们项目使用的三层设计，Controller 控制请求接收，Service 负责业务处理，Mapper 负责数据库交互

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210111153124692.png)

Mapper 层也就是我们常说的数据库映射层，负责对数据库的操作，比如对数据的查询或者新增、删除等

大胆设想下，项目没有使用 Mybatis，需要在 Mapper 实现层写数据库交互，会写一些什么内容？

会写一些常规的 JDBC 操作，比如：

```java
// 装载Mysql驱动
Class.forName(driveName);
// 获取连接
con = DriverManager.getConnection(url, user, pass);
// 创建Statement
Statement state = con.createStatement();
// 构建SQL语句
String stuQuerySqlStr = "SELECT * FROM student";
// 执行SQL返回结果
ResultSet result = state.executeQuery(stuQuerySqlStr);
...
```

如果项目中所有 Mapper 实现层都要这么玩，那岂不是很想打人...

![](https://images-machen.oss-cn-beijing.aliyuncs.com/没意思.png?x-oss-process=image/resize,h_200,w_200)

所以 Mybatis 结合项目痛点，应运而生，怎么做的呢

1. 将所有和 JDBC 交互的操作，底层采用 JDK 动态代理封装，使用者只需要自定义 Mapper 和 .xml 文件
2. SQL 语句定义在 .xml 文件或者 Mapper 中，项目启动时通过解析器解析 SQL 语句组装为 Java 中的对象

> 解析器分为多种，因为 Mybatis 中不仅有静态语句，同时也包含动态 SQL 语句

这也就是为什么 Mapper 接口不需要实现类，**因为都已经被 Mybatis 通过动态代理封装了，如果每个 Mapper 都来一个实现类，臃肿且无用**。经过这一顿操作，展示给我们的就是项目里用到的 Mybatis 框架

上面铺垫这么久，终于要到主角了，**为什么 Mybatis Mapper 接口没有实现类也可以实现动态代理**

> 想要严格按照先后顺序介绍 Mybatis 动态代理流程，而不超前引用未介绍过的术语，这几乎是不可能的，笔者尽量说的通俗易懂

## 无实现类完成动态代理

核心点来了，拿起小本本坐板正了

![](https://images-machen.oss-cn-beijing.aliyuncs.com/快闪开，他要开始装逼了.jpg?x-oss-process=image/resize,h_300,w_300)

我们先来看下普通动态代理有没有可能不用实现类，仅靠接口完成

```java
public interface Subject {
    String sayHello();
}

public class ProxyInvocationHandler implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(" 🧱 🧱 🧱 进入代理调用处理器 ");
        return "success";
    }
}
```

根据代码可以看到，我们并没有实现接口 Subject，继续看一下怎么实现动态代理

```java
public class ProxyTest {
    public static void main(String[] args) {
        Subject proxy = (Subject) Proxy
                .newProxyInstance(
                        subject.getClass().getClassLoader(),
                        new Class[]{Subject.class},
                        new ProxyInvocationHandler());

        proxy.sayHello();
        /**
         * 打印输出如下
         * 调用处理器：🧱 🧱 🧱 进入代理调用处理器
         */
    }
}
```

可以看到，对比文初的 Demo，这里对 `Proxy.newProxyInstance` 方法的参数作出了变化

之前是通过实现类获取所实现接口的 Class 数组，而这里是把接口本身放到 Class 数组中，殊归同途

有实现接口和无实现接口产生的动态代理类有什么区别

1. 有实现接口是对 `InvocationHandler#invoke` 方法调用，invoke 方法通过反射调用被代理对象（SubjectImpl）方法（sayHello）
2. 无实现接口则是仅对 `InvocationHandler#invoke` 产生调用。**所以有接口实现返回的是被代理对象接口返回值，而无实现接口返回的仅是 invoke 方法返回值**

`InvocationHandler#invoke` 方法返回值是 success 字符串，定义个字符串变量，是否能成功返回

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112222532385.png)

现在第一个问题答案已经浮现，**Mapper 没有实现类，所有调用 JDBC 等操作都是在 Mybatis InvocationHandler 实现的**

问题既然已经得到了解决，给人一种感觉，好像没那么难，但是你不好奇，Mybatis 底层怎么做的么？

![](https://images-machen.oss-cn-beijing.aliyuncs.com/难搞啊.jpg)

先抛出一个问题，然后带着问题去看源码，可能让你记忆 Double 倍深刻

咱们 Demo 里的接口是固定的，Mybatis Mapper 可是不固定的，怎么搞？



**Mybatis 是这么说的**

![](https://images-machen.oss-cn-beijing.aliyuncs.com/我不觉得这是个问题.jpeg?x-oss-process=image/resize,h_300,w_300)

看看 Mybatis 底层它怎么实现的动态接口代理，小伙伴只需要关注标记处的代码即可

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112130314572.png)

和我们的 Demo 代码很像，核心点在于 **mapperInterface** 它是怎么赋值的

先来说一下 Mybatis 代理工厂中具体生成动态代理类具体逻辑

1. 根据 .xml 上关联的 namespace, 通过 `Class#forName` 反射的方式返回 Class 对象（不止 .xml namespace 一种方式）
2. 将得到的 Class 对象（实际就是接口对象）传递给 Mybatis 代理工厂生成代理对象，也就是刚才 mapperInterface 属性

谜底揭晓，Mybatis 使用接口全限定名通过 `Class#forName` 生成 Class 对象，这个 Class 对象类型就是接口

为了方便大家理解，通过 Mybatis 源码提供的测试类举例。假设已有接口 AutoConstructorMapper 以及对应的 .xml 如下

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112132212106.png)

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112132257008.png)

执行第一步，根据 .xml namespace 得到 Class 对象

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112132433980.png)

1. 首先第一步获取 .xml 上 mapper 标签 namespace 属性，得到 mapper 接口全限定信息
2. 根据 mapper 全限定信息获取 Class 对象
3. 添加到对应的映射器容器中，等待生成动态代理对象

如果此时调用生成动态代理对象，代理工厂 newInstance 方法如下：

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112131357019.png)

至此，文初提的 Proxy、Mybatis 动态代理相关问题已全部答疑

## 抽象类能否 JDK 动态代理

说代码前结论先行，**不能！**

```java
public abstract class AbstractProxy {
    abstract void sayHello();
}

AbstractProxy proxyInterface = (AbstractProxy) Proxy
        .newProxyInstance(
                ProxyTest.class.getClassLoader(),
                new Class[]{AbstractProxy.class},
                new ProxyInvocationHandler());
proxyInterface.sayHello();
```

毫无疑问，报错是必然的，JDK 是不能对类进行代理的

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112133350848.png)

带着小疑惑我们看一下 Proxy 源码报错位置，JDK 动态代理在生成代理类的过程代码中，会有是否接口验证

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210112133513069.png)

抽象类终归是类，加个 abstract 也成不了接口（就像我，虽然胖了 60 斤，但依然是帅哥）

![](https://images-machen.oss-cn-beijing.aliyuncs.com/嘻嘻嘻.jpg)

下次面试官如果有问这问题的，**斩钉截铁一点**，就是不能

## 结言

结合 Mybatis 使用 JDK 动态代理相关的问题，展开了文章的讲述，这里总结下

**Q：JDK 动态代理能否对类代理？**

> 因为 JDK 动态代理生成的代理类，会继承 Proxy 类，由于 Java 无法多继承，所以无法对类进行代理

**Q：抽象类是否可以 JDK 动态代理？**

> 不可以，抽象类本质上也是类，Proxy 生成代理类过程中，会校验传入 Class 是否接口

**Q：Mybatis Mapper 接口没有实现类，怎么实现的动态代理？**

> Mybatis 会通过 `Class#forname` 得到 Mapper 接口 Class 对象，生成对应的动态代理对象，核心业务处理都会在 `InvocationHandler#invoke` 进行处理



希望读过的小伙伴都能有所收获，如果对于文章内容有所疑惑，可以通过留言或者添加作者好友的方式沟通，祝好!

