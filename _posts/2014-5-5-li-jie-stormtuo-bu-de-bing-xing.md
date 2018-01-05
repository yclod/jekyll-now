---
layout: post
title: 理解Storm的并行
---

翻译自：http://storm.incubator.apache.org/documentation/Understanding-the-parallelism-of-a-Storm-topology.html

### 是什么让topology运行起来的：worker processes，executeors和tasks

真正让topology在Storm集群上运行起来的，是以下三个不同的实体：

	1. Worker processes（工人进程）
    2. Executors（执行线程）
    3. Tasks
    
这是它们之间关系的简单示意图：

![](http://storm.incubator.apache.org/documentation/images/relationships-worker-processes-executors-tasks.png)

`worker process`运行topology的一个子集。一个工人进程属于一个特定的topology，可能为该topology上的一个或多个组件（spouts或bolts）运行着多个executor。在Storm集群中，一个正在运行的topology有很多这样的进程，并且运行在很多台不同的机器上。

`executor`是worker process中的一个线程。它为同一个组件（spout或者bolt）运行一个或多个tasks。

`tasks`扮演着真正的数据处理者 -- 你代码实现的每一个spout或bolt都在集群里运行着很多tasks。在一个topology的生命周期中，某个组件运行tasks的个数保持不变，但组件里执行的线程（executor）随时都在改变。这意味着有这样的等式：#threads ≤ #tasks. 默认情况下，tasks的数量被设置等于executors的数量，比如Storm中每个线程上运行一个task。

### 配置topology的并行

请注意，在Storm的术语中，“并行”特指*parallelism
 hint*，表示一个组件初始化时候executor（线程）的数量。在本文档中，我们使用“并行”更宽泛的意义，既表示你可以设置的executor数量，也表示worker processes的数量和Storm中topology上tasks的数量。我们在需要使用Storm中正常狭义的“并行”的定义时，会重点强调。

以下章节给出了各个配置选项的概述，以及告诉你如何在代码中设置这些配置选项。有不止一个办法设置这些选项，表中仅列出了其中一些。Storm目前的配置按照以下优先级顺序：

>defaults.yaml < storm.yaml < topology-specific configuration < internal component-specific configuration < external component-specific configuration.

#### 工人进程（worker processes）的数量

* 描述：在整个集群的所有机器上，该topology包含的工人进程的数量
* 配置选项：[TOPOLOGY_WORKERS](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html#TOPOLOGY_WORKERS)
* 如何设置你的代码（例子）
	* [Config#setNumWorkers](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html)


#### 执行线程（executors -- threads）的数量

* 描述：每个组件上执行的总线程的数量
* 配置选项：？
* 如何设置你的代码（例子）
	* [TopologyBuilder#setSpout()](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * [TopologyBuilder#setBolt()](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * 注意在Storm 0.8版本中，parallelism_hint参数表示bolt上初始化的executors的数量（而不是tasks的数量！）。

#### tasks的数量

* 描述：每个组件上tasks的数量
* 配置选项：[TOPOLOGY_TASKS](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html#TOPOLOGY_TASKS)
* 如何设置你的代码（例子）
	* [ComponentConfigurationDeclarer#setNumTasks()](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/ComponentConfigurationDeclarer.html)
    
这是一个实践中的代码片段的例子：

<pre><code class="language-java">
topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout);
</code></pre>

在上面的代码中，我们配置Storm去运行叫做GreenBolt的bolt，初始化了两个executors和4个相对应的tasks。Storm会在每个执行线程上运行两个tasks。如果你不想明确的配置tasks的数量，Storm会使用默认配置--每个executor一个task。

### 运行topology的例子

下图表示一个简单的topology是如何工作的。这个topology由3个组件组成：一个叫做BlueSpout的Spout和两个分别叫做GreenBolt，YellowBolt的bolts。这些组件相互连接，由BlueSpout发送其输出到Greenbolt，然后再将其输出发送给YellowBolt。

![](http://storm.incubator.apache.org/documentation/images/example-of-a-running-topology.png)

GreenBolt的配置如上述代码片段中所示，BlueSpout和YellowBolt只设置了parallelism hint（executors的数量）。以下是相应的代码：

<pre><code class="language-java">
Config conf = new Config();
conf.setNumWorkers(2); // use two worker processes

topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // set parallelism hint to 2

topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");

topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
               .shuffleGrouping("green-bolt");

StormSubmitter.submitTopology(
        "mytopology",
        conf,
        topologyBuilder.createTopology()
    );
</code></pre>

当然，在Storm中还有一些额外的配置选项用来控制topology的并行，包括：

* [TOPOLOGY_MAX_TASK_PARALLELISM](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html#TOPOLOGY_MAX_TASK_PARALLELISM): 这个选项为单个组件可以设置的执行线程数设了一个上限。它通常用在测试中，在本地模式下限制运行topology的线程的数量。你可以通过来设置[Config#setMaxTaskParallelism()](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html)该选项。

### 如何改变运行中topology的并行状态

Storm一个很有意思的功能是你可以增加或减少工人进程或执行线程的数量而不需要重启集群或topology。这样的做法也称为：rebalancing（重新平衡）。

你有两个选项可以用来重新平衡一个topology：

	1. 使用Storm web UI来重新平衡topology。
    2. 使用storm的CLI工具
 
下面是使用CLI工具的例子：

<pre><code class="language-bash">
# Reconfigure the topology "mytopology" to use 5 worker processes,
# the spout "blue-spout" to use 3 executors and
# the bolt "yellow-bolt" to use 10 executors.

$ storm rebalance mytopology -n 5 -e blue-spout=3 -e yellow-bolt=10
</code></pre>

### 参考文献

* [概念](http://storm.incubator.apache.org/documentation/Concepts.html)
* [配置](http://storm.incubator.apache.org/documentation/Configuration.html)
* [在生产环境集群上运行topology](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)
* [本地模式](http://storm.incubator.apache.org/documentation/Local-mode.html)
* [Storm教学](http://storm.incubator.apache.org/documentation/Tutorial.html)
* [Storm API文档](http://storm.incubator.apache.org/apidocs/)，尤其是其中的Config类
   
