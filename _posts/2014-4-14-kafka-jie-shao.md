---
layout: post
title: Kafka 介绍
---

最近工作中用到了两个很给力的项目，一个是Kafka，一个是Strom。本着自我学习并方便他人的目的，我会将我觉得比较有用的英文文档翻译在此（保留系统专有名词不作翻译）。

好了，废话不多说了，以下文档翻译自[Apache Kafka 0.8.1官方文档](https://kafka.apache.org/documentation.html#introduction)。

## Kafaka 0.8.1 文档
### 1. 让我们开始吧
#### 1.1 介绍

Kafka是一个分布式的（distributed），可划分的（partitioned），冗余备份的持久性的日志服务（replicated commit log service）. 提供消息系统（messaging system）的功能，拥有独一无二的设计。

这是什么意思呢？

首先，让我们看一下“消息（messaging）”的基本术语：

* `topics`: （翻译过来叫话题）特指Kafka处理的消息源（feeds of messages）的不同分类。
【注：feeds翻译为源，饲料，在计算机领域，特指一种用户从数据源接收消息的机制（见[Wikipedia](http://en.wikipedia.org/wiki/Data_feed)），比如RSS feeds等。这里指接收messages.】

* `producers`：向Kafka的一个topic发布消息的过程叫做*producers*。

* `consumers`：订阅topics并处理其发布的消息的过程叫做*consumers*。

* `broker`：Kafa集群中的一台或多台服务器统称为*broker*。

所以，从总体来说，`produces`通过网络向Kafka的集群发送消息并转由`consumers`来处理。这个过程如下图所示：

![alt](https://kafka.apache.org/images/producer_consumer.png)

客户端和服务器端的通信，是基于简单，高性能，且与编程语言无关的[TCP协议](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)。虽然我们为Kafka提供了Java版本的客户端，但是客户端其实可以使用[多种语言](https://cwiki.apache.org/confluence/display/KAFKA/Clients)。

##### Topics与日志

让我们首先来研究一下Kafka提供的这个抽象的概念 -- `topic`

一个`topic`是指发布的消息的一个类别或消息源的名字。Kafka集群将为每一个`topic`划分一个日志（partitioned log）如下图所示：

![alt](https://kafka.apache.org/images/log_anatomy.png)

每一个partition是一个排好序的，不可变的消息序列。新的消息不断的追加到序列的尾部。--即持久性日志（commit log）。每一个分区（`partition`）中的消息，有一个名叫`offset` 的顺序编号，作为这条消息在`partition`中的标识。

Kafka集群将在设定的时间范围内，保存所有被发布的消息，不论该消息是否被处理完成。例如，如果一个日志被设置为保存2天，那么在它发布的两天之内，它都是可以被处理的，而在2天之后，它就会被系统销毁并释放掉。

Kafka的性能与数据的大小之间是常数的关系，所以保存大量的数据是没有问题的。

实际上，每个`consumer`中仅有的元数据（metadata）的主要部分是该`consumer`在日志中的位置信息，叫做`offset`（偏移量）。这个`offset`由`consumer`控制，在一般情况下，`consumer`按照`offset`的顺序读取消息，但事实上`consumer`可以控制位置，可以以任何想要的顺序处理消息。例如一个`consumer`可以重置并重新处理一个已经处理过的`offset`。

这些特点的组合使Kafka的`consumers`变得特别的廉价--它们能来去自如而不会对集群或者其它的`consumers`造成多大影响。比如，你可以使用我们的命令行工具来“tail”任意`topic`中的内容，而不会改变任何被已有`consumers`处理过的内容。

日志服务的分区有几个目的。首先，它允许日志扩展到超过单台服务器允许的大小。因为虽然每一个单独的分区必须适应承载它们的服务器，但是一个`topic`可以包含多个分区，所以能处理任意大小的数据。其次，它们作为并行单元--一会儿我们会了解更多。

##### 分布式

日志的分区`partitions`分布式地部署在Kafka服务器集群上，每个服务器为一个共享的分区处理数据和请求。每一个分区的数据被冗余的备份在多台服务器上用于容错，可以通过配置来设定用于备份的服务器的数量。

每个分区有一台服务器扮演“领导”的角色，有0或多台服务器扮演“随从”。领导负责所有对分区的读写请求，同时随从们被作为领导的备份。一旦领导挂掉，随从中的一个会自动变成新的领导。每一台服务器都同时在一些分区里扮演领导而在另外一些分区中充当随从，这让集群拥有良好的负载均衡。

##### Producers （生产者）

`producers`向他们选择的`topics`发布数据。每个`producer`负责在`topic`中选择将哪些消息分配给哪些分区。这可以通过简单的“循环赛”的方式来或是根据一些语义划分的方法（比如根据一些消息中的键）来实现负载均衡。我们会在之后介绍如何使用分区时提供更多的信息。

##### Consumers （消费者）

传统的消息系统有两种模式：[消息队列](http://zh.wikipedia.org/wiki/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97)和[发布/订阅](http://zh.wikipedia.org/wiki/%E5%8F%91%E5%B8%83/%E8%AE%A2%E9%98%85)。在消息队列模式中，一池子的consumers可能从一个服务器上读取数据，每个消息被分发给其中一个consumer；而在发布订阅模式中，消息被以广播的方式发给所有的consumers。Kafka通过提供了一个叫做*消费者群组*（*consumer group*）的抽象概念涵盖以上两种模式。

`Consumers`上标记有他们所属的*consumer group*的名字，每个被发布到`topic`上的消息消息会被分发到所有订阅该`topic`的*consumer group*内部的一个*consumer实例*上。*Consumer实例*可以是一个单独的进程也可以是一个单独的机器。

如果所有的consumer实例都具有相同的群组，那么就像传统的队列模式一样平衡着各个consumer的负载。

如果所有的consumer实例均有不同的群组，那么这就如同发布/订阅模式，所有的消息被广播给所有的消费者。

更常见的情况是，我们发现topics一般只有很少的消费者群组，一个群组一般对应一个“逻辑订阅”单元。而每一个群组由大量的consumer实例构成，用来提供可扩展性和容错性。这其实就是发布/订阅模式的一种特殊情况，只不过订阅者是一个consumers的集群而非一个单独的进程而已。

![](https://kafka.apache.org/images/consumer-groups.png)

同时，Kafka具有比传统消息系统更强大的顺序保障。

传统的队列在服务器上按照一定的顺序存储消息，然后当多个consumers从队列中处理消息时，系统按照消息存储的顺序分发消息。然而，虽然系统是按照顺序送出消息的，但是是按异步的方式送达到consumer手中，所以当消息到达不同consumer手中的时候，已经没有顺序可言了。这意味着消息的顺序在并行处理中不复存在了。消息系统常常有一个权宜之计来应对这种情况，就是使用了一个叫做“独家消费”的概念，就是只允许一个进程处理队列，不过这么做的话，并行处理当然也就不复存在了。

Kafka在这一点上做的比较好。通过一个叫做“排比(parallelism)”的概念--即并行--在topics中， Kafka能够同时提供顺序保证和一池子消费进程间的负载均衡。

Kafka只能保证每个分区内部的消息的总体顺序，而保证同一个topic在不同分区中消息的顺序。这种每个分区有序并可以按照数据的键去分区的特性对于大多数应用都已经足够。但是，如果你需要保证所有消息的总体顺序，可以通过使用只有一个分区的topic去完成，不过这样做就意味着只有一个consumer进程了。

##### 保障
在高层次上Kafka提供如下保障：

* 由producer发送给特点topic分区的消息按照发送的先后顺序排序。也就是说，如果同一个producer发送了消息M1和M2，M1先被发送，那么M1的offset比M2的小，且M1先出现在日志中。
* 一个consumer实例按照消息在日志中存储的顺序收到消息。
* 对于一个有N个备份的topic，我们允许其中N-1个服务器挂掉，依然能保证不丢失任何持久性日志中的消息。

更多关于保障更多的细节，请参阅文档关于设计的章节。

#### 1.2 使用案例

下文会描述一些Apache Kafka流行的使用案例。对于其中一些领域实践的概述，请参考[此博客文章](http://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)。

##### 消息

Kafka可以很好的作为更为传统消息代理的替代品。消息代理可以用来做很多事（将消息处理与数据生产解耦，缓冲未处理消息，等）。比起大多数的消息系统来说，Kafka有更好的吞吐量，内置的分区，冗余及容错性，这让Kafka成为了一个很好的大规模消息处理应用的解决方案。

根据我们的经验，消息系统一般吞吐量相对较低，但是需要更小的端到端延时，并尝尝依赖于Kafka提供的强大的持久性保障。

在这个领域，Kafka足以媲美传统消息系统，如[ActiveMR](http://activemq.apache.org/)或[RabbitMQ](https://www.rabbitmq.com/)。

##### 网站行为跟踪

最原始的Kafka是作为一个实时发布/订阅的数据源的集合用来跟踪和重建一个用户的行为。这意味着网站的活动（页面浏览，搜索，或是其他可能的用户行为）被发布到中央topic集群上，其中每一个topic对应于一个活动类型。这些数据源可以提供一系列用例的订阅包括实时处理，实时监控，载入Hadoop或离线数据仓库（data warehousing）系统的离线处理及报告。

行为跟踪的规模一般非常大，因为每个用户的页面浏览会产生大量的活动消息。

##### 指标

Kafka常常用于运行时数据监控。这包括从分布式应用到产生统一的运行数据的汇总统计。

##### 日志聚合

很多人使用Kafka代替日志聚合（log aggregation）。日志聚合一般来说是从服务器上收集日志文件，然后放到一个集中的位置（文件服务器或HDFS）进行处理。然而Kafka忽略掉文件的细节，将其更清晰地抽象成一个个日志或事件的消息流。这就让Kafka处理过程延迟更低，更容易支持多数据源和分布式数据处理。比起以日志为中心的系统比如Scribe或者Flume来说，Kafka提供同样高效的性能和因为复制导致的更高的耐用性保证，以及更低的端到端延迟。

##### 流处理

很多用户会将那些从原始topic来的数据进行阶段性处理，汇总，扩充或者以其他的方式转换到新的topic下再继续后面的处理。例如一个文章推荐的处理流程，可能是先从RSS数据源中抓取文章的内容，然后将其丢入一个叫做“文章”的topic中；后续操作可能是需要对这个内容进行清理，比如回复正常数据或者删除重复数据，最后再将内容匹配的结果返还给用户。这就在一个独立的topic之外，产生了一系列的实时数据处理的流程。[Strom](http://storm.incubator.apache.org/)和[Samza](http://samza.incubator.apache.org/)是非常著名的实现这种类型数据转换的框架。

##### 事件源

事件源是一种应用程序设计的方式，该方式的状态转移被记录为按时间顺序排序的记录序列。Kafka可以存储大量的日志数据，这使得它成为一个对这种方式的应用来说绝佳的后台。

##### 持久性日志（commit log）

Kafka可以为一种外部的持久性日志的分布式系统提供服务。这种日志可以在节点间备份数据，并为故障节点数据回复提供一种重新同步的机制。Kafka中日志压缩功能为这种用法提供了条件。在这种用法中，Kafka类似于Apache BookKeeper项目。

#### 1.3 快速开始

以下教程假设你是一位初学者并且没有Kafka或者ZooKeeper的数据。

##### 第一步：下载代码

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.8.1/kafka_2.9.2-0.8.1.tgz)0.8.1版本并解压缩：
<pre><code class="language-markup">
> tar -xzf kafka_2.9.2-0.8.1.tgz
> cd kafka_2.9.2-0.8.1
</code></pre>

##### 第二步：启动服务

Kafka使用zookeeper，所以如果没有的话，你首先需要启动一个zookeeper服务。你可以使用包中含带的简易的脚本来快速获得一个“肮脏”的zookeeper单节点实例。

<pre><code class="language-markup">
> bin/zookeeper-server-start.sh config/zookeeper.properties
[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
...
</code></pre>

现在启动Kafka服务：
<pre><code class="language-markup">
> bin/kafka-server-start.sh config/server.properties
[2013-04-22 15:01:47,028] INFO Verifying properties (kafka.utils.VerifiableProperties)
[2013-04-22 15:01:47,051] INFO Property socket.send.buffer.bytes is overridden to 1048576 (kafka.utils.VerifiableProperties)
...
</code></pre>

##### 第三步：创建一个topic

让我们在一个分区上创建一个只有一个副本的topic，取名叫“test”：
<pre><code class="language-markup">
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
</code></pre>

我们现在可以通过运行`list`命令来查看topic：
<pre><code class="language-markup">
> bin/kafka-topics.sh --list --zookeeper localhost:2181
test
</code></pre>

此外，你也可以通过配置broker，使得一旦有消息发布到一个不存在的topic时，自动创建该topic。

##### 第四步：发送一些消息

Kafka可以通过一个命令行工具，从文件或者标准输入来接受并向Kafka集群发送消息。在默认情况下每一行都会被当做一条单独的消息发送。

运行producer（消息的生产者），并向控制台敲一些消息消息发送到服务器。

<pre><code class="language-markup">
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
This is a message
This is another message
</code></pre>

##### 第五步：启动consumer（消息消费者）

Kafka同样包含一个命令行工具用来从“标准输出”向外倾倒消息：

<pre><code class="language-markup">
> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
This is a message
This is another message
</code></pre>

如果你在控制台的不同窗口运行上述的命令，那么你现在应该可以在其中producer的那个窗口输入一些消息，然后在consumer的那个窗口里看到这些消息。

##### 第六步：配置多个broker集群。

到目前为止，我们都是在单个broker上运行的，但是这没啥好玩的。对于Kafka来说，单个broker其实就是一个大小为1的集群，所以对于启动多个broker的实例来说，道理也是一样的，并没有太多变化。但是为了感觉一下他，就让我们将我们的集群扩充道3个节点（仍然全部运行在我们的本地机器上）。

首先我们为每一个broker建一个配置文件：

<pre><code class="language-markup">
> cp config/server.properties config/server-1.properties 
> cp config/server.properties config/server-2.properties
</code></pre>

现在，编辑这些新文件，并设置以下属性：

<pre><code class="language-bash">
config/server-1.properties:
    broker.id=1
    port=9093
    log.dir=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    port=9094
    log.dir=/tmp/kafka-logs-2
</code></pre>

其中`broker.id`属性是一个不重复的常量，用来表示集群中每个节点的名字。我们在这里不得不重写`port`和`log.dir`，这只是因为我们是在同一台机器上运行这些命令，而我们要防止多个borker使用同一个端口注册而覆盖彼此的内容。

我们已经有了Zookeeper并且我们的单节点已经启动，所以我们现在需要启动这两个新节点：

<pre><code class="language-markup">
> bin/kafka-server-start.sh config/server-1.properties &
...
> bin/kafka-server-start.sh config/server-2.properties &
...
</code></pre>

现在创建一个有三个备份因子的新topic：

<pre><code class="language-markup">
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
</code></pre>


好了，现在我们有一个集群了，但是我们怎么知道每个个broker都在做什么呢？让我们运行“describe topics”命令来看看：

<pre><code class="language-markup">
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: my-replicated-topic	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
</code></pre>

这是上面输出的说明。第一行给出了所有分区的总结，此外每一行都是一个分区的信息。因为我们现在在这个topic上只有两个分区，所以就只有两行。

* "leader" 负责给定分区中所有的读和写的任务。分区将随即选取一个节点作为leader。

* “replicas” 列出了所有当前分区中的副本节点。不论这些节点是否是leader或者是否处于激活状态，都会被列出来。

* “isr” 是表示“在同步中”的副本节点的列表。是replicas列表的一个子集，包含了当前处于激活状态的节点，并且leader节点开头。

注意在我们的例子中，节点1该topic仅有的一个分区中的leader节点。

我们可以在之前我们创建的topic中运行同样的命令，来看看是什么情况：

<pre><code class="language-markup">
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
</code></pre>

看，和猜测的一样 -- 在之前的topic下没有副本节点，且其运行在server 0上，它是我们在创建topic时在集群中创建的唯一一个server。

让我们向我们的新topic发布一些消息：

<pre><code class="language-markup">
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C 
</code></pre>

现在让我们消费这些消息：

<pre><code class="language-markup">
> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
</code></pre>

现在让我们测试一下容错性。Broker 1是其中的leader，让我们关了它：

<pre><code class="language-markup">
> ps | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home/bin/java...
> kill -9 7564
</code></pre>

Leader节点转移了，并且1号节点不再存在于“正在同步”的副本集合内：

<pre><code class="language-markup">
> bin/kafka-topics.sh --describe --zookeeper localhost:218192 --topic my-replicated-topic
Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: my-replicated-topic	Partition: 0	Leader: 2	Replicas: 1,2,0	Isr: 2,0
</code></pre>

但是这些消息仍然可以用来消费，即便是原本负责写的leader节点被关掉了：

<pre><code class="language-markup">
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
</code></pre>


#### 1.4 生态系统

Kafka整合了大量的与其他分布式系统模块交互的工具。生态系统页面列出了很多工具，比如针对数据流处理系统，Hadoop的集成，监控系统和用来部署环境的工具。

#### 1.5 版本的更新：

##### 从0.8.0到0.8.1

0.8.1与0.8完全兼容。可以通过依次地对每个broker进行：关闭，更新代码和重新启动的简单操作来升级。

##### 从0.7

从0.8开始，我们添加了冗余备份的功能，这是我们第一个向后兼容的版本：主要更新在于API，ZooKeeper的数据结构，协议和配置。从0.7到0.8.x的升级需要借助特殊的迁移工具。这种迁移无需停机即可完成。

（第一章完）