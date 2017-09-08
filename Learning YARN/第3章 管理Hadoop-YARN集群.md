在之前的章节，我们已经涉及了去配置一个单节点和多节点的Hadoop-YARN集群的安装步骤。作为一个Hadoop-YARN集群的管理员或者用户，知道服务是如何进行配置或者管理是非常重要的。比如，一个管理员必要我去监控整个集群中多有节点的健康状况，一个用户应该能够查看应用提交的日志。  

Hadoop-YARN不但预定义了一组用户命令，而且还有管理命令。它将监控数据作为服务度量暴露出来，并且提供了对监控数据简单的交互方式，使用工具，比如Ganglia，Nagios等等。它也定义了一种用于高可用和恢复的机制。  

在本章，我们将会涉及到：  
* YARN用户命令和管理员命令
* 配置、管理和监控YARN服务
* 监控NodeManager服务  

### 使用Hadoop-YARN命令  
YARN命令使用hadoop包中的bin/yarn脚本中进行调用。yarn命令基本的语法如下：  
```shell
yarn [--config confdir] COMMAND COMMAND_OPTIONS
```  

如果运行YARN命令不带任何参数，那么会打印命令的所有描述。config选项时可选的，而且它的默认值是$HADOOP_PREFIX/etc/hadoop。  

YARN命令分为用户命令和管理员命令，如下图所示：  
![image](/Images/YARN/yarn-command-type.png)  

#### 用户命令  
Hadoop-YARN客户端可以执行用户命令。客户端通过在yarn-site.xml文件中指定的配置连接到YARN服务。正如前面章节所介绍的，yarn-site.xml文件和其他配置文件都被放置在Hadoop的配置目录中($HADOOP_HOME/etc/hadoop)。YARN框架中主要有六个用户命令。  

##### Jar  
jar命令是用来运行编写了YARN代码的jar文件，也就是，提交一个YARN应用到ResourceManager。  
* 用法：yarn jar <jar_path> <mainClass> args
* 类：org.apache.hadoop.util.RunJar  

RunJar类的main方法会被调用。它会检查参数列表然后验证jar文件。它会提取jar文件并且运行在第二个参数指定的Java类的main方法。  

##### Application  
application命令是用来在任何客户端上打印提交到ResourceManager上的应用列表。它也可以用来报告和杀死一个应用。  
* 用法：yarn application <options>
* 类：org.apache.hadoop.yarn.client.cli.ApplicationCLI  

**命令选项**  
* -status ApplicationId：status选项被用于以应用报告的形式打印应用的状态。对于一个存在的/有效的应用ID，它会打印从org.apache.hadoop.yarn.api.records.ApplicationReport类对象中取回的数据。对于一个不存在的应用ID，它会抛出一个ApplicationNotFoundException。  

**示例输出**  
```shell
Application Report:
Application-Id: application_1389458248889_0001
Application-Name: QuasiMonteCarlo
Application-Type: MAPREDUCE
User: Root
Queue: Default
Start-Time: 1389458385135
Finish-Time: 1389458424546
Progress: 100%
State: FINISHED
Final-State: SUCCEEDED
Tracking-URL: http://slave1:19888/jobhistory/job/job_1389458248889_0001
RPC Port: 34925
AM Host: slave3
Diagnostics:
```   

* –list –appTypes=[ ] –appStates=[ ]：list选项会打印出基于应用类型和状态的所有应用。它支持两个子选项，appTypes和appStates。如果没有指定选项，默认情况下，所有RUNNING，ACCEPTED，SUBMITTED状态的应用会被列出来。用户可以指定以逗号作为分隔符的值列表作为类型和状态过滤条件(不带有空格)。  
    * appTypes：MAPREDUCE，YARN
    * appStates：ALL，NEW，NEW_SAVING，SUBMITTED，ACCEPTED，RUNNING，FINISHED，FAILED，KILLED  

* -kill ApplicationId：kill选项被用于杀死一个正在运行的/已经提交的应用。如果应用已经完成了，应用的状态要么是FINISHED，KILLED，要么是FAILED，然后会在命令行打印信息。换句话说，它是发送了一个请求到ResourceManager去杀死一个应用。  

##### Node  
YARN由运行着NodeManager守护进程的Java程序的节点组成。ResourceManager保存着节点信息，yarn node命令使用 org.apache.hadoop.yarn.api.records.NodeReport类以节点报告的形式打印节点信息。  
* 用法：yarn node <options>
* 类：org.apache.hadoop.yarn.client.cli.NodeCLI  

**命令选项**  
* -status NodeId：status选项被用于以节点报告的形式打印节点状态。NodeId参数是一个字符串，代表了org.apache.hadoop.yarn.api.records.NodeId类的一个对象，也就是，节点主机名和与NodeManager守护进程通信端口，IPC服务监控端口的组合。对于存在的/有效的节点ID，它会打印从NodeReport类对象获取的数据。  

**示例输出**  
```shell
Node Report:
Node-Id: slave1:36801
Rack: /default-rack
Node-State: RUNNING
Node-Http-Address:slave1: 8042
Last-Health-Update: Sun 09/Feb/14 11:37:53:774IST
Health-Report:
Containers: 0
Memory-Used: 0MB
Memory-Capacity: 8192MB
CPU-Used: 0 vcores
CPU-Capacity: 8 vcores
```  

* -list：list选项会打印出基于节点状态所有节点的列表。它支持一个可选的用法-states去基于节点状态进行过滤，-all列出所有节点：  
    * 用户可以指定以逗号进行分割的状态值列表进行过滤。org.apache.hadoop.yarn.api.records.NodeState是一个代表了节点不同状态的枚举。
    * 状态有： NEW, RUNNING, UNHEALTHY, DECOMMISSIONED, LOST,REBOOTED
    * list命令的输出一系列有关节点的信息，比如,Node-Id,Node-State,Node-Http-Address和运行的container的数量。  

##### Logs  
logs命令是用来获取已完成应用的日志，也就是说，任意以下三种状态之一的应用——FAILED，KILLED或者FINISHED。  

想要通过命令行查看日志，用户需要开启YARN集群的日志聚合(log-aggregation)。想要启用日志聚合功能，用户需要设置yarn-site.xml文件中的 yarn.log-aggregation-enable属性为true。用户也可以基于应用的container ID和节点ID查看日志。  
* 用法：yarn logs -applicationId <application ID> <options>
* org.apache.hadoop.yarn.client.cli.LogsCLI  

**命令选项**  
* -applicationId applicationID：applicationId命令是托管式的，用于从ResourceManager获取应用细节
* -appOwner AppOwner：可选的，加入没有指定，默认是当前用户
* -nodeAddress NodeAddress -containerId containerId：nodeAddress和containerId命令可以指定特定节点上指定container的日志。nodeAddress是host:port字符串的形式(与NodeId相同)。  

##### Classpath  
classpath命令被用于打印YARN集群当前CLASSPATH的值。这个命令对于开发人员和集群管理员非常有用，因为它展示了正在运行的YARN的PATH库列表。  
* 用法：yarn classpath
* 脚本：echo $CLASSPATH  

##### Version  
version命令被用于所部属YARN集群的版本。因为YARN是与Hadoop紧紧绑定在一起的，所以该命令会使用HadoopUtil类去获取使用的hadoop包的版本。  
* 用法：yarn version
* 类：org.apache.hadoop.util.VersionInfo  

#### 管理命令  
YARN管理命令主要用于在一个单独节点上启动集群服务。集群管理员也使用管理命令去管理集群的节点，队列，与访问控制列表相关的信息，等等。  

##### ResourceManager/NodeManager/ProxyServer  
这些命令被用于在一个单独的节点上启动YARN服务。对于ResourceManager和NodeManager服务，脚本会将日志属性添加到classpath环境变量。想要修改日志属性，用户需要集群配置目录下指定配置目录(rm-config和nm-config)创建log4j.properties文件。YARN脚本也会使用环境变量定义服务使用的JVM堆的大小。  

* 用法：yarn resourcemanager  
* 类：org.apache.hadoop.yarn.server.resourcemanager.ResourceManager
* 用法：yarn nodemanager
* 类：org.apache.hadoop.yarn.server.nodemanager.NodeManager
* 用法：yarn proxyserver
* 类：org.apache.hadoop.yarn.server.webproxy.WebAppProxyServer  

##### RMAdmin  
rmadmin命令会从命令行启动一个资源管理器的客户端。它用于刷新访问控制策略，调度器策略和ResourceManager注册节点。在rmadmin刷新命令后，并且集群没有请求重启相关服务，那么策略的更改会直接反应到集群上的。  

RMAdminCLI类使用YARN protobuf服务去调用在org.apache.hadoop.yarn.server.resourcemanager包中的AdminService类中的方法。  

* 用法：yarn rmadmin <options>
* 类：org.apache.hadoop.yarn.client.cli.RMAdminCLI  

**命令选项**  
* -refreshQueues：重新加载队列的访问控制，状态和调度器属性。它会使用最新的配置文件重新初始化配置的调度器。
* -refreshNodes：刷新ResourceManager中节点的信息。它会读取ResourceManager节点上include和exclude文件进而更新集群中included和excluded节点列表。
* -refreshUserToGroupsMappings：




##### DaemonLog  






### 配置Hadoop-YARN服务  




#### ResourceManager服务  






#### NodeManager服务  



#### Timeline服务



#### web application proxy服务  





#### 端口摘要  







### 管理Hadoop-YARN服务  


#### 管理服务日志  



#### 管理pid文件  



### 监控YARN服务  


#### JMX监控  


##### ResourceManager JMX  

##### NodeManager JMX  



#### Ganglia监控  



##### Ganglia守护进程  



##### Ganglia与Hadoop交互  



### 理解ResourceManager高可用  



#### 架构  



#### 故障切换原理  



#### 配置ResourceManager高可用  



##### 定义节点  


##### RM状态存储机制  



##### 故障切换代理  




##### 自动恢复  





#### 高可用管理命令  







### 监控NodeManager健康  





#### 健康检查脚本  





### 总结
