本章将会介绍Spark全部的设计原理和它在大数据生态系统中的位置。Spark常常被认为是Apache MapReduce的替代者，以为Spark同样可以和Hadoop一起进行分布式数据处理。随着我们本章的讨论，你会发现Spark的设计原理与那些在MapReduce中的完全不一样。与MapReduce不同的是，Spark不是必须要与Hadoop一起运行——尽管它经常那这样做。Spark从其他已经存在的计算框架借鉴了部分API，设计和支持的格式，特别是DryadLINQ。然而，Spark的内核与大多数传统的系统不同，尤其是它如何处理失败。Spark在内存计算中的惰性计算能力使得它非常的独一无二。Spark的创建者相信它能够成为一个首屈一指的用于快速分布式数据处理的高级别编程语言。  

想要更深入的学习Spark，弄明白Spark的设计原理是非常重要的，并且还应该粗略的了解Spark程序是如何执行的。本章中，我们将会提供一个Spark并行计算模型的概览和对Spark调度引擎和执行引擎一个深入的讲解。本章中的涉及到概念将会贯穿全文。在未来，对于其他Spark用户提到的或者在Spark文档中遇到的，我们希望这些讲解将会提供给你一个更精确的理解。  

### Spark如何适应大数据生态系统  
Apache Spark是一个开源的  


#### Spark的组件  
Spark提供了一个高级别的查询语言来处理数据。Spark Core，Spark生态系统中主要的数据处理框架，拥有Scala、Java、Python和R API。Spark是围绕着一种叫做RDD(Resilient Distributed Datasets 弹性分布式数据集)的数据抽象进行构建的。RDD是惰性计算、静态类型和分布式集合的一种表现。RDD拥有有一些预先定义的"粗粒度的"转换(transformation，是一种被应用于全部数据集上的函数)，比如map，join和reduce等用来操作分布式数据集，就像通过I/O在分布式数据存储系统和Spark JVM之间读取或者写入数据一样。  

提示：尽管Spark也支持R语言，但是目前R语言的RDD接口是不可用的。我们将会在第7章中详细地介绍使用Java、Python、R和其他语言的技巧。  

除了Spark Core，Spark生态系统中还包含一些其他内部的组件，像Spark SQL、Spark MLlib、Spark ML、Spark Streaming和Spark GraphX，每一个都提供了特定场景下的数据处理函数。  


### Spark并行计算模型：RDD  
Spark允许用户  


### Spark任务调度  
一个Spark应用包含一个高级别的编写Spark逻辑的driver进程，和一个系列可以跨节点分散开来的executor进程。Spark程序自己运行在driver所在的节点上并且发送一些指令到executor。一个Spark集群可以并发的运行多个Spark应用。Spark应用通过集群管理器(cluster manager)进行调度，相当于一个SparkContext。Spark应用可以依次运行多个并发的job。job相当于给定的应用中的RDD上执行action操作。本节中，我们将会描述Spark应用和它是如何运行Spark job：计算RDD转换(transformation)的过程。  

#### 应用间资源分配  
Spark提供了两种应用间资源分配方式：静态资源分配和动态资源分配。静态资源分配中，每个应用都会被分配一个集群中限定的最大资源容量并且在应用运行期间都会占用那些资源(只要SparkContext一直在运行)。在静态分配范畴内，有多种资源分配可用，取决于集群。更多相关信息可以参考Spark文档  

直到1.2，Spark提供了动态资源分配的选项，它扩展了静态分配的功能。在动态分配中，Spark应用根据需要可以增加和删除executor，基于一组对资源预估的启发式条件。我们将会在"分配集群资源和动态资源"中讨论资源分配。  

####Spark应用  
一个Spark应用相当于在driver程序中的SparkContext定义的一组job。Spark应用开始于SparkContext的启动。当Spark启动之后，一个driver和一系列executor会在集群中的worker节点上启动。每个executor有它自己的Java虚拟机(JVM)，一个executor不可能跨多个节点尽管一个节点可能包含多个executor。  

SparkContext决定了每个executor可以分配多少资源。当一个Spark job被启动之后，，每个executor都会有槽位(slot)用来运行需要计算RDD的task。这样，我们可以认为一个SparkContext就是作为运行Spark job的一组配置参数。这些参数在用来创建SparkContext对象的SparkConf对象中被暴露。我们将会在Appendix A中讨论如何使用参数。

### Spark任务剖析  
