## 前言

继 SpringCloud Feign 之后的第二篇分布式框架文章，同样秉承单周末一个 SpringCloud 组件的大目标为原则

如果想看 Feign 的小伙伴，[猛戳这里](https://mp.weixin.qq.com/s/zYfhJDyHji58Skw3IqIWzg)，Feign 核心原理与你不期而遇

在平常使用 SpringCloud 中，一般会使用 Feign，因为 Feign 内部集成了 Ribbon

但是 Ribbon 又是一个不可忽视的知识点，并且比 Feign 要难很多。列举文章大纲主题

> 1. 如何获取注册中心服务实例
> 2. 非健康服务实例如何下线
> 3. Ribbon 底层原理实现
> 4. 自定义 Ribbon 负载均衡策略

文章使用 SpringCloud Ribbon 源代码 Hoxton.SR9 版本：**2.2.6.RELEASE**

_另外在文章结尾，说了一些看源码过程中的感想，以及 Ribbon 中笔者实现不合理的流程说明_

## 概念小贴士

### 负载均衡

负载均衡是指通过负载均衡策略分配到多个执行单元上，常见的负载均衡方式有两种

- 独立进程单元，通过负载均衡策略，将请求进行分发到不同执行上，类似于 Nginx
- 客户端行为，将负载均衡的策略绑定到客户端上，客户端会维护一份服务提供者列表，通过客户端负载均衡策略分发到不同的服务提供者

### Ribbon

Ribbon 是 Netflix 公司开源的一款负载均衡组件，负载均衡的行为在客户端发生，所以属于上述第二种

一般而言，SpringCloud 构建以及使用时，会使用 Ribbon 作为客户端负载均衡工具。但是不会独立使用，而是结合 RestTemplate 以及 Feign 使用，Feign 底层集成了 Ribbon，不用额外的配置，开箱即用

文章为了更贴切 Ribbon 主题，所以使用 RestTemplate 充当网络调用工具

RestTemplate 是 Spring Web 下提供访问第三方 RESTFul Http 接口的网络框架

## 环境准备

注册中心选用阿里 Nacos，创建两个服务，生产者集群启动，消费者使用 RestTemplate + Ribbon 调用，调用总体结构如下

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210122182633581.png)

生产者代码如下，将服务注册 Nacos，并对外暴露 Http Get 服务

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210123112352094.png)

消费者代码如下，将服务注册 Nacos，通过 RestTemplate + Ribbon 发起远程负载均衡调用

RestTemplate 默认是没有负载均衡的，所以需要添加 @LoadBalanced

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210123112532490.png)

启动三个生产者实例注册 Nacos，启动并且注册成功如下所示

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210123123103613.png)

想要按严格的先后顺序介绍框架原理，而不超前引用尚未介绍过的术语，**这几乎是不可能的**，笔者尽可能介绍明白

## 如何获取注册中心服务实例

先来看一下 Ribbon 是如何在客户端获取到注册中心运行实例的，这个点在之前是我比较疑惑的内容

> 服务注册相关的知识点，会放到 Nacos 源码解析说明

先来举个例子，当我们执行一个请求时，肯定要进行负载均衡对吧，这个时候代码跟到负载均衡获取服务列表源码的地方

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124134853454.png)

解释一下上面标黄色框框的地方：

- RibbonLoadBalancerClient：负责负载均衡的请求处理
- ILoadBalancer：接口中定义了一系列实现负载均衡的方法，相当于一个路由的作用，Ribbon 中默认实现类 ZoneAwareLoadBalancer
- unknown：ZoneAwareLoadBalancer 是多区域负载均衡器，这个 unkonwn 代表默认区域的意思
- allServerList：代表了从 Nacos 注册中心获取的接口服务实例，upServerList 代表了健康实例

现在想要知道 Ribbon 是如何获取服务实例的就需要跟进 **getLoadBalancer()**

### getLoadBalancer

首先声明一点，getLoadBalancer() 方法的语意是从 Ribbon 父子上下文容器中获取名称为 **ribbon-produce**，类型为 **ILoadBalancer.class** 的 Spring Bean

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124141655558.png)

之前在讲 Feign 的时候说过，Ribbon 会为每一个服务提供者创建一个 Spring 父子上下文，这里会从子上下文中获取 Bean

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124142007712.png)

看到这里并没有解决我们的疑惑，以为方法里会有拉取服务列表的代码，然鹅只是返回一个包含了服务实例的 Bean，所以我们只能去跟下这个 Bean 的上下文

我们需要从负载均衡客户端着手，因为默认是 ZoneAwareLoadBalancer，那我们需要跟进它何时被创建，初始化都做了什么事情

### ZoneAwareLoadBalancer

ZoneAwareLoadBalancer 是一个根据区域（Zone）来进行负载均衡器，因为如果不同机房跨区域部署服务列表，跨区域的方式访问会产生更高的延迟，ZoneAwareLoadBalancer 就是为了解决此类问题，不过默认都是同一区域

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124171342217.png)

ZoneAwareLoadBalancer 很重要，或者说它代表的 **负载均衡路由角色** 很重要。进行服务调用前，会使用该类根据负载均衡算法获取可用 Server 进行远程调用，所以我们要掌握创建这个负载均衡客户端时都做了哪些

ZoneAwareLoadBalancer 是在服务第一次被调用时通过子容器创建

```java
@Bean @ConditionalOnMissingBean  // RibbonClientConfiguration 被加载，从 IOC 容器中获取对应实例填充到 ZoneAwareLoadBalancer
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
                                        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
                                        IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
    ...
    return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
            serverListFilter, serverListUpdater);
}

public ZoneAwareLoadBalancer(IClientConfig clientConfig, IRule rule,
                             IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,
                             ServerListUpdater serverListUpdater) {
    // 调用父类构造方法
    super(clientConfig, rule, ping, serverList, filter, serverListUpdater);
}
```


在 DynamicServerListLoadBalancer 中调用了父类 BaseLoadBalancer 初始化了一部分配置以及方法，另外自己也初始化了 Server 服务列表等元数据

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                     ServerList<T> serverList, ServerListFilter<T> filter,
                                     ServerListUpdater serverListUpdater) {
    // 调用父类 BaseLoadBalancer 初始化一些配置，包括 Ping（检查服务是否可用）Rule（负载均衡规则）
    super(clientConfig, rule, ping);  
    // 较重要，获取注册中心服务的接口
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    // 初始化步骤分了两步走，第一步在上面，这一步就是其余的初始化
    restOfInit(clientConfig);
}
```

先来说一下 BaseLoadBalancer 中初始化的方法，这里主要对一些重要参数以及 Ping、Rule 赋值，另外根据 IPing 实现类执行定时器，下面介绍 Ping 和 Rule 是什么

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124173636638.png)

方法大致做了以下几件事情：

1. 设置客户端配置对象、名称等关键参数
2. 获取每次 Ping 的间隔以及 Ping 的最大时间
3. 设置具体负载均衡规则 IRule，默认 ZoneAvoidanceRule，根据 server 和 zone 区域来轮询
4. 设置具体 Ping 的方式，默认 DummyPing，直接返回 True
5. 根据 Ping 的具体实现，执行定时任务 Ping Server

这里会介绍被填入的 IPing 和 IRule 是什么东东，并且都有哪些实现

### IPing 服务探测

IPing 接口负责向 Server 实例发送 ping 请求，判断 Server 是否有响应，以此来判断 Server 是否可用

接口只有一个方法 isAlive，通过实现类完成探测 ping 功能

```java
public interface IPing {
    public boolean isAlive(Server server);
}
```

IPing 实现类如下：

- PingUrl：通过 ping 的方式，发起网络调用来判断 Server 是否可用（一般而言创建 PingUrl 需要指定路径，默认是 IP + Port）
- PingConstant：固定返回某服务是否可用，默认返回 True，表示可用
- NoOpPing：没有任何操作，直接返回 True，表示可用
- DummyPing：默认的类，直接返回 True，实现了 initWithNiwsConfig 方法

### IRule 负载均衡

IRule 接口负责根据不用的算法和逻辑处理负载均衡的策略，自带的策略有 7 种，默认 ZoneAvoidanceRule

1. BestAvailableRule：选择服务列表中最小请求量的 Server
2. RandomRule：服务列表中随机选择 Server
3. RetryRule：根据轮询的方式重试 Server
4. ZoneAvoidanceRule：根据 Server 的 Zone 区域和可用性轮询选择 Server
5. ...

上面说过，会有两个初始化步骤，刚才只说了一个，接下来说一下 这个其余初始化方法 `restOfInit`，虽然取名叫其余初始化，但是就重要性而言，那是相当重要

```java
void restOfInit(IClientConfig clientConfig) {
    boolean primeConnection = this.isEnablePrimingConnections();
    // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
    this.setEnablePrimingConnections(false);
    // 初始化服务列表，并启用定时器，对服务列表作出更新
    enableAndInitLearnNewServersFeature();
    // 更新服务列表，enableAndInitLearnNewServersFeature 中定时器的执行的就是此方法
    updateListOfServers();
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections()
                .primeConnections(getReachableServers());
    }
    this.setEnablePrimingConnections(primeConnection);
    LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
}
```


获取服务列表以及定时更新服务列表的代码都在此处，值得仔细看着源码。关注其中更新服务列表方法就阔以了

```java
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        // 获取服务列表数据
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);
        }
    }
    // 更新所有服务列表
    updateAllServerList(servers);
}
```

第一个问题兜兜转转，终于要找到如何获取的服务列表了，**serverListImpl 实现自 ServerList**，因为我们使用的 Nacos 注册中心，所以 ServerList 的具体实现就是 **NacosServerList**

```java
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers();
    public List<T> getUpdatedListOfServers();
}
```

ServerList 中只有两个接口方法，分别是 **获取初始化服务列表集合、获取更新的服务列表集合**，Nacos 实现中两个调用都是一个实现方法，可能设计如此

相当于 Ribbon 提供出接口 ServerList，注册中心开发者们谁想和 Ribbon 集成，那你就实现这个接口吧，到时候 Ribbon 负责调用 ServerList 实现类中的方法实现

> Ribbon 和各服务注册中心之间，这种实现方式和 JDBC 与各数据库之间很像

兜兜转转中问题已经明朗，一起总结下注册中心获取服务实例这块内容

1. 负载均衡客户端在初始化时向 **Nacos 注册中心获取服务注册列表信息**
2. 根据不同的 IPing 实现，向获取到的服务列表 **串行发送 ping**，以此来判断服务的可用性。没错，就是串行，如果你的实例很多，可以 **考虑重写 ping 这一块的逻辑**
3. 如果服务的可用性 **发生了改变或者被人为下线**，那么重新拉取或更新服务列表
4. 当负载均衡客户端有了这些服务注册类列表，自然就可以进行 **IRule 负载均衡策略**

## 非健康服务实例如何下线

首先笔者做了两个 **"大胆" 的实验**，第一次是对生产者 SpringBoot 项目执行关闭流程，这时候 Nacos 注册中心是 **实时感知到并将此服务下该实例删除**

证明 Nacos 客户端是有 **类似于钩子函数的存在**，感知项目停止就会向 Nacos 服务端注销实例。但是这个时候要考虑一件事情，那就是在 **暴力 Kill 或者执行关闭操作** 的情况下，**存在于 Ribbon 客户端服务列表缓存能不能感知**

第二次我这边测试流程是这样的，可以极大程度还原生产上使用 Ribbon 可能会遇到的问题

1. 改变客户端负载均衡策略为 **随机负载 RandomRule**，大家自己可以进行测试，不固定负载规则
2. 注册三个生产者服务实例到 Nacos 上，检查 **确保服务组下实例正常注册**
3. 操作重点来了，先通过消费方实例请求下对应的生产者接口，保证 **Ribbon 将对应 Server 缓存到客户端**
4. 停掉一个生产者服务，此时 **马上使用 Jmeter 调用**，Jmeter 线程组发起请求 100 次（一定要赶到更新 Server 缓存之前发起 Jmeter 请求）
5. 这时就会看到会发生随机失败，也就是说停掉一个服务后，**最坏结果会有 30 秒的生产服务不可用**，这个时间可配置，后面会讲到为什么 30 秒

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210125103040301.png)

### 服务列表定时维护

针对于服务列表的维护，在 Ribbon 中有两种方式，都是通过定时任务的形式维护客户端列表缓存

1. 使用 IPing 的实现类 PingUrl，**每隔 10 秒会去 Ping 服务地址**，如果返回状态不是 200，那么默认该实例下线
2. Ribbon 客户端内置的扫描，**默认每隔 30 秒去拉取 Nacos 也就是注册中心的服务实例**，如果已下线实例会在客户端缓存中剔除

这一块源码都不贴了，放两个源代码位置，感兴趣自己看看就行了

```diff
+ DynamicServerListLoadBalancer#enableAndInitLearnNewServersFeature
+ BaseLoadBalancer#setupPingTask
```

如果你面试的时候，面试官问了本小节相关内容，把这两个点都能答出来，基本上 SpringCloud 源码就差不多了

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210125180520015.png)

## Ribbon 底层原理实现

底层原理实现这一块内容，会先说明使用 Ribbon **负载均衡调用远端请求的全过程**，然后着重看一下 RandomRule **负载策略底层是如何实现**

1. 创建 ILoadBalancer 负载均衡客户端，初始化 Ribbon 中所需的 **定时器和注册中心上服务实例列表**
2. 从 ILoadBalancer 中，通过 **负载均衡选择出健康服务列表中的一个 Server**
3. **将服务名（ribbon-produce）替换为 Server 中的 IP + Port**，然后生成 HTTP 请求进行调用并返回数据

上面已经说过，ILoadBalancer 是负责负载均衡路由的，内部会使用 IRule 实现类进行负载调用

```java
public interface ILoadBalancer {
    public Server chooseServer(Object key);
  	...
}
```

chooseServer 流程中调用的就是 IRule 负载策略中的 choose 方法，在方法内部获取一个健康 Server

```java
public Server choose(ILoadBalancer lb, Object key) {
    ... 
    Server server = null;
    while (server == null) {
        ...
        List<Server> upList = lb.getReachableServers();  // 获取服务列表健康实例
        List<Server> allList = lb.getAllServers();  // 获取服务列表全部实例
        int serverCount = allList.size();  // 全部实例数量
        if (serverCount == 0) {  // 全部实例数量为空，返回 null，相当于错误返回
            return null;
        }
        int index = chooseRandomInt(serverCount);  // 考虑到效率问题，使用多线程 ThreadLocalRandom 获取随机数
        server = upList.get(index);  // 获取健康实例
        if (server == null) {
            // 作者认为出现获取 server 为空，证明服务列表正在调整，但是！这只是暂时的，所以当前释放出了 CPU
            Thread.yield();
            continue;
        }
        if (server.isAlive()) {  // 服务为健康，返回
            return (server);
        }
        ...
    }
    return server;
}
```

简单说一下随机策略 choose 中流程

1. 获取到全部服务、健康服务列表，**判断全部实例数量是否等于 0**，是则返回 null，相当于发生了错误
2. 从全部服务列表里获取下标索引，然后去 **健康实例列表获取 Server**
3. 如果获取到的 Server 为空会放弃 CPU，然后再来一遍上面的流程，**相当于一种重试机制**
4. 如果获取到的 Server 不健康，设置 Server 等于空，再歇一会，继续走一遍上面的流程

比较简单，有小伙伴可能就问了，如果健康实例小于全部实例怎么办？这种情况下存在两种可能

1. 运气比较好，从全部实例数量中随机了比较小的数，刚好健康实例列表有这个数，那么返回 Server
2. 运气比较背，从全部实例数量中随机了某个数，健康实例列表数量为空或者小于这个数，直接会下标越界异常

留下一个思考题：

**为什么不直接从健康实例中选择实例呢**

如果直接从健康实例列表选择，就能规避下标越界异常，为什么作者要先从全部实例中获取 Server 下标？

## 自定义 Ribbon 负载均衡策略

这种自定义策略，在框架中都支持的比较友好，根据上面提的问题，我们自定义一款策略

```java
@Slf4j
public class MyRule extends AbstractLoadBalancerRule {
    @Override
    public Server choose(Object key) {
        ILoadBalancer loadBalancer = getLoadBalancer();
        while (true && ) {
            Server server = null;
            // 获取已启动并且可访问的服务列表
            List<Server> reachableServers = loadBalancer.getReachableServers();
            if (CollectionUtils.isEmpty(reachableServers)) return null;
            int idx = ThreadLocalRandom.current().nextInt(reachableServers.size());
            server = reachableServers.get(idx);
            if (server == null || server.isAlive()) {
                log.warn("Ribbon 服务实例异常, 获取为空 || 状态不健康");
                Thread.yield();
                continue;
            }
            return server;
        }
    }

    ... initWithNiwsConfig 不用实现
}
```

说一下我们自己实现的 MyRule 负载的逻辑：

1. **IRule 获取服务列表没有在调用方实现**，而是抽象 AbstractLoadBalancerRule，所以我们要获取服务列表继承就好了
2. 和随机负载规则大致相似，只不过这里简化了流程，**直接从健康的服务实例列表获取 Server 实例**
3. **确定 Server 不为空并且节点健康后返回**，如果不符合则打印日志，睡一会再重复
4. 如果保险起见，**最好在 while 中加入一个循环次数的条件**，避免死循环

然后把 MyRule 注册到 SPring IOC 容器中就好了，在初始化时就会代替默认的 Rule 负载规则


## 关于 IPing 的思考

在阅读 Ribbon Ping 这一块源代码时，发现了两处个人认为不太合理的地方

1. **setPingInterval** 设置 Ping 间隔时执行设置 Ping 任务无意义
2. **BaseLoadBalancer** 构造函数中 ping 为 null，**又再次调用了 setPingInterval**，结果只会返回空

setPingInterval 和 setPing 两个方法发生在 BaseLoadBalancer 初始化时，相当于接着上面逻辑继续。先说明执行逻辑，再看下不合理的地方

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124185708642.png)

setupPingTask 用于定期执行对 Server 的 ping 任务，也就是检测 Server 是否可用

个人觉得在 setPingInterval 中没有必要执行 setupPingTask 方法

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124190554174.png)

作出上述结论有以下依据：

1. 第一次执行 setPingInterval 时，ping 必然为空，那么会在 canSkipPing 中返回 True，继而直接结束 setPingInterval 方法
2. 后来想了下，会不会在别的地方引用，需要强制刷新，然鹅全局搜索了下引用，只有在此次初始化时调用，当然不排除别的依赖包会使用此方法
3. 综上所述，setPingInterval 执行设置 Ping 任务的方法无意义

另外还有一点，作者感觉代码中调用的方法没有实际意义。和上述类似，都是在 ping 为空时执行了 setPingInterval 方法

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20210124192423863.png)

以上这两点是笔者跟源码时，发现不妥当的地方，所以在这里占了一些篇幅说明，主要是想表达两点自己的想法给读者朋友

1. **不要对源码有敬畏之心，应该有敬畏之心的是生产环境！** 不要感觉看框架源码是一件高不可攀的事情，其实有时候你理解不了的代码，可能只是多人维护后，混乱的产物，条件允许的情况下，还是要多跟进源码去看一看
2. **直言说出自己的见解**，如果只有自己去想，那么很可能没有答案，通过文章间接的方式让更多小伙伴看到，指正错误言论亦或者得到肯定

## 结言

整体来看，文章更 **注重表达设计思想以及源码分析**，所以阅读文章需要一定的源码功底。同时文章是针对问题而展开叙述，哪怕源码不理解也能有所收获

Ribbon 这块内容从初始化负载均衡客户端 ILoadBalancer 说起，讲述了初始化过程中具体的内容，包括如何开启 IPing 定时器以及服务列表更新定时器

另外通过源码查看到 Ribbon 的服务列表其实是向 **Nacos 提供的接口发起服务调用** 获取并保存到本地缓存，继而牵引出如何保证不健康实例下线：**IPing 定时器和服务更新定时器**

文末章节说了下请求 Ribbon 负载均衡的全链路以及如何自己定义一个负载均衡算法。最最后面也说了下自己看源码过程中对 SpringCloud IPing 感觉无意义的代码，*当然，不排除是为了别的包集成而留下的*