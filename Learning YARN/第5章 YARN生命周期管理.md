YARN框架由ResourceManager服务和Nodemanager服务组成。这些服务维护着与YARN生命周期相关的不同组件，比如：application，container，resource等等。本章将关注YARN框架的核心实现，并且描述了ResourceManager和NodeManager如何在分布式环境下管理应用的执行。  

无所谓你是否是一个Java开发者，一个开源contributor，一个集群管理者，或者是一个用户；本章提供了一个简单的并且容易的方法去洞穿YARN的内部细节。在本章中，我们将会讨论下面主题：  
* 状态管理模拟的介绍
* 对于一个节点、一个应用、一个应用的attempt和一个container的ResourceManager的预览
* 对于一个应用、一个container和一个resource的NodeManager的预览
* 通过logs分析转换  

### 状态管理模拟介绍  
任何系统中，在事件驱动实施的组件中，生命周期管理都是重要的一部分。  



### ResourceManager的关注点
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

#### 关注点 1 - Node  
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



#### 关注点 2 - Application  
ResourceManager在application中的关注点表示在YARN集群上执行的应用运行期间的生命周期的管理。在之前的章节，我们讨论了应用执行的不同阶段。本节，我们将会对ResourceManager如何管理应用的生命周期给出一个更详细的说明。  

这里是一系列涉及到的枚举和类：  
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMApp：这是一个ResourceManager中应用程序的接口
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl：这是一个被用于访问应用状态/报告的各种更新和定义应用中状态转换
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppEventType：这是一个定义了一个应用中不同事件类型的枚举
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppState：这是一个定义了一个应用中不同状态的枚举  

下面的状态转换图说明了ResourceManager对于一个应用的关注点：
![image](/Images/YARN/yarn-resourcemanager-application-view.png)



#### 关注点 3 - 一个应用的attempt


#### 关注点 4 - Container


### NodeManager的视野  
YARN中的NodeManager服务向ResourceManager更新它的资源容量和跟踪运行在本节点上的container的执行。除了节点的健康，NodeManager服务主要负责下面的事：
* 一个应用的执行并和与它相关的containers
* 提供给应用相关的containers本地化执行
* 管理不同应用的logs  

NodeManager拥有它自己的关注点：
* Application：管理着应用的执行、日志和资源
* Container：作为一个独立的进程管理着container的执行
* 本地资源：包含container执行所需要的文件

#### 关注点 1 - Application  
NodeManager管理着应用的container的生命周期和应用执行期间使用的资源。NodeManager观察一个应用表示的是NodeManager如何管理container的执行，资源和应用的日志。  

下面是设计到的一些枚举和类的列表。所有的这些类都被定义在org.apache.hadoop.yarn.server.nodemanager.containermanager.application包中。  
* Application：这个是一个接口，对应于NodeManager上应用。它仅仅只用于存储应用的元数据。
* ApplicationImpl：这个类定义了应用状态的转变和与之相关联的事件处理器。
* ApplicationEventType：这是一个枚举类型，定义了一个应用的不同的事件类型。
* ApplicationState：这是一个枚举类型，定义了一个应用的不同状态。  

NodeManager服务仅仅存储了与应用相关的最基本的信息。应用元数据包括：  
* application ID
* NodeManager所关心的application的状态
* 相关联的container列表
* 用户名  

下面的状态转换图说明了NodeManager对一个应用的关注点：  
![image](/Images/YARN/yarn-application-state-change.png)  

NodeManager对一个应用所关注的初始状态和最终状态如下：  
* 初始状态：NEW
* 最终状态：FINISHED  

在一个应用执行期间，ApplicationMaster作为应用的第一个container运行。ResourceManager接受应用的请求并且为ApplicationMaster服务分配资源。在NodeManager内部的ContainerManager服务接受应用的请求并且在NodeManager节点上启动ApplicationMaster服务。  

NodeManager将一个应用的状态标记为NEW，初始化应用，并且将应用的状态更改为INITING。在这一个转变过程中，日志聚集服务连同应用访问控制列表一起也会被初始化。如果在初始化应用或者创建日志目录期间出现异常，那么应用将会停留在INITING状态并且NodeManager会发送警告信息给用户。NodeManager会等待要么是Application_Inited事件要么是Finish_Application事件。  

注意：如果日志聚集被开启，但是创建日志目录失败了，那么一条警告信息如Log Aggregation service failed to initialize, there will be no logs for this application将会被记录。  

如果Application_Inited完成了，应用的状态将会被改为RUNNING。在应用执行期间，应用会请求并且运行一些container。诸如 Application_Container_Finished和Container_Done_Transition的事件会更新应用的container列表，但是应用的状态不会被改变。  

当应用执行完成时，Finish_Application事件将会被触发。NodeManager会等待应用所有当前正在的运行的container的执行完成。应用的状态会改为Finishing Containers Wait。在所有的container完成之后，NodeManager服务会清理所有被应用所使用的资源并且为应用执行日志聚集。一旦资源清理完成，应用将会被标记为FINISHED。  

#### 关注点 2 - Container  
正如之前讨论的，NodeManager负责提供资源，container的执行，资源的清理，等等。NodeManager中一个container的生命周期被定义在org.apache.hadoop.yarn.server.nodemanager.containermanager.container包中。  

下面是涉及到的一些枚举和类的列表：  
* Container：这是一个接口，对应于NodeManager上的container。
* ContainerImpl：这个类定义了container的状态转化和与之相关联的事件处理器。
* ContainerEventType：这是一个枚举，定义了container中不同的事件类型。
* ContainerState：这是一个枚举，定义了container中不同的状态。  

下面的状态转化图说明了NodeManager对于一个container的关注点：  
![image](/Images/YARN/yarn-container-state-change.png)  

NodeManager对一个container所关注的初始状态和完成状态如下：  
* 初始状态：NEW
* 完成状态：Done  

ResourceManager分配container到一个单独的机器上。因为container是被一个应用所需要，NodeManager会在节点初始化container对象。NodeManager会请求ResourceLocalizationManager服务去下载运行container所需要的资源并且将container的状态标记为Localizing。  

对于本地化，使用的是特定的服务。辅助服务拥有container服务数据的信息。当资源被成功本地化之后，Resource_Localized事件将会被触发。如果资源成功本地化或者资源并不需要本地化，那么container将会直接进入Localized状态。如果资源本地化失败，那么container的状态就会被变为Localization Failed。如果container直接进入Done状态，那么container运行将会被跳过。  

一旦container的资源需求条件被满足，那么Container_Launched事件将会被触发并且container状态将会被改为Running。在这个过渡中，ContainerMonitor服务会被用于去监控container的资源使用情况。NodeManager会等待container的完成。如果container被成功执行，那么一个成功事件将会被调用并且container的退出状态会被标记为0。container的状态会被更改为EXITED_WITH_SUCCESS。如果container失败了，那么退出码和诊断信息将会被更新，并且container的状态会被更改为EXITED_WITH_FAILURE。  

在任务状态，如果一个container接受到了一个kill信号，那么container都会变为KILLING状态，当container被回收完成，那么container的状态会被更改为Container_CleanedUp_After_Kill。对于一个container来说，回收它使用的资源是强制性的。当资源被回收完成，Container_Resources_CleanedUp事件会被调用并且状态会被标记为Done。  

#### 关注点 3 - 本地化资源  
资源本地化是作为执行container之前下载container所需要的资源文件被定义的。比如，如果一个container的执行需要一个jar文件，那么一个本地资源就会配置在ContainerLaunchContext中。它负责给NodeManager服务下载资源文件到NodeManager所在节点的本地文件系统。想要了解更多有关资源本地化的内容，你可以参考第8章 深入理解YARN组件。  

NodeManager维护了本地化资源的生命周期。它存储了与资源相关的信息。信息包括：  
* 在本地文件系统上的资源路径
* 资源大小
* container使用的资源列表
* 资源的可见性，类型，模式和下载路径
* LocalResourceRequest：NodeManager中本地资源的生命周期，定义在 org.apache.hadoop.yarn.server.nodemanager.containermanager.localizer包中。
* LocalizedResource：这个类存储了资源的信息，定义了状态的转换和与之相关联的事件处理器。
* ResourceState：这是一个枚举，定义了一个资源的不同的状态。
* event.ResourceEventType：一个枚举，定义了一个资源的不同的事件类型。  

下面的状态转换图说明NodeManager对资源的关注点：  
![image](/Images/YARN/yarn-resource-state.png)  

对于资源，NodeManager的初始关注点和最终关注点如下：  
* 初始状态：INIT
* 最终状态：LOCALIZED/FAILED  

作为NodeManager中对container指定的关注点，资源本地化在INIT_CONTAINER事件期间被初始化。在container运行上下文中指定了资源初始化的状态为INIT。  

当请求一个资源的时候，FetchResourceTransition处理会被调度，并且它会初始化一些资源的信息，比如位置，可见性，上下文等等。资源的状体也会被更改为DOWNLOADING。  

一旦资源下载成功，资源的状态会被标记为LOCALIZED，并且资源路径，大小和引用都会被更新。如果资源本地化失败，那么资源会被标记为FAILED，并且上下文会被更新，带有失败原因的诊断信息。  

如果对DOWNLOADING和LOCALIZED状态的资源有任何后来的请求，那么会提供本地资源的路径和上下文进行处理。  

多个container可以使用在同一时间使用相同的资源。NodeManager服务对每一个本地化的资源都维护了一个container队列。在资源被请求期间，container的引用会被添加到队列中，在资源释放之后会从队列中删除container的引用。  

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
