# MapReduce深度分析：

***

### MapReduce总结构分析：



#### 数据流向分析：

因为它是一个并行数据处理框架，所以数据流向基本算是最重要的系统问题。

![数据流向分析](/resources/数据流向分析.jpg)

1. **输入文件从HDFS流向到Mapper结点**。（一般情况下，Mapper运行节点就是数据存储节点，尽量使存储靠近计算。）但用户数据在整个集群中往往分布不均匀，因此某些计算节点需要从其他存储节点获取数据。**Hadoop会从距离计算节点最近的数据副本传输。（会有数据拷贝，但是少）**
2. **Mapper输出到内存缓冲区。**输出的是处理后的<k,v>对，缓冲区到达一定阈值时，数据以临时文件形式写入本地磁盘（**不每次都IO**），当整个map任务处理完时，**合并所有临时文件**，产生最终输出文件。**partition()就发生在这个阶段**。
3. **从内存缓冲区写到本地磁盘。**（默认缓冲区100MB，溢写比例0.8）当缓冲区数据80MB时溢写线程锁定这80MB数据，向本地磁盘写（**称为Spill溢写**）并且会对这80MB数据依据key的序列化字节进行**排序**。如果用户设置了Combiner，会在溢写前进行规约，减少溢写的数据量。
4. **从Mapper端的本地文件系统流向Reduce端**。也即Reduce中的Shuffle阶段。
   1. 对多个Reduce情况，如果Reduce在远程，将相应分区Region复制到远端
   2. 如果本机有Reduce槽位，即向本机Reduce缓冲区写（缓冲区到阈值，也写入本地）
   3. 本机Reduce可能接收其他Mapper的输出。
5. **从Reduce端内存缓冲区流向Reduce端的本地磁盘**。也即Merge和Sort阶段。将内存文件合并，磁盘文件合并，以Key进行排序，最后形成相同排序的Key的value聚集，并输出。
6. **之后直接流向Reduce()函数处理**。
7. **完毕后写入HDFS中**。



***

#### 处理流程分析：

数据流向也是处理的过程，涉及到的组件包括：MapTask、ReduceTask、JobTracker、TaskTracher、JobClient、JobInProgress、TaskInProgress等

![处理流程分析](/resources/处理流程分析.jpg)

1. 用户的应用通过JobClient类提交到JobTracker；在JobClient类中，会将用户应用程序的Mapper类、Reducer类以及配置JobConf**打包成JAR文件**，并保存在HDFS上。JobClient提交作业时会把JAR文件的**路径**一起提交到JobTracker的master服务（**即作业调度器**）。
2. 作业提交后，**JobTracker**创建一个**JobInProgress来跟踪和调度这个作业**，并将其**添加到调度器队列**中。
3. **JobInProgress**会根据JAR文件中数据集的信息，创建相应数量的TaskInProgress用于监控和调度MapTask；同时再创建指定书目的TaskInProgress来监控调度ReduceTask。（和之前的不同，默认为1个）
4. JobTracker启动任务时通过TaskInProgress来启动作业，这时会把Task对象（MapTask和ReduceTask）序列化写入相应的TaskTracker服务中。
5. （之前都是在JobTracker中）TaskTracker收到后，创建新的TaskInProgress（这个实在TaskTracker中）用于监控和调度运行该Task。
6. 启动具体Task进程，**TaskTracker**通过TaskInProgress管理的**TaskRunner对象来运行Task**。
7. **TaskRunner加载JAR文件**，配置好环境后，启动独立的Java子进程来执行Task，**首先调用MapTask。**
8. MapTask**先调用Mapper**，生成中间键值对，如果用户还定义了Combiner，再调用Combiner，把相同key的value做归约处理，减少Map输出的键值对集合。
9. MapTask的任务**全部完成后**，TaskRunner**再调用ReduceTask**进行来启动Reducer，注意MapTask和ReduceTask**不一定**运行在同一个TaskTracker节点中。（全都由TaskRunner调用，只不过顺序分先后。）
10. ReduceTask直接调用Reducer类处理Mapper的输出结果，生成最终结果，写入HDFS中。



****

### MapTask实现分析：

MapTask总逻辑流程：

![MapTask流程图](/resources/MapTask流程图.png)

1. Read阶段，通过RecordReader对象，**对HDFS文件进行split切分**，调用用户指定的**输入文件格式类**来解析每一个split文件，**输出<k,v>对**。
2. Map阶段
3. Collector和Partitioner阶段，Collector对象**收集Mapper输出**，并在OutputCollector函数内部对键值对进行**Partitioner分区**，以便确定相应的Reducer处理。最终输出到内存缓冲区。
4. Spill阶段，包括Sort和Combiner，缓冲区到达阈值**溢写，同时进行排序**。（设置了Combiner则调用Conbiner()函数）
5. Merge阶段，对Spill生成的本地磁盘中的**小文件进行多次合并**。

