


## 摘要

Kafka 是一款非常优秀的开源消息引擎，以消息吞吐量高、可动态扩容、可持久化存储、高可用的特性，以及完善的文档和社区支持成为目前最流行的消息队列中间件。

Kafka 的开发社区一直非常活跃，在消息引擎的领域取的不俗成绩之后，不断拓展自己的领域，在基于事件的流处理平台方向一直发力，不断自我更新迭代力图成为这个领域内的事实标准。

Kafka 的消息引擎功能十分强大，但是一直没有停下自我突破的脚步，随着 3.0 版本的中 KRaft 协议的推出，Zookeeper 的退出进程正式启动，Kafka 开始了又一次的自我蜕变。

ZK 的移除是一个非常大胆的举动，因为 ZK 在 Kafka 的集群管理中居于核心的地位，不会轻易取代，那为什么 Kafka 选择了自行实现选举机制的路线？

此外，虽然 Kafka 具备诸多优秀的特性，这些如今被视为最佳实践的特性也是不断演化而来的，从其不断升级改进的过程中也能间接反映出生产环境所面临的现实问题，那么 Kafka 在实际的生产环境中的表现究竟如何？

作为业务方，使用 Kafka 作为消息中间件进行业务开发，保证服务平稳运行需要避开哪些雷区？

这篇文档将从一个比较高的视角，从 Kafka 的设计理念、架构到实现层面进行深入解读，随着对 Kafka 相关机制的深入了解，这些问题的答案将浮出水面。

## 须知事项

* 这篇文档基于 Kafka 最近刚刚发布的 3.2 版本源码为基础进行介绍，主要讨论 Java 和 Scala 语言实现的原版客户端和服务端，其他语言版本的客户端与这篇文档介绍的机制在实现上会有较大出入，需要留意
* 此外，字节的业务很多使用的都是自研的 BMQ [3]，在客户端协议上是完全兼容的，但是服务端进行了完全的重构，本文介绍的相关服务端机制并不适用
* Kafka 整个项目包括 Core、Connect、Streams，只有 Core 这一部分是我们通常说的核心消息引擎组件，另外两个都是基于这个核心实现的上层应用，这篇文章主要介绍的就是 Kafka Core 相关的内容，下面的 「Kafka 的应用架构部分」会对这一点做简要介绍

## 名词对照

下面的表格给出了 Kafka 中出现的一些高频和重要概念的对照解释

|**英文名**|**中文名**|**解释**|**备注**|
| -----------------------| --------------------| -----------------------------------------------------------------------------------------------------------------------------------------| ------------------------------------------------------------------------------------------------------------------------------------------------|
|KIP|Kafka 改进提案|KIP（Kafka Improvement Proposal）是针对 Kafka 的一些重大功能变更的提案，通常包括改进动机、提议的改进内容、接口变更等内容||
|Partition|分区|一个独立不可再分割的消息队列，分区中会有多个副本保存消息，他们的状态应该是一致的|Kafka 分区副本的同步机制不是纯异步的，有高水位机制去跟踪从副本的同步进度，并且有对应的领导者副本选举机制保证分区整体对外可见的消息都是已提交的|
|Replica|副本|分区中消息的物理存储单元，通常对应磁盘上的一个日志目录，目录中会对消息文件进一步进行分段保存||
|Leader Replica|主副本、领导者副本|指一个 Partition 的多个副本中，对外提供读写服务的那个副本|Kafka 集群范围有对等地位的组件是 Controller|
|Consumer|消费者|Kafka 客户端消费侧的一个角色，负责将 Broker 中的消息拉取到客户端本地进行处理，还可以使用 Kafka 提供的消费者组管理机制进行消费进度的跟踪||
|Consumer Group Leader|消费者组领导者|通常指 Consumer Group 中负责生成分区分配方案的 Consumer|这个概念非常容易和上面的 Leader Replica 混淆|
|Log start offset|消息起始偏移|Log start offset，Kafka 分区消息可见性的起点|此位置对应一条消息，对外可见可以消费|
|LSO|上次稳定偏移|Last stable offset，和 Kafka 事务相关的一个偏移|当消费端的参数isolation.level 设置为“read_committed"的时候，那么消费者就会忽略事务未提交的消息，既只能消费到LSO(LastStableOffset)的位置|
|LEO|消息终止偏移|Log end offset，Kafka 分区消息的终点|LEO 是下一条消息将要写入的位置，对外不可见不可供消费|
|HW|高水位|High water mark，用于控制消息可见性，HW 以下的消息对外可见|HW 的位置可能对应一条消息，但是对外不可见不可以消费，HW 的最大值是 LEO|
|LW|低水位|Low water mark，用于控制消息可见性，LW 及以上的消息对外可见|一般情况下和 Log start offset 可以等价替换，代码里也是这个逻辑|
|ISR|已同步副本|In sync replica 指满足副本同步要求的副本集合，包括领导者副本|副本同步是按照时间差进行判定的，而非消息偏移的延迟|

## Kafka 的应用生态

下面这张是我根据 [Confluent 博客](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.confluent.io%2Flearn-kafka%2Farchitecture%2Fget-started%2F "https://developer.confluent.io/learn-kafka/architecture/get-started/")的一张资料图重绘的 Kafka 应用生态架构图，在正式开始介绍本文的主题之前，我们先了解一下 Kafka 的整个应用生态

​![](assets/8cf964323e804f97903c3c6f8b9446eftplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-mwmwvpj.jpg)​

这张图中居于核心地位的是 Kafka Core 的集群，也是我们常用的消息引擎的这部分功能，是我们这篇文档重点介绍的对象

在核心的周围，第一层是 Producer 和 Consumer 的基础 API，提供基础事件流消息的推送和消费

而基于这些基础 API Kafka 提供了更加高级的 Connect API，能够实现 Kafka 和其他数据系统的连接，比如消息数据自动推送到 MySQL 数据库或者将 RPC 请求转换为事件流消息进行推送

此外，Kafka 基于自己的消息引擎打造了一个流式计算平台 Streams，提供流式的计算和存储一体化的服务

## Kafka Core 架构

Kafka Core 架构部分的解读从模型、角色和实体、典型架构三个方向层层递进进行介绍

## 消息模型

Kafka 的消息模型主要由生产消费模型、角色和实体，以及实体关系构成，前者表示了消息的生产消费模式，后者描述了为了实现前者，其内部角色和实体存在怎样的逻辑关系

基本消息生产消费模型如下图所示：

​![流程图 (3).jpg](assets/f85fc3b20a244e62bce50a81d4db01cftplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-ymfqj27.jpg)​

> 图中展示了一个非常基本的生产消费场景，生产端向队列尾部发送消息，消费端从队列头部开始消费

从左往右看分别是消费端、消息队列、生产端，这三块我们分开进行详细介绍

### 消费端

在消费端有众多消费者，它们之间用**消费者组**关联起来

注意图中 Consumer 0 是没有分配到分区进行消费的，因为消费者组主要起个负载均衡的作用，一个分区被两个消费者消费从业务视角来看就是在重复消费了

对已经分配到分区的消费者来说，消费从队列的头部开始，在 HW 前结束

### 消息队列

消息队列处于整个消息模型中心的地位，是连接生产端和消费端的枢纽，Kafka 在性能优化上做的工作最多的就是这一个部分

因为 Kafka 的消息存储是队列的数据结构，只允许消息追加写入，这样的设计能最大化利用现有持久化存储介质的写入性能（SSD 和 HDD 都存在顺序写入性能远大于随机写入的特性），实现消息队列的高吞吐量

此外，Kafka 的队列还设计了高水位机制，避免未被从副本完成同步的消息被消费者感知并消费

### 生产端

生产端的 Producer 持续发送消息到队列中，消息追加到队列尾部，通过指定分区算法可以决定消息发往 Topic 下的哪个分区

### 小结

Kafka 的整个消息模型还是基于经典的消息模型去设计和改进的，消息模型的设计还是非常简洁易懂的，它的创新和优势就是在于将这一套模型用分布式的多机模式实现出来，能支撑住大并发、高吞吐量场景下的低时延、高可用的业务需求

当然这套模型之下，还有一些比较小的话题值得去讨论，我这里选了两个话题展开叙述来结束这一节

#### Push vs Pull

在 Kafka 定义的消息模型中，消费端是通过主动拉取消息的方式来消费的，但是与之对应的还有消息推送模型，Broker 对生产者推送过来的消息进行主动分发和推送到消费端

直觉上我们会觉得这种方式很自然，或者认为这是消息引擎的唯一范式，但是实际上关于为什么选择 Pull 的方式来进行消费，Kafka 的官方文档中关于这部分设计有专门列出来，主要讨论的点是消息消费的流控策略应该放在 Broker 端还是 Consumer 端，感兴趣的可以去阅读一下 [Apache Kafka Documentation](https://link.juejin.cn/?target=https%3A%2F%2Fkafka.apache.org%2Fdocumentation%2F%23design_pull "https://kafka.apache.org/documentation/#design_pull")

#### 零拷贝（Zero-Copy）

零拷贝从广义的角度来看不是一种具体的技术实现（仅指操作系统实现的零拷贝机制），而是一种优化思想或者技巧，针对程序运行中不可变的数据或者不可变的部分尽量减少或者取消内存数据的拷贝，用内存地址去引用这些数据

Kafka 的消息队列的核心功能就是进行各种数据的 IO 和转发（IO 密集型应用），零拷贝带来的收益非常明显：

* 减少了 JVM 堆内存占用，降低了 GC 导致的服务暂停和 OOM 风险
* 减少了大批量频繁内存拷贝的时间，能大幅优化数据吞吐性能

所以很有必要进行这样的优化

Kafka 的实例是运行在 JVM 里的，零拷贝的技术落地也离不开 Java 运行时提供的环境，具体到实现上主要依赖 Java 提供的 FileChannel 去映射文件

针对消息拉取消费的场景，直接将日志段 FileChannel 中对应偏移和长度（Kafka 的日志段都有对应的索引文件，所以不需要读取原始消息日志段文件就能拿到这些信息）的数据发送到网络栈，规避应用层的数据拷贝中转

针对消息推送生产的场景，从网络栈读取出来处理好的消息直接从内存 Buffer 中向 FileChannel 写入追加，当然这个场景并没有实现严格意义上的零拷贝（JVM 堆内存存在于用户空间，写入文件中必须要拷贝到内核），只不过 Kafka 用了 MemoryRecords 这个类基于 Buffer 去管理内存中的消息，规避了使用对象结构的方式管理可能存在的内存拷贝和数据序列化行为（这个优化的思路和 String 以及 StringBuilder 一致）

这里只是以场景的例子提供一些分析零拷贝实现机制的视角（系统原生支持 + 处理逻辑层面优化），零拷贝单独展开也是一个很大的话题，总体来讲就是在各个环节尽可能减少内存拷贝的次数，提高数据读写性能

## 角色和实体

在 Kafka 对上述消息模型的实现中，定义了一系列负责执行的角色和表达数据结构的实体，每个角色和实体都有其对应的责任边界，这些角色和实体之间共同配合完成整个消息引擎的运作

Kafka 中有这么一些比较重要的角色和实体：

* Broker 是一个独立的 Kafka 服务端实例，是最大的实体范围，其他角色的实例都通过对象成员的形式引用进来，自身不负责请求的处理逻辑
* Controller 是整个 Kafka 集群的管理者角色，任何集群范围内的状态变更都需要通过 Controller 进行，在整个集群中是个单点的服务，可以通过选举协议进行故障转移
* Replica 是一个独立的消息队列实体，负责消息在物理层面上的存储
* Partition 是逻辑层面的“队列”实体，实际上是一组 Replica 的集合
* Topic 是 Partition 的实体集合
* Producer 是消息生产者角色，会发送消息到对应主题的分区中，写入到 LEO 的位置去
* Consumer 是消息的消费者角色，能消费到 Partition 对外可见的消息
* Consumer Group 是 Consumer 的集合实体，并对应一组管理机制用来协调 Consumer 的消费
* Group Coordinator 是 Broker 中一个负责管理对应消费者组元数据的角色，比较重要且熟知的功能就是负责消费进度的管理

虽然这里已经列举了比较多的角色和实体定义，但是 Kafka 中定义的角色和实体远不止列举的这些，不过大部分都不是本文需要介绍的相关内容，就不在这里一一列举了

上面我们已经了解了 Kafka 消息引擎部分的一些设计抽象层面的知识，下面从 Kafka 的实现角度深入介绍一下上面出现的一些角色和实体

### Broker

这一节开始对 Broker 的简介中的定义是一个 Kafka 服务端实例，如果进一步追问这个实例是什么，在代码中如何体现的话，答案就是 KafkaServer，这是个继承了 KafkaBroker 的实现类

服务端进程启动的入口就在这里，此外一些简单的请求可以直接在 KafkaServer 中处理掉，比如一些读取元数据相关的请求就不需要进入其他角色的逻辑中处理了，直接读取数据组装结构体返回即可

不过我们以架构视角去看 Kafka 的话不需要这么具体，就抽象地把它看做服务端实例即可

### Controller

Controller 是 Broker 中对 Kafka 集群来说非常重要的一个角色，负责集群范围内的一些关键操作：

1. 主题的新建和删除
2. 主题分区的新建、重新分配
3. Broker 的加入、退出
4. 触发分区 Leader 选举

每个 Broker 里都有一个 Controller 实例，多个 Broker 的集群同时最多只有一个 Controller 可以对外提供集群管理服务，Controller 可以在 Broker 之间进行故障转移

Controller 承担的责任在我们眼里更像是集群的 Leader，不过在 Kafka 的其他地方也出现了 Leader 这个角色，避免混淆还是先记住 Controller 也是集群中的重要角色吧

### Partition

​![流程图 (4).jpg](assets/fb729988b8554745aea5cf89bef3b9actplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-fqqjing.jpg)​

Partition 是一个独立的消息队列，从数据结构的角度看可以理解为一个用数组实现的队列，起点是 Log start offset，此偏移会随着消息过期时间等配置的影响，逐渐向右移动

HW 是已提交消息的可见性的边界，仅在此偏移之下的消息对外是可见的（注意，不含 HW 本身），该偏移的移动和 Kafka 的副本同步机制紧密关联，下面会专门介绍此机制

Log start offset 和 HW 共同配合，形成了已提交消息的可见范围，需要注意的是受 Broker 的消息过期清理配置的影响，从副本的 Log start offset 的值通常小于等于领导者副本的 Log start offset，可见范围同样会因此缩减

LEO 是消息队列的终点，下一条消息将在这个地方写入，同时 HW 的最大值就是更新到这里

LW 的作用不是很大，因为分区的 Leader 副本一旦初始化完成，其 Log start offset 的值更新机制就是 LW 的更新机制，两者可以等价替换

上面说的这几个偏移的管理主要和 Kafka 的副本管理机制相关，尤其是 HW 更新机制，因为消息数据需要在多个副本之间同步，所以需要这样的机制来管理数据同步的进度

### Topic

一个 Topic 就是一组 Partition 的集合，效果相当于是给一组 Partition 做了个命名，唯一提供的实际功能应该就是增加集合中的 Partition 数量

值得注意的是，先前版本的 Kafka 中仅使用 Topic 名称作为标识符去区分不同的 Topic，但是新版本中加入了 UUID 去进行判断 [KIP-516](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-516%253A%2BTopic%2BIdentifiers "https://cwiki.apache.org/confluence/display/KAFKA/KIP-516%3A+Topic+Identifiers") ，主要是为了解决删除、新建重名的 Topic 场景下的一些问题

### Producer

Producer 是无状态的（不使用事务机制的情况下），和 Partition 之间是多对多的关系

Producer 可根据分区算法自行决定一条消息应该发往哪个分区，该机制会在下面的文章中进行简要分析

### Consumer

Consumer 是有状态的（不使用消费者组静态成员或者不使用无消费者组机制的情况下），这个状态以 Consumer Group 为单位进行维护

和 Consumer 自身关系比较大的应该就是消息消费偏移提交机制了，这个功能在服务端 0.9.0 版本之前的实现是用 ZK 来保存的，但是后面版本中 Kafka 开始用内部主题来持久化消息偏移了

### Consumer Group

消费者组是 Kafka 中的一个重要实体了，因为消费者组不仅仅是一个消费者的集合，而是以 Group 为中心辐射出一组消费的的管理机制：

* 分区分配方案，由消费者组选举出的消费者 Leader 执行生成，Coordinator 负责分发
* 消费者加入、退出机制，由 Coordinator 负责协调执行
* 消费者组消费进度管理，由 Coordinator 负责持久化管理

### 小结

这一节只从集群大的视角列举了一些比较重要的角色和实体，在后面的介绍中会有更加细分的角色和实体的深入介绍

通过对各个角色和实体的概念和职责建立起清晰认知，对我们理解 Kafka 的集群架构设计、机制原理、问题定位有很大的帮助

#### 角色和实体关系

Kafka 中的角色和实体概念比较多，我这里梳理了一下比较核心的这些角色和实体之间的对应关系，能更好地帮助理解这些概念

​![流程图 (5).jpg](assets/4e5ddafac0be41c787803cce33171852tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-nsju1kr.jpg)​

注意上面关系图中，Controller 和其他对象之间的关系描述的是管理视角的，而非对象实体的具体包含关系

因为从对象实体的包含关系上说，Controller 和 Broker 之间是一对一的关系，但是这样的关系描述没有实际意义

## 集群架构解析

一个具有代表性的 Kafka 集群通常具备 1 个独立的 ZK 集群、3 个部署在不同节点的 Broker 实例，这里我以一个这样的典型集群的为例来介绍 Kafka 的整体架构，集群情况如下：

* 3 Broker、多个 Consumer（属于某个消费者组）、多个 Producer、1 AdminClient
* 1 Topic、1 Partition（Leader 副本在 Broker 1 上）
* 当前 Controller 位于 Broker 0 上
* Consumer 所属消费者组的 Coordinator 位于 Broker 0 中

架构图如下所示：

​![流程图 (6).jpg](assets/8eb8b0d3d1354fb09b893c1569885c42tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-3u40y8a.jpg)​

下面我将结合上面的架构图，从集群管理、消费、生产这几个大的视角来解读一下

### 集群管理

集群管理是一个重要命题，因为 Kafka 集群需要管理大规模的 Broker 实例、消费者、生产者还有主题分区的消息日志数据

#### ZK 事件监听

集群管理的工作主要是由 Controller 来完成的，而 Controller 又通过监听 Zookeeper 节点的变动来进行监听集群异动事件

Controller 进行集群管理需要保存集群元数据，监听集群状态异动情况并进行处理，以及处理集群中修改集群元数据的请求，这些工作主要都是通过 Zookeeper 来实现的

当前示例集群中是 Broker 0 的 Controller 正在负责管理，监听 ZK 中的相关节点异动情况，而其他 Broker 中的 Controller 处于备用状态，监听 /controller 节点准备下一轮选举

#### ZK 目录结构

我梳理了一下 Kafka 在 Zookeeper 中的目录结构，因为没有实测 Kafka 的所有集群功能，所以末级节点可能不完整有缺失，但是重要比较核心的 ZNode 我都覆盖到位了

梳理出的 ZNode 树状结构图如下：

​![image.png](assets/88eab49a26164cb6abc2339ce88ac606tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-0wsoxwf.jpg)​

除了节点的名称，节点中还有 Kafka 序列化的 JSON 数据，部分节点的数据结构如下：

* Partition 相关节点的值

/brokers/topics/test-topic/partitions/1/state

```JSON
{
   "controller_epoch":1,
   "leader":0,// leader 副本所在的 broker id
   "version":1,// 代码里硬编码的一个值，始终是 1
   "leader_epoch":0,
   "isr":[// 已同步副本集合，Leader 副本包含在内
      0,
      1,
      2
   ]
}
复制代码
```

* Controller 相关节点的值

/controller 是个临时节点，Session 超时过期等原因会导致此节点被删除

```JSON
{
   "version":1,// 代码里硬编码的一个值，始终是 1
   "brokerid":0,// 当前是 Controller 的 Borker ID
   "timestamp":"1649930940915"
}
复制代码
```

/controller_epoch 是个永久节点，数据会持久化存储

```JSON
1  // 值就是个递增的数字，表示选举周期
复制代码
```

* Topic Config 相关的值

此节点用于储存 Topic 级别的动态配置

/config/topics/test-topic

```JSON
{
   "version":1,// 代码里硬编码的一个值，始终是 1
   "config":{
      "min.insync.replicas":"2"    // topic 级别的配置
   }
}
复制代码
```

#### 集群管理请求转发

如上面的架构图所示，Broker 2 收到了 AdminClient 发送过来的 CreateTopicRequest 请求，并没有进行处理，而是转发到了 Controller

碰到这类集群管理的请求，Broker 都先对自身状态进行判定，不是 Controller 的情况下会对满足要求的请求进行转发

通常是对于集群状态有修改的请求会进行转发，对于读取集群状态的请求则通过本地的元数据缓存来处理

### 副本管理

上面介绍的典型集群架构只有一个分区，三个副本，这里拓展成三个分区，九个副本的集群来简要介绍一下 Kafka 副本管理的模式

​![流程图 (7).jpg](assets/187ecd87e34444f2bd4c91d437bb3c65tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-0lhihqw.jpg)​

前面的消息模型和架构图中里我们已经了解了分区、副本的概念，这里通过上面这张图梳理一下分区和副本在集群中的分布关系

首先，同一个分区有多个副本，这里设定的是 3 个，尽可能均匀分布在 Broker 中

其次，分区副本里有一个 Leader Replica，也就是领导者副本，负责分区消息数据的读写，所以领导者副本在 Broker 之间也需要均匀分布，这样才能保证负载均衡

结合上面的图例，有几个点需要注意：

1. 副本自身是没有专门的编号的，副本在哪个 Broker 上，对应的 Broker ID 就是它的编号（这里也间接限制了副本数量的最大值必须小于 Broker 节点数量）
2. 我这里举例用绿色表示了从副本，假定的是已同步的状态，实际场景中会存在从副本未同步完成的情况

#### 读写分离

读写分离是关于 Kafka 副本管理的一个热点话题，Kafka 目前是支持消费从副本消息数据的，[KIP-392](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-392%253A%2BAllow%2Bconsumers%2Bto%2Bfetch%2Bfrom%2Bclosest%2Breplica "https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica") 的提案就是关于这个机制

但是从上面的图中我们也可以看出，你读取的从副本所在的 Broker 也是另一个分区的领导者副本所在的位置，大多数场景下使用这个功能只会导致热点 Broker 的出现，并承担数据同步延迟的代价，并不能达到我们减轻领导者副本负载的目的

这个提案改进这个小功能点主要是为了解决跨数据中心/机架部署的场景下，尽可能从本地数据中心消费数据，降低网络通讯的成本开销，提高数据吞吐量

另外一点需要注意的是，Broker 的分区副本同步只能从领导者副本消费消息进行拉取，无法从其他从副本获取数据，支持读写分离的是客户端消费者

### 消费

消费的流程我们只讨论的是使用了 Kafka 提供的消费者组管理机制的消费者，对于手动管理消费进度的情况这里不予讨论

消息消费的大体流程是：

1. 连接到任意 Broker，获取集群元数据
2. 通过上一步的元数据，找到自己所属 Coordinator 所在的 Broker
3. 加入消费者组，获取分区消费方案
4. 获取相关分区消费进度，从上次消费的地方开始继续拉取消息，同时本地保存消费进度
5. 异步提交分区本地消费进度到 Coordinator

上面的架构图中蓝紫色的箭头展示了这样的消息消费的流程顺序

采用这样的流程去消费数据和 Kafka 的架构也是有密切联系的，因为消费的数据通过分区分布在整个集群的 Broker 中，所以需要获取整个集群的元数据了解自己需要获取的分区数据所在位置

同样在消费侧因为用了消费者组去进行负载均衡和容灾，所以消费者之间需要进行沟通、协调消费方案，但是消费者也是分布式运行的实例，所以需要 Broker 提供 Coordinator 这样的中介在消费者之间架起沟通的桥梁

#### 并发性

消费侧的并发性需要考虑两个问题：

1. 消息拉取到客户端
2. 消息偏移的提交和获取

前者支持并发，但是后者则不然

从代码上看，同一个消费者组的消费进度是没法并发提交的，有加可重入锁保护消费者组的元数据对象，每次写入的时候都需要先获取到锁

```Go
// 针对消费者组元数据的很多操作都是在临界区中完成的
group.inLock {
    ...
}
复制代码
```

更加反直觉的是，消费进度的读取操作也是同样的一把锁保护，无法并发获取，具体原因不详，但是此锁的作用可能是：

1. 保护正在使用中的消费者组不被删除
2. 消费进度出现变动（偏移过期被删除、分区扩容有新分区进度加入等），等待其他操作完成再执行

总的来讲针对一个消费者组的几乎所有操作都不支持并发（读写都是），主要目的可能就是为了保护正在使用的资源不被意外删除

### 生产

消息生产的大概流程是：

1. 连接到任意 Broker，获取集群元数据
2. 发送消息到指定的分区 Leader 副本所在的 Broker
3. 其他 Broker 上的副本向 Leader 副本同步

上面的架构图绿色箭头展示了这个流程

在这个流程中消息是通过集群元数据的提示，发往对应分区 Leader 副本所在的 Broker 上的，注意这里不允许消息在 Broker 之间进行转发

#### 并发性

一句话总结：同一个 Topic 不同分区之间是支持并发写入消息的，同一个分区不支持并发写入消息

这很好理解，单个分区是临界资源，需要用锁来进行冲突检测保证同一时间只有一批消息在写入避免出现消息乱序或者写入被覆盖的情况

### 小结

这一节的架构解析选取了 Kafka 集群中比较重要的几个角色和主流程来进行讲述，可见为了实现前面所描述的基本消息模型，需要一系列的管理机制协调、进行数据同步，还有容灾机制保障整个集群的有效运行

在早期的 Kafka 版本中，对 ZK 形成了强依赖，客户端都是通过直连 ZK 的方式去获取集群配置和更新自己的状态，不过后面的版本中逐步进行了抽象层隔离和解耦，现在需要客户端直接和 ZK 交互的地方已经没有了，都是和 Broker 打交道

这样的依赖解耦带来了简洁的接口抽象，降低了技术上的门槛，同时将部分职责从 ZK 转移到 Kafka 还提升了服务的性能

另外，目前存在的所有需要通过 ZK 去干预 Kafka 集群行为的方法，都可以通过 Admin API 或者其他接口去进行干预，这种早期需要 Hack 的暴力干预方式已经被完全淘汰

## 总结

这一部分介绍的是 Kafka 的架构，按照通常的分析思路应该先用 CAP 理论对此系统做一个定性（CP AP CA？），然后再继续展开介绍

但是我这里并没有急于给出这样的定性“结论”，究其主要原因是我认为这样的定性描述其实不够精准，很容易使我们陷入一种定式思维去看待 Kafka 并使得我们忽视了隐藏在其内部的一些细节

既然这篇文章就是对 Kafka 内在架构和机制的拆解和解读，一些细节不可忽视，所以我们继续深入探究一下 Kafka 再去讨论这个问题就能形成一个比较完整的看法了

## 核心机制

整个 Kafka Core 中权重最大、使用频率最高的三个角色是 Broker、Producer 和 Consumer，这几个角色的使用和我们的业务开发也是息息相关，对这些角色的核心机制进行深入了解对后续的业务开发、故障排查是有很大帮助

## Broker

Broker 端是 Kafka 整个内部处理流程最复杂的组件了，这当中的机制没有办法一个一个列举出来详细说，我这里选择了 Controller 和 Broker 管理机制，还有副本管理中的高水位机制来进行介绍

之所以选择这几个机制进行解读，是因为他们对帮助理解集群故障转移过程中的行为、影响面有很大帮助，其他机制都是围绕着这些核心的一些外围机制，是一些辅助角色

### Controller 选举

我下面画了一张集群故障转移的图，描述的是 Controller 因网络、硬件故障等原因下线，整个集群重新选举 Controller 的过程

​![流程图 (8).jpg](assets/167520dfe8774b99ab535652d801093ctplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-7j7e57p.jpg)​

如图所示，整个 Controller 选举的过程分四个阶段进行：

* 阶段 1：因为 Controller 和 Zookeeper 之间的会话因为超时、网络连接断开等原因失效，导致临时节点 /controller 被删除
* 阶段 2：Broker 1 和 Broker 2 监听到了 /controller 删除的事件，触发了 Controller 的重新选举
* 阶段 3：Broker 1 成功创建 /controller 节点并写入数据，Broker 2 检测到了新写入的 /controller 数据中止选举
* 阶段 4：Broker 1 作为 Controller 初始化完成，向集群中的其他节点发送更新集群元数据的请求，同步最新的数据

这个过程称之为「选举」其实有些不合适，因为这里其实基于锁的一种选主机制，先抢到锁的获得资源使用权，因为后面 Kafka 推出了基于 KRaft 选举协议的 Controller，所以这里想做一些特别说明

注意，Controller 的选举之后往往伴随着 Broker 的下线，因为 Controller 的重新选举一般就是 Broker 失效引起的，下一节会介绍这其中的相关机制

### Broker 上线下线

在线的 Controller 通过监听 /brokers 节点的异动情况处理 Broker 的上线、下线事件，这里梳理了一下整个事件处理的流程

​![流程图 (9).jpg](assets/afc0c54c83e04f44915e6ccf66881ec2tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-cimh9z8.jpg)​

整个处理的流程还是比较清晰的，分支不多，值得注意的点有几个：

1. 异动数据是通过比对 Controller 中的元数据和 ZK 的数据差异计算出来的
2. 这是个异步处理流程，在 ControllerEventManager 中用队列进行了解耦
3. 针对 bouncedBroker 的处理方式是先移除，再添加
4. KafkaController 中的 onBrokerStartup 方法执行了 Broker 上线后的存在 新增/离线 副本的分区进行领导者选举

Broker 的异动在集群中是一个非常重要的事件，因为其影响到了集群整体的可用性：

* Coordinator 需要转移到其他 Broker 上，否则与之绑定的消费者组无法正常运行，且转移期间消费者组无法正常消费
* 分区副本，尤其是领导者副本需要在 Broker 中重新分布，并且会触发分区领导者副本选举

上面两点我认为是集群 Broker 异动过程中比较核心的地方，因为 Controller 端处理完成 Broker 的元数据变更，后面的更新机制都是围绕这两个点进行

### 高水位更新

高水位是 Kafka 设计的一套用于跟踪从副本异步复制进度和保证数据一致性的机制

在架构部分简要说了一下 Kafka 的副本管理中副本数据的分布情况，这里进一步介绍一下对一个分区来说，是如何通过高水位管理数据同步进度的

这里我们用一个三副本的分区的场景来介绍该场景下高水位的值是如何更新到 4 的，如下图所示：

​![流程图 (10).jpg](assets/019590a19b0743f5be9c57fc537bef34tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-mp3tfgk.jpg)​

注意：

* 为了方便讨论，这里假设三个副本始终都在 ISR 中
* 已写入领导者副本的消息在写入时均满足最小已同步副本要求

#### 更新规则

在分析这个更新流程之前，我们先明确一下更新规则：

1. 高水位的值就是远程副本状态中远程 LEO 的最小值，注意这里不判定 ISR 是否满足最小已同步副本要求
2. 从副本同步时拉取消息的起始偏移，会被记录为此副本在 ISR 中的远程 LEO
3. 从副本拉取消息时，返回数据中包括当前最新的高水位值

整个高水位的更新流程都是基于上面这三条规则去运行的，这三条规则一起看能有点眼花缭乱，总结一下就是每次从副本发起消息同步请求的时候干两件事：

1. 上报自己的拉取消息起点，领导者副本将其当做 LEO
2. 获取领导者副本的 HW 用于更新同步本地的 HW

#### 流程解读

现在来看下这三条规则是如何应用的，更新流程如下：

* 阶段1：副本 0 和 2 消息都完全同步，仅副本 1 存在 2 条消息的延迟，这时候副本 1 发出同步请求，远程副本状态中对应的远程 LEO 更新为 4，本地 LEO 更新为 5
* 阶段2：因为远程副本状态中的远程 LEO 发生变化，领导者副本的高水位更新为 4，随后从副本 2 发出同步请求，获取到了最新的高水位 4 并更新本地值，LEO 不发生变化
* 阶段3：从副本 1 继续发出同步请求，远程副本状态的远程 LEO 此时被更新为 5，请求返回后获取到了最新的高水位 4 并更新本地值，同时远程 LEO 的更新引起领导者副本 0 高水位的变化，更新为 5，随后从副本 2 通过同步请求获取到了变化后的值，高水位也随之更新为 5

后续重复以上流程，最终所有副本的高水位和 LEO 都会更新到 5

### 小结

这部分我们介绍的是 Broker 端高可用、一致性方面的机制，其实服务端还要很多优秀机制的实现值得继续深入挖掘和学习，比如主题分区副本的绑定机制、日志文件的管理等

文档后面的【[学习资源](https://bytedance.feishu.cn/docx/doxcnb71GNKyiiwcbLHT88QLHuc#doxcnoyuYeyOWwm2Ekh1Md3trYc "https://bytedance.feishu.cn/docx/doxcnb71GNKyiiwcbLHT88QLHuc#doxcnoyuYeyOWwm2Ekh1Md3trYc")】这一部分我提供了一些线索，对这方面感兴趣的可以继续深入去看

## Producer

### 消息发送机制

目前生产端的消息发送是基于异步发送机制实现的，通过 RecordAccumulator 去做了解耦了消息生产和网络请求

这里需要先说明一下消息生产请求的结构：

​![image.png](assets/ec51df04e256413c80e41a30534ae9detplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-w25kncw.jpg)​

注意一下当前版本的 Kafka 只允许每个分区有一个批次的消息，不允许一个请求发送多个批次

下面这张场景流程图描述了 RecordAccumulator 两侧各自的处理流程，用户侧调用 send 方法之后，消息被追加到 RecordAccumulator，异步线程轮询，满足条件之后调用网络客户端的 send 方法向 Broker 发送消息生产请求

​![流程图 (11).jpg](assets/35d28a74321a47b88defce4e3475da75tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-n3tin22.jpg)​

Java 版本的实现中，send() 返回的是一个 Future 对象，所以 send().get() 这样的用法就能起到同步阻塞等待消息发送成功再返回的效果，但是本质上还是在异步发送消息

#### 消息乱序问题

我们通常认为生产者发送的消息总是能够保证分区有序，这是一种误解，因为这里有一个陷阱，就是 `max.in.flight.requests.per.connection`​ 这个客户端网络配置

查阅 Kafka 的 [官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fkafka.apache.org%2Fdocumentation%2F%23producerconfigs_max.in.flight.requests.per.connection "https://kafka.apache.org/documentation/#producerconfigs_max.in.flight.requests.per.connection") 此配置的默认值是 5，表示一个连接中可以同时有 5 个消息批次在途，文档中也明确指出了由于错误重试的关系，这种场景下消息会乱序

所以，当我们业务上对消息顺序有硬性需求的时候，这个点必须引起重视

### 消息分区机制

消息分区机制可以认为是生产端的负载均衡机制，下面梳理了一张分区计算的流程图，不同的分支对应不同的分区场景

需要注意的一点就是分区函数的入参不只是消息的 Key，Topic、Value、Cluster（集群元数据）都可以作为该函数的输入信息去计算分区

​![流程图 (12).jpg](assets/697ed6604e154bf593cd6dc7964b56a0tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-y0b2jbt.jpg)​

### 小结

这部分针对 Producer 的一些基础功能进行了一些介绍，对了解 Producer 客户端的运行已经足够

另外针对生产侧聊的比较多、比较深入的话题应该是分区消息有序性、消息幂等、精确一次语义等，这些问题单独展开都是一个大话题，在这里就不一一讨论了

## Consumer

### 成员管理

消费者组是 Kafka 消费端实现负载均衡、动态扩容、故障转移的重要机制，此机制的运行和流转需要 Broker 端的 Coordinator 和消费端的 Consumer 通过建立长连接进行交互和状态流转来完成此项工作

#### Coordinator 的定位

这里插入一个小话题，那就是消费者怎么知道自己的 Coordinator 在哪个 Broker 上，计算的过程非常简明，就是根据消费者组名的 HashCode 对 __consumer_offset 主题的分区数进行取余，代码如下：

```Scala
def partitionFor(groupId: String): Int = Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount
复制代码
```

计算出的分区领导者副本所在的 Broker 就是对应 Coordinator 的位置

注意，上述计算过程中所需的各种关于集群的信息，在获取集群元数据的阶段都缓存在了本地，这在本文的「架构-集群架构解析-消费」这一部分已经介绍过了

#### 消费者组状态机

消费者组有一个状态集合，整个消费者组就是在这几个状态之间流转的，下面我用表格和状态机图例说明这些状态怎么流转的，状态列表如下：

|**状态**|**前置状态**|**备注**|
| ---------------------| --------------------------------------------------------| -------------------------------------------------------|
|Empty|PreparingRebalance|此状态同时也是初始状态|
|PreparingRebalance|Empty, CompletingRebalance, Stable||
|CompletingRebalance|PreparingRebalance||
|Stable|CompletingRebalance|转移条件一般是 Coordinator 收到领导者发来的组同步请求|
|Dead|Empty, PreparingRebalance, CompletingRebalance, Stable|通常是 Coordinator 出现转移会导致组状态变成 Dead|

状态流转图如下：

​![流程图 (13).jpg](assets/8ab68b6f42974821a0e7747fb5cbb182tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-2iard4y.jpg)​

上图展示的是所有可能的状态流转路径，对一个新创建的消费者组来说，符合预期的流转路径是 1 → 3 → 5，下一小节介绍重平衡机制的时候会详细说明流转过程

此外，这个状态流转图中有一个危险的“死亡循环”，也就是 3 ⇆ 4 这两条路径组成的循环，下面介绍的重平衡机制与之相关

#### 重平衡机制

重平衡机制是整个消费者组管理的重要机制，因为消费者组加入、退出、消费方案的分配这些核心功能基本都囊括其中

下面我将以一个新创建的消费者组为例，介绍一下重平衡机制，案例场景的情况如下：

* Coordinator 位于 Broker 2 中，对应的消费者组成员只与其交互
* 待加入的消费者组当前不存在
* 三个消费者，且都能在规定的超时时间内入组并成功
* Consumer 0 最先发起入组请求并被处理

整个重平衡分两个大的阶段进行，第一阶段申请入组，主要是等待所有待加入消费者组的成员入组并分配成员 ID，第二阶段组同步，主要是将消费者组领导者生成的分区消费方案向全组成员进行同步

重平衡的场景流程图如下：

​![流程图 (14).jpg](assets/064e03c064584d4780ff74c0ce3e4f5etplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-wh6w0ot.jpg)​

阶段一按时序关系细分了几个步骤：

* 步骤一 Consumer 0 发起入组请求
* 步骤二因为没有成员 ID 入组请求被 Coordinator 拒绝并返回了一个有效的成员 ID
* 步骤三 Consumer 0 带入步骤二返回的成员 ID 再次入组并成功，Consumer 0 入组成功之后，其他成员陆续发起入组请求
* 步骤四 Coordinator 直接赋予其领导者身份，因为是第一个入组成功的成员

这一阶段整个消费者组状态从 Empty → PreparingRebalance，触发原因是步骤三有消费者申请入组成功（步骤一、二未触发原因是没有成员 ID 导致入组失败）

​![流程图 (15).jpg](assets/73071dc23de243aa9e96cb97446b4196tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-oj6hak4.jpg)​

阶段二按照时序关系分这么几个步骤：

* 步骤一入组等待时间结束，向所有消费者发送入组成功结果
* 步骤二所有消费者向 Coordinator 发送组同步请求，领导者 Consumer 0 发送的同步请求中携带了基于入组成功结果计算的整个消费者组的分区消费方案
* 因为步骤二收到了消费者组的分区消费方案，所以步骤三 Coordinator 向组成员广播了这个方案

这一阶段消费者组状态从 PreparingRebalance → CompletingRebalance → Stable，触发原因分别是：

1. 入组等待时间结束
2. 领导者发起了组同步请求

除了新建的消费者组之外，已有的消费者组因为很多事件也会触发重平衡机制，而且整个平衡的过程和这里的案例会有所区别

这里举了个例子只是为了帮助读者对整个重平衡过程有个大体的印象，了解整个过程中发生的主要流程，其他场景下的重平衡过程就不一一举例铺开叙述了

### 分区消费方案

分区消费方案的形成主要考虑两个步骤：

1. 消费方案的生成
2. 消费方案的分发

步骤 2 在上一节已经介绍过了，这里就说一下步骤 1

如果消费者入组成功之后被指定为了领导者，那么后续它发送的组同步请求中就带入了已经生成好的分区分配方案

生成消费方案常见的策略是这两个：

* org.apache.kafka.clients.consumer.RangeAssignor
* org.apache.kafka.clients.consumer.CooperativeStickyAssignor

值得注意的是在 Kafka 的架构上消费方案是由消费者负责生成的，主要原因我想除了性能考量之外，还有一个原因就是为了更方便消费者自定义消费策略

#### 不均衡分配问题

我们通常说的分区分配策略默认是 RangeAssignor，该策略按照主题进行分配，尽量保证每个主题的分区在消费者之间尽量平均绑定

不过这种分配策略在实现上有些问题：

1. 对消费者列表进行了排序
2. 排序后的顺序，按主题进行循环分配

因为对消费者做了排序再分配，会导致排序后的最后一个消费者总是分配到比其他消费者少的分区，造成不均衡的分配方案

​![流程图 (16).jpg](assets/a6210af8414f404baaa2b9526632b24ftplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-jbqwwhc.jpg)​

> 每一个 Topic 的分配都是不均衡的，这个偏差会逐渐累积

其他的分配策略或多或少都有类似的问题，选择业务上使用的分配策略时需要注意这一点

### 心跳保活机制

消费者加入消费者组之后，还需要保活机制维持其组成员的这个身份，保活主要通过两条路径来进行：

1. 客户端每次 poll 尝试拉取消息，Consumer 中运行在异步线程的 ConsumerCoordinator 会判定两次 poll 的时间间隔是否超出 `max.poll.interval.ms`​ 设定的值，超过则判定失效发起主动离组请求
2. 异步线程定时发送心跳包，间隔超过`session.timeout.ms`​ 则服务端判定失效，强制剔除出消费者组

如果两者之一失效消费者会被移出消费者组并触发重平衡机制，整个过程和上面介绍的重平衡机制类似

要注意上面两条路径一个是客户端本地判定，另一个是服务去判定的，第一条因为是客户端的实现，有些语言的客户端可能没这个机制

### 小结

消费者组也是一个独立的分布式服务集群，运行着业务代码，靠 Kafka 进行分布式场景下的协调和数据持久化工作

正因为消费端也是一个分布式系统，所以分布式场景下的所有问题在这里都同样存在：生产端分区、消费端分区分配机制出现问题都可能引起数据倾斜；消费者数量上去总会有节点失效的情况出现，需要对应的灾备机制进行处理；消费者之间依赖中心化的服务去协调调度，进行消费任务的分配，中心化服务的失效同样会引起消费者组的故障

## 业务场景的挑战

固然 Kafka 在设计之初对需要面临的挑战做了充分的设计和论证，但是面临真实的使用场景它的表现究竟如何？

在我们平时的业务开发场景里，使用到 Kafka 主要是做一个分布式微服务下的异步事件的生产和消费，而且绝大部分业务不会有非常大的消息吞吐量，在这些场景下 Kafka 的性能表现优异，对业务来说无法感知到性能瓶颈

但是在极端场景下，比如大数据分析的场景中，系统数据吞吐量很大，Kafka 集群中的各个组件都在承受巨大压力，任何一个单个组件的故障失效或者行为异常，都可能在集群内大范围扩散导致雪崩，引起系统性能的急剧下降，甚至是故障时效无法提供服务

至于可能造成这些不符合预期表现的原因，从上面介绍的架构、机制中我们可以找到这些问题的答案

## ZK 的巨大压力

在 1000+ Broker 的场景下，主题、分区、消费者组的数量巨大，需要在 Zookeeper 中保存大量的数据，而且这么多的节点在集群运行的过程中会频繁产生节点和数据的变动，触发事件通知 ZK 客户端

这里有个案例 [生产故障|Kafka消息发送延迟达到几十秒的罪魁祸首竟然是... | HeapDump性能社区](https://link.juejin.cn/?target=https%3A%2F%2Fheapdump.cn%2Farticle%2F3207789%3FfromComment%3Dtrue "https://heapdump.cn/article/3207789?fromComment=true") 分享的就是 ZK 对客户端请求的处理延迟过高，心跳包无法及时处理引起 Controller 和 Broker 大面积掉线，消息无法写入

ZK 是强一致性的存储系统，写入性能不佳，面对如此高频率的写入请求自然是很难应付的过来，是约束集群规模进一步扩张的重要条件

因此为了突破这一约束，进一步释放 Kafka 作为高性能消息引擎的潜力，在新发布的版本中自行实现了一套分布式一致性协议 KRaft 并支持 Controller 独立部署

​![](assets/e4deaf7ecb204663b3e98ce80bbefdcftplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-a4m3f4b.jpg)​

> 图片来源：[developer.confluent.io/learn/kraft…](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.confluent.io%2Flearn%2Fkraft%2F "https://developer.confluent.io/learn/kraft/")

目前 KRaft 版本的 Kafka 在生产环境上落地的案例很少，后续我会持续关注新机制给 Kafka 带来的变化和性能提升

另外，这里的举例只是用了一个比较夸张的集群规模，受限于硬件配置和软件版本等原因，实际的集群可能在几十个 Broker 又或者 Topic 非常多的场景下就会出现 ZK 的性能瓶颈

## 不堪重负的 Controller

前面的集群架构部分我们已经了解到，所有的 Broker 中都有一个 Controller 角色，但是同时只有一个对外提供服务，这里讨论一下这个集群唯一的 Controller 的负载问题

同样是考虑在 1000+ Broker 集群的场景下，Controller 所在的 Broker 负载会比其他 Broker 大，因为要处理整个集群范围内所有集群管理相关的请求，那么这个 Broker 就很可能因为负载过大导致节点失效，引起 Controller 选举和故障转移

在小规模的集群中这样的故障转移可以很快速，代价很小，但是在我们现在讨论的场景中集群元数据很多，同时伴随着大量的主题和分区消息数据，整个故障转移的代价非常大

转移过程中可能出现的一些异常情况：

* Controller 选举过程时间长，选举期间无法执行新建主题、分区扩容等操作
* Broker 之间进行分区副本数据的转移，大量的文件读写导致页缓存大规模失效，Broker 无法读取到到页缓存，也加入到了频繁的 IO 操作中进一步恶化 IO 性能
* 没有 Controller 导致集群元数据无法及时更新，导致客户端获取到无效的数据，无法正常工作

Controller 在集群中的地位非常重要，Kafka 及其类似的消息系统都对这一个组件做了诸多重构和优化，形成了不同的解决方案：

1. 可以将集群中的几个 Broker 独立出来，提升硬件配置，专门负责 Controller 选举
2. BMQ [3] 对 Kafka 的这部分功能进行了重构

## 不稳定的消费者

在这里我们考虑一下实际消费场景下的情况，假设有一个 100+ 消费者的消费组

前面我们已经介绍了一种场景下的重平衡机制，这里需要讨论关于重平衡对业务的影响，因为发起重平衡之后，消费者组就无法继续消费数据了，必须要等到消费者组重新进入稳定状态才可以继续消费

理想情况下，消费者成功入组之后就能持续消费，稳定运行，但是实际场景中面临如下挑战：

* 首次入组，因为不同消费者启动速度有差异，导致 99 个消费者成功入组之后，最后一个消费者申请入组触发重平衡（默认是等待 3s 进入 PrepareRebalancing）
* 消费者消费过程中，因为数据倾斜部分消费者负载高，因 GC 等原因下线或心跳超时，触发重平衡
* 消费者组运行过程中，发现消费进度跟不上，故对消费者组扩容触发重平衡

重平衡的代价很大，需要等所有消费者停止消费，然后开启申请入组、组同步的这个流程，整个重平衡期间消费者组无法消费将加剧消息消费的延迟

所以在这种消费者数量多的情况下，保证每个消费者能够稳定运行非常重要，避免因 GC 或者网络抖动等内外因素触发重平衡

虽然 Kafka 提供了消费者组这样的机制去帮助实现消费端的负载均衡和弹性扩容，但是这种扩容也是有边界的，消费集群的规模也不是能够无限扩张的，保证消费集群的稳定性是个很大问题

针对消费场景的重平衡问题，比较常见的做法是绕过这套机制自行管理分区的消费，比如我接触过的 Spark 和 Flink 大数据计算框架就是主要使用自行分配绑定分区消费，并且不使用 Kafka 提供的消息偏移管理机制或仅作为辅助手段

业务上也可以参考这种方案去实现一套消费方案的管理机制，对出现故障的消费者予以告警和及时介入，隔离故障节点和对应的分区，不要影响其他分区的正常消费

至于 Kafka 提供的消费者组静态成员的机制，这个业务案例不多，就不做介绍了

## 不可靠的代码

核心机制中介绍了生产者的消息分区函数，这是生产端负载均衡的重要机制，最常见的无 Key 或者使用哈希值计算分区的场景下，Key 总是能在分区中均匀分布

实际业务场景中分区函数不一定按照我们预期的行为向 Broker 分发消息，因为代码问题还是可能导致Key 的计算不符合预期，分区数据产生倾斜，引起部分 Broker 负载过高

因为在 Kafka Core 集群的架构里存储和计算没有分离，这种场景下因为存储导致的压力无法向其他 Broker 均摊，反而会连累整个 Broker 一起挂掉

此外，除了 Key 分区引起的数据倾斜之外，过大的消息体也可能造成问题（比如把整个文件当成消息体发送），如果因为代码错误向某个分区持续发送比较大的消息体造成数据倾斜（实际情况没有这么夸张，因为服务端对单批次的消息最大值有限制，默认是 1048588 Bytes ≈ 1MB）

> 如果是把 Kafka 当成文件系统来使用确实可能出现这个问题，因此大文件的异步消费最好是只传递文件的元信息

​![](assets/3e8ddd9ed194486f8b87d2ddce03160dtplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-onl0qc3.jpg)​

## 故障转移带来的系统冲击

前面几个场景已经多次提到了故障转移场景下的种种问题，这里做个总结性质的陈述

故障转移机制解决的是可用和自动化的问题，保证部分节点失效的情况下，系统整体是可用的，能够继续对外提供服务，而自动化能够保证故障的第一时间有一个应对机制，降低对业务的影响，给后续的人工处理和介入争取时间

因此在故障转移的场景下，我们必须考虑因为负载在节点间的重新分配，导致的节点负载变化，优化整个转移过程，提高整个服务的可用性，避免因为故障转移导致服务的严重降级，影响用户体验，或者形成尽管服务还活着但是产品事实不可用的局面

故障转移是从系统失衡进入到另一个稳态，这个转移过程必定是对原系统有冲击的，经过一段时间的重新平衡之后再回到稳定状态

在自动化的机制之外，还需要建立业务上的响应机制，提前准备好灾备方案，以便出现故障时能够及时人工介入

故障转移是一个非常值得深入讨论的技术话题，这篇文档对这个问题无法进行太深入的探究，这里就浅谈几句

## 总结

Kafka 诞生在云原生的概念尚未出现的时代，从存储底层开始一步一步构建出一套自成一体的分布式消息系统，成为业界标准的消息引擎

通过对 Kafka 架构和核心机制的深入了解，我们不难发现为了实现这个消息引擎的高可用、高性能和强一致，可以说各方面都做到了极致优化，而且每个优化的环节都是彼此关联，环环相扣，甚是巧妙

比如 Kafka 消息索引的设计就是一个好的例子，不止是提升了通过消息偏移查找的速度，更是利用到这个信息去实现了消息消费阶段的零拷贝，可以说是做到了一箭双雕，性能加倍的效果

另外，Kafka 是分布式系统的一种工程实现，不是纯理论的理想模型，如果按照 CAP 理论对其进行生搬硬套一定是行不通的，这也是文章的前面我都没有用这个理论先去解释一番的原因

按照常见的理论解释，Kafka 应该是 AP 系统，用消息分区进行负载均衡提升性能，用副本机制保证 Broker 失效的场景下的可用性，副本之间的消息是最终一致性的关系

很显然，这样的 AP 系统是无法在线上使用的（想想看 unclean.leader.election.enable 参数线上集群能用否？），副本之间是纯异步复制，随时会有未同步副本上线对外提供服务，没有人愿意承担这种因为数据一致性问题带来的消息永久丢失风险

所以 Kafka 在设计和实现的时候就是在 CAP 三角中结合自己系统的应用场景做出了取舍，形成了有限的分区容错（分区失效服务会短暂不可用）、整体可用（集群可以部分失效）、弱一致性（高水位和副本选举机制）的可能三角，各方面都有兼顾

> 你以为的 AP 系统，其实是……

​

也正是因为种种现实原因的限制，Kafka 的设计和实现上注定有不完美的地方，也有很多的历史遗留问题，这些问题我们文章的前面也都有提到，这些问题或大或小，正是这些问题的存在进一步推动了 Kafka 自身的迭代进化，也催生出了其他的可能性

Pulsar 这样的云原生消息系统就是在架构上对 Kafka 计算存储一体的架构做了改进，存储计算分离，充分利用了当今云原生环境的带来的扩展性优势，成为当下热门的项目

此外还有字节自研的 BMQ 这样的消息中间件，同样对 Kafka 架构还有内部核心机制的实现上做了重构和一些优化，去支撑自己的业务需求

> Pulsar 的架构图如下，可以明显看到集群架构中的存储与计算分离的设计，负责存储的 Bookie 是独立的集群，与消息引擎解耦，BMQ 也有类似的设计，只不过用的组件不同

​![](assets/8617f585c97d40ddae64f4d1855ac02btplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-kocp5eo.jpg)​

总的来说 Kafka 作为一个出道十几年的项目，久经考验，生命力顽强，确实是消息系统的经典设计

## 学习资源

## 源码学习

### 入门指南

Kafka 是 Java 和 Scala 语言混合的项目，目前 IDEA 对这种项目的支持度最好，想要学习源码的话推荐用这个 IDE，省去很多解决环境问题的时间

开始阅读服务端代码的话，推荐从两个地方开始：

* KafkaApis 这个类是各种外部请求 Handler 的入口，从这里能看到 Kafka 各种接口的实现
* KafkaServer 这个类是服务端 Broker 对应的实现类，从这里作为入口能学习到整个服务端启动的流程

阅读客户端代码的话，Producer 可以关注 RecordAccumulator 这个类，这是解耦本地消息缓存和网络发送线程的枢纽

Consumer 应该关注 ConsumerCoordinator 的实现，关于消费者组管理客户端的实现都在这里，想了解重平衡等客户端核心机制都需要在这里找答案

### 代码导航

我把源码中出现的重要角色按照类名做了梳理，介绍了主要职责，供大家学习源码的时候参考

​![image.png](assets/ada076510e7940a0a4cb821334027db0tplv-k3u1fbpfcp-zoom-in-crop-mark1512000-20230313172242-s07gt0v.jpg)​

## KIP

KIP 是 Kafka Improvement Proposal[5] 的缩写，一些重大的功能变更都是用这样的改进提案的方式发起的，这些提案对了解一些内部机制实现的动机和过程有非常大的帮助，属于非常优质的第一手学习资料

这篇文章中引用了很多经典的 KIP，这里梳理了一个列表出来方便后续对社区动态的跟进和源码的学习

|**KIP 编号**|**提议内容**|**备注**|
| --| -----------------------------------------------------------------------------------| ------------------------------------------------------------------------------------|
|[KIP-63](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-62%253A%2BAllow%2Bconsumer%2Bto%2Bsend%2Bheartbeats%2Bfrom%2Ba%2Bbackground%2Bthread "https://cwiki.apache.org/confluence/display/KAFKA/KIP-62%3A+Allow+consumer+to+send+heartbeats+from+a+background+thread")|允许消费者从后台线程发送心跳包||
|[KIP-82](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-82%2B-%2BAdd%2BRecord%2BHeaders "https://cwiki.apache.org/confluence/display/KAFKA/KIP-82+-+Add+Record+Headers")|消息添加头部信息（Headers）|除了消息体之外，允许添加键值对形式的头部信息，用于路由或者过滤等功能|
|[KIP-106](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-106%2B-%2BChange%2BDefault%2Bunclean.leader.election.enabled%2Bfrom%2BTrue%2Bto%2BFalse "https://cwiki.apache.org/confluence/display/KAFKA/KIP-106+-+Change+Default+unclean.leader.election.enabled+from+True+to+False")|unclean.leader.election.enabled 配置的默认值从 true 变为 false||
|[KIP-134](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-134%253A%2BDelay%2Binitial%2Bconsumer%2Bgroup%2Brebalance "https://cwiki.apache.org/confluence/display/KAFKA/KIP-134%3A+Delay+initial+consumer+group+rebalance")|延迟首次初始化消费者组的重平衡操作||
|[KIP-392](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-392%253A%2BAllow%2Bconsumers%2Bto%2Bfetch%2Bfrom%2Bclosest%2Breplica "https://cwiki.apache.org/confluence/display/KAFKA/KIP-392%3A+Allow+consumers+to+fetch+from+closest+replica")|允许消费者从最近的副本拉取数据|消费者的定义不含从副本向领导者副本的消费角色|
|[KIP-500](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-500%253A%2BReplace%2BZooKeeper%2Bwith%2Ba%2BSelf-Managed%2BMetadata%2BQuorum "https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum")|移除对 Zookeeper 的依赖，基于 Raft 协议开发出一套新的选举协议进行集群元数据的管理||
|[KIP-516](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-516%253A%2BTopic%2BIdentifiers "https://cwiki.apache.org/confluence/display/KAFKA/KIP-516%3A+Topic+Identifiers")|主题唯一标识符|阅读 Motivation 的部分了解为什么要添加这个标识符即可，其他的技术细节不是特别重要了|
|[KIP-555](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-555%253A%2BDeprecate%2BDirect%2BZookeeper%2Baccess%2Bin%2BKafka%2BAdministrative%2BTools "https://cwiki.apache.org/confluence/display/KAFKA/KIP-555%3A+Deprecate+Direct+Zookeeper+access+in+Kafka+Administrative+Tools")|Kafka 管理工具中废弃与 Zookeeper 的直连功能||
|[KIP-584](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-584%253A%2BVersioning%2Bscheme%2Bfor%2Bfeatures "https://cwiki.apache.org/confluence/display/KAFKA/KIP-584%3A+Versioning+scheme+for+features")|集群功能版本控制|用于解决集群滚动升级，以及不同版本客户端兼容的场景下的一些问题|
|[KIP-630](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-630%253A%2BKafka%2BRaft%2BSnapshot "https://cwiki.apache.org/confluence/display/KAFKA/KIP-630%3A+Kafka+Raft+Snapshot")|Kafka Raft 快照|属于 KIP-500 的后续功能性 KIP|
|[KIP-631](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKIP-631%253A%2BThe%2BQuorum-based%2BKafka%2BController "https://cwiki.apache.org/confluence/display/KAFKA/KIP-631%3A+The+Quorum-based+Kafka+Controller")|基于选举的 Kafka Controller|属于 KIP-500 的后续功能性 KIP|

## 其他

* Kafka 官方文档[1] 是最权威的第一手资料，平时用来当手册查询各种配置的文档非常方便
* BMQ 的设计文档[3] Motivation 的部分指出了 Kafka 作为大规模消息队列集群的问题，整个设计文档就是对 Kafka 的再思考和重构，是从另一个角度审视 Kafka 的好材料
* [Why BMQ](https://bytedance.feishu.cn/docs/doccnTg2ZHwEbq7aQv0VXoW6gDb "https://bytedance.feishu.cn/docs/doccnTg2ZHwEbq7aQv0VXoW6gDb") [4] 这篇文档针对不同实际运维场景，将 Kafka 和 BMQ 进行对比，指出了各自优缺点

## 参考文献

1. [Kafka 官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fkafka.apache.org%2Fdocumentation%2F "https://kafka.apache.org/documentation/")
2. [Apache Kafka® Internal Architecture - The Fundamentals](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.confluent.io%2Flearn-kafka%2Farchitecture%2Fget-started%2F "https://developer.confluent.io/learn-kafka/architecture/get-started/")
3. [BMQ](https://bytedance.feishu.cn/wiki/wikcnqUcaMdIzezRkJxhVODMEEz "https://bytedance.feishu.cn/wiki/wikcnqUcaMdIzezRkJxhVODMEEz") 设计文档
4. [Why BMQ](https://bytedance.feishu.cn/docs/doccnTg2ZHwEbq7aQv0VXoW6gDb "https://bytedance.feishu.cn/docs/doccnTg2ZHwEbq7aQv0VXoW6gDb") 9.[Kafka Improvement Proposal](https://link.juejin.cn/?target=https%3A%2F%2Fcwiki.apache.org%2Fconfluence%2Fdisplay%2FKAFKA%2FKafka%2BImprovement%2BProposals "https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals") (KIP)

---
