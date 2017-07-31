YARN框架由ResourceManager服务和Nodemanager服务组成。这些服务维护着与YARN生命周期相关的不同组件，比如：application，container，resource等等。本章将关注YARN框架的核心实现，并且描述了ResourceManager和NodeManager如何在分布式环境下管理应用的执行。  

无所谓你是否是一个Java开发者，一个开源contributor，一个集群管理者，或者是一个用户；本章提供了一个简单的并且容易的方法去洞穿YARN的内部细节。在本章中，我们将会讨论下面主题：  
* 状态管理模拟的介绍
* 对于一个节点、一个应用、一个应用的attempt和一个container的ResourceManager的预览
* 对于一个应用、一个container和一个resource的NodeManager的预览
* 通过logs分析转换  

### 状态管理模拟介绍  
任何系统中，在事件驱动实施的组件中，生命周期管理都是重要的一部分。  



### ResourceManager的视野
作为master服务，ResourceManager服务管理着下面内容：  
* 集群资源(集群中的节点)
* 提交到集群上的应用
* 应用运行的尝试次数
* 运行在集群节点上的Containers  

ResourceManager服务拥有它自己的关注点，是与YARN管理和YARN中应用执行相关的不同进程。下面是ResourceManager的目标：  
* **Node**：是带有NodeManager进程的机器
* **Application**：是被客户端提交到ResourceManager的程序
* **Application Attempt**：与应用执行相关的attempt
* **Container**：是运行被提交应用的业务逻辑的进程  

#### 视野 1 - Node  
节点角度是ResourceManager管理着集群内部所有的NodeManager节点的生命周期。对于集群中的每一个，ResourceManager都会维护着一个RMNode对象。每个节点的状态和事件类型都被定义在枚举NodeState和RMNodeEventType中。  

下面是涉及到枚举和类：  
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNode：这是一个接口，定义了一个NodeManager节点上关于可用资源的信息，比如：它的容量，已经执行的应用，正在运行的container，等等。
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeImpl：该类被用于持续跟踪一个节点上正在运行的应用/container和定义节点状态的转换。  
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeEventType：这是一个枚举，定义节点中不同的事件类型。
* org.apache.hadoop.yarn.api.records.NodeState：这是一个枚举，定义了节点中不同的状态。  

下面的状态转换图说明了ResourceManager对于一个节点的关注点：  
![image](/Images/YARN/yarn-resourcemanager-state-update.png)  

一个节点在ResourceManager中开始和最终的情形如下：  
* 开始状态：**NEW**
* 最终状态：**DECOMMISSION/REBOOTED/LOST**  

NodeManager一旦向ResourceManager进行注册，该节点就会被标记为NEW状态。在注册成功之后，状体会被更新为RUNNING。



#### 视野 2 - Application  
ResourceManager在application中的关注点表示在YARN集群上执行的应用运行期间的生命周期的管理。在之前的章节，我们讨论了应用执行的不同阶段。本节，我们将会对ResourceManager如何管理应用的生命周期给出一个更详细的说明。  

这里是一系列涉及到的枚举和类：  
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMApp：这是一个ResourceManager中应用程序的接口
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl：这是一个被用于访问应用状态/报告的各种更新和定义应用中状态转换
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppEventType：这是一个定义了一个应用中不同事件类型的枚举
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppState：这是一个定义了一个应用中不同状态的枚举  

下面的状态转换图说明了ResourceManager对于一个应用的关注点：
![image](/Images/YARN/yarn-resourcemanager-application-view.png)



#### 视野 3 - 一个应用的attempt


#### 视野 4 - Container


### NodeManager的视野  
YARN中的NodeManager服务向ResourceManager更新它的资源容量和跟踪运行在本节点上的container的执行。除了节点的健康，NodeManager服务主要负责下面的事：
* 一个应用的执行并和与它相关的containers
* 提供给应用相关的containers本地化执行
* 管理不同应用的logs  

NodeManager拥有它自己的关注点：
* Application：管理着应用的执行、日志和资源
* Container：作为一个独立的进程管理着container的执行
* 本地资源：包含container执行所需要的文件

#### 视野 1 - Application  


#### 视野 2 - Container  


#### 视野 3 - 本地化资源  



### 通过日志分析内部变化  
在YARN中的ResourceManager和NodeManager服务都会生成日志文件并且以.log格式的文件将它们存放在以HADOOP_LOGS_DIR变量指定的本地文件目录内。默认情况下，日志文件会被存放在HADOOP_PREFIX/logs目录中。YARN中所有的状态转换都会被记录在日志文件中。在本节，我们将会涉及到下面几个状态转换和这些转换期间的日志生成。  

**设置日志级别**：Hadoop-YARN使用Apache log4j库进行日志生成并且它使用一个位于HADOOP_PREFIX/etc/hadoop配置文件目录的log4j.properties日志配置文件。log4j支持6个日志级别-TRACE，DEBUG，INFO，WARN，ERROR和FATAL。集群管理员可以设置Hadoop-YARN服务的日志级别，默认的日志级别是INFO。hadoop.root.logger属性被用来更改Hadoop-YARN服务的日志级别。想要了解更多有关Apache Log4j库，你可以参考官方网站 http://logging.apache.org/log4j 。  

#### NodeManager向ResourceManager进行注册  
ResourceManager包含ResourceTracker服务，负责监控整个集群资源。NodeManager服务向ResourceManager进行注册。注册信息包括节点使用使用的端口和内存信息。在注册成功之后，节点的状态会从改为RUNNING。  

你可以参考下面的ResourceManager在NodeManager进行注册期间的日志：  
```shell
2014-10-25 08:24:15,183 INFO org.apache.hadoop.yarn.server.resourcemanager.ResourceTrackerService:
NodeManager from node master(cmPort: 37594 httpPort: 8042) registered with capability: <memory:8192, vCores:8>, assigned nodeId master:37594
2014-10-25 08:24:28,079 INFO org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeImpl:
master:37594 Node Transitioned from NEW to RUNNING
```  

#### 应用提交  
下面的ResourceManager的日志描述了应用执行期间状态的变化。ClientRMService分配一个新的application ID，RMAppImpl初始化application对象并将其设置为NEW状态。一旦应用被提交，它将会被分配到一个队列并且应用的状态将会从SUBMITTED更改为ACCEPTED。  
```shell
org.apache.hadoop.yarn.server.resourcemanager.ClientRMService:Allocated new applicationId: 1

org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl:Storing application with id application_1414205634577_0001

org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl:application_1414205634577_0001 State change from NEW to NEW_SAVING org.apache.hadoop.yarn.server.resourcemanager.recovery.RMStateStore:
Storing info for app: application_1414205634577_0001

org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl:application_1414205634577_0001 State change from NEW_SAVING to
SUBMITTED

org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.ParentQueue: Application added - appId: application_1414205634577_0001 user: akhil leaf-queue of parent: root #applications: 1

org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler: Accepted application application_1414205634577_0001from user: akhil, in queue: default

org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl:application_1414205634577_0001 State change from SUBMITTED to
ACCEPTED
```  

#### Container资源分配  
ResourceManager调度器负责给YARN上的应用分配Container。下面的日志描述了分配的container的细节(container ID，内存和core)。它同样包含了分配的host，container的数量，使用的内存和分配之后可用的内存等概要信息。  
```shell
org.apache.hadoop.yarn.server.resourcemanager.scheduler.SchedulerNode: Assigned container container_1414205634577_0001_01_000003 of capacity <memory:1024, vCores:1> on host master:37594, which has 3
containers, <memory:4096, vCores:3> used and <memory:4096, vCores:5> available after allocation
```  

#### 资源本地化  
NodeManager服务负责提供container执行期间所需要的资源。NodeManager从所支持的源(比如HDFS，HTTP等等)将资源下载到NodeManager所在节点的本地目录中。  

你可以参考下面NodeManager在资源本地化期间的日志：  
```shell
2014-10-25 14:01:26,224 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer.
LocalizedResource: Resource hdfs://master:8020/tmp/hadoopyarn/staging/akhil/.staging/job_1414205634577_0001/job.splitmetainfo(
->/tmp/hadoop-akhil/nm-localdir/usercache/akhil/appcache/application_1414205634577_0001/filecache
/10/job.splitmetainfo) transitioned from DOWNLOADING to LOCALIZED
```  

### 总结
