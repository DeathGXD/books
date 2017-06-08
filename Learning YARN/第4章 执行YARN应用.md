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
![image](/Images/application-completion.png)  

### 提交一个MapReduce任务样本  
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
一旦应用被成功的提交并且被ResourceManager成功接收，那么集群的资源度量会被更新并且应用的进展在ResourceManager web接口是可见的。正如下面截图所展示的：  
![image](/Images/update-resourcemanager-web-ui.png)  

YARN的HTTP web接口在ResourceManager节点是可用的，通过yarn-site.xml配置文件中的yarn.resourcemanager.webapp.address属性中的指定的主机名和端口号进行访问。  

你可以从你的主节点(也就是ResourceManager所在的节点)上以URL http://localhost:8088/ 访问ResourceManager。  

另外，如果你正在使用的是一个多节点的Hadoop-YARN集群，那么使用实际的ResourceManager主机名代替localhost即可。  

YARN的初始页面包含了集群度量的一个概要和集群中所有应用的列表。它同样也包含了应用的元数据信息，比如：当前的状态，应用名称等等。web接口同样包含了与NodeManager和可用资源队列相关的信息。  

#### 清楚应用的进度  
在第一步中，当一个应用被提交到Hadoop-YARN集群上时，RunJar类会被实例化。RunJar类是一个被用做客户端和ResourceManager之间接口的java程序。如下图所示：  
![image](/Images/runjar.png)  

想要检测系统上所有运行的Java进程，可以使用jps命令。jps命令在客户端上的输出将会包含一个叫做RunJar的进程。命令的输出同样也会包含所有Java进程的ID。  

想要查看进程信息可以使用ps aux命令。它会列出节点上所有正在运行的进程。想要过滤结果，你可以使用grep命令并且带有进程ID。命令如下：  
```shell
ps aux | grep <processID>
```  

#### 跟踪应用详情  
ResourceManager的web接口在http://<RMHost>:<WebPort>或者http://<RMHost>:<WebPort>/cluster/apps/<application_id>提供了应用的具体细节。  

ResourceManager的web接口提供了关于那些提交到YARN集群上应用的通用的信息。如下图所示：  
![image](/Images/application-information.png)  

对于每个应用，在同一个节点或者不同的节点上可能会有多个尝试(attempt)。应用的细节页面包含了用于应用执行的每个尝试的列表。并且提供了应用生成的日志文件的连接。  

#### ApplicationMaster进程  
ApplicationMaster是一个应用的第一个container。每个应用框架都会有一个预先定义的ApplicationMaster去管理应用的执行。为了管理MapReduce应用的执行，Hadoop绑定了MRAppMaster服务。正如下面的截图所示：  
![image](/Images/mrappmaster.png)  

MRApplicationMaster对于MapReduce应用来说是作为一个Java进程运行。进程的名字叫做MRAppMaster。类似于RunJar进程，你可以在正在运行MRApplicationMaster服务的节点上执行jps或者ps aux命令来查看MRApplicationMaster服务。  

#### 集群节点信息  
ResourceManager的web接口在http://<RMHost>:<WebPort>/cluster/nodes提供了节点列表信息。如下图所示：  
![image](/Images/yarn-node-information.png)  

它提供了NodeManager节点的列表。节点的元数据，包括机架名称，当前状态，RPC和HTTP地址，节点容量。它同样也会提供类似于应用列表页面上可用度量的集群度量信息。你可能发现节点的使用随着任务的进展会被更新。  

#### 节点的container列表  
所有的NodeManager进程提供了一个web接口用来监控运行在该节点上的container。NodeManager的web接口地址是http://<NMHost>:<WebPort>/node。NodeManager的web接口默认端口号是8042，如下图所示：  
![image](/Images/allcontainer-information.png)  

一个NodeManager节点上当前正在运行的所有container的具体细节在http://<NMHost>:<WebPort>或者http://<NMHost>:<WebPort>/node/allContainers上可以看到。  

它提供了container的当前状态和一个container生成的日志文件的连接。  

#### YARN子进程  
Container被认为是worker服务。实际上MapReduce任务会在内部执行container。在Hadoop-YARN集群中container是作为一个叫做YarnChild的Java进程运行。每个MapReduce任务(每个map任何或者每个reduce任务都会作为一个YarnChild)都将会被作为YarnChild执行并且一个节点可能会同时运行多个YarnChild进程，如下图所示：  
![image](/Images/yarnchild.png)  

类似于RunJar和MRAppMaster进程，你可以在正在运行MapReduce任务的节点上执行jps或者ps aux命令来查看YarnChild。  

#### 应用完成后的详情  
一旦应用被完成，应用的状态和最终的结果会被更新。跟踪URL同样也会获取更新后的结果到指定的应用历史服务器。我们将会在第6章 从MRv1迁移到MRv2中进一步讨论应用历史服务器。  
![image](/Images/application-completion-page.png)  

### 处理YARN中的故障  
YARN中的应用成功的执行依赖YARN中所有的组件健壮的协调，包括container，ApplicationMaster，NodeManager和ResourceManager。组件协调中任何的失败或者缺乏充足的集群资源都有可能导致应用的失败。YARN在处理应用执行的不同阶段的失败是健壮的。应用的容错和恢复依赖于执行的当前阶段哪个组件发生了问题。下面的小节中解释YARN组件层恢复的原理。  

#### Container故障  
Container被初始化用于map和reduce任务的执行。正如前面小节中提到的，container在Hadoop-YARN集群中是作为YarnChild进程的Java程序在运行。在执行中可能会有一些异常或者由于缺乏足够的资源而导致JVM异常的终止。来自于container的失败要么主动发送给ApplicationMaster，要么当ApplicationMaster在超过一段时间后都没有收到来自于container的响应时被发现(超时时间通过mapreduce.task.timeout属性设置)。在这两个情景中，task尝试都会被标记会失败。  

ApplicationMaster会在指定数量的task尝试失败后尝试重新调度或者重新执行task，这样的话，完成的task也同样会被认为失败。mapreduce.map.maxattempts用来配置map任务的最大重试次数，mapreduce.task.maxattempts用来配置reduce任务的最大重试次数。在一个Job执行期间，如果一定百分比的map任务或者reduce任务失败了，那么这个Job就会被认为是失败的。通过mapreduce.map.failures.maxpercent和mapreduce.reduce.failures.maxpercent属性分别设置mapper和reducer任务的最大失败百分比。  

ApplicationMaster管理着应用的执行和container的运行。一个应用总是只有一个ApplicationMaster实例在运行。ApplicationMaster会定期的发送心跳信息给ResourceManager。万一ApplicationMaster失败了，那么在特定时间间隔内ResourceManager将不会收到来自于ApplicationMaster的任何心跳信息，那么将会认为ApplicationMaster发生了故障。你也可以通过yarn-site.xml文件中的yarn.am.liveness-monitor.expiry-interval-ms属性来配置ApplicationMaster向ResourceManager发送报告的超时间隔。  


一个应用可能会有多个尝试失败。如果ApplicationMaster在应用执行中间失败了，那么任务的尝试也会被标记为失败。应用执行的attempt最大重试次数使用yarn.resourcemanager.am.max-retries属性进行配置。默认情况下，它的值是2并且这个属性是一个全局设置，集群中的所有的ApplicationMaster公用这一属性。2这个值被认为是对于集群来说是定义的最大值。一个ApplicationMaster可以指定它的最大重试次数，但是独立设置的次数不能大于指定的全局的上限。  

#### NodeManager故障  
NodeManager进程会持续的定期发送活着的心跳信息给ResourceManager。ResourceManager维护一系列活跃的NodeManager节点列表。如果一个NodeManager节点发生了故障，ResourceManager会在指定的时间间隔内无法等到来自于NodeManager的心跳信息。ResourceManager等待的时间间隔通过yarn.resourcemanager.nm.liveness-monitor.expiry-interval-ms属性以毫秒值得形式进行设置。那么ResourceManager会从活跃的NodeManager节点列表中将删除故障的NodeManager节点信息并将移除节点标记为Lost节点。  

MRApplicationMaster可以将一个NodeManager节点加入黑名单。如果一个task在一个节点上失败了多次，那么ApplicationMaster将会将该节点加入黑名单。你可以使用mapreduce.job.maxtaskfailures.per.tracker属性设置一个节点所被允许最大重试次数。默认值为3，意味着如果超过3个task在一个NodeManager节点失败了，那么ApplicationMaster将会将该NodeManager节点加入黑名单，并且会在一个不同的NodeManager节点上重新调度task。  

#### ResourceManager故障  
在Hadoop 2.4.1版本之前，Hadoop-YARN集群中的ResourceManager存在单点故障。随着Hadoop 2.4.1版本开始，为了实现ResourceManager的高可用，手动和自动故障转移控制实现了。关于高可用的详细说明和实现在第3章 管理一个Hadoop-YARN集群中被提到。管理员同样可以启用ResourceManager恢复机制以达到处理故障的情况。默认情况下，恢复机制是被关闭的，你可以通过设置yarn.resourcemanager.recovery.enabled属性值为true来启用恢复机制。  

如果恢复机制被启用，那么你需要配置一个状态存储机制来存储ResourceManager的信息。想要阅读更多有用的有关状态存储机制的信息，你可以参考ResourceManager高可用或者apache文档http://hadoop.apache.org/docs/r2.5.1/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html。  

### YARN应用日志  
随着YARN集群上应用的执行，不同的组件会生成很多运行的日志。这些日志可以概括的分为下面几类：  

#### 服务日志  
ResourceManager和NodeManager进程在集群中是24x7小时运行。这些服务持续追踪集群的运行和其他程序之间的协调，比如：ApplicationMaster和container。YARN服务的日志被创建在Hadoop的日志目录HADOOP_PREFIX目录下。你可以参考之前的章节中的管理服务日志小节。  

#### 应用日志  
ApplicationMaster，和container一样运行在集群中，并且生成应用日志。日志可以用来调试和分析应用。默认情况下，应用生成的日志被放在Hadoop安装文件夹下的日志目录下的user_logs目录。你可以使用下面的属性来配置container日志目录的地址：  
```xml
  <property>
    <name>yarn.nodemanager.log-dirs</name>
    <value>/home/hduser/hadoop-2.5.1/logs/yarn</value>
  <property>
```  
运行在YARN集群上的应用都被提供了一个唯一的ID。Container的ID是由应用ID通过追加container编号派生出来的。一个应用的日志存放在以应用ID作为名字的目录中，该目录中又包含了以container ID作为名字的目录，用来存放container产生的日志。  

每个container都会产生下面三种类型的日志文件：  
* stderr：这个文件包含的是每个独有的container在执行期间发生的错误
* syslog：这个文件包含的是配置好的日志级别的信息。默认的日志是INFO，WARN，ERROR和FATAL类型的日志信息。
* stdout：这个文件包含的是在每个container执行过程中打印的信息。  

下面的截图中展示了三种日志：  
![image](/Images/three-type-logs.png)  

总结
