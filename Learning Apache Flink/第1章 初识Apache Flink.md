随着分布式技术的持续发展，工程师一直尝试着突破那些技术的极限。早期，人们寻找着更快，成本更低的方式去处理数据。当Hadoop被引进时，满足了当时人们的需要。每个人都开始使用Hadoop，开始使用基于Hadoop的生态系统工具替代他们的ETL工具。现在，使用Hadoop的方式让人们感到满意，并且在很多公司，Hadoop也越来越多的被用在生产环境。  






### 架构  
Flink 1.x的架构包含了各种组件，比如：得票；deploy，核心编程和API等。我们可以很容易比较最近的架构和Stratosphere的架构，并且看到它的变化。下面的图展示了组件，API和库：  
![image](/Images/Flink/flink-architecture.png)  

Flink有一个分层的架构，每一个组件都是特定层级的一部分。每一层都是建立在其他明确的抽象之上。Flink被设计成可以运行在本地机器上，运行在YARN集群上和运行在云上。Runtime是Flink核心的数据处理引擎，它接受通过API以JobGraph形式的程序。JobGraph是一个简单的，并行的数据流控(data flow)，带有一组生产和消费数据流(data stream)的tasks。DataStream和DataSet是API接口，程序员可以用它们定义Job。当程序编译完成，JobGraph就通过这些API产生。一旦被编译，DataSet API允许优化器生成最优的执行计划，而DataStream API使用一个流构造器实现更高效的执行的计划。  

优化后的JobGraph之后会根据部署模式被提交到executors上。你可以选择一个local，remote，或者YARN部署模式。如果你已经有一个Hadoop集群正在运行，选择YARN部署模式也许会更好。  





### 分布式执行  
Flink分布式执行包含两个重要的进程，master和worker。当一个Flink程序被执行，多个进程会参与到执行中，就是Job Manager，Task Manager和Job Client。  

下图展示了Flink程序的执行：  
![image](/Images/Flink/flink-program-execution.png)  

Flink程序需要被提交到一个JobClient，然后JobClient会提交Job到JobManager。JobManager的职责是协调资源的分配和任务的执行，当然它做的第一件事是给任务分配需要的资源。一旦资源被分配完成，task会被提交到各自的TaskManager上。在接受到task后，TaskManager会启动一个线程去执行。当task在合适的地方执行后，TaskManger会持续向JobManager报告task状态的改变。task可以有多种状态，比如：开始执行，正在执行，或者任务完成。一旦Job执行完成，执行结果会被发送回client。  

#### Job Manager  
master进程，也被叫做JobManager，协调和管理程序的执行。它的主要职责包括调度任务，管理checkpoints，失败恢复，等等。  

可以有多个Master并行的运行，分担这些职责，这个有助于实现高可用性。master其中之一需要成为leader，如果leader节点挂掉，standby master节点将会被选举成为新的leader。  

JobManager包含下面重要的组件：  
* Actor system
* Scheduler
* Check pointing  

Flink内部使用Akka actor system用作JobManager和TaskManager之间的通信。  

##### Actor system  
Actor system是一个带有多种规则的actor的容器(container)。它对外提供服务，比如：scheduing，configuration，logging，等等。它也有一个actor线程池，所有的actor都从那里创建。所有的actor都属于同一层次。每个新创建的actor将会被分配一个父actor。Actor之间通过一个消息系统相互通信。每一个actor都有一个属于它自己的邮箱，actor从邮箱中读取消息。如果actor是本地模式(local)，那么消息将会通过共享内存被共享，但是，如果actor是远程模式(remote),那么消息将会通过RPC调用被传递。每个父actor负责监管它的子actor。如果子actor有任何错误发生，父actor将会得到通知。如果父actor可以解决子actor发生的问题，那么它会重启子actor。如果父actor无法解决子actor发生的问题，那么父actor会升级问题到它的父actor。  
![image](/Images/Flink/flink-akka-actor-system.png)  

在Flink中，actor是一个带有状态和行为的容器。actor线程会顺序地持续地处理它收到的在它的邮箱中的消息。actor的状态和行为由它收到的消息决定。  

##### Schedulering  
在Flink中，executors被定义为task槽(slot)。每个TaskManager需要去管理一个活着更多的task槽。在内部，Flink决定哪个task需要共享的槽，哪个task必须被放入到特定的槽。它通过SlotSharingGroup和CoLocationGroup定义。  

##### Check pointing  
Check pointing是Flink提供一致性容错的核心。它持续不断地获得分布式数据流和executor状态的一致性快照。它的灵感来自于Chandy-Lamport算法，但是因为Flink特有的需求已经被修改了。关于Chandy-Lamport算法的细节可以在 http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf 中找到。  

关于快照精确的实现细节提供在下面的研究论文中:Lightweight Asynchronous Snapshots for Distributed Dataflows(http://arxiv.org/abs/1506.08603)。  

Flink容错的原理是持续不断地创建data flow的轻量级快照。因此它们可以持续地没有任何显著超负载地工作。通常Flink中的data flow的状态被隐藏在一个可配置的地方，比如：HDFS。  

万一有任何故障，Flink会停止executors，重置它们，并且从最新的可用的checkpoint中从新开始执行。Stream的界线(barrier)是Flink快照的核心元素。它们被加入到data streams，但是没有影响data flow。Barriers数绝对不会超过记录数records。它们将records分组到快照中。每个barrier都携带着唯一的ID。下图展示了barriers如何被注入到data stream中进行快照。  
![image](/Images/Flink/flink-check-pointing.png)  

每个快照都会向JobManager的快照协调器报告状态信息。当开始绘制快照时，Flink会处理records的校准，为了避免因为任何失败而重新处理相同的records。校准通常会消耗一些时间，但是毫秒级别的。但是对于一些高密集应用，即使是毫秒也是不可被接受的。我们有一个选项来选择低延迟而不是只对记录处理一次。默认情况下，Flink对每个record只处理一次。如果有任何应用对低延迟，并且要求对record至少处理一次，我们可以切换到触发器。这样将会跳过record校准，并且增加延迟。  

#### Task manager  
TaskManager是worker节点，在一个或者多个JVM线程中执行任务。任务的并行度由每个TaskManager上的可用的task槽决定的。每个task代表了一组被分配给task槽的资源。比如：如果一个TaskManager有4个槽，那么它会分配25%的内存给每个槽。那么就可以有一个或者更多的线程在一个task槽中运行。在相同槽中的线程共享相同的JVM。在相同JVM中的Task共享TCP连接和心跳信息：  
![image](/Images/Flink/flink-task-manager.png)  

#### Job client  
JobClient不是Flink程序内部执行的一部分，但是它是任务开始执行的起点。JobClient负责从用户那里接受程序，创建一个数据流程并且提交数据流程到JobManager为了以后的执行。一旦Job执行完成，JobClient会提供返回的结果给用户。数据流程是执行计划。参考一个非常简单的wordcount例子：  
![image](/Images/Flink/flink-wordcount-example.png)  

当一个client从用户那里接受到程序后，然后它会将程序transformation成一个data flow。对于上述程序的data flow可能看起来像这样：  
![image](/Images/Flink/flink-streaming-dataflow.png)  

在之前的图中展示了一个程序如何被transform成一个data flow。Flink的data flow默认是并行的，分布式的。对于并行的数据处理，Flink分割处理(Operator)和流(Stream)。Operator的分割又被称为sub-task。Stream可以使用一种一对一的方式或者一种重分布的方式分发数据。  

Data flow直接从source到map操作，因为不需要shuffle数据。但是对于GroupBy操作，Flink可能需要通过key重分发数据，为了得到正确的结果：  
![image](/Images/Flink/flink-parallel-streaming-dataflow.png)  

### 特点  

#### 高性能  



#### 只执行一次的状态计算  



#### 灵活的流窗口  



#### 容错  



#### 内存管理  


#### 优化器  



#### 流处理和批处理在同一平台  



#### 库  




#### 事件时间语义  




### 快速入门  




### 集群设置  
