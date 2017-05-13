YARN框架由ResourceManager服务和Nodemanager服务组成。这些服务维护着与YARN生命周期相关的不同组件，比如：application，container，resource等等。本章将关注YARN框架的核心实现，并且描述了ResourceManager和NodeManager如何在分布式环境下管理应用的执行。  

无所谓你是否是一个Java开发者，一个开源contributor，一个集群管理者，或者是一个用户；本章提供了一个简单的并且容易的方法去洞穿YARN的内部细节。在本章中，我们将会讨论下面主题：  
* 状态管理模拟的介绍
* 对于一个节点、一个应用、一个应用的attempt和一个container的ResourceManager的预览
* 对于一个应用、一个container和一个resource的NodeManager的预览
* 通过logs分析转换  

### 状态管理模拟介绍  
任何系统中，在事件驱动实施的组件中，生命周期管理都是重要的一部分。  



### ResourceManager的风景
作为master服务，ResourceManager服务管理着下面内容：  
* 集群资源(集群中的节点)
* 提交到集群上的应用
* 应用运行的尝试次数
* 运行在集群节点上的Containers  

ResourceManager服务拥有它自己的  


#### 风景 1 - Node



#### 风景 2 - Application



#### 风景 3 - 一个应用的attempt


#### 风景 4 - Container


### NodeManager的风景

YARN中的NodeManager服务向ResourceManager更新它的资源容量和跟踪运行在本节点上的container的执行。除了节点的健康，NodeManager服务主要负责下面的事：
* 一个应用的执行并和与它相关的containers
* 提供给应用相关的containers本地化执行
* 管理不同应用的logs  

NodeManager拥有它自己的风景：
* Application：管理着应用的执行、日志和资源
* Container：作为一个独立的进程管理着container的执行
* 本地资源：包含container执行所需要的文件

#### 风景 1 - Application  


#### 风景 2 - Container  


#### 风景 3 - 本地化资源  



### 通过日志分析转换
