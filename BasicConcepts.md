﻿# 1.基本概念
## 1.1.Ignite是什么
Apache Ignite内存数据组织是高性能的、集成化的以及分布式的内存平台，他可以实时地在大数据集中执行事务和计算，和传统的基于磁盘或者闪存的技术相比，性能有数量级的提升。

### 1.1.1.特性一览
可以将Ignite视为一个独立的、易于集成的内存组件的集合，目的是改进应用程序的性能和可扩展性，部分组件包括：

 - 高级的集群化
 - 数据网格（JCache）
 - 流计算和CEP
 - 计算网格
 - 服务网格
 - Ignite文件系统
 - 分布式数据结构
 - 分布式消息
 - 分布式事件模型
 - Hadoop加速
 - Spark共享RDD

## 1.2.入门
这部分帮助你开始一个新的Ignite程序，你会发现他瞬间就可以跑起来。

### 1.2.1.准备
Apache Ignite官方在如下环境中进行的测试：

 - JDK：Oracle JDK7及以上
 - OS：Linux（任何版本），Mac OS X（10.6及以上），Windows(XP及以上)，Windows Server（2008及以上）
 - 网络：没有限制（建议10G）

### 1.2.2.安装
下面是安装Apache Ignite的快速摘要：

 - 从[https://ignite.apache.org/](https://ignite.apache.org/)下载Apache Ignite的zip压缩包
 - 将zip压缩包解压到系统安装文件夹
 - 设置IGNITE_HOME环境变量指向安装文件夹，确保没有/结尾（这一步可选）

### 1.2.3.从源代码构建
如果你下载的是源代码包，可以用如下命令构建：
```bash
# Unpack the source package
$ unzip -q apache-ignite-1.4.0-src.zip
$ cd apache-ignite-1.4.0-src
 
# Build In-Memory Data Fabric release (without LGPL dependencies)
$ mvn clean package -DskipTests
 
# Build In-Memory Data Fabric release (with LGPL dependencies)
$ mvn clean package -DskipTests -Prelease,lgpl
 
# Build In-Memory Hadoop Accelerator release
# (optionally specify version of hadoop to use)
$ mvn clean package -DskipTests -Dignite.edition=hadoop [-Dhadoop.version=X.X.X]
```
### 1.2.4.从命令行启动
一个Ignite节点可以从命令行启动，可以用默认的配置也可以传递一个配置文件。可以启动很多很多的节点然后他们会自动地发现对方。
`默认配置`
要启动一个基于默认配置的网格节点，打开命令行然后切换到IGNITE_HOME（安装文件夹），然后输入如下命令：
```bash
$ bin/ignite.sh
```
然后会看到大体如下的输出：
```
[02:49:12] Ignite node started OK (id=ab5d18a6)
[02:49:12] Topology snapshot [ver=1, nodes=1, CPUs=8, heap=1.0GB]
```
ignite.sh启动ignite节点默认情况下会使用`config/default-config.xml`配置文件。
`传递配置文件`
要从命令行显示地传递配置文件，可以在安装文件夹路径下输入`ignite.sh <路径>`，比如：
```bash
$ bin/ignite.sh examples/config/example-cache.xml
```
配置文件的路径既可以是绝对路径，也可以是相对于IGNITE_HOME的相对路径，也可以是相对于类路径的META-INF文件夹。
要在一个交互模式传递配置文件，可以加上-i参数，就想这样：`gnite.sh -i`。

### 1.2.5.从Maven获得
在项目里使用Apache Ignite的另一个方式是使用Maven2依赖管理。
Ignite只需要一个`ignite-core`强依赖，通常你还需要添加`ignite-spring`，来做基于spring的XML配置，还有`ignite-indexing`,来做SQL查询。
将`${ignite-version}`替换为实际的版本。
```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-core</artifactId>
    <version>${ignite.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-spring</artifactId>
    <version>${ignite.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-indexing</artifactId>
    <version>${ignite.version}</version>
</dependency>
```
### 1.2.6.第一个计算网格应用
我们写第一个计算应用，他会计算一句话中非空白字符的字符数量。作为一个例子，我们首先将一句话分割为多个单词，然后通过一个个计算作业来计算每一个独立单词中的字符数量。最后，我们通过结果的简单相加来获得整个的数量。
```java
try (Ignite ignite = Ignition.start("examples/config/example-ignite.xml")) {
  Collection<IgniteCallable<Integer>> calls = new ArrayList<>();

  // Iterate through all the words in the sentence and create Callable jobs.
  for (final String word : "Count characters using callable".split(" "))
    calls.add(word::length);

  // Execute collection of Callables on the grid.
  Collection<Integer> res = ignite.compute().call(calls);

  // Add up all the results.
  int sum = res.stream().mapToInt(Integer::intValue).sum();
 
  System.out.println("Total number of characters is '" + sum + "'.");
}
```
我们注意到，由于Ignite的`零部署`特性，当从IDE运行上面的程序时，远程节点没有经过显示的部署，就获得了计算作业。

### 1.2.7.第一个数据网格应用
我们再来一个小例子，它从分布式缓存中获取和添加数据，并且执行基本的事务。
在应用中使用缓存之后，要确保他是经过配置的，我们将示例中的配置文件传递给Ignite，他已经做了一些缓存的配置。
```bash
$ bin/ignite.sh examples/config/example-cache.xml
```
```java
try (Ignite ignite = Ignition.start("examples/config/example-ignite.xml")) {
    IgniteCache<Integer, String> cache = ignite.getOrCreateCache("myCacheName");
 
    // Store keys in cache (values will end up on different cache nodes).
    for (int i = 0; i < 10; i++)
        cache.put(i, Integer.toString(i));
 
    for (int i = 0; i < 10; i++)
        System.out.println("Got [key=" + i + ", val=" + cache.get(i) + ']');
}
```
### 1.2.8.Ignite Visor管理控制台
最简单的检查Ignite数据网格中的内容，以及执行其他的一系列的管理和监控操作的方式就是使用Ignite Visor命令行工具。
要启动Visor，简单地执行如下命令即可：
```bash
$ bin/ignitevisorcmd.sh
```

## 1.3.Maven配置
Apache Ignite是由很多Maven模块组成的，如果项目里用Maven管理依赖，可以单独地导入各个Ignite模块，
注意，在下面的例子中，要将`ignite.version`替换为实际的版本。

### 1.3.1.常规依赖
Ignite强依赖于`ignite-core.jar`。
```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-core</artifactId>
    <version>${ignite.version}</version>
</dependency>
```
然而，很多时候需要其他的依赖，比如，要使用Spring配置或者SQL查询等。
下面就是最常用的可选模块：

 - ignite-indexing（可选，如果需要SQL查询）
 - ignite-spring（可选，如果需要spring配置）

```xml
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-core</artifactId>
    <version>${ignite.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-spring</artifactId>
    <version>${ignite.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.ignite</groupId>
    <artifactId>ignite-indexing</artifactId>
    <version>${ignite.version}</version>
</dependency>
```

### 1.3.2.导入独立模块
可以一个个地导入Ignite模块，唯一必须的就是`ignite-core`，其他的都是可选的，所有可选模块都可以像核心模块一样导入，只是artifactId不同。
现在提供如下模块：

 - `ignite-spring`：基于Sring的配置支持
 - `ignite-indexing`：SQL查询和索引
 - `ignite-geospatial`：地理位置索引
 - `ignite-hibernate`：Hibernate集成
 - `ignite-web`：Web Session集群化
 - `ignite-schedule`：基于Cron的计划任务
 - `ignite-log4j`：Log4j日志
 - `ignite-jcl`：Apache Commons logging日志
 - `ignite-jta`：XA集成
 - `ignite-hadoop2-integration`：HDFS2.0集成
 - `ignite-rest-http`：HTTP REST请求
 - `ignite-scalar`：Ignite Scalar API
 - `ignite-slf4j`：SLF4J日志
 - `ignite-ssh`；SSH支持，远程机器上启动网格节点
 - `ignite-urideploy`：基于URI的部署
 - `ignite-aws`：AWS S3上的无缝集群发现
 - `ignite-aop`：网格支持AOP
 - `ignite-visor-console`：开源的命令行管理和监控工具

## 1.4.Ignite生命周期
Ignite是基于JVM的，一个JVM可以运行一个或者多个逻辑Ignite节点（大多数情况下，一个JVM运行一个Ignite节点）。在整个Ignite文档中，会交替地使用术语Ignite运行时以及Ignite节点，比如说可以该主机运行5个节点，技术上通常意味着主机上启动5个JVM，每个JVM运行一个节点，Ignite也支持一个JVM运行多个节点，事实上，通常作为Ignite内部测试用。
 - Ignite运行时 == JVM进程 == Ignite节点（多数情况下）

### 1.4.1.Ignition类
`Ignition`类在网络中启动每个Ignite节点，注意一个物理机可以运行多个Ignite节点。
下面的代码是在全默认配置下启动网格节点；
```java
Ignite ignite = Ignition.start();
```
或者传入一个配置文件：
```java
Ignite ignite = Ignition.start("examples/config/example-cache.xml");
```
配置文件的路径既可以是绝对路径，也可以是相对于IGNITE_HOME的相对路径，也可以是相对于类路径的META-INF文件夹。

### 1.4.2.LifecycleBean
有时可能希望在Ignite节点启动和停止的之前和之后执行特定的操作，这个可以通过实现`LifecycleBean`接口实现，然后在spring的配置文件中通过指定`IgniteConfiguration`的`lifecycleBeans`属性实现。
```xml
<bean class="org.apache.ignite.IgniteConfiguration">
    ...
    <property name="lifecycleBeans">
        <list>
            <bean class="com.mycompany.MyLifecycleBean"/>
        </list>
    </property>
    ...
</bean>
```
`LifecycleBean`也可以像下面这样通过编程的方式实现：
```java
// Create new configuration.
IgniteConfiguration cfg = new IgniteConfiguration();
 
// Provide lifecycle bean to configuration.
cfg.setLifecycleBeans(new MyLifecycleBean());
 
// Start Ignite node with given configuration.
Ignite ignite = Ignition.start(cfg)
```
一个`LifecycleBean`的实现可能如下所示：
```
public class MyLifecycleBean implements LifecycleBean {
    @Override public void onLifecycleEvent(LifecycleEventType evt) {
        if (evt == LifecycleEventType.BEFORE_NODE_START) {
            // Do something.
            ...
        }
    }
}
```
也可以将Ignite实例以及其他有用的资源注入`LifecycleBean`实现。

### 1.4.3.生命周期事件类型
当前支持如下生命周期事件类型：

 - `BEFORE_NODE_START`：Ignite节点的启动程序初始化之前调用
 - `AFTER_NODE_START`：Ignite节点启动之后调用
 - `BEFORE_NODE_STOP`：Ignite节点的停止程序初始化之前调用
 - `AFTER_NODE_STOP`：Ignite节点停止之后调用

## 1.5.异步支持
Ignite API中的所有分布式方法都既可以同步执行也可以异步执行。然而，并不是为每个同步方法复制一个异步的方法（比如get()和getAsync()或者put()和putAsync()等等），Ignite选择了一个更优雅的方式来让方法不用复制。

### 1.5.1.IgniteAsyncSupport
`IgniteAsyncSupport`接口为很多Ignite API提供了异步模型，比如，`IgniteCompute`,`IgniteServices`,`IgniteCache`以及`IgniteTransactions`,都实现了`IgniteAsyncSupport`接口。
要启用异步模式，需要调用`withAsync()`方法，他会返回同样API的实例，这样的话，异步功能就启用了。

> 如果异步功能启用了，实际的同步方法的返回值就被忽略了，异步操作中获得返回值的唯一方式是通过`future()`方法。

`计算网格示例`
下面的例子说明了同步计算和异步计算的不同：
```java
IgniteCompute compute = ignite.compute();

// Execute a job and wait for the result.
String res = compute.call(() -> {
  // Print hello world on some cluster node.
  System.out.println("Hello World");
  
  return "Hello World";
});
```
下面是如何将上面的调用异步化：
```java
// Enable asynchronous mode.
IgniteCompute asyncCompute = ignite.compute().withAsync();

// Asynchronously execute a job.
asyncCompute.call(() -> {
  // Print hello world on some cluster node and wait for completion.
  System.out.println("Hello World");
  
  return "Hello World";
});

// Get the future for the above invocation.
IgniteFuture<String> fut = asyncCompute.future();

// Asynchronously listen for completion and print out the result.
fut.listen(f -> System.out.println("Job result: " + f.get()));
```
`数据网格示例`
下面是一个数据网格中同步和异步调用的例子：
```java
IgniteCache<String, Integer> cache = ignite.cache("mycache");

// Synchronously store value in cache and get previous value.
Integer val = cache.getAndPut("1", 1);
```
下面是如何将上面的调用异步化：
```java
// Enable asynchronous mode.
IgniteCache<String, Integer> asyncCache = ignite.cache("mycache").withAsync();

// Asynchronously store value in cache.
asyncCache.getAndPut("1", 1);

// Get future for the above invocation.
IgniteFuture<Integer> fut = asyncCache.future();

// Asynchronously listen for the operation to complete.
fut.listen(f -> System.out.println("Previous cache value: " + f.get()));
```

### 1.5.2.@IgniteAsyncSupported
Ignite API中并不是每个方法都是分布式的，因此也就没必要异步化，为了避免关于哪个方法是分布式的混乱，就是哪个能异步那个不能，Ignite API中的每个分布式方法都被加上了`@IgniteAsyncSupported`注解。

> 尽管事实上并不需要，对于非分布式操作通过异步模式仍然可以获得`future`，`future`也可以正常执行。

## 1.6.客户端和服务器端
Ignite有一个可选的概念，就是客户端节点和服务器端节点，服务器节点参与缓存，计算，流式处理等等，而原生的客户端节点提供了远程连接服务器的能力。Ignite原生客户端可以使用完整的Ignite API，包括近缓存，事务，计算，流，服务等等。
所有的Ignite节点默认都是以`服务器`模式启动的，`客户端`模式需要显示地启用。

### 1.6.1.配置客户端和服务端
可以通过`IgniteConfiguration.setClientMode(...)`配置一个节点，或者为客户端，或者为服务端。
```xml
<bean class="org.apache.ignite.configuration.IgniteConfiguration">
    ...   
    <!-- Enable client mdoe. -->
    <property name="clientMode" value="true"/>
    ...
</bean>
```
为了方便，也可以通过` Ignition`类来打开或者关闭客户端模式，这样可以使客户端和服务端共用一套配置。
```java
Ignition.setClientMode(true);

// Start Ignite in client mode.
Ignite ignite = Ignition.start();
```
### 1.6.2.创建分布式缓存
当在Ignite中创建缓存时，不管是通过XML方式，还是通过` Ignite.createCache(...)`或者`Ignite.getOrCreateCache(...)`方法，Ignite会自动地在所有的服务端节点中部署分布式缓存。

> 当分布式缓存创建之后，他会自动地部署在所有的已有或者未来的服务端节点上。

```java
// Enable client mode locally.
Ignition.setClientMode(true);

// Start Ignite in client mode.
Ignite ignite = Ignition.start();

CacheConfiguration cfg = new CacheConfiguration("myCache");

// Set required cache configuration properties.
...

// Create cache on all the existing and future server nodes.
// Note that since the local node is a client, it will not 
// be caching any data.
IgniteCache<?, ?> cache = ignite.getOrCreateCache(cfg);
```

### 1.6.3.客户端或者服务端计算
`IgniteCompute`默认会在所有的服务端节点上执行作业，然而，也可以通过创建相应的集群组来选择是只在服务端节点还是只在客户端节点上执行作业。
```java
IgniteCompute compute = ignite.compute();

// Execute computation on the server nodes (default behavior).
compute.broadcast(() -> System.out.println("Hello Server"));
```
### 1.6.4.管理慢客户端
很多部署环境中，客户端节点是在主集群外启动的，机器和网络都比较差，在这些场景中服务端可能产生负载（比如持续查询通知）而客户端没有能力处理，导致服务端的输出消息队列不断增长，这可能最终导致服务端出现内存溢出的情况，或者如果打开背压控制时导致整个集群阻塞。
要管理这些状况，可以配置允许向客户端节点输出消息的最大值，如果输出队列的大小超过此值，该客户端节点会从集群断开以防止拖慢整个集群。
下面的例子显示了如何通过XML或者编程的方式配置慢客户端队列限值：
```java
IgniteConfiguration cfg = new IgniteConfiguration();

// Configure Ignite here.

TcpCommunicationSpi commSpi = new TcpCommunicationSpi();
commSpi.setSlowClientQueueLimit(1000);

cfg.setCommunicationSpi(commSpi);
```
```xml
<bean id="grid.cfg" class="org.apache.ignite.configuration.IgniteConfiguration">
  <!-- Configure Ignite here. -->
  
  <property name="communicationSpi">
    <bean class="org.apache.ignite.spi.communication.tcp.TcpCommunicationSpi">
      <property name="slowClientQueueLimit" value="1000"/>
    </bean>
  </property>
</bean>
```
### 1.6.5.客户端重连
有几种情况客户端会从集群中断开：

 - 由于网络故障，客户端无法和服务端重建连接；
 - 与服务端的连接有时被断开，客户端也可以重建与服务端的连接，但是服务端仍然断开客户端节点由于服务端无法获得客户端心跳；
 - 慢客户端会被服务端断开；

当客户端发现与集群断开时他会被赋予一个新的本地节点ID然后试图与服务端重新连接。`注意`：这会产生一个副作用，就是当客户端重建连接时本地`ClusterNode`的`id`属性会发生变化。
 当客户端处于断开状态然后试图重建与集群的连接过程中时，所有的Ignite API都会抛出特定的异常：`IgniteClientDisconnectedException`，这个异常提供了future，当客户端重连结束后他会完成（`IgniteCache`API会抛出`CacheException`，他有一个`IgniteClientDisconnectedException`作为他的cause），这个future也可以通过`IgniteCluster.clientReconnectFuture()`方法获得。
客户端重连也有一些特定的事件（这些事件是本地化的，也就是说他们只会在客户端触发）：
 - EventType.EVT_CLIENT_NODE_DISCONNECTED
 - EventType.EVT_CLIENT_NODE_RECONNECTED

下面的例子显示`IgniteClientDisconnectedException`如何工作：
```java
IgniteCompute compute = ignite.compute();

while (true) {
    try {
        compute.run(job);
    }
    catch (IgniteClientDisconnectedException e) {
        e.reconnectFuture().get(); // Wait for reconnect.

        // Can proceed and use the same IgniteCompute instance.
    }
}
```
客户端自动重连可以通过`TcpDiscoverySpi`的`clientReconnectDisabled`属性禁用，如果重连被禁用了那么当与服务端断开时客户端就停止了。
下面的例子显示了如何禁用客户端重连：
```java
IgniteConfiguration cfg = new IgniteConfiguration();

// Configure Ignite here.

TcpDiscoverySpi discoverySpi = new TcpDiscoverySpi();

discoverySpi.setClientReconnectDisabled(true);

cfg.setDiscoverySpi(discoverySpi);
```
### 1.6.6.客户端节点强制服务端模式
客户端可以请求网络中的服务端节点来启动。
如果这是一个请求，那么可以让已经存在的服务端节点直接启动客户端节点，可以通过如下方式强制客户端节点的服务端模式发现：
```java
IgniteConfiguration cfg = new IgniteConfiguration();

cfg.setClientMode(true);

// Configure Ignite here.

TcpDiscoverySpi discoverySpi = new TcpDiscoverySpi();

discoverySpi.setForceServerMode(true);

cfg.setDiscoverySpi(discoverySpi);
```
这种情况下，如果网络中的所有节点都是服务端节点时发现就会发生。

> 这种情况下使用发现SPI的所有节点的地址应该是可以相互访问的，以使发现可以正常工作。