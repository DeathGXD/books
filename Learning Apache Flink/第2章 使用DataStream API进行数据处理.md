实时分析是当前非常重要的一个问题。许多不同的场景需要实时的处理数据。到目前为止，有多种技术尝试提供了这种能力，比如：Storm和Spark已经出现很长一段时间了。  

物联网的应用程序需要数据被存储，被处理并且实时的或者近实时的被分析。为了满足这样的需求，Flink提供了一个流数据处理的API，叫做DataStream。  

本章，我们将会讨论DataStream API相关的一些细节，主要涉及下面的一些主题：  
* 执行环境
* 数据源
* 转换
* Data sink
* 连接器
* 案例——传感器数据分析  

任何Flink程序都适用于如下特定的定义结构：  
![image](/Images/Flink/flink-datastream-data-flow.png)  

我们将会涉及到每一个步骤，并且如何使用DataStream API实现这些结构。  

### 执行环境  
为了开始编写一个Flink程序，我们首先需要获得一个已经存在的执行环境，或者自己创建一个。取决于你想做什么，Flink支持：  
* 获取一个已经存在的Flink环境
* 创建一个本地环境
* 创建一个远程环境  

通常，你只需要使用getExectionEnvironment()方法即可。这个方法将会基于你的上下文做正确的事。如果你打算在一个本地的IDE中执行，那么你需要启动一个本地的执行环境。除此之外，你还可以执行JAR文件，那么Flink集群管理器将会在分布式环境下执行Flink程序。  

如果你想要在自己的程序中创建一个本地或者远程的执行环境，那么你也可以选择通过使用createLocalEnvironment()和createRemoteEnvironment(String host, int port, String... jarFiles)。  

### 数据源  
数据源是Flink程序期望获取数据的地方。这是Flink程序中的第二步。Flink支持很多预实现的数据源函数。同时也支持编写自定义的数据源函数，因为任何不支持的数据源都可以编程简单的实现。首先让我们弄明白Flink内置的数据源函数。  

#### 基于网络  
DataStream API支持从一个socket读取数据。你仅仅只要指定读取数据的主机和端口即可：socketTextStream(hostName, port);你可以指定数据的分隔符：socketTextStream(hostName,port,delimiter)；你也可以指定每次获取数据的最大数量：socketTextStream(hostName,port,delimiter, maxRetry)。  

#### 基于文件  
你也可以使用Flink中的基于文件的数据源函数从文件中读取数据。你可以使用readTextFile(String path)从特定路径中读取文件中的数据。默认情况下，它读取的是TextInputFormat格式，并且一行一行的读取字符串。  

如果文件格式是其它文本，你可以使用函数进行指定：readFile(FileInputFormat<Out> inputFormat, String path)。Flink同样支持读取一个文件流进行处理，使用readFileStream()函数：readFileStream(String filePath, long intervalMillis, FileMonitoringFunction.WatchType watchType)。  
