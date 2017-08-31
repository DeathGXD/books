YARN框架由ResourceManager服务和Nodemanager服务组成。这些服务维护着与YARN生命周期相关的不同组件，比如：application，container，resource等等。本章将关注YARN框架的核心实现，并且描述了ResourceManager和NodeManager如何在分布式环境下管理应用的执行。  

无所谓你是否是一个Java开发者，一个开源contributor，一个集群管理者，或者是一个用户；本章提供了一个简单容易的途径去洞穿YARN的内部细节。在本章中，我们将会讨论下面主题：  
* 介绍状态管理模拟
* ResourceManager对于一个节点、一个应用、一个应用的attempt和一个container的关注点
* NodeManager对于一个应用、一个container和一个resource的关注点
* 通过logs分析内部转换  

### 状态管理模拟介绍  
在任何系统中，以事件驱动实现的组件中的生命周期管理都是重要的一部分。系统组件通过预定义好的一系列有效状态进行通信传递。状态之间的转换是由相关的状态和动作去执行相对应的事件。  

下面是本章中使用到的一些核心的术语：  
* **State**：在计算机科学中，计算机程序的状态是一个专业术语，指的所有被存储的信息，在指定之间内给定的实例中，哪个程序可以访问。
* **Event**：一个事件是一个动作
* **Event handle**：
* **State transition**：



### ResourceManager的视角
作为master服务，ResourceManager服务管理着下面内容：  
* 集群资源(集群中的节点)
* 提交到集群上的应用
* 正在运行的应用的attempt
* 集群节点上正在运行的Containers  

ResourceManager服务拥有它自己的关注点，是与YARN管理和YARN中应用执行相关的不同进程。下面是ResourceManager的关注点：  
* **Node**：带有NodeManager进程的机器
* **Application**：客户端提交到ResourceManager的程序
* **Application Attempt**：与应用执行相关的attempt
* **Container**：运行提交应用业务逻辑的进程  

#### 视角 1 - Node  
对于节点的关注是ResourceManager管理着集群内部所有的NodeManager节点的生命周期。对于集群中的每一个，ResourceManager都会维护着一个RMNode对象。每个节点的状态和事件类型都被定义在枚举NodeState和RMNodeEventType中。  

下面是涉及到枚举和类：  
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNode：这是一个接口，定义了一个NodeManager节点上关于可用资源的信息，比如：它的容量，已经执行的应用，正在运行的container，等等。
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeImpl：该类被用于持续跟踪一个节点上正在运行的应用/container和定义节点状态的转换。  
* org.apache.hadoop.yarn.server.resourcemanager.rmnode.RMNodeEventType：这是一个枚举，定义节点中不同的事件类型。
* org.apache.hadoop.yarn.api.records.NodeState：这是一个枚举，定义了节点中不同的状态。  

下面的状态转换图说明了ResourceManager对于一个节点的视角：  
![image](/Images/YARN/yarn-resourcemanager-state-update.png)  

一个节点在ResourceManager中开始和最终的情形如下：  
* 开始状态：**NEW**
* 最终状态：**DECOMMISSION/REBOOTED/LOST**  

NodeManager一旦向ResourceManager进行注册，该节点就会被标记为NEW状态。在注册成功之后，状体会被更新为RUNNING。一个AddNodeTransition事件处理器会被初始化，用于更新新节点的调度器和它的容量。如果该节点存储于不活跃的RM节点列表，RM会从不活跃节点列表删除该节点，随着新加入的节点信息更新集群的度量。如果节点是新加入的，那么会直接在集群的度量中增加活跃的节点的数量。  

每个NodeManager节点都会以心跳的形式向ResourceManager发送它的活跃信息。ResourceManager会持续跟踪每个节点发送的最后一次心跳，如果最后一次通信的时间间隔大于集群中配置的所允许的最大时间间隔，那么该节点会被认为是过期的。默认情况下，等待NodeManager发送心跳的最大时间间隔是600000ms，也就是10分钟，如果超过这个时间，那么该节点会被认为死亡。然后会将该节点标记为UNUSABLE，并且会将它添加到ResourceManager中的非活跃节点列表。所有运行在死亡节点上的container也会被认为是死亡的，那么会从新调度新的container到其他NodeManager节点。NodeManager节点同样可以从集群中退役，或者是由于技术原因进行重启。  

NodeManager的心跳信息也包含了与该节点相关的正在运行的或者完成的container的信息。如果节点是健康的，那么节点的信息将会更新为最新的度量，并且为下一次心跳初始化一个更新调度事件。ResourceManager同样也会追踪每个NodeManager上完成的应用和container。  

假如NodeManager重启或者一个服务重启，NodeManager都会尝试重新连接到ResourceManager继续开始它的服务。如果重连节点的配置(节点总容量和节点HTTP端口)改变了，在重连节点相同的情况下(主机IP地址没有改变)，那么NodeManager会替换旧的或者重新设置心跳。调度器伴随着新加入节点事件将会被更新并且节点将会标记为RUNNING。  

YARN包中定义的NodeHealthCheckerService类用于NodeManager判断节点的健康状况。每个节点都会定期执行YARN配置文件中yarn.nodemanager.health-checker.script.path属性中配置的一个健康检查的脚本。默认运行节点中健康检查脚本的频率是600000ms，也就是10分钟，可以通过yarn.nodemanager.health-checker.interval-ms属性进行配置。  

NodeHealthScriptRunner类用来运行节点中健康检查的脚本，解析健康监控脚本的输出并且检查报告中的错误。脚本超时或者脚本中引起了IOException输出都将会被忽略。如果脚本抛出java.io.IOException或者org.apache.hadoop.util.Shell.ExitCodeException，输出被忽略，节点被认为保持健康，因为脚本可能存在语法错误。  

如果存在下面的情况，节点会被认为是非健康：  
* 健康脚本超时
* 健康脚本输出中存在一行以ERROR开头
* 当执行脚本的时候抛出了一个异常  

节点同样会运行一个DiskHealthCheckerService类，去获取节点磁盘的健康信息。想要阅读更多有关节点健康检查脚本的信息，你可以参考第3章 管理一个Hadoop-YARN集群。  

下面是一个ResourceManager对节点视角的总览表：  
![image](/Images/YARN/yarn-resourcemanager-view.png)  

#### 视角 2 - Application  
ResourceManager在application中的关注点表示在YARN集群上执行的应用运行期间的生命周期的管理。在之前的章节，我们讨论了应用执行的不同阶段。本节，我们将会对ResourceManager如何管理应用的生命周期给出一个更详细的说明。  

这里是一系列涉及到的枚举和类：  
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMApp：这是一个ResourceManager中应用程序的接口
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl：这是一个被用于访问应用状态/报告的各种更新和定义应用中状态转换
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppEventType：这是一个定义了一个应用中不同事件类型的枚举
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppState：这是一个定义了一个应用中不同状态的枚举  

下面的状态转换图说明了ResourceManager对于一个应用的视角：
![image](/Images/YARN/yarn-resourcemanager-application-view.png)  

ResourceManager对应用视角的初始状态和最终状态如下：  
* 初始状态：NEW
* 最终状态：FAILED/FINISHED/KILLED  

当客户端提交一个新的应用请求到ResourceManager时，RM会注册该应用并且提供一个唯一的application ID(在前面章节提到的应用执行流程中的第一阶段)。应用的状态会被初始化为NEW。  

RMAppImpl对象维护了一组包含节点当前状态相关信息的RMNode对象。在NODE_UPDATE事件期间，ResourceManager会更新集群中的可用节点和不可用节点的信息。在NEW状态的应用在NODE_UPDATE期间状态不会改变。  

客户端使用application submission context提交应用。如果ResourceManager的恢复机制被开启，应用的submission context会被存储到为ResourceManager配置的状态存储中。一旦应用上下文被保存，那么应用就会被提交到集群上，意味着应用被放入到一个对列中执行了。  

应用会从配置的队列中拿出来执行；如果应用的资源要求被满足，那么该应用将会被接受并且它的状态会被修改为ACCEPTED。  

如果应用由于资源不足或者其他一些异常被拒绝，那么一个App_Rejected事件将会被触发。在这种情况下，应用的状态将会被标记为FAILED。  

ApplicationMaster作为应用执行中的第一个attempt在其中的一个节点上被运行。ApplicationMaster会向ResourceManager注册它的attempt，并且为attempt创建一个RMAppAttempt上下文。在成功注册之后，应用的状态将会被更改为RUNNING。  

一旦成功完成，应用的attempt会注销它的attempt，更改它的状态为REMOVING，之后会更改为FINISHED状态。如果attempt是一个非管理的attempt，那么attempt可以直接从RUNNING变更为FINISHED状态。  

如果一个attempt失败了，那么ResourceManager会在另一台节点上重新执行应用的attempt。伴随着Attempt_Failed事件的触发，应用会被标记为SUBMITTED，并且应用的attempt的数量会增加。如果重试次数超过了配置中指定了最大重试次数，那么应用将会被标记为FAILED。  

你可以在yarn-site.xml文件中指定应用允许的最大attempt次数，配置如下：  
```xml
<property>
  <description>Default value is 2</description>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>2</value>
</property>
```  

在应用的任何状态中，包括SUBMITTED、ACCEPTED、RUNNING、等等，如果一个kill信号或事件被用户发送，那么应用的状态会直接更新为FAILED，并且应用使用的所有的container将会被释放。  

下面是一个ResourceManager对应用视角的总览表：  
![image](/Images/YARN/yarn-resourcemanager-application-view1.png)  
![image](/Images/YARN/yarn-resourcemanager-application-view2.png)  
![image](/Images/YARN/yarn-resourcemanager-application-view3.png)  

#### 视角 3 - 一个应用的attempt  
ResourceManager对于应用attempt的视角，代表了在YARN集群上执行的应用的每个attempt的生命周期。正如我们在应用生命周期中所看到的，当一个应用的状态从ACCEPTED转移到RUNNING，应用的attempt会向ResourceManager进行注册。本节将涉及到应用attempt的状态管理。  

下面是涉及到的类和枚举的列表：  
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttempt：这是ResourceManager中一个应用attempt的接口。一个应用基于配置的attempt最大数量可以有多个attempt。
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttemptImpl：这个类定义了应用attempt状态转换和对应用当前attempt的访问。
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttemptEventType：这是一个定义了应用attempt不同事件类型的枚举。
* org.apache.hadoop.yarn.server.resourcemanager.rmapp.attempt.RMAppAttemptState：这是一个定义了应用attempt不同状态的枚举。  

下面的状态转换图说明了ResourceManager对应用attempt的视角：  
![image](/Images/YARN/yarn-application-attempt-view.png)  

ResourceManager对于一个应用attempt container的初始状态和最终状态的视角如下：  
* 初始状态：NEW
* 最终状态：FINISHED/EXPIRED/RELEASED/KILLED  

当ResourceManager成功接收一个应用，那么该应用的attempt被初始化为NEW状态。一个新的attemptId会被生成给attempt，并且attempt会被加入到应用的attempt列表。  

在那之后，一个RMAppStartAttemptEvent处理器被调用并且attempt的状体会被更改为SUBMITTED。在启动attempt事件期间，attempt首先会向ResourceManager中的ApplicationMasterService进行注册。如果应用运行在安全的模式，那么用户需要经估计应用中的client-token-master-key的验证，在ResourceManager上下文中拥有着相同的key。想要了解更多有关YARN安全的知识，你可以参考第11章，启用YARN集群安全机制。一个AMRMToken会被生成，并且attempt会被添加到调度器中。  

ResourceManager的调度器会接受应用的attempt并且分配为ApplicationMaster程序分配container，依据ContainerLauncheContext对象中的每个条件。如果应用被配置为一个非管理的AM，那么attempt将会被保存并且状态会之间更改为LAUNCHED。  

**非管理的AM**：如果ResourceManager没有管理ApplicationMaster的执行，那么就会认为该应用是非管理的。一个非管理的AM不会要求container的分配，并且ResourceManager也不会启动ApplicationMaster服务。客户端仅仅只能在ResourceManager已经ACCEPTED应用的时候才会启动ApplicationMaster服务。如果ApplicationMaster在其启动期间没有成功连接到ResourceManager，那么ResourceManager会将该应用标记为失败。  

如果ApplicationMaster执行在一个管理环境，那么attempt的状态会被更改为SCHEDULED。之后attempt会请求调度器为期分配container用于ApplicationMasterg服务。  

在成功分配之后，attempt会取得分配的container并且attempt的状态会更改为ALLOCATED。一旦container被分配了，ResourceManager会执行命令去启动ApplicationMaster并且attempt的状态会更改为LAUNCHED。ResourceManager会在ApplicationMaster启动期间等待它的注册，否则ResourceManager会将其标记为FAILED。ApplicationMaster在注册的时候会带着自己所在机器的主机名和端口号和一个用于监控应用执行的URL。ApplicationMaster同样也会向ResourceManager(AMRMClient)和NodeManager(AMNMClient)注册一个通信客户端令牌。  

ApplicationMaster将会向ResourceManager请求container并且管理应用的执行。一旦attempt完成了，它会注销自己并且将状态更改为FINISHING，直到最终状态被保存后，那么attempt会被标记为FINISHED。在attempt执行的任何阶段，如果有任何异常发生，那么attempt都会被标记为FAILED。比如，如果在attempt注册期间出现了一个错误，那么attempt将会被拒绝。类似的，当我们在管理ApplicationMaster的时候，如果没有足够的资源去运行ApplicationMaster，那么运行事件就会失败，attempt也会被标记为FAILED。  

如果客户端发送一个信号去杀死一个应用，那么它的attempt或者所有被分配的container将会直接被标记为FAILED。一个KillAllocatedAMTransition处理器会被调用，清理所有已经被执行的任务。  

下面是一个ResourceManager对应用attempt视角的总览表：  
![image](/Images/YARN/yarn-application-attempt-view1.png)  
![image](/Images/YARN/yarn-application-attempt-view2.png)  
![image](/Images/YARN/yarn-application-attempt-view3.png)  

#### 视角 4 - Container  
ResourceManager管理着所有请求的container的生命周期。应用的执行需要container，在应用完成之后会将container归还给ResourceManager。ResourceManager存储着与每个container相关的元数据，并且相应的对应用进行调度。  

container的元数据包括：  
* **Container ID**：这是对于所有的container来说唯一的ID。  
* **Application attempt ID**：与container相关联的应用attempt ID。
* **Node ID**：这是为container预留的和分配的节点。
* **Resource**：内存和虚拟core。
* **Time-Stamps**：container创建和完成时间。
* **States**：包括ContainerState和RMContainerState。
* **Monitoring Info**：container的诊断信息和日志URL。  

下面是涉及到的类和枚举列表：  
* org.apache.hadoop.yarn.server.resourcemanager.rmcontainer.RMContainer：对于ResourceManager来说一个container的接口。它存储了container的属性，比如：它的优先级，创建时间，attempt ID，等等。
* org.apache.hadoop.yarn.server.resourcemanager.rmcontainer.RMContainerImpl：这个类定义container的状态转换和相关联的事件处理器。
* org.apache.hadoop.yarn.server.resourcemanager.rmcontainer.RMContainerEventType：定义了container中不同的事件类型的枚举。
* org.apache.hadoop.yarn.server.resourcemanager.rmcontainer.RMContainerState：定义了container中不同的状态的枚举。  

下面的状态转换图说明了ResourceManager对于container的视角：  
![image](/Images/YARN/yarn-container-state-transition.png)  

ResourceManager对一个container初始状态和最终状态的视角如下：  
* 初始状态：**NEW**
* 最终状态： COMPLETED/EXPIRED/RELEASED/KILLED  

ResourceManager会初始化一个新的RMContainer作为请求的container被接受。










### NodeManager的视角  
YARN中的NodeManager服务向ResourceManager更新它的资源容量和跟踪运行在本节点上的container的执行。除了节点的健康，NodeManager服务主要负责下面的事：
* 一个应用的执行并和与它相关的containers
* 提供给应用相关的containers本地化执行
* 管理不同应用的logs  

NodeManager拥有它自己的关注点：
* Application：管理着应用的执行、日志和资源
* Container：作为一个独立的进程管理着container的执行
* 本地资源：包含container执行所需要的文件

#### 视角 1 - Application  
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

下面的状态转换图说明了NodeManager对一个应用的视角：  
![image](/Images/YARN/yarn-application-state-change.png)  

NodeManager对一个应用所关注的初始状态和最终状态如下：  
* 初始状态：NEW
* 最终状态：FINISHED  

在一个应用执行期间，ApplicationMaster作为应用的第一个container运行。ResourceManager接受应用的请求并且为ApplicationMaster服务分配资源。在NodeManager内部的ContainerManager服务接受应用的请求并且在NodeManager节点上启动ApplicationMaster服务。  

NodeManager将一个应用的状态标记为NEW，初始化应用，并且将应用的状态更改为INITING。在这一个转变过程中，日志聚集服务连同应用访问控制列表一起也会被初始化。如果在初始化应用或者创建日志目录期间出现异常，那么应用将会停留在INITING状态并且NodeManager会发送警告信息给用户。NodeManager会等待要么是Application_Inited事件要么是Finish_Application事件。  

注意：如果日志聚集被开启，但是创建日志目录失败了，那么一条警告信息如Log Aggregation service failed to initialize, there will be no logs for this application将会被记录。  

如果Application_Inited完成了，应用的状态将会被改为RUNNING。在应用执行期间，应用会请求并且运行一些container。诸如 Application_Container_Finished和Container_Done_Transition的事件会更新应用的container列表，但是应用的状态不会被改变。  

当应用执行完成时，Finish_Application事件将会被触发。NodeManager会等待应用所有当前正在的运行的container的执行完成。应用的状态会改为Finishing Containers Wait。在所有的container完成之后，NodeManager服务会清理所有被应用所使用的资源并且为应用执行日志聚集。一旦资源清理完成，应用将会被标记为FINISHED。  

#### 视角 2 - Container  
正如之前讨论的，NodeManager负责提供资源，container的执行，资源的清理，等等。NodeManager中一个container的生命周期被定义在org.apache.hadoop.yarn.server.nodemanager.containermanager.container包中。  

下面是涉及到的一些枚举和类的列表：  
* Container：这是一个接口，对应于NodeManager上的container。
* ContainerImpl：这个类定义了container的状态转化和与之相关联的事件处理器。
* ContainerEventType：这是一个枚举，定义了container中不同的事件类型。
* ContainerState：这是一个枚举，定义了container中不同的状态。  

下面的状态转化图说明了NodeManager对于一个container的视角：  
![image](/Images/YARN/yarn-container-state-change.png)  

NodeManager对一个container所关注的初始状态和完成状态如下：  
* 初始状态：NEW
* 完成状态：Done  

ResourceManager分配container到一个单独的机器上。因为container是被一个应用所需要，NodeManager会在节点初始化container对象。NodeManager会请求ResourceLocalizationManager服务去下载运行container所需要的资源并且将container的状态标记为Localizing。  

对于本地化，使用的是特定的服务。辅助服务拥有container服务数据的信息。当资源被成功本地化之后，Resource_Localized事件将会被触发。如果资源成功本地化或者资源并不需要本地化，那么container将会直接进入Localized状态。如果资源本地化失败，那么container的状态就会被变为Localization Failed。如果container直接进入Done状态，那么container运行将会被跳过。  

一旦container的资源需求条件被满足，那么Container_Launched事件将会被触发并且container状态将会被改为Running。在这个过渡中，ContainerMonitor服务会被用于去监控container的资源使用情况。NodeManager会等待container的完成。如果container被成功执行，那么一个成功事件将会被调用并且container的退出状态会被标记为0。container的状态会被更改为EXITED_WITH_SUCCESS。如果container失败了，那么退出码和诊断信息将会被更新，并且container的状态会被更改为EXITED_WITH_FAILURE。  

在任务状态，如果一个container接受到了一个kill信号，那么container都会变为KILLING状态，当container被回收完成，那么container的状态会被更改为Container_CleanedUp_After_Kill。对于一个container来说，回收它使用的资源是强制性的。当资源被回收完成，Container_Resources_CleanedUp事件会被调用并且状态会被标记为Done。  

#### 视角 3 - 本地化资源  
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

下面的状态转换图说明NodeManager对资源的视角：  
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
