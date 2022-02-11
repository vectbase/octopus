# RSS社区文档

## 1.ESS缺点

**缺乏隔离性：**

（1）shuffle服务与Yarn NodeManager进程运行在同一台主机上。在相同的Yarn集群中，其中一个节点出现了问题可能会对其他节点造成负面影响；

（2）shuffle服务运行在NodeManager进程中，如果shuffle占用大量内存使得nodemanger变得不可用，集群的效率就会变低。

**可扩展性差：**

集群的所有executor数据集中在单一磁盘中会对磁盘造成很大的压力，并且在没有备份的情况下，如果executor共存的shuffle服务崩溃，则所有数据都将丢失，每个节点不得不重新进行计算。

**在container环境下，存储不一定总是共享的：**

executor写入shuffle数据时，executor总默认写入的文件是共享的。可是k8s和docker中运行在yarn和Mesos并不总是这样，管理员为了安全强制执行隔离策略，最大限度的隔离组件致使这一假设并不总是成立。

**资源浪费：**

每一个executor都需要一个时刻运行服务来执行ESS，空跑的ESS会浪费资源。



## 2.设计目标

（1）可以隔离系统中的每个组件（某个组件出了故障不会连累其他组件）。

（2）K8S和Mesos/Yarn的容器 。

（3）shuffle受application和executor数量的影响可伸缩。  

​	 · Shuffle已经可以很好的扩展单个application，这就是为什么Spark是一个高性能、行业级的分布式计算框架的原因之一。但是在多个application和executor共享同一个shuffle服务时，会遇到伸缩性的问题。

（4）发生故障或关闭系统中的任何一台host，不应该出现spark application重新计算RDD的情况。

（5）即使在所有进程彼此完全隔离的容器化环境中，除了特定的API端点（例如RPC）之外，也可以进行弹性shuffle。

（6）负载不应超过单个组件。



## 3.解决方案

### 3.1 通过shuffle服务上传shuffle数据

#### 3.1.1 步骤

将shuffle看作一个服务器，它可以接收shuffle数据以字节流（streams）的形式进行写入，并提供这些shuffle文件作为字节流读取的端点（Endpoint）。大致结构如下：

（1）在某些节点上启动shuffle服务；

（2）shuffle服务配置远程文件系统将shuffle文件放入其中；

（3）所有shuffle共享元数据/数据库，这些数据库/元数据看作是shuffle文件相关的RDD区块的偏移量。shuffle文件路径可被详细划分。例如：executor ID，shuffle ID，map ID的组合；

（4）Spark Application被给予一个URL用于访问外部ESS：  a. 这个URL可以作为稳定负载平衡器/代理连到（1）中的ESS实例；  b. 所有shuffle服务节点都是等价的，代理可以随机选择任何一个；

（5）当shuffle写入时，可从（4）中给出的URI打开一个可写的字节流；

（6）当shuffle读取时，可从（4）中给出的URI打开一个可读的字节流；



#### 3.1.2 是否达成目标

（1） 每个ESS服务都可以在自己主机或者容器中运行，如果某个ESS服务实例失败，只有运行在该服务容器中的shuffle才会受到影响。实际上，spark executor没有必要与shuffle 服务进行合并。因此也就解决了隔离性的问题。

（2） 随着系统中executor数量的增加，可以独立的扩展元存储、shuffle文件存储系统以及shuffle服务以应对集群过载。shuffle服务在过载均衡器执行之后执行可以确保没有任何shuffle服务过载。因此解决了可扩展性差的问题。

（3） executor可以随时关闭，他们计算的shuffle block被永久的保存在一个单独的存储系统。然而，在这种情况下，shuffle服务实例也可以关闭，并且底层的数据可以由任何剩余的ESS实例提供服务。这就解决了有关可靠性、停机时间和浪费资源的问题。

（4） ESS不需要与executors共享本地存储。实际上，shuffle本身是无状态的。因此，只要metastore和shuffle文件存储系统可以在容器中运行（例如，使用云作为存储备份），此解决方案就兼容容器化和隔离机制。

（5） 所有的executor都可以调用不同的shuffle服务实例，负载均衡器确保将大致的工作分配给不同的shuffle实例。



#### 3.1.3 实施细节

安全性：spark应用不能点对点的去读取彼此的shuffle数据；spark应用不应该去污染其他应用的shuffle数据；

传输数据应该被加密：TLS？SASL？

点对点协议：HTTP？Netty？

如何实现shuffle文件的备份？

存储shuffle文件的位置：hdfs？S3？

shuffle服务节点彼此之间复制数据？

如何实现shuffle文件定位/索引存储？

高可用存储系统中，Metadata/索引文件存储位置，hdfs？S3？

PostGres/MySQL/Cassandra？

文件清理；shuffle服务何时清理数据？

spark driver 可以关闭端点的调用，但是如果集群管理员杀死了driver，端点怎么关闭？通过心跳/投票机制？



#### 3.1.4 优势 && 劣势

**优势：**

shuffle文件索引/缓存/后端存储对spark应用程序完全透明。

**劣势：**

每个shuffle文件必须分三次写入：shuffle服务代理/shuffle服务后端/备份文件存储。



### 3.2 shuffle服务提供远程文件URI

#### 3.2.1 步骤

shuffle继续保留文件路径，但是executor将这些路径视为位于某个远程文件系统上，而不是executor本地的路径。shuffle服务使用数据库为每个应用存储这些路径。工作原理如下：

（1）在某些节点上启动shuffle服务；

（2）所有shuffle服务实例共享一些数据库/元存储，这些数据库/元存储跟踪shuffle文件的路径以及RDD区块对应的shuffle文件的偏移量；

（3）Spark应用被赋予一个URI用于访问ESS：

  	a. 这可以是到负载均衡器/代理的稳定URI，该代理路由到（1）中启动某个ESS实例； 

​	  b. 所有shuffle服务节点都是等价的，代理可以随机选择任何一个；

（4）Spark应用配置一个URI，用于为该应用写入所有shuffle数据，例如：

[hdfs://my-cluster.spark.test:9000/application-1249012851/tmp/](hdfs://my-cluster.spark.test:9000/application-1249012851/tmp/)

（5）每当Spark executor写入shuffle数据时，executor都会打开带有（4）中配置文件的URI根文件：  

​	 a. executor将数据写入远程存储层；  

​	 b. executor根据当前逻辑得到要写入数据的路径。然后使用ESS解析URI路径。

（6）每当executor读取一个shuffle区块时，executor就会联系ESS来查找该区块的分区字节偏移量。与原本一样，shuffle文件的路径是由executor、shuffle ID和map ID组成的。最后executor根据根URI单独解析路径，并直接从远程文件系统读取数据。



#### 3.2.2 优势 && 劣势

**优势：**

只有元数据必须经过多个跃点；

与（1）相比改动小：  

​	· 将对文件的所有引用替换为实现引用某个远程文件。shuffle服务只是将索引文件存储在一些分布式文件系统中，而不是存储在本地磁盘上，shuffle writer和reader使用使用远程文件替代本地文件。 

​	 · 为了更快访问分区元数据而进行的增量更改可以在shuffle服务后端代码中独立完成，无序更改shuffle服务的元数据位置API。

**劣势：**

增加spark应用程序配置的复杂性-需要为无序文件选择备份存储；

要求所有application都可以直接访问备份存储。在方案3.1中，可以通过防火墙等对shuffle数据的访问进行限制，shuffle服务是访问洗牌数据的单一入口点。但是在本方案下，任何能够访问备份集群的应用都可以访问shuffle数据，缺乏安全性。



### 3.3 Driver维护所有元数据，不使用shuffle服务

#### 3.3.1 步骤

application本身可以维护他自己关于application shuffle文件的所有信息，从而完全不需要ESS。实施工作如下：

（1）application跟踪partition ID到文件位置+文件内偏移量之间的映射。

（2）所有文件位置都在某个远程文件存储层中。

（3）executor为shuffle文件打开或关闭流，以便直接备份文件


#### 3.3.2 是否达成目标

（1） 这个解决方案中，删除了ESS服务，剩下的组件只有shuffle数据存储系统和Spark Application本身。Spark应用程序可以被隔离，而存储系统可以与访问其数据的计算作业分离，因此此方案解决了隔离性的问题。

（2） 由于没有ESS，因此该方案的可扩展性取决于Spark shuffle操作的可扩展性以及shuffle数据存储系统的可扩展性。可以主要通过改进Spark Shuffle的扩展性来解决扩展性差的问题。

（3） executor可以随时关闭，写入的shuffle数据仍然可以通过持久备份存储区使用。因此不用担心独立系统的正常运行时间，也不用担心停机时间和资源浪费的情况，有关可靠性的问题得到了解决。

（4） 在容器内运行的Spark Application可以访问远程存储系统进行Shuffle write和Shuffle read。此方案兼容容器化。

（5） 在分布式文件系统中的节点间复制shuffle block，该系统中就不会出现瓶颈。我们不需要一个单独的组件一次加载所有文件——所有的executor都可以获得以分布式方式获取的元数据和数据



#### 3.3.3 实施细节

（1）driver如何以可扩展的方式维护相关数据？ 

​	  a. 不要只存储在driver中；

 	 b. 使用executor可以访问的外部元存储。但是选择什么种类的元存储？文件存储在分布式文件中还是数据库中？

 	 c. 规定map定位文件的位置；索引文件存储的偏移量；索引文件也使用约定的map定位。

（2）文件清理： 

​	   a. 如果driver意外退出，如何清除备份存储中写入的shuffle数据？通过关闭挂钩（shutdown hook）。如果没有挂钩很难实现，因为没有第三方知道写入的洗牌文件。

（3）安全：加密shuffle数据，设置远程文件系统权限。



#### 3.3.4 优势 && 劣势

**优势：**

简化了基础架构，不用担心单独的服务。

**劣势：**

​	· 清理数据不可靠。当spark接受到kill信号时，spark application无法清理自己的shuffle文件。没有其他组件知道此类文件的存在和生命周期。因此这些文件可能无限期的被保留。

​	· 高效读取索引文件比较困难。





### 3.4 备份shuffle文件到分布式文件系统

#### 3.4.1 步骤

先将shuffle文件写入executor的本地磁盘，然后异步上传到分布式文件系统（代替application直接写入分布式文件系统）。即使executor结束，其shuffle文件已经作为备份写入分布式文件系统，仍可以访问备份文件。

（1）executors将shuffle文件写入本地磁盘。他们还不断的将shuffle文件和索引文件异步上传到分布式文件系统。

（2）当executor需要交换shuffle数据时，他们直接互相联系来读取数据（就像当前模型中没有shuffle服务一样）

（3）如果无法访问某个executor，则读取shuffle数据的executor将尝试从HDFS的规范位置读取shuffle文件和shuffle索引文件。（如果可行的话）



#### 3.4.2 是否达成目标

（1）这里也去除了ESS，因此如果一个executor失败，它只会影响executor所属的application。备份存储层具有弹性和高可用性。

（2）一个潜在的问题是，如果executor接受到大量的shuffle数据，则必须花费大量I/O上传备份。因此，如果这个执行器在没有备份完所有shuffle文件时就崩溃了，则必须重新计算所有shuffle数据。备份大量数据的同事可能会永久丢失一些shuffle区块。由此可见，可伸缩性还有待改进。

（3）如果executor在备份完其shuffle文件之前失败，则需要重新计算尚未备份的shuffle区块。

（4）此方案兼容容器化。本地磁盘不需要在系统中的任何两个进程之间共享。在executor之间和executor与远程备份存储之间通过RPC交换shuffle文件。



#### 3.4.3 实施细节

有没有版本可以使用ESS？为什么这么做？

· 假设我们像option1一样通过ESS上传数据，但是shuffle没有同步写入远程存储，而实缓存文件的本地副本，然后异步写入本地存储。

· 主要优点：shuffle服务应该比Spark executor更具有弹性，因为executor所承受的工作负载更重。

· 需要executor找到要读取的特定shuffle服务实例，或者使用shuffle服务代理跟踪位置并发送重定向。

· 需要通过网络写入持久化的shuffle数据，而不是通过本地存储写入。

· 但是，如果一个shuffle服务节点在备份其所有数据之前出现故障，那么所有将shuffle数据写入的application都将受到影响。



#### 3.4.4 优势 && 劣势

**优势：**

最佳情况下，当executor没有崩溃时，网络跳数更少——可以将shuffle写入本地磁盘，而不必通过网络传输或者分布式存储。



**劣势**：

当executor在完成数据备份之前崩溃时，需要重新计算。



### 3.5 上传数据到ESS，Driver跟踪文件位置

#### 3.5.1 步骤

shuffle服务可以负责存储shuffle数据，但是driver可以跟踪文件并复制到shuffle服务的主机上。以上描述的option1-4需要使用分布式文件系统，但是大多数分布式文件系统都需要一个主节点（master）来跟踪有关文件如何在集群中分布元数据。这个master必须随着分布式文件系统中文件的数量而扩展，如果集群需要存储来自系统中运行的所有Spark Application的shuffle文件，那么分布式文件系统中的文件数量可能会更大。下面的设计建议将shuffle文件跟踪分离到Spark Application。Shuffle服务的行为就像一个简单的文件管理器，application负责使用这些文件服务器为shuffle数据提供冗余。设计如下：

（1）executor将shuffle map输入和索引文件写入本地磁盘，将map输出报告给driver；

（2）executor还将shuffle map输出和索引文件作为备份上传到ESS：

​	 a. driver必须在集群中定位ESS实例。这是由特定的集群管理的。例如，对于YARN，每个node manager上都应该运行一个shuffle服务，但是对于K8S，可以指定一组与命名空间中的标签匹配的区域，通过区域IP连接到shuffle服务。

​	 b. executor和driver可以选择用于备份的shuffle服务的主机。 

​	 c. 备份可以是异步的，对executor故障的回复能力较差（例如，如果在executor崩溃之前没有完成备份，则无序数据将永远消失）。

​	 d.  每个shuffle文件都可以复制到任意数量的shuffle服务实例。

（3）executors告知driver备份文件的位置。map输出跟踪器也保存有关这些备份的元数据。

（4）reducer尝试从executor直接读取shuffle数据。如果失败，则从备份中读取shuffle文件。



#### 3.5.2 是否达成目标

（1）此方案比之前的方案更可靠。之前，当ESS崩溃时，与该shuffle服务共享的数据的所有executor的map输出都将丢失。在这种情况下，对备份map输出的shuffle服务与executor位于同一主机上没有严格的要求（executor和shuffle服务可以不在同一个主机上）。此外我们更喜欢executor用于服务他自己的shuffle数据。因此，当任何给定的ESS崩溃时，将数据备份到该shuffle服务的executor可以继续提供自己的shuffle数据（ESS崩溃，也会有executor提供shuffle数据）。通过让executor将shuffle文件备份到多个shuffle服务实例，可以进一步提高恢复能力。当系统中除driver外的任何一个组件发生故障时，额外的冗余为我们提供了更大回复能力。因此，此解决方案实现了目标（1）。

（2）通过两种主要方法来评估此解决方案的可伸缩性，即根据集群中的application数量进行扩展： 

​	 a. 备份shuffle数据会使整个磁盘利用率显著提高。通过高度压缩的格式存储备份可以减少磁盘使用量。备份应该只在executor不能再提供shuffle数据时才需要被引用，因此并不总是用支付解压的费用。

​	  b. 其他需要具有单个主节点的分布式文件系统的解决方案要求该主节点保存有关文件系统中所有文件的元数据。例如，HDFS namenode不能很好的扩展集群中的大量小文件。多个spark application可以编写大量的小文件，给hadoop集群带来的问题。这个方案的优点是：没有单个组件必须跟踪所有Spark Application中的所有shuffle文件。Spark driver只负责跟踪自己的备份shuffle文件。但是，这给每个Spark application带来了更大的压力，因为application需要保存更多关于主map和备份map输出位置的内存状态。

（3）shuffle文件可以被复制，当一个shuffle服务丢失时，其他shuffle服务实例可以提供丢失的块。此解决方案实现了目标（3）。

（4）备份是通过网络写入的，没有两个组件需要共享相同的磁盘空间。这个解决方案可以再容器环境中运行，从而实现目标（4）。

（5）如前所述，不清楚driver是否能够跟踪主map和备份map输出元数据。我们已经观察到大量的分区给Spark驱动带来了麻烦。driver的内存占用可能会急剧增加；我们可以探索map输出元数据的磁盘存储选项。



#### 3.5.3 实施细节

· 同步备份还是异步备份

如果是同步，对job运行时有什么影响？

如果是异步，如何处理再executor完成所有备份之前退出的executor？

· 存储/文件格式？可以通过压缩减少磁盘使用量吗？

·加密/授权。

·文件清理：必须在application完成/崩溃时清理shuffle文件。

·executor本身是否需要首先提供shuffle数据？如果executor只是直接将shuffle文件写入shuffle服务呢？



#### 3.5.4 优势 && 劣势

**优势：**

​	·不依赖第三方分布式存储解决方案（如HDFS），这意味着我们可以完全控制读/写性能。

​	·没有一个节点负责存储所有无序文件元数据。

**劣势：**

​	· 文本被复制时，特别是当executor和shuffle服务都在存储数据的副本时，磁盘使用率急剧上升。

​	· driver必须同事存储主map和备份map输出信息，这给driver带来更大的压力。





