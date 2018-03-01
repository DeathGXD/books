


### Spark应用的生命周期(Spark内部)  




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
Spark中的stage由task构成。每个task相当于一个运行在一个单独的数据块的组合和一组转换
