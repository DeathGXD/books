在第一章，我们讨论过了Hadoop 1.x框架的弊端。Hadoop 1.x仅仅局限于MapReduce编程。  


在本章中，我们将涉及到下面的主题：  
* 介绍YARN API
* 核心的概念和有关的类
* 编写自己的YARN应用
* 在Hadoop-YARN集群上执行应用  

### YARN API介绍  
YARN是一个与Hadoop捆绑打包在一起的Java框架。它提供资源管理，  

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

##### 变量扩展  


#### ApplicationSubmissionContext  
ApplicationSubmissionContext是一个抽象类，包含了给一个application运行ApplicationMaster所需要的所有信息。客户端定义了submission context，包含了应用的属性，运行ApplicationMaster服务的命令和资源请求的列表，等等。在应用提交请求期间，客户端会发送这个context到ResourceManager。ResourceManager使用这个context保存应用的状态并且在一个NodeManager节点上运行ApplicationMaster进程。  

ApplicationSubmissionContext类包含下面的内容：
* 应用ID、名字和类型
* 队列和它的优先级
* AM container规范(ContainerLaunchContext for AM)
* AM非托管的Boolean标志符和container管理
* application最大尝试数和资源请求  

阅读更多关于ApplicatiionSubmissionContext类的细节，可以参考位于(http://hadoop.apache.org/docs/r2.5.2/api/org/apache/hadoop/yarn/api/records/ApplicationSubmissionContext.html)的Hadoop API文档。  

#### ContainerLaunchContext  
ContainerLaunchContext是一个抽象类，包含了在一个节点上启动container所需要的所有信息。NodeManager进程使用launch context启动与application相关联的containers。ApplicationMaster是application的第一个container并且它的launch context被定义在ApplicationSubmissionContext类中。  

ContainerLaunchContext对象包含下面的信息：
* 在启动期间被使用的本地资源的映射
* 定义的环境变量的映射
* 被用来启动container的一组命令
* 与关联的辅助服务和令牌有关的信息
* Application ACLs(应用访问类型，查看和修改应用)  

阅读更多关于ApplicatiionSubmissionContext类的细节，可以参考位于(http://hadoop.apache.org/docs/r2.5.2/api/org/apache/hadoop/yarn/api/records/ContainerLaunchContext.html)的Hadoop API文档。  

#### 通信协议  


##### ApplicationClientProtocol  


##### ApplicationMasterProcotol  


##### ContainerManagementProcotol  

##### ApplicationHistoryProcotol  


#### YARN client API


### 编写自己的YARN应用  


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


#### Step 3-导出项目并且复制资源配置  


#### Step 4-使用bin运行application或者YARN命令
