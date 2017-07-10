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
* yarn.scheduler.capacity.<queue-path>.minimum-user-limit-percent：一个整型值，以百分比的形式指定每个用户在队列中最小的容量限制。当有多个用户在使用同一个队列时，这个属性保证了用户在共享的环境下可以获得最小百分比的集群资源。该属性的默认值是100，意思就是对用户没有强制性限制。假设，如果管理员将该属性值设置为20，并且只有三个用户提交应用到YARN集群上，那么分配给用户的最大的资源就是33%。然而，如果5个甚至更多的用户向YARN集群上提交应用，那么每个用户将会最少分配20%的资源，并且如果集群中没有可用资源，那么应用将会进入等待状态。
* yarn.scheduler.capacity.<queue-path>.user-limit-factor：一个浮点值，指定了一个额外的值为了允许用户获得更多的集群资源。比如，如果该值设置为1.5，并且配置的队列容量是40%，那么用户可以在这个队列中获得(1.5*40%)的集群资源。默认值是1，保证用户仅仅可以消耗配置的集群容量。
* yarn.scheduler.capacity.<queue-path>.maximum-applications：一个整型值，指定一个队列最大可以接受的应用数量。一个已经被容纳的应用指的是应用处于正在运行的状态或者应用处于等待状态(应用正在等待资源的分配但是已经分配了队列)。当达到这个限制时，那么新的应用提交将会被拒绝。
* yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent：一个浮点值，指定ApplicationMaster服务可以使用的最大资源容量的百分比。
* yarn.scheduler.capacity.<queue-path>.state：该属性是用来设置队列的状态。一个队列要么是RUNNING状态要么是STOPPED状态。如果一个队列处于STOPPED状态，那么一个新应用的向该队列或者其子队列的提交请求将会被拒绝。  

你可以参考CSQueue接口的Java代码 http://grepcode.com/file/repo1.maven.org/maven2/org.apache.hadoop/hadoop-yarn-serverresourcemanager/2.5.1/org/apache/hadoop/yarn/server/resourcemanager/scheduler/capacity/CSQueue.java?av=h#CSQueue 。  

#### 公平调度器队列(FSQueue)  
FSQueue是定义在org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair包中的一个抽象类。它实现了Queue接口，表示的是在总的集群内存基础上，基于公平共享容量分配的队列的资源计算。  

类似于CSQueue，FSQueue表示的是对于FairScheduler调度器的一个节点的队列结构。  

下面的两个类继承了FSQueue：  
* FSParentQueue
* FSLeafQueue  

类似于CSQueue，每个FSQueue对象都拥有下面的元素：  
* minResources和maxResources：分配资源给一个队列的最小和最大值。该值以X mb和Y vcores的形式设置。
* maxRunningApps：一个整型值，指定提交到一个队列中运行的或者等待的应用的最大数量。
* maxAMShare：一个浮点值，指定ApplicationMaster服务使用资源的最大百分比。默认值是-1.0f，意思是不启用ApplicationMaster资源使用量检测。
* weight：类似于CSQueue中的user-limit-factor，FSQueue有一个weight属性，为特定的资源指定一个额外的值，为了允许用户获得比其他队列获得更多的资源。
* schedulingPolicy：FSQueue在队列内部使用一个调度策略分配资源。YARN定义三种调度策略，如下：对于一个队列来说默认的策略是fair。下一节我们将会详细的讨论调度策略的概念。
    * 先进先出策略(FIFO)
    * 公平共享策略
    * 主导资源公平策略(Dominant Resource Fairness policy简称DRF)
* aclSubmitApps和aclAdministerApps：定义了一个可以提交应用和杀死应用的用户或者组的列表。
*  minSharePreemptionTimeout：在尝试从其他队列获得container资源使用之前等待的秒数。  

你可以参考FSQueue接口的Java代码http://grepcode.com/file/repo1.maven.org/maven2/org.apache.hadoop/hadoop-yarn-server-resourcemanager/2.5.1/org/apache/hadoop/yarn/server/resourcemanager/scheduler/fair/FSQueue.java?av=h#FSQueue 。  

### 初识调度器  
调度器的职责是给运行的应用的不同任务提供资源。它仅仅负责任务的调度，并不关心任务状态的跟踪和任务的监控。调度确保在内存、core、磁盘和网络的方面满足应用的资源需求。更细粒度的说，调度器会满足每个独有的应用对container的需求。Hadoop默认的调度器使用一个单一队列(root队列)去接受和调度应用。它意味着所以应用都会被提交到root队列。  

你可以通过ResourceManager Web UI地址 (http://&lt;ResourceManagerIP&gt;:8088/cluster/scheduler) 查看配置的调度器的详细信息。正如下面截图所示：  
![image](/Images/YARN/yarn-web-ui-scheduler.png)  

YARN为实现可插拔的调度器提供了接口。下面两个是在Hadoop受欢迎的调度器：  
* 公平调度器
* 容量调度器  

#### 公平调度器  
公平调度器的开发是为了实现运行在Hadoop YARN集群上的所有应用公平的资源共享。内存和CPU是当前在应用之间公平分配的资源。当单独一个应用被提交到集群上时，所有的资源对它来说都是可用的。当另一个应用被提交时，集群中的一部分资源会按照需要分配给第二个应用。不像Hadoop默认的调度器，公平调度器允许并行的一起执行短期的，计算密集型的和冗长的应用，那么所有的应用都会在一个公平的时间里执行。  

公平调度器接受应用并且将其放入一个队列中。默认情况下， 所有的应用都会被放入到一个叫做default的单独队列中。根据用户提交的应用，应用可以被调度在不同的队列中。对于每个队列来说，共享队列中的资源支配着调度策略。  

被提交的应用可以带有优先级，并且当应用被调度到集群上的时候，队列会关注应用的优先级。当提交应用的时候，优先级可以被设置成一个整型值。想要了解更多有关优先级的内容，你可以参考org.apache.hadoop.yarn.api.records.Priority类。在当前的Hadoop版本中(2.6.0)，每一个应用的优先级都不会被传入到YARN的调度器，并且默认所有的应用的优先级都是1。你可以参考 FSAppAttempt类中的getPriority方法。  

一个公平调度器同样也会保证队列中最小的共享资源，意思是用户或者是组提交应用到到队列中时，将会至少获得定义在队列中的最小的共享资源。当一部分资源没有被队列所使用，那么资源将会分给另一个应用。公平调度器同样也会根据每个用户或者每个队列的基数来限制同时执行应用的数量，尽管它可能会接受来自用户提交的任意数量的应用，并且对应用进行排队。  

在本节中，我们将会讨论关于公平调度器的特点和概念。  

##### 分层队列  
公平调度器提供了对分层队列的支持。所有用户定义的队列的父队列是root队列。root队列可以包含任意数量的任意层级的子队列。队列树底部的队列被称为叶子队列。  

**注意**：应用总是通过叶子队列进行调度的。如果用户尝试提交一个应用到一个非叶子队列，那么将会报下面的异常：java.io.IOException: Failed to run job : <Queue_Name> is not a leaf queue。  

一个队列的名字以父队列的名字开始并且后面跟着点字符，然后是当前队列的名字。打个比方，比如slaes队列是root队列的子队列，那么sales队列的路径就是root.sales。类似的，如果sales队列有更深一层的子队列，那么它们的名字将会是root.sales.child1和root.sales.child2。  

##### 调度  
调度是一个实体，可以发起一个任务运行在集群上。调度可能是一个job或者一个队列。在YARN中，它是一个可以用来定义一个公平的共享算法的抽象类，可以应用在一个队列中或者多个队列之间。  

调度有两种类型——JobSchedulables和QueueSchedulabels。  

一个调度的负责下面的任务：  
* 它可以通过它的assignTask()接口启动任务
* 它提供了关于调度器中的job/队列的相关信息，包括：
    * 需求(任务所需要的最大的资源数量)
    * 当前正在运行的任务数
    * 最小的资源共享(对于队列来说)
    * Job/队列的权重(对于公平共享来说)
    * 开始时间和优先级(对于FIFO来说)
* 它可以使用公平调度分配一个公平共享  

##### 调度策略  
FSQueue使用一个调度策略在队列内部分配资源。YARN定义了三种调度策略：  
* 先进先出策略(FIFO)
* 公平共享策略
* 主导资源公平策略(DRF)  

FIFO策略简单并且容易实现。它意味着先被提交的应用将会获取更多的资源。  

公平共享策略是FSQueue默认调度策略。对于公平地共享资源来说有三个规则：  
* **NEED TO DO**
* **NEED TO DO**
* **NEED TO DO**  

##### 配置一个公平调度器  
在本节，我们将会讨论一下配置一个公平调度器的必要步骤。在YARN中，公平调度器的实现被定义在org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler类中。想要配置ResourceManager使用公平调度器，你需要在yarn-site.xml文件中使用下面的属性指定类名：  
```xml
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```  
你也可以在yarn-site.xml文件中配置下面的FairScheduler参数：  

属性                                      | bird
-----------------------------------------|--------------------------------------------------------
yarn.scheduler.fair.allocation.file | 一个包含FSQueue定义和属性的xml文件的路径  
yarn.scheduler.fair.useras-default-queue | 如果没有指定队列名，那么使用用户名作为默认的队列名  
yarn.scheduler.fair.preemption | 如果资源抢占需要被开启，设置为true
yarn.scheduler.fair.preemption.clusterutilization-threshold | 资源抢占中资源利用的临界值，默认是0.8f
yarn.scheduler.fair.sizebasedweight |　这是所有应用程序不论其大小的加权共享或等额共享
yarn.scheduler.fair.update-interval-ms　| 以毫秒的形式计算公平共享，资源需求和资源抢占的请。默认是500ms  

除了在yarn-site.xml文件中配置外，你还需要一个分配文件。分配文件是一个.xml文件，包好了一个FSQueue的定义和它的属性。一个FairScheduler分配文件的样本如下：  
```xml
<?xml version="1.0"?>
<allocations>
  <queue name="queue1" type="parent">
    <minResources>100 mb,1 vcores</minResources>
    <maxResources>8000 mb,8 vcores</maxResources>
    <maxRunningApps>50</maxRunningApps>
    <queue name="sub_queue1">
      <minResources>100 mb,1 vcores</minResources>
    </queue>
  </queue>
  <queue name="queue2">
    <minResources>1000 mb,1 vcores</minResources>
    <maxResources>6000 mb,5 vcores</maxResources>
    <maxRunningApps>40</maxRunningApps>
    <maxAMShare>0.2</maxAMShare>
    <schedulingPolicy>fifo</schedulingPolicy>
    <weight>1.5</weight>
    <schedulingPolicy>fair</schedulingPolicy>
  </queue>
  <queueMaxAMShareDefault>1.0</queueMaxAMShareDefault>
  <userMaxAppsDefault>5</userMaxAppsDefault>
</allocations>
```  
你可以将文件保存在Hadoop配置文件夹下，名为fair-scheduler.xml。你需要在yarn-site.xml文件中使用下面的属性指定该文件的路径：  
```xml
<property>
  <name>yarn.scheduler.fair.allocation.file</name>
  <value>/home/hduser/hadoop-2.5.1/etc/Hadoop/fair-scheduler.xml</value>
</property>
```  
文件中可能也会包含如下与FairScheduler相关的配置：  
* UserElements
* userMaxAppsDefault
* fairSharePreemptionTimeout
* defaultMinSharePreemptionTimeout
* queueMaxAppsDefault
* queueMaxAMShareDefault
* defaultQueueSchedulingPolicy
* queuePlacementPolicy  

提示：想要更深入的阅读与分配文件相关的配置参数，你可以参考FairScheduler的文档 http://hadoop.apache.org/docs/r2.5.1/hadoop-yarn/hadoop-yarn-site/FairScheduler.html#Configuration 。  

在配置完yarn-site.xml文件和分配文件之后，你将需要重新启动ResourceManager服务。想要提交一个job到一个队列，你需要在提交的job的时候，使用-D参数指定队列名。提交一个job到sub_queue1队列的模板如下：  
```shell
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.1.jar pi -Dmapreduce.job.queuename=root.queue1.sub_queue1 2 5
```  
想要查看队列的状态，你可以通过ResourceManager的Web UI查看调度器页面 http://&lt;ResourceManagerIP&gt;:8088/cluster/scheduler?openQueues=root.queue1#root.queue1.sub_queue1 。  

下面的截图显示了root.queue1.sub_queue1对列的状态：  
![image](/Images/YARN/yarn-fairscheduler-stats.png)  

#### 容量调度器  
CapacityScheduler是YARN提供的另一个可插拔的调度器。它允许多个应用通过共享集群的资源一起执行，这样可以最大化集群的吞吐量。它同样对多租户和容量保证提供了支持。CapacityScheduler使用CSQueue对象进行队列的定义。CapacityScheduler的实现定义在org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler类中。  

CapacityScheduler提供了下面的特性：  
* 分层队列
* 容量保证
* 安全性
* 可伸缩性
* 多租户
* 运行时配置
* 消耗性应用
* 基于资源的调度  
想要阅读这么特性的更多内容，可以参考Hadoop文档 http://hadoop.apache.org/docs/r2.6.0/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html 。  

##### 配置CapacityScheduler  
YARN中配置CapacityScheduler就像配置FairScheduler一样简单。想要启用CapacityScheduler，你需要在yanr-site.xml配置下面的属性：  
```xml
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```  
与FairScheduler类似，CapacityScheduler同样有一个分配文件。它与公平调度器一样是一个.xml文件但是与公平调度器的分配文件不同的文件格式。CapacityScheduler默认的分配文件是  $HADOOP_PREFIX/etc/hadoop/capacity-scheduler.xml。  

CapacityScheduler中默认的父队列是root。所有用户定义的队列将会是root队列的子队列。  

在下面的xml文件中为CapacityScheduler定义了三个队列——alpha、beta和default。alpha队列有两个子队列——a1和a2。在每一个层级，一个队列内部的所有容量之和应该是100%。  

你可以参考下面的容量调度器的配置。你可能会发现root队列的子队列的所有容量(alpha-50，beta-30和default-20)之和和alpha队列的子队列的所有容量(a1-60和a2-40)之和都是100，正如下面的配置所示：  
```xml
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>alpha,beta,default</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.alpha.capacity</name>
  <value>50</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.alpha.queues</name>
  <value>a1,a2</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.alpha.a1.capacity</name>
  <value>60</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.alpha.a2.capacity</name>
  <value>40</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.beta.capacity</name>
  <value>30</value>
</property>
<property>
  <name>yarn.scheduler.capacity.root.default.capacity</name>
  <value>20</value>
</property>
```  
与FairScheduler类似，想要提交一个job到一个独立的队列，你需要使用-D参数指定队列名，如下所示：  
```shell
yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.1.jar pi -Dmapreduce.job.queuename=a1 5 10
```  
下面的截图显示了root.alpha.a1对列在应用执行期间的状态：  
![image](/Images/YARN/yarn-capacityscheduler-queue-state.png)  

### 总结
