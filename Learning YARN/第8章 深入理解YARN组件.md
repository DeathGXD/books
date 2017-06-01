YARN包含了多种高效和可扩展的组件, 使得YARN变成一个强大的、健壮的、优秀的资源管理框架。为了对YARN的资源提供有一个更好的理解和如何与YARN进行整合, 使用者需要对YARN的各种组件有更深入的认识。本章讲解了YARN组件的内部细节, 相关的类和YARN组件之间如何相互影响。在本章中将会洞察YARN的组件, 这样能够让使用者更好的利用YARN的特性并且让YARN与分布式应用框架更容易集成。  

在本章中, 我们将会涉及到下面的主题：
* 理解ResourceManager
* 理解NodeManager
* 与辅助服务协同工作、本地资源、日志聚合
* Timeline服务预览、web application proxy和YARN调度负载模拟器  

### 理解ResourceManager  
ResourceManager是YARN框架的核心组件, 负责管理一个多节点集群的资源。它促使资源的分配和记录YARN集群上跨多个节点运行的分布式应用。它与每个节点上运行的NodeManager进程和每个应用的服务ApplicationMaster共同工作。他管理着整个集群的资源和执行YARN应用。  

ResourceManager拥有多个子组件协助它有效的管理一个多节点的集群，并且集群上并行运行着成千上万的分布式的，资源密集的并且有时限的应用。下图展示了具体的情形：  
![image](/Images/yarn-deep-components.PNG)  

#### 客户端和管理接口
ResourceManager暴露方法给client和集群管理员，用来跟ResourceManager进行RPC通信和接受管理命令的优先级。这里是两个用来跟ResourceManager进行通信的类：
1. **ClientRMService**  
ClientRMService类是ResourceManager的客户端接口。所有的客户端用来创建与ResouceManager的RPC连接。这个模块处理所有的ResouceManager的RPC接口。这个服务的实现被定义在org.apache.hadoop.yarn.server.resourcemanager.ClientRMService包中。客户端初始化这个服务使用客户端配置文件，比如yarn-site.xml。客户端请求ResourceManager：  
    * **应用请求**：这个接口暴露了诸如创建新的application请求，提交applications到集群，杀死一个application，列出containers，应用attempt记录等服务给客户端。  
    * **集群度量**：客户端也可以使用这个服务请求ResourceManager分享集群的度量，节点的容量，调度的细节等等信息。  
    * **安全**：为了连接到一个安全的集群环境，客户端需要使用ResourceManager提供的授权令牌和访问控制列表。  
```xml
想要阅读更多有关定义在ClientRMService的不同方法，你可以参考grepcode的网站地http://grepcode.com/file/repo1.maven.org/maven2/org.apache.hadoop/hadoop-yarnserver-resourcemanager/2.6.0/org/apache/hadoop/yarn/server/resourcemanager/ClientRMService.java
```  
2. **AdminService**  
AdminService类被集群管理员用来管理ResourceManager服务。集群管理员在使用命令行选项rmadmin命令的时候，内部使用的就是AdminService。
下面列出了集群管理员通过AdminService可以执行的一些操作：  
<<<<<<< HEAD
    * 刷新集群的节点、访问控制列表和队列  
    * 检查集群的健康状态  
    * 管理ResourceManager的高可用  

这个服务的实现被定义在org.apache.hadoop.yarn. server.resourcemanager.AdminService包中。  

你可以参看rmadmin命令，在第3章 管理一个YARN集群。

#### 核心接口
ResourceManager的核心包扩scheduler和application manager。
=======
    * **刷新集群的节点、访问控制列表和队列**  
    * **检查集群的健康状态**  
    * **管理ResourceManager的高可用**  
    
#### 核心接口
ResourceManager的核心包含调度和应用的管理。下面的类中定义了ResourceManager如何执行任务的调度、应用的管理和状态信息的管理。  
1. YarnScheduler  
YarnScheduler负责资源的分配  

2. RMAppManager  
>>>>>>> origin/master

3. RMStateStore  

4. SchedulingMonitor  

#### NodeManager接口
ResourceManager与NodeManager进行通信。NodeManager会定期向ResourceManager汇报NodeManager节点的健康和资源信息。下面是一些ResourceManager用来管理集群中所有NodeManager节点的类：  
1. NMLivelinessMonitor  

2. ApplicationMasterLauncher  


#### 安全和令牌管理  



### 理解NodeManager  
NodeManager节点是YARN的工作节点，负责向ResourceManager更新一个节点上的可用资源。它同样也负责健康一个节点的健康状况和为一个应用执行container。下面展示了NodeManager进程中包含的多个子组件，接下来会对这些子组件进行详细的描述：  
![image](/Images/yarn-nm-component.png)  

#### 状态更新
YARN集群的可用资源是由该集群上所有的NodeManager节点上的可用资源相加而来。为了有效的利用集群资源，持续对整个集群的资源进行追踪是非常重要的。NodeManager会定期向ResourceManager发送节点资源和健康状况状态的更新。这能够让ResourceManager有效的调度应用的执行和提高集群的性能。接下来的小节中将会提到一些NodeManager中定义的用来发送更新的类。  
1. NodeStatusUpdater  
每个带有NodeManager进程的子节点都会向ResourceManager进行注册。NodeManager向ResourceManager指定它的可用资源。它有一个StatusUpdater服务，用来更新运行在它上面的应用和container相关的当前状态。NodeManager依据下面几个方面计算可用资源：  
    * 物理内存
    * 虚拟内存和物理内存的比例
    * CPU核数
    * 已停止的containers的持续时间  
    
    它对外暴露了公共的接口用来请求和更新container当前的状态。该服务的实现被定义org.apache.hadoop.yarn.server.nodemanger.NodeStatusUpdaterImpl.

2. NodeManagerMetrics
NodeManager进程管理着其所在节点上的可用资源的度量。它存储节点上原始的度量信息，并且以不同的事件更新每个container的度量，比如：container启动，完成，killed，等等。然而当前的资源度量仅仅考虑内存和CPU核数。这个服务的实现被定义在org.apache.hadoop.yarn.server.nodemanager.metrics.NodeManagerMetrics.

#### 状态和健康管理
周期性的检测集群中不同节点的健康状况是非常重要的，并且如果发现有任何节点是不健康的，应该采取必要的措施。NodeManager服务提供了工具可以在任何时间点去管理节点的状态和健康情况。


#### Container管理



#### 安全和令牌管理



### YARN Timeline服务  




### Web application proxy服务



### YARN调度负载模拟器  



### YARN中的资源本地化
