---
title: Trino 二开遇到的坑
catalog: true
p: blog\Trino-second-dev
date: 2019-09-19 11:37:55
subtitle:
header-img:
tags:
---
> Trino 二开遇到的坑
# Trino 二开遇到的坑



## 类型转化异常

### 背景

如大家所知，原生的 Trino 不支持热加载 catalog，每次需要修改或者增加 catalog 时都需要通过修改配置文件并重启整个集群才可以。因此团队内部对此缺点进行了热加载 catalog 功能的开发。总的来说，功能开发没什么太大的难点，主要也是调用 ``ConnectorManager`` 的两个接口：

- 创建 catalog

  io.trino.connector.ConnectorManager#createCatalog(java.lang.String, java.lang.String, java.util.Map<java.lang.String,java.lang.String>)

- 删除 catalog

  io.trino.connector.ConnectorManager#dropConnection

具体如何实现这里就不再赘述了，网上的资料很多。

---



### 小事故

当我在测试 Trino 热加载功能中的增加与删除时，我并没有发现有什么问题。但当我测试更新（添加+删除+添加）时问题发生了。

在我添加完 catalog 后进行查询没有问题，再对 catalog 更新完毕之后通过 cli 查询更新后的 catalog 中的数据时，Trino 内部抛出了一个 class cast exception。

```
trino> select * from "hive_jier".default.pt;

Query 20220126_062145_58564_ebi8q, FAILED, 1 node
Splits: 17 total, 0 done (0.00%)
2.45 [0 rows, 0B] [0 rows/s, 0B/s]

Query 20220126_062145_58564_ebi8q failed: class io.trino.plugin.hive.HiveTableHandle cannot be cast to class io.trino.plugin.hive.HiveTableHandle (io.trino.plugin.hive.HiveTableHandle is in unnamed module of loader io.trino.server.PluginClassLoader @d8a0539; io.trino.plugin.hive.HiveTableHandle is in unnamed module of loader io.trino.server.PluginClassLoader @376a70a5)
```



登陆 Trino web ui 查看查询报错堆栈信息

```
java.lang.ClassCastException: class io.trino.plugin.hive.HiveTableHandle cannot be cast to class io.trino.plugin.hive.HiveTableHandle (io.trino.plugin.hive.HiveTableHandle is in unnamed module of loader io.trino.server.PluginClassLoader @d8a0539; io.trino.plugin.hive.HiveTableHandle is in unnamed module of loader io.trino.server.PluginClassLoader @376a70a5)
	at io.trino.plugin.hive.HivePageSourceProvider.createPageSource(HivePageSourceProvider.java:136)
	at io.trino.plugin.base.classloader.ClassLoaderSafeConnectorPageSourceProvider.createPageSource(ClassLoaderSafeConnectorPageSourceProvider.java:49)
	at io.trino.split.PageSourceManager.createPageSource(PageSourceManager.java:64)
	at io.trino.operator.TableScanOperator.getOutput(TableScanOperator.java:306)
	at io.trino.operator.Driver.processInternal(Driver.java:387)
	at io.trino.operator.Driver.lambda$processFor$9(Driver.java:291)
	at io.trino.operator.Driver.tryWithLock(Driver.java:683)
	at io.trino.operator.Driver.processFor(Driver.java:284)
	at io.trino.execution.SqlTaskExecution$DriverSplitRunner.processFor(SqlTaskExecution.java:1076)
	at io.trino.execution.executor.PrioritizedSplitRunner.process(PrioritizedSplitRunner.java:163)
	at io.trino.execution.executor.TaskExecutor$TaskRunner.run(TaskExecutor.java:484)
	at io.trino.$gen.Trino_8e8d962____20211221_070841_2.run(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
```

---



### 事故分析

#### 报错信息

查看上述堆栈信息我们可以知道问题发生在 ``HivePageSourceProvider`` 的 136 行，可以看到这里对 ``tableHandle`` 进行了一个强转， ``tableHandle`` 确实是一个 ``HiveTableHandle`` 对象但又为什么会强转失败呢。

<img src="https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220126150601159.png" alt="image-20220126150601159" style="zoom:30%;" />



这里涉及到类加载的机制了。每一个类在被类加载器加载之后会在堆中创建一个 class 对象，但是同一个类被多个不同的类加载器加载之后是会在堆中创建多个不同的 class 对象，对于 jvm 而言，这几个 class 对象就是完全不一样的类，因此由这几个 class 对象产生的对象也是不能强转的。

这个 class 对象我们一般通过双亲委派机制来确保程序中的每一个类最终只被一个类加载器加载。 但是 Trino 生成 connector 实例时，对于每一个 connector 实例都会创建一个新的类加载器去加载所需要的类并初始化 connector，所以同一种 connector 不同的 connector 实例之间的类对象也是不一样的。

至此基本可以断定这个 tableHandle 对象强转失败是因为两者的类加载器不同导致的。



#### 问题深入

再回到报错堆栈往下看，可以看到调用 ``HivePageSourceProvider.createPageSource`` 方法的是 ``ClassLoaderSafeConnectorPageSourceProvider.createPageSource`` 。

> ClassLoaderSafeConnectorPageSourceProvider 是 trino 内部用于保证类加载器正确的一个 PageSourceProvider，他会对 ConnectorPageSourceProvider 进行包装，在调用 ConnectorPageSourceProvider 相关方法之前，先会将当前线程的类加载器切换到创建该 connector 实例的类加载器上。

由于 ``ClassLoaderSafeConnectorPageSourceProvider`` 对类加载器进行切换，切换到创建该 connector 实例的类加载器上（以下简称类加载器 A），因此可以确保之后所有使用到的类都是类加载器 A 加载的。所以在 ``HivePageSourceProvider.createPageSource`` 中使用的 ``HiveTableHandle`` 类对象也是类加载器 A 加载的。

那 ``tableHandle`` 对象的类是哪来的呢？是由哪个类加载器加载的呢？这就需要继续跟着堆栈信息继续排查，这里直接给出结论：

这个 ``tableHandle`` 对象是由 worker 的 createOrUpdateTask Api 收到的 json 反序列化出来的。

```java
io.trino.server.TaskResource#createOrUpdateTask
```

coordinator 节点将任务调度给 worker 执行时，通过向 http 发送 ``POST：/v1/task/{taskId}`` 请求将 ``TaskUpdateRequest`` 传递给 worker 节点。

``TaskUpdateRequest`` 中包含执行计划片段 ``PlanFragment`` ，``PlanFragment`` 包含具体的查询计划节点 ``PlanNode`` 。对于报错的 sql，该  ``PlanNode`` 是一个 ``TableScanNode`` ,而 ``TableScanNode`` 中则包含 ``TableHandle`` 对象，该  ``TableHandle`` 对象则封装了一个 ``ConnectorTableHandle`` 对象，这个 ``ConnectorTableHandle`` 对象就是我们关键的 tableHandle 对象。

既然 tableHandle 对象是通过反序列化出来的。那反序列话出来的 tableHandle 对象的所属的类的类加载器是什么呢？这就需要我们去了解下 trino 的反序列话机制。



#### Trino 通信-序列化与反序列化

##### AirLift 序列化机制

Trino 几乎所有操作都依赖 AirLift 框架构建的 RESTful 服务来完成（数据传输，节点通信，心跳感应，计算调度，计算分布等）。包括4类 RESTful 接口，包括 Statement，Query，Stage，Task。

AirLift 的数据传输格式以 Json 为主，对 Json 的支持比较完善。

在 AirLift 中，它提供了一个 ``JsonMapper`` 支持 Jersey 的编解码。``JsonMapper`` 内部使用的 json 解析框架是 ``jackson``, 包含了一个 ``ObjectMapper``，所有的 json 都有这个 ``ObjectMapper`` 进行序列化与反序列化。

<img src="https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220127114019209.png" alt="image-20220127114019209" style="zoom:40%;" />

 ``JsonMapper`` 的构造函数上有个 Inject 注解，说明这个 ``ObjectMapper`` 是通过 guice 注入的。而这个 ``ObjectMapper`` 则是由实现了 guice provider 接口的 ``io.airlift.json.ObjectMapperProvider`` 提供。

```java
public class ObjectMapperProvider
        implements Provider<ObjectMapper>
{
    private final Set<Module> modules = new HashSet<>();

    @Inject(optional = true)
    public void setModules(Set<Module> modules)
    {
        this.modules.addAll(modules);
    }

    @Override
    public ObjectMapper get()
    {
        ObjectMapper objectMapper = new ObjectMapper(jsonFactory);
      	...
        for (Module module : modules) {
            objectMapper.registerModule(module);
        }

        return objectMapper;
    }
}

```

通过 ``io.airlift.json.ObjectMapperProvider#get`` 方法我们可以发现它在这里进行了一个 ``com.fasterxml.jackson.databind.Module`` 的注入以及注册。而这个 ``Module`` 有什么用呢，通过阅读 ``com.fasterxml.jackson.databind.ObjectMapper#registerModule`` 方法的注解我们可以知道我们可以通过这个方法向 ``ObjectMapper`` 添加自定义的序列化与反序列化器。

<img src="https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220127115420214.png" alt="image-20220127115420214" style="zoom:50%;" />



##### Trino 基于 AirLift 的扩展

###### AbstractTypedJacksonModule

``io.trino.metadata.AbstractTypedJacksonModule`` 是 Trino 提供的内部的一个用于扩展自定义序列化和反序列化基类。

<img src="https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220127133554505.png" alt="image-20220127133554505" style="zoom:40%;" />



``io.trino.metadata.AbstractTypedJacksonModule`` 的构造函数有三个参数和一个常量 ``TYPE_PROPERTY``：

- baseClass

  进行序列化的基类

- nameResolver

  nameResolver 是一个 Function 的实现类，接收一个类型或父类型是 baseClass 的对象，然后返回一个字符串。

- classResolver

  nameResolver 是一个 Function 的实现类，接收一个字符串，返回一个类型是 baseClass 子类的类对象。

在构造函数里面，``io.trino.metadata.AbstractTypedJacksonModule`` 还会向 Module 添加一个 ``InternalTypeSerializer`` 用于 baseClass 的序列化器和一个 ``InternalTypeDeserializer`` 用于 baseClass 的反序列化器。

``InternalTypeSerializer`` 在对待序列化的对象进行序列化之前，会待序列化的对象传给 nameResolver 获得一个字符串 key，然后将对象序列化为 json 并会在 json 串中加入一个 kv 值，k 为 ``TYPE_PROPERTY``，v 为字符串 key。

``InternalTypeDeserializer`` 在对待序列化的 json 进行反序列化时，先会去解析 json 串中的 ``TYPE_PROPERTY ``并获取到字符串 key，然后将字符串 key 传进 classResolver 获得一个类对象 Class<? extends T>， 然后安装反序列话规则将 json 反序列化为类为 Class<? extends T> 的对象。



###### HandleJsonModule

Trino 在这里实现了 AbstractTypedJacksonModule，并以 ``@ProvidesIntoSet`` 注解的形式通过 guice 注入到上文的 ``ObjectMapper`` 里面，再在 ``ObjectMapper`` 进行注册，也就是上文所说的口子。

![image-20220127142840021](https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220127142840021.png)

> 如果细心的同学会发现 ``io.trino.metadata.AbstractTypedJacksonModule`` 中也没用需要进行实现的接口，这里是使用匿名内部类来对 ``io.trino.metadata.AbstractTypedJacksonModule`` 进行了一个空实现。既然是空实现，那为什么不直接将 ``io.trino.metadata.AbstractTypedJacksonModule`` 改为普通的类的。这是因为 ``io.trino.metadata.AbstractTypedJacksonModule``  是一个泛型类，这么做是为了防止泛型的类型擦除。

这里的 ``nameResolver`` 与 ``classResolver`` 都是 ``HandleResolver`` 的方法。

- nameResolver

  ``nameResolver`` 具体的逻辑是遍历  ``HandleResolver``  中的 ``MaterializedHandleResolver`` 集合，并将传入的对象与 ``MaterializedHandleResolver`` 中的 ``ConnectorTableHandle`` 类对象进行比较，如果传入的对象的类与 ``ConnectorTableHandle`` 类对象相同则返回 catalogName

  <img src="https://gitee.com/kyotom/ImageBed/raw/master/%20markdown-pic/image-20220127143833398.png" alt="image-20220127143833398" style="zoom:40%;" />

  

- classResolver

   ``classResolver`` 具体的逻辑是根据传入的 catalogName 从 ``HandleResolver`` 中获取该 catalog 对应的 ``MaterializedHandleResolver`` ，并将 ``MaterializedHandleResolver`` 中的 ``ConnectorTableHandle`` 类对象取出返回。

由此可以看出 ``HandleJsonModule`` 的作用：**由于不同的 connector 实例是由不同的类加载器去加载的。而在对 json 进行反序列化后，某些对象可能会被用于 connector 内部的一些处理（比如上面的 ``HiveTableHandle``）。如果按 ``ObjectMapper`` 默认的反序列化逻辑去反序列化出对象则这些对象一旦被强转为 connector 内部的类就会带来 class cast exception。因为 ``HandleResolver`` 中持有这些 connector 实例内部的类对象的引用， ``HandleJsonModule`` 就可以通过以上的手段将类反序列化为每个 connector 实例内部的类对象，这样就不会出现 class cast exception。**

---



### 原因

在对问题进行了分析之后在结合问题的发生情况就可以得出响应的结论：

首先我们在创建了个新的 catalog name 为 hive_1 的 catalog（一个新的 connector 实例），假设这个 catalog 的类加载器为 classload_a，进行了一次查询。查询时 coordinator 向 worker 发送 task 时涉及到 ``HiveTableHandle`` 的序列化与反序列化，这时会向 ``ObjectMapper`` 中增加一个反序列化器的缓存，该缓存时以 catalog name 的值 hive_1 为 key，反序列化器为 value，而这个反序列化器反序列化出来的类为 ``Classload_a.HiveTableHandle``。

然后我们再对这个 catalog 进行了一次更新。删除掉之前的 catalog，重新添加一个新的 catalog name 为 hive_1 的 catalog，假设这个 catalog 的类加载器为 classload_b。再进行查询，同样的 coordinator 向 worker 发送 task，此时 ``ObjectMapper`` 根据 json 中的 ``TYPE_PROPERTY`` 获取到 catalog name 为 hive_1 的反序列化器，反序列化出来的类对象是 ``Classload_a.HiveTableHandle``，而此时 hive_1 内部的 ``HiveTableHandle`` 的类对象则是 ``Classload_b.HiveTableHandle``，因此在进行类型强转时发生异常。

---



### 问题反思

这个问题的原因从根本上来讲是由于 Trino 在对 connector 实例进行卸载时，未对 connector 实例完全卸载干净的原因导致的。当然目前的 Trino 在卸载  connector 实例时还有其他很多的对象没有释放，这种情况一旦积少成多很容易造成大量的内存泄漏，还是需要进一步去优化。所以在对 Trino 二开的时候还是有很多坑要踩的。

> 对于原生的 Trino 来说，这个问题是不会发生的，因为 Trino 不支持 catalog 的热加载。