---
layout: post
title: Storm 介绍
---

今天我们组的两个developer回美国了，临别心里着实有些恋恋不舍，这几天他们为我们细心准备的培训让我们增加了宝贵的经验。会议中再三提到的，自然免不了这个实时处理界的大明星 -- Storm。

以下文档翻译自[Storm官方教学文档](http://storm.incubator.apache.org/documentation/Tutorial.html)。其实这个文档有的部分已经有同学翻译过了，我在这里再次翻译更主要是为了自我学习，如果对大家有所帮助，那就更好了。

好了，那我就开始了。

在这个教学中，你可以了解怎么创建Storm的拓扑并将其部署到Storm的集群里。Java是其中主要用到的语言，但是也有个别例子是用Python写的，记住，毕竟Storm拥有对多种语言兼容的能力。

### 初步准备

这个教学用到的例子在这里：[storm-starter](http://github.com/nathanmarz/storm-starter)。你可以克隆这个项目，然后跟着做做这个例子。这里有两篇文章“[准备开发环境](http://storm.incubator.apache.org/documentation/Setting-up-development-environment.html)“和“[创建新的Storm工程](http://storm.incubator.apache.org/documentation/Creating-a-new-Storm-project.html)”可以用来参考如何设置你的机器。

### Storm集群的各组件们

一个Storm集群表面上看起来很像是一个Hadoop集群。不过在Hadoop上你运行的是"MapReduce jobs"，而在Storm上你运行的是”topologies“。”Jobs“和”topologies“其实是非常不同的 -- 其中一个最大的不同就是MapReduce的任务最终会运行结束，但是topology的进程永远不会结束（除非你把它kill掉）。

在Storm的集群里有两种类型的节点：master节点和worker节点。其中master节点运行着一个守护进程，叫做”Nimbus“（翻译过来意思是：云雨），这个东西和Hadoop里面的”JobTracker“类似。Nimbus负责在集群中分发代码，向机器分配任务以及监控错误。

每一个worker节点运行着一个守护进程，叫做”Supervisor“。这个supervisor监听着分配到这台机器上的任务，根据Nimbus分配的不同任务来启动或者停止一些worker上的进程。每一个worker进程执行一个topology的子集；一个运行的toplogy有很多机器上的很多worker的进程所组成。

![](http://storm.incubator.apache.org/documentation/images/storm-cluster.png)

Nimbus和Supervisors之间所有的协调工作，都交给我们的”动物管理员集群“（[Zookeeper](http://zookeeper.apache.org/) cluster）负责。此外，Nimbus和Supervisor的守护进程都是快速失效和无状态的；所有的状态都被保存在Zookeeper或者本地磁盘上。这意味着你可以使用Kill -9去杀死Nimbus或者Supervisors，然后它们会被重新起起来像什么事也没有发生一样。这种设计可以让Storm集群具有难以置信的稳定性。

### Topologies（拓扑）

如果你要在Storm上做实时的计算，那么你需要创建一个叫做”topologies“的东西。一个topology是一个计算的图。其上每一个节点包含了一个逻辑进程，以及一个用来说明数据在节点间如何传播的链接。

运行一个topology很简单。首先，你需要把你所有的代码打包，然后将所有的依赖放到一个jar中。然后，你运行一个像下面这样的命令：

<pre><code class="language-markup">
storm jar all-my-code.jar backtype.storm.MyTopology arg1 arg2
</code></pre>

这就运行了backtype.storm.MyTopology这个类，使用的参数是arg1和arg2.这个类的主要功能是定义toplogy然后将它提交给Nimbus。storm jar用来连接Nimbus然后上传这个jar文件。

因为topology基于由Thrift结构定义的，而且Nimbus也是一个Thrift service，所以你可以使用多种编程语言创建和提交topologies。以上例子是基于JVM的语言最简单的例子。请参考”[在生产环境的集群上运行topologies](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)“这篇文章来获得更多的关于启动和停止topologies的信息。


### Streams （流）

Storm抽象的核心叫做”stream“（流）。一个流是一个源源不断的tuples序列。Storm提供一种原语，可以以可靠的方式，分布式地将一个流转化为另一个新的流。比如，你可以将一个由twitter的文章组成的流转化成一个关于某个话题流行的趋势的流。

Storm提供的用来转化流的最基本的原语，叫做”spouts“和”bolts“。Spouts和bolts都有一些接口，可以让你用来实现你应用需要的特殊的逻辑。

spout就是一个流的源。比如，一个spout可能是从[Kestrel](http://github.com/nathanmarz/storm-kestrel)的队列里读取一系列的tuples，然后把它们发出去。或者，一个spout可能是连接在Twitter的API上，然后发送tweeter文章的流。

bolt作为消费者，消费着所有输入过来的流，经过一些处理，然后可能会发出新的流。比较复杂的流的转换，比如从一个tweeter文章的流转化成一个关于某话题流行趋势的流，需要很多的步骤，那就需要好几个bolts。Bolts可以做很多事情，比如运行函数，过滤tuples，做流的聚合，做流的join，还有和数据库对话，等等。

由spouts和bolts组成的网络，形成了”topology“这一顶端的抽象，你可以把它丢给Storm的集群去运行。topology就是一个表示流变换的图，其中每一个节点是一个spout或者一个bolt。图中的每个边，表示bolt订阅了某个流。当一个spout或者bolt往一个流上发射一个tuple的时候，它将该tuple发布到每一个订阅了该流的bolt上。

![](http://storm.incubator.apache.org/documentation/images/topology.png)

topology中在每个节点之间的链接表示tuples是如何传递的。比如，如果在Spout A和Bolt B之间有一条链接，Spout A和Bolt C有一条链接，然后Bolt B和Bolt C也有一条链接的话，那么每次Spout A如果发布一个tuple，它会同事将这个tuple发给Bolt B和Bolt C。所有的Bolt B的输出，也将交给Bolt C。

每一个Storm toplogy中的节点都是并行运行的。在topology中，你可以对每一个节点指定不同的并行数，然后Storm会在集群里建立这个数目的进程来执行任务。

topology是永远运行的，除非你kill它。Storm会自动的重新分配失败的任务。此外，Storm保证所有的数据都不会丢失，就算机器宕机并且丢失了所有的消息也没关系。

### 数据模型

Storm用tuples作为它的数据模型。tuple是个被命名了的列表，包含一系列的值，tuple上的field可以是任意类型的对象。Storm原生支持所有原始类型，字符串以及比图数组作为其field的值。如要使用对象或是其他类型，你只需要为该类型实现[serializer](http://storm.incubator.apache.org/documentation/Serialization.html)（序列化）。

topology中的每一个节点都必须声明它要发出的tuples的fields。比如，如果有一个bolt声明了它发送2个tuples其各自的fields是”double“和”triple“：

<pre><code class="language-java">
public class DoubleAndTripleBolt extends BaseRichBolt {
    private OutputCollectorBase _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple input) {
        int val = input.getInteger(0);        
        _collector.emit(input, new Values(val*2, val*3));
        _collector.ack(input);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("double", "triple"));
    }    
}
</code></pre>

其中declareOutputFields声明了该组件输出的fields是["double", "triple"]。关于bolts其余的知识点，会在后面的章节继续介绍。


### 一个简单的topology

让我们用一个简单的toplogy来说一说更多的概念，然后看看代码是怎么写的。让我们看看storm-starter中那个ExclamationTopology是如何定义的：

<pre><code class="language-java">
TopologyBuilder builder = new TopologyBuilder();        
builder.setSpout("words", new TestWordSpout(), 10);        
builder.setBolt("exclaim1", new ExclamationBolt(), 3)
        .shuffleGrouping("words");
builder.setBolt("exclaim2", new ExclamationBolt(), 2)
        .shuffleGrouping("exclaim1");
</code></pre>

这个topology包含了一个spout和两个bolts。spout发送words，然后每个bolt在每个其上的输入后面追加一个"!!!"。这些节点被排成一条线：spout发布到第一个bolt，然后再由第一个bolt发给第二个bolt。如果spout发的tuples是["bob"]和["john"]，那么第二个bolt就会发出这样的words：["bob!!!!!!"]和["john!!!!!!"]。

这个代码定义了节点使用setSpout和setBolt两个方法。这些方法那一个用户自定义的id作为输入，对象包含了处理的流程，和你需要该节点并行的总数。在这个例子里，spout给赋予了一个id叫做”words“，然后bolts被赋予了id分别是”exclaim1“和”exclaim2“。

上述对象包含了逻辑流程，实现了spouts的[IRichSpout](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichSpout.html)接口和bolts的[IRichBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichBolt.html)接口。

最后一个参数，你需要该节点有多少并行数，是一个可选的参数。它表示在集群上执行任务需要多少个进程。如果你不填它，Storm默认的在该节点上使用一个进程。

`setBolt`返回一个[InputDeclarer](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/InputDeclarer.html)对象, 该对象用来定义Bolt的输入。在这里，组件"exclaim1"定义了它需要读取所有的由组件”words“以shuffle grouping方式发送的tuples，组件”exclaim2“定义了它需要读取所有的由组件”exclaim1“以shuffle grouping方式发送的tuples。”shuffle grouping“的意思是tuples从输入到发送给bolt的任务，是以随机的方式进行的。这里有很多种方式用来表示组件间数据的分组。这会在一会儿的章节中介绍。

如果你想要组件”exclaim2“去读取所有由组件”words“和”exclaim1“发送的tuples，你可以将”exclaim2“写成下面这个样子：

<pre><code class="language-java">
builder.setBolt("exclaim2", new ExclamationBolt(), 5)
            .shuffleGrouping("words")
            .shuffleGrouping("exclaim1");
</code></pre>

像你看到的这样，对bolt的输入的定义可以链上多个源。

让我们继续深入到topology中spouts和bolts的实现上。Spouts负责发送新的消息到topology。在这个topology中`TestWordSpout`每过100毫秒就从列表["nathan","mike","jackson","golda","bertels"]中随机的选取一个单词作为一个tuple。在TestWordSpout中，`nextTuple()`的实现是像下面这个样子的：

<pre><code class="language-java">
public void nextTuple() {
    Utils.sleep(100);
    final String[] words = new String[] {"nathan", "mike", "jackson", "golda", "bertels"};
    final Random rand = new Random();
    final String word = words[rand.nextInt(words.length)];
    _collector.emit(new Values(word));
}
</code></pre>

正如你看到的这样，实现是如此简单而直接。

`ExclamationBolt`会在它的输入后面添加一个"!!!"字符串。让我们看看`ExclamationBolt`完整的实现：

<pre><code class="language-java">
public static class ExclamationBolt implements IRichBolt {
    OutputCollector _collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    public void cleanup() {
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }

    public Map getComponentConfiguration() {
        return null;
    }
}
</code></pre>

其中`prepare`方法为bolt提供了一个`OutputCollector`，用来从bolt中发送tuples。Tuples可以在任何时候从bolt中发送 -- 在`prepare`，`execute`或`clearnup`方法中，或者甚至异步地在其它进程中都可以发送。这里的`prepare`的实现只是简单地将`OutputCollector`作为实例保存起来，供之后在`execute`方法中使用。

`execute`方法从其中一个bolt的输入中接受tuple。`ExclamationBolt`从tuple中抓取第一个field，在其后面追加一个字符串"!!!"，然后发送一个新的tuple。如果你需要实现一个订阅多个输入源的bolt，你可以通过使用`Tuple#getSourceComponent`方法来定位[Tuple](http://storm.incubator.apache.org/apidocs/backtype/storm/tuple/Tuple.html)是从哪个组件过来的。

在`execute`方法中还有一些其它的东西需要注意，好比那个输入的tuple是以第一个参数传给`emit`的，然后在最后一行被acked。这些其实是Storm的可靠性API在保证数据不丢失的具体体现，这会在之后的教学中阐述。

`cleanup`方法会在一个Bolt被关闭的时候被调用，它可以用来清理所有被打开的资源。在集群上该方法不能保证会被调用：比如，如果一个正在运行任务的机器突然挂掉了，那没有任何方法去调用这个方法。`cleanup`的意义在于当你在[本地模式](http://storm.incubator.apache.org/documentation/Local-mode.html)下运行topologies时（在进程中模拟Strom集群），如果你需要运行和杀死一些topologies，可以不用担心资源泄露。

`declareOutputFields`方法声明了`ExclamationBolt`发送的tuples的field为”word“。

`getComponentConfiguration`方法让你可以配置组件不同的运行方式。更多该方面的类容，可以参考这里：[配置](http://storm.incubator.apache.org/documentation/Configuration.html)。

像`cleanup`和`getComponentConfiguration`这样的方法，一般不需要在bolt中实现。你可以用过使用基类提供的一些合适的默认的实现来简洁的定义bolts。`ExclamationBolt`可以通过扩展`BaseRichBolt`来简洁的实现，像这样：

<pre><code class="language-java">
public static class ExclamationBolt extends BaseRichBolt {
    OutputCollector _collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }    
}
</code></pre>

### 在本地模式运行ExclamationTopology

让我们看看如何在本地模式下运行`ExclamationTopology`，然后看看它是如何工作的.

Storm有两个操作模式：本地模式和分布式模式。在本地模式中，Storm通过完全执行在一个进程中，然后通过多线程模拟多个worker节点。本地模式对于测试和开发topologies非常有用。当你在storm-starter中运行toplogies时，它们会运行在本地模式，你可以看到每个组件发布的消息。你可以在[本地模式](http://storm.incubator.apache.org/documentation/Local-mode.html)这个页面了解更多关于在本地模式下运行topologies的类容。

在分布式模式下，Storm以一个集群的方式运行。当你往一个master提交一个topology时，你同时提交了所有运行topology需要的代码。这个master会帮你部署代码和分配workers来运行你的topology。如果workers挂掉，那么master会在其它位置重新为你分配。你可以阅读“[在生产集群上运行topologies](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)”来了解更多关于在集群上运行topologies的知识。

这里的代码在本地模式运行`ExclamationTopology`：

<pre><code class="language-java">
Config conf = new Config();
conf.setDebug(true);
conf.setNumWorkers(2);

LocalCluster cluster = new LocalCluster();
cluster.submitTopology("test", conf, builder.createTopology());
Utils.sleep(10000);
cluster.killTopology("test");
cluster.shutdown();
</code></pre>

首先，代码通过新建一个`LocalCluster`对象来定义了一个运行的集群。向虚拟集群提交topologies和向分布式集群提交topologies是一样的。通过调用submitTopology将一个topology提交给LocalCluster，该方法需要一下3个参数：需要运行的topology的名字，topology的配置信息，以及该topology自身。

topology的名字用来表示topology，以便之后能kill它。一个topology会永远运行下去直到你kill它。

配置信息用来切换topology的不同运行方式。一下是两个非常常见的配置选项：

* 1. TOPOLOGY_WORKERS（通过setNumWorkers设置）表示你需要在集群上有多少个进程来运行topology。每个topology中的组件可以又多个线程执行。对应于给定组件的线程的数量可以通过`setBolt`和`setSpout`方法来配置。这些线程存在于worker的进程中。每一个worker的进程包含多个对应于多个组件的线程。例如，你可能有300个线程囊括了你所有的组件，并在配置文件中定义了50个worker进程。每一个worker进程将执行6个线程，这线程可以对应在不同组件上。你可以通过调节每个组件的并行情况以及worker进程上运行的线程的数量来改变Storm topologies的性能。

* 2. TOPOLOGY_DEBUG（通过setDebug设置）当设置为true的时候，意味着Storm会将组件的每一个发送的消息都记录到日志里。这在本地模式下测试topologies时很有用，但是当在集群运行topologies时，估计你想把它关掉。

topology有很多的配置选项，详见：[the JavaDoc for Config](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html)。

关于如何配置开发环境，以便在本地模式（如Eclipse）中运行topologies，请参阅“[创建新的Storm工程](http://storm.incubator.apache.org/documentation/Creating-a-new-Storm-project.html)”文档。

### 流群

流群告诉topology如何在两个组件间发送tuples。还记得，spouts和bolts以多tasks（任务）并行的方式运行在集群中。如果你想从task的这个层面来查看一个topology是如何运行的，那么它看起来有点像这个样子：

![](http://storm.incubator.apache.org/documentation/images/topology-tasks.png)

当一个Bolt A的task向Bolt B发送一个tuple，这个tuple应该发到哪一个task上呢？

“流群”通过高速Storm如何在task的集合之间发送tuples来回到这个问题。在我们深入讲解不同类型的流群之前，让我们来看看[storm-starter](http://github.com/nathanmarz/storm-starter)中的另一个topology。该[WordCountTopology](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/WordCountTopology.java)从一个spout读取句子，然后从WordCountBolt中流出看见同一单词的次数。

<pre><code class="language-java">
TopologyBuilder builder = new TopologyBuilder();

builder.setSpout("sentences", new RandomSentenceSpout(), 5);
builder.setBolt("split", new SplitSentence(), 8)
	.shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 12)
	.fieldsGrouping("split", new Fields("word"));
</code></pre>

`SplitSentence`每当接收到一个句子的时候，发送
一个包含该句子中每一个单词的tuple，`WordCount`在内存里保存了一个（单词，个数）的map。每当`WordCount`收到一个单词，它就会更新其状态然后发出一个新的单词数量。

流群有几种不同的类型：

最简单的一类流群叫做“shuffle grouping”，该类型会将tuple随机地发往一个task。shuffle grouping被在WordCountTopology中，从RandomSetenceSpout向SplitSentence的bolt发送tuples。它可以均匀地将处理tuples的任务分发给所有SplitSentence bolt上的task.

更有意思的一类流群叫做“fields grouping”（字段分组）。fields grouping被用在splitSentence bolt和WordCount bolt之间。它严格的保证了在WordCount bolt上同一个单词总是去到同一个task上。否则，试想如果超过一个的task得到同一个单词，，那么由于他们都得到该单词完整的信息，所以发出的单词数量都是不正确的。

fields grouping允许你通过流的fields的子集来建群。这使得具有相同fields子集的值发送到同一个task上。例中WrodCount使用一个叫做“word“的field订阅了SplitSentence的fields gourping类型的输出流，使得同样的单词总是去到同一个task中，bolt的输出结果就是正确的了。

fields grouping是实现流的连接和聚合以及其他很多用例的基础。在系统底层，fields grouping是使用MOD散列实现的。

还有一些其他类型的流群,请参考[Concepts（概念）](http://storm.incubator.apache.org/documentation/Concepts.html)。

### 使用其它语言定义Bolts

Bolts可以用任意语言定义。以其它语言定义的bolts被当做子进程执行，Storm通过stdin/stdout（标准输入输出）以JSON格式的消息与各个子进程交互。这种通信协议只需要一个100行左右的适配器包，并且对于Ruby, Python的Fancy，Storm不需要该包。

以下是WordCountTopology例子中SplitSentence bolt的定义：

<pre><code class="language-java">
public static class SplitSentence extends ShellBolt implements IRichBolt {
    public SplitSentence() {
        super("python", "splitsentence.py");
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}
</code></pre>

SplitSentence重写了ShellBolt，并且声明它使用splitsentence.py为参数，运行python。这是splitsentence.py的实现：

<pre><code class="language-python">
import storm

class SplitSentenceBolt(storm.BasicBolt):
    def process(self, tup):
        words = tup.values[0].split(" ")
        for word in words:
          storm.emit([word])

SplitSentenceBolt().run()
</code></pre>

更多关于如何用其它语言写spouts和bolts的内容，以及学习如何使用其他语言创建topologies（比如完全抛弃Java虚拟机），请参阅：[在Storm上使用非JVM语言](http://storm.incubator.apache.org/documentation/Using-non-JVM-languages-with-Storm.html)。

### 消息处理的保障机制

在之前的教学中，我们没有提及tuples如何发送这个话题。这里的内容是Storm可靠的API的一部分：Storm似乎如何保证每一条来自spout的message能够被全部处理。[消息处理的保障机制](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)介绍了这个内容以及如何使用Storm带来的可靠性。

### Transactional Topologies（事务性拓扑结构）

Storm保证每条消息至少在topology中处理一次。一个常被问到的问题是：“怎么样做诸如在Storm顶层计数这样的事情？会不会过多计数？“ Strom有一个称为transactionaltopologies的功能，可以在大多数计算中达到”一个消息处理且处理一次“的要求。更多关于Transactional Topologies的内容，请查阅[这里](http://storm.incubator.apache.org/documentation/Transactional-topologies.html)。

###分布式RPC

这篇教程介绍了如何在Storm上做基本的流处理。使用Storm的原语，你还可以做更多的事情。其中一个最有意思的Storm应用是分布式RPC，在其上你飞快的并行计算大量的函数。更多关于分布式RPC的内容请参阅[这里](http://storm.incubator.apache.org/documentation/Distributed-RPC.html)。

### 结语

这篇教程给出了Storm开发，测试和部署的概述。剩下的文档在如何使用Storm方面做更深入的介绍。

（完）
