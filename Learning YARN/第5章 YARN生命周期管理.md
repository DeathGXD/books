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


### NodeManager的关注点

YARN中的NodeManager服务向ResourceManager更新它的资源容量和跟踪运行在本节点上的container的执行。除了节点的健康，NodeManager服务主要负责下面的事：
* 一个应用的执行并和与它相关的containers
* 提供给应用相关的containers本地化执行
* 管理不同应用的logs  

NodeManager拥有它自己的关注点：
* Application：管理着应用的执行、日志和资源
* Container：作为一个独立的进程管理着container的执行
* 本地资源：包含container执行所需要的文件

#### 关注点 1 - Application  


#### 关注点 2 - Container  


#### 关注点 3 - 本地化资源  



### 通过日志分析转换
