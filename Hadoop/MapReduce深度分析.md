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

##### MapTask总逻辑流程：

![MapTask流程图](/resources/MapTask流程图.png)

1. Read阶段，通过RecordReader对象，**对HDFS文件进行split切分**，调用用户指定的**输入文件格式类**来解析每一个split文件，**输出<k,v>对**。
2. Map阶段
3. Collector和Partitioner阶段，Collector对象**收集Mapper输出**，并在OutputCollector函数内部对键值对进行**Partitioner分区**，以便确定相应的Reducer处理。最终输出到内存缓冲区。
4. Spill阶段，包括Sort和Combiner，缓冲区到达阈值**溢写，同时进行排序**。（设置了Combiner则调用Conbiner()函数）
5. Merge阶段，对Spill生成的本地磁盘中的**小文件进行多次合并**。



**各个阶段分别创建了什么对象，调用什么方法处理，在这就不记录了，太细节的东西，记也记不住，反而抓住脉络，观其大略就好。**

***

##### ReduceTask总逻辑流程：

![ReduceTask流程图](/resources/ReduceTask流程图.png)

这本书的内容有点老，好像之后发现都是不太一样的。

1. Shuffle阶段（也称Copy阶段），运行Reducer的TaskTracker需要从**各个Mapper节点远程复制属于自己处理**的一段数据。
2. Merge阶段，复制时从**很多Mapper节点**复制同一partition段的数据，需要多次合并，**以防ReduceTask节点内存中使用太多或小文件过多**。
3. Sort阶段，经过Shuffle和Merge后**不一定有序**<k,v>，在Reduce端进行**多轮归并排序**。
4. Reduce阶段，**Reduce任务的输入须<k,v>是排序的**，因此只能在Sort阶段**完成后才可调用**。





##### Shuffle阶段：

![Shuffle阶段](/resources/Shuffle阶段.png)

- 1）maptask 收集我们的 map()方法输出的 kv 对，放到内存缓冲区中
- 2）从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件
- 3）多个溢出文件会被合并成大的溢出文件
- 4）在溢出过程中，及合并的过程中，都要**调用 partitioner 进行分区和针对 key 进行排序**
- 5）reducetask 根据自己的分区号，去各个 maptask 机器上取相应的结果分区数据
- 6）reducetask 会取到同一个分区的来自不同 maptask 的结果文件，reducetask 会将这些
- 文件再进行合并（归并排序）
- 7）**合并成大文件后，shuffle 的过程也就结束了**，后面进入 reducetask 的逻辑运算过程
- （从文件中取出一个一个的键值对 group，调用用户自定义的 reduce()方法）



补充：只要一个Map任务完成，就开始复制，多线程并行复制，默认开5个线程。

copy和merge包括Sort在执行时基本是同时进行的，但在逻辑上分成三个阶段。



***

##### JobTracker分析：

​	JobTracker是MapReduce的Master，是Hadoop中最重要的后台守护进程之一。**JobTracker只有一个。**所有用户作业都由他调度并分配给TaskTracker执行。

主要功能（启动多个服务和多个线程来维护）：

- 管理任务调度
- 管理TaskTracker
- 监控作业执行
- 运行作业容错机制



##### TaskTracker分析：

​	是slave节点，运行在DataNode上，主要功能是执行JobTracker分发的任务。启动在JobTracker之后。



***

##### 心跳机制（用于通信）：

HDFS的NameNode和DataNode之间

MapReduce的JobTracker和TaskTracker之间

1. Master/Slave之间用过一个周期性信号进行通信。
2. 在Hadoop的Master启动的时候回开启一个**IPC（进程间通信）Server**以**等待**Slave的心跳数据包。
3. Slave启动时会主动连接Master，并周期性（3s）向Master发送一个心跳数据包，将自己的状态告诉Master，然后通过从Master传回的返回的值来执行命令。
4. JobTracker通过记录responseID并发送给TaskTracker，对比收到的包+1，来确定收到的心跳包是否丢失。类似建立连接。。。



***

##### 作业创建分析：

作业的创建主要由JobClient类负责完成。同时负责提交作业、跟踪作业状态、访问子任务报告、日志等、获取MapReduce集群状态信息等任务。



JobClient创建作业时：

1. 检查输入/输出的有效性，
2. 计算作业的Splits，
3. 复制作业的jar包和配置文件到HDFS的Mapred系统目录，
4. 提交作业给JobTracker并跟踪作业状态。





##### 

