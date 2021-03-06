


### Spark应用的架构  



#### 执行模式  


##### Cluster模式  


##### Client模式  


##### Local模式  




### Spark应用的生命周期(Spark外部)  



#### 客户端请求  



#### 启动  



#### 执行  



#### 完成  





### Spark应用的生命周期(Spark内部)  


#### SparkSession  



##### SparkContext  



#### 逻辑指令  



##### 逻辑指令到物理执行  




#### Spark Job  
通常，一个action操作对应着一个Spark job。Action操作总是会返回计算结果。每个job都会分割成一系列stage，stage的数量取决于需要发生多少shuffle操作。  

这个job会分割成为下列的stage和task：
* Stage 1 with 8 Tasks
* Stage 2 with 8 Tasks
* Stage 3 with 6 Tasks
* Stage 4 with 5 Tasks
* Stage 5 with 200 Tasks
* Stage 6 with 1 Tasks  

我希望大家至少要关注我们是如何得到这些shuffle数量的，那么我们就将时间花在更好的理解内部的细节上了。  

#### Stage  
Spark中的Stage表示的是一组可以被一起执行的task，这些任务在多个机器上可以执行相同的操作。通常，Spark会尝试将尽可能多的任务打包到相同的stage(在job内部尽可能多的转换操作)，但是执行引擎只有在shuffle操作之后才会开始新的stage。shuffle操作表示的是数据物理上的重新分区——比如，对一个DataFrame进行排序，或者对从文件中通过key装载的数据进行分组(用于那些需要将相同key的记录发送到同一个节点上)。这种类型的再分区需要协调executor之间进行传输数据。Spark会在每个shuffle后开始一个新的stage，并且持续跟踪stage计算到最终结果的顺序。  

在我们之前看到的job中，开始的两个stage相当于你执行了range为了创建DataFrame。默认情况下，当你使用range创建DataFrame时，它会有8个分区。接下来是重分区，通过shuffle数据来改变分区的数量。这些DataFrame会被shuffle成6个和5个分区，相当于stage 3和stage 4中task的数量。  

stage 3和stage 4在那些DataFrame上执行，最后的stage是一个join(一个shuffle操作)。忽然，我们的拥有了200个task。这是因为Spark SQL的一个配置，spark.sql.shuffle.partitions，默认值是200，这意味着在任务运行期间一个shuffle操作执行后，默认会输出200个shuffle分区。你可以更改这个值，那么输出分区的数量也会随之改变。  

**提示**：在第19章，我们会涉及到更多有关分区数的细节，因为它是一个非常重要的参数。这个值应该根据你集群CPU核心数进行设置，进而保证任务高效的执行。下面是如何进行设置：  
```scala
spark.conf.set("spark.sql.shuffle.partitions", 50)
```  

一个很好的做法是分区的数量应该大于你集群中executor的数量，可能的因素取决于你的工作负载。如果你在本地的机器上运行代码，那么理应将这个值设置的比较低，因为你本地的机器不太可能并行的执行较大数量的任务。这就是默认情况下一个集群会有更多的CPU core供executor来使用。不管分区的数量是多少，整个stage的计算都是并行的。最终的结果都会合并那些单独的分区，在发送最终结果给driver之前，会将所有分区合并为一个单独的分区。我们将会在本书的部分课程中多次看到这个配置。  

#### Task  
Spark中的stage由task构成。每个task相当于运行在一个单独的executor上的数据块的组合和一组转换。如果在数据集中只有一个大的分区，那么将会只有一个task。如果有1000个小的分区，那么我将会有1000个task并行的执行。一个task只是一个应用在一个数据单元(分区)的计算单元。将数据分割到一个合适的分区数，意味着可以并行的执行更多的任务。这并非是万能的，但的确是优化最简单的方式。  

### 执行细节  
Task和stage中有很多重要的属性，在我们结束这一章之前是很值得回顾的。首先，Spark自动完成了可以在一起完成的stage和task，比如，一个map操作可以紧接着另一个map操作之后。第二，对于所有的shuffle操作，Spark会将数据写入到可靠的存储中(比如磁盘)，可以跨多个job重复使用。下面我们会依次讨论这些概念，因为在你通过Spark UI检查应用时，它们就可能用到。  

#### 流水线  




#### Shuffle持久化  
另一个比较常见的属性是shuffle持久化。当Spark需要执行一个不得不跨节点进行数据传输的操作时，比如reduceByKey操作(输入的相同key的数据需要从不同的几点汇聚在一起)，那么执行引擎不会在执行流水线操作，取而代之就是跨网络的shuffle操作。Spark在执行stage期间进行shuffle，总是首先将"源"task发送的数据以shuffle文件的形式写到task所在的本地磁盘。然后，执行分组和规约的stage会启动并运行task从shuffle文件拉取对应的记录执行计算(比如拉取并处理特定范围key的数据)。保存shuffle文件到磁盘，使得Spark运行这个stage作为源stage(如果没有足够的executor同时运行两个stage)，并且也可以让执行引擎重新启动失败的没有重新运行所有输入任务的reduce任务。  

你会看到shuffle持久化一个显而易见的作用是，在已经shuffle的数据上运行新的job，不需要重新运行shuffle的源程序。因为shuffle文件早已写入到磁盘，Spark知道可以使用它们运行job后面的stage，而不需要重做更早的stage。在Spark UI和logs中，你将会看到之前的shuffle stage被标记为skipped。

























### 总结  
