YARN被用于在一个多节点集群上管理资源和执行不同的应用。它允许用户提交任何类型的应用到集群上。它通过提供一个应用执行的通用实现解决了可扩展性和MapReduce框架相关的问题。  

本章的目标是让YARN用户或者开发者开发出明白应用执行流程的应用。执行流程图和图像抓拍可以帮助我们了解YARN组件在应用执行期间彼此之间如何进行通信。  

本章，我们将会涉及到：  
* 理解YARN的执行流程
* 提交一个MapReduce样本应用
* 处理YARN中的失败
* 导出container和application的日志  

### 理解应用的执行流程  
一个YARN应用可以是一个简单的shell脚本，MapReduce任务，或者是任何一组任务。本节将会讨论YARN应用的提交和执行流程。为了管理YARN上应用的执行，客户端需要定义一个ApplicationMaster，提交一个应用上下文到ResourceManager。按照每个应用的需求，ResourceManager为ApplicationMaster分配内存，为应用的执行分配container。  

应用的执行流程可以概括的分为6个阶段，如下图所示：  
![image](/Images/yarn-application-execution-flow.png)  

#### 阶段 1 - 应用的初始化和提交  
在应用执行的第一个阶段，客户端会连接到ResourceManager中的应用管理(Application Manager)服务并且向ResourceManager请求一个新的应用ID。ResourceManager会检验客户端的请求，如果客户端用户是经过授权的，那么ResourceManager会发送一个新的并且是唯一的应用ID，连同集群metrics一起给客户端。客户端将会使用这个应用ID提交一个应用到ResourceManager，正如下图所描述的一样：  
![image](/Images/client-submit-application.png)  

客户端将会把ApplicationSubmissionContext和提交请求一起发送给ResourceManager。应用提交上下文包含了关于应用的元数据信息，比如：应用所在队列，名称等等。它也包含了在一个独有的节点上启动ApplicationMaster服务的信息。应用的提交是一个需要等待应用执行完成的阻塞调用。在后台，ResourceManager服务将会接受应用并且分配containers给应用执行。  

#### 阶段 2 - 分配内存和启动ApplicationMaster  
在第二个阶段，ResourceManager将会在一台NodeManager节点上启动	一个ApplicationMaster服务。ResourceManager内部的调度(Scheduler)服务负责节点的选择。为ApplicationMaster container选择一个节点最基本的条件是该节点必须满足ApplicationMaster服务大量的内存需求。如下图所示：  
![image](/Images/yarn-start-applicationmaster.png)  

客户端提交的ApplicationSubmissionContext包含了ApplicationMaster container的LaunchContext。LaunchContext包含了ApplicationMaster的内存需求，启动ApplicationMaster的命令等信息。  

ResourceManager中的调度服务在LaunchContext分配指定的内存，并且发送context给NodeManager启动ApplicationMaster服务。  

#### 阶段 3 - ApplicationMaster的注册和资源的分配  
ApplicationMaster的container创建客户端与集群中的ResourceManager和NodeManager通信。然后ApplicationMaster会使用 AMRMClient服务向ResourceManager进行注册。它会指定ApplicationMaster container所在的主机名和端口号。当开发应用时，开发者也可以使用AMRMClientAsync，一个ResourceManager的AppMaster客户端异步实现。  

ApplicationMaster也会发送一个跟踪应用的URL。跟踪URL是一个应用程序特定的框架，用来监控应用的执行。  

ResourceManager返回与访问控制列表，集群容量和访问令牌有关的注册响应信息，如下图所示：  
![image](/Images/applicationmaster-register.png)  

ApplicationMaster请求ResourceManager在NodeManager节点上分配container执行应用的任务。请求包括工作的container期望得的容量，以内存和CPU核的形式提供并且带有应用的优先级。可选的参数包括container执行节点和机架的规格。  

ResourceManager遍历请求的container，过滤掉黑名单中的container，然后创建一组用以发布使用的container。  

#### 阶段 4 - 运行和监控container  
一旦ResourceManager给ApplicationMaster分配了需要的container，ApplicationMaster就会使用AMNMClient与NodeManager节点进行连接。ApplicationMaster会发送每个工作的container的LaunchContext到NodeManager节点。ResourceManager也可能在一台NodeManager节点上分配两个container。然后NodeManager节点会使用LaunchContext中的信息去启动container。每个container作为一个YARN的子进程在NodeManager节点运行。  

ApplicationMaster会请求正在运行的的container的当前状态。container状态请求的响应信息包含一系列最新创建的和已经完成的container的信息。整个流程正如下图所示：  
![image](/Images/launch-and-monitor-container.png)  

对于所有的containers，ResourceManager会执行下面的动作：  
* ApplicationMaster的存活检测
* 更新请求的/发布的/黑名单的container的列表
* 确认新的资源需求，分配资源并且更新集群资源度量  

#### 阶段 5 - 应用进展报告  
一个为了监控应用的应用程序特定的框架是通过那个应用的跟踪URL来提供的。YARN客户端通过跟踪URL来监控一个应用的当前状态。跟踪URL一般会包含应用的度量。比如：如果应用是一个MapReduce任务，那么跟踪URL会提供该任务的mapper和reducer的列表，如下图所示：  
![image](/Images/tracking-url.png)  

在任何时间点，YARN客户端都可能请求ResourceManager中的应用管理服务去获得应用的当前状态。ResourceManager以应用报告的形式发送应用状态给客户端。  

#### 阶段 6 - 应用完成  
一个应用完成之后，ApplicationMaster会	发送一个注销请求到ResourceManager。ApplicationMaster会终止自己的运行并且将使用的内存归还给NodeManager。对于一个应用来说，会有一个最终的结果和一个最终的状态。ResourceManager会将应用最终结果标记为FINISHED。应用的最终的状态是通过ApplicationMaster进行设置的，指的是应用已经执行过。  

YARN客户端可能在任何时间点通过发送kill请求到ResourceManager来中断应用的执行。ResourceManager会杀死那个应用正在运行的container并且将应用的结果更改为已经完成的。  
![image](/Image/application-completion.png)  

### 提交一个MapReduce样本任务  
当一个MapReduce任务提交到Hadoop-YARN集群上时，一系列事件会发生在不同的组件中。本节，我们将会提交一个Hadoop-YARN样本任务到集群上。我们将在图像抓怕的帮助下讨论应用的流程并且弄明白一系列事件是如何发生的。  

#### 提交一个应用到集群  
正如第3章 管理一个Hadoop-YARN集群中讨论的，yarn jar命令是用来提交MapReduce应用到Hadoop-YARN集群。一个作为MapReduce例子的jar包被打包在Hadoop中。它包含MapReduce的样本程序，比如word count，pi统计，模式搜索等。下图是一个展示：  
![image](/Images/submit-yarn.png)  

在上图所示中，我们提交了一个带有5和10作为参数的pi任务。第一个参数5表示了map任务的数量，第二个参数10代表了job中的每个map任务的参数样本。  
```shell
yarn jar <jarPath> <JobName> <arguments>
```  

连接到Hadoop master节点并且执行图中所示的命令。一旦任务被提交，命令会读取配置文件和创建一个到ResourceManager的连接。它使用在yarn-site.xml文件指定的ResourceManager主机名和RPC端口进行连接。ResourceManager会注册请求并且给应用提供一个唯一的ID。application ID以application_作为前缀，ResourceManager提供的ID号作为后缀。对于MapReduce任务，带有job_前缀的ID也会被创建。  

ResourceManager接受应用并且在NodeManager节点中的一个节点上启动一个 MRApplicationMaster服务用来管理应用的执行。然后ApplicationMaster会向ResourceManager进行注册。在注册成功后，一个跟踪URL会被提供用来跟踪那个应用的状态和已经执行的不同container的进展。  

#### 更新ResourceManager Web UI  
一旦应用被成功的提交并且被ResourceManager成功接收，那么集群的资源度量会被更新并且应用的进展在ResourceManager wen接口是可见的。正如下面截图所展示的：  
