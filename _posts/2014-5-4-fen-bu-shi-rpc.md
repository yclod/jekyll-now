---
layout: post
title: 分布式RPC
---

翻译自：http://storm.incubator.apache.org/documentation/Distributed-RPC.html

DRPC（Distrubuted Remote Procedure Call）背后的理念是用来并行的处理大规模计算。Storm的topology将流作为函数的输入，并将函数的调用结果作为流输出出来。

DRPC并不是一个Storm的功能，而是一个基于Storm原语（流，spouts，bolts和topologies）的模式。虽然DRPC可以单独的从Storm中打包出来，但是它和Storm绑在一起使用是非常强大的。

### 高层次概述

分布式RPC需要搭配一个”DPRC服务器“（Strom的包里含有一个这个的实现）。DRPC服务器接收RPC请求，向Storm的topology发送请求，从Storm的topology接收结果，然后将结果发送回等待的客户端。从客户端来看，一个分布式RPC调用看起来就像是一个传统的RPC调用一样。例如，下面是一个客户端怎样使用参数http://twitter.com 调用”reach“函数产生计算结果：

<pre><code class="language-java">
DRPCClient client = new DRPCClient("drpc-host", 3772);
String result = client.execute("reach", "http://twitter.com");
</code></pre>

该分布式RPC的工作流程如下图所示：
![](http://storm.incubator.apache.org/documentation/images/drpc-workflow.png)

一个客户端向DRPC服务器发送了一个需要执行的函数的名字以及其参数。Topology使用DRPCSpout实现了一个函数用来接收从DRPC服务器过来的函数调用请求的流。DRPC服务器会给每一个函数调用打上一个唯一id标识。然后Topology计算出结果并最终topology中的一个bolt会调用ReturnResult连接到DRPC服务器，将该id标识的函数调用的结果交给服务器。DRPC服务器使用该id与正在等待的客户端匹配结果，解锁等待中的客户端，然后将该结果发给客户端。

#### LinearDRPCTopologyBuilder

Storm里有一个topology的builder，叫做LinearDRPCTopologyBuilder。它基本上可自动化与DRPC有关的所有步骤，包括：

	1. 设置spout
    2. 返回结果到DRPC服务器
    3. 为bolts提供tuples群组间的有限聚合的能力。（doing finite aggregations over groups of tuples）
    
让我们来看一个简单的例子。这是DRPC topology的一个实现，返回值是输入参数后加了一个"!"：


<pre><code class="language-java">
public static class ExclaimBolt extends BaseBasicBolt {
	public void execute(Tuple tuple, BasicOutputCollector collector){
    	String input = tuple.getString(1);
        collector.emit(new Values(tuple.getValue(0), input + "!"));
    }
    
    public void declareOutputFields(OutputFieldsDeclarer declarer){
    	declarer.declare(new Fields("id", "result"));
    }
}

public static void main(String[] args) throws Exception {
	LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("exclamation");
    builder.addBolt(new ExclaimBolt(), 3);
    // ...
}
</code></pre>

像你看到的一样，没啥好说的。当创建一个LinearDRPCTopologyBuilder时，你告诉它DRPC方法在topology中的名字。一个单DRPC服务可能包含多个函数，这个函数名称用来区别彼此。你声明的第一个bolt需要输入两个tuples，第一个的field是请求的id，第二个的field是请求的参数。LinearDRPCTopologyBuilder设定最后一个bolt会发出包含两个tuples的表[id, result]的流。最终，所有过程中的tuples必须包含请求的id作为第一个field。

在这个例子中，EclaimBolt简单地在tuple的第二个field后追加了一个"!"。LinearDRPCTopologyBuilder负责接收链接DRPC服务器得到的结果，然后将结果返回。

### 本地模式DRPC

DRPC可以运行在本地模式。以下是如何在本地模式运行的例子：

<pre><code class=
"language-java">
LocalDRPC drpc = new LocalDRPC();
LocalCluster cluster - new LocalCluster();

cluster.submitTopology("drpc-demo", conf, builder.createLocalTopology(drpc));

System.out.println("Results for 'hello':" + drpc.execute("exclamation", "hello"));

cluster.shutdown();
drpc.shutdown();
</code></pre>

首先创建一个LocalDRPC对象。这个对象在进程中模拟一个DRPC服务器，就如同LocalCluster模拟Storm cluster一样。然后你创建一个LocalCluster在本地模式下运行topology。LinearDRPCTopologyBuilder有独立的方法来创建本地topology和远程topologies。在本地模式下，LocalDRPC对象没有绑定任何端口，所以topology需要知道它通信的对象是哪个。这就是为什么createLocalTopology需要将LocalDRPC对象作为一个输入的参数。

在搞定topology以后，你可以通过localDRPC的execute方法来进行DRPC调用。

### 远程模式DRPC

在实际的集群中使用DRPC也很直接。有3个步骤：

	1. 启动DRPC服务器
    2. 配置DRPC服务器的地址
    3. 向Storm集群提交DRPC的topologies
    
可以通过storm脚本启动DRPC服务，这和启动Nimbus或者UI是一样的：

<pre><code class="language-markup">
bin/storm drpc
</code></pre>

接下来，你需要配置你的Storm集群，让它知道DRPC服务器的地址。这就是DRPCSpout怎样知道去哪里读取函数调用。这可以通过配置storm.yaml文件或者topology的配置来实现。通过storm.yaml来配置的方法，类似以下：

<pre><code class="language-markup">
drpc.servers:
  - "drpc1.foo.com"
  - "drpc2.foo.com"
</code></pre>

最后，像启动其它topology一样启动DRPC。在远程模式运行以上例子，你可以像这样做：

<pre><code class="language-java">
StormSubmitter.submitTopology("exclamation-drpc", conf, builder.createRemoteTopology());
</code></pre>

createRemoteTopology被用来创建适用于Storm集群上的topologies。

### 一个更复杂的例子

刚刚DRPC的例子是用来说明DRPC概念的玩具例子。让我们来看一个更复杂的例子，它真正的需要Storm集群提供的并行能力来运算DRPC的功能。在这个例子中，我们将看到的是如何计算Twitter上一个URL的到达次数。

URL的到达次数是指不同人收到Twitter上一个URL的次数。为了计算它，你需要：

	1. 获得所有推过该URL的人
    2. 获得所有这些人的粉丝
    3. 粉丝去重获得粉丝集合（集合中粉丝没有重复）
    4. 计算粉丝集合中粉丝的数目
    
单单一个到达次数在运算时可能需要涉及到几千个数据库的访问和数以千万记的粉丝记录。这真的真的是一个非常大的计算量。如你看到的哪样，在Storm上实现一个这样的功能是简单的不能再简单了。在一个单机上，到达次数需要几分钟运算；在Storm集群上，再变态的URL计算到达次数也不过几秒钟。

在storm-starter中一个实现到达次数的例子在这里。下面是告诉你如何定义一个到达次数的topology：

<pre><code class="language-java">
LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("reach");
builder.addBolt(new GetTweeters(), 3);
builder.addBolt(new GetFollowers(), 12)
        .shuffleGrouping();
builder.addBolt(new PartialUniquer(), 6)
        .fieldsGrouping(new Fields("id", "follower"));
builder.addBolt(new CountAggregator(), 2)
        .fieldsGrouping(new Fields("id"));
</code></pre>

topology按照以下四个步骤执行：

	1. GetTweeters 获得所有推过该URL的用户。它将一个[id, url]类型的输入流转换成一个[id, tweeter]类型的输出流。每一个url tuple会映射成为多个twtter tuples。
    2. GetFollowers 获得tweeters的粉丝。它将一个[id, tweeter]类型的输入流转换成一个[id, follower]的输出流。在所有的tasks中，有很多情况下粉丝的tuple是重复的，比如一个人可能粉了的多个人中都推了同一个URL。
    3. PartialUniquer 将粉丝按照粉丝id分组。这样做是让同一个粉丝去到同一个task中。所以每一个PartialUniquer的task会收到彼此相互独立的粉丝。一旦PartialUniquer对同一个请求id的粉丝tuple收集完毕，它会讲粉丝集合的粉丝数量发送出去。
    4. 最后，CountAggregator从各个PartialUniquer的task中收到该部分的统计结果，然后将它们加起来完成到达数量的计算。
    
让我们来看一看PartialUniquer bolt：

<pre><code class="language-java">
public class PartialUniquer extends BaseBatchBolt {
    BatchOutputCollector _collector;
    Object _id;
    Set<String> _followers = new HashSet<String>();

    @Override
    public void prepare(Map conf, TopologyContext context, BatchOutputCollector collector, Object id) {
        _collector = collector;
        _id = id;
    }

    @Override
    public void execute(Tuple tuple) {
        _followers.add(tuple.getString(1));
    }

    @Override
    public void finishBatch() {
        _collector.emit(new Values(_id, _followers.size()));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("id", "partial-count"));
    }
}
</code></pre>

PartialUniquer通过继承BaseBachBolt实现了IBatchBolt接口。批量的bolt作为一个整体的单元提供了一个一类API去处理一批tuples。Storm为每一个请求的id建立了一个新的实例，该实例对应着一批bolt，Storm会在适当的时候清理这些实例。

当PartialUniquer在execute方法中接收到一个粉丝tuple，它会将其加到一个内部HashSet做成的请求id的集合中。

批量的bolts提供了finishBatch方法，该方法会在晚上所有针对这批任务的tuples被处理完成以后被调用。在回调中，PartialUniquer发送一个包含了针对该唯一粉丝id的粉丝总数。

在底层，CoordinatedBolt用来检查一个给定的bolt什么时候接收完某请求id的所有tuples。CoordinatedBolt利用直接的流来管理这种协同方式。

剩余的topology应该读者自己就可以看懂了。像你们看到的那样，在计算到达数量的时候，每一步都是并行的，并且定义DRPC的topology是如此的简单。

### 非线性DRPC拓扑

LinearDRPCTopologyBuilder仅能处理”线性“DRPC拓扑，也就是运算可以被表示成一系列步骤（比如统计到达数）。不难想象我们可能需要更加复杂的支持分发和归并的拓扑。就目前，这么做的话你需要直接使用CoordinatedBolt。如果你需要使用非线性DRPC拓扑，别忘了去邮件列表里说一下你的用例，以便将DRPC拓扑建立成更一般的抽象。

### LinearDRPCTopologyBuilder是如何工作的

* DRPCSpout发送[args, return-info]。return-info包含了DRPC服务器的主机地址和端口号，以及DRPC服务器为其生成的id。
* topology的构成包含了：
	* DRPCSoupt
    * PrepareRequest（生成request id并且创建一个返回信息的流和一个返回参数的流）
    * CoordinatedBolt 包装和直接分组
    * JoinReulst（使用返回的信息连接结果）
    * ReturnResult（连接DRPC服务器并返回结果）
* LinearDRPCTopologyBuilder是一个创建在Storm的原语顶层高层次抽象的很好的例子。

### 更进一步

* KeyedFairBolt在同一时间处理多个请求。
* 如何直接使用CoordinatedBolt