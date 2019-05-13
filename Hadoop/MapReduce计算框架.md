### MapReduce计算框架：

​	Hadoop MapReduce是一个使用简捷的软件框架，是Google云计算模型MapReduce的Java开源实现，基于它写出的程序可以运行在由上千台普通机器组成的大型集群系统中，并以一种可靠的、容错的方式并行处理上T级别的数据集。

​	过程：输入数据集分成若干块；Map任务完全并行处理（数据块切分，机器数众多）；将输出排序后交由Reduce任务处理；将输出结果储存在文件系统中。

​	**<u>MapReduce框架和分布式文件系统是运行在一组相同的节点上，即计算节点和存储节点通常在一起。（考虑调度的速度）</u>**

​	MapReduce框架也是主从结构，单一JobTracker（负责调度一个作业的所有任务，监控，任务重启等等）和多个TaskTracker（任务的执行）节点共同组成。

​	

​	MapReduce计算框架**本质上是一种基于磁盘的批处理并行计算系统**。



***

#### MapReduece模型：



##### MapReduce编程模型：

Map函数：处理输入的键值对，并产生一组中间的键值对

MapReduce框架收集所有键值相同的中间键值对（迭代器来传递），并发送给Reduce函数处理

Reduce函数：处理中间键值对，相关的中间键值对合并，形成较小的集合

![MapReduce编程模型](/resources/MapReduce编程模型.png)



***

##### MapReduce实现原理：

- **步骤1：**把输入文件**分成M块**（每一块大小Hadoop默认是64M，可修改）。
- **步骤2：**master选择**空闲的**worker节点，把总共M（对应文件块M）个Map任务和R（中间过程产生）个Reduce任务分配给他们，如上图中（2）所示。
- **步骤3：一个分配了Map任务的worker读取并处理输入数据块**。从数据块中解析出key/value键值对，把他们传递给用户自定义的Map函数，由Map函数生成并输出中间key/value键值对，**暂时缓存在内存中**，如上图中（3）所示。
- **步骤4：**缓存中的key/value键值对通过**分区函数**分成R个区域，之后**周期性地写入本地磁盘上**。并把本地磁盘上的存储位置回传给master，由**master负责把这些存储位置传送给Reduce worker**，如上图中（4）所示。
- **步骤5：**当Reduce worker接收到master发来的存储位置后，使用**RPC协议**从Map worker所在主机的磁盘上读取数据。在获取所有中间数据后，通过**对key排序**使得相同具有key的数据聚集在一起。如上图中（5）所示。
- **步骤6：**Reduce worker程序对排序后的中间数据进行遍历，对每一个唯一的中间key，Reduce worker程序都会将这个key对应的中间value值的集合传递给用户自定义的Reduce函数，完成计算后输出文件（每个Reduce任务产生一个输出文件）。如上图中（6）所示。
- **步骤7：**输出至R个文件中（每个Reduce任务产生一个输出文件，由用户指定存储目录），一般不合并，为了继续做下一个MapReduce任务的输入。

![MapReduce实现原理图](/resources/MapReduce实现原理图.jpg)



***

##### 计算流程与机制：

###### 作业提交流程：

- 步骤1：**命令行提交。**用户使用Hadoop命令行脚本提交MR程序到集群。
- 步骤2：**作业上传。**这一步骤包括了很多**初始化工作**，如获取用户作业的JobId，创建HDFS目录，上传作业、相关依赖库等到HDFS上。（自动完成）
- 步骤3：**产生切分文件。**
- 步骤4：**提交作业到JobTracker。**（创建JobInProgress对象，**跟踪作业的状态和进度**；**检查**用户是否具有向指定队列提交作业的**权限**）



###### Mapper：

1. Mapper的输入文件位于HDFS上，InputSplit对输入数据进行切分；
2. 每一个Split对应一个Mapper任务；
3. 通过RecordReader对象从输入分块中**读取并生成**<k,v>；
4. 执行Map函数，输出中间键值对，**partition()函数区分并写如缓冲区**，同时调用**sort()进行排序**。

**combiner将中间输出在本地聚集**，通过压缩等方法，减少Mapper到Reducer的数据传输量。

patition()将不同<k,v>映射到不同的Reducer。

**这里的工作都是针对同一个Mapper。**

![Mapper流程](/resources/Mapper流程.jpg)



###### Reducer：

​	**主要三个阶段：Shuffle、Sort、Reduce**

1. Shuffle：Mapper输入已经排好序了，这个阶段是如何有效地传送到Reduce端，进行大量的复制操作，也称为数据复制阶段。
2. Sort：按照Key值对Reducer的输入进行分组（**针对不同Mapper**，不同Mapper可能会形成相同Key）。**和Shuffle阶段同时进行**。基于内存（Quick？）和基于磁盘的混合排序，需多次Merge。
3. Reduce：对上述操作的输出<key,(list of values)>，调用Reduce()函数处理（每一个<key,(list of values)>，调用一次）。

![Reducer流程](/resources/Reducer流程.jpg)



###### Reporter和OutputCollector：

Reporter：报告进度，设定应用级别的状态信息，更新计数器。

​	**避免超时无法及时通知而被杀死（因为数据量大，处理时间长，可能超时，但其实仍正常工作）**

OutputCollector：收集Mapper和Reducer输出，**已被Context类代替**。





***

##### MapReduce输入输出格式：

输入格式：

​	InputFormat为作业描述输入的细节规范：

1. 检查作业输入的有效性。
2. 把输入文件切分成多个逻辑InputSplit实例，并把每一个实例分发给一个Mapper。（FileSplit继承InputSplit基类，通过write(DataOutput out) 和readFields(DataInput in)实现序列化和反序列化）。
3. 提供RecordReader的实现。（负责处理边界情况；把数据表示成<k,v>格式，方便Mapper处理）

输出格式：

​	OutputFormat主要作用是检验和写出：

1. 检验作业的输出。
2. 检验结果类型是否和Config中配置的一致。
3. 提供一个RecordWriter的实现。（write()把输出结果写到FileSystem中；close()关闭）



***

##### 核心问题：

Map和Reduce的数量：1没并行，太多资源开销大

Map的数量：有Hadoop集群的DFS块大小确定，即输入文件的总块数。

​	Map数量并行规模大致一个Node结点10~100个；算上初始化时间，每个Map执行至少应超过1分钟。

​	Map数量本质上是由splitSize决定的：

```C++
splitSize = max(minSize, man(goalSize, blockSize));
minSize = max(job.getLong("mapred.min.split.size", 1), minSplitSize);
goalSize = totalSize/(numSplits == 0 ? 1: numSplits);
//numSplits是用户设置的map数目
//totalSize是文件总大小
//mapred.min.split.size用户自定义的最小切分大小
```



**Reduce结点资源相对缺少，同时运行速度比较慢。**

reduce任务的个数：0.95或1.75 * 节点数 * mapred.tasktracher.tasks.maximum参数值

0.95则所有reduce任务同时进行

1.75则分两批执行

增加reduce的数量，可改善负载均衡，降低任务失败可能，但会增加系统资源的开销。

reduce数量可以为0，则不规约，直接存储。



###### 容错机制：

TaskTracker容错检测通过心跳检测，黑名单，灰名单机制来对失效的TaskTracker节点进行及时处理。

再执行：失败的任务重新调度（至其他节点）执行一次（任务失败执行任务，作业失败执行全部作业）

推测式执行：“落伍者”（一个任务进度远慢于其他任务。被认定为落伍者）调度到其他节点重新执行（两个同时执行，取先完成的）。



###### 作业调度：

作业调度器是一个可插拔组件，相对独立。

负责分配底层计算资源。

FIFO调度器、容量调度器、公平调度器。





***

计数器（作业、MapReduce、文件系统）、分布式缓存（缓存文本，档案，jar包等）、数据压缩。。。不记了，没意思。