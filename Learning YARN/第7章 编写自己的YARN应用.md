在第一章，我们讨论过了Hadoop 1.x框架的弊端。Hadoop 1.x仅仅局限于MapReduce编程。你不得不为数据的处理逻辑编写map任何和reduce任务。随着对Hadoop 2.x版本中YARN的介绍，现在你可以在HDFS上的数据执行不同的数据处理算法。YARN将资源管理和数据处理框架分割成两个不同的组件：ResourceManager和ApplicationMaster。  

在前几个章节中，我们已经学习到关于应用的执行流程，YARN组件之间如何进行通信和应用声明周期的管理。你在YARN集群上执行的一个MapReduce应用是与MRApplicationMaster组件一同工作的。在本章中，你将会学习到如何使用YARN Java APIs创建我们自己的YARN应用。学习本章知识要求你必须具备一定的Java背景和基本的Eclipse IDE知识。本章适合那些想要在YARN集群上创建并且执行应用的开发人员和开源软件贡献者。  

在本章中，我们将涉及到下面的主题：  
* 介绍YARN API
* 核心的概念和相关  的类
* 编写自己的YARN应用
* 在Hadoop-YARN集群上执行应用  

### YARN API介绍  
YARN是一个与Hadoop捆绑打包在一起的Java框架。它提供资源管理，以及简单的与存储在HDFS上的数据进行数据处理算法和访问算法的整合。Apache Storm、Giraph和HAMA是几个使用YARN进行资源管理的数据处理框架。详细的与YARN进行整合的技术在第12章，使用YARN进行实时数据分析中介绍。  

Hadoop-YARN API被定义在org.apache.hadoop.yarn.api包中。当编写你自己的YARN应用时，你将会使用到YARN API中的一些类。在继续之前，列举一些被使用到的类和明白它们的角色是至关重要。本节将会涉及到一些重要的定义在org.apache.hadoop.yarn.api包中的类。  

#### YARNConfiguration  
YARNConfiguration类定义在org.apache.hadoop.yarn.conf包中，它继承自org.apache.hadoop.conf.Configuration类。与Configuration类相似，它读取YARN配置文件(yarn-default.xml和yarn-site.xml)和提供访问Hadoop-YARNHadoop-YARN配置参数的入口。下面都是由YARNConfiguration类进行负责：  

##### 加载资源  
Hadoop配置文件包含name/value属性作为XML数据。这些文件按它们被添加的顺序被加载。YARNConfiguration类将会先加载yarn-default.xml文件，然后再是yarn-site.xml文件。在yarn-site.xml文件指定的值将会被使用。在*-site.xml文件中指定的属性值会覆盖那些在*-default.xml指定的属性值，剩余的属性值依然会从*-default.xml文件中使用。  

思考下面一个例子，一个属性在yarn-default.xml中有一个默认值，并且用户在yarn-site.xml文件中定义相同的属性。  
下面的属性被定义在yarn-default.xml文件中：  
```xml
<property>
   <name>yarn.resourcemanager.hostname</name>
   <value>0.0.0.0</value>
</property>
```  
你在yarn-site.xml文件中指定了相同的属性，如下：  
```xml
<property>
   <name>yarn.resourcemanager.hostname</name>
   <value>masternode</value>
</property>
```  
对于yarn.resourcemanager.hostname属性，YARNConfiguration类将会返回masternode。  

##### Final属性  
一个属性可能被声明为final属性。如果一个管理员不希望任何客户端去更改一些参数的值，那么管理员可以将属性定义为final，就像下面给出的一样：
```xml
<property>
   <name>yarn.acl.enable</name>
   <value>true</value>
   <final>true</final>
</property>
```  

##### 扩展变量
一个属性值可能包含其他定义在配置文件中的属性或者Java进程属性需要的值。可以参考下面resourcemanager主机名的例子：
```xml
<property>
   <name>yarn.resourcemanager.hostname</name>
   <value>masternode</value>
</property>
<property>
   <name>yarn.resourcemanager.webapp.address</name>
   <value>${yarn.resourcemanager.hostname}:8088</value>
</property>
```  

属性yarn.resourcemanager.webapp.address的值使用了yarn.resourcemanager.hostname的属性值。  

提示：在YARN中普遍使用到的一个Java系统属性扩展变量是${user.name}。  

想要阅读更多有关YARNConfiguration类的内容，你可以参考Hadoop API文档 http://hadoop.apache.org/docs/r2.5.1/api/org/apache/hadoop/yarn/conf/YarnConfiguration.html 。  

#### ApplicationSubmissionContext  
ApplicationSubmissionContext是一个抽象类，包含了给一个application运行ApplicationMaster所需要的所有信息。客户端定义了submission context，包含了应用的属性，运行ApplicationMaster服务的命令和资源请求的列表，等等。在应用提交请求期间，客户端会发送这个context到ResourceManager。ResourceManager使用这个context保存应用的状态并且在一个NodeManager节点上运行ApplicationMaster进程。  

ApplicationSubmissionContext类包含下面的内容：
* 应用ID、名字和类型
* 队列和它的优先级
* AM container规范(ContainerLaunchContext for AM)
* AM非托管的Boolean标志符和container管理
* application最大尝试数和资源请求  

阅读更多关于ApplicatiionSubmissionContext类的细节，可以参考位于http://hadoop.apache.org/docs/r2.5.2/api/org/apache/hadoop/yarn/api/records/ApplicationSubmissionContext.html 的Hadoop API文档。  

#### ContainerLaunchContext  
ContainerLaunchContext是一个抽象类，包含了在一个节点上启动container所需要的所有信息。NodeManager进程使用launch context启动与application相关联的containers。ApplicationMaster是application的第一个container并且它的launch context被定义在ApplicationSubmissionContext类中。  

ContainerLaunchContext对象包含下面的信息：
* 在启动期间被使用的本地资源的映射
* 定义的环境变量的映射
* 被用来启动container的一组命令
* 与关联的辅助服务和令牌有关的信息
* Application ACLs(应用访问类型，查看和修改应用)  

阅读更多关于ApplicatiionSubmissionContext类的细节，可以参考位于http://hadoop.apache.org/docs/r2.5.2/api/org/apache/hadoop/yarn/api/records/ContainerLaunchContext.html 的Hadoop API文档。  

#### 通信协议  
YARN API包含4种通信协议用来与YARN客户端进行交互和ApplicationMaster与YARN服务进行交互，比如：ResourceManager、NodeManager和Timeline Server。这些协议都被定义在org.apache.hadoop.yarn.api包中。本节给这些接口和它们的用法一个简答的介绍：  
![image](/Images/YARN/yarn-communication-protocol.PNG)  

##### ApplicationClientProtocol  
ApplicationClientProtocol接口定义客户端与ResourceManager服务之间的通信协议。  

客户端使用这个接口：  
* 创建/提交/杀死应用
* 获取应用/container/应用attempts的记录
* 获取集群的度量/节点/队列信息
* 使用过滤器获取应用和节点列表(GetApplicationRequest和GetClusterNodesRequest)
* 请求一个新的授权令牌或者更新已经存在  

##### ApplicationMasterProcotol  
ApplicationMasterProtocol接口被活跃的ApplicationMaster实例用来与ResourceManager服务进行通信。一旦ApplicationMaster启动之后，它就会向ResourceManager进行注册。ApplicationMaster实例发送AllocateRequest给ResourceManager去请求新的containers和释放不用的或者被列入黑名单的containers。在应用执行完成后，ApplicationMaster会使用finishApplicationMaster()方法给ResourceManager发送一个通知。  

##### ContainerManagementProcotol  
ContainerManagementProtocol接口被用作活跃的ApplicationMaster和NodeManager服务之间的通信协议。ResourceManager给ApplicationMaster实例分配containers，然后ApplicationMaster提交启动container请求给相应的NodeManager。  

一个活跃的ApplicationMaster使用这个接口去：  
* 请求NodeManager使用ContainerLaunchContext启动所有container
* 获取当前containers的状态
* 停止对应ID的container  

##### ApplicationHistoryProcotol  
ApplicationHistoryProtocol是从Hadoop 2.5开始新增加的协议。该协议被客户端用来与application history服务(Timeline服务)进行通信，以获取相关完成的applications的信息。Timeline服务保留着提交到YARN集群上的applications信息的历史数据。客户端可以使用这个接口获取已经完成的applications、containers和application attempts的记录。  

想要阅读更多可用的通信协议信息，你可以参考Hadoop API文档 http://hadoop.apache.org/docs/r2.5.1/api/org/apache/hadoop/yarn/api/package-summary.html  

#### YARN客户端API
YARN客户端API请参考定义在org.apache.hadoop.yarn.api包中的类。这些类使用早前提到的协议，当编写基于Java的YARN应用时被使用。这些是暴露给客户端/ApplicationMaster服务与YARN进程进行通信的类。  

下面是一些在客户端API中的类：  
* **YarnClient**：这个类是客户端与ResourceManager之间通信的桥梁。客户端可以通过这个类提交应用，请求应用的状态/记录和获取集群metrics。
* **AMRMClient/AMRMClientAsync**：这些有助于阻塞式AMRMClient和非阻塞式AMRMClientAsync在ApplicationMaster与ResourceManager之间的进行通信。正如第5章，理解YARN的生命周期中提到的，ApplicationMaster使用AMRMClient与ResourceManager服务进行连接。ApplicationMaster使用AMRMClient去注册AM服务，从ResourceManager那里请求资源，获取集群可用的资源。
* **NMClient/NMClientAsync**：这些有助于阻塞式AMRMClient和非阻塞式AMRMClientAsync在ApplicationMaster与NodeManager之间的进行通信。类似于与ResourceManager连接，ApplicationMaster创建一个连接到分配了container的NodeManager。ApplicationMaster使用NMClient去请求启动/停止containers和获得container的状态。
* **AHSClient/TimelineClient**：这个有助于客户端与Timeline服务之间的通信。一旦applications完成，客户端可以从Timeline服务中获取application的记录。客户端使用AHSClient去获取已经完成的application的列表，attempts和containers。  

想要阅读更多有关YARN客户端API，你可以参考Hadoop API文档 http://hadoop.apache.org/docs/r2.5.1/api/org/apache/hadoop/yarn/api/package-summary.html  

### 编写自己的YARN应用  
YARN框架可以灵活的在集群环境中运行任何应用。应用可以像一个Java进程，一个shell脚本或者一个简单的date命令一样简单。ResourceManager管理着集群资源的分配，NodeManager通过特定的应用框架执行任务；比如Hadoop MapReduce任务是map任务和reduce任务。  

在本节中，你将会编写你自己的通过YARN运行在分布式环境中的应用。  

完整的程序可以概括为4个步骤，就如下面图中所示：  
![image](/Images/YARN/create-yarn-app-step.PNG)

#### Step 1-创建一个新的项目并且添加Hadoop-YARN JAR文件  
我们将会在用Eclipse创建一个新的Java项目，并且使用YARN client API写一个简单的YARN application。你要么创建一个简单的Java项目，要么创建一个Maven项目。  

你需要添加下面的jar文件到你的项目的构建路径：
* hadoop-yarn-client-2.5.1.jar
* hadoop-yarn-api-2.5.1.jar
* hadoop-yarn-common-2.5.1.jar
* hadoop-common-2.5.1.jar  

如果你选择创建一个简单的Java项目，你可以在你的项目中创建一个library文件夹(被叫做lib)去存储需要的jar文件，并且添加需要的jar文件到library文件夹。如果你选择创建一个Maven项目，那么你将需要在你的pom.xml文件中添加下面的依赖，并且安装项目去解析这些依赖：  
```xml
<dependency>
  <groupId>org.apache.hadoop</groupId>
  <artifactId>hadoop-yarn-client</artifactId>
  <version>2.5.1</version>
</dependency>
<dependency>
  <groupId>org.apache.hadoop</groupId>
  <artifactId>hadoop-yarn-common</artifactId>
  <version>2.5.1</version>
</dependency>
<dependency>
  <groupId>org.apache.hadoop</groupId>
  <artifactId>hadoop-yarn-api</artifactId>
  <version>2.5.1</version>
</dependency>
<dependency>
  <groupId>org.apache.hadoop</groupId>
  <artifactId>hadoop-common</artifactId>
  <version>2.5.1</version>
</dependency>
```

#### Step 2-定义ApplicationMaster和client类  
客户端需要定义类给ApplicationMaster管理应用的执行和YARN客户端提交应用到ResourceManager。  

当编写ApplicationMaster和YARN客户端时，下面是client的规则：
* 定义Application：
    * 初始化AMRMClient和NMClient客户端
    * 向ResourceManager注册尝试次数
    * 定义ContainerRequest和增加containers请求
    * 请求分配、定义ContainerLaunchContext并且启动containers
    * 完成后，从ResourceManager移除ApplicationMaster的注册
* 提交应用到ResourceManager
    * 读取YARNConfiguration并且初始化YARNClient
    * 连接到RM并且请求一个新的application id
    * 为Application Master定义ContainerLaunchContext
    * 创建ApplicationSubmissionContext
    * 提交应用并且等待完成  

##### 定义一个ApplicationMaster  
创建一个新的包并且创建一个新的带有main方法的ApplicationMaster.java类到你的项目中。你需要添加下面的代码片段到你ApplicationMaster.java类：  
```java
package com.packt.firstyarnapp;

import java.util.Collections;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.net.NetUtils;
import org.apache.hadoop.yarn.api.ApplicationConstants;
import org.apache.hadoop.yarn.api.protocolrecords.AllocateResponse;
import org.apache.hadoop.yarn.api.records.Container;
import org.apache.hadoop.yarn.api.records.ContainerLaunchContext;
import org.apache.hadoop.yarn.api.records.ContainerStatus;
import org.apache.hadoop.yarn.api.records.FinalApplicationStatus;
import org.apache.hadoop.yarn.api.records.Priority;
import org.apache.hadoop.yarn.api.records.Resource;
import org.apache.hadoop.yarn.client.api.AMRMClient;
import org.apache.hadoop.yarn.client.api.AMRMClient.ContainerRequest;
import org.apache.hadoop.yarn.client.api.NMClient;
import org.apache.hadoop.yarn.conf.YarnConfiguration;
import org.apache.hadoop.yarn.util.Records;

public class ApplicationMaster {
   public static void main(String[] args) throws Exception {
      System.out.println("Running ApplicationMaster");
      final String shellCommand = args[0];
      final int numOfContainers = Integer.valueOf(args[1]);
      Configuration conf = new YarnConfiguration();

      // Point #2
      System.out.println("Initializing AMRMCLient");
      AMRMClient<ContainerRequest> rmClient = AMRMClient.createAMRMClient();
      rmClient.init(conf);
      rmClient.start();
      System.out.println("Initializing NMCLient");
      NMClient nmClient = NMClient.createNMClient();
      nmClient.init(conf);
      nmClient.start();

      // Point #3
      System.out.println("Register ApplicationMaster");
      rmClient.registerApplicationMaster(NetUtils.getHostname(), 0, "");

      // Point #4
      Priority priority = Records.newRecord(Priority.class);
      priority.setPriority(0);
      System.out.println("Setting Resource capability for Containers");
      Resource capability = Records.newRecord(Resource.class);
      capability.setMemory(128);
      capability.setVirtualCores(1);
      for (int i = 0; i < numOfContainers; ++i) {
         ContainerRequest containerRequested = new ContainerRequest(capability, null, null, priority, true);
         // Resource, nodes, racks, priority and relax locality flag
         rmClient.addContainerRequest(containerRequested);
      }

      // Point #6
      int allocatedContainers = 0;
      System.out.println("Requesting container allocation from ResourceManager");
      while (allocatedContainers < numOfContainers) {
         AllocateResponse response = rmClient.allocate(0);
         for (Container container : response.getAllocatedContainers()) {
            ++allocatedContainers;
            // Launch container by creating ContainerLaunchContext
            ContainerLaunchContext ctx = Records.newRecord(ContainerLaunchContext.class);
            ctx.setCommands(Collections.singletonList(shellCommand + " 1>"
                           + ApplicationConstants.LOG_DIR_EXPANSION_VAR
                           + "/stdout" + " 2>"
                           + ApplicationConstants.LOG_DIR_EXPANSION_VAR
                           + "/stderr"));
            System.out.println("Starting container on node : " + container.getNodeHttpAddress());
            nmClient.startContainer(container, ctx);
         }
         Thread.sleep(100);
      }

      // Point #6
      int completedContainers = 0;
      while (completedContainers < numOfContainers) {
         AllocateResponse response = rmClient.allocate(completedContainers / numOfContainers);
         for (ContainerStatus status : response.getCompletedContainersStatuses()) {
            ++completedContainers;
            System.out.println("Container completed : " + status.getContainerId());
            System.out.println("Completed container " + completedContainers);
         }
         Thread.sleep(100);
      }
      rmClient.unregisterApplicationMaster(FinalApplicationStatus.SUCCEEDED,"", "");
   }
}
```  
ApplicationMaster的代码片段的解释如下：  
1. **读取YARN的配置和输入参数**：ApplicationMaster使用YARNConfiguration类去加载Hadoop-YARN配置文件并且读取指定的输入参数。在这个例子中，第一个参数是shellCommand，比如/bin/date；第二个参数是numofContainers在application执行期间被执行：
```java
Public static void main(String[] args) throws Exception {
   final String shellCommand = args[0];
   final intnumOfContainers = Integer.valueOf(args[1]);
   Configuration conf = new YarnConfiguration();
}
```  
2. **初始化AMRMClient和NMClient客户端**：ApplicationMaster首先会创建并且初始化与ResourceManager进行通信的接口AMRMClient和与NodeManager进行通信的接口NMClient，代码如下：
```java
AMRMClient<ContainerRequest> rmClient = AMRMClient.createAMRMClient();
rmClient.init(conf);
rmClient.start();
NMClient nmClient = NMClient.createNMClient();
nmClient.init(conf);
nmClient.start();
```  
3. **向ResourceManager注册attempt**：ApplicationMaster向ResourceManager进行注册。它需要为attempt指定主机名，端口和一个URL。在注册成功后，ResourceManager会将application的状态更新为RUNNING。  
```java
rmClient.registerApplicationMaster(NetUtils.getHostname(), 0,"");
```
4. **定义ContainerRequest并且添加container的请求**：客户端定义工作的containers在内存和cores方面的需求条件(org.apache.hadoop.yarn.api.records.Resource)。客户端可能也会指定containers的优先级，一个优先的节点列表和资源所在地的机架。客户端创建一个ContainerContext的引用并且在调用allocate()方法前添加请求：  
```java
Priority priority = Records.newRecord(Priority.class);
priority.setPriority(0);

Resource capability = Records.newRecord(Resource.class);
capability.setMemory(128);
capability.setVirtualCores(1);
for (inti = 0; i<numOfContainers; ++i) {
   ContainerRequest containerRequested = new
   ContainerRequest(capability, null, null, priority, true);
   // Resource, nodes, racks, priority and relax locality flag
   rmClient.addContainerRequest(containerRequested);
}
```  
5. **请求分配、定义ContainerLaunchContext和启动containers**：ApplicationMaster请求ResourceManager分配请求的containers并且通知ResourceManager关于当前application的进展。因此，进度指示器在第一次分配期间的值为0。来自ResourceManager的响应中包含分配的containers的数量。ApplicationMaster为每个分配container创建一个ContainerLaunchContext并且请求响应的NodeManager启动container。ApplicationMaster将会等待containers的执行。在这个例子中，运行containers的命令作为ApplicationMaster的第一个参数(/bin/date命令)：  
```java
intallocatedContainers = 0;
while (allocatedContainers<numOfContainers) {
AllocateResponse response = rmClient.allocate(0);
for (Container container : response.getAllocatedContainers()) {
   ++allocatedContainers;
   // Launch container by creating ContainerLaunchContext
   ContainerLaunchContext ctx = Records.newRecord(ContainerLaunchContext.class);
   ctx.setCommands(Collections.singletonList(shellCommand +
   " 1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stdout"
   +
   " 2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stderr"
   ));
   nmClient.startContainer(container, ctx);
   }
   Thread.sleep(100);
}
```
6. **完成后，从ResourceManager注销ApplicationMaster**：分配响应同样也包含已完成的containers的列表。一旦在响应中获得的所有containers开始在不同的NodeManager上开始执行，ApplicationMaster将会等待它们的完成。ContainerStatus类提供了执行中的container的当前状态。为了注销ApplicationMaster，需要在AMRMClient引用中调用unregisterApplicationMaster()方法。随着注销方法的调用，ApplicationMaster会将application的最终状态，application的消息和application的URL作为参数进行发送：  
```java
intcompletedContainers = 0;
while (completedContainers<numOfContainers) {
   AllocateResponse response = rmClient.allocate(completedContainers/numOfContainers);
   for (ContainerStatus status : response.getCompletedContainersStatuses()) {
      ++completedContainers;
      System.out.println("Completed container " +
      completedContainers);
   }
   Thread.sleep(100);
}
rmClient.unregisterApplicationMaster(FinalApplicationStatus.SUCCEEDED, "", "");
```  
##### 定义一个YARN客户端
创建一个新的带有main方法的类Client.java到你的项目中。为了简单起见，你可以在相同的项目中创建它。  

Client.java文件中的代码如下：  
```java
package com.packt.firstyarnapp;

import java.io.File;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.yarn.api.ApplicationConstants;
import org.apache.hadoop.yarn.api.ApplicationConstants.Environment;
import org.apache.hadoop.yarn.api.records.ApplicationId;
import org.apache.hadoop.yarn.api.records.ApplicationReport;
import org.apache.hadoop.yarn.api.records.ApplicationSubmissionContext;
import org.apache.hadoop.yarn.api.records.ContainerLaunchContext;
import org.apache.hadoop.yarn.api.records.LocalResource;
import org.apache.hadoop.yarn.api.records.LocalResourceType;
import org.apache.hadoop.yarn.api.records.LocalResourceVisibility;
import org.apache.hadoop.yarn.api.records.Resource;
import org.apache.hadoop.yarn.api.records.YarnApplicationState;
import org.apache.hadoop.yarn.client.api.YarnClient;
import org.apache.hadoop.yarn.client.api.YarnClientApplication;
import org.apache.hadoop.yarn.conf.YarnConfiguration;
import org.apache.hadoop.yarn.util.Apps;
import org.apache.hadoop.yarn.util.ConverterUtils;
import org.apache.hadoop.yarn.util.Records;

public class Client {
   public static void main(String[] args) throws Exception {
      try {
         Client clientObj = new Client();
         if (clientObj.run(args)) {
            System.out.println("Application completed
            successfully");
         } else {
            System.out.println("Application Failed / Killed");
         }
         } catch (Exception e) {
            e.printStackTrace();
         }
      }

      public boolean run(String[] args) throws Exception {
         // Point #1
         final String command = args[0];
         final int n = Integer.valueOf(args[1]);
         final Path jarPath = new Path(args[2]);
         System.out.println("Initializing YARN configuration");
         YarnConfiguration conf = new YarnConfiguration();
         YarnClient yarnClient = YarnClient.createYarnClient();
         yarnClient.init(conf);
         yarnClient.start();

         // Point #2
         System.out.println("Requesting ResourceManager for a new
         Application");
         YarnClientApplication app =
         yarnClient.createApplication();

         // Point #3
         System.out.println("Initializing ContainerLaunchContext
         for ApplicationMaster container");
         ContainerLaunchContext amContainer = Records.newRecord(ContainerLaunchContext.class);
         System.out.println("Adding LocalResource");
         LocalResource appMasterJar =
         Records.newRecord(LocalResource.class);
         FileStatus jarStat = FileSystem.get(conf).getFileStatus(jarPath);
         appMasterJar.setResource(ConverterUtils.getYarnUrlFromPath(jarPath));
         appMasterJar.setSize(jarStat.getLen());
         appMasterJar.setTimestamp(jarStat.getModificationTime());
         appMasterJar.setType(LocalResourceType.FILE);
         appMasterJar.setVisibility(LocalResourceVisibility.PUBLIC);

         // Point #4
         System.out.println("Setting environment");
         Map<String, String> appMasterEnv = new HashMap<String, String>();
         for (String c : conf.getStrings(YarnConfiguration.YARN_APPLICATION_CLASSPATH,
                                    YarnConfiguration.DEFAULT_YARN_APPLICATION_CLASSPATH))
         {
            Apps.addToEnvironment(appMasterEnv,
            Environment.CLASSPATH.name(),
            c.trim());
         }
         Apps.addToEnvironment(appMasterEnv,
         Environment.CLASSPATH.name(),
         Environment.PWD.$() + File.separator + "*");
         System.out.println("Setting resource capability");
         Resource capability = Records.newRecord(Resource.class);
         capability.setMemory(256);
         capability.setVirtualCores(1);
         System.out.println("Setting command to start
         ApplicationMaster service");
         amContainer.setCommands(Collections.singletonList("/usr/lib/jvm/jdk1.8.0/bin/java"
         + " -Xmx256M" + "com.packt.firstyarnapp.ApplicationMaster"
         + " " + command + " " + String.valueOf(n) + " 1>"
         + ApplicationConstants.LOG_DIR_EXPANSION_VAR +
         "/stdout"
         + " 2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR
         + "/stderr"));
         amContainer.setLocalResources(Collections.singletonMap("first-yarn-app.jar", appMasterJar));
         amContainer.setEnvironment(appMasterEnv);
         System.out.println("Initializing ApplicationSubmissionContext");
         ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
         appContext.setApplicationName("first-yarn-app");
         appContext.setApplicationType("YARN");
         appContext.setAMContainerSpec(amContainer);
         appContext.setResource(capability);
         appContext.setQueue("default");
         ApplicationId appId = appContext.getApplicationId();
         System.out.println("Submitting application " + appId);
         yarnClient.submitApplication(appContext);
         ApplicationReport appReport = yarnClient.getApplicationReport(appId);
         YarnApplicationState appState = appReport.getYarnApplicationState();
         while (appState != YarnApplicationState.FINISHED
               && appState != YarnApplicationState.KILLED
               && appState != YarnApplicationState.FAILED) {
            Thread.sleep(100);
            appReport = yarnClient.getApplicationReport(appId);
            appState = appReport.getYarnApplicationState();
         }
         if (appState == YarnApplicationState.FINISHED) {
            return true;
         } else {
            return false;
      }
   }
}
```  

你需要添加给定的代码片段到Client.java类的run()方法中：  
1. **读取YARNConfiguration和初始化YARNClient**：类似于ApplicationMaster，客户端同样使用YARNConfiguration类去加载Hadoop-YARN的配置文件和读取指定的输入参数。客户端在客户端节点启动一个YARNClient服务。  在本例中，前两个参数是直接传给ApplicationMaster中的ContainerLaunchContext的，第三个参数是位于HDFS的路径用来给job执行的数据(带有ApplicationMaster的jar文件)：  
```java
public Boolean run(String[] args) throws Exception {
   final String command = args[0];
   final int n = Integer.valueOf(args[1]);
   final Path jarPath = new Path(args[2]);
   YarnConfigurationconf = new YarnConfiguration();
   YarnClientyarnClient=YarnClient.createYarnClient();
   yarnClient.init(conf);
   yarnClient.start();
}
```
2. **连接到ResourceManager并且请求一个新的application ID**：客户端连接到ResourceManager服务请求一个新的application。请求的响应(YarnClientApplication-GetNewApplicationResponse)中包含一个新的application ID和集群中最小和最大的资源容量。  
```java
YarnClientApplication app = yarnClient.createApplication();
```  
3. **为ApplicationMaster定义ContainerLaunchContext**：一个application中的第一个container是作为ApplicationMaster的container。客户端会定义一个包含启动ApplicationMaster服务的ContainerLaunchContext。其中ContainerLaunchContext会包含下面的信息：  
    * **为ApplicationMaster设置jar文件**：NodeManager应该能够找到jar文件。其中jar文件是位于HDFS上并且被NodeManager作为一个LocalResource访问，代码如下：  
    ```java
      ContainerLaunchContextamContainer = Records.newRecord(ContainerLaunchContext.class);
      LocalResourceappMasterJar = Records.newRecord(LocalResource.class);
      FileStatusjarStat = FileSystem.get(conf).getFileStatus(jarPath);
      appMasterJar.setResource(ConverterUtils.
      getYarnUrlFromPath(jarPath));
      appMasterJar.setSize(jarStat.getLen());
      appMasterJar.setTimestamp(jarStat.getModificationTime());
      appMasterJar.setType(LocalResourceType.FILE);
      appMasterJar.setVisibility(LocalResourceVisibility.PUBLIC);
    ```
    * **为ApplicationMaster设置CLASSPATH**：你可能会使用shell命令运行你的ApplicationMaster，这样就需要一些环境变量。客户端可以指定一系列环境变量。  
    ```java
      Map<String, String>appMasterEnv = new HashMap<String, String>();
      for (String c : conf.getStrings(YarnConfiguration.YARN_APPLICATION_CLASSPATH,
            YarnConfiguration.DEFAULT_YARN_APPLICATION_CLASSPATH))
      {
         Apps.addToEnvironment(appMasterEnv, Environment.CLASSPATH.name(),c.trim());
      }
      Apps.addToEnvironment(appMasterEnv,Environment.CLASSPATH.name(),Environment.PWD.$() + File.separator + "*");
    ```
    * **为ApplicationMaster设置资源需求条件**：ApplicationMaster对资源的需求以内存和CPU cores的形式定义。  
    ```java
      Resource capability = Records.newRecord(Resource.class);
      capability.setMemory(256);
      capability.setVirtualCores(1);
    ```
    * **启动ApplicationMaster服务的命令**：在本例中，ApplicationMaster是一个Java程序，因此，客户端需要定义一个Java的jar命令去启动ApplicationMaster。  
    ```java
      amContainer.setCommands(Collections.singletonList("$JAVA_HOME/bin/java" + " –Xmx256M"
      + " com.packt.firstyarnapp.ApplicationMaster" + " " + command
      + " " + String.valueOf(n) + " 1>"
      + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stdout" + " 2>"
      + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stderr" ));
      amContainer.setLocalResources(Collections.singletonMap("firstyarn-app.jar",appMasterJar));
      amContainer.setEnvironment(appMasterEnv);
    ```  
4. **创建ApplicationSubmissionContext**：客户端为application定义ApplicationSubmissionContext。submission context包含了诸如application名、队列、优先级等等的信息。  
```java
   ApplicationSubmissionContextappContext = app.getApplicationSubmissionContext();
   appContext.setApplicationName("first-yarn-app");
   appContext.setApplicationType("YARN");
   appContext.setAMContainerSpec(amContainer);
   appContext.setResource(capability);
   appContext.setQueue("default");
```  
5. **提交application并等待完成**：客户端提交application并且等待它的完成。他会请求ResourceManager要application的状态。  
```java
   ApplicationIdappId = appContext.getApplicationId();
   System.out.println("Submitting application " + appId);
   yarnClient.submitApplication(appContext);
   ApplicationReportappReport = yarnClient.getApplicationReport(appId);
   YarnApplicationStateappState = appReport.getYarnApplicationState();
   while (appState != YarnApplicationState.FINISHED &&
         appState != YarnApplicationState.KILLED &&
         appState != YarnApplicationState.FAILED) {
      Thread.sleep(100);
      appReport = yarnClient.getApplicationReport(appId);
      appState = appReport.getYarnApplicationState();
   }
```  

#### Step 3-导出项目并且复制资源  
你需要将Java项目导出为jar文件，并且将jar文件上传到HDFS上。如果你创建为Client.java和ApplicationMaster.java创建了两个不同的项目，那么你需要将两个项目都导出jar文件，并且将ApplicationMaster jar文件上传到HDFS上。在这个案例中，你仅仅只需要创建一个jar文件。为了复制文件到HDFS上，你可以使用Hadoop中的hdfs命令，要么使用put选项要么使用copyFromLocal选项。假如jar文件的名字是first-yarn-app.jar，那么hdfs命令应该像这样：  
```shell
bin/hdfs dfs -put first-yarn-app.jar /user/hduser/first-yarn-app.jar
```  

#### Step 4-使用bin运行application或者YARN命令  
最后一步是使用Hadoop-bin文件价($HADOOP_PREFIX/bin)下的yarn命令提交应用到yarn上。  

正如前面所提及的Client.java类中的main方法，你需要传入三个参数：
* shell命令
* 需要的container数量
* HDFS上包含ApplicationMaster类的jar文件路径  

提交应用到YARN的命令看起来像下面一样：  
```shell
bin/yarn jar first-yarn-app.jar com.packt.firstyarnapp.Client /bin/true 1 hdfs://master:8020/user/hduser/first-yarn-app.jar
```  
前面应用的输出看起来像下面这样：  
```xml
Initializing YARN configuration
15/07/05 23:52:51 INFO client.RMProxy: Connecting to ResourceManager
at master/192.168.56.101:8032
Requesting ResourceManager for a new Application
Initializing ContainerLaunchContext for ApplicationMaster container
Adding LocalResource
Setting environment
Setting resource capability
Setting command to start ApplicationMaster service
Initializing ApplicationSubmissionContext
Submitting application application_1436101688138_0009
15/07/05 23:52:53 INFO impl.YarnClientImpl: Submitted application
application_1436101688138_0009
Application completed successfully
```  
程序的输出将会展示在终端。你也可以在ResourceManager web UI上查看被提交应用的状态。就像下面截图所展示的一样：  
![Image](/Images/YARN/ownyarnapp.png)  

提示：编写一个完整的YARN兼容的分布式应用是一个非常复杂的任务并且它不允许开发者去关注业务逻辑。一个开发者/管理员也需要去监控和管理运行的应用。Apache Slider和Apache Twill是两个当前正在孵化状态的项目，这两个项目目的是为了减少在YARN上编写应用的复杂性和更简单的与YARN进行集成。想要阅读更多有关这些框架的信息，可以参考它们的官方文档http://slider.incubator.apache.org/和http://twill.incubator.apache.org。  

### 总结
编写自己的Hadoop-YARN应用允许用户在分布式环境中实现自己的业务逻辑(不同于MapReduce编程)。本章涉及了YARN APIs的基本知识并且通过编写一个简单的YARN应用和你一起走过了这一章。想要阅读更多关于这一主题的内容，你可以参考Hadoop文档http://hadoop.apache.org/docs/r2.5.1/hadoop-yarn/hadoop-yarn-site/WritingYarnApplications.html。  

Hadoop文档粗略的涉及到编写YARN应用时用到的所有YARN API。你也可以参考Hortonworks在GitHub上的一个例子 https://github.com/apache/hadoop-common/tree/trunk/hadoop-yarnproject/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applicationsdistributedshell。 你可以编译并且在Hadoop-YARN集群上运行它。

下一章涉及到YARN内部细节和YARN组件的核心服务。它有助于Java开发者和开源代码贡献者更好的理解YARN组件之间的通信和更深入的学习YARN的架构。
