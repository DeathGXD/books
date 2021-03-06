YARN包含了多种高效和可扩展的组件, 使得YARN变成一个强大的、健壮的、优秀的资源管理框架。为了对YARN的资源提供有一个更好的理解和如何与YARN进行整合, 使用者需要对YARN的各种组件有更深入的认识。本章讲解了YARN组件的内部细节, 相关的类和YARN组件之间如何相互影响。在本章中将会洞察YARN的组件, 这样能够让使用者更好的利用YARN的特性并且让YARN与分布式应用框架更容易集成。  

在本章中, 我们将会涉及到下面的主题：
* 理解ResourceManager
* 理解NodeManager
* 与辅助服务、资源本地化、日志聚合协同工作
* Timeline服务预览、web application proxy和YARN调度负载模拟器  

### 理解ResourceManager  
ResourceManager是YARN框架的核心组件, 负责管理一个多节点集群的资源。它促使资源的分配和记录YARN集群上跨多个节点运行的分布式应用。它与每个节点上运行
的NodeManager进程和一个叫做ApplicationMaster的应用服务共同工作。他管理着整个集群的资源和执行YARN应用。  

ResourceManager拥有多个子组件协助它有效的管理一个多节点的集群，并且集群上并行运行着成千上万的分布式的，资源密集的并且有时限的应用。
下图展示了具体的情形：  
![image](/Images/YARN/yarn-deep-components.PNG)  

#### 客户端和管理接口
ResourceManager暴露方法给client和集群管理员，用来跟ResourceManager进行RPC通信和按照优先级接受管理命令。这里是两个用来跟ResourceManager进行通信的类：
1. **ClientRMService**  
    ClientRMService类是ResourceManager的客户端接口。所有的客户端用来创建与ResouceManager的RPC连接。这个模块处理所有的ResouceManager的RPC接口。这个服务的实现被定义在org.apache.hadoop.yarn.server.resourcemanager.ClientRMService包中。客户端初始化这个服务使用客户端配置文件，比如yarn-site.xml。  

    客户端请求ResourceManager：  
    * **应用请求**：这个接口暴露了诸如创建新的application请求，提交applications到集群，杀死一个application，列出containers，应用attempt记录等服务给客户端。  
    * **集群度量**：客户端也可以使用这个服务请求ResourceManager分享集群的度量，节点的容量，调度的细节等等信息。  
    * **安全**：为了连接到一个安全的集群环境，客户端需要使用ResourceManager提供的授权令牌和访问控制列表。  

    提示：想要阅读更多有关定义在ClientRMService的不同方法，你可以参考grepcode的网站地http://grepcode.com/file/repo1.maven.org/maven2/org.apache.hadoop/hadoop-yarnserver-resourcemanager/2.6.0/org/apache/hadoop/yarn/server/resourcemanager/ClientRMService.java  

2. **AdminService**  
    AdminService类被集群管理员用来管理ResourceManager服务。集群管理员在使用命令行选项rmadmin命令的时候，内部使用的就是AdminService。  

    下面列出了集群管理员通过AdminService可以执行的一些操作：  
    * 刷新集群的节点、访问控制列表和队列  
    * 检查集群的健康状态  
    * 管理ResourceManager的高可用  

  这个服务的实现被定义在org.apache.hadoop.yarn.server.resourcemanager.AdminService包中。  

 你可以参看rmadmin命令，在第3章 管理一个YARN集群。  

#### 核心接口
ResourceManager的核心组件包含调度器和应用的管理。下面的类中定义了ResourceManager如何执行任务的调度、应用的管理和状态信息的管理。  
1. YarnScheduler  
    YarnScheduler类负责在多个应用之间分配资源和回收资源，那意味着集群中跨多个节点的应用调度是基于一些预先定义的说明。YarnScheduler是一个基于可插拔策略的插件。这个插件负责在多个应用之间，多个队列之间，等等，进行集群资源(CPU，内存，磁盘等等)的分割。它为被调起的应用维护了一个队列，并且有集群资源所具有的信息，比如，集群中节点数，最大和最小的资源容量，等等。YarnScheduler定义在org.apache.hadoop.yarn.server.resourcemanager.scheduler.YarnScheduler接口中，YarnScheduler支持MapReduce的两个可用的实现是：  
    * FairScheduler
    * CapacityScheduler  
    关于调度器和调度器配置更详尽的说明在第10章 YARN应用的调度。  

2. RMAppManager  
    RMAppManager负责为ResourceManager管理在YARN集群上执行的应用列表。它是作为ResourceManager内部的一个服务运行。它创建并且记录ApplicationSummary，那是与每个应用相关的运行时信息。YARN客户端连接到这个服务可以请求任何与应用相关的信息。  

    这个服务的实现定义在org.apache.hadoop.yarn.server.resourcemanager.RMAppManager类中。  

3. RMStateStore  
    处理ResourceManager在故障期间的恢复，RMStateStore是一个抽象实现，用来存储ResourceManager服务的状态信息。它也存储与正在运行的应用相关的信息和它们的attempt。  

    当前，YARN定义了四种存储ResourceManager状态的机制：  
    * FileSystemRMStateStore
    * MemoryRMStateStore
    * ZKRMStateStore
    * NullRMStateStore  

    ZKRMStateStore是最可靠的和被推荐的存储RMState的机制，但是它需要一个Zookeeper作为Znode共同存储信息。ResourceManager的状态存储机制在配置ResourceManager高可用(HA)时同样也会被用到。更多状态存储相关的内容，你可以参考在第3章，管理一个Hadoop-YARN集群中的ResourceManager高可用中的状态存储配置。  

4. SchedulingMonitor  
    这个接口提供了对container的监控和编辑一个定期调度的规定。它同样提供了一个调整资源监控监控的规定和定义SchedulingEditPolicy的规定。想要阅读更多有关SchedulingMonitor的内容，你可以org.apache.hadoop.yarn.server.resourcemanager.monitor.SchedulingMonitor类。  

#### NodeManager接口
ResourceManager与NodeManager进行通信。NodeManager会定期向ResourceManager汇报NodeManager节点的健康和资源信息。下面是一些ResourceManager用来管理集群中所有NodeManager节点的类：  
1. NMLivelinessMonitor  
    NMLivelinessMonitor帮助ResourceManager持续追踪集群中所有活着的NodeManager节点，更重要的是，它是系统中最与众不同的。它频繁从所有的NodeManager节点接受心跳信息，如果在600000毫秒或者10分钟(默认)内没有收到一个节点的任何心跳信息，那么它会认为这个节点宕机了。这个时间间隔可以通过YARN配置属性中的RM_NM_EXPIRY_INTERVAL_MS进行配置。当一个节点被标记为宕机时，所有正在运行在那个节点上的container都会被认为是失败的并且不会再有新的container被调用在那台节点上。ResourceManager会为那台宕机节点上运行的container重新分配资源。  

    想要覆盖逾期时间间隔的默认值，管理员可以在yarn-site.xml文件配置下面的属性：  
    ```xml
    <property>
      <name>yarn.am.liveness-monitor.expiry-interval-ms</name>
      <value>600000</value>
    </property>
    ```  

    这个服务的实现被定义在org.apache.hadoop.yarn.server.resourcemanager.NMLivelinessMonitor类中。  

2. ApplicationMasterLauncher  
    ApplicationMasterLauncher为提交到YARN集群上的不同应用维护了一个application master的队列。它启动一个AM作为服务，每次都会从队列中获得一个应用。它同样维护了一个启动AM的线程池，并且当应用完成时或者被强制终止时回收AM。这个服务被定义在org.apache.hadoop.yarn.server.resourcemanager.amlauncher.ApplicationMasterLauncher类中。  

#### 安全和令牌管理  
ResourceManager管理着一组令牌，用以在不同RPC通信管道之间的验证和授权。ResourceManager管理这些服务在接下来的安全机制章节中会进行解释。  

1. RMAuthenticationHandler  
    RMAuthenticationHandler的职责是基于授权头验证一个请求并且给请求返回一个有效的令牌。它继承自KerberosAuthenticationHandler类，并且当Kerheros安全机制开启时被使用。  

    Kerberos是一个用于鉴定正在运行的服务的标识符的认证协议并且在不同的节点之间的不安全网络中进行通信。Kerberos的概述以及YARN中Kerberos的配置将会在第11章 启用YARN安全机制中被提及。想要阅读更多有关这个服务的内容，你可以参考org.apache.hadoop.yarn.server.resourcemanager.security.RMAuthenticationHandler包。  

2. QueueACLsManager  
    YarnScheduler使用QueueACLsManager类检测是否一个用户访问了一个特定的队列。如果ACLs没有被开启，那么将会允许所有用户提交应用到所有队列。关于队列和队列ACLs的更深入的说明在第10章 YARN应用的调度和第11章 YARN中启用安全机制中被给出。  

3. TokenSecretManagers for RM  
    ResourceManager定义TokenSecretManagers为了管理跨多个应用，container和节点之间的安全。  

    几个TokenSecretManagers是：  
    * AMRMTokenSecretManager
    * ClientToAMTokenSecretManagerInRM
    * NMTokenSecretManagerInRM
    * RMContainerTokenSecretManager
    * RMDelegationTokenSecretManager  

### 理解NodeManager  
NodeManager节点是YARN的工作节点，负责向ResourceManager更新一个节点上的可用资源。它同样也负责健康一个节点的健康状况和为一个应用执行container。下面展示了NodeManager进程中包含的多个子组件，接下来会对这些子组件进行详细的描述：  
![image](/Images/YARN/yarn-nm-component.png)  

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
周期性的检测集群中不同节点的健康状况是非常重要的，并且如果发现有任何节点是不健康的，应该采取必要的措施。NodeManager服务提供了工具可以在任何时间点去管理和监控节点的状态和健康情况。作为失败/故障管理和恢复的一部分，即使一个不健康的节点出现了网络中断或者直接故障了，NodeManager也会提供服务恢复该节点的资源状态并且再次积极地参与集群的运作。  

1. NodeHealthCheckerService  
    这个服务提供了NodeManager节点上当前健康状态的报告。它会在节点执行一个用户定义的脚本使用NodeHealthScriptRunner服务去监控节点的健康情况。  

    脚本执行结束后会带有下面任意一种结果：  
    * SUCCESS
    * TIMED_OUT
    * FAILED_WITH_EXIT_CODE
    * FAILED_WITH_EXCEPTION
    * FAILED  

    在内部，这个服务仅仅会检测脚本的输出中是否以ERROR开始。如果在脚本执行期间，服务发现任何匹配的错误或者超时，那么它会将节点标记为不健康的并且向请求报告信息的服务发现报告。  

2. NMStateStoreService  
    NodeManager的职责是供应本地的container资源。在YARN中，提供资源的container的被称之为资源本地化。NMStateStoreService服务存储了任意时间点上NodeManager节点中本地化资源和正在使用的资源的状态。它也提供了一个用户资源的恢复状态和本地化状体的处理。  

#### Container管理  
NodeManager为了满足运行一个container和container监控的先决条件实现了不同的服务。比如，一个container为了自己的执行可能需要下载额外的资源或者需要任何辅助的服务。在接下来的小节中对NodeManager为了container的管理管理的各种不同的服务提供了一个更详细的说明：  

1. ContainerExecutor  
    这是一个NodeManager的提供的一个接口，负责满足container运行的先决条件，包括，资源的本地化，container目录的创建(用户和应用指定的目录和缓存)和最终执行请求的container。它同样可以方便的杀死一个container，检查一个container是否存活和发送信号给一个container。  

2. ResourceLocalizationService  
    NodeManager实例化ResourceLocalizationService是为了确保container运行应用的任务所需要的资源的位置。ResourceLocalizationService会将资源下载到一个应用运行所对应的NodeManager的本地文件系统中。Container使用这些资源用于应用的执行。当container执行完成后，ResourceLocalizationService会将下载的资源从磁盘中清理掉。  

3. ContainersLauncher  
    ContainerLauncher服务负责在节点上启动container。这个服务只能够在ResourceLocalizationService在本地文件系统创建目录和下载任何container所需要的资源之后才能启动。这个服务一个接着一个启动container。它接受下面两个其中之一的ContainersLauncherEvent：  
    * LAUNCH_CONTAINER：如果事件类型是launch_container，那么ContainerLauncher服务会使用ExecuterService启动container。
    * CLEANUP_CONTAINER：如果事件类型是cleanup_container，那么ContainerLauncher服务会发送信号去杀死container进程并且回收container在本地磁盘的文件目录。  

4. ContainersMonitor  
    ContainerMonitor是一个监控container在节点上运行的服务。它维护了一个NodeManager需要监控的container的列表。当一个新的container在节点上启动时，ContainerMonitor会添加ContainerId和ProcessTreeInfo到列表中并且从列表中删除已经完成的container。  

5. 辅助服务(Auxiliary service)  
    辅助服务是一个用来定义在YARN集群上的每个节点运行应用所需要的用户自定义服务的框架。举一个Hadoop MapReduce任务中的一个例子，map阶段的输出被传输到reduce节点。NodeManager提供了一个处理方法为MapReduce服务检索元数据。元数据可能是在MapReduce任务执行期间，mapper和reducer之间传输map输出文件的连接信息。Hadoop提供了mapreduce_shuffle作为NodeManager的辅助服务，如下图所示：  
    ![image](/Images/YARN/yarn-auxiliary-service.png)  

6. LogHandler和日志聚合  
    LogHandler是一个记录应用日志和container日志的可插拔实现。非聚集日志是在YARN上执行的应用生成在本地文件系统的日志。这些日志将会在配置的保存时间后被删除，默认是3小时，可以通过NM_LOG_RETAIN_SECONDS属性进行设置。  

    YARN提供了一个选项去聚集一个应用在不同NodeManager节点上生成的container日志到一个集中的地方。LogAggregationService作为NodeManager的ContainerManager组件实现的一部分，为运行在YARN集群上的每个应用运行聚合器，将NodeManager节点上的日志推到HDFS上。  

#### 安全和令牌管理器  
NodeManager管理着下面的安全服务：  
* NMTokenSecretManagerInNM：这个服务管理着身份验证的key和NodeManager节点和准备运行在NodeManager节点上的应用之间的验证。
* NMContainerTokenSecretManager：这个服务管理和验证NodeManager节点上运行的container。它会检查container中的key在ResourceManager中是否有相同的key；在验证成功之后，它会允许container运行在NodeManager节点上。  

### YARN Timeline服务  
它会持续保存在YARN集群上当前正在执行的和历史的应用的信息。它会执行下面两个重要的任务：  
* 关于已经完成的引用的一般信息。它为一个应用提供了下面的一些信息：  
    * 队列名
    * 用户信息
    * 应用attempts
    * 为每个应用attempts运行的container
    * Containers  
* 每个正在运行的框架信息和已经完成的应用信息：  
    * 应用或者特定框架的信息，比如一个MapReduce应用的map任务和reduce任务的数量。  
    * 用户来自于客户端或者application master的发布信息  

想要配置和启动一个YARN TimeLine服务，你可以参考第3章， 管理一个Hadoop-YARN集群。  

### Web application proxy服务  
YARN中引入Web application proxy是为了减少通过YARN进行网络攻击的可能性。默认情况下，它是作为ResourceManager的一部分运行，但是管理员可以通过设置yarn-site.xml中的属性进行配置。  

YARN中，Application Master负责提供一个Web UI和连接到ResourceManager的链接信息。这导致服务通信容易受到某些常见的攻击。web applicatiion proxy通过警告不拥有给定应用程序的用户，他们将连接到一个不可信的站点的方式缓和了风险。  

### YARN调度负载模拟器  
SLS是一个工具，用来在一台机器上模拟出相当于大规模YARN集群的负载。它帮助研究人员和开发人员去体验新调度器的特性，并预知大规模集群的性能表现。集群的大小和应用负载的大小可以从配置文件中进行配置。模拟器将产生近实时的度量：  
* 整个集群和每个队列的资源使用量
* 详细的应用执行追踪，用于分析调度器在吞吐量，公平性，job的转变时间等方面的表现
* 调度器算法的核心度量，比如每个调度器操作的时间  

### YARN中的资源本地化  
资源指的是容器执行分配任务时所需的任何东西。因为container被NodeManager运行和管理在不同的节点上，NodeManager要负责确认请求的资源在每个节点都可用。YARN通过提供ResourceLocalizationService服务促使了NodeManager的这一特性。这个服务负责下载本地的应用资源到NodeManager所在节点的文件系统并且确保对于container运行那个应用是可用的。  

#### 资源本地化术语  
在本节，我们将会讨论一些YARN中与资源本地化相关的术语。为了配置和使用资源本地化，明白下面的概念的非常重要的：  
* LocalResource：它被定义为container用于执行应用所需要的资源。NodeManager负责在运行container前确保本地文件系统中的资源可用。LocalResource拥有下面的属性：  
    * URL：可用的并且可以下载的资源的位置
    * LocalResourceType：它定义了被NodeManager所下载的资源的类型。它是下列之一：  
    ARCHIVE：NodeManager自动解压缩归档资源  
    FILE  
    PATTERN：包含了部分归档文件和正常文件  
    * Size：这是需要下载的资源的大小
    * Timestamp：这是下载资源的创建时间戳。被用于验证。
    * LocalResourceVisibility：指定了需要下载到NodeManager节点本地文件系统上的资源的可见性。它被定义为：  
    PUBLIC：被节点上多有的用户共享  
    PRIVATE：被同一个用户的所有应用共享  
    APPLICATION：被一个独立应用中的所有运行的container所共享  
* ResourceState：代表了任意时间点中资源的状态。有效的状态如下：  
    * INIT
    * DOWNLOADING
    * LOCALIZED
    * FAILED  
* LocalizerTracker：这是ResourceLocalizationService的一个子组件。它实现了ContainerLocalizer的生成。它生成和跟踪私有的和公有的localizer。
* PublicLocalizer：作为特定的线程启动，并且下载public资源到NodeManager所在节点的本地文件系统。在同一个时间点，如果多个container请求同一份资源的情况下，通过检查资源的状态是否违反下载状态确保下载的安全。
* LocalizerRunner：作为特定的程序启动，并运行ContainerLocalizer带有用户访问证书。  

#### 本地化资源的目录结构  
对于本地化资源，YARN创建并且使用Hadoop默认的日志目录下面的用户目录。用户可以指定一个目录用来下载和存储资源，正如下图所展示的一样：  
![image](/Images/YARN/yarn-resource-localization.png)  

ResourceLocalizationService要么使用默认的本地目录要么使用自定义的本地目录，并且创建子目录用于缓存和存储资源。依赖于资源的可见性，在接下来的部分中，将对目录进行分类：  
1. Public  
    带有public可见性的资源，所有用户之间共享。所有的资源被放置在NodeManager节点本地目录中的一个单独的目录，也就是<yarn.nodemanager.local-dirs>/filecache，如下图所示：  
    ![image](/Images/YARN/yarn-resource-localization-public.png)  
2. Private  
    带有private可见性的资源，用户的所有应用之间共享。所有资源存储在用户指定的缓存目录，也就是<yarn.nodemanager.local-dirs>/usercache/<username>/filecache，如下图所示：  
    ![image](/Images/YARN/yarn-resource-localization-private.png)  
3. Application  
    带有application可见性的资源，应用的所有container之间共享，被存储应用指定的缓存目录，也就是<yarn.nodemanager.local-dirs>/usercache/<username>/appcache/<appId>/，如下图所示：  
    ![image](/Images/YARN/yarn-resource-localization-application.png)  

### 总结
