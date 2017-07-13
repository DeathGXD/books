早在2006年，Apache Hadoop作为一个处理计算机集群中的大数据集并且使用一种编程模型的分布式处理框架就已经出现。Hadoop作为一个处理大数据的低成本和尽可能方便的解决方案被开发出来。Hadoop包含一个存储层，那就是，Hadoop分布式文件系统(Hadoop Distributed File System，HDFS)和一个用于管理集群上资源的利用和任务的执行的MapReduce框架。由于Hadoop实现了高性能的并发数据处理和在通用硬件上工作的能力，Hadoop被用于通过MapReduce编程去对历史数据进行大数据分析和批处理。  









### 初识YARN组件  
YARN将JobTracker的职能分离到了特定的组件，每个组件都有特定的工作去执行。在Hadoop 1中，JobTracker负责集群资源的管理，任务的调度和任务的监控。YARN将JobTracker的这些职能分离到了ResourceManager和ApplicationMaster。YARN使用NodeManager作为工作进程代替TaskTracker去执行map-reduce任务。ResourceManager和NodeManager构成了YARN的计算框架，ApplicationMaster是应用特有的application管理框架。  
![image](/Images/YARN/yarn-components.png)  

#### ResourceManager  
ResourceManager是集群级别的服务(每个集群一个)，管理应用计算资源的调度。依据公平的协议ResourceManager优化了集群内存，CPU资源的利用。它允许不同的限制策略，依据不同的调度算法，比如capacity调度器(容量调度器)，fair调度器(公平调度器)，它有独有的资源分配方法。  

ResourceManager有两个主要组件：  
* Scheduler：是一个纯粹的可插拔的仅仅负责给提交到集群上的应用分配资源的组件，  通过应用容量和队列的限制。Scheduler不提供任何任务计算或者监控的保证，它分配资源仅仅受任务本身的性质和资源的请求的支配。
* ApplicationsManager(AsM)  


#### NodeManager  



#### ApplicationMaster  



#### Container  



### YARN的架构  
在之前的章节中，我们讨论了YARN的组件。现在我们将要讨论YARN的高级架构并且看看这些组件是如何进行交互的。  
![image](/Images/YARN/yarn-architecture.png)  

Resourcemanager服务运行在集群的master节点上。一个YARN客户端提交应用到ResourceManager上。应用可以是单独的MapReduce任务，一个有向无环图的任务，一个java应用，或者是任何的shell脚本。YARN客户端同样也定义了一个ApplicationMaster和在一个节点上启动ApplicationMaster的命令。  

ResourceManager中的ApplicationManager服务将会接受和验证来自客户端应用的请求。ReourceManager中的Scheduler服务将会在一个节点上为ApplicationMaster申请一个container，然在那个节点上的NodManager将会使用命令启动ApplicationMaster。每一个YARN应用都会有一个特殊的叫做ApplicationMaster的container。ApplicationMaster container将会是一个应用的第一个container。  









### YARN如何满足大数据的需要  


### YARN支持的项目  







### 总结
