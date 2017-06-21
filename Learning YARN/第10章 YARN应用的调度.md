在YARN中，调度意味着对集群中运行的应用的集群资源分配。调度器是ResourceManager服务核心的组件之一并且它负责基于container资源请求的资源分配。随机存取存储器(RAM)和处理器是每个container执行的所必须的两个决定性的资源。随着container并发量的增加，对于每个container来说合理的需求和集群中资源的管理将会成为应用执行成功与否的决定性因素。  

在本章，你将会学习到关于YARN中的调度机制和YARN中的调度器如何配置。你同样也会学习到关于YARN中的队列和不一样的可用的调度算法。  

我们将会涉及到下面的主题：  
* 介绍YARN中的调度
* 介绍队列和队列类型
* 容量调度器
* 公平调度器  

### 介绍YARN中的调度  
YARN中的调度器是为了管理集群的资源而编写的高效的算法。YARN中的ResourceManager服务拥有一个可插拔的纯粹的调度器组件，也就是说，它不负责跟踪和监控集群中应用的执行。它仅仅负责跟运行的应用分配资源。  

你可能想问，什么是资源分配？为什么它很重要？好吧，让我们设想一个简单的场景。假设一个机构拥有一个100台节点的Hadoop-YARN集群并且有N个团队(比如A、B、... N)使用着这个集群。每个团队有10到15个成员并且每个成员都可以提交在集群上提交应用。那么，对象集群管理员来说，想要提供一个共享的多租户和高效利用率的集群，集群资源分配就扮演了一个非常重要的角色。集群的资源可以基于一个可插拔的策略对不同的团队或者团队成员进行划分。当在定义集群如何共享的参数时，调度器应该足够灵活以达到支持下面的场景：  
![image](/Images/YARN/YARN-cluster-resource-allocation-scenario.png)  
* 假设团队A工作的客户端要求所有的任务需要40%的集群资源。那么集群管理员就需要确保团队A工作的客户端所需要的集群资源是合适的。
* 一个团队可能有多个子团队，那么集群中的资源需要在那些子团队之间共享。意思是，如果集群中20%的资源分配给了团队N，那么这20%的集群资源需要在不同的子团队中进行共享，分散。  

Hadoop-YARN集群有下面列举出来的两个预先定义好的调度器：  
* 容量调度器(Capacity scheduler)：这是一个按照集群资源总量百分比进行分配的调度器
* 公平调度器(Fair scheduler)：这是一个基于内存和处理器需求进行分配的调度器  

想要配置和使用调度器，管理员首先需要定义队列。在我们深入讨论这么调度器的细节之前，我们首先会学习学习一些队列的概念以及YARN中定义的不同类型的队列。  

### 初识队列  
对于提交到YARN集群上的应用来说，队列就相当于是数据结构或者说是占位符。队列是提交到YARN集群上的应用的逻辑分组。应用总是会被提交到队列中。调度器依据正确的参数对应用进行出列然后分配资源和启动应用并执行。  

队列基本的结构使用接口org.apache.hadoop.yarn.server.resourcemanager.scheduler.Queue进行定义，如下图所示：  
![image](/Images/YARN/yarn-queue-structure.png)  
一个队列对象包含下面一些信息：  
* **队列名称**：这是分配给队列的名称。假如是分层队列，那么队列完整的路径名就是带有父队列名的名称。稍后我们将会详细讨论分层队列。
* **队列信息**：YARN定义了一个抽象的类QueueInfo用来存储与队列有关的信息。它被定义在org.apache.hadoop.yarn.api.records包中。QueueInfo包含下面一些信息：  
    * 队列名称
    * 配置后的队列容量
    * 队列的最大容量
    * 队列的当前容量
    * 子队列
    * 正在运行的应用列表
    * 队列的QueueState  

队列的容量是一个浮点值，表示的是给一个独立的队列分配的内存大小。一个队列也可能包含一系列子队列(分层队列)。RUNNING和STOPPED是YARN中定义的两种队列状态。当一个队列处于STOPPED状态时，它将不会接受新的应用的提交。  
* 队列度量：QueueMetrics类被定义在org.apache.hadoop.yarn.server.esourcemanager.scheduler包中。它包含了对一个特定队列中的应用、container和用户的统计。
* 队列访问控制列表：队列访问控制列表是一个定义了用户和组对一个特定队列权限的机制。意思是你可以定义一个被允许提交应用到队列上的用户或者是组的列表。想要了解更多细节，你可以参考第11章 启用YARN安全策略。  

你可以通过ResourceManager Web接口或者ResourceManager中关于调度器的REST API查看队列的列表和它们的属性。更多关于ResourceManager REST API的信息，你可以参考Hadoop的文档 http://hadoop.apache.org/docs/r2.6.0/hadoopyarn/hadoop-yarn-site/ResourceManagerRest.html#Cluster_Scheduler_API 。  

### 队列类型  
本章之前提到了YARN定义了两种调度器(容量调度器和公平调度器)。这些调度器使用它们自己的队列接口实现。下图是一个类图，表示了YARN中定义的不同队列：  
![image](/Images/YARN/yarn-queue-type.png)  

#### 容量调度器队列(CSQueue)  
CSQueue是一个继承自Queue接口的接口。它被定义在org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity包中。CSQueue接口代表了CapacityScheduler调度器中的分层队列树中的一个节点的队列结构。  

下面两个类实现了CSQueue接口：  
* ParentQueue
* LeafQueue  

与CapacityScheduler队列相关联的属性如下：  
* yarn.scheduler.capacity.<queue-path>.capacity：一个浮点值，以百分比(%)的形式给一个队列指定容量大小。对于每一级队列来说(分层队列)，所有队列的容量之和必须等于100。为了实现应用弹性和集群的高效性，应用可能会消耗比定义的容量更多的资源。
* yarn.scheduler.capacity.<queue-path>.maximum-capacity：一个浮点值，以百分比的形式指定队列的最大容量。这个属性是用来限制队列能够使用的最大容量。默认情况下，该值的初始值为-1，这意味着没有限制，就是说集群资源可用，那么队列可以使用100%的资源。
* yarn.scheduler.capacity.<queue-path>.minimum-user-limit-percent：一个整型值，以百分比的形式指定每个用户在队列中最小的容量限制。当有多个用户在使用同一个队列时，这个属性保证了用户在共享的环境下可以获得最小百分比的集群资源。


#### 公平调度器队列(FSQueue)  
