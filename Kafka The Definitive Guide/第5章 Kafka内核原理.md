理解Kafka内核对在生产环境运行Kafka或者使用Kafka编写应用是非必要的。然而，知道Kafka工作原理在对Kafka进行故障排查或者尝试去理解Kafka的一些行为表现时提供了来龙去脉。由于涉及每个单独的实现细节和设计方案超出了本书的范围，在本章，我们将关注与Kafka使用者特别相关的三个主题：  
* Kafka备份原理
* Kafka如何处理生产者和消费者请求
* Kafka存储原理，比如文件格式和索引  

深入理解这三个主题对优化Kafka特别有用——理解这些机制目的是可以长时间更好的使用Kafka，而不是随便摆弄它。  

### 集群成员  
Kafka使用Apache Zookeeper维护集群的当前成员broker的列表。每个broker都有一个唯一标识，该标识要么在broker配置文件中进行配置，要么自动生成。每当一个broker进程启动，它会使用它的ID向Zookeeper注册一个临时的节点。Kafka的不同组件都会订阅Zookeeper中broker的注册路径/brokers/ids，这样当增加或者删除broker的时候它们就会获得通知。  

如果你尝试使用同一个ID去启动另一个broker，那么你将会获得一个错误——the new broker will try to register, but fail because we already have a Zookeeper node for the same broker ID(新的broker尝试去注册，但是失败了，因为我们已经拥有一个具有相同broker ID的Zookeeper节点)。  

当一个broker与Zookeeper失去了连接，



### Controller  




### 备份  




### 请求处理  



#### 生产请求  



#### 拉取请求  




#### 其他请求  




### 物理存储  




#### 分区分配  






#### 文件管理  




#### 文件格式  




#### 索引  





#### 合并  





#### 合并机制  




#### 删除事件  




#### Topic合适合并
